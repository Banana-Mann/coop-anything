# Pitfall: The Lost Expedition (2004)

| | |
|---|---|
| Engine | unknown (Edge of Reality in-house; disc ships `EORPS2IO.IRX`) |
| Versions tested | PS2 NTSC-U `SLUS-20408`, PCSX2 ELF CRC `394F402E`, disc VER 1.00, on PCSX2 v2.8.2 (Windows) |
| Projects | none playable; research and tools in this folder. Directory entry: "Pitfall: The Lost Expedition", kind `gap` |
| QA level | none |
| Last updated | 2026-10-23 |

## Where things stand

There is no way to play the story together yet. This folder has a confirmed memory map for the
PS2 US release (player, position, facing, health, level), a read-only logger that follows two
PCSX2 instances at 20 Hz, and the decompiled player spawn path. A first attempt to spawn a
second Harry built the object but broke the level.

## Existing work

- No co-op or multiplayer mod found (web search for co-op mod, multiplayer, decompilation, two
  player; 2026-10-23). No issues or PRs for the game in this repo.
- Native PC port by Beenox, published by Aspyr in October 2004
  ([Wikipedia](https://en.wikipedia.org/wiki/Pitfall:_The_Lost_Expedition)). Not used here because
  the owner plays the PS2 version; per the playbook it may be the easier co-op target.
- [Avasam/ptle-tools](https://github.com/Avasam/ptle-tools) (LGPL-3.0): GameCube US memory watches,
  an entrance randomizer for Dolphin, and `world_infos.json` with the level CRC32s that the PS2
  build also uses.
- GOG's listing names "split screen" as a mode
  ([GOG](https://www.gog.com/dreamlist/game/pitfall-the-lost-expedition-2004)); no other source
  confirms a two-player mode.

## How the game works (for co-op)

- No netcode. Single player.
- A per-level load state machine (`FUN_0026d250`, state object at `0x0052F190`) allocates and
  constructs the player (`FUN_00173910`, 0x1B30 bytes), places him at the level's spawn point
  and adds him to the world. Spirit-temple levels build a different player class.
- The player pointer `0x0052F198` is a field of that state object, read in 430 places.
- Objects share a base layout: class table pointer at +0, bounding box at +0x3C, 4x4 world
  matrix at +0x160 (drawn position in its last row). The player's matrix is rebuilt every frame
  from position +0x340 and facing.
- Save only stores progress at level entry (in-game warning). Level ids are CRC32s of asset names.

Full memory map, code addresses and experiment logs: [NOTES.md](NOTES.md).

## What we changed

Nothing in the game yet. Tools in [tools/](tools/):

- `ptle_logger.py`: read-only logger (level, position, facing, health) for one or more PCSX2
  instances over PINE; refuses to run on another serial or CRC.
- `pine_*.py`, `pcsx2_keys.py`: PINE client helpers and a no-focus key sender to drive PCSX2
  from scripts.
- `pos_search.py`, `ram_diff.py`, `ram_pattern.py`, `actor_scan.py` and friends: savestate RAM
  diffing used to find the fields.
- `DecompileTargets.java`, `elf_xref.py`, `mips.py`, `vtable_dump.py`: static analysis helpers.
- `spawn_patch.py`: experimental second-Harry spawn stub (breaks the level; see Known issues).

## QA

No co-op to test. The logger was checked against two live instances: 160 ticks in 8.00 s at
20 Hz, each instance's position changing only when that instance moved.

## How to play

Not playable.

## Known issues

- Spawning a second Harry with `spawn_patch.py` creates the object but leaves the screen black
  and the level stuck in a reload loop. Restore a savestate taken before applying it.
- Addresses are for NTSC-U `SLUS-20408` / `394F402E` only.

## Next steps

1. Debug the spawn: compare global state before and after the second constructor, try spawning
   after the level has fully loaded, and load a PCSX2 savestate into Ghidra
   (`PCSX2SaveStateImporter.java`) to read heap objects.
2. Stop the second Harry reading pad 1 (find the pad buffer) and drive him from a mailbox the
   bridge fills over PINE.
3. Check the player pointer in other levels; find the animation id.
4. Consider the 2004 PC port as an alternative target.
