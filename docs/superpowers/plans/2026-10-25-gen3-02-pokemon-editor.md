# Gen III Pokemon Data Core and Editor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The Flipper can build, edit, save and load a Gen III Pokemon that any Gen III game would load
cleanly. Gen III → Edit Pokemon offers every option in the spec's "Editing on the Flipper" list.
Load .pk3 and Save .pk3 move Pokemon to and from PKHeX's file format on the SD card. No link and no
trade yet: plan 3 sends this editor's Pokemon to Ruby.

**Architecture:** A pure-C core in `src/gen3/` is unit-tested on the host. It holds the 100-byte
Pokemon in PKHeX's decrypted `.pk3` byte layout, with field accessors, checksum, the 24 substructure
orders and encryption. On top of that come the game rules (exp, stats, natures, PID-derived traits,
PID search, default moveset) and an editor model that keeps the Pokemon game-valid after every change.
Species, move, item, ability, nature and type facts are generated once from pret/pokeemerald by a
committed Python script into a committed C file. Thin Gen III scenes and one new view drive the
editor through the app's existing scene manager and shared GUI modules.

**Tech Stack:** C11, Flipper SDK via `ufbt` (release SDK 1.5.1, API 88.2 per plan 1), Python 3
standard library (table generator only), `make` + host `cc` for unit tests, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-10-25-gen3-gba-design.md`. This plan covers "Shared core",
"Editing on the Flipper", the Phase 1 `.pk3` and OT bullets, and calls #2–#4.

## Plan sequence

This is plan 2 of 5.

1. Link connection test (done).
2. **This plan:** Gen III data core, editor and `.pk3` files.
3. Cable Club trade with Ruby, reusing this editor for traded Pokemon.
4. FireRed/LeafGreen trades.
5. Direct save access (multiboot helper, cartridge save read/write, ROM dump).

## Global Constraints

- C on the Flipper SDK, built with `ufbt`. CI builds against both the dev and release SDK channels, and both must keep passing with 0 warnings. Locally: `ufbt` (release) and `UFBT_HOME=/Users/kyvo/.claude/jobs/72354b36/tmp/ufbt-dev ufbt` (dev).
- MIT licensed. PKHeX (GPLv3) and the pret decompilations are references for behaviour, constants and layouts only; copy no code from them. Game facts (ids, base stats, names) come out of the clone only through `tools/gen3_tables.py`.
- The shared core must be testable on a normal computer (macOS clang; Linux gcc in CI) without a Flipper. `make -C tests` builds every `tests/test_*.c` against all non-`_furi` `src/gen3/*.c`.
- Gen I/II keeps working exactly as before. This plan changes no Gen I/II file: `src/pokemon_data.c`, `src/pokemon_data_i.h`, `src/pokemon_table.c`, `src/pokemon_attribute.c`, `src/pokemon_char_encode.c`, `src/named_list.c`, `src/*_nl.c`, `src/views/select_pokemon.c`, `src/views/trade.c`, and the Gen I/II scenes. Shared registration points change by appending only: `src/scenes/include/pokemon_scene_config.h` (new lines at the end, so existing scene ids keep their values), `src/include/pokemon_app.h`, `src/pokemon_app.c`.
- Western (non-Japanese) games only. Never modify `lib/flipper-gblink`.
- Commits are authored by the user (LawlessNPC, Aleksandr Kofman) with **no** AI-attribution trailers: no `Co-Authored-By`, no `Claude-Session`, no "Generated with". Check with `git log -1 --format='%(trailers)'`, which must print nothing.
- Stage only the files your task lists, by explicit path. Never `git add -A`, `git add .`, a directory, or `git commit -a`.
- Pull requests go to the user's fork, `LawlessNPC/Flipper-Zero-Game-Boy-Pokemon-Trading`. No `fap_version` bump in this plan; v3.0 is the finished Phase 1.
- **Game-valid invariant:** every Pokemon the editor holds, saves or (in plan 3) sends must load cleanly in-game: checksum valid, has-species flag set, not an egg, party stats and level consistent with exp, moves packed with no duplicates, ability bit 0 for one-ability species.
- **Pure-module rule:** every file in `src/gen3/` whose name does not end in `_furi.c`/`_furi.h` includes only `<stdint.h> <stdbool.h> <stddef.h> <string.h> <stdio.h>` and other pure `src/gen3/` headers. Flipper APIs live only in `*_furi.*` files and in `src/scenes/`, `src/views/`.
- **ISR rule** (carried over): everything called from the link frame callback runs in interrupt context, with no `malloc`/`free`, no `FURI_LOG_*`, no blocking calls, and returns in under 200 µs. Nothing in this plan runs there.
- **Budgets:** the app stack is 2 KB, so no scene or view keeps a local buffer over 128 bytes. Tables are `const` (`.rodata`). A FAP is loaded into RAM whole, so `.rodata` costs RAM: `gen3_data_gen.c` ≤ 33,500 bytes of `.rodata`, and `text + data` of `dist/debug/pokemon_d.elf` ≤ 112,000 bytes when the plan is done (baseline 56,348 at a09d0c7). Measure with `$(ls -d ~/.ufbt/toolchain/*/bin)/arm-none-eabi-size dist/debug/pokemon_d.elf`.
- `src/gen3/gen3_data_gen.c` is written only by `python3 -I tools/gen3_tables.py <pokeemerald>` and never edited by hand. The pinned clone is at `/Users/kyvo/.claude/jobs/72354b36/tmp/pokeemerald` (commit `731ad5bfd6e6f265508d0efcca0ba42f9dcf5881`). It is read-only data: don't build or run anything from it. Elsewhere: `git clone https://github.com/pret/pokeemerald && git -C pokeemerald checkout 731ad5bfd6e6f265508d0efcca0ba42f9dcf5881`.
- Includes use the repo's root-relative style, e.g. `#include <src/gen3/gen3_pkm.h>`. Follow `.clang-format`, snake_case functions and CamelCase typedefs, as in the existing code. Write multi-byte fields byte by byte, little-endian, with no struct-packing tricks.
- Setup in a fresh checkout: `git submodule update --init`, then `ufbt update --channel=release` once.
- Implementers can't run on-device checks. UI tasks write their checklist into `docs/gen3/editor-test.md`, and the user runs it at the gate (Task 12).

## Review Focus

1. **Species past #251** (Treecko to Deoxys): the game stores its *internal* id (Treecko is 277), and internal ids 252–276 are "?" placeholders. Storing the National Dex number gives the wrong Pokemon or a "?". *(Task 2: `test_data_internal_national_map`; Task 6: `test_edit_species_stores_internal`; Task 7: `test_import_bad_species`.)*
2. **One-ability species with an odd PID:** the game reads the ability bit, not the PID, and bit 1 on a one-ability species means no ability at all. *(Task 5: `test_pid_find_matrix`; Task 6: `test_edit_ability_bit_single_ability`.)*
3. **A new PID re-orders and re-keys the 48 secure bytes:** changing nature, gender, shiny, ability or Unown form, including on a loaded Pokemon, must keep every other byte and encrypt with the new PID's order and key, or the game shows a Bad Egg. *(Task 3: `test_substruct_orders`, `test_roundtrip_all_orders`; Task 6: `test_edit_traits_preserve_other_bytes`.)*
4. **Odd files on the SD card:** a 100-byte party file from PKHeX, a wrong-size or corrupt file, an egg, or out-of-range moves must be accepted or refused with a clear message, never half-loaded. *(Task 7: `test_import_100_keeps_bytes`, `test_import_sizes`, `test_import_bad_checksum`, `test_import_egg`, `test_import_bad_move_item`; Task 11 checklist.)*
5. **Out of memory at launch:** the FAP, tables included, loads into RAM; too big and the app won't start. *(Task 2: `.rodata` size step; Task 12: size gate and on-device `free` check.)*

## Design decisions

**D1. Gen III-specific model and scenes, not a GEN_III branch in PokemonData.** `PokemonData` and the
Gen I/II scenes don't fit Gen III, for these reasons:
- **8-bit types throughout:** `NamedList.index` is `uint8_t`, but Gen III move ids go to 354 and item ids to 376. `dex_max`, `bitmap_num`, `table_stat_base_get(…, uint8_t num, …)`, the select view's `uint8_t curr_pokemon` and `pokemon_move.c`'s `uint8_t num` all truncate the 386 species and 411 internal ids.
- **16-bit accessors:** `pokemon_stat_get` returns `uint16_t`, but exp, PID and OT ID are 32-bit in Gen III.
- **Different trait rules:** `pokemon_attribute.c` derives gender, shiny and Unown form from IVs (Gen II). Gen III derives them from the PID (spec call #2).
- **Not host-testable:** `pokemon_data.c` includes `furi.h`, so a GEN_III branch couldn't be tested on the host, which the spec requires.
- **Risk to Gen I/II:** every accessor and most scenes would gain a third branch, against the "don't break Gen I/II" anti-goal. With separate files, reviewers can check that the Gen I/II files are untouched.

What is reused:
- the shared GUI modules owned by `PokemonFap` (submenu, text input, variable item list, DialogEx);
- the scene-alias pattern, where one handler is registered under several scene ids and the caller puts "which one" in a scene state, as Nickname/OTName/UnownForm do;
- the letter-index two-level submenus for moves and items;
- the Pokerus strain/days VariableItemList layout and its A–D strain mapping;
- `stat_list` and the `EvIv` order for the IV/EV preset names;
- `all_sprites.fxbm` and its 404-byte record format;
- `gen3_text`.

`pokemon_icon_get` is *not* reused. It needs a `PokemonData`, caches the species number in a
`uint8_t`, and on a failed read it points `bitmap` at the static `__000_fxbm` and later `free()`s it.
For species 252–386 that read always fails. The editor *logic* (species defaults, PID regeneration,
presets, PP) lives in pure `gen3_edit.c`, so it is tested. The scenes only format text and call it.
Plan 3 points this editor at a traded Pokemon, and plan 5 at a box Pokemon.

**D2. Tables.**
- `tools/gen3_tables.py` reads the pinned pokeemerald clone and writes one file, `src/gen3/gen3_data_gen.c`. Both are committed.
- The hand-written `gen3_data.h` declares the types, the counts and the accessors. `gen3_data.c` implements the accessors. The generated arrays are only reached through `gen3_data.c`.
- Everything is indexed by National Dex number. The game's internal ids appear only in the Pokemon bytes and in `gen3_internal_to_national_tab`.
- Names keep the game's spelling (uppercase), mapped to ASCII: é → E, ♀ → `\200`, ♂ → `\201` (the app's existing Nidoran convention), and `{POKEBLOCK}` → POKEBLOCK.
- Size (all `.rodata`): species info 387×16 = 6,192 B; species names 387×11 = 4,257; internal→national 412×2 = 824; learnset starts 388×2 = 776; learnsets 3,947×2 = 7,894; move names 355×13 = 4,615 + PP 355 + sorted ids 708; item names 377×15 = 5,655 + holdable sorted ids 486; abilities 78×13 = 1,014; natures 200; types 126. Total ≈ 33,100 B.
- The exp tables aren't stored. The six formulas from `experience_tables.h` are ~200 B of code.
- **Fallback** if the RAM gate in Task 12 fails: move learnsets and item/move names (≈ 18 KB) to an SD asset written by the same script. This would be a follow-up after reporting, not part of this plan.

**D3. One byte image.** `Gen3Pkm` is the 100-byte party form in PKHeX `.pk3` layout:
- decrypted, with the substructures unshuffled at 0x20 Growth, 0x2C Attacks, 0x38 EVs/Condition and 0x44 Misc, and the party fields at 0x50;
- fields are read and written through one offset/width table;
- bytes the editor doesn't model (ribbons, contest stats, markings, filler, the unknown u16) survive load → edit → save untouched.

**D4. IV/EV presets** keep Gen II's six names and order:
- Random IV: 5 random bits each.
- Max IV: 31 each.
- Max EV: 85 each, so the six total 510, the game's `MAX_TOTAL_EVS`. 255 each (1,530 in total) would go over that cap.
- Max EV / Level: `85 * level / 100` each.
- Zero EV: 0.
- **Loaded Pokemon:** a preset applies only once it is chosen, or on a new species. Its real IVs/EVs stay until then, and the menu shows `IVs/EVs: as loaded`.

**D5. Species change (call #3)** creates a fresh Pokemon of the new species, the way `CreateBoxMon` does.
- **Kept:** OT name, TID, SID, OT gender, level and the preset.
- **PID:** the first non-shiny random PID, so nature, gender, ability slot and Unown form are random, as in the game.
- **Reset:** nickname = species name; moves = default moveset with full PP; PP Ups 0; friendship = species base value; no held item; IVs/EVs from the preset.
- **Also reset:** contest stats, ribbons, markings and Pokerus 0; ball Poke Ball; met location 0xFF ("fateful encounter"); met level = level; met game 2 (Ruby); status 0; mail 0xFF.
- **Fateful-encounter ribbon bit** set only for Mew (151) and Deoxys (386); FireRed/LeafGreen/Emerald need it for those two to obey.
- Picking the species it already is changes nothing, which protects a loaded Pokemon from a stray OK.

**D6. PID (call #2).**
- `gen3_pid_find` rejection-samples random PIDs until nature, gender, shiny, ability slot and Unown form all match. Shiny candidates are built so they are shiny by construction.
- The ability bit is `PID & 1` for two-ability species and 0 otherwise, as `CreateBoxMon` does.
- Changing OT TID/SID keeps the current shininess by finding a new PID when needed.

**D7. Met data** for Flipper-made Pokemon:
- met location 0xFF: Ruby's summary shows "obtained in a trade" or "fateful encounter", and never looks up a map name for it;
- met level 1–100: never 0, which Ruby would read as "hatched" and use to look up a location name;
- met game Ruby.

**D8. Placeholder sprite** for species 252–386: a drawn rounded frame with a "?" (no new art, no asset).

## File map

| File | Responsibility |
|---|---|
| `src/gen3/gen3_text.{h,c}` | modify: ♀/♂ ↔ `\200`/`\201` |
| `tools/gen3_tables.py` | generator: pokeemerald clone → `gen3_data_gen.c` |
| `src/gen3/gen3_data.h`, `gen3_data.c` | table types, counts, lookups by National Dex / move / item id |
| `src/gen3/gen3_data_gen.c` | generated, committed arrays |
| `src/gen3/gen3_pkm.{h,c}` | 100-byte Pokemon: fields, names, checksum, shuffle, encrypt/decrypt |
| `src/gen3/gen3_rules.{h,c}` | exp ↔ level, stats, natures, gender/shiny/Unown, party recalc, RNG, PID search, default moves |
| `src/gen3/gen3_edit.{h,c}` | editor model: every edit keeps the Pokemon game-valid |
| `src/gen3/gen3_pk3.{h,c}` | `.pk3` import/export, validation, file names |
| `src/gen3/gen3_pk3_furi.{h,c}` | `.pk3` files on the SD card |
| `src/views/gen3_species_select.{h,c}` | species picker view (sprite or placeholder) |
| `src/scenes/pokemon_gen3_edit.c` | editor menu |
| `src/scenes/pokemon_gen3_species.c`, `pokemon_gen3_text.c`, `pokemon_gen3_number.c`, `pokemon_gen3_pick.c`, `pokemon_gen3_move.c`, `pokemon_gen3_item.c`, `pokemon_gen3_pokerus.c`, `pokemon_gen3_pk3.c` | editor sub-scenes, load/save |
| `src/scenes/pokemon_gen3_menu.c`, `src/scenes/include/pokemon_scene_config.h`, `src/include/pokemon_app.h`, `src/pokemon_app.c` | wiring |
| `tests/test_gen3_{data,pkm,rules,edit,pk3}.c`, `tests/gen3_golden.h`, `tests/Makefile`, `tests/test_gen3_text.c` | host tests |
| `docs/gen3/editor-test.md`, `README.md`, `changelog.md` | user checklist and docs |

---

### Task 1: Gen III text: ♀ and ♂

**Files:**
- Modify: `src/gen3/gen3_text.c`, `src/gen3/gen3_text.h` (comment only)
- Test: `tests/test_gen3_text.c`

**Interfaces:**
- Produces: no new functions.
  - `gen3_text_encode` maps `'\200'` → 0xB6 (♀) and `'\201'` → 0xB5 (♂).
  - `gen3_text_decode` maps 0xB6 → `'\200'` and 0xB5 → `'\201'`. Every other mapping is unchanged.

- [ ] **Step 1: Record the baseline.** Run `ufbt`, then `arm-none-eabi-size dist/debug/pokemon_d.elf` (path in Global Constraints). Put the `text`/`data` numbers in your report. The expected value is about 56,348 for text + data.
- [ ] **Step 2: Write the failing tests** (add them to `main`):

```c
TEST(test_encode_gender_symbols) { /* "NIDORAN\200", field 10, max 10 -> returns 8;
                                      C8 C3 BE C9 CC BB C8 B6 FF FF.  "\201" -> B5 FF */ }
TEST(test_decode_gender_symbols) { /* {0xB5, 0xB6, 0xFF} -> "\201\200", returns 2 */ }
```

- [ ] **Step 3: Run** `make -C tests`. Expected: these 2 fail and the existing 7 pass.
- [ ] **Step 4: Implement** the two mappings in `gen3_char_encode`/`gen3_char_decode` (compare against the `'\200'`/`'\201'` literals, since `char` is signed on the host and unsigned on ARM). Update the header comment.
- [ ] **Step 5: Run** `make -C tests`. Expected: `test_gen3_text: 9 passed`.
- [ ] **Step 6: Commit** `gen3: text: encode and decode the female and male symbols`.

---

### Task 2: Table generator and Gen III data

**Files:**
- Create: `tools/gen3_tables.py`, `src/gen3/gen3_data.h`, `src/gen3/gen3_data.c`, `src/gen3/gen3_data_gen.c` (generated)
- Test: `tests/test_gen3_data.c`

**Interfaces:**
- Produces (`gen3_data.h`):
  ```c
  #define GEN3_NATIONAL_COUNT 386   /* National Dex 1..386 */
  #define GEN3_INTERNAL_COUNT 412   /* game species ids 0..411 (412 is the egg, excluded) */
  #define GEN3_MOVE_COUNT 355       /* move ids 0..354 */
  #define GEN3_ITEM_COUNT 377       /* item ids 0..376 */
  #define GEN3_ABILITY_COUNT 78
  #define GEN3_NATURE_COUNT 25
  #define GEN3_TYPE_COUNT 18
  #define GEN3_LEARNSET_COUNT 3947
  #define GEN3_HOLDABLE_COUNT 243
  #define GEN3_SPRITE_MAX_NATIONAL 251 /* files/all_sprites.fxbm covers 1..251 */
  #define GEN3_NATIONAL_MEW 151
  #define GEN3_NATIONAL_UNOWN 201
  #define GEN3_NATIONAL_SHEDINJA 292
  #define GEN3_NATIONAL_DEOXYS 386
  #define GEN3_GENDER_MALE_ONLY 0
  #define GEN3_GENDER_FEMALE_ONLY 254
  #define GEN3_GENDERLESS 255
  typedef enum { Gen3GrowthMediumFast, Gen3GrowthErratic, Gen3GrowthFluctuating,
                 Gen3GrowthMediumSlow, Gen3GrowthFast, Gen3GrowthSlow } Gen3Growth; /* game values 0..5 */
  typedef struct {
      uint8_t base[6];      /* HP, Atk, Def, Spe, SpA, SpD */
      uint8_t type[2];      /* game type ids; one-type species repeat it */
      uint8_t gender_ratio; /* 0 male only, 254 female only, 255 genderless, else female if (pid & 0xFF) < ratio */
      uint8_t growth;       /* Gen3Growth */
      uint8_t ability[2];   /* ability[1] == 0: one ability */
      uint8_t friendship;   /* base friendship */
      uint16_t internal;    /* game species id */
  } Gen3SpeciesInfo;        /* 16 bytes */
  const Gen3SpeciesInfo* gen3_species(uint16_t national);          /* NULL unless 1..386 */
  const char* gen3_species_name(uint16_t national);                /* "?" unless 1..386 */
  uint16_t gen3_national_from_internal(uint16_t internal);         /* 0 = not a real species */
  uint16_t gen3_internal_from_national(uint16_t national);         /* 0 unless 1..386 */
  size_t gen3_learnset(uint16_t national, const uint16_t** entries);/* entries are (level << 9) | move, in game order; 0 + NULL if invalid */
  const char* gen3_move_name(uint16_t move);                       /* "???" if >= GEN3_MOVE_COUNT; move 0 is "-" */
  uint8_t gen3_move_pp(uint16_t move);                             /* base PP; 0 if out of range */
  uint16_t gen3_move_by_name(size_t i);                            /* i < GEN3_MOVE_COUNT - 1: ids 1..354 sorted by name */
  const char* gen3_item_name(uint16_t item);                       /* "???" if unused or >= GEN3_ITEM_COUNT */
  bool gen3_item_holdable(uint16_t item);
  uint16_t gen3_holdable_by_name(size_t i);                        /* i < GEN3_HOLDABLE_COUNT, sorted by name */
  const char* gen3_ability_name(uint8_t ability);                  /* "???" if out of range */
  const char* gen3_nature_name(uint8_t nature);                    /* "???" if >= 25 */
  const char* gen3_type_name(uint8_t type);                        /* "???" if >= 18 */
  /* Generated arrays (gen3_data_gen.c). Use the functions above. */
  extern const Gen3SpeciesInfo gen3_species_info_tab[GEN3_NATIONAL_COUNT + 1];
  extern const char gen3_species_name_tab[GEN3_NATIONAL_COUNT + 1][11];
  extern const uint16_t gen3_internal_to_national_tab[GEN3_INTERNAL_COUNT];
  extern const uint16_t gen3_learnset_start_tab[GEN3_NATIONAL_COUNT + 2]; /* [n]..[n+1]; [0]=[1]=0, [387]=3947 */
  extern const uint16_t gen3_learnset_tab[GEN3_LEARNSET_COUNT];
  extern const char gen3_move_name_tab[GEN3_MOVE_COUNT][13];
  extern const uint8_t gen3_move_pp_tab[GEN3_MOVE_COUNT];
  extern const uint16_t gen3_move_by_name_tab[GEN3_MOVE_COUNT - 1];
  extern const char gen3_item_name_tab[GEN3_ITEM_COUNT][15];      /* "" for unused ids */
  extern const uint16_t gen3_holdable_by_name_tab[GEN3_HOLDABLE_COUNT];
  extern const char gen3_ability_name_tab[GEN3_ABILITY_COUNT][13];
  extern const char gen3_nature_name_tab[GEN3_NATURE_COUNT][8];
  extern const char gen3_type_name_tab[GEN3_TYPE_COUNT][7];
  ```

**Generator (`tools/gen3_tables.py`)**
- **Usage:** `python3 -I tools/gen3_tables.py <pokeemerald>`, standard library only. All paths below are relative to the clone.
- **Commit check:** it first runs `git -C <path> rev-parse HEAD` and exits 1 unless the result is `731ad5bfd6e6f265508d0efcca0ba42f9dcf5881`.

Inputs:
- **Ids:**
  - `SPECIES_*`: `#define`s in `include/constants/species.h`.
  - `NATIONAL_DEX_*`: the first `enum` in `include/constants/pokedex.h`, counted from `NATIONAL_DEX_NONE` = 0.
  - `ITEM_*`: the first `enum` in `include/constants/items.h`, counted from `ITEM_NONE` = 0.
  - `MOVE_*` (`include/constants/moves.h`) and `ABILITY_*` (`include/constants/abilities.h`): `#define`s.
  - `TYPE_*`, `GROWTH_*`, `NATURE_*` and `STANDARD_FRIENDSHIP`: from `include/constants/pokemon.h`.
- **Internal → national:** each `SPECIES_TO_NATIONAL(X)` line in `sSpeciesToNationalPokedexNum` (`src/pokemon.c`) maps `SPECIES_X` to `NATIONAL_DEX_X`. Values above 386 (the old Unown 252–276) become 0.
- **Species info:** each `[SPECIES_X] = { … },` block in `src/data/pokemon/species_info.h` gives:
  - `.baseHP` … `.baseSpDefense`, `.types`, `.abilities`, `.growthRate`;
  - `.friendship`: a number or `STANDARD_FRIENDSHIP`;
  - `.genderRatio`: `MON_MALE` 0, `MON_FEMALE` 254, `MON_GENDERLESS` 255, or `PERCENT_FEMALE(p)` = `min(254, int(p * 255 / 100))`, so 12.5 → 31 and 50 → 127.
  - Blocks using `OLD_UNOWN_SPECIES_INFO` are skipped.
- **Learnsets:** each `static const u16 sXLevelUpLearnset[] = { LEVEL_UP_MOVE(l, MOVE_Y), … }` (`level_up_learnsets.h`), assigned to species through `[SPECIES_X] = sXLevelUpLearnset` (`level_up_learnset_pointers.h`). Store `(l << 9) | id`.
- **Moves:**
  - names: `[MOVE_X] = _("…")` in `src/data/text/move_names.h`;
  - PP: `.pp = n` in each `[MOVE_X] =` block of `src/data/battle_moves.h`.
- **Species names:** `[SPECIES_X] = _("…")` in `src/data/text/species_names.h`.
- **Items:** each `[ITEM_X] = { … },` block in `src/data/items.h`.
  - **Id:** the designator's enum value. If the designator is an alias (e.g. `ITEM_TM_FOCUS_PUNCH`), use `.itemId` instead.
  - **Name:** `.name`. `"????????"` becomes `""`.
  - **Holdable:** the name isn't `"????????"`, the block has no `.importance = 1`, `.pocket` isn't `POCKET_KEY_ITEMS`, and the id isn't `ITEM_ENIGMA_BERRY` (175). The Enigma Berry carries e-Reader data that a trade would need too.
- **Abilities:** `[ABILITY_X] = _("…")` inside `gAbilityNames` in `src/data/text/abilities.h`.
- **Natures:** `[NATURE_X] = sYNatureName` resolved through `sYNatureName[] = _("…")` in `src/data/text/nature_names.h`.
- **Types:** `[TYPE_X] = _("…")` inside `gTypeNames` in `src/battle_main.c`.

Text mapping:
- é → `E`, ♀ → `\200`, ♂ → `\201` (written as C escapes), `{POKEBLOCK}` → `POKEBLOCK`.
- Any other character outside 0x20–0x7E is a fatal error.

Checks (exit 1 and name the first failure):
- every national 1..386 has exactly one internal id, and internal 252..276 map to 0;
- there are 355 move names and 355 PP values, 377 items, 78 abilities, 25 natures and 18 types;
- names fit (species ≤ 10 characters, move ≤ 12, item ≤ 14, ability ≤ 12, nature ≤ 7, type ≤ 6);
- every learnset entry has level 1–100 and move 1–354, and every species has a level-1 move (so a default moveset is never empty);
- there are 3,947 learnset entries and 243 holdable items.

Output: `src/gen3/gen3_data_gen.c`.
- **Header comment:** `/* Generated by tools/gen3_tables.py from pret/pokeemerald 731ad5bfd6e6f265508d0efcca0ba42f9dcf5881. Do not edit. Game facts only. */`.
- **Includes:** only `<src/gen3/gen3_data.h>`.
- **Order:** arrays as declared above, in National Dex order, one entry per line with a `/* n NAME */` comment. Sorted lists are ordered by (name bytes, id) ascending.
- **Format:** deterministic, LF line endings, and fully braced initializers, so `-Wall -Wextra` stays quiet.

- [ ] **Step 1: Write the failing tests** in `tests/test_gen3_data.c`:

```c
TEST(test_data_internal_national_map) /* internal_from_national(252)==277; national_from_internal(277)==252, (1)==1, (251)==251,
                                         each of 252..276 == 0, (0)==0, (303)==292, (392)==280, (410)==386, (411)==358, (412)==0,
                                         (0xFFFF)==0; internal_from_national(0)==0, (387)==0; every n in 1..386 round-trips */
TEST(test_data_bulbasaur)    /* 1: base {45,49,49,45,65,65}, type {12,3}, ratio 31, growth 3, ability {65,0}, friendship 70, "BULBASAUR" */
TEST(test_data_treecko)      /* 252: base {40,45,35,70,65,55}, type {12,12}, ratio 31, growth 3, ability {65,0}, "TREECKO" */
TEST(test_data_deoxys)       /* 386: internal 410, base {50,150,50,150,150,50}, type {14,14}, ratio 255, growth 5, ability {46,0}, friendship 0 */
TEST(test_data_shedinja)     /* 292: internal 303, base[0]==1, type {6,7}, ratio 255, growth 1, ability {25,0} */
TEST(test_data_unown)        /* 201: base {48,72,48,48,72,48}, type {14,14}, ratio 255, growth 0, ability {26,0};
                                gen3_learnset(201,&e)==1, e[0]==(1<<9|237) */
TEST(test_data_two_abilities)/* Ralts 280: ability {28,36}, ratio 127, growth 5, friendship 35 */
TEST(test_data_names)        /* 29 "NIDORAN\200" ratio 254; 32 "NIDORAN\201" ratio 0; 83 "FARFETCH'D"; 122 "MR. MIME"; 250 "HO-OH";
                                species_name(0)=="?", (387)=="?"; species(0)==NULL, species(387)==NULL */
TEST(test_data_learnsets)    /* Bulbasaur: 11 entries, [0]==(1<<9|33), [10]==(46<<9|76); Treecko: 11, [2]==(6<<9|71);
                                Ivysaur (2): [3]==(4<<9|45); learnset(0)==0; sum over 1..386 == GEN3_LEARNSET_COUNT */
TEST(test_data_moves)        /* "POUND" pp 35 (1); "THUNDERPUNCH" (9); "HIDDEN POWER" pp 15 (237); "PSYCHO BOOST" pp 5 (354);
                                move_name(0)=="-"; (355)=="???", pp(355)==0; by_name(0)==71 ABSORB, (1)==51 ACID, (353)==192 ZAP CANNON;
                                names strictly ascending (strcmp) over all 354 */
TEST(test_data_items)        /* 1 "MASTER BALL", 4 "POKE BALL", 12 "PREMIER BALL", 175 "ENIGMA BERRY", 200 "LEFTOVERS", 52 "???", 377 "???";
                                holdable: 4,200,289(TM01) true; 0,52,175,259(MACH BIKE),339(HM01),376 false;
                                holdable_by_name(0)==146 AGUAV BERRY, (242)==70 ZINC; ascending; every listed id holdable */
TEST(test_data_small_lists)  /* ability 0 "-------", 65 "OVERGROW", 77 "AIR LOCK", 78 "???"; nature 0 "HARDY", 3 "ADAMANT", 24 "QUIRKY",
                                25 "???"; type 9 "???", 12 "GRASS", 13 "ELECTR", 17 "DARK", 18 "???" */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: compile failure, `gen3_data.h` not found.
- [ ] **Step 3: Write** `gen3_data.h` (as above) and `gen3_data.c` (bounds checks plus array lookups; holdable is a linear scan of the sorted list).
- [ ] **Step 4: Write** `tools/gen3_tables.py` as specified. Run it against the pinned clone, then run it again: `shasum src/gen3/gen3_data_gen.c` must be the same both times. Run it on `/Users/kyvo/.claude/jobs/72354b36/tmp/pokeruby`: it must exit 1 with the commit message.
- [ ] **Step 5: Run** `make -C tests`. Expected: all `test_gen3_data` tests pass, and every existing test still passes.
- [ ] **Step 6: Size check.** Run `T=$(ls -d ~/.ufbt/toolchain/*/bin); $T/arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -Os -std=gnu17 -I. -c src/gen3/gen3_data_gen.c -o /tmp/gen3_data_gen.o && $T/arm-none-eabi-size -A /tmp/gen3_data_gen.o`. Expected: `.rodata` ≤ 33,500, and `.data`/`.bss` 0. Put the number in your report.
- [ ] **Step 7: Build** with `ufbt`. Expected: 0 warnings.
- [ ] **Step 8: Commit** `gen3: add Gen III data tables generated from pokeemerald` (stage the 4 files and the test).

---

### Task 3: Pokemon byte format, checksum, encryption

**Files:**
- Create: `src/gen3/gen3_pkm.h`, `src/gen3/gen3_pkm.c`, `tests/gen3_golden.h`
- Modify: `tests/Makefile` (in the `$(BUILD)/%` prerequisites, replace `test.h` with `$(wildcard *.h)`)
- Test: `tests/test_gen3_pkm.c`

**Interfaces:**
- Consumes: `gen3_text_encode`/`gen3_text_decode`.
- Produces:
  ```c
  #define GEN3_BOX_SIZE 80
  #define GEN3_PARTY_SIZE 100
  #define GEN3_NICKNAME_LEN 10
  #define GEN3_OT_NAME_LEN 7
  #define GEN3_MON_LANGUAGE_ENGLISH 2  /* same value as GEN3_LANGUAGE_ENGLISH in gen3_link_player.h */
  #define GEN3_MAIL_NONE 0xFF
  #define GEN3_METLOC_FATEFUL 0xFF
  typedef enum { Gen3StatHp, Gen3StatAtk, Gen3StatDef, Gen3StatSpe, Gen3StatSpa, Gen3StatSpd, Gen3StatCount } Gen3Stat;
  typedef struct { uint8_t raw[GEN3_PARTY_SIZE]; } Gen3Pkm; /* decrypted, PKHeX .pk3 layout (D3) */
  typedef enum {
      Gen3FieldPid, Gen3FieldOtId, Gen3FieldLanguage, Gen3FieldBadEgg, Gen3FieldHasSpecies, Gen3FieldEggFlag,
      Gen3FieldMarkings, Gen3FieldChecksum,
      Gen3FieldSpecies, Gen3FieldHeldItem, Gen3FieldExp, Gen3FieldPpBonuses, Gen3FieldFriendship,
      Gen3FieldMove1, Gen3FieldMove2, Gen3FieldMove3, Gen3FieldMove4,
      Gen3FieldPp1, Gen3FieldPp2, Gen3FieldPp3, Gen3FieldPp4,
      Gen3FieldEvHp, Gen3FieldEvAtk, Gen3FieldEvDef, Gen3FieldEvSpe, Gen3FieldEvSpa, Gen3FieldEvSpd,
      Gen3FieldPokerus, Gen3FieldMetLocation, Gen3FieldMetLevel, Gen3FieldMetGame, Gen3FieldBall, Gen3FieldOtGender,
      Gen3FieldIvHp, Gen3FieldIvAtk, Gen3FieldIvDef, Gen3FieldIvSpe, Gen3FieldIvSpa, Gen3FieldIvSpd,
      Gen3FieldIsEgg, Gen3FieldAbilityBit, Gen3FieldFateful,
      Gen3FieldStatus, Gen3FieldLevel, Gen3FieldMail, Gen3FieldHp,
      Gen3FieldMaxHp, Gen3FieldAtk, Gen3FieldDef, Gen3FieldSpe, Gen3FieldSpa, Gen3FieldSpd,
      Gen3FieldCount,
  } Gen3Field; /* Gen3FieldEvHp + s, Gen3FieldIvHp + s, Gen3FieldMaxHp + s index by Gen3Stat s */
  uint32_t gen3_pkm_get(const Gen3Pkm* p, Gen3Field f);
  void gen3_pkm_set(Gen3Pkm* p, Gen3Field f, uint32_t value);       /* value masked to the field width */
  size_t gen3_pkm_nickname(const Gen3Pkm* p, char* out, size_t out_size);  /* gen3_text_decode */
  void gen3_pkm_set_nickname(Gen3Pkm* p, const char* ascii);        /* encode, max 10, EOS-filled */
  size_t gen3_pkm_ot_name(const Gen3Pkm* p, char* out, size_t out_size);
  void gen3_pkm_set_ot_name(Gen3Pkm* p, const char* ascii);         /* max 7, EOS-filled */
  uint16_t gen3_pkm_checksum(const Gen3Pkm* p);   /* u16 sum of the 24 LE words at 0x20..0x4F */
  void gen3_pkm_fix_checksum(Gen3Pkm* p);
  /* Game form: header copied, checksum recomputed into out, the 4 substructs placed by PID % 24,
   * each LE u32 of 0x20..0x4F XORed with PID ^ OTID, party bytes copied. */
  void gen3_pkm_encrypt(const Gen3Pkm* p, uint8_t out[GEN3_PARTY_SIZE]);
  /* Inverse. len 80 (party bytes zeroed) or 100. false on any other len or a checksum mismatch; *out unchanged then. */
  bool gen3_pkm_decrypt(const uint8_t* in, size_t len, Gen3Pkm* out);
  ```

Field table:
- Each row gives byte offset, bytes, bit shift and bit width. Rows with no width are whole bytes.
- **Header and Growth/Attacks/EVs:**

  | Field | Off | Bytes | Shift | Bits |
  |---|---|---|---|---|
  | Pid / OtId | 0x00 / 0x04 | 4 | 0 | 32 |
  | Language | 0x12 | 1 | 0 | 8 |
  | BadEgg / HasSpecies / EggFlag | 0x13 | 1 | 0 / 1 / 2 | 1 |
  | Markings / Checksum | 0x1B / 0x1C | 1 / 2 | 0 | 8 / 16 |
  | Species / HeldItem / Exp | 0x20 / 0x22 / 0x24 | 2 / 2 / 4 | 0 | 16 / 16 / 32 |
  | PpBonuses / Friendship | 0x28 / 0x29 | 1 | 0 | 8 |
  | Move1..4 | 0x2C + 2i | 2 | 0 | 16 |
  | Pp1..4 | 0x34 + i | 1 | 0 | 8 |
  | EvHp..EvSpd | 0x38 + s | 1 | 0 | 8 |

- **Misc and party:**

  | Field | Off | Bytes | Shift | Bits |
  |---|---|---|---|---|
  | Pokerus / MetLocation | 0x44 / 0x45 | 1 | 0 | 8 |
  | MetLevel / MetGame / Ball / OtGender | 0x46 | 2 | 0 / 7 / 11 / 15 | 7 / 4 / 4 / 1 |
  | IvHp..IvSpd | 0x48 | 4 | 5s | 5 |
  | IsEgg / AbilityBit | 0x48 | 4 | 30 / 31 | 1 |
  | Fateful | 0x4C | 4 | 31 | 1 |
  | Status / Level / Mail | 0x50 / 0x54 / 0x55 | 4 / 1 / 1 | 0 | 32 / 8 / 8 |
  | Hp, MaxHp, Atk, Def, Spe, Spa, Spd | 0x56 + 2k | 2 | 0 | 16 |

Substructure positions:
- From pokeemerald `SUBSTRUCT_CASE`: `pos[pid % 24][t]` is the position of substruct t (0 Growth, 1 Attacks, 2 EVs, 3 Misc) in the game form.
- Table: `{0,1,2,3},{0,1,3,2},{0,2,1,3},{0,3,1,2},{0,2,3,1},{0,3,2,1}, {1,0,2,3},{1,0,3,2},{2,0,1,3},{3,0,1,2},{2,0,3,1},{3,0,2,1}, {1,2,0,3},{1,3,0,2},{2,1,0,3},{3,1,0,2},{2,3,0,1},{3,2,0,1}, {1,2,3,0},{1,3,2,0},{2,1,3,0},{3,1,2,0},{2,3,1,0},{3,2,1,0}`.

**Golden vector.** It goes in `tests/gen3_golden.h`. The header holds `static const` arrays and two `static inline` helpers: `void golden_build(Gen3Pkm* p)` sets the fields below through `gen3_pkm_set`, and `void golden_game(uint8_t out[100])` assembles GOLDEN_GAME from GOLDEN_PK3 and GOLDEN_GAME_SECURE.
- **Box fields:**
  - PID 0x9E377A3F: Adamant; PID % 24 = 23, so the game order is M E A G; low byte 0x3F ≥ 31, so a male Treecko; not shiny.
  - OT ID 0x000056CE (TID 22222, SID 0); nickname "TREECKO"; OT "FLIPPER"; language 2; HasSpecies 1.
  - Species 277; item 0; exp 135; PP Ups 0; friendship 70.
  - Moves 1/43 (Pound, Leer); PP 35/30; EVs 0.
  - Met location 0xFF, met level 5, met game 2, ball 4, OT gender 0.
  - IVs 31/30/29/28/27/26 (HP/Atk/Def/Spe/SpA/SpD); ability bit 0; ribbons 0.
- **Party fields:** status 0, level 5, mail 0xFF, HP 20/20, Atk 12, Def 9, Spe 13, SpA 10, SpD 11.
- **Where it came from:** a scratch program, not committed. It compiled pokeemerald 731ad5bf's `struct BoxPokemon`/`struct Pokemon` (`include/pokemon.h`) with clang (sizes 80 and 100). It then ran the `GetSubstruct`, `CalculateBoxMonChecksum`, `EncryptBoxMon` and `CalculateMonStats`/`ModifyStatByNature` algorithms from `src/pokemon.c`.
- **Hand checks:**
  - The checksum 0x338D is the u16 sum of the 24 words.
  - The first encrypted word is Misc word 0, 0x2105FF00, XOR (0x9E377A3F ^ 0x000056CE = 0x9E372CF1) = 0xBF32D3F1.

```c
static const uint8_t GOLDEN_PK3[100] = { /* decrypted, G A E M, party form */
    0x3F,0x7A,0x37,0x9E,0xCE,0x56,0x00,0x00,0xCE,0xCC,0xBF,0xBF,0xBD,0xC5,0xC9,0xFF,
    0xFF,0xFF,0x02,0x02,0xC0,0xC6,0xC3,0xCA,0xCA,0xBF,0xCC,0x00,0x8D,0x33,0x00,0x00,
    0x15,0x01,0x00,0x00,0x87,0x00,0x00,0x00,0x00,0x46,0x00,0x00,0x01,0x00,0x2B,0x00,
    0x00,0x00,0x00,0x00,0x23,0x1E,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,
    0x00,0x00,0x00,0x00,0x00,0xFF,0x05,0x21,0xDF,0x77,0xBE,0x35,0x00,0x00,0x00,0x00,
    0x00,0x00,0x00,0x00,0x05,0xFF,0x14,0x00,0x14,0x00,0x0C,0x00,0x09,0x00,0x0D,0x00,
    0x0A,0x00,0x0B,0x00};
/* GOLDEN_GAME[100]: GOLDEN_PK3 with bytes 0x20..0x4F replaced by: */
static const uint8_t GOLDEN_GAME_SECURE[48] = {
    0xF1,0xD3,0x32,0xBF,0x2E,0x5B,0x89,0xAB,0xF1,0x2C,0x37,0x9E,0xF1,0x2C,0x37,0x9E,
    0xF1,0x2C,0x37,0x9E,0xF1,0x2C,0x37,0x9E,0xF0,0x2C,0x1C,0x9E,0xF1,0x2C,0x37,0x9E,
    0xD2,0x32,0x37,0x9E,0xE4,0x2D,0x37,0x9E,0x76,0x2C,0x37,0x9E,0xF1,0x6A,0x37,0x9E};
```

- [ ] **Step 1: Write the failing tests** in `tests/test_gen3_pkm.c`:

```c
TEST(test_field_layout)     /* zeroed pkm: set Pid 0x11223344 -> raw[0..3] 44 33 22 11; MetLevel 5, MetGame 2, Ball 4, OtGender 1 ->
                               raw[0x46..0x47] 05 A1; IVs 31,30,29,28,27,26 + AbilityBit 1 -> raw[0x48..0x4B] DF 77 BE B5;
                               HasSpecies 1 -> raw[0x13]==0x02; Fateful 1 -> raw[0x4F]==0x80; Spd 0x0102 -> raw[0x62..0x63] 02 01 */
TEST(test_field_masking)    /* MetLevel 0xFF -> get 0x7F, MetGame unchanged; IvAtk 40 -> get 8, IvHp/IvDef unchanged */
TEST(test_names)            /* nickname "TREECKO" -> raw[0x08..0x11] CE CC BF BF BD C5 C9 FF FF FF, read back "TREECKO";
                               ot "FLIPPERZERO" -> raw[0x14..0x1A] C0 C6 C3 CA CA BF CC, read back "FLIPPER" */
TEST(test_golden_build)     /* golden_build + fix_checksum: checksum()==0x338D, raw == GOLDEN_PK3 (100 bytes) */
TEST(test_golden_encrypt)   /* encrypt(golden) == GOLDEN_PK3 with 0x20..0x4F = GOLDEN_GAME_SECURE */
TEST(test_golden_decrypt)   /* decrypt(GOLDEN_GAME, 100) true, raw == GOLDEN_PK3; len 80 -> true, raw[0..79] equal, raw[80..99] zero */
TEST(test_substruct_orders) /* for n in 0..23: pid with pid%24==n, ot_id = pid (XOR key 0); raw[0x20]=0xA0, [0x2C]=0xA1,
                               [0x38]=0xA2, [0x44]=0xA3; encrypt; for pos 0..3: out[0x20+12*pos] == 0xA0 + index of ORDER[n][pos] in "GAEM".
                               ORDER = GAEM GAME GEAM GEMA GMAE GMEA AGEM AGME AEGM AEMG AMGE AMEG
                                       EGAM EGMA EAGM EAMG EMGA EMAG MGAE MGEA MAGE MAEG MEGA MEAG (Bulbapedia) */
TEST(test_roundtrip_all_orders) /* secure bytes from a fixed LCG, ot 0xDEADBEEF, pid 0x9E377A3F + n for n 0..23:
                               fix_checksum, encrypt, decrypt -> true and identical */
TEST(test_decrypt_bad_checksum) /* GOLDEN_GAME with [0x30] ^= 1 -> false; out (pre-filled 0x5A) unchanged */
TEST(test_decrypt_bad_len)  /* len 0, 79, 99 -> false */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL (missing `gen3_pkm.h`).
- [ ] **Step 3: Implement** `gen3_pkm.{h,c}`: one `static const` field table and the position table above.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add Gen III Pokemon byte format, checksum and encryption`.

---

### Task 4: Exp, stats, natures, PID-derived traits

**Files:**
- Create: `src/gen3/gen3_rules.h`, `src/gen3/gen3_rules.c`
- Test: `tests/test_gen3_rules.c`

**Interfaces:**
- Consumes: Tasks 2, 3.
- Produces:
  ```c
  uint32_t gen3_exp_for_level(uint8_t growth, uint8_t level);   /* level 0..100 (clamped); 0 -> 0, 1 -> 1 */
  uint8_t gen3_level_for_exp(uint8_t growth, uint32_t exp);       /* GetLevelFromBoxMonExp: largest L <= 100 with exp_for(L) <= exp */
  int8_t gen3_nature_effect(uint8_t nature, Gen3Stat stat);       /* +1, -1, 0; HP 0 */
  /* HP: (2b+iv+ev/4)*L/100 + L + 10. Others: (2b+iv+ev/4)*L/100 + 5, then *110/100 or *90/100 (integer). */
  uint16_t gen3_stat_value(Gen3Stat stat, uint8_t base, uint8_t iv, uint8_t ev, uint8_t level, uint8_t nature);
  void gen3_calc_stats(const Gen3Pkm* p, uint16_t out[Gen3StatCount]); /* level from exp; Shedinja max HP = 1 */
  void gen3_recalc_party(Gen3Pkm* p);  /* sets Level, MaxHp..Spd, Hp = MaxHp; Status and Mail untouched */
  typedef enum { Gen3GenderMale, Gen3GenderFemale, Gen3GenderNone } Gen3Gender;
  Gen3Gender gen3_gender_of(uint8_t gender_ratio, uint32_t pid);  /* GetGenderFromSpeciesAndPersonality */
  bool gen3_is_shiny(uint32_t ot_id, uint32_t pid);               /* (TID ^ SID ^ pid>>16 ^ pid&0xFFFF) < 8 */
  uint8_t gen3_unown_letter(uint32_t pid);                        /* GET_UNOWN_LETTER: 0..25 A..Z, 26 '!', 27 '?' */
  typedef struct {
      uint8_t nature;       /* pid % 25 */
      Gen3Gender gender;
      bool shiny;
      uint8_t ability_slot; /* the stored ability bit */
      uint8_t unown_letter;
  } Gen3Traits;
  void gen3_traits_of(const Gen3Pkm* p, Gen3Traits* out);
  ```

Exp formulas, from `experience_tables.h`, in signed 32-bit integer arithmetic, with n = level ≥ 2:
- **Medium Fast:** n³.
- **Erratic:**
  - n ≤ 50: (100−n)n³/50;
  - n ≤ 68: (150−n)n³/100;
  - n ≤ 98: ((1911−10n)/3)·n³/500;
  - otherwise (160−n)n³/100.
- **Fluctuating:**
  - n ≤ 15: ((n+1)/3+24)n³/50;
  - n ≤ 36: (n+14)n³/50;
  - otherwise (n/2+32)n³/50.
- **Medium Slow:** 6n³/5 − 15n² + 100n − 140.
- **Fast:** 4n³/5.
- **Slow:** 5n³/4.

Nature effect (`gNatureStatTable`):
- With the stats ordered Atk, Def, Spe, SpA, SpD, nature n raises stat n/5 and lowers stat n%5.
- When n/5 == n%5 the nature is neutral.

- [ ] **Step 1: Write the failing tests:**

```c
TEST(test_exp_table)       /* growth 0..5 at L2 / L50 / L100: MF 8/125000/1000000; Erratic 15/125000/600000;
                              Fluct 4/142500/1640000; MS 9/117360/1059860; Fast 6/100000/800000; Slow 10/156250/1250000;
                              Erratic L51 131324, L68 257834, L69 267406, L98 583539, L99 591882;
                              Fluct L15 1957, L16 2457, L36 46656, L37 50653; every growth L0 == 0, L1 == 1 */
TEST(test_level_for_exp)   /* MS: 134 -> 4, 135 -> 5; any growth: 0 -> 0, 1 -> 1, 0xFFFFFFFF -> 100; Fluct 1640000 -> 100;
                              Erratic 599999 -> 99 */
TEST(test_nature_effect)   /* all 25x5 equal the gNatureStatTable values (copy that table's numbers into the test as expected data);
                              HP always 0 */
TEST(test_stat_value)      /* (Atk,49,31,0,100,3)==147; (Spa,65,31,0,100,3)==149; (Hp,45,31,0,100,0)==231;
                              (Def,230,31,255,100,5)==614; (Hp,40,31,85,50,0)==126 */
TEST(test_calc_stats_golden)/* golden_build: {20,12,9,13,10,11} */
TEST(test_shedinja_hp_one) /* species 303, exp for L50 (Erratic), IVs 31, EVs 255, nature 0: out[Hp]==1, out[Atk]==142 */
TEST(test_recalc_party)    /* golden with raw[0x50..0x63] zeroed then Mail=0xFF: recalc_party -> raw[0x50..0x63] == GOLDEN_PK3[0x50..0x63] */
TEST(test_gender_of)       /* ratio 31: pid low 30 -> F, 31 -> M; ratio 0, low 0 -> M; 254, low 255 -> F; 255 -> None */
TEST(test_is_shiny)        /* ot 22222: pid (0<<16)|(22222^7) -> true; |(22222^8) -> false */
TEST(test_unown_letter)    /* 0x00000000->0, 0x00000001->1, 0x03030303->3, 0xFFFFFFFF->3, 0x01000000->8, 0x00010202->26, 0x00010203->27 */
TEST(test_traits_of_golden)/* nature 3, Male, not shiny, ability_slot 0 */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement** `gen3_rules.{h,c}`.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add exp, stat, nature and PID trait rules`.

---

### Task 5: RNG, PID search, default moveset

**Files:**
- Modify: `src/gen3/gen3_rules.h`, `src/gen3/gen3_rules.c`
- Test: `tests/test_gen3_rules.c`

**Interfaces:**
- Produces:
  ```c
  typedef struct { uint32_t state; } Gen3Rng;   /* xorshift32 (13, 17, 5) */
  void gen3_rng_seed(Gen3Rng* rng, uint32_t seed); /* seed 0 -> 0x6D2B79F5 */
  uint32_t gen3_rng_next(Gen3Rng* rng);
  #define GEN3_PID_MAX_TRIES 1000000u
  /* Finds a PID for species `national` (1..386) and OT ot_id whose traits equal want, after normalising want:
   * gender is ignored for ratios 0/254/255, ability_slot for one-ability species, unown_letter unless Unown.
   * false (and *pid untouched) after GEN3_PID_MAX_TRIES. */
  bool gen3_pid_find(Gen3Rng* rng, uint16_t national, uint32_t ot_id, const Gen3Traits* want, uint32_t* pid);
  /* GiveBoxMonInitialMoveset: walk the learnset entries with level <= `level` in order; skip a move already
   * in moves[]; else put it in the first empty slot; when all 4 are full, shift slots 1..3 down and put it in slot 3. */
  void gen3_default_moves(uint16_t national, uint8_t level, uint16_t moves[4]);
  ```

PID search, the part the tests don't pin down:
- Each try takes `r = gen3_rng_next()`, with `lo = r & 0xFFFF` and `hi = r >> 16`.
- If `want->shiny`, then `hi = TID ^ SID ^ lo ^ (gen3_rng_next() & 7)`, which makes the candidate shiny by construction.
- `pid = hi << 16 | lo`.
- **Accept the candidate when all of these hold:**
  - `pid % 25 == nature`;
  - the gender matches (when the ratio allows both);
  - `(pid & 1) == ability_slot` (two-ability species only);
  - `gen3_unown_letter(pid) == unown_letter` (Unown only);
  - `gen3_is_shiny(ot_id, pid) == want->shiny`.
- Expected tries are at most about 700 (Unown: 25 natures × 28 forms).

- [ ] **Step 1: Write the failing tests:**

```c
TEST(test_rng)              /* seed 1 -> first next() == 0x00042021; seed 0 -> state 0x6D2B79F5 */
TEST(test_pid_find_matrix)  /* seed 42; species 1 (31, one ability), 280 Ralts (127, two), 29 (254), 81 Magnemite (255, two), 201:
                               for every nature, shiny {0,1}, gender {M,F}, ability {0,1} (Unown: letters {0,25,26,27}):
                               returns true; gen3_traits re-derived from the pid (gender_of, is_shiny, nature, letter) equal the
                               normalised want; for 280 and 81, (pid & 1) == ability_slot */
TEST(test_pid_find_deterministic) /* same seed and want twice -> same pid */
TEST(test_default_moves)    /* Treecko 252: L5 {1,43,0,0}; L6 {1,43,71,0}; L30 {98,228,103,72};
                               Ivysaur 2: L5 {33,45,73,0}; L22 {22,77,79,75}; Unown L50 {237,0,0,0}; Bulbasaur L1 {33,0,0,0};
                               national 0 -> {0,0,0,0} */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement** as specified.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS, and the host run of `test_gen3_rules` takes under 2 s.
- [ ] **Step 5: Commit** `gen3: add PID search and default moveset`.

---

### Task 6: Editor model

**Files:**
- Create: `src/gen3/gen3_edit.h`, `src/gen3/gen3_edit.c`
- Test: `tests/test_gen3_edit.c`

**Interfaces:**
- Consumes: Tasks 2–5.
- Produces:
  ```c
  #define GEN3_EDIT_DEFAULT_NATIONAL 1
  #define GEN3_EDIT_DEFAULT_LEVEL 5
  #define GEN3_EDIT_DEFAULT_OT_NAME "FLIPPER"  /* same OT as plan 1's GEN3_TEST_TRAINER_*; plan 3 uses the player's */
  #define GEN3_EDIT_DEFAULT_TID 22222
  #define GEN3_EDIT_DEFAULT_SID 0
  #define GEN3_EDIT_MET_GAME 2                 /* Ruby */
  #define GEN3_EDIT_BALL 4                     /* POKE BALL */
  #define GEN3_EDIT_MAX_EV 85                  /* 6 x 85 = 510, the game's MAX_TOTAL_EVS */
  typedef enum { /* same order and values as EvIv in src/include/stat_nl.h */
      Gen3PresetRandIvZeroEv, Gen3PresetRandIvLevelEv, Gen3PresetRandIvMaxEv,
      Gen3PresetMaxIvZeroEv, Gen3PresetMaxIvLevelEv, Gen3PresetMaxIvMaxEv,
  } Gen3Preset;
  typedef struct Gen3Editor {
      Gen3Pkm pkm;
      Gen3Preset preset;
      bool preset_applied; /* false after gen3_edit_load until a preset or species is chosen */
      Gen3Rng rng;
  } Gen3Editor;
  void gen3_edit_init(Gen3Editor* ed, uint32_t seed);              /* Bulbasaur L5, default OT, preset RandIvZeroEv */
  void gen3_edit_load(Gen3Editor* ed, const Gen3Pkm* pkm);         /* takes it byte for byte; preset_applied = false */
  uint16_t gen3_edit_national(const Gen3Editor* ed);
  void gen3_edit_set_species(Gen3Editor* ed, uint16_t national);   /* D5; no-op if invalid or unchanged */
  void gen3_edit_set_level(Gen3Editor* ed, uint8_t level);         /* clamped 1..100; exp = exp_for_level */
  void gen3_edit_set_preset(Gen3Editor* ed, Gen3Preset preset);    /* applies IVs (re-rolls random) and EVs */
  void gen3_edit_set_nickname(Gen3Editor* ed, const char* ascii);  /* "" -> species name (game encoding) */
  void gen3_edit_set_ot_name(Gen3Editor* ed, const char* ascii);
  void gen3_edit_set_ot_id(Gen3Editor* ed, uint16_t tid, uint16_t sid); /* keeps shininess (D6) */
  void gen3_edit_set_item(Gen3Editor* ed, uint16_t item);
  bool gen3_edit_set_move(Gen3Editor* ed, uint8_t slot, uint16_t move); /* false: duplicate, or removing the last move */
  void gen3_edit_default_moves(Gen3Editor* ed);
  void gen3_edit_set_ball(Gen3Editor* ed, uint8_t ball);           /* 1..12 */
  void gen3_edit_set_pokerus(Gen3Editor* ed, uint8_t value);       /* raw byte: strain << 4 | days */
  bool gen3_edit_set_traits(Gen3Editor* ed, const Gen3Traits* want); /* new PID; false only if gen3_pid_find fails */
  const char* gen3_pokerus_status(uint8_t value);                  /* 0 "Clean"; days != 0 "Infected"; else "Cured" */
  ```

Rules:
- **Commit after every edit:** every mutator except `load` ends by setting the ability bit, calling `gen3_recalc_party`, and calling `gen3_pkm_fix_checksum`. The ability bit is `pid & 1` for two-ability species and 0 otherwise.
- **Presets:** applying a preset sets each IV to `(gen3_rng_next() >> 5s) & 31` (random) or 31, and each EV to 0, `85*level/100` or 85.
  - `set_level` re-applies the EVs only when `preset_applied` and the preset is a *LevelEv* one.
- **`set_move` with a move:** sets the move, sets PP to its base PP and clears that slot's 2 PP-Up bits.
- **`set_move` with 0:** clears the slot.
- **Packing:** then the moves are packed toward slot 0, carrying their PP and PP-Up bits along.
- **`set_traits`** only changes the PID, the ability bit, party stats and the checksum.
- **`set_ot_id`:** if the shininess would change, it calls `set_traits` with the old traits.

- [ ] **Step 1: Write the failing tests.** A test-local `static bool edit_consistent(const Gen3Editor* ed)` prints the first broken rule and returns false. Its rules:
  - checksum matches, HasSpecies 1, IsEgg/EggFlag/BadEgg 0;
  - the species maps to a national;
  - Level == level_for_exp, and the stats == calc_stats with Hp == MaxHp;
  - ability bit 0 for one-ability species;
  - moves packed, no duplicates, at least one.

  Every test ends with `ASSERT_TRUE(edit_consistent(&ed))`.

```c
TEST(test_edit_init_defaults)        /* seed 7: species 1, Level 5, Exp 135, nickname "BULBASAUR", OT "FLIPPER", OtId 22222, OtGender 0,
                                        moves {33,45,0,0}, PP {35,40,0,0}, item 0, Ball 4, MetLocation 0xFF, MetLevel 5, MetGame 2,
                                        Friendship 70, Language 2, Mail 0xFF, Status 0, EVs 0, not shiny, preset RandIvZeroEv applied */
TEST(test_edit_species_resets_keeps_ot) /* OT "ASH" 1234/4321, level 30, item 200, ball 1, pokerus 0x41; set_species(252):
                                        nickname "TREECKO", Level 30, Exp 21760, OT/ids/OtGender kept, item 0, Ball 4, Pokerus 0,
                                        moves {98,228,103,72}, MetLevel 30, Friendship 70, not shiny */
TEST(test_edit_species_stores_internal) /* set_species(252): Species == 277, raw[0x20..0x21] 15 01; set_species(386): Species 410 */
TEST(test_edit_species_same_is_noop) /* set nickname "LEAFY", set_species(current) -> raw unchanged */
TEST(test_edit_species_fateful)      /* 151 -> Fateful 1; 386 -> 1; 1 -> 0 */
TEST(test_edit_level)                /* level 50: Exp 117360; moves unchanged; MetLevel unchanged */
TEST(test_edit_level_clamped)        /* 0 -> Level 1 (Exp 1); 101 -> 100 */
TEST(test_edit_presets)              /* MaxIvMaxEv: IVs all 31, EVs all 85 (sum 510); MaxIvLevelEv at L50: EVs 42; RandIvZeroEv: EVs 0 */
TEST(test_edit_level_ev_follows_level) /* MaxIvLevelEv, level 100 -> EV 85; level 20 -> EV 17 */
TEST(test_edit_loaded_evs_kept)      /* load golden with EVs 252,252,6,0,0,0; preset MaxIvLevelEv; set_level(40) -> EVs unchanged */
TEST(test_edit_nickname_default)     /* species 29, set_nickname("") -> raw[0x08..0x11] C8 C3 BE C9 CC BB C8 B6 FF FF */
TEST(test_edit_set_move_pp)          /* PpBonuses 0xFF; set_move(2, 76) -> Move3 76, Pp3 10, PpBonuses 0xCF */
TEST(test_edit_set_move_packs)       /* {33,45,0,0}: set_move(0,0) -> {45,0,0,0}, Pp1 40; set_move(3,22) -> {45,22,0,0} */
TEST(test_edit_set_move_rejects)     /* {33,45,0,0}: set_move(1,33) false, unchanged; {33,0,0,0}: set_move(0,0) false */
TEST(test_edit_default_moves)        /* after set_move edits, default_moves -> gen3_default_moves(national, level) */
TEST(test_edit_traits_matrix)        /* Ralts: every nature x gender x shiny x slot -> set_traits true, traits_of == want */
TEST(test_edit_traits_static_gender) /* species 29, want Male -> true, gender Female */
TEST(test_edit_ability_bit_single_ability) /* species 1, 100 set_traits over natures: AbilityBit always 0, some pid odd */
TEST(test_edit_unown_letters)        /* species 201, letters 0..27 -> unown_letter(Pid) == letter */
TEST(test_edit_traits_preserve_other_bytes) /* load golden with Ribbons/markings/contest/PP Ups/pokerus/met data/EVs/IVs/item/
                                        unknown u16 all nonzero (checksum fixed); set_traits(nature 4): every byte equal except
                                        0x00..0x03, 0x1C..0x1D, 0x4B bit 7, 0x56..0x63; encrypt -> decrypt is true and identical */
TEST(test_edit_ot_id_keeps_shiny)    /* shiny true; set_ot_id(12345,54321) -> shiny, nature/gender/slot unchanged;
                                        shiny false; set_ot_id(22222,0) -> not shiny */
TEST(test_edit_ball_item_pokerus)    /* ball 12 -> 12; item 200 -> 200; pokerus 0x41 "Infected", 0x40 "Cured", 0 "Clean" */
TEST(test_edit_load_keeps_bytes)     /* load(GOLDEN_PK3) -> raw identical, no recalc */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement** `gen3_edit.{h,c}`. Species change builds a zeroed `Gen3Pkm` and copies in the kept fields (D5).
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add the Pokemon editor model`.

---

### Task 7: `.pk3` import/export and validation

**Files:**
- Create: `src/gen3/gen3_pk3.h`, `src/gen3/gen3_pk3.c`
- Test: `tests/test_gen3_pk3.c`

**Interfaces:**
- Consumes: Tasks 2–4.
- Produces:
  ```c
  typedef enum { Gen3CheckOk, Gen3CheckBadSize, Gen3CheckBadChecksum, Gen3CheckBadEgg, Gen3CheckEgg,
                 Gen3CheckBadSpecies, Gen3CheckBadMove, Gen3CheckBadItem, Gen3CheckFileError } Gen3Check;
  const char* gen3_check_text(Gen3Check c); /* "OK", "Not a .pk3 (size)", "Checksum mismatch", "Bad Egg",
                                               "Eggs not supported", "Unknown species", "Unknown move", "Unknown item",
                                               "Can't read file" */
  /* In order: BadEgg flag; IsEgg or EggFlag; species maps to no national; any move >= 355; item >= 377. */
  Gen3Check gen3_pkm_check(const Gen3Pkm* p);
  /* len 80: party fields derived (Status 0, Mail 0xFF, gen3_recalc_party). len 100: kept byte for byte.
   * Order: size, checksum, gen3_pkm_check. On Ok, HasSpecies is set to 1. *out is unchanged unless Ok. */
  Gen3Check gen3_pk3_import(const uint8_t* data, size_t len, Gen3Pkm* out);
  void gen3_pk3_export(const Gen3Pkm* p, uint8_t out[GEN3_BOX_SIZE]); /* raw[0..79], checksum recomputed */
  #define GEN3_PK3_NAME_MAX 16
  void gen3_pk3_default_name(const Gen3Pkm* p, char* out, size_t out_size); /* nickname, keep [A-Za-z0-9-], max 16, "" -> "POKEMON" */
  bool gen3_pk3_name_valid(const char* name); /* 1..16 chars of [A-Za-z0-9_-] */
  ```

- [ ] **Step 1: Write the failing tests:**

```c
TEST(test_import_80_golden)       /* GOLDEN_PK3[0..79] -> Ok; raw == GOLDEN_PK3 (100 bytes, party derived) */
TEST(test_import_100_keeps_bytes) /* GOLDEN_PK3 with Atk set to 999 -> Ok; raw identical (Atk still 999) */
TEST(test_export_roundtrip_every_byte) /* 100-byte input: markings 0x0F, unknown u16 0xBEEF, filler 0x1234, contest 1..6,
                                     Ribbons 0x7FFFFFFF, PpBonuses 0xE4, MetLocation 0x10, checksum fixed -> Ok; export == input[0..79] */
TEST(test_import_sizes)           /* len 0, 79, 81, 99, 101 -> BadSize; out unchanged */
TEST(test_import_bad_checksum)    /* GOLDEN_PK3 with raw[0x30] ^= 1 -> BadChecksum */
TEST(test_import_bad_species)     /* species 0, 252, 276, 412 (checksum fixed) -> BadSpecies */
TEST(test_import_egg)             /* IsEgg 1 -> Egg; EggFlag 1 -> Egg; BadEgg 1 -> BadEgg */
TEST(test_import_bad_move_item)   /* Move2 355 -> BadMove; HeldItem 377 -> BadItem (checksum fixed) */
TEST(test_import_sets_has_species)/* HasSpecies 0 -> Ok and HasSpecies 1 */
TEST(test_pk3_default_name)       /* "MR. MIME" -> "MRMIME"; "HO-OH" -> "HO-OH"; "NIDORAN\200" -> "NIDORAN"; "?!" -> "POKEMON" */
TEST(test_pk3_name_valid)         /* "TREECKO", "my_mon-1", 16 chars true; "", "a b", "a.pk3", 17 chars false */
TEST(test_check_text)             /* every Gen3Check has non-empty text; Gen3CheckEgg is "Eggs not supported" */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement** `gen3_pk3.{h,c}`.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add .pk3 import, export and validation`.

---

### Task 8: Gen III menu, editor menu, text/number/list scenes

**Files:**
- Create: `src/scenes/pokemon_gen3_edit.c`, `pokemon_gen3_text.c`, `pokemon_gen3_number.c`, `pokemon_gen3_pick.c`
- Create: `docs/gen3/editor-test.md`
- Modify: `src/scenes/pokemon_gen3_menu.c`, `src/scenes/include/pokemon_scene_config.h`, `src/include/pokemon_app.h`, `src/pokemon_app.c`

**Interfaces:**
- Consumes: Task 6 (`gen3_edit_*`, `gen3_traits_of`), Task 2 names; `stat_list` (`src/include/stat_nl.h`, which needs `named_list.h` first).
- Produces:
  - **Scene lines** (append, in this order):
    ```
    ADD_SCENE(pokemon, gen3_edit, Gen3Edit)
    ADD_SCENE(pokemon, gen3_text, Gen3Nickname)
    ADD_SCENE(pokemon, gen3_text, Gen3OtName)
    ADD_SCENE(pokemon, gen3_number, Gen3Level)
    ADD_SCENE(pokemon, gen3_number, Gen3Tid)
    ADD_SCENE(pokemon, gen3_number, Gen3Sid)
    ADD_SCENE(pokemon, gen3_pick, Gen3Preset)
    ADD_SCENE(pokemon, gen3_pick, Gen3Nature)
    ADD_SCENE(pokemon, gen3_pick, Gen3Ability)
    ADD_SCENE(pokemon, gen3_pick, Gen3Shiny)
    ADD_SCENE(pokemon, gen3_pick, Gen3Gender)
    ADD_SCENE(pokemon, gen3_pick, Gen3Ball)
    ADD_SCENE(pokemon, gen3_pick, Gen3Unown)
    ```
  - **Alias state:** before `next_scene`, the caller writes the target scene id into the scene state of `Gen3Nickname` (text scenes), `Gen3Level` (number scenes) or `Gen3Preset` (list scenes).
  - **`PokemonFap`** gains `struct Gen3Editor* gen3_editor;`. `pokemon_alloc` sets it to NULL.
  - **Gen III menu `on_enter`:** if `gen3_editor` is NULL, `malloc` it and run `gen3_edit_init(ed, furi_hal_random_get())`.
  - **`free_app`:** `free(gen3_editor)`.
  - **Gen III menu items:** "Edit Pokemon" (`Gen3Edit`), then "Link Test". `on_event` accepts both ids and still ignores other events.

Editor menu (`Gen3Edit`):
- **Layout:** a submenu with no header and the values aligned like `pokemon_gen.c`.
- **Highlight:** the item index is the target scene id, and the last chosen one is kept in `Gen3Edit`'s scene state so it stays highlighted.
- **Rows, in this final order** (*added by Task 9/10* rows come later):
  1. `Pokemon: <SPECIES>` (*Task 9*)
  2. `Nickname: <nick>`
  3. `Level: <n>`
  4. `Held Item: <name or None>` (*Task 10*)
  5. `Select Moves` (*Task 10*)
  6. `<stat_list name>` or `IVs/EVs: as loaded`
  7. `Nature: <NAME>`
  8. `Ability: <name of ability[bit]>`
  9. `Shiny: Yes|No`
  10. `Gender: Male|Female|Unknown`
  11. `Pokerus: <status>` (*Task 10*)
  12. `Unown Form: <A..Z ! ?>` (Unown only)
  13. `Ball: <item name>`
  14. `OT Name: <ot>`
  15. `OT TID: %05u`
  16. `OT SID: %05u`
- **No-op rows:** Ability does nothing for a one-ability species, and Gender does nothing when the ratio is 0/254/255, the same as Gen II gender.
- **Back** returns to the Gen III menu, and the Pokemon is kept for the app's lifetime.

Text scene (`gen3_text`):
- **Mechanics:** `text_input` with a validator. Buffers are static: nickname 11, OT 8.
- **Encodable check:** an input is accepted when decode(encode(x)) == x; otherwise `Unsupported\ncharacter!`.
- **Nickname:** "" means the default name. Max 10 characters.
- **OT name:** 1–7 characters; empty gives `Name can't\nbe empty!`.
- **Saving:** valid input is set through `gen3_edit_set_*`. The result callback sends `PokemonSceneBack`.

Number scene (`gen3_number`):
- Accepts digits only. Buffers are static: level 4, TID/SID 6.
- Level 1–100 (`Level must\nbe 1-100!`); TID and SID 0–65535 (`ID must\nbe 0-65535!`).
- TID/SID call `gen3_edit_set_ot_id` with the other half kept.

List scene (`gen3_pick`): a submenu with the current value pre-selected. Choosing an item applies it and sends `PokemonSceneBack`. Per state:
- **Preset:** the 6 `stat_list` names, applied with `gen3_edit_set_preset`. Add `_Static_assert((int)Gen3PresetMaxIvMaxEv == (int)MAXIV_MAXEV, "...")`.
- **Nature:** 25 entries `HARDY`, `LONELY +Atk -Def`, … (the stat names Atk Def Spe SpA SpD).
- **Ability:** `ability[0]` and `ability[1]` names.
- **Shiny:** "Shiny", "Not Shiny".
- **Gender:** "Female", "Male".
- **Ball:** `gen3_item_name(1..12)`.
- **Unown:** 28 letters.
- **Applying:** every PID trait goes through `gen3_traits_of` → change one field → `gen3_edit_set_traits`.

- [ ] **Step 1: Implement** the scenes, wiring and menu above. Follow `pokemon_gen.c`, `pokemon_name_input.c`, `pokemon_number_input.c` and `pokemon_stats.c` for the patterns, reading them rather than including or editing them.
- [ ] **Step 2: Build** with `ufbt` (release) and the dev SDK. Expected: 0 warnings on both.
- [ ] **Step 3: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 4: Size.** Run `arm-none-eabi-size dist/debug/pokemon_d.elf` and report text + data. The tables are now linked, so expect roughly 100,000–106,000. Stop and report if it is above 112,000.
- [ ] **Step 5: Write `docs/gen3/editor-test.md`.** Give it an intro (build with `ufbt` release, install `dist/pokemon.fap` with qFlipper or `ufbt launch`) and section **A. Editor basics**:
  1. Main menu → Gen III shows `Edit Pokemon` and `Link Test`.
  2. Edit Pokemon shows: Nickname BULBASAUR, Level 5, `Random IV, Zero EV`, a nature, Ability OVERGROW, Shiny No, Ball POKE BALL, OT FLIPPER, TID 22222, SID 00000.
  3. Nickname `Leafy` → the row shows `Leafy`. An empty nickname → `BULBASAUR`. Typing `_` → `Unsupported character!`.
  4. Level `0` and `101` are refused; `50` is accepted.
  5. Nature ADAMANT → the row updates, and Gender/Shiny are unchanged.
  6. Shiny Yes, then OT TID 12345 → Shiny still Yes.
  7. The Ability row on Bulbasaur does nothing.
  8. Back, then Edit Pokemon again → same values.
  9. Gen I and Gen II menus still open and show their editors (no regression).
- [ ] **Step 6: Commit** `gen3: add the Gen III editor menu and its text, number and list scenes`.

---

### Task 9: Species select view and scene

**Files:**
- Create: `src/views/gen3_species_select.h`, `src/views/gen3_species_select.c`, `src/scenes/pokemon_gen3_species.c`
- Modify: `src/scenes/include/pokemon_scene_config.h` (append `ADD_SCENE(pokemon, gen3_species, Gen3Species)`), `src/include/pokemon_app.h` (append `AppViewGen3SpeciesSelect` to `AppView`; field `void* gen3_species_select;`), `src/pokemon_app.c`, `src/scenes/pokemon_gen3_menu.c`, `src/scenes/pokemon_gen3_edit.c`, `docs/gen3/editor-test.md`

**Interfaces:**
- Produces:
  ```c
  typedef void (*Gen3SpeciesSelectCallback)(void* context, uint16_t national);
  void* gen3_species_select_alloc(ViewDispatcher* view_dispatcher, uint32_t view_id);
  void gen3_species_select_free(ViewDispatcher* view_dispatcher, uint32_t view_id, void* select);
  void gen3_species_select_start(void* select, uint16_t national, Gen3SpeciesSelectCallback cb, void* context);
  ```
- Lifetime: the view is allocated next to the editor in the Gen III menu `on_enter`, and freed in `free_app` before `view_dispatcher_free`. `pokemon_alloc` sets the field to NULL.

View:
- **Model:** `ViewModelTypeLocking`, with `uint16_t national`, `bool has_sprite` and `uint8_t sprite[400]` (the fxbm width, height and 392 data bytes).
- **Loading the sprite:**
  - When to load: in the enter callback and in the input callback after every change, never in the draw callback.
  - How: open `APP_ASSETS_PATH("all_sprites.fxbm")`, seek `(national-1)*404 + 4`, and read 400 bytes into the model inside `with_view_model`.
  - `has_sprite` is set only for national ≤ 251 with a full read.
- **Draw:**
  - The sprite with `canvas_draw_xbm(0, 0, 56, 56)`. Without one, `canvas_draw_rframe(4, 4, 48, 48, 4)` and a centered FontPrimary `?`.
  - `elements_frame(55, 0, 71, 18)` with `Select Pokemon` centered at (90, 5).
  - FontPrimary name at (58, 21); FontSecondary `#%03u` at (58, 33) and types `GRASS` or `GRASS/POISON` at (58, 43).
  - `elements_button_center("OK")`.
- **Input** (Press and Repeat):
  - Left/Right: −1/+1, wrapping between 1 and 386.
  - Down/Up: −10/+10, wrapping the same way as `select_pokemon.c`.
  - OK: `cb(national)`.
  - Back: not consumed.

Scene `Gen3Species`:
- `on_enter`: `gen3_species_select_start` with the current national, then switch to the view.
- The callback calls `gen3_edit_set_species` and sends `PokemonSceneBack`.
- `on_event`: `previous_scene`.
- Add the editor row `Pokemon: <SPECIES>`.

- [ ] **Step 1: Implement** the view, scene and wiring.
- [ ] **Step 2: Build** with `ufbt` (release) and the dev SDK. Expected: 0 warnings.
- [ ] **Step 3: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 4: Add section B. Species** to `editor-test.md`:
  1. #001 to #251 show sprites.
  2. #252 TREECKO shows the `?` frame and `GRASS`.
  3. Left from #001 goes to #386 DEOXYS; Up/Down jump by 10; holding Right scrolls.
  4. OK on TREECKO → the menu shows TREECKO with nickname TREECKO, the level unchanged and the OT unchanged.
  5. Set nickname `X`, then pick TREECKO again → the nickname is still `X`.
  6. Pick UNOWN → an `Unown Form` row appears; set `?` → the row shows `?`.
  7. Switch #251 ↔ #252 ↔ #001 20 times → no crash.
- [ ] **Step 5: Commit** `gen3: add the Gen III species picker`.

---

### Task 10: Moves, held item, Pokerus scenes

**Files:**
- Create: `src/scenes/pokemon_gen3_move.c`, `src/scenes/pokemon_gen3_item.c`, `src/scenes/pokemon_gen3_pokerus.c`
- Modify: `src/scenes/include/pokemon_scene_config.h` (append `ADD_SCENE(pokemon, gen3_moves, Gen3Moves)`, `ADD_SCENE(pokemon, gen3_move_letter, Gen3MoveLetter)`, `ADD_SCENE(pokemon, gen3_move_set, Gen3MoveSet)`, `ADD_SCENE(pokemon, gen3_item_letter, Gen3ItemLetter)`, `ADD_SCENE(pokemon, gen3_item_set, Gen3ItemSet)`, `ADD_SCENE(pokemon, gen3_pokerus, Gen3Pokerus)`), `src/scenes/pokemon_gen3_edit.c` (rows Held Item, Select Moves, Pokerus), `docs/gen3/editor-test.md`

**Interfaces:**
- Consumes: `gen3_edit_set_move`, `gen3_edit_default_moves`, `gen3_edit_set_item`, `gen3_edit_set_pokerus`, `gen3_pokerus_status`, `gen3_move_by_name`, `gen3_holdable_by_name`.
- Scene state:
  - `Gen3Moves` holds the slot (0–3) and also the highlight.
  - `Gen3MoveLetter` holds the letter.
  - `Gen3ItemLetter` holds the letter.

Moves (`Gen3Moves`, `Gen3MoveLetter`, `Gen3MoveSet`):
- **Moves screen:** `Move 1: <name or No Move>` × 4, then `Default moves`.
- **Letter screen:** `No Move` first, then one letter per distinct first character of `gen3_move_by_name`. `No Move` is left out when the slot holds the only move.
- **Move set screen:** the moves of that letter, leaving out moves already in another slot.
- **After choosing:** `gen3_edit_set_move`, then `scene_manager_search_and_switch_to_previous_scene(…, PokemonSceneGen3Moves)`.

Item (`Gen3ItemLetter`, `Gen3ItemSet`):
- `No Item`, then letters from `gen3_holdable_by_name`, then the items of that letter.
- After choosing: `gen3_edit_set_item`, then search back to `PokemonSceneGen3Edit`.

Pokerus (`Gen3Pokerus`):
- The `pokemon_pokerus.c` layout (Strain: None/A/B/C/D, Days 0–15, with Days hidden when the strain is None) and its strain mapping, written to `gen3_edit_set_pokerus`.
- The mapping: A = upper nibble 0x4, B = 0x1, C = 0x2, D = 0x3, None = byte 0.

- [ ] **Step 1: Implement** the scenes and rows.
- [ ] **Step 2: Build** with `ufbt` (release) and the dev SDK. Expected: 0 warnings.
- [ ] **Step 3: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 4: Add section C. Moves, item, Pokerus** to `editor-test.md`:
  1. Set species TREECKO and level 30 → Select Moves shows QUICK ATTACK, PURSUIT, SCREECH, MEGA DRAIN.
  2. Move 1 → `No Move` → PURSUIT moves up to Move 1, and Move 4 shows `No Move`.
  3. Letter `P` doesn't list PURSUIT while it's known.
  4. `Default moves` restores them.
  5. Held Item → `L` → LEFTOVERS → the row shows LEFTOVERS.
  6. `E` has no ENIGMA BERRY, and `M` has no MACH BIKE.
  7. Pokerus: strain A, days 3 → `Infected`; days 0 → `Cured`; None → `Clean`.
- [ ] **Step 5: Commit** `gen3: add Gen III move, held item and Pokerus scenes`.

---

### Task 11: `.pk3` files on the SD card

**Files:**
- Create: `src/gen3/gen3_pk3_furi.h`, `src/gen3/gen3_pk3_furi.c`, `src/scenes/pokemon_gen3_pk3.c`
- Modify: `src/scenes/include/pokemon_scene_config.h` (append `ADD_SCENE(pokemon, gen3_text, Gen3SaveName)`, `ADD_SCENE(pokemon, gen3_load, Gen3Load)`, `ADD_SCENE(pokemon, gen3_save, Gen3Save)`, `ADD_SCENE(pokemon, gen3_result, Gen3Result)`), `src/scenes/pokemon_gen3_text.c` (the SaveName state), `src/scenes/pokemon_gen3_menu.c`, `src/include/pokemon_app.h` (include `src/gen3/gen3_pk3.h`; field `char gen3_file_name[GEN3_PK3_NAME_MAX + 1];`), `docs/gen3/editor-test.md`

**Interfaces:**
- Consumes: Task 7, `gen3_edit_load`.
- Produces:
  ```c
  /* Resolves APP_DATA_PATH("gen3") (storage_common_resolve_path_and_ensure_app_directory) into out, so the
   * file browser, which runs outside this app's thread, gets /ext/apps_data/pokemon/gen3; creates it.
   * false if the SD card isn't ready. */
  bool gen3_pk3_dir(Storage* storage, FuriString* out);
  Gen3Check gen3_pk3_load_file(Storage* storage, const char* path, Gen3Pkm* out); /* FileError, BadSize unless 80/100, else import */
  bool gen3_pk3_save_file(Storage* storage, const char* path, const Gen3Pkm* p);   /* 80 bytes, FSOM_CREATE_ALWAYS */
  ```

Flows:
- **Gen III menu** items, in order: Edit Pokemon, Load .pk3 (`Gen3Load`), Save .pk3 (`Gen3SaveName`), Link Test. Plan 3 adds Trade.
- **Save .pk3:** fills `gen3_file_name` with `gen3_pk3_default_name`, then sets the text alias state to `Gen3SaveName`.
- **`Gen3SaveName`:**
  - A text input with buffer 17 and header `File name (.pk3)`. It is validated with `gen3_pk3_name_valid`, else `Use A-Z, 0-9,\n_ or -`.
  - On OK it sends a custom event, and `on_event` goes to `Gen3Save`.
- **`Gen3Load` `on_enter`:**
  1. Run `gen3_pk3_dir`; on failure, go to the result with `No SD card`.
  2. Show the file browser: `dialog_file_browser_set_basic_options(&o, ".pk3", NULL)`, `o.base_path = dir`, then `dialog_file_browser_show`.
  3. Cancel → `scene_manager_previous_scene`.
  4. A picked file → `gen3_pk3_load_file`.
  5. Ok → `gen3_edit_load`; the result is `Loaded` / `<NICK>\n<SPECIES> Lv. <n>`.
  6. Not Ok → the result is `Load failed` / `gen3_check_text`.
- **`Gen3Save` `on_enter`:**
  - The path is `<dir>/<name>.pk3`.
  - If the file exists: DialogEx `Overwrite?` / `<name>.pk3`, left `Cancel` (previous scene) and right `Overwrite`.
  - Otherwise, or after Overwrite: `gen3_pk3_save_file`.
  - The result is `Saved` / `<name>.pk3`, or `Save failed` / `Check the SD card`.
- **`Gen3Result`:** a DialogEx with a header, the text and center `OK`. OK and Back both run `scene_manager_search_and_switch_to_previous_scene(…, PokemonSceneGen3Menu)`.

- [ ] **Step 1: Implement** `gen3_pk3_furi.{h,c}`, the scenes and the menu items.
- [ ] **Step 2: Build** with `ufbt` (release) and the dev SDK. Expected: 0 warnings.
- [ ] **Step 3: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 4: Add section D. Files** to `editor-test.md`:
  1. Save .pk3 suggests the nickname → OK → `Saved`. The file `apps_data/pokemon/gen3/<name>.pk3` is 80 bytes (check with qFlipper).
  2. Save again with the same name → `Overwrite?`. Cancel keeps the old file.
  3. Change the level, then Load .pk3 that file → `Loaded`, and the Edit menu shows the saved level, nature, moves and OT.
  4. Copy any non-`.pk3` file to `gen3/` renamed to `x.pk3` → `Load failed: Not a .pk3 (size)`.
  5. With PKHeX (optional): open the saved file → no checksum warning, and species, level, nature, moves and OT match. Export a Gen III Pokemon from PKHeX as `.pk3` → it loads on the Flipper.
  6. Remove the SD card → Load .pk3 shows `No SD card`.
- [ ] **Step 5: Commit** `gen3: load and save .pk3 files on the SD card`.

---

### Task 12: Docs, final checks, user gate

**Files:**
- Modify: `README.md` (section "Gen III (experimental): Pokemon editor and .pk3 files"), `changelog.md` (under `## Unreleased`), `docs/gen3/editor-test.md`

README section:
- what the editor can change;
- that species 252–386 show a `?` image;
- where the files live (`apps_data/pokemon/gen3/`) and their format (PKHeX `.pk3`: saves 80 bytes, loads 80 or 100);
- that Max EV means 85 per stat (510 total);
- that trading comes in a later update.

`changelog.md` gets one bullet: "Gen III Pokemon editor and .pk3 load/save (experimental)".

- [ ] **Step 1: Write** the README section and the changelog bullet.
- [ ] **Step 2: Add section E. Memory and results** to `editor-test.md`:
  1. Open Gen III → Edit Pokemon. In qFlipper's CLI (or `ufbt cli`), run `free` and record `Free heap size`. It should be ≥ 20,000.
  2. A pass/fail table for sections A–E.
  3. On any failure, report the step, what the screen showed, and the `.pk3` file involved.
- [ ] **Step 3: Final checks:**
  - `make -C tests` passes.
  - `ufbt` (release) and the dev SDK build with 0 warnings.
  - `text + data` ≤ 112,000; put the number in the report.
  - `git diff a09d0c7 --stat -- src/pokemon_data.c src/pokemon_data_i.h src/pokemon_table.c src/pokemon_attribute.c src/pokemon_char_encode.c src/patch_list.c src/named_list.c src/move_nl.c src/item_nl.c src/stat_nl.c src/type_nl.c src/views/select_pokemon.c src/views/trade.c src/scenes/pokemon_menu.c src/scenes/pokemon_gen.c src/scenes/pokemon_select.c src/scenes/pokemon_name_input.c src/scenes/pokemon_number_input.c src/scenes/pokemon_move.c src/scenes/pokemon_item.c src/scenes/pokemon_stats.c src/scenes/pokemon_shiny.c src/scenes/pokemon_gender.c src/scenes/pokemon_pokerus.c src/scenes/pokemon_type.c src/scenes/pokemon_trade.c src/scenes/pokemon_exit_confirm.c src/scenes/pokemon_trade_reset_confirm.c` prints nothing.
- [ ] **Step 4: Commit** `docs: Gen III editor and .pk3 files`.
- [ ] **Step 5: Hand-off.** Push and give the user `dist/pokemon.fap` and `docs/gen3/editor-test.md`. **Stop here until the user reports results.**
  - **Memory:** if `free` reads under 20,000, or the app won't start, report the numbers. The fallback is D2's SD-asset tables, as a follow-up.
  - **Other failures:** fix them and retest only the failed sections.

---

## Spec coverage

| Requirement (spec) | Task |
|---|---|
| 80/100-byte format, PID/OT XOR, 24 orders, checksum | 3 |
| Western text for nicknames and trainer names | 1, 3 (existing `gen3_text`) |
| 386 species: internal ↔ national, base stats, types, gender ratio, abilities, growth | 2 |
| Sensible default moveset | 5, 6 |
| Full move, item, ability, nature lists | 2 |
| Stats from level, base, IVs, EVs, nature, matching the game | 4 |
| Editing: species, nickname, level | 6, 8, 9 |
| Editing: held item, moves | 6, 10 |
| Editing: IVs/EVs with Gen II presets | 6, 8 |
| Editing: shiny, gender, Pokerus | 6, 8, 10 |
| Editing: OT name, TID, SID | 6, 8 |
| Editing: nature, ability, Poke Ball | 6, 8 |
| Unown form | 5, 6, 8 |
| Call #2: PID matches the chosen traits | 5, 6 |
| Call #3: species change resets to defaults, keeps OT | 6 |
| Call #4, Phase 1 `.pk3`: save 80, load 80/100 | 7, 11 |
| Non-goal: placeholder image for 252–386 | 9 |
| Non-goal: no eggs | 7 (rejected on load) |
| Anti-goal: Pokemon load cleanly in-game | 3–7 tests, Global Constraints invariant |
| Anti-goal: don't break Gen I/II | Global Constraints; Task 8 checklist A.9; Task 12 step 3 |
| Constraint: host-testable core | 1–7 |
| Constraint: MIT, no copied code | Global Constraints, 2 |
| Budgets (stack, RAM) | Global Constraints, 2, 8, 12 |
