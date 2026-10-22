# ZE1 (5SH2D) cell balancing: strategy, thresholds, calibration

Tags: **[C]** read directly from the code; **[I]** inferred (arithmetic or interpretation on top of the code).
Addresses are firmware addresses; all names are in the function / globals logs.

## 1. Summary

- Balancing is **charge-based, not voltage-based** [C]. Once per wake, after the pack has rested, the LBC
  converts every cell's open-circuit voltage to a SOC (via the learned OCV curve), turns that into a charge
  (SOC × capacity), and gives each cell a "charge to remove" equal to its excess over the lowest cell.
- Cells then bleed until a **model** of the bleed current has removed that excess [C]. The model integrates
  cell voltage / resistor, it does not re-measure the imbalance while bleeding.
- The target survives sleep in retained RAM [C]. It is only recomputed when a fresh rested snapshot shows a
  reason to (spread too large, pack charged since, or nothing left to do).
- Bleeding is only allowed while the LBC is in a running state, with no confirmed faults, and with the
  highest cell below a limit: **4150 mV** when not charging, **never during quick charge**, and
  "charge voltage limit minus IR margins" during normal AC charge [C].
- Bleed current is weak: the model works out to about **9 mA at 3.9 V (≈ 420 Ω)** [I]. A 1 % SOC imbalance on a
  115 Ah cell takes about **130 h** of allowed bleed time to remove [I].

## 2. Call structure

```
run_main_scheduler_10ms 0007597e
 └ run_cell_balancing_20ms 0000f8a2          (ticks without the 20 ms slot bit: once per 20 ms)
    ├ run_cell_spread_check 0000f53e         (targeting phase, then accounting phase)
    │  ├ calc_cell_socs_and_charges 0000ec62 (snapshot mV -> SOC -> Cell_Bal_Charge)
    │  ├ check_cell_spread_limit 0000f034    (retarget decision)
    │  │  └ calc_cell_bal_charges 0000ef18   (new targets)
    │  └ run_bal_cell_step 0000f4a8          (accounting + requests, one cell per call)
    │     ├ calc_bal_group_drain_term 0000f104
    │     ├ update_group_bleed_charges 0000f314
    │     │  ├ accum_cell_bleed_charge 0000f252
    │     │  ├ drain_cell_bal_charge 0000f2b6
    │     │  └ track_cell_bal_charge_min 0000f2f2
    │     ├ count_cell_flags_per_group 0000f392
    │     └ calc_cell_bleed_request 0000f44c
    ├ calc_bal_cell_max_limits 0000f714
    │  ├ select_bal_cell_max_limit 0000f622
    │  ├ update_bal_enable_gate 0000f692
    │  └ update_bal_pending_flag 0000f6ec
    └ clear_bleed_requests_if_disabled 0000f85e
```

`Bal_Cell_Index` a435 steps 0..95 once per call, so one pass over all 96 cells takes **1.92 s** [C: slot bit1 is set
every second 10 ms tick by `wait_for_10ms_tick`, and the task runs when it is clear].
Cells are in 8 groups of 12 (`DAT_00084f04` = 12 × 8); group 7 holds cells 0..11, group 0 cells 84..95 [C].

## 3. Inputs: the wake snapshot

- `snapshot_cell_voltages` 00073cc6 copies `Cell_Voltage_mV` into `Cell_Voltage_Snapshot_mV` a5c8 **once per wake**,
  right after the first complete cell-monitor scan, and sets 9194 bit2 [C]. This is the closest thing to a rested
  OCV reading the LBC gets.
- The snapshot is only used for targets when the off time `a04c` (seconds, from the sleep clock) is above
  **`Cal_Bal_Rest_Time_s` fd07c = 360 s (6 min)** [C].
- A cell reading 0xFFFF, or an invalid average temperature (`Temp_Avg_degC` = -128), is counted in
  `Bal_Invalid_Cell_Count` a433; any invalid cell blocks the retarget check for that wake [C].

## 4. Targeting (once per wake)

### 4.1 Reference capacity
`Bal_Ref_Capacity_0p1mAh` a414 = min(`Pack_AH_0p1mAh`, **0x1231D0 = 119.24 Ah**), set once per wake [C].
Pack_AH = 0xFFFFFFFF (invalid) skips targeting and only runs accounting.

### 4.2 Cell SOC and charge (`calc_cell_socs_and_charges` 0000ec62) [C]
For every valid snapshot cell:

```
SOC_i      = interp(V_i, OCV_Curve_mV[63] -> Tbl_OCV_SOC_Axis[63]) / 50      (0..1024, 1024 = 100 %)
charge_i   = SOC_i × a414 / 10 / 1024 × 400                                 (units 1/400 mAh = 2.5 µAh)
             clamp 5.184e7 (129.6 Ah)
```

- `OCV_Curve_mV` 3410 is the learned 63-point curve from the SOH record (21 61), not a fixed table, so the SOC
  conversion follows the pack's learned OCV [C].
- Also records `Bal_Snapshot_Min_mV` 98ce, `Bal_Snapshot_Max_mV` 98d0 and the lowest cell SOC `Bal_Min_Cell_SOC` a42a.
- SOC resolution is 1/1024 ≈ 0.1 %, which is ≈ 112 mAh at 115 Ah; this sets the practical precision of the
  targets [I].

### 4.3 Should the targets be recomputed? (`check_cell_spread_limit` 0000f034) [C]
Runs at the last cell of the targeting pass, only when no cell is invalid. It sets bits in `Bal_Retarget_Reasons` a434:

| Bit | Condition | Meaning [I] |
|---|---|---|
| 2 | max − min snapshot mV > `Tbl_Bal_Spread_mV`(min cell SOC) | imbalance large enough to start over |
| 1 | snapshot min mV > `Bal_Target_Min_Cell_mV` 4ece | pack is higher than when the targets were set (charged since) |
| 3 | `Bal_Group_Pending_Bits` 4ea8 = 0 | previous job finished |

If any bit is set: `calc_cell_bal_charges` builds new targets and the snapshot min mV is stored in 4ece.
If none is set, the remaining charges from before sleep are kept and balancing simply carries on.

### 4.4 Spread limit table
`Tbl_Bal_Spread_mV` 0x103cc (values in the code area), X axis `Ax_Bal_Spread_SOC` fd0ec, result clamped to 4550 mV [C]:

| Min cell SOC | 0 % | 10.0 % | 13.0 % | 15.0 % | 22.0 % | 32.0 % | 48.0 % | 64.0 % | 90.0 % | 95.0 % |
|---|---|---|---|---|---|---|---|---|---|---|
| Axis raw (/1024) | 0 | 102 | 133 | 154 | 225 | 328 | 492 | 655 | 922 | 973 |
| Allowed spread mV | 212 | 212 | 52 | 39 | 57 | 19 | 17 | 29 | 10 | 0 |

The limit roughly follows the OCV slope: wide where OCV is steep (empty), narrow on the flat middle, and
**0 mV at 95 % and up**, so any spread seen at a high rested SOC forces a retarget [I].

### 4.5 New targets (`calc_cell_bal_charges` 0000ef18) [C]

```
Cell_Bal_Charge_Min  a410 = charge of the min-SOC cell (same formula, clamp 5.184e7)
Cell_Bal_Charge_Clamped[i] 4b80 = clamp(max(charge_i, a410), ..., 0x3170400 = 129.6 Ah)
Bleed_Charge_Accum[i]      4d00 = 0
```

The amount a cell has to bleed is therefore `4b80[i] − a410` = (SOC_i − SOC_min) × capacity [I].

## 5. Bleed accounting (every pass, 1.92 s per cell) [C]

At the first cell of each group, for all 12 cells of the group:

```
step_i = Cell_Voltage_Raw_Avg_mV[i] × (bleeding ? 0x793 : 0x6D) >> 11     (1939/2048 or 109/2048)
Bleed_Charge_Accum[i] += step_i + Bal_Group_Drain_Term
while accum > 0x753 (1875): Cell_Bal_Charge_Clamped[i] -= 1 (not below 0), accum -= 1875
Cell_Bal_Charge_Min = min over cells of Cell_Bal_Charge_Clamped
```

- `Bal_Group_Drain_Term` a418 = `Cal_Group_V_Offset[g]` fd0ac + group voltage sum × `Cal_Group_V_Gain[g]` fd0cc >> 16.
  Both tables are **all zero** in stock, so this group-level term is unused [C].
- Scale [I]: one 4b80 unit is 2.5 µAh = 9 mA·s. With one accounting step per 1.92 s:
  - bleeding: I ≈ V(mV) × 0.00237 mA → **9.2 mA at 3.9 V**, equivalent to ≈ 420 Ω;
  - not bleeding: I ≈ V × 0.000133 mA → **0.5 mA at 3.9 V** (≈ 7.5 kΩ), most likely the cell monitor's own draw.
  - All cells drain at the idle rate, so the reference minimum falls too; the closing rate is about 8.7 mA.
- The model runs only while the LBC runs; nothing is integrated during sleep [C: everything is in the 20 ms task].

## 6. Bleed request per cell [C]

`calc_cell_bleed_request` 0000f44c:

```
request_i = Cell_Bal_Charge_Clamped[i] > Cell_Bal_Charge_Min + Cal_Bal_Request_Margin (fd038 = 17)
```

17 units = **42.5 µAh**, effectively zero [I]: a cell bleeds until its modelled excess is gone. There is no
voltage hysteresis band.

`count_cell_flags_per_group` 0000f392 sets bit (7 − cell/12) of `Bal_Group_Pending_Bits` 4ea8 while any cell in that
group is more than `Cal_Bal_Group_Pending_Margin` fd07a = **17** above the minimum.

Path to hardware (see also reports/UDS21_services.md, 21 06) [C]:
`Bal_Request_Bitmap` a5bc (96 bits) → `update_bleed_request_bit` 0003ec96 → `Bleed_Request_Group` b1b4 (8 × 12-bit) →
`Bleed_Command_Group` 63f0 → cell-monitor register 0x1A. The switch states are read back into
`Bleed_Readback_Bitmap` b1cc, which 21 06 reports. The 30 FA bleed test replaces one group word with a test pattern.

## 7. When bleeding is allowed (enable gate)

### 7.1 Gate (`update_bal_enable_gate` 0000f692) [C]
`Bal_Gate_Flags` a432 bit0 = 1 only if all are true:

- both limit values valid (not 0xFFFF; they are 0xFFFF when `Chg_Cell_V_Limit_mV`, `Pack_V_Terminal_Est_10mV` or
  `Cell_R_Learned` is invalid);
- `DTC_Group_NG_Confirmed_Lo` = 0 (no confirmed fault in that DTC group);
- 6087 bit0 clear (meaning not traced);
- `Running_State_Flag` bit0 (System_State 609a = 10 or 0x1E);
- **`Cell_Max_mV` < `Bal_Cell_Max_Limit_mV`** a426.

When the gate is off, `clear_bleed_requests_if_disabled` zeroes the whole request bitmap every call; the
remaining charges are kept, so balancing resumes later where it stopped.

### 7.2 Cell-max limit (`calc_bal_cell_max_limits` 0000f714, `select_bal_cell_max_limit` 0000f622) [C]

With R = `Cell_R_Learned` 98be (R/100 used as mΩ [I]) and V_lim = `Chg_Cell_V_Limit_mV` 97ac:

```
keep   a42e = V_lim − 2.0 × R/100 − 15                      clamp 0..4550 mV
margin a430 = (250000 / Pack_V_Terminal_Est_10mV) × R/100 + 15   clamp 4550 mV
```

[I] 2.0 × R is the IR drop at 2 A; 250000 / pack V (10 mV units) is the current at 2.5 kW, so the margin is the
IR drop at 2.5 kW charging plus 15 mV.

| `CAN_5EC_Charge_Type` | Limit a426 | Effect |
|---|---|---|
| 1 normal charge, 4/5 discharge-integrated sessions (V2X inferred) | start: keep − margin (clamp 0..4550); while balancing: keep | balance during AC charge until the top cell nears the charge voltage limit, with hysteresis |
| 2, 3 quick charge | 0 | **never during quick charge** |
| anything else (0, no charge) | **0x1036 = 4150 mV** | balance while driving / on, unless a cell is above 4.15 V |

## 8. Persistence and resets [C]

- `Cell_Bal_Charge_Clamped` 4b80, `Bleed_Charge_Accum` 4d00, `Bal_Group_Pending_Bits` 4ea8 and
  `Bal_Target_Min_Cell_mV` 4ece are in the retained RAM block (03ff45xx..4fxx). They survive sleep.
- `FUN_00083962` (retained-RAM cold init; also called from `restore_soh_record_from_backup`) clears all four. On
  the next valid rested wake, bits 1 and 3 of the retarget check are both true, so new targets are built.
- `Bal_Target_Done_Flags` a5b8 bit0 marks the end of the targeting phase for the wake (ordinary RAM, so it starts
  at 0 every wake).

## 9. Calibration and constants

| Address | Name | Type | Stock | Role |
|---|---|---|---|---|
| fd07c | Cal_Bal_Rest_Time_s | u16 | 360 s | off time needed before the wake snapshot is used |
| fd038 | Cal_Bal_Request_Margin | u32 | 17 (42.5 µAh) | per-cell bleed request margin over the minimum |
| fd07a | Cal_Bal_Group_Pending_Margin | u16 | 17 (42.5 µAh) | group "still balancing" margin |
| 103cc / fd0ec | Tbl_Bal_Spread_mV / Ax_Bal_Spread_SOC | u16[10] | §4.4 | spread that forces a retarget |
| fd0ac / fd0cc | Cal_Group_V_Offset / Cal_Group_V_Gain | u32[8] / u16[8] | all 0 | group drain term (unused) |
| code 0f586 | (immediate) | u32 | 0x1231D0 = 119.24 Ah | reference capacity clamp |
| code 0ed90 0eebe 0ef72 / 0efa4 | (immediates) | float / u32 | 5.184e7 / 0x3170400 = 129.6 Ah | per-cell charge clamp |
| code 0f252 | (immediates) | — | 0x793 / 0x6D, >> 11 | bleed / idle current factors |
| code 0f2b6 | (immediate) | — | 0x753 = 1875 | accumulator units per 2.5 µAh |
| code 0f622 | (immediate) | — | 0x1036 = 4150 mV | cell-max limit when not charging |
| code 0f714 | (float immediates) | — | 2.0, 15.0, 250000.0, 4550.0 | IR margins |

Notes for capacity patches (reports/capacity_patch.md): the 119.24 Ah clamp at 0f586 and the 129.6 Ah clamps must
scale with the pack, otherwise cells above the clamp all look equal and the large-pack imbalance is
under-estimated.

## 10. Open questions

- 6087 bit0 in the enable gate: not traced.
- System_State values 10 and 0x1E: which vehicle states these are (0x1E also gates `run_cell_extremes_stats`).
- 5925 bit1 ("balancing pending" when 0) is read by `000736b0`, which moves the cell monitor from mode 0x20 to 0x30
  only when no balancing is pending or the RLI02 hold flag is 0. Bleeding itself needs the running state, so this
  looks like a post-run hold rather than post-run bleeding; not confirmed.
- The ≈ 420 Ω / 9 mA figure assumes 0x793/2048 models V/R exactly; it has not been checked against hardware
  (21 06 readback timing plus a current measurement would confirm it).
- `Cal_CellV_Low_Enter` / `_Leave` / `_Floor` (fd080..fd084: 2200 / 3100 / 3500 mV) are labelled "Balancing" in
  reports/cal_tables.md but are used by 0002091c / 000209bc, outside this chain; their role was not checked here.
