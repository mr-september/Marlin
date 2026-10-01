# Ender 3 Pro — Firmware Notes

Personal build notes for this machine. Upstream Marlin docs live in `docs/`; this file
records **only what is specific to this printer**, so a future config edit can be traced
back to a reason.

## Hardware

| Item | Value |
|---|---|
| Board | BTT SKR V1.3 (LPC1768, 32-bit) |
| Drivers | TMC2208 on X / Y / Z / E (UART, standalone) |
| Kinematics | Cartesian, bedslinger |
| Bed | 235 × 235 mm |
| Nozzle | 0.4 mm |
| Probe | **None fitted.** Manual grid levelling via the front LCD. |
| Display | Stock Creality TFT (smart/enhanced) |
| Extruder | **Direct drive** (upgraded from stock geared) |
| Z axis | **Dual-Z, two motors wired in parallel** — the firmware sees a single Z stepper. No dual-Z support configured. |

The dual-Z modification is invisible to firmware: two motors share one step/dir pin.
That means no `Z2_*` options are used and Z calibration is a mechanical procedure, not a
firmware one.

**There is no probe on this machine.** An earlier version of this file claimed a
"Factory BLTouch (servo-pin)". That was an unverified assumption and it was wrong — the
config has `BLTOUCH` commented out, no `Z_MIN_PROBE_PIN`, and all probe/levelling options
(`MANUAL_PROBE_START_Z`, `LCD_PROBE_Z_RANGE`) commented out. Bed levelling is done
manually from the LCD (`LEVEL_BED_CORNERS`, `MESH_TEST_*` grid options). Do not enable
BLTouch in any port: with a probe configured but none fitted, `G28` drives Z into the bed.

## Tuning history

### E steps: 93 → 105 → 130 (committed 2024-06-02)

```c
#define DEFAULT_AXIS_STEPS_PER_UNIT   { 80, 80, 400, 130}
```

Three iterations. 93 was inherited from the stock Creality config for a *geared*
extruder. 105 was an intermediate step; 130 is the final value for the direct-drive
extruder. Higher E-steps = finer extrusion resolution, which matters more on a direct
drive because retraction is shorter and less tolerant of volumetric error.

**Recalibrate** (`M92 E`, extrude 100 mm marked, measure, set) if the extruder is ever
swapped or the filament diameter changes.

### E direction: `INVERT_E0_DIR false`

```c
#define INVERT_E0_DIR false
```

A `Configuration-DESKTOP-UBSMMJV.h` cloud-conflict artifact survives on the OneDrive
copy with `INVERT_E0_DIR true` **and** `E0_DRIVER_TYPE TMC2209`. That combination was
tried and reverted; 2209 does not fit the SKR V1.3 driver bays. Current committed values
are TMC2208 + `false`.

### `POWER_LOSS_RECOVERY` → `SDCARD_CONNECTION LCD` (committed 2024-06-02)

```c
-  #define POWER_LOSS_RECOVERY
+  #define SDCARD_CONNECTION LCD
```

Power loss recovery was disabled in favour of SD card support on the LCD. Reason not yet
recorded — verify before reverting either way. If this was traded for reliability on the
Creality TFT, note it here.

## Measured build sizes (2026-10-01, PlatformIO 6.2.0)

Real numbers from successful links, not estimates.

| Build | RAM | Flash |
|---|---|---|
| **Your committed 2.0.x config** (SKR V1.3 + TMC2208, stock Creality TFT) | 14,788 B / 32,736 — **45.2%** | 183,092 B / 475,136 — **38.5%** |
| Stock 2.1.x, SKR V1.3 + TMC2208 | 7,172 B / 32,736 — **21.9%** | 98,108 B / 475,136 — **20.6%** |
| Stock 2.1.x **+ input shaping X&Y** | 8,764 B / 32,736 — **26.8%** | 100,964 B / 475,136 — **21.2%** |

Input shaping costs ~1.6 KB RAM and ~2.9 KB flash. **It fits comfortably**, verified twice
(clean rebuild reproduced 8,764 / 100,964 exactly, and `M593` is present in the linked
binary). Verified per the Source Verification Protocol; these are linked, not estimated.

Note the 2.1.x numbers are from a *minimally configured* build (board + drivers only, no
Creality TFT, no BLTouch, no mesh levelling). Your real 2.1.x port will be larger — likely
in the same region as the 2.0.x figure above, since that one carries the full feature set.

**Conclusion: there is no flash/RAM obstacle to a 2.1.x upgrade with input shaping on this
board.** The "blank screen / boot loop" cause must be something else.

## Known-unknowns / not yet reviewed

These were inherited from the stock config and have **not** been validated on this
machine. Flagged for the optimisation pass.

- `DEFAULT_MAX_FEEDRATE { 500, 500, 5, 25 }` — stock bedslinger values, conservative.
- `DEFAULT_ACCELERATION 500` — very low; likely leaves real speed on the table.
- Junction deviation / jerk settings in `Configuration_adv.h` — untouched stock values.
- Input shaping does not exist in Marlin 2.0.x; it arrived in 2.1.x.

  **Correction (2026-10-01):** an earlier note in this file claimed input shaping was
  STM32-only and therefore unavailable on this LPC1768. **That was wrong** — it was
  asserted without checking the source. Verified against Marlin 2.1.x: the only
  `INPUT_SHAPING` gates in `SanityCheck.h` are *kinematic* (incompatible with COREXZ /
  COREYZ; requires both axes on CoreXY). `src/HAL/LPC1768/inc/SanityCheck.h` contains
  zero references, and the implementation lives in `src/gcode/feature/input_shaping/`,
  not under `HAL/STM32/`. So input shaping **is** available on this board, contingent
  only on fitting in flash/RAM.

  This materially changes the 2.1.x upgrade calculus: input shaping is one of the main
  reasons to move, and it is NOT off the table for this machine. The real constraint is
  measured size, which is not yet known.
- TMC2208 `TCOOLTHRS`/`TBLANK`/`TPWMTHRS` and driver current sense (`CSRS`) unverified.

## Build/flash notes

- Branch: `2.0.x`. Do not jump to `bugfix-2.1.x` without testing — see the version-stability
  concern below.
- PlatformIO env: `LPC1768` (confirmed by the `.pio/build/LPC1768` cache on the OneDrive copy).
- The OneDrive tree is extremely slow for git operations (cloud-placeholder hydration;
  observed 420 s+ timeouts). Work from a local clone.
- Remotes: `origin` = this repo (mr-september), `upstream` = Firedrops fork,
  `onedrive` = the OneDrive working copy.

## Version-stability concern (unresolved)

There is a recollection that newer Marlin versions produced a **blank screen or boot loop**
on one of these printers, forcing a pin to an older release. The printer responsible was
never pinned down — **most likely the Feiyu Mini**, but unverified.

Blank screen / boot loop on Marlin 2.1.x is a known class of failure on 8-bit boards
(LPC1768) and on boards whose bootloader or display assumptions changed. Until this is
diagnosed, treat "upgrade Marlin" as a research task, not a maintenance step:

1. Identify which printer and which firmware pair actually failed.
2. Reproduce the failure in a build, not on hardware.
3. Understand the mechanism before shipping any version bump.

Do not flash an untested firmware version to a working printer.

**Update (2026-10-01):** the earlier related claim — that input shaping is STM32-only and
therefore unavailable on the Ender's LPC1768 — has been **disproven against the source**.
Its `SanityCheck` gates are kinematic, not MCU-based, and the implementation is not under
`HAL/STM32/`. Input shaping is available on both LPC1768 boards, so it cannot be the
reason either machine was pinned to an older Marlin. Whatever caused the blank screen or
boot loop remains unidentified.

Note also that the first attempt at this investigation produced three builds that failed
with `#endif without #if`. That was a self-inflicted error from hand-editing
`Configuration.h` with naive string replacement inside an `#if`-guarded block, **not** a
Marlin defect. Discard any such result. When building test configurations, start from the
stock files under `config/examples/` rather than patching `Configuration.h` by hand.