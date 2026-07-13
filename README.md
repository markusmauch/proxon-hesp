# PROXON P-Serie · HESP-Bus (inoffizielle Doku)

Reverse-Engineering-Dokumentation des internen RS485-Busses (**HESP**, 19200 Baud) der
Zimmermann **PROXON P-Serie** Lüftungsheizung: Protokoll, Datenpunkte und lokale
Home-Assistant-Integration — ohne das kostenpflichtige Hersteller-Gateway.

📖 **Website:** https://markusmauch.github.io/proxon-hesp/

## Inhalt

- **Anlage & Architektur** — Zwei-Platinen-Aufbau, Anschluss X7
- **HESP-Protokoll** — Frame-Format, Typ-Nibble, die geknackte Prüfsumme
- **Datenpunkt-Referenz** — Steuer- und Sensor-DPs
- **Betrieb mit Home Assistant** — Daemon, MQTT-Discovery, Fallstricke
- **Geschichte** — warum der 9600er-Panel-Bus eine Sackgasse ist
- **FAQ & offene Fragen**

## Lokal bauen

```bash
pip install mkdocs-material
mkdocs serve
```

## Mitmachen

Eigene Messungen von P-Serie-/Hermes-/WR3223-Geräten (besonders DP-Zuordnungen anderer
Ausbaustufen) sind als Issue oder PR willkommen.

## Haftungsausschluss

Inoffiziell, entstanden an einer einzelnen privaten Anlage, ohne Beteiligung von
Zimmermann. Nachbau auf eigenes Risiko; Eingriffe können Gewährleistung/Garantie
berühren. Es werden keine geschützten Hersteller-Unterlagen wiedergegeben.
