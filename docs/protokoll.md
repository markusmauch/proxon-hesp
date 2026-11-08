# Das HESP-Protokoll

**HESP** ist das binäre Anfrage/Antwort-Protokoll auf dem 19200-Baud-Bus (X7). Name und
Grundstruktur des Telegramms sind in einem FHEM-Forum-Thread (Topic 96437) zur
verwandten Hermes-/P-Serie vorbeschrieben; die hier dokumentierte Frame-Struktur und das
Prüfsummenmodell wurden aus eigenen Busmitschnitten bestimmt.

## Physische Ebene

- **19200 Baud, 8N1**, RS485 halbduplex, differentielles A/B-Paar auf X7-4/X7-3.
- Master ist der **STM32** des PTC-Moduls; er pollt den LPC1758 etwa alle 5 Sekunden und
  re-asserted dabei den aktuellen Panel-Zustand (Stufe/Modus/Sollwerte).

## Frame-Format

Jeder Frame:

```
<h0> <h1> <h2>   <id_lo> <id_hi>   00 00   <len>   <data …>   <crc_lo> <crc_hi>
```

| Feld | Bytes | Bedeutung |
|---|---|---|
| `h0 h1 h2` | 3 | **Adresskopf**, 24-bit little-endian: Typ, Quell- und Zieladresse (siehe unten) |
| `id_lo id_hi` | 2 | **Datenpunkt-ID, 16-bit little-endian** |
| `00 00` | 2 | Reserviert / Sub-Selektor |
| `len` | 1 | **Nibble-Anzahl** der Daten (`0x08`=4 B, `0x20`=16 B, `0xe0`=224 B) |
| `data` | n | Nutzdaten (float32 LE, int, Array …) |
| `crc` | 2 | Prüfsumme (little-endian) |

### Der Adresskopf

Die ersten drei Bytes bilden vermutlich ein 24-bit-Wort `H = h0 | h1 << 8 | h2 << 16` mit
drei Feldern:

| Bits | Feld | Breite |
|---|---|---|
| 0–3 | Typ (siehe unten) | 4 bit |
| 4–13 | Quelladresse | 10 bit |
| 14–23 | Zieladresse | 10 bit |

Diese Deutung würde die bisher als „fester Kopf-Rest" bzw. „Node-abhängig" beschriebenen
Bytes erklären: Anfrage und Antwort tauschen Quell- und Zieladresse.

| Kopf | Typ | Quelle → Ziel |
|---|---|---|
| `10 80 00` | QUERY | 1 → 2 |
| `22 40 00` | RESPONSE | 2 → 1 |
| `11 80 00` | SET | 1 → 2 |
| `23 40 00` | CONFIRM | 2 → 1 |
| `10 80 07` | QUERY | 1 → 30 |
| `10 00 0a` | QUERY | 1 → 40 |

Adresse 1 ist der pollende Master. Die Zerlegung ist mit den Mitschnitten einer zweiten
Anlage mit weiteren Busteilnehmern (u. a. Adressen 10–15, 30, 40) konsistent: Jede
Antwort trägt dort die vertauschten Adressen der vorausgehenden Anfrage. Eine
Bestätigung durch Herstellerunterlagen liegt nicht vor.

### Der Typ-Nibble

Das Low-Nibble des ersten Bytes kodiert die Telegramm-Art — exakt wie im Forum
dokumentiert und am Draht bestätigt:

| Nibble | Typ | Header-Beispiel |
|---|---|---|
| `0` | **QUERY** (Anfrage) | `10 80 00 …` |
| `1` | **SET** (Schreiben) | `11 80 00 …` |
| `2` | **RESPONSE** (Antwort) | `22 40 00 …` |
| `3` | **CONFIRM** (Schreibbestätigung) | `23 40 00 …` |

### Beispiele

```text
QUERY    10 80 00 60 01 00 00 04 01 00 49 e1        # frage DP 0x0160
RESPONSE 22 40 00 d2 00 00 00 20 …16 B… 5a cb        # Antwort mit 16 B Daten
SET      11 80 00 27 02 00 00 08 00 00 98 41 8a cd  # setze DP 0x0227 = 19.0 (float32)
CONFIRM  23 40 00 27 02 00 00 00 39 a1              # Schreiben bestätigt
```

Nutzdaten sind **IEEE-754 float32 little-endian** für Temperaturen und Sollwerte, sonst
Integer (RPM, Zähler, Flags) oder Arrays (Kennlinien).

## Prüfsumme

Die letzten zwei Bytes jedes Frames bilden eine **CRC-16** über alle vorhergehenden Bytes
(Kopf bis einschließlich Nutzdaten), little-endian abgelegt:

| Parameter | Wert |
|---|---|
| Polynom | `0x8048` in reflektierter Darstellung (LSB zuerst; normal: `0x1201`) |
| Startwert | `0xFFFF` |
| Final-XOR | keines |
| Schiebeschritte je Byte | **16** |

Die Besonderheit ist die Zahl der Schiebeschritte: Nach dem Einrechnen eines Bytes wird
das Register 16-mal statt 8-mal geschoben. Das ist gleichbedeutend damit, dass jedes
Datenbyte als 16-bit-Wort mit High-Byte `0x00` in eine gewöhnliche reflektierte CRC-16
eingeht. Diese Eigenheit erklärt, warum eine Suche über Standard-Parametrierungen (alle
Polynome, Reflect-Varianten, Byte-Reihenfolgen) keine Übereinstimmung liefert.

```python
def hesp_crc(data: bytes) -> int:
    crc = 0xFFFF
    for b in data:
        crc ^= b
        for _ in range(16):
            crc = (crc >> 1) ^ 0x8048 if crc & 1 else crc >> 1
    return crc          # im Frame: crc & 0xFF, dann crc >> 8
```

Geprüft an allen Beispielen dieser Seite sowie an rund 6000 Frames einer zweiten Anlage
(alle vier Telegrammtypen, Frame-Längen 10–134 Byte) ohne Abweichung. Die Parameter sind
aus Mitschnitten ermittelt; eine Bestätigung durch Herstellerunterlagen liegt nicht vor.

### Zusammenhang mit dem affinen GF(2)-Modell

Eine frühere Fassung dieser Seite beschrieb die Prüfsumme empirisch als affine Abbildung
über GF(2) mit einer Tabelle von Beitragsvektoren `V[e]`. Die Linearität ist eine direkte
Folge der CRC. Die Tabelle war jedoch nicht eindeutig bestimmt: Bits, die im
Trainingsmaterial stets gemeinsam bzw. konstant gesetzt waren, lassen sich nicht von der
längenabhängigen Konstante trennen. Tatsächlich ist die Konstante **nicht null**, sondern
die CRC eines Null-Frames gleicher Länge (z. B. `27d8` für 10 Byte, `edc9` für 12 Byte).
Die Tabelle lieferte daher für die beobachteten Bitmuster korrekte Ergebnisse, nicht aber
für beliebige Frames. Die Beitragsvektoren folgen aus dem Algorithmus als
`V[e] = crc(Frame mit nur Bit e gesetzt) ⊕ crc(Null-Frame)` und hängen nur von `e`
ab, nicht von der Frame-Länge.

??? info "Tabelle der Beitragsvektoren `V[e]` (aus dem Algorithmus, `e` = 0–95)"

    Bitposition vom Frame-Ende: Für Byte `i` (0-basiert), Bit `b` (0 = LSB) eines Frames mit
    `N` Bytes ohne CRC gilt `e = (N − 1 − i) · 8 + b`. Damit ist
    `CRC = K(N) ⊕ (⊕ V[e] über alle gesetzten Bits)` mit `K(N)` = CRC eines Null-Frames der Länge `N`.

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

### Rechenbeispiel

Anfrage-Frame für Datenpunkt `0x0160` (siehe Beispiele oben):

```python
>>> hex(hesp_crc(bytes.fromhex("10 80 00 60 01 00 00 04 01 00")))
'0xe149'
```

→ `crc_lo = 49`, `crc_hi = e1`; der vollständige Frame auf dem Bus lautet also

```
10 80 00 60 01 00 00 04 01 00 49 e1
```

und entspricht exakt dem Mitschnitt.

## Meta-Datenpunkte (Selbstauskunft)

Der Bus beschreibt sich teilweise selbst:

| DP | Inhalt |
|---|---|
| `0x0001` | Klartextname des Nodes (z. B. Node `0x41` = „PTC-Modul") |
| `0x0033` | Liste aller Parameter-DP-Nummern (Verzeichnis) |
| `0x0034` | parallele Typcode-Liste dazu |
| `0x0038` | **Zugriffslevel-Maske** je DP (siehe unten) |
| `0x0039` | Einheiten-Code je DP (8=°C, 5=Sekunden, 11=String …) |
| `0x0130` | Fehlerstatus als Klartext („Kein Fehler") |

### Zugriffsmaske `0x0038`

Ein Byte pro DP kodiert die Schreibrechte über vier Zugriffsebenen (je 2 Bit R/W):

| Maske | Bedeutung | trifft |
|---|---|---|
| `0x55` | read-only | alle Messwerte, Zähler |
| `0xFF` | überall schreibbar | die vom Panel gesetzten User-Werte (Raumsoll etc.) |
| `0xD5` | nur oberste Ebene schreibbar | der **Fachmann-Parameterblock** |
| `0xC0` | nur oberste Ebene sichtbar | Werks-Kennlinien, Lüfter-Arrays |

Die Maske bestätigt unabhängig, welche DPs das (PIN-geschützte) Fachmann-Menü ausmachen —
nützlich, um beim Parametrieren nicht blind in geschützte Register zu schreiben.
