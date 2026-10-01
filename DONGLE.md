# moNa2 v2 + nRF52840 MDK USB Dongle

This branch turns the Makerdiary **nRF52840 MDK USB Dongle** into the split
**central** of the moNa2. Both halves become **peripherals** and connect to the
dongle over BLE. The dongle connects to the computer over USB.

```
 [left half: XIAO]  --BLE-->  +-------------------+  --USB-->  PC
   keys + encoder             | MDK USB Dongle    |
                              | (central, keymap, |
 [right half: XIAO] --BLE-->  |  ZMK Studio, LED) |
   keys + PAW3222 trackball   +-------------------+
```

## What changed

| File | Change |
|---|---|
| `boards/shields/mona2/mona2-common.dtsi` | **New.** Physical layout, matrix transform, trackball `zmk,input-split` and `trackball_listener`. Shared by both halves and the dongle, with no XIAO pin references. |
| `boards/shields/mona2/mona2.dtsi` | Keeps only the XIAO hardware (kscan, encoders, sensors) and includes the common file. |
| `boards/shields/mona2/mona2_r.overlay` | The right half is now a peripheral. The PAW3222 is proxied through `&trackball_split` instead of a local listener. |
| `boards/shields/mona2/mona2_dongle.overlay` | **New.** Dongle shield: mock kscan, placeholder sensors (so encoder bindings work), and the enabled trackball listener with the layer‑1 scroll mode. |
| `boards/shields/mona2/Kconfig.*` | New `SHIELD_MONA2_DONGLE`: central, 2 peripherals, `BT_MAX_CONN=7`. The right half no longer sets itself as central. |
| `boards/shields/mdk_dongle/` | **New**, keyboard‑agnostic. ZMK has no board defaults for this dongle, so this shield adds USB HID, NVS settings storage and `.uf2` output. It also sets a flash layout that matches the factory **UF2 bootloader**: app at `0x1000`, 32 KB settings at `0xEC000`, and the bootloader at `0xF4000`. |
| `config/mona2_dongle.conf` | **New.** Battery fetch/proxy for both halves, dongle RGB LED (layer colours, peripheral battery levels) and ZMK Studio. |
| `config/mona2_r.conf`, `config/mona2_l.conf` | Central-only options removed: Studio, layer colours and battery proxy. |
| `build.yaml` | Builds the dongle, both halves and a `settings_reset` for each board. |

Your keymap (`config/mona2.keymap`) doesn't change. The dongle now runs it.

## Flashing (one time)

You'll find all `.uf2` files in the GitHub Actions artifact. Do the steps in this order:

1. **Reset the halves.** Do this on each XIAO half: double‑tap reset, then drop
   `settings_reset_xiao.uf2` on it. This deletes the old
   left↔right pairing.
2. **Flash the halves.** Put `mona2_left.uf2` on the left and `mona2_right.uf2` on the right.
3. **Flash the dongle.** Hold its button and plug it into USB. Let go once the
   RGB LED turns green and a **UF2BOOT** drive appears. Drop
   `mona2_dongle.uf2` on the drive.
   - The dongle is factory‑new, so you don't need its `settings_reset` now.
     Use `settings_reset_dongle.uf2` later if you ever need to re-pair.
4. **Pairing.** Leave the dongle plugged in and power on both halves. They pair to
   the dongle on their own. Then remove the old **"mona2"** Bluetooth entry from
   your computer, because the dongle now handles that connection over USB.

To flash the dongle again later, unplug it, hold the button and plug it back in.

## Dongle LED (zmk-rgbled-widget)

- On boot it blinks the battery level of each half (green/yellow/red, or magenta if a half is disconnected).
- It shows cyan when output goes over USB.
- After that the colour shows the active layer (`CONFIG_RGBLED_WIDGET_LAYER_n_COLOR`).
  Colours: 0 off, 1 red, 2 green, 3 yellow, 4 blue, 5 magenta, 6 cyan, 7 white.

The LEDs on the halves now only show their own battery level and whether they're connected to the dongle.

## ZMK Studio

Plug in the dongle and open ZMK Studio, then connect over USB serial.

## Notes and caveats

- **You need the dongle to use the keyboard.** The halves only talk to the dongle.
  To go back to the old setup, build the `main` branch, flash `settings_reset`
  to both halves, then flash the old firmware.
- `&bt BT_SEL n` and `BT_CLR` still work. They now control the dongle's own
  BLE host profiles, but you'll normally use USB.
- Latency: the right half gets about 1 ms more delay. The left half gets about 6.5 ms less on average,
  because one BLE hop is replaced by USB.
- Battery: the right half isn't central anymore, so it should last noticeably longer.
- If you upgrade ZMK past v0.3 (Zephyr 4.x), the board is called
  `nrf52840_mdk_usb_dongle/nrf52840`. Update `build.yaml` to match.
