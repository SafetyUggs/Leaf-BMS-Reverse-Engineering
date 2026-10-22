# Nissan Leaf AZE0 (24 kWh) battery controller (LBC): diagnostic services

Firmware: **4NC0A** (Li-ion battery controller, AZE0 24 kWh pack, Renesas V850). Everything here was read from the
firmware itself (disassembly and decompiled code), not from Nissan documentation. Names of fields are descriptive
names chosen during the analysis; where a DBC name from the community CAN databases matches, it is given.

A companion document covers the ZE0 (3NA0D) firmware: [ZE0_3NA0D_diagnostic_services.md](ZE0_3NA0D_diagnostic_services.md).
Most of the protocol is the same; the differences are listed at the end of this file.

**Not covered on purpose:** service 0x27 (SecurityAccess) and anything to do with reprogramming. Service 0x27 exists and
some write / routine / actuator services require it; those rows just say "security access". Seed/key handling, the
programming session and the boot loader are not documented here.

Conventions:
- All multi-byte values are **big-endian** in the responses.
- "d0" is the first data byte after the response header, e.g. in `61 01 d0 d1 d2 ...` the header is `61 01`.
- "inferred" marks a meaning that comes from how a value is used, not from a direct proof.
- Invalid / not-available values are given where the firmware uses one (usually 0xFF, 0xFFFF, 0x80000000).

---

## 1. Transport and sessions

| item | value |
|---|---|
| Request CAN ID | **0x79B** (11-bit). No functional / broadcast address is accepted |
| Response CAN ID | **0x7BB**, always DLC 8, padded with 0xFF |
| Protocol | KWP2000-style services on ISO-TP (ISO 15765-2). Service 0x19 uses the UDS layout |
| Request frames | must be DLC 8; single frame length 1..7; multi-frame requests up to 156 bytes |
| Flow control sent by the LBC | `30 01 00` after the first frame and after every consecutive frame (BS = 1, STmin = 0) |
| Multi-frame responses | up to 650 bytes; the tester's flow control (BS, STmin) is honoured; at most one CF per 10 ms |
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

- Wrong length (≠ 2) or unknown session → `7F 10 12`.
- F0 / F1 drop back to 0x81 as soon as the motor torque or the vehicle speed become non-zero.
- In C0, the access level is lost (the session stays C0) when ignition is off, supply < 10 V or speed > 5 km/h.

### Negative response codes

| NRC | meaning | typical cause |
|---|---|---|
| 0x11 | serviceNotSupported | unknown service |
| 0x12 | subFunctionNotSupported / invalidFormat | wrong length, unknown local ID, wrong extended session for that ID |
| 0x21 | busyRepeatRequest | response not ready (second and later), or a different request while busy |
| 0x22 | conditionsNotCorrect | 10 F0/F1 while moving, 14 with no wake input, 30 10 heater test with a low cell |
| 0x23 | routineNotComplete | first "response not ready" notice |
| 0x33 | securityAccessDenied | service needs security access |
| 0x7F | serviceNotSupportedInActiveSession | 30 / 3B in the wrong session; 34 always |

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

Clears the DTC status table, both freeze frames, the aging counters and the communication-DTC aging counters (21 82),
and resets the live fault detectors so present faults are detected again. The EEPROM copy is updated at the next
normal shutdown save; if power is lost before that, the old DTCs come back.

---

## 3. Service 0x19: read DTC information

| request | response |
|---|---|
| `19 02 MM` | `59 02 4E` + n × {DTC high, DTC middle, DTC low, status} for every DTC with `status & MM & 0x4E ≠ 0`; at most 149 DTCs |
| `19 04 D1 D2 D3 FF` | `59 04 D1 D2 D3 SS`, plus `00 92` and 146 snapshot records (§3.3) if a freeze frame exists for this DTC (total 592 bytes) |
| `19 06 D1 D2 D3 81` | `59 06 D1 D2 D3 SS`, plus `81 AA` if the DTC is confirmed (AA = aging counter) |
| anything else | `7F 19 12` (also for an unknown DTC or record number) |

No session or security check. 19 01, 19 0A and other UDS sub-functions are not implemented.

### 3.1 Status byte

Only bits 1, 2, 3, 6 are reported (availability mask **0x4E**):

| bit | meaning |
|---|---|
| 1 | failed in this operation cycle |
| 2 | pending (failed this or the last cycle) |
| 3 | confirmed (set on the first confirmed failure; cleared after 40 clean ignition cycles, except U-codes which never age out) |
| 6 | test not completed this cycle |

Typical masks: `19 02 08` confirmed, `19 02 04` pending, `19 02 0C` both.

### 3.2 DTC list (206 codes)

| codes | count | meaning (where known) |
|---|---|---|
| P0A1F | 1 | battery ECU internal (flash checksum, RAM, ADC reference, current sensor zero ...) |
| P33D6 | 1 | not identified |
| U1000 | 1 | CAN communication (never ages out) |
| P31A4 P3199 P30E0 P31A7 P319C P30E3 P30E4 P30E2 P318D P318E P3191 P3196 P3198 | 13 | CAN receive / transmit faults (missing frames, frozen counters, CRC) of the individual messages |
| P3030 | 1 | cell-monitor communication |
| P3031 … P3048 | 24 | cell-monitor reply error, one per 4-cell group (groups 1..24) |
| P3049 … P3060 | 24 | cell measurement plausibility, one per group |
| P3062 P3061 | 2 | not identified |
| P308B … P30A2 | 24 | per-group cell-monitor checks |
| P33D5 P33D7 P33D9 P33EB P33DD P33DF P33E0 P33E1 | 8 | temperature sensors (high / low / mismatch) and related analog checks (individual mapping not identified) |
| P0A0D | 1 | HV interlock |
| P3302 … P3331 | 48 | per 2-cell module 1..48, first bank of the cell OV / UV checks |
| P3375 … P33A4 | 48 | per 2-cell module 1..48, second bank |
| P30FC P33E2 P33D4 P33E6 P33ED P33EA P33EE P3064 U100C U100D | 10 | P33E6 = cell imbalance (two-trip); others not identified |

The group families are the same as in the ZE0 firmware. Which module bank is over- and which is under-voltage was not
established. The internal DTC index (position in this list, starting at 1) is what the 0x5C0 CAN frame's
`LB_Diagnosis_Trouble_Code` carries, and what snapshot records 14 01 / 14 02 store.

### 3.3 Freeze-frame (snapshot) records of 19 04

The LBC keeps **two** freeze frames: the first confirmed DTC, and the most recent one (each new confirmed DTC
overwrites the second). When the first is cleared by aging, the second moves up. The data is captured once, in the
10 ms pass in which the DTC confirms. Each record is sent as `14 xx` + data.

| record | bytes | content | units |
|---|---|---|---|
| 14 01 | 2 | internal DTC index of this freeze frame | |
| 14 02 | 2 | internal DTC index of the second freeze frame | |
| 14 03 | 2 | number of the internal check that failed | |
| 14 04 | 4 | current sensor, zero-corrected, ×1000 | milli-counts, signed |
| 14 05 | 2 | learned current-sensor zero | ADC counts |
| 14 06 | 4 | battery current | mA, signed, + = charge |
| 14 09 | 4 | current-sensor zero used | ADC counts × 10⁶ |
| 14 0A | 2 | sensor supply reference | ADC counts (512 = 2.5 V) |
| 14 0B | 2 | charge power limit | 10 W |
| 14 0C | 2 | discharge power limit | 10 W |
| 14 0D | 2 | pack voltage from the cell sum | 10 mV |
| 14 0E | 2 | 12 V supply | mV |
| 14 0F | 2 | isolation (IR) sensor wave voltage | ADC counts (0x3FF invalid) |
| 14 10 | 2 | pack resistance (regression) | mΩ |
| 14 11 | 2 | Hx (resistance health) | 0.01 % |
| 14 12 | 4 | SOC | ppm (1 000 000 = 100 %) |
| 14 13 | 4 | full capacity | 0.1 mAh |
| 14 14 | 1 | HV interlock state | 0 / 1 |
| 14 15 … 14 1F, 14 21 … 14 3F, 14 41 … 14 5F, 14 61 … 14 77 | 2 each | cell voltages 1..96 (numbers x00 / x20 / x40 / x60 are skipped) | mV |
| 14 78 | 2 | pack voltage (as on 0x1DB) | 10 mV |
| 14 79 | 2 | inverter voltage from CAN 0x1DA | 10 mV |
| 14 7B | 2 | current sensor | ADC counts |
| 14 7C | 1 | 5 V supply fault wake count | count 0..40 |
| 14 7D | 2 | isolation sensor | ADC counts |
| 14 7E | 1 | isolation sensor malfunction | 0 / 1 |
| 14 7F / 14 81 | 2 / 1 | temperature sensor 1: ADC / °C | counts / °C signed |
| 14 82 / 14 83 | 2 / 1 | temperature sensor 2: ADC / °C | |
| 14 86 / 14 87 | 2 / 1 | temperature sensor 3: ADC / °C | |
| 14 88 | 1 | CAN supervision mode | 1 charging, 2 ignition on / not charging, 3 inactive |
| 14 89 | 1 | charge status from CAN 0x1F2 | 0 none, 1 normal, 2 quick, 3 stop |
| 14 8A / 14 8B | 1 each | weak-cell ratio of the max / min voltage cell (see 21 09) | ratio × 64 |
| 14 8C / 14 8D | 1 each | weak module of the max / min voltage cell (see 21 09) | module 1..48, 0xFF none |
| 14 8E | 2 | capacity derate at full | Q15 (32768 = 1.0) |
| 14 8F | 2 | energy fraction at full | Q15 |
| 14 90 | 2 | usable SOC | Q15 |
| 14 91 | 1 | battery heater on | 0 / 1 |
| 14 92 | 1 | digital input (port bit, probably heater relay feedback, inferred) | 0 / 1 |
| 14 93 | 4 | per-group result bits (24 bits) of a cell-monitor check | bit mask |
| 14 B3 | 2 | max power for charger | 100 W (as on 0x1DC) |
| 14 B4 | 4 | predicted current (current at the last cell scan) | mA, signed |
| 14 B5 | 2 | filtered minimum temperature | °C × 200 |
| 14 B6 | 2 | cell resistance used by the overvoltage limits | 10 µΩ |
| 14 B7 / 14 B8 / 14 B9 | 2 each | cell overvoltage limits A / B / C (see 21 0C) | mV |
| 14 BA | 2 | on-board charger power from CAN 0x390 | 100 W |

---

## 4. Service 0x21: read data by local identifier

`21 xx` (length exactly 2). No session or security is needed for any local ID. Unknown ID or wrong length → `7F 21 12`.

| ID | response length | content |
|---|---|---|
| 00 / 20 / 40 / 60 | 6 | supported IDs bitmap: 00 → `FF E1 00 01` (01–0B, 10), 20 / 40 → `00 00 00 01`, 60 → `F0 00 00 00` (61–64). 0C, 65, 66, 70 and 82 upwards work but are not advertised |
| 01 | 41 | current, power limits, voltages, resistance, Hx, SOC, capacity |
| 02 | 198 | 96 cell voltages |
| 03 | 28 | sensors, SOC at wake, min / max cell, faults, motor data |
| 04 | 16 | temperature sensors |
| 05 | 71 | raw analog / isolation sensor / input snapshot |
| 06 | 26 | cell-monitor bleed status per group |
| 07 | 10 | current-sensor zero and sensor reference |
| 08 | 5 | EEPROM block status |
| 09 | 7 | weak-cell detector |
| 0A | 8 | capacity derate, energy fraction, usable SOC |
| 0B | 49 | mirror of the cyclic CAN signals |
| 0C | 21 | cell overvoltage limits |
| 10 | 5 | battery heater outputs |
| 61 | 12 | SOH stages |
| 62 | 118 | charge counts and usage histograms |
| 63 | 4 | spare NV word |
| 64 | 3 | 5 V supply fault wake count |
| 65 | 166 | time-at-temperature/SOC history |
| 66 | 25 | overvoltage-limit statistics |
| 70 | 3 | usage-history recording disabled flag |
| 82 | 82 | communication DTC records |
| 83 | 26 | identification constants |
| 84 | 22 | 16-byte ID string |
| 90 | 10 | LB code store |
| 92 | 98 | per-cell voltage trims |
| 94 | 3 | balancing status bit |
| 95 | 4 | cell-monitor fault bits |
| F0 / F1 | 26 | identification block from flash |
| FE / FF | 26 | identification block from EEPROM (or F0 / F1 if blank) |

### 21 01: measurement summary

| data | size | content | units / notes |
|---|---|---|---|
| d0–d3 | s32 | current sensor, zero-corrected, × 1000 | milli ADC counts |
| d4–d5 | u16 | learned current-sensor zero | ADC counts; 0x3FF invalid |
| d6–d9 | s32 | battery current | mA, + = charge; 0x80000000 invalid |
| d10–d13 | 4 | always FF FF FF FF | |
| d14–d15 | u16 | charge power limit | 10 W (0x1DC `LB_Charge_Power_Limit` = this / 25) |
| d16–d17 | u16 | discharge power limit | 10 W |
| d18–d19 | u16 | pack voltage from the cell sum | 10 mV; 0xFFFF if any cell invalid |
| d20–d21 | u16 | 12 V supply, reference-corrected | mV |
| d22–d23 | u16 | isolation sensor wave voltage (`LB_IR_Sensor_Wave_Voltage`) | ADC counts; 0x3FF invalid |
| d24–d25 | u16 | pack resistance (V/I regression) | mΩ |
| d26–d27 | u16 | Hx (resistance health) | 0.01 % (10000 = new) |
| d28–d31 | u32 | SOC (`LB_SOC`) | ppm |
| d32–d35 | u32 | full capacity | 0.1 mAh (654 000 = 65.4 Ah) |
| d36–d37 | u16 | max power for charger | 100 W; 0x8000 not set |
| d38 | bits | bit0 HV interlock OK (`LB_INTER_LOCK`); bit1 full charge (`LB_Full_CHARGE_flag`); bit2 interlock sense input, debounced | |

### 21 02: cell voltages

| data | size | content | units |
|---|---|---|---|
| d0–d191 | 96 × u16 | cell voltage 1..96 | mV; 0xFFFF invalid |
| d192–d193 | u16 | pack voltage (as on 0x1DB `LB_Total_Voltage`) | 10 mV |
| d194–d195 | u16 | inverter voltage from CAN 0x1DA | 10 mV (2 V steps); 0xFFFF invalid |

### 21 03: sensors and status

| data | size | content | units / notes |
|---|---|---|---|
| d0–d1 | 2 | always FF FF | |
| d2–d3 | u16 | current sensor | ADC counts (10 bit) |
| d4–d5 | s16 | modelled cell temperature (thermal model from the max temperature and I² heating) | 0.1 °C; 0x8000 invalid |
| d6–d9 | u32 | SOC at wake-up | ppm |
| d10–d11 | u16 | highest cell voltage | mV |
| d12–d13 | u16 | lowest cell voltage | mV |
| d14 | u8 | 5 V supply fault wake count | wakes with a confirmed 5 V rail fault, 0..40 (at 40 the 5 V sensor supply is switched off; reset with 3B 64) |
| d15–d16 | u16 | isolation sensor | ADC counts |
| d17 | u8 | relay cut request (`LB_Relay_Cut_Request`) | 0 or 3 |
| d18 | u8 | failsafe status (`LB_Failsafe_Status`) | bits |
| d19–d20 | u16 | fault summary bits | 16 fault groups |
| d21–d22 | s16 | motor torque from CAN 0x1DA | Nm (0x8000 = no frame) |
| d23–d24 | s16 | motor speed from CAN 0x1DA | rpm (0x8000 = no frame) |
| d25 | bits | bit0 main-relay-on permission (`LB_MainRelayOn_flag`); bit1 main relays closed (from VCM 0x1D4); bit2 isolation sensor malfunction (`LB_IR_Sensor_Malfunction`); bit3 HV-on request (from VCM 0x1D4; meaning inferred from its use) | |

### 21 04: temperatures

| data | size | content | units |
|---|---|---|---|
| d0–d1 / d2 | u16 / s8 | temperature sensor 1: ADC / °C | counts / °C (−40..105) |
| d3–d4 / d5 | u16 / s8 | temperature sensor 2 | |
| d6–d8 | 3 | always FF (AZE0 has three sensors) | |
| d9–d10 / d11 | u16 / s8 | temperature sensor 3 | |
| d12 | s8 | minimum temperature, averaged | °C; 0x80 invalid |
| d13 | u8 | vehicle speed (motor rpm × 9577 / 661500) | km/h, 0..127 |

### 21 05: raw input snapshot

| data | size | content |
|---|---|---|
| d0–d1 | u16 | 12 V supply input, ADC |
| d2–d3 | u16 | sensor supply reference, ADC (nominal 512 = 2.5 V) |
| d4–d5 | u16 | 5 V rail monitor, ADC (12.2 mV per count; valid 4.54–5.46 V) |
| d6–d9 | 4 | always FF |
| d10–d11 / d12–d13 | u16 | temperature sensors 1 / 2, ADC |
| d14–d15 | 2 | always FF FF |
| d16–d17 | u16 | temperature sensor 3, ADC |
| d18–d19 | u16 | current sensor, ADC |
| d20–d23 | s32 | battery current, mA |
| d24–d25 | u16 | isolation sensor wave voltage, ADC |
| d26–d45 | 10 × u16 | isolation sensor samples, drive-low phase (0xFFFF = blocked) |
| d46–d65 | 10 × u16 | isolation sensor samples, drive-high phase |
| d66 | u8 | any wake input active: 0xAA yes / 0x55 no |
| d67 | u8 | second wake input: 0xAA active / 0x55 inactive |
| d68 | bits | bit0 HV-interlock drive output (square wave); bit1 interlock sense input, raw level |

### 21 06: bleed status

| data | size | content |
|---|---|---|
| d0–d23 | 24 × u8 | bleed (balancing) status reported by each 4-cell monitor group 1..24 |

### 21 07: current-sensor zero

| data | size | content | units |
|---|---|---|---|
| d0–d1 | 2 | always FF FF | |
| d2–d5 | u32 | current-sensor zero in use | ADC counts × 10⁶ (default 649 000 000) |
| d6–d7 | u16 | sensor supply reference | ADC counts |

### 21 08: EEPROM / measurement status

| data | size | content |
|---|---|---|
| d0 | u8 | EEPROM block A (capacity, SOH, statistics): 0 main copy used, 1 backup copy used, 0x80 both bad |
| d1 | u8 | EEPROM block B (DTCs): 0 ok, 0x80 bad |
| d2 | u8 | 1 = measurement chain not trusted (system, ADC or reference supply fault) |

### 21 09: weak-cell detector

The detector compares how much the SOC of the highest and the lowest voltage cell moves against the average cell
between two points during charging. A cell that moves 1.81 times faster (ratio ≥ 116) has less capacity than the others.

| data | size | content | units |
|---|---|---|---|
| d0 | u8 | highest ratio seen | ratio × 64 (64 = same as average) |
| d1 | u8 | ratio of the highest-voltage cell | ratio × 64 |
| d2 | u8 | ratio of the lowest-voltage cell | ratio × 64 |
| d3 | u8 | weak module (highest-voltage cell) | module 1..48 (cell index / 2 + 1); 0xFF none |
| d4 | u8 | weak module (lowest-voltage cell) | module 1..48; 0xFF none |

### 21 0A: energy fractions

| data | size | content | units |
|---|---|---|---|
| d0–d1 | u16 | capacity derate at full | Q15 |
| d2–d3 | u16 | energy fraction at full | Q15 |
| d4–d5 | u16 | usable SOC (`LB_Usable_SOC` on 0x1DB = this × 100 / 32768) | Q15 |

### 21 0B: CAN signal mirror

| data | content |
|---|---|
| d0 | `LB_Discharge_Power_Status` (1 normal, 2 high-rate limit, 3 immediate limit) |
| d1 | `LB_Charge_Power_Status` (same coding) |
| d2 | `LB_BPCMAX_UPRATE` |
| d3 | `LB_CODE_CONDITION` |
| d4 / d5 | `LB_CODE1` / `LB_CODE2` |
| d6 | `LB_ALU_ANSWER` |
| d7 | `LB_RefusetoSleep` (1 refuse, 2 ready to sleep) |
| d8–d11 | remaining energy in 0.01 Wh, quantised to 80 Wh (÷ 8000 = gids on 0x5BC) |
| d12 | `LB_Remaining_Capacity_Segments` (capacity or remaining bars, alternating) |
| d13 | `LB_Temperature_Segment_For_Dash` (gauge segment × 20) |
| d14 | `LB_Output_Power_Limit_Reason` |
| d15 | `LB_Remain_Charge_Time_Condition` |
| d16–d17 | `LB_Remain_Charge_Time` (minutes) |
| d18 | 0x5C0 history mux (`LB_Historical_Data_Swich_Flag`: 0 none, 1 max, 2 avg, 3 min) |
| d19 | `LB_HistData_HighLow_Volt_Tim` |
| d20 | `LB_HistData_Temp_WakeupPhase` (°C, signed) |
| d21 | `LB_HistData_Temperature` (°C, signed) |
| d22 | `LB_HistData_IntegratedCurrent` (0.6 Ah, signed) |
| d23 | `LB_HistData_Degr_IntRes_Coeff` (%) |
| d24 | `LB_HistData_Cell_Voltage` |
| d25 | `LB_NextWakeupTimeForBatterHeater` (0 when cold .. 29 when warm) |
| d26 | `LB_Diagnosis_Trouble_Code` (alternates between two stored DTCs) |
| d27–d28 | full energy for QC (`LB_Full_Capacity_for_QC`, 0.1 kWh) |
| d29–d30 | remaining energy for QC (`LB_Remain_Capacity_for_QC`, 0.1 kWh) |
| d31 / d32 | last message counter (PRUN) / CRC received in VCM frame 0x1D4 |
| d33 | VCM network state from 0x50B (0 inactive, 3 active) |
| d34 / d35 / d36 | 0x50C: message counter / ALU question / CRC |
| d37 | charge status from 0x1F2 (0 none, 1 normal, 2 quick, 3 stop) |
| d38 | 0x1F2 message counter |
| d39 | on-board charger mode from 0x390 |
| d40–d41 | on-board charger power from 0x390 (100 W) |
| d42 / d43 | 0x1DA message counter / CRC |
| d44 | ambient temperature from 0x54C (°C, signed; 0x80 invalid) |
| d45 | bit0 `LB_Voltage_Latch_Flag`; bit1 `LB_Capacity_Empty`; bit2 `LB_Remain_Cap_Segment_Swit_Flag`; bit3 / bit4 / bit5 heater start / stop / mail send request (0x5C0); bit6 VCM bit from 0x50B byte2 bit2; bit7 VCM bit from 0x50B byte6 bit5 |
| d46 | bit0 charge-target mode (0x1F2 byte0 bit7); bit1 0x1F2 byte2 bit4; bit2 / bit3 charger states from 0x390 (byte4 bit6 / byte5 bit7) |

### 21 0C: cell overvoltage limits

| data | size | content | units |
|---|---|---|---|
| d0 | u8 | overvoltage limit level: 0 none, 1 max cell ≥ limit B, 2 max cell ≥ limit A, 3 cell overvoltage fault | |
| d1–d4 | s32 | predicted current (current at the last cell scan) | mA |
| d5–d6 | s16 | filtered minimum temperature | °C × 200 |
| d7–d8 | u16 | cell resistance used for the limits (table by SOH and temperature) | 10 µΩ |
| d9–d10 | u16 | limit A = full-charge voltage at the predicted current − charge resistance × 0.06 | mV |
| d11–d12 | u16 | limit B = limit A − resistance × predicted current (lower limit) | mV |
| d13–d14 | u16 | limit C = cell polarisation at the predicted current + resistance × 0.13 | mV |
| d15–d18 | u32 | time the car was off before this wake-up | s |

### 21 10: battery heater

| data | size | content |
|---|---|---|
| d0–d1 | 2 | always FF FF |
| d2 | bits | bit1 heater ON (relay output); bit2 a digital input reported next to it (probably the heater relay feedback, inferred) |

### 21 61: SOH

| data | size | content | units |
|---|---|---|---|
| d0–d1 | u16 | Hx (resistance health) | 0.01 % |
| d2–d3 | u16 | SOH sent on CAN (`LB_Capacity_Deterioration_Rate` × 100), latched at wake-up | 0.01 % |
| d4 | 1 | always FF | |
| d5–d6 | u16 | filtered SOH ((stored + 255 × old) / 256) | 0.01 % |
| d7–d8 | u16 | stored SOH (result of the Hx or capacity estimate, used at the next wake-up) | 0.01 % |
| d9 | bits | bit0 Hx has been measured; bit1 SOH was updated this drive | |

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
60–70, 70–85, ≥ 85 %. Each value is a count of events, saturating at 65535.

### 21 63 / 21 64

| ID | data | content |
|---|---|---|
| 63 | d0–d1 | spare word kept in EEPROM, written only by 3B 63; no other use |
| 64 | d0 | 5 V supply fault wake count (as 21 03 d14) |

### 21 65: time history (minutes)

41 × u32, each saturating:

| data | content |
|---|---|
| d0–d119 | 30 counters: minutes per temperature bin (10) × SOC bin (3), index = temperature bin × 3 + SOC bin. Parked time is added at each wake-up, awake time at each shutdown. Temperature = average of the max temperature at both ends of the period, bin edges 0 / 11 / 20 / 27 / 33 / 38 / 42 / 46 / 50 °C. SOC = average, bin edges 45 / 85 % |
| d120–d155 | 9 counters: awake minutes per temperature bin, edges 0 / 12 / 22 / 30 / 37 / 42 / 46 / 50 °C |
| d156–d159 | total minutes (parked + awake) |
| d160–d163 | total discharge while not charging, A·s |

### 21 66: overvoltage-limit statistics

Counts are 20 ms steps unless noted.

| data | size | content |
|---|---|---|
| d0–d1 | u16 | time with the max cell ≥ limit A |
| d2–d5 | u32 | time with limit B ≤ max cell < limit A |
| d6 | u8 | longest run between limit B and limit A |
| d7 | u8 | drive cycles that reached level 3 |
| d8 | u8 | drive cycles that reached level 2 |
| d9–d10 | u16 | drive cycles that reached level 1 |
| d11–d14 | u32 | time with the max cell within 1/3 of limit C below limit B |
| d15–d18 | u32 | time in the next 1/3 band |
| d19–d22 | u32 | time in the third band |

### 21 70

| data | content |
|---|---|
| d0 | 1 = recording of the 21 65 time history is disabled (set with 3B 70) |

### 21 82: communication DTC records

80 data bytes. Six of them carry a record, all others are 0:

| data | fault |
|---|---|
| d2 | CAN bus-off |
| d3 | LBC transmit failure (one of 0x1DB 0x55B 0x1DC 0x5BC 0x5C0 0x59E 0x45D not sent) |
| d11 | A/C amplifier frame 0x54C lost |
| d19 | inverter frame 0x1DA lost |
| d51 | VCM frames lost (0x1D4, 0x1F2, 0x50B, 0x50C) |
| d79 | on-board charger frame 0x390 lost |

Each record byte: **bit7** failing now, **bit6** confirmed, **bits 0–5** aging counter (40 while the fault is
confirmed, then −1 per ignition cycle without it, stopping at 1). d3, d11, d19 and d79 read 0xFF while the ignition input
is on but the VCM status bit in 0x50B (byte 2 bit 2) is not set. "Lost" means no valid frame for 3 checks (now), confirmed when a leaky
counter (+3 per missing check, −1 per good one) reaches 200.

### 21 83: identification constants

| data | content (stock 4NC0A) |
|---|---|
| d0–d4 | application part number `34 4E 43 30 41` = "4NC0A" |
| d5 | 0x55 |
| d6–d8 | `42 05 17` |
| d9–d13 | 00 × 5 |
| d14–d17 | `02 13 13 21` |
| d18–d19 | first two bytes of the calibration area (00 00) |
| d20–d22 | 00 00 00 |
| d23 | 0x80 |

d0–d8 equal the identification string in the boot area ("4NC0AUB" 05 17).

### 21 84 / 21 90 / 21 92

| ID | data | content |
|---|---|---|
| 84 | d0–d15, d16–d19 | 16-byte ID string from EEPROM (written by 3B 84), then 4 spaces |
| 90 | d0–d7 | 4 pairs of LB code bytes (3B 90); pair n is shown on 0x1DC as `LB_CODE1` / `LB_CODE2` with `LB_CODE_CONDITION` = n |
| 92 | d0–d95 | per-cell voltage trims, signed mV (3B 92). Each cell reading is lowered by the trim (scaled by V / 4150 mV below 4150 mV) |

### 21 94 / 21 95

| ID | data | content |
|---|---|---|
| 94 | d0 | 1 once every cell-monitor group has reported its bleed status bit clear (balancing complete) |
| 95 | d0–d1 | cell-monitor fault-confirmed bits (u16) |

### 21 F0 / F1 / FE / FF: identification blocks

| ID | content |
|---|---|
| F0 | 24 bytes from the boot area: "4NC0AUB" 05 17, zeros, 02 13 13 21, zeros, 80 (the same as 21 83) |
| F1 | 24 bytes from the boot area: 5 × 00, "F0000F0000", 6 × 00, 5C 4B 17 |
| FE / FF | 24 bytes from EEPROM (12 words) if that area has been programmed, else the F0 / F1 contents. May take a few 10 ms passes |

---

## 5. Service 0x30: input/output control

Only in sessions **C0, F0, F1** (else `7F 30 7F`).

| request | session | response | function |
|---|---|---|---|
| `30 00/20/40/60` | C0 / F0 / F1 | `70 xx b1 b2 b3 b4` | supported IDs (C0 advertises 10) |
| `30 FA 00 GG MM` | F0 + security access | `70 FA 01` | **cell bleed test**: cell-monitor group GG (1..24) bleeds the cells in mask MM (bits 0..3) for 30 s, bypassing the normal balancing inhibits |
| `30 FA 01` | F0 + security access | `70 FA SS` | bleed test status: 00 never run, 01 running, 02 ended (timeout, session left or aborted), 11 stopped by tester |
| `30 FA 11` | F0 + security access | `70 FA 11` | stop the bleed test |
| `30 FB 00 MM` | F0 / F1 + security access | `70 FB 01` | **battery heater override** for 30 s: MM 01 = heater on, 00 = heater off |
| `30 FB 01` / `30 FB 11` | F0 / F1 + security access | `70 FB SS` / `70 FB 11` | status (01 running, 02 ended, 11 returned) / return control |
| `30 10 20 MM` | C0 + security access | `70 10 01` | battery heater override without a timer (MM 01 on / 00 off); refused with `7F 30 22` while the lowest cell is below 3306 mV |
| `30 10 01` / `30 10 11` | C0 + security access | `70 10 SS` / `70 10 11` | status (01 running, 02 ended, 11 returned, 12 stopped by low cell voltage) / return control |

All overrides end when the session that started them is left. GG ≥ 0x19 or MM ≥ 0x10 → `7F 30 12`.

---

## 6. Service 0x31: routines

| request | session | response | function |
|---|---|---|---|
| `31 00/20/40/60` | any | `71 xx b1 b2 b3 b4` | supported routines (02 by default, 02 03 04 in C0; FA is not advertised) |
| `31 02 00` / `31 02 01` | any | `71 02 01` / `71 02 02` | end post-key-off balancing. Has no effect in this firmware (the balancing timer is never loaded) |
| `31 03 00` / `31 03 01` | C0 + security access | `71 03 01` / `71 03 02` | **reset SOH**: Hx, SOH and filtered / stored SOH set to 100 %; the estimate re-learns from the next measurement |
| `31 04 00` / `31 04 01` | C0 + security access | `71 04 01` / `71 04 02` | **clear usage statistics** (21 62 counts and histograms) |
| `31 FA 00` / `31 FA 01` | F0 / F1 + security access | `71 FA 01` / `71 FA 01` running, `71 FA 02` done | **write the 3B 84 / 90 / 92 data to EEPROM** (F0: LB codes and cell trims; F1: ID string). Once per power-up |

The second byte `01` only returns a status; it does not run the routine. 31 03 / 31 04 changes reach EEPROM at the
next normal shutdown save.

---

## 7. Service 0x3B: write data by local identifier

Not in the default session (`7F 3B 7F`). Each ID needs its own session; in another extended session → `7F 3B 12`; without
security access → `7F 3B 33`. Values are stored as received, without range checks. The layout of each write equals
the matching 21 read.

| request (length) | session | stores | EEPROM |
|---|---|---|---|
| `3B 00/20/40/60` (2) | C0 / F0 / F1 | supported IDs (C0: 61–64) | |
| `3B 61` + 10 bytes (12) | C0 | SOH stages, layout as 21 61 (byte at d4 ignored) | next shutdown save |
| `3B 62` + 116 bytes (118) | C0 | charge counts and histograms, as 21 62 | next shutdown save |
| `3B 63 HH LL` (4) | C0 | spare word | next shutdown save |
| `3B 64 NN` (3) | C0 | 5 V supply fault wake count; **00 re-enables the 5 V sensor supply** after a lock-out | next shutdown save |
| `3B 65` + 164 bytes (166) | C0 | time history, as 21 65 | next shutdown save |
| `3B 66` + 23 bytes (25) | C0 | overvoltage-limit statistics, as 21 66 | next shutdown save |
| `3B 70 NN` (3) | C0 | bit0 = 1 disables the 21 65 recording | |
| `3B 84` + 16 bytes (18) | F1 | ID string | only through `31 FA 00` |
| `3B 90` + 8 bytes (10) | F0 | LB code bytes (shown on 0x1DC at once) | only through `31 FA 00` |
| `3B 92` + 96 bytes (98) | F0 | per-cell voltage trims (signed mV) | only through `31 FA 00` |

Response `7B <ID>`. 65, 66, 70, 84, 90 and 92 are not advertised in the bitmaps.

---

## 8. Service 0x3E: tester present

| request | response |
|---|---|
| `3E 01` (length 2) | `7E` (one byte, no echo) |
| `3E`, `3E 00`, `3E 02`, `3E 80` | `7F 3E 12` (no suppress-response bit) |

Any answered request, positive or negative, restarts the 5.1 s session timer.

---

## 9. Differences from the ZE0 (3NA0D) firmware

| item | ZE0 3NA0D | AZE0 4NC0A |
|---|---|---|
| pack voltage | HV voltage sensor (LIN) with a 5-point calibration (21 91 / 3B 91) | no HV sensor: pack voltage = cell sum; 21 91 / 3B 91 removed; 21 02 d194 = inverter voltage from 0x1DA |
| temperature sensors | 4 | 3 (unused slots read FF) |
| new 21 IDs | | 0A, 0B, 0C, 10, 65, 66, 70, 94, 95 |
| new 3B IDs | | 65, 66, 70 |
| service 30 | FA bleed test only, sessions C0 / F0 | also F1; new 30 FB / 30 10 battery heater override |
| 31 03 (reset SOH) | also sets capacity bars to 12 | no bars value |
| 21 82 | 5 fault records, other bytes constant | 6 communication DTC records, other bytes 0 |
| 19 02 | up to 161 DTCs | up to 149 DTCs |
| 19 04 freeze frame | 132 records (537 bytes), 3 slots (2 used) | 146 records (592 bytes), 2 slots; adds weak-cell, energy, heater, OV-limit and charger records |
| DTC list | 206 codes | 206 codes, same families, different order; 11 codes differ |
