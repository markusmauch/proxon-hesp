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
  Frames aller Typen korrekt voraus**.

Damit ist die Prüfsumme für selbst erzeugte Frames deterministisch berechenbar.

!!! note "Geltungsbereich des Modells"
    Das affine Modell deckt die im Trainingsdatensatz beobachteten variablen
    Bit-Positionen ab (typische Anfrage/Setz-Frames auf Sub-Adresse 0). Für davon
    abweichende Datenbytes ist der Trainingsdatensatz zu erweitern oder ein
    entsprechender Master-Frame aus einem Mitschnitt unverändert zu übernehmen.

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
