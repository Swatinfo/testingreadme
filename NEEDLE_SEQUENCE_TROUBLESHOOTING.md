# Vial Washer: Needle Water Sequence Reverses After Running a While

## Machine

- Rotary vial washing machine (PETALS), "Flakon Yıkama Odası" (vial washing room 2016)
- PLC: Siemens S7-1200, **CPU 1214C DC/DC/DC** (6ES7 214-1AE30-0XB0)
  - CM 1241 RS485 (communication)
  - SM 1223 DC/DC: 16 DI / 16 DQ (6ES7 223-1BL30-0XB0)
  - SM 1221 DC: 8 DI (6ES7 221-1BF30-0XB0)
  - SM 1231 AI: 4 analog inputs (6ES7 231-4HD30-0XB0)
- Washing media: WFI / purified water / compressed air through the needle manifold, with pumps and tanks on a skid

## Symptom

1. After a start, the machine washes correctly.
2. After it has run for a while, the needles spray water while moving **down**
   when they should spray while moving **up**. The sequence is **reversed by
   exactly half a cycle**.
3. Once it goes wrong, it **stays wrong** until the machine is reset or restarted.

## What the symptom tells us

The fault is **not random**. Random noise or a sticking valve would give
occasional missed sprays. Here the outputs are still in step with the needle
motion, but **180° out of phase**, and the fault **latches**. So the PLC's own
idea of "needle going up" and "needle going down" has drifted away from where
the needles really are.

That nearly always comes from one of these:

| # | Root cause | Probability | Why it fits |
|---|------------|-------------|-------------|
| 1 | **Up/down state is kept by toggling or counting pulses** (one proximity sensor, a counter, a flip-flop, `P_TRIG`, a step counter) and a pulse is **missed or doubled** | **Very high** | One lost or extra pulse turns "up" into "down" for every cycle after it. Nothing ever corrects it, so it stays reversed. |
| 2 | Needle/turret position comes from an **encoder or high-speed counter with no re-zero each cycle** | High | Small counting errors add up until the valve window lands on the wrong half of the stroke. |
| 3 | **Position sensor problem**: loose target or bracket, sensing gap too big, sensor heating up near hot WFI, water on the sensor face, worn cable | High | Works when cold, then starts dropping or chattering pulses after some running time. That feeds cause 1. |
| 4 | **Mechanical slip**: the cam, the sensor flag or a coupling slipping on the shaft (loose grub screw or key), or a timing belt jumping a tooth | Medium | After warm-up and load the cam slips, so the signal moves relative to the real needle position. |
| 5 | **Input filter or scan time**: the pulse is shorter than the DI filter time (default 6.4 ms) or the PLC scan, or the speed setting was raised | Medium | Pulses get missed when the machine reaches full speed. |
| 6 | Valve or pneumatic delay (slow solenoid, low air pressure) | Low | Would *shift* the spray a little. It does not usually make an exact, latched reversal. |

## Step-by-step resolution

### Step 1: Find the needle position input (5 min)

From the electrical drawings or the TIA Portal tag table, find:

- the needle **UP** / **DOWN** position sensor(s), or the cam or encoder input
- the turret index / home sensor
- the water and air valve outputs for the needles

On the hardware, watch the green status LEDs on the CPU 1214C (DI a / DI b) and
on the SM 1223 / SM 1221 rows while the machine runs slowly. Every needle stroke
should give **one clean blink**. Look for blinks that flicker, are missing, or
come twice.

### Step 2: Record the fault with TIA Portal Trace (best evidence)

1. Go online with the CPU, open **Traces**, and add a new trace.
2. Signals to record: needle position input(s), turret home/index input, the
   up/down state bit or step number used in the program, and the water valve
   output(s).
3. Trigger: the valve output goes TRUE while the needle-down signal is active
   (or just record the longest time the buffer allows).
4. Run until the fault appears.

The trace will show exactly when the state flips: a missing pulse, a double
pulse (bounce), or a pulse that is too short.

As a quick alternative, add a counter that counts needle pulses per turret
revolution. It must always be the same number. Any deviation is the fault.

### Step 3: Check the sensors and mechanics (with the machine locked out)

- [ ] Tighten the proximity sensor nuts and brackets. Set the sensing gap to
      about 50–70 % of the rated range (for example 1–1.5 mm on an M12, 2 mm
      sensor).
- [ ] Check the metal target or flag on the needle carriage or cam: is it tight,
      bent, or worn?
- [ ] Check the cam, coupling and timing belt: mark the shaft and cam with a paint
      pen, run for an hour, and see whether the marks have moved apart.
- [ ] Look for water or condensate on the sensor and the M12 connector. Use IP67/IP69K
      stainless sensors in the wash area.
- [ ] Check whether the sensor body gets hot from the hot WFI lines. Many
      standard sensors drift above 60–70 °C.
- [ ] Check the shield and 0 V (M) wiring. Keep sensor cables apart from the pump
      motor and VFD cables.

### Step 4: Check the PLC input settings

In TIA Portal, open **Device configuration → DI → Input filters**:

- Pulse width at full speed: `pulse_ms = (dwell angle of target / 360) × (60000 / strokes per minute)`.
- The filter time must be **well below** the pulse width. A pulse should also
  last **at least 2–3 PLC scan times** (check the cycle time under Online &
  Diagnostics).
- If the pulse is too short, set a lower filter (for example 0.8 ms or 0.2 ms),
  or move the signal to a **high-speed counter (HSC)** or a **hardware interrupt
  (OB40)** so it is never missed.
- Set the filter too low on a noisy signal and it will catch bounce as extra
  pulses. Steps 2 and 3 tell you which way to go.

### Step 5: Fix the PLC logic (permanent fix)

Even with perfect sensors, logic that **toggles a stored up/down bit** will fail
again some day. Change it so the sequence **cannot stay reversed**:

1. **Decide the stroke direction from real position, not from memory.**
   - Best: two sensors, **NEEDLE_UP** (top) and **NEEDLE_DOWN** (bottom). The
     needle is moving up after DOWN switched off and before UP switched on.
   - Or: a cam sensor whose ON-zone covers the up-stroke exactly. Then the water
     valve can be driven straight from the sensor (with a timer for the valve delay).
2. **Re-synchronise every cycle** on the turret or index home sensor: reset the
   step counter or HSC to a known value at every home pulse.
3. **Add a plausibility check**: if UP and DOWN are both ON, or both OFF for
   longer than one stroke time, or the pulse count per revolution is wrong, stop
   the water, raise an alarm **"Needle sequence out of sync"**, and stop the machine.
   That way it can never wash vials wrongly without anyone noticing, which matters
   for GMP.

#### Example SCL (S7-1200, call it in OB1 or in a cyclic OB30)

```pascal
// Inputs  : #NeedleUpSensor  (top end position)
//           #NeedleDownSensor (bottom end position)
//           #HomeSensor (once per turret revolution)
// Outputs : #WaterValve, #SeqFault
// Statics : #StrokeUp (Bool), #R_Down, #R_Up, #R_Home (R_TRIG), #TonPlaus (TON),
//           #PulseCnt (Int)
// Constants: PULSES_PER_REV = number of needle strokes per turret revolution

#R_Down(CLK := #NeedleDownSensor);
#R_Up(CLK := #NeedleUpSensor);
#R_Home(CLK := #HomeSensor);

// Direction comes from the end-position sensors, so it re-syncs on EVERY stroke
IF #R_Down.Q THEN
    #StrokeUp := TRUE;      // needle reached bottom -> next move is UP
    #PulseCnt := #PulseCnt + 1;
END_IF;
IF #R_Up.Q THEN
    #StrokeUp := FALSE;     // needle reached top -> next move is DOWN
END_IF;

// Plausibility: both end positions at once = sensor or mechanical fault
#TonPlaus(IN := #NeedleUpSensor AND #NeedleDownSensor, PT := T#200ms);

// Every turret revolution, check the stroke count and reset it
IF #R_Home.Q THEN
    IF #PulseCnt <> PULSES_PER_REV THEN
        #SeqFault := TRUE;
    END_IF;
    #PulseCnt := 0;
END_IF;

IF #TonPlaus.Q THEN
    #SeqFault := TRUE;
END_IF;

// Water only on the up-stroke, only when the machine runs and the sequence is valid
#WaterValve := #MachineRunning AND #StrokeUp AND NOT #NeedleUpSensor AND NOT #SeqFault;

// #SeqFault is latched: reset only by the operator from the HMI after checking
```

If the machine has only **one** needle sensor and the program toggles a bit on
each pulse, do one of these:
- add a second sensor (recommended), or
- use the turret **home** pulse to force the up/down bit to its known correct
  value every revolution, so a missed pulse corrects itself within one revolution.

### Step 6: Verify

- [ ] Run for at least 2× the time it used to take for the fault to appear, at
      maximum production speed and with hot WFI.
- [ ] Trace and pulse count show no missed or extra pulses.
- [ ] Watch the spray: water starts at the start of the up-stroke and stops before the top.
- [ ] Test the alarm: cover one sensor while running. The machine must stop
      with "Needle sequence out of sync".
- [ ] Record the change (change control / requalification for GMP).

## Quick checklist for the technician

1. Watch the needle sensor LED: one clean blink per stroke?
2. Tighten the sensor, the target, the cam, and the coupling grub screws.
3. Check the input filter time against the pulse width at full speed.
4. Trace the inputs and the valve output in TIA Portal until the fault appears.
5. Change the logic so it re-syncs on real position every stroke or every
   revolution, and add a sequence-fault alarm.

## Information needed to give an exact fix

Please share any of these:
- the TIA Portal project archive (`.zap`) or a screenshot of the needle and valve network(s)
- the I/O list (which DI is the needle sensor, which DQ is the water valve)
- the machine speed (vials/min) and the TIA Portal version
- the CPU **diagnostic buffer** (Online & Diagnostics) after the fault
