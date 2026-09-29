# ADL400 3-Phase Power Meter ↔ KC868-A2v3 Integration

Project documentation covering hardware wiring, meter configuration, the Modbus register map as actually confirmed on this unit's firmware, and the Arduino firmware (Serial + MQTT).

---

## 1. Overview

- **Meter**: Acrel ADL400, 3-phase 4-wire (3P4L), **CT-operated** current variant (external split-core CTs, not direct-connect).
- **Controller**: KinCony KC868-A2v3 (ESP32-S3), communicating over RS485 Modbus RTU.
- **Goal**: pull real-time voltage, current, power, power factor, frequency, and energy data from the meter and publish it via MQTT (EMQX public broker) in addition to Serial Monitor output.
- **Key finding**: this specific ADL400 unit's firmware deviates from Acrel's published manual in several places (setting ranges, which register banks are populated, and register resolution). All addresses/scales below are **confirmed by direct testing on this unit**, not assumed from the manual.

---

## 2. Physical Wiring

### 2.1 Voltage sensing
- Ua, Ub, Uc connected to L1, L2, L3 respectively, via small fused breakers tapping the main distribution busbars (not a direct hard-wire to the bus) — visible in the panel as the breakers feeding red/yellow/blue leads into the meter's voltage terminals.
- N connected to neutral.

### 2.2 Current sensing (CT-operated)
- Three split-core CTs clamp the phase conductors downstream of the main MCCB, ahead of two dedicated small breakers used for isolation.
- CT polarity mark (S1/*) faces the source (incoming supply) on all three phases.
- CT secondary leads (5A rated) run to the meter's Ia\*/Ia, Ib\*/Ib, Ic\*/Ic terminals — torque spec 1.5–2 N·m.
- **Safety**: CT secondaries must never be open-circuited while the primary conductor is energized — short the leads together first if disconnecting.

### 2.3 RS485 connection
- ADL400 485A/485B → KC868-A2v3 RS485 terminal block.
- KC868-A2v3 RS485 pins (confirmed via Kincony's official pin list): **TX = GPIO7, RX = GPIO15**.
- Twisted pair, common ground reference recommended between meter and board.

---

## 3. Meter Configuration (Setup Menu)

Accessed via long-press SET → password `0001` (default).

| Setting | Meaning | Value used | Note |
|---|---|---|---|
| `Addr` | Modbus slave address | 1 | |
| `bAUd` | Baud rate | 9600 | |
| `Pari` | Parity | None | |
| `PL` | Wiring mode | 3P4L | 3-phase 4-wire |
| `Ct` | CT ratio (primary ÷ secondary) | set per installed CT | e.g. 40 for a 200:5 CT |
| `Dir` | Current direction | Forward | confirmed correct — EP+ (imported) accumulates, EP- (exported) stays 0, as expected for a load with no generation source |
| `Led` | Backlight timeout | — | **Firmware deviation**: manual documents range 0–255; this unit's actual range is **0–999**, and the value is entered at address/position `060` in the menu sequence |

**Other confirmed firmware deviations from the published manual** (worth remembering if extending this project):
- The floating-point register bank (`0x5300` onward) is **not populated** on this unit — reads return valid, well-formed responses but all-zero data.
- The 16-bit power/PF register block (`0x0067`–`0x0076`) documented in the manual is also **not populated** — same all-zero pattern.
- The working power table (`0x0164` onward) uses resolution **0.001**, not the 0.0001 stated in the manual.

---

## 4. Modbus Register Map (as confirmed working)

All reads use function code `03H` (Read Holding Registers).

### 4.1 Voltage / Current / Frequency — block read `0x0061`, quantity 23
16-bit values, per the published manual (this block matched documentation).

| Parameter | Register | Format | Scale |
|---|---|---|---|
| Voltage A/B/C | 0x0061 / 0x0062 / 0x0063 | uint16 | ÷10 → V |
| Current A/B/C | 0x0064 / 0x0065 / 0x0066 | uint16 | ÷100 → A |
| Frequency | 0x0077 | uint16 | ÷100 → Hz |

*(Registers 0x0067–0x0076 in this same range — per-phase/total power and PF — are documented but return all-zero on this unit; not used.)*

### 4.2 Power (P/Q/S) — block read `0x0164`, quantity 24
32-bit **signed** values (full two's complement across both registers), confirmed by cross-checking that per-phase values sum exactly to the total registers.

| Parameter | Register | Scale |
|---|---|---|
| Active power A/B/C | 0x0164 / 0x0166 / 0x0168 | ×0.001 → kW |
| Active power total | 0x016A | ×0.001 → kW |
| Reactive power A/B/C | 0x016C / 0x016E / 0x0170 | ×0.001 → kVAr |
| Reactive power total | 0x0172 | ×0.001 → kVAr |
| Apparent power A/B/C | 0x0174 / 0x0176 / 0x0178 | ×0.001 → kVA |
| Apparent power total | 0x017A | ×0.001 → kVA |

Power factor is **not read directly** (no populated register) — derived as `PF = P_total / S_total`.

### 4.3 Energy
32-bit values, scale confirmed against the meter's own LCD (`×0.01`).

| Parameter | Register |
|---|---|
| Total accumulated active energy (Ep) | 0x0000 |
| Imported (forward) active energy | 0x000A |
| Exported (reverse) active energy | 0x0014 *(documented, not yet read in firmware)* |
| Total accumulated reactive energy (Eq) | 0x001E |

**Note**: the imported-energy scale (`0.01`) was confirmed by matching against the LCD. The total Ep/Eq scale is assumed to be the same (same register table) but **should be spot-checked** against the meter's "Total Ep"/"Total Eq" LCD screens.

---

## 5. Troubleshooting Journey (summary)

1. Initial attempt used the **ModbusMaster** library targeting the floating-point register bank (`0x5300`) → consistent `0xE2` (response timeout) errors.
2. Confirmed RS485 pins (GPIO7/15) against Kincony's official forum documentation — correct, not the cause.
3. Tried adjusting stop bits (`SERIAL_8N1` → `SERIAL_8N2`) per strict Modbus RTU spec — no change.
4. Built a raw Modbus frame diagnostic sketch (bypassing the library entirely) → got a **clean, valid response** with all-zero data from the float bank, revealing the library wasn't the issue — the register bank itself was unpopulated on this firmware.
5. Switched to the documented **integer register bank** (`0x0061`) for voltage → got real, correct readings (229–240V range). Confirmed comms, pins, and framing were fine all along.
6. Applied the same approach to power: the documented 16-bit power block (`0x0067`) also returned all-zero. Tested the **alternate 32-bit power table** (`0x0164`) → real data, confirmed via internal consistency (per-phase sums matching totals).
7. Discovered the manual's stated power resolution (0.0001) was wrong — actual resolution is **0.001**, derived by cross-checking against known voltage/current/apparent-power relationships.
8. Final sketch validated end-to-end against the meter's own LCD readings for voltage, current, power, PF, and imported energy.

**Lesson for future extension of this project**: this firmware revision does not reliably match Acrel's published manual. Any new register added should be verified with the raw diagnostic approach (send request, inspect raw hex bytes) before trusting the documented address/scale.

---

## 6. Firmware

Two sketches were produced:

### 6.1 `ADL400_KC868_A2v3.ino` — Serial Monitor only
Polls the meter every 2 seconds over raw Modbus RTU frames (no external library) and prints voltage, current, frequency, power, PF, and imported energy to Serial.

### 6.2 `ADL400_KC868_A2v3_MQTT.ino` — Serial + MQTT
Adds WiFi (2.4GHz only — ESP32-S3 does not support 5GHz) and MQTT publishing via the **PubSubClient** library.

- **Broker**: `broker.emqx.io` (EMQX public sandbox, port 1883, no authentication)
- **Client ID**: `MinD-ADL400-3Phase-PowerMeter`
- **Topic**: `MinD/EnergyMonitoring/Data/3P`
- **Publish interval**: 5 seconds (throttled back from the 2s Serial-only polling rate, out of consideration for shared public-broker infrastructure)
- **Payload**: single JSON object per publish:
  ```json
  {
    "va": 229.4, "vb": 238.1, "vc": 238.5,
    "ia": 8.51,  "ib": 10.53, "ic": 10.93,
    "freq": 50.00,
    "p_kw": 6.994, "q_kvar": 0.107, "s_kva": 7.084, "pf": 0.987,
    "energy_import_kwh": 90.700,
    "ep_total_kwh": 0.0, "eq_total_kvarh": 0.0
  }
  ```

**Known limitation**: `broker.emqx.io` is a public, unauthenticated broker — anyone who knows or guesses the topic name can subscribe. Acceptable for testing; migrate to EMQX Cloud (or self-hosted) with real credentials and TLS before treating this as production/private.

---

## 7. Open Items / Still to Verify

- [ ] Confirm `ep_total_kwh` and `eq_total_kvarh` scale factor against the meter's own Total Ep/Eq LCD screens (currently assumed same `0.01` scale as imported energy, not yet independently verified).
- [ ] Exported (reverse) active energy (`0x0014`) documented but not yet wired into firmware — add if net-metering/export tracking becomes relevant later.
- [ ] `STA` LED meaning on the meter's shell was never conclusively confirmed (RUN, COM, and E were reasoned out from standard conventions; STA remains a best guess).
- [ ] Move MQTT off the public sandbox broker to an authenticated/private broker before relying on this for anything sensitive.
