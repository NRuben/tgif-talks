# TGIF Talk: Warum mich JPA/Hibernate nervt und warum Exposed angenehmer ist

## Ziel des Talks

- Gemeinsamen Entwicklerfrust mit `JPA` und `Hibernate` aufgreifen.
- Zeigen, warum sich `Exposed` fuer viele Teams direkter, expliziter und besser debugbar anfuehlt.
- Kolleg:innen nach dem Talk in die Lage versetzen, `Exposed` praktisch einzuordnen und einen Einstieg zu finden.

## Tonalitaet

- Einstieg: pointiert, leicht provokant, mit ein paar Spitzen gegen `Hibernate`.
- Hauptteil: sachlich, technisch belastbar, konkret.
- Fazit: pragmatisch statt missionarisch.

## Kernthese

`Hibernate` versucht sehr viel Magie zu verstecken. Genau diese Magie macht den Alltag oft schwerer.

`Exposed` ist nicht "komfortabler" im klassischen ORM-Sinn, aber deutlich angenehmer, wenn man Kontrolle, Sichtbarkeit und Vorhersagbarkeit will.

## Grober Aufbau

### 1. Titel / Hook

Arbeitstitel:

`Warum mich JPA/Hibernate nervt und warum Exposed angenehmer ist`

Moegliche Unterzeile:

`Weniger Magie, mehr Kontrolle, naeher an SQL`

### 2. Cold Open: Was mich an JPA/Hibernate nervt

Ziel:
Den Raum sofort abholen, ohne in billiges Framework-Bashing abzurutschen.

Moegliche Punkte:

- `Optimistic Locking` auf grossen Entities oder breiten Aggregaten fuehlt sich schnell teuer und fragil an.
- `JPA`-Methoden, Query-Varianten und Repository-Konventionen sind oft nicht eindeutig genug: Man liest Code und weiss nicht sofort, welche Query wirklich entsteht.
- Projektionen sind moeglich, aber haeufig inkonsistent im Stil und selten wirklich angenehm.
- Wenn die Abfrage komplex genug wird, schreibt man am Ende sowieso wieder `SQL` oder `native queries`.
- `Lazy Loading`, `Flush`, Entity-State und Session-Kontext fuehren zu Nebenwirkungen, die nicht direkt im Code stehen.
- Debugging fuehlt sich haeufig an wie: "Was hat das ORM diesmal in meinem Namen entschieden?"

Moegliche pointierte Formulierung:

`Ich will eine Query bauen und keinen Verhandlungsprozess mit dem Persistence Context fuehren.`

### 3. Die eigentliche Beschwerde

Wichtiger Uebergang:

Nicht die Datenbank ist das Problem.  
Nicht einmal jede ORM-Idee ist das Problem.  
Das Problem ist die Menge an implizitem Verhalten.

Kernaussagen:

- Zu viel versteckter Zustand.
- Zu viele Nebenwirkungen ausserhalb des sichtbaren Codes.
- Zu wenig direkte Verbindung zwischen Kotlin/Java-Code und dem SQL, das wirklich laeuft.

### 4. Exposed als Gegenmodell

Botschaft:

`Exposed` nimmt dir weniger ab, aber dafuer weiss man haeufig besser, was gerade passiert.

Punkte:

- naeher an `SQL`
- expliziter
- Kotlin-nah
- besser zu begruenden
- einfacher zu debuggen

### 5. Deep Dive 1: Das mentale Modell von Exposed

Was das Publikum frueh verstehen sollte:

- `DSL` ist meistens der Default.
- `DAO` existiert, ist aber kein allgemeiner Ersatz fuer alles.
- Transaktionen sind zentral.
- `Exposed` versucht nicht, relationale Komplexitaet vollstaendig wegzuabstrahieren.

Leitfrage:

`Was bekomme ich an Klarheit zurueck, wenn ich ein bisschen mehr Explizitheit akzeptiere?`

### 6. Deep Dive 2: DSL vs DAO

Hier kann das vorhandene Material `exposed-dsl-vs-dao.md` eingebunden werden.

Kernaussagen:

- Fuer die meisten Projekte: `DSL` als Standard.
- `DAO` nur gezielt, wenn das Modell simpel ist und entity-zentrierter Zugriff wirklich hilft.
- Bei komplexeren Queries, Bulk-Operationen, Reporting oder `R2DBC`: klarer Vorteil fuer `DSL`.

Empfohlene Pointe:

`Wenn ich sowieso ueber Queries, Joins und Datenfluesse nachdenke, kann ich auch ein API benutzen, das das offen zugibt.`

### 7. Case Study Teil 1: Die Umstellung eines echten Projekts

Ziel:
Die Theorie mit echter Erfahrung aufladen.

Moegliche Struktur:

- Ausgangslage im alten Stack
- Was konkret wehgetan hat
- Warum ein Wechsel ueberhaupt diskutiert wurde
- Welche Hoffnungen realistisch waren und welche nicht

Wichtig:
Nicht behaupten, dass `Exposed` alles loest. Stattdessen zeigen, welche Art Probleme sich verbessert hat.

### 8. Deep Dive 3: Was Exposed angenehmer macht

Moegliche Punkte:

- Queries bleiben lesbarer und lokaler.
- Der Bezug zum resultierenden `SQL` ist enger.
- Weniger versteckte "Lebenszyklus-Magie".
- Weniger Ueberraschung bei Abfrageverhalten und Seiteneffekten.
- Kotlin-Code fuehlt sich eher wie bewusst geschriebener Datenzugriff an statt wie das Fuettern eines ORM-Zustandsautomaten.

### 9. Deep Dive 4: Cache, Flush und Transaktionsverhalten

Hier kann das vorhandene Material `exposed-cache-explained.md` eingebunden werden.

Kernaussagen:

- `Exposed` hat Cache-Verhalten, aber anders als viele aus `Hibernate` gewohnt sind.
- Der relevante DAO-Cache ist transaktionsbezogen, nicht "globale Black Magic".
- Man sollte `EntityCache` als Identitaets- und Write-Behind-Mechanismus verstehen, nicht als mystischen Performance-Topf.
- Gerade fuer Umsteiger ist wichtig, was `Exposed` bewusst nicht an Session-/Proxy-/Detach-Semantik nachbaut.

### 10. Case Study Teil 2: Was in der Migration praktisch passiert ist

Moegliche Punkte:

- Welche Klassen oder Query-Muster liessen sich gut migrieren?
- Wo musste das Team umdenken?
- Was wurde einfacher?
- Was wurde expliziter, aber dadurch auch ehrlicher?
- Welche Hibernate-Gewohnheiten mussten aktiv abgelegt werden?

### 11. Grenzen von Exposed

Wichtiger Teil fuer Glaubwuerdigkeit.

Moegliche Aussagen:

- `Exposed` ist nicht automatisch kuerzer oder bequemer.
- Wer maximale ORM-Automatik, Objektgraphen und klassische Session-Semantik will, wird nicht alles daran lieben.
- Man bekommt Kontrolle und Sichtbarkeit, bezahlt aber mit mehr bewussten Entscheidungen.

### 12. Fazit / Handlungsempfehlung

Klare Abschlussbotschaft:

- Wenn euch `Hibernate` vor allem wegen seiner impliziten Komplexitaet nervt, ist `Exposed` sehr interessant.
- Wenn ihr neu startet oder migriert: mit `DSL` anfangen.
- `DAO` nur gezielt einsetzen.
- `Exposed` ist besonders stark fuer Teams, die lieber expliziten Datenzugriff als ORM-Magie wollen.

## Roter Faden in einem Satz

`Mein Problem mit Hibernate ist nicht, dass es viel kann, sondern dass es mir zu oft verschweigt, was es gerade tut. Exposed fuehlt sich angenehmer an, weil es weniger verspricht und dadurch mehr erklaert.`

## Material, das wir schon haben

- `DSL vs DAO`: gute Basis fuer die Architektur- und Entscheidungsfolie
- `Cache Explained`: gute Basis fuer den technischen Deep Dive zu Cache, Flush und Transaktionslogik

## Noch offen fuer spaeter

- Konkrete Inhalte aus der echten Projektmigration
- Beispielqueries vorher/nachher
- Eventuell eine Abschlussfolie mit "Wann ich trotzdem nicht zu Exposed greifen wuerde"
