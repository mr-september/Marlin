# Marlin 2.1.2.8 port — Ender 3 Pro (SKR V1.3)

Built and verified 2026-10-01 with PlatformIO 6.2.0, `env:LPC1768`.

## Result

```
RAM:   [=====     ]  53.4% (used 17468 bytes from 32736 bytes)
Flash: [====      ]  37.0% (used 175584 bytes from 475136 bytes)
SUCCESS
```

`firmware.bin` = 175,584 bytes. Feature set: SKR V1.3 + TMC2208 UART (X/Y/Z/E),
BLTouch with Z_SAFE_HOMING, stock Creality TFT (`CR10_STOCKDISPLAY`),
`INPUT_SHAPING_X/Y`, `ADAPTIVE_STEP_SMOOTHING`, SD card.

For comparison, the committed 2.0.x config links at RAM 14,788 B (45.2%) /
Flash 183,092 B (38.5%). RAM is up ~2.7 KB for the added features; flash is
slightly *lower*. Comfortable headroom on both.

## Provenance

Ported from the **official** `MarlinFirmware/Configurations` branch
`release-2.1.2.8`, `config/examples/Anet/E16/BTT SKR 1.3` — the only official
2.1.2.8 config targeting `BOARD_BTT_SKR_V1_3`. That base was used unmodified
except for the changes listed below, so this port is auditable against upstream
rather than hand-built from a template.

Note: upstream Marlin ships `config/` containing only a README — the example
configs live in the separate `Configurations` repo, on per-release branches
named `release-<tag>` (not the tag itself).

## Changes applied to the base

Machine values, carried over from the working 2.0.x config:

| Setting | Value | Note |
|---|---|---|
| `DEFAULT_AXIS_STEPS_PER_UNIT` | `{ 80, 80, 400, 130}` | E=130, direct-drive, calibrated 2024-06 |
| `INVERT_E0_DIR` | `false` | direct drive |
| `X_BED_SIZE` / `Y_BED_SIZE` | `235` | Ender 3 Pro |
| `DEFAULT_MAX_FEEDRATE` | `{ 300, 300, 5, 25 }` | bedslinger-safe ceilings |
| `DEFAULT_ACCELERATION` | `1000` | raised from stock 500 |
| `Z_MIN_ENDSTOP_INVERTING` | `false` | **required** by BLTouch |

Features:

| Change | Reason |
|---|---|
| `REPRAP_DISCOUNT_FULL_GRAPHIC_SMART_CONTROLLER` → disabled | Anet base display; wrong machine |
| `CR10_STOCKDISPLAY` → enabled | the Creality TFT this printer actually has |
| `BLTOUCH` + `Z_SAFE_HOMING` → enabled | factory probe; Marlin requires Z_SAFE_HOMING with probe homing |
| `INPUT_SHAPING_X/Y` → enabled | available on LPC1768 — gates are kinematic, not MCU |
| `ADAPTIVE_STEP_SMOOTHING` → enabled | already enabled in the 2.0.x config |

Total delta vs the official base: 11 insertions, 15 deletions in `Configuration.h`.

## Build errors hit along the way (both are Marlin working correctly)

1. `BLTOUCH requires Z_MIN_ENDSTOP_INVERTING set to false` — fixed by porting the
   value from the working 2.0.x config rather than guessing.
2. `Please select only one LCD controller option` — the Anet base already had
   `REPRAP_DISCOUNT_FULL_GRAPHIC_SMART_CONTROLLER` active. Enabling
   `CR10_STOCKDISPLAY` without disabling it is a genuine conflict.

Both are compile-time sanity checks doing their job. Neither is a Marlin defect.

## NOT YET DONE — flash safety

This is a **build-verified** config only. Nothing has been flashed to hardware.

Before flashing:

- [ ] Flash over USB/SD with **the bootloader intact** — a blank screen with no
      serial output may mean the SKR V1.3 is in DFU mode (flash at 16 KB reads
      `0xFFFFFFFF`), which is a different problem from any firmware bug.
- [ ] Re-tune probe offset (`G28`, then `M851`) — BLTouch on a 235 mm bed will
      not carry over the old offset.
- [ ] Set input shaping frequencies with `M593` before enabling high acceleration.
      Without a measured resonance frequency the defaults (40 Hz, zeta 0.15)
      will over-damp or under-damp and can make ringing *worse*.
- [ ] Verify the Creality TFT renders correctly; it is a different LCD path than
      the Anet base was built for.

Keep the working 2.0.x firmware available to revert to.
