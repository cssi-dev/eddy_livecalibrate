# eddy_livecalibrate

Live Z-center calibration for Klipper with an eddy-current probe (e.g. BTT
Eddy). A single macro, `CALIBRATE_Z_CENTER`, that probes two points just before
a print and sets a live gcode Z offset from their difference — so your first
layer tracks gantry/frame thermal drift instead of relying on one cold
calibration.

## How it works

It probes two spots and applies:

```
z_offset = z_a - z_b + z_adjust
```

| Term       | Meaning                                                        |
|------------|----------------------------------------------------------------|
| `z_a`      | Probe trigger height at point **A** (reference, near gantry)   |
| `z_b`      | Probe trigger height at point **B** (bed center)               |
| `z_adjust` | Fixed fine-tune term for your bed/nozzle (you calibrate this)  |

The A−B difference captures how far the center has moved relative to the
reference between prints (thermal expansion, bed sag), and `z_adjust` dials in
the absolute first-layer height. The result is applied with
`SET_GCODE_OFFSET Z=... MOVE=0`.

The macro is split into three parts because Klipper needs a command to
*complete* before `printer.probe.last_z_result` is readable — so each `PROBE`
hands off to the next macro:

1. `CALIBRATE_Z_CENTER` — probes **A**, then calls…
2. `_CALIBRATE_Z_SAVE_A` — stores `z_a`, probes **B**, then calls…
3. `_CALIBRATE_Z_APPLY` — computes and applies the offset.

## Requirements

- Klipper with a probe exposing `printer.probe.last_z_result` after a bare
  `PROBE` (eddy-current probes like BTT Eddy do).
- The printer must be **homed** and any gantry leveling (**QGL** / **Z_TILT**)
  done *before* calling this, or the two probe points aren't comparable.

## Install

Copy `eddy_livecalibrate.cfg` into your config directory and include it:

```ini
# printer.cfg
[include eddy_livecalibrate.cfg]
```

Restart Klipper (`FIRMWARE_RESTART`).

## Usage

Call once in your print-start sequence, **after** homing + leveling + mesh:

```gcode
CALIBRATE_Z_CENTER
```

### Parameters

All optional; defaults are for a ~350 mm Voron-style bed — **change them for
your machine**:

| Param               | Default | Meaning                               |
|---------------------|---------|---------------------------------------|
| `POS_A_X` `POS_A_Y` | 175, 295 | Reference probe point                |
| `POS_B_X` `POS_B_Y` | 175, 175 | Bed-center probe point               |
| `PROBE_Z`           | 3       | Travel / probe-start Z                |
| `Z_ADJUST`          | -0.815  | Fine-tune offset (**calibrate this**) |

Example:

```gcode
CALIBRATE_Z_CENTER POS_A_X=175 POS_A_Y=295 POS_B_X=175 POS_B_Y=175 Z_ADJUST=-0.80
```

## Tuning `Z_ADJUST`

- **Positive** → nozzle farther from bed (higher first layer).
- **Negative** → nozzle closer to bed (lower first layer).

Start from your known-good single-point Z offset and adjust in ~0.02 mm steps
using a first-layer test print.

> The default `Z_ADJUST = -0.815` is specific to the original machine. It is
> **not** universal — calibrate your own before relying on it.

## License

GPLv3 — see [LICENSE](LICENSE). (Matches Klipper's own license.)
