# Raw data

One folder per lab. **Raw** means the original files the measurements came from
(serial logs, CSV, captures), not the plots in the reports. Every number in the
DDR has to point to a file here.

Capture the serial console to a file directly:

```bash
picocom -b 115200 --logfile data/lab1/20261002_ping_ch15_10m.log /dev/ttyACM0
```
