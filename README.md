# netEspNow v1.4 — Red ESP-NOW fractal de 3 niveles

<p align="center">
  <img alt="Hardware" src="https://img.shields.io/badge/hardware-ESP8266-0b0e14?style=flat-square&labelColor=1a2130&color=6c8cd9">
  <img alt="Nodos" src="https://img.shields.io/badge/nodos-22-0b0e14?style=flat-square&labelColor=1a2130&color=4caf7d">
  <img alt="Niveles" src="https://img.shields.io/badge/niveles-3-0b0e14?style=flat-square&labelColor=1a2130&color=d4a853">
  <img alt="Transporte" src="https://img.shields.io/badge/transporte-ESP--NOW-0b0e14?style=flat-square&labelColor=1a2130&color=6c8cd9">
  <img alt="Cifrado" src="https://img.shields.io/badge/cifrado-ECC%20%2B%20PMK-0b0e14?style=flat-square&labelColor=1a2130&color=d4a853">
  <img alt="Estado" src="https://img.shields.io/badge/estado-dise%C3%B1o%20validado-0b0e14?style=flat-square&labelColor=1a2130&color=c98b6b">
  <img alt="Licencia" src="https://img.shields.io/badge/licencia-MIT-0b0e14?style=flat-square&labelColor=1a2130&color=8b8fa3">
</p>

> **Una red de radio que crece como un árbol, no como una estrella.**
> 22 ESP8266 en tres niveles, con enrutamiento por ruta explícita, descubrimiento automático de gateways, reparación de ramas caídas y una clave por nivel. Ningún nodo guarda una tabla de rutas: la topología vive en el PC y viaja en cada mensaje.

---

## ⚠️ Estado real de este repositorio

> **Esto es una arquitectura especificada, no un despliegue de 22 nodos en producción.**
>
> El diseño está cerrado: topología, mensajes, enrutamiento, claves, latencia, RAM, despliegue y pruebas. El firmware **todavía no está escrito**: las carpetas `nivel_*` son andamiaje vacío, no código compilado.

| Componente | Estado | Qué significa |
|---|---|---|
| Topología y roles | ✅ | 22 nodos, 3 células, 4 firmwares definidos |
| Formato de mensajes | ✅ | Structs empaquetados, tipos, tamaños, semántica |
| Ruta explícita | ✅ | Ida, vuelta, consumo por gateway, errores |
| Seguridad | ✅ | ECC heredado + PMK por nivel + canal dedicado |
| Latencia y RAM | ✅ | Modelo por componente, con rangos y supuestos |
| Plan de despliegue | ✅ | 4 fases, orden de flasheo, criterios |
| Dashboard de PC | 🟡 | HTML estático, sin conexión serie real |
| Los 4 firmwares | ❌ | Especificados, no implementados |
| Controlador Python y despliegue físico | ❌ | API y CLI especificadas; requiere hardware |

**Por qué publicarlo igual:** el valor no está en el código, está en el diseño. Enrutar con una ruta explícita en vez de una tabla distribuida, hacer que un ESP8266 con una sola radio viva en dos redes cifradas y documentar sin maquillaje por qué la capa ECC heredada es débil: eso es reutilizable. El código es la parte fácil; el diseño es la que se equivoca.---

## Tabla de contenidos

1. [Qué es netEspNow](#qué-es-netesponw)
2. [El problema](#el-problema-que-resuelve)
3. [Topología](#topología-3-niveles-22-nodos)
4. [Principio de funcionamiento](#principio-de-funcionamiento)
5. [Protocolo de mensajes](#protocolo-de-mensajes)
6. [Ruta explícita](#ruta-explícita)
7. [Beacons y auto-descubrimiento](#beacons-y-auto-descubrimiento)
8. [Auto-reparación](#auto-reparación-y-tolerancia-a-fallos)
9. [Seguridad en tres capas](#seguridad-en-tres-capas)
10. [Latencia](#análisis-de-latencia)
11. [RAM](#análisis-de-ram)
12. [Límites](#límites-del-sistema)
13. [Estructura](#estructura-del-repositorio)
14. [Requisitos](#requisitos)
15. [Compilación y carga](#compilación-y-carga)
16. [Despliegue por niveles](#despliegue-por-niveles)
17. [Dashboard de PC](#dashboard-de-pc)
18. [Pruebas](#pruebas)
19. [Riesgos](#riesgos-y-mitigaciones)
20. [Limitaciones y futuro](#limitaciones-y-trabajo-futuro)
21. [Cómo se construyó](#cómo-se-construyó-esto)
22. [Referencias](#referencias)
23. [Licencia](#licencia)

---

## Qué es netEspNow

netEspNow v1.4 es el diseño de una red de sensores y actuadores sobre **ESP-NOW**, el protocolo de radio de Espressif que envía tramas entre dispositivos sin punto de acceso, sin IP, sin TCP y sin DHCP: cada trama va de MAC a MAC, con acknowledgment de enlace y 250 bytes de payload máximo.

La versión anterior (v1.3) era una **estrella plana**: un Bridge al PC por USB, un Gateway, seis nodos. Funcionaba. Se desplegaron siete ESP8266, se midió un RTT de **1385 ms** y se aprendió dónde estaba el techo: agregar un nodo era fácil, agregar un nivel de anidación no. La v1.4 reemplaza la estrella por un **árbol de células**, donde un endpoint de la célula N puede además hacer de puente hacia el gateway de la célula N+1.

Las tres decisiones clave: **la ruta va en el mensaje** (un comando a E2.3 lleva `1:1:3`; cada gateway lee el primer número, lo borra y reenvía), **cada nivel tiene su clave** y **el PC es el cerebro** (la topología y el cálculo viven en Python).

---

## El problema que resuelve

En v1.3 el Gateway mantenía una lista de nodos y el Bridge traducía entre serial y radio. Dos límites duros: **peers** (ESP-NOW permite 20 por interfaz, así que 19 endpoints más un bridge ya es el techo) y **alcance** (todos los nodos deben estar al alcance del único gateway).

Poner un segundo gateway abre un problema peor: ¿a cuál mando el comando? Si no se hablan, son dos redes. Si se hablan, necesitan una tabla de rutas, y una tabla de rutas en 80 KB de RAM es una mala idea.

El reflejo de cualquier ingeniero de redes es la tabla distribuida: cada nodo guarda "para llegar a X, salí por Y". Es lo que hace IP, y también lo que hace que un nodo que se apaga provoque una inundación de actualizaciones. netEspNow toma la decisión contraria: **ningún nodo mantiene estado de topología**. El que quiere hablar calcula la ruta completa y la escribe en el mensaje; los intermedios leen, recortan y reenvían. Si la topología cambia, cambia el cálculo en el PC; los nodos no se enteran.

El costo: el emisor debe conocer la topología, cada mensaje lleva hasta 16 bytes de ruta redundantes y un nodo que cambia de padre invalida rutas cacheadas. Vale la pena para 22 nodos semi-estáticos; para una red móvil de cientos de nodos habría que ir a una malla real (BATMAN, Babel, RPL).

---

## Topología: 3 niveles, 22 nodos

| Nivel | Nodos | Composición | Clave | Rol |
|---|---|---|---|---|
| 0 | 6 | Bridge_0 + Gateway_0 + E0.1 … E0.4 | `PMK_N0` | Raíz, conectada al PC |
| 1 | 10 | Gateway_1 + E1.1 … E1.9 | `PMK_N1` | Célula intermedia |
| 2 | 6 | Gateway_2 + E2.1 … E2.5 | `PMK_N2` | Célula hoja |
| **Total** | **22** | | | |

Los 22 no son un número mágico: son el inventario real (6 de v1.3 con uno de repuesto no desplegado, 10 comprados para nivel 1, 6 para nivel 2). La arquitectura soporta más.

| Firmware | Unidades | Responsabilidad | Complejidad |
|---|---|---|---|
| `firmware_bridge_root.ino` | 1 — Bridge_0 | Serial ↔ ESP-NOW. Cifra el enlace ECC. **No interpreta rutas.** | Media |
| `firmware_gateway.ino` | 3 — Gateway_0/1/2 | Distribuye comandos, consume el primer tramo, emite beacon, reenvía anuncios. | Media |
| `firmware_endpoint_bridge.ino` | 2 — E0.1, E1.1 | Doble PMK, reenvía entre niveles sin tocar la ruta. Time-division. | Alta |
| `firmware_endpoint.ino` | 16 — el resto | Ejecuta GPIO/ADC, escanea beacons, reporta a su gateway. | Baja |

El mismo binario de gateway sirve para los tres niveles: solo cambia `GW_LEVEL`. Igual con el endpoint: `MI_NIVEL` y `NODE_ID`.

```
PC ──serial── BRIDGE_0 ══ECC══ GATEWAY_0 ─┬─ E0.1 ══PMK_N1══ GATEWAY_1 ─┬─ E1.1 ══PMK_N2══ GATEWAY_2 ─┬─ E2.1
                                          ├─ E0.2                       ├─ E1.2                       ├─ E2.2
                                          ├─ E0.3                       ├─ E1.3 … E1.9                ├─ E2.3
                                          └─ E0.4                       └─ (9 endpoints)              ├─ E2.4
                                                                                                      └─ E2.5
```

Renderizado en [`docs/assets/architecture.svg`](docs/assets/architecture.svg), [`routing.svg`](docs/assets/routing.svg) y [`security.svg`](docs/assets/security.svg).

---

## Principio de funcionamiento

### La célula

Una célula es un gateway más sus endpoints directos, compartiendo PMK y canal. Es la unidad de **cifrado** (todo va con la PMK de la célula), de **direccionamiento** (el gateway conoce MAC e ID de cada endpoint) y de **descubrimiento**. Una célula no sabe cuántas hay arriba ni abajo: es la misma estructura repetida a distintas profundidades.

### El ciclo de un comando

Para encender el LED de E2.3, el controlador calcula la ruta `1:1:3` y la escribe en el puerto serie:

```
PC ─[1:1:3]→ Bridge_0 ─[ECC]→ Gateway_0 ─[1:3]→ E0.1
                                                 └─[1:3]→ Gateway_1 ─[3]→ E1.1
                                                                          └─[3]→ Gateway_2
                                                                                   └─[ ]→ E2.3 EJECUTA
```

Bridge_0 cifra con ECC. Gateway_0 descifra, lee el `1`, lo quita y reenvía. E0.1 no ejecuta: cambia a PMK_N1 y entrega con la ruta intacta. Gateway_1 lee el `1`, lo quita. E1.1 cambia a PMK_N2. Gateway_2 ve un número sin dos puntos —destino local— y entrega con ruta vacía; E2.3 ejecuta. La respuesta vuelve por el camino inverso, cada nodo a su padre. Cinco saltos de radio, tres lecturas de ruta, cero tablas.

### El invariante

> **El primer número de la ruta es siempre el endpoint del nivel que acaba de recibir el mensaje.**

Si quedan más números, ese endpoint es un puente: hay que entregárselo para que lo baje. Si queda uno solo, es el destino final de ese nivel: se entrega y se vacía la ruta. Si el número no existe, se devuelve error inmediato. Ese invariante es todo el algoritmo: no hay Dijkstra, ni tablas de vecinos, ni convergencia.

### Time-division

Un ESP8266 tiene **una sola radio**. `esp_now_set_pmk()` reemplaza la clave de toda la interfaz, así que E0.1 alterna entre sus dos contextos cada 100 ms. Los mensajes que llegan en el contexto equivocado no se pierden: el callback los deja en un **buffer circular de 8 por contexto** y se procesan en el siguiente turno. Si se llena, se descarta el más antiguo.

El costo es real: **hasta 100 ms de latencia extra en el peor caso**, cuando la trama llega justo tras el cambio. Para domótica es irrelevante; para control de motores, este diseño no sirve.

---

## Protocolo de mensajes

`__attribute__((packed))` es obligatorio: sin él el compilador inserta padding y los tamaños dejan de coincidir entre firmwares compilados por separado.

```c
typedef struct __attribute__((packed)) {
  uint8_t  msg_type;        // 0 = CMD_DOWN, 1 = RESP_UP
  uint8_t  ruta_len;        // bytes válidos en ruta
  uint8_t  ruta[16];        // ruta explícita: "1:1:3" (ASCII, sin NUL)
  uint8_t  msg_id[4];       // identificador único del mensaje
  uint32_t timestamp_ms;    // timestamp del PC al emitir
  uint8_t  cmd;             // LED, READ_SENSOR, PING, RELAY...
  uint8_t  pin;             // GPIO o ID de sensor
  uint16_t val;             // on/off, duty, umbral
  uint8_t  payload[8];      // datos específicos del rol
} EspNowMessageV14;         // 38 bytes empaquetados
```

**Nota de consistencia:** la suma real es **38 bytes** (1+1+16+4+4+1+1+2+8); el diseño original hablaba de 35 B porque omitía el `timestamp_ms` o el `payload` al contar. Frente a los 21 B de v1.3 la trama crece porque ahora lleva ruta, ID y payload de rol: esa es la razón de que los endpoints viejos **no sean compatibles a nivel de trama**.

```c
// Beacon: 6 bytes empaquetados, emitido cada 2 s con la PMK del nivel
{ msg_type=2, gw_level, gw_id, pmk_id=0|1|2, uptime_s }

// ANNOUNCE_UP: 5 bytes, sube sin filtrar hasta el PC
{ msg_type=3, gw_level, gw_id, rssi(int8), hops }
```

| Tipo | Valor | Dirección | Tamaño | Semántica |
|---|---|---|---|---|
| `CMD_DOWN` | 0 | PC → Gateway → Endpoint | 38 B | Comando bajando |
| `RESP_UP` | 1 | Endpoint → Gateway → PC | 38 B | Respuesta subiendo |
| `BEACON` | 2 | Gateway → broadcast de célula | 6 B | "Existo, nivel N, PMK M" |
| `ANNOUNCE_UP` | 3 | Endpoint → gateway padre | 5 B | "Tengo un gateway debajo" |

| `cmd` | Nombre | Respuesta | `val` / `payload` |
|---|---|---|---|
| 1 | `PING` | sí | — |
| 2 | `LED` | no | `val` 0/1 |
| 3 | `READ_SENSOR` | sí | dato en la vuelta |
| 4 | `SET_GPIO` | no | `val`, `payload[0]` = GPIO |
| 5 | `READ_ADC` | sí | `uint16` LE en la vuelta |
| 6 | `RELAY` | no | `payload[0]`: 1=on, 2=off, 3=toggle |
| 7 | `CONFIG` | no | parámetros de rol |
| 8 | `STATS` | sí | contadores del nodo |

### Protocolo serial PC ↔ Bridge_0

| Comando | Ejemplo | Acción |
|---|---|---|
| `ping <id>` | `ping 2` | Ping a endpoint de nivel 0 (v1.3) |
| `set <id> <on\|off>` | `set 1 on` | GPIO de nivel 0 (v1.3) |
| `stats` / `list` | `stats` | Estadísticas / nodos conocidos |
| `auto <on\|off>` | `auto on` | Polling automático |
| `json {...}` | `json {"ruta":"1:1:3","cmd":"READ_SENSOR"}` | **Nuevo:** ruta explícita |
| `route <destino>` | `route E2.3` | Confirma la ruta calculada |

```
N1 c=6 u=34567ms i=1 a=512 o=0 rssi=-45          ← v1.3, se conserva
{"msg_id":1234,"ruta":"1:1:3","val":512,"rtt_ms":18}
{"event":"gateway_detectado","nivel":2,"puente":"E1.1","rssi":-45}
{"event":"rama_aislada","nivel":1,"motivo":"sin_puente"}
```

La compatibilidad es deliberada: el Bridge puede quedarse con firmware viejo respondiendo `ping` y `set` para el nivel 0 mientras el resto migra, lo que permite desplegar por fases.

---

## Ruta explícita

El controlador mantiene un mapa aprendido por beacons y persistido en JSON, y calcula la ruta caminando hacia arriba desde el destino:

```python
def calcular_ruta(destino):          # 'E2.3' -> '1:1:3'
    nivel, eid = parsear_destino(destino)
    tramos = []
    for n in range(nivel, 0, -1):
        tramos.insert(0, topologia[f"nivel_{n-1}"]["puente_hacia"][n])
    tramos.append(eid)
    return ":".join(map(str, tramos))
```

Para `E0.3` la ruta es `3`; para `E1.5`, `1:5`; para `E2.3`, `1:1:3`. Verificable a mano en [`docs/assets/routing.svg`](docs/assets/routing.svg).

```c
void gateway_forward(uint8_t *ruta, uint8_t ruta_len, EspNowMessageV14 *msg) {
  if (ruta_len == 0) { entregar_local(msg); return; }        // ya llegó

  int sep = buscar_dos_puntos(ruta, ruta_len);
  uint8_t id = parsear_entero(ruta, sep < 0 ? ruta_len : sep);

  peer_t *p = buscar_endpoint(id);
  if (!p) { enviar_error(msg, ERR_ENDPOINT_NOT_FOUND, id); return; }

  if (sep < 0) {                       // último tramo: entregar
    msg->ruta_len = 0;
  } else {                             // quedan tramos: recortar "N:"
    memmove(ruta, ruta + sep + 1, ruta_len - sep - 1);
    msg->ruta_len = ruta_len - sep - 1;
  }
  esp_now_send(p->mac, (uint8_t*)msg, sizeof(*msg));
}
```

El gateway **no distingue** entre endpoint simple y endpoint-bridge: para él, E0.1 es un endpoint más con un ID. Que después baje el mensaje al nivel 1 es problema del puente. Esa indiferencia es lo que hace la estructura fractal: el gateway no necesita saber que existe un nivel inferior.

### El camino de vuelta

La respuesta sube por la misma cadena, pero **no recorre la ruta al revés**. Cada nodo sabe quién es su padre: el endpoint simple conoce la MAC de su gateway, el puente conoce la de arriba y la de abajo, y el gateway conoce la del puente que lo conecta hacia arriba (Gateway_0 conoce la de Bridge_0). La vuelta es trivial: cada nodo entrega a su padre, sin álgebra de rutas ni estado por mensaje.

`RESP_UP` viaja con `ruta` copiada de la ida (para verificar simetría) y con el `msg_id` original. El PC correlaciona por `msg_id`, calcula el RTT y descarta duplicados.

Invertir la ruta en el destino (`1:1:3` → `3:1:1`) no funciona: en el nivel 1, el puente hacia arriba es un endpoint del **nivel 0** (E0.1), así que un índice relativo al nivel no alcanza. Se necesitaría una ruta absoluta con nivel incluido (`N0.1:N1.1:N2.3`), que duplica el tamaño y complica el parseo. La **memoria de padre** cuesta 6 bytes de MAC por nodo y el auto-descubrimiento ya resuelve el re-aprendizaje.

---

## Beacons y auto-descubrimiento

Cada gateway emite un beacon cada **2 segundos**, cifrado con la PMK de su nivel. Eso tiene una consecuencia elegante: **el beacon es invisible para los niveles superiores**, porque no tienen esa clave. Se evita la tormenta que habría si todos los niveles compartieran clave: cada nodo solo "ve" los de su célula.

Cada endpoint abre una **ventana promiscua cada 5 segundos**, de 200 ms:

```
guardar estado ESP-NOW → callback sin filtro de PMK → escuchar 200 ms
  si llega un beacon con gw_level > mi_nivel: enviar ANNOUNCE_UP a mi gateway
restaurar PMK y callback
```

Es la única forma de que un endpoint descubra un gateway inferior sin conocer su clave. Es una concesión consciente: durante 200 ms cada 5 s (4% del tiempo) el nodo acepta tramas de cualquier PMK, y solo procesa beacons con formato válido. Una trama malformada se descarta sin ejecutar nada.

```
Gateway_2 emite beacon
  → E1.1 lo detecta → ANNOUNCE_UP {gw_level:2, rssi:-45, hops:1} → Gateway_1
Gateway_1 registra "Gateway_2 cuelga de E1.1"
  → ANNOUNCE_UP {hops:2} → Gateway_0 → Bridge_0 → PC
  → {"event":"gateway_detectado","nivel":2,"puente":"E1.1","rssi":-45}
```

**Los anuncios se forwardean hacia arriba sin filtrar**, incrementando `hops`. Son 5 bytes: el tráfico es despreciable, y el PC recibe todo para reconstruir el mapa. Si el PC arranca tarde no hace falta estado en los ESP: se reconstruye esperando un ciclo de beacons más la propagación (el controlador persiste el JSON igual, para no esperar).

**Limitación:** el `ANNOUNCE_UP` no se autentica más allá de la PMK que lo transporta: un endpoint comprometido en nivel 1 podría anunciar un gateway falso de nivel 2. Aceptable en un despliegue de confianza; inaceptable en uno hostil.

---

## Auto-reparación y tolerancia a fallos

### Muere el endpoint-bridge

Gateway_0 deja de recibir respuestas de E0.1 y, tras **3 timeouts consecutivos** (3 × 5 s = 15 s), lo marca caído. Entonces consulta a sus otros endpoints: "¿alguno ve a Gateway_1?". Ya estaban escaneando cada 5 s, así que la respuesta está en su último reporte, y si alguno lo ve, Gateway_0 redirige el tráfico.

**Failover efectivo:** la conmutación es inmediata —un cambio de índice en la tabla de peers, sin re-asociación Wi-Fi, sin handshake, sin DHCP— pero **la detección tarda hasta 15 s**. Decir "failover de 0 ms" sería marketing: lo que es cero es el costo de establecimiento, no el tiempo de reacción.

### El gateway queda aislado

Tras 30 s sin tráfico de Gateway_0, Gateway_1 entra en modo búsqueda: pide escaneo promiscuo inmediato a sus endpoints, uno de ellos (E1.3) detecta Gateway_0 a través de E0.4, y Gateway_1 reconstruye su camino con E0.4 como padre. **Re-enraizamiento: 5 a 30 s**, y durante ese lapso los comandos al subárbol devuelven error.

### No hay redundancia

Si nadie en nivel 0 ve a Gateway_1, el sistema no miente: Gateway_0 reporta `{"alerta":"rama_aislada","nivel":1,"motivo":"sin_puente"}`, el PC marca el subárbol como `INACCESIBLE` y los comandos fallan **al instante** con error explícito, no con timeout. Gateway_1 sigue respondiendo a sus propios endpoints: una rama aislada no es una rama muerta.

### Watchdogs

| Nodo | Condición | Acción |
|---|---|---|
| Bridge_0 | 10 timeouts consecutivos con Gateway_0 | `ESP.restart()` |
| Gateway_N | 120 s sin tráfico del bridge superior | `ESP.restart()` |
| Endpoint | 120 s sin comando ni respuesta | `ESP.restart()` |

Reiniciar es barato porque no hay estado de red persistente: el nodo vuelve, escanea beacons y se reincorpora solo.

**Lo que la auto-reparación no hace:** no elige el mejor camino (elige el primero que funciona, sin métrica ni balanceo), no evita bucles (si dos gateways se ven mutuamente y ambos se re-enraizan puede formarse un ciclo; el firmware no lo previene, el PC puede detectarlo) y no repara el Bridge_0, que sigue siendo punto único de fallo total.

---

## Seguridad en tres capas

### C1 — ECC asimétrico (Bridge_0 ↔ Gateway_0)

Único enlace con clave pública, heredado de v1.3 sin cambios por compatibilidad. Usa `uECC` (ECDSA) con BearSSL disponible y protege el tramo entre el PC y la red de radio: el único punto donde se podrían inyectar comandos sin estar dentro del alcance de un endpoint. Cuesta **50–200 ms por operación**.

**Debilidades conocidas y aceptadas:** no es ECDH real (la clave compartida es un XOR, sin forward secrecy); el digest no es SHA-256 sino un XOR de 32 bytes, así que la firma es maleable; el mismo par de claves está compilado en ambos extremos; y BearSSL está configurado pero subutilizado. Se documenta en lugar de esconderse porque el objetivo de v1.4 es agregar cifrado intra-nivel sin romper el Bridge existente; arreglar C1 exige reflashear ambos extremos y cambiar la trama. Lo que sí garantiza es que un tercero en canal 1 no lea ni inyecte en el tramo PC↔Gateway_0 sin las claves: ofuscación fuerte, no criptografía robusta.

### C2 — PMK intra-nivel (nuevo en v1.4)

La mejora principal. Cada nivel tiene su **Primary Master Key** de 16 bytes, aplicada por hardware.

| Nivel | Clave | La conocen |
|---|---|---|
| 0 | `PMK_N0` | Bridge_0, Gateway_0, E0.1 … E0.4 |
| 1 | `PMK_N1` | Gateway_1, E1.1 … E1.9, y E0.1 como puente |
| 2 | `PMK_N2` | Gateway_2, E2.1 … E2.5, y E1.1 como puente |

Cifrar por nivel en lugar de por par es una ventaja de gestión: **3 claves para 22 nodos** en vez de 231 pares, y rotar una afecta a un nivel completo, no a una matriz.

```c
esp_now_set_pmk(pmk_del_nivel);   // 16 bytes, una vez al arrancar
esp_now_add_peer(&peer);          // el SDK deriva la LMK del par
```

La PMK **nunca viaja por radio**. Se graba en EEPROM al aprovisionar con magic `NS14`, la clave en el offset 4 y las MACs de los endpoints a partir del offset 20. Los endpoints la llevan compilada: es el modelo de confianza de v1.3, si tenés el binario tenés la clave.

**Qué protege:** el tráfico dentro de cada célula. Un ESP8266 capturado en nivel 2 no descifra nivel 0 ni nivel 1. Esa segmentación es lo que v1.3 no tenía.

### C3 — Canal dedicado

Todos los nodos usan el **canal Wi-Fi 1** sin asociarse a un AP: sin SSID, sin DHCP, sin IP, sin TCP. Elimina la superficie de ataque de la pila IP: no hay puerto que escanear, servicio que explotar ni handshake WPA que capturar. No oculta el tráfico: cualquiera puede capturar en canal 1, y sin la PMK no descifra, pero ve que hay red y puede intentar denegación de servicio.

### El doble contexto

E0.1 conoce dos claves y eso lo hace el nodo más sensible: si se compromete, se comprometen dos niveles. Mitigaciones: máximo **10 peers por contexto**, buffer de 8 con descarte del más antiguo, y PMK recargada desde RAM en cada cambio para no desgastar flash ni sumar latencia.

| Capa | Estado | Garantía |
|---|---|---|
| C1 ECC | Heredada, débil | Ofuscación fuerte, no criptografía robusta |
| C2 PMK | Nueva, sólida | AES-128 real aplicado por hardware |
| C3 Canal | Heredada, sólida | Elimina superficie IP, no oculta el tráfico |

C2 es el aporte real de v1.4. C1 y C3 se mantienen porque cambiarlas rompe compatibilidad y porque el modelo de amenaza —una casa, un galpón, un operador— no justifica más.

---

## Análisis de latencia

**Todos los números son estimaciones de diseño, no mediciones de este repositorio.** La única medición real es la de v1.3 (RTT 1385 ms), usada como referencia.

| Componente | Tiempo | Fundamento |
|---|---|---|
| Serial PC → Bridge_0 | < 1 ms | 115200 baud, trama serializada ~80 B |
| Cifrado ECC (Bridge_0) | 50–200 ms | uECC en ESP8266, dependiente de IRAM |
| Entrega ESP-NOW por salto | 1–3 ms | Medido en v1.3 sin cifrado |
| Cifrado PMK por salto | ~1 ms | Acelerador de hardware |
| Time-division del puente | 0–100 ms | Peor caso: trama justo tras el cambio |

| Destino | Saltos | RTT estimado |
|---|---|---|
| `E0.2` | 2 | **5–15 ms** |
| `E1.5` | 4 | **15–100 ms** |
| `E2.3` | 6 | **25–200 ms** |

La diferencia con v1.3 no es la topología, es **dónde se aplica el cifrado**. En v1.3 cada mensaje del Gateway a un nodo pasaba por el Bridge para ser cifrado, aunque el nodo estuviera a un salto, con dos operaciones ECC por comando. En v1.4 el ECC se aplica **solo en el enlace que lo necesita** y el resto usa PMK, que es casi gratis.Donde duele es el time-division: un mensaje al nivel 2 cruza **dos** puentes, cada uno con hasta 100 ms de espera, o sea hasta 200 ms que dependen de la fase del reloj y no de la distancia. Mitigaciones no implementadas: time-slice de 50 ms, sincronizar los slices de todos los puentes, o acortar el turno si hay mensajes pendientes.

No se midió la latencia real del ECC con este firmware (no existe), ni la pérdida con 22 nodos emitiendo, ni el jitter bajo carga, ni el comportamiento con paredes o microondas. Cualquier cifra de esta sección es una **cota de diseño**, no una promesa.

---

## Análisis de RAM

El ESP8266 tiene **80 KB de RAM de usuario** y una IRAM compartida con el código: la restricción no es el heap, es la IRAM.

| Firmware | RAM estimada | Librerías pesadas | Margen |
|---|---|---|---|
| `bridge_root.ino` | ~45 KB | uECC + BearSSL | ⚠️ IRAM ~91%, igual que v1.3 |
| `gateway.ino` | ~30 KB | ESP-NOW nativo | ✅ Holgado |
| `endpoint_bridge.ino` | ~28 KB | ESP-NOW × 2 contextos | ✅ Holgado |
| `endpoint.ino` | ~22 KB | ESP-NOW + WiFi | ✅ Holgado |

El Bridge_0 es el problema: ya en v1.3 estaba al 91%. Por eso la regla es **no agregarle código pesado**. Solo recibe del serial, cifra, envía, recibe de radio, descifra, escribe al serial y forwardea anuncios. Si hiciera falta más, la salida no es optimizar: es mover el Bridge a un ESP32.

| Nodo | Estado que guarda | Tamaño |
|---|---|---|
| Gateway_N | MACs de endpoints directos + MAC padre | ~300 B |
| Endpoint-bridge | 2 MACs + buffer 8 × 38 B × 2 | ~700 B |
| Endpoint simple | 1 MAC + buffer de recepción | ~300 B |
| PC | Mapa topológico completo | Ilimitado |

**Ningún ESP8266 mantiene una tabla de rutas global.** El consumo de la lógica de red es constante: da igual si hay 3 niveles o 30, 22 nodos o 2000. Cada nodo solo conoce a sus vecinos directos, y el costo se traslada al PC. La trama crece 81% (21 → 38 B), pero el estado de red no crece con la red: ese es el intercambio.

---

## Límites del sistema

| Parámetro | Límite | Razón | Cómo ampliarlo |
|---|---|---|---|
| Peers por interfaz ESP-NOW | 20 | Límite del SDK | No se puede |
| Endpoints por gateway | 19 | 20 peers − 1 bridge | Encadenar otra célula |
| Tramos de ruta | 8 | `ruta[16]`, 2 B por tramo | `ruta[32]` → 16 tramos |
| Profundidad práctica | 3–4 niveles | Latencia acumulada y buffer | Time-slice más corto |
| Payload máximo ESP-NOW | 250 B | Límite del protocolo | Fragmentar (no implementado) |

Con un puente por rama, cada nivel agrega hasta 19 endpoints. La red crece en abanico solo si un mismo puente sirve a varios gateways hijos, cosa que el diseño permite pero no implementa: para 22 nodos la forma es una cadena. Y sobre los "50 niveles" del diseño teórico: con `ruta[16]` y tramos de 2 B, el límite real son **8 tramos**.

---

## Estructura del repositorio

```
netEspNow-v1.4/
├── README.md · LICENSE
├── docs/       datasheet.md · developer.md · REFERENCES.md · BITACORA.md
│   └── assets/ architecture.svg · routing.svg · security.svg
├── nivel_0/    firmware del nivel raíz (pendiente)
├── nivel_1/    firmware del nivel intermedio (pendiente)
├── nivel_2/    firmware del nivel hoja (pendiente)
└── pc/         dashboard.html (prototipo visual)
```

Las carpetas `nivel_*` están vacías a propósito: cada sketch irá en una carpeta con el mismo nombre que el `.ino`, requisito de arduino-cli.

---

## Requisitos

| Elemento | Cantidad | Notas |
|---|---|---|
| ESP8266 (NodeMCU v3, D1 mini) | 18 | Con USB para flashear |
| ESP-01 con relé | 4 | Requieren adaptador USB |
| Fuente 3.3 V ≥ 1 A | 1 por nodo | ESP-NOW transmite en ráfagas |
| PC con USB | 1 | Bridge_0 y controlador |

**Sobre la alimentación:** es la causa número uno de fallos inexplicables. El módulo puede pedir 300 mA en microsegundos, y un cargador de 500 mA con cable fino produce caídas que reinician el chip sin dejar rastro en el serial. Fuentes de al menos 1 A, cables cortos y un capacitor de 470 µF junto al módulo.

| Software | Mínimo | Notas |
|---|---|---|
| `arduino-cli` | 0.35 | 1.x recomendado |
| Core `esp8266:esp8266` | 3.1.x | Incluye ESP-NOW y BearSSL |
| Python + `pyserial` | 3.8+ / 3.5+ | Controlador y dashboard |

`ESP8266WiFi`, `espnow` y `EEPROM` vienen con el core; `uECC` (curva elíptica del Bridge) y `DHT` (endpoints TEMP/HUM) son externas y se instalan con `arduino-cli lib install`.

---

## Compilación y carga

```bash
# 1. Instalar arduino-cli (Linux/macOS)
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | BINDIR=$HOME/.local/bin sh
arduino-cli version

# 2. Instalar el core de ESP8266
arduino-cli config init
arduino-cli config add board_manager.additional_urls \
  https://arduino.esp8266.com/stable/package_esp8266com_index.json
arduino-cli core update-index
arduino-cli core install esp8266:esp8266

# 3. Verificar placa y core
arduino-cli core list
arduino-cli board listall esp8266 | head -20
```

En Windows, bajá el `.zip` de releases y agregalo al `PATH`. El FQBN de este proyecto es `esp8266:esp8266:nodemcuv2`; para un ESP-01, `esp8266:esp8266:generic`.

Cada sketch vive en su carpeta, y **la carpeta debe llamarse igual que el `.ino`**:

```
nivel_0/
└── firmware_gateway/
    └── firmware_gateway.ino
```

```bash
# 4. Compilar
arduino-cli compile --fqbn esp8266:esp8266:nodemcuv2 nivel_0/firmware_gateway
arduino-cli compile --fqbn esp8266:esp8266:nodemcuv2 nivel_0/firmware_gateway --warnings all

# Para un ESP-01 (1 MB de flash)
arduino-cli compile --fqbn esp8266:esp8266:generic:eesz=1M,xtal=80 nivel_1/firmware_endpoint

# 5. Detectar el puerto real — NUNCA hardcodearlo
arduino-cli board list

# 6. Cargar
arduino-cli upload -p /dev/ttyUSB0 --fqbn esp8266:esp8266:nodemcuv2 nivel_0/firmware_gateway

# 7. Monitor serial
arduino-cli monitor -p /dev/ttyUSB0 --config baudrate=115200

# 8. Todo en un paso
arduino-cli compile --fqbn esp8266:esp8266:nodemcuv2 nivel_0/firmware_gateway \
  && arduino-cli upload -p /dev/ttyUSB0 --fqbn esp8266:esp8266:nodemcuv2 nivel_0/firmware_gateway
```

Revisá la línea de tamaño que imprime el compilador. Si `Global variables` supera el 80%, el nodo fallará en runtime de forma intermitente: es el síntoma clásico de RAM, no de lógica. El puerto puede ser `/dev/ttyUSB0` (CH340/CP2102) o `/dev/ttyACM0` (CDC nativo); en Windows `COM3`, `COM4`. **Si `board list` no muestra la placa, no hay nada que subir**: revisá el cable (muchos solo cargan) y el driver del chip serial.

| Error | Causa | Solución |
|---|---|---|
| `Sketch not found` | Carpeta ≠ nombre del `.ino` | Renombrar la carpeta |
| `region 'iram1' overflowed` | IRAM llena (Bridge) | Quitar funcionalidad, no optimizar |
| `espcomm_open failed` | Puerto ocupado o sin modo boot | Cerrar el monitor y reintentar |

---

## Despliegue por niveles

Cada fase deja el sistema funcionando. No hay big bang de 22 nodos.

**Fase 0 — Preparación, sin hardware.** Generar tres PMK de 16 bytes con un CSPRNG (`python3 -c "import secrets; print(secrets.token_hex(16))"`), guardarlas fuera del repositorio con permisos `600`, anotar las MAC de los 22 módulos y asignar IDs lógicos según la topología.

**Fase 1 — Nivel 0 (1 día).** Respaldar los `.bin` de v1.3. Flashear Gateway_0 (`GW_LEVEL=0`), E0.1 (`endpoint_bridge`, `MI_NIVEL=0`) y E0.2…E0.4 (`endpoint`, `MI_NIVEL=0`). Verificar que `ping 0` y `set 1 on` siguen funcionando desde cutecom, y que E0.1 reporta `ANNOUNCE_UP` al ver un beacon de nivel 1. **Aceptación:** los comandos v1.3 responden con el mismo formato; si no, no se avanza.

**Fase 2 — Nivel 1 (2 días).** Flashear Gateway_1, E1.1 como puente y E1.2…E1.9 como endpoints, todos con nivel 1. Encender Gateway_1 y comprobar el beacon cada 2 s, el `ANNOUNCE_UP` de E0.1 y el alta en el mapa del PC. Probar ida y vuelta a E1.5, el failover apagando E0.1 y el re-enraizamiento aislando Gateway_1. **Aceptación:** RTT a `E1.5` < 150 ms y recuperación tras caída de E0.1 en < 40 s.

**Fase 3 — Nivel 2 (1 día).** Flashear Gateway_2 y E2.1…E2.5, verificar el descubrimiento por E1.1 y el comando extremo a extremo a E2.5. **Aceptación:** RTT a `E2.3` < 250 ms y 100 comandos sin pérdidas.

**Fase 4 — Estrés (1 día).** Apagar E0.1 (failover), apagar Gateway_1 (rama aislada), apagar y encender todo (descubrimiento), 1000 comandos consecutivos y 24 h midiendo consumo.

**Orden importante:** **nunca flashees un gateway antes que sus endpoints.** Un gateway nuevo emite beacons con una PMK que los endpoints viejos no tienen; si están en otra clave, el gateway parece muerto y se pierden horas depurando un problema de orden. Endpoints primero, gateway después.

---

## Dashboard de PC

`pc/dashboard.html` es una **maqueta visual estática** del panel de operación. Se abre con doble clic, sin servidor, sin dependencias, sin build.

**No está conectado a ningún puerto serie.** Los envíos son simulados con `setTimeout`: ilumina la ruta, escribe en el log y actualiza el RTT con un valor fijo de 18 ms. No hay Web Serial, ni backend, ni `fetch`. Es una herramienta de diseño, no de operación: para operar hace falta el controlador Python (pendiente), que abre el puerto serie y habla el protocolo JSON.

Existe igual porque una topología de 22 nodos en tres niveles es difícil de explicar con palabras y muy fácil con círculos que se tocan.

---

## Pruebas

Los casos están definidos en el diseño y **ninguno se ejecutó todavía**: no hay firmware.

| # | Prueba | Procedimiento | Criterio |
|---|---|---|---|
| CP1 | Compatibilidad v1.3 | `ping 0` desde cutecom | Mismo formato carácter por carácter |
| CP2 | JSON crudo | `json {"ruta":"2","cmd":"READ_SENSOR"}` | El Bridge forwardea sin interpretar |
| CP3 | Nivel 1 | Comando a `E1.5` | Ruta `1:5`, respuesta < 150 ms |
| CP4 | Nivel 2 | LED a `E2.3` | Ruta `1:1:3`, respuesta < 250 ms |
| CP5 | Descubrimiento | Encender Gateway_1 | El PC lo ve en < 10 s sin intervención |
| CP6 | Failover | Apagar E0.1 con E0.2 viendo el gateway | Reanudación en < 20 s |
| CP7 | Re-enraizamiento | Aislar Gateway_1 | Reconexión en < 60 s |
| CP8 | Watchdog | Desconectar Bridge_0 | Los nodos reinician y se re-descubren |

Las pruebas de estrés quedan definidas por su criterio: 1000 comandos a E0.2 sin pérdidas ni duplicados, ruta de 6 saltos a E2.5 con RTT < 250 ms, error explícito ante un endpoint inexistente (`E2.9`) en < 100 ms, 22 nodos emitiendo 1 h con tráfico de gestión < 1% del canal, y reinicio del controlador a mitad de comando con reconstrucción del mapa.

En cada prueba hay que registrar `msg_id`, ruta, timestamps, RTT, RSSI, saltos y resultado: sin esos datos, una prueba es una anécdota.

---

## Riesgos y mitigaciones

| # | Riesgo | Prob. | Impacto | Mitigación |
|---|---|---|---|---|
| R1 | Bridge_0 es fallo único total | Baja | 🔴🔴🔴 | Dos Bridge en v1.5; hoy aceptado |
| R2 | E0.1 cae y nadie ve Gateway_1 | Media | 🔴🔴 | Múltiples endpoints con capacidad de puente |
| R3 | IRAM del Bridge al 91% | Alta | 🟠🟠 | No agregar código; migrar a ESP32 |
| R4 | Time-division suma hasta 100 ms | Media | 🟡 | Aceptable para domótica; slice configurable |
| R5 | Colisiones en canal 1 | Media | 🟡 | ACK y reintento de ESP-NOW |
| R6 | Desborde del buffer del puente | Media | 🟠 | 8 por contexto, descarte del más viejo |
| R7 | Corrupción de PMK en EEPROM | Baja | 🟠 | Magic `NS14`; si no coincide, regenerar |
| R8 | El PC pierde el mapa | Baja | 🟠 | JSON persistido + reconstrucción por anuncios |
| R9 | Flasheo en orden equivocado | Media | 🟡 | Endpoints primero, gateway después |
| R10 | Alimentación insuficiente o puente comprometido | Alta | 🟠 | Fuente ≥ 1 A; y en lo segundo, modelo de amenaza doméstico |

---

## Limitaciones y trabajo futuro

1. **El Bridge_0 es punto único de fallo.** Si muere, no hay red.
2. **La capa ECC es débil:** XOR en lugar de ECDH, XOR en lugar de SHA-256, claves compartidas.
3. **Las claves están en firmware y EEPROM.** Quien tenga el hardware, tiene la clave.
4. **El time-division cuesta latencia:** hasta 100 ms por puente en el peor caso.
5. **No hay autenticación de nodos ni cifrado de extremo a extremo.** La PMK autentica la célula; cada gateway descifra y re-cifra, así que uno comprometido ve todo lo que pasa.
6. **No hay fragmentación:** más de 250 B no se puede enviar, y el PC es imprescindible para las rutas.
7. **La implementación no existe todavía.** La limitación más importante y la más honesta.

| Versión | Objetivo |
|---|---|
| v1.4.1 | Implementar los 4 firmwares del nivel 0 y validar con hardware real |
| v1.4.2 | Implementar `fractal_controller.py` |
| v1.5 | ECDH real + SHA-256 en C1, y doble Bridge_0 con failover |
| v1.6 | Múltiples gateways hijos por puente y firmware unificado por rol |
| v2.0 | Migración a ESP32 |

El diseño ya contempla 12 roles de endpoint (`node_function`) que v1.3 declaró y nunca implementó: TEMP, HUM, GAS, PRESION, PUERTA, CONTADOR, RELE_MEC, RELE_SSR, PWM_DIMMER, ADC_RAW e IO, cada uno con una posición fija en `payload[8]` (TEMP usa `[0]` entero y `[1]` décimas; RELE usa `[0]` con 1=on, 2=off, 3=toggle). La idea es un firmware único con compilación condicional (`#define NODE_FUNC ...`) que genere todos los binarios desde un solo sketch.

---

## Cómo se construyó esto

1. **Inventario antes que diseño.** Se contaron los ESP disponibles, se midieron las MAC y se decidió que el despliegue sería de 22 nodos porque eso era lo que había.
2. **Auditoría de la versión anterior.** Antes de escribir una línea se revisó v1.3: el ECC usaba XOR en lugar de ECDH, el tramo Gateway↔Nodos iba en texto plano y no había PMK en ningún enlace. Ese hallazgo definió la prioridad: agregar cifrado intra-nivel sin tocar el ECC existente.
3. **Elección de la ruta explícita.** Se compararon tabla distribuida y estado de enlace tipo RPL, y se descartaron por RAM y convergencia. La ruta explícita gana porque traslada la complejidad al nodo con más recursos.
4. **El problema de la radio única.** Se evaluó poner dos módulos por puente (duplica costo y consumo) y usar una PMK global (rompe la segmentación). Se eligió alternar contextos por software, con buffer circular.
5. **Documentar antes de implementar.** Escribir el diseño completo antes del firmware permitió detectar inconsistencias en papel —el tamaño real de la trama, el largo máximo de ruta, la contradicción de la ruta invertida— que en código habrían costado días. Los descartes están en [`docs/BITACORA.md`](docs/BITACORA.md) para no volver a intentarlos.
6. **Lo que falta.** La implementación. Este repositorio es el plano, no el edificio. Si algo de lo anterior está mal, se descubrirá cuando el primer Gateway_1 emita su beacon y nadie lo escuche.

---

## Referencias

Las fuentes externas, con enlaces y notas de lectura, están en [`docs/REFERENCES.md`](docs/REFERENCES.md): **ESP-NOW** de Espressif (peers, payload, PMK/LMK), **Arduino core for ESP8266** (`espnow.h`, IRAM), **uECC**, **BearSSL**, **IEEE 802.11** y literatura de **redes de sensores**. Documentación propia: [`docs/datasheet.md`](docs/datasheet.md) para el operador, [`docs/developer.md`](docs/developer.md) para el desarrollador y [`docs/BITACORA.md`](docs/BITACORA.md) para las decisiones.

---

## Advertencias

- **Este repositorio no contiene firmware funcional.** No hay nada que compilar.
- **No uses esto para controlar nada crítico.** No hay autenticación de nodos ni cifrado de extremo a extremo.
- **La alimentación mata más ESP8266 que el código,** y el canal 1 lo compartís con tu Wi-Fi: si tu red está ahí, movela.

---

## Licencia

MIT. Ver [`LICENSE`](LICENSE).

```
Copyright (c) 2026 wanderc137
```

Se puede usar, copiar, modificar y distribuir conservando el aviso de copyright. Sin garantía de ningún tipo.
