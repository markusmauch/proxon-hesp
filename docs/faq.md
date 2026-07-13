# Offene Fragen & FAQ

## Kann ich meine PROXON P-Serie so auch steuern?

Wahrscheinlich ja, wenn es dieselbe Zwei-Platinen-Generation mit LPC1758-Hauptplatine
und PTC-Modul ist. Nötig ist ein Zugang zum **X7**-Steckverbinder und ein USB-RS485-
Adapter. Alle Werte hier stammen von **einer** Anlage — DP-Zuordnungen und Ausstattung
(Kühlfunktion, Bypass, EWT) können abweichen. Vorgehen: erst passiv mitlesen, DP-Inventar
selbst ziehen, am Panel Stufe/Modus ändern und beobachten, welcher DP springt.

## Brauche ich das Hersteller-Gateway / die kostenpflichtige Software?

Nein — genau das ist der Punkt. Lesen und Schreiben laufen direkt über den internen Bus.

## Ist das gefährlich für die Anlage?

Das Lesen ist unkritisch. Beim Schreiben gilt: nur bekannte, reversible Kommandos, und
die Anlage hat ein Fail-Safe (läuft bei Master-Ausfall weiter). Trotzdem: **eigenes
Risiko**, und Eingriffe können Gewährleistung/Garantie berühren.

## Warum nicht einfach den Panel-Bus nachbauen?

Weil er jede Injektion an einer analogen Wellenform-Eigenschaft erkennt und ablehnt —
siehe [Geschichte](geschichte.md). X7 ist der richtige Angriffspunkt.

## Warum stimmen manche Sensornamen nicht mit dem WR3223 überein?

Der WR3223 ist ein verwandtes Gerät und diente lange als Namensquelle — liegt aber teils
falsch (z. B. ist Proxon-**T7 = Abluft**, nicht „nach EWT"). Maßgeblich sind die
offiziellen T-Nummern aus dem Panel-Menü „Aktuelle Messwerte".

## Offene technische Punkte

| Frage | Stand |
|---|---|
| T1 vs. T10 (`0x0194`/`0x022d`) trennen | erst im WP-Betrieb (Winter) möglich |
| Bypass-Status-Bit in `0x0066` | über Abend-Crossing-Messung zu bestätigen |
| Filter-Restlaufzeit als DP | Suche über Wert-Abgleich (Panel zeigt Resttage) |
| STM32: Brücke oder Smart-Slave? | Timing-Mitschnitt 9600 ↔ X7 nötig |
| Reboot über Bus | nicht gefunden — Trigger sitzt auf der 9600-Seite/HW |
| Fachmann-Parameter `0x0233`–`0x023e` benennen | Vergleich mit PIN-Menü nötig |

## Mitmachen

Wer eine P-Serie (oder Hermes/WR3223) besitzt und eigene Messungen beisteuern kann:
Issues und Pull Requests auf [GitHub](https://github.com/markusmauch/proxon-hesp)
sind willkommen — besonders DP-Zuordnungen anderer Ausbaustufen.
