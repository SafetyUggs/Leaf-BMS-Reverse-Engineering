# Nissan Leaf ZE0 battery controller (LBC): cyclic CAN frames

Firmware: **3NA0D** (ZE0 24 kWh pack, Renesas V850). Everything here was read from the firmware itself: the signal
layouts come from the descriptor tables the firmware packs with, the scaling and limits from the packing code, and
the meanings from the code that computes each value. Signal names follow the community DBC where one matches.

Companion documents: [AZE0_4NC0A_cyclic_CAN_frames.md](AZE0_4NC0A_cyclic_CAN_frames.md) (AZE0 firmware, with a list of
differences) and the diagnostic services ([ZE0_3NA0D_diagnostic_services.md](ZE0_3NA0D_diagnostic_services.md)).

Conventions:
- **Layout** is given as `byte.bit / length`: the signal's most significant bit is bit `bit` of byte `byte` (bit 7 = MSB
  of the byte), and longer signals continue into the following bytes (Motorola / big-endian order). **LSB** is the
  position of the least significant bit as byte × 8 + bit, the "start bit" most DBC editors show for Motorola signals.
- "invalid" is the value sent when the source is not available.
- "inferred" marks a meaning that comes from how a value is used, not from a direct proof.

## Frames

| ID | period | DLC | content | counter / CRC |
|---|---|---|---|---|
| 0x1DB | 10 ms | 8 | current, pack voltage, relay and status bits | PRUN + CRC8 |
| 0x1DC | 10 ms | 8 | power limits, charger power, LB codes | PRUN + CRC8 |
| 0x55B | 100 ms | 8 | SOC, ALU answer, isolation sensor, capacity empty, sleep | PRUN + CRC8 |
| 0x5BC | 500 ms | 8 | gids, full capacity, bars, temperature, SOH, charge time | none |
| 0x5C0 | 500 ms | 8 | history data, DTC | none |

All frames are sent only while the LBC is awake with CAN active (one 10 ms tick is the base period; 100 and 500 ms are
every 10th / 50th tick). Bytes not covered by a signal are 0.

**PRUN** is a 2-bit message counter 0, 1, 2, 3, 0 … (one step per frame). **CRC8** in byte 7 is computed over bytes 0–6.

---

## 0x1DB (10 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Current` | 0.7 / 11 | 13 | signed, 0.5 A | battery current, **+ = charge**; 100 ms mean; clamp −400 A … +200 A; 0x3FF invalid |
| `LB_Relay_Cut_Request` | 1.4 / 2 | 11 | | 0 none, 3 = open the main relays (latched): cell OV / UV, pack OV / UV, over-temperature (measured or modelled), or a startup check failure |
| `LB_Failsafe_Status` | 1.2 / 3 | 8 | bits | bit0 battery cold (≤ −26 °C); bit1 hardware / self-test / communication faults; bit2 any fault or relay cut |
| `LB_Total_Voltage` | 2.7 / 10 | 30 | 0.5 V | pack voltage from the **separate HV voltage sensor** (LIN), calibrated (see 3B 91); clamp 450 V; 0x3FF invalid |
| `LB_MainRelayOn_flag` | 3.5 / 1 | 29 | | main relay ON permission (set once per wake when the startup checks pass; cleared by a relay cut) |
| `LB_Full_CHARGE_flag` | 3.4 / 1 | 28 | | full charge: normal charge at the 80 % target or charge power tapered below threshold for 2 s; quick charge taper or energy SOC 50 → 90 %; or the max cell reaching the temperature-dependent full voltage |
| `LB_INTER_LOCK` | 3.3 / 1 | 27 | | 1 = HV interlock loop closed (the LBC drives a 300 ms square wave and checks the sense input follows it) |
| `LB_Discharge_Power_Status` | 3.2 / 2 | 25 | | 1 normal, 2 high-rate limit (demand above the power that pulls the lowest cell to 2.35 V), 3 immediate limit (above the 2.20 V power, or a serious fault) |
| `LB_PRUN_1DB` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

---

## 0x1DC (10 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Discharge_Power_Limit` | 0.7 / 10 | 14 | 0.25 kW | allowed discharge power (predicted power to 2.5 V per cell, power map, health, fault caps, derates), clamp 254 kW; 0x3FF invalid |
| `LB_Charge_Power_Limit` | 1.5 / 10 | 20 | 0.25 kW | allowed charge (regen) power, max 70 kW; 0x3FF invalid |
| `LB_MAX_POWER_FOR_CHARGER` | 2.3 / 10 | 26 | 0.1 kW, offset −10 kW | power the charger may deliver during a charge session (keeps the max cell below the full voltage, with thermal derate); −10 kW when full; 0x3FF not set |
| `LB_Charge_Power_Status` | 3.1 / 2 | 24 | | 1 normal, 2 charge above the power to the full voltage, 3 above the power to 4.20 V (or fault) |
| `LB_BPCMAX_UPRATE` | 4.7 / 3 | 37 | | 7 during a charge session, 0 otherwise |
| `LB_CODE_CONDITION` | 4.4 / 3 | 34 | | index 0..3 of the code pair currently shown (steps every frame) |
| `LB_CODE1` | 4.1 / 8 | 42 | | first byte of code pair n (stored in EEPROM, written by diagnostics 3B 90; not DTCs) |
| `LB_CODE2` | 5.1 / 8 | 50 | | second byte of code pair n |
| `LB_PRUN_1DC` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

---

## 0x55B (100 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_SOC` | 0.7 / 10 | 14 | 0.1 % | state of charge (coulomb counted, seeded at wake, slewed towards the OCV estimate); clamp 100.0 %; 0x3FF invalid |
| `LB_ALU_ANSWER` | 2.7 / 8 | 16 | | the VCM's question (0x50C byte 4) − 8 if the CPU self-test signature is correct, else the question unchanged |
| `LB_IR_Sensor_Wave_Voltage` | 4.7 / 10 | 46 | ADC counts (× 5000 / 1024 mV) | isolation sensor: mean of 10 samples at the end of the drive-high phase minus mean of 10 at the end of the drive-low phase; 0x3FF invalid |
| `LB_IR_Sensor_Malfunction` | 5.0 / 1 | 40 | | latched result of the isolation sensor self-checks (ADC stuck high / low, flat response, slow wave) |
| `LB_Capacity_Empty` | 6.7 / 1 | 55 | | set when the remaining energy stays below 380 Wh for 60 s, cleared above 1.5 kWh for 60 s (fallback: discharge power limit < 2 kW / ≥ 10 kW) |
| `LB_RefusetoSleep` | 6.5 / 2 | 52 | | 1 refuse to sleep, 2 ready to sleep (ignition input off and 0x1F2 stop request, or 0x1F2 lost for 2 s). Note: some DBCs place this at bit 53 |
| `LB_PRUN_55B` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

---

## 0x5BC (500 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Remain_Capacity_GIDS` | 0.7 / 10 | 14 | 1 gid = 80 Wh | remaining energy (SOH × 23 616 Wh × energy fraction from cell voltage, with gid hysteresis; separate path during quick charge), clamp 300; 0x3FF invalid |
| `LB_New_Full_Capacity` | 1.5 / 10 | 20 | gids − 250 | a "full capacity" estimate 250..300 gids, sent as value − 250 (0..50). After a cold start it starts at 295 and follows the remaining energy down, so it is not a true capacity in that case |
| `LB_Remaining_Capacity_Segments` | 2.3 / 4 | 16 | bars | alternates every 5 frames: **capacity bars** (from the filtered SOH, 6.25 % per bar, decrease-only) when the switch bit is 1, **charge bars** (8 % per bar with ±4 % hysteresis) when 0; clamp 12, 15 invalid |
| `LB_Average_Battery_Temperature` | 3.7 / 8 | 24 | °C + 40 | 10-sample mean of (Tmax + Tmin) / 2 over the 4 sensors; clamp 0..145 (−40 … 105 °C); 0xFF invalid |
| `LB_Capacity_Deterioration_Rate` | 4.7 / 7 | 33 | 1 % | SOH, clamp 100; 100 when invalid |
| `LB_Remain_Cap_Segment_Swit_Flag` | 4.0 / 1 | 32 | | 1 = byte 2 carries capacity bars, 0 = charge bars |
| `LB_Output_Power_Limit_Reason` | 5.7 / 3 | 45 | | 0 not limited, 1 low cell voltage, 2 fault / other, 3 high temperature (Tmax ≥ 51 °C or modelled ≥ 55.1 °C), 4 low temperature (Tmin < −10 °C) |
| capacity balancing complete | 5.2 / 1 | 42 | | no group needs balancing, or the post-key-off balancing timer is 0; that timer is never loaded in this firmware, so the bit is always 1 |
| `LB_Remain_Charge_Time_Condition` | 5.1 / 5 | 53 | | which charge time bytes 6–7 carry (see below); 0x1F invalid |
| `LB_Remain_Charge_Time` | 6.4 / 13 | 56 | minutes | time for that condition; 0x1FFF unavailable |

The condition steps 0 → 9 → 10 → 17 → 18 → 0, five calls each:

| condition | time to | charger power |
|---|---|---|
| 0 | quick-charge target (90 % or 100 %) | quick-charger power (CAN 0x380), only during a quick charge |
| 9 / 10 | 100 % / 80 % | high class: AC supply in the upper band (200 V) at 1.4–3.3 kW (default 3.0 kW, then measured) |
| 17 / 18 | 100 % / 80 % | low class: AC supply in the lower band (100 V) at 0.7–1.2 kW (default 1.0 kW, then measured) |

The class is chosen from the charger frame 0x380 once its power has been stable for 100 passes (and 300 passes into a
normal charge): AC voltage field (9 bits + 140, raw 180–319 = low class, 320–527 = high class) and charge power. At the
0.5 V per bit the field appears to use, the bands are 90–160 V and 160–264 V, i.e. 100 V and 200 V mains. The stored
class power is the reported power − 100 W, later replaced by the measured power. Normal charge times are a table of minutes by power
class and SOC, scaled by SOH, smoothed and rounded up (10 min below 1 h, 30 min below 10 h, 60 min above).

---

## 0x5C0 (500 ms)

The history fields are multiplexed by `LB_Historical_Data_Swich_Flag`: each frame steps 1 → 2 → 3 → 1 … (MAX, AVG, MIN);
0 = not calculated (first 500 ms after a reset). The history is restarted at each charge start (normal or quick) and at
each ignition on, and is not stored in EEPROM.

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Historical_Data_Swich_Flag` | 0.7 / 2 | 6 | | mux 0 / 1 max / 2 avg / 3 min |
| `LB_HistData_HighLow_Volt_Tim` | 0.3 / 4 | 0 | count | mux 1 / 2: +1 per 2 s with the max cell at the full voltage; mux 3: +1 per 2 s with the min cell ≤ 2000 mV; clamp 10 |
| `LB_HistData_Temp_WakeupPhase` | 1.7 / 7 | 9 | °C + 40 | Tmax / Tavg / Tmin latched once after the reset; 0x7F invalid |
| `LB_HistData_Temperature` | 2.7 / 7 | 17 | °C + 40 | Tmax / Tavg / Tmin now; 0x7F invalid |
| `LB_HistData_IntegratedCurrent` | 3.7 / 8 | 24 | signed, 0.6 Ah | net charge since the reset, + = charge |
| `LB_HistData_Degr_IntRes_Coeff` | 4.7 / 7 | 33 | % | resistance health (Hx), clamp 100, same for every mux value; 0x7F invalid |
| `LB_HistData_Cell_Voltage` | 5.7 / 6 | 42 | | latched once after the reset. mux 1: (max − min cell) / 5 mV; mux 2 / 3: (avg / min cell − 1900 mV) / 40 mV; clamp 62; 0x3F invalid |
| `LB_Diagnosis_Trouble_Code` | 7.7 / 8 | 56 | | DTC index (position in the 19 xx DTC list) of the two stored freeze frames, alternating every frame |
