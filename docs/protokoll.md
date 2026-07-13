# Das HESP-Protokoll

**HESP** ist das binäre Query/Response-Protokoll auf dem 19200-Baud-Bus (X7). Der Name
und die Grundstruktur stammen aus einem FHEM-Forum-Thread (User *mkneppe*, Topic 96437)
zur verwandten Hermes-/P-Serie; die konkrete Frame- und Prüfsummen-Auflösung hier stammt
aus eigenen Mitschnitten.

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

## Die Prüfsumme — geknackt

Die letzten zwei Bytes sind eine Prüfsumme, die in den Foren als „ungeknackt" galt.

- **Keine Standard-CRC-16.** Alle 65536 Polynome × Reflect-Varianten × Byte-Order gegen
  echte Frame-Paare getestet → kein Treffer.
- **Aber linear über GF(2):** Die Abbildung Nutzframe → Prüfsumme ist affin. Ein
  end-ausgerichtetes affines Modell (variable End-Bits + Konstante pro Frame-Länge),
  aus ~1650 kurzen Frames gelöst, sagt **331 von 331 zurückgehaltenen Frames aller Typen
  korrekt voraus**.

Damit ist die Prüfsumme für selbstgebaute Frames berechenbar — die Voraussetzung fürs
Schreiben.

!!! note "Grenze des Prüfsummen-Modells"
    Das affine Modell deckt nur die im Training beobachteten variablen Bit-Positionen ab
    (typische Query/Set-Frames auf Sub-Adresse 0). Für exotische Datenbytes muss man
    entweder den Trainingsmitschnitt verbreitern oder einen echten Master-Frame mit
    genau diesem Wert aus einem Capture verbatim wiederverwenden.

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
