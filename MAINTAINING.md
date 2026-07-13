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

## Änderungs-Ablauf

```bash
# 1. Inhalt in docs/ anpassen
# 2. Schutzprüfung: keine internen Daten / kein wertendes Vokabular
grep -rniE '192\.168|10\.32|hejira|/dev/serial|passwd|token|geknackt|knacken|durchbruch|sackgasse|angriff|injekt|hacker|\btap\b|hebel|bespielen' docs/ && echo 'PRUEFEN!' || echo 'sauber'
# 3. Strikt bauen
pip install mkdocs-material && mkdocs build --strict
# 4. committen + auf main pushen -> GitHub Action baut & deployt
# 5. Live prüfen: https://markusmauch.github.io/proxon-hesp/
```
