# Experiment #006 — MPU6050 Estimating Attitude from the Gravity Vector

Raw datasets accompanying the EmbedStudio experiment:

**#006 MPU6050 Roll and Pitch: Estimating Attitude from the Gravity Vector**

## Objective

Estimate the orientation of the sensor from the accelerometer alone, by
treating the three measured channels as one gravity vector and converting it
into roll and pitch. Investigate how the chosen representation behaves during
large rotations, how stable it is, and what linear acceleration does to it.

## Acquisition

- Sensor: MPU6050
- Accelerometer range: ±2 g
- Gyroscope range: ±250 °/s
- Acquisition rate: 1 kSPS
- Acquisition tool: EmbedStudio

## Attitude convention

Roll and pitch are computed on-target by the firmware and recorded as the
`attitude_deg` channels:

```
roll  = atan2(a_y, a_z)
pitch = atan2(-a_x, sqrt(a_y^2 + a_z^2))
```

Roll spans −180°…+180°. Pitch is limited to −90°…+90°, because the second
argument is a square root and so is never negative.

## Datasets

| Dataset | Configuration | Purpose |
|---|---|---|
| [Stationary](data/capture_20260829_153127_stationary.hdf5) | At rest | Reference before the rotations |
| [Roll rotation](data/capture_20260829_153445_roll.hdf5) | Rotation about X | Roll wrap at ±180° |
| [Pitch rotation](data/capture_20260829_153509_pitch.hdf5) | Rotation about Y | Pitch limited to ±90°, roll taking over |
| [260 Hz DLPF](data/capture_20260829_160044_noise_DLPF_260Hz.hdf5) | 260 Hz DLPF, at rest | Attitude noise at maximum bandwidth |
| [22 Hz DLPF](data/capture_20260829_160158_noise_DLPF_22Hz.hdf5) | 22 Hz DLPF, at rest | Attitude noise with bandwidth narrowed |
| [Linear movement](data/capture_20260829_160441_linear_movement.hdf5) | Horizontal motion | Gravity and linear acceleration confused |

HDF5 files are raw binary datasets and cannot be previewed directly by
GitHub. Download the dataset and open it with EmbedStudio or another
compatible HDF5 tool.

## Channels

Each capture carries the three calibrated accelerometer channels
(`accel_g.__struct.x/y/z`), their magnitude (`accel_magnitude_g`), the
calculated attitude (`attitude_deg.roll`, `attitude_deg.pitch`), the die
temperature and three firmware timing counters.

The sensor samples at 1 kHz. Samples reach the host over SWD at roughly 420
per second, which is the polling rate of the acquisition rather than a
property of the sensor — do not read the sample count of a capture as a
sensor rate.

## Related article

**#006 MPU6050 Roll and Pitch: Estimating Attitude from the Gravity Vector**

The article describes the experimental setup, measurements, analysis and
results obtained from these datasets.
