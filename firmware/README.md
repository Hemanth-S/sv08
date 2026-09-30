# Firmware build configs

Klipper firmware settings for the two boards that run Klipper-built firmware.
The Cartographer runs Cartographer's own firmware and is not built from here.

| File | Board | Key settings |
|------|-------|--------------|
| `octopus.config` | BTT Octopus Pro v1.1 (STM32H723) | 128KiB bootloader, 25 MHz crystal, USB on PA11/PA12 |
| `nitehawk.config` | LDO Nitehawk-36 (RP2040) | No bootloader, W25Q080, USB, `INITIAL_PINS=!gpio9` (hotend heater held low at boot) |

The files are minimal: `make olddefconfig` expands them with Klipper's defaults.
Both were checked against Klipper master (2026-09-28) and expand to exactly
what the boards reported running before the update: stm32h723xx @ 520 MHz,
app at 0x8020000; rp2040 @ 12 MHz, app at 0x10000100, `!gpio9`.

## Reflashing after a Klipper update

Only needed if Klipper refuses to connect and asks for a reflash. On the Pi:

```bash
sudo systemctl stop klipper
cd ~/klipper

# Nitehawk-36: flashes over USB, no BOOT button needed
cp ~/sv08/firmware/nitehawk.config .config && make olddefconfig && make clean && make -j4
make flash FLASH_DEVICE=/dev/serial/by-id/usb-Klipper_rp2040_303339383405A682-if00

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

If the Nitehawk is missing after the power-cycle (`ls /dev/serial/by-id/`
shows no `rp2040`), it's the boot-order issue: the toolhead hub came up late.
Run `FIRMWARE_RESTART` once Klipper is started.

Copy each file to `~/klipper/.config` rather than pointing `KCONFIG_CONFIG` at
it: `make olddefconfig` rewrites the config file in place, which would leave
this repo dirty on the Pi and block the next `git pull`.

`flash-sdcard.sh` needs a microSD card in the Octopus (FAT32, MBR, 32 GB or
smaller, can be empty) and the stock BTT bootloader. The Octopus has the stock
bootloader (confirmed 2026-09-30: no Katapult on the Pi).

If the Nitehawk doesn't come back after `make flash`, hold its BOOT button while
powering up (it appears as `2e8a:0003`), then run
`make flash FLASH_DEVICE=2e8a:0003`.
