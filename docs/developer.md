# Developer guide — netEspNow v1.4

**Cómo está pensado el firmware, cómo funciona el protocolo y cómo se extiende.**
Versión 1.4.0 · `Copyright (c) 2026 wanderc137` · MIT

---

## 1. Modelo mental

Antes de mirar una línea de código, hay que entender una idea: **este firmware no enruta, solo reenvía**.

Un gateway no sabe dónde está el destino final. No tiene tabla de vecinos ni mapa de la red. Todo lo que hace es:

1. Recibir una trama.
2. Mirar el primer número de la ruta.
3. Buscar la MAC de ese endpoint.
4. Recortar ese número.
5. Reenviar.

Si el número era el último, entrega localmente. Si no existe, devuelve error. Nada más. Toda la inteligencia de topología vive en el PC.

Eso tiene una consecuencia de diseño que atraviesa todo el código: **el estado de un nodo es mínimo y cabe en una estructura plana**. Un gateway guarda una lista de hasta 19 peers (MAC + ID). Un endpoint guarda una MAC de padre. Un puente guarda dos MAC y dos buffers. No hay árboles, ni tablas hash grandes, ni listas enlazadas con allocaciones dinámicas.

**Regla de oro:** si una función necesita saber "dónde está" un nodo que no es vecino directo, el diseño está mal. Esa información va en la ruta del mensaje.

---

## 2. Estructura de un sketch

### 2.1 Layout obligatorio

arduino-cli exige que la carpeta del sketch tenga el mismo nombre que el `.ino`:

```
nivel_0/
├── firmware_gateway/
│   └── firmware_gateway.ino
├── firmware_endpoint_bridge/
│   └── firmware_endpoint_bridge.ino
└── firmware_endpoint/
    └── firmware_endpoint.ino
```

Si la carpeta y el archivo no coinciden, el compilador responde `Sketch not found` y no hay nada que depurar.

### 2.2 Orden interno recomendado

```
1. Includes y constantes de configuración (#define)
2. Estructuras empaquetadas (deben ser idénticas entre firmwares)
3. Estado global (peers, buffers, contadores)
4. Prototipos de funciones
5. setup()
6. loop()
7. Callbacks de ESP-NOW
8. Funciones auxiliares (parseo, envío, diagnóstico)
```

Las estructuras van arriba y **copiadas textualmente** en los cuatro firmwares. Si una difiere en un byte, los mensajes se corrompen de una forma que no produce error de compilación y que cuesta horas encontrar.

---

## 3. Los cuatro firmwares

### 3.1 `firmware_bridge_root.ino`

**Rol:** traducir entre el PC y la red de radio.

```
setup()
  ├── Serial.begin(115200)
  ├── WiFi.mode(WIFI_STA); WiFi.disconnect()
  ├── esp_now_init()
  ├── cargar claves ECC desde EEPROM
  ├── registrar Gateway_0 como peer
  └── esp_now_register_recv_cb(on_recv)

loop()
  ├── si hay línea en Serial → parsear comando → cifrar → esp_now_send
  ├── si hay respuesta pendiente → descifrar → Serial.println
  └── watchdog: 10 timeouts → ESP.restart()
```

**Qué NO hace:** no interpreta `ruta`, no conoce los niveles, no mantiene topología. Si le llega un `ANNOUNCE_UP`, lo imprime tal cual en el serial sin analizarlo.

**Restricción crítica:** IRAM al ~91%. Este sketch no puede crecer. Cualquier función nueva acá debe reemplazar a otra, no sumarse. Si necesitás más lógica, va en el PC.

### 3.2 `firmware_gateway.ino`

**Rol:** distribuir dentro de su célula y conectar con su padre.

```
setup()
  ├── Serial.begin(115200)          // solo para debug
  ├── WiFi.mode(WIFI_STA); WiFi.disconnect()
  ├── esp_now_init()
  ├── esp_now_set_pmk(PMK_N{level})
  ├── cargar MACs de endpoints desde EEPROM
  ├── registrar cada endpoint como peer
  ├── registrar el puente de arriba como peer
  └── callbacks de envío y recepción

loop()
  ├── cada BEACON_INTERVAL → emitir beacon
  ├── procesar cola de recepción
  ├── watchdog: 120 s sin tráfico del padre → ESP.restart()
  └── si está aislado → modo búsqueda
```

**Máquina de estados:**

```
         ┌──────────────┐
         │  ARRANCANDO  │
         └──────┬───────┘
                │ peers registrados
                ▼
         ┌──────────────┐   sin tráfico 120 s   ┌──────────────┐
         │   OPERATIVO  │──────────────────────▶│ AISLADO      │
         └──────┬───────┘                       └──────┬───────┘
                │                                      │ encontró padre
                │◀─────────────────────────────────────┘
                │
                └── watchdog → ESP.restart() → ARRANCANDO
```

En `AISLADO` el gateway sigue respondiendo a sus endpoints y sigue emitiendo beacon. Solo deja de aceptar comandos del PC (porque no le llegan).

### 3.3 `firmware_endpoint_bridge.ino`

**Rol:** el nodo más complejo. Vive en dos células.

```
setup()
  ├── WiFi / esp_now_init
  ├── contextos[0] = { PMK_ARRIBA, peer: GW_ARRIBA }
  ├── contextos[1] = { PMK_ABAJO,  peer: GW_ABAJO  }
  ├── activar contexto 0
  └── callbacks

loop()
  ├── cada TIME_SLICE_MS → alternar contexto
  ├── al cambiar: esp_now_set_pmk(nueva), re-registrar peers, vaciar buffer
  ├── cada 5 s → ventana promiscua para detectar beacons de abajo
  └── watchdog dual: 120 s sin tráfico en cada contexto
```

**Detalle del cambio de contexto:**

```c
void cambiar_contexto(uint8_t idx) {
  ctx_activo = idx;

  // 1. Vaciar la PMK anterior y limpiar peers
  esp_now_deinit();
  esp_now_init();
  esp_now_set_pmk(contextos[idx].pmk);

  // 2. Re-registrar el peer de este contexto
  esp_now_add_peer(&contextos[idx].peer);
  esp_now_register_recv_cb(on_recv);

  // 3. Drenar el buffer de mensajes que llegaron en el turno anterior
  drenar_buffer(idx);
}
```

Reiniciar ESP-NOW en cada cambio es más caro que solo cambiar la PMK, pero es la única forma confiable de que no queden peers de la clave anterior en la tabla. Medí el costo antes de optimizarlo: si el cambio tarda más de 5 ms, el time-slice de 100 ms queda muy comprometido.

### 3.4 `firmware_endpoint.ino`

**Rol:** ejecutar. Es el más simple y el que más se va a compilar (16 variantes por rol).

```
setup()
  ├── WiFi / esp_now_init
  ├── esp_now_set_pmk(PMK_N{mi_nivel})
  ├── registrar gateway padre
  ├── inicializar GPIO/ADC según NODE_FUNC
  └── callbacks

loop()
  ├── cada 5 s → ventana promiscua (buscar beacons de nivel > mi_nivel)
  ├── ejecutar comandos pendientes
  └── watchdog: 120 s sin tráfico → ESP.restart()
```

---

## 4. Protocolo en detalle

### 4.1 La trama

```c
typedef struct __attribute__((packed)) {
  uint8_t  msg_type;
  uint8_t  ruta_len;
  uint8_t  ruta[16];
  uint8_t  msg_id[4];
  uint32_t timestamp_ms;
  uint8_t  cmd;
  uint8_t  pin;
  uint16_t val;
  uint8_t  payload[8];
} EspNowMessageV14;
```

**Por qué `packed`:** sin el atributo, el compilador alinea `uint32_t` a 4 bytes y `uint16_t` a 2, insertando padding. El `sizeof()` resultante depende del compilador y de las opciones, no del protocolo. Con `packed`, el tamaño es exactamente la suma de los campos: **38 bytes**. Verificalo siempre con una aserción en tiempo de compilación:

```c
static_assert(sizeof(EspNowMessageV14) == 38, "trama mal empaquetada");
```

**Endianness:** todos los enteros multi-byte van en **little-endian**, que es el orden nativo del ESP8266. Si algún día se agrega un nodo big-endian, habrá que convertir explícitamente.

### 4.2 `ruta`

- Buffer de 16 bytes, **no** terminado en NUL.
- `ruta_len` indica los bytes válidos.
- Contenido ASCII: dígitos y `:`.
- Ruta vacía (`ruta_len == 0`) significa "este nodo es el destino".
- Máximo práctico: 8 tramos (`1:2:3:4:5:6:7:8` = 15 bytes).

**Nunca uses `strlen()` sobre `ruta`.** No hay NUL, y el buffer puede no estar inicializado más allá de `ruta_len`. Siempre `ruta_len`.

### 4.3 `msg_id`

Cuatro bytes elegidos por el PC. No se interpretan en los nodos: solo se copian a la respuesta para que el PC correlacione. El PC debe mantener una ventana de IDs pendientes y descartar duplicados por si un ACK se pierde y el emisor reintenta.

### 4.4 `timestamp_ms`

Lo pone el PC al emitir. Los nodos lo copian sin tocarlo. Sirve para calcular el RTT de ida y vuelta. **No uses el reloj del nodo para esto:** los ESP8266 no tienen RTC y sus `millis()` no están sincronizados entre dispositivos.

### 4.5 `cmd`, `pin`, `val`, `payload`

`cmd` es la operación. `pin` es el GPIO o el ID de sensor **dentro del nodo**, no el ID del nodo. `val` es el valor escalar. `payload` es específico del rol.

La separación entre `pin` y `payload` es deliberada: `pin` identifica el recurso físico, `payload` lleva datos del dominio (temperatura, contador, estado).

---

## 5. El parser de ruta

Es la función central del gateway. Vale la pena entenderla línea por línea.

### 5.1 Contrato

```c
// Devuelve el ID del endpoint del primer tramo.
// Deja en *resto_len la longitud de la ruta sin ese tramo.
// Devuelve 0 si la ruta es inválida.
uint8_t primer_tramo(const uint8_t *ruta, uint8_t ruta_len, uint8_t *resto_len);
```

### 5.2 Algoritmo

```
entrada: ruta, ruta_len
si ruta_len == 0 → destino local, no hay tramo
buscar el primer ':' en ruta[0 .. ruta_len-1]
  si no hay ':'  → el tramo es toda la ruta; tras consumirlo, resto_len = 0
  si hay ':' en i → el tramo es ruta[0 .. i-1]; resto_len = ruta_len - i - 1
parsear los dígitos del tramo como entero (1..19)
validar que el resultado esté en rango
```

### 5.3 Implementación de referencia

```c
uint8_t primer_tramo(const uint8_t *ruta, uint8_t ruta_len, uint8_t *resto_len) {
  if (ruta_len == 0 || ruta_len > 16) return 0;

  uint8_t i = 0, id = 0;
  while (i < ruta_len && ruta[i] != ':') {
    if (ruta[i] < '0' || ruta[i] > '9') return 0;   // basura
    id = id * 10 + (ruta[i] - '0');
    if (id > 99) return 0;                          // desborde
    i++;
  }

  if (i == ruta_len) *resto_len = 0;                 // era el último
  else               *resto_len = ruta_len - i - 1;  // quedan tramos

  return id;                                         // 0 si no hay
}

void recortar(uint8_t *ruta, uint8_t ruta_len, uint8_t sep, uint8_t *nuevo_len) {
  uint8_t resto = ruta_len - sep - 1;
  memmove(ruta, ruta + sep + 1, resto);
  *nuevo_len = resto;
}
```

### 5.4 Casos borde que hay que cubrir

| Entrada | `ruta_len` | Resultado esperado |
|---|---|---|
| `""` | 0 | destino local |
| `"3"` | 1 | id 3, resto 0 → entregar y vaciar |
| `"1:3"` | 3 | id 1, resto 1, ruta queda `"3"` |
| `"1:1:3"` | 5 | id 1, resto 3, ruta queda `"1:3"` |
| `"01:3"` | 4 | id 1 (ceros a la izquierda válidos) |
| `"1:"` | 2 | id 1, resto 0 → ruta inválida semánticamente |
| `":3"` | 2 | id 0 → error |
| `"1::3"` | 4 | id 1, resto 1 (`":3"`) → el siguiente gateway falla |
| `"1:3:9"` con `ruta_len` 5 | 5 | id 1, resto 3 |
| `"a:3"` | 3 | id 0 → error de formato |
| 30 caracteres | 30 | rechazo por `ruta_len > 16` |

**Recomendación:** rechazá rutas malformadas en el Bridge_0, no en los gateways. Un error temprano con mensaje claro ahorra depuración en el campo.

### 5.5 Validación de ID

El parser devuelve un número; el gateway decide si ese número corresponde a un endpoint conocido:

```c
peer_t *buscar_endpoint(uint8_t id) {
  for (uint8_t i = 0; i < n_endpoints; i++)
    if (endpoints[i].id == id) return &endpoints[i];
  return NULL;
}
```

Si devuelve `NULL`, se emite `endpoint_not_found` **inmediatamente**. No se reintenta: reintentar un ID que no existe solo consume aire y retrasa el error.

---

## 6. Máquinas de estado

### 6.1 Recepción en el gateway

```
on_recv(msg, mac_origen)
  │
  ├─ msg_type == BEACON?      → ignorar (un gateway no escucha beacons de su nivel)
  ├─ msg_type == ANNOUNCE_UP? → reenviar al padre, incrementar hops
  ├─ msg_type == RESP_UP?     → reenviar al padre
  ├─ msg_type == CMD_DOWN?
  │    ├─ ruta_len == 0 → entregar local por msg.pin
  │    ├─ parsear primer tramo
  │    │    ├─ inválido → enviar error
  │    │    ├─ no existe → enviar endpoint_not_found
  │    │    └─ existe   → recortar y enviar
  │    └─ actualizar timestamp de último tráfico
  └─ otro → descartar y contar
```

### 6.2 Recepción en el endpoint

```
on_recv(msg, mac_origen)
  ├─ BEACON?      → anotar nivel y RSSI (solo en ventana promiscua)
  ├─ CMD_DOWN con ruta vacía? → ejecutar y responder
  ├─ CMD_DOWN con ruta no vacía? → error (no soy puente)
  └─ otro → descartar
```

### 6.3 Recepción en el puente

```
on_recv(msg, mac_origen)
  ├─ si mac_origen == GW_ARRIBA:
  │    └─ guardar en buffer_abajo (se enviará en el contexto de abajo)
  ├─ si mac_origen == GW_ABAJO:
  │    └─ guardar en buffer_arriba (se enviará en el contexto de arriba)
  └─ si mac_origen desconocida:
       └─ descartar y contar intrusos
```

**El puente no modifica la ruta.** Ni la parsea. Copia la trama y la entrega al otro contexto tal cual.

---

## 7. Time-division en detalle

### 7.1 Por qué no alcanza con cambiar la PMK

`esp_now_set_pmk()` cambia la clave de la interfaz, pero los peers registrados siguen ahí con su LMK derivada de la clave anterior. Al cambiar de contexto hay que reconstruir la tabla de peers. Por eso el cambio completo implica `deinit` + `init` + `set_pmk` + `add_peer`.

### 7.2 Los buffers

```c
#define BUF_SIZE 8
typedef struct {
  EspNowMessageV14 msg[BUF_SIZE];
  uint8_t mac[BUF_SIZE][6];
  uint8_t head, tail, count;
} ring_t;
```

- Un ring por contexto.
- `count == BUF_SIZE` → descartar el más antiguo (`tail++`).
- El drenaje ocurre inmediatamente después del cambio de contexto.
- **Nunca** se hace `malloc` en el callback: el heap del ESP8266 se fragmenta y el callback corre en un contexto restringido.

### 7.3 Sincronización con el beacon

El puente también debe escuchar beacons del gateway de abajo para saber que sigue vivo. Eso ocurre en la **ventana promiscua**, que se abre desde el contexto de arriba: durante 200 ms cada 5 s se ignora la PMK y se acepta cualquier trama. Si en esa ventana llega un beacon con `gw_level` mayor al propio, se reporta con `ANNOUNCE_UP`.

Cuidado: en esa ventana también pueden llegar comandos legítimos de arriba. Hay que distinguirlos por `msg_type` y por MAC, no por la PMK (que está desactivada).

---

## 8. EEPROM

### 8.1 Layout

```
Offset 0:      magic "NS14"      (4 B)
Offset 4:      PMK del nivel     (16 B)
Offset 20:     n_endpoints       (1 B)
Offset 21:     MACs               (6 B × n)
Offset 21+6n:  IDs lógicos        (1 B × n)
```

### 8.2 Reglas

- Escribir **una sola vez** al aprovisionar. La EEPROM del ESP8266 tiene ~100 000 ciclos; escribirla en cada arranque la mata en meses.
- Al arrancar, leer y validar el magic. Si no coincide, usar los valores compilados y reportar `eeprom_invalida` por serial.
- `EEPROM.commit()` es obligatorio después de escribir; sin eso los cambios no persisten.
- Reservar `EEPROM.begin(512)` y no tocar los últimos bytes: el SDK usa parte del sector.

### 8.3 Rotación

No hay rotación en caliente. Cambiar la PMK implica reflashear la célula y reescribir la EEPROM del gateway. Documentá el procedimiento en la bitácora de la instalación, no solo en el repo.

---

## 9. API de funciones

Las firmas que el resto del firmware va a esperar. Están agrupadas por responsabilidad.

### 9.1 Radio

| Función | Firma | Devuelve |
|---|---|---|
| Inicializar | `bool radio_init(void)` | `true` si ESP-NOW arrancó |
| Fijar PMK | `bool radio_set_pmk(const uint8_t *pmk)` | `true` si se aplicó |
| Agregar peer | `bool radio_add_peer(const uint8_t *mac, uint8_t id)` | `true` si entró en la tabla |
| Enviar | `bool radio_send(const uint8_t *mac, const EspNowMessageV14 *m)` | `true` si se encoló |
| MAC propia | `void radio_own_mac(uint8_t *out)` | — |

### 9.2 Ruta

| Función | Firma | Devuelve |
|---|---|---|
| Primer tramo | `uint8_t primer_tramo(const uint8_t *ruta, uint8_t len, uint8_t *resto)` | ID o 0 |
| Recortar | `void recortar(uint8_t *ruta, uint8_t len, uint8_t sep, uint8_t *nuevo)` | — |
| Construir (PC) | `char *construir_ruta(const char *destino)` | string o NULL |
| Validar | `bool ruta_valida(const uint8_t *ruta, uint8_t len)` | `true`/`false` |

### 9.3 Peers

| Función | Firma | Devuelve |
|---|---|---|
| Buscar | `peer_t *buscar_endpoint(uint8_t id)` | puntero o NULL |
| Contar | `uint8_t contar_peers(void)` | cantidad |
| Cargar de EEPROM | `bool cargar_peers(void)` | `true` si el magic era válido |

### 9.4 Protocolo

| Función | Firma | Devuelve |
|---|---|---|
| Entregar local | `void entregar_local(const EspNowMessageV14 *m)` | — |
| Enviar error | `void enviar_error(const EspNowMessageV14 *orig, uint8_t code, uint8_t arg)` | — |
| Responder | `void responder(const EspNowMessageV14 *orig, uint16_t val)` | — |
| Beacon | `void emitir_beacon(void)` | — |
| Anuncio | `void enviar_anuncio(uint8_t nivel, uint8_t gw_id, int8_t rssi, uint8_t hops)` | — |

### 9.5 Diagnóstico

| Función | Firma | Devuelve |
|---|---|---|
| Volcar estado | `void dump_estado(void)` | — |
| Contar descartes | `uint16_t descartes(void)` | cantidad |
| Último error | `uint8_t ultimo_error(void)` | código |

---

## 10. Cómo extender

### 10.1 Agregar un comando

1. Elegí un código libre en la tabla (`cmd`).
2. Documentalo en la tabla del README y del datasheet.
3. Implementá el `case` en `entregar_local()` del endpoint.
4. Si devuelve datos, definí el layout de `payload` y respetalo.
5. Agregá un caso de prueba con el criterio de aceptación.

**Regla:** un comando nuevo no puede cambiar el tamaño de la trama. Si necesitás más de 8 bytes de payload, el comando está mal diseñado: partilo en dos o usá `CONFIG` con múltiples pasos.

### 10.2 Agregar un rol de endpoint

```c
#define NODE_FUNC_TEMP      1
#define NODE_FUNC_GAS       3
// ...

#ifndef NODE_FUNC
#define NODE_FUNC NODE_FUNC_IO
#endif

#if NODE_FUNC == NODE_FUNC_TEMP
  #include <DHT.h>
  DHT dht(PIN_DHT, DHT22);
#endif
```

En `setup()` y en `entregar_local()` usás `#if` para compilar solo lo necesario. La ventaja: un solo `.ino`, 12 binarios, y el compilador se encarga de que no haya código muerto.

### 10.3 Agregar un nivel

1. Definí `PMK_N3` y asignala a los nodos nuevos.
2. Agregá el nivel al mapa del PC (`topologia["nivel_3"]`).
3. El endpoint-bridge del nivel 2 pasa a tener `PMK_ABAJO = PMK_N3`.
4. El gateway del nivel 3 se flashea con `GW_LEVEL=3`.

**No hay que tocar** el gateway del nivel 0, ni el del 1, ni el Bridge. Esa es la propiedad fractal: la profundidad no cambia el firmware de los niveles superiores.

**Límite:** 8 tramos en `ruta[16]`. Para más, cambiá el buffer a 32 bytes en los cuatro firmwares simultáneamente y actualizá el `static_assert`.

### 10.4 Agregar un gateway hijo adicional

Hoy cada nivel tiene un gateway. Para poner dos gateways colgando del mismo puente:

1. El puente debe conocer la MAC de ambos hijos.
2. El gateway padre debe poder elegir a cuál de los dos puentes enviar (hoy elige por ID de endpoint).
3. La ruta necesita distinguir entre ramas: `1:1:3` deja de ser suficiente si hay dos "1".

Este cambio rompe el invariante actual. La solución limpia es una ruta con nivel explícito (`N1.1:N2.3`) o un ID de endpoint único por nivel. Está identificado como trabajo de v1.6, no lo improvises.

---

## 11. Depuración

### 11.1 Qué imprimir y qué no

| Nivel de log | Cuándo | Costo |
|---|---|---|
| Solo arranque | Producción | Ninguno |
| Errores | Producción | Bajo |
| Cada trama | Depuración | Alto: el serial a 115200 tarda ~1 ms por línea y bloquea |
| Volcado hexadecimal | Casos difíciles | Muy alto |

**Nunca dejes logs por trama en producción.** El `Serial.print` es bloqueante y, en un gateway que debe reenviar en milisegundos, introduce latencia y pérdidas.

### 11.2 Señales útiles

- **Uptime que se reinicia solo:** alimentación o watchdog.
- **Contador de descartes subiendo:** buffer lleno o tramas de otra PMK.
- **RSSI que oscila más de 10 dBm:** multipath o antena deficiente.
- **`endpoint_not_found` intermitente:** el gateway perdió la tabla de peers (¿reinició?).
- **Respuestas duplicadas:** `msg_id` no se está correlacionando bien en el PC.

### 11.3 Herramientas

```bash
arduino-cli monitor -p /dev/ttyUSB0 --config baudrate=115200
```

Para capturar tráfico de radio hace falta hardware adicional (un ESP8266 en modo promiscuo con un sketch sniffer). Sin la PMK solo se ven las tramas cifradas: longitud y MAC de origen, nada de contenido.

### 11.4 Prueba mínima antes de desplegar

```c
// Sketch de humo: enciende el LED y responde a PING
// Sin rutas, sin niveles, sin PMK. Solo verifica radio y alimentación.
```

Si ese sketch no funciona, el problema no es el firmware de netEspNow: es el módulo, el cable o la fuente.

---

## 12. Decisiones de diseño y su motivo

| Decisión | Alternativa descartada | Motivo |
|---|---|---|
| Ruta en el mensaje | Tabla de rutas en cada gateway | RAM constante y sin convergencia |
| Memoria de padre para la vuelta | Invertir la ruta | El puente ascendente no pertenece al nivel actual |
| PMK por nivel | ECC en todos los enlaces | 200 ms vs 1 ms por salto |
| Time-division | Dos radios por puente | El ESP8266 tiene una sola |
| Buffer circular de 8 | Cola dinámica | Sin `malloc` en el callback |
| Beacons dentro de la PMK | Broadcast global | Evita tormenta y filtra por célula |
| `packed` en las tramas | Serialización manual | Menos código, mismo resultado si se valida el tamaño |
| Validar rutas en el Bridge | Validar en cada gateway | Error temprano y un solo lugar que mantener |

---

## 13. Trampas conocidas

1. **`sizeof()` sin `packed`.** El struct crece y los nodos dejan de entenderse. Usá `static_assert`.
2. **`strlen()` sobre `ruta`.** No hay NUL. Usá `ruta_len`.
3. **`memmove` con solapamiento mal calculado.** Copiá siempre hacia adelante con el tamaño exacto.
4. **Registrar peers sin limpiar la tabla al cambiar de PMK.** Quedan peers con la LMK vieja y las tramas se descartan en silencio.
5. **Llamar `esp_now_send` desde el callback de recepción.** Puede fallar por reentrada; encolá y enviá desde `loop()`.
6. **Asumir que el ACK de ESP-NOW significa que el nodo ejecutó.** Solo significa que la radio recibió. La confirmación real es la respuesta.
7. **Imprimir por serial dentro del callback.** Bloquea y puede disparar el watchdog.
8. **Usar `delay()` en el gateway.** Frena el reenvío de todos los mensajes.
9. **Cambiar el time-slice sin medir.** Un slice muy corto satura el buffer; uno muy largo dispara la latencia.
10. **Agregar código al Bridge_0.** La IRAM está al 91%: el linker va a fallar o, peor, va a fallar en runtime.

---

## 14. Referencias internas

- [`../README.md`](../README.md) — visión general, topología y despliegue.
- [`datasheet.md`](datasheet.md) — manual de operador.
- [`REFERENCES.md`](REFERENCES.md) — fuentes externas.
- [`BITACORA.md`](BITACORA.md) — decisiones y descartes.

---

*netEspNow v1.4 · guía de desarrollador · `Copyright (c) 2026 wanderc137` · MIT*
