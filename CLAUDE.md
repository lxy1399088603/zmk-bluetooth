# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

ZMK firmware configuration for a split Sofle keyboard using SuperMini NRF52840 controllers (nice!nano-compatible). Migrated from QMK to ZMK to add Bluetooth split support. The firmware is built entirely by GitHub Actions — there is no local build toolchain in this repo.

## Building

Firmware is built via GitHub Actions on every push/PR using `.github/workflows/build.yml` and `build.yaml`. Push changes to trigger a build; download artifacts from the Actions run.

Build targets defined in `build.yaml`:
- `sofle_supermini_left` — left half firmware
- `sofle_supermini_right` — right half firmware
- `settings_reset` — clears persistent Bluetooth/pairing state
- `tester_pro_micro` — hardware pin-connectivity diagnostic

The board name `nice_nano//zmk` is the post-Zephyr-4.1 format. For older ZMK, change to `nice_nano_v2`.

## Flashing

1. Download the firmware zip from GitHub Actions artifacts.
2. Flash `sofle_supermini_left` to the left controller and `sofle_supermini_right` to the right.
3. Each SuperMini enters bootloader mode via double-tap on reset button; it mounts as a USB drive and accepts `.uf2` files.

**Settings reset** (use if Bluetooth name is stale, pairing broken, or keyboard connects but sends no keys):
1. Flash `settings_reset`.
2. After reboot, flash the normal half firmware.
3. Remove old host-side pairing and re-pair.

**Pin tester** (use if Bluetooth works but no physical keys register):
1. Flash `tester_pro_micro` to left controller.
2. Press keys and observe which Pro Micro pins trigger.
3. Compare against expected rows `D5–D9` and columns `D10, D14, D15, D16, D18, D19`.

## Architecture

### File Layout

```
config/
  sofle_supermini.keymap      # All layers, macros, behaviors
  sofle_supermini.conf        # Feature toggles (name, sleep, OLED, encoders)
  west.yml                    # ZMK dependency manifest (tracks zmk main)
  boards/shields/sofle_supermini/
    sofle_supermini.dtsi           # Shared hardware: matrix rows, encoders, OLED
    sofle_supermini_left.overlay   # Left-half column GPIOs, enables left encoder
    sofle_supermini_right.overlay  # Right-half column GPIOs, enables right encoder
    Kconfig.defconfig              # Split role: left=central, right=peripheral
    Kconfig.shield                 # Shield symbol definitions
    sofle_supermini.conf           # Shield-level build options (if any)
build.yaml                    # Build targets (board + shield combinations)
.github/workflows/build.yml   # Delegates to zmkfirmware/zmk build workflow
```

### Hardware

- **Matrix:** 5 rows × 6 columns per half, `col2row` diode direction.
- **Rows** (shared, both halves): GPIO P0.24, P1.00, P0.11, P1.04, P1.06.
- **Columns:** Different GPIOs per half, defined in the per-half `.overlay` files.
- **Encoders:** EC11, 80 steps. Left: A=P0.29, B=P0.31. Right: A=P0.31, B=P0.29 (reversed). Disabled in `.dtsi`, enabled per half in overlays.
- **OLED:** SSD1306 128×32 at I2C address `0x3c` on Pro Micro I2C pins (D0/D1).

### Matrix Transform

The `default_transform` in `sofle_supermini.dtsi` maps physical matrix `RC(row, col)` positions to logical keymap positions. Columns 0–5 are the left half, 6–11 are the right half. The transform was derived from the VIA layout coordinates with the fake encoder column (`C7`) removed. If physical keys appear in the wrong positions, fix the `RC()` entries in the transform — not the keymap bindings.

### Keymap Layers

Defined in `config/sofle_supermini.keymap`:

| Layer | Index | Purpose |
|-------|-------|---------|
| BASE  | 0     | Default typing layer |
| GAME  | 1     | Gaming layer (WASD-oriented) |
| LOWER | 2     | F-keys, Bluetooth profiles, arrows, symbols |
| RAISE | 3     | Navigation, window management, BT/output controls |

**Mode key** (left thumb): `mode_hold` hold-tap behavior — hold activates GAME layer momentarily, tap toggles LOWER layer.

**Custom macros:** `alt_tab_fwd`, `alt_tab_rev`, `move_win_right`, `move_win_left`, `word_backspace` (Ctrl+Backspace).

**Sensor bindings** (same on all layers): left encoder = Page Up/Down, right encoder = Up/Down.

### Configuration Options (`sofle_supermini.conf`)

| Option | Value | Effect |
|--------|-------|--------|
| `CONFIG_ZMK_KEYBOARD_NAME` | `"lxy-kb"` | Bluetooth device name |
| `CONFIG_ZMK_SLEEP` | `y` | Enable idle sleep |
| `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT` | `900000` | 15-minute sleep timeout |
| `CONFIG_EC11` | `y` | Rotary encoder driver |
| `CONFIG_ZMK_DISPLAY` | `y` | OLED display |

## Known Limitations (Not Yet Ported from QMK)

- Alt-Tab **hold** behavior — current macros do a single tap, not a sustained hold.
- Shift+Backspace word-delete — replaced with Ctrl+Backspace (`word_backspace` macro).
- RGB underglow.
- OLED pet animation — current OLED shows ZMK built-in status screen only.
- VIA/ZMK Studio dynamic remapping.
