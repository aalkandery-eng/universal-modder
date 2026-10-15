---
kind: game
title: A native PC port of Digimon World 3 from its matching decompilation, with 240 Hz game logic
game: Digimon World 3
games_also: []
game_version: "PS1 USA, SLUS-01436 (SLUS_014.36 SHA-1 444653259f78ddb483fd22af72cce9276f42f214), from the user's own disc (CHD)"
platform: windows
engine: unknown
route: decomp-recomp
tools: ["ReGame-Labs/dw3_decomp (MIT)", "Wyrelade/DW2-PC PC layer (CC0)", "WinLibs i686 GCC 16.2", "SDL3 3.4.18", "CMake + Ninja", "libchdr 0.3.0", "gdb", "SwarmUI (ComfyUI 0.24, SDXL, tile ControlNet)", "Real-ESRGAN ncnn-vulkan"]
anti_cheat: "none (single-player PS1 game)"
status: working
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-15'
links:
- "https://github.com/ReGame-Labs/dw3_decomp"
- "https://github.com/Wyrelade/DW2-PC"
- "https://github.com/rtissera/libchdr"
tags: [psx, ps1, decompilation, native-port, psyq, high-refresh, 240hz, pgxp, sub-pixel, widescreen, hd-textures, sdxl, chd]
---
# A native PC port of Digimon World 3 from its matching decompilation, with 240 Hz game logic

> The PS1 game's matching C decompilation compiled for 32-bit Windows, with PsyQ replaced by a PC layer reused
> from the Digimon World 2 port. The game reads its data from the user's own disc, the `.chd` included. It is
> playable end to end: field, battles, card game, menus, saves, sound and movies.
>
> The enhancements are true 16:9, an HD renderer with PGXP, an AI-upscaled texture pack, near-instant scene
> changes, and real 90/120/240 Hz game logic, not interpolation. At 240 Hz every moving thing advances each
> tick, and the field draws at sub-pixel positions. All of it was checked with deterministic replays against
> the 60 Hz build.

## Setup
- **Disc:** the user's own `.chd`. chdman `extractcd` gives a .bin/.cue for the decomp's extractor. The port
  later reads the CHD directly through libchdr: its hunks are 8 frames of 2448 bytes, and track 1 is
  MODE2_RAW.
- **Starting point:** the matching decomp (USA, 3,380 functions: executable, 21 overlays, 238 stages; the
  PsyQ libraries stay asm).
- **PC layer:** taken from DW2-PC: PsyQ on PC; GPU/SPU/XA/MDEC emulation with an HD renderer, PGXP and a
  Vulkan path (SDL_GPU); SDL3 host. It covers almost the same PsyQ set.
- **Toolchain:** i686 MinGW GCC 16.2, CMake/Ninja, a fixed image base `0x01000000`, no ASLR.
- **Build dirs:** a dev build (RelWithDebInfo, dev logs on) and a release build (-O2). A running exe blocks
  its own relink, so keep two or three build dirs.

## Route and why
`decomp-recomp`: the game's C is compiled natively. It is not a static recompilation of MIPS, and not an
emulator. With a 100% matching decomp and an existing sibling port (DW2-PC) built the same way, this route
gives readable code to change for enhancements, and everything stays under `#if DW3_NATIVE` so the PS1
build still matches.

Static recompilation was considered: it gives machine-translated code with nowhere sane to put new logic.
Emulator enhancements were considered too: they can't add real high-rate logic.

## How the game works (what we had to learn)
- **Memory.** 32-bit keeps every game struct at its PS1 layout. PS1 RAM, the heap and the image share one
  16 MB window, so the 24-bit OT links work as host pointers. Anything malloc'd lies outside that window
  and can't be linked into an OT; keep packet memory in a static pool inside the image.
- **Overlays and stages.** All are linked into one exe.
  - Each overlay's `.data`/`.bss` is reset when it "loads" (the DW2-PC `ld -r` trick).
  - The 238 stages all define the same names (`setupStage`, ...), so each is relinked with a `WSTAGnnn_`
    prefix on its defined globals. References to undefined symbols stay unprefixed, so stage code can
    call port functions.
- **Tasks.** A task tree, `executeTask` → `update(task, children)` → children. Tasks are attached by their
  creator storing them in its children array: a menu's text windows run right after the menu.
- **Layers.** Drawing goes into layers, each with its own OT, sent in order by `layerDraw`. Sprites come
  from one sprite drawer (SPRT, or POLY_FT4 when scaled). Field actors draw raw POLY_FT4s plus a SPRT
  shadow. Text windows run their control codes (waits, page breaks) *while drawing*.
- **Positions.** Field actors keep 8.8 fixed-point positions. The camera scrolls the map layer by whole
  pixels.
- **Frame cadence.** The game runs 60 game frames a second in the field and menus. It counts time in
  frames (`getFrameTime`).

## Build steps
1. Clone the decomp. Import the DW2-PC `pc/` layer, adding the PsyQ pieces DW3 needs that DW2 didn't
   (libds CD calls, card/api shims, a few libsnd functions).
2. Sound: DW2-PC's libsnd/libspu were translated asm at DW2 addresses. Align the 298 PsyQ functions the
   two games share (identical instruction for instruction) and remap 730 address pairs.
3. Build the game C with `-fno-toplevel-reorder -malign-data=abi`: some code reaches a global through its
   neighbour's address.
4. The R3000A never traps on division by zero (the game does `rand() % 0`). A vectored exception handler
   finishes `div`/`idiv` the MIPS way.
5. Enhancements, each under `#if DW3_NATIVE`: 16:9 (more map columns, a second VRAM bank), fast
   transitions (the game marks its transition frames; the clock runs them unthrottled), CHD, HD textures,
   and high-rate logic (below).

## High-rate logic (the main technique)
The user asked for real 240 fps, not interpolation. The approach:
- **Ticks.** The game loop waits for ticks at hz/60 × the VBlank rate; the PS1 VBlank stays at 59.94 Hz.
  `Logic_tick60` marks the tick that starts a 60 Hz frame.
- **Unconverted tasks** run only on those frames. Their packets are *kept* and replayed on the ticks in
  between: rebuilt into scratch OTs, in the 60 Hz frame's order (placeholders per owner), and shifted by
  the layer's scroll change.
- **Converted tasks** run every tick. They either step by `LOGIC_STEP(v) = v·dt/256`, or, more often,
  evaluate their *display* at the fraction `Logic_sub` of the next frame's step without touching state.
  That second pattern keeps game logic bit-identical to 60 Hz.
- **Stays at 60 Hz:** input, RNG, file loading, mode switches and buffer parity.
- **Partial conversions** mark packet ranges by owner (`Logic_OwnBegin/End`).
- **Converted so far:**
  - field actors with per-tick input, followers, camera, map, icons, balloons, banner, the yes/no panel;
  - battle cameras, models (sub-frame keyframes), moves and jumps, sprite effects, HUD bars, lights,
    stage fades, white flash;
  - card game sprites, windows and panels; title; screen fades;
  - stage tasks that move (found by surveying all 239 field modes);
  - menu panel open/close in 34 overlay update functions, generated from each overlay's header.
- **Fine 2D.** Each HD command gets a sub-pixel offset: the layer's base (−frac of the camera scroll) plus
  packet ranges (+frac of an actor, for its body, shadow, icon and balloon). The GPU path shifts
  fractionally (no 1× block texel rule, texels clamped to the primitive); the software path moves by
  whole HD pixels. On a slope at 240 Hz ×4 the map now moves exactly 1 HD px every tick, not 4 px every
  fourth tick.

## Verification
- **Determinism.** `--control DIR` takes an agent's command file (run N / press / hold / shot / state /
  mode M / quit), and replaying it resumes a session after a rebuild. The 60 Hz mode stays byte-identical:
  a new-game session (title → opening → names → field → first battle → report) gives 51/51 identical
  screenshots and the same mode timeline after every change.
- **Per-tick motion.** `DW3_TICK_SHOTS=FROM-TO` saves a picture per tick. Measure the per-tick shift
  (best integer translation of a crop) instead of eyeballing. HD shots need `--scale 4` and, for the
  GPU path, `--no-window --renderer gpu`; headless defaults to the software renderer.
- **Coverage.** `DW3_KEPT_LOG` (dev) lists, by function, the unconverted tasks and callbacks that still
  draw, per game mode. The survey requests every field mode 0x200..0x2EE in turn, 240 frames each, at
  240 Hz. It doubles as a crash test.
- **Not verified:** real movies (the user's image is an FMV-stripped rip with placeholder STRs), and the
  EU version.

## Gotchas
1. **240 Hz crashed in the first battle, deep in PGXP.**
   - **Cause:** a latent undefined behaviour. A vertex-cache entry marked "too many points" (n = CAP+1)
     still ran its duplicate loop over `c[0..n)`, reading one past the array. GCC 16 inferred n ≤ CAP and
     compiled the `n < CAP` guard as `n != CAP`, so a marked entry grew into its neighbours.
   - High-rate logic records several ticks per PGXP generation, which made it common.
   - **Fix:** return before the loop when n > CAP. Found with a gdb hardware watchpoint on the corrupted
     entry; the run is deterministic, so the slot index from the first crash was enough.
2. **A between-frame draw can read state the 60 Hz frame never drew.**
   - **Cause:** a flyer created during a 60 Hz frame was still in INIT, with sprite frame 0 (a whole
     picture). Drawn on the sub-ticks, it overran the 25 KB packet buffer into the PC layer's statics.
   - **Fix (rule):** a between path draws only what its 60 Hz frame drew. `Logic_RanState(task)` records
     the state each converted task's update ran in at the 60 Hz frame, and is emptied each frame so a
     reused task pointer has none.
3. **Kept packets drift for parallax and screen-fixed things in a scrolled layer.**
   - Replays are shifted by the full scroll change, so 1/8 or 2/3 parallax backgrounds and a countdown
     drawn "at scroll + offset" moved with the map between frames and snapped back. Convert those tasks
     to draw every tick against the current scroll.
   - With fine 2D the same applies to sub-pixel offsets: sprites that don't follow the scroll and every
     text window take a "still" range, and kept packets carry their range to the copy.
4. **Converted tasks that are dying** must be destroyed at a 60 Hz frame. A parent may still read them,
   which crashed the talk box.
5. **The mode switch, buffer toggle and PGXP generation** must advance only on 60 Hz frames. The battle
   transition moves on buffer-0 frames only, and the flip must show the buffer drawn last.
6. **-O3 release crashed after the title movie.** `allocMemHigh` was declared void and relied on a callee
   leaving the block in v0/eax. Bisect by building the game and the PC layer at different -O levels.
7. **Two static gotchas:** stage symbol collisions (relink with a prefix), and a mismatch between a
   US-only and an EU-only stage. A converted-list entry for an EU-only stage fails to link; guard it with
   `#if VERSION_EU`.
8. **Text windows can't simply be drawn between frames.** Their draw runs control codes (button waits,
   page breaks) that change state. Draw a copy, or leave text kept. This is why lerped menu slides that
   carry text stay at 60 Hz.
9. **Shell heredocs** turned `\\n` into real newlines in generated C several times. Write patch scripts
   with a file tool, not a heredoc.

## Assets
- **HD texture pack** (separate pipeline): runtime replacement keyed by upload hash plus CLUT hash, like
  DuckStation.
  - Field ground: each of the 237 maps' composite → Real-ESRGAN x4plus → SDXL img2img with tile ControlNet
    in overlapping 1024 windows (denoise 0.5), keeping the low frequencies → sliced back into 11,420 tiles.
  - Characters: Scale4x → animagine XL img2img with tile ControlNet.
  - Everything else: Real-ESRGAN.
  - 21,392 images, 5.6 GB, kept outside git.
- Everything ran on the user's local SwarmUI install (its ComfyUI backend on a separate port, settings
  untouched).

## Cost and time
About two days of agent time over several sessions: boot on day 1, sound/saves/16:9 the same day, high-rate
logic, HD textures and fine 2D on day 2. Most of the effort went into the long tail of conversions and the
replay-based checks.

## Open questions
- Lerped menu slides (their text windows), the memory card screen, the item shop's info panel and the battle
  menus still animate at 60 Hz.
- An x86_64 build should be possible with the low-window layout, but hasn't been tried.
- Real FMV needs a full disc dump.
