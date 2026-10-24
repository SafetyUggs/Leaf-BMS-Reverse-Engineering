# CAN TX packers: review of the Pass 1/2 (Qwen + AZE0 matching) work (ENV200)

Scope: the six cyclic LBC messages packed by `build_cyclic_can_messages` 00014388 and queued by `send_can_messages` 0002e756:
0x1DB, 0x1DC, 0x55B, 0x5BC, 0x5C0, **0x59E**. Every descriptor table was decoded byte by byte from the image, and every packed
variable was traced to its writer(s). Legend: **[C]** from code, **[I]** inferred. RAM addresses shortened to the last 4 hex digits
(03ffxxxx). Comparison with AZE0 uses the reports in `C:\RE\v850-aze0\reports`.

## Summary of findings

| item | Pass 1/2 state | verdict / fix (logged pass 3) |
|---|---|---|
| `Pack_1DB/1DC/55B/5BC/5C0_Message` | names from AZE0 code match | **correct**; bit positions verified (tables below) |
| `pack_battery_display_fields` 000151ec | "packs voltage current and energy into display buffer" | **wrong**: it is `Pack_59E_Message` (buffer queued as ID 0x59E) |
| `build_sensor_and_status_reports` 00014388 | evidence "not CAN packers" | **wrong**: it calls the six packers → `build_cyclic_can_messages` |
| `init_feature_status` 0002e756 | "stores command data … feature bits" | **wrong**: it queues the six frames by ID → `send_can_messages` |
| `feature_status_flags` d620 | flags byte | **wrong**: 8-byte buffer of 0x59E → `CAN_59E_MSG_Buffer` |
| tx flags c92c..c931 | `can_5bc_tx_pending_flag` on c92d, `can_tx_pending_flag_5` … | c92d is **55B**, not 5BC; all six renamed `CAN_<id>_Tx_Queued_flag` |
| `alu_self_test_result` d549 | "self test" | it is the ALU answer (`update_alu_answer`) → `LB_ALU_Answer` |
| `can_61_byte2_motor_rpm_dc9a` | motor rpm | **wrong**: 5C0 history field (cell voltage slot) → `LB_HistData_Cell_Voltage` |
| `active_dtc_code` dc9b | | 5C0 history DTC → `LB_HistData_DTC` |
| `LOW_heater_timeout_ticks` d5e9 | timeout ticks | **wrong**: a level (30 / 0xFF) sent in 5C0 byte 6 → `Batt_Thermal_Req_Level` |
| `heater_status_code` d5ea | heater | live writer is the hot-charge (cooling) path → `LOW_Batt_Cooling_Req_Status` |
| `Supply_12V_ADC` e32c, `supply_12v_temp_degC` dcd4 | 12 V supply | **wrong**: a 5th NTC thermistor (table 53 → −30 °C) → `Temp_Aux_NTC_ADC` / `_degC` |
| `avg_battery_temperature_x100` dd57 | "x100" | **wrong** scale: s8 °C, 9-sample average of dcd4 → `Temp_Aux_NTC_Avg_degC` |
| `DAT_03ffc929` | unnamed | IR sensor fault flags (bit2 = malfunction summary) → `IR_Sensor_Fault_flags` |
| `DAT_03ffd5f0` | unnamed | thermal state flags → `Batt_Thermal_State_flags` |
| `relay_cut_state`, `lb_code_pair_value/_byte1`, `remain_charge_time_min` | ok but inconsistent | → `LB_Relay_Cut_Request`, `LB_CODE1`, `LB_CODE2`, `LB_Remain_Charge_Time_min` |
| descriptor tables 0x1010c..0x101b7 | unnamed `DAT_/UNK_` | named `CAN_<id>_SigDesc` |
| heater code 0x13fb4..0x14388 | 5 functions missing from export | created with `decompile_at`; all **unreferenced** (see §5C0) |

All other packed variables (`LB_Current_mA`, `LB_SOC_ppm`, `LB_SOH_x100`, power limits, history data, etc.) carry AZE0 names whose
use here matches the packing (scale, clamp, invalid value) [C].

## Packing mechanism [C]

- `set_bits_from_descriptor(desc, buf, value)` 00015c9c: descriptor = 3 bytes {start byte, MSB bit, length}; bits of `value` are written
  MSB-first, moving down through the byte and on to bit 7 of the next byte (Motorola order). It refuses to write unless `buf` and
  `buf+7` lie in SRAM 03ffb096..`sram_end_ptr` and `desc` < 0x60000.
- Quirk: a 0 bit is cleared with `(0xFE << bit)`, which also clears every lower bit of that byte. Harmless here, because every packer
  writes its fields in descending order, so lower fields are rewritten afterwards; unused low bits end up 0.
- `build_cyclic_can_messages`: 1DB and 1DC every call; 55B + 5BC when `Sched_Slot_flags` bit2; 5C0 + 59E when bit3.
- `send_can_messages` queues exactly these six IDs (length 8) through `store_command_data`. **There is no 0x45D** (AZE0 sent 45D with 59E).

## 0x1DB (descriptors `CAN_1DB_SigDesc` 0x1010c, 12 × 3) — byte 7 = CRC8 over bytes 0..6

| # | byte/msb/len | signal | packed value [C] |
|---|---|---|---|
| 0 | 0 / 7 / 11 | current | `LB_Current_mA` dd4c / 500 (**signed** `divh`; the C shows unsigned), clamp −800..+400, signed 11 bit; 0x80000000 → 0x3FF |
| 1 | 1 / 4 / 2 | relay cut request | `LB_Relay_Cut_Request` df46 |
| 2 | 1 / 2 / 3 | failsafe status | `LB_Failsafe_Status` df47 |
| 3 | 2 / 7 / 10 | total voltage 0.5 V | `LB_Total_Voltage_10mV` dd94 / 50 clamp 900; 0xFFFF → 0x3FF |
| 4 | 3 / 5 / 1 | main relay on | `LB_MainRelayOn_flags` df48 bit1 |
| 5 | 3 / 4 / 1 | full charge | `Full_Charge_flags` dc1d bit6 |
| 6 | 3 / 3 / 1 | interlock | `Interlock_Drive_flags` df29 bit2 |
| 7 | 3 / 2 / 2 | discharge power status | `LB_Discharge_Power_Status` dcb3 |
| 8 | 3 / 0 / 1 | cell-monitor heartbeat | `CellMon_Status_flags` d75e bit5 |
| 9 | 4 / 6 / 7 | usable SOC % | `LB_Usable_SOC_Q15` d734 × 100 >> 15 clamp 100; 0xFFFF → 0x7F |
| 10 | 6 / 1 / 2 | PRUN | `LB_PRUN_1DB` d5f4, 0..3 |
| 11 | 7 / 7 / 8 | CRC8 | `calc_crc8(buf, 7)` |

**Difference from AZE0:** AZE0 had 13 signals; the cell-OV limit level (AZE0 byte 6 bits 3..2) is **absent** here. Byte 5 unused in both.

## 0x1DC (`CAN_1DC_SigDesc` 0x10130, 10 × 3) — byte 7 = CRC8. Same layout and scaling as AZE0 [C]

| # | byte/msb/len | signal | packed value [C] |
|---|---|---|---|
| 0 | 0 / 7 / 10 | discharge power limit 0.25 kW | `LB_Discharge_Power_Limit_10W` dcae / 25 clamp 1016; 0xFFFF → 0x3FF |
| 1 | 1 / 5 / 10 | charge power limit 0.25 kW | `LB_Charge_Power_Limit_10W` dcb0 / 25 clamp 1016 |
| 2 | 2 / 3 / 10 | max power for charger 0.1 kW, −10 kW offset | `LB_Max_Power_For_Charger_100W` dc10 + 100 clamp 0..1000; 0x8000 → 0x3FF |
| 3 | 3 / 1 / 2 | charge power status | `LB_Charge_Power_Status` dcb4 |
| 4 | 4 / 7 / 3 | BPCMAX uprate | `LB_BPCMAX_UPRATE` dc18 clamp 7 |
| 5 | 4 / 4 / 3 | code condition | `LB_CODE_CONDITION` df4d, 0xFF → 7 |
| 6 | 4 / 1 / 8 | code 1 | `LB_CODE1` df4e (`rotate_lb_code_pairs`) |
| 7 | 5 / 1 / 8 | code 2 | `LB_CODE2` df4f |
| 8 | 6 / 1 / 2 | PRUN | `LB_PRUN_1DC` d5f5 |
| 9 | 7 / 7 / 8 | CRC8 | |

## 0x55B (`CAN_55B_SigDesc` 0x10150, 8 × 3) — byte 7 = CRC8. Same layout as AZE0 [C]

| # | byte/msb/len | signal | packed value [C] |
|---|---|---|---|
| 0 | 0 / 7 / 10 | SOC 0.1 % | `LB_SOC_ppm` b218 / 1000 clamp 1000; 0xFFFFFFFF → 0x3FF |
| 1 | 2 / 7 / 8 | ALU answer | `LB_ALU_Answer` d549 (`update_alu_answer`) |
| 2 | 4 / 7 / 10 | IR sensor wave voltage | `LB_IR_Sensor_Wave_Voltage` de4e, 0x3FF kept, clamp 0x3FE |
| 3 | 5 / 0 / 1 | IR sensor malfunction | `IR_Sensor_Fault_flags` c929 **bit2** (set by `update_error_flags` 55ec8 when bit3 = IR ADC / stuck / flat fault, or comm fault bit1) |
| 4 | 6 / 7 / 1 | capacity empty | `NV_Status_flags` b768 bit6 |
| 5 | 6 / 5 / 2 | refuse to sleep | `LB_RefusetoSleep` df50 |
| 6 | 6 / 1 / 2 | PRUN | `LB_PRUN_55B` d5f6 |
| 7 | 7 / 7 / 8 | CRC8 | |

Note: AZE0 took the malfunction bit from a different flags byte (bit6); here it is c929 bit2. Same meaning [C for the fault chain].

## 0x5BC (`CAN_5BC_SigDesc` 0x10168, 8 × 3) — no CRC. Same layout as AZE0 [C]

| # | byte/msb/len | signal | packed value [C] |
|---|---|---|---|
| 0 | 0 / 7 / 10 | remaining energy (gids, 80 Wh) | `Remaining_Energy_quant_0p01Wh` d738 / 8000 clamp 300; 0xFFFFFFFF → 0x3FF |
| 1 | 2 / 7 / 8 | segment bars | `LB_Remaining_Segment_bars` d73e clamp 240 (0xFF kept) |
| 2 | 3 / 7 / 8 | temperature gauge | `LB_Temp_Gauge_x20` d73f clamp 240 (0xFF kept) |
| 3 | 4 / 7 / 7 | SOH % | `LB_SOH_x100` b26e / 100 clamp 100; 0xFFFF → 100 |
| 4 | 4 / 0 / 1 | capacity segment switch | `LB_Capacity_Segment_Switch_flags` d746 bit0 |
| 5 | 5 / 7 / 3 | output power limit reason | `LB_Output_Power_Limit_Reason` dcb2 clamp 7 |
| 6 | 5 / 1 / 5 | charge time condition | `LB_Remain_Charge_Time_Condition` dc17 clamp 30; 0xFF → 0x1F |
| 7 | 6 / 4 / 13 | remaining charge time (min) | `LB_Remain_Charge_Time_min` dc12 clamp 0x1FFE; 0xFFFF → 0x1FFF |

## 0x5C0 (`CAN_5C0_SigDesc` 0x10180, 13 × 3) — no CRC. Same layout as AZE0, different thermal sources

| # | byte/msb/len | signal | packed value [C] |
|---|---|---|---|
| 0 | 0 / 7 / 2 | historical data select | `LB_Historical_Data_Select` dc94 |
| 1 | 0 / 5 / 1 | heater request A | `Batt_Thermal_State_flags` d5f0 **bit1** — **always 0** (see below) |
| 2 | 0 / 4 / 1 | heater request B | d5f0 **bit2** — **always 0** |
| 3 | 0 / 3 / 4 | HighLow_Volt_Tim | dc95 clamp 10 |
| 4 | 1 / 7 / 7 | Temp_WakeupPhase | dc96 + 40 (−0x80 → 0x7F, clamp 0..0x7E) |
| 5 | 1 / 0 / 1 | heater request (any) | `Heater_Request_flags` d5f1 **bit3** — **always 0** |
| 6 | 2 / 7 / 7 | Temperature | dc97 + 40 |
| 7 | 3 / 7 / 8 | IntegratedCurrent | dc98, signed 8 bit (`encode_signed_field_sat`) |
| 8 | 4 / 7 / 7 | Degradation_IR | dc99 clamp 100; 0xFF → 0x7F |
| 9 | 4 / 0 / 1 | constant | 0 |
| 10 | 5 / 7 / 6 | Cell_Voltage (history) | `LB_HistData_Cell_Voltage` dc9a clamp 62; 0xFF → 0x3F (`select_hist_data_by_mode` from dc87 / dc88 / dc89 by mode) |
| 11 | 6 / 4 / 5 | thermal request level | `Batt_Thermal_Req_Level` d5e9 clamp 30; 0xFF → 0x1F |
| 12 | 7 / 7 / 8 | Diagnosis_Trouble_Code | `LB_HistData_DTC` dc9b (`select_next_hist_dtc`) |

### Thermal module 0x13834..0x14388 [C mechanics, I purpose]
- `Batt_Thermal_State_flags` d5f0: bit0 = output command, copied to port FFFFF400 bit0 by `config_ports_by_state` and compared with a
  digital input (feedback) in `update_diag_status_flags`; bit3 = cold (Tmin < −16 °C, clear at ≥ −10 °C, as AZE0); bits 4..6 = charge
  state (OCV 2601 / 3000 mV while charging, remaining energy 5.0 / 5.5 kWh otherwise); bit7 = remaining energy < 5.0 kWh (5.5 kWh hysteresis).
- `Heater_Request_flags` d5f1 (AZE0 name kept): bit0 init, bits 1/2 cold request (`set_heater_request_bits`), **bit4 hot-while-charging**
  (`LOW_update_hot_charge_cool_flag`: QC Tmax > 19 °C, off ≤ 15 °C; normal charge with 0x390 mode 2: Tmax > 24 °C and above ambient
  (0x54C) + 5), bits 5/6 OCV state while charging, bit7 timer running.
- **Live path (cooling):** `LOW_update_batt_cooling_outputs` 13f9a → `LOW_set_batt_cooling_req_status` (status 3 when the code d5de = 0xB:
  hot bit4 + OCV bit5 + timer < 180 000 passes; clears d5f0 bit0) → `LOW_set_batt_cooling_req_level` (level **30**, else 0xFF) →
  `clear_heater_request_bits` (clears d5f0 bits 1/2 and d5f1 bit3 every pass).
- **Heater path is dead code:** `LOW_heater_clear_outputs_unused` 13fb4, `LOW_heater_level_invalid_unused` 13fe6,
  `LOW_heater_clear_req_bits_unused` 13ff4, `LOW_heater_drive_output_unused` 141ee (the only code that sets d5f0 bit0, when cold code
  d5dc = 0xB) and `LOW_heater_select_level_unused` 142ea (levels 6 / 4 / 0 / map 43576, the AZE0 `select_heater_level` logic) have no
  callers and no stored pointers (`findptr`). They were absent from the export and were created with `decompile_at`.
- Consequences [C]: 5C0 byte0 bits 5/4 and byte1 bit0 are always 0; port FFFFF400 bit0 is never driven high; 5C0 byte 6 carries only 30
  (hot-charge request active) or 0x1F (none). "Cooling" is inferred from the hot-temperature trigger [I].

## 0x59E (`CAN_59E_SigDesc` 0x101a8, 5 × 3) — no CRC. **3 signals more than AZE0**

| # | byte/msb/len | signal | packed value [C] |
|---|---|---|---|
| 0 | 0 / 7 / 2 | **(new)** cooling request status | `LOW_Batt_Cooling_Req_Status` d5ea: 0 or 3 |
| 1 | 1 / 7 / 8 | **(new)** aux NTC temperature | `Temp_Aux_NTC_Avg_degC` dd57: (T × 100 + 3000) / 33 → ≈ 0.33 °C/bit, −30 °C offset, clamp 0xFE; −0x80 → 0xFF |
| 2 | 2 / 7 / 2 | **(new)** "charge timeout status" | `charge_timeout_status` dc75: 0 or 3 (from `valid_sample_count` e5cf, see open questions) |
| 3 | 2 / 4 / 9 | full usable energy 0.1 kWh | `LB_Full_Energy_0p1kWh` d740 clamp 500 (no invalid check) |
| 4 | 3 / 3 / 9 | remaining usable energy 0.1 kWh | `LB_Remain_Energy_0p1kWh` d742 clamp 500; 0xFFFF → 0x1FF |

Aux NTC chain [C]: `Temp_Aux_NTC_ADC` e32c → `convert_temp_adc_ntc` with its own 15-point table `Tbl_NTC_ADC_Axis` 3e08c /
`Tbl_NTC_Temp_degC_Axis` 3e0ac (ADC 74 → 53 °C … ADC 924 → −30 °C), clamp −30..53 → `Temp_Aux_NTC_degC` dcd4 → history buffer dd38
→ 9-sample average dd57. This is a fifth thermistor, separate from the four pack sensors (which use the 30-point table 3e030).

## Differences from AZE0

| item | AZE0 | ENV200 |
|---|---|---|
| messages | 1DB 1DC 55B 5BC 5C0 59E 45D | 1DB 1DC 55B 5BC 5C0 59E (**no 45D**) |
| 1DB | 13 signals incl. cell-OV limit level | 12 (no OV limit level) |
| 1DC / 55B / 5BC | | same layout and scaling |
| 5C0 heater bits / level | live heater module (map level 0..29) | heater bits always 0; level 30 / 0x1F from a hot-charge request |
| 59E | full + remaining energy | + cooling status, aux NTC temperature, 2-bit status from dc75 |
| port FFFFF400 bit0 | heater output [I] | never set (setter unreferenced) |

## Open questions
1. What drives `charge_timeout_status` dc75 / `valid_sample_count` e5cf (`update_sample_average` 5d9c2 in the digital-input debounce):
   the Qwen names of that chain look generic and were not re-traced.
2. Where the aux NTC sits (air inlet / cooling duct?) and what consumes the 59E thermal fields on the car [I].
3. The task period of the thermal module, to turn the 180 000-pass limit into minutes (30 min if 10 ms).
4. Whether the unreferenced heater functions are reached in another build variant (no pointer to them exists in this image).
5. The temperature naming in `calc_byte_min_max_avg` / `store_history_samples_and_tick` (e.g. `LOW_cell_temp_min_s8` stored into
   `temp_max_hist_buf_s8`) looks inconsistent; not reviewed here.
