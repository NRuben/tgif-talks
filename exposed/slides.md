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

<!-- column: 2 -->

### `DAO`

- ok für einfache entity-zentrierte Fälle
- angenehm bei simplem CRUD
- bei komplexen Queries awkward
- kein `R2DBC`

<!-- reset_layout -->
<!-- new_lines: 2 -->

Faustregel:  
`Wenn ihr unsicher seid, startet mit DSL.`

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

Case Study: echte Migration
===========================

<!-- new_lines: 3 -->

Hier kommt später euer umgestelltes Projekt rein.

<!-- new_lines: 2 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 3 -->
<!-- column: 1 -->

### Zeigen

- Ausgangslage im alten Stack
- welche Query- und Mapping-Muster wehgetan haben
- warum der Wechsel überhaupt diskutiert wurde

<!-- column: 2 -->

### Belegen

- was nach der Migration sofort anders war
- wo `Exposed` konkret angenehmer wurde
- wo der Wechsel auch echte Kosten hatte

<!-- reset_layout -->
<!-- new_lines: 2 -->

Merksatz:  
`Der Talk wird glaubwürdig, sobald nicht nur Theorie, sondern echte Reibung sichtbar wird.`

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
