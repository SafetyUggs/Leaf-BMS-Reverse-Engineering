# Nissan Leaf ZE1 (40 kWh) battery controller (LBC): cyclic CAN frames

Firmware: **5SH2D** (ZE1 40 kWh pack, Renesas V850). Everything here was read from the firmware itself: the signal
layouts come from the descriptor tables the firmware packs with, the scaling and limits from the packing code, and
the meanings from the code that computes each value. Signal names follow the community DBC where one matches.

Companion documents: [AZE0_4NC0A_cyclic_CAN_frames.md](../AZE0_24(4NC0A)/AZE0_4NC0A_cyclic_CAN_frames.md) (AZE0
firmware), [ZE0_3NA0D_cyclic_CAN_frames.md](../ZE0(3NA0D)/ZE0_3NA0D_cyclic_CAN_frames.md) (ZE0 firmware) and the
diagnostic services ([ZE1_5SH2D_diagnostic_services.md](ZE1_5SH2D_diagnostic_services.md)).

Conventions:
- **Layout** is given as `byte.bit / length`: the signal's most significant bit is bit `bit` of byte `byte` (bit 7 = MSB
  of the byte), and longer signals continue into the following bytes (Motorola / big-endian order). **LSB** is the
  position of the least significant bit as byte × 8 + bit, the "start bit" most DBC editors show for Motorola signals.
- "invalid" is the value sent when the source is not available. A valid value is clamped to one below the invalid
  value (or lower where stated).
- "inferred" marks a meaning that comes from how a value is used, not from a direct proof.

## Frames

| ID | period | DLC | content | counter / CRC |
|---|---|---|---|---|
| 0x1DB | 10 ms | 8 | current, pack voltage, relay and status bits | PRUN + CRC8 |
| 0x1DC | 10 ms | 8 | power limits, charger power, LB code bytes | PRUN + CRC8 |
| 0x55B | 100 ms | 8 | SOC, self-test flag, isolation sensor, low energy, sleep | PRUN + CRC8 |
| 0x5BC | 100 ms | 8 | gids, bars, temperature gauge, SOH, charge time | none |
| 0x5C0 | 500 ms | 8 | drive-session statistics (6 pages), heater bits | none |
| 0x59E | 500 ms | 8 | full / remaining energy for quick charge, SOC | none |
| 0x5EB | 500 ms | 8 | per-charge-session history (5 slots × 9 pages) | none |
| 0x18CC07BB | event | 7 | newly confirmed DTC (29-bit ID) | none |

All frames are sent only while the LBC is awake with CAN active. Bits not listed for a frame are not written by the
packer. The diagnostic response 0x7BB uses the same CAN controller (see the diagnostic document).

**PRUN** is a 2-bit message counter 0, 1, 2, 3, 0 … (one step per frame). **CRC8** in byte 7 is computed over bytes 0–6.

---

## 0x1DB (10 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Current` | 0.7 / 11 | 13 | signed, 0.5 A | battery current, **+ = charge**; clamp −500 A … +200 A; 0x3FF invalid. Blend of two current sensors (see 21 01 in the diagnostic document) |
| `LB_Relay_Cut_Request` | 1.4 / 2 | 11 | | relay cut request (as 21 03 d17) |
| `LB_Failsafe_Status` | 1.2 / 3 | 8 | bits | failsafe status (as 21 03 d18) |
| `LB_Total_Voltage` | 2.7 / 10 | 30 | 0.5 V | pack voltage = sum of the 96 cells; clamp 450 V; 0x3FF invalid |
| `LB_MainRelayOn_flag` | 3.5 / 1 | 29 | | main relay ON permission |
| `LB_Full_CHARGE_flag` | 3.4 / 1 | 28 | | full charge reached |
| `LB_INTER_LOCK` | 3.3 / 1 | 27 | | 1 = HV interlock loop closed |
| `LB_Discharge_Power_Status` | 3.2 / 2 | 25 | | 1 normal, 2 / 3 limited (which power solver limits discharge); 3 on a fault |
| `LB_PRUN_1DB` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

Bit 3.0, bytes 4–5 and bits 6.7–6.2 are not written: this firmware sends no `LB_Voltage_Latch_Flag`, no
`LB_Usable_SOC` and no cell OV limit level on 0x1DB.

---

## 0x1DC (10 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Discharge_Power_Limit` | 0.7 / 10 | 14 | 0.25 kW | allowed discharge power, clamp 254 kW; 0x3FF invalid |
| `LB_Charge_Power_Limit` | 1.5 / 10 | 20 | 0.25 kW | allowed charge (regen) power, clamp 254 kW; 0x3FF invalid |
| `LB_MAX_POWER_FOR_CHARGER` | 2.3 / 10 | 26 | 0.1 kW, offset −10 kW | power the charger may deliver, 0..1000 (−10 … +90 kW); 0x3FF not set |
| `LB_Charge_Power_Status` | 3.1 / 2 | 24 | | same coding as `LB_Discharge_Power_Status` |
| `LB_BPCMAX_UPRATE` | 4.7 / 3 | 37 | | only two values are ever written: 0 after a charge-end reset, 7 once the charge power control has run |
| `LB_CODE_CONDITION` | 4.4 / 3 | 34 | | index 0..3 of the byte pair currently shown (steps every frame); 7 = not available |
| `LB_CODE1` | 4.1 / 8 | 42 | | byte 2n of the 8-byte record written by diagnostics 3B 90 (read back with 21 90) |
| `LB_CODE2` | 5.1 / 8 | 50 | | byte 2n + 1 of that record |
| `LB_PRUN_1DC` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

---

## 0x55B (100 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_SOC` | 0.7 / 10 | 14 | 0.1 % | state of charge; clamp 100.0 %; 0x3FF invalid |
| self-test NG | 1.0 / 1 | 8 | | set when any of the ADC self-test, the external monitor link, the monitor echo check, or a few related internal checks is NG; cleared when all are OK |
| `LB_IR_Sensor_Wave_Voltage` | 4.7 / 10 | 46 | ADC counts (× 5000 / 1024 mV) | isolation sensor: mean of 10 samples with the drive high minus mean of 10 with the drive low; 0x3FF invalid |
| `LB_IR_Sensor_Malfunction` | 5.0 / 1 | 40 | | latched isolation sensor fault (carried over between cycles) |
| `LB_Capacity_Empty` | 6.7 / 1 | 55 | | low remaining energy: set after 56.5 s with the remaining energy below a threshold map (by average temperature and Hx, in 20 Wh units); cleared after 56.5 s above 1.5 kWh |
| `LB_RefusetoSleep` | 6.5 / 2 | 52 | | **1 sleep allowed, 2 refuse** (wake input high and charge mode 3, or a wake-request status active) |
| `LB_PRUN_55B` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

Bytes 2–3 are not written: there is no `LB_ALU_ANSWER` on this firmware (the CPU self-test is checked by an external
monitor chip over a serial link instead; see 21 12 in the diagnostic document).

---

## 0x5BC (100 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Remain_Capacity_GIDS` | 0.7 / 10 | 14 | 1 gid = 80 Wh | remaining energy (gids), quantised with hysteresis; clamp 1022; 0x3FF invalid |
| `LB_Remaining_Capacity_Segments` | 2.7 / 8 | 16 | bars × 20 | multiplexed over 10 frames: **capacity bars** (from SOH) for 5 frames while the switch bit is 1, **SOC × 240** for 5 frames while it is 0; clamp 240; 0xFF invalid |
| `LB_Temperature_Segment_For_Dash` | 3.7 / 8 | 24 | segment × 20 | temperature gauge segment 0..12 (× 20 = 0..240); 0xFF invalid |
| `LB_Capacity_Deterioration_Rate` | 4.7 / 7 | 33 | 1 % | SOH, clamp 100; 100 when invalid |
| `LB_Remain_Cap_Segment_Swit_Flag` | 4.0 / 1 | 32 | | 1 = byte 2 carries capacity bars, 0 = SOC |
| `LB_Output_Power_Limit_Reason` | 5.7 / 3 | 45 | | reason code, clamp 7 |
| first-pass flag | 5.4 / 1 | 44 | | set on the first gid calculation pass after wake-up (inferred) |
| `LB_Remain_Charge_Time_Condition` | 5.1 / 5 | 53 | | which charge time byte 6–7 carries; clamp 31 |
| `LB_Remain_Charge_Time` | 6.4 / 13 | 56 | minutes | charge time for that condition; 0x1FFF unavailable |

Temperature gauge: the input is a blend of max and min temperature (weight on the minimum 128/128 at ≤ 0 °C, falling
to 0 at ≥ 25 °C). Segment thresholds with hysteresis:

| segment step | 0→1 | 1→2 | 2→3 | 3→4 | 4→5 | 5→6 | 6→7 | 7→8 | 8→9 | 9→10 | 10→11 | 11→12 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| up at (°C) | −24 | −18 | −10 | 2 | 14 | 25 | 36 | 41 | 46 | 50 | 54 | 58 |
| down at (°C) | −26 | −21 | −14 | −6 | 6 | 17 | 28 | 34 | 40 | 46 | 52 | 56 |

The charge time condition codes of this firmware were not traced.

---

## 0x5C0 (500 ms): drive-session statistics

On ZE1 this frame no longer carries the AZE0 max / avg / min history. It carries six pages of statistics of the current
drive, selected by the page field.

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Heating_Start_Send_Request` | 0.5 / 1 | 5 | | always 0 in this firmware |
| `LB_Heating_Stop_Send_Request` | 0.4 / 1 | 4 | | always 0 |
| page | 0.3 / 4 | 0 | | 1..6; 0 = no data (all fields invalid) |
| field A | 1.7 / 7 | 9 | | per page (below); 0x7F invalid, clamp 126 |
| `Batt_Heater_Mail_Send_Request` | 1.0 / 1 | 8 | | always 0 |
| field B | 2.7 / 8 | 16 | | per page; clamp 254 |
| field C | 3.7 / 13 | 35 | | per page; 0x1FFF invalid, clamp 0x1FFE |
| `LB_HEATEXIST` | 4.0 / 1 | 32 | | always 0 |
| field D | 5.7 / 7 | 41 | | per page; 0x7F invalid, clamp 126 |
| field E | 5.0 / 4 | 53 | count | high-cell count (odd pages) or low-cell count (even pages); 0xF invalid, clamp 14 |
| `LB_NextWakeupTimeForBatterHeater` | 6.4 / 5 | 48 | | always 0x1F (invalid): the calculator is never called in this build |

Byte 7 is not written (no `LB_Diagnosis_Trouble_Code`; see 0x18CC07BB for the DTC broadcast).

| page | A (7 bit) | B (8 bit) | C (13 bit) | D (7 bit) | E (4 bit) |
|---|---|---|---|---|---|
| 1 | start Tmax + 40 | start max cell code | mean \|power\|, 0.1 kW | Hx, % | high-cell count |
| 2 | start Tavg + 40 | start avg cell code | best discharge burst > 20 kW, 5 Wh | capacity bar metric, % | low-cell count |
| 3 | start Tmin + 40 | start min cell code | duration of that burst, s | 0 | high-cell count |
| 4 | start max-cell number | now max cell code | output P400.0 on-time, 10 min (always 0, see below) | now max-cell number | low-cell count |
| 5 | start min-cell number | now avg cell code | net Ah, signed (+256 if negative) | now min-cell number | high-cell count |
| 6 | now Tavg + 40 | now min cell code | now Tmax + 40 | now Tmin + 40 | low-cell count |

- Temperatures are °C + 40. Cell numbers are 1..96. "start" = snapshot taken 0.5 s into the drive; "now" = updated
  every 100 ms.
- **Cell code**: ≤ 2000 mV → 0; 2001..3100 mV → (mV − 2000) / 50; 3101..4260 mV → (mV − 3100) / 5 + 22; above → 254.
- **High-cell count**: 2 s periods with the max cell at or above the charge voltage limit (cap 10). **Low-cell
  count**: 2 s periods with the min cell below 2001 mV (cap 10).
- **Burst**: continuous discharge above 20 kW (pack voltage × current); energy = Σ W × 0.1 s / 18000 (5 Wh units).
- The page steps 1 → 2 → … → 6 → 1, one step per frame, only while driving. While charging or with the car off the
  page is 0 and every field is invalid.
- The statistics are reset at each ignition on.
- Output P400.0 is never driven in this firmware, so the on-time field always reads 0.

---

## 0x59E (500 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Full_Capacity_for_QC` | 2.4 / 9 | 28 | 0.1 kWh | energy of a full pack: SOH × 40.749 kWh × usable energy fraction × derate; clamp 50.0 kWh |
| `LB_Remain_Capacity_for_QC` | 3.3 / 9 | 35 | 0.1 kWh | remaining energy for quick charge; clamp 50.0 kWh; 0x1FF invalid |
| SOC | 7.7 / 7 | 57 | 1 % | display SOC, clamp 100; 0x7F invalid (new on ZE1) |

---

## 0x5EB (500 ms): charge-session history

New on ZE1. The LBC records up to five charge (or quick-charge-port) sessions and pages through them.

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| slot | 0.7 / 3 | 5 | | session shown, 1..5; 7 invalid |
| session count | 0.4 / 5 | 0 | | sessions recorded, 0..6; 0x1F invalid |
| sub-page | 1.7 / 4 | 12 | | 0..8 (below) |
| field A | 1.3 / 4 | 8 | | 0xF invalid, clamp 14 |
| field B | 2.7 / 8 | 16 | | clamp 254 |
| field C | 3.7 / 8 | 24 | | clamp 254 |
| field D | 4.7 / 8 | 32 | | clamp 254 |
| field E | 5.7 / 10 | 54 | | 0x3FF invalid, clamp 0x3FE |
| field F | 6.5 / 14 | 56 | | 0x3FFF invalid, clamp 0x3FFE |

| sub | A | B | C | D | E (10 bit) | F (14 bit) |
|---|---|---|---|---|---|---|
| 0 | start month | start year | start day | start hour | start minute | start second |
| 1 | end month | end year | end day | end hour | end minute | end second |
| 2 | 0 | start Tavg + 40 | start Tmax + 40 | start Tmin + 40 | start gids | discharged Ah |
| 3 | 0 | end Tavg + 40 | end Tmax + 40 | end Tmin + 40 | end gids | mean \|power\|, 0.1 kW |
| 4 | session type | start avg cell code | start max cell code | start min cell code | start power | output P400.0 on-time, 10 min (always 0) |
| 5 | 0 | net Ah (+256 if negative) | start max-cell number | start min-cell number | 0 | best discharge burst, 5 Wh |
| 6 | 0 | end avg cell code | end max cell code | end min cell code | 0 | burst duration, s |
| 7 | 0 | end max-cell number | end min-cell number | max SOC % | mean SOC % | min SOC % |
| 8 | 0 | mean Tmax + 40 | highest Tmax + 40 | lowest Tmin + 40 | 0 | 0 |

- Cell codes, temperatures and bursts as in 0x5C0 (bursts here count discharge at any power).
- **Session type**: 0 normal charge on a 0.7..1.2 kW on-board charger class, 1 normal charge on the higher class,
  2 quick charge, 3 session type 4 or 5 from 0x5EC (inferred: power output through the quick-charge port).
- **Start power**: normal charge = first non-zero on-board charger power from 0x390 (0.1 kW); quick charge and type 4/5
  = the charger current limit from 0x3B8 at the start (0.25 A).
- **Clock**: date and time from 0x5CD; year clamp 30, month 1..12, day 1..31.
- **Sessions**: a session starts when 0x5EC reports a charge type (1 normal; 2 / 3 quick; 4 / 5 other). While it runs,
  its slot is rewritten every 100 ms (start values once, "end" values = latest). Only the first five sessions are
  stored; the count stops at 6. The count and all slots are cleared once per wake on the first clear request from
  0x56E. The slots are in RAM only, not in EEPROM.
- **Paging**: the sub-page advances every frame 0..8, then the slot advances 1 … min(count, 5). It only advances while
  driving; while charging or with the car off all fields are invalid.

---

## 0x18CC07BB (29-bit, event)

DLC 7. Sent once each time a broadcast-enabled DTC becomes confirmed.

| signal | layout | LSB | content |
|---|---|---|---|
| frame ID | 0.4 / 29 | 24 | constant 0x18CC07BB (the frame repeats its own ID: bytes `18 CC 07 BB`) |
| DTC | 4.7 / 24 | 48 | 3-byte DTC code (as in service 19) of the newest confirmed broadcast item |

21 of the diagnostic items are broadcast this way. A rising "confirmed" status bit puts that DTC in the frame (one per
10 ms pass) and appends it to an 8-entry list of confirmed DTCs.

---

## Differences from the AZE0 (4NC0A) firmware

| item | AZE0 4NC0A | ZE1 5SH2D |
|---|---|---|
| frames | 1DB 1DC 55B 5BC 5C0 59E 45D | 1DB 1DC 55B 5BC 5C0 59E **5EB 18CC07BB**; no 45D |
| 0x1DB | voltage latch, usable SOC, OV limit level | not sent |
| 0x1DB current clamp | −400 … +200 A | −500 … +200 A |
| 0x55B byte 1 bit 0 / bytes 2–3 | — / `LB_ALU_ANSWER` | self-test NG flag / not sent |
| 0x55B `LB_RefusetoSleep` | 1 refuse, 2 ready to sleep | 1 sleep allowed, 2 refuse |
| 0x5BC gids clamp | 300 | 1022 |
| 0x5BC byte 2 | capacity bars / remaining-charge bars, alternating every 5 frames | capacity bars / SOC × 240, alternating every 5 frames |
| 0x5C0 | max / avg / min history, heater requests, DTC index | six pages of drive statistics; heater bits always 0, no DTC index |
| 0x59E | full / remaining energy | adds SOC in byte 7 |
| DTC on CAN | index in 0x5C0 byte 7 | 3-byte code in 0x18CC07BB when confirmed |
