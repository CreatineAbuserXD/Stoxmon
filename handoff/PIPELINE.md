# Stox — Pipeline, Architektur, offene Entscheidungen

Stand: 8. September 2026 · gehört zu `SPEC.md`, `mock.json`, `DESIGN.md`

Diese Datei beschreibt **wie** ein Signal entsteht — und wo Entscheidungen noch
offen sind. Sie ist als Grundlage für ein `AGENTS.md` gedacht: alles hier ist
Kontext, den ein Agent braucht, um nicht am falschen Ende zu optimieren.

---

# TEIL A — Architektur

## A.1 Überblick

```
                 ┌─────────────────────────────────────────┐
   Quellen ───▶  │  INGEST         (stündlich)             │
                 │  holen, deduplizieren, Company zuordnen │
                 └──────────────┬──────────────────────────┘
                                ▼
                 ┌─────────────────────────────────────────┐
                 │  RELEVANCE      (ohne LLM)              │
                 │  gibt es eine aktive Thesis dazu?       │
                 └──────────────┬──────────────────────────┘
                                ▼
                 ┌─────────────────────────────────────────┐
                 │  TOPIC MATCH    (Embedding)             │
                 │  welche kanonischen Themen berührt es?  │
                 └──────────────┬──────────────────────────┘
                                ▼
                 ┌─────────────────────────────────────────┐
                 │  JUDGE          (LLM, 1× pro Thema)     │
                 │  verdict + why + counter + magnitude    │
                 └──────────────┬──────────────────────────┘
                                ▼
                 ┌─────────────────────────────────────────┐
                 │  FAN-OUT        (SQL, ~kostenlos)       │
                 │  an alle Nutzer mit diesem Thema        │
                 └──────────────┬──────────────────────────┘
                                ▼
   ┌────────────────────────────┴────────────────────────────┐
   ▼                                                          ▼
┌──────────────────────┐                        ┌──────────────────────┐
│ MORNING BRIEF 07:00  │                        │ TASK RUNNER          │
│ Health neu berechnen │                        │ daily/weekly/monthly │
│ Push nur bei weakens │                        │ + event-getriggert   │
└──────────────────────┘                        └──────────────────────┘
```

Rechenschritt-Klassen, absteigend teuer:
`JUDGE (LLM)` ≫ `TASK auf thesis` ≫ `TOPIC MATCH (Embedding)` ≫ `RELEVANCE (SQL)` ≫ `FAN-OUT (SQL)`

Das Produkt lebt davon, den teuersten Schritt **einmal pro Thema** zu machen,
nicht einmal pro Nutzer.

## A.2 Komponenten

| Komponente | Verantwortung | Läuft |
|---|---|---|
| `api` | FastAPI, liest nur aus der DB, rechnet nichts | dauerhaft |
| `ingest` | Quellen abfragen, deduplizieren, `raw_items` schreiben | stündlich |
| `classify` | Relevance → Topic Match → Judge, schreibt `topic_verdicts` | nach `ingest` |
| `fanout` | `topic_verdicts` × betroffene Watchpoints → `signals` | nach `classify` |
| `brief` | Health je Thesis neu, `home`-Antwort vorbereiten, Push | 07:00 lokal |
| `tasks` | fällige Tasks ausführen, Ergebnis als Signal | 07:00 + Event-Hook |
| `reaction` | `market_reaction` nachtragen, wenn Fenster abgelaufen | täglich |
| `resolve` | Thesen prüfen: Horizont erreicht oder `falsify_rule` erfüllt | täglich |

Wichtig: `api` ist **dumm**. Jede Berechnung passiert vorher in einem Job.
Ein `GET /home` darf keinen LLM-Aufruf und keine Aggregation auslösen.

## A.3 Datenbank — die vier Tabellen, die den Unterschied machen

Über `SPEC.md` hinaus:

```sql
raw_items          -- jede eingegangene Meldung, einmal, roh
  id, source, external_id, url, title, body, published_at,
  content_hash,          -- Deduplizierung über Kernfaktum
  company_ids[]          -- kann mehrere betreffen

canonical_watchpoints  -- pro Company wachsende Themenliste
  id, company_id, label, kind, embedding vector(1536),
  user_count, created_from_watchpoint_id

topic_verdicts     -- DAS Cache-Herzstück: EIN Urteil pro (Meldung × Thema)
  id, raw_item_id, canonical_id,
  verdict, headline, why, counter, magnitude, decided_by,
  confidence numeric,    -- siehe A.5
  model, prompt_version, created_at
  UNIQUE (raw_item_id, canonical_id)

signals            -- die Nutzersicht, verweist auf topic_verdict
  id, watchpoint_id, topic_verdict_id, user_feedback, ...
```

`signals` speichert **keinen** Text. Alles Inhaltliche steht in `topic_verdicts`
und wird beim Lesen gejoint. Folge: eine verbesserte Bewertung korrigiert
rückwirkend alle Nutzer, und Speicher skaliert nicht mit der Nutzerzahl.

## A.4 Idempotenz

Jeder Job muss doppelt laufen können, ohne Schaden:
- `raw_items` über `content_hash` unique.
- `topic_verdicts` über `(raw_item_id, canonical_id)` unique.
- `signals` über `(watchpoint_id, topic_verdict_id)` unique.
- `brief` schreibt in eine Tagestabelle mit `(user_id, date)` unique.

Ohne das bekommt der Nutzer bei jedem Neustart dasselbe Signal doppelt — und
verliert sofort das Vertrauen in die App.

## A.5 Confidence-Schwelle

`JUDGE` liefert `confidence` (0–1). Regel:

| Confidence | Verhalten |
|---|---|
| ≥ 0.75 | Signal wird erzeugt |
| 0.55–0.74 | Signal nur, wenn `verdict = weakens` (Warnungen dürfen unsicher sein) |
| < 0.55 | verworfen, zählt in „Meldungen ohne Thesenbezug“ |

**Präzision vor Vollständigkeit.** Drei sichere Signale sind besser als zehn
wackelige — zwei Fehlurteile hintereinander kosten das Vertrauen dauerhaft.
Die Schwelle ist Konfiguration, kein Code-Konstante.

---

# TEIL B — Die Prompts

Alle Prompts sind versioniert (`prompt_version` in `topic_verdicts`), damit man
Qualität über Zeit vergleichen kann. Ausgabe immer strict JSON.

## B.0 Regeln in jedem System-Prompt

```
Du bewertest, ob eine Information die THESE DES NUTZERS stützt oder schwächt.
Du bewertest NICHT das Investment.

Verboten:
- Kauf-, Verkauf- oder Halteempfehlungen
- Kursziele, Kursprognosen, Renditeschätzungen
- Aussagen über die Klugheit der These ("gute These", "riskant")
- Bewertung anhand der Kursentwicklung

Erlaubt und erwünscht:
- Faktenlage nennen, auch wenn sie unvollständig ist
- Unsicherheit benennen statt kaschieren
- Gegenargumente nennen, auch wenn sie das Urteil abschwächen

Wenn die Information den Watchpoint nicht berührt: sage das.
Nichts zu melden ist ein gültiges und häufiges Ergebnis.
```

Der letzte Satz ist der wichtigste. Ohne ihn erfindet jedes Modell einen Bezug,
weil es glaubt, gefragt zu sein.

## B.1 RELEVANCE — ohne LLM

Kein Prompt. SQL und Regeln:
```
1. Meldung → Company über Ticker/ISIN/Namensliste (inkl. Aliasse).
2. Existiert eine aktive Thesis zu dieser Company? Sonst verwerfen.
3. Blockliste: Kursberichte ohne Inhalt ("Aktie steigt um 2 %"),
   reine Kursziel-Meldungen ohne Begründung, Werbung, Kursdaten-Ticker.
```
Erwartete Reduktion: ~214 Meldungen → ~25 Kandidaten.
Dieser Schritt kostet nichts und entscheidet über deine Rechnung.

## B.2 TOPIC MATCH — Embedding, kein LLM

```
1. Meldung einbetten (Titel + erste 800 Zeichen).
2. Cosinus-Ähnlichkeit gegen alle canonical_watchpoints der Company.
3. Treffer: alle mit sim ≥ 0.82, maximal 3.
4. Kein Treffer → zählt als "ohne Thesenbezug", kein LLM-Aufruf.
```

**Neues kanonisches Thema** entsteht nur beim Anlegen eines Watchpoints
(nicht beim Einlesen von Meldungen): schreibt ein Nutzer einen Watchpoint,
der zu keinem Thema seiner Company passt (sim < 0.82), wird daraus ein neues
kanonisches Thema — von dem alle folgenden Nutzer profitieren.

## B.3 JUDGE — der eine teure Aufruf

**Eingabe:** eine Meldung × ein kanonisches Thema. Kein Nutzerbezug, keine
konkrete These — nur das Thema. Genau das macht das Ergebnis teilbar.

```
System: [B.0-Regeln]

Du beurteilst, wie eine Meldung ein einzelnes Beobachtungsthema berührt.

Thema: {canonical.label}
Themenbeschreibung: {canonical.description}
Typ: {qualitative | quantitative}

Meldung:
Titel: {title}
Quelle: {publisher}, {published_at}
Text: {body, max 4000 Zeichen}

Antworte als JSON:
{
  "touches_topic": bool,
  "verdict": "supports" | "mixed" | "weakens",
  "confidence": 0.0-1.0,
  "headline": "Ein Satz, was passiert ist. Faktisch, keine Wertung.",
  "why": "Zwei Sätze: warum das Thema berührt wird und in welche Richtung.",
  "counter": "Was gegen diese Einordnung spricht. PFLICHT. Wenn nichts dagegen
              spricht, nenne die Bedingung, unter der sich das ändern würde.",
  "magnitude": "Größenordnung mit Zahl, wenn die Meldung eine enthält,
                sonst 'keine Zahl genannt'. Niemals schätzen.",
  "decided_by": "Welches künftige Ereignis das endgültig klärt, mit Datum falls bekannt.",
  "quote": "Wörtliche Textstelle (max 25 Wörter), auf die sich das Urteil stützt."
}

Wenn touches_topic false ist, dürfen alle anderen Felder leer bleiben.
```

Vier Felder, die im Prototyp sichtbar sind und deshalb Pflicht sind:
- `counter` — ohne Gegenargument klingt die App wie ein Orakel.
- `magnitude` — „betrifft rund 18 % des Umsatzes“ statt „betrifft die Nachfrage“.
  Ohne Zahl bleibt jede Erklärung Gefühl. **Nie schätzen**, lieber „keine Zahl genannt“.
- `decided_by` — beantwortet „was soll ich als Nächstes beobachten“.
- `quote` — Nachweis. Immer speichern, auch wenn die UI ihn zunächst nicht zeigt.

**Quantitative Themen laufen nicht durch den JUDGE.** Dort wird die Regel gegen
die Kennzahl geprüft, das Urteil ergibt sich mechanisch:
```
rule: gross_margin > 70
Wert 73,0 und fallend → risk    (Puffer < 5 % relativ)
Wert 68,0             → weakens
Wert 76,0 und steigend → supports
kein neuer Wert im Zeitraum → quiet
```
Nur die `why`-Zeile darf per LLM formuliert werden, und nur einmal pro Kennzahl-
Aktualisierung — nicht pro Nutzer.

## B.4 FAN-OUT — Richtung berücksichtigen

Kein Prompt, aber die entscheidende Regel:

```python
# Ein topic_verdict wird zum Signal für jeden Watchpoint mit diesem Thema.
# Bei direction == "down" kehrt sich supports/weakens UM.

def verdict_for(thesis, topic_verdict):
    v = topic_verdict.verdict
    if thesis.direction == "down" and v in ("supports", "weakens"):
        return "weakens" if v == "supports" else "supports"
    return v
```

Grund: „Der Rechenzentrum-Umsatz wächst“ stützt eine Aufwärtsthese und schwächt
eine Abwärtsthese. Dasselbe Faktum, gespiegeltes Urteil. Ohne diese Zeile sind
Negativ-Thesen falsch bewertet — deshalb steht `direction` von Anfang an im Modell.

## B.5 TASK RUNNER

Eine Task ist ein Nutzerauftrag, der **keine Meldung voraussetzt** — „vergleiche
CapEx-Wachstum mit Azure-Umsatz“ steht in keiner News, das muss jemand nachrechnen.
Genau hier entsteht Wert, den kein Feed liefert.

```
System: [B.0-Regeln]

Du führst einen wiederkehrenden Prüfauftrag aus.

Auftrag des Nutzers: "{task.instruction}"
Bezug: {watchpoint.name} | {thesis.text bei target_type=thesis}
Zeitraum: seit {task.last_run_at}

Verfügbares Material:
- Meldungen im Zeitraum: {liste, max 15, gekürzt}
- Kennzahlen: {relevante metrics mit Werten und Vorperiode}
- Letztes Ergebnis dieser Task: {previous.summary}

Antworte als JSON:
{
  "has_finding": bool,
  "changed_since_last_run": bool,
  "verdict": "supports" | "mixed" | "weakens",
  "confidence": 0.0-1.0,
  "headline": "Ein Satz Ergebnis.",
  "why": "Zwei Sätze Begründung mit Zahl, wenn vorhanden.",
  "counter": "Was gegen diese Einordnung spricht.",
  "sources": ["url", ...]
}

Wenn sich nichts geändert hat: has_finding = false.
Der Nutzer erwartet KEINE tägliche Antwort, sondern eine Antwort bei Änderung.
```

**Kein Befund → kein Signal.** Nur `last_run_at` wird gesetzt. Das steht auch in
der UI („Nur bei einer Änderung entsteht ein Signal“) — Erwartung und Verhalten
müssen übereinstimmen, sonst baut sich Enttäuschung auf.

Kostenklassen:
| target | rhythm | Klasse | Bemerkung |
|---|---|---|---|
| watchpoint | weekly/monthly/event | `low` | ein Thema, seltener Lauf |
| watchpoint | daily | `medium` | ein Thema, täglich |
| thesis | weekly/monthly/event | `medium` | alle Watchpoints, seltener |
| thesis | daily | `high` | **einziger wirklich teurer Fall** |

Limits gehören an diese Tabelle, nicht an die reine Anzahl der Tasks.

## B.6 MORNING BRIEF

Kein LLM für Zähler und Health (reine SQL-Aggregation, siehe `SPEC.md` §4).

Ein einziger LLM-Aufruf pro Nutzer und Tag für `next_up`:
```
System: [B.0-Regeln]
Formuliere EINEN Satz: was soll der Nutzer als Nächstes beobachten?

Thesen mit Zustand: {liste}
Bekannte Termine: {earnings, investor days, regulatorische Fristen}
Signale von heute: {liste, gekürzt}

Regeln:
- Nenne genau ein Ereignis mit Datum.
- Nenne die Größe, die entscheidet, nicht das Ereignis allein.
  Gut:    "NVDA berichtet am 18. November. Entscheidend ist die Bruttomarge,
           nicht der Umsatz."
  Falsch: "NVDA berichtet am 18. November."
- Keine Empfehlung, keine Prognose.
```

Push nur bei `verdict = weakens` (Einstellung des Nutzers). Ein „stützt“ ist keine
Unterbrechung wert — das liest man beim nächsten Öffnen.

## B.7 RESOLUTION

Läuft, wenn `horizon_end` erreicht oder `falsify_rule` erfüllt ist.

```
System: [B.0-Regeln]
Bilanziere eine abgelaufene These. Der Nutzer will wissen, was hielt und was nicht.

These: {text}
Richtung: {direction} | Horizont: {created_at} bis {horizon_end}
Watchpoint-Verläufe: {je Watchpoint: Zustände im Zeitverlauf, Signale}
Erwartung des Nutzers: {expectation_pct} | Tatsächlich: {actual_pct}

Antworte als JSON:
{
  "outcome": "correct" | "partly" | "wrong",
  "watchpoint_outcomes": [{"watchpoint_id": "...", "held": bool, "note": "eine Zeile"}],
  "summary": "Zwei Sätze: was am Kern hielt, was nicht. Sachlich, nicht tröstend,
              nicht belehrend."
}

outcome bemisst sich an den WATCHPOINTS, nicht am Kurs.
Eine These kann richtig gewesen sein, obwohl der Kurs fiel — und umgekehrt.
```

Der letzte Satz ist der Kern des Produkts. Wer `outcome` am Kurs bemisst, baut
eine Tippspiel-App.

---

# TEIL C — Nutzererlebnis: was schiefgeht und was es verhindert

Kein Anforderungskatalog. Beobachtete Bruchstellen und die Gegenmaßnahme, die
im Prototyp schon steckt oder noch fehlt.

## C.1 „Ich weiß nicht, was ich schreiben soll“

**Bruch:** Erstnutzung. Wer keine These formulieren kann, ist verloren — und
das ist der Normalfall, nicht die Ausnahme.

**Gegenmaßnahme:** Drei vorgefertigte Thesen pro Company mit Anteilsangabe
(„von 58 % der Nutzer gewählt“) plus „Eigene These schreiben“ mit konkretem
Beispieltext. Die Vorschläge sind das wichtigste Onboarding-Feature, kein Komfort.

**Backend-Folge:** `thesis_templates` je Company sind kuratiert, nicht generiert.
Beim MVP von Hand für das Startuniversum. Vorlagen tragen fertige
`suggested_watchpoints` (kanonische IDs) — deshalb sind Template-Thesen sofort
auswertbar und kosten beim Anlegen nichts.

## C.2 „Ich habe es angelegt und es passiert nichts“

**Bruch:** Tag 2 bis 5. Keine Meldung trifft einen Watchpoint. Die App wirkt tot.

**Gegenmaßnahme:** Der Leerzustand ist die Vertrauensaussage, nicht die Ausrede:
„Stox hat heute 214 Meldungen zu deinen Unternehmen gelesen. Keine betrifft einen
deiner Watchpoints.“ Dazu die nächsten Termine.

**Backend-Folge:** `screened` (Anzahl geprüfter Meldungen) muss gezählt und
gespeichert werden, auch wenn nichts durchkommt. Ohne diese Zahl gibt es keinen
Leerzustand, der Vertrauen schafft. → `daily_stats(user_id, date, screened, matched)`

## C.3 „Das Urteil ist falsch“

**Bruch:** Ein Fehlurteil bleibt hängen, zwölf Treffer nicht. Zwei Fehler
hintereinander und der Nutzer glaubt nichts mehr.

**Gegenmaßnahme:** Quelle und Zitat immer verfügbar, `counter` in jeder Vertiefung,
„Bewertung stimmt nicht“ an jeder Karte, Confidence-Schwelle statt Vollständigkeit.

**Backend-Folge:** Feedback speichert den vollen Kontext (Signal, Watchpoint-Text,
Zitat, erzeugtes Urteil, `prompt_version`) — das ist dein Trainingsdatensatz und
der einzige Weg, Qualität messbar zu verbessern.
→ Offene Entscheidung: ab welcher Menge negativer Rückmeldungen zu einem
  `canonical_id` wird das Thema zur manuellen Prüfung markiert?

## C.4 „Zu viel Lärm“ — der Grund, warum Nutzer kommen

**Bruch:** Wenn der Signale-Tab wie ein News-Feed aussieht, ist der Wedge weg.

**Gegenmaßnahme:** Nur Signale mit Watchpoint-Bezug. Die übrigen 210 Meldungen
als eine ruhige, aufklappbare Zeile — sichtbar, aber nicht gleichberechtigt.
Kein zweiter, ungefilterter Feed.

**Backend-Folge:** Verworfene Meldungen werden gezählt und mit Verwerfungsgrund
gespeichert (`relevance` / `no_topic_match` / `low_confidence`). Das ist zugleich
das Diagnosewerkzeug: kippt das Verhältnis, stimmt etwas mit der Schwelle nicht.

## C.5 „Ich verstehe nicht, warum es mich betrifft“

**Bruch:** Der Zusammenhang zwischen Meldung und eigener Begründung bleibt
behauptet. Der Nutzer nickt, versteht aber nichts.

**Gegenmaßnahme:** Zwei Ebenen — kurzes `why` in der Karte, Vertiefung auf Tippen
(Was passiert ist / Warum es die These betrifft / Was dagegen spricht / Quellen).
`magnitude` und `decided_by` machen aus einer Aussage eine prüfbare Aussage.

**Nicht** „Explain by AI“ nennen: die Kurzbegründung ist bereits KI-Arbeit, ein
solches Etikett würde den Rest entwerten.

## C.6 „Bin ich zu spät?“

**Bruch:** Ein Signal kommt, der Kurs hat längst reagiert. Ohne Einordnung fühlt
sich das nach Versagen der App an.

**Gegenmaßnahme:** `market_reaction` mit „Deine Lesart: Bestätigt / Markt sah es
anders / Läuft“. Der Chart ist eine Chronik, kein Handelssignal — und die
Uneinigkeit ist die interessante Information, nicht der Fehler.

**Backend-Folge:** `reaction`-Job trägt nach Ablauf des Fensters nach. Bis dahin
zeigt die UI ehrlich „noch offen“.
→ Offene Entscheidung: `agrees` mechanisch aus dem Vorzeichen (weakens + Kurs
  fällt = bestätigt) oder differenzierter gegen einen Sektorindex? Mechanisch ist
  erklärbar, index-relativ ist genauer. Für das MVP: mechanisch.

## C.7 „Meine These ist abgelaufen und niemand sagt mir was“

**Bruch:** Ohne Auflösung gibt es keine Trefferquote, keinen Track Record, keine
Community-Ansicht mit Wert — und der emotional stärkste Moment fehlt.

**Gegenmaßnahme:** Karte auf „Heute“, Sheet mit Erwartung gegen Ergebnis,
Watchpoint-Bilanz, Entscheidung verlängern / anpassen / schließen.

**Backend-Folge:** `resolve`-Job täglich. `horizon_end` und `falsify_condition`
sind Pflichtfelder im Anlegen-Flow — nachträglich ist das eine Migration über
alle Bestandsthesen.

## C.8 „Ich habe eine Aufgabe erstellt und höre nichts“

**Bruch:** „Täglich“ suggeriert eine tägliche Antwort.

**Gegenmaßnahme:** Der Hinweis unter der Rhythmus-Auswahl sagt es vorab: „Nur bei
einer Änderung entsteht ein Signal.“ Dazu in der Task-Zeile „letztes Ergebnis
1. Sep“ — sichtbarer Beweis, dass die Aufgabe läuft.

→ Offene Entscheidung: braucht es einen wöchentlichen Sammelhinweis
  („3 Aufgaben liefen, keine Änderung“)? Ich tendiere zu ja, weil Stille sonst
  wie Defekt wirkt.

## C.9 Kosten sind ein UX-Problem

Die Task-Konfiguration ist die **einzige** Stelle, an der der Nutzer bestimmt,
wie oft du rechnest. Deshalb liegen die Limits dort (siehe B.5), nicht bei der
Anzahl der Watchpoints — und Unternehmen sind die Zähleinheit im Abo, weil das
jeder sofort versteht und direkt mit den Kosten korreliert.

---

# TEIL D — Offene Entscheidungen

Reihenfolge = Dringlichkeit. Die ersten drei blockieren den Bau.

## D.1 Quellen (blockierend)

| Bedarf | Kandidaten | Frage |
|---|---|---|
| News | Kommerzielle News-APIs, RSS der Primärquellen | Lizenz für Volltext-Verarbeitung? Reicht Titel + Anriss für `JUDGE`? |
| Filings | SEC EDGAR (frei, US), nationale Register für EU | ASML ist Euronext — wo kommen die Berichte her? |
| Transkripte | kommerziell, teuer | Oder nur Pressemitteilung zum Call, plus Aufnahme später? |
| Kennzahlen | kommerziell | Für quantitative Watchpoints unverzichtbar. Startuniversum klein halten. |
| Kurse | breit verfügbar | Nur Tagesschluss nötig, kein Intraday. Deutlich günstiger. |

**Empfehlung:** Startuniversum auf 50–100 große Titel begrenzen und dort
vollständige Daten haben, statt breit und lückenhaft. Der Chart braucht nur
Tagesschlusskurse.

## D.2 Modellwahl und Kostenrechnung (blockierend)

Zu klären mit echten Zahlen, nicht Schätzungen:
- Welches Modell für `JUDGE`? Ein kleineres für `touches_topic`, ein größeres nur
  für die Bewertung? (Zwei Stufen sparen viel, kosten Latenz — die ist hier egal,
  weil der Lauf nachts passiert.)
- Kosten pro Tag: `Meldungen nach Relevance × Themen pro Meldung × Tokens`.
  Bei 25 Kandidaten × 1,5 Themen ≈ 38 Aufrufe pro Company und Tag.
  Bei 100 Companies im Universum ≈ 3.800 Aufrufe täglich, **unabhängig von der
  Nutzerzahl**. Das ist die Zahl, an der das Geschäftsmodell hängt.
- Embedding-Modell festlegen und **nie wechseln** ohne Neuberechnung aller
  `canonical_watchpoints`.

## D.3 Zeitzonen und „heute“ (blockierend, klein)

Der Morgenlauf ist 07:00 — in welcher Zeitzone? Bei ASML (Euronext) und NVDA
(Nasdaq) in einer Watchlist liegen die Handelstage auseinander.
**Empfehlung:** Nutzer-Zeitzone für den Brief, Börsen-Zeitzone für
`market_reaction` und Termine. Beide Felder getrennt speichern.

## D.4 Konten und Abo (vor Launch, nicht vor Bau)

MVP läuft ohne Anmeldung — aber Thesen müssen ein Gerät überleben.
**Empfehlung:** Sign in with Apple, sonst nichts. Abo über StoreKit 2, Prüfung
serverseitig. Kein eigenes Passwortsystem.

## D.5 Fragen, die ich für dich noch offen sehe

1. **Mehrere Thesen pro Company** — im Modell erlaubt, in der UI vorbereitet
   („Zweite These hinzufügen“). Zählt das im Abo als ein Unternehmen oder zwei?
   Kostenseitig ist es fast gratis (gleiche Themen), also: ein Unternehmen.
2. **Watchpoint-Zustand `quiet`** — nach wie vielen Tagen ohne Signal fällt ein
   Watchpoint auf `quiet` zurück? Vorschlag: 45 Tage bei qualitativ, ein Quartal
   bei quantitativ.
3. **Health-Historie** — speichern wir den Score täglich, um „−11 in 14 Tagen“ zu
   zeigen? Ja, eine Zeile pro Thesis und Tag. Billig und später Grundlage für
   Mustererkennung.
4. **Was passiert bei Thesen-Bearbeitung?** Ändert der Nutzer den Text, sind alte
   Signale streng genommen gegen eine andere These bewertet. Vorschlag:
   Thesis-Versionen mit `revision`, Signale hängen an der Revision. Wichtig für
   die Ehrlichkeit der Trefferquote.
5. **Sprache der Signale** — Quellen sind meist englisch, Ausgabe deutsch. Der
   `JUDGE` übersetzt implizit. `quote` bleibt im Original, damit der Nachweis
   überprüfbar ist.

---

# TEIL E — Was zuerst gebaut wird

1. **`classify` als Skript, ohne API und ohne App.** Drei Companies, 30 Meldungen
   aus einer Woche, Urteile in eine CSV, von Hand gegenlesen. Ziel: mindestens
   25 von 30 nachvollziehbar. Erst danach weiter.
2. Datenbank + `ingest` + `fanout`, noch ohne Endpunkte.
3. `GET /home`, `GET /watchlist`, `GET /signals`, `GET /companies/{id}/detail` —
   die vier Endpunkte, die die App zum Leben braucht.
4. `brief` und `resolve` als Jobs.
5. `tasks` zuletzt: es ist das stärkste Feature, aber es setzt eine funktionierende
   Pipeline voraus.

Schreibende Endpunkte (`POST /theses` etc.) können warten — die App kann anfangs
gegen `MockStore` schreiben.
