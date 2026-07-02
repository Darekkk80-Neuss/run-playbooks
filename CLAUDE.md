# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projektüberblick

"Die üblichen Verdächtigen – SDM Dashboard" ist eine ITSM-Webanwendung (IT Service Management) mit vier Funktionen: Playbooks und Runbooks für 30 IT-Notfallszenarien anzeigen/exportieren, sowie zwei Dokument-Generatoren (Stellungnahme und Management Summary) mit Pflichtfeld-Formularen und PDF-Export. Die UI und alle Inhalte sind auf Deutsch.

**Die gesamte Anwendung ist eine einzige Datei: `index.html`** (~2160 Zeilen). Kein Build-System, keine Dependencies, keine Tests, kein Framework — reines Vanilla-JavaScript (ES5-Stil, IIFE), CSS und HTML in einer Datei.

## Entwicklung & Deployment

- Ausführen: `index.html` direkt im Browser öffnen (kein Server nötig; `localStorage` funktioniert auch über `file://`).
- Für die Claude-Code-Browser-Vorschau: `.claude/launch.json` startet `.claude/server.ps1` (PowerShell-HttpListener auf Port 8321) — auf diesem Rechner sind weder Python noch Node installiert.
- Es gibt keine Build-, Lint- oder Test-Befehle. Verifikation läuft manuell über die Browser-Vorschau.
- **Deployment: GitHub Pages** (Branch `master`, Root) → https://darekkk80-neuss.github.io/run-playbooks/ — ein Push auf `master` genügt; falls der Build nicht anspringt: `gh api -X POST repos/Darekkk80-Neuss/run-playbooks/pages/builds`.

## Aufbau von index.html

Die Datei ist in dieser Reihenfolge organisiert:

1. **`<style>`** — Design-Tokens als CSS-Variablen in `:root` (dunkles Theme, Akzent-Gradient), danach Abschnitte pro Screen. Enthält `@media print`-Regeln für den PDF-Export: sie blenden alles außer `.doc-page` aus.
2. **HTML-Body** — sechs Screens als `<div class="screen" id="screen-*">`: `login`, `home`, `list`, `detail`, `summary` (Stellungnahme-Generator), `msummary` (Management-Summary-Generator). Es ist immer genau einer per `.active` sichtbar (Navigation über `showScreen(name)`).
3. **`<script>`** (eine IIFE) mit klar kommentierten Blöcken: AUTH, SCREEN NAV, DATA, STATE, HOME, MODE TOGGLE, CATEGORY TABS, CARD GRID, DETAIL RENDER, STELLUNGNAHMEN-GENERATOR, MANAGEMENT-SUMMARY-GENERATOR, INIT.

## Zentrale Konzepte

### Datenmodell (im Script-Block, ab "DATA")
- `CATS` — 5 Kategorien (`ir`, `bk`, `op`, `sec`, `ran`) mit Label und Farbe.
- `ESCALATION` — eine Eskalationsmatrix pro Kategorie (nicht pro Szenario).
- `SCENARIOS` — Array mit 30 Szenarien. Jedes hat `id`, `cat`, `sev`, `title`, `trigger`, `note` sowie Schritt-Tabellen `immediate`, `diagnosis`, `resolution`, `recovery` (Spalten: step/description/responsible/duration) und `communication` (audience/channel/message).
- Der Helper `R(cols, tuples)` wandelt kompakte Arrays in Objekt-Zeilen um — neue Szenariodaten immer über `R(COLS4, ...)` bzw. `R(COLS3C, ...)` anlegen.

### Startseite (home)
Vier `.type-card`-Karten in `.type-col`-Wrappern; unter jeder Karte ein `.type-note`-Kästchen (weißer Rahmen/Text) mit einer Definition. `data-mode` steuert das Klickziel: `playbook`/`runbook` → Liste, `summary`/`msummary` → jeweiliger Generator-Screen.

### Playbook vs. Runbook
Beide Modi nutzen dieselben `SCENARIOS`-Daten. `currentMode` steuert nur die Darstellung in `renderStepTable()`: Runbook = detaillierte Tabellen (mit Beschreibung/Befehl und Dauer), Playbook = reduzierte Tabellen (nur Schritt und Verantwortlich).

### Dokument-Generatoren (Stellungnahme & Management Summary)
Beide folgen demselben Muster (Präfix `st*` bzw. `ms*` für Feld-IDs, Buttons und Funktionen):
- Formular-Panel (`.st-panel`) und Ergebnis-Wrapper liegen im selben Screen; nach dem Generieren wird das Panel per `style.display` versteckt und das Dokument in einen `.doc-page`-Container gerendert.
- Validierung: `ST_REQUIRED`/`MS_REQUIRED` (Array von Feld-IDs) → fehlende Felder bekommen `.invalid`. Beim Stellungnahme-Generator ist nur Argument 1 Pflicht; die optionalen Argumente 2/3 sind "alles oder nichts" (`ST_OPTIONAL_GROUPS`): sobald ein Feld einer Gruppe befüllt ist, werden alle drei verlangt.
- Die erzeugten Texte sind Fließtext in Management-Sprache — Nutzereingaben (ganze Sätze) werden über Doppelpunkt-Konstruktionen eingebettet ("… hervorzuheben: {Eingabe}"), weil deutsche Nebensätze (nach "dass") die Verbstellung der Eingabe brechen würden. `endDot()` stellt Satzschlusszeichen sicher.
- Management Summary: Kennzahlen-Textarea wird zeilenweise geparst (`Kennzahl: Wert` → Tabellenzeile); der Risiken-Abschnitt erscheint nur wenn befüllt, die Abschnittsnummerierung passt sich an.

### Auth
Rein clientseitige Zugriffssperre (bewusst kein echter Schutz, siehe Hinweis auf dem Login-Screen): hartkodierte `CREDENTIALS`-Liste, Login-Status in `localStorage` unter dem Key `irb_auth`. Neue Screens brauchen: `user-chip`-Span, Logout-Button (in die ID-Liste beim Logout-Handler eintragen) und ggf. Home-Button.

### Rendering & Export
- Alle Ansichten werden per String-Konkatenation in `innerHTML` gerendert; dynamische Inhalte immer durch `esc()` escapen.
- `.doc-page` (weißes Dokument-Layout) wird von drei Containern geteilt: `#doc` (Detail), `#stDoc` (Stellungnahme), `#msDoc` (Management Summary). Styles dafür nur über die Klasse ergänzen, nicht über IDs.
- HTML-Export (`btnExportHtml`, nur Detail-Screen) baut ein eigenständiges Dokument: nimmt den `#doc`-Inhalt plus die Styles **vor** dem `@media print`-Abschnitt (`split('@media print')[0]`) — die Position dieses Abschnitts im Stylesheet ist daher relevant.
- PDF-Export läuft überall über `window.print()` und die `@media print`-Regeln.

## Konventionen

- Sprache: UI-Texte, Datensätze und Code-Kommentare auf Deutsch; Variablen-/Funktionsnamen auf Englisch.
- JavaScript im bestehenden ES5-Stil halten (`var`, `function`-Ausdrücke, keine Template-Literals) — die Datei ist konsistent so geschrieben.
- Commit-Messages: kurze imperative Betreffzeile; Englisch und Deutsch kommen in der Historie vor.
