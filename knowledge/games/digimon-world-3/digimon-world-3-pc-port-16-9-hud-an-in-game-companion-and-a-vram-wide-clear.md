---
kind: game
title: "Digimon World 3 PC port, part 2: a 16:9 HUD at the edges, an in-game companion from the decomp, and a VRAM-wide clear"
game: Digimon World 3
games_also: []
game_version: "PS1 USA, SLUS-01436, from the user's own disc (CHD)"
platform: windows
engine: unknown
route: decomp-recomp
tools: ["ReGame-Labs/dw3_decomp (MIT)", "Wyrelade/DW2-PC PC layer (CC0)", "WinLibs i686 GCC 16.2", "SDL3 3.4.18", "Dear ImGui", "gdb", "ComfyUI (Flux.1-dev)"]
anti_cheat: "none (single-player PS1 game)"
status: working
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-16'
links:
- "https://github.com/ReGame-Labs/dw3_decomp"
- "https://github.com/jeanheck/digivice"
- "https://github.com/rsigristc/dw2003-qol-patch"
tags: [psx, ps1, decompilation, native-port, psyq, widescreen, hud, companion, quests, fast-travel, 240hz, debugging]
---
# Digimon World 3 PC port, part 2: a 16:9 HUD at the edges, an in-game companion from the decomp, and a VRAM-wide clear

> This note follows "A native PC port of Digimon World 3 from its matching decompilation" (same folder). It covers
> a run of user requests on that port. Each one had to stay byte-identical in the 60 Hz replay, which all of them
> do.
>
> The requests:
> - text, portraits and the cursor vanishing for good after an area change;
> - the HUD and menus anchored to the 16:9 screen edges;
> - an in-game "Digivice" companion (party EXP to next level, area enemies, digivolution requirements, a story
>   objective and side quests, with an on-screen tracker);
> - save points that offer travel to other visited save points;
> - saving from the Start menu, L1/R1 to change the leading partner, a Start menu without its fades;
> - the whole map ready instead of streamed;
> - the encounter swirl in HD.

## Setup
- The port described in the first note: the PS1 decomp compiled for Windows, with PsyQ replaced by a PC layer
  that emulates the GPU in a 1x VRAM plus an HD renderer.
- Game edits stay under `#if DW3_NATIVE`. A deterministic 60 Hz replay is the regression (mode timeline plus 51
  screenshots).
- The user played a release build; problems were reproduced from a copy of their memory card.

## Route and why
- The decomp makes everything a C edit. The companion reads the game's own tables (stages, events, evolution
  lists, encounter lists) instead of hand-made data or emulator memory addresses.
- The existing companion app (jeanheck/digivice) supports only the European release, through DuckStation's
  shared memory. Its data covers the story only up to the West Sector, and it has no licence, so nothing of it
  was reused.
- The HD renderer is the place to move HUD pieces: the 1x picture and the game logic stay untouched.

## How the game works (what we had to learn)
- **Story:** `GAME.progress` steps 0..0x29. Each step n+1 is set by exactly one stage (an end-of-event
  `GAME.progress = n` or a talk action `PROGRESS(n), 1`). So "go to the stage that sets n+1" is an exact story
  objective for step n.
- **Side quests:**
  - Every NPC talk whose action list holds `ITEM(kind, id), 1` gives an item; there are 138 in the USA stages.
  - The talk's conditions plus its character's say when the talk is there. A `FLAG(g, i), 1` among its actions
    marks it done. Without one, holding the item does.
  - `FIELD_VISITED_FLAG(mode)` (the game sets it per field mode) tells whether the player has reached that area.
- **Save points:**
  - 25 USA stages run a script containing `FIELD_COMMAND_PLAY_SAVEDEMO` and ending in a leave for mode `0xC01`
    (the save screen).
  - The script's first `SCRIPT_WALK(2, x, y, dir)` is where the player stands at the save NPC: a good arrival
    point for travel.
  - The field leaves with `FIELDSTG_leaveField(mode, -1, x << 8, y << 8, dir)`.
- **Text:**
  - Game strings use glyph indices ('A' = 0x0E). The font's glyph map (`FONT_GLYPH_MAP`, Shift-JIS code to glyph)
    inverts into a decoder, and NFKC turns its full-width Shift-JIS into ASCII.
  - Icon codes `1,1` and `1,19` are a space and an apostrophe.
  - Text files are a count, then byte offsets.
  - Digimon names are in `DIGNAM` (index `DIGIMON_DATA[i].nameId`); enemy names go through the battle table file
    (0x46-byte entries, `nameId` at +6).
- **Evolutions:**
  - `STFGTREP_evolutions[partner][44]`: Digimon (data index + 1), needs (two Digimon at levels), a stat and its
    minimum. Stats 1-6 are battle stats, 7 is the level, 8-14 resistances.
  - A level comes when `exp > (L³ + 5L - 6) * expRate / 10 + levelExp[tier]` (strictly past the mark).
- **Party leader:** `GAME.party[3]` order; the field's follower actors are `FIELD_CHARACTER_PARTNERS + partner`.
  Each streams its frames from its own cached animation file into its own VRAM image. Rotating the order and
  setting each follower's `key1`, `animFile`, `halfWidth`, `animPos = 0`, `animTime = 0` and `reloadImage = 1`
  changes them at once, with no field reload.

## Build steps
- A generator (`tools/pc/digivice_data.py`) parses the stage sources into C tables. It resolves the condition
  macros and the headers' numeric `#define`s, and keeps only stages that have a USA mode. Names of items and
  Digimon are read at run time from the player's disc, so no game text sits in the repository.
- The game side fills a plain-C snapshot about twice a second (game thread). The ImGui UI (window thread) copies it
  under a mutex and draws:
  - the window on L2, in the F1 menu's style;
  - a small story and tracked-quest box while walking;
  - the save point's menu (Save, visited save points, Cancel).
- The save point menu hooks the event runner's leave command (`0x304`) when the target is `0xC01`. The event
  waits (`running = 0`, pc unchanged) until the window answers. Cancel ends the event the way a script end does
  (`TASK_KILL`). In a headless run with no UI, it saves as the PS1 does.

## Verification
- Headless runs with `--ui-shots` write `NAME_ui.png` (the picture with ImGui). These shots come out one command
  late. A dev control command, `event N`, starts a field event.
- **Travel test:**
  1. From the user's save, warp to the second save point's stage, then back.
  2. Talk to the clerk: the menu lists 1 place, and choosing it lands at that stage's save NPC.
  - The number of Cross presses before the menu matters: Down during the clerk's dialogue picks another answer.
- The 60 Hz replay runs with the new options off: it stays byte-identical with 51 of 51 screenshots.

## Gotchas
1. **Text, portraits and cursor gone for good after an area change:**
   - The field's exit shrinks the map layer's clip around the player (an iris). When the player walks off the
    screen's edge, its width or height goes negative.
   - The port's `SetDrawEnv` emitted the layer's background clear as a GP0(02) fill of `(x1 - x0 + 1) & 0x7FF`,
    which wrapped across all of VRAM. It zeroed the rows holding the font's CLUTs.
   - The real libgpu limits the clear's size to 0 and up. The fix: an empty clip clears nothing.
   - Found by reading the user's running game's VRAM with `ReadProcessMemory` (no ASLR on the image), plus a dev
    `DW3_VRAM_WATCH=x0,y0,x1,y1` that prints every write into a rect.
2. **240 Hz between-tick menus crashed:**
   - Interpolated panel levels reached fully open before the 60 Hz frame did. The "fully open" draw then read an
    item that wasn't chosen yet: `getItem(0)` is NULL, and the PS1 reads low RAM there.
   - Fix: between ticks a panel reaches 0 or ONE only if its frame already has, and NULL item lookups read a
    low-RAM stand-in.
3. **The pillarbox held for only 1 of 4 frames at 240 Hz:** it was requested once per VBlank, and the margins
  showed half-drawn backgrounds. It must be requested at every drawn frame.
4. **"Things appear as you come near" in 16:9 was culling, not streaming:**
   - Map objects and stage animations test a view rect 320 wide. Widening `getViewRect` by the margin (field only)
    filled the margins.
   - Loading the whole map up front was done as well, and drew the same frames as streaming: the stream kept up.
5. **Anchoring the HUD to the edges:**
   - Group by drawing call (one sprite-drawer call = one panel with all its parts), inside explicit zones (field
    menu, menu screens, battle HUD and menus).
   - Text windows and small sprites follow the smallest element that *wholly* contains them. Containing by centre
    put a portrait or a panel inside a long title bar.
   - Full-width things stay. Off-screen copies (the fade drawn ±320) must stay too.
   - Packets kept and copied between 60 Hz ticks need their HUD offset carried to the copy, or text stays behind
    on 3 of 4 frames.
   - A view drawn into its own small layer (the battle's partner camera) needs a draw-area rule: commands with
    exactly that clip move, and the clip moves with them.
6. **The encounter swirl showed the original textures:** it MoveImages the 1x screen and spins pieces of that copy.
  Giving the copy's VRAM record the HD screen (read back from the surface, alpha from the 1x texels) as its
  replacement image fixed it.
7. **Heredoc-written Python patch scripts** turned `\n` inside C string literals into real newlines and `\\f` into
  a form feed. Write such scripts with a file tool, or check the C after patching.

## Assets
- The exe icon is generated art (Flux). Nothing from the game is in any commit; the companion reads the user's
  disc at run time.

## Cost and time
- About 10 hours of agent time over one night, the user away for most of it.

## Open questions
- Side quests come only from item-giving talks; multi-step chains (DRI agents, legendary weapons) show as their
  final gift.
- Story objectives name a stage, not what to do there.
