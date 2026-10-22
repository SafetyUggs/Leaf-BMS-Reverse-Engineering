# How the Leaf ZE0 BMS works out SOH

Firmware analysed: the Leaf Li-ion Battery Controller (LBC), software **3NA0D** (24 kWh pack, 65.6 Ah rated).
Every statement below comes from the disassembled firmware unless it is marked **[theory]** or
**[field result]**.

---

## 1. The short version

- The LBC's SOH is **almost entirely a resistance measurement**. It measures how much the pack voltage sags when you
  accelerate, turns that into a resistance figure (**Hx**), and converts Hx into an SOH % through a fixed table.
- The BMS does **not** count Ah to measure capacity in normal use. A capacity path exists, but it only runs if the
  BMS stays powered between a full charge and a rested low battery, moves SOH by just 2.3 % each time, and the next
  resistance reading overwrites it.
- The **Ah figure** LeafSpy shows is **SOH × 65.6 Ah**. It is calculated from SOH, so it always "agrees" with SOH.
  That agreement says nothing about real capacity.
- If resistance has risen faster than capacity has fallen, SOH **under-reads**. The BMS then treats the pack as
  smaller than it is. The remaining-energy figure (gids), SOC and range estimate shrink with it, and the car ends a
  drive with usable energy still in the cells.
- That explains why an SOH reset can give back real range. In real-world testing, a reset has given back about
  **20 %** range [field result].

---

## 2. How SOH is calculated

### 2.1 The resistance path (this is what sets SOH)
It runs while driving, every time these conditions are met:
- not charging, SOC between **40 % and 95 %**, battery temperature **above 10 °C and below 50 °C**;
- regen of at least **10 A**, fading for at least **1.1 s**, that goes straight through zero into acceleration
  rising for **1–2 s** to 10–300 A.

At that moment the LBC fits a straight line through 2–4 voltage/current points (V = OCV − R × I) to get the pack
resistance R in mΩ. Then:

1. **Hx sample:** R is looked up in a 20 × 6 table (resistance × temperature 10/15/25/35/45/60 °C). The output is
   in %; 100 % = a new pack. Higher resistance gives a lower Hx (roughly Hx ∝ 1/R).
2. **Hx average:** Hx moves **1/256** of the way to the new sample each time (about 180 events to move half-way).
   After a reset, the first event sets Hx straight to the sample (section 4).
3. **SOH:** a fixed 9-point table converts Hx to SOH, and SOH is **replaced** by the result. The old SOH gets
   weight 0, so this step does not average.

| Hx % | 15.8 | 24.1 | 38.5 | 54.1 | 62.5 | 74.1 | 82.0 | 100 | 109.1 |
|---|---|---|---|---|---|---|---|---|---|
| SOH % | 31.9 | 45.0 | 60.0 | 74.5 | 80.0 | 87.0 | 91.9 | 100 | 103.0 |

So SOH falls about half as fast as Hx near the top: Hx 82 % gives SOH 91.9 %, Hx 62.5 % gives SOH 80 %. This table
is Nissan's fixed estimate of how much capacity a pack has lost for a given rise in resistance.

### 2.2 The capacity path (rarely matters)
It measures capacity directly: it counts the Ah from a near-full point (end of a long normal charge, or a rested
voltage ≥ 4.08 V/cell) down to a **rested** low point (2.67–3.80 V/cell after a temperature-dependent rest), and
looks up SOH from the Ah counted and the resting voltage. But:
- the counter is wiped at every **cold start** (when the BMS loses power), so the start and end points must fall in
  the same powered period;
- each result moves SOH only **6/256 (2.3 %)** of the way to the measured value;
- the next resistance event (2.1 step 3) replaces SOH again.

In everyday use, SOH is the resistance estimate.

### 2.3 When it takes effect
The new SOH is saved to EEPROM at shutdown and only used from the **next power-up**. At power-up the LBC does:
```
LB_SOH        = stored SOH
Full_Capacity = SOH × 65.6 Ah        (clamped 6.56 … 68.2 Ah)
```

---

## 3. Why an under-reading SOH costs range

SOH is not just a number on the screen. At every power-up the LBC uses it to size the pack:

| What uses SOH | Effect of an SOH that reads too low |
|---|---|
| **Full capacity** = SOH × 65.6 Ah. SOC is counted against it | SOC drops faster per Ah used and reaches 0 % with real charge still in the cells |
| **Remaining energy (gids, CAN 0x5BC)** = SOH × 23,616 Wh × usable fraction | fewer gids at every state of charge, so the car's range estimate and low-battery warnings come early |
| **SOC from cell voltage** (table with an SOH column) | the voltage-based SOC is also read for a smaller pack |
| **Capacity bars** (follow a filtered SOH) | bars lost |
| **Cell resistance model** (divides by Hx) | higher modelled resistance gives tighter power limits |
| **Charge-time estimate** (scaled by SOH) | shorter predicted charge times |

**[theory]** If the real capacity is higher than SOH says, the car runs out of "gids" and SOC before the cells run
out of charge. The difference is driving range that is there but can't be used. Resetting SOH to 100 % makes the
BMS size the pack at 65.6 Ah again, so more of the real charge becomes usable. If real capacity is 96 % and SOH reads
80 %, a reset raises the BMS's idea of the pack by 96/80 = 1.2, or **+20 %**. That matches the field result.

### Why resistance can over-state capacity loss [theory]
- **Resistance and capacity don't age at the same rate.** Heat, cold, calendar time and fast charging all raise
  resistance by different amounts compared with the capacity they cost. The Hx → SOH table is one fixed average
  curve, so a pack that has gained resistance faster than average reads low.
- **The resistance measured is the whole pack's,** including module connections, busbars, cables and contactors.
  Any rise there lowers Hx without any loss of cell capacity.
- **Short pulses exaggerate it.** The 1–2 s step response is dominated by surface and contact effects, which
  grow faster than the loss of stored lithium.
- **Nothing corrects it.** The capacity path is the only direct measurement, and it is too weak and too rarely
  active to pull SOH back up (2.2).

---

## 4. What a reset does, and how long it lasts

The firmware has two diagnostic ways to change SOH. Both need an extended diagnostic session with security access.

| Method | What it writes | How long it lasts |
|---|---|---|
| **31 03 00** (routine "reset SOH") | Hx, SOH, stored and filtered SOH = 100 %, capacity bars = 12, and **clears** the "first estimate done" flags | Usable SOH = 100 % now; capacity at the next power-up. The **first** qualifying resistance event then sets Hx **straight** to the new sample, and SOH to its table value. From the next power-up after that, SOH is back to the resistance estimate. **Typically one drive.** |
| **3B 61** (write SOH data) | Hx, SOH, bars and filtered/stored SOH to any value, plus the two "first estimate done" flags from the request | If the flags are written **set**, Hx averages from the written value at 1/256 per event. It drifts back over **hundreds of events** (about 180 to go half-way), so the reset lasts much longer |

The written values reach EEPROM at the next normal shutdown. Losing 12 V before that undoes them.

### A caution about resets [theory]
A reset only helps if the real capacity is higher than the reported SOH. If the pack really has lost capacity,
setting SOH to 100 % makes the BMS over-estimate it: SOC and gids read high, and the drive ends on the cells'
**low-voltage protection** instead of on the gauge. That leaves less warning before power is cut back (turtle).
After a reset, watch the lowest cell voltage, not just SOC, until you know what the pack really holds. A capacity
test (full charge, rest, constant-current discharge to cut-off, count the Ah) is the only way to know.

---

## Appendix: firmware references (3NA0D)

| Item | Address |
|---|---|
| Resistance step detector, fit | `run_pack_r_step_estimator` 00015ffc, `fit_pack_resistance_lsq` |
| Hx average (1/256, first event = sample) | `update_hx_ema` 00017698 |
| Hx sample table lookup | `calc_hx_sample` 00017838; tables 0003eee0 (temp), 0003eee8 (R axis), 0003ef10 (20 × 6) |
| Hx → SOH (replaces SOH) | `update_soh_from_hx` 00017b32; tables 0003edbc / 0003edd0 |
| Capacity path (6/256) | `update_soh_from_capacity` 00018028, `calc_soh_from_capacity_tbl` 00018158 |
| Power-up: LB_SOH and Full_Capacity = SOH × 65.6 Ah | `load_soh_at_startup` 00017c7a |
| Remaining energy (gids) from SOH | `calc_full_energy_from_soh` 00044764 → 0x5BC |
| Diagnostic reads | 21 01 (`CAN_21_01` 0005a35c: Ah), 21 61 (`CAN_21_61` 0005b746: Hx, SOH, bars) |
| Resets | 31 03 (`CAN_31_03_reset_soh` 00036132), 3B 61 (`CAN_3B_61` 0005cdd2) |
