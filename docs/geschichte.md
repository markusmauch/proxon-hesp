# Reverse-Engineering-Geschichte

Der Weg zur Steuerung war lang und hatte eine große Sackgasse. Diese Seite hält die
wichtigsten Erkenntnisse fest — vor allem, damit andere die Sackgasse **nicht** wiederholen.

## Die Sackgasse: der 9600er Panel-Bus

Der naheliegende Ansatz — den Bus zwischen Bedienpanel und Gerät (9600 Baud)
mitschneiden und eigene Kommandos einspeisen — **funktioniert nicht**. Über Wochen
wurde belegt:

- Es existiert **kein diskretes Kommando-Telegramm**. Der Steuerzustand (Modus/Stufe)
  wird kontinuierlich in den regulären Poll-Frames des Masters *asserted*, nicht als
  einmaliges Kommando gesendet.
- Eingespeiste Frames kommen **byte-perfekt und dekodierbar** an der Geräteklemme an
  (mit Logic-Analyzer verifiziert) — und werden trotzdem **ignoriert**. Derselbe Frame
  vom echten Panel wird ab dem ersten Byte beantwortet.
- Getestet wurden mehrere Transceiver und Pegel (5-V-MAX485 sauber symmetrisch,
  RS485-HAT rail-to-rail asymmetrisch) — **alle abgelehnt**.

Das Gerät unterscheidet das echte Panel von jeder Injektion an einer **analogen
Eigenschaft der Wellenform** (Bias-Offset, Slew-Rate, Signaturform), die sich mit
Standard-Hardware nicht reproduzieren lässt. Frame-Replay über den Panel-Bus ist damit
eine Sackgasse. Das deckt sich mit dem Verhalten der verwandten WR3223-Familie.

!!! success "Der Ausweg"
    Der **interne 19200er HESP-Bus (X7)** hat diese Hürde nicht. Er ist der eigentliche
    Steuerbus zwischen PTC-Modul und Hauptplatine — und er akzeptiert Schreibzugriffe.

## Der Durchbruch: Lesen auf X7

Mit einem USB-RS485-Adapter ließ sich X7 **bidirektional byte-perfekt** lesen. Damit
fiel das komplette Frame-Format (Typ-Nibble, DP-ID little-endian, Nibble-Längenfeld,
float32-Daten) und das DP-Inventar.

## Der Durchbruch: die „ungeknackte CRC"

Die Prüfsumme galt in den Foren als ungelöst. Sie ist **keine** Standard-CRC-16, aber
**linear über GF(2)**: ein affines Modell sagt alle zurückgehaltenen Testframes korrekt
voraus. Details auf der [Protokoll-Seite](protokoll.md#die-prufsumme-geknackt).

## Der Durchbruch: erster Schreibvorgang

Ein aus einem echten Master-Frame extrahierter (und unabhängig per Prüfsummen-Modell
reproduzierter) SET-Frame für Lüfterstufe 2 wurde gesendet:

- **60 ms später** kam ein CONFIRM vom LPC1758 — eindeutig die Antwort auf unseren Frame.
- Beim zyklischen Halten sprang die **Ziel-Drehzahl** auf den Stufe-2-Wert, die
  **Ist-Drehzahl** stieg um etwa 300 rpm — der Lüfter drehte physisch hoch.
- Danach setzte der Master automatisch auf Stufe 1 zurück — **vollständig reversibel**,
  kein Gerätefehler.

## Was noch offen ist

- **T1 vs. T10** (die beiden Zuluftpfad-Fühler) sind erst im WP-Betrieb trennbar.
- Der genaue **Bypass-Status-Bit** im DigitalOut-Feld `0x0066` wird über eine
  Abend-Crossing-Messung bestätigt.
- **Reboot** wird nicht über X7 ausgelöst (kein Kommando-Frame gefunden).
- Ob der STM32 eine reine Frame-Brücke oder ein aggregierender Smart-Slave ist, ist
  nicht abschließend geklärt.

## Verwandte Quellen

- FHEM-Forum, Topic **96437** (User *mkneppe*): erste öffentliche Beschreibung des
  HESP-Telegrammaufbaus derselben Herstellerfamilie.
- Die diversen **Modbus**-Projekte für die neueren FWT-2.0-/T300-Geräte betreffen einen
  **anderen** Bus und helfen beim rohen HESP-Binärbus nicht.
