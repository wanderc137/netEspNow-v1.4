# Datasheet — netEspNow v1.4

**Manual de operador de la red fractal ESP-NOW de 3 niveles y 22 nodos.**
Versión del documento: 1.4.0 · Licencia MIT · `Copyright (c) 2026 wanderc137`

---

## 0. Antes de empezar: qué es y qué no es

Este documento describe cómo se instala, se opera y se diagnostica netEspNow v1.4. Está escrito para quien va a estar frente al equipo: la persona que flashea los módulos, mira el monitor serie y decide qué hacer cuando algo no responde.

**Lo que este proyecto es:** una arquitectura de red cerrada y documentada, con protocolo, formato de mensajes, esquema de claves y plan de despliegue. El diseño está completo.

**Lo que este proyecto no es todavía:** firmware compilable. Los cuatro sketches están especificados pero no implementados en este repositorio. Las instrucciones de operación describen **el contrato del sistema**: lo que el firmware deberá hacer y cómo se lo operará cuando exista. Si intentás flashear ahora, no vas a encontrar los `.ino`.

> Si tu objetivo es entender el diseño, este documento te va a servir como manual de uso. Si tu objetivo es operar una red ya desplegada, estás en el lugar correcto siempre que el firmware ya esté implementado en tus módulos.

---

## 1. Resumen funcional

| Atributo | Valor |
|---|---|
| Transporte | ESP-NOW sobre Wi-Fi canal 1, sin AP, sin IP |
| Topología | Árbol de 3 células encadenadas (fractal) |
| Nodos | 22 ESP8266 |
| Niveles | 0 (raíz), 1 (intermedio), 2 (hoja) |
| Enrutamiento | Ruta explícita ASCII en cada mensaje (`1:1:3`) |
| Estado de rutas en los nodos | Ninguno (RAM constante) |
| Cifrado | ECC en Bridge↔Gateway_0; PMK por nivel dentro de cada célula |
| Descubrimiento | Beacons cada 2 s + anuncios hacia arriba |
| Recuperación | Re-enrutamiento automático de 5 a 30 s |
| Interfaz de operación | Serial 115200 8N1 + dashboard HTML |
| Alimentación | 3.3 V, ≥ 1 A por nodo |

---

## 2. Nomenclatura

Entender los nombres es la mitad de operar el sistema.

| Nombre | Significado |
|---|---|
| `Bridge_0` | Único nodo conectado al PC por USB. Traduce serial ↔ radio. |
| `Gateway_0/1/2` | Puerta de enlace de cada célula. Distribuye y reenvía. |
| `E0.1` … `E0.4` | Endpoints del nivel 0. `E0.1` es además puente hacia nivel 1. |
| `E1.1` … `E1.9` | Endpoints del nivel 1. `E1.1` es puente hacia nivel 2. |
| `E2.1` … `E2.5` | Endpoints hoja del nivel 2. No tienen nada debajo. |
| `PMK_Nx` | Clave maestra de 16 bytes del nivel `x`. |
| `ruta` | Cadena ASCII con los tramos a recorrer, separados por `:`. |
| `ANNOUNCE_UP` | Aviso de un endpoint a su gateway: "veo un gateway debajo". |
| `INACCESIBLE` | Estado que asigna el PC a una rama sin puente. |

**Regla mnemotécnica:** el primer número de `E2.3` es el nivel; el segundo, el ID dentro de ese nivel. La ruta hacia `E2.3` se lee de arriba hacia abajo: `1:1:3` = "endpoint 1 del nivel 0, endpoint 1 del nivel 1, endpoint 3 del nivel 2".

---

## 3. Requisitos

### 3.1 Hardware mínimo

| Elemento | Cantidad | Observación |
|---|---|---|
| ESP8266 con USB (NodeMCU v3, D1 mini) | 18 | Niveles 0 y 1, más gateways |
| ESP-01 con relé | 4 | Nivel 2, actuadores |
| Fuente 3.3 V ≥ 1 A | 1 por nodo | Imprescindible |
| Capacitor 470 µF | 1 por nodo | Entre VCC y GND, junto al módulo |
| PC con USB | 1 | Bridge_0 y controlador |

### 3.2 Software

| Elemento | Versión | Para qué |
|---|---|---|
| `arduino-cli` | ≥ 0.35 (1.x recomendado) | Compilar y cargar |
| Core `esp8266:esp8266` | 3.1.x | ESP-NOW y BearSSL |
| Python | 3.8+ | Controlador y dashboard |
| `pyserial` | 3.5+ | Puerto serie del controlador |
| Navegador | moderno | Dashboard |

### 3.3 Compatibilidad de placas

| Placa | FQBN | Estado | Nota |
|---|---|---|---|
| NodeMCU v3 (ESP-12E) | `esp8266:esp8266:nodemcuv2` | ✅ Recomendada | 4 MB flash, USB integrado |
| Wemos D1 mini | `esp8266:esp8266:d1_mini` | ✅ | Mismo chip, menos pines |
| ESP-01 (512 KB / 1 MB) | `esp8266:esp8266:generic` | ⚠️ | Requiere adaptador USB; pocos GPIO |
| ESP-01S con relé | `esp8266:esp8266:generic` | ⚠️ | Solo actuador; sin ADC |
| ESP8266 genérico | `esp8266:esp8266:generic` | ⚠️ | Verificar tamaño de flash |
| ESP32 | — | ❌ | No soportado en v1.4 (API distinta) |

**Nunca uses un ESP-01 como gateway.** Le faltan pines y su antena es marginal para sostener una célula de 9 endpoints.

---

## 4. Instalación

### 4.1 Preparación de claves

Se necesitan tres PMK de 16 bytes, una por nivel. Deben ser aleatorias: no uses frases, ni la MAC, ni nada derivable.

```bash
python3 -c "import secrets; print('PMK_N0=' + secrets.token_hex(16))"
python3 -c "import secrets; print('PMK_N1=' + secrets.token_hex(16))"
python3 -c "import secrets; print('PMK_N2=' + secrets.token_hex(16))"
```

Guardalas **fuera del repositorio**, en un archivo con permisos restringidos:

```bash
mkdir -p ~/.config/netesponw
chmod 700 ~/.config/netesponw
# guardar las tres claves en ~/.config/netesponw/keys.env
chmod 600 ~/.config/netesponw/keys.env
```

Si perdés una PMK, no hay recuperación: hay que reflashear la célula completa con la clave nueva.

### 4.2 Inventario de MAC

Cada módulo tiene una MAC única de fábrica. Se lee de tres formas:

1. Imprimiéndola por serial al arrancar (`WiFi.macAddress()`).
2. Desde la tabla de clientes de tu router (si el módulo se conectó alguna vez).
3. Con un sketch mínimo de diagnóstico que la escriba y no haga nada más.

Anotá las 22 MAC en una planilla junto al ID lógico. Ese mapa es el insumo del `topologia.json` del PC.

> **Importante:** las MAC de ejemplo en la documentación (`AA:BB:CC:DD:EE:0N`) son placeholders. Las reales no van al repositorio.

### 4.3 Orden de flasheo

El orden no es negociable:

```
1. Endpoints del nivel 0  (E0.2 … E0.4, E0.1)
2. Gateway_0
3. Endpoints del nivel 1  (E1.2 … E1.9, E1.1)
4. Gateway_1
5. Endpoints del nivel 2  (E2.1 … E2.5)
6. Gateway_2
7. Bridge_0  (por último, cuando todo lo demás ya está arriba)
```

**Por qué:** un gateway emite beacons con su PMK. Si los endpoints todavía tienen la clave vieja, el gateway parece muerto y se pierden horas buscando un problema de radio que es de orden. Endpoints primero, gateway después.

### 4.4 Parámetros por rol

| Firmware | Constantes a ajustar |
|---|---|
| `firmware_gateway` | `GW_LEVEL` (0/1/2), `GW_ID`, `BEACON_INTERVAL`, `WATCHDOG_MS` |
| `firmware_endpoint_bridge` | `MI_NIVEL`, `PMK_ARRIBA`, `PMK_ABAJO`, `GW_ARRIBA_MAC`, `TIME_SLICE_MS` |
| `firmware_endpoint` | `MI_NIVEL`, `NODE_ID`, `NODE_FUNC` |
| `firmware_bridge_root` | Puerto serial, MAC de Gateway_0 |

### 4.5 Compilación y carga

```bash
arduino-cli compile --fqbn esp8266:esp8266:nodemcuv2 nivel_0/firmware_gateway
arduino-cli board list
arduino-cli upload -p /dev/ttyUSB0 --fqbn esp8266:esp8266:nodemcuv2 nivel_0/firmware_gateway
arduino-cli monitor -p /dev/ttyUSB0 --config baudrate=115200
```

El puerto cambia según el chip serial: `/dev/ttyUSB0` para CH340/CP2102, `/dev/ttyACM0` para CDC nativo, `COM3`/`COM4` en Windows.

---

## 5. Operación

### 5.1 Conexión al Bridge_0

1. Conectá el Bridge_0 por USB.
2. Verificá que el puerto aparezca (`arduino-cli board list`).
3. Abrí el monitor a **115200 8N1**.
4. Al reiniciar el Bridge deberías ver un mensaje de arranque con su MAC y su versión.

Si no ves nada: puerto equivocado, baudrate equivocado, o el firmware no está cargado.

### 5.2 Comandos de compatibilidad (nivel 0)

| Comando | Ejemplo | Respuesta |
|---|---|---|
| `ping <id>` | `ping 2` | `N2 c=… u=… i=… a=… o=… rssi=…` |
| `set <id> <on\|off>` | `set 1 on` | `OK` o error |
| `stats` | `stats` | Contadores del bridge y de la red |
| `list` | `list` | Tabla de nodos conocidos |
| `auto <on\|off>` | `auto on` | Activa el polling periódico |

Estos comandos son idénticos a los de la versión anterior. Si actualizaste el firmware y dejaron de responder, algo se rompió en la compatibilidad: revisá el formato de respuesta.

**Lectura de la respuesta de `ping`:**

```
N2 c=6 u=34567ms i=1 a=512 o=0 rssi=-45
│  │   │        │  │     │    └── RSSI en dBm (más cerca de 0 = mejor)
│  │   │        │  │     └────── salida digital (0/1)
│  │   │        │  └──────────── entrada analógica (0-1023)
│  │   │        └─────────────── entrada digital (0/1)
│  │   └──────────────────────── uptime del nodo
│  └──────────────────────────── contador de mensajes
└─────────────────────────────── ID lógico del nodo
```

### 5.3 Comandos con ruta explícita

Para cualquier nivel, el comando es un JSON en una línea:

```
json {"ruta":"1:1:3","cmd":"READ_SENSOR","msg_id":1234}
```

| Campo | Obligatorio | Descripción |
|---|---|---|
| `ruta` | sí | Tramos separados por `:`. `""` no es válido desde el PC. |
| `cmd` | sí | `PING`, `LED`, `READ_SENSOR`, `SET_GPIO`, `READ_ADC`, `RELAY`, `CONFIG`, `STATS` |
| `msg_id` | recomendado | Si se omite, el Bridge asigna uno. |
| `pin` | según comando | GPIO o ID de sensor. |
| `val` | según comando | 0/1, duty, umbral. |
| `payload` | opcional | Bytes específicos del rol. |

**Ejemplos por nivel:**

```
json {"ruta":"2","cmd":"LED","val":1}                  → E0.2
json {"ruta":"1:5","cmd":"READ_SENSOR"}                → E1.5
json {"ruta":"1:1:3","cmd":"READ_ADC"}                 → E2.3
json {"ruta":"1:1:1","cmd":"RELAY","payload":[1]}      → E2.1 (relé on)
```

### 5.4 Respuestas y eventos

```
{"msg_id":1234,"ruta":"1:1:3","val":512,"rtt_ms":18}          respuesta normal
{"error":"endpoint_not_found","ruta":"1:9","id":9}            endpoint inexistente
{"error":"rama_aislada","nivel":1}                             no hay puente
{"event":"gateway_detectado","nivel":2,"puente":"E1.1"}        descubrimiento
{"event":"nodo_offline","id":"E1.4","timeouts":3}              caída detectada
```

**Cómo leer un error:**

| Error | Significa | Qué hacer |
|---|---|---|
| `endpoint_not_found` | El gateway de ese nivel no conoce el ID | Revisar la ruta y la tabla de endpoints |
| `rama_aislada` | No hay ningún puente hacia ese nivel | Revisar el endpoint-bridge y su beacon |
| `timeout` | Se envió pero no hubo respuesta | Revisar alimentación y RSSI del nodo |
| `ruta_invalida` | Formato incorrecto (falta `:`, número no numérico) | Corregir el JSON |
| `pmk_mismatch` | El nodo no descifra: clave distinta | Reflashear la célula con la PMK correcta |

### 5.5 Dashboard

Abrí `pc/dashboard.html` con doble clic. No necesita servidor.

> **Aviso:** el dashboard es una **maqueta visual**. Los comandos que envía son simulados en el navegador: ilumina la ruta, escribe en el registro y muestra un RTT fijo de 18 ms. No hay conexión serie. Para operar de verdad, usá el controlador Python o el monitor serial.

Sirve para explicar la topología, no para diagnosticar.

---

## 6. Calibración

### 6.1 Time-slice del endpoint-bridge

`TIME_SLICE_MS` define cuánto tiempo el puente escucha en cada contexto. Valor por defecto: 100 ms.

| Valor | Efecto | Cuándo usarlo |
|---|---|---|
| 200 ms | Menos overhead, hasta 200 ms de latencia | Red con poco tráfico y comandos tolerantes |
| 100 ms | Equilibrio | **Recomendado** |
| 50 ms | Menos latencia, más cambios de contexto | Comandos interactivos, pocos nodos |
| 20 ms | Casi conmutación por paquete | Solo si mediste que el buffer no desborda |

El límite práctico: si el buffer de 8 se llena seguido, el slice es demasiado corto o el tráfico es demasiado alto para ese puente.

### 6.2 Intervalo de beacon

Por defecto 2000 ms. Bajarlo a 1000 ms acelera el descubrimiento y el failover, a costa de más tráfico de gestión. Subirlo a 5000 ms reduce el tráfico pero hace que un gateway tardío tarde más en aparecer.

**Regla:** el intervalo de beacon debe ser al menos 5 veces menor que el timeout de detección, o el sistema declarará caído un nodo que solo estaba entre beacons.

### 6.3 Umbral de RSSI

No hay un umbral fijo en el firmware: el RSSI se reporta y el operador decide. Como guía:

| RSSI | Calidad | Acción |
|---|---|---|
| −30 a −50 dBm | Excelente | Nada |
| −50 a −65 dBm | Buena | Nada |
| −65 a −75 dBm | Marginal | Reubicar o agregar un puente |
| < −75 dBm | Mala | Enlaces inestables, pérdidas |

### 6.4 Umbrales de sensor

Los roles con umbral (GAS, por ejemplo) guardan el valor en `payload[1]`. Se ajusta con un comando `CONFIG` o recompilando. Calibrá con el sensor en aire limpio: el valor de reposo de un MQ-2 varía por unidad y por envejecimiento del sensor.

---

## 7. Mantenimiento

### 7.1 Rutina semanal

- Revisar `stats` en el Bridge: contadores de error y timeouts.
- Verificar que los 22 nodos aparezcan en `list`.
- Mirar la tabla de RSSI: cualquier nodo por debajo de −75 dBm es candidato a fallo.

### 7.2 Rutina mensual

- Reiniciar el sistema completo de forma controlada (apagar y encender) para verificar el auto-descubrimiento.
- Revisar que el `topologia.json` del PC coincida con la realidad: si hay divergencia, el descubrimiento está fallando.
- Comprobar la alimentación de los nodos con más carga (los que tienen relé).

### 7.3 Rotación de claves

No hay rotación en caliente. Para cambiar una PMK:

1. Generar la nueva clave.
2. Reflashear **todos** los nodos de esa célula (endpoints primero).
3. Reflashear el gateway de esa célula.
4. Verificar que la célula responde.
5. Actualizar `keys.env`.

Mientras la célula está a medio migrar, los nodos con clave nueva no se ven entre sí. Planificá la ventana de mantenimiento.

### 7.4 Respaldo

| Qué respaldar | Dónde | Frecuencia |
|---|---|---|
| `keys.env` | Fuera del equipo, cifrado | Al crear y al rotar |
| `topologia.json` | PC | Automático al cambiar |
| Binarios compilados | `~/firmware-backup/` | Tras cada compilación estable |
| Este repositorio | Git | Cada cambio |

---

## 8. Solución de problemas

| Síntoma | Causa probable | Diagnóstico | Acción |
|---|---|---|---|
| El nodo no aparece en `list` | PMK distinta, fuera de alcance, sin alimentación | Monitor serial del nodo; ¿emite algo al arrancar? | Reflashear con la PMK correcta; acercar; medir 3.3 V |
| `ping` responde pero `json` no | Firmware viejo en el Bridge | `stats` muestra versión | Actualizar el Bridge |
| Comandos al nivel 1 dan `rama_aislada` | E0.1 caído o sin beacon | `list` no muestra E0.1 | Revisar E0.1; ver si otro endpoint ve Gateway_1 |
| RTT por encima de 500 ms | Time-slice largo, reintentos, congestión | Medir por nivel | Bajar `TIME_SLICE_MS`; revisar interferencia |
| Nodos que reinician solos | Alimentación insuficiente | Medir VCC durante transmisión | Fuente ≥ 1 A, capacitor 470 µF |
| El gateway emite beacon y nadie lo ve | Endpoints en otra PMK | Comparar claves | Reflashear endpoints primero |
| `endpoint_not_found` con ruta correcta | ID mal asignado en el firmware | `list` del gateway | Corregir `NODE_ID` |
| El PC perdió el mapa | Crash del controlador | `topologia.json` vacío o viejo | Esperar un ciclo de beacons; reconstruir |
| El dashboard no responde | Es una maqueta | No hay puerto serie | Usar el controlador o el monitor |
| Se cae todo al apagar el Bridge | Comportamiento esperado | — | El Bridge es punto único de fallo |

### 8.1 Diagnóstico rápido en 60 segundos

```
1. ¿El LED de alimentación del nodo está encendido?          → si no: alimentación
2. ¿El monitor serial muestra el arranque?                    → si no: firmware/puerto
3. ¿El Bridge responde a `stats`?                             → si no: Bridge o cable
4. ¿El nodo aparece en `list`?                                → si no: PMK o alcance
5. ¿`ping` responde con RSSI razonable?                       → si no: antena/entorno
6. ¿El comando con ruta llega al nivel esperado?              → si no: puente caído
```

### 8.2 Cuándo reiniciar y cuándo no

**Reiniciar** si: el nodo dejó de responder y el watchdog no actuó, o si cambiaste la PMK.

**No reiniciar** si: estás diagnosticando un fallo intermitente. Un reinicio borra la evidencia (contadores, uptime, último error). Leé `stats` primero.

---

## 9. Especificaciones técnicas

### 9.1 Radio

| Parámetro | Valor |
|---|---|
| Protocolo | ESP-NOW (802.11 vendor action frames) |
| Banda | 2.4 GHz |
| Canal | 1 (fijo) |
| Potencia | Según módulo (típico 20 dBm) |
| Alcance | 30–100 m en interior, según obstáculos |
| Payload máximo | 250 B |
| Peers por interfaz | 20 |
| Peers por contexto (puente) | 10 |

### 9.2 Protocolo

| Parámetro | Valor |
|---|---|
| Trama de comando | 38 B empaquetados |
| Beacon | 6 B |
| Anuncio | 5 B |
| Tamaño de ruta | 16 B (8 tramos) |
| Codificación de ruta | ASCII, separador `:` |
| Identificador de mensaje | 4 B |
| Timestamp | 32 bits, ms |

### 9.3 Tiempos

| Parámetro | Valor por defecto |
|---|---|
| Intervalo de beacon | 2000 ms |
| Ventana de escucha promiscua | 200 ms cada 5000 ms |
| Time-slice del puente | 100 ms |
| Timeout de detección | 3 × 5000 ms |
| Timeout de watchdog | 120 s |
| Re-enraizamiento | 5–30 s |
| RTT nivel 0 (estimado) | 5–15 ms |
| RTT nivel 2 (estimado) | 25–200 ms |

### 9.4 Consumo

| Nodo | Corriente típica | Pico |
|---|---|---|
| Endpoint (Wi-Fi activo) | 70–80 mA | 300 mA |
| Gateway | 80–90 mA | 320 mA |
| Bridge_0 | 90–100 mA | 350 mA |

---

## 10. Advertencias de seguridad

- **Las PMK son secretas.** Si se filtran, cualquiera con una radio en canal 1 puede descifrar el tráfico de esa célula.
- **El cifrado C1 es débil por diseño.** No lo uses para datos sensibles fuera del modelo de amenaza doméstico.
- **No hay autenticación de nodos.** Un dispositivo con la PMK correcta es aceptado como parte de la célula.
- **La EEPROM es legible físicamente.** No dejes módulos con claves en lugares accesibles a terceros.
- **El canal 1 es público.** Cualquiera puede intentar denegación de servicio saturando el aire.
- **No uses esta red para funciones de seguridad de personas.** No es un sistema certificado.

---

## 11. Glosario

| Término | Definición |
|---|---|
| **Célula** | Gateway más sus endpoints directos, con PMK propia. |
| **Endpoint-bridge** | Endpoint con doble rol: recibe de su gateway y reenvía al de abajo. |
| **PMK** | Primary Master Key de ESP-NOW, 16 bytes, aplicada por hardware. |
| **LMK** | Local Master Key derivada por el SDK para cada par de peers. |
| **Ruta explícita** | Secuencia de tramos escrita en el mensaje. |
| **Time-division** | Alternancia de contextos de PMK en un nodo con una sola radio. |
| **Beacon** | Mensaje periódico de un gateway anunciando su existencia. |
| **ANNOUNCE_UP** | Mensaje de un endpoint reportando un gateway inferior detectado. |
| **Fractal** | Estructura que se repite igual a distintas escalas. |

---

*netEspNow v1.4 · datasheet de operador · `Copyright (c) 2026 wanderc137` · MIT*
