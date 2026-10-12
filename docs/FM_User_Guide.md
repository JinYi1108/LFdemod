# Broadcast-FM User Guide

This document describes the broadcast-FM post-processing interface, command-line
parameters, automatic output naming, and saved products. The project name is
written as `LFdemod`; the command-line program and public Python package both
use the lowercase name `lfdemod`.

For a concise list of current options, run:

```bash
lfdemod fm --help
```

## Current scope

The current processing chain is:

```text
480 MS/s real raw voltage
-> digital downconversion (DDC) to the requested RF frequency
-> filtering and decimation to 240 kS/s complex IQ
-> adjacent-sample phase-difference FM discrimination
-> FM composite baseband (MPX)
-> MPX power spectrum and diagnostic figure
```

This path is intended to test whether a candidate contains the composite
baseband structure of broadcast FM. It does not yet recover left and right
audio channels, write an audio file, or automatically classify a window as FM.

## Recommended interface

After an editable installation:

```bash
python -m pip install -e .
```

Inspect the root command and the FM subcommand:

```bash
lfdemod --help
lfdemod fm --help
```

Process one RF frequency and file window:

```bash
lfdemod fm \
  --input /data/20250415-1940-0.dat \
  --rf-frequency 98.33e6 \
  --start 30 \
  --duration 3
```

When no output directory is specified, the results are written to:

```text
results/20250415-1940-0/98.33MHz/30_3/
```

## File-window parameters

Use either the time-in-seconds mode or the fractional-file mode, not both.

### Selection in seconds

| Option | Meaning | Example |
| --- | --- | --- |
| `--start` | Start time relative to the beginning of the raw file, in seconds | `30` |
| `--duration` | Amount of data to process, in seconds | `3` |

```bash
--start 30 --duration 3
```

This selects 30--33 s in the raw file.

### Selection by file fraction

| Option | Meaning | Example |
| --- | --- | --- |
| `--start-fraction` | Start fraction of the file, in the range 0--1 | `0.25` |
| `--stop-fraction` | Stop fraction of the file, in the range 0--1 | `0.30` |

```bash
--start-fraction 0.25 --stop-fraction 0.30
```

This selects 25%--30% of the total file duration. The automatic output path
still uses the resolved start time and duration in seconds. The requested
fractions are retained in `run_config.json` and `summary.json`.

## Common signal and data parameters

| Option | Default | Meaning |
| --- | ---: | --- |
| `--rf-frequency` | required | Absolute RF frequency in hertz, for example `98.3e6` |
| `--dtype` | `<i2` | Little-endian signed 16-bit raw samples |
| `--sample-rate` | `480e6` | Sampling rate of the real raw-voltage stream |
| `--intermediate-rate` | `2.4e6` | Sampling rate after the first decimation stage |
| `--channel-rate` | `240e3` | Final complex-IQ sampling rate |
| `--passband` | `90e3` | One-sided usable channel passband |
| `--stopband` | `120e3` | One-sided start of the final filter stopband |
| `--stopband-attenuation` | `60` | Requested stopband attenuation in decibels |
| `--padding` | `0.020` | Extra filtering context on each side of the requested window, in seconds |
| `--chunk-samples` | `10000000` | Number of high-rate input samples processed per block |
| `--label` | `direct_fm_test` | User label stored in the figure and summary |

`--chunk-samples` controls memory use and throughput. It does not change the
duration selected by the user.

## Time-domain panel parameters

| Option | Default | Meaning |
| --- | ---: | --- |
| `--waveform-start` | unset | Start of the third panel within the selected analysis window; unset means centred |
| `--waveform-duration` | `0.050` | Duration shown in the third panel, in seconds |

For example, the following command analyzes 30--33 s of the raw file but shows
only a 20 ms interval beginning 1.2 s into that selected window:

```bash
lfdemod fm \
  --input /data/20250415-1940-0.dat \
  --rf-frequency 98.3e6 \
  --start 30 \
  --duration 3 \
  --waveform-start 1.2 \
  --waveform-duration 0.02
```

The third panel therefore corresponds to 31.2--31.22 s in the original file.
The first two spectral panels still use the full 30--33 s interval.

Do not confuse the following three durations:

- `duration`: how much raw-file data is analyzed;
- `compute_mpx_psd(segment_seconds=0.050)`: the segment length used by the
  Welch MPX spectrum estimator;
- `waveform-duration`: how much time is displayed in the third panel.

## Automatic output paths and overwrite protection

Without `--output-dir`, the output path is constructed as:

```text
output root/raw-file stem/frequency in MHz/start seconds_duration seconds/
```

For example:

```text
results/20250415-1940-0/98.3MHz/30_3/
results/20250415-1940-0/98.33MHz/30_3/
results/20250415-1940-0/98.33MHz/30.5_0.05/
```

Numbers retain at most six decimal places and unnecessary trailing zeroes are
removed. Thus `98.30` becomes `98.3MHz`, whereas `98.33` remains `98.33MHz`.

If the base directory already exists, it is not overwritten by default.
Successive runs create:

```text
30_3/
30_3_run02/
30_3_run03/
```

### Change the automatic output root

```bash
--output-root /path/to/FM_results
```

Only the outermost root changes; the raw-file, frequency, and time hierarchy
is still generated automatically.

### Specify an output directory directly

```bash
--output-dir /path/to/my_test
```

The automatically generated hierarchy is then disabled and the supplied path
is used as the base directory. If it already exists and overwriting is not
enabled, `my_test_run02` is created.

### Explicitly allow overwriting

```bash
--output-dir /path/to/my_test --overwrite
```

Existing result files in the base directory are replaced only when
`--overwrite` is supplied.

## Output products

Each result directory contains:

| File | Contents |
| --- | --- |
| `fm_psd.png` | Three-panel IQ spectrum, MPX spectrum, and local instantaneous-frequency diagnostic |
| `fm_arrays.npz` | IQ, MPX, and MPX power-spectrum arrays for the full analysis window |
| `summary.json` | Resolved output directory, resolved window, and basic measurements |
| `run_config.json` | Complete requested FM and DDC configuration |

Load the saved arrays with:

```python
import numpy as np

data = np.load("results/.../fm_arrays.npz")
iq = data["iq"]
mpx_hz = data["mpx_hz"]
frequencies = data["mpx_frequencies_hz"]
psd = data["mpx_psd"]
fs = float(data["sample_rate_hz"])
```

## Responsibilities of the FM implementation files

| File | Primary responsibility | Normally called directly? |
| --- | --- | --- |
| `fm/fm_ddc.py` | Convert real raw voltage to complex IQ for one RF channel | For stand-alone DDC experiments |
| `fm/demod.py` | IQ discrimination, MPX spectrum, and 19 kHz pilot-candidate measurement | For intermediate-result studies |
| `fm/pipeline.py` | File input, time selection, processing orchestration, and output writing | Recommended complete entry point |
| `fm/plotting.py` | Generate the standard three-panel diagnostic figure | Normally called by the pipeline |

Programmatic use of the complete pipeline:

```python
from lfdemod.fm import FMDDCConfig, FMPSDConfig, analyze_fm_psd_file

ddc = FMDDCConfig()
config = FMPSDConfig(
    rf_frequency_hz=98.3e6,
    start_seconds=30.0,
    duration_seconds=3.0,
    waveform_start_seconds=None,
    waveform_duration_seconds=0.050,
    ddc=ddc,
)

summary = analyze_fm_psd_file(
    "/data/20250415-1940-0.dat",
    None,
    config,
    output_root="results",
    overwrite=False,
)

print(summary["output_dir"])
```

Inspect the Python docstrings with:

```python
from lfdemod.fm import analyze_fm_psd_file, downconvert_fm_voltage

help(analyze_fm_psd_file)
help(downconvert_fm_voltage)
```

## Single-file educational implementation

`examples/fm_psd_one_file.py` presents the processing steps in execution order
within one file and is useful for reading or temporary debugging. For normal
processing of real observations, prefer `lfdemod fm`.
`examples/analyze_fm_psd.py` is a compatibility wrapper around the same CLI;
it does not duplicate the parser or pipeline implementation.
