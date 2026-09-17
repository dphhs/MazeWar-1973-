# Mazewar

**A first-person 3D maze game rendered entirely in hardware on a DE1-SoC FPGA.**

No CPU, no GPU, no soft core. Every pixel comes from a state machine in SystemVerilog driving a 640×480 VGA display — using **zero DSP blocks**, because the render path does no multiplication or division at runtime.

![Language](https://img.shields.io/badge/SystemVerilog-RTL-blue)
![Board](https://img.shields.io/badge/board-DE1--SoC-green)
![FPGA](https://img.shields.io/badge/FPGA-Cyclone%20V%205CSEMA5F31C6-green)
![Toolchain](https://img.shields.io/badge/Quartus-Prime%2018.1%20Lite-orange)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

<!-- MEDIA: demo GIF. 5-8s loop, ~700px wide, <10MB so GitHub autoplays it inline. -->
<p align="center">
  <img src="doc/media/demo.gif" alt="Mazewar running on the DE1-SoC" width="700">
</p>

<p align="center"><em>▶ <a href="#">Full demo video</a></em></p>

---

## What it is

A hardware reimplementation of [Mazewar](https://en.wikipedia.org/wiki/Maze_War) — the 1973 game credited as the first first-person 3D shooter — built as a digital design course project.

You walk an 11×21 maze in first person, with a live top-down minimap showing position and facing. The original ran on Imlac minicomputers; this runs on nothing but logic and block RAM.

## Features

- Perspective-projected first-person view — corridors, corner openings, end walls
- Live 2D minimap with directional player marker
- Grid collision detection — walls block movement before it commits
- 640×480 framebuffer at 9-bit color, 60 Hz
- Hardware primitives: Bresenham lines, filled rectangles and triangles, arbitrary quads
- No multipliers, dividers, or floating point anywhere in the render path

## Hardware and controls

| | |
|---|---|
| **Board** | Terasic DE1-SoC |
| **FPGA** | Intel Cyclone V `5CSEMA5F31C6` |
| **Display** | VGA, 640×480 @ 60 Hz, 9-bit color |
| **Input** | PS/2 keyboard |
| **Clock** | 50 MHz `CLOCK_50`; pixel clock from PLL |

| Control | Action |
|---|---|
| <kbd>W</kbd> / <kbd>S</kbd> | Move forward / backward |
| <kbd>A</kbd> / <kbd>D</kbd> | Turn left / right |
| `SW[0]` | System reset |
| `SW[1]` | Reset player to start position |

---

## How it works

### Architecture

<!-- MEDIA: export doc/TopLevelDiagram.drawio to SVG at doc/media/architecture.svg -->
<p align="center">
  <img src="doc/media/architecture.svg" alt="Top-level block diagram" width="800">
</p>

| Module | Responsibility |
|---|---|
| `player` | Position and facing. Decodes input, tests the target cell, commits the move only if free. |
| `draw_screen` | The renderer — an 11-state FSM that walks the maze and emits geometry. |
| `vga_adapter` | Framebuffer and VGA timing. Accepts `(x, y, color)` writes. |

`player` publishes `px`, `py`, `direction`; `draw_screen` streams pixel writes into the framebuffer. Frames redraw only on movement, so the renderer idles most of the time.

### Perspective without arithmetic

A textbook raycaster casts one ray per screen column and divides to find each wall's projected height — 640 divisions per frame. Dividers are expensive in hardware.

This design exploits a constraint the general algorithm ignores: on a grid, facing one of four directions, **every visible wall sits at one of 16 discrete depths**. So every wall edge can only land at one of 16 screen positions, and those are precomputed:

```systemverilog
parameter logic [12:0] Distance [0:15] = '{
    13'd35,  13'd65,  13'd90,  13'd111,   // depth 0-3
    13'd128, 13'd141, 13'd151, 13'd158,   // depth 4-7
    ...
};
```

The values crowd together with depth — that non-linear spacing *is* the perspective projection, evaluated once at design time instead of 640 times per frame. At runtime the renderer only indexes the table and adds. **Hence zero DSP blocks.**

Each frame, the FSM:

1. Clears the screen and redraws the minimap cell by cell
2. Walks forward from the player to find how far the corridor extends
3. **Walks the left wall**, counting *runs* of consecutive same-type cells and emitting one quad per run — a five-cell wall costs one quad, not five
4. Walks the right wall, mirroring the same logic
5. Draws the viewport frame and the end wall

Run-length merging is what keeps the primitive count inside the frame budget.

### Primitives

Everything reduces to one pixel-per-clock line engine:

| Primitive | Implementation |
|---|---|
| `draw_line` | Bresenham, Zingl's integer formulation — one pixel per clock |
| `draw_rectangle_fill` | Scanline fill: one horizontal line per row |
| `draw_triangle_fill` | Minimap player marker |
| `draw_quad` | Four chained `draw_line` calls — the wall bands |

Each is its own FSM behind a uniform `start`/`busy`/`done` handshake, so `draw_screen` composes them without tracking timing.

### Collision

The maze is 21 rows of 11 bits, one bit per cell. Since movement is one cell along one axis, collision is a single lookup tested before the move commits:

```systemverilog
if (map[py + forward_y][px + forward_x] == 0) begin
    px <= px + forward_x;
    py <= py + forward_y;
end                       // else: move rejected, position holds
```

`forward_x`/`forward_y` decode combinationally from the 2-bit `direction`, so three lines cover all four facings.

---

## Results

Post-fit, Quartus Prime 18.1 Lite, `5CSEMA5F31C6`:

| Resource | Used | Available | Utilization |
|---|---|---|---|
| Logic (ALMs) | 1,193 | 32,070 | **4%** |
| Registers | 1,137 | — | — |
| Block memory | 2,764,800 bits | 4,065,280 bits | **68%** |
| RAM blocks | 338 | 397 | **85%** |
| **DSP blocks** | **0** | 87 | **0%** |
| PLLs | 1 | 6 | 17% |
| I/O pins | 98 | 457 | 21% |

Timing at 50 MHz (slow 1100 mV 85 °C): setup slack **+6.987 ns**, hold slack +0.231 ns, **TNS 0.000**.

**The design is memory-bound, not logic-bound.** The framebuffer is 640 × 480 × 9 = 2,764,800 bits — exactly the block memory figure above, and 85% of the device's RAM blocks, against 4% logic. Future features are constrained by memory, not area: there's no room for a second framebuffer, so double buffering would require dropping color depth.

Timing closes with ~7 ns margin because the FSM spreads drawing across cycles rather than into deep combinational paths.

---

## Repository layout

```
├── code Submit/          Final submitted design — canonical source
│   ├── game_top.sv           Top level: player + renderer + VGA adapter
│   ├── player.sv             Position, facing, collision
│   ├── draw_screen.sv        The rendering FSM
│   ├── draw_quad.sv          Wall-band primitive
│   ├── draw_line.sv          Bresenham line engine
│   └── draw_rectangle_fill.sv
├── src/                  Development tree: full primitive set, earlier iterations
│   ├── draw/                 draw_triangle_fill, draw_polygon, draw_rectangle, ...
│   ├── vga_adapter/          DE1-SoC VGA driver and PLL
│   └── MIF/                  Memory initialisation files
├── test/                 ModelSim testbenches, one per module
├── project/              Quartus project (.qpf/.qsf) and prebuilt bitstream
├── doc/                  Design diagrams and original proposal
└── Fun Version/          Bitstream snapshots from development
```

## Build and run

Requires Quartus Prime 18.1 Lite, a DE1-SoC, a VGA monitor, and a PS/2 keyboard.

**Flash the prebuilt bitstream** — no compilation needed:

1. Connect over USB-Blaster and power on
2. Quartus → **Tools → Programmer**
3. Select `project/vga_demo.sof` → **Start**

**Build from source:**

1. Open `project/vga_demo.qpf`
2. **Processing → Start Compilation**
3. Program the resulting `.sof` as above

Pin assignments are already in the `.qsf`.

> **Note** — the top-level module is still named `vga_demo`, inherited from the Terasic template. If you rename it, update `TOP_LEVEL_ENTITY` in the `.qsf`.

## Simulation

Each module has a ModelSim testbench under `test/` with a `.do` script that compiles, loads waveforms, and runs stimulus:

| Directory | Covers |
|---|---|
| `test/draw_screen_test/` | Rendering FSM end to end |
| `test/player_test/` | Movement, turning, collision rejection |
| `test/trig_test/` | Sine LUT from the earlier trig approach |
| `test/vertex_RAM_test/` | Vertex RAM read path |
| `test/vertex_load_test/` | Vertex loader |

```tcl
cd test/draw_screen_test
do tb_draw_screen.do
```

<!-- MEDIA (optional): ModelSim waveform screenshot at doc/media/waveform.png -->

---

## Design evolution

The design that shipped isn't the one proposed, and the gap is the interesting part.

**Proposed** (`doc/Schematic.drawio`): a conventional raycaster — one ray per column, compute distance, divide for projected height, write one column per cycle. It called for a `Ray_Caster`, `Distance_Calculator`, `Column_Update`, and two clock domains.

**The problem:** arithmetic cost. Per-column projection needs division, 640 times per frame — either many dividers or many stalled cycles. An early sine-LUT attempt survives in `test/trig_test/`.

**What shipped** (`doc/TopLevelDiagram.drawio`) replaced that with the 16-entry LUT and run-length-merged quads, removing runtime arithmetic entirely.

**What I'd do differently:**

- **Free-angle movement.** The LUT is what makes the design cheap *and* what locks the player to a grid and four facings. Supporting arbitrary angles means going back to real projection arithmetic — a trade of area for freedom, not a free win.
- **Merge the mirrored FSM states.** `INITLEFT`/`DRAWLEFT` and `INITRIGHT`/`DRAWRIGHT` differ mainly in sign; parameterizing one path would roughly halve the state machine.
- **One source of maze truth.** The map is declared identically in both `player` and `draw_screen`.
- **Double buffering**, if color depth were cut to free the RAM.

---

## Credits and license

Drawing primitives derive from the **[Project F](https://projectf.io) Verilog library** by Will Green (MIT) — the line, rectangle, and triangle engines. `draw_quad` extends Project F's rectangle module to arbitrary quadrilaterals. The line algorithm follows [Alois Zingl's formulation](http://members.chello.at/~easyfilter/bresenham.html).

The VGA adapter, address translator, and PLL are University of Toronto / Terasic DE1-SoC reference components.

Everything else — the renderer, the LUT approach, player and collision logic, and system integration — is my own work. Built as a course project with one collaborator.

MIT licensed. See [LICENSE](LICENSE).
