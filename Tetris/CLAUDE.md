# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Browser-based Tetris game implemented as a single self-contained HTML file (`tetris.html`) with embedded CSS and JavaScript. No build tools, dependencies, or server required.

## Development

Open `tetris.html` directly in a browser to run. On macOS: `open tetris.html`

## Architecture

- **Single-file architecture**: All markup, styles, and game logic live in `tetris.html`
- **Rendering**: HTML5 Canvas (`<canvas>`) with a 10×20 grid (30px blocks)
- **Audio**: Web Audio API for synthesized sound effects (no external audio files) — oscillator-based tones and generated noise buffers
- **Game loop**: `requestAnimationFrame` with delta-time accumulator for gravity/drop timing
- **Piece system**: 7 standard tetrominoes defined as 2D arrays with numeric color indices; rotation via matrix transposition with wall-kick offsets
- **Collision detection**: `valid()` function checks shape against board boundaries and occupied cells
