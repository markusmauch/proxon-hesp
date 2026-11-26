# Data point reference

All IDs are 16-bit. The high byte groups them roughly by function. Values in brackets are
example readings from a running unit. **This assignment applies to a P 2H-L** and may
differ on other configurations.

## Control data points (writable)

Identified unambiguously by deliberate operation at the panel — changing the respective
quantity at the panel changes exactly this data point:

| DP | Type | Meaning | Values |
|---|---|---|---|
| `0x00e1` | int16 | **Fan stage** | 0=off, 1, 2, 3, 4 |
| `0x020a` | int16 | **Operating mode** | 0=off, 1=Eco summer, 2=Eco winter, 3=Comfort, 4=Stove |
| `0x0227` | float32 | **Room setpoint** | 15–25 °C (panel rotary control) |
| `0x0226` | float32 | Actual room temperature (panel sensor, can be fed in) | — |

Write frames:

```text
Fan stage:      11 80 00 e1 00 00 00 04 <NN> 00 <crc16>   # NN = 01/02/03/04
Operating mode: 11 80 00 0a 02 00 00 04 <NN> 00 <crc16>
Room setpoint:  11 80 00 27 02 00 00 08 <float32 LE> <crc16>
```

!!! note "Operating mode encoding"
    The encoding was determined directly on the bus: **off=0, Eco summer=1, Eco winter=2,
    Comfort=3, Stove=4**. It differs from the assignment sometimes passed around, which
    is derived from the related WR3223 range.

## Temperature sensors

Ten physical channels. The names correspond to the **T numbers** from the panel's
"Current readings" menu and were assigned by comparing simultaneous bus values against
the panel display.

| DP | Sensor |
|---|---|
| `0x0194` | T1 supply air |
| `0x0231` | T3 fresh air |
| `0x0230` | T4 exhaust air |
| `0x0190` | T5 before evaporator |
| `0x0191` | T6 evaporator |
| `0x022e` | T7 extract air |
| `0x0192` | T8 after preheater |
| `0x022d` | T10 condenser |
| `0x0193` | T12 before condenser |
| `0x0195` | T13 compressor |

!!! note "Separating the supply-air pair `0x0194`/`0x022d`"
    Both data points sit in the supply air path and read identically while the heat pump
    is off (constant idle offset of about 0.6 K). The assignment was determined during a
    controlled heat pump run (which can be triggered in summer too: Comfort mode,
    setpoint 5–6 K above room temperature; the compressor started in under two minutes
    here). On compressor start `0x022d` rises earlier and far more steeply than `0x0194`
    (the sensor at the heat source leads), and after shutdown it holds residual heat
    longer once corrected for the idle offset. Both observations identify `0x022d` as
    T10 condenser and `0x0194` accordingly as T1 supply air. Reference values from the
    run: T13 compressor 21 → 74 °C, T4 exhaust air 22 → 6 °C, T6 evaporator 24 → 7 °C,
    supply-air pair 21 → 49 °C.

## Fans & speeds

| DP | Meaning |
|---|---|
| `0x00c9` | Actual speed, 2× float32 `[supply, extract]` |
| `0x00d7` | Target speed, 2× float32 `[supply, extract]` |

## Status data points

| DP | Meaning |
|---|---|
| `0x0160` | **Summer bypass status**: 0 = closed (heat recovery active), 1 = bypass active |
| `0x00ed` | **Filter remaining runtime** in days (u32) |
| `0x0130` | Fault text (plain text) |
| `0x006c` | **Status word** (u32); bit 7 is set during defrosting |

!!! note "Determining the bypass status"
    In an overnight long-term measurement `0x0160` was the only data point to change from
    0 to 1 at exactly the moment when the fresh air temperature fell below the room
    temperature in the evening. From that moment the supply air temperature measurably
    detached from the extract air temperature and approached the fresh air — the damper
    movement is thereby evidenced against an independent quantity. The intermediate
    position of the supply air (between full heat recovery and full bypass) fits a
    **controlled** summer bypass. At the same moment `0x01f6` changed from `0x40` to
    `0x20` (status word candidate, meaning open).

!!! note "Defrost bit in `0x006c`"
    In normal operation `0x006c` read a constant 39 on the unit examined (bits 0, 1, 2 and
    5), and 167 during defrosting (bit 7 in addition). In three consecutive defrost cycles
    bit 7 was set about 66 s before the fans stopped and cleared exactly when they
    restarted. On the P 2H-L both fans stand still during defrosting (about 6 min, every
    38 min or so with the outdoor air around 0 °C); on an FWT 2-R, according to Martin
    Kneppe, they keep running at 20 % supply and 60 % extract. The pointer to the bit came
    from Martin Kneppe; whether it is also set briefly when the heat pump starts has not yet
    been observed on the P 2H-L.

!!! note "Filter remaining runtime"
    At the time of measurement `0x00ed` matched the panel's filter remaining runtime
    display exactly and was the only data point carrying that value in the entire address
    space. The daily decrement is confirmed; the only open question is the moment at
    which it happens (not midnight).

## Counters and further data points read

| DP | Meaning |
|---|---|
| `0x02df` | **Operating hour counter array** (u32 list, 9 slots in use, incremented hourly); individual values also readable as `0x02d0`–`0x02d9` |
| `0x0066` | Array of 12 flags (u32) — presumably switched outputs; assignment open, the bypass status does **not** report here but in `0x0160` |
| `0x0064` | Raw analogue values of the sensor channels (float32, mV; open channels sit at the 3.3 V reference) |
| `0x03e8` | Status / log text of the controller (e.g. "Writing parameters to NVRAM backup") |
| `0x032e` | **Date and time** of the controller, packed (see below) |
| `0x00d2` / `0x00d3` | Fan stage → percentage characteristic curves |
| `0x0233`–`0x023e` | Parameter block (winter / heat pump), writable only at the top access level |
| `0x0229` / `0x022a` | written cyclically by the panel; on the unit examined **constant 21.0** (float32) in every operating mode; no discernible function without room modules |

The parameter directory `0x0033` of the ventilation unit has 163 entries (for reading it
page by page see [Protocol](protokoll.md#reading-the-directory-page-by-page)).

!!! note "Clock `0x032e`"
    The value is a uint32 with minute resolution: bits 0–5 minute, 6–10 hour,
    11–13 weekday (0 = Sunday), 14–20 year − 2000, 21–24 month, 25–29 day. A
    SET `0x032e` (node 1 to 2, 4 bytes) is acknowledged immediately; date, hour and
    minute are taken over, the controller keeps counting the seconds itself.
    Without the panel nobody sets the clock: on the unit examined it was running on a
    date in 2011 after the panel had been removed. The format comes from Martin Kneppe
    (FWT 2-R) and was confirmed on the P 2H-L by setting and reading back.

## Heat pump demand

| DP | Value | Meaning |
|---|---|---|
| `0x0115` | 0.4 | Room switch-on threshold in K: demand as soon as the room is this far below the setpoint |
| `0x011c` | 2.0 | Continuous-run limit in °C: demand regardless of the room as long as the outdoor air (T3) is below it |

!!! note "Observation in Eco-Winter"
    Ten days of minute data were evaluated (room setpoint 20 °C). All eleven starts
    triggered by the room occurred at a room temperature of 19.6 °C; the heat pump stopped
    as soon as the room reached the setpoint again. Three further starts occurred with the
    room above that threshold, each shortly after T3 had fallen to 2.0 °C or below. No start
    occurred without one of the two conditions. The continuous run ended at T3 around
    2.9 °C, in four of five cases directly after a defrost cycle, because the outdoor air
    sensor briefly warms up in the stationary air. The meaning of both data points comes
    from Martin Kneppe (FWT 2-R); on the P 2H-L it rests on the matching thresholds, not on
    changing the values.

## PTC module (node 30)

| DP | Type | Meaning |
|---|---|---|
| `0x0191` | int16 | written cyclically by the panel: **0** in Eco-Summer and Eco-Winter, **1** in Stove and Comfort, `0xdc00` while the service menu is open |

The values were observed in captures with the panel connected (operating mode test and
service menu). In the operating mode test the panel question "enable heating elements"
was answered yes in all three heating modes; in Eco-Winter `0x0191` nevertheless stayed
0. The value therefore follows the operating mode, not the answer to that question. A
comparison with the switching state "PTC living" is still pending. A capture from an FWT 2-R shows the same pattern: the panel writes
`0x0191` to node 30 about every 11 seconds. Without the panel nobody writes this data
point; whether the PTC module ever enables the heating elements in that case is open.

## Observed constraints when writing

The following points are observations from the investigation, not instructions. They
describe the behaviour of the bus that a writing implementation has to take into account:

- **A complete frame including the checksum** is required; the checksum is described on
  the [protocol page](protokoll.md#checksum).
- **Cyclic repetition:** the STM32 master retransmits the panel state about every
  5 seconds. A single set frame is overwritten again by this; holding a state permanently
  requires repetition at a shorter interval (≈ 1.5 s proved effective in this
  investigation).
- **Independent verification of effect:** the effect of a set frame can be observed on a
  quantity independent of it, for example the actual speed `0x00c9`.
- **Fail-safe behaviour:** if the master drops out, the unit keeps ventilating in a
  fallback state; it does not come to a standstill.
