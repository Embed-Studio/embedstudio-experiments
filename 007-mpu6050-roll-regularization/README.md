# Experiment #007 — When Accelerometer Roll Estimation Fails

Raw datasets accompanying the EmbedStudio experiment:

**#007 When Accelerometer Roll Estimation Fails: The Pitch Singularity**

## Objective

Roll estimated from the accelerometer alone breaks down as the board
approaches vertical: gravity swings onto the X axis, the YZ projection the
roll formula depends on shrinks into the noise floor, and the estimate starts
swinging through hundreds of degrees while the board is not rolling at all.

These captures record that failure, and record five roll estimates computed
side by side on-target — one unregularized and four with a regularization
coefficient μ added to the denominator — so the stabilization and the
distortion it costs can be measured on the same samples.

## Acquisition

- Sensor: MPU6050
- Accelerometer range: ±2 g
- Gyroscope range: ±250 °/s
- DLPF bandwidth: 21 Hz
- Acquisition rate: 1 kSPS
- Acquisition tool: EmbedStudio

## Attitude convention

Roll and pitch are computed on-target by the firmware and recorded as the
`attitude_deg` channels:

```
roll  = atan2(a_y, a_z)
pitch = atan2(-a_x, sqrt(a_y^2 + a_z^2))
```

The regularized roll adds μ times the X component under the square root, and
carries the sign of `a_z` so the estimate can still report a board past
vertical:

```
roll_reg = atan2(a_y, copysign(1, a_z) * sqrt(a_z^2 + mu * a_x^2))
```

Five coefficients are evaluated on every firmware iteration and recorded as
`attitude_reg_deg[i]`. **The subscript is an index into the firmware's μ
table, not a value, and the order is not monotone:**

| Channel | μ | |
|---|---|---|
| `attitude_reg_deg[0].roll` | 0 | unregularized, identical formula to `attitude_deg.roll` |
| `attitude_reg_deg[1].roll` | 0.1 | strongest |
| `attitude_reg_deg[2].roll` | 0.01 | |
| `attitude_reg_deg[3].roll` | 0.001 | |
| `attitude_reg_deg[4].roll` | 0.0001 | weakest |

Regularization changes the roll formula only, so only `attitude_reg_deg[0]`
carries a pitch channel.

## Datasets

| Dataset | Movement | Purpose |
|---|---|---|
| [Roll singularity](data/capture_20260901_205235_roll_singularity.hdf5) | Rotated to vertical and back, pitch to −89.4° | The failure itself, and what each μ does about it |
| [Arbitrary inclinations](data/capture_20260901_205326_arbitrary_inclinations.hdf5) | Hand-held, roll and pitch within ±45° | What regularization costs at everyday angles |

In the first capture `|pitch|` exceeds 85° between 13.8 s and 21.9 s. From
16.30 s onwards `a_z` changes sign: past that point the estimates diverge
because the board really has passed vertical, not because of noise.

In the second capture the four regularized estimates track the unregularized
one to a median of 0.0001°, 0.0008°, 0.0052° and 0.0298° for μ = 0.0001,
0.001, 0.01 and 0.1 respectively. **Compare these channels on the median, not
the peak** — see the note on sampling skew below.

HDF5 files are raw binary datasets and cannot be previewed directly by
GitHub. Download the dataset and open it with EmbedStudio or another
compatible HDF5 tool.

## Channels

Each capture carries the three calibrated accelerometer channels
(`accel_g.__struct.x/y/z`), their magnitude (`accel_magnitude_g`), the
calculated attitude (`attitude_deg.roll`, `attitude_deg.pitch`), the five
regularized roll estimates, the die temperature and three firmware timing
counters.

The sensor samples at 1 kHz. Samples reach the host over SWD at roughly 280
per second in these captures, which is the polling rate of the acquisition
rather than a property of the sensor — do not read the sample count of a
capture as a sensor rate.

Each variable is read in its own SWD transaction, so two channels recorded in
the same row can come from firmware iterations a millisecond apart. During
fast movement that skew is worth a few degrees: `attitude_deg.roll` and
`attitude_reg_deg[0].roll` are the same quantity computed twice on-target and
still differ by a median of about 0.06°, and the peak difference between μ =
0.0001 and μ = 0 in the second capture is 2.7° — the same 2.7° as for μ =
0.01, which is how it is identified as skew rather than regularization error.

## Related article

**#007 When Accelerometer Roll Estimation Fails: The Pitch Singularity**

The article describes the experimental setup, measurements, analysis and
results obtained from these datasets.
