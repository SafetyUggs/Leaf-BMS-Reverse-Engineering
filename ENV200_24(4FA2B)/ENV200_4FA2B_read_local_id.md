# Nissan e-NV200 (24 kWh) battery controller (LBC): service 0x21 read data by local identifier

Firmware: **4FA2B** (Li-ion battery controller, e-NV200 24 kWh pack, Renesas V850). Everything here was read from the
firmware itself, not from Nissan documentation. Field names are descriptive names chosen during the analysis; where a
CAN signal name from the community DBCs matches, it is given.

Companion documents: [ENV200_4FA2B_cyclic_CAN_frames.md](ENV200_4FA2B_cyclic_CAN_frames.md) and, for the Leaf AZE0,
`AZE0_4NC0A_diagnostic_services.md` (transport, sessions and the other services work the same way there). The
differences from the AZE0 are listed at the end of this file.

**Not covered on purpose:** security access, reprogramming and the boot loader.

Conventions:
- All multi-byte values are **big-endian** in the responses.
- "d0" is the first data byte after the response header, e.g. in `61 01 d0 d1 d2 ...` the header is `61 01`.
- "response length" counts the whole response including `61 xx`.
- "always FF" marks filler bytes the firmware writes as constants.
- "inferred" marks a meaning that comes from how a value is used, not from a direct proof.

---

## Service 0x21

`21 xx` (length exactly 2). No session or security is needed for any local ID. Unknown ID or wrong length → `7F 21 12`.

| ID | response length | content |
|---|---|---|
| 00 / 20 / 40 / 60 | 6 | supported IDs bitmap: 00 → `FF F1 00 01` (01–0C, 10), 20 / 40 → `00 00 00 01`, 60 → `F0 00 00 00` (61–64). 82 upwards work but are not advertised |
| 01 | 41 | current, power limits, voltages, resistance, Hx, SOC, capacity |
| 02 | 198 | 96 cell voltages, pack and inverter voltage |
| 03 | 28 | sensors, SOC at wake, min / max cell, faults, motor data |
| 04 | 37 | four pack temperature sensors, auxiliary sensor, filtered temperatures |
| 05 | 71 | raw analog / isolation sensor / input snapshot |
| 06 | 26 | cell-monitor bleed status per group |
| 07 | 10 | current-sensor zero |
| 08 | 5 | EEPROM block status |
| 09 | 7 | weak-cell detector |
| 0A | 6 | timer value and a sampled input |
| 0B | 89 | mirror of the cyclic CAN signals and received CAN values |
| 0C | 31 | thermal (cooling) logic state, temperature-delta check |
| 10 | 5 | thermal output and its feedback input |
| 61 | 12 | SOH stages |
| 62 | 118 | charge counts and usage histograms |
| 63 | 4 | spare NV word |
| 64 | 3 | 5 V supply fault wake count |
| 82 | 82 | communication DTC records |
| 83 | 26 | identification constants |
| 84 | 22 | 16-byte ID string |
| 90 | 10 | LB code store |
| 91 | 12 | five diagnostic calibration words |
| 92 | 98 | per-cell voltage trims |
| 94 | 3 | balancing status bit |
| 95 | 4 | cell-monitor fault bits |
| F0 / F1 | 26 | identification block from flash |
| FE / FF | 26 | identification block from EEPROM (or F0 / F1 if blank) |

The bitmap uses the OBD style: byte 0 bit 7 = ID+1 … byte 3 bit 0 = ID+0x20, which also means "next range supported".

### 21 01: measurement summary

| data | size | content | units / notes |
|---|---|---|---|
| d0–d3 | s32 | current sensor, zero-corrected, × 1000 | milli ADC counts |
| d4–d5 | u16 | learned current-sensor zero | ADC counts |
| d6–d9 | s32 | battery current | mA |
| d10–d13 | 4 | always FF FF FF FF | |
| d14–d15 | u16 | charge power limit | 10 W (0x1DC `LB_Charge_Power_Limit` = this / 25) |
| d16–d17 | u16 | discharge power limit | 10 W |
| d18–d19 | u16 | pack voltage from the cell sum | 10 mV |
| d20–d21 | u16 | 12 V supply, reference-corrected | mV |
| d22–d23 | u16 | isolation sensor wave voltage (`LB_IR_Sensor_Wave_Voltage`) | ADC counts; 0x3FF invalid |
| d24–d25 | u16 | pack resistance (regression) | mΩ |
| d26–d27 | u16 | Hx (resistance health) | 0.01 % |
| d28–d31 | u32 | SOC (`LB_SOC`) | ppm (1 000 000 = 100 %) |
| d32–d35 | u32 | full capacity | 0.1 mAh |
| d36–d37 | u16 | max power for charger | 100 W |
| d38 | bits | bit0 HV interlock OK (`LB_INTER_LOCK`); bit1 full charge (`LB_Full_CHARGE_flag`); bit2 a debounced digital input (interlock sense, inferred) | |

### 21 02: cell voltages

| data | size | content | units |
|---|---|---|---|
| d0–d191 | 96 × u16 | cell voltage 1..96 (cell n at d(2n − 2)) | mV |
| d192–d193 | u16 | pack voltage (as on 0x1DB `LB_Total_Voltage`) | 10 mV |
| d194–d195 | u16 | inverter voltage from CAN 0x1DA | 10 mV; 0xFFFF invalid |

### 21 03: sensors and status

| data | size | content | units / notes |
|---|---|---|---|
| d0–d1 | 2 | always FF FF | |
| d2–d3 | u16 | current sensor | ADC counts (10 bit) |
| d4–d5 | 2 | always FF FF | |
| d6–d9 | u32 | SOC at initialisation | ppm |
| d10–d11 | u16 | highest cell voltage | mV |
| d12–d13 | u16 | lowest cell voltage | mV |
| d14 | u8 | 5 V supply fault wake count | count |
| d15–d16 | u16 | isolation sensor | ADC counts (10 bit) |
| d17 | u8 | relay cut request (`LB_Relay_Cut_Request`) | 0 or 3 |
| d18 | u8 | failsafe status (`LB_Failsafe_Status`) | bits |
| d19–d20 | u16 | fault summary bits | bit field |
| d21–d22 | s16 | motor torque from CAN 0x1DA | Nm, clamp ±274; 0x8000 invalid |
| d23–d24 | s16 | motor speed from CAN 0x1DA | rpm, clamp ±16383; 0x8000 invalid |
| d25 | bits | bit0 main-relay-on permission (`LB_MainRelayOn_flag`); bit1 VCM status bit 1; bit2 isolation sensor malfunction (`LB_IR_Sensor_Malfunction`); bit3 VCM status bit 0 | |

### 21 04: temperatures

ADC values are 10 bit; pack sensor temperatures are signed °C from the NTC curve, clamped −40..105 °C.

| data | size | content | units |
|---|---|---|---|
| d0–d1 / d2 | u16 / s8 | temperature sensor 1: ADC / °C | |
| d3–d4 / d5 | u16 / s8 | temperature sensor 2 | |
| d6–d7 / d8 | u16 / s8 | temperature sensor 3 | |
| d9–d10 / d11 | u16 / s8 | temperature sensor 4 | |
| d12 | s8 | minimum pack temperature, averaged | °C |
| d13 | u8 | vehicle speed (from the 0x1DA motor speed) | km/h, 0..127 |
| d14–d15 | s16 | filtered minimum temperature | °C × 200 |
| d16–d19 | s32 | filter state of d14 | °C × 204 800 |
| d20–d21 | s16 | filtered maximum temperature | °C × 200 |
| d22–d25 | s32 | filter state of d20 | °C × 204 800 |
| d26 | s8 | maximum pack temperature, averaged | °C |
| d27–d28 | 2 | always FF FF | |
| d29–d30 | u16 | auxiliary temperature sensor, ADC | counts |
| d31 | s8 | auxiliary temperature, 9-sample average (sent on 0x59E) | °C, −30..53; 0x80 invalid |
| d32–d34 | 3 | always FF | |

### 21 05: raw input snapshot

| data | size | content |
|---|---|---|
| d0–d1 / d2–d3 / d4–d5 | 3 × u16 | three raw analog inputs, ADC (d0–d1 is used by the supply voltage calculation) |
| d6–d9 | 4 | always FF |
| d10–d17 | 4 × u16 | temperature sensors 1..4, ADC |
| d18–d19 | u16 | current sensor, ADC |
| d20–d23 | s32 | battery current, mA |
| d24–d25 | u16 | isolation sensor wave voltage, ADC |
| d26–d45 | 10 × u16 | isolation sensor samples, first phase |
| d46–d65 | 10 × u16 | isolation sensor samples, second phase |
| d66 | u8 | any wake input active: 0xAA / 0x55 |
| d67 | u8 | second wake input, debounced: 0xAA / 0x55 |
| d68 | bits | bit0 HV-interlock drive output; bit1 a digital input (interlock sense, inferred) |

### 21 06: bleed status

| data | size | content |
|---|---|---|
| d0–d23 | 24 × u8 | bleed (balancing) status reported by each 4-cell monitor group 1..24 |

### 21 07: current-sensor zero

| data | size | content | units |
|---|---|---|---|
| d0–d1 | 2 | always FF FF | |
| d2–d5 | u32 | current-sensor zero in use | ADC counts × 10⁶ |
| d6–d7 | u16 | raw analog input (as 21 05 d2–d3) | ADC counts |

### 21 08: EEPROM / measurement status

| data | size | content |
|---|---|---|
| d0 | u8 | which copy of EEPROM block A was loaded |
| d1 | u8 | EEPROM block B invalid flag |
| d2 | u8 | 1 = measurement chain not trusted |

### 21 09: weak-cell detector

| data | size | content | units |
|---|---|---|---|
| d0 | u8 | highest ratio seen | ratio × 64 (64 = same as average) |
| d1 | u8 | ratio of the highest-voltage cell | ratio × 64 |
| d2 | u8 | ratio of the lowest-voltage cell | ratio × 64 |
| d3 | u8 | weak module (highest-voltage cell) | module number; 0xFF none |
| d4 | u8 | weak module (lowest-voltage cell) | module number; 0xFF none |

### 21 0A: timer value and sampled input

| data | size | content |
|---|---|---|
| d0 | 1 | always FF |
| d1 | u8 | 0, 40 or 100, depending on the thermal state, the 0x1F2 charge request and the VCM status, with a 150 s timer (21 0C d27–d28) |
| d2 | u8 | mean of the last 10 valid samples of d3 (0 until 10 are collected) |
| d3 | u8 | latest sample: an event count per 10 cycles × 10, clamp 255 (what is counted is not identified) |

### 21 0B: CAN signal mirror

| data | content |
|---|---|
| d0 | `LB_Discharge_Power_Status` |
| d1 | `LB_Charge_Power_Status` |
| d2 | `LB_BPCMAX_UPRATE` |
| d3 | `LB_CODE_CONDITION` |
| d4 / d5 | `LB_CODE1` / `LB_CODE2` |
| d6 | `LB_ALU_ANSWER` |
| d7 | `LB_RefusetoSleep` |
| d8–d11 | remaining energy in 0.01 Wh (÷ 8000 = gids on 0x5BC) |
| d12 | `LB_Remaining_Capacity_Segments` |
| d13 | `LB_Temperature_Segment_For_Dash` |
| d14 | `LB_Output_Power_Limit_Reason` |
| d15 | `LB_Remain_Charge_Time_Condition` |
| d16–d17 | `LB_Remain_Charge_Time` (minutes) |
| d18 | 0x5C0 history mux (`LB_Historical_Data_Swich_Flag`) |
| d19 | `LB_HistData_HighLow_Volt_Tim` |
| d20 | `LB_HistData_Temp_WakeupPhase` (°C, signed) |
| d21 | `LB_HistData_Temperature` (°C, signed) |
| d22 | `LB_HistData_IntegratedCurrent` (signed) |
| d23 | `LB_HistData_Degr_IntRes_Coeff` (%) |
| d24 | `LB_HistData_Cell_Voltage` |
| d25 | 0x5C0 byte 6 thermal request level (30 or 0xFF) |
| d26 | `LB_Diagnosis_Trouble_Code` |
| d27–d28 | full energy for QC (`LB_Full_Capacity_for_QC`, 0.1 kWh) |
| d29–d30 | remaining energy for QC (`LB_Remain_Capacity_for_QC`, 0.1 kWh) |
| d31 / d32 | message counter / CRC received in VCM frame 0x1D4 |
| d33 | VCM network state from 0x50B |
| d34 / d35 / d36 | 0x50C: message counter / ALU question / CRC |
| d37 | charge status from 0x1F2 |
| d38 | 0x1F2 message counter |
| d39 | on-board charger mode from 0x390 |
| d40–d41 | on-board charger power from 0x390 (100 W) |
| d42 / d43 | 0x1DA message counter / CRC |
| d44 | ambient temperature from 0x54C (°C, signed) |
| d45 | bit0 `LB_Voltage_Latch_Flag`; bit1 `LB_Capacity_Empty`; bit2 `LB_Remain_Cap_Segment_Swit_Flag`; bit3 / bit4 / bit5 the 0x5C0 heater request bits (always 0); bit6 / bit7 VCM status bits 2 / 7 |
| d46 | bit0 / bit1 / bit2 / bit3 VCM status bits 3 / 6 / 5 / 4; bit4 `LB_Full_CHARGE_flag`; bit5 VCM status bit 0; bit6 `LB_IR_Sensor_Malfunction`; bit7 VCM status bit 1 |
| d47 | bit0 charge session active; bit1 `LB_MainRelayOn_flag`; bit2 `LB_INTER_LOCK` |
| d48–d49 | usable SOC, Q15 (32768 = 100 %) |
| d50–d51 | max power for charger (100 W) |
| d52–d55 | SOC (ppm) |
| d56–d59 | battery current (mA, signed) |
| d60–d61 | always FF FF |
| d62 | 0x59E cooling request status (0 or 3) |
| d63 | `LB_Relay_Cut_Request` |
| d64–d65 | `LB_IR_Sensor_Wave_Voltage` |
| d66–d67 | SOH (0.01 %) |
| d68 | `LB_Failsafe_Status` |
| d69–d70 | charge power limit (10 W) |
| d71–d72 | discharge power limit (10 W) |
| d73–d74 | motor speed from 0x1DA (rpm, signed) |
| d75 | vehicle speed (km/h) |
| d76–d77 | motor torque from 0x1DA (Nm, signed) |
| d78–d79 | pack voltage (10 mV) |
| d80–d81 | inverter voltage from 0x1DA (10 mV) |
| d82 | 0x59E 2-bit status (0 or 3) |
| d83–d84 | energy fraction at full, Q15 |
| d85–d86 | capacity derate at full, Q15 |

### 21 0C: thermal logic and temperature-delta check

The thermal logic drives the 0x5C0 / 0x59E cooling request (see the cyclic-frames document). Several fields are only
partly identified.

| data | size | content |
|---|---|---|
| d0 | bits | bit0 remaining energy below 5.0 kWh (off at 5.5 kWh); bit1 thermal logic initialised; bit2 / bit3 cell OCV state while charging (≥ 3.0 V / charging seen); bit4 battery hot while charging; bit5 / bit6 / bit7 charge-state bits (cell OCV 2.6 / 3.0 V while charging, remaining energy 5.0 / 5.5 kWh otherwise) |
| d1 | bits | bit0 battery cold (min temperature below −16 °C, off at −10 °C); bit1 / bit2 cold-request bits; bit3 / bit4 status bits (not identified); bit6 / bit7 minimum-temperature fault bits |
| d2 | bits | bit0 / bit1 status bits (not identified); bit2 thermal output command and its feedback input disagree; bit3 supply voltage fault; bit4 a status bit (not identified) |
| d3 | u8 | diagnostic output override active (service 0x30 input/output control), bit field |
| d4 | u8 | output state requested by that override |
| d5 | u8 | cooling request source: 0 none, 1 automatic, 2 diagnostic override |
| d6 | u8 | cooling mode: 0 or 3 |
| d7–d10 | u32 | hot-charge request timer (passes; the request ends at 180 000) |
| d11 | u8 | heater drive source (always 0, heater logic not called) |
| d12 | u8 | heater drive level (always 0) |
| d13–d14 | u16 | cell open-circuit voltage at the present SOC, mV |
| d15 | u8 | not identified |
| d16–d17 | 2 | always FF FF |
| d18–d19 | s16 | minimum temperature snapshot, °C × 200 |
| d20–d23 | u32 | temperature-delta check counter (the check runs when it reaches 252 001) |
| d24–d25 | u16 | temperature-delta check value |
| d26 | u8 | temperature-delta fault count |
| d27–d28 | u16 | 10 ms timer (15 000 = 150 s) used by 21 0A d1 |

### 21 10: thermal output

| data | size | content |
|---|---|---|
| d0–d1 | 2 | always FF FF |
| d2 | bits | bit1 thermal output (relay) command: never set by this firmware; bit2 the digital input reported next to it (probably the relay feedback, inferred) |

### 21 61: SOH

| data | size | content | units |
|---|---|---|---|
| d0–d1 | u16 | Hx (resistance health) | 0.01 % |
| d2–d3 | u16 | SOH sent on CAN (`LB_Capacity_Deterioration_Rate` × 100) | 0.01 % |
| d4 | u8 | capacity bars | bars × 20 |
| d5–d6 | u16 | filtered SOH | 0.01 % |
| d7–d8 | u16 | stored SOH | 0.01 % |
| d9 | bits | bit0 / bit1 SOH-estimate status bits | |

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
60–70, 70–85, ≥ 85 %. Each value is a count of events, saturating at 65535. Cleared by routine 31 04, written by 3B 62.

### 21 63 / 21 64

| ID | data | content |
|---|---|---|
| 63 | d0–d1 | spare word kept in EEPROM, written by 3B 63 |
| 64 | d0 | 5 V supply fault wake count (as 21 03 d14) |

### 21 82: communication DTC records

80 data bytes. Six of them carry a record:

| data | fault |
|---|---|
| d2 | CAN bus-off |
| d3 | LBC transmit failure |
| d11 | A/C amplifier frame 0x54C lost |
| d19 | inverter frame 0x1DA lost |
| d51 | VCM frames lost |
| d79 | on-board charger frame 0x390 lost |

Each record byte: **bit7** failing now, **bit6** confirmed, **bits 0–5** aging counter. d3, d11, d19 and d79 read 0xFF
in one CAN supervision state; d2 and d51 are always updated. The other bytes were not traced; a capture of `21 82`
shows what they contain.

### 21 83: identification constants

Stock 4FA2B response: `61 83 34 46 41 32 42 53 42 05 17 00 00 00 00 00 03 05 12 11 00 00 00 00 00 80`

| data | content |
|---|---|
| d0–d4 | application part number `34 46 41 32 42` = "4FA2B" |
| d5 | 0x53 |
| d6–d8 | `42 05 17` |
| d9–d13 | 00 × 5 |
| d14–d17 | `03 05 12 11` |
| d18–d19 | first two bytes of the calibration area (00 00) |
| d20–d22 | 00 00 00 |
| d23 | 0x80 |

### 21 84 / 21 90 / 21 91 / 21 92

| ID | data | content |
|---|---|---|
| 84 | d0–d15, d16–d19 | 16-byte ID string from EEPROM (written by 3B 84), then 4 spaces |
| 90 | d0–d7 | 4 pairs of LB code bytes (3B 90); pair n is shown on 0x1DC as `LB_CODE1` / `LB_CODE2` |
| 91 | d0–d9 | 5 × u16 diagnostic calibration words, same layout as the 3B 91 write |
| 92 | d0–d95 | per-cell voltage trims, signed mV (3B 92). Each cell reading is lowered by the trim (scaled down below 4150 mV) |

### 21 94 / 21 95

| ID | data | content |
|---|---|---|
| 94 | d0 | 1 = capacity balancing complete |
| 95 | d0–d1 | cell-monitor fault-confirmed bits (u16) |

### 21 F0 / F1 / FE / FF: identification blocks

| ID | content |
|---|---|
| F0 | 24 bytes from flash, identical to 21 83 d0–d23 |
| F1 | 24 bytes from flash: 5 × 00, "F0000F0000", 6 × 00, 5C 16 16 |
| FE / FF | 24 bytes from EEPROM if that area has been programmed, else the F0 / F1 contents. May take a few 10 ms passes |

---

## Differences from the AZE0 (4NC0A) firmware

| item | AZE0 4NC0A | e-NV200 4FA2B |
|---|---|---|
| temperature sensors | 3 (21 04 is 16 bytes) | 4 pack sensors + an auxiliary sensor; 21 04 is 37 bytes and adds filtered min / max temperatures |
| advertised IDs | 01–0B, 10, 61–64 | 01–0C, 10, 61–64 |
| 21 03 d4–d5 | modelled cell temperature | always FF FF |
| 21 0A | capacity derate, energy fraction, usable SOC | timer value and a sampled input (the energy values moved to 21 0B) |
| 21 0B | 49 bytes | 89 bytes: adds SOC, current, power limits, SOH, motor data, voltages, 0x59E fields, energy fractions |
| 21 0C | cell overvoltage limits | thermal (cooling) logic state and temperature-delta check |
| 21 10 | battery heater output | thermal output; never set |
| 21 61 d4 | always FF | capacity bars |
| 21 65 / 66 / 70 | present | not supported |
| 21 91 | removed | five diagnostic calibration words |
