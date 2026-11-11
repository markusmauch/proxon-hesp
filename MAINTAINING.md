# Pflege dieser Dokumentation

Diese Datei liegt bewusst im Repo-Root (nicht in `docs/`) und erscheint daher **nicht**
auf der Website. Sie hält fest, wie die Dokumentation gepflegt wird, damit Ton, Umfang
und Struktur über die Zeit konsistent bleiben.

## Grundsatz

Die öffentliche Doku (`markusmauch/proxon-hesp`, GitHub Pages) wird mit **jedem
gesicherten Erkenntnisgewinn** aus der laufenden Untersuchung aktualisiert. Die
Detail-/Laborarbeit findet in einem separaten, privaten Repository statt; hierher fließt
nur der **verifizierte, allgemein verwertbare** Stand.

Aktualisiert wird, wenn eine Erkenntnis **gesichert** ist (reproduziert / am Bus
bestätigt) — nicht bei bloßen Hypothesen. Bestätigt sich ein bislang offener Punkt, wird
er aus „Randbedingungen & offene Punkte" in die zuständige Sachseite übernommen.

## Ton

Sachlich-neutral, formal, wie ein technischer Messbericht. Implementierungsneutral: die
Schnittstelle wird beschrieben, keine bestimmte Anwendung vorgeschrieben.

- **Vermeiden:** geknackt, knacken, Durchbruch, Sackgasse, Angriff/Angriffspunkt,
  Injektion, Hacker, Hebel, bespielen, „Tap".
- **Stattdessen:** bestimmt, reproduzierbar, nachgewiesen, modelliert/rekonstruiert,
  mitlesen/Abgriff, Einspeisung/aktives Senden, Ansatzpunkt/Zugang.
- Beobachtungen als Beobachtungen formulieren, nicht als Handlungsanweisung.

## Umfang — was NICHT veröffentlicht wird

- Interne IP-Adressen, Hostnamen, Netz-/Infrastrukturdetails.
- Geräte- oder Adapter-Seriennummern, konkrete Gerätepfade.
- Zugangsdaten, Tokens, Passwörter jeglicher Art.
- Die urheberrechtlich geschützte Hersteller-Bedienungsanleitung (nur auf das Original
  beim Hersteller verweisen).
- Plattformspezifische Integrationsanleitungen (die Doku bleibt implementierungsneutral).
- Der Geltungsbereichs-Hinweis „Messungen an einer Anlage" bleibt erhalten.

## Struktur (Navigation) — stabil halten

1. Einführung — `index.md`
2. Anlage & Architektur — `anlage.md`
3. HESP-Protokoll — `protokoll.md`
4. Datenpunkt-Referenz — `dp-referenz.md`
5. Randbedingungen & offene Punkte — `offene-punkte.md`

Buy-Me-a-Coffee-Button und Favicon über Releases hinweg **unverändert** lassen.

`overrides/main.html` (`theme.custom_dir`) enthält den Google-Site-Verification-Meta-Tag.
Er beweist Google den Besitz der Property und muss auf **allen** Seiten stehen bleiben —
Datei und `custom_dir`-Eintrag nicht entfernen, sonst verliert die Search Console den Zugriff.

## Zweisprachigkeit (seit 2026-11-11)

Die Seiten erscheinen deutsch unter `/` und englisch unter `/en/`. Technisch über
`mkdocs-static-i18n` im **Suffix-Verfahren**: Zu jeder `docs/<name>.md` gehört eine
`docs/<name>.en.md`. Deutsch ist die Quellsprache.

- **Jede inhaltliche Änderung muss in beide Fassungen.** Eine Seite, die nur deutsch
  aktualisiert wird, verbreitet den alten Stand unter `/en/` weiter — schlimmer als gar
  keine Übersetzung.
- **Hex-Tabellen und Zahlenwerte nicht abtippen**, sondern aus der deutschen Datei
  übernehmen. Gegenprobe:
  `python3 -c "import re;f=lambda p:re.findall(r'\`([0-9a-f]{4})\`',open(p).read());print(f('docs/protokoll.md')==f('docs/protokoll.en.md'))"`
- **Verweise mit Anker** zeigen auf den deutschen Dateinamen (`protokoll.md#...`), das
  Plugin löst die Sprache auf. Der **Anker** selbst muss aber der jeweiligen Sprache
  folgen (`#prufsumme` ↔ `#checksum`) — sonst zeigt der Link ins Leere, ohne dass der
  strikte Build das merkt.
- **Vokabelprüfung auch englisch** fahren (siehe Ablauf unten).
- `theme.features: navigation.instant` ist **bewusst entfernt**: Es ist mit dem
  Sprachumschalter unvereinbar und ließ `mkdocs build --strict` abbrechen. Nicht
  zurücknehmen, ohne den Umschalter zu prüfen.
- Die Action installiert `mkdocs-material` **und** `mkdocs-static-i18n` — bei einem neuen
  Plugin die pip-Zeile im Workflow mitziehen.
- **Die README ist zweisprachig, und hier ist ENGLISCH die Vorderseite:** `README.md` ist
  die englische Fassung (GitHub zeigt nur diese Datei an), die deutsche liegt in
  `README.de.md`. Das ist die **Umkehrung** der Website, wo Deutsch unter `/` bleibt —
  nicht aus Versehen, sondern weil die Repo-Startseite ein internationales Publikum hat.
  Verwiesen wird über eine Sprachzeile unter der Überschrift, nicht über das Plugin.
  Beide liegen außerhalb von `docs/` und werden vom strikten Build **nicht** geprüft —
  Änderungen dort von Hand in beide Fassungen und die Vokabelprüfung auf beide Dateien
  laufen lassen.

## Änderungs-Ablauf

```bash
# 0. IMMER ZUERST: Remote-Stand holen — Markus editiert gelegentlich direkt auf
#    GitHub (Web-Editor). Niemals force-pushen; bei Konflikt seine Version sichten
#    und einarbeiten statt überschreiben.
git pull --rebase origin main
# 1. Inhalt in docs/ anpassen
# 2. Schutzprüfung: keine internen Daten / kein wertendes Vokabular
grep -rniE '192\.168|10\.32|hejira|/dev/serial|passwd|token|geknackt|knacken|durchbruch|sackgasse|angriff|injekt|hacker|\btap\b|hebel|bespielen' docs/ && echo 'PRUEFEN!' || echo 'sauber'
grep -rniE '\bcrack|breakthrough|dead end|\battack|\binject|\bhack|\btap\b|leverage|exploit' docs/*.en.md && echo 'PRUEFEN!' || echo 'sauber'
# 3. Strikt bauen
pip install mkdocs-material && mkdocs build --strict
# 4. committen + auf main pushen -> GitHub Action baut & deployt
# 5. Live prüfen: https://markusmauch.github.io/proxon-hesp/
#    Geduld: der Lauf kann einige Minuten auf sich warten lassen (2026-09-13: gut 4 min,
#    Tag erst nach ~5 min live). In dem Fenster liefert ein manueller `gh workflow run`
#    HTTP 500 und die Run-Liste bleibt leer — das ist NICHT ein kaputtes Repo, also
#    weder Actions-Einstellungen umstellen noch erneut pushen, sondern nachsehen mit:
#    gh run list --limit 1
```
