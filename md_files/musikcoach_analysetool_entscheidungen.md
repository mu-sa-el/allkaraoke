# Music Coach Analysetool – Änderungshistorie & Entscheidungen

## Repository

**GitHub:** https://github.com/mu-sa-el/allkaraoke
**Branch:** `dev_semitone_tolerance_and_pitch_tolerance`

---

## Abgeschlossene Änderungen

---

### Änderung 1: OLA Time Stretching (abgeschlossen)

**Problem:**
`resampleBuffer()` verwendete `playbackRate` der Web Audio API. Das verändert
Speed und Pitch gleichzeitig. Bei `resampleSpeed = 1.0217` wurden alle Noten
um ~37 Cents zu hoch erkannt → falsche `distance` → Score ~53%.

**Lösung:**
`resampleBuffer()` wurde auf OLA (Overlap-Add) Time Stretching umgebaut.
Speed ändert sich, Pitch bleibt konstant.

**Ergebnis:**
Score von 53% auf 88% (Klavier-Test).

**Betroffene Datei:**
`standaloneScoreCalculation.ts` – Funktion `resampleBuffer()`

**Empfohlene Parameter:**
- Input Lag: `163ms`
- Resample Speed: `1.0217`

---

## Geplante Änderungen

---

### Änderung 2: Lücken innerhalb MIDI-Fenster überbrücken

**Problem:**
Piano-Töne klingen aus (natürlicher Decay). Aubio gibt `frequency = 0`
zurück wenn das Signal zu leise wird. Lücken von ~150–280ms entstehen.
Übersteigen diese 100ms (`SINGING_BREAK_TOLERANCE_MS`), startet ein neues
Segment und die Beats in der Lücke zählen nicht.

**Lösung:**
Wenn `frequency = 0` und der Zeitstempel noch innerhalb des aktiven
MIDI-Noten-Fensters liegt und die Lücke kleiner als 0.4 Beats ist →
vorheriges Segment weiter verlängern statt neues zu starten.

**Bedingungen für Überbrückung (alle drei müssen zutreffen):**
```
1. frequency = 0  (echte Stille, kein Messrauschen)
2. Zeitstempel liegt innerhalb note.start bis note.start + note.length
3. Lücke < 0.4 Beats (= 100ms bei 60 BPM, skaliert automatisch)
```

**Design-Entscheidung:**
Ein Schüler der mitten in einer Note aufhört verliert Punkte für die
verpassten Beats. Die Überbrückung gilt nur für kurze technische Lücken
(Decay, Messunschärfe), nicht für echte Pausen.

**Betroffene Datei:**
`standaloneScoreCalculation.ts` – Funktion `appendFrequencyToPlayerNotesStandalone()`

**Erwartetes Ergebnis:**
Klavier-Score von 88% auf ~97%.

---

### Änderung 3: Kurze Fehldetektionen ignorieren

**Problem:**
Am Ende einer Note erkennt Aubio kurz die falsche Frequenz (z.B. E3 statt
E4, oder F#4 statt F4). Das liegt an Messrauschen beim Ausklingen des Tons.
`isDistanceDifferent` wird `true` → neues kurzes Segment mit `distance ≠ 0`
→ vorheriges korrektes Segment stoppt frühzeitig → ~0.1–0.4 Beats verloren.

**Lösung:**
Wenn ein neues Segment mit `distance ≠ 0` kürzer als 0.5 Beats wäre →
Segment nicht erstellen, vorheriges korrektes Segment weiter verlängern.

**Schwellenwert Beat-basiert (nicht ms-basiert) weil:**
- 0.5 Beats skaliert automatisch mit dem Tempo
- Bei 60 BPM = 125ms, bei 120 BPM = 62.5ms
- Echte Fehler eines Schülers dauern länger als 0.5 Beats

**Betroffene Datei:**
`standaloneScoreCalculation.ts` – Funktion `appendFrequencyToPlayerNotesStandalone()`

**Erwartetes Ergebnis:**
Klavier-Score von ~97% auf ~100%.

---

## Offene Fragen & spätere Verbesserungen

| Punkt | Beschreibung | Priorität |
|---|---|---|
| Pitch-Shift-Parameter | Hat das Tool einen separaten Pitch-Shift-Parameter? Würde Kombination Resample + Pitch-Korrektur ermöglichen | Mittel |
| Zeitfenster konfigurierbar | `getNoteAtBeat` Toleranz von 0.5 Beats als Parameter | Niedrig |
| isPerfect Score-Einfluss | Aktuell Multiplikator 0, für späteres Feedback-System relevant | Niedrig |

---

## Teststrategie

**Referenztest:** Audinom spielt selbst vor und nimmt auf.
Erwarteter Score: nahe 100%.

**Testfiles:**
- `test_files/TestmelodieKlavier.mp3` + `test_files/Testmelodie.mid`
- Einstellungen: Input Lag 163ms, Resample Speed 1.0217

**Score-Verlauf:**

| Stand | Score | Änderung |
|---|---|---|
| Ausgangszustand | ~53% | playbackRate (Pitch-Problem) |
| Nach Änderung 1 | 88% | OLA Time Stretching |
| Nach Änderung 2 | ~97% | Lücken überbrücken (geplant) |
| Nach Änderung 3 | ~100% | Fehldetektionen ignorieren (geplant) |
