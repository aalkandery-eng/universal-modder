---
kind: game
title: 'Digimon World 2 PC port (DW2-PC): reading a .chd, and rips with the movies taken out'
game: Digimon World 2
games_also: []
game_version: "PS1 USA, SLUS-01193 (SLUS_011.93 SHA-1 e55ed5bf354def07f0cbf4e1fb7fb5f99204f220); DW2-PC v0.2.1 (64feb636)"
platform: windows
engine: unknown
route: decomp-recomp
tools: ["Wyrelade/DW2-PC v0.2.1 (CC0)", "libchdr 0.3.0", "WinLibs i686 GCC 16.2 (CMake + Ninja bundled)", "SDL3 3.4.18", "chdman 0.289"]
anti_cheat: "none (single-player PS1 game)"
status: working
agents:
- Claude Code (Opus 5.5)
humans: []
date: '2026-10-15'
links:
- "https://github.com/Wyrelade/DW2-PC"
- "https://github.com/rtissera/libchdr"
tags: [psx, ps1, decompilation, native-port, chd, libchdr, disc-image, rip, fmv-stripped, xa-audio]
---
# Digimon World 2 PC port (DW2-PC): reading a .chd, and rips with the movies taken out

> DW2-PC (the native PC port built from the matching Digimon World 2 decompilation) builds its data pack
> `dw2.pak` from the player's disc image on first start, but only from raw `.bin/.cue`, `.img` or `.iso`.
> We added `.chd` input through libchdr. The user's CHD was a space-saving rip (movies blanked, XA voice
> audio silenced), so we also taught the pack build to accept exactly those two kinds of difference and to
> skip the missing movies. A pack built from the CHD is byte-identical to one built from the `.bin` that
> chdman extracts from it, and the game boots to the title in a window with the intro skipped.

## Setup
- **Starting point:** DW2-PC v0.2.1, the commit the official release is built from. Building it needs the
  disc's exe and 7 overlays in `dumps/disc/` plus `assets/Ovl_LoadArea.bin`. The README says to use
  `dumpsxiso -pt`, which has no Windows build in the repo.
  - A 70-line Python script does the same job. It takes `SLUS_011.93` from the ISO root and finds the
    `AAA/3.PRO/STAG*.PRO` overlays through the **ISO 9660 path table**: `AAA/` is not reachable from the
    root directory records.
  - It checks each file against the retail SHA-1s listed in `tools/dw2pack.py`.
  - `Ovl_LoadArea.bin` is splat's bin segment of the exe, file offsets `0x53B60..0x9F000` (the exe is
    exactly `0x9F000` bytes).
- **Toolchain:** WinLibs i686 MinGW (it ships cmake.exe and ninja.exe in `bin/`) and the SDL3 MinGW
  development release.
  - `python tools/build_native.py --release` with `DW2_MINGW` and `DW2_SDL3` set, and the MinGW `bin` on
    PATH (the script finds ninja with `shutil.which`).
  - A full build takes about 20 s.
  - The official release is x64. The x86 build works the same, but it needs the **x86 `SDL3.dll`** next to it.
  - Strip debug info for the player copy: 12.7 MB → 3.4 MB.

## Route and why
`decomp-recomp`: DW2-PC already is a native build of the matching decompilation. Only its host side (pack
builder, pack loader, the libcd STR shim) needed changes. There was a simpler alternative:
`chdman extractcd` once, then let the stock exe build the pack (after that the image is never read again). We
didn't take it because the user asked for the game itself to take their `.chd`. And for this user's rip, the
stock pack check rejects the image anyway.

## How the game works (what we had to learn)
- **The disc is read once.** `host/pakbuild.c` reads every one of the game's 0xE5B files (found through the
  file table in `SLUS_011.93`, not the ISO tree), checks each SHA-256 against
  `configs/USA/pack_manifest.txt` and writes `dw2.pak`. At run time libcd reads sectors from the pack, so
  CHD support is only needed in the builder.
- **CHD layout (CD):**
  - hunks of 8 frames;
  - each frame is 2448 bytes: 2352 of sector and 96 of subcode;
  - track 1 metadata `CHT2`: `TRACK:1 TYPE:MODE2_RAW ... FRAMES:196426 PREGAP:0`.
  - Sector `n` of track 1 is frame `n` when no pregap is stored (PGTYPE starting with `V` means it is).
  - Decompress one hunk and keep it until a sector outside it is asked for: the pack build reads the disc
    in order, so each hunk is decompressed once.
- **Movies:** six STR files, `Stg10_MovieFileIds = {0x282, 0xE2F, 0x510, 0x511, 0xE28, 0xD4E}` in
  `src/stag1000/stag1000.c`. Movie playback goes through `CdRead2` / `StGetNext` in `psyq/libcd.c`.
  - Headless runs have a stub: `StGetNext` returns one frame whose frame number is past the end, so the
    movie task ends on its first frame.
  - That stub is the clean way to skip a missing movie in any run.
- **XA voice audio** (skill shots, DigiLab, battle results; e.g. file 0x10C from `skilleffect.c`,
  0x512 from `digilab.c`) lives in 28 Form 2 files whose sectors are all XA audio (submode bit `0x04`).

## Build steps
1. `chdman extractcd -i "<game>.chd" -o x.cue -ob x.bin` (only for the build's data tables), then run
   the dump script → `dumps/disc/`, `assets/Ovl_LoadArea.bin` (all gitignored).
2. Vendor libchdr as one unit (`unity.c`: LZMA SDK, zstd decoder, libchdr sources). DW2-PC already links
   **miniz** for its self-updater. Drop libchdr's bundled miniz.c and point `src/codec_zlib.h` at the
   host's `miniz/miniz.h`, or the link fails on duplicate `mz_*` symbols.
3. In the pack builder's `Disc`:
   - open the `.chd` with `chd_open_core_file_callbacks` over an `SDL_IOStream` (UTF-8 paths on Windows,
     unlike libchdr's `fopen`), with a no-op close callback so only one side owns the stream;
   - read sectors from hunks;
   - add `.chd` to the folder search and the file-dialog filter.
4. Rip acceptance (builder, Python reference `tools/dw2pack.py`, and loader):
   - **Builder:** a file that differs from the manifest is kept only if it is a movie with all-zero
     sectors (the sync check lets those through only for movie files) or a file whose XA audio sectors
     all have an all-zero payload (2324 bytes after the subheader).
   - **Loader:** accepts a difference only in a movie or a format 1 (Form 2) file.
   - **Playback:** `CdRead2` doesn't stream a movie the loader flagged, so the stub ends it.

## Verification
- The same exe built packs from the `.chd` and from the chdman `.bin`: **byte-identical** (406 MB).
  `tools/dw2pack.py` on the `.bin` wrote the same bytes.
- Headless runs from both packs gave identical title-screen PNGs at VBlank waits 600 and 899.
- `--movies` headless and a real window (SDL_GPU Vulkan) log `[movie] file 0x282: not on this disc image
  (taken out), skipped` and reach the title screen.
- First-start flow: with the CHD next to the exe and no `--disc`, it was found and built. A romhack CHD in
  the same folder was skipped by the exe hash check, as it should be.
- `gte_test` passes.
- **Not verified:** a long play session, and that silenced XA plays as silence in battle and DigiLab
  without side effects. The sector headers and EOF flags are intact, so the stream logic sees what it sees
  on retail.

## Gotchas
1. **`sector 31318: bad sync / header` from a CHD that chdman reads fine.** **Cause:** not libchdr. The
   rip itself has all-zero sectors (the `.bin` that chdman extracts has the same zeros). The CHD was
   35 MB for a 462 MB disc, which is the giveaway. **Fix:** see what the files are first (the zero runs
   all fell inside the six movie files, each keeping its first 404 sectors). Then accept only that
   pattern.
2. **After the movies were accepted, 28 more files mismatched.** **Cause:** the same rip zeroed the
   payload of every XA audio sector but kept the sync, headers and subheaders (`01 00 64 01`), so the
   sector check passes and only the hash differs. **Fix:** accept a file only when *all* of its audio
   sectors are silent; one retail sector of sound fails it.
3. **Duplicate `mz_*` symbols** when adding libchdr to a program that already has miniz. **Fix:** compile
   libchdr against the existing miniz (its zlib-compatible names map `inflate*` to `mz_inflate*`; the
   host's copy is compiled with `MINIZ_NO_ZLIB_COMPATIBLE_NAMES`, which only changes macros, not
   symbols).
4. **libchdr only closes the file on some failure paths of `chd_open_core_file_callbacks`.** **Fix:** make
   the close callback a no-op and always close the stream yourself.
5. **The self-updater.** DW2-PC can replace its own exe with the official release. The official exe
   rejects a pack built from a rip (manifest mismatch) and can't read a `.chd`. **Fix:** tell the player
   that taking an official update undoes this. Keep the original release next to it (`original-v0.2.1/`).

## Assets
None. Nothing from the disc is in the repo: `dumps/`, `assets/` and `build/` are gitignored.

## Open questions
- Upstream: CHD support is self-contained (libchdr + about 120 lines in `pakbuild.c`) and could go to
  Wyrelade/DW2-PC as a PR. Accepting rips is a policy call for the maintainer.
- A rip's movies could be filled in from another source; out of scope here.
