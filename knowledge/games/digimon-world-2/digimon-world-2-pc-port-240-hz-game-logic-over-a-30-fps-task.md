---
kind: game
title: 'Digimon World 2 PC port: 240 Hz game logic over a 30 fps task engine, with the game frames unchanged'
game: Digimon World 2
games_also: [Digimon World 3]
game_version: "PS1 USA, SLUS-01193; Wyrelade/DW2-PC v0.2.1 (64feb636) plus local changes"
platform: windows
engine: unknown
route: decomp-recomp
tools: ["Wyrelade/DW2-PC v0.2.1 (CC0)", "WinLibs i686 GCC 16.2 + gdb", "SDL3 3.4.18"]
anti_cheat: "none (single-player PS1 game)"
status: working
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-15'
links:
- "https://github.com/Wyrelade/DW2-PC"
tags: [psx, ps1, decompilation, native-port, high-refresh, 240hz, game-logic, determinism, replay, task-system]
---
# Digimon World 2 PC port: 240 Hz game logic over a 30 fps task engine, with the game frames unchanged

> DW2-PC (the native build of the matching Digimon World 2 decompilation) got real 90/120/240 Hz frames:
> - the game loop runs at every tick of a host tick clock;
> - the game's own logic keeps its 30/60 Hz cadence exactly;
> - what moves is drawn where its own motion has it at each tick (no blending between past frames, so no
>   added delay).
>
> Proof: a step-play replay from a new game to a won dungeon battle (8918 game frames) gives the same mode,
> random index and cadence at every game frame, and the same final state, at 90, 120 and 240 Hz as at 60. The
> PS1 matching build stays byte-identical. The same session carried over other enhancements from the DW3 port
> (fast transitions, battle speed, battle text speed, EXP/bits multipliers, frame cap, a PGXP fix, R3000A
> division).

## Setup
- **Source:** DW2-PC v0.2.1 plus the CHD support from the companion note.
- **Toolchain:** i686 WinLibs GCC 16.2, `tools/build_native.py`.
- **Disc data:** the build reads data tables from the disc (`dumps/disc`).
- **Headless verification only:** `--no-window`, step-play `--control DIR` (from the DW3 port), with `--fast`
  so replays are deterministic.

## Route and why
`decomp-recomp`. DW3's machinery could not be copied: DW3 keeps every unconverted task's packets and re-links
them. DW2's task system is friendlier: each task has a separate update (pass 0) and draw (pass 1) callback
(`Task_Run`). So the frames between run the draw pass for real, and the logic is gated:
- updates and destroys run at game frames only, unless the task is "converted";
- input, RNG, CD/sound loading, the frame and task counters, buffer parity and the mode switch run at game
  frames only.
Interpolating between past frames was ruled out (it adds a frame of delay, and the user asked for real logic).

## How the game works (what we had to learn)
- **Most of DW2 is 30 fps.**
  - City, domains, battles, title and VS menus call `Sys_SetFrameRate30` (`vsyncWait` 2); menus run at 60.
  - So 240 Hz is 8 ticks per game frame, and the gain over 30 is large.
- **The flip lives in the VBlank handler.**
  - `Sys_VSyncHandler` flips when `Sys_FlipPending && Sys_VSyncsSinceFlip >= vsyncWait - 1`.
  - It counts every VBlank since the last flip, including VBlanks the game waits for inside a frame.
  - At high rate: set `Sys_FlipPending` at a game frame and keep it pending through the ticks. The handler ends
    it at the VBlank it would have flipped at, without drawing. The next frame is then a game frame.
  - That mirrors the PS1 cadence exactly. Counting only the VBlanks in tick waits drifted.
- **Showing a tick without breaking buffer parity.**
  - The normal flip draws OT `b` with `draw[!b]` and shows `disp[!b]`.
  - In every `Gpu_InitDoubleBuffer` mode, `draw[!b]` and `disp[b]` are the same VRAM area.
  - So each tick draws OT `b` with `draw[!b]` and shows `disp[b]`: no latency.
  - `bufIndex` itself toggles at game frames only.
- **One ordering table** of 0x100C entries per buffer, `ClearOTagR`-style (each entry links to the one before).
  A single walk from the last entry visits every packet with its entry.
- **3D actor transforms:** `Obj209` / `AllocC40` are one object (`prevPos` 0x20, `pos` 0x30, the frame's
  `move` 0x48, `t` 0x14). The draw callback's `Actor_UpdateTransform` applies the move and sets
  `prevPos = pos`.
- **Domains:** entities slide tile to tile over `moveFrames` (12). `Stg40_ObjStepMove` computes
  `pos = target - (target - prev) * left / frames`, so the position at any fraction of a frame is exact. The
  floor grid is projected in the floor task's update (`Stg40_ProjectGrid`), not its draw.
- **Model animation:** pose tables, one key per 2 VBlanks. After a step, `animPos` always points at the next
  key: the end code rewinds to the last key, the loop code to the loop's first. That gives a clean "next pose".

## Build steps
1. Add a tick clock to the VBlank clock (`Host_Tick`: ticks at the VBlank base times hz/60, the VBlanks due
   run inside). In `Sys_Main`: tick wait, tick flip, game-frame gating, `Logic_EndFrame`.
2. Gate `Task_Run`:
   - logic at game frames, or every tick when converted;
   - the draw at every tick, but only for tasks that drew at the game frame;
   - the counters at game frames.
3. Run the mode switch (the root task returned 0) at a game frame only.
4. For draw callbacks that also run logic, use "kept draws": record their game-frame packets and re-link them
   in the frames between. In DW2 that is `Text_UpdateAllBoxes`, which types dialogue, opens windows, plays
   sounds and waits for input while drawing.
5. Conversions, all draw-time: set the moving state for the tick before the draw pass, draw, restore it all.
   The logic never sees a tick.
   - Domain: entities at their exact tile move, plus the view and floor re-projected.
   - City walkers: a walker in the moving state is drawn one fraction of its last step further on.
   - Battle camera: the last game frame's change, skipping cuts.
   - Model poses: lerp toward the next key.
   - Fade: the level stepped by its own speed.
6. Verify every step with the replay at 60 vs 240. Log one line per game frame (VBlank, mode, random index) and
   diff the two logs.

## Verification
- Replay new game → opening → world map → Boot Domain → `encounter` → battle won.
  - 8918 game frames: mode, `Rand_Index` and cadence identical at 60, 90, 120 and 240.
  - Final state identical: bits, EXP, HP, positions.
  - Re-run after every conversion.
- Tick pictures (dev `DW2_TICK_SHOTS=FROM-TO`, HD at 2x) differ at every tick while the beetle drives, an NPC
  walks, or the battle camera sweeps.
- Pacing, paced headless, scene changes at original speed: VBlank at 59.93 Hz at 240 Hz (59.94 at 60), no
  clock restarts.
- `python tools/build_dw2.py`: SLUS_011.93 and all 7 STAGxxxx.PRO are still OK (byte-identical).
- **Not verified:** the window's real-time feel on a 240 Hz display (only the GPU pacing was measured, at
  120 fps with the cap). Battle fighters' hops, skill effects and menus still move at game frames.

## Gotchas
1. **At 240 Hz the attract movie started 2 frames early.** **Cause:** two gates were missing. The mode switch
   (`slot == 0`) ran at the next loop pass, an in-between tick. And a task set to state 3 was destroyed at the
   next pass 0, also an in-between tick. **Fix:** switch modes, and give unconverted tasks any logic (destroy
   included), at game frames only.
2. **A crash in `Anim_StepModelAnim` from `Stg20_WalkerDraw`.** **Cause:** a walker created during the game
   frame's update finishes its setup in its next update, but an in-between tick drew it first. **Fix:** ticks
   draw only the tasks that drew at the game frame (a pointer + id hash, emptied at each game frame).
3. **Gameplay drifted by one random number in a city cutscene.** Draw callbacks with side effects:
   - the model texture animation re-rolls its blink with `Rand_Next` when its timer is 0, and steps the texture
     animation by 2 when `frameDelta != 1` (0 counts);
   - `Actor_UpdateTransform` sets `prevPos = pos` on every call.
   **Fix:** game-frame-only for those parts. Find them with gdb, `break Rand_Next if Logic_tickGame == 0`.
4. **Text advanced between game frames.** `Text_UpdateAllBoxes` is the text task's *draw* callback but runs the
   dialogue's control codes while drawing: `Task_Create` of window frames, sounds, input waits.
   **Fix:** a kept draw. Found with a memory diff: snapshot the 2 MB PS1 RAM before and after the in-between
   draw pass, report changed ranges outside the packet buffers, then a hardware watchpoint on one. That listed
   every leak in one run.
5. **i686 MinGW names the linker's `__data_start__` `_data_start__` in C** (the leading underscore). x64 doesn't.
6. **Battle speed with XA voices:** DW2 streams XA voice lines in battles, which DW3 doesn't. A 1.25x VBlank
   clock feeds XA sectors too fast, so the battle speed is 1.00 while a stream plays (`Psyq_CdStreaming() == 1`).
7. **Instant scene changes at 240 Hz cost 4x the CPU** (four ticks per VBlank, unseen). With no limit, run a
   VBlank per tick (`--fast` keeps the cadence for replays).

## Assets
None.

## Open questions
- Converting battle fighters (axis-motion hops), skill effects, HUD slides, the domain's event camera
  (`stag4000/camera.c` moveFrames) and its scroll states 1/2, and the beetle's height.
- Pose lerp is linear on rotation matrices. It looks fine between DW2's 15 Hz keys, but a slerp would be safer
  for big swings.
