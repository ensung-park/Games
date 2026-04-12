# Sweet Match Match-3 — World-Class Upgrade Design

## Overview

Transform the basic match-3 prototype into a polished, addictive game with special candies, level progression, obstacles, canvas rendering, procedural audio, and full visual polish. Single self-contained HTML file, no external dependencies.

## Core Mechanics

### Special Candies

| Trigger | Creates | Effect |
|---------|---------|--------|
| Match 4 in a row | Striped candy (direction of match) | Clears entire row or column |
| Match in L or T shape | Wrapped candy | Explodes 3x3 area, then again after falling |
| Match 5 in a row | Color bomb (rainbow) | Swap with any candy to clear all of that color |

### Special + Special Combos

| Combo | Effect |
|-------|--------|
| Striped + Striped | Clears full row AND column (cross) |
| Striped + Wrapped | Clears 3 rows + 3 columns |
| Wrapped + Wrapped | 5x5 explosion |
| Color bomb + any candy | All candies of that color become striped, then detonate |
| Color bomb + Color bomb | Clears entire board |

### Blockers

| Blocker | Behavior |
|---------|----------|
| Ice (1-2 layers) | Covers a candy. Match the candy to remove one layer. |
| Chocolate | Spreads to one adjacent empty cell per turn if not cleared. Match adjacent to remove. |
| Stone | Indestructible. Blocks grid space permanently. |

## Level System

- 30 levels across 3 difficulty tiers (Easy 1-10, Medium 11-20, Hard 21-30)
- Each level defines: grid layout, move limit, 1/2/3-star score thresholds, blocker placements
- Level 1 is a tutorial (no blockers, generous moves)
- New mechanics introduced gradually: specials at level 3, ice at level 8, chocolate at level 14, stone at level 20
- End-of-level: remaining moves spawn random special candies that detonate for bonus points
- Level select screen with star display and lock/unlock gating

## Rendering

- HTML5 Canvas at 60fps via requestAnimationFrame
- Candies drawn as rounded rectangles with radial gradients and glossy highlight
- Each candy type has a distinct shape silhouette (circle, diamond, square, triangle, pentagon, heart) for accessibility
- Particle system: sparkles on match, burst on special activation, confetti on level complete
- Smooth tweened animations: gravity drop with bounce easing, swap slide, scale-in for new candies
- Screen shake on 4+ cascades or special combos
- Floating score text (+points) at match location, drifts upward and fades

## Audio (Web Audio API)

- Match pop: short sine wave burst, pitch increases with cascade level
- Special candy creation: rising arpeggio
- Special candy activation: explosion (noise burst + low sine)
- Combo sounds: escalating chime sequence
- Win jingle: major chord arpeggio
- Lose sound: descending minor notes
- UI clicks: short tick
- Sound toggle button in header

## UX

- Hint system: after 5s idle, one valid move pulses gently
- Auto-shuffle when no valid moves exist (with visual shuffle animation)
- Move counter and score bar with animated fill toward star thresholds
- Star award animation on level complete (stars fly in and bounce)
- Level failed overlay with retry option
- localStorage persistence: level progress, star ratings, high scores
- Responsive: works on mobile and desktop, touch and mouse

## Scoring

- 3-match: 30 points
- 4-match: 60 points
- 5-match: 100 points
- L/T match: 80 points
- Cascade multiplier: cascade_level * base_points
- Special activation bonus: +50 per special
- End-of-level bonus: remaining_moves * 50 (each spawns a special that detonates)

## File Structure

```
sweet-match/
  index.html   (~2500-3000 lines, fully self-contained)
```

## Non-Goals

- No multiplayer or online features
- No IAP or real-money mechanics
- No infinite/procedural level generation (fixed 30 levels)
- No external asset files or CDN dependencies
