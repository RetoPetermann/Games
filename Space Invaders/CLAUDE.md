# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file Space Invaders browser game written entirely in one HTML file (`Space Invaders.html`). No build system, no dependencies, no package manager – just vanilla HTML5 Canvas + JavaScript.

## How to Run

Open `Space Invaders.html` directly in a browser. No server required.

## Architecture

Everything lives in a single `<script>` block inside the HTML file, organized into clearly marked sections:

- **Sound Engine** (Web Audio API): Functions like `playShoot()`, `playInvaderKill()`, `playUFOKill()` etc. generate all sounds procedurally via oscillators. UFO has a continuous looping sound (`startUFOSound`/`stopUFOSound`).
- **Game State**: Global variables for score, lives, level, and entity arrays (invaders, bullets, shields, explosions, UFO).
- **Drawing**: Pixel-art rendering via `ctx.fillRect()` calls. Invaders use pixel coordinate arrays for two animation frames per type (3 types: red/green/blue).
- **Level Setup**: `createInvaders()` builds a 5×11 grid, `createShields()` places 4 destructible shields. `initLevel()` resets per-level state with difficulty scaling based on level number.
- **Update Loop**: Handles player input (arrow keys + space), invader grid movement with edge-bounce-and-drop, collision detection (bullet-invader, bullet-shield, bullet-player, invader-shield), UFO spawning, and level/game-over transitions.
- **Draw Loop**: Renders all entities, HUD (score/level/lives), start screen, game-over and win screens.

## Key Game Mechanics

- Invader movement speeds up as fewer invaders remain (dynamic `invaderMoveInterval`).
- Difficulty increases per level: faster invaders, shorter move intervals, higher shoot chance. 10 levels to win.
- Shields are pixel-based and destructible – bullets and invaders eat away at them with splash damage.
- UFO appears randomly after a cooldown, awards random points (50/100/150/300).
- UI text is in German (Schweizer Hochdeutsch): "Punkte", "Leben", "Leertaste", "Pfeiltasten".
