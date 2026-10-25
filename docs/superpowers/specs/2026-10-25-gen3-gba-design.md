# Gen III (Game Boy Advance) support — design

Status: draft for review (2026-10-25)

## Intent

The Pokemon Trade Tool already lets a Flipper Zero trade with Gen I/II games
over a Game Link Cable: you can trade a Pokemon to the Flipper, change it, and
trade it back, or build one on the Flipper and send it over. This design
extends the same idea to the Game Boy Advance games: Ruby, Sapphire, Emerald,
FireRed and LeafGreen.

The motivating moment: the user was playing **Pokemon Ruby** on a real GBA and
wanted to turn their party into all starters. They own a GBA, a GBA (Gen 3)
link cable, and the GPIO link hardware. Inspiration comes from
[PKHeX](https://github.com/kwsch/pkhex) (how Gen III Pokemon data is modelled
and edited) and Bulbapedia's
[Generation III](https://bulbapedia.bulbagarden.net/wiki/Generation_III) pages
(data structures).

## What it feels like to use

The work ships in two phases, in this order (the user's decision).

### Phase 1: Cable Club trade

1. On the Flipper's main menu, pick **Gen III**. Build a Pokemon or keep the
   default, the same way Gen I/II work today.
2. Plug the GBA into the Flipper with the GBA link cable and open **Trade** on
   the Flipper.
3. In Ruby, go upstairs in any Pokemon Center to the Cable Club and choose the
   Trade Center. The Flipper shows up as the other player.
4. Trade. The Pokemon the GBA sends stays on the Flipper. Back out, edit it
   (change its species to Treecko, say), go back to Trade and send it back.

For the user's goal, they repeat that once per party member. Trades are one
Pokemon at a time, as in the real game. Ruby needs at least two Pokemon in the
party to trade.

### Phase 2: direct save access

1. Power on the GBA with the Ruby cartridge in while holding **Start + Select**.
   The GBA skips the game and waits for a program over the cable.
2. On the Flipper, pick **Gen III → Cartridge**. The Flipper sends the GBA a
   small helper program, which reports the cartridge (game, save type) and
   lets the Flipper read and write the save.
3. The Flipper backs up the whole save to the SD card first. Then you can browse
   the party and all PC boxes, edit any Pokemon, put a Pokemon in any empty
   slot, and write the save back. The Flipper verifies what it wrote.
4. Also available: restore a save from an SD backup, and copy the cartridge
   ROM to the SD card.

Both phases use the same Pokemon editor screens.

## Goals

**Shared core (built first, used by both phases)**

- Read and write the Gen III Pokemon format: 80 bytes in the PC, 100 bytes in
  the party. This covers the PID/OT-ID XOR encryption, the 24 orderings of the
  four data blocks (picked by PID mod 24), and the checksum.
- Gen III game text (Western character set) for nicknames and trainer names.
- Data for all 386 species: the game's internal species numbers and how they
  map to National Dex numbers (they differ after #251), base stats, types,
  gender ratio, the one or two possible abilities, growth rate, and a sensible
  default moveset. Also the full Gen III move, item, ability and nature lists.
- Stats calculated from level, base stats, IVs, EVs and nature, matching the
  game.

**Editing on the Flipper.** At least the Gen II options carried forward, plus
the Gen III-only ones:

- species, nickname, level
- held item, moves
- IVs and EVs (the same presets as Gen II)
- shiny, gender, Pokerus
- original trainer name and ID, including the hidden "secret ID"
- nature, ability, Poke Ball (Gen III only)

**Phase 1**

- Cable Club trade with **Ruby**; Sapphire comes with it, since it's the same
  game code. This is the must-have.
- Trade back and forth repeatedly in one session, like Gen I/II today.
- Learn the GBA player's name and full trainer ID during the link. Default a
  Pokemon's original trainer to the player, so Pokemon made on the Flipper
  count as the player's own. That means they obey and don't get the "traded"
  experience boost. The user can still change it.
- Save a Pokemon to the SD card and load it back, in PKHeX's `.pk3` format, so
  Pokemon can move between this app and PKHeX in both directions. This is my
  call and can follow just after the trade works.
- Emerald, FireRed and LeafGreen trades after Ruby works. They're wanted, but
  not required to ship Phase 1.

**Phase 2**

- Send the helper program to a GBA over the link cable (GBA "multiboot").
- Read and write the cartridge save safely: back up first, write, verify.
- Edit the party and all 14 PC boxes in place. Put Pokemon in empty slots.
- Restore the save from an SD backup, and copy the cartridge ROM to the SD card.

**Always**

- Gen I/II trading keeps working exactly as before for existing users.

## Non-goals

- Japanese Gen III games. Like the existing app, Western releases only.
- Wireless Adapter, Union Room, battles, Record Mixing, contests, Berry
  Blending, and the GameCube games.
- Full legality checking like PKHeX's (where it was met, whether it could exist
  in the wild). Pokemon only need to be valid as far as the game is concerned
  (see Anti-goals).
- Creating eggs.
- Editing the bag, money or other trainer data. Phase 1 can't reach them at
  all; Phase 2 could later, but not in this design.
- Modifying ROMs (patches, ROM hacks). Copying a ROM to the SD card is in;
  changing it is out (my call).
- New sprite art for species #252–386. They get a placeholder image for now
  (my call).

## Anti-goals: failures even if it "works"

- **Damaging the player's save.** Every Pokemon the Flipper sends or writes must
  load cleanly in-game: no "Bad Egg", no checksum failures, no garbled data.
  The end of a trade must not leave the game stuck mid-save.
- **Writing to a cartridge without a verified backup on the SD card** (Phase 2).
- **Breaking Gen I/II** for the people already using the app.

## Constraints

**Hardware**

- Flipper Zero, an STM32WB55 at 64 MHz running FreeRTOS. App memory is tight:
  the default app stack is 2 KB, and a whole 128 KB save does not fit in RAM,
  so Phase 2 works through the save in 4 KB pieces.
- Link hardware goes through the existing `flipper-gblink` pin setup. Its
  presets already name a fourth pin, SD, which the GBA link needs and Gen I/II
  never used:

  | Preset | SI | SO | CLK | SD |
  |---|---|---|---|---|
  | Original (Flipper GB Link board, MALVEKE 2.5.1+) | PC3 | PB3 | PB2 | PA4 |
  | MALVEKE EXT1 (MALVEKE 2.5 and older) | PA6 | PA7 | PB3 | PA4 |

  The app's Pinout screen only shows SI, SO and CLK, and the saved Custom
  pinout doesn't store SD. Both need SD added.
- `flipper-gblink` only does byte-wide Game Boy Color transfers, so the GBA link
  needs new link code. It lives in this repo (my call). It could be offered to
  `flipper-gblink` later.

**How the games link** (from the pret/pokeruby and pokeemerald decompilations)

- Gen III link uses the GBA's "multiplayer" serial mode at 115200 baud (about
  8.7 µs per bit). Each transfer is a 16-bit frame with start and stop bits on
  the shared SD line. The parent paces transfers about every 0.75 ms (timer 3,
  reload 65339, 64-cycle prescaler).
- Game-level messages are 8 words plus a checksum word: handshake, player info
  exchange, held keys (`0x4444`), block transfers (`0xBBBB` / `0x8888`), and
  so on.
- In the Trade Center both players have avatars in the room, so the Flipper
  has to act as a player who is seated at the table, not just a data source.
- Which GBA is the parent depends partly on which end of the cable goes in it
  (one plug grounds SI). The design has to pick a role for the Flipper, and the
  README must say which plug goes where.
- Sources of truth: [pret/pokeruby](https://github.com/pret/pokeruby) for Ruby;
  [pret/pokeemerald](https://github.com/pret/pokeemerald) and
  [pret/pokefirered](https://github.com/pret/pokefirered) for the others; and
  [GBATEK](https://problemkaputt.de/gbatek.htm) for serial timing and the BIOS
  multiboot protocol.

**Project**

- C on the Flipper SDK, built with `ufbt`. CI builds against both the dev and
  release SDK channels, and both must keep passing.
- The repo is MIT licensed. PKHeX is GPLv3, so use it as a reference for
  behaviour and formats only; don't copy its code. Species, move and item
  facts can be generated from the pret data files or another permissive source.
- Only the user has the hardware. The shared core must be testable on a normal
  computer (macOS clang) without a Flipper: encryption round-trips, checksums,
  stat calculation, and `.pk3` load/save against known-good test data. Hardware
  steps are handed to the user to run, with a clear checklist.
- Commits are authored by the user (LawlessNPC, Aleksandr Kofman) with no
  AI-attribution trailers.

## Decisions made by the user

- Real hardware: a GBA, a GBA link cable and the GPIO link board.
- Both phases; Cable Club trade first, direct save access second.
- Ruby is the main test game.

## My calls (flagged for the user to check)

1. **Connection test before the full trade.** The first hardware milestone is
   only that Ruby's Cable Club sees the Flipper as a linked player: the
   handshake plus the player-info exchange. This is the riskiest part, because
   the Flipper has to keep the serial timing. If it can't, stop and report;
   the likely fallback is to do Phase 2 first.
2. **Placing a Pokemon on the Flipper picks a PID that matches what you
   chose.** Nature, gender, shiny status and ability all come from the PID in
   Gen III, so changing any of them generates a new consistent PID, the way
   PKHeX does.
3. **Changing species resets the other options to that species' defaults**,
   matching today's Gen I/II behaviour, but keeps the original trainer.
4. **Pokemon files on the SD card use PKHeX's formats.** `.pk3` is the
   unencrypted 80-byte form; also accept the 100-byte party form when loading.
5. **The helper program's source lives in the repo**, built with a GBA
   toolchain (devkitARM or arm-none-eabi). The built binary is committed into
   `files/` so building the Flipper app never needs the GBA toolchain.
6. **Each phase ships as its own pull request and release.**

## Left to the builder

- How the link driver meets the timing: timer capture/compare, DMA, interrupt
  priorities. Also which role (parent or child) the Flipper takes, with the
  reasoning written down.
- File layout, how the data tables are generated, the test setup.
- Screen layouts, within the app's existing look and scene structure.
- Which Gen III game version the Flipper claims to be during a trade, as long
  as Ruby accepts the trade and the Pokemon.
- How Emerald and FireRed/LeafGreen differences (National Dex gating, version
  checks) are handled once Ruby works.

## Risks to check early

- Whether the Flipper can keep 115200-baud multiplayer timing reliably under
  FreeRTOS. Biggest risk; addressed by call #1.
- Whether the user's board actually wires SD to PA4. The Original preset says
  so; confirm on the hardware.
- The link save handshake at the end of a trade has to complete correctly. If
  it can't, abort before the game starts saving.
- Ruby's handling of non-Hoenn species received in a trade. Sources say it
  unlocks the National Dex rather than blocking it; confirm in pokeruby.
