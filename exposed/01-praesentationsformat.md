# Praesentationsformat fuer den TGIF Talk

## Entscheidung

Wir nutzen `presenterm` fuer den Talk.

## Warum diese Entscheidung?

- Die Slides bleiben `Markdown` und passen damit gut zu unserem Arbeitsstil.
- Die Praesentation laeuft direkt im Terminal.
- `presenterm` ist ein echtes Slide-Tool und nicht nur ein Markdown-Reader.
- Code, Listen und technische Inhalte lassen sich gut darstellen.
- Das Format passt gut zu einem entwicklerzentrierten TGIF-Talk.

## Was wir damit konkret meinen

- Eine Praesentation ist eine einzelne Markdown-Datei.
- Die Folien werden mit `presenterm` gerendert.
- Die Slide-Trennung erfolgt mit:

```html
<!-- end_slide -->
```

- Optional kann die Datei Front Matter fuer Titel, Untertitel und Autor enthalten.

## Arbeitskonvention fuer dieses Talk-Material

- Referenz und Planung bleiben in den Dateien unter `TGIF/`.
- Das eigentliche Deck sollte als `TGIF/slides.md` angelegt werden.
- Inhaltlich orientieren sich die Slides an `TGIF/00-aufbau.md`.
- Bestehende Deep-Dive-Inhalte koennen aus `TGIF/exposed-dsl-vs-dao.md` und `TGIF/exposed-cache-explained.md` uebernommen oder verdichtet werden.

## Konsequenz fuer die naechsten Schritte

Wenn wir das eigentliche Deck bauen, schreiben wir es direkt im `presenterm`-Format und nicht als generisches Markdown fuer einen normalen Reader.
