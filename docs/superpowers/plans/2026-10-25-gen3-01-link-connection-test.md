# Gen III Link Connection Test Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Pokemon Ruby's Cable Club accepts the Flipper as the other player. Both
players walk into the Trade Center and idle there with no link error, and the GBA
player can leave cleanly. This is the spec's first hardware milestone (design
call #1).

**Architecture:** The Flipper is the GBA multiplayer **parent**. A TIM2-driven
driver bit-bangs 115200-baud 16-bit frames on SD, one per tick, in 9-transfer
bursts at the GBA frame rate. After each frame it hands the received words to a
pure-C protocol stack, which mirrors pokeruby's `link.c` from the master side:
handshake, checksummed bursts, a command queue, block transfer, link-player and
trainer-card blocks, and a Cable Club state machine. The stack is unit-tested on
the host against a scripted fake Ruby child. A new Gen III → Link Test screen
shows progress and writes a frame log to the SD card, because only the user has
the hardware.

**Tech Stack:** C11, Flipper SDK via `ufbt` (release SDK 1.5.1, API 88.2 at time
of writing), STM32 LL (TIM2), `make` + host `cc` for unit tests, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-10-25-gen3-gba-design.md`. Protocol
reference (read alongside; sections cited as §A3 etc.):
`docs/gen3/link-protocol.md`.

## Plan sequence

This is plan 1 of 5. Plans 2+ are written after this plan's hardware gate
(Task 12), because its result can reorder them (spec call #1).

1. **This plan:** link connection test.
2. Gen III Pokemon data core and editor: the 80/100-byte format, crypto,
   checksums, species/move/item/ability/nature tables, stat calculation, editor
   scenes, and `.pk3` import/export on the SD card.
3. Cable Club trade with Ruby (protocol doc Part B), building on 1 and 2.
4. FireRed/LeafGreen trades (protocol doc Part C).
5. Direct save access: multiboot helper, cartridge save read/write with
   backup and verify, ROM dump.

## Global Constraints

- C on the Flipper SDK, built with `ufbt`. CI builds against both the dev and release SDK channels, and both must keep passing.
- MIT licensed. PKHeX (GPLv3) and the pret decompilations are references for behaviour, constants and layouts only; copy no code from them.
- The shared core must be testable on a normal computer (macOS clang; Linux gcc in CI) without a Flipper.
- Gen I/II trading keeps working exactly as before for existing users.
- Western (non-Japanese) games only.
- The SD pin choice is stored in this app's own config. Never modify `lib/flipper-gblink`.
- Commits are authored by the user (LawlessNPC, Aleksandr Kofman) with **no** AI-attribution trailers: no `Co-Authored-By`, no `Claude-Session`, no "Generated with". Check with `git log -1 --format='%(trailers)'`, which must be empty.
- Pull requests go to the user's fork, `LawlessNPC/Flipper-Zero-Game-Boy-Pokemon-Trading`. No `fap_version` bump in this plan; v3.0 is the finished Phase 1.
- Never drop the GBA to a link-error screen during normal use. Never damage the player's save.
- **Pure-module rule:** every file in `src/gen3/` whose name does not end in `_furi.c`/`_furi.h` includes only `<stdint.h> <stdbool.h> <stddef.h> <string.h> <stdio.h>` and other pure `src/gen3/` headers. Flipper APIs live only in `*_furi.*` files and in `src/scenes/`, `src/views/`.
- **ISR rule:** everything called from the link frame callback runs in interrupt context. That means no `malloc`/`free`, no `FURI_LOG_*`, and no blocking furi calls. It must return in under 200 µs.
- Includes use the repo's root-relative style, e.g. `#include <src/gen3/gen3_link.h>`. Follow `.clang-format`, snake_case functions and CamelCase typedefs, as in the existing code.
- Setup in a fresh checkout: `git submodule update --init`, then `ufbt update --channel=release` once.

## Review Focus

1. **GBA not ready:** the GBA is off or not yet in the Cable Club when the Link Test starts. The slot reads 0xFFFF indefinitely; the Flipper must stay in handshake quietly and connect whenever the GBA arrives. *(Task 2: `test_handshake_idle`; Task 5: `test_club_gba_not_ready`.)*
2. **GBA resets or links twice:** the GBA power-cycles mid-session, or the player leaves and re-enters the Cable Club. The Flipper must return to waiting and link again without restarting the app. *(Task 2: `test_reconnect_after_disconnect`; Task 6: `test_club_second_linkup`, `test_club_gba_reset`.)*
3. **One corrupted word** (bad contact or noise): the Flipper counts it, shows it and logs it, and never crashes or keeps streaming a half-sent block. *(Task 2: `test_checksum_mismatch`; Task 6: `test_club_checksum_error`.)*
4. **The GBA player picked another counter** (Colosseum, Record Corner): the Flipper says so and answers 0x5FFF, so the GBA can back out cleanly. *(Task 5: `test_club_other_room`.)*
5. **Cable plugged in backwards:** the Flipper must not start transfers (two parents would fight over SD) and must tell the user to swap ends. *(Task 8 code step; Task 11 checklist step 4.)*

---

## File map

| File | Responsibility |
|---|---|
| `application.fam` | exclude `tests/` from the FAP build |
| `tests/Makefile`, `tests/test.h` | host test runner and assertion macros |
| `tests/fake_ruby.{h,c}` | scripted model of Ruby's child-side link (test only) |
| `src/gen3/gen3_text.{h,c}` | Gen III Western text ↔ ASCII |
| `src/gen3/mlink_types.h` | driver ↔ protocol frame structs |
| `src/gen3/gen3_link.{h,c}` | link layer: handshake, bursts, checksum, send queue |
| `src/gen3/gen3_block.{h,c}` | 0xBBBB/0x8888 block send and receive |
| `src/gen3/gen3_link_player.{h,c}` | `LinkPlayerBlock` and trainer-card byte layouts |
| `src/gen3/gen3_club.{h,c}` | Cable Club state machine (linkup → room → leave) |
| `src/gen3/gen3_link_log.{h,c}` | ISR-safe frame log ring buffer and formatter |
| `src/gen3/mlink_hw_furi.{h,c}` | TIM2 + GPIO multiplayer-parent driver |
| `src/gen3/gen3_pins_furi.{h,c}` | load/save the SD pin in the app's config |
| `src/gen3/gen3_session_furi.{h,c}` | wires driver + club + log; log file thread |
| `src/views/gen3_link_test.{h,c}` | Link Test view |
| `src/scenes/pokemon_gen3_menu.c`, `pokemon_gen3_link_test.c`, `pokemon_gen3_link_exit.c` | scenes |
| `src/scenes/include/pokemon_scene_config.h`, `src/scenes/pokemon_menu.c`, `src/scenes/pokemon_pins.c`, `src/include/pokemon_app.h`, `src/pokemon_app.c` | wiring |
| `docs/gen3/hardware-test-1.md`, `README.md`, `changelog.md` | user-facing docs |
| `.github/workflows/build.yml` | host-test CI job |

---

### Task 1: Host test harness and Gen III text

**Files:**
- Modify: `application.fam` (add `sources=["*.c*", "!tests"],` on the line after `fap_category="GPIO",`)
- Modify: `.gitignore` (add `tests/build/`)
- Modify: `.github/workflows/build.yml` (add job `host-tests`)
- Create: `tests/Makefile`, `tests/test.h`, `tests/test_gen3_text.c`
- Create: `src/gen3/gen3_text.h`, `src/gen3/gen3_text.c`

**Interfaces:**
- Produces:
  ```c
  #define GEN3_EOS 0xFF
  /* Encodes up to max_chars chars of ascii, then fills the rest of field_len with GEN3_EOS.
   * Unsupported chars encode as '?' (0xAC). Returns chars encoded. Requires max_chars <= field_len. */
  size_t gen3_text_encode(const char* ascii, uint8_t* out, size_t field_len, size_t max_chars);
  /* Decodes until GEN3_EOS or in_len. Unknown bytes become '?'. Always NUL-terminates. Returns strlen(out). */
  size_t gen3_text_decode(const uint8_t* in, size_t in_len, char* out, size_t out_size);
  ```
- Produces (test harness): `tests/test.h` with `TEST(name)` (declares `static void name(void)`), `RUN(name)`, `ASSERT_TRUE(x)`, `ASSERT_EQ(a, b)` (prints both as hex), `ASSERT_MEM_EQ(a, b, n)`, `ASSERT_STR_EQ(a, b)`, `TEST_EXIT()` (returns 1 if any failure). One test binary per `tests/test_*.c`; each has its own `main`.
- Produces (build): `make -C tests` builds every `tests/test_*.c` with `-std=c11 -Wall -Wextra -Werror -g -fsanitize=address,undefined -I..`, links it with `$(filter-out %_furi.c,$(wildcard ../src/gen3/*.c))` and `$(wildcard fake_*.c)`, runs each, and exits non-zero on any failure. Output goes to `tests/build/`.

Character table (verified against pokeruby `charmap.txt`): space 0x00; `0`–`9` 0xA1–0xAA; `!` 0xAB; `?` 0xAC; `.` 0xAD; `-` 0xAE; `…` 0xB0; `“` 0xB1; `”` 0xB2; `‘` 0xB3; `'` 0xB4; `♂` 0xB5; `♀` 0xB6; `,` 0xB8; `/` 0xBA; `A`–`Z` 0xBB–0xD4; `a`–`z` 0xD5–0xEE; `:` 0xF0; EOS 0xFF.
- Encode extras: ASCII `"` → 0xB2.
- Decode extras: 0xB1/0xB2 → `"`; 0xB3/0xB4 → `'`; 0xB0 → `.`; anything else unmapped (including 0xB5/0xB6) → `?`.

- [ ] **Step 1: Write the harness and the failing tests** in `tests/test_gen3_text.c`:

```c
TEST(test_encode_name) {
    uint8_t b[11];
    ASSERT_EQ(gen3_text_encode("FLIPPER", b, 11, 7), 7);
    const uint8_t e[11] = {0xC0,0xC6,0xC3,0xCA,0xCA,0xBF,0xCC,0xFF,0xFF,0xFF,0xFF};
    ASSERT_MEM_EQ(b, e, 11);
}
TEST(test_encode_truncates) {        /* "FLIPPERZERO" into an 8-byte, 7-char field */
    /* returns 7; bytes C0 C6 C3 CA CA BF CC FF */
}
TEST(test_encode_charset) {          /* "az09 !?.-,/:'" */
    /* D5 EE A1 AA 00 AB AC AD AE B8 BA F0 B4, then FF */
}
TEST(test_encode_unsupported) {      /* "#" -> AC FF */ }
TEST(test_decode_stops_at_eos) {     /* {BB,D5,A1,00,AB,FF,C0} -> "Aa0 !", returns 5 */ }
TEST(test_decode_unknown) {          /* {0x01,0xFF} -> "?" */ }
TEST(test_decode_no_eos_truncates) { /* {BB,BC,BD}, out_size 3 -> "AB" */ }
```

- [ ] **Step 2: Run** `make -C tests`. Expected: compile failure, `gen3_text.h` not found.
- [ ] **Step 3: Implement** `src/gen3/gen3_text.{h,c}` per the interface and table above.
- [ ] **Step 4: Run** `make -C tests`. Expected: `test_gen3_text: 7 passed`, exit 0.
- [ ] **Step 5: Verify the FAP still builds without the tests.** Run `ufbt` from the repo root. Expected: ends with `APPCHK ... pokemon.fap`; no `tests/` path in the `CC` lines.
- [ ] **Step 6: Add the CI job** to `.github/workflows/build.yml`:

```yaml
  host-tests:
    runs-on: ubuntu-latest
    name: 'Host unit tests'
    steps:
      - uses: actions/checkout@v4
      - run: make -C tests
```

- [ ] **Step 7: Commit** (and do the same for every later commit step):

```bash
git add application.fam .gitignore .github/workflows/build.yml tests src/gen3
git commit -m "gen3: add host test harness and Gen III text encoding"
git log -1 --format='%(trailers)'   # must print nothing
```

---

### Task 2: Link layer (handshake, bursts, checksum, queue)

**Files:**
- Create: `src/gen3/mlink_types.h`, `src/gen3/gen3_link.h`, `src/gen3/gen3_link.c`
- Test: `tests/test_gen3_link.c`

**Interfaces:**
- Produces (`mlink_types.h`). This is the driver contract: the `MlinkRx` passed to the protocol is the result of the `MlinkTx` the protocol produced in the previous frame.
  ```c
  #define MLINK_BURST_LEN 9
  #define MLINK_NO_DATA 0xFFFFu
  typedef struct { uint8_t count; uint16_t word[MLINK_BURST_LEN]; } MlinkTx; /* count: 1 (handshake) or 9 (burst) */
  typedef struct {
      uint8_t count;                  /* == count of the MlinkTx that produced it */
      uint16_t word[MLINK_BURST_LEN]; /* GBA's word per transfer, MLINK_NO_DATA if no answer */
      uint16_t timeout_mask;          /* bit t: no start bit seen in transfer t */
      uint16_t framing_mask;          /* bit t: stop bit low in transfer t */
  } MlinkRx;
  ```
- Produces (`gen3_link.h`):
  ```c
  #define GEN3_CMD_LEN 8
  #define GEN3_MASTER_HANDSHAKE 0x8FFF
  #define GEN3_SLAVE_HANDSHAKE 0xB9A0
  #define GEN3_LINK_QUEUE_LEN 24
  typedef enum {
      Gen3LinkEventNone, Gen3LinkEventConnected, Gen3LinkEventCommand,
      Gen3LinkEventChecksumError, Gen3LinkEventDisconnected,
  } Gen3LinkEvent;
  typedef struct { uint32_t bursts, checksum_errors, timeouts, framing_errors; } Gen3LinkStats;
  typedef struct Gen3Link Gen3Link; /* full struct defined in the header so callers can embed it; no heap */
  void gen3_link_init(Gen3Link* link);
  bool gen3_link_connected(const Gen3Link* link);
  /* Process the previous frame's result. On Gen3LinkEventCommand, cmd_out holds the GBA's 8 words. */
  Gen3LinkEvent gen3_link_rx(Gen3Link* link, const MlinkRx* rx, uint16_t cmd_out[GEN3_CMD_LEN]);
  /* Build this frame's transfers. Call exactly once per frame, after gen3_link_rx (or right after init). */
  void gen3_link_tx(Gen3Link* link, MlinkTx* tx);
  bool gen3_link_send(Gen3Link* link, const uint16_t cmd[GEN3_CMD_LEN]); /* false if queue full */
  size_t gen3_link_queued(const Gen3Link* link);
  void gen3_link_flush(Gen3Link* link); /* drop every queued command */
  void gen3_link_stats(const Gen3Link* link, Gen3LinkStats* out);
  ```

Behaviour (§A2, §A3):
- **Handshake mode:** `tx.count = 1`. The word is `0x8FFF` if the last evaluated player count was ≥ 2, else `0xB9A0`. Evaluate pokeruby's `DoHandshake` on slots `{word F sent, rx.word[0], 0xFFFF, 0xFFFF}`. On success, return `Connected`, set "checksum not yet valid", and make the next `tx` a burst.
- **Bursts:** `tx.word[0]` is the checksum of the previous burst: the u16 sum over t = 1..8 of both players' words. `tx.word[1..8]` is the queue head, or zeros. In `rx`, the event priority is:
  1. All nine words are 0xFFFF → `Disconnected`: clear the queue, back to handshake.
  2. `rx.word[1..8]` all 0xB9A0 → `Disconnected` (the GBA is still handshaking, U7).
  3. `rx.word[0]` ≠ the stored checksum, while the checksum is valid → `ChecksumError`.
  4. Any of `rx.word[1..8]` non-zero → `Command`.
  5. Otherwise `None`.
- **Stats:** `timeouts += popcount(timeout_mask)`, likewise for framing, and `bursts++`.

- [ ] **Step 1: Write the failing tests.** A test-local helper `connect(link)` runs the success script below. Tests:

```c
TEST(test_handshake_idle)        /* 600 frames: tx.count==1, tx.word[0]==0xB9A0; rx 0xFFFF -> None; !connected */
TEST(test_handshake_success)     /* G answers 0x0000, 0xB9A0, 0xB9A0: F sends B9A0, B9A0, 8FFF; 3rd rx -> Connected;
                                    next tx.count==9 */
TEST(test_first_burst_unchecked) /* first burst rx.word[0]==0xB9A0 (stale), rest 0 -> None, checksum_errors==0 */
TEST(test_checksum_value)        /* send {0x2222,0x1133,0,...}; burst; G all zero -> next tx.word[0]==0x3355 */
TEST(test_checksum_match)        /* G's next word[0]==0x3355 -> None */
TEST(test_checksum_mismatch)     /* G's next word[0]==0x1234 -> ChecksumError; stats.checksum_errors==1 */
TEST(test_command_received)      /* G words[1..8]={0x2FFE,0,...} -> Command, cmd_out[0]==0x2FFE */
TEST(test_queue_one_per_burst)   /* 3 sends -> 3 bursts carry them in order; 4th burst words[1..8] all 0 */
TEST(test_queue_full)            /* 24 sends true, 25th false; gen3_link_flush -> queued==0 */
TEST(test_disconnect)            /* connected; rx 9x0xFFFF -> Disconnected; next tx.count==1, word 0xB9A0; queued==0 */
TEST(test_reconnect_after_disconnect) /* after test_disconnect state, success script -> Connected again */
TEST(test_slave_still_handshaking)    /* connected; rx words[1..8] all 0xB9A0 -> Disconnected */
TEST(test_timeouts_counted)      /* rx.timeout_mask=0x0005 -> stats.timeouts==2 */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL (missing `gen3_link.h`).
- [ ] **Step 3: Implement** `mlink_types.h` and `gen3_link.{h,c}` per the interfaces and behaviour above. The queue is a fixed ring of `GEN3_LINK_QUEUE_LEN` × 8 words.
- [ ] **Step 4: Run** `make -C tests`. Expected: all `test_gen3_link` tests pass.
- [ ] **Step 5: Commit** `gen3: add multiplayer link layer (handshake, bursts, checksum)`.

---

### Task 3: Block transfer

**Files:**
- Create: `src/gen3/gen3_block.h`, `src/gen3/gen3_block.c`
- Test: `tests/test_gen3_block.c`

**Interfaces:**
- Consumes: `gen3_link_send`, `gen3_link_queued`, `GEN3_CMD_LEN`, `GEN3_LINK_QUEUE_LEN` (Task 2).
- Produces:
  ```c
  #define GEN3_CMD_INIT_BLOCK 0xBBBB
  #define GEN3_CMD_CONT_BLOCK 0x8888
  #define GEN3_BLOCK_MAX 256
  #define GEN3_BLOCK_CHUNK_BYTES 14
  /* Queues the header {BBBB, size, 0x80+sender_id, 0...} plus ceil(size/14) chunks {8888, w0..w6}, little-endian.
   * All or nothing: returns false and queues nothing if size==0, size>GEN3_BLOCK_MAX, or the queue lacks room. */
  bool gen3_block_send(Gen3Link* link, const uint8_t* data, uint16_t size, uint8_t sender_id);
  typedef struct { uint8_t data[GEN3_BLOCK_MAX]; uint16_t size; uint16_t pos; bool active; } Gen3BlockRx;
  typedef enum { Gen3BlockRxIgnored, Gen3BlockRxStarted, Gen3BlockRxProgress, Gen3BlockRxDone, Gen3BlockRxBad } Gen3BlockRxResult;
  void gen3_block_rx_reset(Gen3BlockRx* rx);
  /* Feed every received command. Non-block commands -> Ignored. Done when pos >= size (§A4). */
  Gen3BlockRxResult gen3_block_rx_feed(Gen3BlockRx* rx, const uint16_t cmd[GEN3_CMD_LEN]);
  ```

- [ ] **Step 1: Write the failing tests:**

```c
TEST(test_block_header)       /* 60 bytes -> queued==6; first burst words[1..8]=={0xBBBB,0x003C,0x0080,0,0,0,0,0} */
TEST(test_block_chunk_le)     /* data[i]=i: second burst {0x8888,0x0100,0x0302,0x0504,0x0706,0x0908,0x0B0A,0x0D0C} */
TEST(test_block_100_chunks)   /* 100 bytes -> queued==9 */
TEST(test_block_roundtrip)    /* 60 pseudo-random bytes: feed each burst's words[1..8] into rx -> last result Done,
                                 rx.size==60, data identical */
TEST(test_block_too_big)      /* 257 bytes -> false, queued==0 */
TEST(test_block_no_room)      /* queue holding 20 cmds; 100-byte block (9 cmds) -> false, queued still 20 */
TEST(test_rx_chunk_without_header) /* {0x8888,...} first -> Ignored */
TEST(test_rx_oversize_header)      /* {0xBBBB,0x0200,0x81,...} -> Bad */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement** `gen3_block.{h,c}`.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add link block transfer`.

---

### Task 4: LinkPlayer and trainer-card blocks

**Files:**
- Create: `src/gen3/gen3_link_player.h`, `src/gen3/gen3_link_player.c`
- Test: `tests/test_gen3_link_player.c`

**Interfaces:**
- Produces:
  ```c
  #define GEN3_LINK_PLAYER_BLOCK_SIZE 60
  #define GEN3_TRAINER_CARD_BLOCK_SIZE 100
  #define GEN3_LINKTYPE_TRADE_SETUP 0x1133
  #define GEN3_VERSION_RUBY 0x4002
  #define GEN3_LANGUAGE_ENGLISH 2
  typedef struct {
      uint16_t version; uint16_t lp_field_2; uint32_t trainer_id; /* low 16 = TID, high 16 = SID */
      uint8_t name[11]; uint8_t gender; uint32_t link_type; uint16_t id; uint16_t language;
  } Gen3LinkPlayer;
  void gen3_link_player_block_build(const Gen3LinkPlayer* lp, uint8_t out[GEN3_LINK_PLAYER_BLOCK_SIZE]);
  /* false if either magic's first 15 bytes differ from "GameFreak inc.\0" (byte 15 is not compared) */
  bool gen3_link_player_block_parse(const uint8_t in[GEN3_LINK_PLAYER_BLOCK_SIZE], Gen3LinkPlayer* lp);
  /* 100-byte trainer card for block request type 2 (§A5 table), from lp's gender, TID and name[0..7] */
  void gen3_trainer_card_build(const Gen3LinkPlayer* lp, uint8_t out[GEN3_TRAINER_CARD_BLOCK_SIZE]);
  ```
- Layout: little-endian, offsets from §A5. The magic is ASCII `47 61 6D 65 46 72 65 61 6B 20 69 6E 63 2E 00 00` at 0x00 and 0x2C. Fields: `version` 0x10, `lp_field_2` 0x12, `trainer_id` 0x14, `name` 0x18 (11 bytes), `gender` 0x23, `link_type` 0x24 (u32), `id` 0x28, `language` 0x2A.
- Trainer card: `gender` 0x00, `stars` 0x01 = 0, `hasPokedex` 0x02 = 1, `trainerId` 0x0E (u16 TID), `var_28[4]` 0x28 = 0xFFFF ×4, `playerName` 0x30 (8 bytes), every other byte 0.

- [ ] **Step 1: Write the failing tests:**

```c
TEST(test_lp_build_layout) /* lp {0x4002,0,0x12345678,name "FLIPPER"+EOS fill,gender 1,0x1133,0,2}:
                              out[0..15]==magic; out[0x10..0x11]=={0x02,0x40}; out[0x14..0x17]=={0x78,0x56,0x34,0x12};
                              out[0x18..0x22]==name; out[0x23]==1; out[0x24..0x27]=={0x33,0x11,0,0};
                              out[0x2A..0x2B]=={0x02,0}; out[0x2C..0x3B]==magic */
TEST(test_lp_roundtrip)        /* build -> parse -> every field equal */
TEST(test_lp_bad_magic1)       /* out[3]^=1 -> parse false */
TEST(test_lp_bad_magic2)       /* out[0x2C+3]^=1 -> parse false */
TEST(test_lp_magic_byte15_ignored) /* out[15]=0x55, out[0x3B]=0x55 -> parse true */
TEST(test_card_layout)         /* out[0]==gender, out[1]==0, out[2]==1, out[0x0E..0x0F]==TID LE,
                                  out[0x28..0x2F]==0xFF x8, out[0x30..0x37]==name[0..7], out[0x24..0x27]==0,
                                  out[0x38..0x63] all 0 */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement** `gen3_link_player.{h,c}`, writing bytes explicitly (no struct packing tricks).
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add LinkPlayer and trainer card block layouts`.

---

### Task 5: Fake Ruby child and Cable Club linkup (to trainer cards)

**Files:**
- Create: `src/gen3/gen3_club.h`, `src/gen3/gen3_club.c`
- Create: `tests/fake_ruby.h`, `tests/fake_ruby.c` (test infrastructure; built only by `tests/Makefile`)
- Test: `tests/test_gen3_club.c`

**Interfaces:**
- Consumes: Tasks 1–4.
- Produces (`gen3_club.h`):
  ```c
  typedef enum {
      Gen3ClubPhaseWaiting, Gen3ClubPhaseLinked, Gen3ClubPhasePlayers, Gen3ClubPhaseCards,
      Gen3ClubPhaseRoom, Gen3ClubPhaseClosing, Gen3ClubPhaseError,
  } Gen3ClubPhase;
  typedef enum { Gen3ClubErrorNone, Gen3ClubErrorChecksum, Gen3ClubErrorBadPlayerBlock, Gen3ClubErrorOtherRoom } Gen3ClubError;
  typedef struct { const char* name; uint16_t tid; uint16_t sid; uint8_t gender; } Gen3ClubConfig;
  typedef struct {
      Gen3ClubPhase phase; Gen3ClubError last_error; /* last_error survives the reset to Waiting */
      bool partner_valid; Gen3LinkPlayer partner;
      uint32_t frames; uint32_t room_frames; Gen3LinkStats link;
  } Gen3ClubStatus;
  typedef struct Gen3Club Gen3Club; /* full struct in header: Gen3Link, Gen3BlockRx, own blocks, counters */
  void gen3_club_init(Gen3Club* club, const Gen3ClubConfig* cfg);
  /* One call per frame from the driver callback. rx is NULL on the first call. */
  void gen3_club_frame(Gen3Club* club, const MlinkRx* rx, MlinkTx* tx);
  void gen3_club_status(const Gen3Club* club, Gen3ClubStatus* out);
  #define GEN3_CLUB_CARD_DELAY_BURSTS 10 /* idle bursts between both player blocks and 0xCCCC (§A6, U3 margin >= 5) */
  ```
- Produces (`tests/fake_ruby.h`, test only):
  ```c
  typedef enum { FakeRubyOff, FakeRubyHandshake, FakeRubyLinkup, FakeRubyWalkIn, FakeRubyRoom, FakeRubyLeaving, FakeRubyClosed } FakeRubyPhase;
  typedef struct FakeRuby FakeRuby; /* full struct in the header so tests allocate it on the stack */
  void fake_ruby_init(FakeRuby* g, const Gen3LinkPlayer* me); /* me.link_type picks the counter */
  void fake_ruby_open_link(FakeRuby* g);   /* player chose the counter: link opens after 6 frames (§A3 timeline) */
  void fake_ruby_power_off(FakeRuby* g);   /* answers 0xFFFF from now on */
  void fake_ruby_leave_room(FakeRuby* g);  /* walks out: sends CAFE 0017, then 5FFF, then closes */
  void fake_ruby_corrupt_next(FakeRuby* g);/* flips one bit of its next command word */
  void fake_ruby_frame(FakeRuby* g, const MlinkTx* tx, MlinkRx* rx); /* simulates one frame of transfers */
  FakeRubyPhase fake_ruby_phase(const FakeRuby* g);
  const char* fake_ruby_error(const FakeRuby* g); /* NULL, or "checksum", "magic", "stale card", "lag", "standby timeout" */
  const Gen3LinkPlayer* fake_ruby_partner(const FakeRuby* g); /* the Flipper's block as Ruby parsed it */
  ```

Fake Ruby behaviour. Each point mirrors the cited protocol section, and it must be faithful, since it stands in for hardware:
1. **Handshake (§A3):** first transfer after the link opens answers 0x0000, then 0xB9A0. Runs the `DoHandshake` count and success rule on both slots. On success, its next transfer is a checksum slot.
2. **Bursts (§A2):** keeps its checksum exactly like `DoRecv`. Doesn't check the first burst. Any mismatch sets error "checksum" and stops answering (models `CB2_LinkError`). More than 10 frames in a row with no transfers while connected sets "lag".
3. **Queues:** a burst becomes a received command only if some word in either slot is non-zero. It acts on a received command one burst later.
4. **On 0x2222 (§A4):** sends its own `LinkPlayerBlock`: header, two idle bursts, then one chunk per burst. The first block received from the Flipper is parsed as a `LinkPlayerBlock`; a bad magic sets "magic". If the Flipper's `link_type` ≠ its own, it sends `5FFF`, waits for the Flipper's `5FFF`, and then closes (answers 0xFFFF).
5. **Trainer card (§A6):** its card is ready 3 bursts after both player blocks are complete. A `CCCC 0002` before then sets "stale card". Otherwise it sends its 100-byte card block the same way as in 4.
6. **Walk-in (§A7):** once both cards are in, 30 frames of zero commands, then one `2FFE`. If the Flipper's `2FFE` hasn't arrived within 600 frames, it sets "standby timeout". Then it enters Room and sends `CAFE 0011` every burst.

Club behaviour (§A6 "What the Flipper sends", §A8 steps 1–6):
- `Connected` → Linked. Queue `{2222,1133}`, then its own `LinkPlayerBlock` from `cfg`: `version 0x4002`, `lp_field_2 0`, `trainer_id = sid<<16 | tid`, name encoded (7 chars, EOS-filled to 11), `link_type 0x1133`, `language 2`, `id 0`.
- The GBA's player block is complete:
  - bad magic → Error / `BadPlayerBlock`;
  - `link_type` ≠ 0x1133 → Error / `OtherRoom`;
  - else store it as the partner → Players.
- After `GEN3_CLUB_CARD_DELAY_BURSTS` idle bursts: queue `{CCCC,0002}` and its own trainer card. When the GBA's card block is complete → Cards.
- **In any phase:** a received `5FFF` is answered with exactly one `{5FFF}` → Closing. `ChecksumError` → `gen3_link_flush`, then Error / `Checksum`. `Disconnected` → reset to Waiting, keeping `last_error`.

- [ ] **Step 1: Write `fake_ruby.{h,c}` and the failing tests.** The test-local helper `run(club, g, frames)` loops `gen3_club_frame` ↔ `fake_ruby_frame`. Club config is `{"FLIPPER", 22222, 0, 0}`; the fake is `{name "RUBY", tid 12345, sid 54321, gender 0, link_type 0x1133}`.

```c
TEST(test_club_reaches_cards)  /* open_link; run 400 -> fake error NULL; club phase >= Cards;
                                  partner name decodes "RUBY", TID 12345, SID 54321;
                                  fake_ruby_partner(): name decodes "FLIPPER", trainer_id==22222 */
TEST(test_club_gba_not_ready)  /* 600 frames before open_link -> then reaches Cards within 400; fake error NULL */
TEST(test_club_other_room)     /* fake link_type 0x2233 -> club phase Error, last_error OtherRoom; club sent one 5FFF;
                                  fake reaches Closed; club back to Waiting; fake error NULL */
TEST(test_club_card_delay_needed) /* gen3_club_set_card_delay(club, 0) -> fake error "stale card";
                                     the default delay never trips it (test_club_reaches_cards) */
```

For the last test, expose `void gen3_club_set_card_delay(Gen3Club* club, uint8_t bursts)`. It's for tests; production uses `GEN3_CLUB_CARD_DELAY_BURSTS`.

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement** `gen3_club.{h,c}` through the Cards phase.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add Cable Club linkup state machine and fake Ruby child`.

---

### Task 6: Trade Center room: walk-in, idle, leaving, recovery

**Files:**
- Modify: `src/gen3/gen3_club.c`, `tests/fake_ruby.c`
- Test: `tests/test_gen3_club.c`

**Interfaces:**
- Consumes: Task 5.
- Produces: no new API. Behaviour (§A7):
  - Cards: zero bursts; on the GBA's `2FFE` queue exactly one `{2FFE}` → Room.
  - Room: idle sends all-zero commands (U6), not `CAFE 0011`. A `2FFE` is answered with one `{2FFE}`; `CAFE 0017` with one `{CAFE,0017}`; `5FFF` with one `{5FFF}` → Closing. `room_frames++` per frame in Room.
  - Closing: keep bursting until `Disconnected` → Waiting.

- [ ] **Step 1: Extend the fake** with `leave_room` (§B7: `CAFE 0017`, wait for the Flipper's `CAFE 0017`, then `5FFF`, wait for the Flipper's `5FFF`, then 0xFFFF), `power_off`, and `corrupt_next`. **Write the failing tests:**

```c
TEST(test_club_room_idle)        /* reach Room, then 3600 frames: fake phase Room, error NULL; club phase Room;
                                    room_frames >= 3600; link.checksum_errors==0 */
TEST(test_club_answers_2ffe_once)/* count of Flipper commands with word0==0x2FFE during walk-in == 1 */
TEST(test_club_leave_room)       /* in Room, fake_ruby_leave_room -> club sends one {CAFE,0017} and one {5FFF};
                                    fake Closed, error NULL; club Waiting, last_error None */
TEST(test_club_second_linkup)    /* after leave_room: fake_ruby_init + open_link again -> club Room again, same club */
TEST(test_club_gba_reset)        /* in Room: power_off 120 frames (club -> Waiting), init + open_link -> Room again */
TEST(test_club_checksum_error)   /* in Room: corrupt_next -> club phase Error, last_error Checksum (fake also errors);
                                    link queue empty; fake power_off -> club Waiting; no crash */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement** the Room/Closing behaviour in `gen3_club.c`.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS, including Task 5's tests.
- [ ] **Step 5: Commit** `gen3: handle Trade Center walk-in, idle, leaving and link loss`.

---

### Task 7: Frame log ring buffer

**Files:**
- Create: `src/gen3/gen3_link_log.h`, `src/gen3/gen3_link_log.c`
- Test: `tests/test_gen3_link_log.c`

**Interfaces:**
- Consumes: `MlinkTx`, `MlinkRx` (Task 2).
- Produces:
  ```c
  #define GEN3_LOG_RING_LEN 64
  typedef struct {
      uint32_t frame, idle_before; uint8_t phase, count;
      uint16_t timeout_mask, framing_mask; uint16_t tx[MLINK_BURST_LEN], rx[MLINK_BURST_LEN];
  } Gen3LogRec;
  typedef struct Gen3LogRing Gen3LogRing; /* full struct in header; single producer (ISR), single consumer (thread) */
  void gen3_log_init(Gen3LogRing* ring);
  /* Producer. Idle frames are counted, not stored. A frame is idle when the phase is unchanged AND either
   * (count==1 and rx.word[0]==0xFFFF) or (count==9, tx.word[1..8] and rx.word[1..8] all 0, masks 0).
   * A stored record carries the idle frames before it. Full ring -> dropped++. */
  void gen3_log_frame(Gen3LogRing* ring, uint32_t frame, uint8_t phase, const MlinkTx* tx, const MlinkRx* rx);
  bool gen3_log_pop(Gen3LogRing* ring, Gen3LogRec* out);
  uint32_t gen3_log_dropped(const Gen3LogRing* ring);
  size_t gen3_log_format(const Gen3LogRec* rec, char* buf, size_t len);
  typedef bool (*Gen3LogWriter)(void* ctx, const char* line, size_t len);
  /* Pops everything; formats and writes each record. After the first writer failure, keeps discarding
   * without calling the writer, and gen3_log_failed() returns true. */
  void gen3_log_drain(Gen3LogRing* ring, Gen3LogWriter writer, void* ctx);
  bool gen3_log_failed(const Gen3LogRing* ring);
  ```
- Format (exact): `"%08lu p%u n%u i%lu T:" + count×"%04X " + "R:" + count×"%04X " + "E:%04X/%04X\n"`. The words are tx then rx, and E is the timeout and framing masks.

- [ ] **Step 1: Write the failing tests:**

```c
TEST(test_log_fifo)              /* 3 non-idle frames -> pop returns frames 1,2,3 then false */
TEST(test_log_idle_counted)      /* 5 idle burst frames then 1 non-idle -> one record, idle_before==5 */
TEST(test_log_handshake_idle)    /* count 1, rx 0xFFFF, same phase -> not stored */
TEST(test_log_overflow)          /* 70 non-idle frames, no pops -> dropped==6; first pop is frame 0 */
TEST(test_log_format_burst)      /* frame 42, phase 1, idle 0, tx {3355,2222,1133,0..}, rx {B9A0,0..} ->
  "00000042 p1 n9 i0 T:3355 2222 1133 0000 0000 0000 0000 0000 0000 R:B9A0 0000 0000 0000 0000 0000 0000 0000 0000 E:0000/0000\n" */
TEST(test_log_format_handshake)  /* frame 7, phase 0, idle 3, tx {B9A0}, rx {0000} ->
  "00000007 p0 n1 i3 T:B9A0 R:0000 E:0000/0000\n" */
TEST(test_log_writer_failure)    /* writer returns false -> gen3_log_failed() true; later drains don't call writer
                                    and still empty the ring */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement** `gen3_link_log.{h,c}`. Use `head`/`tail` as `volatile uint32_t`, since there is one producer and one consumer and no locks.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add ISR-safe link frame log`.

---

### Task 8: Multiplayer-parent hardware driver

**Files:**
- Create: `src/gen3/mlink_hw_furi.h`, `src/gen3/mlink_hw_furi.c`

**Interfaces:**
- Consumes: `MlinkTx`, `MlinkRx` (Task 2).
- Produces:
  ```c
  typedef struct { const GpioPin* sc; const GpioPin* sd; const GpioPin* so; const GpioPin* si; } MlinkPins;
  typedef enum { MlinkCableOk, MlinkCableReversed } MlinkCable;
  /* Called in ISR context after each frame's last transfer: rx is that frame's result; fill next. */
  typedef void (*MlinkFrameCallback)(void* ctx, const MlinkRx* rx, MlinkTx* next);
  typedef struct MlinkHw MlinkHw;
  MlinkHw* mlink_hw_alloc(const MlinkPins* pins);
  void mlink_hw_free(MlinkHw* hw);
  MlinkCable mlink_hw_check_cable(MlinkHw* hw);   /* SI read low with pull-up => Flipper holds the parent plug */
  /* false (and nothing enabled) if TIM2 is already on (furi_hal_bus_is_enabled) or the cable is reversed */
  bool mlink_hw_start(MlinkHw* hw, MlinkFrameCallback cb, void* ctx, const MlinkTx* first);
  void mlink_hw_stop(MlinkHw* hw);                /* pins back to GpioModeAnalog, TIM2 off, ISR cleared */
  ```

Pinned values (§A2 "Wire level", Timing numbers; GBATEK):

| Item | Value |
|---|---|
| Pin modes while running | SC push-pull out, idle high; SO push-pull out, idle **high** (it is the GBA's SI); SD open-drain out with pull-up, released = high; SI input, pull-up |
| Tick timer | TIM2 via LL, prescaler 0, ARR 48095 (751.5 µs at 64 MHz); ISR set with `furi_hal_interrupt_set_isr(FuriHalInterruptIdTIM2, …)` |
| Frame cadence | accumulator in 0.1 µs: +7515 per tick; when ≥ 167400 (16.740 ms ≈ 59.74 Hz), subtract and start a frame. Transfer *t* runs on tick *t* of the frame (*t* < `tx.count`); then call `cb` |
| One transfer | with `__get_PRIMASK`/`__disable_irq`/`__set_PRIMASK`: SC low; wait 1 bit; send start 0, 16 data bits **LSB first**, stop 1 on SD; release SD; SO low; wait ≤ 100 µs (6400 cycles) for SD low, else word `0xFFFF` and set `timeout_mask` bit; sample data bit *k* at start-edge + (1.5 + *k*) bits, stop at +17.5 bits (low → `framing_mask` bit); SC high; SO high |
| Bit timing | `DWT->CYCCNT`; edge *n* at `t0 + (n * 64000000u + 57600u) / 115200u` cycles (no drift) |
| Bit order switch | `#define MLINK_LSB_FIRST 1` in `mlink_hw_furi.c`, for the hardware gate (U1) |

The callback runs with interrupts enabled. Copy `next` into a driver-owned buffer before the next frame starts. `mlink_hw_check_cable`: configure SI as input with pull-up, wait 1 ms, read it.

- [ ] **Step 1: Implement** `mlink_hw_furi.{h,c}` per the table. Follow `lib/flipper-gblink/gblink/clock_timer.c` for the LL/TIM2/bus calls (read it; don't include it).
- [ ] **Step 2: Build** with `ufbt`. Expected: success with no warnings in `mlink_hw_furi.c`.
- [ ] **Step 3: Run** `make -C tests`. Expected: still PASS. The `_furi` file is excluded from host builds.
- [ ] **Step 4: Commit** `gen3: add GBA multiplayer parent driver (TIM2, bit-banged SD)`.

---

### Task 9: SD pin in the pinout settings

**Files:**
- Create: `src/gen3/gen3_pins_furi.h`, `src/gen3/gen3_pins_furi.c`
- Modify: `src/scenes/pokemon_pins.c` (add an `SD:` row; `select_pins_pin_callback` case 4 → `PIN_SD`; `on_exit` also calls `gen3_sd_pin_save`)
- Modify: `src/pokemon_app.c` (call `gen3_sd_pin_load(pokemon_fap->gblink_handle)` right after `gblink_pinconf_load`)

**Interfaces:**
- Produces:
  ```c
  /* File APP_DATA_PATH(".gen3_sd_pin"), FlipperFormat header "Pokemon Gen3 SD Pin" v1, key "SD" (uint32 gpio_pins index).
   * load: if present and valid, gblink_pin_set(handle, PIN_SD, index); returns false if absent or invalid. */
  bool gen3_sd_pin_load(void* gblink_handle);
  bool gen3_sd_pin_save(void* gblink_handle); /* writes gblink_pin_get(handle, PIN_SD) */
  ```
- In preset modes the SD row shows the preset's pin and isn't editable (one value, like the existing rows), e.g. `SD: PA4` for Original. In Custom it steps through pins like SI/SO/CLK.

- [ ] **Step 1: Implement** `gen3_pins_furi.{h,c}` and the pins scene and app changes.
- [ ] **Step 2: Build** with `ufbt`. Expected: success.
- [ ] **Step 3: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 4: Commit** `pins: show and persist the SD pin for Gen III`.

---

### Task 10: Gen III menu, Link Test screen, session

**Files:**
- Create: `src/gen3/gen3_session_furi.h`, `src/gen3/gen3_session_furi.c`
- Create: `src/views/gen3_link_test.h`, `src/views/gen3_link_test.c`
- Create: `src/scenes/pokemon_gen3_menu.c`, `src/scenes/pokemon_gen3_link_test.c`, `src/scenes/pokemon_gen3_link_exit.c`
- Modify: `src/scenes/include/pokemon_scene_config.h` (append `ADD_SCENE(pokemon, gen3_menu, Gen3Menu)`, `ADD_SCENE(pokemon, gen3_link_test, Gen3LinkTest)`, `ADD_SCENE(pokemon, gen3_link_exit, Gen3LinkExit)`)
- Modify: `src/scenes/pokemon_menu.c` (new item `"Gen III  (GBA non-JPN)"` after Gen II → `PokemonSceneGen3Menu`)
- Modify: `src/include/pokemon_app.h` (`AppViewGen3LinkTest`; field `void* gen3_link_test;`)

**Interfaces:**
- Consumes: Tasks 5, 7, 8, 9; `gblink_pin_get_by_gpiopin(handle, PIN_SERIN/PIN_SEROUT/PIN_CLK/PIN_SD)`.
- Produces:
  ```c
  typedef struct Gen3Session Gen3Session;
  Gen3Session* gen3_session_alloc(void* gblink_handle);
  void gen3_session_free(Gen3Session* s);
  MlinkCable gen3_session_check_cable(Gen3Session* s);
  bool gen3_session_start(Gen3Session* s); /* false if cable reversed or TIM2 busy */
  void gen3_session_stop(Gen3Session* s);
  void gen3_session_status(Gen3Session* s, Gen3ClubStatus* out, bool* log_ok);
  #define GEN3_TEST_TRAINER_NAME "FLIPPER"
  #define GEN3_TEST_TRAINER_TID 22222
  #define GEN3_TEST_TRAINER_SID 0
  ```
- Session internals:
  - **Frame callback (ISR):** `gen3_club_frame`, then `gen3_log_frame`, then write the status into one of two buffers and flip a `volatile` index. The reader copies the other buffer.
  - **Log thread** `"Gen3Log"`, stack 1024: every 100 ms, `gen3_log_drain` to `APP_DATA_PATH("gen3_link.log")`. The file is opened at start with `FSAM_WRITE, FSOM_CREATE_ALWAYS`, and the first line is `# gen3 link log v1`.
  - **No SD card:** if the file can't be opened, `log_ok = false` and drains discard. The link keeps running.
- Mapping from the gblink pins: SC = `PIN_CLK`, SD = `PIN_SD`, SO = `PIN_SEROUT`, SI = `PIN_SERIN`.
- **View** (128×64): title `Gen III Link Test` (FontPrimary). Then FontSecondary lines:
  - `Cable: OK` / `Cable: SWAP ENDS`
  - `Link: <phase>`
  - `<partner name> TID <tid>` when `partner_valid`
  - `Frm <frames> Err <checksum_errors> T/O <timeouts>`
  - `Log: on` / `Log: off`
- Phase text:
  - Waiting → `waiting for GBA`
  - Linked → `linked`
  - Players → `players exchanged`
  - Cards → `trainer cards`
  - Room → `in Trade Center`
  - Closing → `closing`
  - Error: Checksum → `error: checksum`, BadPlayerBlock → `error: bad data`, OtherRoom → `GBA chose other room`
- The view refreshes from a 250 ms `FuriTimer`.
- **Link Test scene:**
  - `on_enter`: alloc the session. If the cable is reversed, show `SWAP ENDS` and re-check on every timer tick; start once it reads OK.
  - Back while the phase is Linked..Room → `Gen3LinkExit`. Otherwise stop the session and go back.
  - `on_exit`: stop and free.
- **`Gen3LinkExit`** (DialogEx): header `Stop link?`; text `Your GBA will show a link error and need a restart. It saved before linking.`; left `Stay`, right `Stop`.

- [ ] **Step 1: Implement** the session, view and scenes above, following the patterns in `src/views/trade.c` and `src/scenes/pokemon_exit_confirm.c`.
- [ ] **Step 2: Build** with `ufbt`. Expected: success.
- [ ] **Step 3: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 4: Smoke-test the Gen I/II menus by reading the code:** the main menu still lists Gen I, Gen II and Pinout with unchanged indexes and scene states.
- [ ] **Step 5: Commit** `gen3: add Gen III menu and Link Test screen`.

---

### Task 11: Hardware test checklist and docs

**Files:**
- Create: `docs/gen3/hardware-test-1.md`
- Modify: `README.md` (short "Gen III (experimental): link test" section: which cable plug goes in the Flipper, the `SWAP ENDS` hint, and a link to the checklist)
- Modify: `changelog.md` (new top section `## Unreleased`: "Gen III link test (experimental)")

`hardware-test-1.md` must contain, in order:
1. **Build and install:** `ufbt` (release), then copy `dist/pokemon.fap` to `apps/GPIO/` with qFlipper, or run `ufbt launch`.
2. **Ruby prerequisites:** Pokedex received, at least 2 Pokemon in the party, no Enigma Berry held. Ruby saves at the counter before linking, so a power-cycle can't lose progress.
3. **Pinout check:** Main menu → Select EXT Pinout → Original shows `SD: PA4`.
4. **Cable orientation (Review Focus 5):** plug one end into the Flipper and open Gen III → Link Test. `Cable: SWAP ENDS` → swap the plugs; it must change to `Cable: OK` with no transfers started before that. Write down which plug colour went into the Flipper.
5. Ruby: Pokemon Center 2F → right counter (Trade Center) → YES → save → "Please wait".
6. **Expected on the Flipper:** `linked` → `players exchanged` (partner name and TID shown) → `trainer cards` → `in Trade Center`. **Expected on Ruby:** "Please enter.", then you're in the room with a second avatar.
7. Walk around for 60 s. Talk to the other avatar, which shows the trainer card `FLIPPER`. `Err` stays 0.
8. Walk to the exit, face south, choose YES. Ruby returns to the Pokemon Center with no error, and the Flipper shows `waiting for GBA`.
9. Repeat steps 5–8 once without restarting the app (Review Focus 2).
10. **Results table:** pass/fail per step. On any failure, the step number, what each screen showed, and the file `apps_data/pokemon/gen3_link.log` from the SD card.

- [ ] **Step 1: Write** the checklist, README section and changelog entry.
- [ ] **Step 2: Build** with `ufbt` and run `make -C tests`. Expected: both pass.
- [ ] **Step 3: Commit** `docs: add Gen III hardware test checklist`.

---

### Task 12: Hardware gate (the user runs it)

- [ ] **Step 1:** Push the branch and give the user `dist/pokemon.fap` and `docs/gen3/hardware-test-1.md`. **Stop here until the user reports results.**
- [ ] **Step 2: All steps pass** → write plan 2 (data core and editor) and plan 3 (Ruby trade).
- [ ] **Step 3: Any failure** → read the log:
  - Every rx word bit-reversed (e.g. `059D` instead of `B9A0`) → flip `MLINK_LSB_FIRST`.
  - Timeouts on every transfer → the hand-over or timeout window is wrong (U1); widen the 100 µs and recheck SO/SD wiring.
  - Checksum errors in the room → transfer gaps or ISR jitter (U2).
  
  Fix, rebuild, and retest only the failing steps.
- [ ] **Step 4:** If the timing can't be held reliably, stop and report to the user with the logs. Per spec call #1, the fallback is to do plan 5 (direct save access) first.

## Amendments made during execution

The tasks above are kept as written. These changes were made while executing them; the code is the reference.

- **Burst gap:** the transfer tick is ≈1073 µs start-to-start (`MLINK_TIM_ARR` 68671, `MLINK_TICK_DECI_US` 10730 in `src/gen3/mlink_hw_furi.c`, not ARR 48095 / +7515), so with a ≈321.5 µs transfer the gap from the end of one transfer to the start of the next is 751.5 µs, matching the GBA.
- **SD drive:** SD is driven push-pull for the Flipper's own word, start bit through stop bit, and released to input with pull-up otherwise (`MLINK_SD_PUSH_PULL` 1 in `mlink_hw_furi.c`; 0 restores the open-drain design).
- **Cable check:** the cable line reads `Cable: OK` / `Cable: plug in/swap ends` (`src/views/gen3_link_test.c`), not `SWAP ENDS`. It stays live through a per-frame SI guard, with SI also latched on every idle tick, that skips transfers and releases SC/SO (SD is already released between transfers) (`mlink_hw_isr`, `mlink_lines_release`, `mlink_hw_cable_ok`).
- **CAFE 0017:** it is answered only after the GBA's next command, because Ruby drops the key of the update right after its own 0x17 (pokeruby `overworld.c: sub_8054FF8`; `on_room_command` and `exit_answer_due` in `src/gen3/gen3_club.c`).
- **Checksum errors:** they no longer flush the queue or enter Error. The error is counted (`Err`), the club sets `last_error` and keeps answering, and the link layer still delivers that burst's command (`burst_rx` in `src/gen3/gen3_link.c`, `Gen3LinkEventChecksumError` in `gen3_club_frame`). The club resets on Disconnected. So `error: checksum` is gone from the Link Test.
- **Link Test lifecycle:** Stay or Back on the "Stop link?" dialog keeps the session running; only `Stop`, or Back outside the club phases, stops it (`src/scenes/pokemon_gen3_link_exit.c`, `pokemon_gen3_link_test.c`). The dialog says the GBA *may* show a link error.
- **Driver additions:** insomnia while linked (`furi_hal_power_insomnia_enter` in `mlink_hw_start`, so TIM2 keeps running), and SC raised at +18 bits after the child's start edge, the end of its stop bit (`MLINK_RX_END_HALF` 36).
