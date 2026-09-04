# Corne ZMK config

Firmware configuration for my wireless Corne (crkbd) split keyboard, built with
[ZMK](https://zmk.dev). Every setting lives in this repo; GitHub Actions
compiles the firmware on each push.

## Hardware

| Part        | Value                                                        |
| ----------- | ------------------------------------------------------------ |
| Keyboard    | Corne (crkbd), 42 keys, split                                |
| Controllers | ProMicro / SuperMini nRF52840 (nice!nano v2 compatible)      |
| Displays    | 0.91in 128x32 OLED (SSD1306) on both halves, nice!oled widgets |
| ZMK build   | `nice_nano_v2` board + `corne_left` / `corne_right` shields  |
| ZMK version | `v0.3` (pinned in `config/west.yml` and the workflow)        |

The **left half is the central**: it pairs with the computer and the right
half only talks to the left half.

## Layout of this repo

```
build.yaml                    which firmware files to build (board + shield matrix)
config/corne.conf             firmware settings (Bluetooth power, screens, sleep, ...)
config/corne.keymap           the key layout, 3 layers
config/west.yml               which ZMK version and modules to build against
.github/workflows/build.yml   GitHub Actions build
boards/shields/               empty; custom shields could go here later
docs/superpowers/specs/       design notes
```

## Edit, build, flash

1. Edit `config/corne.keymap` or `config/corne.conf`.
2. Commit and push. The **Build ZMK firmware** workflow runs for about
   five minutes. Watch it with:
   ```sh
   gh run watch
   ```
3. Download the firmware zip from the run page, or from the terminal:
   ```sh
   gh run download -n firmware -D firmware/
   ```
   The `firmware/` folder is git-ignored, so downloaded binaries never get
   committed.
   You get three files:
   - `corne_left.uf2`
   - `corne_right.uf2`
   - `settings_reset.uf2`
4. Flash each half (see below).

### Flashing a half

1. Plug the half into the computer with a data USB-C cable.
2. Double-tap the controller's reset button quickly. A USB drive named
   `NICENANO` appears.
3. Drag the matching `.uf2` onto that drive. The drive disappears by itself
   when the copy finishes and the half reboots with the new firmware.
4. Unplug and repeat for the other half with the other file.

Only ever copy `corne_left.uf2` to the left half and `corne_right.uf2` to the
right half.

### First flash after a config change to Bluetooth, or when the halves stop pairing

Do a full reset so the halves pair with each other from scratch:

1. Flash `settings_reset.uf2` to the **left** half.
2. Flash `settings_reset.uf2` to the **right** half.
3. Flash `corne_left.uf2` to the left half.
4. Flash `corne_right.uf2` to the right half.
5. Power both halves on at roughly the same time. They pair within a few
   seconds.
6. On the computer, **forget** the old "Corne" Bluetooth device and pair
   again. The settings reset also wiped the computer pairing.

While `settings_reset` is on a half, Bluetooth is disabled on purpose, so
nothing shows up in any device list until step 3 and 4 are done.

## The split-connection fix

The halves used to lose each other when moved a few centimetres apart.
ZMK ships with the radio at 0 dBm. `config/corne.conf` raises it to the
nRF52840 maximum:

```ini
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y
```

This applies to both halves, so both the half-to-half link and the link to
the computer get stronger. ZMK's documentation recommends exactly this and
notes the battery cost is negligible:
<https://zmk.dev/docs/troubleshooting/connection-issues#unreliableweak-connection>

### If they still drop after the fix

At that point it is almost certainly physical, not firmware. Check, in
order:

- **Battery placement.** On many builds the battery sits directly on top of
  the controller's antenna (the zig-zag trace at the USB end of the board).
  Move it off the antenna or under the PCB.
- **OLED placement.** The display sits over the controller. Make sure
  its PCB and pins are not covering the antenna trace.
- **Metal.** A metal case, metal plate, or a metal desk right under the
  antenna kills range.
- **Battery level.** A nearly flat battery on either half causes drops long
  before it shows as "empty". Charge both fully and retest.
- **Interference.** USB 3 hubs and SSDs next to the keyboard, and 2.4 GHz
  dongles, are common culprits. Move them.
- **The clone itself.** SuperMini / ProMicro nRF52840 clones have a poorly
  tuned antenna on some batches. If one half is much worse than the other,
  swap the controllers between halves to confirm, then replace the bad one.

## Screens

The OLEDs run the [nice!oled](https://github.com/mctechnology17/zmk-nice-oled)
module, pulled in through `config/west.yml` and the `nice_oled` shield in
`build.yaml`. Left half: Bluetooth profile, battery, words per minute with a
speedometer and Luna the dog, active modifiers, layer name. Right half:
battery and a cat animation.

To change the animations, edit the `Display` block in `config/corne.conf`;
the alternatives are listed there as commented lines. The module is pinned
to one commit in `config/west.yml`; bump that revision to update it.

## Remapping keys live with ZMK Studio

The left half is built with ZMK Studio, so you can change keys from a
browser without rebuilding:

1. Plug the **left** half into the computer over USB.
2. Open <https://zmk.studio> in Chrome or Edge, click Connect, and pick the
   Corne serial device.
3. Hold the Lower thumb key and press the key under `Z` (`UNLK` on the
   Lower layer). The keyboard stays unlocked while you use Studio.
4. Click a key in Studio, choose a new binding, and it applies immediately.

Studio edits are stored on the keyboard, not in this repo. The keymap in
`config/corne.keymap` is the baseline: Studio's restore-stock-settings
option returns to it, and so does flashing `settings_reset.uf2`. Once you
like a layout, copy it into the keymap file so it survives resets and
rebuilds.

On Linux your user needs access to the USB serial port:

```sh
sudo usermod -aG dialout $USER
```

then log out and back in.

## Changing the keymap in the repo

The layout lives in `config/corne.keymap`. Each layer is a `bindings` list
of 42 entries in the order of the physical keys: three rows of 12 (6 left,
6 right), then the 6 thumb keys (3 left, 3 right). The comment block above
each list draws the layer; keep it in sync when you edit.

Bindings you will use most:

| Binding          | Meaning                                                   |
| ---------------- | --------------------------------------------------------- |
| `&kp KEY`        | press a key: `&kp ESC`, `&kp LCTRL`, `&kp N1`, `&kp F5`    |
| `&mo N`          | hold to activate layer N                                  |
| `&trans`         | transparent: use whatever the layer below has              |
| `&none`          | do nothing                                                |
| `&mt MOD KEY`    | hold for modifier, tap for key: `&mt LCTRL ESC`            |
| `&lt N KEY`      | hold for layer N, tap for key: `&lt 1 SPACE`               |
| `&bt BT_SEL 0`   | switch to Bluetooth profile 1 (profiles are 0 to 4)        |
| `&studio_unlock` | unlock the keyboard for ZMK Studio                        |

Worked example: make the key under `Tab` (currently `LCTRL`) send Escape
when tapped and act as Control when held. In the default layer change

```
&kp LCTRL &kp A &kp S ...
```

to

```
&mt LCTRL ESC &kp A &kp S ...
```

Then commit, push, download the firmware, and flash both halves (the right
half needs the new keymap too, or its keys keep the old meaning).

- Key codes: <https://zmk.dev/docs/keymaps/list-of-keycodes>
- Behaviors (layers, mod-tap, combos, ...): <https://zmk.dev/docs/keymaps/behaviors>
- A visual editor that commits straight to this repo:
  <https://nickcoutsos.github.io/keymap-editor/>

Layers as shipped:

| Layer   | Reached by           | Contents                                          |
| ------- | -------------------- | ------------------------------------------------- |
| Default | always               | QWERTY, Tab/Ctrl/Shift, GUI/Space, Enter/Alt      |
| Lower   | hold left thumb key  | numbers, Bluetooth profiles, arrows, Studio unlock |
| Raise   | hold right thumb key | symbols and brackets                              |

Bluetooth profiles on the Lower layer: `BT1` to `BT5` switch between up to
five paired computers, `BTCLR` forgets the current profile's pairing.

## Upgrading ZMK later

Change the revision in `config/west.yml` and the `@v0.3` tag in
`.github/workflows/build.yml` together. Note that ZMK after `v0.3` renamed
the board id from `nice_nano_v2` to `nice_nano//zmk`, so `build.yaml`
will need that change at the same time.
