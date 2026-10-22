# Watchdog / monitor IC (5SHBV) — what it does

[C] = read in the code, [I] = inferred. Security functions are out of scope: the UDS 0x27 handler
(`diag_27_security` 0001a5d4) and anything reached only through it were not opened (`watchdogIC/analysis/security_excluded.txt`).

## Summary
The "watchdog IC" is a full second V850 controller on the LBC board: an **independent cell-voltage and temperature
protection monitor that also cross-checks the ZE1 CPU**. It:

1. **Listens to the cell-monitor SPI bus** (CSIB1 as receive-only SPI slave plus DMA). The ZE1 drives that bus, and
   this chip decodes the same 8 cell-monitor ICs × 12 channels = 96 cell voltages itself [C].
2. Runs its **own over/undervoltage check on every cell** (OV > 4.259 V, UV ≤ 2.090 V, 84-pass debounce), checks
   **3 thermistors** (circuit faults, disagreement between sensors, over-temperature) and keeps its own copy of the
   **cell-monitor IC fault items** (same DTC families as the ZE1) [C].
3. **Exchanges a mutual CPU self-test with the ZE1 over UART** (ZE1 UARTD3 ↔ this chip's UARTD1). Each side runs
   the same 18 instruction-set tests and checks the other's results against the same expected table [C].
4. **Reports to the vehicle on CAN 0x1C2**: one byte = status nibble (5 = OK, 0xA = fault) + a 4-bit rolling
   counter [C].
5. Monitors the VCM frames **0x1D4, 0x1DA, 0x1F2, 0x50B** (timeout, DLC, rolling counter, CRC) and two wake inputs.
   It switches its CAN controller off at shutdown, runs a shutdown test of an output line (P4.3), and has its own
   UDS server on **0x796 / 0x7B6** and an external SPI EEPROM [C].

## Setup (separate from the ZE1 project)
| item | value |
|---|---|
| image | `watchdogIC/5SHBV_sml(ZE1).bin` → copy `WDG_5SHBV.bin` (batch files cannot take the parentheses); part ID "SHBV" at 0x7D090 |
| CPU | V850E, V850ES/Fx3 peripheral map (UARTD FFFFFA00+0x10n, CSIB FFFFFD00/FD10, FCAN0 03FEC000, DMA, TMM0) |
| Ghidra | project `watchdogIC/ghidra/WDG_5SHBV`, export `watchdogIC/export` (same format as the ZE1 export) |
| names | `watchdogIC/analysis/renames.tsv` (173 functions) + `globals.tsv` (78 variables), applied with `watchdogIC/apply_wdg.bat` |
| scripts | `import_wdg.bat`, `setgp_wdg.bat` (gp 0x03FFA124), `addblocks_wdg.bat`, `decompile_wdg.bat a+b`, `reexport_wdg.bat`, `apply_wdg.bat` |
| memory | `ghidra_scripts/AddPeripheralBlocks.java` adds volatile blocks CAN 03FEC000..03FEDFFF and IO_MIRROR 3FFFF000.. (without them Ghidra decompiles CAN polling loops as endless) |
| layout | boot 0..~0x7FFF (own CAN/UDS, RAM handler table 70a4..70bc), app 0xD000..0x4xxxx, calibration 0x7D000.., partial dead copy of the app shifted +0x2F000 at 0x4xxxx..0x7xxxx |
| startup | `reset_entry` 6bac (clock, gp, sp) → `main` 28ef0: RAM pattern test (0x55../0xAA..) of 1 KB, ROM sums (`startup_rom_check`: boot = 0xDFAC, app = 0x7D004), clock register check, wake inputs |

## Main loop and power states
`main` loops while `power_state` 90c4 ≥ 0x1E. Each pass ends in `wait_main_tick` (TMM0 compare 0x1AE; flag F14C).
Sub-rates: `tick_flags` bit1 every 2, bit2 every 10, bit3 every 50 ticks. The tick period depends on the CPU clock,
not decoded [I: about 10 ms]. `toggle_heartbeat` flips **PCS1 (F008 bit1) every pass**: a heartbeat output [C], whose
receiver is not identified [I: external watchdog].

| state | work done | leaves when |
|---|---|---|
| 0x1E run | `state_run_task`: 0x1C2 frame, CAN config check, all monitoring tasks, diagnostics | P1.0 inactive **and** `vcm_shutdown_req` = 2 → `can_stop_controller`, go to 0x28 |
| 0x28 | monitoring continues, CAN off; `shutdown_p43_test_task` | a wake input active again → 10; ZE1 state invalid / P4.3 test done → UART, DMA, CSIB off, go to 0x3C |
| 0x3C / 0x46 / 0x50 | EEPROM write-back side (3f844, 3ab50, 3cd82, 405d8, not decoded) | input active → 10; done flags → next; 0x50 + 70e3 = 0xA5 → `sleep_wait_for_wake` |
| 10 | stop peripherals, jump to reset (clean restart) | – |

Wake inputs [C]: **P0.3** (F400.3) → `input_p03_state` 70e1 and **P1.0** (F402.0) → `input_p10_state` 70e0.
0x55 = high, 0xAA = low (10 / 5 reads debounce). The ZE1 reads the same two pins: low = active, F400.3 = ignition.
With both inactive at power-up the chip goes straight to `sleep_wait_for_wake`. That routine toggles the heartbeat,
pulses P4.3 every 30 loops and waits for P0.3 to go low.
`vcm_shutdown_req` 9092 = 2 when P0.3 is inactive **and** (0x1F2 charge mode = 3 or any 0x1F2 item is NG-confirmed).

## UART link with the ZE1 (this chip UARTD1 ↔ ZE1 UARTD3)
Both ends: CTL1 = 1, CTL2 = 4 (same baud divider), 8 data bits, **even parity**, 1 stop, LSB first [C].
RX by interrupt (`ud1_rx_isr` / `ud1_rx_error_isr`), TX polled (`ud1_send_polled`). A frame timer is started on the
first byte. If it expires (`ud1_frame_timeout_isr`) the partial frame is dropped.

### Request ZE1 → this chip, 10 bytes
| byte | here | ZE1 source (`build_monitor_request` 5957c) |
|---|---|---|
| 0 | `ze1_system_state` 90b9 | System_State |
| 1 | `ze1_mode` 90bb | 5922 run mode: **0x20 running**, 0x30 / 0x40 stop and sleep phases |
| 2 | `ze1_req_b2` 90bd (1..14) | 63e9: cell-monitor command step [I] |
| 3 | `ze1_req_b3` 90bf (1..0x75) | 63ea: cell-monitor command step in stop mode [I] |
| 4 | `ze1_selftest_index` 90b6 | ZE1 self-test number 0..17 |
| 5..8 | `ze1_selftest_result` 90a8 (BE) | ZE1 self-test result |
| 9 | CRC-8 over bytes 0..8 | |

### Reply this chip → ZE1, 7 bytes, sent immediately after a complete request
| byte | content |
|---|---|
| 0 | own self-test number (`reply_selftest_index`) |
| 1..4 | own self-test result (BE) |
| 5 | status `reply_status`: bit7 = fault_summary bit17 (ZE1 self-test NG, item 8, or 0x1C2 status = fault); bit6 = fault_summary bit19 (any of: UART link items 0x10/0x11, items 7/9/0x0F.., shutdown test 70f0, NTC / IC items); bit0 = 9536 bit3 |
| 6 | CRC-8 over bytes 0..5 |

**CRC-8 (both chips, identical 256-byte tables):** reflected polynomial 0x4D (table polynomial 0xB2), init 0,
no final XOR.

### Link supervision [C]
- `monitor_rx_check`: any parity/framing/overrun bit on any byte, or a CRC mismatch → bad frame (90b4 bit5); else good (bit4).
- **Item 0x10** (90e8, DTC **P33F9**): good / bad frames; NG confirmed after **150** consecutive bad frames.
- **Item 0x11** (90e9, DTC **P33F9**): no frame for > 3 passes = NG now; confirmed after **150**.
- Both are evaluated only while ZE1 `System_State` < 0x29.
- When link items are NG, `monitor_apply_request` sets all ZE1 fields to 0xFF. Cell measurement then stops
  (cell values 0xFFFF, OV/UV checks idle) and the ZE1 self-test check stops.

## Mutual CPU self-test (the "ALU check")
Both chips run the same 18 tests, one per pass, with all interrupts masked and every register saved. Each checks the
other's results against the same table (`Tbl_Selftest_Expected` 0x40808 = ZE1 `Tbl_Monitor_Expected_Echo` 0x5eb74).

- This chip checks the ZE1 (`check_ze1_selftest_results`): stores the ZE1 result at `ze1_selftest_results[n]` and
  compares all 18 (0xFFFFFFFF = not received yet). **Item 0x12** (90ea) NG is confirmed after **> 89** consecutive
  mismatches, the same rule as the ZE1's `confirm_monitor_echo_status`. The ZE1 checks this chip from reply bytes 0..4.

| # | instructions exercised | expected | # | instructions exercised | expected |
|---|---|---|---|---|---|
| 0 | register file r1..r29, not, add | 0x55555544 | 9 | sasf, all conditions | 0xf |
| 1 | byte/half/word load/store, sign/zero extend | 0xa5b4ffa7 | 10 | setf, all conditions | 0x11 |
| 2 | sld/sst (ep), hsw | 0xa5b5c4d0 | 11 | zxb zxh sxb sxh | 0x1205bb4 |
| 3 | mul mulh mulhi mulu | 0x8006fff0 | 12 | div divh divu divhu | 0xf419e7 |
| 4 | cmp + PSW flags | 0xa5a55a5f | 13 | every conditional branch | 0x4000000 |
| 5 | satadd, PSW.SAT | 0x1f | 14 | set1 clr1 not1 tst1 | 0x4 |
| 6 | xor tst not + flags | 0x1 | 15 | callt / CTBP, ei/di | 0x70007 |
| 7 | sar hsw bsw bsh + flags | 0xb4c378d7 | 16 | prepare / dispose | 0x75324440 |
| 8 | cmov, all conditions | 0x55555155 | 17 | trap round trip | 0x10001 |

## Cell voltage monitoring (SPI listener)
- **CSIB1** = SPI **slave, receive only** (`csib1_init_slave_rx`: CB1CTL1 = 0x1F external clock; RXE on, TXE off) [C].
  DMA1 copies CB1RX into `spi_snoop_buf` 3ffa29b (26-byte slots).
- **Vector 0xE0** (external pin interrupt, `spi_snoop_start_isr`): picks the expected byte count of the next
  transaction from command tables 0x40A92..0x40DAE, chosen by ZE1 mode (request byte 1 = 0x20 / 0x30) and step (byte 2 / 3),
  and starts DMA1. **Vector 0x380** (DMA1 end, `spi_snoop_dma_done_isr`) moves to the next slot and step.
  So the chip follows, transaction by transaction, the cell-monitor exchanges the ZE1 performs and announces over the
  UART [C]. The interrupt pin is probably the SPI chip select [I].
- Decode: 8 ICs × 12 channels (`Tbl_Cells_Per_IC` 0x1005C = 0C ×8, channel masks 0x10064..), raw 14-bit →
  **mV = (raw >> 2) × 5000 / 16384** (`raw_to_mv`). An IC with a status error gives 0xFFFF for its cells.
  Each cell has a 5-sample ring and is averaged over the valid samples into `cell_voltage_mv` 8f54[96] (UDS 21 02).
- Valid only while **ZE1 mode = 0x20 (ZE1 running)**; otherwise all cells read 0xFFFF [C].

| check | NG when | OK when | confirm | result |
|---|---|---|---|---|
| overvoltage (`check_cell_overvoltage`) | any cell > **4259 mV** | < 4160 mV | 84 passes | per-cell 8670, `cell_volt_faults` 908a bits4..7 |
| undervoltage (`check_cell_undervoltage`) | any cell ≤ **2090 mV** (`uv_threshold_mv`) | > 3000 mV | 84 passes | per-cell 8610, 908a bits0..3 |
| OV event count | retained `ov_event_count` 70d2 ≥ 255 (0x7D084) | | | 908a bit8 |
| UV event count | retained `uv_event_count` 70d3 ≥ 2 (0x7D085) | | | 908a bit9 |

The cell-monitor IC fault items (8 per family, one per IC) use the same DTC families as the ZE1 (calibration DTC table
0x7D1A2, entries {item, DTC big-endian, flag}): items 0x23..0x2C → P33A5..P33AC, 0x2D..0x35 → P33AF..P33B7,
0x36..0x3D → P33C8..P33CF, 0x3E..0x45 → P32B6..P32BD, 0x46..0x4D → P32BE..P32C5, 0x4E..0x5D → P32C6..P32D5,
0x5E..0x65 → P307B..P3082, 0x85..0x8D → P3302..P330A. The bit meanings inside each family were not decoded.

## Temperatures
- 3 thermistors, ADC `ntc_adc` 9190..9194 → °C via table 0x7D10C / 0x7D148, clamped −40..105 (`convert_ntc_temps`).
- Items 0x76..0x7B, circuit faults: ADC ≥ 0x3E0 (high side) or < 0x1F (low side), 200 passes.
- Items 0x7C..0x81, sensor disagreement: pairwise differences above 15 / 21 / 12 / 13 / 12 / 23 (`eval_temp_spread_items`).
- fault_summary bit18 = **over-temperature**: filtered value 9070 > 66, 46 passes, latched [C]. 9070 is probably the
  hottest sensor in °C [I]. Bit16 = too many thermistor items NG at once (temperature not measurable) [C].

## CAN (FCAN0)
| dir | ID | MB | content |
|---|---|---|---|
| TX | **0x1C2** | 0 | 1 byte, sent every 0x1E pass: bits7..4 = `monitor_status_code` (5 = OK, 0xA = fault), bits3..0 = `can_1c2_counter` 0..15 (+1 per pass) |
| TX | 0x7B6 | 1 | UDS responses |
| RX | 0x1D4 | 16 | check: DLC 8, rolling counter byte4 bits7..6 must change, CRC-8 poly 0x85 in byte7; timeout 3 passes |
| RX | 0x50B | 17 | DLC ≥ 7; byte3 state → `can_50b_state` (0 / 3); timeout 30 passes |
| RX | 0x1DA | 18 | same checks as 1D4; timeout 3 passes |
| RX | 0x1F2 | 19 | same checks; charge mode → `can_1f2_charge_mode`; timeout 3 passes |
| RX | 0x796 | 20 | UDS requests (interrupt driven, ISO-TP) |

RX check codes (`rx_check_*` 923a..d): 0x80 not received, 0x40 DLC short, 0x20 counter unchanged, 0x10 CRC bad, 0 OK.
Bus-off is recovered by `can_busoff_recover` (C0BRP = 1, C0BTR = 0x140F). The CAN registers and mailbox IDs are
re-verified every pass (`check_can_rx_config` / `check_can_tx_config`).

**0x1C2 status = 0xA** (`monitor_prepare_reply_data`) when any of these holds [C]:
- cell UV confirmed, cell OV confirmed, or the OV event limit is reached (`908a & 0x311`);
- temperature not measurable or over-temperature (`fault_summary & 0x50000`);
- cell-monitor IC fault summary (`8b7a & 0x9ab`);
- heartbeat pulse rate NG (item 0x0F: 9..11 pulses per 10 passes, NG after 12);
- ZE1 self-test NG (item 0x12).

**The frame stops** when the chip leaves state 0x1E (P1.0 inactive with `vcm_shutdown_req` = 2 → CAN off), shuts
down, restarts, or fails its ROM / clock checks [C]. In state 0x1E, while `vcm_shutdown_req` = 2, it is sent only
if 0x50B state = 3 and the 0x50B items are not NG [C].

## Shutdown test of P4.3 and the pulse count
- `drive_p43_output`: P4.3 (F412.3) goes high when `p43_test_request` is set, when the pulse-rate item 0x0F is NG,
  or on a UDS test command. While high, most checks are suppressed (`io_state_flags` bit4) [C].
- `shutdown_p43_test_task` (state 0x28): after the ZE1 has reported state ≥ 0x3C for 200 passes it drives P4.3 and
  expects the pulse count to drop to 0 (OK within 2 passes, NG after 10 → retained item 70f0 → reply status bit6) [C].
- Pulses are counted in `pulse_count` by `pulse_count_isr` 3bc00 and latched every 10 passes. Its trigger was not
  found [I: an input that P4.3 interrupts]. What P4.3 drives on the board is unknown [I: a hardware cut-off or
  enable line].

## Other parts
- **External SPI EEPROM** on CSIB0 (master, chip select P0.6): commands 3 READ, 2 WRITE, 5 RDSR, 1 WRSR, 16-bit
  address < 0x2000 (8 KB). Holds the fault log and ID; see "UDS services and the EEPROM log".
- **UDS server** on 0x796 → 0x7B6: see "UDS services and the EEPROM log".
- NMI (vector 0x10): every port to a safe state. Vector 0x60: endless loop.

## UDS services and the EEPROM log
Request 0x796, response 0x7B6 (ISO-TP). Security access **0x27 is not opened** (`diag_27_security` 0001a5d4).

| SID | access | what it does |
|---|---|---|
| 10 | any | sessions 0x81 default, 0xC0 extended, 0xF0 (only if c082/c085 clear, else NRC 22); 0x85 → NRC 12 (programming is in the boot block) |
| 14 | `14 FF FF FF`, only while a wake input is active (else NRC 22) | clears DTCs **and wipes every log record below** (`nv_data_clear` via 16eae). **Read the logs before sending it.** |
| 19 | any session | 02 DTC by status, **04 freeze frame**, 06 extended data |
| 21 | any session (length 2) | live data + **the EEPROM logs (LID 13 / 14 / 15)** + ID (FE / FF) |
| 27 | – | security access: not investigated |
| 30 / 31 / 3B | session 0xF0 plus extra access flags (d0d4 / d0e4 / d0ea), probably security-gated | 30 FC I/O control, 31 FA routine, 3B 90 write LID 90: **not followed** |
| 3E | any | tester present |
| 34 | – | NRC 7F (not in application) |

### EEPROM (8 KB SPI, CSIB0)
Loaded at startup by `eeprom_load_all`. Every block has a header word whose low 6 bits must be 0x1A, plus a checksum
(`eeprom_select_valid`); 21 08 reports the result.

| EEPROM | size | RAM | content |
|---|---|---|---|
| 0x0000 / 0x0002 / 0x0004 / 0x0006 | 4 × u16 | a908 / a904 / a90a / a906 | headers of record 0x08, record copy A, image, record copy B |
| 0x0008 | 18 | a912 | small record (not decoded) |
| 0x001A / 0x001D | 3 + 3 | a90c / a90f | two copies of a 3-byte record (not decoded) |
| 0x0020..0x1458 | 0x1439 | `eeprom_image` a924 → retained records (`nv_restore_from_eeprom`) | **the log** (table below) |
| 0x1FD0 / 0x1FE8 | 24 + 24 | read on request | **21 FE / FF**: ID strings; blank (0x1FFD = FF) → default "5SH2DVB" + version / "F0000F0000" |

| EEPROM | size | retained RAM | log content | read with |
|---|---|---|---|---|
| 0x0020 | 0x2C8 | `nv_freeze_frames` 7230 | 2 DTC freeze frames (first + latest confirmed DTC) | **19 04** |
| 0x02E8 | 58 × 21 | `nv_raw_frames_0` 78dc | raw cell-monitor SPI frames at snapshot 0 | **21 13** |
| 0x07AA | 25 × 21 | `nv_raw_frames_1` 7da2 | raw frames at snapshot 1 | **21 14** |
| 0x09B7 | 25 × 21 | `nv_raw_frames_2` 7faf | raw frames at snapshot 2 | **21 15** |
| 0x0BC4 / 0x0DD1 | 25 × 21 each | `ff_raw_frames_0/1` 81bc / 83c9 | raw frames with freeze frame 0 / 1 | **19 04** (appended) |
| 0x0FDE | 3 × 0x140 | `nv_snapshot` 74f8 / 7638 / 7778 | snapshot records 0 / 1 / 2 | **21 13 / 14 / 15** |
| 0x139E.. | ~0xBB | 85d6.., 7d9e, 7da0, 78b8, c004 | counters and the stored DTC list (30 × {item, status}) | inside the snapshots / freeze frames |

The data is written back to the EEPROM in the shutdown states (0x3C..0x50, `state_3c_task` → 3ab50 / 3abb0) [C].

### What is logged and when
`nv_log_task` runs every pass. Each snapshot is taken **once, at the first occurrence**, and stays until 0x14 clears it [C]:
- **Snapshot 0 (21 13)**: item 0x23 (cell-monitor measurement, DTC P33A5) goes to "fault now" → record + 58 raw frames.
- **Snapshot 1 (21 14)**: any per-IC item 0x24..0x2C (P33A6..P33AE) goes to "fault now" → record + 25 raw frames.
- **Snapshot 2 (21 15)**: any per-IC item 0x24..0x2C reaches "fault confirmed" → record + 25 raw frames.
- **Freeze frames (19 04)**: slot 0 = first confirmed DTC, slot 1 = latest; same data model as the ZE1's 19 04.

**Debug value [I]:** high for cell-monitor / SPI faults. The records hold all 96 cell voltages, the cell-monitor IC
error bitmap and status words, the SPI parse/error flags, the ZE1's request bytes (mode and command step) at the moment
of the fault, retained per-IC fault counters, and the raw cell-monitor reply frames. There is nothing about SOC,
capacity or charging, and an unused record reads all zeros.

### 21 13 / 21 14 / 21 15 (0x606 bytes each)
Bytes 0..1 = `61 13` / `61 14` / `61 15`. Byte n (2..321) = snapshot record offset n − 2. Multi-byte values are big-endian.

| byte | rec | size | variable | meaning |
|---|---|---|---|---|
| 2 | 0x00 | u8 | ze1_system_state | ZE1 state byte (UART request byte 0) |
| 3 | 0x01 | u8 | ze1_mode | ZE1 run mode: 0x20 running, 0x30 / 0x40 stop |
| 4 | 0x02 | u8 | ze1_req_b2 | cell-monitor step (running) |
| 5 | 0x03 | u8 | ze1_req_b3 | cell-monitor step (stop mode) |
| 6 | 0x04 | u8 | cellmon_ic_error_bitmap | bit n = IC n communication or status error |
| 7 | 0x05 | u8 | bd7b | step b2 latched from the last good frame |
| 8 | 0x06 | u8 | req_b2 | step b2 as received |
| 9 | 0x07 | u8 | monitor_rx_buf[3] | raw request byte 3 |
| 10 | 0x08 | u8 | bd7c | step b3 latched |
| 11 | 0x09 | u8 | req_b3 | step b3 as received |
| 12 | 0x0A | u8 | spi_parse_result | 0 OK, 1 overrun, 8 / 0x42 / 0x43 / 0x46 parse errors |
| 13 | 0x0B | u8 | spi_step_check_flags | bits 5..7: step check results |
| 14 | 0x0C | u8 | item11_uart_timeout | UART-link item status (bit7 NG now, bit6 OK now, bit5 NG confirmed, bit4 OK confirmed) |
| 15..23 | 0x0D | 9 × u8 | ic_fault_count_nv | retained "fault now" counts, items 0x24..0x2C (per IC) |
| 24..32 | 0x16 | 9 × u8 | ic_fault_count_stop_nv | same, counted only while ZE1 mode = 0x30 |
| 33..41 | 0x1F | 9 × u8 | ic_fault_count | same, since power-up |
| 42 | 0x28 | u8 | item2d_count_nv | retained count of item 0x2D (P33AF) |
| 43 | 0x29 | bits | | bit0 8b80.0 (ZE1 step stable), bit1 9536.7 (SPI data being processed), bit2 / bit3 monitor_frame_status bit6 / bit7 |
| 44..45 | 0x2A | u16 | adc_ref_2v5 | 2.5 V reference, ADC counts |
| 46..47 | 0x2C | u16 | supply_12v_mv | 12 V supply, mV |
| 48..49 | 0x2E | u16 | spi_comm_error_flags | SPI communication error bits |
| 50..241 | 0x30 | 96 × u16 | cell_voltage_mv | cells 1..96, mV (0xFFFF = invalid) |
| 242..259 | 0xF0 | 9 × u16 | cellmon_status_or | 9 status words, OR over the ICs (= ZE1 14 D7..DF) |
| 260..261 | 0x102 | u16 | item23_count_nv | retained count of item 0x23 fault-now passes |
| 262..263 | 0x104 | u16 | item23_count_stop_nv | same, ZE1 mode 0x30 only |
| 264..265 | 0x106 | u16 | item23_count | same, since power-up |
| 266..269 | 0x108 | u32 | cellmon_fault_ctx_state | 12 V mV \| power_state << 16 \| ZE1 state << 24, at the last cell-monitor fault |
| 270..273 | 0x10C | u32 | cellmon_fault_ctx_mode | ZE1 mode \| b2 << 8 \| b3 << 16 \| parse result << 24 |
| 274..277 | 0x110 | u32 | cellmon_fault_ctx_status | spi_comm_error_flags \| IC status OR bits 16..19 \| bit20 |
| 278..281 | 0x114 | u32 | link_fault_ctx_state | 12 V mV \| power_state << 16 \| ZE1 state << 24, at the last UART-link fault |
| 282..285 | 0x118 | u32 | link_fault_ctx_comm | item 0x10 NG count \| item 0x11 NG count << 8 \| frame flags bits 16..19 |
| 286..321 | 0x11C | 9 × u32 | ic_fault_nibble_counts | word n = item family 0x2E + 8n; nibble k = IC k fault-now count (max 15) |
| 322.. | | 21 each | raw frames | 21 13: 58 frames (to byte 1539); 21 14 / 15: 25 frames (to byte 846) |
| rest | | | FF | padding to 0x606 bytes |

The raw 21-byte frames are the cell-monitor replies as received from the SPI bus (`spi_snoop_buf` parsed into 9dd5).
Their bit meanings are those of the cell-monitor IC and were not decoded [I].

### 19 04 freeze frame (`19 04 <DTC 3 bytes> FF`, 0x75C bytes)
Header `59 04 DTC status 00 9A`, then {`14 xx`, value} pairs (big-endian). The tags are the same as the ZE1's 19 04.

| tag | value | tag | value |
|---|---|---|---|
| 14 01 | u16 item of slot-0 DTC | 14 B8..BB | ZE1 state, mode, b2, b3 |
| 14 02 | u16 item of slot-1 DTC | 14 BC / BD | ZE1 self-test index / result (u32) |
| 14 03 | u16 item that set the DTC | 14 BE | pulse_count_10 |
| 14 0A | 2.5 V reference ADC | 14 D5 | spi_comm_error_flags |
| 14 0E | 12 V supply mV | 14 D6 | cellmon_ic_error_bitmap |
| 14 15..1F, 21..3F, 41..5F, 61..77 | cells 1..96 mV (0x20 / 0x40 / 0x60 skipped) | 14 D7..DF | cellmon_status_or[0..8] |
| 14 7F / 81 | NTC1 ADC / °C | 14 E1..E3 | cell-monitor fault context state / mode / status |
| 14 82 / 83 | NTC2 ADC / °C | 14 E4 / E5 | UART-link fault context state / comm |
| 14 86 / 87 | NTC3 ADC / °C | 14 E6..EC | step b2 latched, b2 raw, req byte 3, b3 latched, b3 raw, parse result, step checks |
| 14 B4..B6 | ADC F212 / F20E / F20C (ADC self-test channels) | 14 ED | item 0x11 status |
| 14 EE / F1 / F3 | item23 counts (retained / stop / since power-up) | 14 EF / F2 / F4 | per-IC counts ×9 (retained / stop / since power-up) |
| 14 F5 | 9 × u32 IC nibble counters | 14 F6 | item 0x2D count |
| 14 F7..FA | flag bits 0..3 (as byte 43 above) | 14 FB | 25 raw cell-monitor frames + FF padding |

### Other 21 LIDs (live, not logged)
00/20/40/60 supported-LID bitmaps · 01 12 V mV · 02 96 cell mV · 03 c084 bit0 · 04 NTC ADC ×3 · 05 supply / ref / NTC ADCs
and the two wake-input states · 08 EEPROM validity (record copy, image) · 11 ADC test channels · 12 ZE1 self-test index /
result, state, mode, steps, pulse count · 82 0x52 bytes from d0ef · 83 version / part data from calibration ·
90 8 retained bytes 70c0 (written by 3B 90) · F0 / F1 the flash default of the FE / FF ID · 81 not supported (NRC 12).

## Open questions
- **Why the 0x1C2 counter stops when the ZE1 stops sending.** In the code, losing the UART link (items 0x10/0x11)
  does **not** by itself set the 0x1C2 status to 0xA or stop the frame. It sets reply status bit6 and stops cell
  and self-test monitoring (fields 0xFF). The observed stop would need one of the "frame stops" conditions above,
  for example the ZE1 releasing P1.0 or P0.3, or the chip being held in reset or unpowered [I]. Logging P0.3, P1.0,
  P4.3 and PCS1 on the bench would settle it.
- Exact tick period and baud rate (depend on the PLL setting in `reset_entry`).
- Meaning of the cell-monitor command tables 0x40A92.. and of the per-IC fault bits.
- Who calls `pulse_count_isr` 3bc00; what P4.3 and the pulse input are wired to.
- 0002768a and the EEPROM write-back details (3abb0, 3b2ae) were not decoded; the small EEPROM records at 0x08 / 0x1A / 0x1D are unknown.
- Bit meanings of the 21-byte raw cell-monitor frames, `cellmon_status_or` and `spi_comm_error_flags`.
