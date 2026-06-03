# Claude Code Prompt – SoundTouch Time Stretching

## Kontext

In `src/modules/GameEngine/GameState/Helpers/standaloneScoreCalculation.ts`
gibt es eine Funktion `resampleBuffer` (Zeilen ~174–192) die zum Anpassen
der Abspielgeschwindigkeit einer MP3 verwendet wird. Das Problem: Sie nutzt
`source.playbackRate.value = speed` über die Web Audio API, was Speed und
Pitch gleichzeitig verändert. Bei `speed = 1.0217` wird dadurch die Tonhöhe
um ~37 Cents angehoben, was zur falschen Notenerkennung führt.

Die Lösung: Time Stretching statt Resampling. Speed ändern, Pitch bleibt
konstant. Das soll mit der Library `soundtouch-js` implementiert werden.

---

## Testdateien vorbereiten

Lege folgende zwei Dateien ins Projekt (falls noch nicht vorhanden):

```
test_files/TestmelodieKlavier.mp3
test_files/Testmelodie.mid
```

Die Dateien liegen bereits im Projekt-Root. Verschiebe sie in den Ordner
`test_files/` – lege den Ordner an falls er noch nicht existiert.

---

## Aufgabe

### Schritt 1 – Library installieren

Prüfe zuerst ob `soundtouch-js` bereits in `package.json` vorhanden ist.
Falls nicht, installiere es:

```
npm install soundtouch-js
```

Falls `soundtouch-js` nicht verfügbar oder nicht kompatibel ist (TypeScript-
Typen fehlen, Bundler-Probleme), als Alternative `@soundtouchjs/soundtouch-js`
oder `soundtouch-web` prüfen. Wähle die Variante die im Projekt am saubersten
funktioniert.

---

### Schritt 2 – Funktion ersetzen

**Datei:** `src/modules/GameEngine/GameState/Helpers/standaloneScoreCalculation.ts`

**Ersetze** die bestehende Funktion `resampleBuffer` (Zeilen ~174–192):

```typescript
// DIESE FUNKTION ENTFERNEN:
async function resampleBuffer(buffer: AudioBuffer, speed: number): Promise<AudioBuffer> {
  const newDuration = buffer.duration / speed;
  const offlineCtx = new OfflineAudioContext(
    buffer.numberOfChannels,
    Math.ceil(newDuration * buffer.sampleRate),
    buffer.sampleRate,
  );
  const source = offlineCtx.createBufferSource();
  source.buffer = buffer;
  source.playbackRate.value = speed;
  source.connect(offlineCtx.destination);
  source.start();
  const renderedBuffer = await offlineCtx.startRendering();
  return renderedBuffer;
}
```

**Ersetze sie durch** eine neue Implementierung die SoundTouch verwendet.
Die neue Funktion muss:
- Denselben Funktionsnamen `resampleBuffer` behalten (wird in `decodeAudioFile`
  aufgerufen – dieser Aufruf bleibt unverändert)
- Dieselbe Signatur haben: `(buffer: AudioBuffer, speed: number): Promise<AudioBuffer>`
- Einen `AudioBuffer` zurückgeben mit korrekter Länge für `speed > 1` (kürzer)
  und `speed < 1` (länger)
- Bei `speed === 1.0` den Buffer unverändert zurückgeben (kein Processing nötig)
- Mono und Stereo korrekt verarbeiten

Implementierungshinweise:
- SoundTouch erwartet interleaved Float32Array als Input (für Stereo: L R L R ...)
- Das `tempo`-Property von SoundTouch steuert die Geschwindigkeit ohne Pitch-Änderung
- Die Ausgabelänge berechnet sich aus `Math.ceil(buffer.length / speed)`
- Alle anderen Stellen in der Datei bleiben unverändert

---

### Schritt 3 – Verifizieren dass nichts anderes geändert wurde

Stelle sicher dass:
- Nur `resampleBuffer` ausgetauscht wurde
- Der Import-Block am Dateianfang nur um den SoundTouch-Import erweitert wurde
- Keine andere Funktion in der Datei verändert wurde
- `decodeAudioFile` (Zeile ~197) exakt gleich geblieben ist

---

### Schritt 4 – App starten und testen

Starte den Dev-Server:
```
pnpm start:mock
```
(oder `npm run dev` falls kein pnpm-Script vorhanden – schaue in `package.json`
nach dem richtigen Start-Befehl)

Öffne `score_calc.html` im Browser. Lade dort manuell hoch:
- MP3: `test_files/TestmelodieKlavier.mp3`
- MIDI: `test_files/Testmelodie.mid`

Trage im Interface ein:
- Input Lag: `163`
- Resample Speed: `1.0217`

Klicke „Run Score Calculation Test".

Erwartetes Ergebnis: Alle oder fast alle Noten werden mit `distance: 0`
erkannt (kein Pitch-Shift mehr durch Resampling). Falls weiterhin
`distance: 1` auftritt, liegt ein Fehler in der SoundTouch-Implementierung
vor – dann debuggen und beheben.

Falls der Dev-Server nicht startet oder TypeScript-Fehler auftreten: Fehler
beheben bevor du abschließt.

---

## Was NICHT geändert werden soll

- Alle anderen Funktionen in `standaloneScoreCalculation.ts`
- `score_calc.html`
- Alle anderen Dateien im Projekt
- Die Funktionssignatur von `resampleBuffer`

---

## Zusammenfassung

Einzige inhaltliche Änderung: `resampleBuffer` verwendet künftig SoundTouch
`tempo`-Property statt Web Audio `playbackRate`. Das behebt den Pitch-Shift-
Effekt bei Resample Speed ≠ 1.0, ohne sonst irgendetwas in der Pipeline
anzufassen.
