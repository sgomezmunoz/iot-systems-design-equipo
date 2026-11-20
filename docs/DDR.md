# Design & Decision Record (DDR) — SoilSense
**GreenField Technologies | IoT Systems Design (4201327), UNAL Manizales** — actualizado hasta el **Lab 5**

**Integrantes:**
1. Samuel Elias Gomez Munoz — sgomezmu@unal.edu.co (GitHub: @sgomezmunoz)
2. Juan Fernando Martinez Cabrera — jumartinezcab@unal.edu.co

> **Este DDR es la versión consolidada de los informes de laboratorio.** Si un informe no
> coincide con este documento, vale el DDR; las diferencias están en la
> [fe de erratas](informes/README.md).

Informes por laboratorio: [docs/informes/](informes/). Cada número de este documento cita el
archivo de [`data/`](../data/) del que sale; las tablas de `data/` están transcritas de los
informes y las capturas de consola extraídas de ellos (ver [data/README.md](../data/README.md)).

---

## 1. Descripción del sistema

*   **Tipo de sistema:** [ ] Componente (Lab 1-2) | [x] Sistema (Lab 3-6) | [ ] Entorno (Lab 7-8)
*   **Descripción:** Red mesh Thread/6LoWPAN de nodos ESP32-C6 (Zephyr + OpenThread) en el
    canal 15, PAN ID personalizado (0x2345 / 0xABCD). Los nodos exponen servicios CoAP/CBOR
    (`/env/temp` con Observe y `/act/valve` con PUT CON); el actuador es un Sleepy End Device con
    `pollperiod` de 5 s. Un OpenThread Border Router (host ESP32-S3 + RCP ESP32-C6 por UART)
    conecta la malla con la red Wi-Fi.
*   **Nodos:** Node A (Leader, servidor de telemetría), Node B (Router, cliente CLI), Node V (SED,
    electroválvula en GPIO8), C6 como RCP y ESP32-S3 como host del OTBR.
*   **Sensor:** DHT22, el que entregó el profesor; es el único sensor integrado.

---

## 2. Registro de laboratorio y resúmenes para las partes interesadas

### Lab 0: HTTP y MQTT
*   Firmware `lab0_http` completado (capacidades `WIFI`, `HTTP_SERVER`, `JSON_LIBRARY`, `LED_STRIP`).
    Sin informe propio; HTTP/JSON y MQTT/JSON se usan como referencia de bytes en el Lab 3.

### Lab 1: Caracterización RF
*   **Para Samuel:** canal 15 (−105 dBm de piso de ruido) frente a canal 26 (−64 dBm). Con TX a
    0 dBm el enlace es simétrico hasta 10 m (ΔRSSI ≤ 2 dB).
*   **Para Edwin:** separar los nodos como máximo 10 m: a esa distancia el PER es 1,0 %; a 20 m sube
    a 3,5–4,0 % y a 30 m a 10–12 %.

### Lab 2: Thread mesh
*   **Para Samuel:** 1 salto A→B: 15,85 ms promedio, 0 % de pérdida (20/20). 2 saltos A→B→C:
    30,15 ms promedio, 0 % de pérdida. Las aplicaciones deben usar la ML-EID, que no cambia
    cuando el nodo cambia de padre; la RLOC16 sí cambia.
*   **Para Edwin (Tractor Test):** al desconectar el Router B durante 200 pings, se perdieron las
    secuencias 71–78 (8 s, 4,0 % de pérdida) y la ruta se recuperó sola por el enlace directo A–C
    (16,08 ms promedio).

### Lab 3: CoAP y CBOR
*   **Para Daniela y Gustavo:** una lectura de temperatura ocupa ~36 B en el aire frente a ~530 B con
    HTTP/JSON (−93 %). Con Observe, el sensor envió 7 notificaciones en 5 min frente a 300 consultas
    de un poller a 1 Hz (−97,7 % de tráfico de radio).

### Lab 4: Sensores reales (DHT22)
*   **Para Daniela:** el DHT22 mide con ±0,5 °C (datasheet), ruido de ±0,10 °C en 1 min, offset de
    +0,25 °C frente a la referencia y tiempo de subida de ~4 s; se satura al 100 %RH.
*   **Para Edwin:** "sensor muerto" = el nodo responde **5.03** (Max-Age 10, "no fresh reading") a
    los 60 s de perder el sensor; "nodo caído" = el cliente recibe ResponseTimeout tras ~45 s de
    retransmisiones CON. La válvula responde en menos de 5 s.

### Lab 5: Border Router
*   **Para Daniela:** la temperatura y la válvula se consultan desde el celular o la laptop por
    Wi-Fi. Si el Border Router se apaga, la malla sigue funcionando sola (OTBR-off test: PUT
    `/act/valve` de B a V correcto con la S3 desenchufada).
*   **Para Edwin:** latencia dentro de la malla de 65–100 ms (mediana 81 ms); desde fuera, vía OTBR,
    de 59–469 ms (mediana 190 ms); 0 % de pérdida en ambos casos.

---

## 3. Architecture Decision Records (ADR)

### ADR-001: Canal 802.15.4 y separación entre nodos (Lab 1)
*   **Contexto:** la banda de 2,4 GHz se comparte con Wi-Fi y los 16 canales 802.15.4 se solapan con
    los canales Wi-Fi de 20 MHz. Escaneo pasivo con `ot scan energy 500`, TX a 0 dBm.
*   **Decisión:** canal 15 (2425 MHz) y separación entre nodos ≤ 10 m.
*   **Rationale (mediciones):** piso de ruido de −105 dBm en el canal 15 frente a −64 dBm en el canal
    26 ([scan_energy.csv](../data/lab1/scan_energy.csv)); el canal 15 cae en el hueco entre los
    canales Wi-Fi 1 y 6. PER de 0,0 % hasta 5 m, 1,0 % a 10 m (RSSI −65/−67 dBm), 3,5–4,0 % a
    20 m y 10–12 % a 30 m ([rssi_per_vs_distancia.csv](../data/lab1/rssi_per_vs_distancia.csv)).
    El umbral PER ≤ 1 % solo se cumple hasta **10 m**, así que la meta del curso (> 20 m) **no se
    alcanza** en interiores; la separación se fija con esa medición.
*   **Alternativas:** canal 26 (el más ruidoso); separación de 20 m (PER 3,5–4 %, descartada).
*   **Estado:** Aceptada.

### ADR-002: Topología Thread (mesh) frente a LoRaWAN (estrella) (Lab 2)
*   **Contexto:** 10 hectáreas con telemetría y control de válvulas en tiempo real.
*   **Decisión:** Thread/6LoWPAN en malla; 1–2 Routers con alimentación y el resto como Sleepy End
    Devices.
*   **Rationale (mediciones):** latencia de 15,85 ms en 1 salto y 30,15 ms en 2 saltos, con 0 % de
    pérdida; ~15 ms por salto adicional. En el Tractor Test la malla se recuperó en 8 s (secuencias
    71–78, 4,0 % de pérdida en 200 pings), muy por debajo del SLA de 120 s
    ([metricas_mesh.csv](../data/lab2/metricas_mesh.csv), capturas en
    [data/lab2/capturas](../data/lab2/capturas)). LoRaWAN tiene un duty cycle regulado del 1 %
    (segundos o minutos por comando) y depende de un único gateway.
*   **Contadores MAC en el Router B:** 42 TxUnicast, 42 TxAcked, 7 TxRetry: el relay funciona sin
    pérdidas.
*   **Estado:** Aceptada.

### ADR-003: CoAP/CBOR + Observe frente a MQTT/JSON (Lab 3)
*   **Contexto:** protocolo de aplicación para nodos de batería sobre 6LoWPAN/Thread.
*   **Decisión:** CoAP/UDP con payload CBOR de 6 B (`{"t": float16}`, Content-Format 60) y Observe
    (RFC 7641) con notificación si |ΔT| > 0,5 °C o heartbeat.
*   **Rationale (mediciones):** ~36 B por lectura frente a ~530 B con HTTP/JSON (14,7×, −93 %) y
    ~40 B con MQTT más el costo de mantener TCP abierto
    ([bytes_en_cable.csv](../data/lab3/bytes_en_cable.csv)); toda la transacción cabe en una trama
    802.15.4 de 127 B. Observe: 7 y 9 notificaciones frente a 300 GET en ventanas de 5 min, es decir
    −97,7 % y −97,0 % de tráfico de radio ([observe_vs_polling.csv](../data/lab3/observe_vs_polling.csv)).
*   **Estado:** Aceptada.

### ADR-004: `pollperiod` de 5 s para la válvula SED (Labs 3–4)
*   **Contexto:** el actuador funciona con batería y no puede mantener el receptor encendido (~15 mA en RX).
*   **Decisión:** Node V como Sleepy End Device con `pollperiod` = 5000 ms; PUT CON idempotente.
*   **Rationale (mediciones):** ([sed_poll_period.csv](../data/lab4/sed_poll_period.csv))

    | Poll period | Latencia peor caso | Corriente | Autonomía 2×AA | Decisión |
    |---|---|---|---|---|
    | 1 s | 1,2 s | ~1,50 mA | 47 días | descartado (batería) |
    | **5 s** | **4,8 s (medido)** | ~0,155 mA | **238 días** | **elegido** |
    | 30 s | 28,3 s | ~0,030 mA | 1389 días | descartado (latencia) |

    Un PUT llegó duplicado (el retransmit CON a 2–3 s es menor que el poll de 5 s) y no tuvo efecto
    porque PUT es idempotente.
*   **Estado:** Aceptada.

### ADR-005: OpenThread Border Router con host ESP32-S3 y RCP ESP32-C6 (Lab 5)
*   **Contexto:** la malla era una isla accesible solo desde una laptop en campo; Daniela necesita
    acceso remoto y Samuel integración con IP.
*   **Decisión:** OTBR con la ESP32-S3 como host Wi-Fi y una ESP32-C6 como RCP por UART; forwarding
    IPv6 Thread ↔ Wi-Fi, anuncio del prefijo OMR (`fd5f:4e3e:7bbf:1::/64`) y NAT64/DNS64.
*   **Rationale (mediciones):** RTT en malla con mediana de 81 ms (promedio 79,5); laptop → OTBR →
    Node A con mediana de 190 ms (promedio 221,0, máx. 469); 0 % de pérdida en 10 paquetes
    ([rtt_otbr.csv](../data/lab5/rtt_otbr.csv)). El OTBR agrega **+109 ms** y **no cumple la meta de
    < 100 ms** por 9 ms (traducción de cabeceras, forwarding en la S3, handoff a 802.15.4 y jitter
    del Wi-Fi). Se acepta igual porque el comando a la válvula (~300 ms extremo a extremo) queda
    muy dentro del requisito de < 5 s del Lab 4, y el OTBR-off test pasó.
*   **Alternativas:** nodo Thread conectado directo a Wi-Fi (descartado: Wi-Fi y Thread simultáneos
    en un C6 son inestables y mezclan planos).
*   **Estado:** Aceptada.

### ADR-T01: Tamaño de flash y consola USB por placa (equipo, Labs 2–5)
*   **Contexto:** la configuración del curso no coincidía con la flash de nuestras placas; además se
    necesitaba la consola por el puerto USB.
*   **Decisión:** `ESPTOOLPY_FLASHSIZE_4MB` en los C6 y `_16MB` en la S3; consola y shell por
    `usb_serial` (cambios listados en el [README](../README.md)).
*   **Estado:** Aceptada.

---

## 4. Mapeo ISO/IEC 30141

### Mapeo de dominios (Labs 1–5)

| Componente | Dominio ISO | Justificación |
|---|---|---|
| Suelo, aire, canal RF de 2,4 GHz | PED | Medio físico donde operan los sensores y se propaga la señal |
| ESP32-C6, radio 802.15.4, DHT22, checks de plausibilidad, válvula en GPIO8, SED parent-poll | SCD | Sensado, actuación y gestión de la red de proximidad |
| Leader y Routers Thread, MLE | RAID | Enrutamiento mesh, asignación de RLOC y Network Data |
| `/env/temp` (GET, Observe, Max-Age, 5.03) y `/act/valve` (PUT CON, CBOR `{v: 0\|1}`) | ASD | Servicios de aplicación |
| OTBR (ESP32-S3 + RCP C6) | RAID | Gateway pattern: conversión de protocolo y mapeo de direcciones |

Las redes Wi-Fi vecinas son SCD de otro sistema que comparte nuestro PED.

### Recorrido de las cuatro redes (Tabla A.4)

| Red | En SoilSense | Direcciones observadas |
|---|---|---|
| Proximity | Malla Thread (Node A, B, V, RCP) | Mesh-Local `fd78:98e:7330::/64`, OMR `fd5f:4e3e:7bbf:1::/64` |
| Access | Wi-Fi del OTBR (ESP32-S3) y hotspot | IPv4 `192.168.43.x` |
| Services | Servicios CoAP y dashboard | IPv6 OMR alcanzable desde la red local |
| User | Celular de Daniela y laptop de campo | Asignada por el hotspot |

### Funciones de gateway del OTBR (Tabla A.3)

| Función | Estado | Evidencia |
|---|---|---|
| Protocol Conversion | Activa | Handoff 802.15.4 ↔ 802.11 |
| Address Mapping | Activa | Prefijo OMR (SLAAC) y NAT64/DNS64 |
| Data Processing | Inactiva | No modifica los payloads CoAP/CBOR |
| Information Fusion | Inactiva | No agrega datos de varios sensores |
| Certification / Security | Parcial | Network Key de Thread; DTLS en el Lab 6 |
| Equipment Management | Activa | Web GUI del OTBR (commissioning, topología) |

### Capacidades de los componentes

| Categoría | Subcategoría | Componente/característica | Activa/Latente | Lab |
|---|---|---|---|---|
| Transferring | Network | `CONFIG_WIFI` | Activa | 0 |
| Interface | Application | `CONFIG_HTTP_SERVER` | Activa | 0 |
| Data Processing | Encoding | `CONFIG_JSON_LIBRARY` | Activa | 0 |
| Interface | Human UI | `CONFIG_LED_STRIP` (rol Thread / válvula) | Activa | 0 |
| Interface | Network | Servidor CoAP `/env/temp` (GET + Observe) | Activa | 3 |
| Data Transferring | Publish/Subscribe | Notificaciones Observe (RFC 7641) | Activa | 3 |
| Data Processing | Encoding | CBOR (payload de 6 B) | Activa | 3 |
| Interface | Application | `/act/valve` (PUT/GET CON) | Activa | 3 |
| Supporting | Power Mgmt | Sleepy End Device (`pollperiod` 5 s) | Activa | 3 |
| Sensing | Temperatura/humedad | DHT22 | Activa | 4 |
| Transferring | Gateway | OTBR (OMR, NAT64) | Activa | 5 |

---

## 5. Reflexiones de primeros principios

**Lab 1:** el canal 15 cae en el hueco entre los canales Wi-Fi 1 y 6. La ganancia de procesamiento
de DSSS (~9–10 dB) explica el enlace limpio a corta distancia. En interiores, a 10 m el RSSI ya es
−65 dBm, muy lejos de los >100 m del modelo de espacio libre.

**Lab 2:**
1. Un Router mantiene el receptor siempre encendido (~15 mA de RX + ~12 mA de CPU ≈ 30 mA); un SED
   duerme (~5 µA) y solo despierta para consultar a su padre.
2. Cada salto suma tiempo en el aire (~2,3 ms por 72 B a 250 kbps), procesamiento en el Router y
   backoff de CSMA/CA; se midieron ~15 ms por salto.
3. 6LoWPAN IPHC (RFC 6282) comprime la cabecera IPv6 de 40 B a 2–7 B, reconstruyéndola desde las
   direcciones MAC de la trama.

**Lab 3:**
1. CoAP usa UDP para no pagar el handshake ni el estado de TCP; la confiabilidad la dan los mensajes
   CON con backoff exponencial (RFC 7252 §4.8).
2. Con Observe, el servidor transmite solo cuando el valor cambia; sin cambios, la radio no transmite.
3. El PUT duplicado a la válvula SED fue inofensivo porque PUT es idempotente.
4. CBOR conserva el modelo JSON, se puede validar con CDDL y cabe en una trama 802.15.4.

**Lab 4:**
1. `/env/temp` responde 5.03 en vez de devolver un dato viejo como si fuera fresco (Max-Age).
2. "Sensor muerto" se ve como un 5.03 a los 60 s; "nodo inalcanzable", como ResponseTimeout a ~45 s.
3. Un check de plausibilidad (−40..85 °C, > 5 °C en 2 s) no detecta un valor falso pero creíble.
4. El DHT22 es de un solo hilo con timing por software; las interrupciones de radio pueden corromper
   la lectura, cosa que no le pasa a un bus I2C/SPI de hardware.
5. El `pollperiod` es una política de la capa MAC/Thread, no de CoAP.

**Lab 5:** el Border Router es una puerta adicional, no un punto único de falla: el plano funcional
(CoAP en la malla) sigue operando sin él (OTBR-off test).

---

## 6. Líneas base de desempeño

| Métrica | Objetivo | Medido | Fuente | Estado |
|---|---|---|---|---|
| Lab 1: alcance máx. (PER ≤ 1 %) | > 20 m | 10 m | `data/lab1/rssi_per_vs_distancia.csv` | **FAIL** |
| Lab 2: tiempo de healing | < 120 s | 8 s | `data/lab2/metricas_mesh.csv` | PASS |
| Lab 2: latencia 1 / 2 saltos | < 200 / < 500 ms | 15,85 / 30,15 ms | `data/lab2/metricas_mesh.csv` | PASS |
| Lab 3: bytes por lectura | ≤ 40 B | ~36 B | `data/lab3/bytes_en_cable.csv` | PASS |
| Lab 3: latencia CoAP | < 200 ms | no medida en el Lab 3 (Lab 5: 81 ms in-mesh) | `data/lab5/rtt_otbr.csv` | [ ] Pendiente |
| Lab 3: latencia de poll (periodo de 5 s) | < 5 s | 4,8 s | `data/lab4/sed_poll_period.csv` | PASS |
| Lab 4: tiempo en detectar un sensor perdido | ≤ 60 s | 60 s (5.03) | informe Lab 4, §2.2 | PASS |
| Lab 4: fallas de lectura con la radio ocupada | < 5 % | no medida | — | [ ] Pendiente |
| Lab 5: latencia agregada por el OTBR | < 100 ms | 109 ms (190 − 81) | `data/lab5/rtt_otbr.csv` | **FAIL** |
| Lab 5: RTT NAT64 | < 200 ms | sin dato: el hotspot no enruta NAT64/IPv6 de salida | informe Lab 5, Task E | [ ] Pendiente |

---

## 7. Ética y sostenibilidad

*   [x] **Lab 1:** PAN ID propio en el canal 15 para no interferir a otros grupos; `ot thread stop`
    al terminar cada sesión.
*   [x] **Lab 3:** estándares abiertos (CoAP, CBOR, CDDL); Observe reduce un 97 % las transmisiones.
*   [ ] **Lab 4:** default seguro sin dato del sensor: el 5.03 está implementado; el watchdog que
    cierra la válvula está **diseñado pero no implementado** en el firmware de los Labs 3–4.
*   [x] **Lab 4:** `pollperiod` de 5 s: 238 días de batería frente a 47 con 1 s (−80 % de pilas
    descartadas).
*   [x] **Lab 5:** el sistema funciona localmente sin la nube (OTBR-off test).

---

## 8. Análisis por viewpoint

| Viewpoint | Labs | Preocupaciones documentadas |
|---|---|---|
| Foundational | 1–5 | Dominios PED/SCD/ASD/RAID, cuatro redes |
| Business | — | Pendiente (Lab 8) |
| Usage | 3–5 | Daniela consulta y riega desde el celular; guía de diagnóstico para Edwin |
| Functional | 2–5 | Roles Thread → Orchestration / Transferring / Sensing; contratos CoAP; gateway |
| Trustworthiness | 4–5 | Auditoría de la sección 9 |
| Construction | 0–5 | Firmware Zephyr por lab, overlays de devicetree, flash por placa |

---

## 9. Auditoría de trustworthiness (primera pasada, Labs 4–5)

| Característica | ¿Cubierta? | Cómo | Brechas |
|---|---|---|---|
| Availability | Sí | Self-healing en 8 s; el OTBR-off test mantiene la malla; 5.03 explícito al perder el sensor | Sin redundancia de Leader ni failover del OTBR |
| Confidentiality | Parcial | Network Key de Thread | Payloads CoAP en claro; DTLS en el Lab 6 |
| Integrity | Parcial | CRC 802.15.4; plausibilidad −40..85 °C; PUT idempotente | Inyección maliciosa posible sin firmas (Lab 6) |
| Reliability | Sí | CON con backoff exponencial (4 reintentos); ResponseTimeout a ~45 s | Pérdidas > 45 s requieren reintento en la aplicación |
| Resilience | Sí | Re-parenting automático de SED; reconvergencia mesh | — |
| Safety | No | Watchdog "default closed" diseñado | No implementado en el firmware de los Labs 3–4 |
| Compliance | Parcial | RFC 7252, 7641, 8949; ISO/IEC 30141 | Validación formal con CDDL pendiente |

---

## 10. Declaración de uso de IA

| Herramienta | Qué hizo |
|---|---|
| Asistente de código (Claude) | Organizó este repositorio, transcribió las tablas de los informes a CSV y consolidó este DDR a partir de los informes del equipo. Los valores medidos provienen de los informes. |
| | |
