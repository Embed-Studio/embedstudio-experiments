# Experiment #009 — What Does a Complementary Filter Actually Fix?

Raw datasets accompanying the EmbedStudio experiment:

**#009 What Does a Complementary Filter Actually Fix?**

## Objective

An accelerometer sees gravity, so its attitude estimate never walks away — but
it is noisy, it cannot tell gravity from any other acceleration, and its roll
estimate collapses near vertical. A gyroscope measures rotation, so its
estimate is smooth and handles fast motion — but it drifts, and it does not
know where it started.

A complementary filter blends the two in a fixed proportion. These captures
measure what that actually buys and what it does not.

**Six filters run on the target at once**, over the same samples, with time
constants of 10, 50, 100, 500, 1000 and 5000 ms. Every comparison in the
article is therefore between estimates of the *same* motion rather than between
two recordings.

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
`gyro_angles_deg` channels and kept as the reference the filters are compared
against:

```
theta[k+1] = theta[k] + omega[k] * dt
```

Each complementary filter blends the two, recorded as the
`complementary_filer_attitude_deg` channels:

```
roll [k] = alpha * (roll [k-1] + omega_x[k] * dt) + (1 - alpha) * accel_roll
pitch[k] = alpha * (pitch[k-1] + omega_y[k] * dt) + (1 - alpha) * accel_pitch
yaw  [k] = yaw[k-1] + omega_z[k] * dt
```

`dt` is the interval measured per sample (experiment #004), not a nominal one.

**Yaw is not filtered.** The accelerometer carries no heading information, so
the yaw channel of every filter is the same pure integration as
`gyro_angles_deg.z`. Nothing in this dataset characterises that axis.

### The filter constants

`alpha` is computed once at startup as `tau / (tau + LOOP_TIME_MS)` with
`LOOP_TIME_MS` equal to 1:

| τ | α |
|---|---|
| 10 ms | 0.909091 |
| 50 ms | 0.980392 |
| 100 ms | 0.990099 |
| 500 ms | 0.998004 |
| 1000 ms | 0.999001 |
| 5000 ms | 0.999800 |

Note that the *nominal* interval is used for `alpha` while the *measured*
interval is used for `dt`. The real interval is 1.0009 ms, so every effective
time constant is 0.09 % longer than its label. The `tau_ms` channels carry the
nominal values.

⚠️ **The five-minute stationary capture carries four filters, not six.** It was
recorded with an earlier build; τ = 1000 ms and τ = 5000 ms exist only in the
other three captures. **Do not read a missing channel as a filter that failed.**

### Initialization

The gyro integrator and all filter accumulators are seeded **together, from the
same value**, once `gyro_zero.ready` is 1 and 100 accelerometer attitude
samples have been averaged. Roll and pitch come from that average and yaw from
zero. `cnt_accel_samples` counts the averaging window, `accu_accel_attitude_deg`
carries the running sum, and `gyro_inited` goes to 1 when the seed is applied.
`gyro_init_request` is cleared by the firmware faster than the host polls, so it
reads 0 in every capture.

## Datasets

| Dataset | Movement | Purpose |
|---|---|---|
| [Stationary, 5 min](data/capture_20260912_172932_stationary.hdf5) | None | Noise and drift, with the board set down and left alone |
| [Dynamic response](data/capture_20260912_174843_dynamic_response.hdf5) | Hand-held, all three axes | Time-constant comparison, and the error the integration accumulates |
| [Linear acceleration](data/capture_20260912_175250_linear_acceleration.hdf5) | Translated, not rotated | How much of an acceleration disturbance each filter passes through |
| [Roll instability](data/capture_20260912_174243_roll_instability.hdf5) | Pitched past vertical | The ±90° limitation, which the filter does not remove |

### What the numbers are

In the **stationary** capture, over a 297 s window from 5.0 s (64 650 samples),
the standard deviation of the roll estimate was **0.056°** for the
accelerometer, **0.173°** for gyro integration alone and **0.038 / 0.021 /
0.016 / 0.009°** for τ = 10 / 50 / 100 / 500 ms. Pitch gives 0.057, 0.979 and
0.039 / 0.022 / 0.016 / 0.008.

Over the same window the gyro-only estimate moved **+0.21° in roll and
+3.59° in pitch**, while every filter ended within 0.06° and 0.02° of
where it started. ⚠️ The gyro-only standard deviation is the largest of the
three and **is not noise** — it is the slow wander of a curve that is
individually smooth. Read the shape, not the number.

In the **dynamic response** capture the board is turned about all three axes by
hand for fourteen seconds and put back down. When it came to rest the gyro-only
estimate was **23.4° wrong in roll and 7.7° wrong in pitch**, while τ = 10, 50
and 100 ms were within 0.01°, τ = 500 ms within 0.03° and τ = 1000 ms within
0.5°.

⚠️ **That 23.4° is not drift.** Fourteen seconds at the residual bias measured
in the stationary capture is 0.2°. It is the axis-coupling error of experiment
#008 — three Euler angles integrated independently, which is only valid near
level.

The same capture carries a two-sample accelerometer glitch at 11.408 and
11.413 s, reading **−134.90°** and **+44.95°** of roll while the board was
within a few degrees of level. Measured as each filter's excursion from its own
value one sample before the glitch, τ = 10 ms moved **27.1°**, τ = 50 ms 7.8°,
τ = 100 ms 3.9°, τ = 500 ms 0.6°, τ = 1000 ms 0.2° and τ = 5000 ms 0.4°. τ = 10 ms
was back within 15 ms of the glitch and τ = 100 ms within 56 ms. ⚠️ **Measure this
against the sample BEFORE the glitch, not against a window median** — the board
is still moving here, so a window wide enough to hold a median also holds real
rotation.

In the **linear acceleration** capture the board was moved without being turned,
and the gyro says how well that worked: the integrated roll moved **2.40°**
peak-to-peak and the integrated pitch **3.13°** over the whole recording. Over
6.0–11.7 s the accelerometer roll reached **21.2° from its own median**, against
14.3° for τ = 10 ms, 10.3° for τ = 50 ms, 7.7° for τ = 100 ms, 3.0° for
τ = 500 ms and 1.8° for τ = 1000 ms.

⚠️ **The τ = 5000 ms roll figures in this capture are not a measurement of
disturbance rejection.** That filter enters the recording carrying a 12.6°
error and is still converging throughout. Its pitch figures are clean.

In the **roll instability** capture the board is pitched forward past vertical.
The accelerometer pitch reaches −89.8° and folds back; the integrated pitch
passes through to −99.4°. While the pitch was beyond 80° the accelerometer roll
swung between **+154° and −178°** while the integrated roll stayed inside
**±3.6°**. Every filter followed it down — to −166.0°, −159.2°, −155.7°,
−130.0°, −98.1° and −29.6° for the six time constants — and took 0.30, 0.43,
0.44, 1.58 and 2.84 s to recover, with τ = 5000 ms still 10.6° out when the
capture ended 3.68 s later.

⚠️ **This is the pitch→±90° singularity of experiment #007, not a roll
singularity.** Roll is `atan2(a_y, a_z)`, and as pitch approaches vertical both
of those components shrink into the noise. **Rolling through 90° is not
singular and nothing in this dataset should be read as saying it is.**

⚠️ **A longer time constant reports a smaller excursion but is not more
correct** — it is slower, and it stays wrong after the board has returned to a
normal attitude.

The same capture contains an **angle wrap**: consecutive samples read +154.06°
and −162.95°, which are 43° apart as orientations and 317° apart as numbers.
The filter blends the numbers. The firmware here does no wrap handling,
deliberately.

HDF5 files are raw binary datasets and cannot be previewed directly by GitHub.
Download the dataset and open it with EmbedStudio or another compatible HDF5
tool.

## Workspace

[`MPU-6050_009.esws`](MPU-6050_009.esws) is the EmbedStudio workspace these
captures were recorded with. It carries an extra **Article figures** dashboard
whose views are the ones the screenshots in the article were taken from:

| View | Article figure |
|---|---|
| Roll · three estimates | 2, 4 (upper) |
| Pitch · three estimates | 3 |
| Roll · four time constants | 4 (lower) |
| Roll · linear acceleration | 6 |
| Roll · the accelerometer glitch | 7 |
| Pitch · past vertical | 8 (upper) |
| Roll · at high pitch | 8 (lower) |
| Die temperature | — |
| Zero-offset state | — |

Open the workspace, then open a capture in review mode. The glitch view draws
circle markers, because what it shows is two individual samples. Figure 6 draws
the gyro-only roll with a per-curve **offset** of −20.46°, the measured median gap
between the two estimates, so its shape can be compared on the accelerometer's
scale; the recorded channel is the raw angle. Opening the stationary capture
leaves the τ = 1000 ms and τ = 5000 ms curves empty, because that capture does
not carry them.

`elf_path` and the SWD probe serial were cleared before publishing. Neither is
needed to review a capture.

## Channels

Each capture carries the accelerometer attitude (`attitude_deg.roll`,
`attitude_deg.pitch`), the three integrated gyro angles
(`gyro_angles_deg.__struct.x/y/z`, which are roll, pitch and yaw), the roll,
pitch and yaw output of every complementary filter
(`complementary_filer_attitude_deg[i]`), the filter constants themselves
(`tau_ms[i]`, `alpha[i]`), the seeding state (`accu_accel_attitude_deg.*`,
`cnt_accel_samples`, `gyro_inited`, `gyro_init_request`, `gyro_zero.ready`), the
die temperature and two firmware timing counters.

The firmware's own spelling of the filter identifier is kept, typo included, so
the channel names here match the channel names in the captures. Filters 0–3
expose their three values as struct members `.x .y .z` and filters 4–5 as array
elements `.v[0] .v[1] .v[2]`; the difference is how the symbol was resolved, not
what it holds. **Index order is τ = 10, 50, 100, 500, 1000, 5000 ms — read
`tau_ms[i]` rather than assuming it.**

`timings.signal_processing_us` is the whole on-target processing chain including
every filter: 45 µs with six filters and 44 µs with four, on a 168 MHz
Cortex-M4. `timings.data_exchange_time_us` is the I²C read of the sensor, at
414 µs, for scale.

The sensor samples at 1 kHz. Samples reach the host over SWD at roughly 200 per
second in these captures and 218 in the stationary one, which is the polling
rate of the acquisition rather than a property of the sensor — do not read the
sample count of a capture as a sensor rate. The difference between the two is
the number of watched channels.

Each variable is read in its own SWD transaction, so two channels recorded in
the same row can come from firmware iterations a millisecond apart. During the
fast hand-held sweeps the accelerometer traces also carry isolated single-sample
excursions. **Compare those captures on the median, not the peak.**

⚠️ Three of the four captures start with one or more filters carrying an
accumulator value from the previous recording. That transient is real filter
behaviour, and it is what the correction time constants in the article were
fitted from, but **it is not a steady-state result.**

⚠️ The die temperature stays within 0.6 °C in all four captures. **Nothing here
characterises the temperature dependence of anything.**

## Related article

**#009 What Does a Complementary Filter Actually Fix?**

The article describes the experimental setup, measurements, analysis and
results obtained from these datasets.
