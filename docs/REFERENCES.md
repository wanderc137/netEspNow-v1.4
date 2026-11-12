# Referencias — netEspNow v1.4

Fuentes externas consultadas y útiles para entender, replicar o auditar este proyecto. Cada entrada incluye por qué está acá y qué mirar.

> Todos los enlaces apuntan a documentación oficial o a repositorios mantenidos. Si un enlace cambia, buscá el título del documento: los contenidos de Espressif y del core de ESP8266 se mueven de URL con frecuencia.

---

## 1. ESP-NOW

**ESP-NOW — ESP-IDF Programming Guide (Espressif)**
<https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html>

La referencia canónica del protocolo. Mirar especialmente:

- La sección de **limitaciones**: 250 bytes de payload, 20 peers por interfaz, y las restricciones de convivencia con Wi-Fi (el canal debe coincidir con el del AP si el nodo está asociado).
- El **modelo de seguridad**: cómo se aplica la PMK, cómo se deriva la LMK por par y qué pasa si dos peers tienen claves distintas (las tramas se descartan sin error visible en la aplicación).
- El ciclo de vida: `init` → `register_callbacks` → `add_peer` → `send`.

Es la fuente que resuelve la mayoría de las dudas de implementación del proyecto.

**ESP-NOW User Guide (PDF)**
<https://www.espressif.com/sites/default/files/documentation/esp-now_user_guide_en.pdf>

Documento más antiguo pero con diagramas de flujo de los callbacks de envío y recepción. Útil para entender **cuándo** se invoca cada callback, que es justo lo que determina si se puede llamar a `esp_now_send` desde dentro de un callback (respuesta: mejor no).

---

## 2. Core de Arduino para ESP8266

**Arduino core for ESP8266 — repositorio oficial**
<https://github.com/esp8266/Arduino>

El código que realmente se compila. Los archivos relevantes:

- `cores/esp8266/espnow.h` — la API C que usa este proyecto.
- `libraries/ESP8266WiFi/src/ESP8266WiFi.cpp` — modo estación, canal, MAC.
- `tools/sdk/` — el SDK de Espressif embebido, donde están los límites reales.

Los *issues* cerrados son una mina de oro: casi cualquier comportamiento extraño (reinicios por IRAM, callbacks que no se llaman, peers que no se registran) ya fue reportado y explicado.

**Documentación de ESP-NOW en el core**
<https://arduino-esp8266.readthedocs.io/en/latest/espnow.html>

Versión "Arduino" de la API: `esp_now_init`, `esp_now_set_pmk`, `esp_now_add_peer`, `esp_now_register_recv_cb`. Menos completa que la de Espressif pero con ejemplos que compilan tal cual.

**Gestor de placas y opciones de compilación**
<https://arduino.github.io/arduino-cli/latest/configuration/>

Para entender `--fqbn`, las opciones de placa (`eesz`, `xtal`, `ip`) y cómo se pasan. Necesario para compilar un ESP-01 con 1 MB de flash sin equivocarse.

---

## 3. Criptografía

**uECC — curvas elípticas para microcontroladores**
<https://github.com/kmackay/micro-ecc>

La librería que usa el Bridge_0. Puntos a mirar:

- Tamaño de código y consumo de pila según la curva elegida (secp160r1 es la más liviana; secp256r1 la más segura).
- La necesidad de una función de números aleatorios (`uECC_set_rng`). Sin ella, la firma es determinista y mucho más débil.
- El costo en ciclos: es la razón por la que el enlace ECC tarda 50–200 ms en un ESP8266.

**BearSSL**
<https://www.bearssl.org/>

La librería de TLS que el core incluye y que este proyecto declara pero no aprovecha. El documento *"BearSSL: Implementing the SSL/TLS protocol in portable C"* explica el diseño orientado a sistemas embebidos: sin allocaciones dinámicas, tamaño de código acotado, y por qué es la base correcta para una futura migración a ECDH real.

**NIST SP 800-57 — gestión de claves**
<https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final>

Referencia conceptual para la decisión de "una clave por nivel" en lugar de "una clave por par". No es un documento de implementación, pero ordena el vocabulario (clave maestra, clave de sesión, rotación, criptoperiodo).

---

## 4. Radio y 802.11

**IEEE 802.11 — estándar**
<https://standards.ieee.org/ieee/802.11/7028/>

Solo para lo básico: por qué los canales 1, 6 y 11 son los únicos que no se solapan en 2.4 GHz, y por qué el proyecto fija el canal 1 (es el que menos solapamiento tiene con las redes domésticas típicas si el operador elige bien su AP).

**ESP8266EX Datasheet**
<https://www.espressif.com/sites/default/files/documentation/0a-esp8266ex_datasheet_en.pdf>

Consumo por modo (transmisión, recepción, sleep), pines disponibles y límites eléctricos. Es la fuente de la advertencia sobre los 300 mA de pico y de la tabla de alimentación.

**ESP8266 Hardware Design Guidelines**
<https://www.espressif.com/sites/default/files/documentation/esp8266_hardware_design_guidelines_en.pdf>

Layout de antena, desacople y qué pasa cuando la fuente no es estable. Explica por qué un capacitor de 470 µF junto al módulo no es opcional.

---

## 5. Protocolos de malla (para comparar)

Estos no se usan en el proyecto, pero son la alternativa que se evaluó y descartó. Sirven para entender por qué la ruta explícita gana en esta escala.

**Babel — routing protocol for mesh networks**
<https://www.irif.fr/~jch/software/babel/>

Protocolo de vector de distancia pensado para redes inalámbricas inestables. Interesante por cómo maneja la métrica y la convergencia rápida. Requiere una tabla de rutas por nodo: exactamente lo que este diseño evita.

**BATMAN-adv**
<https://www.open-mesh.org/projects/batman-adv/wiki>

Malla de capa 2 sobre Wi-Fi. Funciona muy bien en Linux, no tiene sentido en un ESP8266 sin pila IP.

**RFC 6550 — RPL (IPv6 Routing Protocol for Low-Power and Lossy Networks)**
<https://datatracker.ietf.org/doc/html/rfc6550>

El estándar para redes de sensores. Construye un DAG orientado al sumidero, con métricas de costo y reparación local. Es la respuesta "correcta" a este problema, y también la razón por la que no se usa: implementar RPL en 80 KB de RAM no es viable.

**RFC 4944 — Transmisión de IPv6 sobre 802.15.4**
<https://datatracker.ietf.org/doc/html/rfc4944>

Contexto de por qué las redes de sensores suelen evitar IP. ESP-NOW tampoco la usa.

---

## 6. Herramientas

**arduino-cli**
<https://arduino.github.io/arduino-cli/latest/>

Instalación, `core install`, `compile`, `upload`, `monitor`. La guía de instalación es la que se cita en el README; el resto de los comandos están en la referencia de cada subcomando.

**pyserial**
<https://pyserial.readthedocs.io/en/latest/>

Para el controlador del PC: apertura del puerto, timeouts, lectura por líneas y manejo de errores. El detalle importante es que `readline()` puede bloquear si no se configura `timeout`.

---

## 7. Cómo leer estas fuentes

Sugerencia de orden para quien quiera replicar el proyecto:

1. **ESP-NOW User Guide** completo, para entender el modelo de callbacks y el tamaño de trama.
2. **Sección de seguridad de ESP-IDF**, para el modelo PMK/LMK. Es lo que hace posible el cifrado por nivel.
3. **`espnow.h` del core**, para ver qué funciones existen realmente en el ESP8266 (no todas las de ESP32).
4. **uECC**, solo si vas a tocar el enlace ECC. Es la parte más frágil del sistema.
5. **Babel y RPL**, para entender qué se está evitando y por qué.

---

## 8. Lo que no se encontró

Para ser honesto con el estado del arte: no existe una implementación de referencia pública de una red ESP-NOW multi-nivel con enrutamiento por ruta explícita y PMK por nivel. Lo más cercano son proyectos de malla ESP-NOW de dos niveles con tabla de vecinos, y ejemplos de *painlessMesh* (que usa su propio protocolo sobre ESP8266/ESP32 y no ESP-NOW puro).

Eso significa dos cosas: el diseño no se apoya en un trabajo previo directamente aplicable, y tampoco hay una comunidad que haya tropezado ya con estos problemas. La bitácora del proyecto ([`BITACORA.md`](BITACORA.md)) es, en ese sentido, la fuente más útil para quien venga después.

---

*netEspNow v1.4 · referencias · `Copyright (c) 2026 wanderc137` · MIT*
