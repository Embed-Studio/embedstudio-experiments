# Experiment #011 — Mahony AHRS and Online Gyroscope Bias Estimation

Raw datasets accompanying the EmbedStudio experiment:

**#011 Mahony AHRS and Online Gyroscope Bias Estimation**

## Objective

Experiment #010 corrected the quaternion with a proportional gravity term and
found that it leaves a standing error when the gyroscope has a bias. This
experiment adds the integral term of a Mahony filter, so that the estimator
learns the bias while the firmware keeps running.

**Four estimators run on the target at once**, over the same samples, with
integral gains of 0.1, 1, 10 and 0 (the proportional-only baseline). Every
comparison in the article is therefore between estimates of the *same* motion,
not between two recordings.

## Acquisition

- Sensor: MPU6050
- Accelerometer range: ±2 g
- Gyroscope range: ±250 °/s
- DLPF bandwidth: 21 Hz
- Acquisition rate: 1 kSPS
- Acquisition tool: EmbedStudio

## How the estimate is computed

All of it runs on-target, once per sample, with Kp = 1:

```
e     = normalize(a) x g_est                // gravity error
b     = b - Ki * e * dt                     // bias estimate, rad/s
w     = w_gyro - b + Kp * e                 // rate fed to the quaternion
q     = q + 0.5 * (q (x) [0, w]) * dt, then normalise
```

### The gains

| Estimator | Ki | Quaternion recorded |
|---|---|---|
| `ahrs[0]` | 0.1 | no |
| `ahrs[1]` | 1 | yes — `ahrs[1].orientation` |
| `ahrs[2]` | 10 | no |
| `ahrs[3]` | 0 | no — the proportional-only baseline of #010 |

⚠️ **The gains are not channels.** They are a compile-time constant array in
`main.c` and never reach a capture. Use the table above rather than looking for
them in the data.

## Datasets

| Dataset | Condition | Purpose |
|---|---|---|
| [Offset test, 11.9° tilt, 70 s](data/capture_20261010_081806_gyro_bias_step.hdf5) | Still board, three offsets written live | How fast each gain learns a known offset, and what it ends at |
| [Offset test, 2.3° tilt, 61 s](data/capture_20261010_132717_gyro_bias_flat.hdf5) | The same, board almost flat | Most of the offset is observable |
| [Offset test, 48.2° tilt, 64 s](data/capture_20261010_132429_gyro_bias_tilted.hdf5) | The same, board tilted | Most of the offset is not observable |
| [Thermal drift, 606 s](data/capture_20261010_121258_thermal_stabilization.hdf5) | Still board, warmed by about 16 °C and left to cool | Whether the estimators follow a real offset drift |

### The offset test

The gyroscope calibration offset `mpu_cal.gyro_bias` (raw LSB) is overwritten by
hand over SWD, one axis at a time: **+448 LSB on X, +573 on Y and +444 on Z**,
with the board standing still. With the scale factors 132.2, 131.1 and
130.9 LSB per °/s this imposes **−3.39, −4.37 and −3.40 °/s**
(−0.0592, −0.0763 and −0.0593 rad/s). The writes are visible as steps in the
`mpu_cal.gyro_bias` channels; nothing else marks them. They reproduce to within
0.0001 rad/s across the three captures.

⚠️ **The estimate does not reach the applied offset, by design.** Gravity
constrains only the part of the bias perpendicular to it. The part along gravity
cannot be seen by the integral term and appears as yaw drift instead. The share
that is observable depends on the tilt:

| Tilt | Observable share of the offset | Yaw drift after the writes |
|---|---|---|
| 2.3° | 86 % | −3.27 °/s |
| 11.9° | 73 % | −4.44 °/s |
| 48.2° | 18 % | −6.39 °/s |

At Ki = 1 and Ki = 10 the estimates end within 1 % of the observable part at all
three tilts. The captures at 2.3° and 48.2° end 27–28 s after the last write, so
Ki = 0.1 has not settled in them.

### The thermal capture

The board is not moved. The die temperature rises from 25.0 to 41.3 °C
(peak at 86.5 s) and falls back, and the gyroscope offset moves by about 54, 47
and 19 LSB on X, Y and Z. No offset is written. The board stands at a tilt of
about 48.7° (accelerometer roll 41.1°, pitch −28.5°).

⚠️ **The offset did not return to its earlier value after the thermal cycle.**
This is one part and one cycle; do not fit a temperature coefficient to it and
do not assume the same on another unit.

### What must not be derived from this data

⚠️ **Each tilt was captured once**, with no independent attitude reference, so
there is no run-to-run spread.

⚠️ **Convergence times come from the first capture only.** The two later offset
captures are too short for Ki = 0.1 to settle.

⚠️ **The imposed offsets are not a measurement of the sensor's own bias.** They
are numbers written into the calibration for the test.

⚠️ **Nothing here characterises heading.** The yaw of `ahrs[1]` is used only for
its slope, and in the 48.2° capture it reaches −249° by the end of the record.

## Channels

Each capture carries the calibrated accelerometer (`accel_g`) and gyroscope
(`gyro_dps`), the accelerometer attitude, the integrated gyro angles, the
calibration offset (`mpu_cal.gyro_bias`), each estimator's bias estimate
(`ahrs[i].bias`), gravity error and predicted gravity direction, the Euler
angles of each estimator (`quaternion_orientation_deg[i]`), the quaternion of
`ahrs[1]`, the die temperature and two firmware timing counters.

- **`ahrs[i].bias` is in rad/s**, in the sensor frame, and is subtracted from
  the rate.
- **`mpu_cal.gyro_bias` is in raw LSB**: `rate = (raw − bias) / scale`.
- **`gyro_dps` is in degrees per second**; the estimator is fed radians per
  second, converted at the call site.
- **`quaternion_orientation_deg[i].x` is roll and `.y` is pitch.** Yaw is not
  watched; compute it from `ahrs[1].orientation`.
- The firmware's own spelling `complementary_filer_orientation_deg` is kept,
  typo included.

The sensor samples at 1 kHz. Samples reach the host over SWD at about 96 per
second, which is the polling rate of the acquisition rather than a property of
the sensor — **do not read the sample count of a capture as a sensor rate.**
⚠️ Each variable is read in its own SWD transaction, so two channels in the same
row can come from firmware iterations a millisecond apart.

## Results and scripts

`results/` holds the analysis that produced the article's numbers:
`experiment2_analysis.py` (the first offset test), `experiment2_tilts_analysis.py`
(the two tilted repeats), `experiment_a_analysis.py` (the thermal capture), and
the processed CSVs and `experiment2_numbers.json`. The scripts read the files in
`data/` by absolute path; edit the path at the top before running elsewhere.

HDF5 files are raw binary datasets and cannot be previewed directly by GitHub.
Download the dataset and open it with EmbedStudio or another compatible HDF5
tool.

## Related article

**#011 Mahony AHRS and Online Gyroscope Bias Estimation**

The article describes the experimental setup, measurements, analysis and
results obtained from these datasets.
