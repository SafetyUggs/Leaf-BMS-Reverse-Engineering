# Nissan Leaf ZE0 battery controller (LBC): diagnostic services

Firmware: **3NA0D** (Li-ion battery controller, ZE0 24 kWh pack, Renesas V850). Everything here was read from the
firmware itself (disassembly and decompiled code), not from Nissan documentation. Field names are descriptive names
chosen during the analysis; where a DBC name from the community CAN databases matches, it is given.

A companion document covers the AZE0 (4NC0A) firmware: [AZE0_4NC0A_diagnostic_services.md](AZE0_4NC0A_diagnostic_services.md).
Its last section lists the differences between the two.

**Not covered on purpose:** service 0x27 (SecurityAccess) and anything to do with reprogramming. Service 0x27 exists and
some write / routine / actuator services require it; those rows just say "security access". Seed/key handling, the
programming session and the boot loader are not documented here.

Conventions:
- All multi-byte values are **big-endian** in the responses.
- "d0" is the first data byte after the response header, e.g. in `61 01 d0 d1 d2 ...` the header is `61 01`.
- "inferred" marks a meaning that comes from how a value is used, not from a direct proof.

---

## 1. Transport and sessions

| item | value |
|---|---|
| Request CAN ID | **0x79B** (11-bit). No functional / broadcast address is accepted |
| Response CAN ID | **0x7BB**, always DLC 8, padded with 0xFF |
| Protocol | KWP2000-style services on ISO-TP (ISO 15765-2). Service 0x19 uses the UDS layout |
| Request frames | must be DLC 8; single frame length 1..7; multi-frame requests up to 156 bytes |
| Flow control sent by the LBC | `30 01 00` after the first frame and after every consecutive frame (BS = 1, STmin = 0) |
| Multi-frame responses | up to 650 bytes; the tester's flow control (BS, STmin) is honoured; at most one CF per 10 ms; STmin values ≥ 0x80 are treated as 127 ms |
| Transport timeouts | 1 s waiting for a flow control or a consecutive frame; on expiry the transfer is dropped without an NRC |
| Response time (P2) | about **945 ms**. After that `7F <sid> 23` is sent first, then `7F <sid> 21`; the tester must repeat the identical request to get the held response |
| Session timeout (S3) | about **5.1 s** after the last response (positive or negative). Then the session returns to 0x81 and any access level is lost |

### Sessions (service 0x10)

| request | response | session | conditions |
|---|---|---|---|
| `10 81` | `50 81` | default | none |
| `10 C0` | `50 C0` | manufacturer session C0 (workshop) | none to enter |
| `10 F0` | `50 F0` | manufacturer session F0 | vehicle stopped: motor torque 0 (CAN 0x1DA) and speed 0, else `7F 10 22` |
| `10 F1` | `50 F1` | manufacturer session F1 | same as F0 |
| `10 85` | (none from the application) | programming session | not documented here |

- Wrong length (≠ 2) or unknown session → `7F 10 12`. The response has no timing parameters.
- F0 / F1 drop back to 0x81 as soon as the motor torque or the vehicle speed become non-zero.
- In C0, the access level is lost (the session stays C0) when ignition is off, supply < 10 V, speed > 5 km/h or the
  VCM reports the main relays closed.

### Negative response codes

| NRC | meaning | typical cause |
|---|---|---|
| 0x11 | serviceNotSupported | unknown service |
| 0x12 | subFunctionNotSupported / invalidFormat | wrong length, unknown local ID, wrong extended session for that ID |
| 0x21 | busyRepeatRequest | response not ready (second and later), or a different request while busy |
| 0x22 | conditionsNotCorrect | 10 F0/F1 while moving, 14 with no wake input |
| 0x23 | routineNotComplete | first "response not ready" notice |
| 0x33 | securityAccessDenied | service needs security access |
| 0x7F | serviceNotSupportedInActiveSession | 30 / 3B in the wrong session; 34 always |

No `0x78` (response pending) is used.

### Services

| SID | name | notes |
|---|---|---|
| 10 | StartDiagnosticSession | §1 |
| 14 | ClearDiagnosticInformation | §2 |
| 19 | ReadDTCInformation (UDS layout) | §3 |
| 21 | ReadDataByLocalIdentifier | §4, no session or security needed |
| 27 | SecurityAccess | exists, not documented |
| 30 | InputOutputControlByLocalIdentifier | §5 |
| 31 | StartRoutineByLocalIdentifier | §6 |
| 34 | RequestDownload | always `7F 34 7F` in the application |
| 3B | WriteDataByLocalIdentifier | §7 |
| 3E | TesterPresent | §8 |
| other | | `7F <sid> 11` |

The services 21, 30, 31 and 3B answer the local IDs 00 / 20 / 40 / 60 with a 4-byte "supported IDs" bitmap in the OBD
style (byte 0 bit 7 = ID+1 ... byte 3 bit 0 = ID+0x20, which also means "next range supported"). Several working IDs
are **not** advertised in these bitmaps.

---

## 2. Service 0x14: clear DTCs

| request | response | conditions |
|---|---|---|
| `14 FF FF FF` | `54` (1 byte) | any session, no security; a wake input (ignition line or the second wake input) must be active, else `7F 14 22` |
| other group / length | `7F 14 12` | |

Clears the DTC status table, the three freeze frames and the aging / heal counters, the weak-module and
capacity-empty latches, and resets the live fault detectors so present faults are detected again. The `54` comes on
the next 10 ms pass, after the clear has run. The EEPROM copy is updated at the next normal shutdown save; if power is
lost before that, the old DTCs come back.

---

## 3. Service 0x19: read DTC information

| request | response |
|---|---|
| `19 02 MM` | `59 02 4E` + n × {DTC high, DTC middle, DTC low, status} for every DTC with `status & MM & 0x4E ≠ 0`; at most 161 DTCs |
| `19 04 D1 D2 D3 FF` | `59 04 D1 D2 D3 SS`, plus `00 84` and 132 snapshot records (§3.3) if a freeze frame exists for this DTC (total 537 bytes) |
| `19 06 D1 D2 D3 81` | `59 06 D1 D2 D3 SS`, plus `81 AA` if the DTC is confirmed (AA = aging counter, 0 = failed in the current/last cycle, +1 per clean ignition cycle) |
| anything else | `7F 19 12` (also for an unknown DTC, record number FF in 19 06, or 19 01 / 19 0A) |

No session or security check.

### 3.1 Status byte

Only bits 1, 2, 3, 6 are reported (availability mask **0x4E**):

| bit | meaning |
|---|---|
| 1 | failed in this operation cycle |
| 2 | pending (failed this or the last cycle) |
| 3 | confirmed (set on the first confirmed failure; cleared after 40 clean ignition cycles, except U-codes which never age out) |
| 6 | test not completed this cycle |

`19 02 FF` right after power-up or a clear also lists almost every DTC (bit 6), truncated to 161.

### 3.2 DTC list (206 codes)

| index | DTC | meaning (where known) |
|---|---|---|
| 1 | P0A1F | battery ECU internal (flash checksum, RAM, ADC reference, current zero ...) |
| 2 | P30FE | not identified |
| 3 | P33D6 | not identified |
| 4 | U1000 | CAN communication (never ages out) |
| 5–12 | P31A4 P3199 P31A7 P319C P318D P318E P3191 P3196 | CAN receive / transmit faults (transmit stuck; 0x1D4 / 0x1DA / 0x1F2 / 0x380 / 0x50B missing or frozen counter) |
| 13 | P3030 | cell-monitor reply error |
| 14–37 | P3031 … P3048 | cell-monitor reply error per 4-cell group 1..24 |
| 38–61 | P3049 … P3060 | measurement plausibility per group |
| 62 / 63 / 64 | P33E7 / P3062 / P3061 | not identified |
| 65–88 | P308B … P30A2 | per-group cell-monitor checks |
| 89 / 90 / 91 | P30F3 / P30F4 / P30F5 | HV voltage sensor: LIN frame error / voltage out of range / differs from the cell sum by > 36.6 V |
| 92 | P30FD | HV sensor > 124 V while disabled, or answering while its supply is off |
| 93 | P33D5 | not identified |
| 94–97 | P33D7 P33D9 P33DB P33DD | temperature sensor 1..4 ADC high / low |
| 98 / 99 | P33E4 / P33E5 | temperature sensor pair mismatch ≥ 20 °C |
| 100–102 | P33DF / P33E0 / P33E1 | not identified |
| 103 | P0A0D | HV interlock |
| 104–151 | P3302 … P3331 | per 2-cell module 1..48 (cell over / undervoltage, one bank) |
| 152 | P3300 | not identified |
| 153–200 | P3375 … P33A4 | per 2-cell module 1..48 (the other bank) |
| 201–204 | P3373 P30FC P33E2 P33D4 | not identified |
| 205 | P33E6 | cell imbalance (confirms in the second failing cycle) |
| 206 | P33ED | not identified |

Which module bank is over- and which is under-voltage was not established. The internal index is what the 0x5C0
`LB_Diagnosis_Trouble_Code` carries and what snapshot records 14 01 / 14 02 store.

### 3.3 Freeze-frame (snapshot) records of 19 04

Two freeze frames are used: the first confirmed DTC, and the most recent one (each new confirmed DTC overwrites the
second). When the first is cleared by aging, the second moves up. The data is captured once, in the 10 ms pass in
which the DTC confirms. Each record is sent as `14 xx` + data.

| record | bytes | content | units |
|---|---|---|---|
| 14 01 | 2 | internal DTC index of freeze frame 1 | |
| 14 02 | 2 | internal DTC index of freeze frame 2 | |
| 14 03 | 2 | number of the internal check that failed | |
| 14 04 | 4 | current sensor, zero-corrected, × 1000 | milli-counts, signed |
| 14 05 | 2 | learned current-sensor zero | ADC counts (633..664 valid) |
| 14 06 | 4 | battery current | mA, signed, + = charge |
| 14 08 | 2 | 2.5 V reference measured at boot | ADC counts × 10 (5120 = 2.500 V) |
| 14 09 | 4 | current-sensor zero used | ADC counts × 10⁶ |
| 14 0A | 2 | 2.5 V reference | ADC counts |
| 14 0B / 14 0C | 2 each | charge / discharge power limit | 10 W |
| 14 0D | 2 | pack voltage from the cell sum | 10 mV |
| 14 0E | 2 | 12 V supply | mV |
| 14 0F | 2 | isolation (IR) sensor wave voltage | ADC counts (0x3FF invalid) |
| 14 10 | 2 | pack resistance (regression) | mΩ |
| 14 11 | 2 | Hx (resistance health) | 0.01 % |
| 14 12 | 4 | SOC | ppm |
| 14 13 | 4 | full capacity | 0.1 mAh |
| 14 14 | 1 | HV interlock state | 0 / 1 |
| 14 15 … 14 1F, 14 21 … 14 3F, 14 41 … 14 5F, 14 61 … 14 77 | 2 each | cell voltages 1..96 (numbers x00 / x20 / x40 / x60 are skipped) | mV |
| 14 78 | 2 | pack voltage (as on 0x1DB) | 10 mV, 0xFFFF invalid |
| 14 79 | 2 | pack voltage latched at the cell scan | 10 mV |
| 14 7A | 2 | HV sensor voltage, calibrated | 97 mV per count, 0xFFFF invalid |
| 14 7B | 2 | current sensor | ADC counts |
| 14 7C | 1 | 5 V supply fault wake count | 0..40 |
| 14 7D | 2 | isolation sensor | ADC counts |
| 14 7E | 1 | isolation sensor malfunction | 0 / 1 |
| 14 7F / 14 81 | 2 / 1 | temperature sensor 1: ADC / °C | |
| 14 82 / 14 83 | 2 / 1 | temperature sensor 2 | |
| 14 84 / 14 85 | 2 / 1 | temperature sensor 3 | |
| 14 86 / 14 87 | 2 / 1 | temperature sensor 4 | |
| 14 88 | 1 | CAN supervision mode | 1 charging, 2 ignition on / not charging, 3 inactive |
| 14 89 | 1 | charge status from CAN 0x1F2 | 0 none, 1 normal, 2 quick, 3 stop |

---

## 4. Service 0x21: read data by local identifier

`21 xx` (length exactly 2). No session or security is needed for any local ID. Unknown ID or wrong length → `7F 21 12`.

| ID | response length | content |
|---|---|---|
| 00 / 20 / 40 / 60 | 6 | supported IDs bitmap: 00 → `FF 80 00 01` (01–09), 20 / 40 → `00 00 00 01`, 60 → `F0 00 00 00` (61–64). 82 upwards work but are not advertised |
| 01 | 41 | current, power limits, voltages, resistance, Hx, SOC, capacity |
| 02 | 198 | 96 cell voltages + 2 pack voltages |
| 03 | 28 | HV sensor, raw ADCs, SOC at wake, min / max cell, faults, motor data |
| 04 | 16 | 4 temperature sensors, vehicle speed |
| 05 | 71 | raw analog / isolation sensor / input snapshot |
| 06 | 26 | balancing bleed status per group |
| 07 | 10 | boot reference, current-sensor zero |
| 08 | 5 | EEPROM block status |
| 09 | 7 | weak-cell detector |
| 61 | 12 | SOH stages and capacity bars |
| 62 | 118 | charge counts and usage histograms |
| 63 | 4 | spare NV word |
| 64 | 3 | 5 V supply fault wake count |
| 82 | 82 | fault / healing record |
| 83 | 26 | identification constants |
| 84 | 22 | 16-byte ID string |
| 90 | 10 | LB code store |
| 91 | 12 | HV voltage sensor calibration |
| 92 | 98 | per-cell voltage trims |
| F0 / F1 | 26 | identification block from flash |
| FE / FF | 26 | identification block from EEPROM (or F0 / F1 if blank) |

### 21 01: measurement summary

| data | size | content | units / notes |
|---|---|---|---|
| d0–d3 | s32 | current sensor, zero-corrected, × 1000 | milli ADC counts |
| d4–d5 | u16 | learned current-sensor zero | ADC counts |
| d6–d9 | s32 | battery current | mA, + = charge; 0x80000000 invalid |
| d10–d13 | 4 | always FF FF FF FF | |
| d14–d15 | u16 | charge power limit | 10 W |
| d16–d17 | u16 | discharge power limit | 10 W |
| d18–d19 | u16 | pack voltage from the cell sum | 10 mV |
| d20–d21 | u16 | 12 V supply, reference-corrected | mV (max 24000) |
| d22–d23 | u16 | isolation sensor wave voltage (`LB_IR_Sensor_Wave_Voltage`) | ADC counts; 0x3FF invalid |
| d24–d25 | u16 | pack resistance (V/I regression) | mΩ |
| d26–d27 | u16 | Hx (resistance health) | 0.01 % (10000 = new) |
| d28–d31 | u32 | SOC (`LB_SOC`) | ppm |
| d32–d35 | u32 | full capacity | 0.1 mAh (656 000 = 65.6 Ah) |
| d36–d37 | u16 | max power for charger | 100 W; 0x8000 not set |
| d38 | bits | bit0 HV interlock OK (`LB_INTER_LOCK`); bit1 full charge (`LB_Full_CHARGE_flag`); bit2 interlock sense input, debounced | |

### 21 02: cell voltages

| data | size | content | units |
|---|---|---|---|
| d0–d191 | 96 × u16 | cell voltage 1..96 | mV; 0xFFFF invalid |
| d192–d193 | u16 | pack voltage from the HV sensor (as on 0x1DB `LB_Total_Voltage`) | 10 mV |
| d194–d195 | u16 | pack voltage latched at the cell scan | 10 mV |

### 21 03: sensors and status

| data | size | content | units / notes |
|---|---|---|---|
| d0–d1 | u16 | HV sensor voltage after calibration | 97 mV per count |
| d2–d3 | u16 | current sensor | ADC counts |
| d4–d5 | s16 | modelled cell temperature | 0.1 °C; 0x8000 invalid |
| d6–d9 | u32 | SOC at wake-up | ppm |
| d10–d11 | u16 | highest cell voltage | mV |
| d12–d13 | u16 | lowest cell voltage | mV |
| d14 | u8 | 5 V supply fault wake count | 0..40 (at 40 the 5 V sensor supply is switched off; reset with 3B 64) |
| d15–d16 | u16 | isolation sensor | ADC counts |
| d17 | u8 | relay cut request (`LB_Relay_Cut_Request`) | 0 or 3 |
| d18 | u8 | failsafe status (`LB_Failsafe_Status`) | bits |
| d19–d20 | u16 | fault summary bits | |
| d21–d22 | s16 | motor torque from CAN 0x1DA (raw / 2, ±300; Nm inferred) | 0x8000 = no frame |
| d23–d24 | s16 | motor speed from CAN 0x1DA | rpm; 0x8000 = no frame |
| d25 | bits | bit0 main-relay-on permission; bit1 VCM 0x1D4 byte5 bit6 (main relays closed, inferred); bit2 isolation sensor malfunction; bit3 VCM 0x1D4 byte4 bit2 | |

### 21 04: temperatures

| data | size | content |
|---|---|---|
| d0–d1 / d2 | u16 / s8 | temperature sensor 1: ADC / °C |
| d3–d4 / d5 | u16 / s8 | temperature sensor 2 |
| d6–d7 / d8 | u16 / s8 | temperature sensor 3 |
| d9–d10 / d11 | u16 / s8 | temperature sensor 4 |
| d12 | s8 | minimum temperature, averaged (°C; 0x80 invalid) |
| d13 | u8 | vehicle speed from the motor rpm (km/h, 0..127) |

### 21 05: raw input snapshot

| data | size | content |
|---|---|---|
| d0–d1 | u16 | 12 V supply input, ADC (supply ÷ 3.7) |
| d2–d3 | u16 | 2.5 V reference, ADC |
| d4–d5 | u16 | 5 V rail monitor, ADC (÷ 2.5; valid 4.545–5.463 V) |
| d6–d7 | u16 | HV sensor, before calibration (97 mV per count) |
| d8–d9 | u16 | HV sensor, after calibration |
| d10–d17 | 4 × u16 | temperature sensors 1..4, ADC |
| d18–d19 | u16 | current sensor, ADC |
| d20–d23 | s32 | battery current, mA |
| d24–d25 | u16 | isolation sensor wave voltage |
| d26–d45 | 10 × u16 | isolation sensor samples, drive-low phase |
| d46–d65 | 10 × u16 | isolation sensor samples, drive-high phase |
| d66 | u8 | any wake input active: 0xAA yes / 0x55 no |
| d67 | u8 | second wake input: 0xAA active / 0x55 inactive |
| d68 | bits | bit0 HV-interlock drive output; bit1 interlock sense input, raw level |

### 21 06 / 21 07 / 21 08

| ID | data | content |
|---|---|---|
| 06 | d0–d23 | balancing bleed status of each 4-cell group 1..24 |
| 07 | d0–d1 | 2.5 V reference measured at boot, ADC × 10 (5120 = 2.500 V); 0 if not measured |
| 07 | d2–d5 | current-sensor zero in use, ADC counts × 10⁶ |
| 07 | d6–d7 | 2.5 V reference now, ADC |
| 08 | d0 | EEPROM block A: 0 main copy used, 1 backup copy used, 0x80 both bad |
| 08 | d1 | EEPROM block B (DTCs): 0 ok, 0x80 bad |
| 08 | d2 | 1 = ADC reference chain not trusted (system, ADC or reference supply fault) |

### 21 09: weak-cell detector

Runs only during a normal (AC) charge after a long enough rest. It compares how much the SOC of the highest and the
lowest voltage cell moves against the average cell between two points. A ratio ≥ 116 (1.81×) marks a weak module.

| data | size | content |
|---|---|---|
| d0 | u8 | highest ratio seen (ratio × 64; 64 = same as average) |
| d1 | u8 | ratio of the highest-voltage cell |
| d2 | u8 | ratio of the lowest-voltage cell |
| d3 | u8 | weak module of the highest-voltage cell (1..48 = cell index / 2 + 1; 0xFF none) |
| d4 | u8 | weak module of the lowest-voltage cell |

After a cold start (the LBC lost its 12 V supply while parked) the rest time is 0 and the check does not arm, so the
reply is `00 00 00 FF FF`.

### 21 61: SOH

| data | size | content | units |
|---|---|---|---|
| d0–d1 | u16 | Hx (resistance health) | 0.01 % |
| d2–d3 | u16 | SOH sent on CAN, latched at wake-up | 0.01 % |
| d4 | u8 | capacity bars | 0..12 |
| d5–d6 | u16 | filtered SOH | 0.01 % |
| d7–d8 | u16 | stored SOH (used at the next wake-up) | 0.01 % |
| d9 | bits | bit0 Hx has been measured; bit1 stored SOH has been set by an estimate | |

### 21 62: usage statistics (counts)

| data | size | content |
|---|---|---|
| d0–d1 | u16 | normal charge sessions |
| d2–d3 | u16 | quick charge sessions |
| d4–d19 | 8 × u16 | max temperature at each ignition on |
| d20–d35 | 8 × u16 | max temperature at each charge start |
| d36–d51 | 8 × u16 | highest max temperature of each drive |
| d52–d67 | 8 × u16 | highest max temperature of each charge |
| d68–d83 | 8 × u16 | SOC at each ignition on |
| d84–d99 | 8 × u16 | SOC at each charge start |
| d100–d115 | 8 × u16 | SOC when full charge is detected |

Temperature bins: < 35, 35–40, 40–45, 45–50, 50–55, 55–60, 60–65, ≥ 65 °C. SOC bins: < 20, 20–30, 30–40, 40–50, 50–60,
60–70, 70–85, ≥ 85 %. Each value is a count of events, saturating at 65535. Each event is counted at most once per
power-up.

### 21 63 / 21 64

| ID | data | content |
|---|---|---|
| 63 | d0–d1 | spare word kept in EEPROM, written only by 3B 63; no other use |
| 64 | d0 | 5 V supply fault wake count (as 21 03 d14) |

### 21 82: fault / healing record

80 data bytes: d0–d1 are the constants `12 47`, d60 is 0x00, and all bytes not listed below are 0xFF. Five bytes carry a
live record: **bit7** and **bit6** are fault status bits, **bits 0–5** a heal counter (40 when the fault is present, then −1
per ignition cycle without it, stopping at 1; 0 = never failed).

| data | record |
|---|---|
| d2 | heal counter B |
| d3 | heal counter D (0xFF in one ignition / VCM mode) |
| d19 | heal counter C (0xFF in one ignition / VCM mode) |
| d51 | heal counter E |
| d79 | heal counter A (0xFF in one ignition / VCM mode) |

Which faults these records stand for was not identified for this firmware. (On AZE0 the same layout carries the
CAN communication DTCs.)

### 21 83: identification constants

| data | content |
|---|---|
| d0–d4 | `33 4E 41 30 44` = "3NA0D" |
| d5 | 0x50 ("P") |
| d6–d8 | `42 05 17` |
| d9–d13 | 00 × 5 |
| d14–d15 | `96 45` |
| d16–d22 | 00 |
| d23 | 0x80 |

### 21 84 / 21 90 / 21 91 / 21 92

| ID | data | content |
|---|---|---|
| 84 | d0–d15, d16–d19 | 16-byte ID string from EEPROM (written by 3B 84; `*` if never written), then 4 spaces |
| 90 | d0–d7 | 4 pairs of LB code bytes (3B 90); pair n is shown on 0x1DC as `LB_CODE1` / `LB_CODE2` with `LB_CODE_CONDITION` = n |
| 91 | d0–d9 | HV voltage sensor calibration, 5 × s16 in 97 mV units (3B 91) |
| 92 | d0–d95 | per-cell voltage trims, signed mV (3B 92). Each cell reading is lowered by the trim (scaled by V / 4300 mV below 4300 mV) |

### 21 F0 / F1 / FE / FF: identification blocks

| ID | content |
|---|---|
| F0 | 24 bytes from flash: "3NA0CPB" 05 17, zeros, 96 06, zeros, 80 |
| F1 | 24 bytes from flash: 5 × 00, "F0000F0000", 6 × 00, 5C C3 48 |
| FE / FF | 24 bytes from EEPROM (12 words, low byte first) if that area has been programmed, else the F0 / F1 contents. Nothing in the application writes this area. May take a few 10 ms passes |

---

## 5. Service 0x30: input/output control

Only in sessions **C0 and F0** (else `7F 30 7F`).

| request | session | response | function |
|---|---|---|---|
| `30 00/20/40/60` | C0 / F0 | `70 xx 00 00 00 00` | supported IDs (nothing advertised) |
| `30 FA 00 GG MM` | F0 + security access | `70 FA 01` | **cell bleed test**: cell-monitor group GG (1..24) bleeds the cells in mask MM (bits 0..3) for 30 s, bypassing the normal balancing inhibits (OV / UV, monitor faults, "balancing allowed"). A second start restarts the 30 s |
| `30 FA 01` | F0 + security access | `70 FA SS` | status: 00 never run, 01 running, 02 ended (timeout, session left or aborted by another request), 11 stopped by tester |
| `30 FA 11` | F0 + security access | `70 FA 11` | stop the test |

GG ≥ 0x19 or MM ≥ 0x10 → `7F 30 12`. 30 FA in C0 → `7F 30 12`. No other actuator overrides exist.

---

## 6. Service 0x31: routines

| request | session | response | function |
|---|---|---|---|
| `31 00/20/40/60` | any | `71 xx b1 b2 b3 b4` | supported routines (02 by default, 02 03 04 in C0; FA is not advertised) |
| `31 02 00` / `31 02 01` | any | `71 02 01` / `71 02 02` | end post-key-off balancing. Has no effect in this firmware (the balancing timer is never loaded) |
| `31 03 00` / `31 03 01` | C0 + security access | `71 03 01` / `71 03 02` | **reset SOH**: Hx, SOH, filtered and stored SOH to 100 %, capacity bars to 12; the estimate re-learns from the next measurement (one resistance-step event) |
| `31 04 00` / `31 04 01` | C0 + security access | `71 04 01` / `71 04 02` | **clear usage statistics** (21 62). Events already counted in this power-up are not counted again until the next one |
| `31 FA 00` / `31 FA 01` | F0 / F1 + security access | `71 FA 01` / `71 FA 01` running, `71 FA 02` done | **write the 3B 84 / 90 / 91 / 92 data to EEPROM** (F0: LB codes, HV calibration, cell trims, about 3.2 s; F1: ID string, about 0.3 s). Once per power-up; keep the session alive with 3E while it runs |

The second byte `01` only returns a status. 31 03 / 31 04 changes reach EEPROM at the next normal shutdown save; a
power loss before that loses them.

---

## 7. Service 0x3B: write data by local identifier

Not in the default session (`7F 3B 7F`). Each ID needs its own session; in another extended session → `7F 3B 12`; without
security access → `7F 3B 33`. Values are stored as received, without range checks. The layout of each write equals
the matching 21 read.

| request (length) | session | stores | EEPROM |
|---|---|---|---|
| `3B 00/20/40/60` (2) | C0 / F0 / F1 | supported IDs (C0: 61–64) | |
| `3B 61` + 10 bytes (12) | C0 | SOH stages and capacity bars, layout as 21 61 | next shutdown save |
| `3B 62` + 116 bytes (118) | C0 | charge counts and histograms, as 21 62 | next shutdown save |
| `3B 63 HH LL` (4) | C0 | spare word | next shutdown save |
| `3B 64 NN` (3) | C0 | 5 V supply fault wake count; **00 re-enables the 5 V sensor supply** after a lock-out | next shutdown save |
| `3B 84` + 16 bytes (18) | F1 | ID string | only through `31 FA 00` |
| `3B 90` + 8 bytes (10) | F0 | LB code bytes (shown on 0x1DC at once) | only through `31 FA 00` |
| `3B 91` + 5 × s16 (12) | F0 | HV voltage sensor calibration: corrections at raw 144 / 192 / 228 / 336 / 418 V, 97 mV units; a positive value lowers the reported pack voltage | only through `31 FA 00` |
| `3B 92` + 96 bytes (98) | F0 | per-cell voltage trims (signed mV) | only through `31 FA 00` |

Response `7B <ID>`. 84, 90, 91 and 92 are not advertised in the bitmaps.

Typical workshop sequences (inferred from the access rules): `10 C0` → security access → `3B 61..64` → key off (saved);
`10 F0` → security access → `3B 90 / 91 / 92` → `31 FA 00` → poll `31 FA 01` until `71 FA 02`.

---

## 8. Service 0x3E: tester present

| request | response |
|---|---|
| `3E 01` (length 2) | `7E` (one byte, no echo) |
| `3E`, `3E 00`, `3E 02`, `3E 80` | `7F 3E 12` (no suppress-response bit) |

Any answered request, positive or negative, restarts the 5.1 s session timer.
