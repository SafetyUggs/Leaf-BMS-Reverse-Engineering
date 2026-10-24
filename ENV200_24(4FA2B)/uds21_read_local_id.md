# UDS / KWP 0x21 ReadDataByLocalIdentifier: full response map

Firmware: 4FA2B application (V850). Source of truth: the decompiled handlers in `export/`,
names as logged in `analysis/function_log.csv` / `globals_log.csv` (pass 3).

## How the service works

| Layer | Address | Name | What it does |
|---|---|---|---|
| Service dispatcher | 0002664a | `CAN_service_dispatch` | `CAN_RX_Buffer[0] == 0x21` -> `CAN_21`. No session or security check for 0x21 at this level. |
| 0x21 dispatcher | 00059de2 | `CAN_21` | Request length must be exactly 2 (`21 xx`), else NRC `7F 21 12`. Switches on `uds_sub_function_id` (03ffc453 = request byte 1), calls the handler, sets `Diag21_Response_Length` (03ffe2e0), then `diag_send_response(CAN_TX_Buffer, len)`. Unknown IDs -> `7F 21 12`. |
| Response buffer | 03ffc4f0 | `CAN_TX_Buffer` | Shared response image. Byte 0 = `0x61`, byte 1 = the local ID, data from byte 2. |

Conventions in every table below:

- **Byte** = offset in the positive response, counting `0x61` as byte 0 and the local ID as byte 1.
  (Drop 2 if your tool strips the `61 xx` header.)
- All multi-byte values are **big-endian** (MSB first). The firmware copies the high byte of
  each little-endian SRAM variable first.
- `FF` = constant filler written by the handler (reserved/unused field).
- `Len` = total response length the dispatcher sends, including `61 xx`.
- SRAM addresses are given so each field can be traced; names are the current project names.
- Sign: s8/s16/s32 where the producing code is known to be signed, otherwise u.

### Security

No application 0x21 local ID checks session or security access: not the dispatcher, and
not any of the handlers below (all were read in full). None needs elevated access.
Security code (`CAN_27` and everything in `analysis/security_excluded.txt`) was not opened.
The only security-related 0x21 code is the **bootloader's** F0/F1 (see the end of this report).

## Supported local IDs

The ECU answers these IDs (dispatcher switch):

| LID | Len | Handler | Content (short) | Advertised in bitmap |
|---|---|---|---|---|
| 00, 20, 40, 60 | 6 | 0005a190 `CAN_21_supported_bitmap` | supported-ID bitmap pages | yes |
| 01 | 41 | 0005a1de `CAN_21_01` | current, power limits, voltages, Hx, SOC, capacity | yes |
| 02 | 198 | 0005a4e4 `CAN_21_02` | 96 cell voltages, pack and inverter voltage | yes |
| 03 | 28 | 0005a5a2 `CAN_21_03` | ADCs, SOC init, cell max/min, relay/failsafe, torque/speed | yes |
| 04 | 37 | 0005a7d4 `CAN_21_04` | 4 NTCs + aux NTC, min/max/filtered temperatures | yes |
| 05 | 71 | 0005aa1c `CAN_21_05` | raw ADCs, current, insulation (IR) samples, wake inputs | yes |
| 06 | 26 | 0005acd0 `CAN_21_06` | balancing bleed status, 24 groups | yes |
| 07 | 10 | 0005ad06 `CAN_21_07` | current-sensor zero offset | yes |
| 08 | 5 | 0005adaa `CAN_21_08` | NV block status | yes |
| 09 | 7 | 0005adf0 `CAN_21_09` | weak cell / dSOC ratios | yes |
| 0A | 6 | 0005ae5a `CAN_21_0A` | heater/charge request level, sampled input | yes |
| 0B | 89 | 0005aea6 `CAN_21_0B` | LBC status mirror, history select, CAN RX snapshots | yes |
| 0C | 31 | 0005b602 `CAN_21_0C` | thermal/heater/cooling state, temp-delta check | yes |
| 10 | 5 | 0005b9f8 `CAN_21_10` | 2 status bits | yes |
| 61 | 12 | 0005ba7c `CAN_21_61` | SOH, Hx, capacity bars | yes |
| 62 | 118 | 0005bb6e `CAN_21_62` | charge counters + 7 usage histograms | yes |
| 63 | 4 | 0005bdfa `CAN_21_63` | diagnostic NV word | yes |
| 64 | 3 | 0005be36 `CAN_21_64` | 5 V fault wake count | yes |
| 82 | 82 | 0005be5e `CAN_21_82` | communication-DTC image | no |
| 83 | 26 | 0005be80 `CAN_21_83` | application part/software ID (constants) | no |
| 84 | 22 | 0005bf6a `CAN_21_84` | NV ID string | no |
| 90 | 10 | 0005bfd4 `CAN_21_90` | LB code store | no |
| 91 | 12 | 0005c00a `CAN_21_91` | 5 diag calibration words | no |
| 92 | 98 | 0005c080 `CAN_21_92` | 96 per-cell voltage trims | no |
| 94 | 3 | 0005c0b8 `CAN_21_94` | capacity balancing complete | no |
| 95 | 4 | 0005c0ec `CAN_21_95` | cell-monitor fail confirmation bits | no |
| F0, F1 | 26 | 0005c128 `CAN_21_F0_F1` | ID blocks from flash | no |
| FE, FF | 26 | 0005c18e `CAN_21_FEFF` | ID blocks from EEPROM (flash fallback) | no |

Any other LID (e.g. 0D-0F, 11-1F, 65-81, 85-8F, 93, 96-EF, F2-FD) -> `7F 21 12`.

The advertised bitmap (`Tbl_CAN_Feature_Bitmap` @ 0003f724, 16 bytes) is
`FF F1 00 01 | 00 00 00 01 | 00 00 00 01 | F0 00 00 00`, which advertises
01-0C, 10, 20 (next page), 40, 60, 61-64. Everything from 0x82 upward is answered but
not advertised (workshop/supplier IDs).

---

## 0x00 / 0x20 / 0x40 / 0x60: supported-ID bitmap (Len 6)

Handler `CAN_21_supported_bitmap` (0005a190). Copies 4 bytes from
`Tbl_CAN_Feature_Bitmap + (LID >> 3)`.

| Byte | Size | Content |
|---|---|---|
| 2-5 | 4 | bitmap for LIDs (LID+1) .. (LID+0x20); MSB of byte 2 = LID+1, LSB of byte 5 = next page present |

## 0x01: main battery state (Len 41 = 0x29)

Handler `CAN_21_01` (0005a1de).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-5 | 32 | `Current_Corrected_mcounts` (03ffdd48) | corrected current, sensor counts x1000 |
| 6-7 | u16 | `Current_Zero_Learned_ADC` (03ffdc58) | learned current-sensor zero, ADC counts |
| 8-11 | 32 | `LB_Current_mA` (03ffdd4c) | pack current, mA |
| 12-15 | - | `FF FF FF FF` | filler |
| 16-17 | u16 | `LB_Charge_Power_Limit_10W` (03ffdcb0) | charge power limit, 10 W |
| 18-19 | u16 | `LB_Discharge_Power_Limit_10W` (03ffdcae) | discharge power limit, 10 W |
| 20-21 | u16 | `Pack_Voltage_CellSum_10mV` (03ffdd6c) | sum of cell voltages, 10 mV |
| 22-23 | u16 | `Supply_Voltage_RefCorr_mV` (03ffdf34) | 12 V supply, reference-corrected, mV |
| 24-25 | u16 | `LB_IR_Sensor_Wave_Voltage` (03ffde4e) | insulation (IR) sensor wave voltage |
| 26-27 | u16 | `Pack_Resistance_Regr_mOhm` (03ffdebc) | pack resistance from regression, mOhm |
| 28-29 | u16 | `Hx_x100` (03ffb274, high byte 03ffb275) | Hx, %x100 |
| 30-33 | u32 | `LB_SOC_ppm` (03ffb218) | SOC, ppm (1e6 = 100 %) |
| 34-37 | u32 | `Full_Capacity_0p1mAh` (03ffb220) | full capacity, 0.1 mAh |
| 38-39 | u16 | `LB_Max_Power_For_Charger_100W` (03ffdc10) | max power for on-board charger, 100 W |
| 40 | bits | flags | bit0 = `Interlock_Drive_flags` (03ffdf29) bit2; bit1 = `Full_Charge_flags` (03ffdc1d) bit6; bit2 = `Digital_Input_State_flags` (03ffe5cd) bit2; bits3-7 = 0 |

## 0x02: cell voltages (Len 198 = 0xC6)

Handler `CAN_21_02` (0005a4e4).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-193 | 96 x u16 | `Cell_Voltage_mV[0..95]` (03ffd76c) | cell n at byte 2+2n, mV |
| 194-195 | u16 | `LB_Total_Voltage_10mV` (03ffdd94) | pack voltage, 10 mV |
| 196-197 | u16 | `CAN_1DA_Inverter_Voltage_10mV` (03ffd630) | inverter voltage from CAN 0x1DA, 10 mV (byte x200), FFFF = invalid |

## 0x03: sensors, cell max/min, relay and fault status (Len 28 = 0x1C)

Handler `CAN_21_03` (0005a5a2).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-3 | - | `FF FF` | filler |
| 4-5 | u16 | `Current_Sensor_ADC` (03ffe318) | 10-bit raw ADC |
| 6-7 | - | `FF FF` | filler |
| 8-11 | u32 | `SOC_Init_ppm` (03ffd6e0) | SOC at initialisation, ppm |
| 12-13 | u16 | `Cell_Voltage_Max_mV` (03ffdd6e) | highest cell, mV |
| 14-15 | u16 | `Cell_Voltage_Min_mV` (03ffdd70) | lowest cell, mV |
| 16 | u8 | `Supply_5V_Fault_Wake_Count` (03ffb2a6) | count |
| 17-18 | u16 | `IR_Sensor_ADC` (03ffe31a) | 10-bit raw ADC, insulation sensor |
| 19 | u8 | `LB_Relay_Cut_Request` (03ffdf46) | relay cut request (as sent in 0x1DB) |
| 20 | u8 | `LB_Failsafe_Status` (03ffdf47) | failsafe status |
| 21-22 | u16 | `Fault_Summary_flags` (03ffb776) | fault summary bit field |
| 23-24 | s16 | `CAN_1DA_Motor_Torque_Nm` (03ffd632) | motor torque from 0x1DA, Nm, clamp +-274, 0x8000 = invalid |
| 25-26 | s16 | `CAN_1DA_Motor_Speed_rpm` (03ffd634) | motor speed from 0x1DA, clamp +-16383, 0x8000 = invalid |
| 27 | bits | flags | bit0 = `LB_MainRelayOn_flags` (03ffdf48) bit1; bit1 = `LB_VCM_Status_flags` (03ffd628) bit1; bit2 = `IR_Sensor_Fault_flags` (03ffc929) bit2; bit3 = `LB_VCM_Status_flags` bit0 |

## 0x04: temperatures (Len 37 = 0x25)

Handler `CAN_21_04` (0005a7d4). ADC values are 10-bit (0..0x3FF); degC values are s8
from the NTC table, clamped -40..105.

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-3 | u16 | `Temp_Sensor1_ADC` (03ffe31c) | NTC 1 raw ADC |
| 4 | s8 | `Temp_Sensor1_degC` (03ffdd53) | NTC 1, degC |
| 5-6 | u16 | `Temp_Sensor2_ADC` (03ffe31e) | NTC 2 raw ADC |
| 7 | s8 | `Temp_Sensor2_degC` (03ffdd54) | NTC 2, degC |
| 8-9 | u16 | `Temp_Sensor3_ADC` (03ffe320) | NTC 3 raw ADC |
| 10 | s8 | `Temp_Sensor3_degC` (03ffdd55) | NTC 3, degC |
| 11-12 | u16 | `Temp_Sensor4_ADC` (03ffe322) | NTC 4 raw ADC |
| 13 | s8 | `Temp_Sensor4_NTC_degC` (03ffdd56) | NTC 4, degC |
| 14 | s8 | `Temp_Min_Avg_degC` (03ffdd50) | min pack temperature, degC |
| 15 | u8 | `Vehicle_Speed_kmh` (03ffd636) | vehicle speed from 0x1DA motor rpm, km/h (0..127) |
| 16-17 | s16 | `Temp_Min_Filt_x200` (03ffdd5a) | filtered min temp, degC x200 (-8000..21000) |
| 18-21 | s32 | `Temp_Min_Filt_state` (03ffdd68) | filter state, degC x204800 |
| 22-23 | s16 | `Temp_Max_Filt_x200` (03ffdd58) | filtered max temp, degC x200 |
| 24-27 | s32 | `Temp_Max_Filt_state` (03ffdd64) | filter state, degC x204800 |
| 28 | s8 | `Temp_Max_Avg_degC` (03ffdd51) | max pack temperature, degC |
| 29-30 | - | `FF FF` | filler |
| 31-32 | u16 | `Temp_Aux_NTC_ADC` (03ffe32c) | auxiliary NTC raw ADC |
| 33 | s8 | `Temp_Aux_NTC_Avg_degC` (03ffdd57) | auxiliary NTC, 9-sample average, degC, 0x80 = invalid |
| 34-36 | - | `FF FF FF` | filler |

## 0x05: raw ADC, current and insulation samples (Len 71 = 0x47)

Handler `CAN_21_05` (0005aa1c).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-3 | u16 | `adc_channel_ref_raw_10bit` (03ffe324) | raw ADC (I/O fffff222), used by supply-voltage calc |
| 4-5 | u16 | `adc_channel_5_raw_10bit` (03ffe326) | raw ADC (fffff226) |
| 6-7 | u16 | `adc_channel_6_raw_10bit` (03ffe328) | raw ADC (fffff228) |
| 8-11 | - | `FF FF FF FF` | filler |
| 12-13 | u16 | `Temp_Sensor1_ADC` (03ffe31c) | |
| 14-15 | u16 | `Temp_Sensor2_ADC` (03ffe31e) | |
| 16-17 | u16 | `Temp_Sensor3_ADC` (03ffe320) | |
| 18-19 | u16 | `Temp_Sensor4_ADC` (03ffe322) | |
| 20-21 | u16 | `Current_Sensor_ADC` (03ffe318) | |
| 22-25 | 32 | `LB_Current_mA` (03ffdd4c) | mA |
| 26-27 | u16 | `LB_IR_Sensor_Wave_Voltage` (03ffde4e) | |
| 28-47 | 10 x u16 | `IR_Samples_Low[0..9]` (03ffde64) | insulation sensor samples, entry n at 28+2n |
| 48-67 | 10 x u16 | `IR_Samples_High[0..9]` (03ffde50) | entry n at 48+2n |
| 68 | u8 | `Wake_Input_Any_55AA` (03ffb78b) | 0x55/0xAA style state byte |
| 69 | u8 | `Wake_Input2_Debounced_55AA` (03ffb789) | 0x55/0xAA style state byte |
| 70 | bits | flags | bit0 = `Interlock_Drive_flags` bit0; bit1 = `Digital_Input_State_flags` bit3 |

## 0x06: cell balancing (bleed) status (Len 26 = 0x1A)

Handler `CAN_21_06` (0005acd0).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-25 | 24 x u8 | `CellMon_Bleed_Status_Group[0..23]` (03ffdac0) | one byte per balancing group (4 cells each); byte 2+g = group g |

## 0x07: current-sensor zero offset (Len 10)

Handler `CAN_21_07` (0005ad06).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-3 | - | `FF FF` | filler |
| 4-7 | 32 | `Current_Zero_Offset_ucounts` (03ffdc5c) | current zero offset, micro-counts (written by zero calibration) |
| 8-9 | u16 | `adc_channel_5_raw_10bit` (03ffe326) | raw ADC (also used for current gain ratio) |

## 0x08: NV block status (Len 5)

Handler `CAN_21_08` (0005adaa).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2 | u8 | `nv_block_source_state` (03ffe579) | which NV copy was loaded |
| 3 | u8 | `NV_BlockB_Invalid` (03ffe57a) | NV block B invalid flag |
| 4 | bit | `Meas_Invalid_flags` (03ffdc6d) bit0 | 0/1 |

## 0x09: weak cell / dSOC ratio (Len 7)

Handler `CAN_21_09` (0005adf0).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2 | u8 | `Cell_dSOC_Ratio_Peak_x64` (03ffb792) | x64 |
| 3 | u8 | `Cell_dSOC_Ratio_MaxCell_x64` (03ffdf22) | x64 |
| 4 | u8 | `Cell_dSOC_Ratio_MinCell_x64` (03ffdf23) | x64 |
| 5 | u8 | `Weak_Module_MaxCell` (03ffb791) | cell index |
| 6 | u8 | `Weak_Module_MinCell` (03ffb790) | cell index |

## 0x0A: heater/charge request level and sampled input (Len 6)

Handler `CAN_21_0A` (0005ae5a).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2 | - | `FF` | filler |
| 3 | u8 | `charge_timeout_value` (03ffdc74) | 0, 40 or 100: set by heater state / 0x1F2 charge request / VCM status with a 15000 x 10 ms timer |
| 4 | u8 | `sample_avg_of_10` (03ffe5cf) | mean of the last 10 valid samples (0 until 10 collected) |
| 5 | u8 | `LOW_sample_latest_x10` (03ffe5ce) | latest sample = event count per 10 cycles x10, clamped 255 |

Open question: the event counter feeding 0x0A bytes 4-5 (`sample_avg_index` 03ffe5ac) is
incremented by code not in the export (probably an interrupt), so what it counts is unknown.

## 0x0B: Li-ion battery status mirror (LBC CAN fields, history select, CAN RX snapshots) (Len 89 = 0x59)

Handler `CAN_21_0B` (0005aea6). Most fields are the same values the BMS transmits on CAN
(`LB_*` = values for the LBC frames; `CAN_xxx_*` = last value received in that frame).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2 | u8 | `LB_Discharge_Power_Status` (03ffdcb3) | |
| 3 | u8 | `LB_Charge_Power_Status` (03ffdcb4) | |
| 4 | u8 | `LB_BPCMAX_UPRATE` (03ffdc18) | |
| 5 | u8 | `LB_CODE_CONDITION` (03ffdf4d) | |
| 6 | u8 | `LB_CODE1` (03ffdf4e) | |
| 7 | u8 | `LB_CODE2` (03ffdf4f) | |
| 8 | u8 | `LB_ALU_Answer` (03ffd549) | answer to `CAN_50C_ALU_Question` |
| 9 | u8 | `LB_RefusetoSleep` (03ffdf50) | |
| 10-13 | u32 | `Remaining_Energy_quant_0p01Wh` (03ffd738) | remaining energy, 0.01 Wh |
| 14 | u8 | `LB_Remaining_Segment_bars` (03ffd73e) | capacity bars |
| 15 | u8 | `LB_Temp_Gauge_x20` (03ffd73f) | temperature gauge |
| 16 | u8 | `LB_Output_Power_Limit_Reason` (03ffdcb2) | |
| 17 | u8 | `LB_Remain_Charge_Time_Condition` (03ffdc17) | |
| 18-19 | u16 | `LB_Remain_Charge_Time_min` (03ffdc12) | minutes |
| 20 | u8 | `LB_Historical_Data_Select` (03ffdc94) | |
| 21 | u8 | `LB_HistData_HighLow_Volt_Tim` (03ffdc95) | |
| 22 | u8 | `LB_HistData_Temp_WakeupPhase` (03ffdc96) | |
| 23 | u8 | `LB_HistData_Temperature` (03ffdc97) | |
| 24 | u8 | `LB_HistData_IntegratedCurrent` (03ffdc98) | |
| 25 | u8 | `LB_HistData_Degradation_IR` (03ffdc99) | |
| 26 | u8 | `LB_HistData_Cell_Voltage` (03ffdc9a) | |
| 27 | u8 | `Batt_Thermal_Req_Level` (03ffd5e9) | |
| 28 | u8 | `LB_HistData_DTC` (03ffdc9b) | |
| 29-30 | u16 | `LB_Full_Energy_0p1kWh` (03ffd740) | 0.1 kWh |
| 31-32 | u16 | `LB_Remain_Energy_0p1kWh` (03ffd742) | 0.1 kWh |
| 33 | u8 | `CAN_1D4_PRUN` (03ffe4d9) | RX 0x1D4 rolling counter |
| 34 | u8 | `CAN_1D4_CRC` (03ffe4dd) | RX 0x1D4 CRC |
| 35 | u8 | `CAN_50B_Network_State` (03ffd62a) | RX 0x50B |
| 36 | u8 | `CAN_50C_PRUN` (03ffe4da) | RX 0x50C rolling counter |
| 37 | u8 | `CAN_50C_ALU_Question` (03ffd62b) | RX 0x50C |
| 38 | u8 | `CAN_50C_CRC` (03ffe4de) | RX 0x50C CRC |
| 39 | u8 | `CAN_1F2_Charge_Status_Req` (03ffd62c) | RX 0x1F2 |
| 40 | u8 | `CAN_1F2_PRUN` (03ffe4db) | RX 0x1F2 rolling counter |
| 41 | u8 | `CAN_390_Charger_Mode` (03ffd62d) | RX 0x390 |
| 42-43 | u16 | `CAN_390_Charger_Power_100W` (03ffd62e) | RX 0x390, 100 W |
| 44 | u8 | `CAN_1DA_PRUN` (03ffe4dc) | RX 0x1DA rolling counter |
| 45 | u8 | `CAN_1DA_CRC` (03ffe4df) | RX 0x1DA CRC |
| 46 | s8 | `CAN_54C_Ambient_Temp_degC` (03ffd637) | RX 0x54C ambient temperature |
| 47 | bits | flags A | bit0 `CellMon_Status_flags` b5; bit1 `NV_Status_flags` (03ffb768) b6; bit2 `LB_Capacity_Segment_Switch_flags` (03ffd746) b0; bit3 `Batt_Thermal_State_flags` (03ffd5f0) b1; bit4 `Batt_Thermal_State_flags` b2; bit5 `Heater_Request_flags` (03ffd5f1) b3; bit6 `LB_VCM_Status_flags` b2; bit7 `LB_VCM_Status_flags` b7 |
| 48 | bits | flags B | bit0 VCM b3; bit1 VCM b6; bit2 VCM b5; bit3 VCM b4; bit4 `Full_Charge_flags` b6; bit5 VCM b0; bit6 `IR_Sensor_Fault_flags` b2; bit7 VCM b1 (VCM = `LB_VCM_Status_flags` 03ffd628) |
| 49 | bits | flags C | bit0 `charge_session_active_flag` (03ffd629) b0; bit1 `LB_MainRelayOn_flags` b1; bit2 `Interlock_Drive_flags` b2 |
| 50-51 | u16 | `LB_Usable_SOC_Q15` (03ffd734) | usable SOC, Q15 (32768 = 100 %) |
| 52-53 | u16 | `LB_Max_Power_For_Charger_100W` (03ffdc10) | 100 W |
| 54-57 | u32 | `LB_SOC_ppm` (03ffb218) | ppm |
| 58-61 | 32 | `LB_Current_mA` (03ffdd4c) | mA |
| 62-63 | - | `FF FF` | filler |
| 64 | u8 | `LOW_Batt_Cooling_Req_Status` (03ffd5ea) | 0 or 3 |
| 65 | u8 | `LB_Relay_Cut_Request` (03ffdf46) | |
| 66-67 | u16 | `LB_IR_Sensor_Wave_Voltage` (03ffde4e) | |
| 68-69 | u16 | `LB_SOH_x100` (03ffb26e) | SOH, % x100 |
| 70 | u8 | `LB_Failsafe_Status` (03ffdf47) | |
| 71-72 | u16 | `LB_Charge_Power_Limit_10W` (03ffdcb0) | 10 W |
| 73-74 | u16 | `LB_Discharge_Power_Limit_10W` (03ffdcae) | 10 W |
| 75-76 | s16 | `CAN_1DA_Motor_Speed_rpm` (03ffd634) | rpm |
| 77 | u8 | `Vehicle_Speed_kmh` (03ffd636) | km/h |
| 78-79 | s16 | `CAN_1DA_Motor_Torque_Nm` (03ffd632) | Nm |
| 80-81 | u16 | `LB_Total_Voltage_10mV` (03ffdd94) | 10 mV |
| 82-83 | u16 | `CAN_1DA_Inverter_Voltage_10mV` (03ffd630) | 10 mV |
| 84 | u8 | `charge_timeout_status` (03ffdc75) | |
| 85-86 | u16 | `Energy_Frac_At_Full_Q15` (03ffb77c) | Q15 |
| 87-88 | u16 | `Cap_Derate_At_Full_Q15` (03ffb77e) | Q15 |

## 0x0C: thermal management, heater/cooling requests, temperature-delta check (Len 31 = 0x1F)

Handler `CAN_21_0C` (0005b602). Several of these variables are only LOW-confidence named
(heater/cooling logic, partly in code Ghidra did not export); the byte positions are exact.

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2 | bits | flags | bit0 `Batt_Thermal_State_flags` b7; bit1 `Heater_Request_flags` b0; bit2 HR b5; bit3 HR b6; bit4 HR b4; bit5 BTS b6; bit6 BTS b4; bit7 BTS b5 (BTS = `Batt_Thermal_State_flags` 03ffd5f0, HR = `Heater_Request_flags` 03ffd5f1) |
| 3 | bits | flags | bit0 BTS b3; bit1 HR b1; bit2 HR b2; bit3 03ffdc6f b3; bit4 03ffdc6f b1; bit6 `min_temp_fault_flags` (03ffd54c) b0; bit7 `min_temp_fault_flags` b1 |
| 4 | bits | flags | bit0 03ffb769 b3; bit1 03ffb769 b2; bit2 03ffb769 b1; bit3 `supply_voltage_fault_flag` (03ffdc76) b0; bit4 `IR_Sensor_Fault_flags` b6 |
| 5 | u8 | `LOW_diag30_ioctl_override_flags` (03ffe2da) | set by SID 0x30 IO-control, cleared on return-control/session change |
| 6 | u8 | `LOW_diag30_ioctl_output_on` (03ffe2db) | SID 0x30 override output state |
| 7 | u8 | `LOW_cooling_req_source` (03ffd5e7) | 0 none, 1 auto (code 0x0B), 2 SID 0x30 override |
| 8 | u8 | `LOW_heater_mode_state` (03ffd5e8) | 0 or 3 |
| 9-12 | u32 | `heater_request_timeout_ticks` (03ffd5ec) | timer |
| 13 | u8 | `LOW_heater_drive_source` (03ffd5e6) | 0 none, 1 auto (code 0x0B), 2 SID 0x30 override |
| 14 | u8 | `LOW_heater_drive_level` (03ffd5e5) | 0 or 3 |
| 15-16 | u16 | `Cell_OCV_At_SOC_mV` (03ffd6f4) | cell OCV for current SOC, mV |
| 17 | u8 | 03ffd5eb (`LOW_can_tx_data_byte_2_d5eb`) | written by code at 00043576 (heater module, not reviewed) |
| 18-19 | - | `FF FF` | filler |
| 20-21 | s16 | `min_temp_x200` (03ffd54e) | min temperature snapshot, degC x200 |
| 22-25 | u32 | `temp_delta_check_counter` (03ffd550) | counts to 0x3D861 then runs the temp-delta check |
| 26-27 | u16 | `temp_delta_fault_value` (03ffd554) | |
| 28 | u8 | `temp_delta_fault_count` (03ffc0e8) | |
| 29-30 | u16 | `charge_timeout_value_10ms` (03ffb786) | 10 ms timer (15000 = 150 s) used by 0x0A byte 3 |

## 0x10: digital status (Len 5)

Handler `CAN_21_10` (0005b9f8).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-3 | - | `FF FF` | filler |
| 4 | bits | flags | bit1 = `Batt_Thermal_State_flags` b0; bit2 = `Digital_Input_State_flags` (03ffe5cd) b6; others 0 |

## 0x61: SOH / Hx (Len 12)

Handler `CAN_21_61` (0005ba7c).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-3 | u16 | `Hx_x100` (03ffb274) | Hx, % x100 |
| 4-5 | u16 | `LB_SOH_x100` (03ffb26e) | SOH, % x100 |
| 6 | u8 | `Capacity_Bars_x20` (03ffb2aa) | |
| 7-8 | u16 | `SOH_Filtered_x100` (03ffb272) | % x100 |
| 9-10 | u16 | `SOH_Stored_x100` (03ffb270) | % x100 (writable via SID 0x7B path, see `handle_write_soh_data`) |
| 11 | bits | flags | bit0 = `SOH_Est_Init_flags` (03ffb2ab) b2; bit1 = `SOH_Est_Init_flags` b1 |

## 0x62: usage statistics / histograms (Len 118 = 0x76)

Handler `CAN_21_62` (0005bb6e). Every histogram is 8 x u16 saturating counters
(stop at 0xFFFF), bin k at byte `start + 2k`.

Temperature bins (`find_temp_bin_35_65C`, edges at 0003f4f8 = 35,40,45,50,55,60,65 degC):
bin0 < 35, bin1 35-39, bin2 40-44, bin3 45-49, bin4 50-54, bin5 55-59, bin6 60-64, bin7 >= 65.
SOC bins (`find_soc_bin_20_85pct`, edges at 0003f500 in ppm = 20,30,40,50,60,70,85 %):
bin0 < 20 %, bin1 20-30, bin2 30-40, bin3 40-50, bin4 50-60, bin5 60-70, bin6 70-85, bin7 >= 85 %.

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-3 | u16 | `Stat_Normal_Charge_Count` (03ffb228) | normal (AC) charge count |
| 4-5 | u16 | `Stat_Quick_Charge_Count` (03ffb22a) | quick charge count |
| 6-21 | 8 x u16 | `Stat_IgnOn_Tmax_Hist` (03ffb286) | Tmax at ignition on (temp bins) |
| 22-37 | 8 x u16 | `Stat_ChgStart_Tmax_Hist` (03ffb23c) | Tmax at charge start (temp bins) |
| 38-53 | 8 x u16 | `Stat_Drive_MaxTmax_Hist` (03ffb25e) | max Tmax during drive (temp bins) |
| 54-69 | 8 x u16 | `Stat_Chg_MaxTmax_Hist` (03ffb24e) | max Tmax during charge (temp bins) |
| 70-85 | 8 x u16 | `Stat_IgnOn_SOC_Hist` (03ffb276) | SOC at ignition on (SOC bins) |
| 86-101 | 8 x u16 | `Stat_ChgStart_SOC_Hist` (03ffb22c) | SOC at charge start (SOC bins) |
| 102-117 | 8 x u16 | `Stat_FullChg_SOC_Hist` (03ffb296) | SOC bins, full-charge event |

These are cleared by `CAN_31_04_clear_usage_stats` and written by `CAN_3B_62`.

## 0x63: diagnostic NV word (Len 4)

Handler `CAN_21_63` (0005bdfa).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-3 | u16 | `Diag_NV_Word_3B63` (03ffb24c) | value written by SID 0x3B 0x63; NV block A default on init |

## 0x64: 5 V supply fault wake count (Len 3)

Handler `CAN_21_64` (0005be36).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2 | u8 | `Supply_5V_Fault_Wake_Count` (03ffb2a6) | same byte as 0x03 byte 16 |

## 0x82: communication DTC image (Len 82 = 0x52) (not advertised)

Handler `CAN_21_82` (0005be5e; was `copy_can_tx_to_diag_response`, wrong direction).
It does **not** write a header: it copies the whole 82-byte
`Diag_21_82_Response_Image` (03ffe3a6..03ffe3f7) over the response, **bytes 0-81 included**.

`pack_21_82_comm_dtc_image` (0002bf30, 10 ms task) updates six bytes of that image. Each is
`bit7 | bit6 | 6-bit age counter`. CFI = `Comm_Fault_Indicator_flags` (03ffe3a2),
TXF = `Comm_TX_Fault_flags` (03ffe3a3). "FF in 0x10" = forced to 0xFF while
`vcm_mode_state` (03ffc03f) = 0x10; updated only while it is 0x20 or 0x30, otherwise unchanged.

| Byte | Variable (SRAM) | bit7 | bit6 | bits0-5 |
|---|---|---|---|---|
| 4 | `Diag21_82_BusOff_status` (03ffe3aa) | TXF b1 | `comm_fault_status_flags` (03ffbb85) b0 | `Comm_DTC_Age_BusOff` (03ffb79c), always updated |
| 5 | `Diag21_82_TX_status` (03ffe3ab) | TXF b2 | TXF b0 | `Comm_DTC_Age_TX` (03ffb79f), FF in 0x10 |
| 13 | `Diag21_82_AC_status` (03ffe3b3) | CFI b3 | CFI b7 | `Comm_DTC_Age_AC` (03ffb7a1), FF in 0x10 |
| 21 | `Diag21_82_Inverter_status` (03ffe3bb) | CFI b1 | CFI b5 | `Comm_DTC_Age_Inverter` (03ffb79e), FF in 0x10 |
| 53 | `Diag21_82_VCM_status` (03ffe3db) | CFI b0 | CFI b4 | `Comm_DTC_Age_VCM` (03ffb79d), always updated |
| 81 | `Diag21_82_Charger_status` (03ffe3f7) | CFI b2 | CFI b6 | `Comm_DTC_Age_Charger` (03ffb7a0), FF in 0x10 |

Open question: no exported code writes the other 76 image bytes, including bytes 0-1, so
the `61 82` header and the remaining slots are not explained. Either code Ghidra did not
export fills them, or they stay at their power-on RAM value. Byte offsets above are exact.
A live capture of `21 82` will show the template.

## 0x83: application software / part ID (Len 26 = 0x1A) (not advertised)

Handler `CAN_21_83` (0005be80). All flash constants, so the response is fixed for this image:

`61 83 34 46 41 32 42 53 42 05 17 00 00 00 00 00 03 05 12 11 00 00 00 00 00 80`

| Byte | Size | Source (flash) | Value |
|---|---|---|---|
| 2-6 | 5 | string at 0003f740 | `"4FA2B"` (part number suffix) |
| 7 | 1 | constant | `0x53` `'S'` |
| 8-10 | 3 | `LOW_Tbl_CAN_Status_Data` 0003f734 | `42 05 17` |
| 11-15 | 5 | `LOW_Tbl_CAN_21_83_Payload` 0003f738 | `00 00 00 00 00` |
| 16 | 1 | 0003e0d3 | `03` |
| 17 | 1 | 0003e0d2 | `05` |
| 18 | 1 | 0003e0d5 | `12` |
| 19 | 1 | 0003e0d4 | `11` |
| 20 | 1 | 0003e008 | `00` |
| 21 | 1 | 0003e009 | `00` |
| 22-24 | 3 | constant | `00 00 00` |
| 25 | 1 | constant | `80` |

## 0x84: NV identification string (Len 22 = 0x16) (not advertised)

Handler `CAN_21_84` (0005bf6a).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-17 | 16 | `NV_ID_Data[0..15]` (03ffb0c0) | written by SID 0x3B 0x84 (16 bytes); NV-backed |
| 18-21 | 4 | `20 20 20 20` | ASCII spaces (pads to a 20-char field) |

## 0x90: LB code store (Len 10) (not advertised)

Handler `CAN_21_90` (0005bfd4).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-9 | 8 | `LB_Code_Store[0..7]` (03ffb0da) | written by SID 0x3B 0x90; also used by `rotate_lb_code_pairs` |

## 0x91: diagnostic calibration words (Len 12) (not advertised)

Handler `CAN_21_91` (0005c00a).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-11 | 5 x u16 | `NV_DiagCal_Words_3B91[0..4]` (03ffb0e2) | word n at 2+2n, BE; same layout as the SID 0x3B 0x91 request bytes 2-11 |

## 0x92: per-cell voltage trims (Len 98 = 0x62) (not advertised)

Handler `CAN_21_92` (0005c080).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-97 | 96 x s8 | `Cell_V_Corr_3B92_mV[0..95]` (03ffb0ec) | cell n at 2+n, signed mV, subtracted from the measured cell voltage (scaled down proportionally below 4150 mV). Written by SID 0x3B 0x92. |

## 0x94: capacity balancing complete (Len 3) (not advertised)

Handler `CAN_21_94` (0005c0b8).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2 | u8 | `LB_Capacity_Bal_Complete_flags` (03ffdad8) bit7 | 0 or 1 |

## 0x95: cell-monitor failure confirmation bits (Len 4) (not advertised)

Handler `CAN_21_95` (0005c0ec).

| Byte | Size | Variable (SRAM) | Notes |
|---|---|---|---|
| 2-3 | u16 | `CellMon_FailConf_bits` (03ffdaec) | bit field |

## 0xF0 / 0xF1: ID blocks from flash (Len 26 = 0x1A) (not advertised)

Handler `CAN_21_F0_F1` (0005c128). Copies 24 bytes from flash `0000dfd0 + (LID & 1) * 24`.

| LID | Byte 2-25 (flash) | Content in this image |
|---|---|---|
| F0 | 0000dfd0..0000dfe7 | `34 46 41 32 42 53 42 05 17 00 00 00 00 00 03 05 12 11 00 00 00 00 00 80` (identical to 0x83 bytes 2-25) |
| F1 | 0000dfe8..0000dfff | `00 00 00 00 00 46 30 30 30 30 46 30 30 30 30 00 00 00 00 00 00 5C 16 16` (`"F0000F0000"` + trailer) |

## 0xFE / 0xFF: ID blocks from EEPROM, flash fallback (Len 26 = 0x1A) (not advertised)

Handler `CAN_21_FEFF` (0005c18e), resumed every 10 ms by `CAN_21_FEFF_poll` (0005c4aa) while
`Diag21_FEFF_Pending` (03ffe2e3) = 1.

1. If the EEPROM is busy it sets Pending and retries on the next tick.
2. It reads the EEPROM word at 0x7FC. If its high byte (EEPROM 0x7FD) is 0xFF (blank), it answers
   exactly like **F0** (for FE) / **F1** (for FF) from flash.
3. Otherwise it reads 12 words (24 bytes) from EEPROM `0x7D0 + (LID & 1) * 0x18`, 4 words per
   10 ms tick (`Diag21_FEFF_State` 03ffe2e8: 0xFA idle, 0xF5 reading), then sends.

| Byte | Size | Source | Notes |
|---|---|---|---|
| 2-25 | 24 | EEPROM 0x7D0-0x7E7 (FE) or 0x7E8-0x7FF (FF), in address order | same layout as F0/F1; usually the in-field copy of the ID blocks |

---

## Bootloader 0x21 (only active in the loader / reprogramming mode)

`loader_21_read_local_id` (00003a3c), reached from `loader_service_dispatch`.
Request length must be 2, otherwise `7F 21 12`.

| LID | Len | Handler | Content |
|---|---|---|---|
| 00/20/40/60 | 6 | 00003b62 `loader_21_supported_bitmap` | 4 bytes from flash 0x7E00 + LID/8 (all zero in this dump) |
| 83 | 26 | 00003c4a `loader_21_83_read_boot_id` | 24 bytes from flash 0xDFB0 (boot ID block) |
| F0/F1 | 26 | 00003cb8 | **security-related**: the handler is on the security exclusion list, so it was not opened |
| FE/FF | 26 | 00003d40 `loader_21_fe_ff_read_eeprom` (+ `loader_21_fe_ff_poll` 00004030) | EEPROM ID read, same idea as the application FE/FF |

---

## Notes and open questions

- **Security:** none of the application 0x21 local IDs needs elevated access. The matching
  **write** services (`CAN_3B_84`, `CAN_3B_90`, `CAN_3B_91`, `CAN_3B_92`) are gated (a
  session check plus a flag at 03ffe302, else NRC 0x33). That gate belongs to the security
  path and was not followed. Only the bootloader's 0x21 F0/F1 is security-related.
- **0x82:** 76 of the 82 image bytes, including bytes 0-1, have no writer in the exported code.
- **0x0A bytes 4-5:** the event counter (`sample_avg_index` 03ffe5ac) is incremented by code
  outside the export, so what it measures is unknown.
- **0x0C byte 17 and the heater/cooling bytes:** written by heater code that is partly
  outside the export (0x42cb8..0x435b4); their names are LOW confidence.
- Signedness: given as s8/s16/s32 only where the producing code shows it. Otherwise the size
  comes from the copy width, and a value may still be signed (e.g. `LB_Current_mA`).

