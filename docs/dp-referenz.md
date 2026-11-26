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
| `0x0194` | T1 Zuluft |
| `0x0231` | T3 Frischluft |
| `0x0230` | T4 Fortluft |
| `0x0190` | T5 vor Verdampfer |
| `0x0191` | T6 Verdampfer |
| `0x022e` | T7 Abluft |
| `0x0192` | T8 nach Vorwärme |
| `0x022d` | T10 Kondensator |
| `0x0193` | T12 vor Kondensator |
| `0x0195` | T13 Kompressor |

!!! note "Trennung des Zuluftpfad-Paares `0x0194`/`0x022d`"
    Beide Datenpunkte sitzen im Zuluftpfad und messen ohne laufende Wärmepumpe
    identisch (konstanter Ruhe-Offset ~0,6 K). Die Zuordnung wurde in einem
    kontrollierten Wärmepumpenlauf bestimmt (auch im Sommer auslösbar: Betriebsart
    Komfort, Sollwert 5–6 K über Raumtemperatur; Verdichterstart hier nach unter
    zwei Minuten). Beim Verdichterstart steigt `0x022d` früher und deutlich steiler
    als `0x0194` (Fühler an der Wärmequelle führt) und hält nach dem Abschalten —
    um den Ruhe-Offset korrigiert — länger Restwärme. Beides weist `0x022d` als
    T10 Kondensator aus, `0x0194` entsprechend als T1 Zuluft. Referenzwerte des
    Laufs: T13 Kompressor 21 → 74 °C, T4 Fortluft 22 → 6 °C, T6 Verdampfer
    24 → 7 °C, Zuluftpfad-Paar 21 → 49 °C.

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
| `0x006c` | **Zustandswort** (u32); Bit 7 steht während der Abtauung |

!!! note "Bestimmung des Bypass-Status"
    `0x0160` wechselte in einer nächtlichen Langzeitmessung als einziger Datenpunkt
    genau in dem Moment von 0 auf 1, in dem die Frischlufttemperatur abends unter die
    Raumtemperatur fiel. Die Zulufttemperatur löste sich ab diesem Zeitpunkt messbar
    von der Ablufttemperatur und näherte sich der Frischluft — die Klappenbewegung ist
    damit an einer unabhängigen Größe nachgewiesen. Die Zwischenlage der Zuluft
    (zwischen Voll-Wärmerückgewinnung und Voll-Bypass) passt zum **geregelten**
    Sommerbypass. Zeitgleich wechselte `0x01f6` von `0x40` auf `0x20`
    (Statuswort-Kandidat, Bedeutung offen).

!!! note "Abtau-Bit in `0x006c`"
    Im Normalbetrieb stand `0x006c` bei der untersuchten Anlage konstant auf 39 (Bits 0, 1, 2
    und 5), während der Abtauung auf 167 (zusätzlich Bit 7). An drei aufeinanderfolgenden
    Abtauungen wurde Bit 7 etwa 66 s vor dem Stillstand der Lüfter gesetzt und fiel genau mit
    ihrem Wiederanlauf ab. Bei der P 2H-L stehen während der Abtauung beide Lüfter still
    (Dauer etwa 6 min, Abstand etwa 38 min bei Frischluft um 0 °C); an einer FWT 2-R laufen
    sie nach Angabe von Martin Kneppe mit 20 % Zuluft und 60 % Abluft weiter. Den Hinweis auf
    das Bit gab Martin Kneppe; ob es auch beim Anlauf der Wärmepumpe kurz gesetzt wird, ist an
    der P 2H-L noch nicht beobachtet.

!!! note "Filter-Restlaufzeit"
    `0x00ed` stimmte zum Messzeitpunkt exakt mit der Panel-Anzeige der
    Filter-Restlaufzeit überein und war der einzige Datenpunkt mit diesem Wert im
    gesamten Adressraum. Das tägliche Dekrement ist bestätigt; offen ist allein der
    Zeitpunkt, zu dem es erfolgt (nicht Mitternacht).

## Zähler und weitere gelesene Datenpunkte

| DP | Bedeutung |
|---|---|
| `0x02df` | **Betriebsstundenzähler-Array** (u32-Liste, 9 belegte Slots, Inkrement stündlich); Einzelwerte auch als `0x02d0`–`0x02d9` lesbar |
| `0x0066` | Array aus 12 Flags (u32) — vermutlich Schaltausgänge; Zuordnung offen, der Bypass-Status meldet sich **nicht** hier, sondern in `0x0160` |
| `0x0064` | Analog-Rohwerte der Fühlerkanäle (float32, mV; offene Kanäle liegen auf der 3,3-V-Referenz) |
| `0x03e8` | Status-/Logtext des Controllers (z. B. „Writing parameters to NVRAM backup") |
| `0x032e` | **Datum und Uhrzeit** des Controllers, gepackt (siehe unten) |
| `0x00d2` / `0x00d3` | Lüfterstufen-→-Prozent-Kennlinien |
| `0x0233`–`0x023e` | Parameterblock (Winter/Wärmepumpe), nur auf oberster Zugriffsebene schreibbar |
| `0x0229` / `0x022a` | vom Panel zyklisch geschrieben, bei der untersuchten Anlage in allen Betriebsarten **konstant 21,0** (float32); ohne Raummodule keine erkennbare Funktion |

Das Parameterverzeichnis `0x0033` des Lüftungsgeräts umfasst 163 Einträge (seitenweises
Lesen siehe [Protokoll](protokoll.md#verzeichnis-seitenweise-lesen)).

!!! note "Uhr `0x032e`"
    Der Wert ist ein uint32 mit Minutenauflösung: Bit 0–5 Minute, 6–10 Stunde,
    11–13 Wochentag (0 = Sonntag), 14–20 Jahr − 2000, 21–24 Monat, 25–29 Tag. Ein
    SET `0x032e` (Knoten 1 an 2, 4 Byte) wird sofort quittiert; übernommen werden
    Datum, Stunde und Minute, die Sekunden führt der Controller selbst weiter.
    Ohne Panel stellt niemand die Uhr: Bei der untersuchten Anlage lief sie nach dem
    Ausbau des Panels auf einem Stand von 2011. Das Format stammt von Martin Kneppe
    (FWT 2-R) und ist an der P 2H-L durch Setzen und Zurücklesen bestätigt.

## Anforderung der Wärmepumpe

| DP | Wert | Bedeutung |
|---|---|---|
| `0x0115` | 0,4 | Einschaltschwelle Raum in K: Anforderung, sobald der Raum um diesen Betrag unter dem Sollwert liegt |
| `0x011c` | 2,0 | Dauerlauf-Grenze in °C: Anforderung unabhängig vom Raum, solange die Frischluft (T3) darunter liegt |

!!! note "Beobachtung in Eco-Winter"
    Ausgewertet wurden zehn Tage Minutendaten (Raumsoll 20 °C). Alle elf Anläufe, die über
    den Raum ausgelöst wurden, lagen bei einer Raumtemperatur von 19,6 °C; abgeschaltet
    wurde, sobald der Raum das Soll wieder erreichte. Drei weitere Anläufe erfolgten bei
    einem Raum oberhalb dieser Schwelle, jeweils kurz nachdem T3 auf 2,0 °C oder darunter
    gefallen war. Ein Anlauf ohne eine der beiden Bedingungen trat nicht auf. Der Dauerlauf
    endete bei T3 um 2,9 °C, in vier von fünf Fällen unmittelbar nach einer Abtauung, weil
    der Frischluftfühler im stehenden Luftstrom kurzzeitig wärmer wird. Die Bedeutung beider
    Datenpunkte stammt von Martin Kneppe (FWT 2-R); an der P 2H-L stützt sie sich auf die
    Übereinstimmung der Schwellen, nicht auf eine Änderung der Werte.

## PTC-Modul (Knoten 30)

| DP | Typ | Bedeutung |
|---|---|---|
| `0x0191` | int16 | vom Panel zyklisch geschrieben: **0** in Eco-Sommer und Eco-Winter, **1** in Ofen und Komfort, `0xdc00` solange das Service-Menü geöffnet ist |

Die Werte wurden in Mitschnitten mit angeschlossenem Panel beobachtet (Betriebsarten-Test
und Service-Menü). Im Betriebsarten-Test wurde die Panel-Frage „Wärmeelemente freigeben“
in allen drei Heizbetriebsarten bejaht; in Eco-Winter blieb `0x0191` dennoch 0. Der Wert
folgt damit der Betriebsart, nicht der Antwort auf diese Frage. Ein Abgleich mit dem
Schaltzustand „PTC-Wohnen“ steht aus. Ein Mitschnitt einer FWT 2-R zeigt dasselbe Muster: das Panel schreibt
`0x0191` an Knoten 30 etwa alle 11 Sekunden. Ohne Panel schreibt diesen Datenpunkt
niemand; ob das PTC-Modul die Wärmeelemente dann jemals freigibt, ist offen.

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
