# PECMEN: A Pac-Man Inspired FPGA Game

A maze-chase game built entirely in Verilog for the **Xilinx Basys 3** FPGA board. The player navigates a procedurally generated maze, collects pellets, and avoids three ghosts with distinct AI behaviours — all rendered live on a 96×64 OLED display.

This project was completed as a final design project for a digital design / FPGA module, by a four-person team (Team S5_06).

---

## Overview

PECMEN reimagines the classic Pac-Man formula as a from-scratch hardware system rather than a software game. Every element — maze generation, ghost pathfinding, collision detection, audio, and rendering — runs as dedicated Verilog logic on the FPGA, with no soft-core processor involved.

Core gameplay features:
- A **freshly randomised maze** every round, generated procedurally rather than hardcoded
- A pellet-collection win condition, with **3 lives** and a post-collision immunity window
- **Three ghost AI behaviours** (Chase, Random, Charge), unlocked progressively across **three difficulty levels**
- **Power pellets** with two distinct effects: a frightened mode and a speed-boost mode
- Pac-Man **mouth animation**, a **live score display**, and **in-game background music and sound effects**
- A **cheat switch** (SW15) that disables collisions for testing/demo purposes

---

## Implementation Platform

- **HDL:** Verilog
- **FPGA Board:** Digilent Basys 3 (Xilinx Artix-7, `xc7a35tcpg236-1`)
- **Toolchain:** Xilinx Vivado 2018.2
- **Audio Output:** Digilent PmodAMP2 amplifier module
- **Display:** 96×64 OLED, addressed as a 19×12 tile grid

---

## Team & Component Ownership

| Member | Component(s) |
|---|---|
| Lim Jun Hao Bjorn | Ghost AI (Chase / Random / Charge) · Power Pellet Logic · Score Display |
| Liau Chee Lok | Random Maze Generation · Pellet Manager · Maze Query Bus · Audio System |
| Petra Quek E-Ling | Game FSM · Renderer |
| Go Qhin Cheng Clarkson | Player Controller · Mouth Animation · 3-Lives System |

---

## Core Modules & Technical Design

### Ghost AI
- **Chase Ghost (Red):** Distance-based pathfinding that actively pursues the player, with a no-reverse rule unless no other move is available. Moves on a 6 Hz tick.
- **Random Ghost (Orange):** Driven by a 12-bit Fibonacci LFSR cycling through four priority direction orderings, taking the first valid non-reverse move. Also a 6 Hz tick.
- **Charge Ghost (White):** Travels in a straight line until blocked, then picks a new heading at random via the LFSR. If it shares a row or column with the player, it locks on and charges straight at them, on a faster 8 Hz tick.

Difficulty scaling: **Easy** = Chase only · **Normal** = Chase + Random · **Hard** = Chase + Random + Charge.

### Power Pellets
- **Frightened pellet (purple):** Ghosts turn purple and flee using a maximum-distance heuristic for 4 seconds, with collisions disabled during this window.
- **Speed pellet (green):** Player sprite turns green and movement speed jumps from 16 Hz to 24 Hz for 4 seconds.

### Random Maze Generation
An FSM combined with an LFSR procedurally builds a new maze each round, enforcing:
- A forced centre opening and symmetrical layout
- Fixed outer border walls
- Flood-fill verification so no region of the maze is disconnected
- Dead-end pruning
- Open-tile counting, triggering a regeneration if the maze comes out too cramped or unplayable

Ghost spawn points are validated against a minimum distance from the player and confirmed to not land on a wall. Pellet count scales with the generated layout, and the remaining-pellet count drives the win condition. Power pellets are placed randomly within the generated maze.

### Maze Query Bus & Pellet Manager
A shared query bus exposes maze-tile and pellet-state information to the modules that need it (renderer, ghosts, player controller), keeping maze data centralised rather than duplicated across modules.

### Audio System (PmodAMP2)
- A looping Pac-Man-style background melody
- Dedicated sound effects for collisions, winning, losing, eating a pellet, power-ups, and button presses (via edge detection on BTNC)
- A sound-priority scheme to arbitrate between simultaneous audio events

### Game FSM & Renderer
- A 4-state game FSM: `DIFF_SEL → PLAYING → DEAD/WIN`, with difficulty chosen via BTNU/BTND and BTNC used to confirm selections and return to the menu after a win/loss
- Win triggers when no pellets remain; loss triggers when lives reach zero
- A fully combinational pixel renderer draws the maze, pellets, power pellets, ghosts, the title/menu/difficulty screens, and the win/lose screens onto the 96×64 OLED

### Player Controller
- Tile-based movement at 10 Hz, with **desired-direction latching** so a queued turn executes as soon as the path opens up
- 4-direction wall collision checking, using separate wires for the current vs. desired heading so the player continues straight if a turn is currently blocked
- Boundary clamping to prevent running off the edge of the maze
- A spawn lock that holds the player at the centre tile until the first input is received

### Mouth Animation & Lives System
- Pellet-eaten events are edge-detected into a single pulse, which drives a 125 ms countdown timer that keeps the mouth visibly open after each pellet is eaten
- A 2-bit facing-direction register, updated only on successful movement, keeps the mouth sprite oriented correctly
- 3 lives are tracked and shown on the board's LEDs, with one life lost per ghost collision
- A 2-second immunity window follows each spawn or collision, indicated by a 4 Hz LED blink

---

## Repository Structure

```
PECMEN-PacMan-FPGA/
├── README.md
├── docs/
│   └── Report_S5_06.pdf      # Full project report (component write-ups, screenshots, references)
└── vivado_project/
    └── FDP_Pacman.xpr        # Vivado project descriptor (Vivado 2018.2, target part xc7a35tcpg236-1)
```

> **Note:** This repository currently includes the Vivado project descriptor and full report, but not the underlying `.v` source files / `.xdc` constraints themselves. The `.xpr` file references the following modules, which can be dropped into a `sources_1/new/` folder alongside it to restore a buildable project: `top_student.v`, `game_fsm.v`, `renderer.v`, `Oled_display.v`, `chase_ghost.v`, `random_ghost.v`, `charge_ghost.v`, `maze_generator.v`, `pellet_manager.v`, `score_display.v`, `audio_bgm.v`, `clk_divider.v`, `debounce.v`, and the constraint file `my_constraints.xdc`.

---

## Hardware Setup

- Digilent **Basys 3** board, programmed via Vivado
- **PmodAMP2** connected for audio output
- **SW15** acts as a cheat switch, disabling all collisions for demo/testing purposes
- Score and lives are shown on the board's onboard 7-segment display and LEDs respectively

---

## References

1. Arduino Forum thread on sound effect implementation — referenced for the audio playback approach. [forum.arduino.cc](https://forum.arduino.cc/t/sound-effect/49743)
2. `ridvancelikcs/Songs` GitHub repository — referenced for the Pac-Man background melody. [github.com/ridvancelikcs/Songs](https://github.com/ridvancelikcs/Songs)
3. Digilent Pmod AMP2 Reference Manual — referenced for PmodAMP2 hardware configuration. [digilent.com](https://digilent.com/reference/pmod/pmodamp2/reference-manual)

---

## Acknowledgements

Built by Team S5_06:
Lim Jun Hao Bjorn · Liau Chee Lok · Petra Quek E-Ling · Go Qhin Cheng Clarkson
