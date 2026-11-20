# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**, updated through **Lab 5**

**Team Members:**
1. Samuel Elias Gomez Munoz (GitHub: @sgomezmunoz)
2. Juan Fernando Martinez Cabrera

Per-lab reports: [docs/informes/](informes/).

> Rule: every number in this document cites the raw file it came from in
> [`data/`](../data/). A number without a source is not evidence.

---

## 1. System Overview

*   **System Type:** [ ] Component (Lab 1-2) | [x] System (Lab 3-6) | [ ] Environment (Lab 7-8)
*   **Description:** SoilSense nodes on ESP32-C6 (Zephyr + OpenThread) forming a
    Thread mesh, exposing CoAP/CBOR resources (`/env/temp`, `/act/valve`) with a
    DHT22 sensor, joined to Wi-Fi and the Internet through an OpenThread Border
    Router (ESP32-S3 host + ESP32-C6 RCP).
*   **Hardware:** ___ × ESP32-C6-DevKitC, 1 × ESP32-S3-DevKitC (PSRAM ___), DHT22.

---

## 2. Lab Log & Stakeholder Summaries

### Lab 0: HTTP and MQTT
*   **Request/response vs. publish/subscribe:**

### Lab 1: RF Characterization
*   **To Samuel (Architect):**
*   **To Edwin (Ops):**

### Lab 2: Thread mesh
*   **To Samuel:**

### Lab 3: Thread & CoAP
*   **To Daniela (Customer):**

### Lab 4: Real Sensors
*   **To Edwin:**

### Lab 5: Border Router
*   **To Daniela:**

---

## 3. Architecture Decision Records (ADRs)

**ADR-001: 802.15.4 channel and node spacing (Lab 1)**
*   **Context:** 2.4 GHz is shared with Wi-Fi; the 16 802.15.4 channels overlap the 20 MHz Wi-Fi channels. Passive scan with `ot scan energy 500`, TX power 0 dBm.
*   **Decision:** Channel 15 (2425 MHz). Node spacing ≤ 10 m.
*   **Rationale (measurements):** noise floor −105 dBm on ch 15 vs −64 dBm on ch 26 (`data/lab1/scan_energy.csv`). PER 0.0 % up to 5 m, 1.0 % at 10 m (RSSI −65/−67 dBm), 3.5–4.0 % at 20 m, 10–12 % at 30 m (`data/lab1/rssi_per_vs_distancia.csv`). The PER ≤ 1 % threshold holds only up to **10 m**, so the course target (> 20 m) is **not met** indoors; spacing is set by that measurement.
*   **Alternatives considered:** ch 26 (noisiest, −64 dBm); a spacing of 20 m (PER 3.5–4 %, rejected).
*   **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

**ADR-002: Mesh topology (Lab 2)**
*   **Context:**
*   **Decision:**
*   **Rationale (measurements):** RTT 1 hop ___ ms, 2 hops ___ ms; healing time ___ s. Source: `data/lab2/...`
*   **Alternatives considered:**
*   **Status:** [ ] Proposed | [ ] Accepted | [ ] Deprecated

**ADR-003: CoAP confirmable vs. non-confirmable (Lab 3)**
*   **Context:**
*   **Decision:**
*   **Rationale (measurements):** CON latency ___ ms vs NON ___ ms. Source: `data/lab3/...`
*   **Status:** [ ] Proposed | [ ] Accepted | [ ] Deprecated

**ADR-004: Observe vs. polling, and the poll period (Lab 3)**
*   **Context:**
*   **Decision:**
*   **Rationale (measurements):** poll latency (5 s period) ___ s; Observe ___. Source: `data/lab3/...`
*   **Status:** [ ] Proposed | [ ] Accepted | [ ] Deprecated

**ADR-005: Border Router architecture (Lab 5)**
*   **Context:**
*   **Decision:** OTBR on ESP32-S3 (Wi-Fi host) + ESP32-C6 as RCP over UART; IPv6 forwarding Thread ↔ Wi-Fi, OMR prefix, NAT64/DNS64.
*   **Rationale (measurements):** in-mesh RTT median 81 ms (avg 79.5); laptop → OTBR → node median 190 ms (avg 221.0, max 469), 0 % loss over 10 packets (`data/lab5/rtt_otbr.csv`). Latency added by the OTBR = **+109 ms**, which **misses the < 100 ms target** by 9 ms; the cost is header translation, forwarding on the S3 and the 802.15.4 handoff. Accepted anyway because the end-to-end valve command (~300 ms) stays far inside the < 5 s requirement from Lab 4, and the OTBR-off test passed (the mesh keeps working without Wi-Fi).
*   **Alternatives considered:** Thread node directly on Wi-Fi (rejected: unstable Wi-Fi + Thread on one C6, mixes planes).
*   **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

**ADR-T01: Flash size and USB console per board (team, Labs 2–5)**
*   **Context:** The course firmware did not match the flash of our boards, and the default UART console ___.
*   **Decision:** Force `ESPTOOLPY_FLASHSIZE_4MB` on the C6 and `_16MB` on the S3; console and shell over `usb_serial`.
*   **Rationale (evidence):** build/flash error without the change: ___ (log in `data/lab3/...`).
*   **Status:** [x] Accepted

---

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification |
|-----------|------------|---------------|
| ESP32-C6 node + DHT22 | PED | |
| Thread mesh | SCD | |
| ESP32-S3 Border Router | | |

### Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---------------------|-------------|-------------------|---------------|----------------|
| | | `CONFIG_WIFI` | | 0 |
| | | `CONFIG_HTTP_SERVER` | | 0 |
| | | `CONFIG_JSON_LIBRARY` | | 0 |
| | | `CONFIG_LED_STRIP` | | 0 |

---

## 5. First Principles Reflections

**Lab 1:**
1.

**Lab 2:**
1.

**Lab 3:**
1.

**Lab 4:**
1.

**Lab 5:**
1.

---

## 6. Performance Baselines

| Metric | Target | Measured | Source | Status |
|--------|--------|----------|--------|--------|
| Lab 1: Max Range (PER ≤ 1 %) | > 20m | 10 m | `data/lab1/rssi_per_vs_distancia.csv` | **FAIL** |
| Lab 2: Healing Time | < 120s | ___ s | `data/lab2/` | [ ] Pass |
| Lab 3: CoAP Latency | < 200ms| ___ ms | `data/lab3/` | [ ] Pass |
| Lab 3: Poll Latency (5 s period) | < 5s | ___ s | `data/lab3/` | [ ] Pass |
| Lab 4: Time to see a lost sensor | ≤ 60s | ___ s | `data/lab4/` | [ ] Pass |
| Lab 4: Read failures, radio busy | < 5% | ___ % | `data/lab4/` | [ ] Pass |
| Lab 5: OTBR added latency | < 100ms | 109 ms (median 190 − 81) | `data/lab5/rtt_otbr.csv` | **FAIL** |
| Lab 5: NAT64 RTT | < 200ms | not measured | — | [ ] Pending |

---

## 7. Ethics & Sustainability Checklist

*   [ ] **Lab 1:** Verified interference doesn't disrupt neighbors.
*   [ ] **Lab 4:** Safe default when sensor data is missing (Safety).
*   [ ] **Lab 5:** System works locally without cloud (Sustainability).

---

## 8. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|-----------|----------------|-------------------------|
| Foundational |            |                         |
| Business |               |                         |
| Usage |                  |                         |
| Functional |             |                         |
| Trustworthiness | 4      |                         |
| Construction |           |                         |

---

## 9. Trustworthiness Audit (Lab 4, first pass)

| Characteristic | Addressed? | How | Gaps |
|----------------|------------|-----|------|
| Availability |            |     |      |
| Confidentiality |         |     |      |
| Integrity |               |     |      |
| Reliability |             |     |      |
| Resilience |              |     |      |
| Safety |                  |     |      |
| Compliance |              |     |      |

---

## 10. AI tool declaration

| Tool | What it did |
|------|-------------|
| | |
