# Experiment #010 — Quaternion Attitude Estimation

Raw datasets accompanying the EmbedStudio experiment:

**#010 Quaternion Attitude Estimation**

## Objective

Experiment #009 blended the accelerometer and the gyroscope, and the blend
worked — on three Euler angles, which stop being three independent numbers near
±90° of pitch. This experiment keeps the fusion idea and changes what the
estimator stores. Instead of roll, pitch and yaw it keeps a quaternion,
propagated by the gyroscope and corrected by a proportional gravity feedback
term.

**Three estimators run on the target at once**, over the same samples, with
proportional gains of 0.1, 1 and 10. Every comparison in the article is
therefore between estimates of the *same* motion rather than between two
recordings.

## Acquisition

- Sensor: MPU6050
- Accelerometer range: ±2 g
- Gyroscope range: ±250 °/s
- DLPF bandwidth: 21 Hz
- Acquisition rate: 1 kSPS
- Acquisition tool: EmbedStudio

## How the estimate is computed

All of it runs on-target, in the firmware, once per sample.

**Propagation.** The angular velocity goes into a quaternion with a zero scalar
part, and the orientation is advanced by one Euler step and renormalised:

```
Omega = [0, wx, wy, wz]                     // rad/s, BODY frame
q     = q + 0.5 * (q (x) Omega) * dt
q     = q / |q|
```

`dt` is the interval measured per sample (experiment #004), not a nominal one.

**Correction.** The estimate predicts where gravity should be, the accelerometer
says where it is, and the cross product of the two is the error:

```
g_est = q* . [0, 0, 1]                      // world gravity, into the sensor frame
e     = normalize(a) x g_est
w_corrected = w_gyro + Kp * e
```

**The correction is added to the RATE, not to the orientation.** Nothing pushes
the quaternion towards the accelerometer. The loop produces an angular velocity
and that angular velocity reaches the orientation through the same integrator
the gyroscope feeds. This is the structural difference from the complementary
filter of experiment #009, which moved the angle itself.

There is no trigonometric function in the update. The only transcendental cost
is one square root per normalise.

### The gains

| Estimator | Kp | Quaternion recorded |
|---|---|---|
| `ahrs[0]` | 0.1 | no |
| `ahrs[1]` | 1 | yes — `ahrs[1].orientation` |
| `ahrs[2]` | 10 | no |

⚠️ **The gain is not a channel.** It is a compile-time constant array in
`main.c` and never reaches a capture. Use the table above rather than looking
for it in the data. All three estimators expose their gravity error, error
vector, predicted gravity direction and Euler angles; only `ahrs[1]` has its
quaternion itself watched.

## Datasets

| Dataset | Movement | Purpose |
|---|---|---|
| [Convergence, 106 s](data/capture_20260919_103402_convergence.hdf5) | None — set down and left alone, with one deliberate orientation reset | How fast each gain recovers, and what each one is still carrying a minute later |
| [Pitch sweep, 27 s](data/capture_20260919_103633_pitch_sweep.hdf5) | Hand-held, through ±90° of pitch, twice each way | Whether the ±90° singularity is in the estimator or only in the Euler angles |

### The orientation reset

The convergence capture is stationary, which makes a working estimator
invisible — a correct estimate and a correct measurement look identical. So the
firmware carries a reset that deliberately breaks it.

At **6.835 s** every quaternion estimator is re-initialised as though gravity
pointed along the sensor's **+Y** axis. The board was lying level, so that is
90° of roll away from the truth, and the quaternion becomes
`[0.7071, 0.7071, 0, 0]`. The gyroscope reads nothing. Everything after that
instant is the accelerometer pulling the estimate back.

⚠️ **Find the reset by the step in `gravity_error_deg`, not by the
`reset_orientation` channel.** The firmware clears that flag faster than the
host polls, so it reads 0 in every sample of both captures — the same behaviour
`gyro_init_request` has in experiment #009.

⚠️ **The complementary filter is reset differently.** The same call passes the
same vector `(0, 1, 0)` to `ahrs_gyro_reset()`, where it is an *angle triple in
degrees* rather than an accelerometer reading. So
`complementary_filer_orientation_deg[3]` steps by about 1° and 4°, not by 90°.
The two are not comparable at that instant. ⚠️ **The gyro integrator is not
reset at all.**

### What the numbers are

**Convergence.** Time to come back from the 90.7° error to under 1°:

| Kp | To under 5° | To under 1° | Fitted time constant |
|---|---|---|---|
| 0.1 | 32.2 s | 47.8 s | 9.86 s |
| 1 | 3.14 s | 4.79 s | 0.9925 s |
| 10 | 0.31 s | 0.47 s | 0.0996 s |

**The loop's time constant is 1/Kp seconds.** The fits are least squares on
`ln(gravity_error_deg − asymptote)` over the stretch where the error is between
30° and 3°. ⚠️ **Above 30° the fit does not apply** — the error signal is
`sin(theta_e)`, so the first part of each decay is slower than an exponential.

**Settled.** Over 72–105 s, 4217 samples, with the board still:

| | Kp = 0.1 | Kp = 1 | Kp = 10 |
|---|---|---|---|
| gravity angle error, mean | 0.640° | 0.182° | 0.170° |
| gravity angle error, sd | 0.152° | 0.115° | 0.111° |
| roll offset from the accelerometer's mean | +0.571° | +0.054° | +0.005° |
| pitch offset from the accelerometer's mean | +0.251° | +0.026° | +0.003° |

The accelerometer's own roll over that window is −0.8315° with a standard
deviation of 0.1565°, and the estimators' roll standard deviations are 0.029,
0.007 and 0.039°.

**The low gain leaves a standing error and it scales as 1/Kp.** A proportional
controller needs a non-zero error to produce the correction that cancels the
gyroscope bias. Removing it needs an integral term.

⚠️ **The two error means at Kp = 1 and Kp = 10 are at the accelerometer's own
noise floor.** `gravity_error_deg` is an `acos`, so it cannot be negative and
noise biases it upward. Those two are not distinguishable from each other here;
the roll and pitch offsets are, because they are signed.

**Pitch sweep.** The board goes to −88.5° and +89.5° of pitch. Over the whole
27 seconds:

- the largest single-sample change in any quaternion component is **0.0051**,
  and it happens **outside** the near-vertical region;
- inside the stretches beyond 85° of pitch, the largest is **0.0027**;
- the largest single-sample change in the Euler roll computed from that same
  quaternion is **36.4°**, at 89.5° of pitch;
- the quaternion never leaves unit length by more than **0.00026**.

The sharpest stretch is 22.577 → 22.639 s, 62 ms long: the Euler roll moves
from −156.5° to −16.9°, a swing of **139.6°**, while no quaternion component
moves by more than **0.0143** and the physical orientation — the angle between
the two quaternions, `2·acos(|q1·q2|)` — changes by **3.17°**.

⚠️ **This is not a defect in the estimate.** The Euler angles are a correct
reading of a correct quaternion. The singularity belongs to the three-angle
representation near ±90° of pitch, which is experiment #007's boundary seen
from the other side.

### What must not be derived from this data

⚠️ **Neither capture measures what a high gain costs during motion.** A high
gain lets more of the accelerometer through, and an accelerometer cannot tell
gravity from any other acceleration. The convergence recording is **stationary**,
so the accelerometer is never wrong in it. The pitch sweep is hand-held and does
carry linear acceleration (|a| between 0.886 and 1.150 g), but it has no ground
truth and no deliberate translate-without-rotating segment. **Experiment #009's
linear-acceleration capture is the one that measures this.**

⚠️ **Do not read the gains' `gravity_error_deg` during the sweep as disturbance
sensitivity.** At a high gain the estimate follows the accelerometer, disturbed
or not, so a *small* error there can mean a *wrong* estimate.

⚠️ **Nothing here characterises yaw.** Gravity is a direction, so the correction
constrains two degrees of freedom out of three and leaves yaw as the pure
integration of experiment #008. The quaternion's yaw is not recorded at all.

⚠️ **Nothing here characterises gyroscope bias or its temperature dependence.**
The gyro-only integration walks +5.24° of roll, +2.49° of pitch and +0.16° of
yaw over 104.7 s in the convergence capture. That is one run at one temperature,
included as the uncorrected reference of experiment #008. The die temperature
moves 0.2 °C in that capture and 1.0 °C in the sweep.

⚠️ **Do not derive a per-estimator cost from the timing channel.**
`timings.signal_processing_us` is the whole on-target chain — three quaternion
estimators, six complementary filters and everything else. It reads 81 µs in
10890 of the stationary capture's 13458 samples, against a 414 µs sensor read.
In the hand-held sweep it spreads over 78–100 µs, and nothing here explains
that.

HDF5 files are raw binary datasets and cannot be previewed directly by GitHub.
Download the dataset and open it with EmbedStudio or another compatible HDF5
tool.

## Workspace

[`MPU-6050_010.esws`](MPU-6050_010.esws) is the EmbedStudio workspace these
captures were recorded with. It carries an extra **Article figures** dashboard
whose views are the ones the screenshots in the article were taken from:

| View | Article figure |
|---|---|
| Quaternion · convergence | 6 |
| Gravity · measured against estimated | 7 |
| Gravity error · three gains | 8, and 9 (upper) |
| Roll · three gains against the accelerometer | 9 (lower) |
| Quaternion · through vertical | 10 (upper) |
| Euler angles · through vertical | 10 (lower) |
| Quaternion against the complementary filter | — |
| Die temperature | — |
| Zero-offset and reset state | — |

Open the workspace, then open a capture in review mode. Every view draws on both
captures.

⚠️ **The gravity view draws the accelerometer scaled by 0.976243**, which is
1/|a| measured in the convergence capture, because `accel_g` is recorded in g
while `gravity_estimated` is a unit vector. The recorded channel is the raw
value. In that view the measured direction is drawn 6 px at 40 % opacity under
the estimated one at 2 px, because after convergence the two carry the same
numbers and one would otherwise simply hide the other.

`elf_path` and the SWD probe serial were cleared before publishing. Neither is
needed to review a capture.

## Channels

Each capture carries the calibrated accelerometer (`accel_g`) and gyroscope
(`gyro_dps`), the accelerometer attitude (`attitude_deg`), the integrated gyro
angles (`gyro_angles_deg`), all three estimators' gravity error
(`ahrs[i].gravity_error_deg`, `ahrs[i].error_norm`, `ahrs[i].error`) and
predicted gravity direction (`ahrs[i].gravity_estimated`), the Euler angles
derived from each estimator (`quaternion_orientation_deg[i]`), the quaternion
itself for `ahrs[1]`, the τ = 500 ms complementary filter of experiment #009
(`complementary_filer_orientation_deg[3]`), the zero-offset and reset state, the
die temperature and two firmware timing counters.

A few things that are easy to read wrong:

- **`accel_g` is in g and is not normalised.** Its magnitude is about 1.0243 in
  the convergence capture. The firmware normalises its own copy inside
  `ahrs_update()`. To compare it with `gravity_estimated`, which is a unit
  vector, scale it by 1/|a|.
- **`gyro_dps` is in degrees per second.** ⚠️ The estimator is fed radians per
  second; the conversion happens at the call site and is not a recorded channel.
- **`gravity_error_deg` carries a handful of NaN samples** — 10 and 12 in the
  convergence capture for Kp = 1 and Kp = 10, and 3 and 2 in the sweep. The dot
  product of two unit vectors rounds fractionally above 1 in float32 and
  `acosf()` returns NaN. That is the error being essentially zero, not a fault.
  **Drop those samples rather than clamping the channel.**
- **`quaternion_orientation_deg[i].x` is roll and `.y` is pitch.** Yaw is not
  watched.
- **`ahrs[0].error.v[0..2]`** holds the same three values as
  `ahrs[0].error.__struct.x/y/z`; the difference is how the symbol was resolved,
  not what it holds.
- The firmware's own spelling of the complementary filter identifier is kept,
  typo included, so the channel names here match the channel names in the
  captures.

The sensor samples at 1 kHz. Samples reach the host over SWD at about 127 per
second in these captures, which is the polling rate of the acquisition rather
than a property of the sensor — **do not read the sample count of a capture as a
sensor rate.**

⚠️ **Each variable is read in its own SWD transaction**, so two channels recorded
in the same row can come from firmware iterations a millisecond apart. At the
reset, `ahrs[1].orientation` shows the new value one row before
`quaternion_orientation_deg[1]` does. That is the skew, not a firmware ordering.
**Compare fast events on the median, not the peak.**

## Related article

**#010 Quaternion Attitude Estimation**

The article describes the experimental setup, measurements, analysis and
results obtained from these datasets.
