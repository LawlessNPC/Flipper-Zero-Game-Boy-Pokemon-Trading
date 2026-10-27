# Gen III FireRed/LeafGreen Cable Club Trade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The plan-3 Ruby trade also works with English FireRed and LeafGreen. That includes players without the National Dex, who get a clear reason whenever the Flipper can't offer its Pokemon. Phase 1 then ships as v3.0.

**Architecture:** The club reads the partner's game and National Dex flag from the GBA's own LinkPlayerBlock, which plan 3 receives before sending its own. Plan 3's one claim function then claims FireRed/LeafGreen with the FRLG trainer card. The club also mirrors the FRLG master's exchange delays. It never sits down with, or shows, a Pokemon that this partner's rules forbid. The offer reaches the club with its traits (non-Kanto, unfateful Mew/Deoxys), computed on the app side, so the ISR never decrypts more than plan 3 allows. The plan-3 fake Ruby gains an FRLG mode, selected by its own `version`, which models pokefirered wherever Part C says FRLG differs.

**Tech Stack:** C11, Flipper SDK via `ufbt` (release SDK 1.5.1, API 88.2 per plan 1), `make` + host `cc` for unit tests, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-10-25-gen3-gba-design.md`: Goals → Phase 1 ("FireRed and LeafGreen trades come next"), Anti-goals, "Left to the builder" (FRLG differences: National Dex gating, version checks), and the version line under Constraints → Project ("Phase 1 ships as version 3.0"). Protocol: `docs/gen3/link-protocol-trade.md` Part C (cited §C1 … §C6), Part B (§B…), Uncertain (T…); `docs/gen3/link-protocol.md` Part C. Plans: `docs/superpowers/plans/2026-10-25-gen3-01-link-connection-test.md` (with its Amendments), `…-02-pokemon-editor.md`, `…-03-ruby-trade.md`.

## Plan sequence

This is plan 4 of 5.

1. Link connection test (done).
2. Gen III data core, editor and `.pk3` files (done before this plan starts).
3. Cable Club trade with Ruby/Sapphire (done before this plan starts).
4. **This plan:** FireRed/LeafGreen trades, then the Phase 1 release, v3.0.
5. Direct save access, v3.1.

Work on branch `gen3-gba-design`, where plans 1–3 live.

## Global Constraints

- **Commits:** authored by the user (LawlessNPC, Aleksandr Kofman) with **no** AI-attribution trailers (no `Co-Authored-By`, no `Claude-Session`, no "Generated with"). `git log -1 --format='%(trailers)'` must print nothing.
- **Staging:** stage only the files your task lists, by explicit path. Never use `git add -A`, `git add .`, a directory, or `git commit -a`.
- **Builds:** `ufbt` (release) and `UFBT_HOME=/Users/kyvo/.claude/jobs/72354b36/tmp/ufbt-dev ufbt` (dev) both finish with 0 warnings. `make -C tests` (`-Wall -Wextra -Werror`, so `-Wswitch` is an error) passes after every task, with every plan 1–3 test still green.
- **Pure-module rule:** every file in `src/gen3/` whose name doesn't end in `_furi.c`/`_furi.h` includes only `<stdint.h> <stdbool.h> <stddef.h> <string.h> <stdio.h>` and other pure `src/gen3/` headers.
- **ISR rule:** everything called from the link frame callback runs in interrupt context: no `malloc`/`free`, no `FURI_LOG_*`, no blocking calls, under 200 µs. `gen3_club_frame` decrypts at most one Pokemon per call (on the GBA's AABB) and copies at most 220 bytes per call. This plan adds no decrypt to the ISR path.
- **App ↔ ISR rule:** app code reaches the club only through `gen3_session_*`. Each call runs inside `primask = __get_PRIMASK(); __disable_irq(); … __set_PRIMASK(primask);` and copies at most one 100-byte Pokemon plus a few scalars there, as in plan 3. `gen3_session_set_offer_traits` adds only the traits byte, and nothing decrypts inside that section.
- **Licence:** MIT. pret and PKHeX are references for behaviour, constants and layouts only; copy no code. The clone `/Users/kyvo/.claude/jobs/72354b36/tmp/pokefirered` (commit `ce6406d8`) is read-only data: read it, never build or run anything from it. Cite it as `fr: file: Function`.
- **Gen I/II untouched:** no change to `src/pokemon_data.c`, `src/pokemon_data_i.h`, `src/pokemon_table.c`, `src/pokemon_attribute.c`, `src/pokemon_char_encode.c`, `src/patch_list.c`, `src/named_list.c`, `src/*_nl.c`, `src/views/select_pokemon.c`, `src/views/trade.c`, or the Gen I/II scenes. Never modify `lib/flipper-gblink`. Western games only.
- **Fake independence (plan 3):** `tests/fake_ruby.{h,c}` include only `<string.h> <stdbool.h> <stdint.h> <stddef.h>` and `mlink_types.h`, `gen3_link_player.h`, `gen3_pkm.h`, `gen3_text.h`.
- **Fake FRLG fields:** the fake reads and writes the progress byte and card byte 0x38 by raw index, with the pokefirered offset in a comment. It never calls `gen3_link_player_progress`, `gen3_link_player_set_progress` or `gen3_trainer_card_build_frlg`, never includes `gen3_partner.h`, and every FRLG branch names the pokefirered function it models.
- **Budget, stack:** the app stack is 2 KB, so no scene or view keeps a local buffer over 128 bytes.
- **Budget, club:** `sizeof(Gen3Club)` ≤ 2,600 (plan 3's `test_club_size`).
- **Budget, size:** `text + data` of `dist/debug/pokemon_d.elf` grows by at most 4,000 bytes over the Task 1 baseline and stays ≤ 128,000. Measure with `$(ls -d ~/.ufbt/toolchain/*/bin | head -1)/arm-none-eabi-size dist/debug/pokemon_d.elf`.
- **Budget, heap:** on device, free heap stays ≥ 12,000 with a trade session running and Edit Pokemon open.
- **Save safety on the wire:** plan 3's rules hold unchanged for every partner (CCDD rule, DCBA only after ABCD, one 2FFE per GBA 2FFE, offers pass `gen3_trade_offer_check`). This plan adds: never put a Pokemon in a party block that the partner's own rules (§C3) forbid.
- **Names:** use plan 2's and plan 3's names exactly as they declare them. If the code differs from a name this plan uses, stop and report BLOCKED with the difference.
- **Emerald** is out of scope but not ruled out: every per-game choice is a `switch` on `gen3_link_game(…)` with an explicit `Gen3LinkGameE` case, which does what R/S does. Ignore the Switch-build-only paths (`REVISION >= 0xA`).
- **Code style:** root-relative includes (`#include <src/gen3/gen3_partner.h>`), `.clang-format`, snake_case functions and CamelCase typedefs. Multi-byte fields are written byte by byte, little-endian.
- **Setup in a fresh checkout:** `git submodule update --init`, then `ufbt update --channel=release` once.
- **Release:** `fap_version` becomes `[3,0]` in Task 7 only. PRs go to `LawlessNPC/Flipper-Zero-Game-Boy-Pokemon-Trading`; updating PR descriptions isn't part of this plan.
- **On-device checks:** implementers can't run them. UI tasks write their checklist into `docs/gen3/hardware-test-frlg.md`, and the user runs everything at the gate (Task 8).

## Review Focus

1. **A FireRed player without the National Dex, while the Flipper holds a Pokemon from #152 up.** This is the most common FRLG save, and the Flipper often holds such a Pokemon: the Treecko from the user's Ruby trades, a Hoenn `.pk3`, a Johto Pokemon. It also covers an edit that lands while the GBA links and sits, between sitting and round 1, or just after a trade. The GBA must never see that Pokemon in a party block, and the Flipper must say why and what to do. *(Task 1: `test_offer_traits`, `test_partner_refuse`; Task 3: `test_frlg_no_natdex_never_sits`, `test_frlg_unusable_before_round1`, `test_frlg_unusable_in_menu`; Task 4: `test_frlg_unusable_after_trade`; Task 5: `test_sync_frlg_no_natdex`, `test_guide_frlg_rows`; Task 6 checklist C.)*
2. **The FireRed player looks at the Flipper:** its avatar in the room, its trainer card (front and back), and the summary of its Pokemon in the trade menu. A card in the Ruby format, a version byte that doesn't match the claim, or `stars` above 4 can show garbage or index past `sKantoTrainerCardPals` (T13: other display paths were not traced). *(Task 1: `test_card_frlg_layout`; Task 2: `test_frlg_claim_on_wire`, the fake's "bad card" check; Task 6 checklist A.4–A.5, D.2.)*
3. **Swapping cartridges in one app session.** The user owns both games: Ruby, then FireRed, then Ruby again, without restarting the app or ending the trade session. The claim, the card, the exchange delays and the offer rules must follow the GBA linked now, never the one before. *(Task 2: `test_frlg_swap_cartridge`; Task 3: `test_frlg_swap_delays`; Task 6 checklist H.)*
4. **Mew or Deoxys without the fateful-encounter bit**, from a `.pk3` or a Ruby trade. FireRed answers YES with "The other TRAINER's POKéMON can't be traded now." and a BBCC 180 frames later. The Flipper must not offer such a Pokemon to FRLG, must say why, and must let the user set the bit. *(Task 1: `test_offer_traits`; Task 3: `test_frlg_unfateful_mew_refused`, `test_frlg_unfateful_mew_bypass`; Task 5: `test_sync_unfateful_mew`, `test_edit_set_fateful`; Task 6 checklist E.)*
5. **A deadline Ruby never had: FRLG's 300-frame limit on the 0x1144 relink** (`fr: trade_scene.c: CB2_LinkTrade` states 3/4 → `CheckLinkTimeout`). Anything that delays the Flipper's 0x1144 LinkPlayerBlock now ends in a link error right after YES, where Ruby simply waited. Causes include a long PRIMASK section, work added to the frame callback, or a stalled link queue. *(Task 4: `test_frlg_trade_relink_in_time`, `test_frlg_trade_relink_timeout`, `test_ruby_trade_relink_no_timeout`; Task 6 step 3, the ISR review; Task 6 checklist D.3.)*

## Design decisions

**D1. The claim follows the GBA's game** (§C1, §C2). The club decides per session from the GBA's LinkPlayerBlock of that session. Plan 3 D2 already has that block before the club sends its own.

| GBA `version & 0xFF` | Flipper `version` | `lp_field_2` | `name[8..10]` | Card | Round delays (first / later) |
|---|---|---|---|---|---|
| 1 Sapphire, 2 Ruby | 0x4002 (plan 3) | 0, or 0x8000 in the 0x1144 block with the opt-in | FF FF FF | Ruby layout (§A5) | 10 / 6 |
| 4 FireRed, 5 LeafGreen | 0x4000 + the GBA's byte (0x4004 / 0x4005) | 0x8000 in every block | 01 00 01 | FRLG card, byte 0x38 = 4 / 5 | 16 / 12 |
| 3 Emerald, anything else | as R/S (untested) | as R/S | as R/S | as R/S | 10 / 6 |

- **Why an FRLG version.** Against an R/S partner, `fr: trade.c: GetGameProgressForLinkTrade` refuses the linkup unless the FRLG player has `FLAG_SYS_CAN_LINK_WITH_RS`, which comes after the Sevii Islands. The refusal is a plain `CloseLink` with no 5FFF (`fr: cable_club.c: Task_LinkupExchangeDataWithLeader`). An FRLG version also selects the FRLG avatar (`fr: overworld.c: CreateLinkPlayerSprite`) and the Kanto card (`fr: trainer_card.c: GetCardType`).
- **Why the same byte.** Mirroring FR → FR and LG → LG makes the Flipper look like a real partner of the same game; nothing in the trade reads the difference.
- **Why progress 0x01** (§C1). `fr: trade.c: CanTradeSelectedMon` lets the FRLG player send Eggs and species above #151 to a non-R/S partner only when that partner's `progressFlagsCopy & 0x0F` is non-zero. The receiver copies LinkPlayer byte 0x10 (block 0x20) into 0x12 (block 0x22) (`fr: link.c: ConvertLinkPlayerName`). The high nibble stays 0. Bit 0x10 means "can link with R/S"; the nibble as a whole is the gate's "cleared the game" test, which only an Emerald GBA or our own gate applies.
- **Why 0x8000.** It is what a real FRLG sends (`fr: link.c: InitLocalLinkPlayer`), and FRLG never reads it (`fr: trade_scene.c: TryEnableNationalDexFromLinkPartner` is commented out).

**D2. Partner rules and offer traits** (§C3, T14).
- **Partner rules:** `{game, national_dex}`, taken from every GBA LinkPlayerBlock of any link type. They are kept across relinks like plan 3's trade context, and are `{Unknown, true}` until the first block arrives.
- **FRLG `national_dex`:** `progressFlags & 0x0F`, at block offset 0x20 (`name[8]`). Never read 0x22: the GBA never writes its own `progressFlagsCopy`, so it goes out stale.
- **Offer traits:**
  - `NonKanto`: National Dex number above 151, or no valid species.
  - `Unfateful`: Mew (151) or Deoxys (386) with the fateful bit 0.
- **Refusals:** only an FRLG partner refuses, in this order:
  - `NonKanto` without the National Dex → `Gen3RefuseNationalDex`. FRLG's receiver accepts such a Pokemon, but what happens next is unanalysed (`fr: pokemon.c: SpeciesToPokedexNum` returns 0xFFFF; T14). So the Flipper mirrors FRLG's sender rule (§C3 row 5).
  - Then `Unfateful` → `Gen3RefuseFateful` (`fr: trade.c: CheckValidityOfTradeMons`).
- **Unchanged from plan 3:** the egg, Mail, ENIGMA BERRY and item rules.
  - Items ≥ 349 stay refused for every partner. FRLG's items 349–374 are its key items (`fr: include/constants/items.h`, `ITEMS_COUNT` 375), and a Pokemon on the Flipper may go to Ruby next.
- **Not refused:** a Kanto Pokemon whose trade evolution leaves Kanto (Onix holding Metal Coat). FRLG evolves it with any real partner too (`fr: pokemon.c: GetEvolutionTargetSpecies`, `EVO_TRADE_ITEM`).

**D3. The club enforces the rules; the app explains them.**
- **Why the club.** Plan 3 D6 freezes the app tick while the user edits. So an offer published before the GBA linked can meet an FRLG player without the National Dex, and the check can't live only in the app.
- **The traits come with the offer.** The sync computes them from the editor's decrypted Pokemon. Then `usable = has_offer && gen3_partner_refuse(rules, traits) == None`. For R/S, `usable == has_offer`, so plan 3's behaviour is unchanged.
- **Room:** the club sends CAFE 0016 only while the offer is usable, and latches `party_mon = offer` in that same frame.
- **Round 1:** `party_mon = offer` only if the offer is usable. Otherwise `party_mon` keeps the Pokemon the GBA last saw or just traded us. Then `offer ≠ party_mon`, and plan 3's stale rule sends EEBB.
- **Menu:** `offer_stale` also holds while the offer isn't usable.
- **Finalize:** the GBA's former Pokemon becomes the offer. Its traits come from the AABB decrypt plan 3 already does (`gen3_trade_check_incoming`'s scratch).
- **Result:** no party block ever carries a Pokemon the partner's own rules forbid.

**D4. Exchange delays.**
- **Why.** FRLG runs 11-frame delay states before every CCCC (`fr: trade.c: BufferTradeParties` cases 2/6/10/14/18, `timer > 10`), on the child too. An FRLG master's round 1 therefore comes about 14 frames after both LinkPlayerBlocks (§C4: use ≥ 15; T1).
- **Values.** The club mirrors that master: 16 idle bursts before round 1 and 12 before rounds 2–5. The hard constraint is unchanged (the GBA must process round 1's CCCC at G0 + 3 or later), and the fake enforces it.
- **Override.** An explicit `gen3_club_set_round_delays` call (tests only) fixes both values for every partner until the next `gen3_club_init`.

**D5. Unchanged on the wire for FRLG** (§C4–§C6).
- **Same as Ruby:** the codes, 20-byte blocks, arbiter, ABCD/DCBA, three 2FFE syncs and plan 3's save windows (D5 there). The fake shows it.
- **5FFF word 1:** FRLG sends `gReadyCloseLinkType` there: 32 before the trade (`CB_WaitToStartTrade`), 12 on both-cancel (`CB_InitExitCanceledTrade`), 0 otherwise. The club reads only word 0 and answers `{5FFF, 0}`.
- **0x1144 deadline:** FRLG gives the relink 300 frames (Review Focus 5). Ruby gives it none.

**D6. The National Dex opt-in is for Ruby/Sapphire only** (plan 3 D10). FRLG ignores `lp_field_2`, and the dialog says so.

**D7. A `Fateful: Yes/No` editor row for Mew and Deoxys**, so the user can fix a loaded or traded one. A species change still sets the bit (plan 2 D5).

**D8. One fake with an FRLG mode, not a second fake.**
- **Selection:** `fake_ruby_init` with `me->version & 0xFF` equal to 4 or 5 turns the mode on, just as `me->link_type` picks the counter.
- **Why not a separate fake:** Part C says the link layer, command words, block sizes, link types, seats, room and save structure are identical. The roughly ten real differences are small branches. A separate `fake_frlg` would duplicate the whole plan 1–3 child stack and drift from it.
- **One bench for both:** with one model, every `club_bench.h` helper and every Ruby scenario runs unchanged against FireRed.
- **Strictness:** FRLG mode turns no Ruby check off. It adds FRLG's own checks: the linkup gate, the card check, the 0x1144 timeout and the trade rules.

## File map

| File | Responsibility |
|---|---|
| `src/gen3/gen3_link_player.{h,c}` | modify: version constants, FRLG progress byte, FRLG trainer card |
| `src/gen3/gen3_partner.{h,c}` | create: the GBA's game, partner rules, offer traits, partner refusals |
| `src/gen3/gen3_trade_block.h` | modify: two `Gen3Refuse` values |
| `src/gen3/gen3_club.{h,c}` | modify: claim per game, partner rules, delays per game, offers with traits |
| `src/gen3/gen3_trade_sync.{h,c}` | modify: partner-aware offer refusal, traits, guide rows |
| `src/gen3/gen3_edit.{h,c}` | modify: `gen3_edit_set_fateful` |
| `src/gen3/gen3_session_furi.{h,c}` | modify: `gen3_session_set_offer_traits` |
| `src/scenes/pokemon_gen3_trade.c`, `pokemon_gen3_edit.c`, `pokemon_gen3_natdex.c` | modify: tick, Fateful row, dialog text |
| `tests/fake_ruby.{h,c}`, `tests/club_bench.h`, `tests/trade_mons.h` | modify: FRLG mode, bench helpers, test Pokemon |
| `tests/test_gen3_partner.c`, `tests/test_gen3_frlg.c` | create: host tests |
| `tests/test_gen3_link_player.c`, `test_gen3_trade_sync.c`, `test_gen3_edit.c` | modify: host tests |
| `docs/gen3/hardware-test-frlg.md`, `README.md`, `changelog.md`, `application.fam` | checklist, docs, v3.0 |

---

### Task 1: Prerequisites, partner game, FRLG bytes and offer rules (pure)

**Files:**
- Modify: `src/gen3/gen3_link_player.h`, `src/gen3/gen3_link_player.c`
- Create: `src/gen3/gen3_partner.h`, `src/gen3/gen3_partner.c`
- Modify: `src/gen3/gen3_trade_block.h` (append two `Gen3Refuse` values), `src/gen3/gen3_trade_sync.c` (their R texts), `tests/trade_mons.h`
- Modify only if its `Gen3Refuse` sweep stops at `Gen3RefuseBadData`: `tests/test_gen3_trade_sync.c`
- Test: `tests/test_gen3_link_player.c`; create `tests/test_gen3_partner.c`

**Interfaces:**
- Consumes:
  - **plan 2:** `Gen3Pkm`, `gen3_pkm_get/set/fix_checksum/encrypt`, `Gen3FieldSpecies`, `Gen3FieldFateful`, `gen3_national_from_internal`, `GEN3_NATIONAL_MEW`, `GEN3_NATIONAL_DEOXYS`, `Gen3Editor`, `gen3_edit_init/set_species`;
  - **plan 1:** `Gen3LinkPlayer`, `gen3_link_player_block_build/parse`, `gen3_trainer_card_build`;
  - **plan 3:** `Gen3Refuse`, `gen3_trade_guide`, `tm_make`.
- Produces (`gen3_link_player.h`, appended):
  ```c
  #define GEN3_VERSION_SAPPHIRE  0x4001
  #define GEN3_VERSION_EMERALD   0x4003
  #define GEN3_VERSION_FIRERED   0x4004
  #define GEN3_VERSION_LEAFGREEN 0x4005
  #define GEN3_LP_PROGRESS_INDEX 8 /* name[8] = block 0x20: FRLG/Emerald progressFlags (fr: include/link.h) */
  #define GEN3_FRLG_PROGRESS_NATIONAL_DEX 0x01 /* bits 0-3 read as "has National Dex" (fr: trade.c CanTradeSelectedMon) */
  #define GEN3_FRLG_PROGRESS_CAN_LINK_RS  0x10 /* FLAG_SYS_CAN_LINK_WITH_RS (fr: link.c InitLocalLinkPlayer) */
  #define GEN3_FRLG_CARD_VERSION 0x38          /* struct TrainerCard.version (fr: include/trainer_card.h) */
  uint8_t gen3_link_player_progress(const Gen3LinkPlayer* lp);           /* name[8] */
  /* name[8] = flags, name[9] (neverRead) = 0, name[10] (progressFlagsCopy) = flags (§C1) */
  void gen3_link_player_set_progress(Gen3LinkPlayer* lp, uint8_t flags);
  /* §C2: bytes 0x00..0x37 as gen3_trainer_card_build, [0x38] = lp->version & 0xFF, 0x39..0x63 zero */
  void gen3_trainer_card_build_frlg(const Gen3LinkPlayer* lp, uint8_t out[GEN3_TRAINER_CARD_BLOCK_SIZE]);
  ```
- Produces (`gen3_trade_block.h`, appended to `Gen3Refuse` after `Gen3RefuseBadData`):
  ```c
  Gen3RefuseNationalDex, /* FRLG partner without the National Dex; offer above #151 (T14 policy) */
  Gen3RefuseFateful,     /* FRLG partner; Mew/Deoxys without the fateful-encounter bit (§C3) */
  ```
- Produces (`gen3_partner.h`, new; includes `gen3_link_player.h`, `gen3_pkm.h`, `gen3_trade_block.h`):
  ```c
  typedef enum { Gen3LinkGameUnknown, Gen3LinkGameRS, Gen3LinkGameE, Gen3LinkGameFRLG } Gen3LinkGame;
  /* version & 0xFF, as fr: trade.c GetGameProgressForLinkTrade: 1, 2 RS; 3 E; 4, 5 FRLG; anything else Unknown */
  Gen3LinkGame gen3_link_game(uint16_t version);
  typedef struct {
      Gen3LinkGame game;
      bool national_dex; /* FRLG: progressFlags & 0x0F; every other game: true (Ruby never refuses, §B2.5) */
  } Gen3PartnerRules;
  /* gba NULL -> {Unknown, true} */
  void gen3_partner_rules(const Gen3LinkPlayer* gba, Gen3PartnerRules* out);
  #define GEN3_NATIONAL_KANTO_LAST 151 /* fr: KANTO_SPECIES_END (SPECIES_MEW) */
  enum { Gen3OfferNonKanto = 1u << 0, Gen3OfferUnfateful = 1u << 1 };
  #define GEN3_OFFER_TRAITS_UNKNOWN (Gen3OfferNonKanto | Gen3OfferUnfateful) /* a game form that doesn't decrypt */
  uint8_t gen3_offer_traits(const Gen3Pkm* p); /* national 0 (no valid species) counts as NonKanto */
  /* D2: FRLG only. NonKanto without national_dex -> NationalDex; then Unfateful -> Fateful; else None. */
  Gen3Refuse gen3_partner_refuse(const Gen3PartnerRules* rules, uint8_t traits);
  ```
- Produces (`tests/trade_mons.h`, `static inline`): `void tm_make_unfateful(uint16_t national, uint32_t seed, uint8_t out[GEN3_PARTY_SIZE]);`. It runs `tm_make(national, 5, seed)`, then sets Fateful to 0, refixes the checksum and encrypts again.

- [ ] **Step 1: Check the prerequisites.**
  - Run `make -C tests`; it must pass.
  - `src/gen3/gen3_trade_block.h`, `src/gen3/gen3_trade_sync.h`, `tests/club_bench.h`, `tests/trade_mons.h` and `docs/gen3/hardware-test-2.md` must exist.
  - Every plan 2/3 name in this task's and later tasks' Consumes must exist with the signature its plan gives. If one differs, stop and report BLOCKED.
  - Run `ufbt` and `arm-none-eabi-size` (path in Global Constraints). Report `text + data`: it is the size baseline for this plan.
  - Report `BASE=$(git rev-parse HEAD)`. Task 7's diff check uses it.
- [ ] **Step 2: Write the failing tests.** In `tests/test_gen3_link_player.c`:

```c
TEST(test_lp_progress_bytes)  /* test_lp_build_layout's lp; set_progress(0x11): block[0x20] 0x11, [0x21] 0x00, [0x22] 0x11;
                                 name[0..7] unchanged; parse -> gen3_link_player_progress == 0x11 */
TEST(test_card_frlg_layout)   /* lp version 0x4004, gender 1, trainer_id 22222, name FLIPPER+EOS: out[0..0x37] ==
                                 gen3_trainer_card_build(lp)[0..0x37]; out[0x38] == 0x04; out[0x39..0x63] all 0;
                                 version 0x4005 -> out[0x38] == 0x05 */
```

And in `tests/test_gen3_partner.c`. Pokemon come from the plan-2 editor (`gen3_edit_init` seed 7, `gen3_edit_set_species`); "Fateful 0" means `gen3_pkm_set(…, Gen3FieldFateful, 0)` followed by `fix_checksum`.

```c
TEST(test_link_game)       /* 0x4001, 0x4002 RS; 0x4003 E; 0x4004, 0x4005, 0x0104 FRLG (low byte only); 0x4000, 0x4006, 0 Unknown */
TEST(test_partner_rules)   /* 0x4004 + progress 0x01 -> {FRLG, true}; 0x0F true; 0x11 true; 0x10 false; 0x00 false;
                              0x4002 with name[8] 0xFF or 0x00 -> {RS, true}; 0x4003 -> {E, true}; NULL -> {Unknown, true} */
TEST(test_offer_traits)    /* species 1: 0; 151 (fateful set by plan 2 D5): 0; 151 Fateful 0: Unfateful; 152: NonKanto;
                              252 (internal 277): NonKanto; 386: NonKanto; 386 Fateful 0: NonKanto|Unfateful; raw species 0: NonKanto */
TEST(test_partner_refuse)  /* RS, E, Unknown x {0, NonKanto, Unfateful, both}: None. FRLG natdex: 0 None, NonKanto None,
                              Unfateful Fateful, both Fateful. FRLG no natdex: 0 None, NonKanto NationalDex, Unfateful Fateful,
                              both NationalDex */
TEST(test_tm_make_unfateful) /* tm_make_unfateful(151, 9): decrypts (checksum valid), species 151, Fateful 0 */
```

- [ ] **Step 3: Run** `make -C tests`. Expected: compile failure, `gen3_partner.h` not found.
- [ ] **Step 4: Implement** the constants, helpers and module above.
  - In `gen3_trade_sync.c`, add the R texts (plan 3's reason line): NationalDex `National Dex needed.`, Fateful `it's not fateful.`.
  - Search `src/` and `tests/` for every `switch` over `Gen3Refuse` and every loop that ends at `Gen3RefuseBadData`. Each must cover the two new values; plan 3's `test_guide_line_lengths` sweep must reach `Gen3RefuseFateful`.
- [ ] **Step 5: Run** `make -C tests`. Expected: PASS, every older test included.
- [ ] **Step 6: Commit** `gen3: tell the partner's game from its LinkPlayer; FRLG link bytes and offer rules`. Stage the nine files, plus `tests/test_gen3_trade_sync.c` if you changed it.

---

### Task 2: FRLG linkup: the fake's FRLG mode, the claim and the card (§C1, §C2, §C5)

**Files:**
- Modify: `tests/fake_ruby.h`, `tests/fake_ruby.c`, `tests/club_bench.h`, `src/gen3/gen3_club.h`, `src/gen3/gen3_club.c`
- Create: `tests/test_gen3_frlg.c`

**Interfaces:**
- Consumes: Task 1. Plan 3: `gen3_club_claim_game` (signature unchanged), `Gen3ClubStatus.trainer`/`trainer_valid`, bench helpers `bench_init_trade`, `bench_to_room`, `bench_raw_relink`, `reach_room`, `leave_room`, `run_until_room`.
- Produces (`fake_ruby.h`):
  ```c
  bool fake_ruby_is_frlg(const FakeRuby* g); /* me->version & 0xFF is 4 or 5 at fake_ruby_init */
  /* FRLG mode, before fake_ruby_open_link: the save's National Dex and FLAG_SYS_CAN_LINK_WITH_RS.
   * Default both false (the strict case). Writes me.name[8] (fr: link.c InitLocalLinkPlayer). */
  void fake_ruby_set_frlg_progress(FakeRuby* g, bool national_dex, bool can_link_rs);
  const uint8_t* fake_ruby_partner_card(const FakeRuby* g); /* the Flipper's 100-byte card block as received */
  /* fake_ruby_error adds: "not ready" (linkup gate: closes, no 5FFF), "bad card" */
  ```
- Produces (`gen3_club.h`): `Gen3ClubStatus` appended `Gen3PartnerRules partner_rules;` (D2).
- Produces (`club_bench.h`, `static inline`):
  - `Bench` gains `bool natdex, can_link_rs;` and `uint16_t g_close_type[16]; uint8_t n_g_close;`. The second pair is word 1 of every GBA 5FFF: `step()` and `bench_raw_relink` record rx word 2 of each 9-word burst whose rx word 1 is 0x5FFF.
  - `void bench_init_frlg(Bench* b, uint16_t version, bool natdex)` works like `bench_init_trade`, but with an FRLG fake:
    - FireRed: name "RED", gender 0, TID 13579, SID 24680. LeafGreen (0x4005): name "LEAF", gender 1.
    - It calls `fake_ruby_set_frlg_progress(natdex, false)`.
    - Default offer `tm_make(4, 5, 21)` (CHARMANDER, Kanto); party as plan 3's bench.
  - `Gen3LinkPlayer bench_frlg_player(uint16_t version)` returns the fake LinkPlayer `bench_init_frlg` uses; `Gen3LinkPlayer bench_ruby_player(void)` returns plan 3's RUBY / 12345 / 54321 one. Both use link type 0x1133.
  - `void bench_swap_gba(Bench* b, const Gen3LinkPlayer* me, bool natdex)`: a fresh fake with `me` and the same club, with progress applied if FRLG. It doesn't open the link.
  - `ruby_relink` re-applies `natdex` / `can_link_rs` after `fake_ruby_init`.

Fake FRLG mode. Each point models the cited code, and every Ruby check stays on:
1. **Own LinkPlayer** (`fr: link.c: InitLocalLinkPlayer`):
   - `lp_field_2` is 0x8000.
   - `name[8]` = (National Dex ? 0x01 : 0) | (can link with R/S ? 0x10 : 0).
   - `name[9]` = 0, and `name[10]` = 0: its own `progressFlagsCopy` is never written.
2. **Received LinkPlayers** (`fr: link.c: ProcessRecvCmds`, `ConvertLinkPlayerName`), applied to every slot:
   - a sender `version & 0xFF` of 1 or 2: `name[8..10]` = 0;
   - otherwise: `name[10] = name[8]`.
3. **Linkup gate**, on the 0x1133 link only (`fr: link.c: GetLinkPlayerDataExchangeStatusTimed` → `fr: trade.c: GetGameProgressForLinkTrade`). Once both link types match:
   - an FRLG partner is ready;
   - an R/S partner needs its own `name[10] & 0xF0`;
   - any other partner needs its own and the partner's `name[10] & 0xF0`.
   - Not ready: `CloseLink` with no 5FFF (`fr: cable_club.c: Task_LinkupExchangeDataWithLeader`), error "not ready", phase Closed.
4. **Trainer card** (`fr: trainer_card.c: TrainerCard_GenerateCardForLinkPlayer`):
   - Its own card: plan 1's card bytes, plus `[0x38] = me.version & 0xFF`.
   - On receiving the Flipper's card (`fr: cable_club.c: Task_LinkupAwaitTrainerCardData`): keep a copy for `fake_ruby_partner_card`. Error "bad card" when `card[0x01] > 4`, or when the Flipper's version byte is 4 or 5 and `card[0x38]` differs from it.
5. **Room, seats, walk-in, 0x1111 re-establish, 300-frame tasks:** Ruby's code unchanged (§C5; `fr: cable_club.c: Task_WaitForLinkPlayerConnection`, cartridge build).

Club behaviour (D1, D2):
- **`gen3_club_claim_game`** switches on `gen3_link_game(partner ? partner->version : 0)`:
  - FRLG: version `0x4000 | (partner->version & 0xFF)` and `lp_field_2` 0x8000 for every link type (`national_dex` ignored). Then `gen3_link_player_set_progress(lp, GEN3_FRLG_PROGRESS_NATIONAL_DEX)` and card `gen3_trainer_card_build_frlg`. Every other field is as plan 3.
  - RS, E and Unknown: plan 3's Ruby claim, byte for byte.
- **`partner_rules`** are updated from every GBA LinkPlayerBlock that parses, of any link type, before the club's own block is built. Disconnected doesn't reset them; `gen3_club_init` sets them to `{Unknown, true}`.

- [ ] **Step 1: Write the fake mode, the bench helpers and the failing tests** in `tests/test_gen3_frlg.c`. Unless a test says otherwise, it starts with `bench_init_frlg(&b, GEN3_VERSION_FIRERED, false)`, and "reach_room" means `bench_to_room`. Club config: `{"FLIPPER", 22222, 0, 0, true}`.

```c
TEST(test_frlg_claim_game)      /* direct call, partner version 0x4004: for each link type 0x1133/0x1122/0x1144/0x1111 and national_dex
                                   on/off: version 0x4004, lp_field_2 0x8000, trainer_id 22222, gender 0, id 0, language 2, link_type as given,
                                   name C0 C6 C3 CA CA BF CC FF 01 00 01; card == gen3_trainer_card_build_frlg(&lp), card[0x38] == 4.
                                   Partner 0x4005: version 0x4005, card[0x38] 5. Partner NULL, 0x4001, 0x4002, 0x4003, 0x4000: plan 3's
                                   Ruby bytes (version 0x4002, name ... CC FF FF FF FF, lp_field_2 0x8000 only for 0x1144 + on) */
TEST(test_frlg_gba_block)       /* natdex true: reach_room; status partner version 0x4004, lp_field_2 0x8000, name[8] 0x01, name[10] 0x00;
                                   partner_rules {FRLG, true}; trainer "RED" 13579/24680. natdex false: name[8] 0x00, rules {FRLG, false} */
TEST(test_frlg_claim_on_wire)   /* reach_room; fake error NULL; fake_ruby_partner(): version 0x4004, lp_field_2 0x8000, trainer_id 22222,
                                   name decodes "FLIPPER", name[8] 0x01, name[10] 0x01; partner card [0x01] 0, [0x02] 1, [0x0E..0x0F] CE 56,
                                   [0x30..0x37] C0 C6 C3 CA CA BF CC FF, [0x38] 0x04, [0x39..0x63] all 0 */
TEST(test_frlg_leafgreen)       /* bench_init_frlg(0x4005, true): reach_room; partner version 0x4005; card[0x38] 0x05; error NULL */
TEST(test_frlg_ruby_claim_refused) /* fake_ruby_open_link; bench_raw_relink(the plan 3 Ruby claim from gen3_club_claim_game(cfg, NULL,
                                   0x1133, false, ...), 200): fake error "not ready", n_g_close 0, fake Closed. Same with
                                   fake_ruby_set_frlg_progress(false, true): error NULL after 200 frames (gate passed) */
TEST(test_frlg_room_and_leave)  /* reach_room; 1800 frames: fake Room, error NULL; club Room; link.checksum_errors 0;
                                   fake_ruby_leave_room: one Flipper CAFE 0017 and one 5FFF; fake Closed; club Waiting; error NULL */
TEST(test_frlg_link_test)       /* bench_init_frlg, then gen3_club_init with {"FLIPPER", 22222, 0, 0, false} (Link Test): reaches Room
                                   with the FRLG claim (partner version 0x4004); fake_ruby_sit(0), 600 frames: sent_cafe16 == 0 */
TEST(test_frlg_swap_cartridge)  /* bench_init_trade (Ruby): reach_room; leave_room. bench_swap_gba(bench_frlg_player(0x4004), false);
                                   reach_room: fake_ruby_partner() version 0x4004, name[8] 0x01; trainer "RED"; rules {FRLG, false};
                                   leave_room. bench_swap_gba(bench_ruby_player(), false); reach_room: version 0x4002, lp_field_2 0,
                                   name[7..10] FF FF FF FF; rules {RS, true}; trainer "RUBY". Fake error NULL throughout */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail; every older test passes.
- [ ] **Step 3: Implement** the club behaviour above.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS, including every plan 1–3 club and fake test.
- [ ] **Step 5: Commit** `gen3: claim FireRed/LeafGreen to an FRLG partner, with its trainer card`.

---

### Task 3: FRLG trade menu: exchange delays and the offer rules in the club (§B2, §C3, §C4)

**Files:**
- Modify: `tests/fake_ruby.h`, `tests/fake_ruby.c`, `tests/club_bench.h`, `src/gen3/gen3_club.h`, `src/gen3/gen3_club.c`, `tests/test_gen3_frlg.c`

**Interfaces:**
- Consumes: Tasks 1–2. Plan 3: `gen3_club_set_offer`, `gen3_club_set_round_delays`, `GEN3_CLUB_ROUND1_DELAY_BURSTS`, `GEN3_CLUB_ROUND_DELAY_BURSTS`, `gen3_trade_check_incoming`, `offer_stale`, `bench_offer`, `bench_reach_menu`, `bench_set_party`, `bench_codes`, `fake_ruby_offer/answer/press_a/resit`, `fake_ruby_enemy`, `fake_ruby_menu_state`.
- Produces (`fake_ruby.h`):
  ```c
  typedef enum { /* fr: trade.c CANT_TRADE_* (cartridge build, no BUGFIX) */
      FakeRubyCantTradeNone, FakeRubyCantTradeLastMon, FakeRubyCantTradeNational,
      FakeRubyCantTradeInvalidMon, FakeRubyCantTradeEggYet, FakeRubyCantTradePartnerEggYet,
  } FakeRubyCantTrade;
  FakeRubyCantTrade fake_ruby_cant_trade(const FakeRuby* g); /* latest local refusal of fake_ruby_offer; None after an AABB */
  bool fake_ruby_partner_mon_invalid(const FakeRuby* g);      /* latest YES found the Flipper's Mew/Deoxys unfateful */
  uint16_t fake_ruby_max_enemy_species(const FakeRuby* g);    /* highest internal species (egg 412) in any Flipper party
                                                                 block this GBA stored since init; 0 if none */
  ```
- Produces (`gen3_club.h`):
  ```c
  #define GEN3_CLUB_ROUND1_DELAY_BURSTS_FRLG 16 /* FRLG master ~14 frames (fr: trade.c BufferTradeParties); §C4 >= 15 */
  #define GEN3_CLUB_ROUND_DELAY_BURSTS_FRLG 12  /* the FRLG master's 11-frame delay before every CCCC, + 1 */
  /* As gen3_club_set_offer, with the offer's gen3_offer_traits (ignored when game is NULL). No decrypt: the
   * session calls this inside its PRIMASK section. false and nothing changed if trades_seen != trades_done. */
  bool gen3_club_set_offer_traits(Gen3Club* club, const uint8_t* game, uint32_t trades_seen, uint8_t traits);
  /* gen3_club_set_offer (plan 3) keeps its signature: it now decrypts game for its traits
   * (GEN3_OFFER_TRAITS_UNKNOWN if that fails) and calls gen3_club_set_offer_traits. Tests and app context only. */
  /* Gen3ClubStatus, appended: */
      uint8_t offer_traits;
      bool offer_usable; /* has_offer && gen3_partner_refuse(&partner_rules, offer_traits) == Gen3RefuseNone */
  ```

Fake FRLG mode, continued:
1. **Trade menu creation** (`fr: trade.c: CB2_CreateTradeMenu`): as Ruby, plus state 5, one frame before the exchange.
2. **Party exchange** (`fr: trade.c: BufferTradeParties`, cases 0–21):
   - Plan 3's copy, wait and reset cases, renumbered.
   - Delay cases 2, 6, 10, 14 and 18 hold the child for 11 frames (`timer > 10`).
   - The round-1 deadlock and the "stale block" and "exchange stalled" errors keep plan 3's rules.
   - Every Flipper party block the fake stores updates `max_enemy_species` (decrypt, Species).
3. **`fake_ruby_offer`** (`fr: trade.c: CanTradeSelectedMon`, the cartridge code without `BUGFIX`), in this order:
   - Without its own National Dex: a species-or-egg above 151 (an Egg is 412) → National.
   - If the Flipper's version isn't R/S and the Flipper's `name[10] & 0x0F` is 0: an Egg → PartnerEggYet; a species above 151 → InvalidMon.
   - Its own Mew or Deoxys with Fateful 0 → InvalidMon.
   - No other non-egg Pokemon → LastMon.
   - Any refusal sends nothing and opens Message (`QUEUE_MON_CANT_BE_TRADED` etc.); A goes back to Browsing.
4. **YES on "Is this trade okay?"** (`fr: trade.c: CommunicateWhetherMonCanBeTraded` → `CheckValidityOfTradeMons`): if the Flipper's slot (DDDD arg) holds species 151 or 410 with Fateful 0, it sets `partner_mon_invalid` and shows the message. It sends BBCC 180 frames later (`QueueAction(180, QUEUE_SEND_DATA)`) → Standby; the Flipper's DDEE → Message.

Club behaviour (D3, D4):
- **Round delays:** when the GBA's 0x1122 block parses, take 16 / 12 for FRLG and plan 3's 10 / 6 otherwise. A `gen3_club_set_round_delays` call since init wins.
- **Usable offer:**
  - Room: sit (CAFE 0016) only when `offer_usable`, and set `party_mon = offer` in that frame. This replaces plan 3's `has_offer` test.
  - Round 1: `party_mon = offer` only when `offer_usable`.
  - Menu: `offer_stale` also holds while `!offer_usable`.
- **Pending traits:** on the AABB answered with DDDD, `pending_traits = gen3_offer_traits(&scratch)` (the decrypt plan 3 already does). At finalize (Done or Assumed), `offer_traits = pending_traits`.

- [ ] **Step 1: Write the fake mode and the failing tests.**
  - Unless a test says otherwise, it starts as in Task 2: `bench_init_frlg(&b, GEN3_VERSION_FIRERED, false)`.
  - Y = the default offer CHARMANDER, T = TREECKO `tm_make(252, 5, 21)`, M = `tm_make_unfateful(151, 22)`.
  - A margin is `f_cccc_burst[k]` minus the later `last_burst` of the two blocks before it.
  - Set a G party with `bench_set_party` before `bench_reach_menu`.
  - Partner rules are `{Unknown, true}` until the GBA links, so check `offer_usable` only after `reach_room`.

  The tests:

```c
TEST(test_frlg_exchange)          /* bench_reach_menu: Browsing; club Menu, round 5; enemy_count 1, enemy(0) == Y;
                                     Flipper CCCC words 0001 0001 0001 0003 0004, once each; error NULL */
TEST(test_frlg_exchange_margins)  /* round 1 margin >= 17; rounds 2..5 margins >= 13 */
TEST(test_frlg_round1_delay_needed) /* set_round_delays(0, 12): fake error "stale block" or "exchange stalled" within 900 frames */
TEST(test_frlg_no_natdex_never_sits) /* bench_offer(T); reach_room: offer_traits NonKanto, offer_usable false; sit(1); 900 frames:
                                     sent_cafe16 == 0; fake Seated; max_enemy_species 0; error NULL */
TEST(test_frlg_natdex_sits)       /* natdex true, bench_offer(T); bench_reach_menu: offer_usable; enemy(0) == T */
TEST(test_frlg_unusable_before_round1) /* reach_room; sit(1); in the frame sent_cafe16 becomes 1: bench_offer(T) (true); run to Room again:
                                     max_enemy_species 4 (Y only); f_codes {EEBB}; no DDDD; fake passed Return; error NULL */
TEST(test_frlg_unusable_in_menu)  /* bench_reach_menu; bench_offer(T): one EEBB, header within 12 bursts; fake Room; resit(); 600 frames:
                                     sent_cafe16 unchanged; max_enemy_species 4 */
TEST(test_frlg_gba_cant_offer_non_kanto) /* G party {A, B, tm_make(152,5,14)}: offer(2): g_codes empty, cant_trade National, Message;
                                     press_a -> Browsing */
TEST(test_frlg_gba_offers_non_kanto) /* natdex true, same party: offer(2): g_codes {AABB 2}; one DDDD 0 (our byte 0x20 = 0x01 let it) */
TEST(test_frlg_gba_egg)           /* G {A, tm_make_egg, X}. natdex true: offer(1): AABB 1 -> DDEE, refused Egg.
                                     natdex false: offer(1): nothing sent, cant_trade National (412 > 151) */
TEST(test_frlg_unfateful_mew_refused) /* natdex true, bench_offer(M); reach_room: offer_traits Unfateful, offer_usable false;
                                     sit(1), 900 frames: sent_cafe16 0 */
TEST(test_frlg_unfateful_mew_bypass)  /* natdex true; gen3_club_set_offer_traits(&b.club, M, 0, 0) (lying traits); bench_reach_menu;
                                     offer(2) -> DDDD; answer(yes): partner_mon_invalid; no BBBB; the GBA's BBCC header 180..190 bursts after
                                     the fake left Prompt; one DDEE; f_codes {DDDD, DDEE}; no CCDD; fake Message; error NULL */
TEST(test_frlg_swap_delays)       /* bench_init_trade (Ruby), bench_offer(Y): bench_reach_menu: round 1 margin 11..16; fake_ruby_power_off,
                                     120 frames: club Waiting. bench_swap_gba(bench_frlg_player(0x4004), false): bench_reach_menu:
                                     margin >= 17; power off. bench_swap_gba(bench_ruby_player(), false): bench_reach_menu: margin 11..16.
                                     Error NULL throughout (15 CCCC fit f_cccc[16]) */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail; every older test passes. For Ruby, `offer_usable == has_offer`, so no plan-3 assertion changes.
- [ ] **Step 3: Implement** the club behaviour.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS, including `test_club_size` (≤ 2,600).
- [ ] **Step 5: Commit** `gen3: FRLG trade menu: mirror its exchange delays, never show a Pokemon it can't take`.

---

### Task 4: FRLG trade, link save, relinks and leaving (§B4–§B7, §C4, §C6)

**Files:**
- Modify: `tests/fake_ruby.h`, `tests/fake_ruby.c`, `tests/test_gen3_frlg.c`
- Modify only if a test below exposes a club bug (name it in the report): `src/gen3/gen3_club.c`

**Interfaces:**
- Consumes: Tasks 2–3. Plan 3: `bench_trade`, `bench_mute`, `fake_ruby_saved_trades`, `fake_ruby_save_corrupt`, `fake_ruby_national_dex`, `fake_ruby_party`, `fake_ruby_relink_frames`, `fake_ruby_power_off`, `gen3_club_get_received`, `gen3_club_set_national_dex`, `gen3_club_request_leave`, `Gen3TradeResult*`.
- Produces: no new API. `fake_ruby_error` adds "link timeout" in the 0x1144 relink (FRLG only).

Fake FRLG mode, continued:
1. **5FFF word 1** (`fr: link.c: BuildSendCmd`, `gReadyCloseLinkType`): 32 from `CB_WaitToStartTrade` (after CCDD), 12 from `CB_InitExitCanceledTrade` (after EEBB), 0 from every `SetCloseLinkCallback`.
2. **0x1144 relink** (`fr: trade_scene.c: CB2_LinkTrade`): Ruby's timeline, and the child in state 4 runs `CheckLinkTimeout`. More than 300 frames without the exchange: `CloseLink`, then error "link timeout".
3. **`TradeMons`** (`fr: trade_scene.c: TradeMons`): Ruby's swap, Friendship 70 and mail. `lp_field_2` never sets the National Dex, so `fake_ruby_national_dex` stays false.
4. **Link save** (`fr: trade_scene.c: CB2_SaveAndEndTrade`, cartridge): the child path is Ruby's, including state 40's 51 frames. `Random() % 30` is master-only.

- [ ] **Step 1: Write the fake mode and the failing tests.**
  - Unless a test says otherwise, it starts with `bench_init_frlg(&b, GEN3_VERSION_FIRERED, false)` and `bench_reach_menu`.
  - G party is `{A, B, X = tm_make(7,10,13)}` and the offer is Y = `tm_make(4,5,21)`.
  - P = PIKACHU `tm_make(25,12,31)`, T = TREECKO `tm_make(252,5,21)`.

```c
TEST(test_frlg_round_trip)      /* bench_trade(2): saved_trades 1, save_corrupt false; saved party[2] decrypts to Y with Friendship 70;
                                   trades_done 1, last_result Done, get_received == X; post-save exchange enemy(0) == X; club Menu;
                                   f_codes {DDDD, CCDD, DCBA}; exactly 3 Flipper 2FFE between DCBA and the GBA's last 5FFF; error NULL */
TEST(test_frlg_close_types)     /* from sitting through one bench_trade: g_close_type == {0, 32, 0}; each Flipper 5FFF burst has word 2 == 0 */
TEST(test_frlg_save_window)     /* save_window false until save_syncs 1, true at 1..2, false from 3; trades_done 0 -> 1 in that frame */
TEST(test_frlg_trade_relink_in_time) /* the 0x1144 relink_frames < 100 */
TEST(test_frlg_trade_relink_timeout) /* offer(2), answer(yes); at fake phase Trade: bench_mute(true): fake error "link timeout"
                                   355..375 frames later (OpenLink +2, 61 frames, then > 300 in state 4); saved_trades 0; unmute 60
                                   frames: club Waiting, trades_done 0; ruby_relink (power cycle), reach_room: last_result Canceled,
                                   trade_pending false (plan 3 D3) */
TEST(test_ruby_trade_relink_no_timeout) /* Ruby bench_init_trade, same mute at Trade for 1200 frames: fake error NULL (§B4.1: no deadline) */
TEST(test_frlg_national_dex_ignored) /* set_national_dex(true): fake_ruby_partner()->lp_field_2 0x8000 in Room, Menu and Trade;
                                   after bench_trade fake_ruby_national_dex false */
TEST(test_frlg_lost_in_window)  /* plan 3's test_trade_lost_in_window on FRLG: fake "lag", save_corrupt; club RolledBack, trades_done 0;
                                   saved party == b.party */
TEST(test_frlg_lost_after_sync2)/* plan 3's test_trade_lost_after_sync2 on FRLG: club Assumed, trades_done 1, get_received == X */
TEST(test_frlg_unusable_after_trade) /* bench_trade setup; in the frame trades_done becomes 1: bench_offer(T) (true): post-save exchange
                                   enemy(0) == X; one unsolicited EEBB; fake Return then Room; max_enemy_species <= 151 */
TEST(test_frlg_edit_resit)      /* bench_trade(2); bench_offer(P): EEBB; g_close_type last == 12; Room via 0x1111, resit_needed;
                                   fake_ruby_partner() "FLIPPER", 22222, 0x4004 (CheckLinkPlayersMatchSaved passed); resit(): enemy(0) == P;
                                   bench_trade(2): saved_trades 2; error NULL */
TEST(test_frlg_natdex_trade_non_kanto) /* bench_init_frlg(FireRed, true), bench_offer(T), bench_reach_menu: bench_trade(2): saved
                                   party[2] species internal 277 */
TEST(test_frlg_leave)           /* bench_init_frlg + bench_to_room: request_leave: one CAFE 0017, fake Closed, left. Then from the
                                   default start (menu): request_leave: EEBB -> Room -> CAFE 0017 -> Closed; left */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail (no FRLG close types, no timeout); every older test passes.
- [ ] **Step 3: Implement** the fake mode. Fix the club only for a real club bug, and name it in the report.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: FRLG trade, link save and room return, with FireRed's relink deadline`. Stage the three files, plus `src/gen3/gen3_club.c` if you changed it.

---

### Task 5: Sync, guide and editor for FRLG (pure)

**Files:**
- Modify: `src/gen3/gen3_trade_sync.h`, `src/gen3/gen3_trade_sync.c`, `src/gen3/gen3_edit.h`, `src/gen3/gen3_edit.c`
- Test: `tests/test_gen3_trade_sync.c`, `tests/test_gen3_edit.c`

**Interfaces:**
- Consumes: Task 1; Task 2's `Gen3ClubStatus.partner_rules`. Plan 3: `Gen3TradeSync`, `gen3_trade_sync_poll`, `gen3_trade_sync_offer_taken`, `gen3_trade_guide`, `GEN3_GUIDE_LINES`, `GEN3_GUIDE_LINE_LEN`. Plan 2/3: `Gen3Editor`, `pristine`, the commit step, the test helper `edit_consistent`.
- Produces:
  ```c
  /* Gen3TradeSync, appended: */
      uint8_t offer_traits; /* gen3_offer_traits(&ed->pkm) when offer_valid, else 0 */
  /* gen3_trade_sync_poll step 3 becomes: offer_refuse = gen3_trade_offer_check(&ed->pkm);
   * offer_valid = (offer_refuse == None); when valid: offer_traits, then
   * offer_refuse = gen3_partner_refuse(&st->partner_rules, offer_traits). The offer is still published while
   * the partner refuses it (the club decides, D3), and re-offered only when (offer_valid, offer) changes. */
  /* gen3_edit.h: set the fateful-encounter bit (Gen3FieldFateful), then plan 2's commit step; clears pristine. */
  void gen3_edit_set_fateful(Gen3Editor* ed, bool on);
  ```

Guide rows: insert these two rows just before plan 3's `offer_refuse ≠ None` row.

| Case | Line 1 | Line 2 | Line 3 |
|---|---|---|---|
| `offer_refuse` NationalDex | `Can't offer it: GBA` | `has no National Dex.` | `Get it, or pick 1-151` |
| `offer_refuse` Fateful | `Can't offer it: FRLG` | `needs Fateful: Yes on` | `Mew/Deoxys: edit it.` |

- [ ] **Step 1: Write the failing tests.** Statuses carry `partner_rules` as given, with no `refused` set. T = an editor at species 252; Mew = species 151.

```c
TEST(test_sync_frlg_no_natdex)    /* {FRLG, false}, T: poll -> Offer; offer_valid; offer == encrypt(ed.pkm); offer_traits NonKanto;
                                     offer_refuse NationalDex */
TEST(test_sync_frlg_natdex)       /* {FRLG, true}, T: offer_refuse None, offer_traits NonKanto */
TEST(test_sync_ruby_ignores_rules)/* {RS, true}: T -> None; Mew with set_fateful(false) -> None */
TEST(test_sync_unfateful_mew)     /* {FRLG, true}, Mew, set_fateful(false): poll -> Offer, traits Unfateful, refuse Fateful;
                                     set_fateful(true): poll -> Offer, traits 0, refuse None */
TEST(test_sync_partner_change)    /* {RS}, T: Offer, offer_taken; same editor, status {FRLG, false}: poll -> 0; offer_refuse NationalDex */
TEST(test_sync_check_before_partner) /* {FRLG, false}, T holding item 121: offer_valid false, offer_refuse Mail, offer_traits 0 */
TEST(test_guide_frlg_rows)        /* the two rows above, exact lines; with !cable_ok, save_window or trade_pending set, plan 3's earlier
                                     rows still win (one case each) */
TEST(test_edit_set_fateful)       /* init; set_species(151): Fateful 1; set_fateful(false): Fateful 0, pristine false, edit_consistent,
                                     raw equal except 0x4F bit 7 and 0x1C..0x1D; set_fateful(true): Fateful 1 */
```

  Plan 3's `test_guide_line_lengths` already sweeps the new `Gen3Refuse` values (Task 1).
- [ ] **Step 2: Run** `make -C tests`. Expected: the new tests fail.
- [ ] **Step 3: Implement** the sync step, the rows and `gen3_edit_set_fateful`.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: tell the user why FireRed can't take the Pokemon; fateful flag in the editor model`.

---

### Task 6: Session, trade tick, Fateful row, National Dex text; FireRed checklist

**Files:**
- Modify: `src/gen3/gen3_session_furi.h`, `src/gen3/gen3_session_furi.c`, `src/scenes/pokemon_gen3_trade.c`, `src/scenes/pokemon_gen3_edit.c`, `src/scenes/pokemon_gen3_natdex.c`
- Create: `docs/gen3/hardware-test-frlg.md`

**Interfaces:**
- Consumes: Tasks 3 and 5; plan 3's tick (`gen3_trade_tick`) and `Gen3NatDex` dialog.
- Produces:
  ```c
  /* Replaces plan 3's gen3_session_set_offer (delete it; the tick was its only caller).
   * gen3_club_set_offer_traits inside the PRIMASK section: copies the 100-byte offer and the traits byte. */
  bool gen3_session_set_offer_traits(Gen3Session* s, const uint8_t* offer, uint32_t trades_seen, uint8_t traits);
  ```

Behaviour:
- **Tick:** on `Gen3SyncOffer`, call `gen3_session_set_offer_traits(s, offer_valid ? offer : NULL, trades_seen, sync->offer_traits)`. If it returns true, call `gen3_trade_sync_offer_taken`.
- **Editor row** `Fateful: Yes|No`, right after `Ball`, shown only for national 151 and 386.
  - Its index is `#define GEN3_EDIT_ROW_FATEFUL (PokemonSceneNum + 1)`. Plan 3's OT row already uses `PokemonSceneNum`.
  - OK calls `gen3_edit_set_fateful(ed, !current)` and rebuilds the menu, keeping the highlight.
- **`Gen3NatDex` text:** `Ruby/Sapphire only.\nThe next trade unlocks\nit for good.` (D6). The header and buttons are as plan 3.

- [ ] **Step 1: Implement** the function, the tick change, the row and the text.
- [ ] **Step 2: Build** `ufbt` and the dev SDK. Expected: 0 warnings. Run `make -C tests`: PASS.
- [ ] **Step 3: Review the ISR path** (Review Focus 5) and put the answers in your report:
  - the frame callback still calls only `gen3_club_frame`, `gen3_club_status` and `gen3_log_frame`;
  - `gen3_partner_refuse` is the only new code the ISR reaches, and it is a few compares;
  - every PRIMASK section copies at most one Pokemon plus scalars, and calls nothing that decrypts or blocks.
- [ ] **Step 4: Write `docs/gen3/hardware-test-frlg.md`.**
  - **Intro:**
    - Goal: trades with FireRed (LeafGreen works the same way; note it if you test one), plus the Link Test, the editor and `.pk3` files with FireRed.
    - You need: the Flipper with the Game Link module, a GBA, a GBA link cable, genuine FireRed, and genuine Ruby for H.
    - Build `ufbt` (release) and install `dist/pokemon.fap`.
    - The log is `apps_data/pokemon/gen3_link.log`, overwritten at each Trade or Link Test start.
    - FireRed's Trade Center is on Pokemon Center 2F: the right-hand attendant, then Trade Center.
    - First write down whether your FireRed has the National Dex (the Pokedex offers a NATIONAL list).

  Then these sections:
  - **A. Link Test with FireRed**
    1. FireRed: at least 2 Pokemon, none holding the ENIGMA BERRY, one you're happy to trade; saved.
    2. Plug the cable as in test 1, same plug in the Flipper.
    3. Gen III → `Link Test`, then enter the Trade Center on FireRed.
    4. The Flipper reaches `in Trade Center` and shows your FireRed name and TID. On FireRed the second avatar is a FireRed/LeafGreen player (Red), not Brendan or May (Review Focus 2).
    5. Talk to it: a Kanto-style card with `FLIPPER` and ID 22222. Press A to flip it, then flip it back: no garbage, no crash.
    6. Walk for 60 s (`Err` stays 0), then walk out: no error.
  - **B. Start a trade.** Start the app fresh → Gen III → `Trade` → enter the Trade Center (`Linking...`, then `In the Trade Center: ...`). Gen III menu → Edit Pokemon: the OT is your FireRed name and IDs.
  - **C. National Dex rules** (Review Focus 1)
    - **C1, without the National Dex:**
      1. Set the species to TREECKO and go back to the trade screen: `Can't offer it: GBA` / `has no National Dex.` / `Get it, or pick 1-151`.
      2. Sit on the GBA: "Please wait..." stays; B stands up.
      3. Set CHARMANDER and sit: the menu shows CHARMANDER.
      4. If FireRed holds a Pokemon from #152 up, TRADE on it: FireRed itself refuses ("can't be traded now").
      5. In the menu, set TREECKO on the Flipper: FireRed goes back to the room and never shows TREECKO; sitting again does nothing.
    - **C2, with the National Dex:** set TREECKO and sit: the menu shows TREECKO. Trade it in D.
  - **D. Trade both ways**
    1. Run test 2's C–D with FireRed: trade one of yours to the Flipper, edit it, sit again (Up, then Down), and trade it back. On FireRed it has you as OT and obeys.
    2. In the trade menu, open SUMMARY on the Flipper's Pokemon: every page shows, with no crash; write down the "met" text (Review Focus 2).
    3. Answer YES while the Flipper shows the Gen III menu rather than the trade screen: the trade still starts with no link error (Review Focus 5).
  - **E. Mew/Deoxys** (Review Focus 4)
    1. Set the species to MEW: the editor shows `Fateful: Yes`.
    2. Set `Fateful: No`: the trade screen shows `Can't offer it: FRLG` / `needs Fateful: Yes on` / `Mew/Deoxys: edit it.`, and sitting does nothing.
    3. Set `Fateful: Yes`: the menu shows MEW. Trading it is optional.
  - **F. Refusals**
    1. TRADE → NO, then CANCEL → YES, as test 2 F.1–F.2.
    2. If FireRed has an Egg, offer it: the Flipper shows `Can't take that one: it's an Egg.`, and FireRed says the trade was canceled (or, without the National Dex, refuses it itself).
  - **G. End and options**
    1. `End trade` from the room, then from the menu: FireRed walks out by itself, and the Flipper shows `Link ended`.
    2. Gen III menu → `National Dex: Off` shows `Ruby/Sapphire only.`; choose Cancel.
  - **H. Swap cartridges** (Review Focus 3)
    1. Don't end the trade: walk out on FireRed (the Flipper shows `Go to the Cable Club`) and turn the GBA off.
    2. Put Ruby in and enter its Trade Center: the Flipper's card is Hoenn-style and its avatar is Brendan. Trade once.
    3. Walk out, swap back to FireRed and trade once more. No link errors.
  - **I. `.pk3` with FireRed**
    1. Save .pk3 the Pokemon received in D, then Load .pk3 it: the species, level and OT match.
    2. Optional: PKHeX opens it with no checksum warning.
- [ ] **Step 5: Commit** `gen3: hand offer traits to the link; Fateful row; FireRed hardware checklist`.

---

### Task 7: v3.0: version, description, README, changelog, final checks

**Files:**
- Modify: `application.fam`, `README.md`, `changelog.md`, `docs/gen3/hardware-test-frlg.md`

Exact values:
- `application.fam`: `fap_version=[3,0],` and `fap_description="Pokemon exchange from Flipper Zero to Game Boy and Game Boy Advance, supports Generation I & II and Gen III Ruby/Sapphire/FireRed/LeafGreen non-Japanese games",`. No other line changes.
- `changelog.md`: the `## Unreleased` heading becomes `## Version 3.0`, and its body is replaced by:
  ```
  **New Features**
  - Gen III (Game Boy Advance, non-Japanese): Cable Club trades with Ruby, Sapphire, FireRed and LeafGreen, one Pokemon at a time and back again without leaving the Trade Center
  - Gen III Pokemon editor: species, nickname, level, held item, moves, IVs/EVs, nature, ability, shiny, gender, Pokerus, Poke Ball, OT name/TID/SID, Unown form, fateful flag for Mew/Deoxys
  - Load and save Gen III Pokemon as PKHeX `.pk3` files
  - Gen III Link Test with a link log, and the SD pin on the Pinout screen
  - Hardware checklists in `docs/gen3/`
  ```
- **`README.md`:** replace plans 1–3's three sections (`Gen III (experimental): link test`, `… Pokemon editor and .pk3 files`, `… trading with Ruby/Sapphire`) with one section, `## Gen III (Game Boy Advance)`. Update the Contents list to match, and add one Introduction sentence: "Version 3.0 adds Generation III: Ruby, Sapphire, FireRed and LeafGreen (non-Japanese)." The section covers, in this order:
  1. **Games:** English Ruby, Sapphire, FireRed and LeafGreen. Tested on genuine Ruby and FireRed. Emerald is untested.
  2. **Cable:** which plug goes in the Flipper, the `Cable: plug in/swap ends` hint, and the SD pin (kept from plan 1's text).
  3. **Trading:**
     - Gen III → Trade, then Pokemon Center 2F, right-hand counter, Trade Center; sit down; one Pokemon per trade; the Flipper keeps what it receives.
     - Editing between trades: the GBA goes back to the room; step Up, then Down.
     - The OT default, and the `OT: use <NAME>` row.
     - The save lock, End trade, and "Stop link?".
  4. **What the Flipper won't offer or take:** Eggs, Mail, the ENIGMA BERRY, and items Ruby lacks. With FireRed/LeafGreen also: Pokemon from #152 up while that game has no National Dex ("get it in that game first, or trade one from #1–151"), and Mew/Deoxys unless `Fateful: Yes`.
  5. **National Dex option:** Ruby/Sapphire only, off by default, permanent once used.
  6. **Editor:** what it changes, the `?` image for #252–386, Max EV 85 each (510 total), and the `Fateful` row.
  7. **`.pk3` files:** `apps_data/pokemon/gen3/`, PKHeX format, saved as 80 bytes, loaded from 80 or 100.
  8. **Link Test, log and checklists:** links to `hardware-test-1.md`, `editor-test.md`, `hardware-test-2.md` and `hardware-test-frlg.md`.
- **`hardware-test-frlg.md` section J:**
  1. **Memory:** with the FireRed trade running and Edit Pokemon open, `free` in the qFlipper CLI (or `ufbt cli`) shows `Free heap size` ≥ 12,000.
  2. **Results:** a pass/fail table for A–J.
  3. **On failure:** report the step, what each screen showed, and the log copied off before the next Trade or Link Test.
  4. **Hints for whoever reads the log:**
     - FireRed closes the link at the counter: check the claimed `version` in the Flipper's first block (D1).
     - A link error right after YES: the 0x1144 relink took over 300 frames.
     - A garbled card: card byte 0x38 or `stars` (T13).

- [ ] **Step 1: Write** the values and sections above.
- [ ] **Step 2: Final checks:**
  - `make -C tests` passes.
  - Both SDK builds have 0 warnings.
  - `text + data` ≤ the Task 1 baseline + 4,000 and ≤ 128,000; put the number in the report.
  - `git diff <Task 1 base>..HEAD --stat` over the Gen I/II files in Global Constraints and `lib/flipper-gblink` prints nothing.
  - `grep -n 'fap_version' application.fam` prints `fap_version=[3,0],`.
- [ ] **Step 3: Commit** `docs: Gen III Phase 1 with Ruby/Sapphire/FireRed/LeafGreen trades; version 3.0`.

---

### Task 8: Phase 1 hardware gate (the user runs it)

- [ ] **Step 1: Hand-off.** Push, then give the user `dist/pokemon.fap` from a release build of the final commit, with these checklists in order. **Stop here until the user reports results.**
  1. `docs/gen3/hardware-test-1.md`, all steps: the Link Test with Ruby.
  2. `docs/gen3/editor-test.md` A–E: the editor and `.pk3` files.
  3. `docs/gen3/hardware-test-2.md` A–J: the Ruby trade.
  4. `docs/gen3/hardware-test-frlg.md` A–J: FireRed (Link Test, trade, editor, `.pk3`, cartridge swap).
  5. A Gen I/II smoke test: the main menu still lists Gen I, Gen II and Pinout, and each opens its editor and Trade screen (`editor-test.md` A.9).
- [ ] **Step 2: All pass.** Phase 1 is done as v3.0. Report to the controller; plan 5 comes next.
- [ ] **Step 3: Failures.**
  - Fix, rebuild, and retest only the failed sections.
  - A failure in 1, 2, 3 or 5 blocks v3.0.
  - A FireRed-only failure that has no quick fix: report it with the log. The user decides whether v3.0 waits, because the spec doesn't require FireRed/LeafGreen for the first Phase 1 release. Don't change the version or the docs on your own.

---

## Spec coverage

| Requirement (spec, brief) | Task |
|---|---|
| FireRed/LeafGreen trades after Ruby, planned and tested on the user's FireRed | 2–6, 8 |
| Detect the partner's game; claim per Part C (0x4004/0x4005, `progressFlags` 0x01) to avoid the R/S/E gate | 1, 2 (D1) |
| FRLG trainer card (96-byte format, version at 0x38) | 1, 2 |
| Read the GBA's National Dex from what FRLG sends (byte 0x20) | 1, 2 (D2) |
| No species above #151 to FRLG without the National Dex, with a reason and what to do (T14) | 1, 3, 4, 5, 6 (D2, D3) |
| Mew/Deoxys fateful bit: set for Flipper-made ones (plan 2), checked on loaded/traded ones, fixable | 1, 3, 5, 6 (D7) |
| Trade menu and link save differences (§C4–§C6): exchange delays, 5FFF types, 0x1144 deadline; Switch build ignored | 3, 4 (D4, D5) |
| Fake: FRLG mode as strict as Ruby's (decision D8); FRLG linkup, cards, room, trade, save, leave tests; Ruby tests green | 2–4 |
| National Dex opt-in is Ruby-only (FRLG ignores `lp_field_2`) | 2, 4, 6 (D6) |
| Anti-goals: no damaged save, no link error in normal use | Global Constraints, 3, 4, Review Focus 5 |
| Emerald not ruled out | Global Constraints, D1 |
| Gen I/II unchanged | Global Constraints, 7, 8 |
| Phase 1 as v3.0: `fap_version`, description, changelog, README Gen III section | 7 |
| Combined FireRed checklist; final gate listing every hardware run | 6, 7, 8 |

## Amendments made during execution

The tasks above are kept as written. Decisions taken while they ran amended them; this is a summary, and the code is the reference. Plan 3's rules (its Amendments, R5–R8 and O1–O3) carried over unchanged unless a line below says otherwise.

**Size, ISR and stack**
- **Size budget:** `text + data` ≤ 112,000 for both builds (not 128,000), and dev ≤ the Task 1 baseline + 4,000 = 101,716 at every commit. Measure the build directories (`~/.ufbt/build/pokemon_d.elf` and the dev SDK's), not `dist/`, which both builds write.
- **Final figures** (Task 7): release 88,724 and dev 98,888. That is +1,012 and +1,172 since the Task 1 baseline (87,712 / 97,716, plan 3's final figures), and leaves dev 2,828 under the gate.
- **ISR:** plan 3's bounds stand, not this plan's "220 bytes": a `gen3_club_frame` call copies at most 600 B and decrypts at most once (the GBA's AABB). The ISR-reachable additions are pure and flash-only: `gen3_link_game`, `gen3_partner_rules`, `gen3_link_player_progress` and `gen3_link_player_set_progress`, `gen3_trainer_card_build_frlg`, `gen3_partner_refuse`, and `gen3_offer_traits`, which runs on the decrypt scratch the AABB check has already filled, so it adds no decrypt.
- **App ↔ ISR:** each PRIMASK section copies at most 104 B and never decrypts or `memcmp`s. `gen3_session_set_offer_traits` copies 102 B: the offer, `has_offer` and the traits byte. `gen3_club.c` decrypts nowhere but on the AABB path.
- **Stack** (Task 7, `-fstack-usage`, plus 16 B for the TIM2 dispatch; budget 600 B): the deepest chain is 432 B in release (the AABB path down to `gen3_pkm_decrypt`, the same as plan 3) and 400 B in dev. The GBA-block path through `gen3_trainer_card_build_frlg` is 400 / 360 B, and the round start 392 / 320 B. With SysTick (72 B) and a 32-B exception frame beneath it, the ISR stack peaks at about 536 B of 1,024 in release (504 B in dev).

**Interfaces**
- Plans 2 and 3's real names replaced the ones written here, and the tests use the bench's own helpers: `bench_to_room` (the plan's `reach_room` re-inits a Ruby Link Test bench), the bench's `club_cfg` and `club_cfg_trade` for every `Gen3ClubConfig`, and `tm_make_unfateful` built on `tm_make_pkm`.
- `gen3_club_set_offer_traits(club, game, trades_seen, traits)` hands the club an offer with its traits. `gen3_club_set_offer` stays for tests and passes unknown traits, which are never usable with an FRLG partner. `gen3_session_set_offer_traits` replaces `gen3_session_set_offer`, which is gone.
- `Gen3ClubStatus` appends `partner_rules`, `offer_traits` and `offer_usable` after `relinking`. `GEN3_FRLG_PROGRESS_NATIONAL_DEX_MASK` (0x0F) is public, beside `GEN3_FRLG_PROGRESS_NATIONAL_DEX`.
- Plan 3's `Gen3Refuse` sweeps in `test_gen3_trade_sync.c` had to change for the two new values; the sweep's guide-call count grew to match.

**The claim and the partner's rules**
- The claim follows the GBA's game for every link type, the Link Test and another room included. A partner with no block yet, or after a bad block, reads as Unknown and gets plan 3's Ruby claim.
- The partner rules come from every GBA LinkPlayerBlock that parses, before the club answers it. They start as `{Unknown, true}` and outlive relinks, session resets and disconnects; a cartridge swap updates them at the new GBA's first block.
- Round delays are chosen per game at the GBA's 0x1122 block: 16 / 12 bursts for FRLG, plan 3's 10 / 6 otherwise. An explicit `gen3_club_set_round_delays` (tests only) fixes them until the next `gen3_club_init`.

**Offers the partner can't take**
- **An unusable offer counts as no offer.** The club doesn't sit with it, and round 1 sends an empty party (zeros) instead of the Pokemon the GBA last saw, as D3 had it. pokefirered shows that FRLG accepts an empty partner party, as Ruby does, so the GBA never sees the Pokemon, and the club never sends DDDD to it. A late sit works with FRLG as with Ruby: once the offer becomes usable, a seated GBA gets the trade menu.
- **The traits are computed on the app thread.** The sync computes them from the editor's Pokemon, never in the club. A traded-in Pokemon's traits come from the AABB check's decrypt and apply when the trade ends `Done` or `Assumed`.
- **The sync keeps plan 3's O2:** a valid offer still needs a clean pickup, so a traded-away Pokemon is never offered again. The partner refusal is computed only for a valid offer. The offer is still published while the partner refuses it; the club decides.
- **`offer_stale` tests `!offer_usable` in place of `!has_offer`, and adds `left`.** With `left`, the menu after a late 0x1144 relink's save never starts a second trade: the GBA gets EEBB and goes back to a room where the Flipper no longer sits.
- **The FRLG guide rows** come after the pickup rows and before plan 3's `Can't offer it` rows, with no "stay seated" variant. They show only while a GBA is linked (not Waiting, or relinking), so the previous game's rule doesn't show while the user walks another one to the counter. For under 1 s after a cartridge swap, the previous game's row can still show until the new GBA's first block arrives; this is accepted.
- **`left` with a GBA linked** shows `Walk out on the GBA, then unplug.` instead of `Link ended. You can unplug the GBA.`, since unplugging then would give the GBA a link error.
- **`gen3_edit_set_fateful`** keeps `pristine` when no byte changes (plan 3's R4) and is in the fuzz test, which also cross-checks the offer traits after every step.

**The fake's FRLG mode**
- It checks a trainer card's `stars` (at most 4) in both modes, which closed a gap in plan 3's Ruby mode; the version byte at 0x38 is checked in FRLG mode only. It writes its own `name[8..10]`.
- Its LastMon rule in FRLG mode has no HP test (`fr: trade.c: CanTradeSelectedMon`). FireRed's check of its own Pokemon's HP on YES (`PLAYER_MON_INVALID`) isn't modelled: with all the GBA's other Pokemon fainted, the fake sends BBBB where FireRed sends BBCC. The club can't influence this; it is accepted.
- The 0x1144 deadline counts every frame in `fr: trade_scene.c: CB2_LinkTrade` state 4, the stricter reading. A boundary test shows it doesn't trip early: the relink lands up to about 361 frames after FireRed's OpenLink and trips at 363. On the bench the club relinks in about 24 frames.
- The FRLG trade, link save and room return needed no change to `gen3_club.c`.
- The trade-run helpers shared by the Ruby and FRLG suites live in `tests/club_bench.h`.

**Docs**
- `docs/gen3/link-protocol-trade.md` §C4: FRLG waits 11 frames before each `CCCC` on master and child alike, and its round-1 `CCCC` needs G0 + 4, one frame more than Ruby.
- Plan 3's sync-2 comment is corrected in its code, test and Amendments: an `Assumed` misreport clones the GBA's Pokemon and loses only the Flipper's offer, which can be made again.
- Comments state their reasons, cite pokefirered as `fr: file: Function`, and cite this plan's decisions as "plan 4 D3" or in words.
- The README merges the three Gen III sections into `## Gen III (Game Boy Advance)`. It says no game is tested on hardware until the gate passes. The v3.0 changelog keeps the bulk-asset bullet.
- `docs/gen3/hardware-test-frlg.md` has sections A–J (J is memory), then the results table, what to report on failure, and hints for reading the log. `docs/gen3/hardware-test-1.md` points to it for FireRed.

**Hardware gate (Task 8): pending with the user.** By the user's decision ("finish everything first"), the gate waits until plans 2–5 and the follow-ups are built. Task 7 prepared the checklists, the README and changelog, and the final size and stack figures. The gate runs, in order: `docs/gen3/hardware-test-1.md` (all steps), `docs/gen3/editor-test.md` A–F (F is memory), `docs/gen3/hardware-test-2.md` A–K (J is the OT row, K memory), `docs/gen3/hardware-test-frlg.md` A–J, and the Gen I/II smoke test, `editor-test.md` A.13. On a full pass, the README's Games line becomes "Tested on genuine Ruby and FireRed. Emerald is untested.", with LeafGreen added if it was tested.
