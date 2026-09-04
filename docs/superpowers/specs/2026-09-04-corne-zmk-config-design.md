# Corne ZMK config repo: design

Date: 2026-09-04
Status: approved

## Problem

The wireless Corne's two halves lose their Bluetooth link to each other
when separated by a couple of centimetres. There was no versioned place
to keep the keyboard's firmware settings and keymap.

## Goals

1. Fix, or at least maximise the chance of fixing, the split-half link
   through firmware configuration.
2. Keep every keyboard setting in a git repository under `~/dev` so future
   changes (keymap, Bluetooth, sleep, displays) are tracked and rebuildable.

## Hardware (confirmed with the owner)

- Corne (crkbd) split keyboard.
- ProMicro / SuperMini nRF52840 controllers: nice!nano v2 compatible,
  ZMK board id `nice_nano_v2` on ZMK v0.3.
- 0.91in 128x32 SSD1306 OLED displays on both halves, on the 4-pin OLED header.
- Keymap starts from ZMK's default Corne keymap.

## Decision: build with GitHub Actions

Options considered:

1. GitHub Actions using ZMK's reusable `build-user-config.yml` workflow.
   Chosen: no local toolchain, the standard zmk-config workflow, and `gh`
   is already authenticated on this machine.
2. Local Docker build. Rejected for now: multi-GB container and `west`
   setup with no current need.
3. Both. Rejected: two paths to maintain.

The repo is public on GitHub (`MarcosBuratovich/corne-zmk-config`). It
contains only keymap and config, no secrets; public repos get unlimited
Actions minutes.

## Repo layout

Based on `zmkfirmware/unified-zmk-config-template`, pinned to ZMK `v0.3`.

```
build.yaml                    matrix: nice_nano_v2 x {corne_left, corne_right}
                              plus a settings_reset build
config/west.yml               ZMK v0.3
config/corne.conf             connection fix, OLED enable, commented options
config/corne.keymap           ZMK default Corne keymap, verbatim
.github/workflows/build.yml   reusable ZMK workflow @v0.3
zephyr/module.yml, boards/    template scaffolding for future custom shields
README.md                     edit / build / flash / reset / troubleshooting
```

## The connection fix

`CONFIG_BT_CTLR_TX_PWR_PLUS_8=y` in `config/corne.conf`. Raises the BLE
transmit power from 0 dBm to +8 dBm on both halves. ZMK's troubleshooting
docs recommend it for weak links and state it also improves the split
link; power cost is negligible.

Left commented, with explanation, for later:

- `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y` for a flaky link to the computer.
- `CONFIG_ZMK_SLEEP=y` plus idle timeout for battery life.

`CONFIG_ZMK_DISPLAY=y` in `corne.conf` turns on the OLED. The Corne shield
declares the SSD1306 node and enables I2C when the display is on, so no
extra display shield is needed.

## Verification

- Push the repo; the Actions run must succeed.
- Download the `firmware` artifact and confirm three `.uf2` files exist:
  `corne_left`, `corne_right`, `settings_reset`.
- Flashing and the physical range test are manual, done by the owner.
  The README documents the settings-reset-then-flash procedure so the
  halves pair fresh after the change.

## Known limits

- If +8 dBm does not fix it, the cause is physical (battery or display
  over the antenna, metal case, low battery, interference, or a badly
  tuned clone antenna). The README lists the checks.
- ZMK after v0.3 renames the board id to `nice_nano//zmk`; upgrading
  requires updating `build.yaml`, `west.yml`, and the workflow together.

## Amendment, 2026-09-04: display type corrected

The first build targeted a nice!view (Sharp memory LCD over SPI) based on
an early answer about the hardware. After flashing, both screens stayed
blank while the split link worked. The displays light up and use a 4-pin
header, which identifies them as SSD1306 OLEDs. The nice!view shields
selected the Sharp LCD as the display and disabled the I2C bus the OLED
sits on. Fix: plain `corne_left` / `corne_right` shields plus
`CONFIG_ZMK_DISPLAY=y`. Verified in the compiled firmware: chosen display
is the OLED node and `CONFIG_SSD1306=y`.

## Amendment, 2026-09-04: screens and live remapping

Approved after the OLED fix worked.

- Screens: the nice!oled module (`mctechnology17/zmk-nice-oled`, pinned to
  commit `46f824a`, tested by its author against ZMK v0.3.0) via the
  `nice_oled` shield on both halves and
  `CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y`. Left: speedometer, WPM number and Luna,
  set explicitly because the module's default at this commit is bongo cat
  with no speedometer. Right: the cat (module default). Modifier icons in
  Windows/Linux style. Alternatives left as commented lines in the conf.
- ZMK Studio on the left (central) half only: `studio-rpc-usb-uart`
  snippet plus `CONFIG_ZMK_STUDIO=y` in `build.yaml`, and a
  `&studio_unlock` binding on the Lower layer under `Z`. Studio edits live
  on the keyboard; the repo keymap remains the baseline.
- README: screens, Studio, and a keymap-editing guide with a worked example.
