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


## Settings Reset

If the keyboard appears with an old Bluetooth name, cannot be discovered after deleting the host-side pairing, or connects but sends no keys, clear ZMK persistent settings:

1. Download the latest GitHub Actions firmware zip.
2. Flash the `settings_reset` firmware to the controller.
3. Wait for the controller to reboot. It will not advertise over Bluetooth while running reset firmware.
4. Flash the normal `sofle_supermini_left` or `sofle_supermini_right` firmware again.
5. Remove/forget the old keyboard entry on the host and pair again.

For the current one-half test, flash `settings_reset` to the left controller, then flash `sofle_supermini_left` again. After reset, the Bluetooth name should become `Sofle SuperMini`.

## Pro Micro Pin Tester

If Bluetooth works but no key on the PCB produces input, flash the `tester_pro_micro` firmware to the controller while it is installed on the PCB. This ZMK test firmware helps verify whether the Pro Micro footprint pins are electrically connected as expected.

Use it before changing the matrix map:

1. Flash `tester_pro_micro` to the left controller.
2. Pair/connect it if needed.
3. Press keys across the left half and record which Pro Micro pins trigger.
4. Compare the result with the expected row pins `D5-D9` and column pins `D10, D16, D14, D15, D18, D19`.
5. Flash `sofle_supermini_left` again after testing.
## Left-Half Mode Layer

The left thumb mode key uses a tap-dance behavior:

- Single tap: no action.
- Double tap: toggles the Lower mode layer.
- On the Lower layer, pressing the same physical mode key toggles Lower off.

Current left-half Base layout:

```text
ESC   1     2     3     4     5
TAB   Q     W     E     R     T
LSFT  A     S     D     F     G
LCTL  Z     X     C     V     B
      `     LGUI  LALT  MODE  SPACE
```

Current left-half Lower mode controls:

```text
F1      F2      F3    F4       F5      F6
BT0     BT1     UP    BT2      BT3     BT4
BLE     LEFT    DOWN  RIGHT    USB     OUT_TOG
BT_CLR  BT_PRV  BT_NXT
              MODE_OFF
```

Bluetooth profile controls use ZMK profiles 0-4. `BLE`, `USB`, and `OUT_TOG` switch the output target.
