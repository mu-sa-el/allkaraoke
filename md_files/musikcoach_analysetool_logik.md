# Music Coach Analysetool – Technische Logik

## Repository

**GitHub:** https://github.com/mu-sa-el/allkaraoke
**Branch:** `dev_semitone_tolerance_and_pitch_tolerance`

---

## Die 4-stufige Pipeline

```
MP3-Datei
  → Schritt 1: Dekodierung + OLA Time Stretching
    → Schritt 2: Pitch Detection (Aubio)
      → Schritt 3: Frequency Records → Player Notes
        → Schritt 4: Score-Berechnung
```

---

## Schritt 1: Dekodierung + OLA Time Stretching

**Funktion:** `decodeAudioFile()` → `resampleBuffer()`

Die MP3 wird dekodiert und anschließend zeitgestreckt um den Timing-Drift
zwischen MP3 und MIDI zu korrigieren.

**OLA (Overlap-Add):** Ändert das Tempo ohne den Pitch zu verändern.
Das ist der Unterschied zu `playbackRate` (ändert beides gleichzeitig).

```
Resample Speed 1.0217:
  → MP3 wird um 2.17% beschleunigt
  → Pitch bleibt unverändert
  → Timing stimmt mit MIDI überein
```

Danach wird auf Mono gemischt (L+R / 2).

---

## Schritt 2: Pitch Detection

**Funktion:** `detectPitchesFromSamples()`
**Library:** Aubio (aubiojs), Algorithmus: YIN

```
FFT-Größe: 2048 Samples
Samplerate: 44100 Hz

→ Zeitauflösung: 2048 / 44100 = ~46ms pro Frame
→ ~21 Messungen pro Sekunde
```

Jede Messung liefert:
- Eine Frequenz in Hz (z.B. 261.9 Hz = C4)
- Oder `0` wenn kein Ton erkannt wird (zu leise, Stille)

**Das gesamte Audio wird gemessen** – unabhängig davon was das MIDI erwartet.
Die Zuordnung zu MIDI-Noten passiert erst in Schritt 3.

---

## Schritt 3: Frequency Records → Player Notes

**Funktion:** `convertFrequencyRecordsToPlayerNotes()`

### Input Lag Korrektur

```typescript
adjustedTimestamp = record.timestamp - inputLagMs
```

Verschiebt alle Messungen um den Input Lag zurück, damit sie mit den
MIDI-Positionen übereinstimmen.

### Beat-Zuordnung

```typescript
recordBeat = adjustedTimestamp / beatLength
```

Bei 60 BPM und `bar = 4`:
```
beatLength = (60 / 60 / 4) * 1000 = 250ms
→ 1 Beat = 250ms = eine 16tel-Note
→ 4 Beats = 1000ms = eine Viertelnote
```

### Noten-Zuordnung

```typescript
getNoteAtBeat(section, recordBeat, 0)      // exakt
  ?? getNoteAtBeat(section, recordBeat, 0.5) // ±0.5 Beats Toleranz
```

Liegt der Beat außerhalb jeder MIDI-Note → Messung wird ignoriert.

### Distance-Berechnung

```typescript
pitchFromFrequency(freq) → MIDI-Notennummer (gerundet)
calcDistanceBetweenPitches(erkannte_note, ziel_note, tolerance)
  → 0:  richtige Note
  → 1:  einen Halbton daneben
  → -2: zwei Halbtöne zu tief
  usw.
```

### Segment-Logik (appendFrequencyToPlayerNotesStandalone)

Eine Player Note ist ein **kontinuierliches Segment** mit gleichbleibender
`distance`. Ein neues Segment wird gestartet wenn EINES zutrifft:

```
1. Eine neue MIDI-Note beginnt
2. Die distance hat sich geändert (z.B. 0 → 1 → 0)
3. Lücke zum letzten Segment > SINGING_BREAK_TOLERANCE_MS (100ms)
```

**isPerfect** wird am Ende jedes Frame-Updates geprüft:
```typescript
isPerfect = distance === 0
  && Math.abs(segment.length - note.length) < 0.5
```

**Wichtig:** `isPerfect` hat aktuell Multiplikator `0` in der Score-Formel
und beeinflusst den Prozentsatz nicht. Es wird nur für spätere Anzeige
gespeichert.

---

## Schritt 4: Score-Berechnung

**Funktion:** `calculateDetailedScoreDataStandalone()`

```typescript
for (jedes Segment) {
  if (distance !== 0) → überspringen (0 Punkte)
  if (distance === 0) → segment.length zu counts.normal addieren
}

score% = counts.normal / maxCounts.normal
```

`maxCounts.normal` = Summe aller MIDI-Notenlängen in Beats (= 88 Beats
im Testfile).

**Score-Formel:**
```
MAX_POINTS = 3.500.000
pointsPerBeat = MAX_POINTS / maxCounts.normal
score = counts.normal * pointsPerBeat
score% = score / MAX_POINTS * 100
```

---

## Warum werden Beats verloren?

### Grund 1: Piano-Decay (Hauptproblem)

```
Piano-Note klingt → Amplitude sinkt → Aubio gibt 0 zurück
→ Frames werden übersprungen
→ Lücke entsteht
→ Lücke > 100ms (SINGING_BREAK_TOLERANCE_MS)
→ neues Segment
→ Beats in der Lücke zählen nicht
```

Violine hat keinen Decay → keine Lücken → 99.9%.
Piano hat Decay → Lücken von ~150–280ms → 88%.

### Grund 2: Messrauschen am Notenende

```
Note klingt korrekt → letzter Frame vor Notenende:
  Aubio erkennt kurz falsche Frequenz (z.B. E3 statt E4)
→ isDistanceDifferent = true → neues Segment (distance ≠ 0)
→ vorheriges korrektes Segment stoppt frühzeitig
→ ~0.1–0.4 Beats verloren pro betroffene Note
```

---

## Beat-Referenz

| Tempo | beatLength | 4 Beats | 0.4 Beats | 0.5 Beats |
|---|---|---|---|---|
| 60 BPM | 250ms | 1000ms | 100ms | 125ms |
| 90 BPM | 167ms | 667ms | 67ms | 83ms |
| 120 BPM | 125ms | 500ms | 50ms | 63ms |

---

## Konstanten in standaloneScoreCalculation.ts

| Konstante | Wert | Bedeutung |
|---|---|---|
| `MAX_POINTS` | 3.500.000 | Maximaler Score |
| `SINGING_BREAK_TOLERANCE_MS` | 100ms | Lücken-Toleranz |
| `tolerance` (Parameter) | 0 | Halbton-Toleranz bei distance-Berechnung |
| `fftSize` | 2048 | FFT-Fenstergröße für Aubio |

---

## Multiplier-Tabelle (noteTypesMultipliers)

| Typ | Multiplikator | Verwendung |
|---|---|---|
| normal | 1.0 | Standard-Note |
| star | 2.0 | Goldene Note (doppelte Punkte) |
| perfect | 0.0 | Cosmetic only, kein Score-Einfluss |
| vibrato | 0.0 | Cosmetic only, kein Score-Einfluss |
| freestyle | 0.25 | Freie Note, weniger Punkte |
| rap | 0.25 | Rap-Note |
