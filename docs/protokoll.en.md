# The HESP protocol

**HESP** is the binary request/response protocol on the 19200 baud bus (X7). The name and
the basic telegram structure are described in an FHEM forum thread (topic 96437) on the
related Hermes / P series; the frame structure and the checksum documented here were
determined from our own bus recordings.

## Physical layer

- **19200 baud, 8N1**, RS485 half duplex, differential A/B pair on X7-4/X7-3.
- The master is the **STM32** of the PTC module; it polls the LPC1758 about every
  5 seconds and re-asserts the current panel state (stage / mode / setpoints) as it does.

## Frame format

Every frame:

```
<h0> <h1> <h2>   <id_lo> <id_hi>   00 00   <len>   <data …>   <crc_lo> <crc_hi>
```

| Field | Bytes | Meaning |
|---|---|---|
| `h0 h1 h2` | 3 | **Address header**, 24-bit little-endian: type, source and destination address (see below) |
| `id_lo id_hi` | 2 | **Data point ID, 16-bit little-endian** |
| `00 00` | 2 | Reserved / sub-selector |
| `len` | 1 | **Nibble count** of the data (`0x08`=4 B, `0x20`=16 B, `0xe0`=224 B) |
| `data` | n | Payload (float32 LE, int, array …) |
| `crc` | 2 | Checksum (little-endian) |

### The address header

The first three bytes presumably form a 24-bit word `H = h0 | h1 << 8 | h2 << 16` with
three fields:

| Bits | Field | Width |
|---|---|---|
| 0–3 | Type (see below) | 4 bit |
| 4–13 | Source address | 10 bit |
| 14–23 | Destination address | 10 bit |

This reading would explain the bytes previously described as a "fixed remainder of the
header" or as "node-dependent": request and response swap source and destination address.

| Header | Type | Source → destination |
|---|---|---|
| `10 80 00` | QUERY | 1 → 2 |
| `22 40 00` | RESPONSE | 2 → 1 |
| `11 80 00` | SET | 1 → 2 |
| `23 40 00` | CONFIRM | 2 → 1 |
| `10 80 07` | QUERY | 1 → 30 |
| `10 00 0a` | QUERY | 1 → 40 |

Address 1 is the polling master. The decomposition is consistent with recordings from a
second unit carrying further bus participants (addresses 10–15, 30 and 40 among others):
there, every response carries the swapped addresses of the preceding request. No
confirmation from manufacturer documentation is available.

### The type nibble

The low nibble of the first byte encodes the kind of telegram — exactly as documented in
the forum and confirmed on the wire:

| Nibble | Type | Header example |
|---|---|---|
| `0` | **QUERY** | `10 80 00 …` |
| `1` | **SET** (write) | `11 80 00 …` |
| `2` | **RESPONSE** | `22 40 00 …` |
| `3` | **CONFIRM** (write acknowledgement) | `23 40 00 …` |

### Examples

```text
QUERY    10 80 00 60 01 00 00 04 01 00 49 e1        # query DP 0x0160
RESPONSE 22 40 00 d2 00 00 00 20 …16 B… 5a cb        # response carrying 16 B of data
SET      11 80 00 27 02 00 00 08 00 00 98 41 8a cd  # set DP 0x0227 = 19.0 (float32)
CONFIRM  23 40 00 27 02 00 00 00 39 a1              # write acknowledged
```

Payloads are **IEEE-754 float32 little-endian** for temperatures and setpoints, otherwise
integers (RPM, counters, flags) or arrays (characteristic curves).

## Checksum

The last two bytes of every frame form a **CRC-16** over all preceding bytes (header
through payload inclusive), stored little-endian:

| Parameter | Value |
|---|---|
| Polynomial | `0x8048` in reflected form (LSB first; normal: `0x1201`) |
| Init value | `0xFFFF` |
| Final XOR | none |
| Shift steps per byte | **16** |

The peculiarity is the number of shift steps: after a byte has been mixed in, the
register is shifted 16 times instead of 8. This is equivalent to each data byte entering
an ordinary reflected CRC-16 as a 16-bit word with high byte `0x00`. That quirk explains
why a search over standard parameter sets (all polynomials, reflection variants, byte
orders) yields no match.

```python
def hesp_crc(data: bytes) -> int:
    crc = 0xFFFF
    for b in data:
        crc ^= b
        for _ in range(16):
            crc = (crc >> 1) ^ 0x8048 if crc & 1 else crc >> 1
    return crc          # in the frame: crc & 0xFF, then crc >> 8
```

Verified against every example on this page as well as against some 6000 frames from a
second unit (all four telegram types, frame lengths 10–134 bytes) without deviation. The
parameters were derived from recordings; no confirmation from manufacturer documentation
is available.

### Relation to the affine GF(2) model

An earlier version of this page described the checksum empirically as an affine map over
GF(2) with a table of contribution vectors `V[e]`. The linearity is a direct consequence
of the CRC. The table was not uniquely determined, however: bits that were always set
together, or held constant, in the training material cannot be separated from the
length-dependent constant. The constant is in fact **not zero** but the CRC of an
all-zero frame of the same length (for example `27d8` for 10 bytes, `edc9` for 12 bytes).
The table therefore produced correct results for the observed bit patterns, but not for
arbitrary frames. The contribution vectors follow from the algorithm as
`V[e] = crc(frame with only bit e set) ⊕ crc(all-zero frame)` and depend only on `e`, not
on the frame length.

??? info "Table of contribution vectors `V[e]` (from the algorithm, `e` = 0–95)"

    Bit position counted from the end of the frame: for byte `i` (0-based), bit `b`
    (0 = LSB) of a frame of `N` bytes excluding the CRC, `e = (N − 1 − i) · 8 + b`. Then
    `CRC = K(N) ⊕ (⊕ V[e] over all set bits)`, where `K(N)` is the CRC of an all-zero frame of length `N`.

    | `e` | `V[e]` | `e` | `V[e]` | `e` | `V[e]` | `e` | `V[e]` |
    |---|---|---|---|---|---|---|---|
    | 0 | `d1f4` | 1 | `a379` | 2 | `4663` | 3 | `8cc6` |
    | 4 | `191d` | 5 | `323a` | 6 | `6474` | 7 | `c8e8` |
    | 8 | `80fc` | 9 | `0169` | 10 | `02d2` | 11 | `05a4` |
    | 12 | `0b48` | 13 | `1690` | 14 | `2d20` | 15 | `5a40` |
    | 16 | `cd56` | 17 | `9a3d` | 18 | `34eb` | 19 | `69d6` |
    | 20 | `d3ac` | 21 | `a7c9` | 22 | `4f03` | 23 | `9e06` |
    | 24 | `0534` | 25 | `0a68` | 26 | `14d0` | 27 | `29a0` |
    | 28 | `5340` | 29 | `a680` | 30 | `4d91` | 31 | `9b22` |
    | 32 | `b823` | 33 | `70d7` | 34 | `e1ae` | 35 | `c3cd` |
    | 36 | `870b` | 37 | `0e87` | 38 | `1d0e` | 39 | `3a1c` |
    | 40 | `78a8` | 41 | `f150` | 42 | `e231` | 43 | `c4f3` |
    | 44 | `8977` | 45 | `127f` | 46 | `24fe` | 47 | `49fc` |
    | 48 | `8e67` | 49 | `1c5f` | 50 | `38be` | 51 | `717c` |
    | 52 | `e2f8` | 53 | `c561` | 54 | `8a53` | 55 | `1437` |
    | 56 | `0c91` | 57 | `1922` | 58 | `3244` | 59 | `6488` |
    | 60 | `c910` | 61 | `92b1` | 62 | `25f3` | 63 | `4be6` |
    | 64 | `cc6b` | 65 | `9847` | 66 | `301f` | 67 | `603e` |
    | 68 | `c07c` | 69 | `8069` | 70 | `0043` | 71 | `0086` |
    | 72 | `a403` | 73 | `4897` | 74 | `912e` | 75 | `22cd` |
    | 76 | `459a` | 77 | `8b34` | 78 | `16f9` | 79 | `2df2` |
    | 80 | `96f1` | 81 | `2d73` | 82 | `5ae6` | 83 | `b5cc` |
    | 84 | `6b09` | 85 | `d612` | 86 | `acb5` | 87 | `59fb` |
    | 88 | `a03b` | 89 | `40e7` | 90 | `81ce` | 91 | `030d` |
    | 92 | `061a` | 93 | `0c34` | 94 | `1868` | 95 | `30d0` |

    | `N` | 8 | 9 | 10 | 11 | 12 | 13 | 14 |
    |---|---|---|---|---|---|---|---|
    | `K(N)` | `81dd` | `bfd9` | `27d8` | `ee21` | `edc9` | `4cfb` | `f5be` |

### Worked example

Query frame for data point `0x0160` (see the examples above):

```python
>>> hex(hesp_crc(bytes.fromhex("10 80 00 60 01 00 00 04 01 00")))
'0xe149'
```

→ `crc_lo = 49`, `crc_hi = e1`; the complete frame on the bus is therefore

```
10 80 00 60 01 00 00 04 01 00 49 e1
```

which matches the recording exactly.

## Meta data points (self-description)

The bus describes itself in part:

| DP | Content |
|---|---|
| `0x0001` | Plain-text name of the node (e.g. node 30 = "PTC module") |
| `0x0033` | List of all parameter DP numbers (directory) |
| `0x0034` | Parallel list of type codes for it |
| `0x0038` | **Access level mask** per DP (see below) |
| `0x0039` | Unit code per DP (8=°C, 5=seconds, 11=string …) |
| `0x0130` | Fault status as plain text ("no fault") |

!!! note "Node numbers"
    The node number is to be taken from the address header (see above), not from a single
    header byte: the PTC module's responses begin with `e2 41 00`, which under the 24-bit
    decomposition means *RESPONSE from node 30 to node 1*. An earlier version of this page
    read the middle byte as a node ID ("node `0x41`").

### Access mask `0x0038`

One byte per DP encodes the write permissions across four access levels (2 bits R/W each):

| Mask | Meaning | Applies to |
|---|---|---|
| `0x55` | read-only | all measured values, counters |
| `0xFF` | writable everywhere | the user values set from the panel (room setpoint etc.) |
| `0xD5` | writable at the top level only | the **installer parameter block** |
| `0xC0` | visible at the top level only | factory characteristic curves, fan arrays |

The mask independently confirms which DPs make up the (PIN-protected) installer menu —
useful to avoid writing blindly into protected registers when parameterising.
