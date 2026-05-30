# ergo_right — ZMK shield + config

ZMK firmware for the **right half** of a custom split ergo keyboard:

- **Controller:** Seeed XIAO nRF52840 Plus
- **Column expander:** MCP23008 (I2C, address 0x20)
- **Trackpad:** Azoteq TPS43 — *currently disabled in the build*
- **ZMK target:** main branch, Zephyr 4.1

This repo is structured as both a ZMK **module** and a **user config**
(per the unified-zmk-config-template layout), which is the only layout
that doesn't trip the `config/boards` deprecation warning on Zephyr 4.1.

## Layout

```
.github/workflows/build.yml         GitHub Actions runner
build.yaml                          Build matrix (xiao_ble//zmk + ergo_right)
zephyr/module.yml                   Registers this repo as a ZMK module
boards/shields/ergo_right/          Shield definition (the module half)
    Kconfig.shield
    Kconfig.defconfig
    ergo_right.overlay              Pinctrl, I2C, MCP23008, kscan, transform, physical layout
    ergo_right.keymap               Default 22-key keymap
    ergo_right.zmk.yml              Metadata
config/                             User config (overrides the defaults above)
    west.yml                        West manifest pinning ZMK main
    ergo_right.conf                 User-facing Kconfig settings
```

## What was wrong with the prior build and what's fixed

| Symptom in the log                                                                    | Fix in this repo                                                                                                                                                |
| ------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fatal error: nordic/nrf_pinctrl.h: No such file or directory`                        | Include path updated to `<zephyr/dt-bindings/pinctrl/nrf-pinctrl.h>` (the Zephyr 4.1 location)                                                                   |
| `The config/boards folder is deprecated`                                              | Whole repo restructured as a module: shield moved to `boards/shields/ergo_right/` at the repo root, plus a `zephyr/module.yml`                                  |
| Earlier overlay used `microchip,mcp230xx` with `ngpios`                               | Updated to `microchip,mcp23008`, `ngpios` removed (the binding was split per-chip in Zephyr 4.0)                                                                |
| Earlier `.conf` used `CONFIG_I2C_MCP230XX` / `CONFIG_GPIO_MCP230XX`                   | Now `CONFIG_GPIO_MCP23XXX`, which is the symbol that actually exists                                                                                            |
| Earlier overlay set `compatible = "nordic,nrf-twi"` on `&i2c0`                        | Removed — the XIAO board already sets `nordic,nrf-twim`; overriding it was unnecessary and incorrect                                                            |
| Earlier overlay had `dr-gpios` for the trackpad                                       | The AYM1607 driver uses `rdy-gpios`. Trackpad node now uses the right names but is **commented out** since it also needs `reset-gpios`, which is not yet wired  |
| Earlier `chosen` used `zmk,matrix-transform`                                          | Replaced with `zmk,physical-layout` plus a `zmk,physical-layout`-compatible node, which is what ZMK on Zephyr 4.1 expects                                       |
| Board ID was `seeeduino_xiao_ble`                                                     | Now `xiao_ble//zmk` (HWMv2)                                                                                                                                     |

## How to use

1. Create a new GitHub repository (empty, no README).
2. Drop everything in this archive into it, keeping the directory layout.
3. Push to `main`. GitHub Actions runs the build workflow automatically.
4. After the green check, open the workflow run → Artifacts → download
   the `firmware` zip. Inside is the `.uf2` file. Double-tap reset on
   the XIAO and drag the UF2 onto the USB drive that appears.

## Re-enabling the trackpad later

Right now the trackpad is intentionally absent from the build because
your wiring is missing a reset line and the AYM1607 driver requires
one. To turn it back on once that's sorted:

1. Wire a free XIAO pin (D0, D1, D7, or D10) to the trackpad's **RST**
   pad. Note the corresponding `gpio0`/`gpio1` port and pin number.
2. In `config/west.yml`, uncomment the `AYM1607` remote block and the
   `zmk-driver-azoteq-iqs5xx` project block.
3. In `config/ergo_right.conf`, uncomment `CONFIG_INPUT=y` and
   `CONFIG_ZMK_POINTING=y`.
4. In `boards/shields/ergo_right/ergo_right.overlay`, uncomment the
   trackpad node at the bottom of the `&i2c0` block, filling in the
   `<PORT>` and `<PIN>` for the reset pin you wired.
5. Push. The driver will then be pulled in via west, and the trackpad
   node will resolve against the AYM1607 binding.

If you'd rather use a different fork (e.g. `Ahmed-M-Osman1/zmk-driver-azoteq`),
the property names may differ slightly — check that fork's README and
adjust before pushing.

## Split note

This build treats `ergo_right` as a **standalone unibody keyboard** so
you can flash and test the right half on its own. When you build the
left half, you'll want to:

- Add `ergo_left` as a sibling shield in `boards/shields/ergo_left/`
- Set `ZMK_SPLIT_ROLE_CENTRAL` on the left in its `Kconfig.defconfig`
- Set `ZMK_SPLIT` on both halves
- Add `siblings: [ergo_left, ergo_right]` to the `.zmk.yml` files

That's a follow-up — none of it is needed to get this build green.
