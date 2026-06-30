---
title: "Warum mich `JPA`/`Hibernate` nervt und warum `Exposed` angenehmer ist"
sub_title: "Weniger Magie, mehr Kontrolle, näher an SQL"
options:
  implicit_slide_ends: false
---

Die These
=========

<!-- new_lines: 6 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

`Hibernate` nimmt dir nicht nur Arbeit ab.  
Es versteckt auch Arbeit.

<!-- new_lines: 2 -->

`Exposed` löst nicht jedes Problem.  
Aber es fühlt sich oft angenehmer an, weil es ehrlicher ist.

<!-- end_slide -->

Warum dieser Talk?
=================

<!-- incremental_lists: true -->
<!-- new_lines: 2 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

- Weil viele von uns `JPA`-Schmerz nicht aus fehlendem Wissen haben, sondern trotz Erfahrung.
- Weil relationale Komplexität nicht verschwindet, nur weil ein ORM sie wegmodellieren will.
- Weil `Exposed` für Kotlin-Teams eine interessante Alternative ist, wenn Vorhersagbarkeit wichtiger ist als Magie.

<!-- end_slide -->

Was mich an `JPA`/`Hibernate` nervt
==================================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

- `Optimistic Locking` auf großen Entities oder breiten Aggregaten wird schnell teuer und fragil.
- Repository-Methoden und Query-Varianten sind oft nicht eindeutig genug.
- Projektionen sind möglich, aber selten wirklich angenehm.

<!-- column: 2 -->

- Bei genügend komplexen Abfragen schreibt man am Ende sowieso wieder `SQL`.
- `Lazy Loading`, `Flush` und Entity-State erzeugen Seiteneffekte, die nicht im sichtbaren Code stehen.

<!-- end_slide -->

`val` ist keine Garantie
========================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

```kotlin
@Entity
class Person(val name: String)
// Hibernate setzt name via Reflection
// val ≠ immutable hier
```

<!-- column: 2 -->

- `val` kompiliert zu `final` in Java
- JPA-Spec verlangt non-final Felder
- Hibernate umgeht das via Reflection
- Kotlins Immutabilitätsgarantie gilt hier nicht

<!-- end_slide -->

Null Safety ist eine Illusion
==============================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

```kotlin
@Entity
class User(var name: String) // non-nullable

// DB enthält NULL
// → Hibernate setzt name = null
// Compiler schweigt
```

<!-- column: 2 -->

- `String` im Code, `null` zur Laufzeit
- Reflection umgeht Kotlins Null-Checks
- Constraint-Drift zwischen DB und Code reicht
- kein Warning, keine Chance

<!-- end_slide -->

`data class` und `JPA` passen nicht zusammen
============================================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

### `data class` als Entity

- final → kein Proxy-Subclassing möglich
- kein No-Arg-Constructor
- `equals`/`hashCode` über alle Felder
- inkompatibel mit JPA-Spec

<!-- column: 2 -->

### Default-Werte werden ignoriert

- JPA ruft No-Arg-Constructor auf
- dann Reflection für alle Felder
- dein Constructor läuft beim Laden nicht
- Default-Werte haben keinen Effekt

<!-- end_slide -->

Und das ist noch nicht alles
=============================

<!-- incremental_lists: true -->
<!-- new_lines: 2 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

- `plugin.spring` + `plugin.jpa` + `allOpen` nötig — damit JPA mit Kotlin überhaupt compiliert
- Bis Kotlin 2.2: Annotations landeten am falschen Target — stille Mapping-Fehler
- JEP 500 plant, `final` wirklich final zu machen — `val`-Entities könnten brechen

<!-- end_slide -->

Die eigentliche Beschwerde
=========================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

### Es geht nicht primär um

- `SQL`
- die Datenbank
- oder darum, dass ein ORM überhaupt existiert

<!-- column: 2 -->

### Es geht um

- zu viel implizites Verhalten
- zu viel versteckten Zustand
- zu wenig direkte Verbindung zwischen Code und Query

<!-- end_slide -->

Die pointierte Version
=====================

<!-- new_lines: 4 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

> Ich will eine Query bauen und keinen Verhandlungsprozess
> mit dem Persistence Context führen.

<!-- new_lines: 2 -->

> Wenn ich SQL debuggen will, möchte ich SQL sehen
> und keine archäologische Grabung im Session-Zustand machen.

<!-- end_slide -->

Was `Exposed` anders macht
==========================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

- näher an `SQL`
- expliziter in den Datenzugriffspfaden
- klarer in Transaktionen und Seiteneffekten

<!-- column: 2 -->

- besser lesbar für Leute, die ohnehin in Queries denken
- weniger "trust me bro" im ORM-Layer

<!-- end_slide -->

Das mentale Modell
==================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

### Nicht lesen als

`Hibernate in Kotlin`

<!-- column: 2 -->

### Sondern eher als

- `DSL` für explizite Queries
- optional `DAO` für einfachere entity-zentrierte Fälle
- Transaktionen als zentrales Laufzeitmodell
- weniger Vollautomatik, dafür mehr Sichtbarkeit

<!-- end_slide -->

Nicht eindeutige Methoden vs sichtbare Query
============================================

<!-- new_lines: 2 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

### JPA-/Repository-Gefühl

```kotlin
fun findTop20ByStatusAndCreatedAtAfterOrderByPriorityDesc(
    status: Status,
    createdAfter: Instant
): List<OrderSummary>
```

- lesbar bis zu einem Punkt
- später schnell ein Zauberspruch
- reale SQL-Form bleibt implizit

<!-- column: 2 -->

### Exposed-Gefühl

```kotlin
Orders
    .selectAll()
    .where { Orders.status eq status }
    .andWhere { Orders.createdAt greater createdAfter }
    .orderBy(Orders.priority to SortOrder.DESC)
    .limit(20)
    .map { it.toOrderSummary() }
```

- mehr Schreibarbeit
- aber sichtbarere Query-Struktur
- leichter zu begründen

<!-- end_slide -->

`DSL` vs `DAO`
==============

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

### `DSL`

- Default für neue Projekte
- stark bei komplexen Queries
- stark bei Bulk-Operationen
- `R2DBC`-fähig
- Domain-Typen gehören euch

<!-- column: 2 -->

### `DAO`

- ok für einfache entity-zentrierte Fälle
- angenehm bei simplem CRUD
- bei komplexen Queries awkward
- kein `R2DBC`
- explizit, aber kein Hibernate-Klon

<!-- reset_layout -->
<!-- new_lines: 2 -->

Faustregel:  
`Wenn ihr unsicher seid, startet mit DSL.`

<!-- new_lines: 1 -->

Im `DSL`-Modus keine managed Entities — eure Domain-Typen sind eure Sache.

<!-- end_slide -->

Warum `DSL` meistens gewinnt
============================

<!-- incremental_lists: true -->
<!-- new_lines: 2 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

- Es skaliert besser, sobald Queries nicht mehr trivial sind.
- Es bleibt näher am resultierenden `SQL`.
- Es ist einfacher, Performance und Verhalten zu erklären.
- Es fühlt sich ehrlicher an, wenn man ohnehin in Joins, Filtern und Aggregaten denkt.

<!-- end_slide -->

Exposed in Produktion
=====================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

### `tarifrechner`

- Exposed `DAO`
- ein Entity, ein Zustand
- Optimistic Locking über Versionsspalte
- [TarifrechnerState.kt](https://github.com/dvag/tarifrechner-referenzimplementierung-backend/blob/develop/backend/src/main/kotlin/com/dvag/vp/digital/tarifrechner/referenzimplementierung/backend/persistence/TarifrechnerState.kt)

<!-- column: 2 -->

### `vorlagenverwaltung`

- Exposed `DSL`
- pure Queries, explizites Mapping
- Domain-Typen als `data class`
- [TemplateRepository.kt](https://github.com/dvag/vp-digital-tarifierung-formular-template-service/blob/main/src/main/kotlin/com/dvag/vpdigital/vorlagenverwaltung/persistence/template/TemplateRepository.kt)

<!-- end_slide -->

Tabellenstruktur ohne `@Entity`
================================

<!-- new_lines: 2 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

```kotlin
object TemplatesTable : Table("templates") {
    val templateId = uuid("template_id")
    val vbId       = text("vb_id")
    val formId     = text("form_id")
    val deleted    = bool("deleted")
    val rowVersion = long("row_version")

    override val primaryKey =
        PrimaryKey(templateId)
}
```

<!-- column: 2 -->

- Schema als `object`, kein managed Entity
- Felder sind Kotlin-Properties
- kein `@Entity`, kein `@Column`
- Domain-Typ ist separat — eine `data class`

<!-- reset_layout -->
<!-- new_lines: 1 -->

[TemplatesTable.kt](https://github.com/dvag/vp-digital-tarifierung-formular-template-service/blob/main/src/main/kotlin/com/dvag/vpdigital/vorlagenverwaltung/persistence/template/TemplatesTable.kt)

<!-- end_slide -->

Dynamische Filter ohne Criteria API
=====================================

<!-- new_lines: 2 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

```kotlin
var where = (vbId eq vbId) and
            (deleted eq false)
formId?.let {
    where = where and (formId eq it)
}
TemplatesTable
    .selectAll()
    .where(where)
    .orderBy(createdAt to SortOrder.ASC)
    .map(::toTemplateDto)
```

<!-- column: 2 -->

- `whereClause` ist ein normaler Ausdruck
- optionale Filter: einfaches `?.let`
- kein `CriteriaBuilder`, kein Predicate-Baukasten
- die Query ist lesbar bevor sie läuft

<!-- reset_layout -->
<!-- new_lines: 1 -->

[TemplateRepository.kt](https://github.com/dvag/vp-digital-tarifierung-formular-template-service/blob/main/src/main/kotlin/com/dvag/vpdigital/vorlagenverwaltung/persistence/template/TemplateRepository.kt)

<!-- end_slide -->

Optimistic Locking — sichtbar in der Query
==========================================

<!-- new_lines: 2 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

```kotlin
TemplatesTable.updateReturning(
    where = {
        (templateId eq id) and
        (rowVersion eq expectedVersion)
    }
) {
    it[name] = template.name
    it[rowVersion] = expectedVersion + 1
}.singleOrNull()
```

<!-- column: 2 -->

- Version-Check ist Teil der `WHERE`-Clause
- kein `@Version`, kein Dirty-Check im Hintergrund
- kein Ergebnis → Konflikt, Fehler explizit
- alles sichtbar, alles im Code

<!-- reset_layout -->
<!-- new_lines: 1 -->

[TemplateRepository.kt](https://github.com/dvag/vp-digital-tarifierung-formular-template-service/blob/main/src/main/kotlin/com/dvag/vpdigital/vorlagenverwaltung/persistence/template/TemplateRepository.kt)

<!-- end_slide -->

Was in der Praxis angenehmer wird
=================================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

- Query-Code bleibt lokaler.
- Der Bezug zum resultierenden `SQL` ist enger.
- Überraschungen durch Session-/Lifecycle-Magie nehmen ab.

<!-- column: 2 -->

- Datenzugriff fühlt sich eher wie bewusst geschriebener Code an.
- Komplexe Abfragen sehen nicht plötzlich wie ein Spezialfall aus, sondern wie der Normalfall.

<!-- end_slide -->

Warum `Exposed DAO` trotzdem Cache hat
======================================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

### Scope

`Diese Folie beschreibt das DAO-Laufzeitmodell.`

`Wenn ihr nur DSL nutzt, ist das nicht euer zentrales Laufzeitmodell.`

Kurze Antwort:  
`Weil DAO mehr tut als nur Query-Strings bauen.`

<!-- column: 2 -->

### Was der Cache liefern soll

- Identity Map innerhalb einer Transaktion
- Write-Behind für Entity-Änderungen
- Wiederverwendung geladener Relationen
- kohärentes Verhalten vor Flush und Commit

<!-- end_slide -->

`EntityCache` statt Session-Magie (`DAO`)
=========================================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

### Hibernate

- Hauptkontext: `Session`
- Identität: Session-gebunden
- Write-Behind: ja
- Lazy/Proxy-Modell: zentral
- Komplexität: hoch

<!-- column: 2 -->

### Exposed DAO

- Hauptkontext: Transaktion
- Identität: Transaktions-gebunden
- Write-Behind: ja
- Lazy/Proxy-Modell: deutlich kleiner
- Komplexität: fokussierter

<!-- reset_layout -->
<!-- new_lines: 2 -->

`Exposed` hat Cache-Verhalten, aber nicht dieselbe Art Laufzeitmaschine wie `Hibernate`.

`Gemeint ist hier das DAO-Modell, nicht Exposed-DSL allgemein.`

<!-- end_slide -->

Was man als Umsteiger wissen sollte
===================================

<!-- incremental_lists: true -->
<!-- new_lines: 2 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

- `DAO` ist nicht das Zentrum von allem.
- Direkte `DSL`-Writes können `DAO`-Cachezustand invalidieren.
- Flush-Verhalten ist Teil der Korrektheit, nicht nur ein Performance-Detail.
- Nicht jede Hibernate-Gewohnheit sollte 1:1 übertragen werden.

<!-- end_slide -->

Wo `Exposed` nicht magisch ist
==============================

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

- `Exposed` ist nicht automatisch kürzer.
- `Exposed` nimmt dir nicht jede Modellierungsentscheidung ab.

<!-- column: 2 -->

- Wer maximale ORM-Vollautomatik will, wird nicht alles daran lieben.
- Man tauscht Komfort gegen Sichtbarkeit und Kontrolle.

<!-- end_slide -->

Meine Empfehlung
================

<!-- incremental_lists: true -->
<!-- new_lines: 2 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

- Wenn euch `Hibernate` vor allem wegen impliziter Komplexität nervt, schaut euch `Exposed` an.
- Wenn ihr neu startet oder migriert: startet mit `DSL`.
- Nutzt `DAO` gezielt statt reflexhaft.
- Denkt in Queries, Transaktionen und Datenflüssen, nicht in ORM-Hoffnungen.

<!-- end_slide -->

Abschluss
=========

<!-- new_lines: 5 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

`Mein Problem mit Hibernate ist nicht, dass es viel kann.`

<!-- new_lines: 2 -->

`Mein Problem ist, dass es mir zu oft verschweigt, was es gerade tut.`

<!-- new_lines: 2 -->

`Exposed` fühlt sich angenehmer an, weil es weniger verspricht und dadurch mehr erklärt.

<!-- end_slide -->

Diskussion
==========

<!-- new_lines: 8 -->
<!-- column_layout: [1, 2] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

Fragen, Widerspruch, echte Migrationsnarben?
