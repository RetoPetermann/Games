# CLAUDE.md – Pac-Man

## Datei
`Pac-Man/index.html` – alles inline (HTML, CSS, JS), keine externen Abhängigkeiten.

## Canvas
560 × 670 px · TILE=20 · 28×31 Tiles · HUD oben 50px · Lives-Bar unten 40px

## Maze
`MAZE_TEMPLATE`: festes 31×28-Array (Row-Major). Geister-Haus-Wände werden in `buildMaze()` darüber gelegt.  
Tile-Typen: `T.WALL=0, T.DOT=1, T.PELLET=2, T.EMPTY=3, T.DOOR=4`  
Tunnel-Zeile: Row 12, Cols 0 und 27 (EMPTY) → wrap-around  
Geister-Haus: Rows 11–15, Cols 10–18; Tür bei Row 11, Cols 13–14

## Spielobjekte
- **pac**: Tile-Position (tx/ty), Pixel-Position (x/y), Richtung (dx/dy), Puffer-Richtung (ndx/ndy), Fortschritt innerhalb Tile (prog 0..1)
- **ghosts[4]**: Farben: Blinky=#ff0000, Pinky=#ffb8ff, Inky=#00ffff, Clyde=#ffb852
- **Geister-Modi**: `house` | `scatter` | `chase` | `frightened` | `eaten`
- **Dot-Limits für Rauslass**: [0, 30, 60, 90] Geist-Dots

## Steuerung
Pfeiltasten → Pac-Man · ENTER/Leertaste → Start / Neustart

## Audio (Web Audio API, prozedural)
- `beep(freq, endFreq, dur, type, vol)` – Basis-Helper
- `playDotEat(alt)` – alternierend 2 Töne (Waka-Waka via Timer)
- `playPowerPellet()` – Sweep aufsteigend
- `playGhostEat()` – Sweep absteigend
- `playDeath()` – 7-Ton-Leiter absteigend
- `playLevelWin()` – 6-Ton-Jingle aufsteigend
- `startFrightSound()` / `stopFrightSound()` – Sawtooth + LFO continuous

## Spielzustände
`START → READY (2.5s) → PLAYING → DYING (2.8s) → READY / GAMEOVER`  
`PLAYING → LEVELWIN (3.2s) → READY` (level++)

## Punkte
Dot=10 · Pellet=50 · Geist=200/400/800/1600 (Combo) · Frucht=100  
Extra-Leben alle 10'000 Punkte · Highscore in `localStorage` Key: `pacman_hi`

## localStorage
Key: `pacman_hi` (Highscore als String)
