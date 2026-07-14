# Datenpunkt-Referenz

Alle IDs sind 16-bit. Das High-Byte gruppiert grob nach Funktion. Werte in Klammern
sind Beispielwerte einer laufenden Anlage. **Diese Zuordnung gilt für eine P 2H-L** und
kann bei anderen Ausbaustufen abweichen.

## Steuer-Datenpunkte (schreibbar)

Durch gezielte Bedienung am Panel eindeutig identifiziert — bei Änderung der jeweiligen
Größe am Panel ändert sich genau dieser Datenpunkt:

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

!!! note "Betriebsart-Kodierung"
    Die Kodierung wurde direkt am Bus bestimmt: **Aus=0, Eco-Sommer=1, Eco-Winter=2,
    Komfort=3, Ofen=4**. Sie weicht von der teils kolportierten, aus der verwandten
    WR3223-Baureihe abgeleiteten Zuordnung ab.

## Temperaturfühler

Zehn physische Kanäle. Die Namen entsprechen den **T-Nummern** aus dem Panel-Menü
„Aktuelle Messwerte" und wurden durch Abgleich zeitgleicher Buswerte mit der
Panelanzeige zugeordnet.

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

## Status-Datenpunkte

| DP | Bedeutung |
|---|---|
| `0x0160` | **Sommer-Bypass-Status**: 0 = geschlossen (Wärmerückgewinnung aktiv), 1 = Bypass aktiv |
| `0x00ed` | **Filter-Restlaufzeit** in Tagen (u32) |
| `0x0130` | Fehlertext (Klartext) |

!!! note "Bestimmung des Bypass-Status"
    `0x0160` wechselte in einer nächtlichen Langzeitmessung als einziger Datenpunkt
    genau in dem Moment von 0 auf 1, in dem die Frischlufttemperatur abends unter die
    Raumtemperatur fiel. Die Zulufttemperatur löste sich ab diesem Zeitpunkt messbar
    von der Ablufttemperatur und näherte sich der Frischluft — die Klappenbewegung ist
    damit an einer unabhängigen Größe nachgewiesen. Die Zwischenlage der Zuluft
    (zwischen Voll-Wärmerückgewinnung und Voll-Bypass) passt zum **geregelten**
    Sommerbypass. Zeitgleich wechselte `0x01f6` von `0x40` auf `0x20`
    (Statuswort-Kandidat, Bedeutung offen).

!!! note "Filter-Restlaufzeit"
    `0x00ed` stimmte zum Messzeitpunkt exakt mit der Panel-Anzeige der
    Filter-Restlaufzeit überein und war der einzige Datenpunkt mit diesem Wert im
    gesamten Adressraum. Die abschließende Bestätigung (tägliches Dekrement) steht aus.

## Zähler und weitere gelesene Datenpunkte

| DP | Bedeutung |
|---|---|
| `0x02df` | **Betriebsstundenzähler-Array** (u32-Liste, 9 belegte Slots, Inkrement stündlich); Einzelwerte auch als `0x02d0`–`0x02d9` lesbar |
| `0x0066` | Array aus 12 Flags (u32) — vermutlich Schaltausgänge; Zuordnung offen, der Bypass-Status meldet sich **nicht** hier, sondern in `0x0160` |
| `0x0064` | Analog-Rohwerte der Fühlerkanäle (float32, mV; offene Kanäle liegen auf der 3,3-V-Referenz) |
| `0x03e8` | Status-/Logtext des Controllers (z. B. „Writing parameters to NVRAM backup") |
| `0x032e` | Betriebszeit seit Start (Sekunden) |
| `0x00d2` / `0x00d3` | Lüfterstufen-→-Prozent-Kennlinien |
| `0x0233`–`0x023e` | Parameterblock (Winter/Wärmepumpe), nur auf oberster Zugriffsebene schreibbar |

## Beobachtete Randbedingungen beim Schreiben

Die folgenden Punkte sind Beobachtungen aus der Untersuchung, keine Handlungsanweisung.
Sie beschreiben das Verhalten des Busses, das eine schreibende Implementierung
berücksichtigen muss:

- **Vollständiges Frame inklusive Prüfsumme** ist erforderlich; das Prüfsummenmodell ist
  auf der [Protokoll-Seite](protokoll.md#prufsumme) beschrieben.
- **Zyklische Wiederholung:** Der STM32-Master überträgt den Panel-Zustand etwa alle
  5 Sekunden erneut. Ein einzelner Setz-Frame wird dadurch wieder überschrieben; ein
  dauerhaft gehaltener Zustand erfordert Wiederholung in kürzerem Intervall (in dieser
  Untersuchung ≈ 1,5 s wirksam).
- **Unabhängige Wirkungskontrolle:** Der Effekt eines Setz-Frames lässt sich an einer
  davon unabhängigen Größe beobachten, etwa der Ist-Drehzahl `0x00c9`.
- **Fail-Safe-Verhalten:** Fällt der Master aus, lüftet die Anlage in einem
  Rückfallzustand weiter; sie kommt nicht zum Stillstand.
