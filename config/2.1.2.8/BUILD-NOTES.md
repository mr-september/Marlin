# Marlin 2.1.2.8 port — Ender 3 Pro (SKR V1.3)

Built and verified 2026-10-01 with PlatformIO 6.2.0, `env:LPC1768`.

## Result

```
RAM:   [=====     ]  53.2% (used 17404 bytes from 32736 bytes)
Flash: [====      ]  35.5% (used 168568 bytes from 475136 bytes)
SUCCESS
```

`firmware.bin` = 168,568 bytes. Feature set: SKR V1.3 + TMC2208 UART (X/Y/Z/E),
stock Creality TFT (`CR10_STOCKDISPLAY`), `INPUT_SHAPING_X/Y`,
`ADAPTIVE_STEP_SMOOTHING`, SD card. **No probe — this machine has none.**

For comparison, the committed 2.0.x config links at RAM 14,788 B (45.2%) /
Flash 183,092 B (38.5%). Comfortable headroom on both.

## CORRECTION — no probe on this machine

An earlier revision of this port enabled `BLTOUCH`. **That was wrong**: it was
my assumption, not a fact from the config. The working 2.0.x config has
`BLTOUCH` commented out at line 1093, no `Z_MIN_PROBE_PIN`, and its
probe/levelling options (`MANUAL_PROBE_START_Z`, `LCD_PROBE_Z_RANGE`) all
commented out. This machine does manual grid levelling with the front LCD.

This port therefore builds **probe-less**. BLTouch is explicitly disabled rather
than merely absent, so it cannot be switched on by accident.

This also removes a crash risk: with a probe configured but none fitted, `G28`
would drive Z into the bed.

## Provenance

Ported from the **official** `MarlinFirmware/Configurations` branch
`release-2.1.2.8`, `config/examples/Anet/E16/BTT SKR 1.3` — the only official
2.1.2.8 config targeting `BOARD_BTT_SKR_V1_3`, used unmodified except for the
changes below. So this port is auditable against upstream.

Upstream Marlin ships `config/` containing only a README; example configs live
in the separate `Configurations` repo on branches named `release-<tag>`.

## Changes applied to the base

Carried over from the working 2.0.x config:

| Setting | Value | Note |
|---|---|---|
| `DEFAULT_AXIS_STEPS_PER_UNIT` | `{ 80, 80, 400, 130}` | E=130, direct-drive, calibrated 2024-06 |
| `INVERT_E0_DIR` | `false` | direct drive |
| `X_BED_SIZE` / `Y_BED_SIZE` | `235` | Ender 3 Pro |
| `Z_MAX_POS` | `250` | matches working config (base had 300) |
| `X/Y/Z_MIN_POS` | `0` | matches working config |
| `Z_MIN_ENDSTOP_INVERTING` | `false` | matches working config |
| `DEFAULT_MAX_FEEDRATE` | `{ 300, 300, 5, 25 }` | bedslinger-safe ceiling |
| `DEFAULT_ACCELERATION` | `1000` | raised from stock 500 |

Features:

| Change | Reason |
|---|---|
| `REPRAP_DISCOUNT_FULL_GRAPHIC_SMART_CONTROLLER` → disabled | Anet base display; wrong machine |
| `CR10_STOCKDISPLAY` → enabled | the Creality TFT this printer actually has |
| `BLTOUCH` → explicitly disabled | **no probe fitted** |
| `INPUT_SHAPING_X/Y` → enabled | available on LPC1768 — gates are kinematic, not MCU |
| `ADAPTIVE_STEP_SMOOTHING` → enabled | already enabled in the 2.0.x config |

## Build errors hit along the way (both are Marlin working correctly)

1. `Please select only one LCD controller option` — the Anet base already had
   `REPRAP_DISCOUNT_FULL_GRAPHIC_SMART_CONTROLLER` active; enabling
   `CR10_STOCKDISPLAY` without disabling it is a real conflict.
2. An earlier revision also hit `BLTOUCH requires Z_MIN_ENDSTOP_INVERTING false`
   while carrying the incorrect BLTouch assumption.

## NOT YET DONE — flash safety

Build-verified only. Nothing has been flashed to hardware.

Before flashing:

- [ ] Flash with the **bootloader intact**. A blank screen with no serial output
      may mean the SKR V1.3 is in DFU mode (flash at 16 KB reads `0xFFFFFFFF`) —
      a different problem from any firmware bug.
- [ ] Set input shaping frequencies with `M593` before using high acceleration.
      Without a measured resonance frequency the defaults (40 Hz, zeta 0.15) can
      make ringing *worse*, not better.
- [ ] Verify the Creality TFT renders correctly — it is a different LCD path than
      the Anet base was built for. This is the most likely thing to differ.
- [ ] Re-do the manual bed-level procedure after flashing; nothing about the
      physical machine changes, but Z homing behaviour should be spot-checked
      (`G28`, confirm the nozzle clears the bed at the corners).

Keep the working 2.0.x firmware available to revert to.
