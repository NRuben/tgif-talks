---
title: "OpenTelemetry: Metriken und Business Events"
sub_title: "Logs, Traces, Metriken und fachliche Ereignisse sauber trennen"
author: Noah Ruben
options:
  implicit_slide_ends: false

theme:
  name: dark
  override:
    code:
      alignment: left
      margin:
        fixed: 0
---

<!-- new_lines: 3 -->
<!-- column_layout: [1, 4, 4, 4] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
# Agenda

### 1. Theorie

- Was sind Logs?
- Was sind Traces?
- Was sind Metriken?
- Business Events?

<!-- column: 2 -->

<!-- new_lines: 2 -->
### 2. Praxis

- `tft-service`
- `wt-ref-prod`
- Event-Definitionen
- technische Metriken

<!-- column: 3 -->
<!-- new_lines: 2 -->
### 3. Demo

Vom Signal zum Dashboard.

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

<!-- new_lines: 1 -->

Ein Klick kann Frontend, Backend, Partner-Service, Datenbank und Messaging berühren.

<!-- new_lines: 1 -->

Observability hilft, daraus wieder eine zusammenhängende Geschichte zu machen.
- Metriken sind zur Laufzeit erfasste Messwerte eines Systems und zeigen als numerische Daten, wie sich Zustand und Verhalten über die Zeit entwickeln.
- Logs zeigen, was zu bestimmten Zeitpunkten im System passiert ist und liefern detaillierten Kontext zu Ereignissen, Fehlern und Entscheidungen.
- Traces zeigen, wie sich eine einzelne Anfrage durch das System bewegt und wo sie entlang des Weges Zeit verbringt.

<!-- new_lines: 1 -->

Fachliche Events ermöglichen Business-Dashboards direkt aus Telemetrie (z. B. genutzte Vorlagen, abgeschlossene Anträge).
Das heißt OpenTelemetry-Daten lassen sich direkt zu fachlichen KPIs und Dashboards aggregieren oder als diese emittieren.
<!-- end_slide -->

<!-- column_layout: [1, 4, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
<!-- new_lines: 1 -->
## Logs
<!-- new_lines: 2 -->
### Frage

Was ist genau passiert?

### Definition
Ein Log ist ein mit einem Zeitstempel versehener Texteintrag,
der entweder strukturiert oder unstrukturiert sein kann und optional Metadaten (attributes) enthält.

<!-- reset_layout -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- column: 1 -->
<!-- font_size: 1 -->
### Gut für

- Fehlerdiagnose
- Stacktraces
- Detailkontext
- seltene Einzelfälle

<!-- column: 2 -->

### Nicht gut für

- stabile Fachreports
- harte SLIs ohne Aggregation
- Freitext-Semantik

<!-- end_slide -->

<!-- column_layout: [1, 4, 1] -->
<!-- column: 1 -->
## Beispielhaft: 
<!-- font_size: 1 -->
```java
val logMessageBuilder = logging.atLevel(logLevel)

logMessageBuilder.addKeyValue(key, value)
logMessageBuilder.setCause(exception)
logMessageBuilder.log(message)
```
```json
{
  "timestamp": "2026-05-04T14:32:10.123Z",
  "level": "ERROR",
  "logger": "com.example.MyClass",
  "thread": "main",
  "message": "Failed to process request",
  "attributes": {
    "key": "someKey",
    "value": "someValue"
  },
  "exception": {
    "type": "java.lang.IllegalStateException",
    "message": "invalid state",
    "stackTrace": [
      "com.example.MyClass.process(MyClass.kt:42)",
      "com.example.App.main(App.kt:10)"
    ]
  }
}
```
<!-- speaker_note: Beispiele für SLIs
Antwortzeit einer API (z. B. „95% der Requests < 200ms“)
Erfolgsrate von Requests (z. B. „99,9% ohne Fehler“)
Verfügbarkeit (z. B. „Service ist erreichbar“) -->
<!-- end_slide -->

<!-- column_layout: [1, 4, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Traces
<!-- new_lines: 1 -->
### Frage

Welchen Pfad nimmt der Request?

### Definition
Traces zeigen, wo ein Request seine Zeit verbringt.

Technisch ist ein Trace eine Reihe zusammengehöriger Spans.

Ein Span beschreibt eine einzelne Arbeitseinheit:
Name, Start, Ende, Attribute, Status und Eltern-Span.

Die gemeinsame `traceId` verbindet die Spans.
Die `parentSpanId` baut daraus die Hierarchie.

<!-- reset_layout -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- column: 1 -->
<!-- font_size: 1 -->
### Gut für

- Service-Ketten
- Latenzursachen
- Fehlerpfade
- Service-Kontext

<!-- column: 2 -->

### Nicht gut für

- Mengenberichte allein
- Partner-Reports allein
- Trends ohne Aggregation

<!-- end_slide -->

<!-- column_layout: [1, 4, 1] -->
<!-- column: 1 -->
## Beispielhaft:
<!-- font_size: 1 -->
```java
val root = tracer.spanBuilder("POST /tarifrechner")
                 .setSpanKind(SpanKind.SERVER)
                 .startSpan()

root.makeCurrent().use {
  val child = tracer.spanBuilder("tarifrechner.create")
                    .startSpan()

  child.makeCurrent().use {
    val db = tracer.spanBuilder("db.insert.tarifrechner")
                   .setSpanKind(SpanKind.CLIENT)
                   .startSpan()

    service.createTarifrechner(request)

    db.end()
  }

  child.end()
}

root.end()
```
```json
[
  // root span
  {
    "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
    "spanId": "7c9b8f4a1d3e2c10",
    "parentSpanId": null,
    "name": "POST /tarifrechner",
    "kind": "SERVER"
  },
  // child span
  {
    "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
    "spanId": "00f067aa0ba902b7",
    "parentSpanId": "7c9b8f4a1d3e2c10",
    "name": "tarifrechner.create",
    "kind": "INTERNAL",
    "durationMs": 164,
    "attributes": {
      "tarifrechner.id": "326ade31-891a-4589"
    }
  },
  // dependency span
  {
    "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
    "spanId": "3d4a9f2b6c8e1020",
    "parentSpanId": "00f067aa0ba902b7",
    "name": "db.insert.tarifrechner",
    "kind": "CLIENT",
    "durationMs": 41,
    "attributes": {
      "db.system": "postgresql"
    }
  }
]
```
<!-- end_slide -->

<!-- column_layout: [1, 4, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Metriken
<!-- new_lines: 1 -->
### Frage

Wie oft, wie schnell, wie viel?

### Definition
Eine Metrik ist eine zur Laufzeit erfasste Messung.

Der einzelne Messmoment heißt `Metric Event`:
Wert, Zeitpunkt und Metadaten.

Instrumente erfassen diese Messungen,
z. B. `Counter`, `Gauge` oder `Histogram`.

Aggregation verdichtet viele Messungen pro Zeitfenster.

<!-- reset_layout -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- column: 1 -->
<!-- font_size: 1 -->
### Gut für

- Zeitreihen
- Alerts
- SLOs
- Trends

<!-- column: 2 -->

### Nicht gut für

- einzelne fachliche Fakten
- einzelne Request-Lebenszyklen
- hohe Kardinalität
- Audit-Storys

<!-- end_slide -->

<!-- column_layout: [1, 4, 1] -->
<!-- column: 1 -->
## Beispielhaft:
<!-- font_size: 1 -->
```java
val cleanupRuns = meter
        .counterBuilder("cleanup.runs")
        .setUnit("{run}")
        .build()

val cleanupDuration = meter
        .histogramBuilder("cleanup.duration")
        .setUnit("ms")
        .build()

cleanupRuns.add(1, Attributes.of(outcome, "success"))
cleanupDuration.record(164, Attributes.of(outcome, "success"))
```
```json
[
  // counter metric event
  {
    "name": "cleanup.runs",
    "unit": "{run}",
    "type": "counter",
    "time": "2026-05-04T14:32:10.287Z",
    "value": 1,
    "attributes": {
      "outcome": "success"
    }
  },
  // histogram metric event
  {
    "name": "cleanup.duration",
    "unit": "ms",
    "type": "histogram",
    "time": "2026-05-04T14:32:10.287Z",
    "count": 1,
    "sum": 164,
    "attributes": {
      "outcome": "success"
    }
  }
]
```
<!-- end_slide -->

<!-- column_layout: [1, 4, 1] -->
<!-- column: 1 -->
### Annotation-basiert
<!-- font_size: 1 -->

```kotlin
@Counted("tarifrechner.created")
@Timed("tarifrechner.create.duration")
fun createTarifrechner(request: TarifrechnerRequest) {
  service.createTarifrechner(request)
}
```

```json
[
  // generated counter
  {
    "name": "tarifrechner.created",
    "type": "counter",
    "value": 1
  },
  // generated timer / histogram
  {
    "name": "tarifrechner.create.duration",
    "type": "histogram",
    "count": 1,
    "sum": 164
  }
]
```

Kurz gesagt:
Annotationen sind bequem,
die explizite API macht Semantik und Attribute sichtbarer.

<!-- end_slide -->

<!-- column_layout: [1, 4, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Business Events
<!-- new_lines: 1 -->
### Frage

Was ist **fachlich** passiert?

### Definition
Ein Business Event ist ein fachlicher Fakt,
der zu einem konkreten Zeitpunkt passiert ist.

Es ist kein eigener stabiler OTEL-Signaltyp.

Es ist auch keine OTEL-Metrik.
Die Metrik aggregiert Messwerte.
Das Business Event beschreibt den fachlichen Einzelfall.

Bei uns wird es als strukturierter OTEL Log Record emittiert.
Bei aktivem Trace landet es zusätzlich als Span Event am aktuellen Span.

Es besteht aus Name, Kategorie, Version,
gemeinsamem Kontext und event-spezifischen Attributen.

Aus vielen Business Events können Reports, Alarme (in der DVAG sogar mit Kritikalität) und Dashboards entstehen.
Das einzelne Event bleibt aber ein Einzelfall. 

<!-- reset_layout -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- column: 1 -->
<!-- font_size: 1 -->
### Gut für

- Fachbereichs-Reports
- Produktpartner-Dashboards
- Prozessnachvollzug
- spätere Aggregation

<!-- column: 2 -->

### Nicht gut für

- technische SLIs allein
- Request-Latenzpfade
- unmodellierte Debugdetails
- Personen- oder Rohdaten

<!-- end_slide -->

<!-- column_layout: [1, 4, 1] -->
<!-- column: 1 -->
## Beispielhaft:
<!-- font_size: 1 -->

```kotlin
data class TemplateCtx(
  @OtelKey("vb.id") val vbId: String,
)

data class TemplateCreatedAttrs(
  @OtelKey("template.id") val templateId: String,
  @OtelKey("form.id") val formId: String,
  @OtelKey("template.version") val version: Int,
)

@OTelEvent(name = "template.created", category = "business", version = 1)
class TemplateCreated(
  override val context: TemplateCtx,
  override val attributes: TemplateCreatedAttrs,
  override val durationMs: Long? = null,
) : BusinessEvent<TemplateCtx, TemplateCreatedAttrs>

emitter.emit(TemplateCreated(ctx, attrs))
```

```json
[
  // OTel log record
  {
    "signal": "log",
    "body": "business.template.created",
    "severity": "INFO",
    "attributes": {
      "ai.event.name": "template.created",
      "event.name": "template.created",
      "event.category": "business",
      "event.version": 1,
      "vb.id": "007",
      "template.id": "tpl-42",
      "form.id": "frm-9",
      "template.version": 3
    }
  },
  // optional span event
  {
    "signal": "span_event",
    "name": "template.created",
    "attributes": {
      "event.body": "business.template.created",
      "event.name": "template.created"
    }
  }
]
```

<!-- end_slide -->

<!-- column_layout: [1, 4, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->

## OTEL als gemeinsames Modell
<!-- new_lines: 3 -->

OTEL gibt uns:

- gemeinsame APIs
- gemeinsame Attribute
- gemeinsame Propagation
- gemeinsame Export-Pipeline

<!-- new_lines: 2 -->

Dadurch können technische und fachliche Daten im gleichen Werkzeugraum landen.

Das heißt nicht, dass sie dieselbe Bedeutung haben.

<!-- end_slide -->


## Stage 2+3: Real-Life-Code und Demo

<!-- new_lines: 4 -->
<!-- font_size: 1 -->

1. Code: Wo entstehen Metriken?
2. Code: Wo entstehen Business Events?
3. Query: Wo liegen die Daten?
4. Dashboard: Welche Fragen beantworten wir?


<!-- end_slide -->

<!-- new_lines: 3 -->
<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->

### `tft-service`

- Vorlagen-Events
- Formular-Events

<!-- column: 2 -->

### `wt-ref-prod`

- Tarifrechner-Events
- Policieren-Events
- Fehler-Events

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


<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Von Event zu Dashboard
<!-- new_lines: 3 -->
### Fragen

- Nutzung pro Vorlage?
- aktive Formulare?
- alte Versionen?

<!-- column: 2 -->
<!-- new_lines: 5 -->

### Dashboard

- Anzahl pro Tag
- Top-Formulare
- Nutzung pro Partner
- Trend nach Version

<!-- end_slide -->

<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Naming und Kardinalität
<!-- new_lines: 3 -->

### Namen

- klein schreiben
- mit Punkten gliedern
- fachlich stabil halten
- Version bewusst erhöhen

<!-- column: 2 -->
<!-- new_lines: 5 -->

### Attribute

- Dimensionen bewusst wählen
- IDs sparsam in Metriken
- keine Personendaten
- Fachkontext explizit

<!-- end_slide -->

<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Fazit: Warum Business Events?
<!-- new_lines: 3 -->
### Technische Sicht

Logs, Traces und Metriken erklären,
was technisch im System passiert.

Sie beantworten Fragen wie:

- welcher Service?
- welcher Fehler?
- welche Latenz?
- welche Fehlerrate?

<!-- column: 2 -->
<!-- new_lines: 5 -->

### Fachliche Sicht

Business Events erklären,
was fachlich passiert ist.

Sie beantworten Fragen wie:

- welche Vorlage?
- welche Version?
- welcher Partner?
- welcher Prozessschritt?

<!-- end_slide -->

<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Metrik vs. Business Event
<!-- new_lines: 3 -->
### Metrik

```text
template.applied = 42
```

Eine Metrik zählt.

Sie ist stark für:

- Trends
- Alerts
- KPIs
- Aggregationen

<!-- column: 2 -->
<!-- new_lines: 5 -->

### Business Event

```text
template.applied
template.id = tpl-42
template.version = 3
form.id = frm-9
time = 14:32:10
```

Ein Business Event erklärt den fachlichen Einzelfall.

<!-- end_slide -->

<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Der entscheidende Unterschied
<!-- new_lines: 3 -->
### Aus Events wird Auswertung

Aus Business Events können später Metriken entstehen.

Beispiele:

- Vorlagen pro Tag
- Nutzung pro Version
- Fehlerquote pro Partner
- Abschlüsse pro Produkt

<!-- column: 2 -->
<!-- new_lines: 5 -->

### Umgekehrt nicht

Aus einer Metrik entsteht kein Einzelfall zurück.

`template.applied = 42` sagt nicht:

- welche Vorlage
- welche Version
- welcher Kontext
- welcher Zeitpunkt

<!-- end_slide -->

<!-- column_layout: [1, 2, 2, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Was Business Events ermöglichen
<!-- new_lines: 3 -->
### Für Fachbereich und Partner

- Alerting in Checkly
- Fachbereichsreports
- Partner-Dashboards
- Feature-Nutzung
- Prozessanalyse
- belastbare KPIs

<!-- column: 2 -->
<!-- new_lines: 6 -->

### Für Betrieb und Produkt

- Business Impact im Incident
- betroffene Prozesse
- betroffene Partnerstrecken
- alte Versionen im Umlauf
- Reibungspunkte im Ablauf

<!-- end_slide -->

<!-- column_layout: [1, 4, 4, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Warum Typsicherheit zählt
<!-- new_lines: 3 -->
### Nicht so

```kotlin
emit(
    "template.applied",
    mapOf(
        "templateId" to templateId,
        "formId" to formId,
    ),
)
```


<!-- column: 2 -->
<!-- new_lines: 5 -->

### Sicher

```kotlin
TemplateApplied(
  context = TemplateCtx(vbId),
  attributes = TemplateAttrs(
    templateId,
    formId,
    version,
  ),
)
```


<!-- end_slide -->

<!-- column_layout: [1, 4, 4, 1] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
# Abschließend
<!-- new_lines: 3 -->
### Das Modell

Logs liefern Details.

Traces erklären den technischen Weg.

Metriken zeigen Mengen und Trends.

Business Events machen fachliche Fakten berichtbar.

<!-- column: 2 -->
<!-- new_lines: 5 -->

### Allgemein

OTEL liefert den Transport.

Die Modellierung entscheidet über den Wert.

Erst fachliche Events sauber modellieren.

Dann werden Reports, Dashboards und KPIs belastbar.

<!-- end_slide -->

<!-- column_layout: [1, 9] -->
<!-- font_size: 1 -->
<!-- column: 1 -->
## Quellen und Anker
<!-- new_lines: 3 -->
### Quellen

- [DVAG Tech Blog: Distributed Tracing](https://dvag.github.io/posts/distributed-tracing/).
- [OpenTelemetry Docs](https://opentelemetry.io/docs/).

### Dashboards

- [ADX Telemetrie-Dashboard](https://dataexplorer.azure.com/dashboards/3d389d4f-65b3-4cef-ad1f-72d4f822e14d?p-_startTime=30days&p-_endTime=now&p-_pFilter=all&p-_pInvertFilter=v-false&p-_pLogLevel=all&p-aggInterval=v-1h&p-_pContainerName=all#0a0d9f68-ce69-42d9-8d01-9b4f6b2412b7).
- [ADX Business-Events-Dashboard](https://dataexplorer.azure.com/dashboards/2edb0d3c-777c-4eff-b485-e3c0885664b3?p-_startTime=30days&p-_endTime=now&p-_pFilter=all&p-_pInvertFilter=v-false&p-_pLogLevel=all&p-aggInterval=v-1h&p-_pFormId=all&p-_pEvents=all&p-_pVB=all#97e68097-81c8-4f45-b4f4-eadafb32a251).

### Code-Anker

- `Vorlagen Service`: [VorlagenEvents.kt](https://github.com/dvag/vp-digital-tarifierung-formular-template-service/blob/main/src/main/kotlin/com/dvag/vpdigital/vorlagenverwaltung/traceability/VorlagenEvents.kt).
- `Referenzimplementierung`:
    - [Events.kt](https://github.com/dvag/tarifrechner-referenzimplementierung-backend/blob/develop/backend/src/main/kotlin/com/dvag/vp/digital/tarifrechner/referenzimplementierung/backend/traceability/Events.kt).
    - [TarifrechnerMetrics.kt](https://github.com/dvag/tarifrechner-referenzimplementierung-backend/blob/develop/backend/src/main/kotlin/com/dvag/vp/digital/tarifrechner/referenzimplementierung/backend/traceability/TarifrechnerMetrics.kt).
- `otel-business-events-lib`: [BusinessEventEmitter.kt](https://github.com/dvag/otel-business-events-lib/blob/develop/src/main/kotlin/com/dvag/otel/emitter/BusinessEventEmitter.kt).

<!-- end_slide -->
