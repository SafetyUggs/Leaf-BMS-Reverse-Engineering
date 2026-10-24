# Nissan e-NV200 (24 kWh) battery controller (LBC): cyclic CAN frames

Firmware: **4FA2B** (e-NV200 24 kWh pack, Renesas V850). Everything here was read from the firmware itself: the signal
layouts come from the descriptor tables the firmware packs with, and the scaling and limits from the packing code.
Signal names follow the community DBC where one matches.

Companion documents: [ENV200_4FA2B_read_local_id.md](ENV200_4FA2B_read_local_id.md) (diagnostic service 0x21) and, for
the Leaf AZE0, `AZE0_4NC0A_cyclic_CAN_frames.md`. Most frames are packed by the same code as on the AZE0 (4NC0A), so the
AZE0 document also describes how those values are computed. The differences are listed at the end of this file.

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
| 0x5C0 | 500 ms | 8 | history data, thermal request, DTC | none |
| 0x59E | 500 ms | 8 | cooling status, auxiliary temperature, full / remaining energy | none |

There is no 0x45D frame (the AZE0 sends one). Bytes not covered by a signal are 0.

**PRUN** is a 2-bit message counter 0, 1, 2, 3, 0 … (one step per frame). **CRC8** in byte 7 is computed over bytes 0–6.

---

## 0x1DB (10 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Current` | 0.7 / 11 | 13 | signed, 0.5 A | battery current, clamp −400 A … +200 A; 0x3FF invalid |
| `LB_Relay_Cut_Request` | 1.4 / 2 | 11 | | 0 none, 3 = open the main relays |
| `LB_Failsafe_Status` | 1.2 / 3 | 8 | bits | failsafe status bits |
| `LB_Total_Voltage` | 2.7 / 10 | 30 | 0.5 V | pack voltage, clamp 450 V; 0x3FF invalid |
| `LB_MainRelayOn_flag` | 3.5 / 1 | 29 | | main relay ON permission |
| `LB_Full_CHARGE_flag` | 3.4 / 1 | 28 | | full charge |
| `LB_INTER_LOCK` | 3.3 / 1 | 27 | | 1 = HV interlock loop closed |
| `LB_Discharge_Power_Status` | 3.2 / 2 | 25 | | 1 normal, 2 high-rate limit, 3 immediate limit |
| `LB_Voltage_Latch_Flag` | 3.0 / 1 | 24 | | toggles while the cell monitoring runs without failures (heartbeat) |
| `LB_Usable_SOC` | 4.6 / 7 | 32 | 1 % | usable SOC (dash SOC), clamp 100; 0x7F invalid |
| `LB_PRUN_1DB` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

Unlike the AZE0, there is no cell OV limit level at 6.3 / 2.

---

## 0x1DC (10 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Discharge_Power_Limit` | 0.7 / 10 | 14 | 0.25 kW | allowed discharge power, clamp 254 kW; 0x3FF invalid |
| `LB_Charge_Power_Limit` | 1.5 / 10 | 20 | 0.25 kW | allowed charge (regen) power, clamp 254 kW; 0x3FF invalid |
| `LB_MAX_POWER_FOR_CHARGER` | 2.3 / 10 | 26 | 0.1 kW, offset −10 kW | power the charger may deliver, 0..1000 (−10 … +90 kW); 0x3FF not set |
| `LB_Charge_Power_Status` | 3.1 / 2 | 24 | | 1 normal, 2 high-rate limit, 3 immediate limit |
| `LB_BPCMAX_UPRATE` | 4.7 / 3 | 37 | | power uprate level, clamp 7 |
| `LB_CODE_CONDITION` | 4.4 / 3 | 34 | | index of the code pair currently shown; 7 = not available |
| `LB_CODE1` | 4.1 / 8 | 42 | | first byte of code pair n (EEPROM, written by diagnostics 3B 90) |
| `LB_CODE2` | 5.1 / 8 | 50 | | second byte of code pair n |
| `LB_PRUN_1DC` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

---

## 0x55B (100 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_SOC` | 0.7 / 10 | 14 | 0.1 % | state of charge, clamp 100.0 %; 0x3FF invalid |
| `LB_ALU_ANSWER` | 2.7 / 8 | 16 | | answer to the VCM's ALU question (0x50C) |
| `LB_IR_Sensor_Wave_Voltage` | 4.7 / 10 | 46 | ADC counts | isolation sensor wave voltage, clamp 0x3FE; 0x3FF invalid |
| `LB_IR_Sensor_Malfunction` | 5.0 / 1 | 40 | | isolation sensor fault: ADC, stuck-low or flat-response fault confirmed, or a related communication fault |
| `LB_Capacity_Empty` | 6.7 / 1 | 55 | | remaining energy below the empty threshold (kept in EEPROM) |
| `LB_RefusetoSleep` | 6.5 / 2 | 52 | | 1 refuse to sleep, 2 ready to sleep |
| `LB_PRUN_55B` | 6.1 / 2 | 48 | | message counter |
| CRC | 7.7 / 8 | 56 | | CRC8 of bytes 0–6 |

---

## 0x5BC (100 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Remain_Capacity_GIDS` | 0.7 / 10 | 14 | 1 gid = 80 Wh | remaining energy (gids), clamp 300; 0x3FF invalid |
| `LB_Remaining_Capacity_Segments` | 2.7 / 8 | 16 | bars × 20 | capacity bars or remaining-charge bars (see the switch bit); clamp 240; 0xFF invalid |
| `LB_Temperature_Segment_For_Dash` | 3.7 / 8 | 24 | segment × 20 | temperature gauge segment, clamp 240; 0xFF invalid |
| `LB_Capacity_Deterioration_Rate` | 4.7 / 7 | 33 | 1 % | SOH, clamp 100; 100 when invalid |
| `LB_Remain_Cap_Segment_Swit_Flag` | 4.0 / 1 | 32 | | 1 = byte 2 carries capacity bars, 0 = remaining-charge bars |
| `LB_Output_Power_Limit_Reason` | 5.7 / 3 | 45 | | reason the output power is limited, clamp 7 |
| `LB_Remain_Charge_Time_Condition` | 5.1 / 5 | 53 | | which charge time bytes 6–7 carry; clamp 30; 0x1F invalid |
| `LB_Remain_Charge_Time` | 6.4 / 13 | 56 | minutes | charge time for that condition, clamp 0x1FFE; 0x1FFF unavailable |

---

## 0x5C0 (500 ms)

The history fields are multiplexed by `LB_Historical_Data_Swich_Flag` (1 max, 2 avg, 3 min; 0 = not calculated).

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| `LB_Historical_Data_Swich_Flag` | 0.7 / 2 | 6 | | mux 0 / 1 max / 2 avg / 3 min |
| `LB_Heating_Start_Send_Request` | 0.5 / 1 | 5 | | **always 0** in this firmware |
| `LB_Heating_Stop_Send_Request` | 0.4 / 1 | 4 | | **always 0** |
| `LB_HistData_HighLow_Volt_Tim` | 0.3 / 4 | 0 | count | clamp 10 |
| `LB_HistData_Temp_WakeupPhase` | 1.7 / 7 | 9 | °C + 40 | temperature at wake-up; 0x7F invalid |
| `Batt_Heater_Mail_Send_Request` | 1.0 / 1 | 8 | | **always 0** |
| `LB_HistData_Temperature` | 2.7 / 7 | 17 | °C + 40 | temperature now; 0x7F invalid |
| `LB_HistData_IntegratedCurrent` | 3.7 / 8 | 24 | signed | net charge since the history reset |
| `LB_HistData_Degr_IntRes_Coeff` | 4.7 / 7 | 33 | % | resistance health (Hx) or SOH depending on mux, clamp 100; 0x7F invalid |
| `LB_HEATEXIST` | 4.0 / 1 | 32 | | always 0 |
| `LB_HistData_Cell_Voltage` | 5.7 / 6 | 42 | | cell-voltage history value, clamp 62; 0x3F invalid |
| thermal request level | 6.4 / 5 | 48 | | **30** while a hot-charge (cooling) request is active, otherwise 0x1F; see below |
| `LB_Diagnosis_Trouble_Code` | 7.7 / 8 | 56 | | DTC index of the stored freeze frames |

The battery heater logic of the AZE0 is present in the firmware but never called, so the three heater request bits stay
0. Byte 6 carries a fixed level of 30 instead of the AZE0 heater-time map. The level is set while all of these hold:
- the battery is hot while charging (quick charge: max temperature above 19 °C, off at 15 °C; normal charge with charger
  mode 2 from 0x390: max temperature above 24 °C and more than 5 °C above the 0x54C ambient temperature);
- the cell open-circuit voltage is at least 3.0 V (off below 2.6 V);
- the request has been active for less than 180 000 passes.

Reading this as a battery cooling request is inferred from the high-temperature trigger.

---

## 0x59E (500 ms)

| signal | layout | LSB | scaling | content |
|---|---|---|---|---|
| cooling request status | 0.7 / 2 | 6 | | 3 while the hot-charge request above is active (or forced by diagnostics), else 0 |
| auxiliary temperature | 1.7 / 8 | 8 | (°C + 30) / 0.33 | auxiliary NTC temperature, 9-sample average; clamp 0xFE; 0xFF invalid |
| status | 2.7 / 2 | 22 | | 0 or 3 (source not identified) |
| `LB_Full_Capacity_for_QC` | 2.4 / 9 | 28 | 0.1 kWh | energy of a full pack, clamp 50.0 kWh |
| `LB_Remain_Capacity_for_QC` | 3.3 / 9 | 35 | 0.1 kWh | remaining usable energy, clamp 50.0 kWh; 0x1FF invalid |

The auxiliary temperature comes from a fifth thermistor with its own 15-point curve (53 °C … −30 °C), separate from the
four pack temperature sensors. Where it sits on the vehicle is not known (an air inlet or cooling duct is inferred).

---

## Differences from the AZE0 (4NC0A) firmware

| item | AZE0 4NC0A | e-NV200 4FA2B |
|---|---|---|
| frames | 1DB 1DC 55B 5BC 5C0 59E 45D | 1DB 1DC 55B 5BC 5C0 59E (no 45D) |
| 0x1DB | includes the cell OV limit level | no OV limit level |
| 0x1DC / 0x55B / 0x5BC | | same layout and scaling |
| 0x5C0 heater bits | battery heater requests | always 0 (heater code not called) |
| 0x5C0 byte 6 | next heater wake-up time from a map (0..29) | 30 during a hot-charge request, else 0x1F |
| 0x59E | full / remaining energy only | adds cooling request status, auxiliary temperature and a 2-bit status |
