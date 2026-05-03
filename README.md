# Sofle SuperMini NRF52840 ZMK Config

This is a first-pass ZMK migration for the existing QMK Sofle firmware in `D:\cwork\keyboard\sofle`.

The target hardware is a Pro Micro-footprint SuperMini NRF52840, compatible with the nice!nano style pinout. The goal is to preserve the existing PCB, physical key positions, and familiar layers while adding Bluetooth split support, Bluetooth profile switching, and USB/BLE output selection.

## Current Scope

Included now:

- Custom `sofle_supermini` split shield.
- Left and right build targets.
- 5x6 physical matrix per half.
- The old QMK/VIA fake `C7` encoder column is removed.
- Four keymap layers converted from `sofle VIA layout.json`.
- Bluetooth profile selection/next/previous/clear on the raise layer.
- USB/BLE output selection on the raise layer.
- Encoders modeled as ZMK sensors.
- RGB and OLED disabled for the first validation build.

Not yet restored exactly:

- QMK's delayed Alt-Tab hold behavior. The first version uses tap macros.
- QMK's left/right Shift + Backspace direction-specific word deletion. The first version maps the custom key to Ctrl+Backspace.
- RGB underglow and OLED pet display.
- VIA/ZMK Studio dynamic remapping.

## Build

GitHub Actions can build the firmware from this repository using `.github/workflows/build.yml` and `build.yaml`.

Current `build.yaml` uses the post-Zephyr-4.1 ZMK board name:

```yaml
board: nice_nano//zmk
```

If building against an older ZMK revision, change it to:

```yaml
board: nice_nano_v2
```

## Flashing

Build outputs should produce one firmware file per half:

- `sofle_supermini_left`
- `sofle_supermini_right`

Flash the left firmware to the left SuperMini and the right firmware to the right SuperMini.

## First Validation Order

1. Flash both halves.
2. Pair the keyboard over Bluetooth.
3. Test every physical key with a key tester.
4. If a column is reversed or offset, fix the corresponding `col-gpios` or matrix transform entry.
5. Test the raise layer Bluetooth controls.
6. Test encoder direction.
7. Only after that, consider enabling RGB or OLED.

## Source Notes

The matrix pin order is taken from the current QMK `config.h`:

- Left rows: `C6, D7, E6, B4, B5` -> Pro Micro `D5, D6, D7, D8, D9`
- Left columns: `B6, B2, B3, B1, F7, F6` -> Pro Micro `D10, D16, D14, D15, D18, D19`
- Right columns: `F6, F7, B1, B3, B2, B6` -> Pro Micro `D19, D18, D15, D14, D16, D10`
- Encoders: `F5/F4` -> Pro Micro `D20/D21`

The SuperMini NRF52840 pinout maps those Pro Micro labels to nRF52840 GPIOs internally through the nice!nano-compatible board definition.

