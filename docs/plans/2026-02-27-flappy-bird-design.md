# Flappy Bird - Level Edition: Design Document

**Date:** 2026-02-27
**Status:** Approved

## Overview

A Flappy Bird clone with 9 distinct visual levels that change every 15 points. Built as a single HTML file using HTML5 Canvas, targeting both PC and mobile browsers. Deployed to play.fun.

## Assets

Source: `Flappy Bird Assets 1.6` by Megacrash (CC0 license)

- 9 backgrounds (Background1-9.png)
- 2 bird styles x 7 color variants (14 birds total, individual frame PNGs)
- 5 tile styles (each with TileStyle, PipeStyle, SimpleStyle spritesheets)
- Pipe spritesheets contain 8 color variants each in a grid layout

## Game Screens

### 1. Loading Screen
- Black background, animated bird sprite (flapping + bobbing) centered
- Progress bar below bird (0-100%) tracking asset loads
- "FLAPPY BIRD" pixel title above bird
- Auto-transitions to home screen on completion

### 2. Home Screen
- Background1 (orange sunset) as backdrop with scrolling ground tiles
- "FLAPPY BIRD" large pixel title at top
- Animated bird flying in place
- Large PLAY button (pixel art styled)
- Best score display if exists

### 3. Gameplay Screen
- Classic Flappy Bird mechanics
- Score counter top-center
- Level indicator top-left ("LVL X")
- Scrolling ground tiles at bottom
- Pipes spawn right, move left

### 4. Game Over Screen
- Semi-transparent dark overlay on frozen game state
- Stats panel: Score, Best Score, Level Reached, Pipes Passed
- Retry button + Home button

## Level System

9 levels, infinite loop. Every 15 points, instant visual swap (no transition):

| Level | Background | Tile Style | Pipe Color | Bird |
|-------|-----------|-----------|-----------|------|
| 1 | BG1 (orange sunset) | Style 1 | Green | Bird1-1 (orange) |
| 2 | BG2 (cyan day) | Style 2 | Orange | Bird1-2 (green) |
| 3 | BG3 (dark blue night) | Style 3 | Blue | Bird1-3 (cyan) |
| 4 | BG4 (starry night) | Style 4 | Red | Bird2-1 (yellow) |
| 5 | BG5 (starry city) | Style 5 | Purple | Bird2-2 (red) |
| 6 | BG6 (cyberpunk) | Style 1 | Cyan | Bird2-3 (green) |
| 7 | BG7 (green hills) | Style 2 | Brown | Bird1-4 (purple) |
| 8 | BG8 (brown desert) | Style 3 | Yellow | Bird2-4 (blue) |
| 9 | BG9 (dark desert) | Style 4 | White | Bird1-5 (pink) |

After level 9 (135 points), loops to level 1 visuals. Difficulty continues to increase up to a cap.

## Difficulty Progression

| Level | Pipe Speed (px/frame) | Gap Size (px) | Pipe Spacing (px) |
|-------|----------------------|--------------|-------------------|
| 1 | 2.0 | 130 | 220 |
| 2 | 2.2 | 125 | 210 |
| 3 | 2.4 | 120 | 200 |
| 4 | 2.6 | 115 | 195 |
| 5 | 2.8 | 110 | 190 |
| 6 | 3.0 | 105 | 185 |
| 7 | 3.2 | 100 | 180 |
| 8 | 3.4 | 95 | 175 |
| 9 | 3.6 | 90 | 170 |

Difficulty caps after level 9 values on subsequent loops.

## Controls

- **PC:** Spacebar or left-click to flap
- **Mobile:** Tap anywhere to flap
- Responsive canvas (portrait-friendly on mobile, centered on desktop)

## Audio (Web Audio API - Programmatic)

- **Flap:** Short chirp (sine wave, quick pitch rise)
- **Score:** Pleasant ding (triangle wave)
- **Hit/Die:** Low thud + descending tone
- **Level Change:** Quick ascending arpeggio
- **Background Music:** Looping chiptune melody (8-bit style)

## Technical Architecture

- **Single `index.html`** with inline `<script>` and `<style>`
- **HTML5 Canvas** for all rendering
- **`requestAnimationFrame`** game loop with delta-time
- **Assets** loaded as `Image()` objects from `/assets/` folder
- **`localStorage`** for best score persistence
- **Viewport meta tag** + responsive scaling for mobile
- **Web Audio API** for all sound (no audio file dependencies)

## Deployment

Target: https://play.fun/
Method: playdotfun CLI plugin
