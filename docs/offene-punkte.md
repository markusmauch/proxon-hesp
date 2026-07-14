# Randbedingungen & offene Punkte

Diese Seite fasst Randbedingungen der Schnittstelle sowie die Punkte zusammen, die noch
nicht abschließend bestimmt sind. Sie markiert damit den Ausgangspunkt für weitere Arbeit.

## Der 9600-Baud-Panel-Bus

Zwischen Bedienpanel und PTC-Modul besteht ein separater Bus mit **9600 Baud**. Für einen
Auslese- oder Steuerzugriff eignet er sich nach den vorliegenden Messungen **nicht**:

- Der Steuerzustand (Betriebsart, Lüfterstufe) wird auf diesem Bus **kontinuierlich in
  den regulären Poll-Frames des Masters übertragen**; ein separierbares Einzelkommando
  konnte nicht identifiziert werden.
- Von einem Fremdgerät erzeugte Frames erreichen die Geräteklemme nachweislich
  byte-genau und korrekt kodiert (mit Logic-Analyzer verifiziert), werden vom Gerät
  jedoch nicht beantwortet. Identische Frames vom Original-Panel werden ab dem ersten
  Byte beantwortet.
- Dieses Verhalten war über mehrere Transceiver-Typen und Pegel reproduzierbar. Es
  deutet darauf hin, dass das Gerät die Signalquelle an einer analogen Eigenschaft der
  Wellenform unterscheidet, die mit Standardhardware nicht ohne Weiteres nachgebildet
  wird.

Der Auslese- und Steuerzugriff erfolgt daher über den internen **19200-Baud-Bus (HESP)**
am Steckverbinder X7. Ob und wie sich das Verhalten des 9600-Baud-Busses vollständig
erklären lässt, ist offen.

## Weitere Randbedingungen der Anlage

Diese Punkte betreffen jede Implementierung, die das Bedienpanel ersetzt oder ergänzt:

- **Zeitprogramme und Nachtabsenkung** werden im Bedienpanel geführt, nicht im
  LPC1758-Controller. Ohne Panel entfällt diese Funktion und muss anderweitig
  bereitgestellt werden.
- **Filter-Restlaufzeit:** Das Gerät führt eine Filterstandzeit (im Panel in Tagen
  angezeigt). Wird der Filter nach Ablauf nicht gewechselt, schaltet das Zentralgerät
  nach einer Frist selbsttätig ab. Der zugehörige Datenpunkt ist als `0x00ed`
  identifiziert (siehe [Datenpunkt-Referenz](dp-referenz.md)).

## Offene Punkte

| Thema | Stand |
|---|---|
| Fühlerpaar `0x0194` / `0x022d` (T1 Zuluft ↔ T10 Kondensator) | erst im Wärmepumpenbetrieb unterscheidbar |
| Zuordnung der 12 Flags in `0x0066` (Schaltausgänge) | offen — der Bypass-Status liegt separat in `0x0160` |
| Filter-Restlaufzeit `0x00ed` | identifiziert (Abgleich mit Panelanzeige); Bestätigung über tägliches Dekrement steht aus |
| Benennung der einzelnen Betriebsstundenzähler in `0x02df` | erfordert Abgleich mit dem Panel-Systeminfo-Menü |
| Statuswort `0x01f6` (wechselt synchron zum Bypass) | Bedeutung offen |
| Rolle des STM32 (reine Frame-Brücke oder aggregierender Slave) | offen |
| Auslösung eines Geräte-Neustarts über X7 | kein entsprechender Frame gefunden |
| Benennung des Parameterblocks `0x0233`–`0x023e` | erfordert Abgleich mit dem geschützten Parametermenü |
| Vollständige Erklärung des 9600-Baud-Bus-Verhaltens | offen |

## Verwandte Quellen

- FHEM-Forum, Topic **96437** — erste öffentliche Beschreibung des HESP-Telegrammaufbaus
  derselben Herstellerfamilie.
- Für die neueren FWT-2.0-/T300-Geräte existieren **Modbus**-basierte Projekte; diese
  betreffen einen anderen Bus und sind auf den hier beschriebenen HESP-Binärbus nicht
  übertragbar.

## Beiträge

Messungen von weiteren Geräten der P-Serie sowie der verwandten Hermes-/WR3223-Baureihe
sind willkommen — insbesondere Datenpunkt-Zuordnungen anderer Ausbaustufen. Sie helfen,
gerätespezifische von baureihenweiten Eigenschaften zu trennen. Rückmeldungen und
Ergänzungen können über das
[Projekt-Repository](https://github.com/markusmauch/proxon-hesp) eingebracht werden.
