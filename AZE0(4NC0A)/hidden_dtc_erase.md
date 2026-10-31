# Nissan Leaf AZE0 VCM: hidden DTC erase with the accelerator pedal

The Vehicle Control Module (VCM) of the 2013-2017 Nissan Leaf (AZE0) has a built-in
way to erase its stored fault codes (DTCs) without a scan tool: a short accelerator pedal
sequence right after switching the car on.

This was found by reading the VCM firmware (static analysis of an AZE0 VCM image, part
number suffix 4NL1A). **It has not been tried in a car yet.** Other software versions may
differ. Use it at your own risk.

## What it does

- It erases the VCM's own stored DTCs, the same as the "erase DTC" command (UDS service
  0x14) from a scan tool. It does not touch other modules (BMS/LBC, inverter, charger,
  meter, ...).
- The erase is **not immediate**. The pedal sequence only stores an "erase request" in
  the VCM's non-volatile memory. The VCM carries out the erase at its **next power-up**.
- A fault that is still present will be detected again and come back.

## Conditions

All of these must be true while you do the sequence:

1. **Power ON.** The sequence must be finished within a short window after the power is
   switched on (about 10 seconds, see "Timing"). Switching the power off and on again
   starts a new window.
2. **Shifter in P.** The VCM must see the park position.
3. **Accelerator pedal sensors working.** The VCM checks that the pedal signal is valid.
4. One further vehicle state must not be active. The firmware checks a state code that
   has not been identified yet. If the sequence never works, this is the likely reason.

Start with the power ON but not READY. If that does not work, try it again in READY,
still in P.

## The sequence

| Step | Pedal | Hold |
|---|---|---|
| 1 | Press past **2/3 of travel** (more than about 67%) | about 1 second or more, then release |
| 2 | **Fully released** (below about 5%) | about 1 second or more, then press |
| 3 | Press past **2/3 of travel** again | about 1 second or more |

After step 3 the erase request is stored. Switch the power off, wait a few seconds, and
switch it on again. The VCM erases its DTCs during that start-up.

Details:

- Each hold only counts while the pedal stays in its zone. If it drops out early (for
  example below 2/3 during step 1 or 3), that step's timer starts again.
- If any condition from the list above stops being true, the sequence starts over from
  step 1.
- In practice: switch on, then straight away press firmly (floor it), release completely,
  floor it again. Hold each position for 1-2 seconds and finish everything within about
  8 seconds of switching on.

## Timing

Each hold is 100 cycles of the VCM's control loop, and the whole sequence must be done
within 1000 cycles after power-on. The loop period has not been confirmed from the
firmware. With the usual 10 ms period that is about 1 second per hold and a 10 second
window, which is what this page assumes.

## Is there any feedback?

No dedicated confirmation was found: no chime or flashing lamp tells you that the request
was accepted.

- While a request is stored and not yet carried out, the VCM does not record new faults.
- After the next power-up, the VCM's stored DTCs are gone. The erase also resets the
  warning bits the VCM derives from its DTCs, so a warning that came only from a stored
  DTC may go out (not traced through to a specific dash lamp).
- The reliable way to check is to read the VCM DTCs with a scan tool before and after.

## The same erase from a scan tool

The VCM also accepts UDS ClearDiagnosticInformation in the extended session:

```
Request ID 0x797, response ID 0x79A
10 C0          -> 50 C0
14 FF FF FF    -> 54
```

This uses the same mechanism: the `54` reply only means the request was stored, and the
erase happens at the next power-up. While a request is still waiting, the VCM answers
`7F 14 22`. The 4-byte form is required: the 3-byte `14 FF 00` used by older Nissan
modules is rejected with `7F 14 12`.

## Unknowns

- Not tested in a car.
- One of the conditions (a vehicle-state code) is not identified.
- The exact loop period, so the exact hold and window times.
- Only one firmware version was examined.
