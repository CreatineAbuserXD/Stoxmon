# Stox — Spezifikation MVP

Stand: 8. September 2026 · Referenz: `Stox v7.dc.html` (Prototyp, klickbar)

---

## 1. Produktkern in einem Satz

Der Nutzer schreibt auf, **warum** er an ein Unternehmen glaubt. Stox zerlegt diese
Begründung in prüfbare Teile (Watchpoints) und bewertet jede neue Information
gegen genau diese Teile.

Die App bewertet **nie** ein Investment und spricht **nie** eine Empfehlung aus.
Sie bewertet ausschließlich die These des Nutzers.

---

## 2. Objektmodell

```
Company 1 ── n Thesis 1 ── n Watchpoint 1 ── n Signal
                  │                 │
                  │                 └── n Task        (Auftrag an den Agenten)
                  ├── 0..1 Position                   (optional, Kontext)
                  └── 0..1 Resolution                 (bei Ablauf/Abbruch)
```

Regeln:
- Eine Company ohne Thesis existiert im Produkt nicht.
- Ein Watchpoint gehört immer genau einer Thesis.
- Ein Signal trifft **genau einen** Watchpoint als Hauptwirkung. Nebenwirkungen
  auf weitere Watchpoints sind erlaubt, aber nachgeordnet (`secondary_watchpoint_ids`).
- Eine Task hängt entweder an einer Thesis (ganzheitlich) oder an einem Watchpoint.

---

## 3. Entitäten

### Company
| Feld | Typ | Anmerkung |
|---|---|---|
| `id` | UUID | |
| `ticker` | str | z. B. `NVDA` |
| `isin` | str \| null | Schlüssel für Logo-Zuordnung |
| `name` | str | `Nvidia` |
| `exchange` | str | `Nasdaq` |
| `sector` | str | `Halbleiter` |
| `logo_url` | str \| null | serverseitig gehostet |

### Thesis
| Feld | Typ | Anmerkung |
|---|---|---|
| `id` | UUID | |
| `company_id` | UUID | |
| `text` | str (≤ 280) | die Begründung, wörtlich vom Nutzer |
| `direction` | enum | `up` \| `down` — **Negativ-These ist Teil des Modells** |
| `horizon_end` | date \| null | `null` = offen |
| `falsify_condition` | str | Klartext; bei quantitativer Fassung zusätzlich `falsify_rule` |
| `falsify_rule` | obj \| null | `{metric, operator, value, consecutive_quarters}` |
| `expectation_pct` | int \| null | optional, **nie** mit Health verrechnet |
| `origin` | enum | `template` \| `custom` — Template-Thesen sind cachebar |
| `template_id` | UUID \| null | Verweis auf kanonische Vorlage |
| `created_at` | datetime | Ankerpunkt der gestrichelten Linie im Chart |
| `status` | enum | `active` \| `resolved` \| `paused` |

### Watchpoint
| Feld | Typ | Anmerkung |
|---|---|---|
| `id` | UUID | |
| `thesis_id` | UUID | |
| `name` | str | `Bruttomarge über 70 %` |
| `kind` | enum | `qualitative` (KI-Prüfung) \| `quantitative` (Regel) |
| `rule` | obj \| null | nur bei `quantitative`: `{metric, operator, value, unit}` |
| `canonical_id` | UUID | **Schlüssel fürs Caching**, siehe §7 |
| `state` | enum | `supports` \| `risk` \| `weakens` \| `quiet` |
| `state_note` | str | eine Zeile Begründung, z. B. `73,0 %, leicht rückläufig` |
| `order` | int | feste Reihenfolge — die Punkte-Reihe darf nicht springen |

### Signal
| Feld | Typ | Anmerkung |
|---|---|---|
| `id` | UUID | |
| `watchpoint_id` | UUID | Hauptwirkung |
| `secondary_watchpoint_ids` | UUID[] | |
| `kind` | enum | `news` \| `earnings_call` \| `fundamentals` \| `analyst` \| `task_result` |
| `verdict` | enum | `supports` \| `mixed` \| `weakens` |
| `headline` | str | ein Satz, was passiert ist |
| `why` | str | zwei Zeilen: warum es die These betrifft |
| `deep` | obj \| null | Vertiefung, siehe unten |
| `sources` | Source[] | mind. 1 |
| `occurred_at` | datetime | |
| `market_reaction` | obj \| null | `{window_days, pct, agrees}` — erst nach Ablauf des Fensters |
| `user_feedback` | enum \| null | `agree` \| `disagree` — **Trainingsmaterial**, siehe §8 |

`deep`:
```json
{
  "what": "Was passiert ist, faktisch.",
  "impact": "Warum es diese These betrifft.",
  "counter": "Was dagegen spricht — PFLICHTFELD.",
  "magnitude": "betrifft rund 18 % des Systemumsatzes",
  "decided_by": "Quartalsbericht 18. November, Margenzeile"
}
```
`counter` ist Pflicht. Ohne Gegenargument klingt die App wie ein Orakel — und verliert
beim ersten Fehlurteil das Vertrauen dauerhaft.

### Source
`{title, publisher, url, published_at, quote}` — `quote` ist die Textstelle, auf die
sich das Urteil stützt. Immer speichern, auch wenn die UI sie zunächst nicht zeigt.

### Task
| Feld | Typ | Anmerkung |
|---|---|---|
| `id` | UUID | |
| `target_type` | enum | `thesis` \| `watchpoint` |
| `target_id` | UUID | |
| `instruction` | str | Klartext-Auftrag des Nutzers |
| `rhythm` | enum | `daily` \| `weekly` \| `monthly` \| `event` |
| `event_trigger` | enum \| null | `earnings_report` \| `earnings_call` \| `analyst_note` \| `price_move` |
| `event_params` | obj \| null | z. B. `{threshold_pct: 8}` |
| `active` | bool | pausieren statt löschen |
| `last_run_at` | datetime \| null | |
| `next_run_at` | datetime \| null | |
| `cost_class` | enum | abgeleitet: `thesis`+`daily` = teuer, siehe §9 |

Ergebnis einer Task ist **immer ein Signal** mit `kind = task_result` — keine zweite
Liste, kein zweiter Ort für Erkenntnisse.
Läuft eine Task ohne Befund, entsteht **kein** Signal (nur `last_run_at` wird gesetzt).

### Position (optional)
`{thesis_id, quantity, avg_price, currency}` — dient **nur** als Kontext für einen Satz
(„Du liegst im Plus, aber der Grund für den Kauf ist schwächer geworden“).
Kein Depotwert, keine Gesamtperformance, kein eigener Screen.

### Resolution
| Feld | Typ | Anmerkung |
|---|---|---|
| `thesis_id` | UUID | |
| `resolved_at` | datetime | |
| `trigger` | enum | `horizon_reached` \| `falsify_hit` \| `manual` |
| `outcome` | enum | `correct` \| `partly` \| `wrong` |
| `expectation_pct` / `actual_pct` | int | Vergleich für die Bilanz |
| `watchpoint_outcomes` | `[{watchpoint_id, held: bool, note}]` | |
| `summary` | str | zwei Sätze: was hielt, was nicht |
| `next_action` | enum | `extended` \| `revised` \| `closed` |

---

## 4. Zustände und Farben

| Zustand | Wort (DE) | Farbe | Bedeutung |
|---|---|---|---|
| `supports` | Stützt | `#1E8A5F` | Information bestätigt den Watchpoint |
| `risk` / `mixed` | Risiko / Gemischt | `#B8801F` | zweischneidig oder Puffer schrumpft |
| `weakens` | Schwächt | `#C4553C` | Information widerspricht dem Watchpoint |
| `quiet` | Still | `#C7C7CC` | keine neuen Daten im Zeitraum |

**Grün und Rot bedeuten niemals Kursrichtung**, ausschließlich Thesenwirkung.
Bei `direction = down` bedeutet „Stützt“ korrekt, dass die *Abwärtsthese* bestätigt wird.

### Thesis Health
Wird **serverseitig** berechnet, nie in der App:

```
score = gewichteter Anteil stützender Watchpoints
  supports = 1.0 | risk = 0.5 | weakens = 0.0 | quiet = neutral (fällt aus dem Nenner)

Wort:
  ≥ 0.75 und kein weakens        → "Intakt"
  ≥ 0.60 und genau ein weakens   → "Intakt, ein Riss"
  ≥ 0.40                         → "Unter Druck"
  <  0.40                        → "Gebrochen"
```
Die Kursentwicklung geht **nicht** in den Score ein. Sonst bewertet sich der Nutzer
am Kurs statt an seinen Watchpoints — genau der Unterschied zu jeder anderen App.

---

## 5. API

Alle Antworten JSON, alle Zeiten ISO 8601 UTC.
Die App **formatiert nur** — sie entscheidet nichts (kein Score, keine Zustandslogik).

| Methode | Pfad | Zweck |
|---|---|---|
| `GET` | `/companies/search?q=` | Ticker-/Namenssuche für Schritt 1 |
| `GET` | `/companies/{id}/thesis-templates` | vorgefertigte Thesen + Watchpoint-Vorschläge |
| `POST` | `/theses` | Thesis anlegen (Schritte 2–4 zusammen) |
| `PATCH` | `/theses/{id}` | bearbeiten |
| `POST` | `/theses/{id}/watchpoints` | Watchpoint ergänzen |
| `GET` | `/home` | Tagesbriefing: fällige Resolutions, Tagesbilanz, Thesenzustände, Ausblick |
| `GET` | `/watchlist` | Karten mit Kurszeile, Kurz-These, Punkten |
| `GET` | `/signals?since=` | Chronik über alle Thesen + Zähler unerheblicher Meldungen |
| `GET` | `/companies/{id}/detail` | Chart-Punkte, Watchpoints, Signale, Tasks, Konsens |
| `POST` | `/signals/{id}/feedback` | `{verdict: agree\|disagree}` |
| `POST` | `/tasks` · `PATCH` `/tasks/{id}` | anlegen, pausieren |
| `POST` | `/theses/{id}/resolve` | `{next_action, new_horizon?}` |

### `GET /home` (die wichtigste Antwort)
```json
{
  "date": "2026-09-08",
  "checked_at": "2026-09-08T07:00:00Z",
  "due_resolutions": [{"thesis_id": "...", "company": "Nvidia", "reason": "horizon_reached"}],
  "signal_tally": {"supports": 1, "mixed": 2, "weakens": 1, "screened": 214},
  "thesis_states": [{"thesis_id": "...", "company": "Nvidia", "health_word": "Unter Druck",
                     "watchpoint_states": ["supports","risk","weakens","weakens","quiet"]}],
  "next_up": "NVDA berichtet am 18. November. Entscheidend ist die Bruttomarge, nicht der Umsatz."
}
```

---

## 6. Signal-Pipeline

```
Quellen (News-API, Filings, Transkripte, Kennzahlen)
   ↓ Deduplizierung nach Kernfaktum
Vorfilter: betrifft die Meldung eine Company mit aktiver Thesis?     [billig, ohne LLM]
   ↓
Themenzuordnung: welche kanonischen Watchpoints berührt sie?          [Embedding]
   ↓
Bewertung je Thema: supports / mixed / weakens + why + counter        [LLM, EINMAL pro Thema]
   ↓
Fan-out an alle Nutzer mit diesem kanonischen Watchpoint              [Datenbank, kostenlos]
   ↓
Personalisierung nur für das wichtigste Tagessignal                   [LLM, optional]
```

Quantitative Watchpoints laufen **nicht** durch das LLM: die Regel wird gegen die
Kennzahl geprüft, das Urteil ergibt sich mechanisch.

Grundsatz: **Präzision vor Vollständigkeit.** Lieber drei sichere Signale als zehn
wackelige. Zwei Fehlurteile hintereinander kosten das Vertrauen dauerhaft.

---

## 7. Kanonische Watchpoints (Kostenhebel)

Pro Company existiert eine wachsende Liste kanonischer Themen. Jeder frei
geschriebene Watchpoint wird per Embedding auf das nächstliegende Thema abgebildet
(Schwellwert ~0,82 Cosinus); darunter entsteht ein neues kanonisches Thema, von dem
alle folgenden Nutzer profitieren.

Folge: Kosten skalieren mit **Meldungen × Themen pro Company**, nicht mit Nutzerzahl.
Bei hundert Nvidia-Thesen fallen 80 % der Watchpoints auf dieselben zehn Themen.

Kaltstart: Companies aus einem vordefinierten Universum sind sofort verfügbar,
alles darüber hinaus „ab morgen früh“ — passt zum Tagesrhythmus, hält Kosten planbar.

---

## 8. Feedback

Jede Signalkarte trägt „Bewertung stimmt nicht“. Das Feedback ist kein Höflichkeits-
feature, sondern der Trainingsdatensatz für die Zuordnung. Speichern mit vollem
Kontext: Signal, Watchpoint-Text, Quellen-Zitat, erzeugtes Urteil.

---

## 9. Limits und Kosten

Gezählt werden **Unternehmen**, nicht Watchpoints — das versteht jeder sofort und
korreliert direkt mit den Kosten.

| | Basis (~10 €/Monat) | Pro |
|---|---|---|
| Unternehmen | 5 | 20 |
| Watchpoints je These | 6 | 6 |
| Eigene Watchpoints außerhalb der Vorschläge | – | ✓ |
| Tasks | 3, kein `daily` auf `thesis` | 15 |
| Marktmeinung, Positionierung, Rückfragen | – | ✓ |

`target_type = thesis` + `rhythm = daily` ist der einzige wirklich teure Fall
(alle Watchpoints werden neu bewertet). Dort ansetzen, nicht bei der Anzahl.

---

## 10. Was bewusst NICHT gebaut wird

- Kein Broker-Sync, kein Depotwert, keine Gesamtperformance.
- Keine Kursziele, keine Empfehlungen, keine Candlestick-Charts, keine Indikatoren.
- Kein ungefilterter News-Feed neben den Signalen (nur die Zeile „210 Meldungen ohne
  Thesenbezug“).
- Kein freies Chatfenster. Rückfragen später nur gebunden an ein konkretes Signal.
- Kein Watchpoints-Tab: ein Watchpoint ist ohne seine These nicht lesbar.

## 11. Geplant, nach dem MVP

1. Auflösungs-Historie mit Trefferquote und Mustererkennung
   („Du überschätzt Wechselkosten — das war die dritte These dieser Art“).
2. Fundamentaldaten als vierter Reiter — jede Kennzahl mit Watchpoint-Bezug,
   wo einer existiert.
3. Marktmeinung in der Tiefe + Positionierung großer Investoren als Signaltyp
   (nur mit Begründung, die auf einen Watchpoint zeigt; nie „X hat gekauft“).
4. Thesen-Cluster über alle Nutzer, mit echter Trefferquote statt Meinung.
5. Depot-Import als Onboarding (CSV) → Thesenvorschlag je Position.
6. Web-Frontend gegen dieselbe API: Archiv und Übersicht, nicht Eingabe.
