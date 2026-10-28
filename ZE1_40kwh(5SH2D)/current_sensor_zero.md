# ZE1 LBC (5SH2D): current-sensor zero ("null") learning

For: developers of the Battery-Emulator / VCM-side controller that drives a ZE1 (40 kWh) Leaf LBC.
Goal: make sure every condition the LBC checks is met, so it learns a clean zero for both current-sensor
channels and raises no current-sensor DTC.

Tags: **[C]** read directly from the 5SH2D firmware code, **[I]** inference. Addresses are RAM (0x03ffxxxx) or
flash; function names are the names used in this RE project. CAN byte numbering is 0-based (byte0 = first data
byte); bit numbering is bit7 = MSB, bit0 = LSB.

---

## 1. Short version (checklist)

The LBC learns the zero **once per MCU start**, during the **first 10 passes of its 10 ms main task**
(about the first 100 ms after it starts sending 0x1DB). It does not wait for conditions to become true. If
the conditions are not met in that window, the chance is gone until the next reset/power-up.

For a good zero, all of the following must be true **from LBC power-up until about 200 ms after the
first 0x1DB frame from the LBC** (100 ms window plus margin):

| # | Condition | Who controls it |
|---|---|---|
| 1 | **0 A through the pack current sensor.** Contactors open, no precharge, no HV load or charger. | Emulator |
| 2 | **0x1D4 byte5 bit6 (mask 0x40) = 0**, or no 0x1D4 sent yet. If the LBC sees this bit set before learning finishes, it **skips learning** and uses fixed defaults. | Emulator |
| 3 | LBC's own "main relay on" flag (0x1DB byte3 bit5) = 0. This flag is internal and is 0 after a clean boot [C/I]. | LBC (check it on 0x1DB) |
| 4 | LBC 12 V supply healthy: > 7.58 V and < 18.4 V (items 0x0A / 0x10). Use a normal 12–14 V. | Emulator / 12 V supply |
| 5 | LBC internal 5 V sensor supply in 4.711–5.273 V (item 0x11, DTC P33D6), 2.5 V ADC reference in 2.290–2.710 V (item 0x0B), ADC self-test OK (item 0x09). | LBC hardware (fails only with a fault or low 12 V) |
| 6 | Sensor outputs at rest: **ch1 667–708 counts (3.26–3.46 V)**, **ch2 410–450 counts (2.00–2.20 V)**. The two zeros must differ by **more than 9 counts**. | Sensor / harness (or your simulated sensor) |

Then, **after** the window: close contactors / set 0x1D4 byte5 bit6 as normal.

To check the result, read UDS `21 01` and `21 07` (no session or security needed, see §6). After a real learn
the zero offsets in `21 07` are normally **not** exactly 687,000,000 / 430,000,000. Exactly those values mean
the defaults were used.

---

## 2. The sensor and the numbers

The LBC reads one current sensor with **two channels** on 10-bit ADC inputs (0..1023 counts,
5000/1024 = 4.883 mV per count) [C]:

| | Ch1 (wide range) | Ch2 (fine range) |
|---|---|---|
| ADC input / RAM | FFFFF210 → `Current_Sensor_Ch1_ADC` 03ff92e8 | FFFFF218 → `Current_Sensor_Ch2_ADC` 03ff92ea |
| Scale | 0.8545 A per count (175 A/V) | 0.3052 A per count (62.5 A/V) |
| Clamp after zero | −500 … +200 A | −100 … +150 A |
| **Nominal / default zero** | **687 counts (3.354 V)** | **430 counts (2.100 V)** |
| **Accepted learned zero** | **667 … 708 counts** (3.257…3.457 V, ≈ −17 … +18 A from nominal) | **410 … 450 counts** (2.002…2.197 V, ≈ ±6 A from nominal) |

Positive current = charge [C].

Current calculation (`calc_current_ch1_corrected` 00065da0 / `calc_current_ch2_corrected` 00065e88) [C]:

```
corrected_mcounts = (ADC * 1,000,000 - Zero_Offset_ucounts) / 1000
ch1_mA = corrected_mcounts * 175 * 5000 / 1000 / 1024        (= x 0.8545)
ch2_mA = corrected_mcounts * 0.3052
```

The zero offsets are stored in **micro-counts** (ADC × 1,000,000). The default values are 687,000,000
(`Current_Ch1_Zero_Offset_ucounts` 03ff98dc) and 430,000,000 (`Current_Ch2_Zero_Offset_ucounts` 03ff98e0).
Both are loaded from flash by the C start-up code on every reset (ROM image 0x6c8a4 / 0x6c8a8) [C]. A learned
zero is the mean of up to 10 samples, so it has sub-count resolution.

The final pack current `LB_Current_mA` (03ff9f70, sent in 0x1DB) blends the two channel averages. Ch2 is used
between about −85 and +135 A (`Tbl_Current_Blend_Weight`) and ch1 outside that range. If one channel is
invalid, the other is used alone (`blend_lb_current` 000663bc) [C].

---

## 3. When the null runs

Call chain [C]:

```
main loop (System_State 0x1E, every 10 ms)
 └ run_main_scheduler_10ms 0007597e
    ├ sample_analog_inputs        (fresh ADC values, earlier in the same pass)
    └ run_current_zero_learning 00026800      (only if System_State == its mirror, is_system_state_mirror_ok)
       ├ already done (Current_Zero_Status a82c bit0)?  -> check_current_adc_rails (ongoing checks, §7)
       ├ 0x1D4 HV bit (VCM_Status_Flags 917a bit1) set? -> use DEFAULTS, mark done   (§5.1)
       └ step_current_zero_learning 000267c8
          ├ Current_Zero_Passes_Left a7d5 -= 1       (starts at 10, from flash 0x177bd)
          ├ sample_current_zero 00026132              (one sample if conditions OK, §4)
          └ on the 10th pass: evaluate_current_zero 00026782   (§5)
```

Timing facts:
- **Start:** the counter `Current_Zero_Passes_Left` (03ffa7d5) and the status byte `Current_Zero_Status`
  (03ffa82c) are loaded from flash by the C start-up code on **every MCU reset**: counter = 10, status = 0 [C].
  So learning runs once per power-up or reset. Clearing DTCs does **not** start it again.
- **Window:** 10 consecutive 10 ms passes, ≈ 100 ms. The counter goes down **on every pass, whether or not a
  sample was taken**. Passes where conditions fail are simply lost [C].
- **Observable start:** the same main-loop state (0x1E) sends 0x1DB every 10 ms. The first 0x1DB frame from the
  LBC marks roughly the start of the window. After about 10 frames the zero has been decided [C for the shared
  loop; I for exact frame alignment]. Keep conditions stable for at least 20 frames (200 ms) as margin.
- Boot time before the main loop starts (flash checks, EEPROM/NV load) is not fixed. Use the first 0x1DB frame,
  not a timer from 12 V power-on [I].

---

## 4. Per-pass sampling conditions (`sample_current_zero` 00026132) [C]

A sample of **both** channels is taken on a pass only if **all** of these are true:

| Check | RAM | Meaning | What breaks it |
|---|---|---|---|
| `System_State == 0x1E` | 03ff609a | LBC in normal running state (not shutting down / sleeping) | LBC going to shutdown state 0x28 during the window |
| `Main_Relay_Flags` bit0 == 0 | 03ff6093 | LBC's own main-relay-on flag (= 0x1DB byte3 bit5) | internal fault-state latch; 0 at boot [C], set from fault-state bits 6091/6094 [I] |
| `VCM_Status_Flags` bit1 == 0 | 03ff917a | **0x1D4 byte5 bit6** (descriptor 0x84f35 = byte 5, bit 6, length 1) | emulator sending this bit = 1 |
| `HW_Item_NG_Now_Flags & 0x0D` == 0 | 03ff919d | bit0: items 2..8 / 0x0E NG now (CPU / memory / checksum checks); bit2: item 0x09 ADC self-test NG; bit3: item 0x0B (2.5 V ref out of 2290..2710 mV) or 0x10 (12 V ≤ 7578 mV) NG | low or unstable 12 V; LBC hardware fault |
| `Current_Sensor_NG_Now_Flags` bit2 == 0 | 03ff919e | item 0x11: 5 V sensor supply outside 4711..5273 mV (DTC P33D6) | sensor supply shorted/overloaded (e.g. harness fault) |

Each sample stores `ADC × 1,000,000` per channel in a 10-entry buffer (`Current_Ch1_Zero_Samples` 03ffa7d8 /
`Current_Ch2_Zero_Samples` 03ffa800) and counts it in `Current_Zero_Sample_Count` (03ffa7d4).

The firmware does **not** check whether current is flowing. It assumes current is zero whenever the relay flag
and the 0x1D4 bit are clear. Any current during the window, including precharge, an HV DC/DC, or a charger,
is learned **as the zero offset** for the whole drive cycle. Each count of error is 0.85 A on ch1 and
0.31 A on ch2. The range checks (§5) only catch errors larger than about ±17 A on ch1 and ±6 A on ch2.

---

## 5. Evaluation and results (`evaluate_current_zero` 00026782) [C]

### 5.1 Learning skipped: 0x1D4 HV bit seen first
If `VCM_Status_Flags` bit1 (0x1D4 byte5 bit6) is 1 on any pass **before** learning is marked done,
`run_current_zero_learning` stops and writes the defaults:

- Zero offsets = 687,000,000 / 430,000,000 µcounts; learned ADC (21 01) = 687 / 430 (0x02AF / 0x01AE).
- Marked done. No DTC. The current is then off by the real sensor offset from nominal.

**Check this first:** if your 0x1D4 template has byte5 bit6 set from the first frame (for example a captured
byte5 value such as 0x46 has bit6 = 1), the LBC never learns a zero. It always runs on the defaults.

### 5.2 Too few samples (fewer than 3 of 10)
- Nothing is applied. The offsets stay at the defaults loaded at reset (687 / 430 counts).
- `Current_Zero_Result_Flags` (03ff9198) bit0 = 1, but `Current_Zero_Status` bit0 stays 0. So the ongoing
  ADC rail checks (§7) **do not run** this cycle.
- Learned ADC values in 21 01 stay **0**.
- Later, when 0x1D4 byte5 bit6 becomes 1, the §5.1 branch runs: the defaults are written again, the learned
  ADC changes to 687 / 430, and the rail checks start.
- No DTC.

### 5.3 Three or more samples
1. **Means** (`calc_current_zero_means` 0002653c): `Current_Ch1/Ch2_Zero_Mean_ucounts` (03ffa7cc / 03ffa7d0).
   Learned ADC = mean / 1e6, truncated → `Current_Ch1_Zero_Learned_ADC` 03ffa828 / `Current_Ch2_Zero_Learned_ADC`
   03ffa82a.
2. **Distinct-zeros check**, item 0x98 → **DTC P33FB** (`check_current_zero_consistency` 000265c4). It fails
   when the learned ch1 and ch2 zeros are **within 9 counts of each other** (|ch1 − ch2| ≤ 9). It passes when
   they differ by more than 9. The normal zeros are about 257 counts apart. A failure means both channels show
   the same signal (shorted together, both stuck at a rail, or both unpowered) [C for the rule, I for the causes].
3. **Ch1 range**, item 0x92 → **DTC P33D5**: learned ch1 zero must be 667..708 counts (`check_current_ch1_zero_range`).
4. **Ch2 range**, item 0x95 → **DTC P33FA**: learned ch2 zero must be 410..450 counts (`check_current_ch2_zero_range`).
   - Items 0x92 / 0x95 only count toward a confirmed fault while measurements are valid (`Meas_Valid_Flags`
     5ff8 bit6 = 0) and item 0x98 is not confirmed.
   - All three items use a debounce of 1/1. One judgement confirms OK or NG straight away.
5. **Select and apply** (`select_valid_current_zeros` 00026678, `apply_current_zero_offsets` 000266ee):

| Result | Zero offsets written | Channel state for this cycle | DTC |
|---|---|---|---|
| All OK | ch1 and ch2 = learned means | both valid | — |
| Ch1 out of range, ch2 OK | ch2 only (ch1 keeps default) | **ch1 invalid**; `LB_Current` = ch2 only (clamped −100…+150 A) | P33D5 |
| Ch2 out of range, ch1 OK | ch1 only | **ch2 invalid**; `LB_Current` = ch1 only (coarse) | P33FA |
| Both out of range | none | both invalid | P33D5 + P33FA |
| Zeros within 9 counts | none | **both invalid**, current invalid | P33FB |

"Invalid" is `Current_Zero_Result_Flags` bit1 (ch1) / bit2 (ch2). These feed `Current_Channel_Invalid_Flags`
(03ff98e6) bit1 / bit2 (bit0 = both) through `update_current_channel_invalid` 0004a67e. While a bit is set,
`clamp_current_channels` 00065f74 replaces that channel with 0x80000000. The bits stay set until the next reset.
The channel faults are also latched in `Sensor_Fault_Latch` 911c (bit21 ch1, bit22 ch2, bit6 both or P33FB)
by `latch_current_sensor_faults` 00049a50.

DTC → item lists (shared-DTC table at 0x10104 / 0x1010c) [C]:
- **P33D5** = items 0x92 (ch1 zero range), 0x93 (ch1 ADC high), 0x94 (ch1 ADC low)
- **P33FA** = items 0x95 (ch2 zero range), 0x96 (ch2 ADC high), 0x97 (ch2 ADC low)
- **P33FB** = item 0x98 (ch1 and ch2 zeros not distinct)

---

## 6. Reading back what it used

Service 0x21 (ReadDataByLocalIdentifier) on **request 0x79B / response 0x7BB** (ISO-TP). No session change
or security access is needed [C]. All three responses are multi-frame, so your tool must send ISO-TP flow
control. Values are **big-endian**. Byte index below = position in the response payload (byte0 = 0x61).

### 21 07: zero offsets actually used (14 bytes)
Request `02 21 07`.

| payload bytes | value |
|---|---|
| 0..1 | 61 07 |
| 2..3 | FF FF (unused) |
| **4..7** | **ch1 zero offset, u32 µcounts** (`Current_Ch1_Zero_Offset_ucounts` 98dc) |
| 8..9 | 2.5 V ADC reference, raw counts (512 = 2.500 V) |
| **10..13** | **ch2 zero offset, u32 µcounts** (`Current_Ch2_Zero_Offset_ucounts` 98e0) |

Decode: `zero_counts = value / 1e6`, `zero_volts = zero_counts × 5 / 1024`.
Offset from nominal in amps: ch1 `(value/1e6 − 687) × 0.8545`, ch2 `(value/1e6 − 430) × 0.3052`.
**Exactly 687,000,000 and 430,000,000 means the defaults are in use** (§5.1 / §5.2), or a channel's learned
value was rejected (§5.3).

### 21 01: learned zero (integer counts) and live current (53 bytes)
Request `02 21 01`.

| payload bytes | value |
|---|---|
| 2..5 | ch1 corrected, s32 milli-counts (≈ 0 at rest after a good learn) |
| **6..7** | **ch1 learned zero, u16 ADC counts** (03ffa828) |
| 8..11 | `LB_Current_mA`, s32 (+ = charge; 0x80000000 = invalid) |
| 12..15 | ch2 corrected, s32 milli-counts |
| 43..46 | ch2 10-sample average, s32 mA |
| 47..50 | ch1 10-sample average, s32 mA |
| **51..52** | **ch2 learned zero, u16 ADC counts** (03ffa82a) |

(Payload index = the "buf" column of reports/UDS21_services.md.)

How to read the learned values:
- **0 / 0**: learning failed (fewer than 3 samples) and the 0x1D4 HV bit has not been seen yet.
- **687 / 430 exactly**: almost certainly the defaults (learning skipped). Confirm with 21 07: a real mean is
  rarely an exact whole count.
- **Anything else**: a real learn. Inside 667..708 / 410..450 it was accepted. Outside, that channel was
  rejected (check the DTC).

### 21 03: live raw sensor ADC (28 bytes)
Request `02 21 03`. Payload bytes 4..5 = ch1 raw ADC, 6..7 = ch2 raw ADC (u16, 0x3FF = saturated). Byte 27
(d25) bit0 = main-relay-on flag, bit1 = 0x1D4 HV bit as the LBC sees it. With contactors open, raw ADC
should sit within a count or two of the learned zero.

### DTCs
Read with service 19 (see reports/UDS19_dtc_info.md): P33D5 / P33FA / P33FB as above, and P33D6 for the 5 V
sensor supply. A DTC freeze frame also records both raw current ADCs and the zero offsets
(`capture_dtc_snapshot_data` reads 98dc / 98e0).

---

## 7. After the null: ongoing checks (every 10 ms, once learning is marked done) [C]

`check_current_adc_rails` 00026114 runs items 0x93 / 0x94 / 0x96 / 0x97, each with a 200-pass (2 s) debounce
(descriptor limits 0xC8):

| Item | Rule (NG) | Judged when | DTC |
|---|---|---|---|
| 0x93 | ch1 ADC > 940 counts (> 4.59 V) | measurements valid and 5 V supply OK | P33D5 |
| 0x94 | ch1 ADC ≤ 61 counts (≤ 0.30 V) | same | P33D5 |
| 0x96 | ch2 ADC > 940 counts | same **and** ch1 ADC in 614..817 counts (≈ −62 … +111 A), or ch1 already faulted | P33FA |
| 0x97 | ch2 ADC ≤ 61 counts | same as 0x96 | P33FA |

Ch2 saturates at high current, so it is only checked against its rails while ch1 says the current is moderate.
These are electrical open/short checks. A healthy sensor never trips them. A simulated sensor must keep
both outputs inside 0.30…4.59 V at all times. While ch1 is in its 614..817 count band, ch2 must also stay off
its rails.

---

## 8. Recommended start-up sequence for the emulator

1. Contactors open, precharge off, no HV load or charger. Pack current = 0 A.
2. Apply LBC 12 V (normal 12–14 V; never ≤ 7.6 V) and the wake/ignition signal.
3. From the start, send 0x1D4 with **byte5 bit6 = 0**. Other 0x1D4 content may be normal. Not sending 0x1D4
   for this period also works.
4. Wait for the LBC's 0x1DB frames. Keep step 1 and 3 conditions until **at least 20 frames (≈ 200 ms) after
   the first 0x1DB**. Check that 0x1DB byte3 bit5 (main relay on) is 0 during this time.
5. Optional check: read `21 01` / `21 07` and confirm a learned (non-default, non-zero) zero inside the windows.
6. Only now set 0x1D4 byte5 bit6 and close contactors / precharge.
7. If you want a new zero (for example after a fault): power-cycle or reset the LBC and repeat. A DTC clear alone
   does not start learning again.

---

## 9. Limits of this analysis and open points

- The window length (10 passes × 10 ms) and all thresholds are read from code [C]. The exact time from 12 V
  power-on to the first main-loop pass is not measured. Use the 0x1DB-based timing above.
- How 0x1D4 frames are accepted (counter/CRC check in `FUN_00072684`) was not traced in detail. A frame the LBC
  rejects is ignored, so the bit keeps its last accepted value (0 after reset) [I].
- `Main_Relay_Flags` bit0 comes from internal fault-state latches (6091 / 6092 / 6094), not from a CAN input.
  It is 0 after a reset [C]. Exactly when the LBC sets it during normal running was not traced [I].
- Sensor scaling (175 A/V ch1, 62.5 A/V ch2) is taken from the firmware's conversion constants, not from a
  sensor datasheet.

## Appendix: firmware references

| Function | Address | Role |
|---|---|---|
| run_current_zero_learning | 00026800 | entry; skip-to-defaults on 0x1D4 HV bit; ongoing rail checks once done |
| step_current_zero_learning | 000267c8 | 10-pass counter, sample, evaluate on last pass |
| sample_current_zero | 00026132 | per-pass gate and sample store |
| evaluate_current_zero | 00026782 | ≥ 3 samples → means / checks / select / apply |
| calc_current_zero_means | 0002653c | means and learned ADC |
| check_current_zero_consistency | 000265c4 | item 0x98 / P33FB: zeros within 9 counts → NG |
| check_current_ch1_zero_range | 00026604 | item 0x92 / P33D5: 667..708 |
| check_current_ch2_zero_range | 0002663e | item 0x95 / P33FA: 410..450 |
| select_valid_current_zeros | 00026678 | usable-channel decision |
| apply_current_zero_offsets | 000266ee | writes 98dc / 98e0 |
| check_current_adc_rails | 00026114 | items 0x93/0x94/0x96/0x97 |
| update_current_channel_invalid | 0004a67e | 98e6 channel-invalid bits |
| process_can_rx_1D4 | 000728ac | 0x1D4 byte5 bit6 → 917a bit1 |
| calc_current_ch1/ch2_corrected | 00065da0 / 00065e88 | use of the offsets |

| Variable | Address | Notes |
|---|---|---|
| Current_Zero_Passes_Left | 03ffa7d5 | 10 at reset |
| Current_Zero_Sample_Count | 03ffa7d4 | 0..10 |
| Current_Zero_Status | 03ffa82c | bit0 done, bit1 ch1 usable, bit2 conditions OK this pass, bit3 ch2 usable |
| Current_Zero_Result_Flags | 03ff9198 | bit0 finished, bit1 ch1 unusable, bit2 ch2 unusable |
| Current_Ch1/Ch2_Zero_Offset_ucounts | 03ff98dc / 03ff98e0 | offsets in use (21 07) |
| Current_Ch1/Ch2_Zero_Learned_ADC | 03ffa828 / 03ffa82a | learned counts (21 01) |
| Current_Channel_Invalid_Flags | 03ff98e6 | bit0 both, bit1 ch1, bit2 ch2 |
| Current_Sensor_NG_Now_Flags | 03ff919e | bit0 ch1 rail, bit1 ch2 rail, bit2 5 V supply |
| Item_92..98 status | 03ff612e..03ff6134 | bit7 NG now, bit6 OK now, bit5 NG confirmed, bit4 OK confirmed |
