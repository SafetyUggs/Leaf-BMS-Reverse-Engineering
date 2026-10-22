# Nissan Leaf ZE1 (40 kWh) battery controller (LBC): diagnostic services

Firmware: **5SH2D** (Li-ion battery controller, ZE1 40 kWh pack, Renesas V850). Everything here was read from the
firmware itself (disassembly and decompiled code), not from Nissan documentation. Names of fields are descriptive
names chosen during the analysis; where a DBC name from the community CAN databases matches, it is given.

Companion documents: [AZE0_4NC0A_diagnostic_services.md](../AZE0_24(4NC0A)/AZE0_4NC0A_diagnostic_services.md) (AZE0
firmware), [ZE0_3NA0D_diagnostic_services.md](../ZE0(3NA0D)/ZE0_3NA0D_diagnostic_services.md) (ZE0 firmware) and the
cyclic frames ([ZE1_5SH2D_cyclic_CAN_frames.md](ZE1_5SH2D_cyclic_CAN_frames.md)).

**Scope of this version:** only service **0x21** (ReadDataByLocalIdentifier) has been analysed on this firmware so
far. Sessions, service 0x19 (DTCs and freeze frames), 0x14, 0x30, 0x31, 0x3B and 0x3E are not covered yet; where a 21
record is written by a 3B service, that is mentioned. Service 0x27 (SecurityAccess) and reprogramming are not covered
on purpose.

Conventions:
- All multi-byte values are **big-endian** in the responses.
- "d0" is the first data byte after the response header, e.g. in `61 01 d0 d1 d2 ...` the header is `61 01`.
- Response lengths include the 2-byte header.
- "inferred" marks a meaning that comes from how a value is used, not from a direct proof.
- Invalid / not-available values are given where the firmware uses one (usually 0xFF, 0xFFFF, 0x80000000).

---

## 1. Transport

| item | value |
|---|---|
| Response CAN ID | **0x7BB** (11-bit) |
| Protocol | KWP2000-style services on ISO-TP (ISO 15765-2) |
| Multi-frame responses | up to 690 bytes |

---

## 2. Service 0x21: read data by local identifier

`21 xx` (length exactly 2). No session or security is needed for any local ID. Unknown ID or wrong length → `7F 21 12`.

| ID | response length | content |
|---|---|---|
| 00 / 20 / 40 / 60 | 6 | supported IDs bitmap (OBD style: d0 bit 7 = ID+1 … d3 bit 0 = ID+0x20). 00 → `FF F1 C0 01` (01–0C, 10–12, 20), 20 → `00 00 00 01`, 40 → `00 00 00 01`, 60 → `FA A9 00 00` (61–65, 67, 69, 6B, 6D, 70). 82, 83, 84, 90, 95, F0, F1, FE and FF work but are not advertised |
| 01 | 53 | currents (two sensors), power limits, voltages, resistance, Hx, SOC, capacity |
| 02 | 198 | 96 cell voltages |
| 03 | 28 | raw current sensors, SOC at wake, min / max cell, faults, motor data |
| 04 | 31 | temperature sensors, vehicle speed |
| 05 | 79 | raw analog / isolation sensor / input snapshot |
| 06 | 26 | per-cell bleed (balancing) switch bitmap |
| 07 | 14 | current-sensor zeros and ADC reference |
| 08 | 5 | NV restore status |
| 09 | 7 | weak-cell (SOC-slope) detector |
| 0A | 12 | energy fractions at full charge |
| 0B | 90 | mirror of the cyclic CAN signals |
| 0C | 7 | distance unit flag and odometer |
| 10 | 5 | digital output and read-back |
| 11 | 10 | raw ADC channels |
| 12 | 7 | external monitor self-test echo |
| 61 | 331 | SOH / aging record |
| 62 | 120 | charge counts and usage histograms |
| 63 | 4 | spare NV word |
| 64 | 3 | 5 V supply fault wake count |
| 65 | 380 | lifetime usage history |
| 67 | 622 | age-milestone history |
| 69 | 7 | odometer rollback record |
| 6B | 8 | cell voltage extremes |
| 6D | 5 | capacity bar metric |
| 70 | 3 | usage-history clear request |
| 82 | 83 | communication DTC records |
| 83 | 26 | identification constants |
| 84 | 22 | 16-byte ID string |
| 90 | 10 | LB code store |
| 95 | 4 | DTC family summary |
| F0 / F1 | 26 | identification block from flash |
| FE / FF | 26 | identification block from EEPROM (or F0 / F1 if blank) |

### 21 01: measurement summary

The ZE1 LBC has **two current sensors**: channel 1 (wide range, 854.5 mA per ADC count) and channel 2 (fine, 305 mA per
count). Battery current is a blend: 100 % channel 2 between −85 A and +135 A, 100 % channel 1 outside −100 / +150 A,
linear in between.

| data | size | content | units / notes |
|---|---|---|---|
| d0–d3 | s32 | current sensor channel 1, zero-corrected, × 1000 | milli ADC counts (1 = 0.8545 mA) |
| d4–d5 | u16 | learned channel 1 zero | ADC counts |
| d6–d9 | s32 | battery current (blended) | mA, + = charge, clamp −500..+200 A; 0x80000000 invalid |
| d10–d13 | s32 | current sensor channel 2, zero-corrected, × 1000 | milli ADC counts (1 = 0.3052 mA) |
| d14–d15 | u16 | charge power limit | 10 W (0x1DC `LB_Charge_Power_Limit` = this / 25), clamp 70 kW; 0xFFFF invalid |
| d16–d17 | u16 | discharge power limit | 10 W, clamp 125 kW; 0xFFFF invalid |
| d18–d19 | u16 | pack voltage from the cell sum | 10 mV; 0xFFFF invalid |
| d20–d21 | u16 | 12 V supply | mV, clamp 24000 |
| d22–d23 | u16 | isolation sensor wave voltage (`LB_IR_Sensor_Wave_Voltage`) | ADC counts; 0x3FF invalid |
| d24–d27 | u32 | pack resistance (ΔV / ΔI between two current levels) | mΩ × 1024, clamp 30 Ω |
| d28–d29 | u16 | Hx (resistance health) | 0.01 %; 0xFFFF invalid |
| d30–d33 | u32 | SOC (`LB_SOC`) | ppm (1 000 000 = 100 %); 0xFFFFFFFF invalid |
| d34–d37 | u32 | full capacity | 0.1 mAh (= SOH × 115.44 Ah) |
| d38–d39 | s16 | max power for charger | 100 W (0x1DC sends + 100); 0x8000 not set |
| d40 | bits | bit0 HV interlock loop closed (`LB_INTER_LOCK`); bit1 full charge (`LB_Full_CHARGE_flag`); bit2 interlock sense input, debounced | |
| d41–d44 | s32 | channel 2 current, 10-sample mean | mA, clamp −100..+150 A; 0x80000000 invalid |
| d45–d48 | s32 | channel 1 current, 10-sample mean | mA, clamp −500..+200 A; 0x80000000 invalid |
| d49–d50 | u16 | learned channel 2 zero | ADC counts |

### 21 02: cell voltages

| data | size | content | units |
|---|---|---|---|
| d0–d191 | 96 × u16 | cell voltage 1..96 | mV, clamp 4550; 0xFFFF invalid (cell-monitor communication error or no sample) |
| d192–d193 | u16 | pack voltage (as on 0x1DB `LB_Total_Voltage`) = cell sum | 10 mV; 0xFFFF invalid |
| d194–d195 | u16 | inverter voltage from CAN 0x1DA | 10 mV; 0xFFFF invalid |

Each cell value is the mean of the last 5 valid cell-monitor readings, minus a connection drop = current × a per-cell
connection resistance from a flash table (stock 0.14..0.52 mΩ). If a cell-monitor group reports a communication or
plausibility error, all its cells read 0xFFFF.

### 21 03: sensors and status

| data | size | content | units / notes |
|---|---|---|---|
| d0–d1 | 2 | always FF FF (no HV voltage sensor) | |
| d2–d3 | u16 | current sensor channel 1 | ADC counts (10 bit); 0x3FF saturated |
| d4–d5 | u16 | current sensor channel 2 | ADC counts; 0x3FF saturated |
| d6–d9 | u32 | SOC when it was initialised this wake (stored or from OCV) | ppm |
| d10–d11 | u16 | highest cell voltage | mV; 0xFFFF invalid |
| d12–d13 | u16 | lowest cell voltage | mV; 0xFFFF invalid |
| d14 | u8 | 5 V supply fault wake count | wakes with a confirmed 5 V supply fault (4711..5273 mV window), 0..40; at 40 the 5 V sensor supply output is no longer enabled; writable with 3B 64 |
| d15–d16 | u16 | isolation sensor | ADC counts; 0x3FF invalid |
| d17 | u8 | relay cut request (`LB_Relay_Cut_Request`) | |
| d18 | u8 | failsafe status (`LB_Failsafe_Status`) | bits |
| d19–d20 | u16 | fault summary bits | 16 fault classes (below) |
| d21–d22 | s16 | motor torque from CAN 0x1DA | Nm, clamp ±340; 0x8000 = no frame |
| d23–d24 | s16 | motor speed from CAN 0x1DA | rpm; 0x8000 = no frame |
| d25 | bits | bit0 main-relay-on permission (`LB_MainRelayOn_flag`); bit1 VCM 0x1D4 byte 5 bit 6 (HV active; enables coulomb counting); bit2 isolation sensor fault latched (`LB_IR_Sensor_Malfunction`); bit3 VCM 0x1D4 byte 4 bit 2 | |

**Fault summary**: each bit is a fail-safe class fed by the fault judgements of the diagnostic items (bit 4 is always
0). Bits 0 or 12 set charge and discharge power to 0; bit 3 sets charge power to 0; every set bit caps the power limits
with its own pair of values from a flash table; bits 1, 2 and 3 enable the inverter-voltage derates.

### 21 04: temperatures

| data | size | content | units |
|---|---|---|---|
| d0–d1 / d2 | u16 / s8 | temperature sensor 1: ADC / °C | counts (0x3FF invalid) / °C (−40..105) |
| d3–d4 / d5 | u16 / s8 | temperature sensor 2 | |
| d6–d8 | 3 | always FF | |
| d9–d10 / d11 | u16 / s8 | temperature sensor 3 | |
| d12 | s8 | minimum temperature, 10-sample average | °C; 0x80 invalid |
| d13 | u8 | vehicle speed (motor rpm × 9577 / 661500) | km/h, 0..127 |
| d14–d28 | 15 | always FF | |

With one or two sensors missing, the max temperature gets a +2 / +4 / +12 °C margin and the min a −15 °C substitute.

### 21 05: raw input snapshot

| data | size | content |
|---|---|---|
| d0–d1 | u16 | 12 V supply input, ADC (12 V = ADC × 5000 / 1024 × 3.7 mV) |
| d2–d3 | u16 | 2.5 V reference, ADC (nominal 512) |
| d4–d5 | u16 | 5 V supply, ADC (5 V = ADC × 5000 / 1024 × 2.5 mV) |
| d6–d7 | u16 | current sensor channel 2, ADC |
| d8–d9 | 2 | always FF FF |
| d10–d11 / d12–d13 | u16 | temperature sensors 1 / 2, ADC |
| d14–d15 | 2 | always FF FF |
| d16–d17 | u16 | temperature sensor 3, ADC |
| d18–d19 | u16 | current sensor channel 1, ADC |
| d20–d23 | s32 | battery current, mA (0x80000000 invalid) |
| d24–d25 | u16 | isolation sensor wave voltage, ADC (0x3FF invalid) |
| d26–d45 | 10 × u16 | isolation sensor samples, drive high (0xFFFF = invalid sample) |
| d46–d65 | 10 × u16 | isolation sensor samples, drive low |
| d66 | u8 | 0x55 = both wake inputs high, 0xAA otherwise |
| d67 | u8 | second wake input: 0x55 high, 0xAA low (debounced) |
| d68 | bits | bit0 HV-interlock drive output (toggles every 150 ms); bit1 interlock sense input, raw level |
| d69–d72 | s32 | channel 2 current, 10-sample mean, mA |
| d73–d76 | s32 | channel 1 current, 10-sample mean, mA |

The isolation sensor drive output toggles every 25 passes; each half period stores one sample into a 10-sample ring.
The wave voltage is the difference of the two ring means.

### 21 06: bleed switch states

| data | size | content |
|---|---|---|
| d0–d11 | 12 bytes = 96 bits | **byte n bit k (LSB first) = cell 8n + k + 1** (21 02 order); 1 = bleed switch on, as read back from the cell monitors (not just the request) |
| d12–d23 | 12 | always FF |

A cell is bled when its value exceeds the balancing reference + 17 (a calibration constant). The read-back is refreshed
one cell per pass, only while the cell-monitor link has no errors.

### 21 07: current-sensor zeros

| data | size | content | units |
|---|---|---|---|
| d0–d1 | 2 | always FF FF | |
| d2–d5 | u32 | channel 1 zero in use | ADC counts × 10⁶ (default 687 000 000) |
| d6–d7 | u16 | 2.5 V reference | ADC counts (512 = 2.500 V) |
| d8–d11 | u32 | channel 2 zero in use | ADC counts × 10⁶ (default 430 000 000) |

The zeros are learned once per wake with the main relay open and HV inactive: up to 10 samples per channel, at least 3
needed, accepted in the range 667..708 counts (channel 1) and 410..450 counts (channel 2). If the VCM already reports
HV active, the defaults are used.

### 21 08: NV restore status

| data | size | content |
|---|---|---|
| d0 | u8 | after a retained-RAM loss, which NV copy of the SOH record was restored: 0 copy A, 1 copy B, 0x80 none (defaults loaded). Only written when the retained RAM was invalid at boot |
| d1 | u8 | 0 = DTC list and freeze frames restored from NV, 0x80 = not valid |
| d2 | u8 | 1 = a hardware / supply diagnostic check failed: analog measurements are treated as invalid |

### 21 09: weak-cell detector

While charging, the max, min and average cell voltages are converted to SOC on the OCV curve (corrected for current ×
resistance). A new snapshot is taken each time the average SOC has moved by about 10 %. A cell whose SOC moves much
faster than the average has less capacity.

| data | size | content | units |
|---|---|---|---|
| d0 | u8 | highest ratio seen | ratio × 64 (64 = same as average) |
| d1 | u8 | ratio of the highest-voltage cell (\|ΔSOC cell\| / \|ΔSOC average\| between the last two snapshots) | ratio × 64, clamp 255 |
| d2 | u8 | ratio of the lowest-voltage cell | ratio × 64 |
| d3 | u8 | weak cell pair (highest-voltage cell), when its ratio > 115 / 64 (1.8) | pair 1..48 (cell index / 2 + 1) |
| d4 | u8 | weak cell pair (lowest-voltage cell) | pair 1..48 |

Two consecutive ratios above 1.8 set DTC P33ED.

### 21 0A: energy fractions

All Q15 (32768 = 100 %).

| data | size | content |
|---|---|---|
| d0–d1 | u16 | expected usable energy fraction at the charge end, latched when full charge was reached |
| d2–d3 | u16 | usable energy fraction from the cell-SOC estimator, latched at the same moment |
| d4–d5 | u16 | display SOC (also sent on 0x59E byte 7) |
| d6–d7 | u16 | learned ratio d2 / d0 over full normal charges (IIR, k = 231 / 256) |
| d8–d9 | u16 | the same over full quick charges |

0x59E full capacity = SOH × 40.749 kWh × (d2 / 32768) × a second derate fraction. The values are retained.

### 21 0B: CAN signal mirror

| data | content | invalid |
|---|---|---|
| d0 | `LB_Discharge_Power_Status` | |
| d1 | `LB_Charge_Power_Status` | |
| d2 | `LB_BPCMAX_UPRATE` (0 or 7) | |
| d3 | `LB_CODE_CONDITION` (0..3) | |
| d4 / d5 | `LB_CODE1` / `LB_CODE2` (bytes 2n, 2n+1 of the 21 90 record) | |
| d6 | always FF | |
| d7 | `LB_RefusetoSleep` (1 sleep allowed, 2 refuse) | |
| d8–d11 | remaining energy in 0.01 Wh, quantised to 80 Wh steps with hysteresis (÷ 8000 = gids on 0x5BC) | 0xFFFFFFFF |
| d12 | `LB_Remaining_Capacity_Segments` (capacity bars or SOC × 240, alternating) | 0xFF |
| d13 | `LB_Temperature_Segment_For_Dash` (segment × 20) | 0xFF |
| d14 | `LB_Output_Power_Limit_Reason` | |
| d15 | `LB_Remain_Charge_Time_Condition` | |
| d16–d17 | `LB_Remain_Charge_Time` (minutes) | 0xFFFF |
| d18 | 0x5C0 page (1..6, 0 none) | |
| d19–d24 | always FF | |
| d25 | `LB_NextWakeupTimeForBatterHeater` source value (always 0xFF in this build) | |
| d26 | always FF | |
| d27–d28 | full energy for QC (`LB_Full_Capacity_for_QC`, 0.1 kWh) | |
| d29–d30 | remaining energy for QC (`LB_Remain_Capacity_for_QC`, 0.1 kWh) | |
| d31 / d32 | last message counter / CRC received in VCM frame 0x1D4 | |
| d33 | VCM state from 0x50B byte 3 bits 7..6: 0 (raw 0..2) or 3 (inferred: 3 = wake request) | |
| d34–d36 | always FF | |
| d37 | charge mode from 0x1F2 (0 none, 1 normal, 2 quick) | |
| d38 | 0x1F2 message counter | |
| d39 | on-board charger power class from 0x390: 1 = 0.7..1.2 kW, 2 = 1.4..6.0 kW | |
| d40–d41 | on-board charger power from 0x390 (100 W) | 0xFFFF |
| d42 / d43 | 0x1DA message counter / CRC | |
| d44 | ambient temperature from 0x54C ((x − 81) / 2 °C, clamp −40..60) | 0x80 |
| d45 | bit0 always 0; bit1 `LB_Capacity_Empty`; bit2 `LB_Remain_Cap_Segment_Swit_Flag`; bits 3..5 heater start / stop / mail send request (always 0); bit6 0x50B byte 2 bit 2; bit7 0x50B byte 6 bit 5 | |
| d46 | bits 0..1 from 0x1F2; bit2 0x3B8 byte 1 bit 4; bit3 0x3B8 byte 1 bit 3; bits 4..7 0 | |
| d47–d48 | charger current limit from 0x3B8 (0.25 A) | 0xFFFF |
| d49–d87 | always FF | |

### 21 0C: distance unit and odometer

Different content from the AZE0 21 0C (cell overvoltage limits).

| data | size | content | units |
|---|---|---|---|
| d0 | u8 | flag from 0x355 byte 6: 0x60 → 1, 0x40 → 0, other values keep the last state (inferred: distance unit miles / km) | |
| d1–d4 | u32 | corrected odometer = 0x5C5 odometer (24 bit, valid 0..999999) + rollback offset (21 69), clamp 0xFFFFFE | 0x5C5 units; last value kept while 0x5C5 is invalid |

Whenever the 0x5C5 odometer goes down, the drop is added to the retained rollback offset and the rollback count is
incremented, so the corrected odometer never decreases.

### 21 10: digital output

| data | size | content |
|---|---|---|
| d0–d1 | 2 | always FF FF |
| d2 | bits | bit1 output P400.0 command; bit2 read-back input of that output |

A mismatch between the two bits sets DTC P33EA. No code ever switches this output on in this firmware, so bit1 is
always 0. Its purpose is unknown (on ZE1 the battery heater request is a CAN signal, not this output).

### 21 11: raw ADC channels

| data | size | content |
|---|---|---|
| d0–d1 | u16 | 5 V supply, ADC |
| d2–d3 | u16 | a further ADC channel, only used in a "channels not stuck" check |
| d4–d5 | u16 | ADC low test channel (must read < 4) |
| d6–d7 | u16 | ADC high test channel (must read ≥ 1020) |

### 21 12: external monitor self-test echo

| data | size | content | invalid |
|---|---|---|---|
| d0 | u8 | self-test number 0..17 echoed by the external monitor | 0xFF |
| d1–d4 | u32 | self-test result word echoed by the monitor | 0xFFFFFFFF |

The LBC runs one of 18 CPU self-tests each pass (RAM patterns etc.) and sends the test number and 32-bit result with a
CRC to an external monitor chip over a UART. The monitor answers with the test number, the result and a status. A bad
reply (receive or CRC error) sets DTC P33F9 and this record reads FF; each echoed result is also compared with an
18-word table of expected values. The monitor chip was not identified (inferred: watchdog / safety monitor IC).

### 21 61: SOH / aging record

On ZE1 this is the **whole retained SOH state**, 329 data bytes, and 3B 61 writes it back in the same layout.

| data | size | content | units / notes |
|---|---|---|---|
| d0–d1 | u16 | Hx (resistance health) = d5–d8 / 1024 | 0.01 % |
| d2–d3 | u16 | SOH = d146 + d156, clamp 1..10000 (full capacity = SOH × 115.44 Ah) | 0.01 % |
| d4 | u8 | capacity bars × 20 (from SOH, only lowered) | 0..240 |
| d5–d8 | u32 | Hx filter (IIR of the Hx sample, weight 9 / 65536) | Hx × 1024 |
| d9–d10 | 2 | always FF FF (ignored by 3B 61) | |
| d11 | bit0 | Hx initialised flag (the first Hx update only loads the defaults) | |
| d12–d15 | u32 | full capacity (recomputed from SOH at every reset) | 0.1 mAh |
| d16–d19 | u32 | weakest-cell capacity = SOH2 × 115.44 Ah (scales the C-rate tables of the power model) | 0.1 mAh |
| d20–d145 | 63 × u16 | OCV curve at the 63 SOC points of the OCV axis (rebuilt from the flash OCV-by-SOH table at every reset) | mV |
| d146–d147 | u16 | SOH base = 10000 − cycle wear − calendar wear | 0.01 % |
| d148–d151 | u32 | total aging time, awake and asleep | s |
| d152–d155 | u32 | aging time at the last SOH aggregation (every 90 days) | s |
| d156–d157 | s16 | SOH offset (slews ±7 per update toward d197) | 0.01 % |
| d158–d164 | 7 × u8 | year (0..62), month, day, hour, minute, second (from 0x5CD) and average temperature (°C) at the last aging pass | |
| d165–d168 | u32 | SOC at the last aging pass | ppm |
| d169–d172 | u32 | SOC stress: ∫ exp(2 (SOC − 1) / 0.865) dt | s (1 / s at 100 % SOC) |
| d173–d176 | u32 | temperature stress: ∫ exp(2 (T − 25 °C) / 12.29) dt | s (1 / s at 25 °C) |
| d177–d180 | u32 | calendar term = SOC stress × temperature stress / total time | s |
| d181–d184 | u32 | awake time | s |
| d185–d188 | u32 | awake temperature stress: ∫ exp((T − 25 °C) / 23.19) dt | s |
| d189–d192 | u32 | charge throughput integral (∫ −I dt while not normal-charging, or quick charge with I < 0) | 2 mAh, clamp 0x7FFFFFFF |
| d193–d194 | u16 | cycle wear | 0.01 % |
| d195–d196 | u16 | calendar wear | 0.01 % |
| d197–d198 | s16 | SOH offset target | 0.01 % |
| d199–d200 | s16 | weighted mean of the sample spreads (mean − min cell capacity) | 0.01 % |
| d201–d240 | 20 × u16 | capacity-test sample weights (largest 20 kept) | |
| d241–d280 | 20 × s16 | capacity-test samples: mean of the 96 cell capacities − SOH base | 0.01 % |
| d281–d320 | 20 × u16 | capacity-test spreads: mean − min cell capacity | 0.01 % |
| d321–d324 | u32 | SOH2 (weakest cell) = SOH base + SOH2 offset − 4.40 %, clamp 0..10000 | 0.01 % |
| d325–d326 | s16 | previous SOH offset target | 0.01 % |
| d327–d328 | s16 | SOH2 offset (slews ±7 toward d197 − d199) | 0.01 % |

Defaults: Hx and SOH 100.00 %, bars 240, capacities 115.44 Ah, OCV curve at 100 % SOH, the clock bytes FF and the
temperature 0x80, the last SOC 0xFFFFFFFF, everything else 0.

The aging step runs every 100 ms while awake (dt = 0.1 s) and once at wake-up with dt = the sleep time (clock now −
clock stored, clamp 1 day; using the mean of the stored and current SOC and temperature). The accumulators feed a
cycle / calendar wear model that gives the SOH base. Every 90 days of aging time the capacity-test samples are
aggregated into the SOH offset target and the sample buffers are cleared.

What survives a 3B 61 write:
- the full and weakest-cell capacities and the OCV curve are recomputed at the next reset (a capacity above 119.24 Ah
  makes the next boot reject the whole record and restore a backup or the defaults);
- SOH alone is undone by the next SOH update, because the SOH base is recomputed from the aging accumulators; the
  accumulators and offsets must be written consistently to move SOH (partly inferred);
- Hx is kept and slowly pulled toward the measured value.

### 21 62: usage statistics (counts)

All u16, saturating at 65535. Written by 3B 62.

| data | size | content |
|---|---|---|
| d0–d1 | u16 | normal charge sessions (0x5EC type 1) |
| d2–d3 | u16 | quick-charge-port sessions (0x5EC types 2..5) |
| d4–d5 | u16 | those d2 sessions whose start current limit (first valid 0x3B8 value) is below 25 A |
| d6–d21 | 8 × u16 | max temperature at ignition on |
| d22–d37 | 8 × u16 | max temperature at charge start |
| d38–d53 | 8 × u16 | highest max temperature while the ignition was on |
| d54–d69 | 8 × u16 | highest max temperature while charging |
| d70–d85 | 8 × u16 | SOC at ignition on |
| d86–d101 | 8 × u16 | SOC at charge start |
| d102–d117 | 8 × u16 | SOC when full charge was first detected |

Temperature bins: < 35, 35–39, 40–44, 45–49, 50–54, 55–59, 60–64, ≥ 65 °C. SOC bins: < 20, 20–30, 30–40, 40–50, 50–60,
60–70, 70–85, ≥ 85 %. Each histogram counts at most once per power-up; invalid inputs are not counted.

0x5EC session types used here and in 21 65 / 21 67: 0 no charge session (driving, inferred), 1 normal charge, 2 / 3
quick charge (type 3 takes the charger current from 0x3B8), 4 / 5 sessions in which only discharge current is
integrated (inferred: power output through the quick-charge port).

### 21 63 / 21 64

| ID | data | content |
|---|---|---|
| 63 | d0–d1 | spare word kept in retained memory, written only by 3B 63 (session C0); default 0; no other use |
| 64 | d0 | 5 V supply fault wake count (as 21 03 d14); written by 3B 64 |

### 21 65: lifetime usage history

All u32 except the last field (u16), saturating. Retained; written by 3B 65 and cleared at the next shutdown after
`3B 70 01`. Power and SOC means are Q32 fractions: value / 2³² × 100 kW, or × 100 %.

| data | size | content |
|---|---|---|
| d0–d3 | u32 | lifetime time-weighted mean \|V × I\| in type 0 (driving) |
| d4–d7 | u32 | mean \|V × I\| in type 4 / 5 |
| d8–d11 | u32 | mean \|V × I\| in type 2 / 3 (quick charge, active) |
| d12–d15 | u32 | mean SOC in type 4 / 5 |
| d16–d19 | u32 | distance: Σ (odometer at shutdown − odometer at wake), 0x5C5 units |
| d20–d139 | 30 × u32 | minutes by max temperature bin (10) × SOC bin (3), index = T × 3 + S |
| d140–d175 | 9 × u32 | awake minutes by max temperature bin (9) |
| d176–d295 | 30 × u32 | as d20, using the min temperature |
| d296–d331 | 9 × u32 | as d140, using the min temperature |
| d332–d335 | u32 | Σ per-power-up net discharge in type 0, A·s |
| d336–d339 | u32 | Σ discharge in type 4 / 5, A·s |
| d340–d343 | u32 | Σ change of 0x3B8 counter A × 0.04 while a received CAN status bit is 0 (bit source and counter meaning not identified) |
| d344–d347 | u32 | the same while that bit is 1 |
| d348–d351 | u32 | Σ change of 0x3B8 counter B × 0.004 |
| d352–d355 | u32 | Σ change of 0x3B8 counter C × 0.004 |
| d356–d359 | u32 | minutes in type 0 |
| d360–d363 | u32 | minutes in type 4 / 5 |
| d364–d367 | u32 | minutes in type 2 / 3, active |
| d368–d371 | u32 | minutes in type 1, active |
| d372–d375 | u32 | total minutes (parked + awake) |
| d376–d377 | u16 | power-ups with HV active (0x1D4 byte 5 bit 6) |

The 30-bin histograms get the parked minutes at each wake-up (clock now − shutdown clock, binned at the mean of the
shutdown and wake values) and the awake minutes at each shutdown. The 9-bin histograms get only the awake minutes.

Bin edges: 10-bin temperature < 0, 0–10, 11–19, 20–26, 27–32, 33–37, 38–41, 42–45, 46–49, ≥ 50 °C; 9-bin temperature
< 0, 0–11, 12–21, 22–29, 30–36, 37–41, 42–45, 46–49, ≥ 50 °C; SOC < 45, 45–84, ≥ 85 %.

### 21 67: age-milestone history

Five slots for the periods 0–1, 1–2, 2–3, 3–5 and 5–8 years of total time (21 65 d372). Slot n is filled at the first
shutdown after the milestone; each slot holds the change over its own period. The day histograms of the running slot
are updated during the period. Arrays are slot-major ([slot][…]).

| data | size | per slot |
|---|---|---|
| d0–d9 | 5 × u16 | mean \|V × I\| in type 0, value / 65536 × 100 kW |
| d10–d19 | 5 × u16 | same, type 4 / 5 |
| d20–d29 | 5 × u16 | same, type 2 / 3 |
| d30–d39 | 5 × u16 | mean SOC in type 4 / 5, value / 65536 × 100 % |
| d40–d49 | 5 × u16 | distance (21 65 d16) × 10 / 64 |
| d50–d199 | 5 × 30 × u8 | days (clamp 255) by max temperature bin × SOC bin (21 65 bins) |
| d200–d244 | 5 × 9 × u8 | awake days by max temperature bin |
| d245–d394 | 5 × 30 × u8 | days by min temperature bin × SOC bin |
| d395–d439 | 5 × 9 × u8 | awake days by min temperature bin |
| d440–d449 | 5 × u16 | quick-charge-port sessions (Δ 21 62 d2) |
| d450–d459 | 5 × u16 | low-current quick-charge sessions (Δ 21 62 d4) |
| d460–d469 | 5 × u16 | Hx at the milestone, 0.01 % |
| d470–d479 | 5 × u16 | discharge in type 0, 200 Ah units |
| d480–d489 | 5 × s16 | discharge in type 4 / 5, 200 Ah units |
| d490–d499 | 5 × u16 | 0x3B8 counter C (Δ × 5 / 2000) |
| d500–d509 | 5 × u16 | 0x3B8 counter A, bit = 1 (Δ × 5 / 2000) |
| d510–d519 | 5 × u16 | 0x3B8 counter B (Δ × 5 / 2000) |
| d520–d529 | 5 × u16 | 0x3B8 counter A, bit = 0 (Δ × 5 / 2000) |
| d530–d549 | 5 × u32 | minutes in type 0 |
| d550–d569 | 5 × u32 | minutes in type 4 / 5 |
| d570–d589 | 5 × u32 | minutes in type 2 / 3 |
| d590–d609 | 5 × u32 | minutes in type 1 |
| d610–d619 | 5 × u16 | SOH at the milestone, 0.01 % |

Firmware quirk: when the firmware subtracts the earlier slots' share of the type-0 mean power, it weights them by the
type-1 minutes instead of the type-0 minutes, so later type-0 slot means are slightly off.

### 21 69: odometer rollback record

| data | size | content |
|---|---|---|
| d0 | u8 | number of times the 0x5C5 odometer went down (cap 255) |
| d1–d4 | u32 | sum of those decreases (clamp 999999); added to 0x5C5 to form the corrected odometer of 21 0C |

Written by 3B 69 (same layout).

### 21 6B: cell voltage extremes

Tracked only during 0x5EC session types 4 / 5. Written by 3B 6B.

| data | size | content |
|---|---|---|
| d0–d1 | u16 | highest max cell voltage ever, mV |
| d2–d3 | u16 | lowest min cell voltage ever, mV |
| d4 | u8 | power-ups with the max cell above 4230 mV (cap 255) |
| d5 | u8 | power-ups with the min cell below 2500 mV (cap 255) |

### 21 6D: capacity bar metric

| data | size | content |
|---|---|---|
| d0–d1 | u16 | lowest bar metric seen, 0.01 % (bar metric = SOH × a bars factor table × 1.011, computed once per reset; only lowered) |
| d2 | u8 | capacity % = min(bar metric at the displayed bar level, d0) / 100, clamp 1..100 |

Written by 3B 6D.

### 21 70

| data | content |
|---|---|
| d0 | 1 = `3B 70 01` was received: the 21 65 / 21 67 history is cleared at the next shutdown |

### 21 82: communication DTC records

81 data bytes. Default contents: d0–d1 = `12 50`, d60 = 00, all others FF. Nine bytes carry a record:

| data | fault | "now" condition |
|---|---|---|
| d2 | CAN bus-off | bus-off now (confirmed after 200 passes) |
| d3 | LBC transmit failure (0x1DB 0x55B 0x1DC 0x5BC 0x5C0 0x59E 0x5EB still pending in the mailbox) | 3 passes |
| d6 | odometer frame 0x5C5 or 0x355 lost | 30 / 12 passes |
| d11 | A/C amplifier frame 0x54C lost | 30 passes |
| d19 | inverter frame 0x1DA lost | 3 passes |
| d23 | frame 0x625 lost | 30 passes |
| d51 | VCM / charge frames lost (0x1D4, 0x1F2, 0x50B, 0x3B8 or 0x5EC) | 3 to 100 passes |
| d79 | on-board charger frame 0x390 lost | 30 passes |
| d80 | frame 0x56E lost (whole byte FF until enabled) | 30 passes |

Each record byte: **bit7** failing now, **bit6** confirmed (a counter +3 per faulty pass / −1 per good pass reached
200), **bits 0–5** healing counter (40 while the fault is confirmed, then −1 per ignition cycle without it, stopping at
1). The counters are retained. In one ignition-on supply mode, d3, d11, d19, d23, d79 and d80 read 0xFF.

### 21 83: identification constants

| data | content (stock 5SH2D) |
|---|---|
| d0–d4 | application part number `35 53 48 32 44` = "5SH2D" |
| d5 | 0x56 "V" |
| d6–d8 | `42 05 17` |
| d9–d13 | 00 × 5 |
| d14–d17 | `07 33 13 11` (from the calibration area) |
| d18–d19 | `00 00` (from the calibration area) |
| d20–d22 | 00 00 00 |
| d23 | 0x80 |

Only d14–d19 come from the calibration area; the rest is fixed in code.

### 21 84 / 21 90

| ID | data | content |
|---|---|---|
| 84 | d0–d15, d16–d19 | 16-byte ID string from NV (written by 3B 84, session F1), then 4 spaces |
| 90 | d0–d7 | 8 bytes written by 3B 90 (session F0); pair n is sent on 0x1DC as `LB_CODE1` / `LB_CODE2` with `LB_CODE_CONDITION` = n |

### 21 95: DTC family summary

Each bit = 1 if any diagnostic item of that family has a confirmed failure.

| data | bit0 | bit1 | bit2 | bit3 | bit4 | bit5 | bit6 | bit7 |
|---|---|---|---|---|---|---|---|---|
| d0 | P33B0–P33B7 | P33A6–P33AE | P33AF | P33A5 | 0 | 0 | 0 | 0 |
| d1 | P307B–P3082 | P32CE–P32D5 | P32C6–P32CD | P32BE–P32C5 | P32B6–P32BD | P33C8–P33CF | P33C0–P33C7 | 8 items with no DTC |

The 8-item families most likely correspond to the 8 cell-monitor groups (inferred).

### 21 F0 / F1 / FE / FF: identification blocks

| ID | content |
|---|---|
| F0 | 24 bytes from the boot area: `35 53 48 32 44 56 42 05 17 00 00 00 00 00 07 33 13 11 00 00 00 00 00 80` ("5SH2DVB" …, the same values as 21 83) |
| F1 | 24 bytes from the boot area: `00 00 00 00 00 46 30 30 30 30 46 30 30 30 30 00 00 00 00 00 00 5C 09 A3` ("F0000F0000" …) |
| FE / FF | 24 bytes from the external EEPROM (FE: 0x1FD0..0x1FE7, FF: 0x1FE8..0x1FFF) if that area has been programmed (EEPROM byte 0x1FFD ≠ 0xFF), else the F0 / F1 contents. If the EEPROM is busy, the answer comes on a later pass |

---

## 3. Differences from the AZE0 (4NC0A) firmware (service 0x21)

| item | AZE0 4NC0A | ZE1 5SH2D |
|---|---|---|
| current sensors | one | two (wide and fine range), blended; 21 01, 21 03, 21 05 and 21 07 carry both channels |
| 21 01 | 41 bytes, resistance u16 mΩ | 53 bytes, resistance u32 mΩ × 1024, channel 2 data appended |
| 21 06 | one bleed status byte per 4-cell group | 96-bit per-cell bleed bitmap |
| 21 0C | cell overvoltage limits | distance unit flag and corrected odometer |
| 21 61 | 12 bytes (Hx, SOH stages) | 331 bytes: whole SOH / aging state incl. OCV curve and capacity-test samples |
| 21 62 | two charge counters | third counter (low-current quick charges) at the front |
| 21 65 | 166 bytes | 380 bytes, max and min temperature histograms, session-type means |
| new IDs | | 11, 12, 67, 69, 6B, 6D |
| removed IDs | | 66, 92, 94 |
| 21 82 | 6 records, other bytes 0 | 9 records, other bytes FF |
| 21 83 / F0 | "4NC0A" … | "5SH2DV" … |
