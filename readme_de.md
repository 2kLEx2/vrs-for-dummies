# Counter-Strike Regional Standings

Dies ist Valves offizielles Modell zur Berechnung der Counter-Strike Regional Standings (VRS) — das System, das bestimmt, welche Teams direkt in die geschlossenen Qualifier eines Majors eingeladen werden.

---

## Was es macht

Teams sammeln Rankingpunkte, indem sie das ganze Jahr über Matches bei Drittanbieter-Events spielen. Die Standings werden genutzt, um Teams direkt in spätere Qualifikationsphasen einzuladen und so den Aufwand für Top-Teams zu reduzieren.

Das Modell ist darauf ausgelegt, **akkurat**, **schwer manipulierbar** und **transparent** zu sein.

---

## Wie das Ranking funktioniert

### Schritt 1 — Datenladen & Filterung

Bevor ein Ranking berechnet wird, werden die Matchdaten bereinigt:

- Nur **abgeschlossene Matches** werden verwendet (beide Teams müssen 5 Spieler haben)
- **Showmatches** werden ausgeschlossen
- **Laufende Events** werden ausgeschlossen — nur abgeschlossene Events zählen
- Nur als **Valve-ranked** markierte Matches werden berücksichtigt (ab Januar 2025)
- Es wird ein **rollierendes 6-Monats-Fenster** verwendet — ältere Matches fallen heraus
- Event-Preispools werden auf **$1.000.000** begrenzt

Qualifikationsketten werden ebenfalls verknüpft: Wenn ein Sieg bei Event A zur Teilnahme an Event B berechtigt, wird der Preiswert weitergegeben (50% pro Kettenglied).

---

### Schritt 2 — Roster-Tracking

Teams werden **nach Roster, nicht nach Org-Namen** verfolgt. Wenn 3 oder mehr Spieler eines früheren Rosters in einem neuen Team auftauchen, gilt es als dasselbe Roster — genauso wie bei Major-Qualifikationsregeln.

Die Region eines Teams wird durch die Staatsbürgerschaft der Spieler bestimmt (Mehrheitsprinzip).

---

### Schritt 3 — Seeding

Bevor Matches verarbeitet werden, erhält jedes Team einen initialen Seed-Wert basierend auf 4 Faktoren:

| Faktor | Was er misst |
|---|---|
| **Bounty Collected** | Wie renommiert die Gegner sind, die du besiegt hast |
| **Bounty Offered** | Wie renommiert du selbst bist (basierend auf deinen Preisgeldgewinnen) |
| **Opponent Network** | Wie viele verschiedene Teams deine besiegten Gegner selbst besiegt haben |
| **LAN Factor** | Wie viele deiner Siege auf LAN-Events errungen wurden |

Jeder Faktor wird relativ zum 5.-besten Team der Welt normalisiert (um extreme Ausreißer zu ignorieren). Nur die **Top-10-Ergebnisse** pro Faktor zählen — Matches bei niedrig dotierten Events schaden nie.

Ältere Ergebnisse zählen weniger — alle Faktoren werden mit der Zeit abgeschwächt.

Seed-Werte werden auf einen Startrang zwischen **400 und 2000** gemappt.

---

### Schritt 4 — Matches verarbeiten (Glicko-ELO)

Matches werden anschließend chronologisch wiederholt. Das verwendete Ratingsystem ist **Glicko-2**, aber mit einer festen Rating Deviation (RD = 75), was es effektiv wie ein Standard-ELO-System verhält.

Wichtige Werte:
- Startrang: **1500**
- Ein Rangunterschied von **400 Punkten** = ~90% erwartete Gewinnchance
- Jedes Match passt die Ratings beider Teams basierend auf dem Ergebnis vs. Erwartung an
- Matches werden nach **Informationsgehalt** gewichtet (Aktualitäts-Modifier)

---

### Schritt 5 — Rankings vergeben

Nach der Verarbeitung aller Matches:

- Teams benötigen **mindestens 5 gespielte Matches**, um in den Standings zu erscheinen
- Teams benötigen **mindestens 1 Sieg** gegen einen anderen gegner
- Teams werden global sortiert, dann in 3 regionale Standings aufgeteilt

**Regionen:**
| Region | Länder |
|---|---|
| **Europa** | EU + MENA |
| **Amerika** | NA + SA |
| **Asien-Pazifik** | AS + OC |

---

## Ausgabe

Das Modell generiert Markdown-Standings-Dateien:

- `live/YYYY/` — wöchentlich aktualisiert während aktiver Phasen
- `invitation/YYYY/` — Snapshot zu Monatsbeginn, genutzt für tatsächliche Einladungen

---

## Modell ausführen

```bash
node model/main.js '[0,1,2]' '../data/matchdata.json'
```

- Argument 1: Regionen (`0` = Europa, `1` = Amerika, `2` = Asien)
- Argument 2: Pfad zur Match-Daten JSON-Datei

Der Beispieldatensatz liegt unter `data/matchdata_sample_20230829.json`.

---

## Modellgenauigkeit

Die Genauigkeit wird gemessen, indem **erwartete Gewinnraten** mit **tatsächlichen Gewinnraten** über alle Matches verglichen werden.

Ergebnisse werden in 5%-Bins eingeteilt und verglichen. Das aktuelle Modell erreicht:

> **Spearman's rho = 0,98**

Die Steigung ist etwas flacher als ideal — das Modell unterschätzt leicht Upsets und überschätzt leicht Favoriten, aber die Gesamtkorrelation ist sehr stark.

<img src="modelfit.png"/>

---

## Dateiübersicht

| Datei | Zweck |
|---|---|
| `model/main.js` | Einstiegspunkt |
| `model/ranking.js` | Koordiniert Seeding + Matchverarbeitung + Ranking |
| `model/glicko.js` | Glicko-ELO Ratingsystem-Implementierung |
| `model/data_loader.js` | Lädt, filtert und bereitet Matchdaten vor |
| `model/team.js` | Team/Roster-Datenstruktur und Seeding-Modifier-Logik |
| `model/report.js` | Generiert Markdown-Ausgabedateien |
| `model/ranking_context.js` | Gemeinsame Konfiguration (Zeitfenster, Modifier) |
| `model/util/region.js` | Mappt Ländercodes auf Regionen |
| `data/` | Matchdatensatz (JSON) |
| `invitation/` | Historische Standings für tatsächliche Einladungen |
| `live/` | Aktuelle Live-Standings |
