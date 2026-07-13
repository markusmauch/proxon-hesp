# PROXON P-Serie · HESP-Bus (technische Dokumentation)

Technische Dokumentation des internen RS485-Busses (**HESP**, 19200 Baud) der Zimmermann
**PROXON P-Serie** Lüftungsheizung: physische Schnittstelle, Telegrammformat und
Datenpunkte als Grundlage für eigene lokale Auslese- und Steuerimplementierungen. Die
Dokumentation ist bewusst implementierungsneutral.

📖 **Website:** https://markusmauch.github.io/proxon-hesp/

## Inhalt

- **Anlage & Architektur** — Zwei-Platinen-Aufbau, Anschluss X7
- **HESP-Protokoll** — Frame-Format, Telegrammtypen, Prüfsummenmodell
- **Datenpunkt-Referenz** — Steuer- und Sensor-Datenpunkte
- **Randbedingungen & offene Punkte** — Bus-Eigenschaften und noch nicht geklärte Fragen

## Lokal bauen

```bash
pip install mkdocs-material
mkdocs serve
```

## Beiträge

Messungen von weiteren Geräten der P-Serie sowie der verwandten Hermes-/WR3223-Baureihe
(insbesondere Datenpunkt-Zuordnungen anderer Ausbaustufen) sind als Issue oder Pull
Request willkommen.

## Geltungsbereich

Alle Angaben stammen aus Messungen an einer einzelnen Anlage, erhoben ohne Beteiligung
des Herstellers. Nachbau auf eigene Verantwortung; Eingriffe können Gewährleistung oder
Garantie berühren. Es werden keine geschützten Hersteller-Unterlagen wiedergegeben.
