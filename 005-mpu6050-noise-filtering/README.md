# Experiment #005 — MPU6050 Noise / Filtering

Raw datasets accompanying the EmbedStudio experiment:

**#005 MPU6050 Noise / Filtering Experiment**

## Objective

Characterize MPU6050 noise at maximum sensor bandwidth and investigate the
effect of hardware and software low-pass filtering.

## Acquisition

- Sensor: MPU6050
- Accelerometer range: ±2 g
- Gyroscope range: ±250 °/s
- Acquisition rate: 1 kSPS
- Acquisition tool: EmbedStudio

## Datasets

| Dataset | Configuration | Purpose |
|---|---|---|
| [260 Hz DLPF](data/capture_20260820_195646_260Hz.hdf5) | ~260 Hz DLPF | Maximum-bandwidth noise baseline |
| [44 Hz DLPF](data/capture_20260820_200144_44Hz.hdf5) | ~44 Hz DLPF | Hardware filtering |
| [20 Hz DLPF](data/capture_20260820_200320_20Hz.hdf5) | ~20 Hz DLPF | Hardware filtering |
| [Gyro X flipping](data/capture_20260823_112024_gyro_x_flipping.hdf5) | Dynamic test | Gyro X response |

HDF5 files are raw binary datasets and cannot be previewed directly by
GitHub. Download the dataset and open it with EmbedStudio or another
compatible HDF5 tool.

## Software filtering

The 22 Hz, 10 Hz and 5 Hz software-filtered results were generated from:

`capture_20260820_195646_260Hz.hdf5`

No separate raw acquisition was required for these configurations.

## Related article

**#005 MPU6050 Noise / Filtering Experiment**

The article describes the experimental setup, measurements, analysis and
results obtained from these datasets.
