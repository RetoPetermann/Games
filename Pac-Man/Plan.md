# Pac-Man – Implementierungsplan

**Zieldatei:** `Pac-Man/index.html`
**Begleitdatei:** `Pac-Man/CLAUDE.md`
**Hub-Update:** `index.html`

---

## 1. Spielkonzept & Features

### Kernfeatures (klassisches Pac-Man)
- Pac-Man frisst Pellets und Power-Pellets in einem Labyrinth
- 4 Geister mit klassischen Namen und eigenem KI-Verhalten
- Power-Pellets schalten Frightened-Modus ein (Geister werden blau, fressbar)
- Punkte: Pellet = 10, Power-Pellet = 50, Geist fressen = 200/400/800/1600 (Combo)
- 3 Leben, Game Over bei 0 Leben
- Level-Sieg wenn alle Pellets gefressen; danach nächstes Level (schnellere Geister)
- Highscore via `localStorage` unter Key `pacman_high`
- Bonus-Frucht erscheint ab 70 gefressenen Pellets (Kirsche = 100 Pkt.)

### Nicht-klassische Vereinfachungen (single-file, realistisch)
- Kein Intermission-Screen zwischen Leveln
- Kein Tunnel-Geschwindigkeitsbonus
- Keine separate Energizer-Blinkanimation in den letzten Sekunden des Frightened-Modus (optional einfach nachrüstbar)

---

## 2. Technische Architektur

### Canvas & Layout
```
Canvas: 560 × 720 px  (28 Tiles breit × 36 Tiles hoch)
Tile-Grösse: 20 × 20 px
Spielfeld: 28 × 31 Tiles (620 px Höhe für Maze)
HUD-Bereich: 5 Tiles oben (Score/Level) + Rand unten (Leben-Anzeige)
```

**Warum 560×720:** Passt zum klassischen Verhältnis (original 224×288 skaliert ×2.5), bleibt in typischen Viewports, liegt im Rahmen der anderen Spiele (Asteroids 800×600, Space Invaders 640×480).

### Koordinatensystem
- Tile-Koordinaten: `tx, ty` (Integer, 0-basiert)
- Pixel-Koordinaten: `px = tx * TILE + TILE/2`, zentriert in Tile
- Pac-Man und Geister bewegen sich in Subpixel-Schritten entlang der Tile-Achsen
- Bewegungsrichtung: `{dx, dy}` mit Werten aus `{-1, 0, 1}`

### Game Loop
```
requestAnimationFrame → gameLoop(timestamp)
  delta = timestamp - lastTime (capped bei 50ms)
  update(delta)
  draw()
```

### Script-Struktur
```
Konstanten & Konfiguration
Audio-Funktionen
Labyrinth-Definition (Tile-Map)
Spielzustand (State-Objekte)
Hilfsfunktionen (Tile-Queries, Pixel-Konvertierung)
Pac-Man Logik (Bewegung, Input)
Geister-Logik (KI, Modi)
Kollisionserkennung
Spielzustands-Maschine (Start/Playing/Paused/GameOver/Victory)
Draw-Funktionen (Maze, Pac-Man, Geister, HUD, Overlays)
Input-Handler
Init & Start
```

---

## 3. Labyrinth-Design

### Tile-Map Datenstruktur

```javascript
// Tile-Typen als Integer-Konstanten
const T = {
  WALL:  0,   // blaue Wand
  DOT:   1,   // kleiner Punkt (10 Pkt.)
  PELLET:2,   // Power-Pellet (50 Pkt.)
  EMPTY: 3,   // freier Boden (kein Punkt)
  DOOR:  4,   // Geister-Tür (nur Geister passieren)
  WRAP:  5,   // Tunnel-Ausgang links/rechts
};

// 28 × 31 Array (Row-Major)
const MAZE_TEMPLATE = [
  // Zeile 0..30, je 28 Werte
  [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],
  // ...
];
```

**Maze-Layout-Strategie:** Das klassische Pac-Man-Labyrinth wird als 28×31-Konstante hardcodiert. Die Wand-Tiles werden als ausgefüllte Rechtecke gezeichnet; für die klassischen abgerundeten Wand-Ecken wird `ctx.arc()` genutzt (vereinfacht: einfache Rechtecke mit leichtem Radius).

**Ghost House:** Tile (11–16, 12–15) — 6×4 Tiles grosses Geister-Haus mit DOOR-Tile bei (13/14, 12). Geister starten innerhalb, Pac-Man startet bei (14, 23).

**Tunnel:** Zeile 14, Spalten 0 und 27 sind WRAP-Tiles. Wer die linke Seite verlässt, erscheint rechts (und umgekehrt).

**Dot-Zählung:** Beim Initialisieren wird `totalDots` aus der Maze gezählt; `dotsEaten` zählt hoch; bei `dotsEaten === totalDots` → Level gewonnen.

### Wand-Rendering
- Wand-Tiles: `ctx.fillStyle = '#1a1aff'` (klassisches Pac-Man Blau), `ctx.fillRect` mit 1px Inset
- Innere Wand-Kanten: nach dem Zeichnen einen dunkleren Innen-Stroke für 3D-Effekt

---

## 4. Spielobjekte

### Pac-Man

```javascript
const pacman = {
  tx: 14, ty: 23,        // Tile-Position
  x: 0, y: 0,            // Pixel-Position (berechnet aus tx/ty)
  dx: 0, dy: 0,          // aktuelle Bewegungsrichtung
  nextDx: 0, nextDy: 0,  // gepufferte nächste Richtung (Input)
  speed: 0.08,           // Tiles pro ms (ca. 4 Tiles/s)
  progress: 0.5,         // Position innerhalb eines Tiles (0..1)
  mouthAngle: 0,         // für Mundanimation (0..0.25 in PI-Einheiten)
  mouthDir: 1,           // öffnet/schliesst
  alive: true,
  deathTimer: 0,         // Countdown für Todesanimation
  invincible: 0,         // kurze Unverwundbarkeit nach Respawn
};
```

**Bewegungslogik:**
1. `nextDx/nextDy` wird sofort beim Tastendruck gesetzt (Puffer)
2. Bei jedem Tile-Übergang: prüfe ob `nextDx/nextDy`-Richtung frei ist → wechsle; sonst halte aktuelle Richtung falls frei; sonst stoppe
3. Subpixel-Bewegung: `progress += speed * delta`; bei `progress >= 1` → Tile-Wechsel

**Mund-Animation:** `mouthAngle` oszilliert zwischen 0.05 und 0.25 (in `Math.PI` Einheiten).

### Geister – Übersicht

| Name    | Farbe     | Primärverhalten (Chase) | Scatter-Ziel |
|---------|-----------|------------------------|--------------|
| Blinky  | `#ff0000` | Direkt auf Pac-Man      | Oben rechts  |
| Pinky   | `#ffb8ff` | 4 Tiles vor Pac-Man     | Oben links   |
| Inky    | `#00ffff` | Flankenmanöver (Blinky + Pac-Man Vektor × 2) | Unten rechts |
| Clyde   | `#ffb852` | Direkt wenn fern, Scatter wenn nah (<8 Tiles) | Unten links  |

```javascript
const GHOST_COLORS = ['#ff0000','#ffb8ff','#00ffff','#ffb852'];
const GHOST_NAMES  = ['blinky','pinky','inky','clyde'];
const SCATTER_TARGETS = [{tx:25,ty:0},{tx:2,ty:0},{tx:27,ty:30},{tx:0,ty:30}];

function createGhost(index) {
  return {
    index,
    tx: [14, 14, 12, 16][index],  // Start-Tiles im Ghost House
    ty: [11, 14, 14, 14][index],
    x: 0, y: 0,
    dx: 0, dy: -1,
    progress: 0.5,
    mode: 'house',   // 'house' | 'scatter' | 'chase' | 'frightened' | 'eaten'
    modeTimer: 0,
    speed: 0.065,
    dotCounter: 0,   // persönlicher Dot-Counter für Raus-Trigger
    dotLimit: [0, 30, 60, 90][index],
  };
}
```

**Geister-Geschwindigkeit:**
- Normal: 0.065 Tiles/ms (~3.9 Tiles/s)
- Frightened: 0.04 Tiles/ms (langsamer)
- Eaten (nur Augen): 0.13 Tiles/ms (doppelt schnell zurück)
- Im Tunnel: 0.04 Tiles/ms (alle)

---

## 5. Geister-KI

### Globaler Modus-Timer (Scatter/Chase-Sequenz)

```javascript
// Level 1 Timings (in Sekunden):
const PHASE_DURATIONS = [7, 20, 7, 20, 5, 20, 5, Infinity];
//                       S   C   S   C  S   C   S    C
let phaseIndex = 0;
let phaseTimer = 0;
let globalMode = 'scatter'; // 'scatter' | 'chase'
```

Bei Frightened wird der Timer pausiert, danach fortgesetzt. Ab Level 2 werden Scatter-Phasen kürzer.

### Ghost House – Rauslass-Logik

- **Blinky:** Startet direkt aussen
- **Pinky:** Wartet kurz (dotLimit=0, sofort raus nach Blinky)
- **Inky:** Persönlicher Dot-Counter: 30 Dots nötig
- **Clyde:** 90 Dots nötig
- Im `'house'`-Modus: Geist schwebt auf/ab
- Wenn Bedingung erfüllt: Ziel = DOOR-Tile → danach normaler Scatter/Chase

### Bewegungsalgorithmus

```
Beim Tile-Wechsel (einmal pro Tile):
1. Bestimme Ziel-Tile (je nach Modus)
2. Für jede mögliche Richtung (ausser Umkehrung der aktuellen):
   a. Prüfe: Ist Nachbar-Tile begehbar?
   b. Berechne Manhattan-Distanz vom Nachbar-Tile zum Ziel-Tile
3. Wähle Richtung mit kleinster Distanz (Tie-Break: Oben > Links > Unten > Rechts)
4. Ausnahme: Frightened → zufällige gültige Richtung (kein Umkehren)
```

### Ziel-Tile-Berechnung je Modus

**Scatter:** `SCATTER_TARGETS[ghost.index]`

**Chase:**
- **Blinky:** `{tx: pac.tx, ty: pac.ty}` (direkt)
- **Pinky:** 4 Tiles in Pac-Mans aktueller Richtung vor Pac-Man
- **Inky:** `mid = {tx: pac.tx + pac.dx*2, ty: pac.ty + pac.dy*2}` → Vektor von Blinky zu mid, verdoppeln
- **Clyde:** Wenn Distanz > 8 Tiles zu Pac-Man → wie Blinky; sonst → Scatter-Ziel

**Frightened:** Zufällige gültige Richtung an jeder Kreuzung

**Eaten:** Ziel = Ghost House Eingang (tx=14, ty=12). Wenn erreicht → Mode wechselt zurück zu globalMode.

### Frightened-Modus Auslösung

```javascript
function activateFrightened() {
  for (const g of ghosts) {
    if (g.mode !== 'eaten') {
      g.mode = 'frightened';
      g.modeTimer = FRIGHTENED_DURATION;
      g.dx = -g.dx; g.dy = -g.dy; // Sofortige Umkehrung
    }
  }
  ghostEatCombo = 0;
}
```

Frightened-Dauer Level 1: 6s. Nimmt pro Level um ~500ms ab, ab Level 17: 0s.

---

## 6. Kollisionserkennung

### Pac-Man ↔ Pellet/Power-Pellet

```javascript
// Im Update, wenn progress ~= 0.5 (Pac-Man ist in der Mitte des Tiles):
if (Math.abs(pacman.progress - 0.5) < 0.1) {
  const tile = maze[pacman.ty][pacman.tx];
  if (tile === T.DOT) {
    maze[pacman.ty][pacman.tx] = T.EMPTY;
    score += 10; dotsEaten++;
    playDotEat(dotsEaten % 2);
  } else if (tile === T.PELLET) {
    maze[pacman.ty][pacman.tx] = T.EMPTY;
    score += 50;
    activateFrightened();
    playPowerPellet();
  }
}
```

### Pac-Man ↔ Geist

```javascript
const dist = Math.hypot(pacman.x - ghost.x, pacman.y - ghost.y);
if (dist < TILE * 0.75) {
  if (ghost.mode === 'frightened') {
    ghost.mode = 'eaten';
    ghostEatCombo++;
    score += 200 * Math.pow(2, ghostEatCombo - 1);
    playGhostEat();
  } else if (ghost.mode !== 'eaten' && pacman.invincible <= 0) {
    pacmanDies();
  }
}
```

**Pac-Man stirbt:**
- Todesanimation (Mund schliesst sich zu Kreis, dann Linie, dann verschwindet)
- Wenn noch Leben übrig: Respawn nach 2s
- Wenn keine Leben mehr: State → GAMEOVER

---

## 7. Soundeffekte (Web Audio API, prozedural)

Alle Funktionen folgen dem Muster: `ensureAudio()` vor jedem Sound, AudioContext lazy-initialisieren.

### Sound-Liste

| Sound | Trigger | Technik |
|-------|---------|---------|
| `playDotEat(alt)` | Pellet fressen | Kurzer Piep: Osc Square, 2 alternierende Töne (440Hz / 480Hz), 60ms |
| `playPowerPellet()` | Power-Pellet | Aufsteigender Sweep: Osc Sawtooth, 200→600Hz in 200ms |
| `playGhostEat()` | Geist fressen | Absteigender Glissando: Osc Square, 800→200Hz, 300ms |
| `playDeath()` | Pac-Man stirbt | Absteigende Tonleiter: 5 Töne, je 120ms, dann Rauschen |
| `playLevelWin()` | Level gewonnen | Kleiner Jingle: 4 Töne aufsteigend, Osc Triangle |
| `playGameOver()` | Game Over | Langsam absteigender Glissando, Osc Sawtooth, 1.5s |
| `playExtraLife()` | Extra-Leben | Kurzer Jingle: 3 Töne G-A-C, 100ms each |
| `startFrightenedSound()` | Frightened-Modus | Osc Sawtooth 300Hz + LFO 6Hz, kontinuierlich |
| `stopFrightenedSound()` | Frightened endet | Sound stoppen |

**Frightened-Sound Implementation:**
```javascript
let frightenedOsc = null;
function startFrightenedSound() {
  if (frightenedOsc) return;
  ensureAudio();
  frightenedOsc = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  const lfo = audioCtx.createOscillator();
  const lfoGain = audioCtx.createGain();
  frightenedOsc.type = 'sawtooth';
  frightenedOsc.frequency.value = 300;
  lfo.frequency.value = 6;
  lfoGain.gain.value = 80;
  lfo.connect(lfoGain); lfoGain.connect(frightenedOsc.frequency);
  g.gain.value = 0.05;
  frightenedOsc.connect(g); g.connect(audioCtx.destination);
  lfo.start(); frightenedOsc.start();
  frightenedOsc._g = g; frightenedOsc._lfo = lfo;
}
```

---

## 8. Spielzustände

```javascript
const STATE = {
  START:    'start',    // Startscreen, ENTER drücken
  READY:    'ready',    // "BEREIT!" Einblendung (2s)
  PLAYING:  'playing',  // Normales Gameplay
  DYING:    'dying',    // Todesanimation läuft (2.5s)
  LEVELWIN: 'levelwin', // Labyrinth blinkt, Jingle (2s)
  GAMEOVER: 'gameover', // "GAME OVER", ENTER zum Neustart
};
```

### Zustandsübergänge

```
START → READY:     ENTER-Taste gedrückt
READY → PLAYING:   stateTimer abgelaufen (2000ms)
PLAYING → DYING:   Pac-Man getroffen
DYING → READY:     stateTimer abgelaufen (2500ms) + lives > 0
DYING → GAMEOVER:  stateTimer abgelaufen + lives === 0
PLAYING → LEVELWIN: dotsEaten === totalDots
LEVELWIN → READY:  stateTimer abgelaufen (2000ms), level++, voller Maze-Reset
GAMEOVER → START:  ENTER-Taste gedrückt (Highscore speichern)
```

### Level-Progression
- Level 1: Standard-Geschwindigkeit, Frightened 6s
- Jedes Level: `pacman.speed *= 1.05`, `ghost.speed *= 1.05`, `frightenedDuration -= 500ms`
- Cap bei Level 10: Maximalgeschwindigkeit, Frightened = 0s

---

## 9. UI / Visuals

### Farbschema (klassisches Pac-Man)

```javascript
const COLORS = {
  bg:          '#000000',
  wall:        '#1a1aff',
  wallInner:   '#000088',
  dot:         '#ffb8ae',
  pellet:      '#ffb8ae',
  pacman:      '#ffff00',
  ghostFright: '#0000ff',
  ghostWhite:  '#ffffff',
  text:        '#ffffff',
  textYellow:  '#ffff00',
  textCyan:    '#00ffff',
  fruit:       '#ff0000',
};
```

### Pac-Man Zeichnung
```javascript
ctx.beginPath();
ctx.arc(x, y, TILE*0.42, rot + mouth*PI, rot + (2-mouth)*PI);
ctx.lineTo(x, y);
ctx.fillStyle = '#ffff00';
ctx.fill();
```

### Geister-Zeichnung (pixel-art mit `fillRect`)
- Körper: halbkreisförmige Haube + gezackter Boden (3 Wellen)
- Augen: 2 weisse Kreise + 2 Pupillen in Bewegungsrichtung
- Frightened: Blauer Körper + weisses Wellenmuster für Mund
- Eaten: Nur Augen (in Bewegungsrichtung schauend)

### HUD
```
Oben: "PUNKTE" [Score] | "HIGHSCORE" [Wert]
Unten: [Leben-Icons: 3x kleiner Pac-Man] [Frucht-Icon] [Level-Indikator]
```

### Overlays
- **START:** `"PAC-MAN"` (gelb, gross) + `"ENTER drücken"` (blinkend) + Highscore
- **BEREIT:** `"BEREIT!"` in Gelb, zentriert
- **GAME OVER:** `"GAME OVER"` in Rot + Endpunktzahl + Neustart-Hinweis
- **LEVEL WIN:** Labyrinth-Blinkeffekt (Wände wechseln 6× zwischen Blau und Weiss)

### Blink-Effekte
- Power-Pellets blinken alle 500ms: `Math.floor(Date.now()/500) % 2`
- Ghost-Frightened blinkt weiss wenn `ghost.modeTimer < 2000`
- Overlay-Text blinkt: `Math.floor(stateTimer/500) % 2`

---

## 10. Implementierungsreihenfolge

### Schritt 1 – Gerüst & Labyrinth (Milestone: Maze sichtbar)
1. `index.html` anlegen: DOCTYPE, Canvas 560×720, inline `<style>`
2. Konstanten: `TILE=20`, Tile-Typ-Enum `T`, Canvas-Dimensionen
3. Maze-Daten: 28×31 Array hardcoden (klassisches Layout)
4. `drawMaze()`: Wände als blaue `fillRect`, Dots als kleine Kreise, Pellets als grosse Kreise
5. Im Browser testen

### Schritt 2 – Pac-Man Bewegung (Milestone: Pac-Man steuerbar)
1. `pacman`-Objekt, Startposition
2. `updatePacman(delta)`: progress-Bewegung, Tile-Wechsel, Wand-Kollision
3. `drawPacman()`: Mund-Animation, Rotationsrichtung
4. Keyboard-Handler: Pfeiltasten → `pacman.nextDx/nextDy`
5. Game Loop: `requestAnimationFrame`
6. Tunnel-Wrapping

### Schritt 3 – Pellet-Logik & Score (Milestone: Pellets fressbar)
1. Dot-Eat-Erkennung
2. Score-Variable, `drawHUD()`
3. `totalDots`/`dotsEaten` → Level-Win-Erkennung
4. `localStorage` Highscore
5. Leben-Anzeige

### Schritt 4 – Audio (Milestone: Sounds hörbar)
1. `ensureAudio()`, `audioCtx` lazy init
2. `playDotEat(alt)`: alternierend 2 Töne
3. `playPowerPellet()`, `playDeath()`, `playLevelWin()`, `playGameOver()`
4. `startFrightenedSound()` / `stopFrightenedSound()`
5. `playGhostEat()`

### Schritt 5 – Geister-Grundbewegung (Milestone: Geister bewegen sich)
1. `createGhost()` für alle 4 Geister
2. `updateGhost(ghost, delta)`: Tile-basierte Bewegung
3. `drawGhost(ghost)`: farbiger Körper + Augen
4. Ghost-House-Logik: Schweben, Rauslass-Bedingungen
5. Scatter-Modus, grundlegendes Tile-Pathfinding

### Schritt 6 – Geister-KI (Milestone: KI-Modi funktionieren)
1. Globaler Phasen-Timer: Scatter ↔ Chase
2. Chase-Ziel je Geist
3. Frightened-Modus: Random-Pathfinding, blauer Körper
4. Frightened blinkt wenn `modeTimer < 2000`
5. `startFrightenedSound()` / `stopFrightenedSound()` einbinden

### Schritt 7 – Kollision & Leben (Milestone: Sterben und Respawn)
1. Pac-Man ↔ Geist Pixel-Distanz-Check
2. `pacmanDies()`: lives--, Todesanimation
3. Respawn-Logik
4. `pacman.invincible` Timer nach Respawn
5. Geist fressen: Score-Combo, 'eaten'-Modus
6. Eaten-Geist findet Weg zurück

### Schritt 8 – Spielzustände & Overlays (Milestone: Komplettes Spiel spielbar)
1. State Machine komplett
2. Start-Overlay
3. BEREIT!-Overlay
4. Level-Win: Labyrinth-Blinkanimation
5. Game-Over-Overlay
6. Level-Zähler, Level-Progression
7. ENTER-Handling für alle States

### Schritt 9 – Polish (Milestone: 80er-Feeling)
1. Bonus-Frucht (Kirsche) bei 70 Dots
2. Punkte-Popup beim Geist fressen
3. Power-Pellets blinken
4. Extra-Leben bei 10'000 Punkten
5. Waka-Waka Beat-Mechanismus
6. HUD: Leben als kleine Pac-Man-Icons

### Schritt 10 – Integration (Milestone: In Sammlung integriert)
1. `CLAUDE.md` im Pac-Man-Ordner anlegen
2. `index.html` (Hub) erweitern: CSS-Klasse `.pacman` + viertes `game-card`-Element

---

## Offene Entscheidungen / Risiken

| Thema | Empfehlung |
|-------|-----------|
| Maze-Hardcoding | Direktes 2D-Array ist am lesbarsten |
| Wand-Zeichnung | Einfache Rechtecke mit 2px `strokeStyle` für Innen-Kante |
| Inky-Zielberechnung | Kann vereinfacht werden wenn Vektor-Dopplung zu komplex |
| Tunnel-Geschwindigkeit | Vereinfachung: volle Geschwindigkeit in WRAP-Tiles |
| Frightened blinkt | `ghost.modeTimer < 2000` → toggle zwischen Blau/Weiss |
