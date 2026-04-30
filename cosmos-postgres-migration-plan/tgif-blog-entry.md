---
title: Disposable One-Time Code
subtitle: Wie KI, lokale Simulation und robuste Einmalwerkzeuge Datenmigrationen sicherer machen
author: Noah Ruben
source: TGIF/tgif-talk.md
tags:
  - migration
  - postgres
  - mongodb
  - ki
  - testing
---

# Disposable One-Time Code

Datenmigrationen sind unangenehm. Sie ändern nicht nur Code, sondern auch Daten, Zustände und das Startverhalten eines Systems. Genau deshalb reicht es nicht, nur darauf zu schauen, ob der Build grün ist oder ob im Log am Ende irgendwo `completed` steht.

Bei einer Migration von einer MongoDB-Quelle nach Postgres wollte ich mehr Sicherheit haben: eine möglichst realistische lokale Umgebung, reproduzierbare Abläufe und überprüfbare Artefakte. Daraus ist ein kleines, bewusst wegwerfbares Test-Harness entstanden. Kein Framework, kein Produktivfeature, sondern ein robustes Einmalwerkzeug mit hohem Nutzen.

## Das Migrationsproblem

Eine Datenmigration ist riskanter als viele normale Codeänderungen. Sie verändert Bestandsdaten, muss mit Zwischenzuständen umgehen und beeinflusst, ob ein Service überhaupt bereit ist, Traffic anzunehmen. Wenn während der Migration etwas abbricht, ist außerdem nicht nur die Frage wichtig, ob der nächste Start wieder funktioniert. Entscheidend ist auch, ob der neue Lauf sauber zurücksetzt, erneut kopiert und am Ende denselben Zielzustand erreicht.

Der typische erste Ansatz ist schnell gebaut:

1. Einen Datenbankabzug per `mongodump` lokal ablegen.
2. Die Daten per `mongorestore` händisch in die lokale MongoDB einspielen.
3. Das Backend starten und hoffen, dass die Migration durchläuft.
4. Logs lesen und sporadisch Datenbanken prüfen.

Das ist ein brauchbarer Startpunkt, aber fehleranfällig und wenig vertrauenswürdig. Man bekommt schnell ein Gefühl dafür, ob etwas grundsätzlich funktioniert. Man bekommt aber noch keinen belastbaren Nachweis, dass Clean Run, Abbruch, Retry und Skip-Verhalten wirklich korrekt sind.

## Lokale Simulation als Basis

Der wichtigste Schritt war, die komplette Umgebung lokal zu simulieren. In meinem Fall heißt das: per Tilt echte Ressourcen starten und damit so nah wie möglich an der späteren Umgebung bleiben.

Konkret laufen lokal:

- MongoDB als Quelle
- Postgres als Ziel
- das Backend mit der Migrationslogik

Das ist bewusst keine isolierte Unit-Test-Welt. Die Migration soll genau die Werkzeuge, Prozesse und Startpfade nutzen, die später auch relevant sind. Dadurch werden Fehler sichtbar, die in einem rein gemockten Setup leicht verschwinden würden: fehlende Tools, falsche Connection Strings, Timing-Probleme, unklare Startreihenfolgen oder kaputte Annahmen über persistente Zustände.

## Warum robuste Skripte wichtig sind

Für diesen Ablauf wollte ich nicht nur lose Bash-Skripte schreiben. Bash ist für kleine Hilfsbefehle gut, wird bei diesem Problem aber schnell fragil:

- Retries, Cleanup und Fehlerpfade werden unübersichtlich.
- Exit Codes, Pipes und Quoting haben viel implizites Verhalten.
- Assertions auf Logs, Datenbankzustände und Reports lassen sich schwer sauber strukturieren.

Python wäre möglich gewesen. Für diesen Zweck wollte ich aber statische Typisierung, einfache Prozess-Orchestrierung und einen Test-Runner, der ohne viel Zusatzstruktur direkt verwendbar ist.

Go war hier der pragmatische Sweet Spot:

- statisch typisiert
- direkt per `go test` ausführbar
- gut für Prozesse, Timeouts, Assertions und Dateien
- explizit genug, damit auch KI-generierter Code schnell prüfbar bleibt
- kompiliert, wodurch viele Fehler früh sichtbar werden

Der Punkt ist nicht, dass Go die einzig richtige Antwort ist. Python, Rust oder Kotlin könnten je nach Team und Umfeld ebenfalls passen. Go war für diesen konkreten Fall einfach klein genug für ein Einmalwerkzeug und robust genug für echte Sicherheit.

## Disposable One-Time Code

Der eigentliche Hebel ist KI. Früher hätte ich mir für so einen Ablauf zweimal überlegt, ob sich ein robustes Einmalwerkzeug lohnt. Es kostet Zeit, saubere Prozesslogik, Assertions, Reports und Fehlerbehandlung zu bauen.

Mit KI verschiebt sich diese Rechnung. Die zusätzliche Arbeit wird deutlich kleiner, während der Sicherheitsgewinn groß bleibt. Dadurch kann man Werkzeuge bauen, die nur für eine konkrete Migration, einen Rebase, einen Refactoring-Vergleich oder eine einmalige Produktionsvorbereitung existieren.

Das ist für mich Disposable One-Time Code:

- Code, der absichtlich kein langlebiges Produktivfeature ist
- Code, der trotzdem robust, lesbar und überprüfbar sein darf
- Code, der konkrete Risiken reduziert
- Code, der nach seinem Zweck auch wieder verschwinden kann

Wegwerfbar heißt dabei nicht schlampig. Gerade weil das Werkzeug nur einen Zweck hat, kann es sehr direkt und explizit sein.

## Was das Test-Harness tut

Das Harness läuft über `go test` und orchestriert die Migration lokal Ende-zu-Ende. Es prüft zuerst, ob alle Voraussetzungen vorhanden sind, zum Beispiel `dump`, `tilt`, `mongosh` und `psql`. Danach startet es die nötigen Tilt-Ressourcen, stellt die Quelldaten in MongoDB wieder her, startet das Backend kontrolliert und beobachtet MongoDB, Postgres und Logs.

Am Ende entstehen nicht nur Exit Codes, sondern konkrete Artefakte:

- Markdown-Reports
- JSON-Snapshots
- Log-Auszüge
- sichtbare Zustände für `completed`, `interrupted` und `skipped`

Diese Artefakte sind entscheidend. Sie machen aus einem Gefühl eine prüfbare Aussage. Statt "sieht gut aus" kann ich nachvollziehen, welche Datenstände erreicht wurden, welche Migrationsphasen sichtbar waren und ob ein Retry wirklich denselben Zielzustand erzeugt.

## Zwei Beweisfälle

Der erste Fall ist der Clean Run:

1. Tilt startet MongoDB und Postgres.
2. `mongodb-restore` bringt die Quelldaten hinein.
3. Das Backend startet und migriert.
4. Genau ein erfolgreicher Lauf wird sichtbar.
5. Ein weiterer Restart überspringt die Migration sauber.

Wichtig ist dabei, nicht nur Logs zu prüfen. Das Harness schaut auch auf echte Zustände in Postgres und MongoDB.

Der zweite Fall ist spannender: Interrupted + Retry.

1. Die Migration startet.
2. Das Backend wird absichtlich abgebrochen.
3. Der erste Versuch bleibt als `interrupted` sichtbar.
4. Das Backend startet erneut.
5. Der Retry setzt zurück und migriert sauber neu.
6. Weitere Restarts überspringen die Migration wieder korrekt.

Dieser zweite Fall beweist den für mich wichtigsten Operator-Pfad: Ein unterbrochener Lauf darf nicht in einem halb kaputten Zielzustand hängen bleiben. Der nächste Start muss kontrolliert zurücksetzen, neu kopieren und am Ende wieder einen stabilen Zustand erreichen.

## Go als pragmatisches Werkzeug

Go funktioniert in diesem Szenario gut, weil es wenig Magie erzwingt. Prozess-Orchestrierung bleibt explizit:

```go
cmd := exec.Command(
    "tilt", "up", "--stream=false",
    "mongodb", "postgres", "mongodb-restore",
)
```

Auch für Datenbankabfragen nutzt das Harness bewusst vorhandene Kommandozeilenwerkzeuge statt Go-Treiber:

```go
mongoCmd := exec.Command("mongosh", "--quiet", "--eval", `JSON.stringify(...)`)
psqlCmd := exec.Command("psql", postgresConnString, "-X", "-A", "-t", "-c", query)
```

Das ist kein Zufall. `mongosh` und `psql` sind leicht manuell reproduzierbar, auch durch einen Agenten. Es gibt wenig Konfigurationsaufwand, und die ausgeführten Befehle bleiben nah an dem, was man beim Debuggen ohnehin verwenden würde.

Bewusst nicht gebaut habe ich eine tiefe Abstraktionsschicht über MongoDB und Postgres. Das Harness soll Prozesse orchestrieren, Zustände prüfen und aussagekräftig fehlschlagen. Mehr braucht es nicht.

## Artefakte statt Bauchgefühl

Ein wichtiger Nebeneffekt der Reports und Snapshots ist, dass sie Vergleiche ermöglichen. Das gleiche Harness kann zweimal laufen:

1. vor einer Änderung, einem Rebase oder einem Refactoring
2. nach der Änderung
3. beide Läufe schreiben Reports, JSON-Snapshots und Logs
4. die KI vergleicht diese Artefakte direkt

Die interessante Frage ist dann nicht mehr nur: "Ist der Build grün?" Die bessere Frage lautet: "Sind die Endzustände und Migrationssignale identisch, abgesehen von Zeitstempeln und erwartbaren Laufzeitunterschieden?"

Ein typischer Auftrag an die KI sieht dann so aus:

```text
Vergleiche die beiden Läufe und finde Unterschiede.
Normalisiere Zeitstempel und Cut-off-Zeitpunkte, damit sie nicht ins Gewicht fallen.
Sind die Läufe ansonsten identisch oder gibt es echte Unterschiede im Verhalten oder in den Ergebnissen?
```

Damit wird die KI nicht nur zum Codegenerator, sondern auch zum Reviewer von Laufartefakten.

## Beispiel: Verhaltensvergleich

Ein konkreter Vorher-Nachher-Vergleich sah so aus:

- case-2 vor Änderung: `20260416T065751Z`
- case-2 nach Änderung: `20260416T122248Z`

MongoDB nach Restore war identisch:

- `forms_v2=42`
- `form_versions=42`
- `templates_v2=460`
- `linkcodes_v2=22`

Postgres nach Retry war ebenfalls identisch:

- `forms=43`
- `templates=460`
- `codes=3`

Auch die `data_migrations`-Phasen passten:

- `open=1`
- `after-interrupt=1`
- `after-final-restart=2`

Die Log- und Report-Signale waren ebenfalls gleich:

- Retry mit `Resetting Postgres schema before retry.`
- danach `completed`
- finaler Restart wieder `skipped`

Zeitstempel und Cut-off-Zeitpunkte muss man für so einen Vergleich normalisieren. Danach bleibt aber genau die relevante Frage übrig: gleiches Verhalten oder echte Regression?

## Ausblick: weniger ORM-Magie

Der nächste naheliegende Themenblock ist der Umgang mit Persistenzschichten selbst: warum mich JPA und Hibernate in solchen Migrationskontexten oft nerven und warum Exposed angenehmer sein kann.

Dort geht es dann um:

- weniger Magie und mehr Kontrolle
- sichtbare Queries statt implizitem ORM-Verhalten
- DSL vs. DAO
- Cache und Transaktionen ohne Session-Mystik
- echte Umstiegs- und Migrationserfahrung

Das hängt direkt mit dem gleichen Grundmotiv zusammen: Bei riskanten Änderungen will ich Verhalten sehen, verstehen und vergleichen können.

## Takeaways

Nutzt KI für Disposable One-Time Code und einfache, robuste Tests. Gerade bei Migrationen kann sich ein kleines Werkzeug lohnen, das nur für einen konkreten Zweck gebaut wird.

Simuliert die Umgebung lokal so realistisch wie möglich, damit die Feedbackschleife kurz bleibt und echte Probleme früh sichtbar werden.

Lasst KI nicht nur Code schreiben, sondern auch Reports, Snapshots und Logs vergleichen. Die Kombination aus lokaler Simulation, robustem Einmalwerkzeug und prüfbaren Artefakten macht riskante Änderungen deutlich kontrollierbarer.
