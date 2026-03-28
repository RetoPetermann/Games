# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Sprache

- Alle UI-Texte in Schweizer Hochdeutsch (kein Dialekt, "ss" statt "ß").

## Überblick

Sammlung von drei Browser-Arcade-Spielen. Keine Build-Tools, keine Abhängigkeiten, keine Frameworks. Jedes Spiel ist eine einzelne HTML-Datei mit inline CSS und JavaScript.

## Entwicklung

Dateien direkt im Browser öffnen – kein Server nötig:
```
open index.html
```

## Architektur

- **Rendering:** HTML5 Canvas 2D API
- **Audio:** Web Audio API – alle Sounds werden prozedural erzeugt (Oszillatoren, Noise-Buffer, Filter). Keine Audio-Dateien.
- **Struktur pro Spiel:** Ein HTML-File enthält alles (Style, Canvas, Script). Script-Aufbau: Konstanten → Audio-Funktionen → Game State → Hilfsfunktionen → Update/Draw Loop → Input Handler → Init/Game Loop.

## Spiele

| Spiel | Pfad | Canvas |
|-------|------|--------|
| Asteroids | `Asteroids/index.html` | 800×600 |
| Space Invaders | `Space Invaders/Space Invaders.html` | 640×480 |
| Tetris | `Tetris/tetris.html` | 300×600 |

`index.html` im Root ist die Hub-Seite mit Links zu allen Spielen.

## Konventionen

- Jedes Spiel hat eine eigene `CLAUDE.md` mit spielspezifischen Details.
- Neue Spiele als einzelne HTML-Datei in eigenem Unterordner anlegen, mit eigener `CLAUDE.md`.
- Hub-Seite (`index.html`) bei neuen Spielen aktualisieren.
- localStorage-Keys mit Spielnamen prefixen (z.B. `asteroids_high`).
