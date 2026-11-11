# PROXON P series · HESP bus (technical documentation)

***English** · [Deutsch](README.de.md)*

Technical documentation of the internal RS485 bus (**HESP**, 19200 baud) of the Zimmermann
**PROXON P series** ventilation heating unit: physical interface, telegram format and data
points as a basis for independent local read and control implementations. The
documentation is deliberately implementation-neutral.

📖 **Website:** https://markusmauch.github.io/proxon-hesp/en/
(German: https://markusmauch.github.io/proxon-hesp/)

## Contents

- **Unit & architecture** — two-board design, connector X7
- **HESP protocol** — frame format, telegram types, checksum
- **Data point reference** — control and sensor data points
- **Constraints & open points** — bus properties and questions not yet settled

## Applicability

All statements derive from measurements on a single unit, obtained without the
manufacturer's involvement. Reproduce at your own risk; working on the unit may affect
warranty or guarantee. No proprietary manufacturer documentation is reproduced here.

## Licence & trademarks

Content under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.en) (see
[`LICENSE`](LICENSE)). Independent project, **not** affiliated with Zimmermann Lüftungs-
und Wärmesysteme GmbH & Co. KG; "PROXON", "Zimmermann" and other names mentioned are
trademarks of their respective owners and are used descriptively.
