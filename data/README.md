# Raw data

One folder per lab. **Raw** means the original files the measurements came from
(serial logs, CSV, captures), not the plots in the reports. Every number in the
DDR has to point to a file here.

Capture the serial console to a file directly:

```bash
picocom -b 115200 --logfile data/lab1/20261002_ping_ch15_10m.log /dev/ttyACM0
```

## Where this data comes from (read first)

The serial consoles were watched live with `picocom`/`minicom` and **no log files
were kept**. What survives is in the reports (`docs/informes/`):

- `labN/capturas/`: console screenshots and figures extracted from each report as
  they are (`fig-<page>-<n>.png`).
- `labN/*.csv`: measurement tables **transcribed** from the report; the first line
  of each file names the report table. They are not original logs.

New measurements are saved directly to a file (`picocom --logfile`, see above).
