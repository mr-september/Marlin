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
| Probe | Factory BLTouch (servo-pin) |
| Display | Stock Creality TFT (smart/enhanced) |
| Extruder | **Direct drive** (upgraded from stock geared) |
| Z axis | **Dual-Z, two motors wired in parallel** — the firmware sees a single Z stepper. No dual-Z support configured. |

The dual-Z modification is invisible to firmware: two motors share one step/dir pin.
That means no `Z2_*` options are used and Z calibration is a mechanical procedure, not a
firmware one.

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

## Known-unknowns / not yet reviewed

These were inherited from the stock config and have **not** been validated on this
machine. Flagged for the optimisation pass.

- `DEFAULT_MAX_FEEDRATE { 500, 500, 5, 25 }` — stock bedslinger values, conservative.
- `DEFAULT_ACCELERATION 500` — very low; likely leaves real speed on the table.
- Junction deviation / jerk settings in `Configuration_adv.h` — untouched stock values.
- Input shaping is not a Marlin 2.0 feature; it arrived in 2.1.x for STM32 only, so it
  is **not** available on this LPC1768 board.
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