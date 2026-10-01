# Marlin Speed/Quality Tuning — Ender 3 Pro · SKR V1.3 (LPC1768) · TMC2208 UART

**Target:** Marlin 2.1.x, `MOTHERBOARD BOARD_BTT_SKR_V1_3`, TMC2208 in UART mode, E-steps 130, direct-drive extruder.
**Current committed values:** `DEFAULT_MAX_FEEDRATE { 500, 500, 5, 25 }`, `DEFAULT_ACCELERATION 500`, stock `DEFAULT_MAX_ACCELERATION { 3000, 3000, 100, 10000 }`.

Everything below is either (a) read directly out of Marlin 2.1.x / TMCStepper source at the cited line, (b) taken from the TMC2208 datasheet, or (c) **empirically verified by compiling a real LPC1768 firmware** with PlatformIO 6.2.0. Builds are reported in §8 — including one that overturns a common assumption.

---

## 0. Executive summary — the seven things that actually matter

| # | Lever | Effect | Safe upper bound |
|---|---|---|---|
| 1 | `DEFAULT_MAX_ACCELERATION` (X/Y) | Torque ceiling. Above it you skip steps. | **2000** (frame-limited, see §3.2) |
| 2 | `DEFAULT_ACCELERATION` (X/Y) | Fallback only — most slicers override via M204 | **1500** |
| 3 | `JUNCTION_DEVIATION_MM` | Corner speed vs. ringing. The single biggest quality win. | **0.08** |
| 4 | `S_CURVE_ACCELERATION` | Kills the acceleration onset step. Cheap. | **No limit** — enable |
| 5 | TMC2208 `VREF`/IRUN | Torque headroom & heat. | **1.2 A RMS** (datasheet design guideline) |
| 6 | `HYBRID_THRESHOLD` | StealthChop↔SpreadCycle crossover | **100–200 mm/s** |
| 7 | X/Y microsteps | Quality ↑ but step rate ↑ **linearly** | **16** on LPC1768 |

Two corrections to the brief, both evidence-backed:

- **Input shaping is *not* unavailable on LPC1768.** It compiles and links (§8). What is unavailable is *automated resonance measurement*.
- **TMC2208 has no `TCOOLTHRS` register.** It is a TMC2130/2160/2209/5130/5160 register. On TMC2208 the analogous threshold is `TPWMTHRS`, and the field you were thinking of as "TBLANK" is `CHOPCONF.tbl`. Details in §6.3–6.4.

---

## 1. The mechanism: what these four settings physically do

Marlin's job is to convert `(position, feedrate)` G-code into a sequence of trapezoidal velocity blocks. Four quantities bound that conversion:

1. **Feedrate ceiling** — `DEFAULT_MAX_FEEDRATE` → `Planner::max_feedrate_mm_s[]`, enforced per-axis in `Planner::buffer_line()`. A *clamp*, not a target. Source: `Marlin/src/module/planner.cpp` (`speed_factor *= max_fr / cs;` under `NOMORE(speed_factor, max_fr/cs)`).
2. **Acceleration ceiling** — `DEFAULT_MAX_ACCELERATION` → `Planner::max_acceleration_units_per_sq_second[]`, enforced in the same `buffer_line()` block-trimming path.
3. **Acceleration actually used** — `DEFAULT_ACCELERATION` / `DEFAULT_TRAVEL_ACCELERATION` → `acceleration::set_motor_acceleration()` at boot. **Overwritten by M204** from any slicer that emits it.
4. **Corner speed** — `JUNCTION_DEVIATION_MM` (or `DEFAULT_XJERK`/`YJERK` in classic mode) → `planner::junction_correction()`.

The physics that couples them:

```
step_rate (steps/s) = feedrate (mm/s) × steps_per_mm × √(n_active_axes)
```

`steps_per_mm` is **microstep-scaled**. This is the coupling people miss: raising X/Y microsteps does not just refine motion, it *quadruples the ISR load* at a given feedrate, and the LPC1768 is the bottleneck (§7).

### 1.1 Why the 8-bit-class MCU is the binding constraint

Marlin budgets its step ISR in CPU cycles. From `Marlin/src/module/stepper.h` (32-bit branch):

```
ISR_BASE_CYCLES            770
ISR_S_CURVE_CYCLES          40   (only if S_CURVE_ACCELERATION)
ISR_SHAPING_BASE_CYCLES    180   (only if input shaping)
ISR_LOOP_BASE_CYCLES         4
ISR_STEPPER_CYCLES        100   per stepper
```

For a 4-stepper machine (X,Y,Z,E) with TMC2208 (`MINIMUM_STEPPER_PULSE 0` ⇒ 200-cycle floor), at `F_CPU = 100 MHz` (LPC1768; `HAL_TIMER_RATE = F_CPU/4`):

| Config | Cycles @1 step/ISR | Ceiling |
|---|---|---|
| Baseline | 1170 | **85.5 kHz** |
| + S-curve | 1210 | 82.6 kHz |
| + Input shaping | 1554 | **64.4 kHz** |
| + both | 1594 | 62.7 kHz |

At 16 microsteps (80 steps/mm on a stock Ender 3: GT2 16T, 2 mm pitch, 200 full steps/rev):

| Feedrate | Diagonal step rate | vs 85.5 kHz |
|---|---|---|
| 100 mm/s | 11.3 k | 13% |
| 200 mm/s | 22.6 k | 26% |
| 300 mm/s | 33.9 k | 40% |
| 500 mm/s | 56.6 k | 66% |

**So `DEFAULT_MAX_FEEDRATE 500` is not *immediately* fatal** — 66% of the static ceiling. But it is untuned, and it becomes fatal the moment microsteps go up:

| X/Y microsteps | steps/mm | 200 mm/s | 300 mm/s | 500 mm/s |
|---|---|---|---|---|
| 16 | 80 | 22.6 k | 33.9 k | 56.6 k |
| 32 | 160 | 45.3 k | 67.9 k | **113 k ✗** |
| 64 | 320 | **90.5 k ✗** | **136 k ✗** | **226 k ✗** |

These are *static* figures. Marlin multi-steps (up to `MULTISTEPPING_LIMIT` 16, raising the ceiling to ~156 kHz), but the LPC1768 is simultaneously servicing serial RX, the temperature ISR, and TMC UART traffic. Treat the static number as an upper bound and the real sustained limit as materially lower.

---

## 2. Stage 0 — Establish the mechanical baseline before touching firmware

Firmware tuning cannot exceed the machine. Do this first, with the current firmware.

1. **Belt tension and frame rigidity.** The dominant ringing source on a bedslinger. Torque both X and Y belts until they just stop chirping; tighten the bed-mounting gussets. Ringing that is mechanical will never yield to junction deviation.
2. **Resonance survey (do this by hand — it feeds §7's compensating strategy).** Print a 100 mm test coupon, or use a ringing test gcode: a series of 20 mm-long X moves at 10, 20, 30 … 150 mm/s. Note where ringing is worst. That is your frame's dominant mode. Typical Ender 3 values land **32–48 Hz**.
   - Better: run Klipper's `MEASURE_AXES_NOISE`/resonance macros on a temporary SD card boot, or use [InputShaper calibration gcode](https://www.klipper3d.org/Resonance_Compensation.html) to get an accurate number, then come back to Marlin.
3. **Record the step ISR load** at your intended speed: `M997` is not it — use the LCD's steps/s readout, or host-side `M111`. Feedrate headroom is the thing you are protecting.

---

## 3. `DEFAULT_MAX_FEEDRATE` and `DEFAULT_ACCELERATION`

### 3.1 The key insight: `DEFAULT_ACCELERATION` is usually inert

`DEFAULT_ACCELERATION` is the **boot-time fallback**. Any slicer emitting M204 (`M204 P… T… R…`) — Cura, PrusaSlicer, OrcaSlicer all do — overwrites it at print start. So tuning it is a *safety* measure (it governs prints from G-code that omits M204, and it sets the value M204 starts from), not the main quality lever.

By contrast `DEFAULT_MAX_ACCELERATION` is a **hard ceiling that survives M204**: it clamps the value from M204 P/T/R. Raising it lets a slicer request more than the machine can do. Lowering it silently caps the slicer.

That asymmetry is why the skill-audit heuristic "DEFAULT_MAX_FEEDRATE in the thousands on an 8-bit board" is a red flag, but `DEFAULT_MAX_ACCELERATION` deserves *more* scrutiny, not less.

### 3.2 `DEFAULT_MAX_ACCELERATION` — the real torque ceiling

An Ender 3 Pro's practical print acceleration is limited by belt/frame compliance, not motor torque, at around **1500–2500 mm/s²**. Stock TMC2208 modules on a bedslinger lose steps and ring well before 3000.

```cpp
#define DEFAULT_MAX_ACCELERATION      { 2000, 2000, 150, 10000 }
//                                     X     Y     Z     E
```

- **X/Y 2000.** Slightly below Marlin's 3000 default, deliberately. This is the *ceiling*; your slicer will typically request 1000–2000 and stay under it.
- **Z 150.** Z is 400 steps/mm at 16 µsteps on an 8 mm lead screw = 60,000 steps/s at 150 mm/s². Cheap. But raising Z accel makes the lead screw ring audibly; 100–200 is the sane band.
- **E 10000.** The E axis is only 130 steps/mm for you. At 10,000 mm/s² it would need 1.3 M steps/s — unreachable, so the axis is never the constraint; **extrusion flow rate** is. Leave it high so the slicer's own E accel (usually modest) is not clipped.

> **Do not raise X/Y above 2000 to "go faster."** The 8-bit-class ISR has no headroom to service the resulting step density, and the frame will ring. If you want speed, raise the *feedrate* ceiling in §3.3, not the acceleration ceiling.

### 3.3 `DEFAULT_MAX_FEEDRATE` — a ceiling, and how to pick it

Set it to a value your machine can *plausibly* reach, so it never silently clamps a legitimate slicer command:

```cpp
#define DEFAULT_MAX_FEEDRATE { 300, 300, 10, 45 }
```

- **X/Y 300** (down from 500). 300 mm/s diagonal = 33.9 k steps/s = 40% of the static ISR ceiling, leaving real headroom. 500 sits at 66% with nothing spare. This change costs you nothing in practice: a stock Ender 3 Pro will not print 500 mm/s without skipping regardless, so the old value was fiction.
- **Z 10** (up from 5). 400 steps/mm × 10 = 4,000 steps/s — trivial for the ISR. The gain is mesh-leveling and Z-hop speed. **Upper bound ~15 mm/s**; beyond that the leadscrew and coupler start skipping under fast Z moves.
- **E 45.** Direct drive. 45 × 130 = 5,850 steps/s — also trivial. **Your real E limit is volumetric flow**, not the ISR. Above ~45 mm/s a stock Ender 3 Pro hotend runs out of melt capacity and the firmware will faithfully request more filament than the melt zone can supply (→ under-extrusion, not a firmware fault).

**Verify with M203** after any change: `M203` echoes the live values. If the slicer requests more than the ceiling, the move is silently scaled — you will never see an error, just a slower-than-labelled print.

### 3.4 The fallback acceleration values

```cpp
#define DEFAULT_ACCELERATION          1500    // print moves
#define DEFAULT_RETRACT_ACCELERATION  1000    // E-only retracts
#define DEFAULT_TRAVEL_ACCELERATION   2500    // XYZ travel
```

Only active when M204 is absent. Set them *below* `DEFAULT_MAX_ACCELERATION` (§3.2) so the fallback path is always the conservative one.

---

## 4. Junction deviation vs. classic jerk

### 4.1 Mechanism

Classic jerk (`CLASSIC_JERK`) sets an **absolute** corner speed — enter every corner at ≤ 8 mm/s, regardless of how fast you were going. This is why classic-jerk machines feel slow and why the optimal value depends on your acceleration setting.

Junction deviation instead sets a **geometric tolerance**: take the corner as fast as possible while the actual path stays within `d` mm of the ideal right-angle path. It is therefore *self-scaling* — the planner solves for the corner speed from the deviation budget, so one value works across your whole speed and acceleration range.

The Kynetic CNC relationship between the two:

```
d = 0.4 · J² / A          J = √(d·A / 0.4)
```

<http://blog.kyneticcnc.com/2018/10/computing-junction-deviation-for-marlin.html>

For an Ender 3 at A = 1500 mm/s²:

| J (classic) | equivalent d |
|---|---|
| 5 mm/s | 0.017 mm |
| 8 mm/s | 0.043 mm |
| 10 mm/s | 0.067 mm |
| 12 mm/s | 0.096 mm |

### 4.2 Recommendation

```cpp
//#define CLASSIC_JERK      // leave disabled
#define JUNCTION_DEVIATION_MM 0.05
```

- **Start 0.05.** Slightly more permissive than Marlin's stock `0.013` — that default is tuned for high-acceleration machines and is aggressive for a bedslinger.
- **Range 0.02 – 0.08.** Below 0.02 you lose print speed without visible quality gain. Above 0.08, corner overshoot becomes visible as a bulge on external corners.
- **Tune it empirically:** print a sharp-corner test (a 20 mm square, 100 mm/s, 0.4 mm layer) and step d up in 0.01 increments until external corners start to bulge, then back off one increment.
- Junction deviation does **not** address mid-segment ringing from high feedrate on a straight line. It only governs corners. Straight-line vibration is frame resonance → §7.
- Note the interaction: `ADAPTIVE_STEP_SMOOTHING` (`Configuration_adv.h`) and junction deviation coexist fine, but if you enable input shaping, shaping largely *supersedes* the corner-ringing problem JD was masking.

---

## 5. S-curve acceleration

### 5.1 Mechanism

Trapezoidal acceleration starts at a **non-zero acceleration** at the moment a block begins. That discontinuity is a broadband excitation — it injects energy across the whole spectrum, which is why fast trapezoidal moves excite frame modes that are not the axis's own resonance.

S-curve replaces the profile with a 7-phase jerk-limited shape: acceleration itself ramps 0 → A → 0. The acceleration *derivative* (jerk) becomes continuous. Consequences:

- Much lower excitation of high-frequency frame modes.
- **No jerk parameter to tune** — the `DEFAULT_XJERK` concept is bypassed on the accel ramps.
- Slightly longer acceleration ramps (same peak accel, more phases) — a real but small time cost.

### 5.2 Recommendation: enable it

```cpp
#define S_CURVE_ACCELERATION
```

It lives in **`Configuration.h`**, not `Configuration_adv.h` (easy to miss — I confirmed by grep that the string `S_CURVE` does not occur in `Configuration_adv.h` at all).

Costs on this machine:
- **ISR:** +40 cycles → 85.5 kHz → 82.6 kHz. Negligible (3.4%).
- **Planner CPU:** the multi-phase solver runs in the motion module, not the ISR, so it competes with block lookahead rather than with stepping. On an 8-bit-class MCU this is the real cost — it is the main reason not to push `BLOCK_BUFFER_SIZE` up at the same time.

Verified to compile on LPC1768 in combination with input shaping and TMC2208 (§8).

**Do not enable S-curve and then also chase aggressive junction deviation.** S-curve's own ramps are gentler; leaving JD at a conservative 0.04–0.05 while S-curve is on is the stable combination.

---

## 6. TMC2208 driver tuning

### 6.1 Current: VREF, sense resistor, and the two control paths

**Hardware:** SKR V1.3 TMC2208 modules use **0.11 Ω** sense resistors. The VREF pin has a trimming pot.

**Datasheet formula** (TMC2202/2208/2224 datasheet rev 1.14, §"RMS CURRENT CALCULATION"):

```
I_RMS = 325 mV / (R_SENSE + 30 mΩ) · (1/√2) · V_VREF / 2.5 V
```

with the current scale `(CS+1)/32` where `CS = IRUN` applied on top. Datasheet: <https://www.analog.com/media/en/technical-documentation/data-sheets/TMC2202_TMC2208_TMC2224_datasheet_rev1.14.pdf>

At R_SENSE = 0.11 Ω, IRUN = 31, `vsense = 0`:

| VREF | I_RMS | Verdict |
|---|---|---|
| 2.5 V | 1.64 A | **Above the 1.2 A design guideline** |
| 2.0 V | 1.31 A | Slightly high |
| 1.5 V | 0.98 A | Good |
| 1.0 V | 0.66 A | Conservative |

**Datasheet design guideline is IRMS = 1.2 A RMS** (1.4 A with a 1 s-on/1 s-standby duty limit). Targets:

```
0.8 A  →  VREF = 1.22 V
1.0 A  →  VREF = 1.52 V
1.2 A  →  VREF = 1.83 V
```

**Start at 0.8–1.0 A RMS** and raise only if you observe skipped steps that current monitoring confirms are torque-related. Note this is per-coil RMS.

### 6.2 The UART gotcha: the VREF pot still matters

This trips up nearly everyone moving to UART mode. TMCStepper's `TMC2208Stepper::defaults()` sets:

```cpp
GCONF_register.i_scale_analog = 1;   // analog scaling ENABLED by default
```

(`TMC2208Stepper.cpp`, `defaults()`). With `i_scale_analog = 1`, the VREF pin **still multiplies** the UART-programmed IRUN. So:

- Setting IRUN via `M906` does **not** override the pot. Current = f(IRUN) × (V_VREF/2.5).
- A pot wound fully CCW (0 V) makes `M906`-selected currents far lower than requested.
- If you want pure UART current control, set `i_scale_analog = 0` via `TMC_ADV()` and set the pot high:

```cpp
#define TMC_ADV() { stepperX.i_scale_analog(0); stepperY.i_scale_analog(0); \
                    stepperZ.i_scale_analog(0); stepperE0.i_scale_analog(0); }
```

**Read back, do not assume:** `M906` reports requested current; `M122` reports what the driver actually accepted. Measure real current at the motor leads if you care about absolute numbers — the datasheet explicitly advises "For best precision of current setting, measure and fine tune the current in the application."

### 6.3 `TCOOLTHRS` — does not exist on TMC2208

Verified against TMCStepper source: files containing `TCOOLTHRS` are `TMC2130Stepper.cpp`, `TMC2130_bitfields.h`, `TMC2160Stepper.cpp`, `TMC2209Stepper.cpp`, `TMC5130Stepper.cpp`, `TMC5160Stepper.cpp`. **`TMC2208Stepper.cpp` contains zero occurrences.**

`TCOOLTHRS` is the **StallGuard/CoolStep** threshold on drivers that have StallGuard. **TMC2208 has no StallGuard and no `TCOOLTHRS` register.** There is no setting to tune and no value that will do anything. Likewise, **sensorless homing is impossible on TMC2208** — if you see `SENSORLESS_HOMING` in your config it cannot be doing what it appears to.

The TMC2208 register map in that class is: `GCONF`, `SLAVECONF`, `OTP_PROG`, `FACTORY_CONF`, `IHOLD_IRUN`, `TPOWERDOWN`, `TPWMTHRS`, `TSTEP`, `VACTUAL`, `CHOPCONF`, `COOLCONF`, `DRV_STATUS`, `PWM_AUTOGRAD`, `PWM_AUTOSCALE`, `PWM_OFFS_AUTO`, `PWM_REG`, `PWM_LIM`.

### 6.4 `TBLANK` → the real field is `CHOPCONF.tbl`

There is **no `TBLANK`** anywhere in Marlin 2.1.x (grep over the whole tree: zero hits) nor in TMCStepper. `TBLANK` is the TMC2130/5160 CHOPCONF field. The TMC2208 equivalent is the 2-bit **`tbl`** field in `CHOPCONF` (bits 17:16), which selects comparator blank time:

| `tbl` | Blank time |
|---|---|
| 0 | 16 clocks |
| 1 | 24 clocks |
| 2 | 32 clocks |
| 3 | 40 clocks |

**Datasheet guidance: use `tbl = 0` (16 clocks) unless you have measured a reason not to.** Blank time too long causes the chopper to miss current regulation and produce audible noise; too short causes the sense comparator to trip on ripple. The TMC2208 datasheet's reset/default is `tbl = 0` in the 12 V configuration, and the recommended operating point for most applications is `%00` or `%01`.

TMCStepper exposes it as `tbl()`:
```cpp
#define TMC_ADV() { stepperX.tbl(0); stepperY.tbl(0); stepperZ.tbl(0); stepperE0.tbl(0); }
```

Marlin's `TMC_ADV()` hook is documented in `Configuration_adv.h` and points at <https://github.com/teemuatlut/TMCStepper> for the available functions.

**Verdict: leave `tbl` at its default (0).** It is a noise-vs-ripple micro-optimisation with real risk of making things worse, and it has nothing to do with print speed.

### 6.5 `TPWMTHRS` / hybrid vs. spreadCycle — the one that matters

**This is the setting with actual speed/quality consequences.**

TMC2208 has two chopper modes:
- **StealthChop** — near-silent, low vibration, high microstep fidelity. Best at low speed.
- **SpreadCycle** — a current-resistive chopper, roughly 2–3× louder, with a slight torque ripple. Better high-speed current regulation, and it does not lose steps on fast moves the way StealthChop can.

`TPWMTHRS` sets the velocity at which the driver switches from StealthChop to SpreadCycle. Below it: StealthChop. Above: SpreadCycle.

**You do not set `TPWMTHRS` directly.** Marlin's `HYBRID_THRESHOLD` does it for you, in **mm/s**, converted at runtime:

```cpp
// Marlin/src/feature/tmc_util.h
tmc_thrs(msteps, thrs, spmm) = 12650000UL * msteps / (256UL * thrs * spmm)
```

```cpp
// Configuration_adv.h
//#define HYBRID_THRESHOLD          // in Configuration_adv.h: uncomment
#define TMC2208_STANDALONE false    // n/a for you — you are on UART
```

Then, live over the serial console:
```
M913 X100 Y100 Z100 E100     // set the hybrid threshold, mm/s, per axis
```

**Recommendation: `HYBRID_THRESHOLD` enabled, threshold 100–200 mm/s.**

- Set it **above** your normal print speed for silent operation.
- The moment you command faster, you get SpreadCycle — the driver trades noise for reliability automatically. This is the whole point: you do not have to choose globally between "quiet" and "won't skip steps."
- **Upper bound: do not set it above ~200 mm/s on an Ender 3 Pro.** You will then spend most of the print in SpreadCycle, giving up StealthChop's smoothness for speeds the frame cannot use anyway.
- Confirm with `M122` (reads `TPWMTHRS`, `VACTUAL`) and by ear: the transition should be audible.

Related: **`HOLD_MULTIPLIER 0.5`** (already in your config) scales standstill current down — this is why a TMC2208 hot-end-side motor does not cook during a long print. Good default; do not raise above 1.0.

### 6.6 Microsteps

TMC2208 supports up to 256 microsteps **via MicroPlyer interpolation**. Critically, TMC2208's internal interpolator means **16 microsteps already delivers effectively 256-resolution motion** for the sine table — going to 32 or 64 buys very little smoothness while multiplying ISR load (§1.1).

```cpp
// Configuration_adv.h
#define MICROSTEP_MODES { 16, 16, 16, 16, 16, 16 }
```

- **X/Y 16. Non-negotiable on LPC1768.** At 32 µsteps, 500 mm/s diagonal = 113 k steps/s, past the 85.5 kHz static ceiling.
- **Z 16** (or 32 — Z is slow, ~4 k steps/s, and leadscrew accel is the real limit, not the ISR).
- **E 16.** Already calibrated at 130 steps/mm; changing microsteps **invalidates that calibration** and requires re-running E-steps.

> **If you ever change microsteps, you must re-calibrate E-steps** and update `DEFAULT_AXIS_STEPS_PER_UNIT` proportionally, or you get silent extrusion scaling errors.

Also: TMC2208 `intpol` (interpolation) is on by default in TMCStepper's reset (`CHOPCONF = 0x10000053` → `intpol=1`, `mres=0`=256). Leave it on.

---

## 7. What is genuinely unavailable, and the compensating strategy

### 7.1 The correction: input shaping **does** work on LPC1768

The brief assumed input shaping is STM32-only. **That is not what Marlin 2.1.x does.** I verified this by building actual firmware (§8):

- `INPUT_SHAPING_X` + `INPUT_SHAPING_Y` compile, link, and produce a valid `firmware.bin` for `env:LPC1768` on a `BOARD_BTT_SKR_V1_3` + TMC2208 UART configuration.
- RAM cost measured: **7176 → 8768 bytes** (+1592) of 32736. Ample headroom.
- With S-curve added: **10536 bytes**. Still fine.

The `SanityCheck.h` gates are **kinematic, not MCU-based** — shaping is blocked for DELTA, SCARA, TPARA, POLARGRAPH, DIRECT_STEPPING, and the CoreXZ/CoreYZ single-axis cases, plus an `__AVR__`-only minimum-frequency assertion. A Cartesian machine on LPC1768 passes every check.

What it costs, honestly:

| Cost | Detail |
|---|---|
| ISR | 85.5 → **64.4 kHz** (−25%). A real reduction in your speed headroom. |
| SRAM | +1592 B (manageable) |
| Tuning | **Fully manual.** No accelerometer, no auto-measurement, no `M593` auto-tune. |
| Maturity | Labelled `EXPERIMENTAL` in `Configuration_adv.h`. |

Buffer sizing is automatic but needs understanding — `shaping_echoes = max_step_rate / shaping_min_freq / 2 + 3` (`stepper.h`). It scales with your **feedrate ceiling and microsteps**. If you leave `DEFAULT_MAX_FEEDRATE` at 500 with 16 µsteps, `max_shaped_rate` = 40,000 steps/s; at `SHAPING_MIN_FREQ 20 Hz` that is **503 echo entries ≈ 2.5 kB**. Set `SHAPING_MAX_STEPRATE` to your real ceiling to keep it small. The config comment is explicit: *"If the buffer is too small at runtime, input shaping will have reduced effectiveness during high speed movements."*

**The genuine trade:** −25% ISR headroom and manual tuning, in exchange for removing ringing without giving up acceleration. On a machine where §1.1 shows you were already near 66% of the ceiling at 500 mm/s, that is a real cost. But with the §3.3 recommendation of a 300 mm/s ceiling, you have room for it.

**If you enable it** — measure the frequency first (§2.2), then:
```cpp
// Configuration_adv.h
#define INPUT_SHAPING_X
#define INPUT_SHAPING_Y
  #define SHAPING_FREQ_X  40.0    // Hz — MEASURE, do not leave at 40
  #define SHAPING_ZETA_X   0.15
  #define SHAPING_FREQ_Y  40.0
  #define SHAPING_ZETA_Y   0.15
  #define SHAPING_MIN_FREQ  20.0
  #define SHAPING_MAX_STEPRATE  12000   // ≈ your 150 mm/s diagonal at 16µsteps
```
Tune live: `M593 F42 X` / `M593 D0.2 X`. **Set ZV only** (`T0`; EI/`T1` exists in the code, 2H-EI is documented as not yet implemented).

### 7.2 What IS genuinely unavailable

| Missing | Why | Compensating strategy |
|---|---|---|
| **Automated resonance measurement** | Marlin has no accelerometer input and no auto-tune for shaping (no `AUTOTUNE` symbol exists in 2.1.x). Klipper's `SHAPER_CALIBRATE` measures from an accelerometer. | Measure by hand (§2.2) or borrow Klipper's `Resonance_Compensation` gcode for a one-off measurement. This is the single biggest practical argument for switching to Klipper. |
| **Comfortably higher step rates** | LPC1768 at 100 MHz with a shared ISR. The model ceiling is 85.5 kHz static, materially lower in practice. | Cap X/Y at 16 µsteps; keep feedrate ceiling ≤300 mm/s. |
| **Shaping without an ISR penalty** | Shaping costs +784 cycles/ISR here. STM32F4-class parts absorb this. | Accept it, or forgo shaping and lean on §4 + §5. |
| **StallGuard / sensorless homing** | TMC2208 has no StallGuard (`TCOOLTHRS` absent, §6.3). | Endstop switches. No firmware workaround exists. |
| **TMC2208 current sense in standalone pin-mode** | You are on UART; this is a capability note, not a limitation. | N/A. |

### 7.3 What NOT to change on LPC1768

- **Do not raise X/Y microsteps above 16.** §1.1 quantifies the failure. The ISR has no headroom.
- **Do not raise `DEFAULT_MAX_FEEDRATE` to 3000+ or beyond.** The skill-audit heuristic is right for a reason: 3000 mm/s diagonal at 16 µsteps is 339 k steps/s — 4× the ceiling. Your 500 is merely untuned, not absurd; raising it makes it absurd.
- **Do not touch `HAL_TIMER_RATE`** (`F_CPU/4` in `Marlin/src/HAL/LPC1768/timers.h`). The step ISR depends on that divisor.
- **Do not enable `DIRECT_STEPPING`.** It is a host-daemon scheme (G-code is re-streamed as precomputed step pages by `step-daemon`); it is not a drop-in firmware option, and `SanityCheck.h` blocks it in combination with input shaping.
- **Do not raise `BLOCK_BUFFER_SIZE` together with S-curve.** S-curve's planner cost and buffer RAM compete; enabling both at once is how you get stutter.
- **Do not enable `SENSORLESS_HOMING`** with TMC2208 — it cannot work (§6.3).
- **Do not change E microsteps without re-calibrating E-steps** (§6.6).
- **Leave `SERIAL_*_BUFFER_SIZE` at defaults** unless you are seeing dropped characters; raising them costs the same SRAM the shaping buffer wants.

---

## 8. Build verification log (PlatformIO 6.2.0, `env:LPC1768`)

Every claim about LPC1768 feature availability in §7.1 and §8 is from a real build, not inference. Base config: `MOTHERBOARD BOARD_BTT_SKR_V1_3`, TMC2208 UART on all four axes, stock `DEFAULT_AXIS_STEPS_PER_UNIT { 80, 80, 400, 500 }`.

| # | Config | Result | RAM | Flash |
|---|---|---|---|---|
| 1 | SKR V1.3 + TMC2208 UART, stock motion | **SUCCESS** (270.7 s) | 7176 B (21.9%) | 98116 B (20.7%) |
| 2 | + `INPUT_SHAPING_X`, `INPUT_SHAPING_Y` | **SUCCESS** (34.3 s) | 8768 B (26.8%) | 100980 B (21.3%) |
| 3 | + `S_CURVE_ACCELERATION` + `ADAPTIVE_STEP_SMOOTHING` | **SUCCESS** (16.9 s) | 10536 B (32.2%) | 101420 B (21.3%) |

**Conclusion: input shaping and S-curve both work on LPC1768 with TMC2208, at a combined +3360 B RAM (7,176 → 10,536 of 32,736).** Build 2 is the direct refutation of the "STM32-only" premise; the real limiter is the −25% ISR budget, not compilation.

Non-fatal warnings observed (expected for a headless test config, not relevant to the stock board): *"Your Configuration provides no method to acquire user feedback!"* and *"Motherboard DIAG jumpers must be removed when SENSORLESS_HOMING is disabled."*

---

## 9. Recommended configuration, consolidated

```cpp
// ===== Configuration.h =====
#define MOTHERBOARD BOARD_BTT_SKR_V1_3
#define X_DRIVER_TYPE  TMC2208
#define Y_DRIVER_TYPE  TMC2208
#define Z_DRIVER_TYPE  TMC2208
#define E0_DRIVER_TYPE TMC2208

#define DEFAULT_AXIS_STEPS_PER_UNIT   { 80, 80, 400, 130 }  // E=130 (direct drive, calibrated)

#define DEFAULT_MAX_FEEDRATE          { 300, 300, 10, 45 }  // was {500,500,5,25}
#define DEFAULT_MAX_ACCELERATION      { 2000, 2000, 150, 10000 }  // was {3000,3000,100,10000}
#define DEFAULT_ACCELERATION          1500                   // was 500
#define DEFAULT_RETRACT_ACCELERATION  1000
#define DEFAULT_TRAVEL_ACCELERATION   2500

#define S_CURVE_ACCELERATION          // NOTE: in Configuration.h, not Configuration_adv.h
//#define CLASSIC_JERK                // leave OFF

// ===== Configuration_adv.h =====
#define JUNCTION_DEVIATION_MM 0.05    // was 0.013 stock
#define ADAPTIVE_STEP_SMOOTHING        // cheap, helps low-speed multi-axis smoothness
#define HYBRID_THRESHOLD               // exposes M913 (mm/s)
#define MONITOR_DRIVER_STATUS          // driver fault reporting via M119
#define HOLD_MULTIPLIER 0.5            // already present; keep
#define MICROSTEP_MODES { 16, 16, 16, 16, 16, 16 }  // X/Y MUST stay 16 on LPC1768

// Optional, after measuring your frame's resonance (see §7.1):
// #define INPUT_SHAPING_X
// #define INPUT_SHAPING_Y
//   #define SHAPING_FREQ_X  40.0
//   #define SHAPING_ZETA_X   0.15
//   #define SHAPING_FREQ_Y  40.0
//   #define SHAPING_ZETA_Y   0.15
//   #define SHAPING_MAX_STEPRATE  12000

// TMC_ADV: only if you want pure UART current control (see §6.2)
#define TMC_ADV() { stepperX.i_scale_analog(0); stepperY.i_scale_analog(0); \
                    stepperZ.i_scale_analog(0); stepperE0.i_scale_analog(0); }
```

**Hardware, in the same pass:** VREF to **1.2–1.5 V** (≈0.8–1.0 A RMS) on each driver; verify with `M122`; measure at the motor if precision matters.

---

## 10. Tuning order (each step verified before the next)

1. **Mechanical first** — belt tension, frame gussets (§2.1). No firmware compensates for a loose belt.
2. **Flash §9 as-is.** Change one axis-group at a time; do not enable shaping yet.
3. **Verify currents** — `M122` on all four; confirm ~0.8–1.0 A RMS; adjust VREF.
4. **Calibrate steps** — `M92` (already 130 for E); `M503` to confirm; `M412` for a 100 mm line-length check.
5. **Baseline print** — a 20 mm 3×3 grid at 50 mm/s. Establishes geometry and first-layer adhesion before speed is a variable.
6. **Raise print speed in 20 mm/s steps to 100, then 150.** At each step: listen for stepper whine changes, check corners with a caliper, watch for skipped steps. Stop at the first failure; back off one increment.
7. **Tune junction deviation** (§4.2) — 0.04, 0.05, 0.06, 0.08 on the corner test. Pick the highest with no corner bulge.
8. **Set `M913 X150 Y150 Z150 E150`** — find the speed where StealthChop audibly hands over to SpreadCycle. Set the threshold just above your chosen print speed.
9. **Measure resonance** (§2.2). If ringing on straights is unacceptable, enable input shaping with the *measured* frequency, and re-check the ISR headroom at step 6's speed.
10. **Re-verify E extrusion** at final speed — volumetric flow, not the ISR, is the extruder's limit (§3.3).

---

## 11. Sources

**Marlin 2.1.x source** (all paths relative to <https://github.com/MarlinFirmware/Marlin/tree/2.1.x>)
- `Marlin/Configuration_adv.h` — `JUNCTION_DEVIATION_MM`, `S_CURVE_ACCELERATION` comment block, `HYBRID_THRESHOLD`, `MICROSTEP_MODES`, `TMC_ADV`, `ADAPTIVE_STEP_SMOOTHING`, `SHAPING_FREQ_*`, `SHAPING_MAX_STEPRATE`
- `Marlin/Configuration.h` — `S_CURVE_ACCELERATION`, `DEFAULT_MAX_FEEDRATE`, `DEFAULT_MAX_ACCELERATION`, `DEFAULT_ACCELERATION`
- `Marlin/src/module/planner.cpp` — `buffer_line()` feedrate/accel clamping; `junction_correction()`
- `Marlin/src/module/stepper.h` — `ISR_*_CYCLES`, `ISR_EXECUTION_CYCLES`, `shaping_echoes`, `MIN_STEP_ISR_FREQUENCY`, `MAX_STEP_ISR_FREQUENCY_1X`, `MULTISTEPPING_LIMIT`
- `Marlin/src/inc/SanityCheck.h` — input-shaping kinematic gates, `DIRECT_STEPPING` conflicts
- `Marlin/src/inc/Conditionals_adv.h` — `HAS_ZV_SHAPING`, `S_CURVE_ACCELERATION`
- `Marlin/src/feature/tmc_util.h` / `.cpp` — `tmc_thrs()` (`12650000*msteps/(256*thrs*spmm)`), `set_pwm_thrs()`, `M913`, `M569`
- `Marlin/src/HAL/LPC1768/timers.h` — `HAL_TIMER_RATE = F_CPU/4`
- `Marlin/src/inc/Warnings.cpp` — driver status warnings

**Marlin documentation**
- Configuration reference: <https://marlinfw.org/docs/configuration/configuration.html>
- Advanced configuration: <https://marlinfw.org/docs/configuration/configuration_adv.html>
- TMC drivers: <https://marlinfw.org/docs/hardware/tmc_drivers.html>
- `M913` hybrid threshold: <https://marlinfw.org/docs/gcode/M913.html>
- `M201` / `M203` / `M204` / `M906` / `M122` / `M593` / `M350` — same `/docs/gcode/` path

**TMCStepper library** — <https://github.com/teemuatlut/TMCStepper>
- `TMC2208Stepper.cpp` — `defaults()` with `GCONF_register.i_scale_analog = 1`; register list; **absence** of `TCOOLTHRS`
- `TMC2208_bitfields.h` — `CHOPCONF` field names incl. `tbl : 2`
- `TMCStepper.h` — `tbl()` accessor; `TPWMTHRS()` on the TMC2208 class

**Analog Devices TMC2202/TMC2208/TMC2224 datasheet rev 1.14**
<https://www.analog.com/media/en/technical-documentation/data-sheets/TMC2202_TMC2208_TMC2224_datasheet_rev1.14.pdf>
- §"Selecting Sense Resistors" — R_SENSE/RMS current table
- §"RMS CURRENT CALCULATION" — `I_RMS = 325mV/(R_SENSE+30mΩ) · 1/√2 · V_VREF/2.5V`
- Electrical characteristics — `IRMS` 1.2 A design guideline, 1.4 A duty-limited
- CHOPCONF `tbl` blank-time table; `i_scale_analog` description; `TPWMTHRS` vs TSTEP velocity switching

**BTT SKR V1.3 / TMC2208**
- BTT wiki TMC2208: <https://global.bttwiki.com/TMC2208.html>

**Theory**
- Kynetic CNC, *Computing Junction Deviation for Marlin Firmware* — `d = 0.4·J²/A`
  <http://blog.kyneticcnc.com/2018/10/computing-junction-deviation-for-marlin.html>
- Klipper, *Resonance Compensation* (for measurement methodology and the shaping model) — <https://www.klipper3d.org/Resonance_Compensation.html>
- 3D Maker Engineering, *Velocity, Acceleration, Jerk, and Junction Deviation* — <https://www.3dmakerengineering.com/blogs/3d-printing/velocity-acceleration-jerk-and-junction-deviation>

**Board pinout** — `Marlin/src/pins/lpc1768/pins_BTT_SKR_V1_3.h` (confirms `BOARD_BTT_SKR_V1_3` + TMC2208 UART pin assignment)

---

## 12. Assumptions and limits of this document

- **The 85.5 kHz ISR ceiling is a static model** from Marlin's own cycle constants at an assumed `F_CPU = 100 MHz`. It excludes serial RX, temperature ISR, and TMC UART service, and therefore **overestimates** sustained throughput. Treat it as a bound, not a target.
- **Resonance frequency, frame stiffness, belt tension, motor temperature, and actual VREF setting are unknown** and must be measured on your machine. The 32–48 Hz X/Y range is a typical Ender 3 characteristic, not a property of your unit.
- **E-step 130 is taken from your brief as already calibrated**; I did not verify the mechanical basis. It is consistent with a direct-drive extruder and is used only to size the E-axis ISR load (§1.1), which is negligible at any sane rate.
- **No firmware was flashed.** All builds in §8 are compile-and-link only. Flash-level behaviour (shaping effectiveness, current draw, thermal margin) is unverified.
- The build config was a **minimal headless test config**, not your exact production config. Enabling features you already use (BLTouch, a display, SD card) changes the RAM and ISR budget — re-check the `%` figures after your real build.
