# PROXON P series · The internal HESP bus

## Scope

This documentation describes the internal RS485 communication bus of the **Zimmermann
PROXON P series** (investigated on the P 2H-L model, counter-flow heat recovery with an
air-to-air heat pump). The bus is designated **HESP** internally.

The aim is to describe the physical interface, the telegram format and the data points
completely enough that an **independent local read and control implementation** can be
built on top of them, on any platform. No manufacturer-provided route exists for this:
according to the manufacturer's own statement, connection to building management systems
is **neither provided for nor retrofittable** on the P series. The "BusBridge" known from
forum posts applies exclusively to the FWT series with its integrated data bus
(manufactured 2012–2018) and does not transfer to the P series. The interface described
here is therefore the only known point of access. This documentation is deliberately
**implementation-neutral**: it describes the interface, not any particular application.

## System overview

The P series is operated from a central TFT control panel, which identifies itself as
"BDE Comfort" in its system information. Internally, two boards communicate through
connector **X7** at **19200 baud, 8N1**. On this bus an NXP LPC1758 controller provides
all sensor readings, fan stages, operating modes and setpoints as typed **data points**.

```mermaid
flowchart LR
    BDE["TFT panel<br/>(BDE Comfort)"] -->|9600 8N1<br/>Cat7| PTC["PTC module<br/>STM32<br/>(X2 ⟷ X7)"]
    PTC -->|19200 8N1<br/>HESP| LPC["Main board<br/>LPC1758"]
    LPC --- SENS["Sensors · fans ·<br/>heat pump · valves"]
```

## State of the investigation

| Aspect | State |
|---|---|
| Physical interface X7 (pinout, levels) | determined |
| Telegram format (header, types, lengths, data types) | determined |
| Checksum | determined as a CRC-16 (polynomial, init value, shift steps) |
| Reading data points over X7 | reproducible |
| Writing control data points over X7 | demonstrated, physical effect confirmed |
| Complete assignment of all temperature sensors | complete but for one sensor pair |

The points not yet conclusively settled are collected under
[Constraints & open points](offene-punkte.md).

!!! note "Applicability"
    All statements derive from measurements on **one** unit and were obtained without the
    manufacturer's involvement. Data point assignments and equipment features (for
    example cooling function, bypass, ground heat exchanger) may differ on other
    configurations. Working on the bus is done at your own risk and may affect warranty
    or guarantee. No proprietary manufacturer documentation is reproduced here.

## Licence & trademarks

This documentation is licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/deed.en):
reuse and adaptation are permitted provided the source is credited. Short protocol and
code examples may additionally be used without attribution.

This is an **independent project** with no connection to Zimmermann Lüftungs- und
Wärmesysteme GmbH & Co. KG. "PROXON", "Zimmermann" and other names mentioned are
trademarks of their respective owners and are used descriptively only.

## Support

[:material-coffee: Support this work](https://buymeacoffee.com/markusmauch){ .md-button .md-button--primary }

This documentation is free and grew out of many hours of measurement work. If it helps
you, I would be glad of a coffee.
