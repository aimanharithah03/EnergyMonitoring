# ADL200 + KC868-A2v3 Energy Monitoring Pipeline

**Project**: Acrel ADL200 single-phase energy meter → KC868-A2v3 (ESP32-S3) → MQTT (EMQX) → Dashboard
**Status**: Modbus communication confirmed working; MQTT publishing and relay indicators implemented; dashboard mockup drafted.

---

## 1. Overview

This project uses an Acrel ADL200 direct-connect single-phase energy meter to monitor AC power consumption of a test load. A KC868-A2v3 board (ESP32-S3) polls the meter over RS485 Modbus RTU, publishes readings to an MQTT broker (EMQX), and drives two relay-connected indicator LEDs reflecting system/link status. A web dashboard consumes the published data for live visualization.

```
ADL200 meter --RS485 Modbus RTU--> KC868-A2v3 (ESP32-S3) --WiFi/MQTT--> EMQX broker --> Dashboard
```

---

## 2. Hardware

| Component | Detail |
|---|---|
| Energy meter | Acrel ADL200, direct-connect single-phase, 10(80)A, 230V nominal (220–264V operating range) |
| Controller | KC868-A2v3 (ESP32-S3-based; **not** pin-compatible with older KC868-A2/V2 boards) |
| Broker | EMQX (self-hosted or EMQX Cloud) |
| Pulse LED | Onboard meter LED, 1000 imp/kWh (1 pulse = 1 Wh) — visual cross-check only, not wired to the ESP32 |
| Relay outputs | K1 (system-on indicator), K2 (Modbus error blink) — drive external LEDs |

### Confirmed KC868-A2v3 GPIO pinout (ESP32-S3)

| Function | Pin |
|---|---|
| RS485 RXD | GPIO15 |
| RS485 TXD | GPIO7 |
| Relay1 (K1) | GPIO40 |
| Relay2 (K2) | GPIO39 |

> **Note:** these pins are specific to the **V3** board. The older KC868-A2/V2 (plain ESP32) uses a completely different pinout (RS485 on GPIO35/32, Relay1/2 on GPIO15/2) — always confirm board revision before reusing reference pinouts, since the same GPIO number means different things across revisions.

Source: KinCony forum thread tid=7958 (A2v3 I/O definition).

---

## 3. Mains Wiring

The ADL200 is a **direct-connect, in-series** meter — current must physically pass through its L terminal, not just be tapped in parallel.

```
Source L ──► ADL200 L-in ──► ADL200 L-out ──► Load L
Source N ──► ADL200 N-in ──► ADL200 N-out ──► Load N
Source E (ground) ─────────────────────────► Load E   (bypasses the meter entirely — ground never routes through the meter)
```

Key lessons from build iteration:
- Ground never passes through the ADL200 (it has no earth terminal). A PSU's earth terminal can be used as a shared junction/bonding point for ground distribution, as long as it's a proper terminal block — but L/N must **not** also be routed through that same junction, or the meter ends up tapped in parallel (reads voltage/frequency but zero current/power).
- Leaving the "out" side of the meter unterminated while "in" is energized creates an open circuit (no power to the load) and a shock hazard from the exposed live "out" terminal.
- Ground must always be connected — floating ground removes fault protection even though the circuit still "works" electrically without it.
- Rated current: 80A max direct connect; virtually any small PSU or household load is well within range.

### RS485 signal wiring

```
ADL200 pin 21 (A+) ──► KC868-A2v3 RS485 A
ADL200 pin 22 (B-) ──► KC868-A2v3 RS485 B
```

(Pins 17/18 on the meter are a separate pulse output, not used for Modbus.)

**Best practice**: de-energize both the meter (unplug from mains) and the ESP32 board before connecting/disconnecting RS485 A/B wiring, even though it's low-voltage signal wiring — avoids accidental contact with adjacent live L/N terminals and hot-plug transients on the bus.

---

## 4. ADL200 Modbus Register Map (used in this project)

| Address | Parameter | Length | Scale/Type |
|---|---|---|---|
| 0x000B | Voltage | 2 regs | ×0.1 V |
| 0x000C | Current | 2 regs | ×0.01 A |
| 0x000D | Active power | 2 regs | ×0.001 kW |
| 0x000E | Reactive power | 2 regs | ×0.001 kvar |
| 0x000F | Apparent power | 2 regs | ×0.001 kVA |
| 0x0010 | Power factor | 2 regs | ×0.001 |
| 0x0011 | Frequency | 2 regs | ×0.01 Hz |
| 0x0000 | Total active energy (EP) | 4 bytes (2 regs), uint32 | ×0.01 kWh |
| 0x00B0 | Total reactive energy (EQ) | 4 bytes (2 regs), uint32 | ×0.01 kvarh |

Configuration (set via meter front panel, password default `0001`): `Addr` (1–247), `bAUd` (1200–38400bps), `PAri` (parity — note communication itself runs no-parity regardless of this menu setting per the manual). Settings and the EP/EQ accumulators persist through power loss; there is no documented reset procedure for the energy accumulators (by design — they're meant to be a permanent record). Software-side "session" tracking (baseline subtraction) is the practical workaround for repeated test runs.

Energy register resolution is 0.01 kWh — visible increments depend on load power (e.g. ~10 min per 0.01kWh tick at 100W).

---

## 5. Firmware Architecture

Single ESP32-S3 sketch (Arduino framework) with these responsibilities:

1. **WiFi connection** with automatic reconnect check each loop.
2. **Modbus RTU master** (`ModbusMaster` library) polling the meter every 2 seconds:
   - One read for V/I/P/Q/S/PF/Hz (registers 0x000B–0x0011, 7 registers)
   - One read for EP (0x0000, 2 registers)
   - One read for EQ (0x00B0, 2 registers)
   - Sets an `online` flag false on any Modbus timeout/error, without zeroing last-known-good values.
3. **MQTT publish** (`PubSubClient` + `ArduinoJson`) of a combined JSON payload to `energy/adl200/data` after each poll, with automatic broker reconnect.
4. **Relay indicators**, updated every loop iteration (independent of the 2s poll interval):
   - **K1 (GPIO40)**: turned on once at boot, stays on for the session — system-running indicator.
   - **K2 (GPIO39)**: blinks at 1000ms interval only while `online == false` (active Modbus error); snaps off immediately once communication recovers.

### Published MQTT payload shape

```json
{
  "online": true,
  "voltage": 240.2,
  "current": 0.34,
  "power": 0.081,
  "reactive": 0.02,
  "apparent": 0.084,
  "pf": 0.912,
  "freq": 50.02,
  "ep_kwh": 1.94,
  "eq_kvarh": 0.31
}
```

Full sketch: `adl200_kc868_mqtt.ino` (delivered separately in this project).

---

## 6. Dashboard (mockup stage)

Planned layout: live tiles for V / I / P / PF / Hz, an accent-highlighted energy total tile (EP), a meter online/offline status indicator, and a live power-over-time trend chart. Intended to extend the existing dyno-logger dashboard architecture (WiFi STA/AP fallback, trend charts, SPIFFS CSV logging) rather than building a separate app from scratch. EQ, relay control, and CSV export are identified as likely next additions to the UI.

---

## 7. Troubleshooting Notes / Lessons Learned

- **Parallel vs. series wiring** was the main early mistake — voltage/frequency reading correctly is not sufficient proof of a correct connection; current/power must also read nonzero under load to confirm the meter is genuinely in series.
- **Board revision pin mismatches**: KC868-A2 (V1/V2) and KC868-A2v3 share a product name but not a pinout — always confirm silkscreen/revision before trusting a reference pinout.
- **Modbus comms only came up after adjusting Addr/baud/parity** on the meter to match the firmware config — worth documenting the final working combination once settled (fill in below).
- The meter's own pulse-output LED (1000 imp/kWh) is a useful independent sanity check on real accumulation, separate from the Modbus/MQTT chain.

**Working meter configuration (fill in once confirmed permanently saved on the meter):**
- Address: ___
- Baud rate: ___
- Parity: ___

---

## 8. Open Items / Next Steps

- Confirm `RELAY_ACTIVE_HIGH` polarity against actual K1/K2 board behavior.
- Build out dashboard HTML/JS (live tiles + chart) consuming the MQTT payload, likely via MQTT-over-WebSocket in-browser.
- Decide on EMQX auth/TLS approach (LAN vs. EMQX Cloud).
- Optional: forward/reverse EP/EQ registers (0x0068, 0x0072, 0x00BA, 0x00C4) if bidirectional (solar/export) monitoring becomes relevant later.
