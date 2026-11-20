# IoT Systems Design (UNAL 4201327): team work, Labs 0–5

Team firmware, raw data and Design & Decision Record (DDR) for the course
[saacifuentesmu/4201327-iot_systems_design](https://github.com/saacifuentesmu/4201327-iot_systems_design)
(SoilSense / GreenField Technologies).

## Team

| Name | Email | GitHub |
|---|---|---|
| Samuel Elias Gomez Munoz | sgomezmu@unal.edu.co | [@sgomezmunoz](https://github.com/sgomezmunoz) |
| Juan Fernando Martinez Cabrera | jumartinezcab@unal.edu.co | |

| Path | Contents |
|---|---|
| [docs/DDR.md](docs/DDR.md) | Design & Decision Record, with ADRs backed by our measurements |
| [docs/informes/](docs/informes/) | Lab reports 1–5 (PDF) |
| [firmware/](firmware/) | Zephyr applications Lab 0–5, with the team's changes |
| [data/](data/) | Raw data per lab (logs, CSV, captures) |
| [west.yml](west.yml) | Zephyr manifest pinned by the course |

## Base and team changes

The firmware starts from the course repository at commit `ff6e675`. Team changes
over that base:

| Lab | File | Change | Why |
|---|---|---|---|
| 0 HTTP | `prj.conf`, `src/main.c` | Task 1 capabilities (`WIFI`, `HTTP_SERVER`, `JSON_LIBRARY`, `LED_STRIP`), `NET_L2_WIFI_MGMT`, completed `main.c` | Lab tasks / Wi-Fi stability |
| 2 | `prj.conf` | `ESPTOOLPY_FLASHSIZE_4MB` | Our C6 boards have 4 MB of flash |
| 3 | `prj.conf`, board `.overlay` | 4 MB flash, console and shell over `usb_serial` | Flash size and USB console |
| 4 | `flash16.conf` (new) | `ESPTOOLPY_FLASHSIZE_16MB` | Board with 16 MB of flash |
| 5 RCP | `prj.conf` | 4 MB flash | C6 used as RCP |
| 5 BR | `prj.conf`, S3 `.overlay` | 16 MB flash, console over `usb_serial` | ESP32-S3 N16R8 |

## Hardware

- ESP32-C6-DevKitC (nodes and RCP)
- ESP32-S3-DevKitC with PSRAM (Border Router host)
- DHT22 sensor (Lab 4)

## Build and flash

Run from the folder of each lab, with the Zephyr environment active (`~/zephyrproject`):

```bash
# Lab 0 HTTP: Wi-Fi credentials in secrets.conf (copy secrets.conf.example; not committed)
west build -p always -b esp32c6_devkitc/esp32c6/hpcore . -- -DEXTRA_CONF_FILE=secrets.conf

# Labs 1–3 (C6 node)
west build -p always -b esp32c6_devkitc/esp32c6/hpcore -S espressif-flash-4M .

# Lab 4 with the DHT22 and the console over USB
west build -p always -b esp32c6_devkitc/esp32c6/hpcore . -- \
  -DSNIPPET=espressif-flash-4M -DEXTRA_CONF_FILE=usb_console.conf \
  -DEXTRA_DTC_OVERLAY_FILE="usb_console.overlay;sensors/dht22.overlay"

# Lab 5: RCP on the C6, Border Router on the S3
west build -p always -b esp32c6_devkitc/esp32c6/hpcore -S espressif-flash-4M .   # lab5_rcp
west build -p always -b esp32s3_devkitc/esp32s3/procpu -S espressif-flash-16M .  # lab5_border_router

west flash -d build --esp-device /dev/ttyACM0
```
