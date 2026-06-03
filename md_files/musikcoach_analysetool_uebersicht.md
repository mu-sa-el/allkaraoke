# Music Coach Analysetool – Projektübersicht

## Repository

**GitHub:** https://github.com/mu-sa-el/allkaraoke
**Branch:** `dev_semitone_tolerance_and_pitch_tolerance`

---

## Was ist dieses Tool?

Das Analysetool ist eine Komponente des Audinom Music Coach Features.
Es vergleicht eine **MP3-Aufnahme** mit einer **MIDI-Datei** und berechnet
wie genau eine Übung gespielt wurde.

Das Tool läuft im Browser (TypeScript/Web Audio API) und ist Teil der
Audinom-Webapp (Supabase Backend).

---

## Warum existiert es?

Audinom ist eine Musik-Lern-App. Schüler*innen spielen Übungen auf ihrem
Instrument, die App nimmt die Aufnahme auf und bewertet ob die richtigen
Töne zur richtigen Zeit gespielt wurden.

Das Analysetool ist das Herzstück dieses Feedback-Systems. Ohne präzise
Analyse kein sinnvolles pädagogisches Feedback.

---

## Ziel

Ein Score von nahe 100% wenn das Audinom eine Übung **selbst vorspielt und
gleichzeitig aufnimmt** (Referenztest). Dieser Wert ist die Obergrenze für
das was ein menschlicher Spieler erreichen kann – er dient als Kalibrierung.

---

## Technischer Stack

| Komponente | Rolle |
|---|---|
| TypeScript | Programmiersprache |
| Web Audio API | Audio-Dekodierung und Verarbeitung |
| Aubio (aubiojs) | Pitch Detection (Frequenzerkennung) |
| OLA Time Stretching | Tempo-Anpassung ohne Pitch-Änderung |
| MIDI Parser (parseMidiToSong.ts) | MIDI → Song-Objekt Konvertierung |

---

## Relevante Files

| File | Rolle |
|---|---|
| `standaloneScoreCalculation.ts` | Herzstück – gesamte Score-Logik |
| `detectVibrato.ts` | Vibrato-Erkennung |
| `notesSelectors.ts` | Hilfsfunktionen für Noten-Zugriff |
| `getSongBeatLength.ts` | Beat-Länge in ms berechnen |
| `Aubio.ts` | Pitch Detection Strategie |
| `parseMidiToSong.ts` | MIDI zu Song-Objekt |
| `interfaces.ts` | TypeScript-Typen |
| `score_calc.html` | Test-Interface im Browser |

---

## Testdateien

| File | Inhalt |
|---|---|
| `test_files/TestmelodieKlavier.mp3` | Audinom-Output Klavier, 60 BPM, 21 Noten |
| `test_files/Testmelodie.mid` | Referenz-MIDI, 60 BPM, 21 Noten |

**Referenzergebnisse:**

| Sound | Score | Erwartung |
|---|---|---|
| MIDI Violine | 99.9% | ✅ Korrekt |
| MIDI Klavier | 88% → nach Fix ~100% | 🔧 In Arbeit |

---

## Bekannte Bugs im Audinom (für Jörn)

Diese Bugs wurden beim Testen entdeckt und sollten dem Entwicklungsteam
gemeldet werden:

| Bug | Beschreibung |
|---|---|
| Pitch-Offset | Audinom spielt ~17–20 Cents zu hoch (Sample-Rate-Mismatch) |
| Timing-Drift | MP3 läuft ~2.17% langsamer als MIDI |
| Timing-Sprung | Ab Takt 8 kommen Noten konsistent ~47ms früher |

---

## Konfigurationsparameter (score_calc.html)

| Parameter | Empfohlener Wert | Beschreibung |
|---|---|---|
| Input Lag | 163ms | Versatz zwischen MIDI-Start und erstem Ton in MP3 |
| Resample Speed | 1.0217 | Tempo-Korrektur via OLA Time Stretching |
