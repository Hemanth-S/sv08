# sv08

Klipper config for a Sovol SV08-based CoreXY printer with a heavily customised toolhead.

## Hardware

- **Frame & motion:** Sovol SV08 (350×350×345 CoreXY, quad-Z)
- **Mainboard:** BIGTREETECH Octopus Pro v1.1 (STM32H723) — motion + bed
- **Toolhead:** [LDO Jabberwocky](https://docs.ldomotors.com/) — Phaetus Conch hotend, Bondtech compound + dual LDO Orbiter extruder, dual 3010 part fans, dual filament sensors, servo-actuated cutter, Nitehawk-36 toolboard (RP2040)
- **Probe:** Cartographer v4 (STM32G431, eddy-current scanner)
- **Bed:** R3men 1000 W graphite AC-mains heated bed, NTC 100K MGB18-104F39050L32 thermistor, switched via SSR on HE1
- **Host:** Raspberry Pi 5 running MainsailOS

All three MCUs talk to the Pi over USB. Octopus has its own cable; Nitehawk-36 and Cartographer share a toolhead-mounted USB hub fed by a second USB cable from the Pi.

## File layout

| File | Contents |
|------|----------|
| `printer.cfg` | Top-level entry — `[include]` directives + `[printer]` kinematics |
| `mcu.cfg` | `[mcu]`, `[mcu nitehawk]`, `[mcu scanner]` |
| `steppers.cfg` | `[stepper_*]` + `[tmc2209 stepper_*]` for X / Y / Z1-Z4 |

Future files (commented out in `printer.cfg` until staged in):
`extruder.cfg`, `heater_bed.cfg`, `scanner.cfg`, `fans.cfg`, `macros.cfg`.

## Workflow

Edit on local, push, pull on the Pi:

```bash
# locally:
git push

# on the Pi:
cd ~/sv08
git pull
# then RESTART via Mainsail's web UI, or:
sudo service klipper restart
```

`SAVE_CONFIG` (Klipper auto-saved calibration values) lives in `~/printer_data/config/printer.cfg` on the Pi (outside this repo, by design — they're per-machine and conflict-prone). This repo is only the hand-edited config.

## Pi setup (one-time)

```bash
cd ~
git clone git@github.com:Hemanth-S/sv08.git
# back up MainsailOS's default config
mv ~/printer_data/config ~/printer_data/config.bak
# point Klipper at the git repo
ln -s ~/sv08 ~/printer_data/config
sudo service klipper restart
```
