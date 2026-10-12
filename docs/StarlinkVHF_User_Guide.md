# Starlink VHF Decoder User Guide

`lfdemod starlink-vhf` processes one known RF frequency within one known raw-file
time window using the following chain:

```text
real raw voltage
  -> Starlink VHF channel DDC assembled from the shared LFdemod DDC primitives
  -> 240 kS/s complex IQ
  -> approximately 83.333 kS/s (two samples per chip)
  -> LoRa preamble detection, synchronisation, demodulation, decoding, and CRC check
  -> conservative Starlink VHF profile selection based on payload length and content
```

It does not search an entire observation, infer a 100 s cadence, propagate a
TLE, or associate a frame automatically with a named spacecraft.

The raw data used to validate the 81-byte profile are described in
[`raw_data/StarlinkVHF_TTC_Data.md`](../raw_data/StarlinkVHF_TTC_Data.md).

## Installation

The LoRa backend is an optional dependency. FM, airband AM, and ORBCOMM do not
require it:

```bash
cd /path/to/LFdemod
python -m pip install -e '.[starlink-vhf]'
```

## Typical command

```bash
lfdemod starlink-vhf \
  -i /path/to/20250415-2200-0.dat \
  -f 137.055e6 \
  -s 5.15 \
  -d 1.4 \
  -o /path/to/results/event_01 \
  -iq
```

Display all long options, short aliases, and explanations with:

```bash
lfdemod starlink-vhf --help
```

The core inputs are:

- `-i/--input`: real raw-voltage file; the default dtype is little-endian
  signed 16-bit integer;
- `-f/--rf-frequency`: known absolute RF frequency to shift to zero frequency,
  in hertz;
- `-s/--start`: start time relative to the beginning of the input file, in
  seconds;
- `-d/--duration`: duration of the decode window, in seconds.

The DDC options describe the input, intermediate, and output sample rates;
the two filter stages; ripple and attenuation requirements; and the processing
block size. The LoRa options describe bandwidth, spreading factor, preamble
length, sync word, header mode, expected and accepted payload lengths, coding
rate, CRC, synchronisation tolerance, and forced LDRO. The defaults correspond
to the currently validated 81-byte Starlink VHF profile.

The public configuration accepts the following payload lengths reported in
public observations:

```text
73, 81, 87, 104, and 227 bytes
```

Only the 81-byte profile has been validated with two real 21CMA events and
interpreted at field level in this project. The other lengths can pass through
LoRa PHY decoding, header and CRC checks, and binary and structured output,
but the decoder does not apply 81-byte field positions or invent field
semantics without a real validation sample. The default remains 81 bytes and
can be changed with `-eb/--expected-payload-bytes` and
`-ab/--accepted-payload-bytes`.

## Validated default PHY parameters

- `SF=8` and `BW=125000/3 Hz`.
- Fifteen ordinary preamble upchirps. The measured interval from the preamble
  start to the PHY-data start is 19.25 symbols, consistent with
  `15 + 2 sync + 2.25 SFD`.
- Conventional LoRa sync byte `0x12`. This is one item of profile evidence and
  does not by itself establish Starlink identity.
- Explicit PHY header. Both real packets report an 81-byte payload, coding
  rate 4/5, and payload CRC enabled.
- Forced `LDRO=True`.

Forced `LDRO=True` is an empirically validated profile parameter rather than
the result of the generic automatic LoRa symbol-duration rule. With SF8 and
BW approximately 41.667 kHz, the symbol duration is 6.144 ms and a generic
automatic rule would normally disable LDRO. However, both independent 21CMA
events produce 148 PHY-data symbols and a valid payload CRC only when LDRO is
enabled. With LDRO disabled, both produce only 113 symbols, different payload
contents, and failed CRC checks. The option remains configurable so that
future packet profiles can be tested independently.

The following lengths must be distinguished carefully:

```text
LoRa explicit PHY header: parsed by the receiver as metadata and absent from the output bytes
81-byte payload: Starlink-profile protocol data
2-byte payload CRC: follows the payload
83-byte decoded payload-and-CRC: 81 + 2
```

The legacy compatibility field `frame_bytes` is therefore 83 bytes for the
current events, but it does not include the preamble, sync symbols, SFD, or
explicit PHY header. The newer outputs separately report
`header_payload_length`, `coding_rate`, `crc_enabled`, `payload_binary`, and
`decoded_payload_and_crc_binary`.

## Output products

- `summary.json`: configuration, packet count, PHY header, CRC, CFO, nominal
  sample time, parsed protocol fields, diagnostic color scale, and paths to
  all saved products;
- `frames.csv`: one structured summary row per packet, including header length,
  coding rate, CRC status, and payload-length profile, suitable for aggregation
  across files;
- `packet_NNN_payload.bin`: payload without the LoRa CRC;
- `packet_NNN_frame.bin`: complete decoded payload followed by the two CRC
  bytes;
- `diagnostic.png`: padded channel spectrogram, IQ envelope, selected-packet
  symbols, and dechirp concentration; orange dashed lines mark the requested
  user window;
- `channel_iq.c64`: complex IQ, written only when `-iq/--save-iq` is supplied.

`starlink_profile_match=true` means that the PHY header, coding rate, CRC,
known payload length, sync word, and available content checks for that length
are mutually consistent. The 81-byte profile also requires its currently
validated fixed-byte characteristics. Other known lengths receive only formal
PHY- and length-level support until real frames are available. This result
supports the interpretation of a Starlink-profile VHF LoRa frame, but neither
one individual criterion nor the Boolean value alone identifies a particular
NORAD object.

Time fields are `nominal sample time` values calculated from the file start
and nominal sample rate. Sub-second alignment with optical observations or
orbit predictions additionally requires an independent assessment of the file
UTC origin, sampling-clock error, and hardware delay.
