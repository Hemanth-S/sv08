# Firmware build configs

Klipper firmware settings for the two boards that run Klipper-built firmware.
The Cartographer runs Cartographer's own firmware and is not built from here.

| File | Board | Key settings |
|------|-------|--------------|
| `octopus.config` | BTT Octopus Pro v1.1 (STM32H723) | 128KiB bootloader, 25 MHz crystal, USB on PA11/PA12 |
| `nitehawk.config` | LDO Nitehawk-36 (RP2040) | No bootloader, W25Q080, USB, `INITIAL_PINS=!gpio9` (hotend heater held low at boot) |
| `nitehawk-xosc-startup-64ms.patch` | Nitehawk-36 | Klipper source patch, see "Nitehawk cold-start patch" below |

The files are minimal: `make olddefconfig` expands them with Klipper's defaults.
Both were checked against Klipper master (2026-09-28) and expand to exactly
what the boards reported running before the update: stm32h723xx @ 520 MHz,
app at 0x8020000; rp2040 @ 12 MHz, app at 0x10000100, `!gpio9`.

## Reflashing after a Klipper update

Only needed if Klipper refuses to connect and asks for a reflash. On the Pi:

```bash
sudo systemctl stop klipper
cd ~/klipper

# Nitehawk-36: build from a PATCHED COPY of Klipper, never from ~/klipper
# (see "Nitehawk cold-start patch"). Flashes over USB, no BOOT button needed.
rm -rf ~/klipper-nitehawk && git clone -q ~/klipper ~/klipper-nitehawk && cd ~/klipper-nitehawk
git apply ~/sv08/firmware/nitehawk-xosc-startup-64ms.patch
cp ~/sv08/firmware/nitehawk.config .config && make olddefconfig && make -j4
make flash FLASH_DEVICE=/dev/serial/by-id/usb-Klipper_rp2040_303339383405A682-if00
cd ~/klipper

# Octopus Pro: writes firmware to the Octopus's microSD card over USB
cp ~/sv08/firmware/octopus.config .config && make olddefconfig && make clean && make -j4
./scripts/flash-sdcard.sh /dev/serial/by-id/usb-Klipper_stm32h723xx_320020001151313531383332-if00 btt-octopus-pro-h723-v1.1

# >>> Power-cycle the PRINTER now (not the Pi). The Octopus's bootloader reads
# >>> the card in SDIO mode, but the script wrote it over SPI, so the board and
# >>> card need a power-cycle before the bootloader flashes firmware.bin.
# >>> Klipper's board definition marks it skip_verify for this reason.

# After it powers back up, verify the flash:
./scripts/flash-sdcard.sh -c /dev/serial/by-id/usb-Klipper_stm32h723xx_320020001151313531383332-if00 btt-octopus-pro-h723-v1.1

sudo systemctl start klipper
```

If the Nitehawk is missing after a power-cycle (`ls /dev/serial/by-id/` shows
no `rp2040`), press RESET on the Nitehawk, then restart the Klipper service.

Copy each file to `~/klipper/.config` rather than pointing `KCONFIG_CONFIG` at
it: `make olddefconfig` rewrites the config file in place, which would leave
this repo dirty on the Pi and block the next `git pull`.

`flash-sdcard.sh` needs a microSD card in the Octopus (FAT32, MBR, 32 GB or
smaller, can be empty) and the stock BTT bootloader. The Octopus has the stock
bootloader (confirmed 2026-09-30: no Katapult on the Pi).

If the Nitehawk doesn't come back after `make flash`, hold its BOOT button while
powering up (it appears as `2e8a:0003`), then run
`make flash FLASH_DEVICE=2e8a:0003`.

## Nitehawk cold-start patch

Symptom (2026-09-30 to 2026-10-07): at printer power-on the Nitehawk's USB hub
and the Cartographer behind it enumerated, but the RP2040 itself never
appeared, 4 of 5 cold starts. RESET usually brought it up; BOOT mode always
worked. So the chip, the hub and USB were fine, and the failure was in the
Klipper firmware's own start-up.

Klipper's RP2040 start-up gives the 12 MHz crystal a fixed 1 ms to stabilise
(`xosc_setup()` in `src/rp2040/main.c`). Raspberry Pi's pico-sdk makes this
configurable (`PICO_XOSC_STARTUP_DELAY_MULTIPLIER`) because some boards'
crystals start slower, and several commercial RP2040 boards ship with x64.
`nitehawk-xosc-startup-64ms.patch` applies the same x64 (47 -> 3008 cycles of
256, about 64 ms; the register field allows up to 8191).

Flashed 2026-10-07 as `v0.13.0-777-g7bc4d0946-dirty-...-sv08`, built in
`~/klipper-nitehawk` (a separate clone) so `~/klipper` stays clean for
Moonraker updates. If `git apply` fails after a future Klipper update, the
line moved: edit `xosc_hw->startup = ...` in `src/rp2040/main.c` by hand.
To undo, flash a build of unpatched `~/klipper` with `nitehawk.config`.
