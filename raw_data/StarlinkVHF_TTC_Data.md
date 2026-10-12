# Starlink VHF TT&C Validation Data

## Data availability status

> **Zenodo upload pending.** The two full raw-voltage files are not stored in
> this Git repository. Their Zenodo record URL, DOI, file checksums, and formal
> citation will be added here after the dataset has been uploaded and
> published.

Planned dataset link:

```text
TO BE ADDED AFTER ZENODO PUBLICATION
```

The files contain the two 137.055 MHz LoRa events used to validate the current
81-byte Starlink VHF telemetry, tracking, and command (TT&C) profile in
LFdemod.

## Expected files

| File | Role | Recommended decode window |
| --- | --- | --- |
| `20250415-2200-0.dat` | Event 1; higher signal-to-noise validation event | start 5.0 s, duration 1.5 s |
| `20250415-2200-12.dat` | Event 2; lower signal-to-noise validation event | start 2.6 s, duration 1.7 s |

Each complete file is expected to contain 4,096,000,000 real samples and to
have a size of 8,192,000,000 bytes (approximately 7.63 GiB).

## Raw format

| Property | Value |
| --- | --- |
| Instrument | One pod of the 21 Centimeter Array (21CMA) |
| Observation date | 2025-04-15 UTC |
| Sample representation | Real voltage |
| Data type | Little-endian signed 16-bit integer (`<i2`) |
| Nominal sample rate | 480,000,000 samples s⁻¹ |
| Nominal file duration | 8.533333333 s |
| First Nyquist interval | 0--240 MHz |
| Starlink VHF channel used here | 137.055 MHz |

The first Nyquist interval describes the digitized real-voltage stream. The
usable observing band also depends on the analog acquisition chain and should
not be inferred from the digital sampling rate alone.

The file may be opened without loading it entirely into memory:

```python
import numpy as np

raw = np.memmap(
    "20250415-2200-0.dat",
    dtype="<i2",
    mode="r",
)
print(raw.size)
```

## Reproducing the two decodes

Install the optional LoRa dependency:

```bash
python -m pip install -e '.[starlink-vhf]'
```

Decode Event 1:

```bash
lfdemod starlink-vhf \
  --input /path/to/20250415-2200-0.dat \
  --rf-frequency 137.055e6 \
  --start 5.0 \
  --duration 1.5 \
  --output-dir results/starlink_vhf_event_1 \
  --save-iq
```

Decode Event 2:

```bash
lfdemod starlink-vhf \
  --input /path/to/20250415-2200-12.dat \
  --rf-frequency 137.055e6 \
  --start 2.6 \
  --duration 1.7 \
  --output-dir results/starlink_vhf_event_2 \
  --save-iq
```

Each run writes `summary.json`, `frames.csv`, `diagnostic.png`, decoded binary
payload/frame files, and, because `--save-iq` is present, `channel_iq.c64`.
See the [Starlink VHF Decoder User Guide](../docs/StarlinkVHF_User_Guide.md)
for the PHY assumptions and detailed output definitions.

## Items to complete after publication

When the Zenodo dataset is available, update this file with:

1. the version-specific Zenodo record URL and DOI;
2. the concept DOI if the dataset will have multiple versions;
3. the exact MD5 or SHA-256 checksum of each DAT file;
4. the dataset creators, license, and preferred citation;
5. any final difference between the filenames above and the published names.

Do not replace these placeholders with the DOI of the archived ORBCOMM
software release or with the DOI of the earlier single-file 21CMA dataset;
the two Starlink validation files are intended to have their own dataset
record.
