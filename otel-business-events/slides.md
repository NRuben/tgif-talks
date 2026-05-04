---
title: "OpenTelemetry: Metriken und Business Events"
sub_title: "Logs, Traces, Metriken und fachliche Ereignisse sauber trennen"
author: Noah Ruben
options:
  implicit_slide_ends: false
---

## Warum dieser Talk?

<!-- new_lines: 5 -->
<!-- font_size: 1 -->

Wir reden oft über `OTEL`, als wäre es ein einziges Ding.

<!-- new_lines: 1 -->

In Wirklichkeit reden wir über unterschiedliche Fragen,
unterschiedliche Datenformen und unterschiedliche Zielgruppen.

<!-- new_lines: 2 -->

Heute geht es darum, diese Dinge sauber zu trennen.

<!-- speaker_note: Einstieg über Dieters Frage. -->
<!-- speaker_note: Wie nutzen wir OTEL für Fachbereichsreports und Partner-Dashboards? -->

<!-- end_slide -->

## Zielbild

<!-- new_lines: 3 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

### 1. Theorie

- Was sind Logs?
- Was sind Traces?
- Was sind Metriken?
- Business Events?

<!-- column: 1 -->

### 2. Praxis

- `tft-service`
- `wt-ref-prod`
- Event-Definitionen
- technische Metriken

<!-- reset_layout -->
<!-- new_lines: 2 -->

### 3. Demo

Vom Signal zum Dashboard.

<!-- end_slide -->

## Der Denkfehler

<!-- new_lines: 6 -->
<!-- font_size: 1 -->

Nicht jedes Ereignis
ist eine Metrik.

<!-- new_lines: 2 -->

Nicht jede Metrik
ist ein Business Event.

<!-- new_lines: 2 -->

Nicht jedes Log
ist fachlich relevant.

<!-- end_slide -->

## Offizielle OTEL-Signale

<!-- new_lines: 3 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

OpenTelemetry spricht primär über:

- `Traces`
- `Metrics`
- `Logs`
- `Baggage`
- `Profiles`

<!-- column: 1 -->

Wir fokussieren heute Logs, Traces und Metriken.

Business Events sind unsere fachliche Perspektive.

Technisch landen sie oft als strukturierte OTEL-Logs und optional als Span Events.

<!-- speaker_note: Wichtig - Business Events nicht als eigenen stabilen OTEL-Signaltyp verkaufen. -->

<!-- end_slide -->

## Warum Observability?

<!-- new_lines: 4 -->
<!-- font_size: 1 -->

Verteilte Systeme machen einfache Nutzeraktionen unsichtbar komplex.

<!-- new_lines: 2 -->

Ein Klick kann Frontend, Backend, Partner-Service, Datenbank und Messaging berühren.

<!-- new_lines: 2 -->

Observability hilft, daraus wieder eine zusammenhängende Geschichte zu machen.

<!-- end_slide -->

## Logs

<!-- new_lines: 3 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

### Frage

Was hat der Code gesehen?

### Gut für

- Fehlerdiagnose
- Stacktraces
- Detailkontext
- seltene Einzelfälle

<!-- column: 1 -->

### Nicht gut für

- stabile Fachreports
- harte SLIs ohne Aggregation
- Freitext-Semantik

<!-- end_slide -->

## Traces

<!-- new_lines: 3 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

### Frage

Welchen Pfad nimmt der Request?

### Gut für

- Service-Ketten
- Latenzursachen
- Fehlerpfade
- Service-Kontext

<!-- column: 1 -->

### Nicht gut für

- Mengenberichte allein
- Partner-Reports allein
- Trends ohne Aggregation

<!-- end_slide -->

## Metriken

<!-- new_lines: 3 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

### Frage

Wie oft, wie schnell, wie viel?

### Gut für

- Zeitreihen
- Alerts
- SLOs
- Trends

<!-- column: 1 -->

### Nicht gut für

- einzelne fachliche Fakten
- Rekonstruktion
- hohe Kardinalität
- Audit-Storys

<!-- end_slide -->

## Business Events

<!-- new_lines: 3 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

### Frage

Was ist fachlich passiert?

### Beispiele

- Vorlage erstellt
- Vorlage angewendet
- Formularversion ergänzt
- Policieren erfolgreich

<!-- column: 1 -->

### Eigenschaften

- benannt
- versioniert
- fachlich modelliert
- Kontext und Attribute

<!-- end_slide -->

## Die klare Abgrenzung

<!-- new_lines: 2 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

### Metrik

```text
cleanup.runs +1
outcome = success
```

- aggregiert
- klein
- alertfähig
- verliert Einzelfall-Details

<!-- column: 1 -->

### Business Event

```text
template.created
vb.id = ...
template.id = ...
form.id = ...
template.version = ...
```

- fachlicher Fakt
- auswertbar
- erklärbar
- später aggregierbar

<!-- end_slide -->

## Der wichtigste Satz

<!-- new_lines: 7 -->
<!-- font_size: 1 -->

Business Events sind keine Metriken.

<!-- new_lines: 2 -->

Aus Business Events können Metriken, Reports und Dashboards entstehen.

<!-- new_lines: 2 -->

Das Event bleibt trotzdem der fachliche Fakt.

<!-- end_slide -->

## OTEL als gemeinsames Modell

<!-- new_lines: 3 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

OTEL gibt uns:

- gemeinsame APIs
- gemeinsame Attribute
- gemeinsame Propagation
- gemeinsame Export-Pipeline

<!-- column: 1 -->

Dadurch können technische und fachliche Daten im gleichen Werkzeugraum landen.

Das heißt nicht, dass sie dieselbe Bedeutung haben.

<!-- end_slide -->

## Stage 2: Real-Life-Code

<!-- new_lines: 3 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

### `tft-service`

- Vorlagen-Events
- Formular-Events
- Emission-API
- gute Testbarkeit

<!-- column: 1 -->

### `wt-ref-prod`

- Tarifrechner-Events
- Policieren-Events
- Fehler-Events
- technische OTEL-Metriken

<!-- end_slide -->

## Technische Metrik im Tarifrechner

<!-- new_lines: 1 -->
<!-- font_size: 1 -->

```kotlin
private val cleanupRunsCounter =
  meter.counterBuilder(
    "cleanup.runs"
  )
    .setUnit("{run}")
    .build()

fun recordSoftDeleteCleanupSuccess(
  removedRows: Int,
  durationMs: Long,
) {
  cleanupRunsCounter.add(1, successAttributes)
  cleanupDurationHistogram.record(
    durationMs,
    successAttributes,
  )
}
```

<!-- end_slide -->

## Anatomie eines Business Events

<!-- new_lines: 2 -->
<!-- font_size: 1 -->

```kotlin
enum class VorlagenEvents : Events {
  VORLAGE_CREATED("created"),
  VORLAGE_SHARED("shared"),
  VORLAGE_APPLIED("applied");

  override val category = "template"
  override val version = 1
}
```

```kotlin
data class VorlagenEventAttrs(
  @OtelKey("template.id") val templateId: String,
  @OtelKey("form.id") val formId: String,
  @OtelKey("template.version") val version: Int,
)
```

<!-- end_slide -->

## Event-Emission im Service

<!-- new_lines: 2 -->
<!-- font_size: 1 -->

```kotlin
emitter.emitVorlagenEvent(
  VORLAGE_CREATED,
  VorlagenCtx(vbId),
  savedEntity.toVorlagenEventAttrs(),
)
```

<!-- new_lines: 1 -->

Nicht:

`counter template_created +1`

Sondern:

`template.created` ist fachlich passiert.

<!-- end_slide -->

## Tarifrechner-Events

<!-- new_lines: 2 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

### Event-Familien

- `tarifrechner.created`
- `tarifrechner.status.updated`
- `tarifrechner.antrag.generated`
- `policieren.success`
- `policieren.fail`

<!-- column: 1 -->

### Gemeinsamer Kontext

- `produkt.pia.api.version`
- `vb.id`
- `produkt.partner`
- `haushalt.id`

<!-- end_slide -->

## Von Event zu Dashboard

<!-- new_lines: 3 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

### Event

`template.applied`

### Fragen

- Nutzung pro Vorlage?
- aktive Formulare?
- alte Versionen?

<!-- column: 1 -->

### Dashboard

- Anzahl pro Tag
- Top-Formulare
- Nutzung pro Partner
- Trend nach Version

<!-- end_slide -->

## Naming und Kardinalität

<!-- new_lines: 3 -->
<!-- column_layout: [1, 1] -->
<!-- font_size: 1 -->
<!-- column: 0 -->

### Namen

- klein schreiben
- mit Punkten gliedern
- fachlich stabil halten
- Version bewusst erhöhen

<!-- column: 1 -->

### Attribute

- Dimensionen bewusst wählen
- IDs sparsam in Metriken
- keine Personendaten
- Fachkontext explizit

<!-- end_slide -->

## Stage 3: Demo

<!-- new_lines: 4 -->
<!-- font_size: 1 -->

1. Code: Wo entstehen Metriken?
2. Code: Wo entstehen Business Events?
3. Query: Wo liegen die Daten?
4. Dashboard: Welche Fragen beantworten wir?

<!-- new_lines: 2 -->

Der Demo-Teil ist kein Folienkino.

Er zeigt, ob die Semantik im Alltag trägt.

<!-- end_slide -->

## Fazit

<!-- new_lines: 4 -->
<!-- font_size: 1 -->

OTEL löst nicht die fachliche Modellierung.

<!-- new_lines: 2 -->

OTEL macht gute Modellierung sichtbar, transportierbar und auswertbar.

<!-- new_lines: 2 -->

Erst Semantik, dann Dashboard.

<!-- end_slide -->

## Quellen und Anker

<!-- new_lines: 2 -->
<!-- font_size: 1 -->

- DVAG Tech Blog: Distributed Tracing.
- OpenTelemetry Docs.
- `tft-service`: `VorlagenEvents.kt`.
- `wt-ref-prod`: Events und Metriken.
- `otel-events`: `BusinessEventEmitter`.

<!-- end_slide -->
