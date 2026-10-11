# Input shaper baseline (2026-10-11, before the hollow X-beam swap)

`SHAPER_CALIBRATE` at bed centre (175, 175, Z20), Cartographer v4 ADXL345.
Stock X beam, umbilical slightly slack. Live config at the time:
X `zv` @ 118.4 Hz, Y `ei` @ 51.0 Hz (from 2026-09-09), `max_accel` 3000.

| Axis | Main resonance | Klipper's pick | Other fits (vibration left, max accel) |
| --- | --- | --- | --- |
| X | ~118 Hz (+ smaller bump ~88 Hz) | `mzv` 125.6 Hz, 10.2 %, accel <= 46500 | `ei` 110.0 Hz: 5.1 %, <= 22000 |
| Y | ~48 Hz (+ small bump ~35 Hz) | `2hump_ei` 80.8 Hz, 1.4 %, accel <= 6900 | `mzv` 48.4 Hz: 1.8 %, <= 6900; `ei` 61.6 Hz: 1.8 %, <= 6900 |

Y limits the machine at ~6900 mm/s^2. Compare against the post-swap run.

Note: the graphs label the dominant response on the *other* axis (the Y test's
48 Hz peak shows in the X channel and vice versa), so `[adxl345] axes_map`
doesn't match how the Cartographer is mounted. The shaper fit uses X+Y+Z, so
the recommendations are unaffected; fix it with `AXES_MAP_CALIBRATION`.
