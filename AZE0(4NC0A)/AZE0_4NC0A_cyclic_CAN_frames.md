# Nissan Leaf AZE0 (24 kWh) battery controller (LBC): cyclic CAN frames

Firmware: **4NC0A** (AZE0 24 kWh pack, Renesas V850). Everything here was read from the firmware itself: the signal
layouts come from the descriptor tables the firmware packs with, the scaling and limits from the packing code, and
the meanings from the code that computes each value. Signal names follow the community DBC where one matches.

Companion documents: [ZE0_3NA0D_cyclic_CAN_frames.md](ZE0_3NA0D_cyclic_CAN_frames.md) (ZE0 firmware) and the diagnostic
services ([AZE0_4NC0A_diagnostic_services.md](AZE0_4NC0A_diagnostic_services.md)).

Conventions:
- **Layout** is given as `byte.bit / length`: the signal's most significant bit is bit `bit` of byte `byte` (bit 7 = MSB
  of the byte), and longer signals continue into the following bytes (Motorola / big-endian order). **LSB** is the
  position of the least significant bit as byte × 8 + bit, the "start bit" most DBC editors show for Motorola signals.
- "invalid" is the value sent when the source is not available.
- "inferred" marks a meaning that comes from how a value is used, not from a direct proof.

## Frames

| ID | period | DLC | content | counter / CRC |
|---|---|---|---|---|
| 0x1DB | 10 ms | 8 | current, pack voltage, relay and status bits, usable SOC | PRUN + CRC8 |
| 0x1DC | 10 ms | 8 | power limits, charger power, LB codes | PRUN + CRC8 |
| 0x55B | 100 ms | 8 | SOC, ALU answer, isolation sensor, capacity empty, sleep | PRUN + CRC8 |
| 0x5BC | 100 ms | 8 | gids, bars, temperature gauge, SOH, charge time | none |
| 0x5C0 | 500 ms | 8 | history data, heater requests, DTC | none |
| 0x59E | 500 ms | 8 | full / remaining energy for quick charge | none |
| 0x45D | 500 ms | 8 | max / min cell voltage and temperature, fault bits | none |

All frames are sent only while the LBC is awake with CAN active. Bytes not covered by a signal are 0.

**PRUN** is a 2-bit message counter 0, 1, 2, 3, 0 … (one step per frame). **CRC8** in byte 7 is computed over bytes 0–6
(the same routine checks the CRC of the VCM frames the LBC receives).

---

## 0x1DB (10 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Current` | 0.7 / 11 | 13 | signed, 0.5 A | battery current, **+ = charge**; clamp −400 A … +200 A; 0x3FF invalid |
| `LB_Relay_Cut_Request` | 1.4 / 2 | 11 | | 0 none, 3 = open the main relays (latched): cell over / undervoltage fault, cell OV limit level ≥ 2, a startup check failure, or an OV limit level 2 seen during a quick charge |
| `LB_Failsafe_Status` | 1.2 / 3 | 8 | bits (latched) | bit0 battery cold: min temperature ≤ −26 °C (confirmed); bit1 cell OV limit level 1, a cell / sensor / temperature fault group, or a current-sensor fault during a quick charge or below 26 °C; bit2 relay cut request, any fault, or OV limit level 2 seen |
| `LB_Total_Voltage` | 2.7 / 10 | 30 | 0.5 V | pack voltage = sum of the 96 cells; clamp 450 V; 0x3FF invalid |
| `LB_MainRelayOn_flag` | 3.5 / 1 | 29 | | main relay ON permission (startup checks passed; cleared by a relay cut) |
| `LB_Full_CHARGE_flag` | 3.4 / 1 | 28 | | full charge: any of usable SOC at the charge target (charge-target mode), allowed charge power tapered below 1.22 kW for 1000 passes, max cell at the full-charge voltage (table by current and temperature), max cell within 44 mV of it in a charge-control fault mode, or the charge-end counter |
| `LB_INTER_LOCK` | 3.3 / 1 | 27 | | 1 = HV interlock loop closed (the LBC drives a square wave and checks the sense input follows it) |
| `LB_Discharge_Power_Status` | 3.2 / 2 | 25 | | 1 normal, 2 high-rate limit (demand above the power that pulls the lowest cell to 2.35 V), 3 immediate limit (above the 2.20 V power, or fault) |
| `LB_Voltage_Latch_Flag` | 3.0 / 1 | 24 | | toggles while the cell monitoring runs without failures (a heartbeat of the cell-voltage scan) |
| `LB_Usable_SOC` | 4.6 / 7 | 32 | 1 % | usable SOC (dash SOC), clamp 100; 0x7F invalid |
| cell OV limit level | 6.3 / 2 | 50 | | 0 none, 1 max cell ≥ limit B, 2 max cell ≥ limit A, 3 cell overvoltage fault (AZE0 only; see 21 0C in the diagnostic document) |
| `LB_PRUN_1DB` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

---

## 0x1DC (10 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Discharge_Power_Limit` | 0.7 / 10 | 14 | 0.25 kW | allowed discharge power, clamp 254 kW; 0x3FF invalid |
| `LB_Charge_Power_Limit` | 1.5 / 10 | 20 | 0.25 kW | allowed charge (regen) power, clamp 254 kW; 0x3FF invalid |
| `LB_MAX_POWER_FOR_CHARGER` | 2.3 / 10 | 26 | 0.1 kW, offset −10 kW | power the charger may deliver during a charge session, 0..1000 (−10 … +90 kW); 0x3FF not set |
| `LB_Charge_Power_Status` | 3.1 / 2 | 24 | | 1 normal, 2 high-rate limit, 3 immediate limit (or fault) |
| `LB_BPCMAX_UPRATE` | 4.7 / 3 | 37 | | power uprate level, clamp 7 |
| `LB_CODE_CONDITION` | 4.4 / 3 | 34 | | index 0..3 of the code pair currently shown (steps every frame); 7 = not available |
| `LB_CODE1` | 4.1 / 8 | 42 | | first byte of code pair n (stored in EEPROM, written by diagnostics 3B 90; not DTCs) |
| `LB_CODE2` | 5.1 / 8 | 50 | | second byte of code pair n |
| `LB_PRUN_1DC` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

The power limits are the result of a battery model: predicted power to a cell-voltage floor / ceiling (from cell
resistance and SOC), power maps by cell voltage and temperature, a health correction, fault caps, and percentage
derates for low cell voltage, pack voltage and temperature.

---

## 0x55B (100 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_SOC` | 0.7 / 10 | 14 | 0.1 % | state of charge (coulomb counted, corrected towards the OCV estimate); clamp 100.0 %; 0x3FF invalid |
| `LB_ALU_ANSWER` | 2.7 / 8 | 16 | | answer to the VCM's ALU question (0x50C byte 4): the result of a CPU arithmetic self-test run on the question, refreshed every 500 ms |
| `LB_IR_Sensor_Wave_Voltage` | 4.7 / 10 | 46 | ADC counts (× 5000 / 1024 mV) | isolation sensor: mean of 10 samples at the end of the drive-high phase minus mean of 10 at the end of the drive-low phase; 0x3FF invalid |
| `LB_IR_Sensor_Malfunction` | 5.0 / 1 | 40 | | latched isolation sensor self-check failure |
| `LB_Capacity_Empty` | 6.7 / 1 | 55 | | set when the remaining energy stays below a threshold map (380 Wh at every temperature and Hx in the stock calibration) for 60 s; cleared above 1.5 kWh for 60 s. When the energy is invalid: discharge power limit < 2 kW / ≥ 10.01 kW for 60 s. Kept in EEPROM |
| `LB_RefusetoSleep` | 6.5 / 2 | 52 | | 1 refuse to sleep (ignition on, or charge-status frame 0x1F2 still active / not lost), 2 ready to sleep. Note: some DBCs place this at bit 53 |
| `LB_PRUN_55B` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

---

## 0x5BC (100 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Remain_Capacity_GIDS` | 0.7 / 10 | 14 | 1 gid = 80 Wh | remaining energy (gids), clamp 300; 0x3FF invalid (energy invalid or certain faults) |
| `LB_Remaining_Capacity_Segments` | 2.7 / 8 | 16 | bars × 20 | alternates every 5 frames: **capacity bars** (from SOH, decrease-only) when the switch bit is 1, **remaining-charge bars** when it is 0; 0..240 (12 bars), 0xFF invalid |
| `LB_Temperature_Segment_For_Dash` | 3.7 / 8 | 24 | segment × 20 | temperature gauge segment 0..12 (× 20 = 0..240), from a blend of max and min temperature with hysteresis; 0xFF invalid |
| `LB_Capacity_Deterioration_Rate` | 4.7 / 7 | 33 | 1 % | SOH, clamp 100; 100 when invalid |
| `LB_Remain_Cap_Segment_Swit_Flag` | 4.0 / 1 | 32 | | 1 = byte 2 carries capacity bars, 0 = remaining-charge bars |
| `LB_Output_Power_Limit_Reason` | 5.7 / 3 | 45 | | 0 not limited, 1 low cell voltage, 2 fault / other, 3 high temperature (Tmax ≥ 51 °C or modelled ≥ 55.1 °C), 4 low temperature (Tmin < −10 °C) |
| `LB_Remain_Charge_Time_Condition` | 5.1 / 5 | 53 | | which charge time byte 6–7 carries (see below); 0x1F invalid |
| `LB_Remain_Charge_Time` | 6.4 / 13 | 56 | minutes | charge time for that condition; 0x1FFF unavailable |

The condition steps 0 → 5 → 8 → 11 → 18 → 21 → 24 → 0 (held at 0 during a quick charge):

| condition | time to | charger power used |
|---|---|---|
| 0 | quick-charge target | quick-charger power (0x390) |
| 5 / 8 / 11 | 100 % | charger power class C / B / A |
| 18 / 21 / 24 | quick-charge target SOC | class C / B / A |

The power classes come from the on-board charger frame 0x390 (charger mode and reported power, once stable):
**class A** = mode 1 at 0.6–1.5 kW, **class B** = mode 2 at 1.7–3.3 kW, **class C** = mode 2 at 3.5–6.0 kW (the stored
value is the reported power − 100 W, later replaced by the measured power; kept in EEPROM). Modes 1 / 2 are most likely
100 V / 200 V supply (inferred from the power ranges).
Times come from a table of minutes by power class, SOC and temperature; they are refreshed every 88 frames.

---

## 0x5C0 (500 ms)

The history fields are multiplexed by `LB_Historical_Data_Swich_Flag`: each frame steps 1 → 2 → 3 → 1 … (MAX, AVG, MIN);
0 = not calculated (first 500 ms after a reset). The history is restarted at each charge start (normal or quick) and at
each ignition on; it is not stored in EEPROM.

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Historical_Data_Swich_Flag` | 0.7 / 2 | 6 | | mux 0 / 1 max / 2 avg / 3 min |
| `LB_Heating_Start_Send_Request` | 0.5 / 1 | 5 | | battery cold (Tmin below −16 °C, until −10 °C) **and** enough charge to heat (cell OCV ≥ 3862 mV, hysteresis 3917 mV); only while the VCM's 0x50B byte 6 bit 5 is set |
| `LB_Heating_Stop_Send_Request` | 0.4 / 1 | 4 | | battery cold **and** not enough charge to heat (cell OCV below the threshold); only while 0x50B byte 6 bit 5 is set |
| `LB_HistData_HighLow_Volt_Tim` | 0.3 / 4 | 0 | count | mux 1 / 2: +1 per 2 s with the max cell at or above the full-charge voltage; mux 3: +1 per 2 s with the min cell ≤ 2000 mV; clamp 10 |
| `LB_HistData_Temp_WakeupPhase` | 1.7 / 7 | 9 | °C + 40 | max / avg / min temperature at wake-up; 0x7F invalid |
| `Batt_Heater_Mail_Send_Request` | 1.0 / 1 | 8 | | battery cold (start or stop condition true), independent of the 0x50B bit |
| `LB_HistData_Temperature` | 2.7 / 7 | 17 | °C + 40 | max / avg / min temperature now; 0x7F invalid |
| `LB_HistData_IntegratedCurrent` | 3.7 / 8 | 24 | signed, 0.6 Ah | net charge since the history reset, + = charge |
| `LB_HistData_Degr_IntRes_Coeff` | 4.7 / 7 | 33 | % | resistance health (Hx), clamp 100; 0x7F invalid |
| `LB_HEATEXIST` | 4.0 / 1 | 32 | | always 0 in this firmware |
| `LB_HistData_Cell_Voltage` | 5.7 / 6 | 42 | | latched once per history reset. mux 1: (max − min cell) / 5 mV; mux 2 / 3: (avg / min cell − 1900 mV) / 40 mV; clamp 62; 0x3F invalid |
| `LB_NextWakeupTimeForBatterHeater` | 6.4 / 5 | 48 | | from a map of ambient (0x54C) × minimum temperature: 0 when cold … 29 when warm; fixed 6 / 4 / 0x1F in some heater states, 0 while heating. The LBC never uses this value itself, so its time unit is defined by the receiver (the VCM), not by this firmware |
| `LB_Diagnosis_Trouble_Code` | 7.7 / 8 | 56 | | DTC index (position in the 19 xx DTC list) of the two stored freeze frames, alternating every frame |

Note: on this firmware the 5C0 fields sit inside single bytes, so Intel and Motorola readings give the same values.

---

## 0x59E (500 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Full_Capacity_for_QC` | 2.4 / 9 | 28 | 0.1 kWh | energy of a full pack at the current SOH and derates, computed once per wake; clamp 50.0 kWh |
| `LB_Remain_Capacity_for_QC` | 3.3 / 9 | 35 | 0.1 kWh | full energy × usable SOC (while normal / quick charging the smaller of that and full × 0.98); clamp 50.0; 0x1FF invalid |

---

## 0x45D (500 ms)

Not in the common community DBCs.

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| max cell voltage | 0.7 / 9 | 15 | 10 mV | highest cell, clamp 4550 mV; 0x1FF invalid |
| min cell voltage | 1.6 / 9 | 22 | 10 mV | lowest cell; 0x1FF invalid |
| max temperature sensor | 2.5 / 3 | 19 | bits | which of the 3 sensors holds the max temperature (1 / 2 / 4); 7 invalid |
| min temperature sensor | 2.2 / 3 | 16 | bits | which sensor holds the min temperature; 7 invalid |
| max cell number | 3.7 / 8 | 24 | | number 1..96 of the highest cell; 0xFF invalid |
| max temperature | 4.7 / 8 | 32 | °C + 40 | 0xFF invalid |
| min temperature | 5.7 / 8 | 40 | °C + 40 | 0xFF invalid |
| cell overvoltage | 6.7 / 1 | 55 | | an overvoltage fault is confirmed on at least one cell |
| cell undervoltage | 6.6 / 1 | 54 | | an undervoltage fault is confirmed on at least one cell |
| model over-temperature | 6.5 / 1 | 53 | | modelled cell temperature ≥ 63.6 °C (confirmed) |
| over-temperature | 6.4 / 1 | 52 | | measured max temperature ≥ 67 °C (confirmed) |
| interlock sense fault | 6.3 / 1 | 51 | | HV interlock sense input high while the drive output is off (short / stuck high) for 10 checks |
| cell-monitor fault | 6.2 / 1 | 50 | | cell-monitor communication / measurement fault confirmed |
| current-sensor fault | 6.1 / 1 | 49 | | current sensor ADC, current zero (learned zero outside 633..664 counts) or 5 V sensor supply fault confirmed |
| temperature lost | 6.0 / 1 | 48 | | no trustworthy temperature sensor left (all three out of range, or out-of-range and mismatched sensors) |
| constant | 7.7 / 2 | 62 | | always 1 |

---

## Differences from the ZE0 (3NA0D) firmware

| item | ZE0 3NA0D | AZE0 4NC0A |
|---|---|---|
| frames | 1DB 1DC 55B 5BC 5C0 | + 59E and 45D |
| 0x5BC period | 500 ms | 100 ms |
| 0x1DB | no voltage latch, usable SOC or OV limit level; pack voltage from a separate HV sensor | adds `LB_Voltage_Latch_Flag`, `LB_Usable_SOC` and the OV limit level; pack voltage = cell sum |
| 0x5BC | gids, new full capacity, 4-bit bars, average temperature, balancing complete | gids, 8-bit bars × 20, temperature gauge segment, no full capacity / balancing bit |
| 0x5BC charge time conditions | 0 / 9 / 10 / 17 / 18 | 0 / 5 / 8 / 11 / 18 / 21 / 24 |
| 0x5C0 | 8 signals | adds heater requests, `LB_HEATEXIST`, next heater wake-up time |
| 0x55B `LB_ALU_ANSWER` | question − 8 if the CPU self-test signature is correct | result of a CPU ALU self-test run on the question |
