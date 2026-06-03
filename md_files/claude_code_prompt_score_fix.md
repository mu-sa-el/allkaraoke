# Claude Code Prompt – Score Fix (Änderung 2 & 3)

## Kontext

Ich arbeite am Analysetool für das Audinom Music Coach Feature.
Das Tool vergleicht MP3-Aufnahmen mit MIDI-Dateien und berechnet einen
Score (0–3.500.000 Punkte, als % angezeigt).

Der aktuelle Klavier-Test ergibt 88% obwohl alle Töne korrekt gespielt
wurden. Grund: zwei technische Probleme in der Funktion
`appendFrequencyToPlayerNotesStandalone()` in
`src/modules/GameEngine/GameState/Helpers/standaloneScoreCalculation.ts`.

---

## Beide Probleme und ihre Lösungen

---

### Problem 1: Piano-Decay erzeugt ungewollte Lücken

**Was passiert:**
Piano-Töne klingen aus (natürlicher Decay). Aubio gibt `frequency = 0`
zurück wenn das Signal zu leise wird. In der aktuellen Logik wird eine
Lücke von `frequency = 0`-Frames nach mehr als 100ms
(`SINGING_BREAK_TOLERANCE_MS`) als Unterbrechung gewertet und ein neues
Segment gestartet. Die Beats in der Lücke gehen verloren.

**Beweis aus Testdaten:**
G4-Note (MIDI Beat 48–52, 1000ms): Aubio erkennt korrekt bis Beat 50.71,
dann Stille bis Beat 51.83 (Lücke = 280ms > 100ms) → neues Segment →
1.12 Beats verloren. Das passiert bei fast jeder G4-Note (8× im Testfile).

**Lösung – Änderung 2:**
Wenn `frequency = 0` und alle drei Bedingungen erfüllt sind:
1. Der aktuelle Zeitstempel liegt noch innerhalb des aktiven MIDI-Noten-
   Fensters (`note.start` bis `note.start + note.length`)
2. Die Lücke seit dem letzten Segment-Ende ist kleiner als `0.4 * beatLength`
3. Das letzte Segment hat `distance === 0`

→ Vorheriges Segment weiter verlängern (length updaten) statt neues zu
starten. Der `breakToleranceBeat`-Check darf für diese Frames nicht
feuern.

**Wichtig:** Nur für `frequency = 0` anwenden, nicht für falsche Frequenzen.

---

### Problem 2: Messrauschen am Notenende erzeugt kurze Falschsegmente

**Was passiert:**
In den letzten 1–2 Frames vor dem Notenende erkennt Aubio kurz die falsche
Frequenz (z.B. E3 statt E4, F#4 statt F4). `isDistanceDifferent` wird
`true`, ein neues kurzes Segment mit `distance ≠ 0` startet, das vorherige
korrekte Segment stoppt frühzeitig.

**Beweis aus Testdaten:**
- E4-Note: Segment 3.57 Beats korrekt, dann 0.25 Beats `distance: 15`
- F4-Note: Segment 3.55 Beats korrekt, dann 0.08 Beats `distance: 1`
- G4-Note: Segment 2.35 Beats korrekt, dann 0.35 Beats `distance: 12`

**Lösung – Änderung 3:**
Bevor ein neues Segment mit `distance ≠ 0` gestartet wird, prüfen ob die
verbleibende Zeit bis zum Ende der aktuellen MIDI-Note kleiner als
`0.5 * beatLength` ist.

Wenn ja:
→ Kein neues Segment starten
→ Vorheriges Segment bis zum Ende der MIDI-Note verlängern
→ `distance` des vorherigen Segments bleibt unverändert

**Warum Beat-basiert (nicht ms-basiert):**
Skaliert automatisch mit dem Tempo. Bei 60 BPM = 125ms, bei 120 BPM = 62ms.

---

## Betroffene Datei und Funktion

**Datei:**
`src/modules/GameEngine/GameState/Helpers/standaloneScoreCalculation.ts`

**Funktion:**
`appendFrequencyToPlayerNotesStandalone()` – nur diese Funktion ändern.

**Was sich nicht ändert:**
- `resampleBuffer()` – bereits fertig, nicht anfassen
- `SINGING_BREAK_TOLERANCE_MS` – Konstante bleibt, wird aber durch
  Änderung 2 für bestimmte Frames nicht mehr ausgelöst
- Alle anderen Funktionen in der Datei
- Alle anderen Files im Projekt
- `score_calc.html`

---

## Test nach den Änderungen

Dev-Server starten:
```
pnpm start:mock
```

`score_calc.html` im Browser öffnen. Testfiles manuell hochladen:
- MP3: `test_files/TestmelodieKlavier.mp3`
- MIDI: `test_files/Testmelodie.mid`

Einstellungen manuell eintragen:
- Input Lag: `163`
- Resample Speed: `1.0217`

„Run Score Calculation Test" klicken.

**Erwartetes Ergebnis:**
- Score nahe 100% (vorher 88%)
- Player Notes: 21 Einträge (eine pro MIDI-Note, keine gesplitteten Segmente)
- Alle `distance: 0`
- Alle `isPerfect: true`

Falls Score immer noch unter 95%: Report ausgeben und Ursache analysieren.
Falls TypeScript-Fehler auftreten: beheben bevor du abschließt.
