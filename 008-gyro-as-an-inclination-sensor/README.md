# Experiment #008 — Can a Gyroscope Be Used as an Attitude Sensor?

Raw datasets accompanying the EmbedStudio experiment:

**#008 Can a Gyroscope Be Used as an Attitude Sensor?**

## Objective

An accelerometer estimates inclination from the direction of gravity, and that
reference fails at certain orientations. A gyroscope does not use gravity at
all: it measures angular velocity, and integrating that velocity gives the
change in orientation since the accumulator was started.

These captures record what that buys and what it costs. They cover the
orientation where the accelerometer estimate breaks down, a rotation about the
vertical axis that the accelerometer cannot see at all, and the drift of the
integrated attitude on a stationary board — at steady temperature and while the
sensor cools.

## Acquisition

- Sensor: MPU6050
- Accelerometer range: ±2 g
- Gyroscope range: ±250 °/s
- DLPF bandwidth: 21 Hz
- Acquisition rate: 1 kSPS
- Acquisition tool: EmbedStudio

## How the attitude is computed

Roll and pitch are computed on-target by the firmware from the accelerometer
and recorded as the `attitude_deg` channels:

```
roll  = atan2(a_y, a_z)
pitch = atan2(-a_x, sqrt(a_y^2 + a_z^2))
```

The gyro attitude is a running sum of the calibrated rates, recorded as the
`gyro_angles_deg` channels:

```
theta[k+1] = theta[k] + omega[k] * dt
```

`dt` is the interval measured per sample (experiment #004), not a nominal one.
The accumulator has to be started from somewhere, so roll and pitch are
initialized from the accelerometer and yaw is initialized to zero.

**The three axes are integrated independently. This is deliberate, and it is
not a correct attitude estimator.** Body-frame angular rates and Euler-angle
rates are the same thing only near level; away from it the axes couple. The
roll capture measures how much that costs.

### The reset rule

`gyro_angles_deg` is reset to the accelerometer attitude on every iteration
while `gyro_zero.ready` is 0, and whenever `gyro_init_request` is set. Once the
zero-offset estimate is ready the resetting stops and the angles are updated by
integration alone. `gyro_init_request` is cleared by the firmware faster than
the host polls, so it reads 0 in every capture even where a manual reset
happened.

## Datasets

| Dataset | Movement | Purpose |
|---|---|---|
| [Gyro initialization](data/capture_20260905_141858_gyro_initializtion.hdf5) | Stationary | The integrator being released the moment the zero-offset estimate is ready |
| [Pitch](data/capture_20260905_135810_pitch.hdf5) | Pitched past vertical and back | Accelerometer pitch folds back at −90°, integrated pitch reaches −100° |
| [Roll](data/capture_20260905_140116_roll.hdf5) | Rolled past vertical and back | The two roll estimates agree — and the other two axes couple |
| [Yaw](data/capture_20260905_145133_yaw.hdf5) | Turned about the vertical axis | A full turn the accelerometer cannot see |
| [Drift, 5 min](data/capture_20260905_151502_drift_temperature_stable_5_min.hdf5) | Stationary, steady temperature | The drift measurement |
| [Drift, 3 min](data/capture_20260905_140627_drift_temperature_stable_3_min.hdf5) | Stationary, steady temperature | The same, shorter, at another orientation |
| [Drift, temperature varying](data/capture_20260905_140221_drift_temperature_varies_3_min.hdf5) | Stationary, 0.6 °C fall | Bias moving with a small temperature change |
| [Drift, cooling down](data/capture_20260905_145902_drift_cooling_down_3_min.hdf5) | Stationary, 8.6 °C fall | Bias moving with a large one |

### What the numbers are

In the **five-minute** capture the integration starts at 6.86 s, where
`gyro_zero.ready` goes high. Over the 305 s that follow, the integrated
attitude moved **+1.11° in roll, −0.80° in pitch and −0.37° in yaw** away from
the attitude it was started at, while the die temperature stayed within
0.14 °C. Those correspond to residual biases of about 0.0036, 0.0026 and
0.0011 °/s. Over the same window the accelerometer attitude wandered over
0.49° and 0.51° without going anywhere.

In the **cooling** capture the board is equally stationary — its accelerometer
roll and pitch stay inside 0.8° and 0.5° bands — and the integrated attitude
runs away by **31.8°, 20.9° and −9.1°** while the die falls from 38.5 °C to
29.9 °C.

⚠️ **That is a qualitative demonstration, not a temperature characterization.**
The cooling rate, the starting temperature and the gradient across the package
were all uncontrolled. **Do not derive a temperature coefficient from these
files.**

In the **pitch** capture the accelerometer pitch reaches −88.9° and folds back,
because `atan2` with a square root as its second argument cannot report past
±90°; its roll jumps to −176° to describe the same orientation. The integrated
pitch passes through to −100.0° as one continuous number, and over 11–13 s the
integrated roll stays inside a 0.55° band.

In the **roll** capture the two roll estimates agree through −100° to a median
of 1.0°. Rolling through 90° is not a singular orientation for
`atan2(a_y, a_z)`. What the capture does show is coupling: gyro pitch goes
−11.94 → −21.40 → −11.79 and gyro yaw 0 → +15.10 → −6.56, an excursion that
appears during the rotation and disappears when it stops.

In the **yaw** capture the board was turned by hand in four steps, with
plateaus at −82.5°, −179.7°, −251.3° and −346.3°. There is no jig and no
heading reference in the recording, so **the plateaus are where the operator
stopped and the total must not be read as a scale error.**

HDF5 files are raw binary datasets and cannot be previewed directly by GitHub.
Download the dataset and open it with EmbedStudio or another compatible HDF5
tool.

## Workspace

[`MPU-6050_008.esws`](MPU-6050_008.esws) is the EmbedStudio workspace these
captures were recorded with. It carries an extra **Article figures** dashboard
whose views are the ones the screenshots in the article were taken from:

| View | Article figure |
|---|---|
| Roll · accel vs gyro | 2, 3 (lower), 4 (upper) |
| Pitch · accel vs gyro | 3 (upper) |
| Axis coupling · pitch and yaw | 4 (lower) |
| Yaw · gyro vs accel | 6 |
| Gyro angles · drift | 7, 8 (lower) |
| Die temperature | 7, 8 (upper) |
| Zero-offset state | 2 |

Open the workspace, then open a capture in review mode. Figures 7 and 8 plot
the gyro angles as **deviation** from the attitude the integrator was started
at; that is a per-curve **offset** set in Configure, equal to minus the angle
at the sample where `gyro_zero.ready` goes high. The recorded channels are the
raw angles.

`elf_path` and the SWD probe serial were cleared before publishing. Neither is
needed to review a capture.

## Channels

Each capture carries the three calibrated accelerometer channels
(`accel_g.__struct.x/y/z`), the accelerometer attitude (`attitude_deg.roll`,
`attitude_deg.pitch`), the three calibrated gyro rates
(`gyro_dps.__struct.x/y/z`), the three integrated gyro angles
(`gyro_angles_deg.__struct.x/y/z`, which are roll, pitch and yaw), the
zero-offset estimator state (`gyro_zero.ready`, `gyro_inited`,
`gyro_init_request`), the die temperature and two firmware timing counters.

The sensor samples at 1 kHz. Samples reach the host over SWD at roughly 275 per
second in these captures, which is the polling rate of the acquisition rather
than a property of the sensor — do not read the sample count of a capture as a
sensor rate.

Each variable is read in its own SWD transaction, so two channels recorded in
the same row can come from firmware iterations a millisecond apart. During the
fast hand-held sweeps the accelerometer traces also carry isolated
single-sample excursions. **Compare those captures on the median, not the
peak.**

## Related article

**#008 Can a Gyroscope Be Used as an Attitude Sensor?**

The article describes the experimental setup, measurements, analysis and
results obtained from these datasets.
