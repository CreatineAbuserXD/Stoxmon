# Stox — Design-Referenz für SwiftUI

Quelle: `Stox v7.dc.html`. Werte sind CSS-Pixel = SwiftUI-Punkte (1:1).

---

## Farben (Asset-Katalog, je mit Dark-Paar)

| Name | Light | Verwendung |
|---|---|---|
| `canvas` | `#F2F2F7` | Bildschirmhintergrund (= `systemGroupedBackground`) |
| `surface` | `#FFFFFF` | Karten, Listenzeilen |
| `surfaceSunken` | `#F2F2F7` | Kacheln **auf** einer Karte |
| `tint` | `#2F6BD4` | Aktionen, aktiver Tab, Auswahl |
| `supports` | `#1E8A5F` | Watchpoint stützt |
| `risk` | `#B8801F` | Risiko / gemischt |
| `weakens` | `#C4553C` | schwächt |
| `quiet` | `#C7C7CC` | keine Daten |
| `label` | `#1C1C1E` | Primärtext |
| `labelSecondary` | `rgba(60,60,67,0.60)` | Sekundärtext |
| `labelTertiary` | `rgba(60,60,67,0.45)` | Sektionslabels |
| `separator` | `rgba(60,60,67,0.10)` | Trennlinien in Karten |

Zwei Regeln:
1. Grün/Rot **nie** für Kursrichtung — nur Thesenwirkung.
2. Im Druck oder auf Weiß nie transparente Graustufen für Fließtext (Kontrast);
   dort `#63636A` verwenden.

---

## Typografie (System-Font, Dynamic Type)

| Rolle | Größe / Gewicht | SwiftUI |
|---|---|---|
| Large Title | 34 bold, `-0.03em` | `.largeTitle.bold()` |
| Screen-Untertitel | 14 regular, sekundär | `.subheadline` |
| Sheet-Titel | 22 semibold, `-0.02em` | `.title2.weight(.semibold)` |
| Kartentitel | 17 semibold, `-0.01em` | `.headline` |
| Listenzeile | 16 regular | `.body` |
| Signaltext | 15–16 regular, Zeilenhöhe 1.42 | `.body` + `.lineSpacing(3)` |
| Sektionslabel | 12.5 semibold, `labelTertiary`, `+0.02em` | `.caption.weight(.semibold)` |
| Meta / Datum | 13 regular, sekundär | `.footnote` |
| Tab-Label | 10.5 medium | Systemvorgabe |
| Kennzahl in Kachel | 17–19 semibold | `.title3.weight(.semibold)` |

Keine zweite Schriftfamilie. Kein Monospace (ab v2 entfernt — wirkte technisch statt nativ).

---

## Maße

| | Wert |
|---|---|
| Bildschirmrand | 20 |
| Kartenradius | 18 |
| Innenabstand Karte | 14–15 |
| Abstand zwischen Karten | 12 |
| Radius kleine Kachel (in Karte) | 10 |
| Radius Button / Eingabefeld | 14 |
| Radius Chip (Auswahl) | 12 |
| Radius Logo-Kachel | 11 (42 pt) · 9 (34/36 pt) · 8 (30 pt) · 13 (52 pt) |
| Zeilenhöhe Liste | 12–13 vertikal |
| Watchpoint-Punkt | 7 × 7, Radius voll |
| Health-Balken | Höhe 5–6, `Capsule()` |
| Chart-Höhe | 112 |
| Chart-Punkt | 12 (ausgewählt 15) + 2 pt weißer Rand |
| Schatten Karte | `0 1 3 rgba(0,0,0,0.04)` — sonst keine Schatten |

Tiefe entsteht durch Fläche, nicht durch Schatten oder Material.

---

## Komponenten

| Name | Inhalt |
|---|---|
| `CompanyCard` | Logo-Kachel, Name, Kurszeile (ohne Ticker), Urteilswort, Kurz-These, 5 Punkte, Watchpoint-Zeile |
| `LogoTile(size:)` | weiße Kachel, 1 pt Kontur, Logo auf ~72 % der Kantenlänge, `contentMode: .fit` |
| `HealthDots` | HStack aus 5 Punkten in **fester** Watchpoint-Reihenfolge |
| `HealthBars` | dieselbe Reihenfolge als Balken (Thesen-Karte) |
| `VerdictLabel` | farbiger Punkt + Wort, nie Farbe allein (Barrierefreiheit) |
| `SignalCard` | Typ-Label, Datum, Headline, `why`, Trennlinie, VerdictLabel + Watchpoint + Quellenzahl, aufklappbare Vertiefung |
| `SignalDeepBlock` | Was passiert ist / Warum es die These betrifft / **Was dagegen spricht** / Quellen / „Bewertung stimmt nicht“ |
| `WatchpointRow` | Zustandspunkt, Name, `kind` + Notiz, Zustandswort rechts; antippbar (filtert Chart) |
| `TaskRow` | Uhr-Symbol in Kachel, Auftragstext, Rhythmus + Ziel + letztes Ergebnis, Toggle (pausieren statt löschen) |
| `ThesisChart` | Linie + Verlaufsfläche, gestrichelte Linie bei `thesis.created_at`, Signalpunkte, Reichweiten-Umschalter 1J/3J/5J |
| `EventDetail` | unter dem Chart: Verdict, Watchpoint, Datum, Headline, Kacheln „Markt 30 Tage danach“ + „Deine Lesart“ |
| `ChoiceRow` | Kreis-Häkchen + Titel + Notiz — für Thesen-Vorschläge, Task-Ziel, Auflösungs-Optionen |
| `ChipRow` | Rhythmus, Horizont, Richtung, Erwartung |
| `SectionLabel` | Großbuchstaben-Label über Gruppen |
| `PrimaryButton` | volle Breite, `tint`, Radius 14, 17 semibold |

---

## Navigation

```
TabView
├── Heute        NavigationStack   Tagesbriefing
├── Watchlist    NavigationStack   Karten → CompanyDetail (push)
└── Signale      NavigationStack   Chronik → CompanyDetail (push, Segment „Signale“)

Sheets (über allen Tabs):
  AddThesisFlow    4 Schritte, .sheet mit eigenem NavigationStack
  AddWatchpoint    1 Seite
  CreateTask       1 Seite: Ziel · Rhythmus · (Ereignis) · Auftrag
  ResolveThesis    1 Seite, ausgelöst über die Karte auf „Heute“

Einstellungen: Zahnrad in der Navigationsleiste von „Heute“ (kein Tab).
```

Company-Detail-Segmente: `Picker(.segmented)` mit These / Watchpoints / Signale.

---

## Interaktionsdetails, die im Prototyp bewusst so sind

- **Druckzustand** auf jeder antippbaren Fläche (`opacity 0.55`).
- **Watchpoint-Reihenfolge ist fix** — sonst ist die Punktereihe über die Zeit nicht lesbar.
- **Chart-Filter**: Chip *oder* Watchpoint-Zeile antippen dimmt fremde Punkte auf 22 %.
- **Kurszeile auf Karten ohne Ticker-Präfix** (Logo + Name reichen), im Detail-Kopf mit.
- **„Genauer erklären“** statt „Explain by AI“: die Kurzbegründung ist bereits KI-Arbeit.
- **Aufgaben-Ergebnisse erscheinen als Signal**, nicht in einer eigenen Liste.
- Kein `scrollIndicator`, iOS-typisch ausgeblendet in Chip-Reihen.

---

## SwiftUI-Skelett

```swift
protocol ThesisStore {
    func home() async throws -> HomeBriefing
    func watchlist() async throws -> [CompanyCardModel]
    func signals(since: Date?) async throws -> SignalFeed
    func detail(companyID: String) async throws -> CompanyDetail
    func createThesis(_ draft: ThesisDraft) async throws -> Thesis
    func createTask(_ draft: TaskDraft) async throws -> Task
    func resolve(thesisID: String, action: ResolutionAction) async throws
    func sendFeedback(signalID: String, agrees: Bool) async throws
}

struct MockStore: ThesisStore { /* liest handoff/mock.json aus dem Bundle */ }
struct APIStore: ThesisStore  { /* FastAPI, gleiche Signaturen */ }
```

Views kennen **nur** das Protokoll. Der Umstieg auf das Backend ist ein Austausch
einer Zeile beim App-Start.

Die App **formatiert nur**. Health-Score, Zustandswort, Verdict und Reihenfolge
kommen fertig vom Server — sonst rechnet die spätere Web-Version anders als iOS.

---

## Barrierefreiheit

- Dynamic Type bis „Groß“ ohne Layoutbruch; Karten wachsen in der Höhe.
- Zustand nie nur über Farbe: Punkt **und** Wort.
- Trefferflächen ≥ 44 pt (Chart-Punkte brauchen eine unsichtbare Vergrößerung).
- `accessibilityLabel` für Chart-Punkte: „Schwächt, Custom-Silicon-Konkurrenz, Februar 2026“.
