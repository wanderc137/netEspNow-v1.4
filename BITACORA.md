# BITACORA.md — netEspNow v1.4 · Arquitectura Fractal

> Registro cronológico de diseño, implementación y lecciones aprendidas.

---

## 2026-08-10 — Diseño Conceptual

### Sesión de Diseño
- **Concepto:** Extender ESP-NOW de topología plana (v1.3) a árbol fractal multi-nivel.
- **Inspiración:** "Como las raíces de un árbol, crece sin planeamiento central."
- **Usuario:** Idea original de endpoint como bridge recursivo + auto-descubrimiento.

### Decisiones Clave

| Decisión | Alternativas | Razón |
|---|---|---|
| Ruta explícita en mensaje | Tabla de rutas global en cada gateway | RAM constante, escalabilidad ilimitada |
| PMK nativa ESP-NOW por nivel | ECC en todos los enlaces | Menos overhead, más rápido, nativo del SDK |
| Time-division en endpoint-bridge | Dos radios (imposible en ESP8266) | Única radio, alternar contextos |
| Beacons solo hacia arriba | Broadcast a todos los niveles | Sin tormenta de beacons, tráfico de gestión O(n) no O(n²) |
| PC mantiene el mapa global | Gateway_0 mantiene tabla | RAM del PC ilimitada, ESPs sin carga |

### Hallazgos de Auditoría de Seguridad (v1.3)

**Revisión del código v1.3 existente:**

1. **Bridge ↔ Gateway: Cifrado ECC+XOR+ECDSA** ✅ Funcional pero con limitaciones:
   - ECDH falso: XOR de claves en vez de Diffie-Hellman real
   - Hash de firma: XOR simple de 32 bytes en vez de SHA-256
   - Mismo par de claves hardcodeado en ambos dispositivos
   - BearSSL configurado pero subutilizado

2. **Gateway ↔ Nodos: TEXTO PLANO** ❌ Sin cifrado ni autenticación:
   - Cualquier dispositivo en Canal 1 puede escuchar/inyectar mensajes
   - Los nodos aceptan cualquier mensaje con node_id y msg_type correctos
   - No se usa PMK/LMK nativa de ESP-NOW

3. **NO hay PMK de ESP-NOW en ningún enlace** ❌

**Decisión para v1.4:**
- Conservar ECC Bridge_0↔Gateway_0 (compatibilidad)
- Agregar PMK nativa en TODOS los enlaces intra-nivel
- Endpoint-bridge usa 2 PMKs simultáneas
- Las limitaciones del ECC (XOR falso, hash débil) se documentan para v1.5

### Distribución de Hardware

| Origen | Cant | Destino |
|---|---|---|
| Red v1.3 existente | 6 | Nivel 0 (Bridge_0 + Gateway_0 + 4 endpoints) |
| Nuevas (ya compradas) | 10 | Nivel 1 (5) + Nivel 2 (5) |
| A comprar | 5 | Completar Nivel 2 (necesita 6, tiene 1 nuevo + 5) |
| **Total** | **22** | 3 niveles |

### Estructura del Proyecto

```
ESP-NOW-v1.4-Fractal/
├── PLAN.md                    ← este documento de diseño
├── BITACORA.md                ← este registro
├── README.md                  ← documentación general
├── nivel_0/
│   ├── firmware_bridge_root.ino    (basado en v1.3 bridge)
│   ├── firmware_gateway.ino        (nuevo, con PMK_N0)
│   ├── firmware_endpoint_bridge.ino (nuevo, E0.1 doble rol)
│   └── firmware_endpoint.ino       (actualizado de v1.3 nodo)
├── nivel_1/
│   ├── firmware_gateway.ino        (mismo que nivel_0, NIVEL=1)
│   ├── firmware_endpoint_bridge.ino (mismo, E1.1 doble rol)
│   └── firmware_endpoint.ino       (mismo que nivel_0)
├── nivel_2/
│   ├── firmware_gateway.ino        (mismo, NIVEL=2)
│   └── firmware_endpoint.ino       (mismo)
├── pc/
│   └── fractal_controller.py
└── docs/
    ├── datasheet.md
    └── developer.md
```

### Pendientes para Implementación

- [ ] Definir PMK_N0, PMK_N1, PMK_N2
- [ ] Obtener MACs de los 10 ESPs nuevos
- [ ] Flashear Nivel 0 con firmware v1.4
- [ ] Probar compatibilidad con cutecom (comandos v1.3)
- [ ] Flashear Nivel 1
- [ ] Probar comunicación N0→N1
- [ ] Flashear Nivel 2 (tras comprar 5 ESPs)
- [ ] Probar comunicación extremo a extremo
- [ ] Medir latencia por nivel
- [ ] Probar failover y auto-reparación

---

## Lecciones Aprendidas

### Del diseño
1. La ruta explícita es más simple y escalable que una tabla de rutas distribuida.
2. El PC (RAM ilimitada) debe hacer el trabajo pesado de topología.
3. El endpoint-bridge es el componente más complejo: doble contexto, time-division, buffer circular.
4. Las claves por nivel son más manejables que claves por par (10 PMKs vs 100+ pares).

### De v1.3
5. El IRAM del Bridge_0 al 91% es una restricción real. No agregar código pesado ahí.
6. El cifrado ECC es lento (~200ms). La PMK nativa es instantánea (~1ms).
7. El formato de mensaje de 21 bytes era demasiado rígido. 35 bytes da margen para rutas.
8. Cutecom como interfaz es práctico para depuración pero limita comandos complejos.
