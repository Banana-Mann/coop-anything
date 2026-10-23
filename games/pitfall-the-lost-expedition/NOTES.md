# Pitfall: The Lost Expedition (PS2): working notes

All work 2026-10-23. Route: build from zero, memory-sync over PCSX2 PINE
([console-emulators.md](../../playbook/console-emulators.md) section 5, Route B).

## Identity

| Field | Value |
|---|---|
| Release | PS2 NTSC-U, serial `SLUS-20408` (PCSX2 GameIndex: "Pitfall - The Lost Expedition", compat 4, `mtvu: 0`) |
| PCSX2 ELF CRC | `394F402E` (matches `patches/SLUS-20408_394F402E.pnach` in PCSX2/pcsx2_patches) |
| ISO SHA-256 | `ED633886E5CF16FE93D6824CCE303F97D4436A5E1FF83BF1E6C219277E011878` (1.25 GB, owner's own dump) |
| Main exe | `SLUS_204.08`, 4,404,656 bytes, `SYSTEM.CNF` `VER=1.00`, `VMODE=NTSC` |
| Main exe SHA-256 | `AF20E4402B34E642A9B325049A9FAF5B56DE8BE51B70846B45B0E1FCEE5280A8` |
| Engine | unknown; developer Edge of Reality, disc ships `EORPS2IO.IRX` |
| Emulator | PCSX2 v2.8.2, Windows 11, portable installs |

ISO download sites list CRC `9CBE62BE` for the US ISO; derived guess: that is a whole-image
CRC32, not PCSX2's ELF CRC. PAL is `SLES-51686` (search result, not checked); addresses below
are NTSC-U only.

## Test setup

- Two portable PCSX2 v2.8.2 installs from `pcsx2-v2.8.2-windows-x64-Qt.7z` (SHA-256
  `7dfc829c...a200b4`, matches the GitHub asset digest), each with `portable.txt`.
- `inis/PCSX2.ini` edits (after letting PCSX2 generate the file; a hand-written minimal ini is
  discarded on first launch): `[EmuCore] EnablePINE = true`, `PINESlot = 28011` / `28012`,
  `SavestateCompressionType = 0`, `[Filenames] BIOS = <owner's BIOS file>`.
- Controller automatic mapping replaces keyboard bindings; duplicate keys in `[Pad1]` (for
  example `LLeft = SDL-0/-LeftX` and `LLeft = Keyboard/A`) keep both.
- Scripts drive the game by posting key messages to one PCSX2 process (`tools/pcsx2_keys.py`,
  hold 400 ms; 120 ms was missed on menus) and check the screen with the F8 screenshot hotkey.

## Observations (things that ran)

### PINE and savestates

- PINE answers version, status, title, serial, CRC, game version and Read32 (`tools/pine_check.py`).
  64 sampled reads in 0x00100000-0x004F0000: 58 non-zero, values look like MIPS code.
- Save/load state opcodes 0x09 / 0x0A with a one-byte slot work (`tools/pine_state.py`).
- `.p2s` savestates are zip files; with compression 0, `eeMemory.bin` (32 MiB) is readable by
  Python `zipfile`. Default Zstandard (zip method 93) is not.

### Boot path

Blank card: "MEMORY CARD slot 1 is unformatted" -> Cross -> intro -> "Press START" -> book
menu (NEW GAME / LOAD GAME / OPTIONS / CREDITS) -> NEW GAME -> prologue: jaguar boss fight in
a burning temple. The prologue was played past by the owner (attack while moving, hit from the
side). Pause menu: MAP / NOTES / OPTIONS / SAVE / QUIT. SAVE warns that only progress at level
entry is saved, then offers to format an unformatted card. QUIT goes to the intro, title, then
attract mode.

### Memory map (NTSC-U SLUS-20408)

| What | Where | How confirmed |
|---|---|---|
| Player object pointer | u32 `0x0052F198` (copy `0x0052F218`) | Points at the player object in the prologue (0x00EB5BA0), in Crash Site (0x00EB5300), and in Crash Site after a cold boot + Load Game (0x00EB5300) |
| Position x, y, z | floats at player + `0x340` / `0x344` / `0x348` | Idle-stable, ~9 units per 0.6 s walk step (`tools/pos_search.py`); writing x moves Harry on screen and shifts 9 render copies (`tools/teleport_test.py`) |
| Facing (read only) | float radians at player + `0x35C` (copy `0x360`) | Equals atan2(+0x164, +0x160) of the drawn rotation matrix + pi in 4 states; the placement function writes its angle argument here |
| Current health | u32 at player + `0x688` | 5 with 5 pips; writing 3 shows 3 full + 2 empty pips |
| Current level | u32 `0x0052F200`, level asset CRC32 as in ptle-tools `world_infos.json` | Prologue = Ruins of El Dorado (Supai/Jaguar) `A02`; jungle = Crash Site `A03` (`tools/level_search.py`) |
| Prologue boss health | u32 `0x014FD194` (this boot; heap) | 10 -> 4 while attacking (`tools/hp_search.py`); writing 8 shows 8 pips |

Axes: ground plane X-Y, Z up. Walking forward from the Crash Site spawn decreases Y. Spawn
point (-1.587, 747.350, -11.028). Position reads 0,0,0 during the level-start cutscene.

Writing the boss health to 0 empties the bar but does not end the fight. Writing the facing
field does not rotate Harry; the matrix at +0x160 is rebuilt every frame from another source.

### Live logger

`tools/ptle_logger.py` (read only, checks serial and CRC first):

- One instance: 120 samples in 6.00 s at 20 Hz while walking left 1 s, right 1 s, forward 1 s.
- Two instances (`--port 28011 --port 28012`): 160 ticks in 8.00 s at 20 Hz. A walked left then
  back while B walked right then forward; each position changed only in its own instance; the
  distance between players went 22 -> 38 -> 28.

### Two instances on one PC

Both portable installs ran at once on PINE 28011 and 28012 and opened the same ISO copy. Level
loads were slower (about 35 s from Load Game to playable on the second instance).

### Other objects in Crash Site

- Base layout shared with the player: +0x000 class table pointer into the ELF (player class
  0x004AB738), +0x03C / +0x048 bounding box min / max, +0x160 4x4 world matrix, last row
  (+0x190) = drawn position.
- `tools/actor_scan.py`: 31 objects with that layout, 13 classes; nearest 8 are class 0x00491CF8.
- Their +0x190 is not rebuilt every frame, so writes stick (`tools/move_test.py`), but moving one
  object of each of the 11 other classes next to or onto Harry showed nothing new in 22
  screenshots. They are invisible here or drawn from other data.

### Code (Ghidra 12.1.4 headless, `r5900:LE:32:default`, `tools/DecompileTargets.java`)

Function names are Ghidra defaults.

- `FUN_00173910`: player constructor. Stores class table 0x004AB738 at +0, +0x004 = 0x780BF795.
- `FUN_00180c10`: player destructor.
- `FUN_0026d250`: per-level load state machine. Its state object is at `0x0052F190`: +0x00
  world, +0x08 player (= `0x0052F198`), +0x0C, +0x64 state, +0x70 level CRC (= `0x0052F200`),
  +0x88 player copy (= `0x0052F218`), +0x8C spawn data. Ghidra finds 430 reads of `0x0052F198`
  and no direct writes; it is written as a field of this object.
  - State 2: `FUN_002f9e90(size, 0x10)` allocates and the player class is picked by level CRC:
    Penguin Temple (Spirit) `0x1F237F32` -> `FUN_001f8928` (0xC50), Scorpion Temple (Spirit)
    `0x0305DC42` -> `FUN_001f9338` (0xE40), Monkey Temple (Spirit) `0x02C7B675` ->
    `FUN_001f6f68` (0xC80), any other level -> Harry `FUN_00173910` (0x1B30). Then class +0x294.
  - State 4: allocates 0x110 and calls `FUN_002697a0(obj, player, *world)`.
  - State 5: at 0x0026D4F4, `jal FUN_001be9b0` with a0 = world, a1 = player, a2 = [level+0x0C].
- Class +0x294 = `FUN_001f29b8`: stores its argument at player +0x7A0.
- Class +0x2A4 = `FUN_00173e38` (placement): a0 = player, a1 = pointer to xyz, f12 = radians.
  Resets state, writes position to +0x340 (and +0x920, +0x720, +0x69C) and the angle to +0x35C.
- `FUN_001be9b0`: world+4 = spawn data (xyz at +0x50, degrees at +0x6C, times 0.0174533);
  calls placement, then `FUN_002d3cd8(world[0], player, 0, 0)`, then level setup.
- EE RAM 0x00095064-0x000FFFFF was zero in every savestate taken (`tools/cave_check.py`).

### Spawn experiment: failed

`tools/spawn_patch.py` assembles a stub at 0x000F0000 (mailbox 0x000EFF00) and replaces the
`jal 0x001BE9B0` at 0x0026D4F4 with `jal 0x000F0000`. The stub makes the original call, then
once per arm: alloc 0x1B30 -> constructor -> class +0x294 with [level+0x8C] -> placement at
spawn + (2,0,0) -> `FUN_002d3cd8(world[0], obj, 0, 0)`.

Applied over PINE, then QUIT -> title -> LOAD GAME -> Crash Site. The stub ran (mailbox cleared,
new object at 0x015E37C0 with class 0x004AB738). Its position read (0.617, 0.241, -43987.9).
After the level-start cutscene the screen stayed black and the loader cycled states 3 -> 5 -> 0
every few seconds, placing the real player at about (0, 0, -7.6) each time. Loading a savestate
taken before the patch restored a normal game. The generator can write a `.pnach`
(`spawn_patch.py pnach`), but it was not tested as a pnach and it breaks the level.

## Derived (not proven)

- Player + `0x684` (5) is probably max health: writing 3 there clamped +0x688 to 3, but the
  HUD slot count did not change within 1 s.
- GameCube offsets from ptle-tools do not carry over with a fixed shift (position GC +0x338 vs
  PS2 +0x340); check each field.
- Spawn failure, two guesses: (a) the constructor or scene add registers the new object in
  global singletons (camera target, checkpoint, listener) that the real player relies on;
  (b) the second Harry's placement did not hold, he fell out of the world, and a fall handler
  restarts the level with broken data.

## Unknown

- Whether `0x0052F198` holds in levels other than the prologue and Crash Site.
- Pad input buffer address, writable facing source, animation id.
- Meaning of `0x0052F204` (prologue: Plane Crash cutscene CRC; Crash Site: Jungle Canyon CRC).
- Engine name.

## Leads

- Avasam/ptle-tools (LGPL-3.0), GameCube US Dolphin memory watches and `world_infos.json`
  (level CRCs): https://github.com/Avasam/ptle-tools
- CodeBreaker codes for NTSC-U, listed encrypted (CB7, master code `B4336FA9 4DFEFB79`):
  https://almarsguides.com/retro/walkthroughs/ps2/games/pitfallthelostexpedition/codebreaker/
- ghidra-emotionengine-reloaded ships `PCSX2SaveStateImporter.java`, which can load live RAM
  (heap objects) into Ghidra.
