# Problemática: juego multijugador en línea

## Escenario

Nébula Games está preparando un juego multijugador en línea con usuarios conectados desde distintas regiones. Cada partida se ejecuta en un servidor de juego asignado por un servicio de matchmaking. Para que los movimientos se sientan fluidos, el cliente envía la posición actual del jugador varias veces por segundo. Si una actualización aislada se pierde, el juego puede continuar con la siguiente; recibir una posición reciente suele ser más útil que esperar la retransmisión de una posición anterior.

Para estudiar el recorrido de los datos, el equipo construyó en Cisco Modeling Labs el siguiente laboratorio:

`PC1 — SW1 — R1 — R2 — SW2 — SRV1`

PC1 representa a un jugador; SRV1 representa un servidor de partida. R1 y R2 interconectan dos LAN distintas. Los switches proporcionan conectividad dentro de cada LAN. La topología permite observar el tráfico que cruza el enlace entre routers.

## Datos conocidos

| Equipo/interfaz | Dirección o red | Función |
|---|---|---|
| PC1 `eth0` | `192.168.10.10/24` | Cliente del juego |
| R1 `GigabitEthernet0/0` | `192.168.10.1/24` | Gateway de PC1 |
| R1 `GigabitEthernet0/1` | `10.0.12.1/30` | Enlace hacia R2 |
| R2 `GigabitEthernet0/1` | `10.0.12.2/30` | Enlace hacia R1 |
| R2 `GigabitEthernet0/0` | `192.168.30.1/24` | Gateway de SRV1 |
| SRV1 `eth0` | `192.168.30.10/24` | Servidor del juego |
| Aplicación | UDP destino `27015` | Puerto del servicio de juego |

PC1 utiliza como gateway `192.168.10.1`; SRV1 utiliza `192.168.30.1`. Para los ejemplos de la actividad, se supondrá que el sistema operativo de PC1 seleccionó el puerto UDP local `53000`. Ese puerto es un ejemplo: normalmente el cliente puede usar un puerto efímero diferente.

R1 debe tener una ruta hacia `192.168.30.0/24` por `10.0.12.2`. R2 debe tener una ruta de retorno hacia `192.168.10.0/24` por `10.0.12.1`.

## Situación problemática

Durante una prueba, el servidor recibe varias actualizaciones de posición, pero el jugador informa que no siempre obtiene una confirmación. En los mensajes de aplicación se observan números de secuencia con saltos. El equipo todavía no sabe si se perdió un datagrama en la red, si falta una ruta de retorno, si el servidor no respondió o si la aplicación no recibió a tiempo la confirmación.

No se debe concluir que todo salto de secuencia demuestra un fallo de router: el cliente puede descartar una actualización deliberadamente, el paquete puede perderse en cualquier punto o la respuesta puede no llegar al cliente. UDP no retransmite por sí solo los datagramas perdidos.

## Trabajo solicitado

1. Completa una tabla con IP/puerto de origen y destino para un datagrama de PC1 a SRV1 y para una posible respuesta.
2. Describe el recorrido de ida y de vuelta, indicando qué router reenvía el paquete y qué gateway utiliza cada host.
3. Escribe las rutas estáticas necesarias en R1 y R2. Explica por qué una ruta de ida sin ruta de retorno no basta para completar el intercambio.
4. Explica qué identifican las direcciones IP, los puertos UDP y el socket del servidor que escucha en `0.0.0.0:27015`.
5. Compara UDP y TCP para actualizaciones frecuentes de posición y justifica por qué una aplicación de juego podría elegir UDP.
6. Si no llega una confirmación, ordena comprobaciones que permitan distinguir: direccionamiento o gateway incorrecto, ruta ausente, servidor sin socket escuchando y pérdida de datagramas.
7. Diseña una prueba con números de secuencia para comparar funcionamiento normal y una pérdida simulada. Indica métricas que registrarías y explica por qué la pérdida simulada en la aplicación no es una medición de pérdida real de la red.
8. Explica cómo una arquitectura con servidores distribuidos y un servicio de matchmaking ayuda a atender jugadores de distintas regiones y qué problemas de coordinación/escalabilidad introduce.

## Evidencias a entregar

- Tabla de direccionamiento y puertos.
- Diagrama o descripción del camino de los paquetes.
- Configuración/rutas y comandos de verificación.
- Evidencia de conectividad mediante ping/traceroute y, si es posible, captura filtrada por `udp.port == 27015`.
- Comparación de las métricas de la prueba sin pérdida y con pérdida simulada.
- Una conclusión que separe los hechos observados de las hipótesis.
