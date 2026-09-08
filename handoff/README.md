# Stox — Übergabepaket

Für die Umsetzung in Xcode (SwiftUI) und PyCharm (FastAPI).
Grundlage für ein eigenes `AGENTS.md`.

| Datei | Inhalt |
|---|---|
| `SPEC.md` | Objektmodell, Entitäten, Zustandslogik, API, Limits |
| `PIPELINE.md` | Architektur, alle Prompts im Wortlaut, Pain Points mit Backend-Folge, offene Entscheidungen |
| `mock.json` | Vollständige Beispieldaten für NVDA, MSFT, ASML — Struktur = spätere Serverantwort |
| `DESIGN.md` | Farben, Typografie, Maße, Komponenten, Navigation, SwiftUI-Skelett |
| `../Stox iOS MVP v7.html` | Klickbarer Prototyp — **die visuelle Wahrheit**, nicht `DESIGN.md` |
| `../logos/` | Firmenlogos (nvda, msft, asml) |

## Reihenfolge

1. **`classify` als Skript, ohne App und ohne API.** 30 Meldungen durch die
   Pipeline, Urteile von Hand gegenlesen (`PIPELINE.md` Teil E). Das Frontend ist
   im Prototyp geklärt — ob der Kern trägt, weiß noch niemand.
2. **App gegen `MockStore`** bauen, zwei Wochen selbst benutzen. Dabei fällt auf,
   welche Felder fehlen, solange sie in keiner Datenbank stehen.
3. **`APIStore`** anschließen. Ein Austausch beim App-Start.

Fürs Design Screen für Screen arbeiten, nicht alles auf einmal: `DESIGN.md` plus
Screenshot des einen Screens, danach gegen den Prototyp vergleichen.

## Drei Regeln, die man später teuer bezahlt

- **Die App entscheidet nichts.** Health-Score, Zustandswort und Verdict kommen
  fertig vom Server, sonst rechnet Web später anders als iOS.
- **Der Modell-Schlüssel gehört ausschließlich auf den Server.**
- **`direction` und `canonical_id` von Anfang an mitschreiben.** Ohne `direction`
  keine Negativ-Thesen, ohne `canonical_id` kein Caching — beides nachträglich
  eine Migration über alle Bestandsdaten.

## Was blockierend offen ist

Quellenauswahl, Modellwahl mit echter Kostenrechnung, Zeitzonen-Definition von
„heute“ — Details in `PIPELINE.md` Teil D.

## Stack

FastAPI · Postgres mit `pgvector` · APScheduler für Morgenläufe und Tasks ·
Railway oder Fly.io. Type Hints überall, Pydantic für alle Modelle, `mypy` im Editor.

```
stox/
├── handoff/
├── backend/
│   ├── app/            api/ · models/ · pipeline/ · jobs/
│   ├── tests/
│   ├── pyproject.toml
│   └── .env.example    Schlüssel nie eingecheckt
└── frontend/
    └── ios/            Stox.xcodeproj · Stox/{App,Features,Components,Stores,Resources}
```
