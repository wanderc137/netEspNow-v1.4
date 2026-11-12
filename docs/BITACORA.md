# Bitácora de ingeniería — netEspNow v1.4

Registro de decisiones, descartes y correcciones del diseño de la red fractal. No es un changelog: es el razonamiento detrás de las decisiones, incluido el que llevó a descartar cosas que parecían buenas ideas.

> **Estado del proyecto al cerrar esta bitácora:** diseño completo y auditado; firmware no implementado. Las carpetas de nivel están vacías.

---

## 2026-08-10 — Punto de partida

### Lo que había

Una red ESP-NOW funcionando: siete ESP8266, topología estrella plana, un Bridge conectado al PC por USB, un Gateway y seis nodos. Comandos por serial desde `cutecom`, formato de texto simple (`ping`, `set`, `stats`), RTT medido de **1385 ms**.

### Lo que se quería

Agregar profundidad. La pregunta inicial no era "¿cómo hago una malla?" sino algo más concreto: **¿qué se rompe primero si cuelgo un gateway de un nodo?**

La respuesta corta: se rompe el direccionamiento. En una estrella, el Gateway conoce a todos sus nodos y el Bridge conoce al Gateway. Con dos niveles, el Bridge tendría que saber a qué gateway mandar cada comando, y ese gateway tendría que saber qué nodos dependen de él y cuáles de un segundo gateway. Es una tabla de rutas, y una tabla de rutas en un ESP8266 es donde empiezan los problemas.

### La idea que destrabó todo

En lugar de que los nodos sepan cómo llegar a un destino, que **el mensaje lleve escrito el camino**. El PC, que tiene RAM de sobra, calcula la ruta completa; los nodos solo leen el primer tramo, lo borran y reenvían.

Esa decisión, tomada en la primera sesión, condicionó todo lo demás. Vale la pena registrar por qué se tomó así y no de otra forma.

---

## 2026-08-10 — Auditoría de la versión anterior

Antes de diseñar nada nuevo se revisó el código existente. El resultado fue incómodo y necesario.

### Hallazgo 1 — El cifrado Bridge↔Gateway es más débil de lo que parece

El enlace usaba curvas elípticas (`uECC`) y firmas (`ECDSA`), lo que en la superficie suena sólido. Al leer el código:

- La clave compartida se obtiene con un **XOR de las claves**, no con una multiplicación de punto. No es ECDH.
- El digest que se firma es un **XOR de 32 bytes**, no SHA-256.
- El **mismo par de claves** está compilado en ambos extremos: no hay identidad por dispositivo.
- BearSSL está incluido en el core y configurado, pero no se usa para nada.

Consecuencia: el enlace ofrece ofuscación fuerte, no criptografía robusta. Un atacante con el firmware en la mano reproduce las claves.

### Hallazgo 2 — El tramo Gateway↔Nodos va en texto plano

Cualquier dispositivo en el canal 1 puede leer los comandos y las respuestas, y puede inyectar mensajes si acierta el `node_id` y el `msg_type`. No hay autenticación ni cifrado.

### Hallazgo 3 — No hay PMK en ningún enlace

ESP-NOW soporta cifrado nativo con PMK/LMK desde siempre. La versión anterior no lo usaba en absoluto.

### La decisión que salió de la auditoría

Arreglar los tres hallazgos a la vez era imposible sin romper compatibilidad y sin reflashear todo. Se priorizó:

| Hallazgo | Qué se hizo | Por qué |
|---|---|---|
| ECC débil | Se conservó tal cual | Cambiarlo obliga a reflashear ambos extremos y cambiar la trama |
| Texto plano intra-nivel | **Se arregló**: PMK por nivel | Es el mayor riesgo real y se resuelve con hardware |
| Sin PMK | **Se agregó** | Mismo punto anterior |

La deuda de C1 quedó documentada explícitamente, en el README y en el código, para que nadie la confunda con seguridad real.

---

## Decisiones de diseño

### D1 — Ruta explícita en el mensaje

**Alternativas evaluadas:**

1. Tabla de rutas en cada gateway, actualizada por un protocolo de estado de enlace.
2. Inundación de mensajes (*flooding*) con identificador para evitar duplicados.
3. Ruta explícita calculada por el emisor.

**Por qué se eligió la 3:**

- La 1 requiere RAM proporcional al tamaño de la red y un protocolo de convergencia. En 80 KB no entra con holgura, y cada nodo nuevo agrega estado a todos.
- La 2 gasta el canal: cada mensaje se transmite N veces. Con 22 nodos y beacons cada 2 s, el aire se satura.
- La 3 gasta bytes (hasta 16 por mensaje) pero **RAM cero** en los nodos. El costo se paga en ancho de banda, que sobra: las tramas son pequeñas y la tasa de polling es baja.

**Consecuencia aceptada:** el emisor necesita conocer la topología. Por eso existen los beacons y el mapa en el PC.

### D2 — PMK por nivel en lugar de por par

**Alternativa:** una clave por par de dispositivos (231 pares para 22 nodos).

**Por qué se descartó:** gestión. Rotar una clave implicaría reflashear decenas de nodos en el orden correcto. Con claves por nivel, rotar afecta a una célula completa y el procedimiento es lineal.

**Costo:** un nodo comprometido en un nivel compromete **ese nivel**, no toda la red. Es una segmentación aceptable para el modelo de amenaza.

### D3 — Time-division en el endpoint-bridge

El puente necesita hablar con dos gateways que usan claves distintas. El ESP8266 tiene **una sola radio**.

**Alternativas:**

1. Dos módulos ESP8266 por puente, uno por nivel. → Duplica costo, consumo y puntos de fallo.
2. Una PMK global para todos los niveles. → Destruye la segmentación; es volver al problema original.
3. Alternar el contexto de PMK cada 100 ms. → Se eligió.

**Por qué la 3, sabiendo que cuesta latencia:** es la única que mantiene la propiedad de segmentación sin agregar hardware. Los 100 ms de peor caso son irrelevantes para domótica y descalifican el diseño para control de motores, cosa que está documentada.

**Detalle de implementación que no era obvio:** cambiar la PMK no alcanza. Los peers registrados conservan su LMK derivada de la clave anterior, así que hay que reconstruir la tabla (`deinit` + `init` + `set_pmk` + `add_peer`). Eso motivó el buffer circular: durante la reconstrucción, las tramas que llegan se pierden si no hay dónde guardarlas.

### D4 — Beacons dentro de la PMK del nivel

**Alternativa:** beacon en texto plano, para que cualquier nodo lo vea.

**Por qué se descartó:** una tormenta de beacons. Con 3 gateways emitiendo cada 2 s, todos los nodos de todos los niveles procesarían todos los beacons. Peor: un nodo de nivel 0 vería beacons de nivel 2 y trataría de anunciarlos, generando anuncios falsos.

**Efecto secundario valioso:** el beacon se vuelve un mecanismo de filtrado natural. Solo lo descifra quien pertenece a la célula.

**Costo:** descubrir un gateway de nivel inferior requiere abrir una ventana promiscua. Eso es una concesión de seguridad (200 ms cada 5 s sin filtro de PMK) que se documenta en el README.

### D5 — El PC mantiene el mapa

**Alternativa:** que Gateway_0 mantenga la topología y la distribuya.

**Por qué se descartó:** Gateway_0 es el nodo más restringido (IRAM al 91%). Darle el mapa lo convierte en el cuello de botella y en el punto único de fallo lógico, además del físico.

**Consecuencia:** sin PC no hay comunicación entre niveles. Es una limitación real que está en la lista de limitaciones aceptadas, no escondida en un pie de página.

---

## Descartes

Estas son las ideas que se evaluaron, se intentaron o se documentaron, y se abandonaron. Están acá para que nadie las reintente sin saber por qué fallaron.

### X1 — Invertir la ruta para la vuelta

**Idea:** el endpoint destino recibe `1:1:3`, la invierte a `3:1:1`, y cada gateway en la vuelta lee el primer elemento para saber a quién entregar.

**Por qué falla:** en el nivel 1, el puente hacia arriba es **E0.1**, un endpoint del nivel 0. Un índice relativo al nivel no puede expresarlo. Para que funcione haría falta una ruta absoluta con nivel (`N0.1:N1.1:N2.3`), que ocupa el doble y complica el parser.

**Qué se hizo en cambio:** memoria de padre. Cada nodo guarda la MAC de su padre (6 bytes) y la vuelta es un reenvío directo. Más simple, más barato y sin estado por mensaje.

### X2 — Dos radios por endpoint-bridge

**Idea:** un ESP8266 por contexto, comunicados por serial o GPIO.

**Por qué se descartó:** duplica el costo por puente, duplica el consumo (crítico si el nodo va a batería) y agrega un enlace interno que puede fallar. Además, el ESP-01 no tiene pines libres para eso.

### X3 — Broadcast de beacons a todos los niveles

**Idea:** que todos los nodos vean todos los beacons y decidan.

**Por qué se descartó:** tráfico O(n²) y anuncios falsos. Un nodo de nivel 0 que ve un beacon de nivel 2 no tiene forma de saber si ese gateway está debajo de él o en otra rama.

### X4 — Tabla de rutas en Gateway_0

**Idea:** Gateway_0 aprende la topología por los anuncios y calcula rutas.

**Por qué se descartó:** IRAM. Gateway_0 ya está al límite con el stack de ESP-NOW; agregarle un grafo y un algoritmo de camino mínimo es pedirle que falle en runtime. Además, si Gateway_0 muere, se pierde el mapa completo.

### X5 — ECDH real en todos los enlaces

**Idea:** reemplazar la PMK por ECDH entre cada par.

**Por qué se descartó:** 200 ms por operación en un ESP8266, multiplicado por cada salto y cada mensaje. La red sería inoperable. La PMK es cifrado de hardware: ~1 ms.

### X6 — JSON en los nodos

**Idea:** que los ESP parseen JSON directamente, como hace el PC.

**Por qué se descartó:** una librería JSON decente ocupa decenas de KB de flash y heap. En el Bridge, con IRAM al 91%, es inviable. La trama binaria empaquetada es más chica, más rápida y no falla por un espacio de más.

### X7 — Comandos de texto para todos los niveles

**Idea:** extender `ping` y `set` con un prefijo de nivel (`ping 2 3`).

**Por qué se descartó:** no escala a rutas arbitrarias y el parseo se vuelve ambiguo. El canal JSON con `ruta` explícita es más expresivo, y los comandos viejos se conservan solo para compatibilidad del nivel 0.

---

## Correcciones al documento de diseño

El diseño se escribió antes que el código, y escribir prosa obliga a hacer cuentas. Estas son las inconsistencias que se detectaron al revisar y cómo se resolvieron. Se registran porque son exactamente el tipo de error que después cuesta días en el hardware.

### C1 — El tamaño de la trama no era 35 bytes

La suma de los campos del struct empaquetado da **38 bytes**:

```
1 (msg_type) + 1 (ruta_len) + 16 (ruta) + 4 (msg_id) + 4 (timestamp)
+ 1 (cmd) + 1 (pin) + 2 (val) + 8 (payload) = 38
```

El número 35 aparecía porque se omitía el timestamp o el payload al contar. Se corrigió en toda la documentación y se agregó la recomendación de usar `static_assert` para que el compilador lo verifique.

### C2 — El límite de profundidad no era 50 niveles

Con `ruta[16]` y tramos de 2 bytes (`N:`), el máximo son **8 tramos**. El "50" era un cálculo teórico hecho antes de fijar el tamaño del buffer. Se documenta el límite real y la vía de escape (`ruta[32]` → 16 tramos).

### C3 — El "failover de 0 ms" era marketing

La conmutación en sí no tiene handshake: es un cambio de índice en la tabla de peers. Pero la **detección** de que el puente murió lleva 3 timeouts, o sea 15 s por defecto. Se separó el concepto: "conmutación sin establecimiento" vs "tiempo de reacción". El README lo dice explícitamente.

### C4 — El ANNOUNCE_UP no está autenticado

El diseño original lo trataba como un mensaje de confianza. No lo es: cualquiera con la PMK del nivel puede emitirlo. Se documentó como limitación conocida en lugar de asumir que el problema no existía.

### C5 — El beacon de 7 bytes son 6

`msg_type` (1) + `gw_level` (1) + `gw_id` (1) + `pmk_id` (1) + `uptime_s` (2) = **6 bytes**. El séptimo venía de un padding que no existe con `packed`. Se corrigió.

### C6 — El ejemplo de ruta en el flujo tenía un error

El primer ejemplo usaba `3:1:3` para llegar a E2.3, asumiendo que E0.3 era el puente. El puente es E0.1, así que la ruta correcta es `1:1:3`. Se corrigió el ejemplo y, más importante, se escribió el **invariante** ("el primer número es el endpoint del nivel que recibe el mensaje") para que el error no se repita.

---

## Lo que se aprendió

### Sobre el diseño

1. **La ruta explícita es más simple que cualquier protocolo de enrutamiento**, y para una topología semi-estática con un operador, es la correcta. La complejidad se traslada al nodo con más recursos.
2. **El PC debe hacer el trabajo pesado.** Intentar que los microcontroladores "sean inteligentes" es la receta para quedarse sin RAM.
3. **El endpoint-bridge es el 80% de la complejidad del sistema.** Doble contexto, time-division, buffers, dos watchdogs. Si algo va a fallar, va a fallar ahí.
4. **Las claves por nivel son manejables; las claves por par, no.** Tres claves se rotan a mano; 231, no.
5. **Escribir el diseño antes del código detectó seis errores de cuenta.** En papel cuestan una tarde; en hardware cuestan días.

### Sobre el hardware

6. **La IRAM del ESP8266 es la restricción real**, no la RAM libre. Un sketch puede compilar y fallar en runtime por IRAM.
7. **El cifrado ECC es lento** (50–200 ms). La PMK es instantánea. Esa diferencia justifica toda la arquitectura de claves.
8. **Una radio es una radio.** No hay forma de estar en dos redes a la vez; solo alternar, y alternar cuesta latencia.
9. **La alimentación explica más fallos que el firmware.** Un módulo que reinicia solo casi nunca es un bug de código.

### Sobre el proceso

10. **Documentar los descartes vale tanto como documentar las decisiones.** Sin el "por qué no", el siguiente intenta lo mismo.
11. **Los números redondos mienten.** "35 bytes", "50 niveles", "0 ms" eran todos aproximaciones que se volvieron afirmaciones al escribirlas.
12. **La honestidad sobre el estado del proyecto no resta valor, lo concentra.** Un diseño claro y un firmware ausente es más útil que un firmware a medias presentado como terminado.

---

## Estado y pendientes

### Hecho

- [x] Topología de 3 niveles y 22 nodos definida
- [x] Cuatro roles de firmware especificados
- [x] Formato de mensajes (comando, beacon, anuncio)
- [x] Algoritmo de ruta explícita, ida y vuelta
- [x] Esquema de seguridad en tres capas
- [x] Análisis de latencia y de RAM con supuestos explícitos
- [x] Plan de despliegue en cuatro fases
- [x] Ocho casos de prueba funcionales y seis de estrés
- [x] Auditoría de la versión anterior y registro de su deuda técnica
- [x] Dashboard visual de la topología (prototipo)
- [x] Documentación de operador y de desarrollador
- [x] Diagramas de arquitectura, ruta y seguridad

### Pendiente, en orden

- [ ] Definir las tres PMK reales y guardarlas fuera del repositorio
- [ ] Obtener las MAC de los 22 módulos
- [ ] Implementar `firmware_gateway.ino`
- [ ] Implementar `firmware_endpoint.ino`
- [ ] Implementar `firmware_endpoint_bridge.ino`
- [ ] Implementar `firmware_bridge_root.ino` (mínimo, sin crecer)
- [ ] Validar el nivel 0 con hardware real
- [ ] Implementar `fractal_controller.py`
- [ ] Medir RTT por nivel y contrastar con el modelo
- [ ] Probar failover y re-enraizamiento
- [ ] Ejecutar las pruebas de estrés

### Deuda técnica declarada

| Deuda | Impacto | Versión objetivo |
|---|---|---|
| C1 usa XOR en lugar de ECDH y de SHA-256 | Alto | v1.5 |
| Bridge_0 es punto único de fallo | Alto | v1.5 |
| No hay autenticación de nodos | Medio | v1.6 |
| No hay cifrado de extremo a extremo | Medio | v2.0 |
| Un solo gateway hijo por puente | Bajo | v1.6 |
| Sin fragmentación de payload | Bajo | sin plan |

---

## Cómo continuar este proyecto

Para quien retome el trabajo:

1. **Empezá por el nivel 0 y no avances hasta que `ping` y `set` funcionen igual que antes.** Es el único criterio de compatibilidad que importa al principio.
2. **No toques el Bridge_0.** Si necesitás agregar algo ahí, probablemente va en el PC.
3. **Medí antes de optimizar el time-slice.** El valor de 100 ms es una hipótesis, no una verdad medida.
4. **Si el primer Gateway_1 emite beacon y nadie lo escucha, revisá el orden de flasheo antes que el código.** Es el error más común y el más caro de diagnosticar.
5. **Actualizá esta bitácora con cada corrección.** El valor del documento está en que registra lo que falló, no solo lo que funcionó.

---

*netEspNow v1.4 · bitácora de ingeniería · `Copyright (c) 2026 wanderc137` · MIT*
