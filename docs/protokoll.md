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
H0  80  00   <id_lo> <id_hi>   00 00   <len>   <data …>   <crc_lo> <crc_hi>
```

| Feld | Bytes | Bedeutung |
|---|---|---|
| `H0` | 1 | Header. **Low-Nibble = Typ** (siehe unten), High-Nibble Länge/Node-abhängig |
| `80 00` | 2 | fester Kopf-Rest (Node/Subadresse; andere Nodes z. B. `…07…`) |
| `id_lo id_hi` | 2 | **Datenpunkt-ID, 16-bit little-endian** |
| `00 00` | 2 | Reserviert / Sub-Selektor |
| `len` | 1 | **Nibble-Anzahl** der Daten (`0x08`=4 B, `0x20`=16 B, `0xe0`=224 B) |
| `data` | n | Nutzdaten (float32 LE, int, Array …) |
| `crc` | 2 | Prüfsumme (little-endian) |

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

Die letzten zwei Bytes jedes Frames bilden eine Prüfsumme.

- Sie entspricht **keiner Standard-CRC-16**: alle 65536 Polynome in Kombination mit
  Reflect-Varianten und Byte-Reihenfolge wurden gegen echte Frame-Paare geprüft, ohne
  Übereinstimmung.
- Die Prüfsumme ist jedoch eine **affine Abbildung über GF(2)** (linear plus Konstante).
  Ein end-ausgerichtetes affines Modell — variable End-Bits zuzüglich einer Konstante je
  Frame-Länge, bestimmt aus rund 1650 kurzen Frames — sagt **331 von 331 zurückgehaltenen
  Frames aller Typen korrekt voraus**. Die gelösten Konstanten sind für alle beobachteten
  Frame-Längen (8–12 Bytes) **null**, das Modell ist also in diesem Bereich rein linear:
  Die Prüfsumme ist ein XOR fester 16-bit-Beitragsvektoren über alle gesetzten
  Nachrichtenbits.

Damit ist die Prüfsumme für selbst erzeugte Frames deterministisch berechenbar.

### Modell in Tabellenform

Bits werden **vom Frame-Ende her** indiziert (GF(2)-Konvention „end-aligned"): Für Byte
`i` (0-basiert ab Frame-Anfang), Bit `b` (0 = LSB) eines Frames mit `N` Bytes (ohne die
2 CRC-Bytes) ist die End-Bitposition

```
e = (N − 1 − i) · 8 + b
```

Die 16-bit-Prüfsumme (dargestellt als `crc_hi·256 + crc_lo`; im Frame steht `crc_lo`
zuerst) ist das XOR der Beitragsvektoren `V[e]` aller gesetzten Nachrichtenbits:

```
CRC = ⊕ V[e]   über alle Bits e mit Wert 1
```

| `e` | `V[e]` | `e` | `V[e]` | `e` | `V[e]` | `e` | `V[e]` |
|---|---|---|---|---|---|---|---|
| 0 | `8bb4` | 1 | `a379` | 2 | `b9e0` | 3 | `0222` |
| 4 | `0000` | 5 | `0000` | 6 | `6474` | 7 | `92a8` |
| 8 | `80fc` | 9 | `0169` | 10 | `02d2` | 11 | `05a4` |
| 12 | `0b48` | 13 | `1690` | 14 | `2d20` | 15 | `0000` |
| 16 | `cd56` | 17 | `9a3d` | 18 | `34eb` | 19 | `69d6` |
| 20 | `d3ac` | 21 | `a7c9` | 22 | `4f03` | 23 | `9e06` |
| 24 | `0534` | 25 | `0a68` | 26 | `14d0` | 27 | `29a0` |
| 28 | `5340` | 29 | `a680` | 30 | `4d91` | 31 | `9b22` |
| 32 | `b823` | 33 | `70d7` | 34 | `e1ae` | 35 | `c3cd` |
| 36 | `870b` | 37 | `0e87` | 38 | `1d0e` | 39 | `3a1c` |
| 40 | `78a8` | 41 | `f150` | 42 | `e231` | 43 | `c4f3` |
| 45 | `c2be` | 46 | `0000` | 48 | `8e67` | 49 | `1c5f` |
| 50 | `38be` | 51 | `717c` | 52 | `e2f8` | 53 | `c561` |
| 54 | `8a53` | 55 | `1437` | 56 | `0c91` | 57 | `1922` |
| 58 | `3244` | 59 | `ecb4` | 61 | `136c` | 62 | `6e15` |
| 63 | `0000` | 64 | `cc6b` | 65 | `9847` | 66 | `301f` |
| 67 | `603e` | 68 | `c07c` | 69 | `8069` | 70 | `0043` |
| 71 | `0086` | 72 | `a403` | 73 | `e47b` | 76 | `6242` |
| 77 | `0000` | 78 | `3b0b` | 79 | `0000` | 81 | `0000` |
| 85 | `0000` | 86 | `0daf` | 87 | `1213` | 88 | `0000` |
| 89 | `0000` | 92 | `0000` | 93 | `0000` |  |  |

!!! note "Geltungsbereich des Modells"
    Die Positionen `e` ∈ {44, 47, 60, 74, 75, 80, 82, 83, 84, 90, 91, 94, 95} fehlen in
    der Tabelle: Diese Bits waren im gesamten Trainingsmaterial konstant, ihre
    Beitragsvektoren sind daher **unbekannt** (nicht null!). Das Modell gilt nur für
    Frames, deren gesetzte Bits von der Tabelle abgedeckt sind (typische Anfrage-/
    Setz-Frames auf Sub-Adresse 0, Längen 8–12 Bytes). Für davon abweichende Datenbytes
    ist der Trainingsdatensatz zu erweitern oder ein entsprechender Master-Frame aus
    einem Mitschnitt unverändert zu übernehmen. Die `0000`-Einträge dagegen sind echte,
    gelöste Nullbeiträge.

### Rechenbeispiel

Anfrage-Frame für Datenpunkt `0x0160` (siehe Beispiele oben), `N = 10` Nutzbytes:

```
10 80 00 60 01 00 00 04 01 00
```

Gesetzte Bits und ihre Beitragsvektoren:

| Byte `i` | Wert | Bit `b` | `e = (9−i)·8+b` | `V[e]` |
|---|---|---|---|---|
| 0 | `10` | 4 | 76 | `6242` |
| 1 | `80` | 7 | 71 | `0086` |
| 3 | `60` | 5 | 53 | `c561` |
| 3 | `60` | 6 | 54 | `8a53` |
| 4 | `01` | 0 | 40 | `78a8` |
| 7 | `04` | 2 | 18 | `34eb` |
| 8 | `01` | 0 | 8  | `80fc` |

XOR aller sieben Vektoren:

```
6242 ⊕ 0086 ⊕ c561 ⊕ 8a53 ⊕ 78a8 ⊕ 34eb ⊕ 80fc = e149
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
