# Datenpunkt-Referenz

Alle IDs sind 16-bit. Das High-Byte gruppiert grob nach Funktion. Werte in Klammern
sind Beispielwerte einer laufenden Anlage. **Diese Zuordnung gilt für eine P 2H-L** und
kann bei anderen Ausbaustufen abweichen.

## Steuer-Datenpunkte (schreibbar)

Per Panel-Perturbation eindeutig identifiziert — beim Ändern am Panel springt genau
dieser DP:

| DP | Typ | Bedeutung | Werte |
|---|---|---|---|
| `0x00e1` | int16 | **Lüfterstufe** | 0=Aus, 1, 2, 3, 4 |
| `0x020a` | int16 | **Betriebsart** | 0=Aus, 1=Eco-Sommer, 2=Eco-Winter, 3=Komfort, 4=Ofen |
| `0x0227` | float32 | **Raumsollwert** | 15–25 °C (Panel-Drehsteller) |
| `0x0226` | float32 | Ist-Raumtemperatur (Panel-Fühler, einspeisbar) | — |

Schreib-Frames:

```text
Lüfterstufe:  11 80 00 e1 00 00 00 04 <NN> 00 <crc16>   # NN = 01/02/03/04
Betriebsart:  11 80 00 0a 02 00 00 04 <NN> 00 <crc16>
Raumsoll:     11 80 00 27 02 00 00 08 <float32 LE> <crc16>
```

!!! warning "Betriebsarten korrigieren gängige Fehlannahmen"
    Die aus dem verwandten **WR3223** abgeleitete Betriebsart-Tabelle liegt hier falsch.
    Am echten Bus gemessen gilt: **Aus=0, Eco-Sommer=1, Eco-Winter=2, Komfort=3, Ofen=4**.

## Temperaturfühler

Zehn physische Kanäle. Die Namen sind die **offiziellen T-Nummern** aus dem BDE-Menü
„Aktuelle Messwerte", per Foto-Abgleich mit gefrorenen Bus-Werten zugeordnet.

| DP | Fühler |
|---|---|
| `0x0231` | T3 Frischluft |
| `0x0230` | T4 Fortluft |
| `0x0190` | T5 vor Verdampfer |
| `0x0191` | T6 Verdampfer |
| `0x022e` | T7 Abluft |
| `0x0192` | T8 nach Vorwärme |
| `0x0193` | T12 vor Kondensator |
| `0x0195` | T13 Kompressor |
| `0x0194` | T1 Zuluft **oder** T10 Kondensator |
| `0x022d` | T10 Kondensator **oder** T1 Zuluft |

!!! note "Das letzte Paar"
    `0x0194` und `0x022d` sitzen beide im Zuluftpfad und messen ohne laufende
    Wärmepumpe identisch. Sie lassen sich erst beim ersten WP-Lauf im Winter trennen
    (der Kondensator wird dann heiß, die Zuluft nicht).

## Lüfter & Drehzahlen

| DP | Bedeutung |
|---|---|
| `0x00c9` | Ist-Drehzahl, 2× float32 `[Zuluft, Abluft]` |
| `0x00d7` | Ziel-Drehzahl, 2× float32 `[Zuluft, Abluft]` |

## Sonstige gelesene Datenpunkte

| DP | Bedeutung |
|---|---|
| `0x0066` | **Bitfeld „DigitalOut"** — Schaltzustände (Bypass, PTC, Magnetventile, EWT) |
| `0x0130` | Fehlertext (Klartext) |
| `0x00d2` / `0x00d3` | Lüfterstufen-→-Prozent-Kennlinien |
| `0x0233`–`0x023e` | Fachmann-Parameterblock (Winter/Wärmepumpe), PIN-geschützt schreibbar |

## Sicherheitscheckliste vor jedem Schreibzugriff

1. Frame-Format inkl. Prüfsumme des Kanals kennen.
2. Nur **bekannte, reversible** Kommandos senden.
3. **Zyklisch senden** — der STM32-Master re-asserted den Panel-Zustand etwa alle 5 s;
   ein einzelner SET wird sonst wieder überschrieben. Zum dauerhaften Halten in kürzeren
   Intervallen (≈1,5 s) nachsetzen.
4. Wirkung an einem unabhängigen Signal verifizieren (z. B. Ist-Drehzahl `0x00c9`).
5. Die Anlage hat ein **Fail-Safe**: fällt der Master weg, lüftet sie weiter — sie steht
   nie still.
