# Design

Shared design notes for both actuator sizes. Size-specific parts and
quantities live in [CY-8318](8318/bom.md) and [CY-RO100](ro100/bom.md).

---

## Why cycloidal

A cycloidal disc transfers load differently. The lobes push the pins in
compression along the pin circle, and at any moment roughly a third of
the pins carry load simultaneously. Compression across many contact
points is a far better match for what an FDM part is good at, and the
load per contact stays low enough that the printed lobe profile survives.

What this costs is precision and part count. The eccentric bearing, the
output pin plate, and two discs at 180° all have to be concentric within
a few hundredths of a millimetre or the losses become the dominant term.
Our RO100 build showed this directly: changing nothing but the housing
clamping force moved viscous friction by 52 % and stall efficiency by
17 % (see [Assembly tolerances](#assembly-tolerances)). The eccentric
also adds rotating unbalanced mass that has to be cancelled by the second
disc.

---

## Parameters

The two sizes share one parametric model. Changing the values below is
enough to target a different motor.

| Parameter | Symbol | CY-8318 | CY-RO100 |
|---|---|---|---|
| Pin circle diameter | D_p | 84mm | 84mm |
| Number of pins | N | 13 | 17 |
| Eccentricity | e | 1.5mm | 1.5mm |
| Disc thickness | t | 6mm | 6mm |
| Number of discs | — | 2 | 2 |
| Reduction ratio | i | 12:1 | 16:1 |


Reduction follows `i = N − 1` for a single-disc cycloidal stage with
`N` pins driving `N − 1` lobes.

Two discs are mounted at a 180° phase offset. This cancels the radial
force from the eccentric and keeps the output shaft from orbiting under
load. Skipping the second disc will run, but it vibrates and the output
bearing wears fast.

---

## Encoder

The encoder reads the **motor** shaft, not the output. Absolute joint
position therefore has to be recovered at startup — see
[Calibration](#calibration).

Sensor is an on-board SPI magnetic encoder on the MKS XDrive Mini
(MA702 family, 14-bit). ODrive v0.5.1 configuration:

```python
odrv0.axis0.encoder.config.mode = 257        # SPI_ABS_AMS
odrv0.axis0.encoder.config.cpr = 16384       # 2^14
odrv0.axis0.encoder.config.abs_spi_cs_gpio_pin = 7
```

**Verifying centring.** Run the following and rotate the shaft slowly by
hand through a full turn:

```python
odrv0.axis0.encoder.pos_estimate
```

A well-centred magnet gives a monotonic, evenly-paced sweep. An off-centre
magnet speeds up and slows down once per revolution — the same sinusoid
mentioned above, visible without any instrumentation.

---

## Printing

| Setting | Value |
|---|---|
| Material | PA-CF |
| Nozzle | 0.4 mm (hardened) |
| Layer height | 0.2 mm |
| Walls | 6 |
| Infill | 40 % gyroid |

Dry the filament before printing. PA absorbs moisture from air within
hours and prints with visible stringing and poor layer adhesion once wet.

**Annealing** — 80 °C for 4 h, then cool slowly in the oven. Parts
shrink slightly; the discs and housing are dimensioned with this
accounted for, so do not skip it on one part and not another.

A hardened nozzle is not optional. Carbon fill erodes brass in a few
hundred grams of filament.

!!! note "Thermal limit is the gearbox, not the motor"
    In continuous-load testing the winding current rose by less than 1 %
    over 15 minutes at every load level we tried, while the gearbox
    housing became too hot to hold. On a printed reducer the duty limit
    is set by friction heating in the plastic, not by the motor. Watch
    the housing, not the motor. See [Bench results](#bench-results).

---

## Assembly tolerances

| Metric | Loose | Tightened | Change |
|---|---|---|---|
| Viscous coefficient | 0.529 | 0.254 N·m·s/rad | −52 % |
| Backlash | 15.45 | 10.51 arcmin | −32 % |
| Reflected inertia | 0.505 | 0.419 kg·m² | −17 % |
| Speed tracking error @ 10 rev/s | +3.4 % | +1.1 % | −68 % |
| Direction asymmetry | 3.19 | 1.51 % | −53 % |
| **Breakaway torque** | **2.13** | **3.03 N·m** | **+43 %** |
| **Stall efficiency** | **41.8** | **34.7 %** | **−17 %** |

---

## Calibration

Because the encoder sits on the motor shaft, joint zero has to be found
after every power cycle.

The encoder is single-turn absolute over the **motor** shaft. Through a
12:1 stage, one output revolution spans twelve encoder revolutions, so
the reading alone leaves twelve candidate output positions. The reduction
ratio is exactly the number of candidates you have to disambiguate.

Conversion, once an offset is known:

```
joint_angle_rad = direction * (motor_pos_turn - offset_turn) / i * 2π
motor_pos_turn  = direction * joint_angle_rad * i / (2π) + offset_turn
```

`direction` is ±1 per joint — encoder sign and mechanical sign do not
always agree, and a sign error produces a positive feedback loop that
drives current straight into the limit. If a joint runs away on the first
closed-loop command, check this before anything else.

**Procedure (pose reference).** Place the leg in a defined pose, record
`pos_estimate` for each joint as its offset, write to a config file, and
reload at startup. Requires that the same pose be reproduced on every
power-up.

**Procedure (mechanical limit).** Drive each joint slowly toward a hard
stop under low torque limit, detect the stall, and offset from the known
stop angle. Robust, but only safe with the robot suspended — a leg that
homes against the floor will lift the robot instead.

Storage format we use:

```json
{
  "L_knee": {
    "node_id": 4, "gear": 16, "offset_turn": 8.115,
    "direction": 1, "limit_rad": [0.0, 2.36]
  }
}
```

---

## Bench results

| Metric | CY-8318 | CY-RO100 | Method |
|---|---|---|---|
| Peak torque | 39.4 N·m | 33.2 N·m | Stall step to 40 A / 24 A, 1.5 s hold |
| Continuous torque | ~20 N·m | not measured | 15 min hold, stopped on housing temperature |
| Backlash | 0.39 arcmin | 10.51 arcmin | Bidirectional approach, motor-side encoder |
| Efficiency | 70 % | 34.7 % | Stall, linear-region slope vs ideal K_t·i |
| Mass | 1.7kg | 2.25kg | — |
| Coulomb friction | 1.06 N·m | 1.44 N·m | Speed sweep, current at steady state |
| Viscous coefficient | 0.073 | 0.254 N·m·s/rad | same |
| Breakaway torque | 2.04 N·m | 3.03 N·m | Torque ramp, 5 mN·m steps, both directions |
| Reflected inertia | 0.182 kg·m² | 0.419 kg·m² | Torque step, angular acceleration fit |
| Max speed | 15.7 rad/s | 5.9 rad/s | Output side, no load |

All torques are at the **output** shaft.

### Setup

Torque is measured with a lever arm clamped to the output and bearing on
a 40 kg bar load cell (HX711, Arduino). Arm length 0.42 m from shaft
centre to contact point, giving 165 N·m full scale. Verified by hanging
1 kg at the contact point: expected 4.12 N·m, read 4.2 N·m.

**Load cell mounting dominates the result.** An early stall test read
45 % efficiency; re-bolting the load cell so it could not deflect took
the same measurement to 67 %. If the cell can move, the number is
meaningless. Re-calibrate whenever the contact point moves — relocating
it once changed our scale factor by 40 %.

Friction, inertia and backlash are derived from motor current and encoder
position and do not involve the load cell.

### Peak torque

Stepped current from 2 A upward, 1.5 s at each step, 20 s cooling between
steps, watching the local slope Δτ/ΔI. The CY-8318 held a linear slope to
40 A with no warning triggered; 39.4 N·m is where we stopped, not a
mechanical limit.

The CY-RO100 was different. Slope fell below 35 % of the ideal
`K_t · i` from 15 A onward, and on a separate run to 40 A the output
torque stopped increasing entirely above 24 A — 36 N·m at 21 A,
36.1 at 24 A, and 39.0 at 40 A, i.e. 16 A of extra current bought
3 N·m. Treat ~33 N·m as the usable figure and 26.6 N·m (80 %) as the
number to put in a controller.

### Continuous torque

Held a fixed torque for 15 minutes and logged current.

| Load (motor shaft) | Output torque | Duration | Current rise | Outcome |
|---|---|---|---|---|
| 0.5 N·m | 3.6 → 5.6 N·m | 14.8 min | 1.05 % | completed |
| 1.0 N·m | 19.0 → 19.3 N·m | 14.9 min | 0.66 % | completed |
| 1.5 N·m | 26.3 → 28.7 N·m | 12.2 min | 0.25 % | **stopped, housing too hot** |

Winding temperature is inferred from current rise — copper resistance
climbs as it heats, so holding torque requires more current. That number
stayed under 1 % at every load, meaning **the motor was never the
constraint**. The run was stopped by hand on housing temperature.

We had no thermometer, so "too hot to hold" is the entire basis for the
20 N·m continuous figure. It is a floor, not a characterisation. Anyone
reproducing this should instrument the gearbox housing — that is where
the duty limit lives.

### Speed

Commanded velocity in steps and compared against the encoder estimate.
The CY-8318 tracked 30 rev/s (motor side) with 0.07 % mean error and did
not reach a limit; testing stopped there because decelerating from 30 to
15 rev/s trips `CURRENT_LIMIT_VIOLATION` — braking the reflected inertia
demands more current than accelerating it.

The CY-RO100 tracked cleanly to 15 rev/s and became unstable above 20.

Velocity estimate scatter is worth reporting alongside the mean, because
it is what a controller actually sees:

| Command (motor rev/s) | CY-8318 spread | CY-RO100 spread |
|---|---|---|
| 5 | ±10 % | ±62 % |
| 10 | ±5 % | ±31 % |
| 15 | ±5 % | ±6 % |
| 20 | ±3 % | ±14 % |

Both are worst at low speed, where cycloidal torque ripple is large
relative to the commanded velocity. The CY-RO100 is roughly 6× noisier
throughout, which we attribute to the frameless rotor's weaker support
moving the magnet relative to the sensor.

### Reproducibility

Coulomb friction over five repeats: 1.061 ± 0.016 N·m, CV 1.5 %.
Breakaway torque over ten repeats: 2.04 ± 0.42 N·m, CV 21 % — the spread
is real, not measurement error, and comes from where the disc happens to
stop relative to the pins.

Warm-up does not change the friction magnitude but does change its
symmetry: direction asymmetry fell from 1–4 % cold to under 0.2 % after
running, and the linear fit R² improved from 0.85 to 0.91. Run the
actuator for a few minutes before taking numbers you intend to publish.
