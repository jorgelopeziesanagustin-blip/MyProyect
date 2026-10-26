# Solución: juego multijugador en línea

## 1. Direccionamiento y puertos

| Sentido | IP origen | Puerto origen UDP | IP destino | Puerto destino UDP |
| --- | --- | ---: | --- | ---: |
| Actualización PC1 → SRV1 | `192.168.10.10` | `53000` (ejemplo efímero) | `192.168.30.10` | `27015` |
| Confirmación SRV1 → PC1 | `192.168.30.10` | `27015` | `192.168.10.10` | `53000` |

Las direcciones IP identifican los extremos de capa de red. Los puertos UDP ayudan al sistema operativo a entregar cada datagrama al proceso adecuado. El socket del servidor se asocia al protocolo UDP y al puerto local `27015`; enlazarse a `0.0.0.0` significa escuchar en todas las interfaces IPv4 locales, no que `0.0.0.0` sea la IP de destino que usa PC1.

El puerto `53000` es sólo un ejemplo para describir la conversación. En la práctica el puerto de origen suele ser asignado dinámicamente y debe confirmarse en la captura.

## 2. Recorrido de los datagramas

**Ida:** PC1 reconoce que `192.168.30.10` no pertenece a su red local `192.168.10.0/24`, por lo que envía el paquete a su gateway `192.168.10.1` (R1). R1 consulta su tabla de rutas y reenvía hacia el siguiente salto `10.0.12.2` (R2) por el enlace `10.0.12.0/30`. R2 reconoce la red `192.168.30.0/24` como conectada y reenvía el datagrama por `GigabitEthernet0/0`, a través de SW2, hasta SRV1.

**Vuelta:** SRV1 envía la respuesta a su gateway `192.168.30.1` (R2). R2 usa la ruta hacia `192.168.10.0/24` vía `10.0.12.1` (R1). R1 entrega el paquete en su red local a PC1.

En cada enlace Ethernet enrutado se usan direcciones MAC propias del enlace siguiente. Sin NAT, las IP origen/destino se conservan de extremo a extremo.

## 3. Configuración de routers

La topología tiene estos puertos: en R1, `GigabitEthernet0/0` mira a SW1/LAN jugadores y `GigabitEthernet0/1` mira a R2. En R2, `GigabitEthernet0/1` mira a R1 y `GigabitEthernet0/0` mira a SW2/LAN servidores.

### R1

```text
enable
configure terminal
hostname R1
no ip domain lookup
interface GigabitEthernet0/0
 description LAN jugadores 192.168.10.0/24
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit
interface GigabitEthernet0/1
 description Enlace a R2 10.0.12.0/30
 ip address 10.0.12.1 255.255.255.252
 no shutdown
exit
ip route 192.168.30.0 255.255.255.0 10.0.12.2
end
```

### R2

```text
enable
configure terminal
hostname R2
no ip domain lookup
interface GigabitEthernet0/1
 description Enlace a R1 10.0.12.0/30
 ip address 10.0.12.2 255.255.255.252
 no shutdown
exit
interface GigabitEthernet0/0
 description LAN servidores 192.168.30.0/24
 ip address 192.168.30.1 255.255.255.0
 no shutdown
exit
ip route 192.168.10.0 255.255.255.0 10.0.12.1
end
```

Las LANs directamente conectadas aparecerán como rutas conectadas. R1 necesita la ruta a la LAN de SRV1 y R2 necesita la ruta de retorno a la LAN de PC1; sin la ruta de retorno, el servidor podría recibir la solicitud pero la respuesta no sabría cómo volver a PC1.

## 4. Configuración de los hosts

La topología define la configuración de PC1 y SRV1 al arranque. En Alpine, las órdenes equivalentes para configurar temporalmente las interfaces son:

### PC1

```sh
ip addr show dev eth0
ip route
ping -c 4 192.168.10.1
ping -c 4 192.168.30.10
traceroute 192.168.30.10
```

### SRV1

```sh
ip addr show dev eth0
ip route
ping -c 4 192.168.30.1
ping -c 4 192.168.10.10
traceroute 192.168.10.10
```

Si se necesita volver a configurar manualmente una interfaz:

```sh
# PC1
ip addr flush dev eth0
ip addr add 192.168.10.10/24 dev eth0
ip link set dev eth0 up
ip route replace default via 192.168.10.1

# SRV1
ip addr flush dev eth0
ip addr add 192.168.30.10/24 dev eth0
ip link set dev eth0 up
ip route replace default via 192.168.30.1
```

## 5. Cómo comprobar la solución

En ambos routers:

```text
show ip interface brief
show ip route
show running-config
```

Se espera que las interfaces usadas estén `up/up`; R1 debe mostrar la ruta estática a `192.168.30.0/24` vía `10.0.12.2`, y R2 la ruta a `192.168.10.0/24` vía `10.0.12.1`. Validar los pings en orden: gateway local, vecino del enlace `10.0.12.x`, host remoto. Un traceroute PC1→SRV1 debería mostrar los routers intermedios (R1 y R2), sujeto al comportamiento de respuesta ICMP de las imágenes.

Un ping exitoso prueba conectividad IP/ICMP; por sí solo no prueba que el proceso UDP del juego esté escuchando ni que el protocolo de aplicación funcione.

### Prueba de UDP

En SRV1 debe ejecutarse un programa de servidor UDP que cree un socket, haga `bind(0.0.0.0, 27015)`, reciba datagramas y responda al par IP/puerto de origen. En PC1 debe ejecutarse el cliente con destino `192.168.30.10:27015`. Los scripts de juego no están instalados automáticamente en IOS ni necesariamente en la imagen Alpine; se pueden copiar/instalar en hosts Linux compatibles o ejecutar en máquinas externas conectadas a las LAN. No afirmar recepción UDP hasta observarla en el servidor o en la captura.

Para una captura en el enlace R1–R2, iniciar una captura de paquetes en CML y generar tráfico durante la captura. En Wireshark usar:

```text
udp.port == 27015
```

La captura debería permitir observar datagramas en ambos sentidos, direcciones IP, puertos y tiempo. El puerto cliente real puede no ser `53000`; usar el valor observado. Una captura vacía no demuestra por sí sola que no hubo tráfico si se eligió el enlace equivocado, la captura no estaba activa o el filtro no coincide.

## 6. UDP frente a TCP

UDP envía datagramas sin establecer una conexión y no garantiza entrega, orden ni retransmisión. Tiene menor sobrecarga y permite a la aplicación decidir qué hacer con los datos perdidos. Para posiciones frecuentes, descartar una posición antigua y usar la más reciente puede dar una sensación más fluida que esperar retransmisiones.

TCP proporciona un flujo de bytes fiable y ordenado, con retransmisiones y control de congestión. Es apropiado cuando no se puede perder información, pero la entrega ordenada puede hacer que datos posteriores esperen por segmentos anteriores. UDP tampoco garantiza por sí mismo baja latencia ni hace fiable una aplicación; la aplicación debe diseñar sus confirmaciones, secuencias y tolerancia a pérdida.

## 7. Diagnóstico cuando falta una respuesta

1. Revisar `show ip interface brief` y `ip addr show`: interfaces activas, IP y máscara correctas.
2. Revisar `ip route` en los hosts: gateway por defecto correcto.
3. Revisar `show ip route` en R1 y R2: ruta directa y ruta de retorno presentes.
4. Probar ping por tramos y traceroute para acotar dónde se interrumpe la conectividad IP.
5. Verificar en SRV1 que el proceso UDP está escuchando en `0.0.0.0:27015` y que el firewall local no descarta los datagramas.
6. Capturar en el enlace R1–R2 y, si es posible, en el lado del cliente/servidor. Si se ve la solicitud entrar en el enlace y no la respuesta salir, investigar servidor/socket y ruta de retorno; si la respuesta cruza el enlace hacia PC1 pero no aparece en el cliente, revisar gateway/firewall/socket cliente.
7. Correlacionar números de secuencia, marcas de tiempo y confirmaciones. Un salto de secuencia muestra que la aplicación no observó esa actualización, pero no identifica dónde se perdió ni demuestra un fallo de router.

## 8. Prueba controlada de pérdidas

Enviar, por ejemplo, 1000 actualizaciones numeradas a una frecuencia fija (10 por segundo) y registrar: mensajes intentados, descartes intencionales del cliente, datagramas recibidos por el servidor, confirmaciones recibidas, secuencias ausentes y latencia de ida y vuelta. Repetir primero sin pérdida simulada y después con una probabilidad de descarte configurada en el cliente. Comparar las métricas y observar que la secuencia más nueva puede seguir actualizando la posición aunque falte una anterior.

La pérdida simulada en el cliente ocurre antes de transmitir el datagrama y no es pérdida de la red. La pérdida de red debe investigarse con captura/contadores y condiciones de enlace en CML. Si se usa link conditioning para simular pérdida en el enlace, documentar explícitamente el porcentaje y recordar que sigue siendo una condición de laboratorio, no una medición de una red real.

## 9. Sistemas distribuidos y escalabilidad

Un conjunto de servidores de partida distribuidos geográficamente puede reducir distancia/latencia y distribuir usuarios/carga. Un servicio de matchmaking puede elegir una instancia por región, capacidad y estado y devolver al cliente su dirección/puerto. Cada partida debe tener autoridad clara para su estado; se necesitan mecanismos para descubrir instancias, asignar/reasignar partidas, monitorear salud, evitar sobrecarga y gestionar fallos. Distribuir servidores no hace que compartan automáticamente el estado ni garantiza consistencia o disponibilidad.

## 10. Conclusión

La ruta prevista es PC1 → gateway R1 → enlace `10.0.12.0/30` → R2 → LAN de SRV1; la respuesta usa las rutas de retorno en sentido inverso. El destino UDP de la aplicación es el puerto `27015`, mientras que el origen del cliente suele ser efímero. La explicación describe la solución lógica y los comandos para demostrarla; resultados de ping, traceroute, recepción UDP o captura sólo deben reportarse después de ejecutarlos en el laboratorio.
- 