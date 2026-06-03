# Music Coach Analysetool – Session Handover
## Übergabe-Prompt für neue Claude Code Sessions

## Repository

**GitHub:** https://github.com/mu-sa-el/allkaraoke
**Branch:** `dev_semitone_tolerance_and_pitch_tolerance`

---

---

## Kontext

Ich arbeite am Analysetool für das Audinom Music Coach Feature. Das Tool
vergleicht MP3-Aufnahmen mit MIDI-Dateien und berechnet wie genau eine
Übung gespielt wurde (ähnlich wie ein Karaoke-Bewertungssystem).

Technischer Stack: TypeScript, Web Audio API, Aubio (Pitch Detection),
OLA Time Stretching.

---

## Aktueller Stand

### Abgeschlossene Änderungen

**OLA Time Stretching** – `resampleBuffer()` in
`standaloneScoreCalculation.ts` wurde von `playbackRate` (verändert Speed
und Pitch) auf OLA umgebaut (verändert nur Speed). Ergebnis: Klavier-Score
von 53% auf 88%.

### Nächste Änderungen

Beide Änderungen betreffen ausschließlich die Funktion
`appendFrequencyToPlayerNotesStandalone()` in
`standaloneScoreCalculation.ts`.

---

**Änderung 2: Lücken überbrücken**

Wenn `frequency = 0` und alle drei Bedingungen erfüllt sind:
1. Zeitstempel liegt innerhalb `note.start` bis `note.start + note.length`
2. Lücke < 0.4 Beats (Beat-basiert, nicht ms-basiert)
3. Das letzte Segment hat `distance === 0`

→ Vorheriges Segment weiter verlängern statt neues Segment zu starten.

Grund: Piano-Töne klingen aus (Decay). Aubio gibt `0` zurück wenn Signal
zu leise. Lücken von ~150–280ms entstehen und verlorene Beats kosten ~10%.

---

**Änderung 3: Kurze Fehldetektionen ignorieren**

Wenn ein neues Segment mit `distance ≠ 0` gestartet werden würde, aber die
erwartete Länge dieses Segments kürzer als 0.5 Beats wäre:
→ Segment nicht erstellen, vorheriges korrektes Segment weiter verlängern.

Schwellenwert Beat-basiert (nicht ms-basiert) damit er mit dem Tempo
skaliert.

Grund: Messrauschen beim Ausklingen einer Note. Aubio erkennt kurz eine
falsche Frequenz → ~0.1–0.4 Beats verloren pro betroffene Note.

---

## Test nach den Änderungen

Dev-Server starten:
```
pnpm start:mock
```

`score_calc.html` im Browser öffnen. Testfiles hochladen:
- MP3: `test_files/TestmelodieKlavier.mp3`
- MIDI: `test_files/Testmelodie.mid`

Einstellungen:
- Input Lag: `163`
- Resample Speed: `1.0217`

Erwartetes Ergebnis nach beiden Änderungen: Score nahe 100%, alle Player
Notes mit `isPerfect: true`, keine gesplitteten Segmente mehr.

---

## Was NICHT geändert werden soll

- `resampleBuffer()` – bereits fertig
- `score_calc.html`
- Alle anderen Files außer `standaloneScoreCalculation.ts`
- Die öffentliche API / Funktionssignaturen

---

## Relevante Dokumente

- `musikcoach_analysetool_uebersicht.md` – Projektübersicht
- `musikcoach_analysetool_logik.md` – Vollständige technische Logik
- `musikcoach_analysetool_entscheidungen.md` – Änderungshistorie
