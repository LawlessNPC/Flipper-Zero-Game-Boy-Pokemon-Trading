# Gen III Ruby Cable Club Trade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A real Cable Club trade with English Ruby/Sapphire. The GBA player trades a party Pokemon to the
Flipper, changes it with the plan-2 editor (into a starter, say) and trades it back, as often as they like,
without leaving the Trade Center, without a link error and without ever risking the save.

**Architecture:** The plan-1 Cable Club state machine (`gen3_club`) learns the Part B sessions: trade menu
(0x1122), trade and link save (0x1144), room re-establish (0x1111). It takes each session's link type from
the GBA's own LinkPlayerBlock and arbitrates the trade menu as the master. The plan-1 fake Ruby grows Ruby's
child side of the same sessions, strict enough to trip on every timing and ordering rule of Part B. A trade
session owned by the app keeps the link running behind every Gen III screen. A pure sync module moves the
editor's Pokemon into and out of the club at safe moments, and a trade screen tells the user what to do on
the GBA.

**Tech Stack:** C11, Flipper SDK via `ufbt` (release SDK 1.5.1, API 88.2 per plan 1), `make` + host `cc`
for unit tests, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-10-25-gen3-gba-design.md`: Phase 1 (Cable Club trade), "The GBA
stays in the Trade Center while you edit" and its fallback, Goals → Phase 1, Anti-goals, calls #7 and #8.
Protocol: `docs/gen3/link-protocol-trade.md` (cited §B1.6, T2, …) and `docs/gen3/link-protocol.md` (§A2 …).
Plan 1: `docs/superpowers/plans/2026-10-25-gen3-01-link-connection-test.md` (and its amendments). Plan 2:
`docs/superpowers/plans/2026-10-25-gen3-02-pokemon-editor.md`.

## Plan sequence

This is plan 3 of 5.

1. Link connection test (done).
2. Gen III data core, editor and `.pk3` files (done before this plan starts).
3. **This plan:** Cable Club trade with Ruby/Sapphire.
4. FireRed/LeafGreen trades (protocol doc Part C). Ships v3.0.
5. Direct save access (multiboot helper, cartridge save read/write, ROM dump).

## Global Constraints

- C on the Flipper SDK with `ufbt`. Both SDK builds end with 0 warnings: `ufbt` (release) and `UFBT_HOME=/Users/kyvo/.claude/jobs/72354b36/tmp/ufbt-dev ufbt` (dev). CI builds both.
- MIT licensed. PKHeX (GPLv3) and the pret decompilations are references for behaviour, constants and layouts only; copy no code. The clones `/Users/kyvo/.claude/jobs/72354b36/tmp/pokeruby` (5784633c) and `.../pokeemerald` (731ad5bf) are read-only data: read them, never build or run anything from them.
- `make -C tests` builds every `tests/test_*.c` against all non-`_furi` `src/gen3/*.c` and `tests/fake_*.c`, and passes after every task.
- Gen I/II keep working exactly as before. No change to `src/pokemon_data.c`, `src/pokemon_data_i.h`, `src/pokemon_table.c`, `src/pokemon_attribute.c`, `src/pokemon_char_encode.c`, `src/patch_list.c`, `src/named_list.c`, `src/*_nl.c`, `src/views/select_pokemon.c`, `src/views/trade.c` or the Gen I/II scenes. Shared registration points change by appending only: `src/scenes/include/pokemon_scene_config.h`, `src/include/pokemon_app.h`, `src/pokemon_app.c`.
- Western games only. Never modify `lib/flipper-gblink`.
- Commits are authored by the user (LawlessNPC, Aleksandr Kofman) with **no** AI-attribution trailers: no `Co-Authored-By`, no `Claude-Session`, no "Generated with". `git log -1 --format='%(trailers)'` must print nothing.
- Stage only the files your task lists, by explicit path. Never `git add -A`, `git add .`, a directory, or `git commit -a`.
- Pull requests go to `LawlessNPC/Flipper-Zero-Game-Boy-Pokemon-Trading`. No `fap_version` bump: v3.0 comes at the end of plan 4.
- **Save safety on the wire.** CCDD only after the GBA's `AABB i` → the Flipper's `DDDD 0` → the GBA's `BBBB`, with no Flipper block in between, the offer unchanged since this exchange's round 1, and no checksum error counted in this 0x1122 session (§B3.4). DCBA only after the GBA's ABCD (§B4.3). Each GBA 2FFE is answered by exactly one 2FFE (§B5.3). The Flipper offers only a Pokemon that passes `gen3_trade_offer_check` (D8).
- **No link error in normal use.** The trade session stops only after the GBA has closed the link itself (room exit), while the GBA isn't linked, or through the "Stop link?" dialog, which is never offered between sync 1 and sync 3 (D10).
- **Pure-module rule:** every file in `src/gen3/` whose name doesn't end in `_furi.c`/`_furi.h` includes only `<stdint.h> <stdbool.h> <stddef.h> <string.h> <stdio.h>` and other pure `src/gen3/` headers. Flipper APIs live only in `*_furi.*` files and in `src/scenes/`, `src/views/`.
- **ISR rule** (plan 1): everything called from the link frame callback runs in interrupt context. No `malloc`/`free`, no `FURI_LOG_*`, no blocking calls, under 200 µs. `gen3_club_frame` decrypts at most one Pokemon per call (on the GBA's AABB) and copies at most 220 bytes per call.
- **App ↔ ISR rule:** app code never touches a `Gen3Club` directly. It uses `gen3_session_*`, which copy at most 100 bytes inside `primask = __get_PRIMASK(); __disable_irq(); … __set_PRIMASK(primask);`.
- **Fake independence:** `tests/fake_ruby.{h,c}` include only `<string.h> <stdbool.h> <stdint.h> <stddef.h>` and the byte-layout headers `mlink_types.h`, `gen3_link_player.h`, `gen3_pkm.h`, `gen3_text.h`. They never include `gen3_club.h`, `gen3_link.h`, `gen3_block.h`, `gen3_trade_block.h` or `gen3_trade_sync.h`, and build their own 20-byte blocks. Every fake behaviour names the pokeruby function it models, as plan 1's fake does.
- **Budgets:** app stack 2 KB, so no scene or view keeps a local buffer over 128 bytes. `sizeof(Gen3Club)` ≤ 2,600 bytes (host test `test_club_size`). `text + data` of `dist/debug/pokemon_d.elf` grows by at most 16,000 bytes over the Task 1 baseline and stays ≤ 128,000. On the device, free heap ≥ 12,000 with a trade session running and Edit Pokemon open. Measure with `$(ls -d ~/.ufbt/toolchain/*/bin)/arm-none-eabi-size dist/debug/pokemon_d.elf`.
- Use plan 2's interfaces exactly as plan 2 names them. If the code differs from a name this plan uses, stop and report.
- Includes use the root-relative style (`#include <src/gen3/gen3_club.h>`). Follow `.clang-format`, snake_case functions and CamelCase typedefs. Write multi-byte fields byte by byte, little-endian.
- Setup in a fresh checkout: `git submodule update --init`, then `ufbt update --channel=release` once.
- Implementers can't run on-device checks. UI tasks add their checklist sections to `docs/gen3/hardware-test-2.md`, and the user runs it at the gate (Task 14).

## Review Focus

1. **The user edits while the GBA sits in the trade menu or the confirm prompt.** This is the most common action. The GBA must never get a half-edited Pokemon, a Pokemon the Flipper no longer shows, or a CCDD for a stale offer, and one editing visit must cost one EEBB and one re-sit. *(Task 7: `test_trade_edit_in_menu_sends_eebb`, `test_trade_edit_after_dddd_never_ccdd`, `test_trade_two_round_trips`; Task 10: `test_sync_holds_while_editing`, `test_sync_pickup_before_offer`; Task 12 checklist D.)*
2. **The link breaks during Ruby's link save** (cable knock, flat battery, a Back press). The Flipper must neither keep a trade the GBA rolled back nor drop one it committed, and the UI must not let Back stop the link in that window. *(Task 6: `test_trade_lost_before_sync1`, `test_trade_lost_in_window`, `test_trade_lost_after_sync2`, `test_trade_save_window`; Task 13 checklist G.)*
3. **A Pokemon one side can't take.** The GBA offers an Egg, a Bad Egg or a Mail holder, or the Flipper holds Mail, the ENIGMA BERRY or an FRLG/Emerald-only item from a loaded `.pk3`. *(Task 1: `test_offer_check`, `test_check_incoming`; Task 5: `test_trade_refuses_egg`, `test_trade_refuses_mail`, `test_trade_refuses_bad_egg`; Task 10: `test_sync_bad_offer_unpublished`; Task 13 checklist F.3, H.3.)*
4. **Ruby drops or delays what the Flipper waits for:** an EEBB lost in the summary screen (T2), the GBA main loop stalling on map loads and flash writes so the round margins see a backlog (T1, T4), a one-sided checksum glitch inside a party block. *(Task 7: `test_trade_eebb_retry_after_summary`, `test_trade_checksum_glitch_resits`; Task 4: `test_trade_exchange_with_backlog`; Task 6: `test_trade_save_long_stalls`.)*
5. **Flipper UI work starving the background link** (the spec's named risk): text entry, sprite reads in the species picker, `.pk3` writes and the file browser while the GBA waits in the menu. Only the device can show it. *(Task 11 step 3: review that the frame callback gained no work outside `gen3_club_frame` and the PRIMASK sections copy ≤ 100 bytes; Task 12 checklist E, with the leave-and-return fallback.)*

## Design decisions

**D1. Session ownership.** `PokemonFap.gen3_session` stays the app's one session, in one of two modes.
- **Link Test** (plan 1, unchanged): created by the Link Test scene, freed when the Gen III menu shows again.
- **Trade:** created by Gen III → Trade and started once the cable reads OK. It survives every Gen III scene, including the editor, its text entry and number entry, the species picker, `.pk3` load/save and dialogs. It ends with End trade (a clean leave, D9), with Gen III menu Back while the GBA isn't linked, or with "Stop link?" after a leave that doesn't finish.
- While a trade session runs, the Gen III menu hides Link Test (TIM2 runs one driver) and shows End trade.
- **Spec fallback** (the background link can't hold its timing): the GBA player walks out of the Cable Club before editing and walks back in afterwards. The session simply waits in Waiting, so the fallback needs no code.

**D2. Link type from the GBA's block** (§A6 "Optional").
- On Connected the club sends `{2222, 1133}`; Ruby doesn't read word 1 (§A1).
- It sends its own LinkPlayerBlock only after the GBA's has arrived, with the same link type: 0x1133, 0x1122 or 0x1111. It uses 0x1144 only when a trade is pending; otherwise it sends 0x1133 and enters Error/Unexpected, so the GBA stalls before anything is written.
- Any other type gets 0x1133 and Error/OtherRoom, as in plan 1.
- The bytes are otherwise identical in every session (§B1.6 table). So a mismatch hang (§B8) can't come from the Flipper losing track of where the GBA is.
- Part A changes on the wire: the Flipper's block now follows the GBA's by ≈ 8 bursts. Checklist A re-verifies Part A at the gate.

**D3. Trade context survives relinks.**
- Per-session state resets on Disconnected, as in plan 1.
- The trade context does not reset: the GBA player (taken from a 0x1133 block), the offer, the Flipper mon the GBA was shown (`party_mon`), the GBA's party from the last exchange, the pending trade, the save syncs, the received Pokemon, `trades_done` and the National Dex flag.
- A GBA block of any type other than 0x1144 while a trade is pending drops it (Canceled): the GBA restarted or came back from the counter.

**D4. Arbiter policy** (§B3.4). The Flipper auto-accepts, like Gen I/II.
- An acceptable `AABB i` gets `DDDD 0`. The `BBBB` that follows gets `CCDD`.
- `AABB` for a slot `gen3_trade_check_incoming` refuses (Empty, Bad Egg, Egg, Mail, Item, BadData) gets `DDEE`, and the reason shows on the Flipper.
- `EEAA` gets `EEBB`. `BBCC` gets `DDEE`.
- The offer may be stale (no offer, or offer ≠ the Flipper mon shown in round 1), a checksum error may have been counted in this 0x1122 session, or a leave may be requested. Then every GBA block is answered with `EEBB`, and when nothing is pending an unsolicited `EEBB` goes out, resent every 180 bursts until the GBA's 5FFF (T2).
- Each reply goes out once, ≥ 6 bursts after the last chunk of the GBA's block (≥ 5, §B3.4).

**D5. Finalize at sync 3; link loss** (§B5.3).
- A pending trade becomes "done" only at the GBA's third save 2FFE. The club then holds the GBA's former Pokemon as its offer, so the automatic post-save exchange (§B6.1) shows it.
- If the link is lost first, the outcome depends on the 2FFE answers the club had queued:

  | Club's answers when lost | GBA after reboot | Flipper |
  |---|---|---|
  | none (animation, or before sync 1) | previous save, no trade | Canceled: keeps its Pokemon |
  | sync 1 only | "The save file is corrupt…", pre-trade save | RolledBack: keeps its Pokemon |
  | sync 2 too | committed (1–2 frames after our answer) | Assumed: takes the GBA's Pokemon, asks the user to check the GBA |

**D6. App ↔ club.**
- The app tick (100 ms, app thread) moves data both ways: it picks up a received Pokemon into the editor, hands it the GBA player and publishes the editor's Pokemon as the offer, in that order.
- It does this only while the current scene is Gen3Trade or Gen3Menu. A half-made edit is never exchanged, and one editing visit costs one EEBB.
- Every offer carries the count of trades the app has picked up. The club refuses an offer from an app that hasn't seen the latest trade, so a pre-trade editor can never overwrite the received Pokemon.
- Edit Pokemon and Load .pk3 are blocked from CCDD to the end of the trade; Edit is also hidden on the trade screen then.
- "After a completed trade, G re-exchanges automatically": that exchange carries the offer current at its round 1. The GBA isn't held on a black screen to wait for edits.

**D7. The GBA player as OT** (amends plan 2 D5 and spec call #3).
- Once the player is known, a species change and the untouched initial default take the player's OT: name bytes raw from the LinkPlayer, TID, SID, gender. The Pokemon then obeys and counts as theirs (spec Goals → Phase 1).
- Before the player is known, call #3 holds: the OT is kept.
- An editor row "OT: use <NAME>" applies the player's OT to any Pokemon, keeping shininess (plan 2 D6).
- The Flipper's own LinkPlayer stays FLIPPER / 22222 / SID 0 (plan 1), so the 0x1111 check (§B6.3) always passes.

**D8. Offer rules beyond plan 2's invariant** (§B2.5).
- No Mail: item 121–132, or mail byte ≠ 0xFF. A stale mail byte makes Ruby replace the held item (§B4.4 step 4).
- No ENIGMA BERRY (175).
- No item ≥ 349, Ruby's `ITEMS_COUNT`. A loaded `.pk3` can hold FRLG/Emerald key items.
- A Pokemon that fails these isn't offered at all. The club then has no offer: it doesn't sit, and from the menu it sends EEBB. The trade screen says why.
- The Flipper also refuses to receive Mail holders and Eggs, so it never has to keep mail data.

**D9. Leaving cleanly** (spec: "Ending the session leaves the Cable Club cleanly from the GBA side").
- End trade sets a leave request.
- In the room the club sends one CAFE 0017, which makes an idle GBA run its own exit (§B7 outline "Flipper-initiated leave"). A seated GBA cancels its seat first (§B1.3 row 1).
- In the menu the club sends EEBB first. During a trade or save it waits for the trade to finish.
- The session stops once the GBA has closed the link after its exit.

**D10. Save-window guard; National Dex.**
- From the club's sync-1 answer to sync 3 (`save_window`):
  - Back on the trade screen does nothing, and the guide says why.
  - End trade shows "The GBA is saving" with no End button.
  - "Stop link?" is never offered.
- National Dex is a per-run setting, default off (never persisted), and needs a confirm dialog to turn on. Only the 0x1144 block carries `lp_field_2 = 0x8000` (§B4.5).

**D11. One function decides the claimed game.** `gen3_club_claim_game` builds the Flipper's LinkPlayer and trainer card from the session's link type and the GBA's LinkPlayer. Plan 3 always claims Ruby; plan 4 changes only this function for FireRed/LeafGreen (§C1, §C2).

## File map

| File | Responsibility |
|---|---|
| `src/gen3/gen3_link_player.h` | modify: Part B link types, National Dex value |
| `src/gen3/gen3_trade_block.{h,c}` | 20-byte trade blocks, exchange rounds, offer/incoming checks |
| `src/gen3/gen3_club.{h,c}` | modify: claimed game, mirrored relinks, seat, exchange, arbiter, trade, save, EEBB, leave |
| `src/gen3/gen3_pkm.{h,c}`, `gen3_edit.{h,c}` | modify (plan 2 files): raw OT name, GBA player as OT |
| `src/gen3/gen3_trade_sync.{h,c}` | app-side sync of editor ↔ club, trade screen guide text |
| `src/gen3/gen3_session_furi.{h,c}` | modify: trade mode, offer/received handoff under PRIMASK |
| `src/views/gen3_trade.{h,c}` | trade screen view |
| `src/views/gen3_link_test.c` | modify: texts for the new phases |
| `src/scenes/pokemon_gen3_trade.c`, `pokemon_gen3_trade_end.c`, `pokemon_gen3_natdex.c` | trade scenes, app tick |
| `src/scenes/pokemon_gen3_menu.c`, `pokemon_gen3_edit.c`, `src/scenes/include/pokemon_scene_config.h`, `src/include/pokemon_app.h`, `src/pokemon_app.c` | wiring |
| `tests/fake_ruby.{h,c}` | modify: Ruby's Part B child side |
| `tests/club_bench.h`, `tests/trade_mons.h` | shared test bench and test Pokemon |
| `tests/test_gen3_trade_block.c`, `test_gen3_trade.c`, `test_gen3_trade_sync.c`, `test_gen3_club.c`, `test_gen3_pkm.c`, `test_gen3_edit.c` | host tests |
| `docs/gen3/hardware-test-2.md`, `README.md`, `changelog.md` | checklist and docs |

---

### Task 1: Prerequisites, trade blocks and offer rules

**Files:**
- Modify: `src/gen3/gen3_link_player.h` (append the constants below)
- Create: `src/gen3/gen3_trade_block.h`, `src/gen3/gen3_trade_block.c`
- Create: `tests/trade_mons.h`, `tests/test_gen3_trade_block.c`

**Interfaces:**
- Consumes (plan 2): `Gen3Pkm`, `GEN3_PARTY_SIZE`, `GEN3_MAIL_NONE`, `gen3_pkm_get/set`, `gen3_pkm_fix_checksum`, `gen3_pkm_encrypt`, `gen3_pkm_decrypt`, `gen3_pkm_check`/`Gen3Check` (`gen3_pk3.h`), `gen3_edit_init/set_species/set_level`, `tests/gen3_golden.h`. Also plan 1's `gen3_block_send`, `tests/link_helpers.h`.
- Produces (`gen3_link_player.h`):
  ```c
  #define GEN3_LINKTYPE_TRADE              0x1111 /* room again after the trade menu (§B0 row 1') */
  #define GEN3_LINKTYPE_TRADE_CONNECTING   0x1122 /* trade menu (§B1.6) */
  #define GEN3_LINKTYPE_TRADE_DISCONNECTED 0x1144 /* trade animation and link save (§B4.1) */
  #define GEN3_LP_NATIONAL_DEX             0x8000 /* lp_field_2 that makes Ruby unlock its National Dex (§B4.5) */
  ```
- Produces (`gen3_trade_block.h`):
  ```c
  #define GEN3_TRADE_BLOCK_SIZE 20              /* §B3.1 */
  #define GEN3_TRADE_ROUNDS 5                   /* §B2.1 */
  #define GEN3_TRADE_ROUND_MAX 220              /* largest round block */
  /* Trade-menu codes: word 0 of a 20-byte block, never word 0 of a burst (§B3) */
  #define GEN3_TRADE_READY       0xAABB /* GBA: TRADE on its slot arg */
  #define GEN3_TRADE_CANCEL      0xEEAA /* GBA: Cancel trade? YES */
  #define GEN3_TRADE_CONFIRM     0xBBBB /* GBA: Is this trade okay? YES */
  #define GEN3_TRADE_CONFIRM_NO  0xBBCC /* GBA: NO, or B */
  #define GEN3_TRADE_SET_MONS    0xDDDD /* Flipper: offer its slot arg */
  #define GEN3_TRADE_PARTNER_CANCEL 0xEECC
  #define GEN3_TRADE_PLAYER_CANCEL  0xDDEE /* "The trade has been canceled." */
  #define GEN3_TRADE_BOTH_CANCEL    0xEEBB /* back to the room */
  #define GEN3_TRADE_START       0xCCDD
  #define GEN3_TRADE_ANIM_DONE   0xABCD /* GBA, end of the animation (§B4.3) */
  #define GEN3_TRADE_ANIM_ACK    0xDCBA /* Flipper, after the GBA's ABCD */
  #define GEN3_ITEM_MAIL_FIRST   121  /* ITEM_ORANGE_MAIL, pokeruby include/constants/items.h */
  #define GEN3_ITEM_MAIL_LAST    132  /* ITEM_RETRO_MAIL */
  #define GEN3_ITEM_ENIGMA_BERRY 175
  #define GEN3_RUBY_ITEMS_COUNT  349  /* ITEMS_COUNT in Ruby; 349..376 are FRLG/Emerald items */
  /* Words: 0 = code, 1 = arg, 2..9 = 0; little-endian. */
  void gen3_trade_block_build(uint16_t code, uint16_t arg, uint8_t out[GEN3_TRADE_BLOCK_SIZE]);
  /* Word 0; word 1 into *arg unless arg is NULL. in holds GEN3_TRADE_BLOCK_SIZE bytes. */
  uint16_t gen3_trade_block_code(const uint8_t* in, uint16_t* arg);
  /* Round 1..5 of §B2.1: CCCC request type 1,1,1,3,4 and block size 200,200,200,220,40.
   * Any other round: 0xFFFF and 0. */
  uint16_t gen3_trade_round_request(uint8_t round);
  uint16_t gen3_trade_round_size(uint8_t round);
  /* The Flipper's block for round 1..5, gen3_trade_round_size(round) bytes: round 1 is mon (game
   * form; NULL -> zeros) then 100 zero bytes (the empty slot 1, §B2.3); every other round zeros. */
  void gen3_trade_round_block(uint8_t round, const uint8_t* mon, uint8_t* out);
  typedef enum {
      Gen3RefuseNone,
      Gen3RefuseEmpty,   /* species 0 */
      Gen3RefuseBadEgg,  /* checksum mismatch or Bad Egg flag */
      Gen3RefuseEgg,
      Gen3RefuseMail,    /* held item 121..132, or mail byte != 0xFF */
      Gen3RefuseItem,    /* held item 175, or >= 349 */
      Gen3RefuseBadData, /* species, move or item outside what gen3_pkm_check accepts */
  } Gen3Refuse;
  /* May the Flipper offer p (decrypted)? In order: gen3_pkm_check (BadEgg -> BadEgg, Egg -> Egg,
   * other failures -> BadData), mail, item. */
  Gen3Refuse gen3_trade_offer_check(const Gen3Pkm* p);
  /* May the Flipper take the GBA's mon (game form)? decrypt fails -> BadEgg; species 0 -> Empty;
   * else gen3_trade_offer_check. scratch receives the decrypted mon. */
  Gen3Refuse gen3_trade_check_incoming(const uint8_t game[GEN3_PARTY_SIZE], Gen3Pkm* scratch);
  ```
- Produces (`tests/trade_mons.h`, test only, all `static inline`):
  ```c
  void tm_make(uint16_t national, uint8_t level, uint32_t seed, uint8_t out[GEN3_PARTY_SIZE]); /* plan-2 editor, encrypted */
  void tm_make_egg(uint8_t out[GEN3_PARTY_SIZE]);      /* tm_make(1,5,1) with IsEgg 1 and EggFlag 1, checksum fixed */
  void tm_make_mail(uint8_t out[GEN3_PARTY_SIZE]);     /* tm_make(1,5,2) with HeldItem 121 and Mail 0 */
  void tm_make_bad_egg(uint8_t out[GEN3_PARTY_SIZE]);  /* tm_make(1,5,3), then out[0x30] ^= 1 */
  ```

- [ ] **Step 1: Check the prerequisites.** Plan 2 must be in the branch.
  - Run `make -C tests`; it must pass.
  - Confirm every plan-2 name in Consumes exists with the signature plan 2 gives. If one differs, stop and report BLOCKED with the difference.
  - Run `ufbt`, then `arm-none-eabi-size dist/debug/pokemon_d.elf` (path in Global Constraints). Put `text + data` in your report: it is the size baseline for this plan.
- [ ] **Step 2: Write the failing tests** in `tests/test_gen3_trade_block.c`. Here and below, `golden_game()` stands for the 100 bytes plan 2's `golden_game(out)` writes.

```c
TEST(test_link_type_values)   /* 0x1111, 0x1122, 0x1133, 0x1144; GEN3_LP_NATIONAL_DEX == 0x8000 */
TEST(test_trade_block_layout) /* build(0xDDDD, 3) -> DD DD 03 00 then 16 x 00; code() == 0xDDDD, arg 3; code(in, NULL) works */
TEST(test_trade_block_on_wire)/* connect(); gen3_block_send(that block, 20, 0); 3 bursts carry {BBBB,0014,0080,0,0,0,0,0},
                                 {8888,DDDD,0003,0,0,0,0,0}, {8888,0,0,0,0,0,0,0} (§B3.1) */
TEST(test_trade_rounds)       /* request(1..5) == 1,1,1,3,4; size == 200,200,200,220,40; request(0), (6) == 0xFFFF, size 0 */
TEST(test_trade_round_block)  /* golden_game() as mon: round 1 [0..99] == it, [100..199] all 0; NULL -> 200 zeros;
                                 rounds 2, 3: 200 zeros; 4: 220 zeros; 5: 40 zeros (§B2.3) */
TEST(test_offer_check)        /* golden_build + fix: None. HeldItem 200, 348: None. 121, 132: Mail. Mail 0: Mail. 175, 349: Item.
                                 IsEgg 1: Egg. BadEgg 1: BadEgg. Species 252 (internal, checksum fixed): BadData. Move2 355: BadData */
TEST(test_check_incoming)     /* golden_game(): None and scratch == GOLDEN_PK3. 100 zero bytes: Empty.
                                 golden_game() with [0x30] ^= 1: BadEgg. golden_game() with [0x55] = 0: Mail.
                                 tm_make_egg: Egg. tm_make_mail: Mail. tm_make_bad_egg: BadEgg. tm_make(252,5,9): None */
```

- [ ] **Step 3: Run** `make -C tests`. Expected: compile failure, `gen3_trade_block.h` not found.
- [ ] **Step 4: Implement** the constants, `gen3_trade_block.{h,c}` and `tests/trade_mons.h`.
- [ ] **Step 5: Run** `make -C tests`. Expected: every test passes.
- [ ] **Step 6: Commit** `gen3: add trade blocks, exchange rounds and trade offer checks` (stage the five files).

---

### Task 2: Test bench, claimed game, own LinkPlayerBlock after the GBA's

**Files:**
- Create: `tests/club_bench.h`
- Modify: `tests/test_gen3_club.c` (helpers moved to `club_bench.h`; new tests)
- Modify: `src/gen3/gen3_club.h`, `src/gen3/gen3_club.c`
- Modify: `src/views/gen3_link_test.c` (phase and error texts)

**Interfaces:**
- Consumes: Task 1; plan 1's club, link, block and link-player modules.
- Produces (`gen3_club.h`):
  ```c
  typedef struct {
      const char* name; uint16_t tid; uint16_t sid; uint8_t gender; /* as plan 1 */
      bool trading; /* new, last. false (Link Test) never sits down. Plan 1's 4-value initializers keep it false */
  } Gen3ClubConfig;
  /* Gen3ClubPhase, appended (log values): Gen3ClubPhaseExchange = 7, Gen3ClubPhaseMenu = 8,
   * Gen3ClubPhaseTrade = 9, Gen3ClubPhaseSave = 10. Gen3ClubError, appended: Gen3ClubErrorUnexpected = 4. */
  /* Gen3ClubStatus, appended: */
      uint32_t link_type;    /* the GBA's link type this session; 0 until its LinkPlayerBlock arrives */
      bool trainer_valid;    /* trainer: the GBA's LinkPlayer from the last 0x1133 linkup, kept across */
      Gen3LinkPlayer trainer;/* disconnects and relinks */
  /* The one place that decides which game the Flipper claims to be (D11). Fills the Flipper's
   * LinkPlayer for a session of link_type, and its trainer card. Plan 3 always claims Ruby:
   * version GEN3_VERSION_RUBY, name cfg->name (7 chars, EOS-filled to 11), trainer_id sid<<16|tid,
   * gender 0/1, id 0, language GEN3_LANGUAGE_ENGLISH, card gen3_trainer_card_build (§A5). lp_field_2
   * is GEN3_LP_NATIONAL_DEX only for GEN3_LINKTYPE_TRADE_DISCONNECTED with national_dex set, else 0.
   * partner (the GBA's LinkPlayer of this session) is unused until plan 4 claims FireRed/LeafGreen
   * for an FRLG partner (§C1, §C2). */
  void gen3_club_claim_game(
      const Gen3ClubConfig* cfg, const Gen3LinkPlayer* partner, uint32_t link_type,
      bool national_dex, Gen3LinkPlayer* lp, uint8_t card[GEN3_TRAINER_CARD_BLOCK_SIZE]);
  ```
- Produces (`tests/club_bench.h`, all `static inline`):
  - Move plan 1's `Bench`, `bench_init`, `ruby_relink`, `count_sent`, `step`, `run`, `club_status`, `run_until_phase`, `at_cards_or_later`, `run_until_room`, `reach_room` and `leave_room` here unchanged. Mark each `static inline`, so a test file that doesn't use one doesn't trip `-Wunused-function`.
  - `Bench` gains:
    ```c
    uint32_t bursts;                     /* frames with tx.count == 9 so far */
    MlinkTx last_tx;                     /* the Flipper's tx of the latest step */
    Gen3BlockRx f_rx, g_rx;              /* the bench's own parse of each side's blocks (tx / rx words 1..8) */
    BenchBlock f_blocks[96], g_blocks[96]; /* every completed block of each side, in order */
    uint8_t n_f_blocks, n_g_blocks;
    uint16_t f_cccc[16]; uint32_t f_cccc_burst[16]; uint8_t n_f_cccc; /* the Flipper's CCCC types and bursts */
    uint32_t g_cafe_seen;                /* bit k: the GBA sent CAFE k (k < 32) */
    uint32_t sent_cafe16, f_cafe16_burst, g_cafe16_burst; /* Flipper CAFE 0016 count; latest 0016 burst per side */
    ```
    with `typedef struct { uint16_t size, code, arg; uint32_t first_burst, last_burst; } BenchBlock;`. `code`/`arg` are the block's words 0 and 1; `first_burst` is the header's burst and `last_burst` the last chunk's.
  - `step()` updates them each frame. Each side's block parse is fed only from non-zero command words.
  - `size_t bench_codes(const BenchBlock* blocks, uint8_t n, uint16_t* out, size_t max)`: the codes of the 20-byte blocks, in order. In the tests below, `f_codes` and `g_codes` are its output for each side, and "the latest LinkPlayerBlock" is the latest size-60 entry.

Club behaviour (D2):
- On Connected: queue `{2222, 1133}` only (plan 1 also queued its own block here).
- When the GBA's LinkPlayerBlock completes:
  - Bad magic: Error/BadPlayerBlock (plan 1).
  - Otherwise store it as the partner and set `link_type`.
  - G's type 0x1133: `trainer = partner`, `trainer_valid` → Players.
  - Every other type in this task: Error/OtherRoom. Tasks 3, 6 and 7 add 0x1122, 0x1144 and 0x1111.
  - In every case, queue the Flipper's own block from `gen3_club_claim_game`, with G's type for the trade types and 0x1133 otherwise. The GBA always gets a block back, so it never waits forever for one.
- Plan 1's card request already waits for the Flipper's queue to empty, so the §A6 margin now counts from the Flipper's last chunk.
- `link_type` resets to 0 on Disconnected. `trainer` / `trainer_valid` never reset.
- `gen3_link_test.c` phase texts:
  - Exchange `trade menu (loading)`, Menu `trade menu`, Trade `trading`, Save `GBA saving`.
  - Error/Unexpected `error: unexpected`.

- [ ] **Step 1: Move the bench.** Create `tests/club_bench.h` and make `test_gen3_club.c` include it. Run `make -C tests`: the 15 plan-1 club tests pass unchanged.
- [ ] **Step 2: Write the failing tests** (in `test_gen3_club.c`):

```c
TEST(test_club_own_block_after_gba) /* the first burst after Connected carries {2222,1133,0..} (word 1 unread by Ruby, §A1);
                                       the Flipper's LinkPlayerBlock first_burst > the GBA's last_burst; reach_room; fake error NULL */
TEST(test_club_trainer_kept)        /* reach_room: trainer_valid; trainer TID 12345, SID 54321, name "RUBY"; link_type 0x1133.
                                       leave_room: Waiting, partner_valid false, link_type 0, trainer_valid still true.
                                       a fake named "SAPH" (TID 1, SID 2) links: trainer "SAPH", 1, 2 */
TEST(test_club_claim_game)          /* cfg {"FLIPPER",22222,0,0}: (0x1144, nat on) -> version 0x4002, lp_field_2 0x8000,
                                       trainer_id 22222, link_type 0x1144, id 0, language 2, gender 0,
                                       name C0 C6 C3 CA CA BF CC FF FF FF FF; (0x1144, off) -> lp_field_2 0; (0x1122, on) -> 0;
                                       the 60-byte blocks of (0x1133, off) and (0x1122, on) differ only in bytes 0x24..0x27;
                                       card == gen3_trainer_card_build(&lp) */
```

- [ ] **Step 3: Run** `make -C tests`. Expected: the 3 new tests fail.
- [ ] **Step 4: Implement** the config field, enums, status fields, `gen3_club_claim_game`, the mirrored own block and the view texts.
- [ ] **Step 5: Run** `make -C tests`. Expected: all pass, including every plan-1 club and fake test.
  - The Flipper's own block now goes out ≈ 8 bursts later. If a plan-1 test's frame budget (such as `run_until_room(b, 400)`) is too tight only because of that, raise that budget by at most 20 frames and list each change in your report.
  - No other change to a plan-1 assertion is allowed.
- [ ] **Step 6: Build** `ufbt` and the dev SDK. Expected: 0 warnings (the Link Test view's phase `switch` covers the new values).
- [ ] **Step 7: Commit** `gen3: club: claim Ruby in one place, send our LinkPlayerBlock after the GBA's`.

---

### Task 3: Sitting down and the 0x1122 relink (§B1)

**Files:**
- Modify: `tests/fake_ruby.h`, `tests/fake_ruby.c`
- Modify: `src/gen3/gen3_club.h`, `src/gen3/gen3_club.c`
- Modify: `tests/club_bench.h` (`bench_offer`, `bench_mute`, `bench_raw_relink`)
- Create: `tests/test_gen3_trade.c`

**Interfaces:**
- Produces (`fake_ruby.h`):
  ```c
  /* FakeRubyPhase, appended: */
  FakeRubySeated,  /* seat script: "Please wait..." until the trade starts or the seat is left */
  FakeRubyMenu,    /* 0x1122: relink, party exchange, trade menu */
  FakeRubyTrade,   /* 0x1144: relink, trade animation, ABCD/DCBA */
  FakeRubySave,    /* link save */
  FakeRubyReturn,  /* after EEBB: back to the room through the 0x1111 re-establish */
  void fake_ruby_sit(FakeRuby* g, uint8_t seat);   /* room: walk onto seat 0 (4,5) or 1 (7,5); no effect while on a seat */
  void fake_ruby_stand_up(FakeRuby* g);            /* "Please wait...": the player presses B */
  uint8_t fake_ruby_link_state(const FakeRuby* g, uint8_t slot); /* 0x80 IDLE, 0x81 BUSY, 0x82 READY, 0x83 EXITING_ROOM */
  uint32_t fake_ruby_relink_frames(const FakeRuby* g); /* latest OpenLink -> gReceivedRemoteLinkPlayers; UINT32_MAX while waiting */
  /* fake_ruby_error adds: "link timeout", "link type" */
  ```
- Produces (`gen3_club.h`):
  ```c
  /* Sets (game != NULL) or clears the offer: the Flipper's one party Pokemon, game form. false and
   * nothing changed if trades_seen != the club's trades_done (D6; trades_done is 0 until Task 6). */
  bool gen3_club_set_offer(Gen3Club* club, const uint8_t* game, uint32_t trades_seen);
  /* Gen3ClubStatus, appended: */
      bool has_offer;
      bool gba_seated;      /* this room session: the GBA sent CAFE 0016 and hasn't sent 001D/001A since */
      bool flipper_seated;  /* this room session: the Flipper sent CAFE 0016 */
      uint32_t trades_done; /* finalized trades since init; stays 0 until Task 6 */
  ```
- Produces (`club_bench.h`):
  - `bench_init_trade(Bench*)`: `bench_init(b, 0x1133)`, but with club config `{"FLIPPER", 22222, 0, 0, true}` and the default offer `tm_make(252, 5, 21)`.
  - `bench_to_room(Bench*)`: `fake_ruby_open_link`, then `run_until_room(b, 420)`; it doesn't re-init (plan 1's `reach_room` does).
  - `bench_offer(Bench*, const uint8_t mon[100])`: `gen3_club_set_offer(&b->club, mon, status.trades_done)`.
  - `bench_mute(Bench*, bool)`: while muted, `step()` doesn't call the club and gives the fake a tx with `count = 0`, so the GBA sees VBlanks and no transfers.
  - `bench_raw_relink(Bench*, const Gen3LinkPlayer* lp, uint32_t frames)`: for `frames` frames a fresh `Gen3Link` is the master instead of the club. It handshakes; on Connected it queues `{2222, 1133}` and `gen3_block_send` of `lp`'s 60-byte block once, then idles.

Fake behaviour. Each point models the cited Ruby code and must be as strict as it:
1. **Seat event** (§B1.1, `sub_8054FF8` → `sub_8055648`):
   - `fake_ruby_sit` acts on the next game update in the room, if the local player is IDLE and `on_seat` is false.
   - The local player then becomes BUSY, the key intercept becomes DeferToEventScript (0x11, controls locked), the seat task starts, and `on_seat` becomes true.
   - Standing on a seat never re-triggers it (`fake_ruby_resit`, Task 7, steps off and back on).
2. **Seat task** (`sub_80839DC`):
   - Case 0 lasts 30 game updates while "Please wait..." prints (the reading time, not a protocol value).
   - Case 1 sets `me.id = seat` (`sub_8007270`) and the key intercept to SetReady: one 0x16, then Ready (`sub_805546C`), which sends 0x11, or one 0x1D after `fake_ruby_stand_up` while own state is READY.
   - Case 2 evaluates `sub_80554F8` every update in the §B1.3 order.
3. **Cancel** (case 3, `sub_8055588`): the key intercept becomes DeferToEventScript. The controls unlock at the next update, and that intercept then sends one 0x1A, which restores SelfIdle (plan 1's 0x1A handling).
4. **Start** (`sub_8083AAC`, §B1.5):
   - Case 0: callback None (`ClearLinkCallback_2`), so no more CAFE and no more lockstep skipping.
   - A 17-frame fade.
   - Case 2: SetCloseLinkCallback, which acts only if the callback is None (`link.c`).
   - Case 3: once closed, wait 3 frames, then `OpenLink` with `me.link_type = 0x1122` and the 300-frame task (§B1.6).
5. **Trade menu creation** (`sub_8047EC0` states 2–4):
   - State 2: 12 frames.
   - State 3 (child): player count ≥ 2.
   - State 4: wait for `gReceivedRemoteLinkPlayers` and every link type equal to slot 0's, then go to state 5 (Task 4). Types that differ set error "link type" (Ruby hangs silently, §B8).
6. **300-frame task** (`sub_8083C50`): more than 300 frames after OpenLink (0x1122 here; 0x1111 in Task 7) without `gReceivedRemoteLinkPlayers`: close, error "link timeout".
7. **Phases:** Seated from the seat event to the close; Menu from the 0x1122 OpenLink on.
   - A handshake success sets FakeRubyLinkup only on the counter's 0x1133 link.
   - Part B tasks run every main-loop frame whether or not the link is open, for the countdowns between CloseLink and OpenLink.

Club behaviour:
- **Room**, if `trading && has_offer && !flipper_seated`: on the GBA's CAFE 0016, queue one CAFE 0016 (§B1.4) and set `flipper_seated`. After that, send zero bursts only (no CAFE 0011, T8, §B1.5).
- The GBA's CAFE 0016 sets `gba_seated`; its CAFE 001D or 001A clears it. The Flipper stays READY if the GBA stands up (T7: staying READY is legal).
- **G's 0x1122 block:** with `trading`, the own block is 0x1122 → Exchange. The rounds come in Task 4; the club idles until then. Without `trading`: own block 0x1133 → Error/Unexpected.
- Seat flags are per room session.

- [ ] **Step 1: Extend the fake and the bench, and write the failing tests** in `tests/test_gen3_trade.c`. Unless a test says otherwise, every test in this file starts with `bench_init_trade`, and "reach_room" means `bench_to_room`. The fake is as in plan 1 (RUBY, 12345/54321).

```c
TEST(test_trade_sit_relinks_menu) /* reach_room; sit(1): sent_cafe16 == 1 and f_cafe16_burst > g_cafe16_burst;
                                     every last_tx word 1..8 from then until the Flipper's 5FFF answer is 0 (§B1.5);
                                     fake phase Menu, relink_frames < 100; club Exchange, link_type 0x1122;
                                     fake_ruby_partner(): link_type 0x1122, trainer_id 22222, name "FLIPPER", lp_field_2 0; error NULL */
TEST(test_trade_waits_for_gba)    /* reach_room; 1200 frames: sent_cafe16 == 0; fake Room */
TEST(test_trade_link_test_never_sits) /* plain bench_init (trading false) + bench_offer: sit(0); 600 frames: sent_cafe16 == 0;
                                         fake Seated; error NULL */
TEST(test_trade_no_offer_never_sits)  /* bench_init_trade, then gen3_club_set_offer(&b.club, NULL, 0): same */
TEST(test_trade_stand_up)         /* no offer; sit(0); 60 frames; stand_up: g_cafe_seen has bits 0x1D and 0x1A; fake Room;
                                     link_state(1) == 0x80; gba_seated false */
TEST(test_trade_relink_timeout_trips_fake) /* sit(1); at fake phase Menu bench_mute(true): fake error "link timeout",
                                              300..310 frames after the OpenLink */
TEST(test_trade_link_type_mismatch_trips_fake) /* sit(1); at fake phase Menu: bench_raw_relink with the club's own
                                              LinkPlayer but link_type 0x1133, 200 frames: fake error "link type" */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail; every older test passes.
- [ ] **Step 3: Implement** the fake behaviour and the club behaviour above.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: sit down with the GBA and relink for the trade menu`.

---

### Task 4: Party exchange (§B2)

**Files:**
- Modify: `tests/fake_ruby.h`, `tests/fake_ruby.c`, `src/gen3/gen3_club.h`, `src/gen3/gen3_club.c`, `tests/club_bench.h`, `tests/test_gen3_trade.c`

**Interfaces:**
- Produces (`fake_ruby.h`):
  ```c
  /* The GBA's party, game form, slots 0..count-1 (also its saved party: it saved at the counter, §A6). */
  void fake_ruby_set_party(FakeRuby* g, const uint8_t party[][GEN3_PARTY_SIZE], uint8_t count);
  const uint8_t* fake_ruby_party(const FakeRuby* g, uint8_t slot, bool saved);
  const uint8_t* fake_ruby_enemy(const FakeRuby* g, uint8_t slot); /* the Flipper's party from the latest exchange */
  uint8_t fake_ruby_enemy_count(const FakeRuby* g);                /* CalculateEnemyPartyCount */
  typedef enum {
      FakeRubyMenuNone,       /* not in the trade menu */
      FakeRubyMenuLoading,    /* relink, exchange, building the menu */
      FakeRubyMenuBrowsing,   /* states 0/1: the player can act */
      FakeRubyMenuSummary,    /* Task 7 */
      FakeRubyMenuStandby,    /* states 5/100: "Link standby...", waiting for the Flipper */
      FakeRubyMenuConfirming, /* states 7/14: after DDDD, before YES/NO */
      FakeRubyMenuPrompt,     /* state 3: "Is this trade okay?" YES/NO */
      FakeRubyMenuMessage,    /* state 8: canceled / friend wants to trade; needs A */
      FakeRubyMenuClosing,    /* states 10-13 or 11-12: fading out to the trade or the room */
  } FakeRubyMenuState;
  FakeRubyMenuState fake_ruby_menu_state(const FakeRuby* g);
  void fake_ruby_stall(FakeRuby* g, uint8_t frames); /* main loop skips frames; VBlank and serial IRQs run (T4) */
  /* fake_ruby_error adds: "stale block", "exchange stalled", "bad egg", "bad offer", "empty party" */
  ```
- Produces (`gen3_club.h`):
  ```c
  #define GEN3_CLUB_ROUND1_DELAY_BURSTS 10 /* idle bursts between both LinkPlayerBlocks and round 1's CCCC (§B2.2: >= 8) */
  #define GEN3_CLUB_ROUND_DELAY_BURSTS 6   /* between both blocks of a round and the next CCCC (§B2.2: >= 5) */
  void gen3_club_set_round_delays(Gen3Club* club, uint8_t first, uint8_t later); /* tests only */
  void gen3_club_gba_mon(const Gen3Club* club, uint8_t slot, uint8_t out[GEN3_PARTY_SIZE]); /* from the last exchange */
  /* Gen3ClubStatus, appended: */  uint8_t round; /* Exchange/Menu: the round in progress or done, 1..5 */
  ```
- Produces (`club_bench.h`):
  - `Bench` gains `uint8_t party[6][GEN3_PARTY_SIZE]; uint8_t party_count;`, the GBA's party as set.
  - `bench_set_party(Bench*, const uint8_t party[][GEN3_PARTY_SIZE], uint8_t count)` sets it and the fake's.
  - `bench_init` calls it with `{tm_make(1,5,11), tm_make(4,5,12), tm_make(7,10,13)}`; `ruby_relink` sets the fake's again.
  - `bench_reach_menu(Bench*)`, on an initialised bench (so a test can set the party first): `bench_to_room`, `fake_ruby_sit(1)`, then run until `fake_ruby_menu_state == Browsing` (≤ 600 frames). Returns false on a fake error.

Fake behaviour:
1. **`sub_8048D44` cases 0–16, as written** (§B2.1):
   - Copy cases 0/4/7/10/13 fill gBlockSendBuffer: party[0..1], [2..3], [4..5]; 220 bytes of mail (zeros: this fake has no mail); the 11 gift-ribbon bytes (zeros), then stale.
   - Case 1 waits for IsLinkTaskFinished, then ResetBlockReceivedFlags if any flag is set.
   - The master-only cases just advance.
   - The wait cases need `GetBlockReceivedStatus() == 0b11` (the Flipper's block and the GBA's own echo). They then copy the Flipper's block (rounds 1–3: 200 bytes into enemy[2k..2k+1]; round 4: 216; round 5: 11) and reset the flags.
2. **CCCC** (ProcessRecvCmds):
   - Types 0, 1, 3 and 4 start `InitBlockSend(gBlockSendBuffer, 200/200/220/40)`, refused while a block is active. Type 2 keeps plan 1's card rule.
   - Error "stale block" when a CCCC-requested block's first chunk is built while gBlockSendBuffer doesn't hold that round's copy. Ruby would send stale bytes (U3).
3. **Error "exchange stalled"** when a wait case (1, 3, 6, 9, 12, 15) isn't left within 600 frames. This is the §B2.2 deadlock; on Ruby it's a black screen.
4. **After case 16** (state 6), CalculateEnemyPartyCount (§B2.5):
   - Walk enemy slots 0..5 until an all-zero or species-0 slot.
   - A slot that fails `gen3_pkm_decrypt`: "bad egg".
   - A decrypted species outside internal 1–251 and 277–411, or an item ≥ 349 or 175: "bad offer".
   - Count 0: "empty party".
   - Then 32 frames to build the menu (states 6–21 and the fade-in), then callback1 → Browsing.
5. **`fake_ruby_stall`:** LinkMain1/LinkMain2 and the game skip `frames` frames, so the receive queue backs up.

Club behaviour:
- **Start.** The exchange starts once the club's own block has left (queue empty) and the GBA's has been parsed. After `first` idle bursts (default 10), for each round r = 1..5:
  - At round 1, `party_mon = offer`.
  - Queue `{CCCC, gen3_trade_round_request(r)}` and `gen3_trade_round_block(r, r == 1 ? party_mon : NULL)` (17 or 18 commands; the queue holds 24).
  - Feed the GBA's block. A size other than `gen3_trade_round_size(r)` → Error/Unexpected. Rounds 1–3 go into GBA slots `2(r−1)` and `2(r−1)+1`.
- **Between rounds.** When both blocks of round r are complete, wait `later` idle bursts (default 6), then start round r+1.
- **End.** After round 5: Menu.

- [ ] **Step 1: Write the failing tests:**

```c
TEST(test_trade_exchange_parties)  /* bench_reach_menu, Browsing within 300 frames of the 0x1122 OpenLink; enemy_count 1;
                                      enemy(0) == offer; enemy(1..5) all 0; club Menu, round 5; gba_mon(0..2) == b.party, (3..5) all 0;
                                      Flipper CCCC words, in order: 0001 0001 0001 0003 0004, once each; Flipper block headers, in order:
                                      {BBBB 00C8 0080} x3, {BBBB 00DC 0080}, {BBBB 0028 0080}; fake error NULL */
TEST(test_trade_exchange_margins)  /* f_cccc_burst[0] - the later last_burst of both latest LinkPlayerBlocks >= 11;
                                      f_cccc_burst[r] - the later last_burst of both round-r blocks >= 7, r = 1..4 */
TEST(test_trade_round1_delay_needed) /* set_round_delays(0, 6): fake error "exchange stalled" within 900 frames of sitting */
TEST(test_trade_exchange_with_backlog) /* the bench calls fake_ruby_stall(4) each time status.round changes:
                                      Browsing, enemy(0) == offer, fake error NULL */
TEST(test_trade_bad_offer_trips_fake)/* offer = golden_game() with [0x30] ^= 1 (bypassing every check): fake error "bad egg" */
TEST(test_club_size)                 /* sizeof(Gen3Club) <= 2600 */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail.
- [ ] **Step 3: Implement** the fake and club behaviour.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: exchange parties for the trade menu`.

---

### Task 5: Trade menu arbiter (§B3)

**Files:**
- Modify: `tests/fake_ruby.h`, `tests/fake_ruby.c`, `src/gen3/gen3_club.h`, `src/gen3/gen3_club.c`, `tests/club_bench.h`, `tests/test_gen3_trade.c`

**Interfaces:**
- Produces (`fake_ruby.h`):
  ```c
  void fake_ruby_offer(FakeRuby* g, uint8_t slot); /* Browsing: select own slot, TRADE */
  void fake_ruby_cancel_trade(FakeRuby* g);        /* Browsing: CANCEL, "Cancel trade?" YES */
  void fake_ruby_answer(FakeRuby* g, bool yes);    /* "Is this trade okay?": applied once the prompt is up */
  void fake_ruby_press_a(FakeRuby* g);             /* Message: dismiss */
  /* fake_ruby_error adds: "unexpected dddd", "bad ccdd", "ccdd hang" */
  ```
- Produces (`gen3_club.h`):
  ```c
  #define GEN3_CLUB_REPLY_DELAY_BURSTS 6 /* idle bursts between the last chunk of the GBA's block and the reply (§B3.4: >= 5) */
  typedef enum {
      Gen3ClubMenuIdle,     /* no GBA request pending */
      Gen3ClubMenuOffered,  /* DDDD sent: the GBA is confirming */
      Gen3ClubMenuStarting, /* CCDD sent */
      Gen3ClubMenuLeaving,  /* EEBB sent */
  } Gen3ClubMenu;
  /* Gen3ClubStatus, appended: */
      Gen3ClubMenu menu;
      uint8_t gba_slot;     /* arg of the GBA's latest AABB */
      Gen3Refuse refused;   /* reason the latest AABB got DDEE; None after a DDDD */
      bool trade_pending;   /* CCDD sent; until done or lost (Task 6) */
  ```
- Produces (`club_bench.h`): `bench_inject_code(Bench*, uint16_t code, uint16_t arg)`, which queues a 20-byte block on `b->club.link` (`gen3_block_send`, sender 0) as if the club sent it.

Fake behaviour:
1. **callback1** (`sub_80494D8` → `sub_80491E4`, §B3.3) runs every frame once the menu is up. It reads and resets each set slot-0 flag; slot-1 flags (its own echo) are only reset. A flag set before callback1 starts persists until then.
   - **EEBB:** 17-frame fade, then state 11, SetCloseLinkCallback, 5FFF, close, phase Return. Task 7 adds the way back to the room.
   - **EECC, DDEE:** Message.
   - **DDDD:** partner index = arg + 6, Confirming. Then 20 frames of slide-in and 121 frames of `DisplayMessageAndContinueTask` → Prompt.
   - **CCDD:** 17-frame fade, then state 10: store (own cursor, partner index) and SetCloseLinkCallback. If the callback is busy, SetCloseLinkCallback is ignored and Ruby hangs in state 13: error "ccdd hang". Otherwise state 13 → closed → phase Trade (Task 6).
   - Other codes are ignored.
2. **Strictness:**
   - DDDD unless this GBA sits in Standby after its own AABB: "unexpected dddd".
   - CCDD unless this GBA's latest blocks were AABB, then (after the Flipper's DDDD) BBBB, with no Flipper block since: "bad ccdd".
   - DDDD while enemy_count is 0: "empty party".
3. **This GBA's 20-byte blocks** (`Trade_SendData`) keep `linkData[10]` between sends, so words 2–9 are stale (§B3.1).
4. **Player actions:**
   - `fake_ruby_offer`: in Browsing, sends AABB slot → Standby. If no other party slot holds a Pokemon, nothing is sent ("That's your only POKéMON", §B3.2).
   - `fake_ruby_cancel_trade`: in Browsing, sends EEAA 5 frames later (QueueAction(5, 0)) → Standby.
   - `fake_ruby_answer`: in Prompt, YES sends BBBB (only if the callback is None) and NO sends BBCC → Standby.
   - `fake_ruby_press_a`: Message → Browsing.

Club behaviour (D4):
- Feed every GBA block in Menu. Answer each completed 20-byte block once, after `GEN3_CLUB_REPLY_DELAY_BURSTS` idle bursts:
  - **`AABB i`** in Idle: `refused = i < 6 ? gen3_trade_check_incoming(gba slot i, &scratch) : Gen3RefuseEmpty`.
    - Refused: `DDEE`.
    - Otherwise: `DDDD 0` → Offered, `gba_slot = i`, `refused = None`.
  - **`BBBB`:** in Offered, `CCDD` → Starting, `trade_pending`, `pending_mon = gba slot gba_slot`. In Idle, `DDEE`.
  - **`BBCC`:** `DDEE` → Idle.
  - **`EEAA`:** `EEBB` → Leaving.
  - **Anything else** (0000 debug, …): no reply.
- In Starting and Leaving, GBA blocks get no reply. A 5FFF goes to Closing (plan 1).

- [ ] **Step 1: Write the failing tests** (each starts from `bench_reach_menu`):

```c
TEST(test_trade_offer_confirm_ccdd) /* offer(2): g_codes {AABB 2}; one Flipper DDDD 0, first_burst >= AABB last_burst + 7;
                                       fake Confirming then Prompt; answer(yes): g BBBB; one CCDD, >= 7 bursts after BBBB's last chunk;
                                       f_codes == {DDDD 0, CCDD}; fake Closing then phase Trade; status: menu Starting, trade_pending, gba_slot 2;
                                       fake error NULL */
TEST(test_trade_gba_says_no)        /* offer(2), answer(no): BBCC -> DDEE; fake Message; press_a -> Browsing; f_codes == {DDDD, DDEE};
                                       menu Idle; trade_pending false */
TEST(test_trade_gba_cancels)        /* cancel_trade: g {EEAA} -> f {EEBB}; the Flipper answers G's 5FFF; fake phase Return;
                                       club Closing, then Waiting; no CCDD */
TEST(test_trade_refuses_egg)        /* G party {A, egg, X}: offer(1) -> f {DDEE}, refused Egg; no DDDD; press_a; offer(0) -> DDDD, refused None */
TEST(test_trade_refuses_mail)       /* G slot 1 = tm_make_mail: offer(1) -> DDEE, refused Mail */
TEST(test_trade_refuses_bad_egg)    /* G slot 1 = tm_make_bad_egg: offer(1) -> DDEE, refused BadEgg */
TEST(test_trade_ccdd_rule_trips_fake)   /* Browsing: bench_inject_code(CCDD, 0): fake error "bad ccdd" */
TEST(test_trade_unasked_dddd_trips_fake)/* Browsing: bench_inject_code(DDDD, 0): fake error "unexpected dddd" */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail.
- [ ] **Step 3: Implement** the fake and club behaviour.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: arbitrate the trade menu as the master`.

---

### Task 6: The trade, the link save and finalizing (§B4, §B5, §B6.1)

**Files:**
- Modify: `tests/fake_ruby.h`, `tests/fake_ruby.c`, `src/gen3/gen3_club.h`, `src/gen3/gen3_club.c`, `tests/club_bench.h`, `tests/test_gen3_trade.c`

**Interfaces:**
- Produces (`fake_ruby.h`):
  ```c
  uint32_t fake_ruby_saved_trades(const FakeRuby* g); /* link saves committed */
  bool fake_ruby_save_corrupt(const FakeRuby* g);     /* the link failed between sync 1 and the commit (§B5.2) */
  bool fake_ruby_national_dex(const FakeRuby* g);     /* saved National Dex flag */
  void fake_ruby_set_sector_stall(FakeRuby* g, uint8_t frames); /* main-loop stall per flash sector; default 2 (T4) */
  /* fake_ruby_error adds: "early dcba", "flipper mail", "stray 2ffe" */
  ```
- Produces (`gen3_club.h`):
  ```c
  typedef enum {
      Gen3TradeResultNone,
      Gen3TradeResultDone,       /* sync 3 seen */
      Gen3TradeResultCanceled,   /* lost before the club answered sync 1: nothing written on the GBA */
      Gen3TradeResultRolledBack, /* lost after the sync-1 answer, before the sync-2 answer: the GBA reports a
                                    corrupt save and loads the one from before the trade */
      Gen3TradeResultAssumed,    /* lost after the sync-2 answer, before sync 3: taken as done (D5) */
  } Gen3TradeResult;
  void gen3_club_set_national_dex(Gen3Club* club, bool on);
  /* The Pokemon of the latest finalized trade (game form; zeros before any) and trades_done. */
  void gen3_club_get_received(const Gen3Club* club, uint8_t out[GEN3_PARTY_SIZE], uint32_t* trades_done);
  /* Gen3ClubStatus, appended: */
      uint8_t save_syncs;   /* GBA 2FFEs answered in this link save, 0..3 */
      bool save_window;     /* sync 1 answered, sync 3 not yet seen: don't stop the link (D10) */
      Gen3TradeResult last_result;
  ```
- Produces (`club_bench.h`): `bench_trade(Bench*, uint8_t gba_slot)`, from Browsing: `fake_ruby_offer`, then `fake_ruby_answer(yes)` once the prompt is up, then run until `trades_done` grows and the fake is Browsing again (≤ 4000 frames). Returns false on a fake error.

Fake behaviour:
1. **The 0x1144 relink** (§B4.1). After state 13 closes:
   - CloseLink and type 0x1144 (+1 frame), OpenLink (+2; no 300-frame task).
   - 61 frames, then state 3 (child) → 4: exchange complete, link types equal, else "link type".
   - 25 frames of pictures and fade, then a 600-frame animation (T16).
   - `sub_804D63C` runs all along: a slot-0 DCBA before this GBA sent ABCD sets "early dcba" (§B4.3).
2. **TradeMons** (§B4.4) on (own cursor i, partner j = (DDDD arg + 6) % 6):
   - The Flipper's mon has mail ≠ 0xFF: "flipper mail".
   - Swap party[i] ↔ enemy[j].
   - If the received mon isn't an egg: Friendship 70 (`gen3_pkm` decrypt, set, encrypt).
   - The Flipper's 0x1144 `lp_field_2 == 0x8000`: National Dex pending.
   - Then the ABCD block, once and only if the callback is None (`sub_804BB78`), then wait for the slot-0 DCBA.
3. **Link save** (`sub_804DC88`, §B5.1):
   - State 0 (1 frame).
   - State 1, **sync 1** (SetLinkStandbyCallback); states 100/101 until it completes.
   - State 2, then 50.
   - 14 × (state 51 for 5 frames, then state 52: one sector, with a main-loop stall of `sector_stall` frames).
   - State 4 (+ stall), then state 40 for 51 frames.
   - State 41, **sync 2**. State 42 when it completes: **commit** (saved party = party, `saved_trades++`, saved National Dex = pending).
   - State 5: 61 frames, then **sync 3**.
   - State 6: 17-frame fade; state 7: 20-frame music fade; state 8: SetCloseLinkCallback.
   - State 9 when closed, then 4 frames → 0x1122 OpenLink with the 300-frame task. The menu of Tasks 4–5 follows with the post-trade parties (§B6.1).
4. **Save corruption:** a link error ("lag", "checksum") or power-off after sync 1 completes and before the commit sets `save_corrupt`. The saved party stays as it was (§B5.2).
5. **Error "stray 2ffe":** a Flipper 2FFE processed while this GBA's callback isn't Standby/StandbyForAll. Such a flag would let a later sync finish without the Flipper (§B5.3).

Club behaviour (D3, D5):
- **The GBA's 0x1144 block:**
  - With `trade_pending`: own block 0x1144 with `lp_field_2` from `gen3_club_claim_game(…, national_dex, …)` → Trade.
  - Without: own 0x1133 → Error/Unexpected.
  - Any other GBA block type while `trade_pending`: drop the pending trade, `last_result = Canceled`, then handle the block as usual.
- **Trade:** zero bursts. On the GBA's ABCD block, after the reply delay, one DCBA → Save with `save_syncs = 0`.
- **Save:** each GBA 2FFE gets one 2FFE and `save_syncs++`. At 1: `save_window`. At 3: **finalize**:
  - `received = pending_mon`, `trades_done++`;
  - `party_mon = offer = pending_mon`, `has_offer`;
  - `trade_pending = false`, `save_window = false`, `last_result = Done`.
- **Disconnected:**
  - In Trade, or in Save with `save_syncs` 0: Canceled.
  - In Save with 1: RolledBack.
  - In Save with 2: Assumed, plus finalize.
  - Each clears `trade_pending` and `save_window`.
- **`gen3_club_set_offer`** returns false when `trades_seen != trades_done`.

- [ ] **Step 1: Write the failing tests.** Each starts with `bench_init_trade` and `bench_reach_menu`. The G party is the bench default `{A, B, X = tm_make(7,10,13)}`, the offer is `Y = tm_make(252,5,21)`, and `Z = tm_make(255,12,31)`.

```c
TEST(test_trade_round_trip)       /* bench_trade(2): saved_trades 1, save_corrupt false; saved party[2] decrypts to Y with Friendship 70
                                     and every other byte of the decrypted Y (checksum refixed); saved party[0..1] unchanged;
                                     club trades_done 1, last_result Done, get_received == X; the post-save exchange shows enemy(0) == X;
                                     club Menu; f_codes == {DDDD, CCDD, DCBA}; exactly 3 Flipper 2FFE from DCBA to the GBA's last 5FFF;
                                     fake error NULL */
TEST(test_trade_dcba_after_abcd)  /* DCBA first_burst >= the GBA's ABCD last_burst + 7 */
TEST(test_trade_save_window)      /* status every frame: save_window false until save_syncs == 1, true while 1..2, false from 3;
                                     trades_done 0 -> 1 in the frame save_syncs becomes 3 */
TEST(test_trade_national_dex_off) /* fake_ruby_partner()->lp_field_2 == 0 when the fake is in Room (0x1133), Menu and Trade;
                                     fake_ruby_national_dex false after the commit */
TEST(test_trade_national_dex_on)  /* set_national_dex(true): Menu 0, Trade 0x8000; fake_ruby_national_dex true after the commit */
TEST(test_trade_lost_before_sync1)/* fake_ruby_power_off in the animation: club Waiting, Canceled, trade_pending false, trades_done 0;
                                     fake saved party == b.party */
TEST(test_trade_lost_in_window)   /* at save_syncs == 1, 4 more frames, then bench_mute(true) 30 frames: fake error "lag", save_corrupt true;
                                     unmute 60 frames: club Waiting, RolledBack, trades_done 0; saved party == b.party, saved_trades 0 */
TEST(test_trade_lost_after_sync2) /* at save_syncs == 2, 4 more frames (assert saved_trades == 1), power_off: club Assumed,
                                     trades_done 1, get_received == X */
TEST(test_trade_save_long_stalls) /* set_sector_stall(10): bench_trade(2) succeeds, save_corrupt false */
TEST(test_trade_early_dcba_trips_fake) /* fake phase Trade, during the animation: bench_inject_code(DCBA, 0): fake error "early dcba" */
TEST(test_trade_extra_2ffe_trips_fake) /* in the frame save_syncs becomes 1, plus 3 frames: inject {2FFE}: fake error "stray 2ffe" */
TEST(test_club_offer_guard)       /* after bench_trade: set_offer(Z, 0) false, offer unchanged; set_offer(Z, 1) true */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail.
- [ ] **Step 3: Implement** the fake and club behaviour.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: trade, answer the link save and keep the received Pokemon`.

---

### Task 7: Editing between trades: EEBB, the 0x1111 room, re-sitting (§B6.3, §B7)

**Files:**
- Modify: `tests/fake_ruby.h`, `tests/fake_ruby.c`, `src/gen3/gen3_club.h`, `src/gen3/gen3_club.c`, `tests/test_gen3_trade.c`

**Interfaces:**
- Produces (`fake_ruby.h`):
  ```c
  void fake_ruby_resit(FakeRuby* g);         /* room: step off the seat (Up) and back on (Down) */
  void fake_ruby_open_summary(FakeRuby* g);  /* Browsing: SUMMARY of a Pokemon */
  void fake_ruby_close_summary(FakeRuby* g); /* back to Browsing */
  /* fake_ruby_error adds: "relink mismatch" */
  ```
- Produces (`gen3_club.h`):
  ```c
  #define GEN3_CLUB_EEBB_RETRY_BURSTS 180 /* resend an EEBB not followed by the GBA's 5FFF (T2: ~3 s) */
  /* Gen3ClubStatus, appended: */
      bool offer_stale;   /* Menu: the GBA's view of the Flipper's party is out of date (no offer, offer
                             changed, or a checksum error in this 0x1122 session): it gets EEBB */
      bool resit_needed;  /* Room via 0x1111: the GBA's avatar is on its seat; it must step off and back on */
  ```

Fake behaviour:
1. **EEBB, back to the room** (§B6.3): 17-frame fade, then state 11 (SetCloseLinkCallback), 5FFF, close. Then `CB2_ReturnToFieldFromMultiplayer`:
   - Every link state becomes IDLE (`sub_8054F70`).
   - After 17 frames, OpenLink 0x1111 with the 300-frame task.
   - After ≥ 10 frames, the child task waits for the exchange; types that differ set "link type".
   - **`CheckLinkPlayersMatchSaved`:** the trainerId and the name (compared up to the first EOS) of slots 0–1 must equal the copies saved at the 0x1133 linkup, else close + "relink mismatch". The fake now saves both LinkPlayers when its Part A exchange completes (`sub_80081C8`, called from `sub_80831F8`).
   - Then StartSendingKeysToLink, 17 frames of fade-in with the controls locked (0x11), then Room. The avatar is still on its seat (`on_seat` stays true).
2. **`fake_ruby_resit`:** in Room with the local player IDLE, 16 game updates holding 0x13 (Up), then 16 holding 0x12 (Down). The steps themselves aren't simulated. Then the seat event of Task 3 point 1.
3. **Summary screen** (T2):
   - `fake_ruby_open_summary` in Browsing → Summary.
   - A slot-0 EEBB handled while the summary is open is lost: `sub_80484F4` resets the menu state.
   - `fake_ruby_close_summary` → Browsing.

Club behaviour (D4):
- **The GBA's 0x1111 block:** own block 0x1111 → Room once it's queued. There's no walk-in and no 2FFE on this path: the GBA starts CAFE at once. Set `resit_needed`; the GBA's next CAFE 0016 or a 0x1133 linkup clears it.
- **Stale or tainted.** In Menu, `stale = !has_offer || offer ≠ party_mon`, and `tainted` = a checksum error was counted in this 0x1122 session. `offer_stale = stale || tainted` (Task 8 adds `|| leave`). While it's set:
  - every GBA block, AABB and BBBB included, gets `EEBB` (→ Leaving) and never DDDD or CCDD;
  - with nothing pending, an unsolicited EEBB goes out `GEN3_CLUB_REPLY_DELAY_BURSTS` after the later of the end of round 5 and the GBA's last block chunk;
  - an EEBB without the GBA's 5FFF is resent every `GEN3_CLUB_EEBB_RETRY_BURSTS`.

- [ ] **Step 1: Write the failing tests.** Offer Y, Z = `tm_make(255,12,31)`, G party as in Task 6.

```c
TEST(test_trade_edit_in_menu_sends_eebb) /* bench_reach_menu; set_offer(Z, 0): one EEBB, header within 12 bursts; fake Return then Room;
                                            relink_frames < 100; club Room, link_type 0x1111, resit_needed; fake_ruby_partner(): "FLIPPER", 22222;
                                            sit(1) then 300 frames: sent_cafe16 unchanged (still on its seat); resit(): CAFE 0016 -> exchange:
                                            enemy(0) == Z; resit_needed false; fake error NULL */
TEST(test_trade_edit_after_dddd_never_ccdd) /* offer(2) -> DDDD; set_offer(Z, 0); answer(yes) -> BBBB -> EEBB;
                                            f_codes == {DDDD, EEBB}; fake Return; no CCDD ever */
TEST(test_trade_edit_before_reexchange)  /* bench_trade setup; in the frame trades_done becomes 1: set_offer(Z, 1);
                                            the post-save exchange shows enemy(0) == Z; then 2000 frames in Browsing: no EEBB */
TEST(test_trade_two_round_trips)         /* bench_trade(2); set_offer(Z, 1) -> EEBB -> Room; resit; bench_trade(2):
                                            saved party[2] == Z with Friendship 70; saved_trades 2; trades_done 2;
                                            get_received == the GBA's slot 2 before trade 2 (Y with Friendship 70) */
TEST(test_trade_eebb_retry_after_summary)/* Browsing; open_summary; set_offer(Z, 0): EEBB #1 lost; close_summary 100 frames later;
                                            EEBB #2, header 180..190 bursts after #1, brings the GBA to Room; EEBB count 2 */
TEST(test_trade_checksum_glitch_resits)  /* sit; fake_ruby_corrupt_next_checksum once status.round == 2: in Menu, offer_stale true,
                                            unsolicited EEBB, no DDDD; resit; clean exchange; bench_trade(0) succeeds; f_codes starts
                                            {EEBB, DDDD, CCDD, DCBA} */
TEST(test_trade_relink_identity_checked) /* EEBB (set_offer(Z, 0)); at fake phase Return with the link open: bench_raw_relink with the
                                            club's LinkPlayer but trainer_id 22223 and link_type 0x1111: fake error "relink mismatch" */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail.
- [ ] **Step 3: Implement** the fake and club behaviour.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: send the GBA back to the room when the offer changes`.

---

### Task 8: Leaving the Cable Club cleanly (D9)

**Files:**
- Modify: `tests/fake_ruby.c`, `src/gen3/gen3_club.h`, `src/gen3/gen3_club.c`, `tests/test_gen3_trade.c`

**Interfaces:**
- Produces:
  ```c
  /* Ask the GBA to walk out (D9); safe in any phase. Done when status.left. */
  void gen3_club_request_leave(Gen3Club* club);
  /* Gen3ClubStatus, appended: */
      bool leaving;  /* requested, not done yet */
      bool left;     /* the GBA walked out and closed the link after a request, or no GBA was linked */
  ```

Fake behaviour: in the seat task's case 2, any slot EXITING_ROOM cancels the seat (§B1.3 row 1, `sub_8054FC0`). Task 3 point 3 then follows, and the 0x1A makes the player IDLE. The plan-1 forced exit then runs: an IDLE player while anyone is EXITING_ROOM goes BUSY, then `TradeRoom_TerminateLink`.

Club behaviour:
- `gen3_club_request_leave` while Waiting: `left` at once.
- **Room** (a GBA CAFE seen this room session): one CAFE 0017, unless the GBA's own 0017 was already answered. The club never sits while leaving. The GBA's later 0017 isn't answered.
- **Menu:** `offer_stale` includes the request, so EEBB follows (Task 7), then the 0x1111 room, then the rule above.
- **Every other phase:** go on until Room or Menu.
- The GBA's 5FFF after an exit, Closing, then Disconnected: `left = true`, `leaving = false`.

- [ ] **Step 1: Write the failing tests:**

```c
TEST(test_leave_from_room)      /* reach_room; request_leave: sent_cafe17 == 1 (ours), never another; the Flipper answers the GBA's 5FFF;
                                   fake Closed, error NULL; club Waiting, left, !leaving */
TEST(test_leave_from_menu)      /* bench_reach_menu; request_leave: EEBB -> fake Return -> Room (0x1111) -> CAFE 0017 -> Closed; left */
TEST(test_leave_while_seated)   /* no offer; sit(0); 60 frames; request_leave: G sends 0x1A, then 0x17; fake Closed; left */
TEST(test_leave_waits_for_save) /* offer(2), answer(yes); request_leave in the frame CCDD is queued: saved_trades 1, Done, then EEBB,
                                   Room, CAFE 0017, Closed; left */
TEST(test_leave_when_waiting)   /* no open_link: request_leave -> status.left at once, leaving false */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail.
- [ ] **Step 3: Implement** the fake and club behaviour.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: walk the GBA out of the Trade Center before ending the link`.

---

### Task 9: Editor: the GBA player as original trainer (D7)

**Files:**
- Modify: `src/gen3/gen3_pkm.h`, `src/gen3/gen3_pkm.c`, `src/gen3/gen3_edit.h`, `src/gen3/gen3_edit.c` (plan 2 files)
- Test: `tests/test_gen3_pkm.c`, `tests/test_gen3_edit.c`

**Interfaces:**
- Produces (`gen3_pkm.h`):
  ```c
  void gen3_pkm_ot_name_raw(const Gen3Pkm* p, uint8_t out[GEN3_OT_NAME_LEN]);        /* raw[0x14..0x1A] */
  void gen3_pkm_set_ot_name_raw(Gen3Pkm* p, const uint8_t name[GEN3_OT_NAME_LEN]);
  ```
- Produces (`gen3_edit.h`):
  ```c
  typedef struct {
      uint8_t name[GEN3_OT_NAME_LEN]; /* game encoding, as the GBA's LinkPlayer name[0..6] (Ruby's OT name bytes) */
      uint16_t tid;
      uint16_t sid;
      uint8_t gender;                 /* 0 male, 1 female */
  } Gen3Trainer;
  /* Gen3Editor gains: bool pristine; bool has_trainer; Gen3Trainer trainer; */
  /* Remember the GBA player. A pristine editor (still the initial default) takes it as OT at once and
   * stays pristine. */
  void gen3_edit_set_trainer(Gen3Editor* ed, const Gen3Trainer* t);
  const Gen3Trainer* gen3_edit_trainer(const Gen3Editor* ed);       /* NULL until set */
  bool gen3_edit_ot_is_trainer(const Gen3Editor* ed);               /* OT bytes, TID, SID and OT gender equal the trainer's */
  /* OT name (raw), OT gender, then gen3_edit_set_ot_id(tid, sid) so shininess is kept (plan 2 D6).
   * false (nothing changed) without a trainer. */
  bool gen3_edit_use_trainer_ot(Gen3Editor* ed);
  ```

Rules:
- `pristine` is true after `gen3_edit_init`. Every other mutator clears it, and so does `gen3_edit_load` (`use_trainer_ot` included); `set_trainer` keeps it.
- `gen3_edit_set_species` with `has_trainer`: the new Pokemon's OT name, TID, SID and OT gender are the trainer's. This amends plan 2 D5; without a trainer, D5 is unchanged. Its first non-shiny PID is searched against that OT ID.
- Every mutator still ends with plan 2's commit step.

- [ ] **Step 1: Write the failing tests** (`test_gen3_edit.c` tests end with `ASSERT_TRUE(edit_consistent(&ed))`). RUBY = `{CC CF BC D3 FF FF FF}`, 12345 / 54321, gender 1:

```c
TEST(test_ot_name_raw)               /* set_ot_name_raw(RUBY) -> raw[0x14..0x1A] == it; ot_name_raw reads it; gen3_pkm_ot_name -> "RUBY";
                                        {01 02 03 04 05 06 07} kept byte for byte */
TEST(test_edit_trainer_pristine)     /* init(7); set_trainer(RUBY): OT bytes RUBY, OtId 0xD4313039, OtGender 1, not shiny;
                                        species 1 and level 5 unchanged; ot_is_trainer; pristine */
TEST(test_edit_trainer_after_edit)   /* init; set_level(10); set_trainer(RUBY): OT still C0 C6 C3 CA CA BF CC, OtId 22222, OtGender 0;
                                        ot_is_trainer false; trainer() != NULL */
TEST(test_edit_species_takes_trainer)/* init; set_level(10); set_trainer(RUBY); set_species(252): OT RUBY, 0xD4313039, OtGender 1, not shiny;
                                        plan-2 test_edit_species_resets_keeps_ot still passes (no trainer: OT kept) */
TEST(test_edit_use_trainer_ot)       /* load golden; set_traits(shiny); set_trainer(RUBY); use_trainer_ot() true: OT RUBY, 12345/54321, 1;
                                        still shiny; nature, gender, ability slot unchanged. Fresh editor without a trainer: false, raw unchanged */
TEST(test_edit_trainer_raw_bytes)    /* trainer name {01 02 03 04 05 06 07} on a pristine editor: OT bytes equal it */
TEST(test_edit_pristine_flags)       /* init: pristine; set_level, set_nickname, set_item and load each clear it; set_trainer keeps it */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail; all plan-2 tests still pass.
- [ ] **Step 3: Implement** the accessors, the editor fields and the rules.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: editor: take the GBA player as original trainer`.

---

### Task 10: App-side sync and trade screen guide (pure)

**Files:**
- Create: `src/gen3/gen3_trade_sync.h`, `src/gen3/gen3_trade_sync.c`
- Test: `tests/test_gen3_trade_sync.c`

**Interfaces:**
- Consumes: Tasks 1–9.
- Produces:
  ```c
  void gen3_trainer_from_link_player(const Gen3LinkPlayer* lp, Gen3Trainer* out); /* name[0..6], TID/SID halves, gender 0/1 */
  typedef struct {
      uint32_t trades_seen;            /* trades whose Pokemon the editor has taken */
      bool trainer_known; Gen3Trainer trainer;  /* last trainer handed to the editor */
      bool offer_valid;                /* offer holds the editor's Pokemon, game form; false: offer nothing */
      uint8_t offer[GEN3_PARTY_SIZE];
      Gen3Refuse offer_refuse;         /* why the editor's Pokemon can't be offered; None if it can */
      bool taken; bool taken_valid; uint8_t taken_offer[GEN3_PARTY_SIZE]; /* what the session holds */
      Gen3Pkm scratch;
  } Gen3TradeSync;
  void gen3_trade_sync_init(Gen3TradeSync* sync);
  enum { Gen3SyncPickedUp = 1u << 0, Gen3SyncPickupBad = 1u << 1, Gen3SyncTrainer = 1u << 2, Gen3SyncOffer = 1u << 3 };
  /* App thread, between UI events. received/received_count come from one gen3_session_get_received.
   * editing (the current scene is neither Gen3Trade nor Gen3Menu): return 0 and do nothing (D6).
   * Else, in order:
   * 1. received_count != trades_seen: decrypt and gen3_pkm_check; Ok -> gen3_edit_load, PickedUp;
   *    else PickupBad, editor unchanged; trades_seen = received_count either way.
   * 2. st->trainer_valid and not the known trainer: gen3_edit_set_trainer, Trainer.
   * 3. offer_refuse = gen3_trade_offer_check(&ed->pkm); offer_valid = (None); encrypt when valid.
   *    Offer if nothing was taken yet, or (offer_valid, offer) differs from what was taken. */
  unsigned gen3_trade_sync_poll(
      Gen3TradeSync* sync, Gen3Editor* ed, const Gen3ClubStatus* st,
      const uint8_t received[GEN3_PARTY_SIZE], uint32_t received_count, bool editing);
  void gen3_trade_sync_offer_taken(Gen3TradeSync* sync); /* the session accepted the offer (or "none") */
  #define GEN3_GUIDE_LINES 3
  #define GEN3_GUIDE_LINE_LEN 22 /* 21 chars + NUL: FontSecondary across 128 px */
  /* What is happening and what to do on the GBA. started: the session runs. Unused lines are "". */
  void gen3_trade_guide(
      const Gen3ClubStatus* st, bool started, bool cable_ok, Gen3Refuse offer_refuse,
      char out[GEN3_GUIDE_LINES][GEN3_GUIDE_LINE_LEN]);
  ```

Guide texts. The first row that matches wins; "R" is the reason line below the table.

| Case | Line 1 | Line 2 | Line 3 |
|---|---|---|---|
| `!cable_ok` | `Plug in the cable` | `(or swap its ends).` | |
| `!started` | `Starting...` | | |
| `save_window` | `The GBA is saving.` | `Don't unplug or turn` | `off. Back is locked.` |
| `left` | `Link ended. You can` | `unplug the GBA.` | |
| `leaving` | `Ending: the GBA walks` | `out of the Trade` | `Center by itself.` |
| `trade_pending` | `Trading...` | `Watch the GBA.` | |
| Error, OtherRoom | `The GBA chose another` | `room. Use the Trade` | `Center (right).` |
| Error, other | `Link error. Restart` | `the GBA, then go to` | `the Cable Club.` |
| `offer_refuse` ≠ None | `Can't offer it:` | R(offer_refuse) | `Edit it first.` |
| Waiting, RolledBack | `Link lost while the` | `GBA saved. It keeps` | `its old save.` |
| Waiting, Assumed | `Link lost at the end.` | `Check the GBA party:` | `it likely traded.` |
| Waiting, Canceled | `Link lost. Nothing` | `was traded.` | |
| Waiting | `Go to the Cable Club:` | `Pokemon Center 2F,` | `right counter.` |
| Linked, Players, Cards | `Linking...` | | |
| Room, `resit_needed` | `Sit again on the GBA:` | `step Up, then Down.` | |
| Room, `gba_seated`; or Exchange | `Opening the trade` | `menu...` | |
| Room | `In the Trade Center:` | `sit at the table on` | `the GBA.` |
| Menu, `offer_stale` or Leaving | `Updating the trade` | `menu: the GBA goes` | `back to the room.` |
| Menu, `refused` ≠ None | `Can't take that one:` | R(refused) | `Pick another one.` |
| Menu, Offered | `Confirm the trade` | `on the GBA.` | |
| Menu, `trades_done` > 0 | `Edit it (OK), or pick` | `the next trade on` | `the GBA.` |
| Menu | `Pick a Pokemon to` | `trade on the GBA.` | |
| otherwise | `Please wait...` | | |

R: Empty `that slot is empty.`, BadEgg `it's a Bad Egg.`, Egg `it's an Egg.`, Mail `it holds Mail.`, Item `it holds a bad item.`, BadData `its data is invalid.`

- [ ] **Step 1: Write the failing tests:**

```c
TEST(test_trainer_from_link_player)  /* name bytes 0..6 copied (8..10 ignored); trainer_id 0xD4313039 -> 12345/54321; gender 5 -> 1 */
TEST(test_sync_first_offer)          /* init editor (seed 7), Waiting status: poll -> Offer; offer_valid; offer == encrypt(ed.pkm);
                                        offer_taken; poll again -> 0 */
TEST(test_sync_trainer)              /* status trainer_valid RUBY: poll -> Trainer|Offer; editor OT RUBY (pristine); same status again -> 0 */
TEST(test_sync_pickup)               /* received golden_game(), count 1: PickedUp|Offer; ed.pkm == GOLDEN_PK3; trades_seen 1;
                                        offer == golden_game() */
TEST(test_sync_pickup_before_offer)  /* set_level(30), then a poll with count 1: the offer is the received Pokemon, not the edit */
TEST(test_sync_bad_received)         /* received golden_game() with [0x30] ^= 1, count 1: PickupBad; ed.pkm unchanged; trades_seen 1 */
TEST(test_sync_holds_while_editing)  /* editing true with a new received, a trainer and an edit: 0, nothing changes; editing false: all three */
TEST(test_sync_bad_offer_unpublished)/* editor HeldItem 121: Offer with offer_valid false, offer_refuse Mail; taken; item 0: Offer, valid */
TEST(test_guide_texts)               /* one status per table row (plus each R reason): exact 3 lines */
TEST(test_guide_line_lengths)        /* every phase x every last_error x {save_window, left, leaving, trade_pending, offer_stale,
                                        resit_needed, gba_seated} x every Gen3Refuse (refused and offer_refuse) x every Gen3TradeResult:
                                        each line <= 21 chars */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL (missing header).
- [ ] **Step 3: Implement** `gen3_trade_sync.{h,c}`.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add the trade sync between editor and link, and the trade guide`.

---

### Task 11: Session trade mode

**Files:**
- Modify: `src/gen3/gen3_session_furi.h`, `src/gen3/gen3_session_furi.c`

**Interfaces:**
- Consumes: Tasks 2–8 club API.
- Produces:
  ```c
  /* As gen3_session_start, but the club runs with trading on (GEN3_TEST_TRAINER_* identity, D7) and
   * the National Dex flag set before the driver starts. No offer yet: the app tick publishes one. */
  bool gen3_session_start_trade(Gen3Session* s, bool national_dex);
  bool gen3_session_running(Gen3Session* s);
  bool gen3_session_trading(Gen3Session* s); /* running and started by gen3_session_start_trade */
  /* App context. Each runs the club call inside a PRIMASK section (Global Constraints). */
  bool gen3_session_set_offer(Gen3Session* s, const uint8_t* offer, uint32_t trades_seen); /* gen3_club_set_offer */
  void gen3_session_get_received(Gen3Session* s, uint8_t out[GEN3_PARTY_SIZE], uint32_t* trades_done);
  void gen3_session_set_national_dex(Gen3Session* s, bool on);
  void gen3_session_request_leave(Gen3Session* s);
  ```
- Unchanged: `gen3_session_start` (Link Test, `trading` false), the status double buffer, the log thread and the log file (same name, header and format; the new phases log as `p7`..`p10`).

- [ ] **Step 1: Implement** the functions. The frame callback stays as in plan 1.
- [ ] **Step 2: Build** `ufbt` and the dev SDK. Expected: 0 warnings. Run `make -C tests`: PASS.
- [ ] **Step 3: Review the ISR path** and put the answers in the report (Review Focus 5):
  - The frame callback calls only `gen3_club_frame`, `gen3_club_status` and `gen3_log_frame`, as before.
  - No new code runs in it.
  - Each PRIMASK section copies ≤ 100 bytes and calls nothing that can block.
- [ ] **Step 4: Commit** `gen3: session: trade mode and safe offer/received handoff`.

---

### Task 12: Trade screen, Gen III menu entry, app tick

**Files:**
- Create: `src/views/gen3_trade.h`, `src/views/gen3_trade.c`, `src/scenes/pokemon_gen3_trade.c`, `docs/gen3/hardware-test-2.md`
- Modify: `src/scenes/pokemon_gen3_menu.c`, `src/scenes/include/pokemon_scene_config.h` (append `ADD_SCENE(pokemon, gen3_trade, Gen3Trade)`), `src/include/pokemon_app.h`, `src/pokemon_app.c`

**Interfaces:**
- Produces (`gen3_trade.h`):
  ```c
  typedef struct {
      char guide[GEN3_GUIDE_LINES][GEN3_GUIDE_LINE_LEN];
      char flipper_line[24]; /* "Flipper: <NICKNAME>" */
      uint32_t errors;       /* status.link.checksum_errors */
      bool can_edit;         /* draws the center "Edit" button; OK works only then */
  } Gen3TradeViewState;
  typedef void (*Gen3TradeEditCallback)(void* context); /* app thread (view input) */
  void* gen3_trade_view_alloc(ViewDispatcher* view_dispatcher, uint32_t view_id);
  void gen3_trade_view_free(ViewDispatcher* view_dispatcher, uint32_t view_id, void* view);
  void gen3_trade_view_set_edit_callback(void* view, Gen3TradeEditCallback cb, void* context);
  void gen3_trade_view_update(void* view, const Gen3TradeViewState* state);
  ```
- Produces (`pokemon_app.h`, appended):
  - `AppViewGen3Trade`;
  - fields `void* gen3_trade_view; struct Gen3TradeSync* gen3_trade_sync; bool gen3_national_dex; bool gen3_trade_ended;`;
  - `void pokemon_gen3_trade_tick(PokemonFap* pokemon_fap);`, defined in `pokemon_gen3_trade.c`.

View (128×64, `ViewModelTypeLocking`; no timer of its own):
- `Gen III Trade` in FontPrimary at (0, 10). When `errors > 0`, `E<n>` right-aligned on that line in FontSecondary.
- The three guide lines in FontSecondary at y 21, 31 and 41, then `flipper_line` at y 51.
- `elements_button_center(canvas, "Edit")` if `can_edit`.
- **Input:** a short OK press with `can_edit` calls the edit callback and is consumed. Back is not consumed.

App tick:
- `pokemon_alloc` sets the four new fields to NULL/false. It calls `view_dispatcher_set_tick_event_callback(view_dispatcher, pokemon_tick_event_callback, furi_ms_to_ticks(100))`; that callback only calls `pokemon_gen3_trade_tick`.
- `free_app` frees any trade view, sync and session before `view_dispatcher_free` (defensive).
- `pokemon_gen3_trade_tick` returns at once unless `gen3_trade_view` exists. Otherwise:
  1. **Start.** If the session isn't running and `!gen3_trade_ended`: once `gen3_session_check_cable` reads OK, call `gen3_session_start_trade(s, gen3_national_dex)`.
  2. **Read.** `gen3_session_status` (with `cable_ok`) and `gen3_session_get_received`.
  3. **Sync.** If trading: `gen3_trade_sync_poll(…, editing)`, with `editing` = `scene_manager_get_current_scene` is neither Gen3Trade nor Gen3Menu. On `Gen3SyncOffer`: if `gen3_session_set_offer(s, offer_valid ? offer : NULL, trades_seen)` returns true, call `gen3_trade_sync_offer_taken`.
  4. **End.** If `status.left`: `gen3_session_stop`, `gen3_trade_ended = true`.
  5. **Show.** Build the view state:
     - `gen3_trade_guide(&status, running || ended, cable_ok, sync->offer_refuse, …)`. `cable_ok` is the driver's live guard while running, `gen3_session_check_cable` before the start, and true once ended.
     - `flipper_line` is `"Flipper: "` and the nickname from `gen3_pkm_nickname(&gen3_editor->pkm, …)`.
     - `can_edit = running && !trade_pending && !save_window && !leaving && !left`.

     Then call `gen3_trade_view_update`.

Scenes:
- **Gen III menu items:** `Trade`, `Edit Pokemon`, `Load .pk3`, `Save .pk3`, then `Link Test` only while `!gen3_session_trading`.
  - `Trade` with no session: alloc the session, the trade view (edit callback → custom event `GEN3_TRADE_EVENT_EDIT 0x47330201u`) and a sync (`gen3_trade_sync_init`), set `gen3_trade_ended = false`, then `next_scene(Gen3Trade)`. With a running trade session, it just opens Gen3Trade.
  - `Edit Pokemon` and `Load .pk3` while trading with `status.trade_pending`: open Gen3Trade instead (D6).
- **Gen III menu `on_enter`:** switch to the submenu first, then free the session unless `gen3_session_trading`. Free the trade view and sync with it.
- **Gen3Trade:**
  - `on_enter` runs one tick and switches to `AppViewGen3Trade`.
  - `on_event`: `GEN3_TRADE_EVENT_EDIT` → `next_scene(Gen3Edit)`.
  - Back: not consumed in this task; Task 13 adds the rules.
  - Gen3Edit's Back must be the scene manager default, so it returns to Gen3Trade. If plan 2 consumed it with a search to Gen3Menu, change it to `scene_manager_previous_scene`.

- [ ] **Step 1: Implement** the view, the scene, the menu changes and the tick.
- [ ] **Step 2: Build** `ufbt` and the dev SDK. Expected: 0 warnings. Run `make -C tests`: PASS.
- [ ] **Step 3: Gen I/II check by reading:** the main menu, Gen I and Gen II scenes are untouched. The tick returns at once when no trade view exists.
- [ ] **Step 4: Write `docs/gen3/hardware-test-2.md`** with an intro (goal; you need: Flipper with the Game Link module, GBA, GBA link cable, genuine Ruby; build `ufbt` release and install `dist/pokemon.fap`). The log is `apps_data/pokemon/gen3_link.log`: overwritten at each Trade or Link Test start, ≈ 8 KB/s while in the room, phases `p0`..`p10`. Then these sections:
  - **A. Setup and Link Test regression**
    1. Ruby: Pokedex received; at least 3 Pokemon, one you're happy to trade; none holding Mail or the ENIGMA BERRY; saved.
    2. Plug the cable as in test 1, same plug in the Flipper.
    3. Gen III → `Link Test`: run test 1 steps 5–8 once. The Flipper's LinkPlayerBlock now goes out after the GBA's (D2); it must still reach `in Trade Center` and leave cleanly.
  - **B. Start a trade**
    1. Start the app fresh and don't edit the Pokemon first. Gen III → `Trade`: `Plug in the cable` until it reads OK, then `Go to the Cable Club: Pokemon Center 2F, right counter.`
    2. Enter the Trade Center: `Linking...`, then `In the Trade Center: sit at the table on the GBA.`
    3. Back to the Gen III menu (the link keeps running) → Edit Pokemon: `OT Name` is your Ruby name and `OT TID`/`OT SID` are yours. The default Bulbasaur took your OT (D7).
  - **C. First trade (GBA → Flipper)**
    1. Walk onto either seat: `Opening the trade menu...`. Ruby shows the trade menu with BULBASAUR alone on the right.
    2. Choose one of yours → TRADE → YES. The Flipper shows `Confirm the trade`, then `Trading...`, then during "SAVING..." `The GBA is saving. ... Back is locked.` Press Back on the Flipper during the save: nothing happens.
    3. After the save Ruby shows the trade menu again, with your Pokemon on the Flipper side; `Flipper: <its nickname>`.
    4. Ruby: BULBASAUR is in your party with you as OT.
  - **D. Edit and trade back** (Review Focus 1)
    1. On the trade screen press OK (`Edit`), change the species to TREECKO, then Back to the trade screen.
    2. The Flipper shows `Updating the trade menu...`. Ruby leaves the menu, with your avatar on its seat.
    3. The Flipper shows `Sit again on the GBA: step Up, then Down.` Hold Up for one step, then Down for one step. If Up is blocked, try Down then Up and note it. The menu opens with TREECKO.
    4. Trade any of yours for TREECKO. Ruby: TREECKO with you as OT; it obeys in battle and gets no "boosted" Exp.
    5. Repeat C–D once without leaving the room.
  - **E. Background link while you work** (Review Focus 5). With the GBA in the trade menu (don't touch it), for about 3 minutes on the Flipper:
    1. Edit Pokemon → Nickname: type 10 letters slowly.
    2. The species picker: scroll through 60 species with sprites.
    3. Back to the Gen III menu → Save .pk3 → Load .pk3 (browse to the file and load it) → Trade.
    4. Expected: Ruby never shows a link error, the title shows no `E<n>`, and trading still works afterwards.
    5. If Ruby shows a link error here, write down what you were doing. Use the spec's fallback for the rest of the test: walk out of the Trade Center before editing and back in afterwards (the Flipper waits at `Go to the Cable Club`).
- [ ] **Step 5: Commit** `gen3: add the trade screen and run the link behind every Gen III screen`.

---

### Task 13: End trade, save-window guard, National Dex, OT row

**Files:**
- Create: `src/scenes/pokemon_gen3_trade_end.c`, `src/scenes/pokemon_gen3_natdex.c`
- Modify: `src/scenes/include/pokemon_scene_config.h` (append `ADD_SCENE(pokemon, gen3_trade_end, Gen3TradeEnd)`, `ADD_SCENE(pokemon, gen3_natdex, Gen3NatDex)`), `src/scenes/pokemon_gen3_menu.c`, `src/scenes/pokemon_gen3_trade.c`, `src/scenes/pokemon_gen3_edit.c`, `docs/gen3/hardware-test-2.md`

Behaviour (D9, D10, D7):
- **Gen III menu:**
  - After `Save .pk3`, insert `National Dex: Off`/`On` (index `PokemonSceneGen3NatDex`). While trading it is followed by `End trade` (index `PokemonSceneGen3TradeEnd`).
  - **National Dex:** when Off, opens Gen3NatDex. When On, turns it off at once (`gen3_national_dex = false`, plus `gen3_session_set_national_dex` if a session exists) and rebuilds the menu.
  - **End trade:** if the status phase is Waiting, stop and free the session, the view and the sync, then rebuild the menu. Otherwise open Gen3TradeEnd.
  - **Back:** while trading with the phase not Waiting, open Gen3TradeEnd (consumed). Otherwise stop and free any session; not consumed, so the app goes to the main menu.
- **Gen3TradeEnd** (DialogEx):
  - With `save_window`: header `The GBA is saving`, text `Wait until it's done,\nthen end the trade.`, center `OK` → previous scene.
  - Otherwise: header `End trade?`, text `The GBA walks out of\nthe Trade Center by\nitself, then the link\nstops.`, left `Stay` (previous scene), right `End`.
  - `End` calls `gen3_session_request_leave`, then `scene_manager_search_and_switch_to_previous_scene(…, Gen3Menu)` and `next_scene(Gen3Trade)`, so the user watches it leave.
  - Back behaves like Stay or OK.
- **Gen3NatDex** (DialogEx): header `Unlock National Dex?`, text `The next trade unlocks\nthe GBA's National Dex\nfor good.`, left `Cancel`, right `Turn on`. `Turn on` sets `gen3_national_dex` and the session's flag, then goes to the previous scene.
- **Gen3Trade Back:**
  - With `save_window`: consumed; the guide already explains.
  - While `leaving` and running: `next_scene(Gen3LinkExit)` (plan 1's "Stop link?").
  - Otherwise not consumed: previous scene, with the session still running.
- **Gen3Edit row** (after `OT SID`): `OT: use <NAME>`, index `PokemonSceneNum` (never a scene id). It shows only when `gen3_edit_trainer(ed)` is set and `!gen3_edit_ot_is_trainer(ed)`. OK calls `gen3_edit_use_trainer_ot` and rebuilds the menu. `<NAME>` is the trainer's name bytes through `gen3_text_decode`.

- [ ] **Step 1: Implement** the scenes, the menu items and rules, the trade-screen Back rules and the editor row.
- [ ] **Step 2: Build** `ufbt` and the dev SDK. Expected: 0 warnings. Run `make -C tests`: PASS.
- [ ] **Step 3: Add sections F–I** to `docs/gen3/hardware-test-2.md`:
  - **F. Cancels and refusals**
    1. Choose yours → TRADE → NO. Ruby: "The trade has been canceled."; A returns to the menu.
    2. CANCEL → "Cancel trade?" YES. Ruby goes back to the room; the Flipper shows `Sit again...`.
    3. Before entering, give one party Pokemon Mail on the GBA, then offer it. The Flipper shows `Can't take that one: it holds Mail. Pick another one.`; Ruby shows "The trade has been canceled." (Review Focus 3)
    4. If you have an Egg, the same gives `it's an Egg.`
  - **G. Summary screen and the save lock** (Review Focus 2, 4)
    1. In the menu, open SUMMARY of any Pokemon on the GBA. On the Flipper, change the level and return to the trade screen. Close the summary: within about 3 s Ruby leaves the menu for the room.
    2. Before answering YES on the GBA, go Back to the Gen III menu on the Flipper. Answer YES. During "SAVING...", press Back on the Gen III menu: `The GBA is saving` / `Wait until it's done...`, and the link keeps running.
  - **H. End trade**
    1. From the room: Gen III menu → `End trade` → `End`. Ruby's player walks out by itself; the Flipper shows `Link ended. You can unplug the GBA.`
    2. The same from the trade menu: Ruby goes to the room first, then walks out.
    3. Edit Pokemon → Held Item → ORANGE MAIL. If the item list has no Mail, load a `.pk3` holding Mail instead, or skip this step. The trade screen shows `Can't offer it: it holds Mail. Edit it first.` (D8). Sit on the GBA: "Please wait..." stays. `End trade` → `End`: Ruby cancels the seat and walks out.
    4. Gen III menu Back while linked shows `End trade?`; `Stay` keeps the link.
  - **I. National Dex** (call #8)
    1. Default: after the trades above, Ruby's Pokedex offers no NATIONAL mode.
    2. Optional, permanent on your save: Gen III menu → `National Dex: Off` → `Turn on`, then trade once; Ruby's Pokedex now offers NATIONAL. Skip this if you don't want it.
- [ ] **Step 4: Commit** `gen3: end the trade cleanly, lock Back while the GBA saves, National Dex opt-in`.

---

### Task 14: Docs, final checks, hardware gate

**Files:**
- Modify: `README.md` (section "Gen III (experimental): trading with Ruby/Sapphire"), `changelog.md` (under `## Unreleased`), `docs/gen3/hardware-test-2.md`

README section:
- How to trade: Gen III → Trade, the Cable Club, sit down, trade.
- Edit between trades: the GBA goes back to the room; step Up, then Down to sit again.
- The OT default, and the `OT: use <NAME>` row.
- What the Flipper refuses (Eggs, Mail, the ENIGMA BERRY, items Ruby lacks).
- The save lock, End trade, the National Dex option (off by default, permanent when used).
- The log, and a link to `docs/gen3/hardware-test-2.md`.

`changelog.md` gets one bullet: "Gen III Cable Club trades with Ruby/Sapphire (experimental)".

- [ ] **Step 1: Write** the README section and the changelog bullet.
- [ ] **Step 2: Add section J** to `hardware-test-2.md`:
  1. **Memory:** with a trade session running and Edit Pokemon open, run `free` in the qFlipper CLI (or `ufbt cli`). `Free heap size` ≥ 12,000.
  2. **Results:** a pass/fail table for A–J.
  3. **On failure:** the step, what each screen showed, and the log copied off the SD card before starting another Trade or Link Test.
  4. **Hints for whoever reads the log:**
     - Black screen on the GBA after sitting: the exchange (`p7`); check the CCCC sequence and `GEN3_CLUB_ROUND1_DELAY_BURSTS`.
     - "Link standby..." forever: a missing reply (`p8`).
     - Link error at the return to the room: the 0x1111 identity check.
- [ ] **Step 3: Final checks:**
  - `make -C tests` passes.
  - Both SDK builds have 0 warnings.
  - `text + data` ≤ baseline + 16,000 and ≤ 128,000; put the number in the report.
  - `git diff <Task 1 base>..HEAD --stat` over the Gen I/II files in Global Constraints and over `lib/flipper-gblink` prints nothing.
- [ ] **Step 4: Commit** `docs: Gen III Ruby trade`.
- [ ] **Step 5: Hand-off.** Push and give the user `dist/pokemon.fap` and `docs/gen3/hardware-test-2.md`. **Stop here until the user reports results.**
  - **E fails** (link errors while editing): keep the fallback (walk out before editing), report the log, and propose a follow-up. Don't change the protocol.
  - **Other failures:** fix them and retest only the failed sections.

---

## Spec coverage

| Requirement | Task |
|---|---|
| Phase 1 Cable Club trade with Ruby (Sapphire: same code) | 3–8, 12 |
| Party of one: the editor's Pokemon (§B2.5) | 4, 10, 12 |
| Received Pokemon becomes the editor's, tradable back, repeatedly | 6, 7, 10 |
| Learn the GBA player's name, TID, SID; default OT to the player | 2, 9, 10 |
| "OT: use GBA player" | 9, 13 |
| Flipper's own identity FLIPPER / 22222 / 0 | 2 (D7), 11 |
| GBA stays in the Trade Center while editing (call #7); background link through all Gen III screens | 11, 12 (D1, D6) |
| Fallback: leave and return | D1, 12 checklist E.5 |
| Pokemon changes → both leave the menu and re-sit; parties exchanged again | 7, 10, 12 |
| Exact GBA instruction to sit again | 10 (guide), 12 checklist D.3 |
| Post-trade automatic re-exchange carries the current party | 6, 7 (`test_trade_edit_before_reexchange`) |
| Ending the session leaves the Cable Club cleanly from the GBA side | 8, 13 |
| National Dex opt-in, off by default, 0x8000 only in the 0x1144 block (call #8) | 2, 6, 13 |
| Anti-goal: never damage the save (CCDD rule, DCBA after ABCD, 2FFE once, game-valid offers, Mail/item rules) | 1, 5, 6, 7 (Global Constraints) |
| Anti-goal: no link error in normal use; Back locked between sync 1 and sync 3 | 6, 8, 12, 13 (D10) |
| Finalize only after sync 3; link loss handled | 6 (D5) |
| Frame log stays on, same file and format, phases extended | 2, 11 |
| FireRed/LeafGreen later; claimed game behind one function | 2 (D11) |
| Link Test still works | 2, 11, 12 checklist A |
| Gen I/II unchanged | Global Constraints, 12 step 3, 14 step 3 |
| Fake Ruby faithful to Part B (rounds/T1, 20-byte blocks, CCDD rule, 0x1144, ABCD/DCBA, 3 syncs and the save window, EEBB, re-sit, 0x1111 check, 300-frame timeouts) | 3–8 |
| Hardware checklist, README, changelog; no `fap_version` bump | 12–14 |

## Amendments made during execution

The tasks above are kept as written. The controller's rulings amended them while they ran; this is a summary, and the code is the reference.

**Size, ISR and stack**
- **New Task 0 (R1):** the move and item names and their sorted-id lists moved from flash to the SD bulk asset (`gen3_bulk.bin` format v2), which saved about 11 KB. The letter lists now read the SD card (checklist E.2a times them).
- **Size budget:** `text + data` ≤ 112,000 for both builds, and dev ≤ 108,500 at every plan 3 commit. This replaces "+16,000 over the Task 1 baseline, ≤ 128,000". Measure the build directories (`~/.ufbt/build/pokemon_d.elf` and the dev SDK's), not `dist/`, which both builds write.
- **Final figures** (Task 14): release 87,624 and dev 97,580; after the final review's fixes, release 87,712 and dev 97,716. That is +480 and +2,024 since plan 3 started (87,144 / 95,556), and +11,592 and +12,976 since Task 0 (76,032 / 84,604).
- **ISR (R2):** the decrypt scratch and the round-block buffer are `Gen3Club` fields, never locals. A `gen3_club_frame` call copies at most 600 B (not 220) and decrypts at most once. The deepest chain from `mlink_hw_isr`, plus 16 B for the TIM2 dispatch, must be ≤ 600 B by `-fstack-usage`. Task 14 measured 432 B in release (the AABB path down to `gen3_pkm_decrypt`) and 392 B in dev, the same as Task 11. With SysTick (72 B) beneath it and a 32-B exception frame, that is about 536 B of the 1,024-B ISR stack.
- **App ↔ ISR:** each PRIMASK section copies at most 104 B (one Pokemon plus scalars), not 100. `Gen3ClubStatus` is never a stack local; app code reads it through `gen3_menu_status`, which copies it into one static buffer.

**Interfaces**
- Plan 2's real names replaced the ones written here: `gen3_*_name_get`, the bool `gen3_edit_*` setters, `gen3_edit_load`, `Gen3ImportInfo`, and `tests/bulk_file.h`.
- `Gen3ClubConfig` gained `trading`; every initializer names all five fields.
- The exported Gen III scene helpers are named `gen3_<scene>_...` after their scene: `gen3_trade_tick`, `gen3_trade_begin_edit` (in `pokemon_app.h`), `gen3_trade_end_decide`, `gen3_trade_end_open` and `gen3_result_set` (in `pokemon_gen3_menu.h`, the Gen III scenes' shared header).
- `gen3_menu_editor_ready` is exported (R3), so `Trade`, `Edit Pokemon` and `Load .pk3` open the bulk file and create the editor before the tick runs.

**Editing and offers**
- **Editing flag (O1):** opening `Edit Pokemon` or `Load .pk3` from the trade flow first sets the club's `editing` flag; the scene opens only if that works. While it is set, the offer is stale (EEBB), and the club never sits and never sends DDDD or CCDD. Setting it fails while a trade is pending. The tick clears it after publishing the new offer. One editing visit therefore costs one EEBB and one re-sit, and an edit is never lost to a trade.
- **Late sit (R5):** if the GBA sits before the Flipper has a valid offer, the Flipper sits as soon as it has one, and the trade screen says why it waits.
- **A received Pokemon the editor can't take (O2):** the offer is unpublished and the reason shown. A transient failure (the bulk file unreadable) is retried; a permanent one is not. The traded-away Pokemon is never offered again, also across session restarts.
- **Refusals:** plan 2 strips Mail on every load, so the bad-offer tests and checklist H use an ENIGMA BERRY in a committed fixture, `docs/gen3/fixtures/enigma-berry.pk3` (R7). D8's item ranges stand (O3).
- **An empty party (R8):** pokeruby shows Ruby accepts a partner party of 0; only a DDDD to it breaks Ruby. The Flipper never sends DDDD when it showed no Pokemon (DDEE instead).
- **Received stats:** Ruby changes EVs without recalculating stats, and `gen3_normalise` recalculates them, so a received Pokemon can come back with different bytes. The tick picks it up and publishes it on the first tick after the trade; if that comes after the post-save round 1, the club falls back to one EEBB and a re-sit.
- **Own link type (D2):** 0x1133 when not trading; 0x1144 mirrored only while a trade is pending; 0x1111 only after the Flipper's EEBB.

**Save safety**
- `Assumed` needs the sync-2 answer to have reached the GBA: a whole burst (no timeout) without a checksum error, and an empty queue. Otherwise a link loss after sync 2 gives `RolledBack`. A stop-bit error alone doesn't hold the answer back: when unsure, `Assumed` (at worst a clone of the Flipper's offer, which can be made again) is safer than `RolledBack` (at worst the GBA's traded Pokemon lost on both sides).
- A pending trade is cancelled on a 0x1122-session link loss unless that session's 5FFF close was answered.
- Only a filed GBA block restarts the 6-burst reply delay. An incomplete block is dropped after 6 silent bursts. The 6 idle bursts between rounds 2–5 stay as a margin that pokeruby doesn't need.

**Ending (R6, D1, D9, D10)**
- "Stop link?" is never offered in the save window, nor with a trade pending while the GBA is linked. A relink's gap counts as linked (`Gen3ClubStatus.relinking`). Leaving stays possible while no GBA is linked, because Ruby writes nothing before sync 1.
- `left` is not set for a close that reopens for a relink; a 300-frame timeout covers a GBA that never relinks. `left` can come with a trade still pending, and the UI never reads it as "trade resolved".
- No stop frees the club before the sync has every received Pokemon. With a pickup waiting for the SD card, `End trade` offers `Retry` or `End anyway`.
- Every dialog button decides again from a fresh status when pressed.
- `left` is sticky. Once it is set the club never sits again in that session, so a GBA that walks back into the Trade Center can't start a trade. The tick ends a left session only when it is settled, no GBA is linked (Waiting, not relinking) and R6 allows it.
- A trade whose link was lost in the GBA's save shows its outcome (`RolledBack` or `Assumed`) ahead of `Link ended`, also after `End trade` during that trade.
- `Trade` in the Gen III menu opens a running session's trade screen without needing the data file; a new session still needs it.
- Task 14: the trade screen reads a relink's gap as linked too (`Linking...`), so `Go to the Cable Club` no longer flashes up for about 20 frames at each relink.

**Docs and the hardware gate**
- `docs/gen3/hardware-test-2.md` has sections A–K. Section J became the OT row, so memory is K, followed by the results table, what to report on failure, and hints for reading the log.
- **Hardware gate: pending with the user.** By the user's ruling ("finish everything first"), Task 14 prepared the checklists (`hardware-test-1.md`, `editor-test.md`, `hardware-test-2.md`), the README and changelog, and the final size and stack figures, but did not run the gate. Step 5 (hand-off, then fixes for any failed section) waits until the user tests on hardware after the remaining plans are built.
