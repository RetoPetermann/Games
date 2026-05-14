# CLAUDE.md

## Überblick

Browser-basiertes Breakout-Spiel als einzelne HTML-Datei (`index.html`) mit inline CSS und JavaScript.

## Entwicklung

Direkt im Browser öffnen: `open index.html`

## Architektur

- **Canvas:** 480×600 px
- **Rendering:** HTML5 Canvas 2D API
- **Audio:** Web Audio API — Oszillatoren für alle Sounds, keine Audiodateien
- **Game Loop:** `requestAnimationFrame` mit Delta-Time (`dt`) für physikbasierte Bewegung
- **Kollision:** AABB-Kreis-Test mit Normalenreflexion (`bounceOffRect`)

## Spielelemente

| Element | Details |
|---------|---------|
| Paddle | 80×12 px, Maus oder Pfeiltasten, Winkelphysik je nach Treffposition |
| Ball | Radius 7, Lichtschweif (8 Frames), Radial-Gradient |
| Bricks | 10×7 Raster, 7 Farbreihen, ab Level 3 harte Bricks (2–3 HP) |
| Partikel | Spawn bei Brick-Treffer, Gravitation, Fade-out |

## Konventionen

- localStorage-Key: `breakout_high`
- Geschwindigkeit steigt mit Level: `240 + level * 18` px/s
- Combo-System: jeder 5. Treffer in Folge erhöht die Punktemultiplikation
- `gameState`: `idle | playing | dead | levelup | gameover`
