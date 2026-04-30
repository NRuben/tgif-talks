---
title: Disposable One-Time Code
sub_title: AI, lokale Simulation und mehr Sicherheit
author: Noah Ruben

theme:
  name: dark
  override:
    code:
      alignment: left
      margin:
        fixed: 0
---

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Das Migrationsproblem

<!-- new_lines: 3 -->

**Datenmigrationen sind unangenehm.**

Sie ändern nicht nur Code,

sondern auch **Daten, Zustände und Startverhalten**.

<!-- new_lines: 2 -->

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Basis: Lokale Simulation

<!-- new_lines: 3 -->

**Die komplette Umgebung lokal simulieren.**

Das heißt per Tilt die echten Ressourcen starten,

um so der realen Umgebung möglichst nahe zu kommen.

<!-- new_lines: 1 -->

In dem Fall ist das:

- MongoDB
- Postgres
- das Backend mit der Migrationslogik

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Erste Schritte
<!-- new_lines: 3 -->
Abzug der Datenbank per mongodump lokal ablegen.

Händisch per mongorestore in die lokale MongoDB bringen.

Backend starten und hoffen, dass die Migration durchläuft.
<!-- new_lines: 3 -->
Vergleichen und debuggen per Logs und sporadischen Datenbank-Checks.

<!-- new_lines: 3 -->
Das ist der typische Startpunkt, aber er ist fehleranfällig und wenig vertrauenswürdig.

Besser wäre es, diesen Ablauf zu automatisieren und kontrolliert durchfahren zu können.

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Robuste Skripte

<!-- new_lines: 3 -->

Für diesen Ablauf wollte ich nicht nur lose Skripte mit `bash`.

<!-- new_lines: 2 -->

Warum kein Bash:

- zu fragil bei Retries, Cleanup und Fehlerpfaden
- zu viel implizites Verhalten bei Exit Codes, Pipes und Quoting
- Assertions auf Logs, Zustände und Reports werden schnell unübersichtlich

<!-- new_lines: 2 -->

Python wäre möglich gewesen.

- aber ohne statische Typisierung
- und mit mehr Laufzeit- und Umgebungsfragen als nötig

<!-- new_lines: 2 -->

Go war hier der pragmatische Sweet Spot:

- statisch typisiert, direkt per `go test` ausführbar
- gut für Prozesse, Assertions und reproduzierbares Fehlschlagen.

Zudem ist go eine hervorragende Sprache für ai-generierten Code, da sie wenig Magie und viel explizite Struktur hat und noch compiliert werden muss, was Fehler schnell sichtbar macht.
<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Disposable One-Time Code

<!-- new_lines: 3 -->

AI nimmt mir einen großen Teil der Extra-Arbeit für solche Werkzeuge ab.

Dadurch kann ich **robuste Einmal-Skripte** sehr leicht erstellen und rechtfertigen.

<!-- new_lines: 2 -->

**Wenig Mehrarbeit.**

**Viel zusätzliche Sicherheit.**

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Deep Dive: was der Test Harness tut

<!-- new_lines: 3 -->

`go test`

- baut die Test-Binaries, damit sie schnell laufen
- gleicht ab, ob alle Voraussetzungen erfüllt sind (`dump`, `tilt`, `mongosh`, `psql`, etc.)
- startet Tilt-Ressourcen
- stellt echte Quelldaten in Mongo wieder her
- startet das Backend kontrolliert
- beobachtet Mongo, Postgres und Logs
- schreibt Reports und Snapshots
- prüft Happy Path und Retry-Verhalten

- kein Framework
- kein Produktivfeature
- ein gezieltes Einmal-Werkzeug mit hohem Nutzen

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 1, 1, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

### Fall 1: Clean Run

<!-- new_lines: 3 -->

1. Tilt startet Mongo und Postgres.
2. `mongodb-restore` bringt die Quelldaten hinein.
3. Das Backend startet und migriert.
4. Genau ein erfolgreicher Lauf wird sichtbar.
5. Ein weiterer Restart skippt die Migration sauber.

Wichtig:

- nicht nur Logs
- auch echte Zustände in Postgres und Mongo

<!-- column: 2 -->

### Fall 2: Interrupted + Retry

<!-- new_lines: 3 -->

1. Migration startet.
2. Das Backend wird absichtlich abgebrochen.
3. Der erste Versuch bleibt als `interrupted` sichtbar.
4. Das Backend startet erneut.
5. Der Retry setzt zurück und migriert sauber neu.
6. Weitere Restarts skippen wieder korrekt.

Das ist der spannendere Beweisfall :^)

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Live-Demo

<!-- new_lines: 3 -->

{}

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Warum Go hier gut (für mich und den Agenten) funktioniert

<!-- new_lines: 3 -->

- disposable wie ein Skript
- aber mit Typisierung
- gute Timeouts und Fehlerbehandlung
- gut für Prozesse, Logs und Dateien
- leicht ausführbar per `go test`
- leicht generierbar und anpassbar durch AI
- leicht verständlich und wartbar für Menschen
- gute Unterstützung für subprocesses und Assertions

Nicht als Dogma:

- Python ginge auch
- Rust ginge auch
- Kotlin ginge auch

Go ist hier einfach ein guter Sweet Spot.

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Exkurs: Go

<!-- new_lines: 3 -->

### Command- und Prozess-Orchestrierung

```go
cmd := exec.Command(
    "tilt", "up", "--stream=false",
    "mongodb", "postgres", "mongodb-restore",
)
```

- keine tiefe Abstraktionsschicht

<!-- column: 2 -->
<!-- new_lines: 5 -->

### Utilities statt Treiber

```go

mongoCmd := exec.Command("mongosh", "--quiet", "--eval", `JSON.stringify(...)`)
psqlCmd := exec.Command("psql", postgresConnString, "-X", "-A", "-t", "-c", query)
```

- leicht manuell reproduzierbar (auch vom Agenten)
- wenig bis kein Konfigurationsaufwand

Bewusst nicht: Mongo, Postgres per Go-Treiber ansteuern

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Artefakte statt Bauchgefühl

<!-- new_lines: 3 -->
<!-- font_size: 2 -->

Das Harness produziert mehr als nur Exit Codes:

- Markdown-Reports
- JSON-Snapshots
- Log-Auszüge
- sichtbare Zustände für `completed`, `interrupted`, `skip`

Damit bekommt man:

- prüfbare Ergebnisse
- besseres Debugging
- mehr Sicherheit bei riskanten Änderungen

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Migrationen vergleichen

<!-- new_lines: 3 -->
<!-- font_size: 2 -->

Das gleiche Harness wird zweimal ausgeführt:

1. vor einer Änderung, einem Rebase oder Refactoring
2. nach der Änderung
3. beide Läufe schreiben wieder Reports, JSON-Snapshots und Logs
4. die AI vergleicht diese Artefakte direkt

Worauf ich schaue:

- gleiche Endzustände statt nur "Build ist grün"
- gleiche Migrationsphasen und Retry-Signale
- gleiche Reports, abgesehen von Zeitstempeln

Frage an die AI:

`Vergleiche die beiden Läufe und finde Unterschiede. Normalisiere Zeitstempel und Cutoff-Zeitpunkte, damit sie nicht ins Gewicht fallen. Sind die Läufe ansonsten identisch oder gibt es echte Unterschiede im Verhalten oder in den Ergebnissen?`

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Beispiel: Verhaltensvergleich

<!-- new_lines: 3 -->
<!-- font_size: 1 -->

Vorher/Nachher:

- case-2 vor Änderung: `20260416T065751Z`
- case-2 nach Änderung: `20260416T122248Z`

Mongo nach Restore: identisch

- `forms_v2=42`
- `form_versions=42`
- `templates_v2=460`
- `linkcodes_v2=22`

Postgres nach Retry: identisch

- `forms=43`
- `templates=460`
- `codes=3`

data_migrations-Phasen: identisch

- `open=1`
- `after-interrupt=1`
- `after-final-restart=2`

Log- und Report-Signale: identisch

- retry mit `Resetting Postgres schema before retry.`
- danach `completed`
- finaler Restart wieder `skipped`

Wichtig:

- Timestamps und Cutoff-Zeitpunkte muss man normalisieren
- danach bleibt die eigentlich interessante Frage übrig:
  gleiches Verhalten oder echte Regression?

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Sneak Peek: Der Nachfolger

<!-- new_lines: 3 -->

Nächster TGIF:

`Warum mich JPA`/`Hibernate` nervt und warum `Exposed` angenehmer ist

Worum es dort geht:

- weniger Magie, mehr Kontrolle, näher an SQL
- sichtbare Queries statt implizitem ORM-Verhalten
- `DSL` vs `DAO`
- Cache und Transaktion ohne Session-Mystik
- echte Umstiegs- und Migrationserfahrung

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

## Takeaways

<!-- new_lines: 3 -->

**Nutzt AI für disposable one-time code und einfache Tests.**

<!-- new_lines: 2 -->

Nutzt die lokale Simulation, um realistische Tests und Vergleiche zu ermöglichen.
So oft wir möglich, damit die Feedbackschleife kurz bleibt.

<!-- new_lines: 2 -->

Lasst AI auch **Reports und Snapshots vergleichen.**

<!-- end_slide -->

<!-- new_lines: 15 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

# Danke für eure Aufmerksamkeit!

<!-- new_lines: 5 -->

Gibt es Fragen, Anmerkungen oder Diskussionen?
