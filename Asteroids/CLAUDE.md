# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Classic Asteroids arcade game clone — a single-file browser game (`index.html`) using HTML5 Canvas and vanilla JavaScript. No build tools, no dependencies, no framework.

## Running

Open `index.html` directly in a browser. No server required.

## Architecture

Everything lives in one `index.html` file with inline `<style>` and `<script>`:

- **Canvas rendering** (800×600): Ship, asteroids, bullets, particles, HUD — all drawn via `ctx` (2D context)
- **Game loop**: `loop()` → `update()` + `draw()` at requestAnimationFrame rate
- **Audio**: Web Audio API for procedural sound effects (shoot, explosions, thrust engine with LFO, background beat that accelerates as asteroids decrease, level-up jingle, game over)
- **State**: Global variables (`ship`, `bullets`, `asteroids`, `particles`, `score`, `lives`, `level`). `init()` resets all state.
- **Collision**: Simple distance-based (bullet↔asteroid, ship↔asteroid)
- **Asteroids**: Three sizes (3→2→1), split on hit, irregular polygon vertices for visual variety
- **Screen wrapping**: All objects wrap around canvas edges via `wrap()`
- **Persistence**: High score stored in `localStorage` under key `asteroids_high`

## UI Language

The game UI is in German (e.g. "PUNKTE", "LEVEL", "ENDPUNKTZAHL", "ENTER drücken zum Starten"). Keep UI text in German when making changes.
