# Gen III Cartridge over a Game Boy Color Cable (Normal Mode) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Pokemon Cartridge (multiboot, backup, edit, write, restore) works over a Game Boy Color link cable. It uses the GBA's
Normal serial mode, which needs no SD line. This is for Game Link modules whose SD line isn't connected, such as KBEmbedded's
module with its SD jumper open. The GBA-cable MultiPlay path keeps working exactly as before.

**Architecture:** The existing TIM2 driver gains a second *wire*: `MlinkWireNormal`. Each tick, it bit-bangs one 32-bit
transfer at 256 kHz on SC/SO/SI, but only once the slave signals ready (SI low). The 16-bit `MlinkTx`/`MlinkRx` contract
stays the same:
- the Flipper's word goes in the low half of the 32-bit word, the GBA's word comes back in the high half;
- `MlinkTx.hi` carries the high half where multiboot needs it;
- a slot whose slave never got ready is reported as *not ready*.

On top of that:
- the multiboot sender gains Normal mode's constants and 32-bit data units;
- the packet pump treats not-ready as BUSY;
- the helper picks its transport from the boot-mode byte the BIOS writes;
- the Cartridge runner picks the wire from SI at Connect.

**Tech Stack:** C11. Flipper SDK via `ufbt` (release SDK 1.5.1, API 88.2) and the dev SDK. STM32 LL/DWT. The helper uses
`arm-none-eabi-gcc` 12.3.1 (ufbt toolchain). Host tests use `make -C tests`.

**Spec:** `docs/superpowers/specs/2026-10-25-gen3-gba-design.md` (Phase 2) as amended by the design decisions below. The
user approved them on 2026-10-30. Technical reference, cited as §n: `docs/gen3/multiboot-and-save.md`. Task 7 adds its §7
and M23–M27.

## Background (why)

- On 2026-10-30 the user's first Ruby link failed. The log showed every frame as `T:B9A0 R:FFFF` with a timeout.
- A probe drove SO low so that the waiting Ruby acted as master. SC (PB2) showed Ruby's transfers. SD showed nothing on
  PA4, PA7, PA6, PC1 or PC0.
- The KBEmbedded module's SD jumper is open. MultiPlay needs SD, so neither the trade nor the current Cartridge app can
  work on it. The user can't solder.
- A GBC link cable crosses SO and SI and carries SC, which is what Normal mode needs (§1; GBATEK "SIO Normal Mode").
- A second probe ran with the GBC cable and the GBA powered on with START+SELECT held. The Flipper ran a JS bit-banged
  Normal master: MSB first, SO set after SC falls, SI read after SC rises, sending `0x62026202`. The BIOS answered
  `FFFFFFFF`, then `0000xxxx`, then a steady `72026203`. So Normal-mode multiboot reaches the BIOS both ways.

## Plan sequence

This is plan 6. It is a follow-up to plan 5 (`2026-10-25-gen3-05-save-access.md`, v3.1).

**Branch:** `gen3-save-access` (draft PR #2) in the worktree `.claude/worktrees/gen3-design`. Before Task 1, record
`BASE=$(git rev-parse HEAD)` in your report (expected `473ec24` or later). The size check uses it.

## Global Constraints

- Commits are authored by the user (LawlessNPC, Aleksandr Kofman), with **no** AI-attribution trailers: no `Co-Authored-By`, no `Claude-Session`, no "Generated with". After every commit, `git log -1 --format='%(trailers)'` must print nothing.
- Stage only the files your task lists, by explicit path. Never use `git add -A`, `git add .`, a directory, or `git commit -a`.
- Both SDK builds finish with 0 warnings: `ufbt` (release) and `UFBT_HOME=/Users/kyvo/.claude/jobs/72354b36/tmp/ufbt-dev ufbt` (dev). Both write `dist/`, so run the release build last. `make -C tests` passes after every task. Any task that touches `gba/helper/` also passes `make -C gba/helper check`.
- **Pure-module rule:** every `src/gen3/` file whose name doesn't end in `_furi.c`/`_furi.h` includes only `<stdint.h> <stdbool.h> <stddef.h> <string.h> <stdio.h>` and other pure `src/gen3/` headers.
- **Helper portable rule:** `gba/helper/src/*` and `gba/helper/include/*` stay portable. Only `gba/helper/gba/*` and `crt0.S` touch GBA registers. Helper symbols start with `hp_`/`Hp`/`HP_`.
- **ISR rule:** the frame callback and everything it calls (`gen3_mb_frame`, `gen3_hlink_frame`) do no allocation, no logging, no furi call, and return quickly.
- **MultiPlay unchanged:** frame mode, the multiplayer wire and every Pokemon Trade Tool path behave exactly as before. Existing tests change only mechanically, for the new parameters (`gen3_mb_init`, `gen3_mb_crypt_init`, `gen3_mb_crc_step`) and for `rx.ticks`. Their assertions don't change.
- **Save safety (plan 5):** never write without a verified backup; never write the slot the game loads; never touch sectors 28–31; never chip-erase; write only the plan 5 allow-list. Nothing in this plan touches `gen3_cart.c`'s rules.
- **Helper image:** `files/gen3_helper.mb` is written only by `make -C gba/helper install` and stays at most 8192 bytes. `HP_PROTO_VERSION` and `HP_BUILD_ID` stay 1, since v3.1 hasn't shipped.
- **Size:** `text + data` of `~/.ufbt/build/pokemon_d.elf` and of `~/.ufbt/build/pokemon_cart_d.elf` each grow by at most 4,000 bytes over `BASE`. They were 89,456 and 70,968 at `473ec24`. Measure with `~/.ufbt/toolchain/current/bin/arm-none-eabi-size`.
- **MIT, no copied code.** GBATEK and afska/gba-link-connection are references for behaviour and constants only.
- `fap_version` stays `[3,1]`. Never modify `lib/flipper-gblink`.
- Implementers can't run on-device checks. The user runs `docs/gen3/hardware-test-3.md` at the end.

## Review Focus

1. **The helper is busy for seconds over the Normal wire** (erase or program, a cart scan, READ_SECTOR's CRC). The slave isn't ready, so no word moves. The request must stay alive within its BUSY budget, never `NoResponse`. *(Task 1: `test_hlink_notready_waiting_is_busy`, `test_hlink_notready_busy_budget`; Task 2: `test_hcli_normal_busy_flash`.)*
2. **The GBC cable is pulled, or the GBA switched off, in the middle of a write.** The game still loads the old or the complete new save. The failure surfaces as `BusyTimeout`/`LinkLost` within the budget, not as a hang, and Reconnect finishes the write. *(Task 2: `test_write_link_drop_sweep_normal`, `test_hcli_normal_unplug`.)*
3. **A word that was never delivered inside a packet**, because the slave wasn't ready mid-request or mid-response. The packet fails as `Wire`, the client's resend succeeds, and no data is misread. *(Task 1: `test_hlink_notready_in_response_is_wire`; Task 2: `test_hcli_normal_resend_after_stall`.)*
4. **A GBA cable whose SI is grounded** (a module with SD connected) is never moved to the Normal wire, and all existing MultiPlay tests pass. *(Task 3: `test_mb_wire_next`; Task 5: the runner calls only `gen3_mb_wire_next`.)*
5. **Normal-mode multiboot details GBATEK leaves implicit:** the data-reply offsets, the zero upper half and the constants. A mismatch restarts the sender and records the expected and received words for the log. It never hangs. *(Task 3: `test_mb_normal_boot`, `test_mb_normal_data_reply_mismatch`, `test_mb_normal_vector`.)*

## Design decisions

**D1. One driver, two wires.**
- `MlinkWire { MlinkWireMulti, MlinkWireNormal }` lives in `mlink_types.h`.
- The Normal wire exists in stream mode only, and has no cable guard. `mlink_hw_cable_ok` reads true while it runs.
- The tick stays 1073 µs, so throughput equals MultiPlay's. A faster tick is a later change.
- SD is left alone, in analog mode: Normal mode doesn't use it.

**D2. The 32-bit word on the Normal wire.**
- The Flipper sends `(uint32_t)tx.hi[t] << 16 | tx.word[t]`, MSB first, and reports the reply's **high** half as
  `rx.word[t]`.
- The driver zeroes `next.hi[]` before every frame callback, so only a producer that wants a high half sets one. Only
  multiboot's data words do.
- This is GBATEK's Normal-mode multiboot convention: the master's value is in the low half with the upper half zero, and
  the reply follows the MultiPlay pattern in the high half. The helper protocol uses the same convention.

**D3. Ready, stall, not-ready.**
- At each tick, the driver reads SI.
  - **SI high:** the slave isn't ready. The slot waits for the next tick, up to `MLINK_NORMAL_STALL_TICKS` (16) ticks.
    After that it is reported *not ready*: `notready_mask` and `timeout_mask` bit set, word `MLINK_NO_DATA`, nothing
    clocked.
  - **SI low:** the driver masks interrupts, waits 4 µs, checks SI is still low, and clocks 32 bits.
- `rx.ticks` reports the ticks the frame took. Callers convert with `rx.ticks * MLINK_TICK_US`. On the MultiPlay wire,
  `ticks = max(count, 1)`, a cable-guard-skipped frame included, so the cores' timing there is unchanged.

**D4. Not ready means BUSY in the pump.**
- A Normal-wire slave can't answer while its CPU runs a command, so the helper sends no `HP_BUSY` words.
- In `gen3_hlink`, a not-ready word is handled like a valid `HP_BUSY` wherever the pump waits: Sending before READY, and
  Waiting.
- In Sync it breaks the run of READYs.
- Inside a response it is a wire error, like a timed-out word.

**D5. The helper picks its transport.**
- The BIOS writes the boot mode at header `0xC4`: 2 = Normal, 3 = MultiPlay (GBATEK "Multiboot Header").
- With 2, `main.c` runs the Normal slave loop below. With anything else it runs today's MultiPlay loop.
- Normal slave transfer, from GBATEK's slave procedure (version B), plus an explicit not-ready after every transfer:
  1. `SIODATA32 = out << 16`.
  2. `SIOCNT = 0x5000`: 32-bit, IRQ, external clock, start 0, SO low ("ready").
  3. `SIOCNT = 0x5088`: start, with SO high after the transfer.
  4. Wait for the transfer's end (IF bit 7, as today) for up to 200 ms.
  5. `SIOCNT = 0x5008`: not ready, whatever the hardware left.
  6. The Flipper's word is `(uint16_t)SIODATA32`.
- After a packet's last word, `hp_execute` runs at once. No BUSY transfer happens, and the next transfer carries the
  response's SOF.

**D6. The Cartridge runner picks the wire at Connect.**
- `gen3_mb_wire_next(wire, si_low, phase)` decides:
  - MultiPlay stays only while SI reads low, or once the GBA has answered (any phase past Detect);
  - SI reading high while the sender is in Wait or Detect switches to Normal;
  - Normal never switches back within one Connect.
- Connect starts with `gen3_mb_wire_next(MlinkWireMulti, si_low_now, Gen3MbPhaseDetect)`.
- A reversed GBA cable or no cable therefore ends up on the Normal wire, showing `Waiting for GBA` and then the power-on
  hint. `Cable: plug in/swap ends` no longer shows during Connect.

## Reference facts (GBATEK; checked against the 2026-10-30 probe where marked)

| Item | MultiPlay (today) | Normal (new) |
|---|---|---|
| Unit on the wire | 16 bits | 32 bits; master's value in the low half, upper half zero; reply in the high half (probe: `7202` in the high half) |
| CRC init / polynomial | `0xFFF8` / `0xA517` | `0xC387` / `0xC37B` |
| Data key xor | `0x6465646F` | `0x43202F2F` |
| Seed multiplier, seed init, `hh`, length word, final CRC form | same | same |
| Data block | two 16-bit halves per word, low first | one 32-bit encrypted word per transfer |
| Data reply | low 16 bits of the byte offset of the halfword (current check) | low 16 bits of the byte offset of the word, `(uint16_t)(0xC0 + 4k)` **(inferred, M23)** |
| Master clock | 115200 bit/s on SD | 256 kHz on SC: bit out after SC falls, sampled after SC rises; slave ready = master's SI low |

Independent Normal-mode vector (Python from GBATEK's formulas), with `pp=0x93` and `cc=0x5A`:
- seed `0xFFFF5A93`, crc `0xC387`;
- word at `0xC0`, plain `0x12345678` → `0x7036291D`, crc `0x4DD0`;
- word at `0xC4`, plain `0x9ABCDEF0` → `0x47EF9D9C`, crc `0x1271`;
- `hh(0x5A) = 0x69`; the final CRC with `rr=0x3C` is `0xF80B`.

---

### Task 1: Link contract for the Normal wire, and not-ready in the packet pump

**Files:**
- Modify: `src/gen3/mlink_types.h`, `src/gen3/gen3_hlink.h` (comments), `src/gen3/gen3_hlink.c`
- Test: `tests/test_gen3_hlink.c`

**Interfaces:**
- Produces, in `mlink_types.h`:
  - `typedef enum { MlinkWireMulti, MlinkWireNormal } MlinkWire;`
  - `#define MLINK_NORMAL_STALL_TICKS 16u`
  - `MlinkTx` gains `uint16_t hi[MLINK_BURST_LEN];`: the Normal wire's upper halves, zeroed by the driver before every
    frame callback, ignored by the MultiPlay wire.
  - `MlinkRx` gains `uint16_t notready_mask;`: Normal wire, bit t set when the slave never got ready and nothing was
    clocked; `timeout_mask` has the bit too.
  - `MlinkRx` gains `uint16_t ticks;`: stream mode, the frame's time in ticks as in D3.
- The `gen3_hlink` API is unchanged. Its behaviour changes per D4.

- [ ] **Step 1: Write the failing tests.** In `tests/test_gen3_hlink.c`, each test builds `MlinkRx` frames by hand as the
  file's helpers do. A *not-ready word* has `word = MLINK_NO_DATA` and its bit set in both `timeout_mask` and
  `notready_mask`. A *not-ready frame* is 9 such words, with `elapsed_us = 9 * MLINK_NORMAL_STALL_TICKS * MLINK_TICK_US`
  (154,512).
  - `test_hlink_notready_waiting_is_busy`: submit HELLO, feed READY, let the request go out, then 37 not-ready frames.
    The state is still `Gen3HlinkWaiting`. Then SOF and a valid HELLO response give `Done`.
  - `test_hlink_notready_busy_budget`: as above with `busy_us = 0` (6 s). After 38 not-ready frames the state is
    `Waiting`. After the 39th it is `Failed` with `Gen3HlinkFailBusyTimeout`.
  - `test_hlink_notready_before_ready`: submit, then 20 not-ready frames (180 words, over
    `GEN3_HLINK_NOT_READY_XFERS`). The state is `Sending`, not `NoReady`. A READY then sends the request.
  - `test_hlink_notready_in_response_is_wire`: a response SOF, w1, `n = 2`, a not-ready word, then the remaining words.
    The result is `Failed`, `Gen3HlinkFailWire`.
  - `test_hlink_notready_sync_resets_run`: `sync(4, 3 s)`. READY, READY, a not-ready word, READY ×3 → still `Sync`. One
    more READY → `Done`.
- [ ] **Step 2: Run them and see them fail to compile or fail.** Run `make -C tests run-test_gen3_hlink`. Expected: FAIL
  (`notready_mask` unknown).
- [ ] **Step 3: Add the types to `mlink_types.h`, then give `hlink_word` a `bool notready` parameter.**
  `gen3_hlink_frame` passes `(rx->notready_mask >> t) & 1`. In `hlink_wait_other`, a not-ready word acts as a valid
  BUSY. In Sync and Receiving it acts as an invalid word. Update the `gen3_hlink.h` comment block per D4. Any test helper
  or fake that builds an `MlinkRx` field by field must zero-initialise it first (`MlinkRx rx = {0};`), so that
  `notready_mask` is never garbage.
- [ ] **Step 4: Run the tests.** Run `make -C tests`. Expected: every suite passes, `test_gen3_hlink` with 5 more tests.
- [ ] **Step 5: Build both SDKs (0 warnings), then commit** `src/gen3/mlink_types.h src/gen3/gen3_hlink.h src/gen3/gen3_hlink.c tests/test_gen3_hlink.c`
  with `gen3: link contract for the GBA Normal wire; the packet pump takes not-ready as busy`.

### Task 2: The fake helper on the Normal wire, and the client and write runs over it

**Files:**
- Modify: `tests/fake_helper.h`, `tests/fake_helper.c`
- Test: `tests/test_gen3_hcli.c`, `tests/test_gen3_cart_write.c`

**Interfaces:**
- Consumes: Task 1's `MlinkWire`, `MLINK_NORMAL_STALL_TICKS`, `MlinkRx.notready_mask` and `.ticks`.
- Produces: `void fake_helper_set_wire(FakeHelper* f, MlinkWire wire);`. The default after `fake_helper_init` is
  `MlinkWireMulti`, and `fake_helper_restart` keeps the wire. In `MlinkWireNormal` the fake differs from MultiPlay in
  four ways:
  1. While a command runs (until `busy_end_us`), every slot is not ready. The word isn't delivered (no `hp_step`), and the
     slot costs `MLINK_NORMAL_STALL_TICKS` ticks.
  2. There is no `HP_BUSY` reply and no extra "fresh transfer". The first delivered slot after `busy_end_us` replies with
     the response's SOF.
  3. Unplugged, or dead after a power cut, every slot is not ready, at `MLINK_NORMAL_STALL_TICKS` ticks each.
  4. `rx->ticks` is the sum: 1 per delivered slot, `MLINK_NORMAL_STALL_TICKS` per not-ready slot, 1 for a count-0 frame.
  On the MultiPlay wire, `rx->ticks = max(count, 1)`. `fake_helper_transport` passes `rx.ticks * MLINK_TICK_US` as
  `elapsed_us` in both wires, which equals today's `mlink_frame_us` on MultiPlay. `fake_helper_transfers` counts every
  slot the Flipper sends, delivered or not ready, so `fake_helper_unplug_at` means the same in both wires.

- [ ] **Step 1: Write the failing tests.**
  - `test_fake_helper_normal_busy_is_notready` (`test_gen3_hcli.c`): Normal wire, HELLO sent word by word. From the slot
    after its last word, every slot is not ready until the modelled busy time passes. Then the next delivered slot
    replies `HP_SOF`, and no slot ever replies `HP_BUSY`.
  - `test_hcli_normal_roundtrip`: Normal wire. Mirror `test_hcli_connect`, `test_hcli_cart_info` and
    `test_hcli_read_sector` in one test, with the same assertions.
  - `test_hcli_normal_busy_flash`: Normal wire. Set up a `WRITE_SECTOR` the way `test_hcli_busy_per_command` does, with
    a modelled busy time of 3 s. It returns `Gen3HcliOk`, and the flash holds the image.
  - `test_hcli_normal_unplug`: Normal wire. An unplug forever from the first word of a `READ_SECTOR` response returns
    `Gen3HcliLinkLost` or `Gen3HcliBusyTimeout`, and `fake_helper_transfers` stays under `FAKE_HELPER_MAX_TRANSFERS`.
  - `test_hcli_normal_resend_after_stall`: Normal wire. An unplug of 1 transfer at the 5th response word of a
    `READ_SECTOR` still returns `Gen3HcliOk`, with `resends >= 1` and the sector data correct.
  - `test_write_link_drop_sweep_normal` (`test_gen3_cart_write.c`): `test_write_link_drop_sweep` with
    `fake_helper_set_wire(&fh, MlinkWireNormal)` right before `snap_take()`, so that every `snap_restore` keeps it. Make
    the result check `r == Gen3CartLinkLost || r == Gen3CartBusyTimeout || r == Gen3CartOk`, and count both failures as
    lost. Every other assertion stays as written.
- [ ] **Step 2: Run them.** Run `make -C tests run-test_gen3_hcli run-test_gen3_cart_write`. Expected: FAIL to compile
  (`fake_helper_set_wire` undefined).
- [ ] **Step 3: Implement `fake_helper_set_wire` and the four behaviours above in `fake_helper.c`, and update the
  comments in `fake_helper.h`.**
- [ ] **Step 4: Run `make -C tests`.** Expected: all pass, old tests unchanged.
- [ ] **Step 5: Commit** `tests/fake_helper.h tests/fake_helper.c tests/test_gen3_hcli.c tests/test_gen3_cart_write.c`
  with `tests: the fake helper on the GBA Normal wire; client and write runs over it`.

### Task 3: Normal-mode multiboot sender and fake BIOS, and the wire choice

**Files:**
- Modify: `src/gen3/gen3_mb.h`, `src/gen3/gen3_mb.c`, `tests/fake_bios.h`, `tests/fake_bios.c`
- Test: `tests/test_gen3_mb.c`, plus mechanical call-site updates wherever these functions are called (`grep -rn "gen3_mb_init\|gen3_mb_crypt_init\|gen3_mb_crc_step" src tests`)

**Interfaces:**
- Consumes: Task 1's types.
- Produces:
  - `uint32_t gen3_mb_crc_step(uint32_t crc, uint32_t data, uint16_t poly);`
  - `Gen3MbCrypt` gains `uint16_t poly; uint32_t key;`.
  - `void gen3_mb_crypt_init(Gen3MbCrypt* k, MlinkWire wire, uint8_t pp, uint8_t cc);` sets the init, polynomial and
    key from the table above. `gen3_mb_crypt_final` uses `k->poly`.
  - `void gen3_mb_init(Gen3Mb* mb, MlinkWire wire, const uint8_t* img, uint32_t len, uint32_t seed);`
  - `Gen3Mb` gains `MlinkWire wire`. `Gen3MbStatus` gains `uint16_t fail_want, fail_got;`: the expected and received
    word of the latest Header/Data/Handshake/Length/Crc failure (`fail_got` is `MLINK_NO_DATA` for an invalid word), and
    0, 0 until one happens.
  - `MlinkWire gen3_mb_wire_next(MlinkWire wire, bool si_low, Gen3MbPhase phase);` per D6.
  - `void fake_bios_set_wire(FakeBios* b, MlinkWire wire);` (default Multi). Both wires set `rx->ticks` as in Task 2.
- Normal data phase in `gen3_mb_frame`:
  - one transfer per 32-bit word: `tx->word[i] = (uint16_t)e`, `tx->hi[i] = (uint16_t)(e >> 16)`;
  - the reply for word k must equal `(uint16_t)(0xC0 + 4 * k)`;
  - `sent = 4 * words acknowledged`;
  - every other word leaves `hi` at 0.
- Normal fake BIOS:
  - every slot is not ready (as in Task 2, item 1) while Off, during the 2 startup transfers, and once Booted;
  - any non-data word with a non-zero `hi` sets the error `"upper half not zero"`;
  - a data transfer takes one 32-bit word, decrypted with Normal constants, and loads the low 16 bits of the next
    expected word's offset as its reply.
  - Its CRC and decryption are written in the fake from GBATEK, not shared with `gen3_mb.c`, as today.

- [ ] **Step 1: Write the failing tests** in `tests/test_gen3_mb.c`. Switch the bench to
  `elapsed = rx.ticks * MLINK_TICK_US`.
  - `test_mb_normal_vector`: the vector above, through `gen3_mb_crypt_init(&k, MlinkWireNormal, 0x93, 0x5A)`,
    `gen3_mb_crypt_word` ×2, `gen3_mb_hh(0x5A) == 0x69`, `gen3_mb_crypt_final(&k, 0x69, 0x3C) == 0xF80B`.
  - `test_fake_bios_normal_vector`: the fake decrypts `0x7036291D` at `0xC0` to `0x12345678` (mirror
    `test_fake_bios_doc_vector`).
  - `test_mb_normal_boot`: Normal wire on both. The sender reaches `Done` and `fake_bios_boots == 1`. The image the fake
    received equals the helper test image byte for byte, `fake_bios_error == NULL`, and `status.sent == status.total`.
  - `test_mb_normal_waits_for_gba`: the BIOS stays Off for 200 frames (all not ready), then powers on. The sender boots
    it, with `restarts == 0`.
  - `test_mb_normal_data_reply_mismatch`: `fake_bios_corrupt_reply` on the 3rd data transfer. The sender restarts once
    (`restarts == 1`, `last_fail == Gen3MbFailData`), `fail_want == 0x00C8`, `fail_got == (0x00C8 ^ 0x0100)`, and then
    it boots.
  - `test_mb_wire_next` (table):
    - `(Multi, true, Detect) → Multi`;
    - `(Multi, false, Detect) → Normal`;
    - `(Multi, false, Wait) → Normal`;
    - `(Multi, false, Header) → Multi`;
    - `(Multi, false, Done) → Multi`;
    - `(Normal, true, Detect) → Normal`;
    - `(Normal, false, Data) → Normal`.
  - Every existing test passes `MlinkWireMulti` (or `0xA517` to `gen3_mb_crc_step`) with unchanged assertions.
- [ ] **Step 2: Run** `make -C tests run-test_gen3_mb`. Expected: FAIL to compile.
- [ ] **Step 3: Implement it in `gen3_mb.c` and `fake_bios.c`, and update the comments in `gen3_mb.h` and
  `fake_bios.h`.** Mechanically update the other callers: `src/gen3/gen3_cart_furi.c` passes `MlinkWireMulti` for now,
  and Task 5 replaces that.
- [ ] **Step 4: Run** `make -C tests` and both SDK builds. Expected: all pass, 0 warnings.
- [ ] **Step 5: Commit** the files above, plus each call site you touched, with
  `gen3: multiboot in GBA Normal mode (32-bit words, Normal keys); fake BIOS on the Normal wire; the Connect wire choice`.

### Task 4: The driver's Normal wire

**Files:**
- Modify: `src/gen3/mlink_hw_furi.h`, `src/gen3/mlink_hw_furi.c`

**Interfaces:**
- Consumes: Task 1's types.
- Produces: `bool mlink_hw_start_wire(MlinkHw* hw, MlinkMode mode, MlinkWire wire, MlinkFrameCallback cb, void* ctx, const MlinkTx* first);`
  - `mlink_hw_start_mode(...)` is exactly `mlink_hw_start_wire(..., MlinkWireMulti, ...)`.
  - `MlinkWireNormal` with `MlinkModeFrame` returns false.
  - `MlinkWireNormal` does no cable check, so it fails only if TIM2 is taken or the driver is running.
  - `mlink_hw_cable_ok` reads true from a Normal start until the next MultiPlay start.
- Normal-wire constants in `mlink_hw_furi.c`, with `static_assert`s:
  - `MLINK_NORMAL_BIT_CYCLES 250u` (256 kHz at 64 MHz);
  - `MLINK_NORMAL_SETTLE_CYCLES 256u` (4 µs);
  - `MLINK_NORMAL_SAMPLE_CYCLES 187u` (SI read 62 cycles after SC rises);
  - `MLINK_NORMAL_SETTLE_CYCLES + 32 * MLINK_NORMAL_BIT_CYCLES` must stay under one tick.
- Pins (Normal): SO and SC push-pull, idle high; SI input with pull-up; SD left analog.
- Per tick in a Normal frame, for slot t, follow D3.
  - The transfer, with interrupts masked: for each bit from 31 down to 0,
    - at `t0 + (31 - b) * BIT`: SC low, SO = bit b;
    - at `+ BIT / 2`: SC high;
    - at `+ SAMPLE`: shift SI in.
  - Then SC and SO go high.
  - `rx.word[t] = reply >> 16`.
  - Count every tick of the frame into `rx.ticks`. Before the callback, memset `next.hi` to 0.
- MultiPlay: set `rx.ticks = max(count, 1)` at each stream-mode frame end, and `notready_mask = 0`. Nothing else
  changes.

- [ ] **Step 1: Implement it.** There's no host test for a furi file. The `static_assert`s and both builds are the check.
  Keep the masked-window comment style of `mlink_hw_transfer`.
- [ ] **Step 2: Build both SDKs.** Expected: 0 warnings. Run `make -C tests`: all pass.
- [ ] **Step 3: Self-check against D3.** Read the diff of the frame-mode paths: it must be empty apart from the `ticks`
  and `notready_mask` assignment.
- [ ] **Step 4: Commit** `src/gen3/mlink_hw_furi.h src/gen3/mlink_hw_furi.c` with
  `gen3: driver: the GBA Normal wire (32-bit, 256 kHz, slave-ready gate, not-ready slots)`.

### Task 5: The Cartridge runner picks and switches the wire

**Files:**
- Modify: `src/gen3/gen3_cart_furi.c`

**Interfaces:**
- Consumes: `gen3_mb_wire_next`, `gen3_mb_init(..., wire, ...)`, `mlink_hw_start_wire`, `rx->ticks`, and
  `Gen3MbStatus.fail_want`/`fail_got`.
- The session gains `MlinkWire wire;`.
- `gen3_cart_frame` uses `elapsed_us = rx->ticks * MLINK_TICK_US`.
- `cart_driver_start`:
  - set `wire = gen3_mb_wire_next(MlinkWireMulti, mlink_hw_check_cable(hw) == MlinkCableOk, Gen3MbPhaseDetect)`;
  - init the sender for that wire and start it with `mlink_hw_start_wire(hw, MlinkModeStream, wire, ...)`;
  - drop the wait-for-cable loop;
  - if a MultiPlay start fails because SI now reads high, go Normal;
  - any other start failure stays `Gen3CartLinkBusy`;
  - log `wire multi` or `wire normal`.
- `cart_multiboot`, on each poll: if `gen3_mb_wire_next(s->wire, mlink_hw_cable_ok(s->hw), st.phase)` differs, stop the
  driver, init the sender for Normal, restart, and log `wire normal (SI high)`.
- `cart_log_mb` appends ` want %04x got %04x`.
- Status: `cable_ok` stays true on the Normal wire.

- [ ] **Step 1: Implement it.** The pump phase keeps `s->wire`; nothing restarts after the sender is Done.
- [ ] **Step 2: Build both SDKs** (0 warnings) and run `make -C tests` (all pass). Check the size budget for both apps.
- [ ] **Step 3: Commit** `src/gen3/gen3_cart_furi.c` with
  `gen3: cart: connect over a GBC cable (Normal wire) when SI isn't grounded, MultiPlay otherwise`.

### Task 6: The helper's Normal transport

**Files:**
- Modify: `gba/helper/gba/main.c`, `files/gen3_helper.mb` (written by `make -C gba/helper install` only)

**Interfaces:**
- In `main.c`:
  - `HP_SIODATA32` is `*(volatile uint32_t*)0x04000120`;
  - `HP_SIOCNT_NORMAL 0x5000u`, `HP_SIOCNT_SO_HIGH 0x0008u`;
  - `HP_BOOT_NORMAL 2u` (header byte `0xC4`, from `hp_hal_own_header()`).
- `static uint16_t hp_xfer_normal(uint16_t out, uint8_t* flags);` follows D5's six steps. A 200 ms timeout gives
  `HP_STEP_TIMEOUT` and sets SO high. There is no error bit, so the flags are otherwise 0.
- `main()`: with boot mode `HP_BOOT_NORMAL`, set `RCNT = 0` and `SIOCNT = 0x5008`, and loop as today's loop does,
  except that a pending command runs `hp_execute` at once (status Busy, then Ready) with no `HP_BUSY` transfer. Any
  other boot mode runs today's loop unchanged.

- [ ] **Step 1: Implement it.** `HP_POLL_IF` stays the end-of-transfer test in both loops.
- [ ] **Step 2: Build and install.** Run `make -C gba/helper && make -C gba/helper size && make -C gba/helper install && make -C gba/helper check`.
  Expected: the image is at most 8192 bytes and `check` passes.
- [ ] **Step 3: Run `make -C tests`** (`test_helper_image` checks the new image) and the release build (it packs the
  asset). Expected: pass, 0 warnings.
- [ ] **Step 4: Commit** `gba/helper/gba/main.c files/gen3_helper.mb` with
  `helper: GBA Normal-mode slave transport when the BIOS booted it in Normal mode`.

### Task 7: Reference, checklists, README and changelog

**Files:**
- Modify: `docs/gen3/multiboot-and-save.md`, `docs/gen3/hardware-test-3.md`, `docs/gen3/hardware-test-1.md`, `docs/gen3/hardware-test-2.md`, `docs/gen3/hardware-test-frlg.md`, `README.md`, `changelog.md`

- [ ] **Step 1: `multiboot-and-save.md`.**
  - Add `# 7. Normal mode over a Game Boy Color cable`:
    - 7.1 the cable (a GBC cable crosses SO and SI, carries SC, has no SD; a module with its SD line open can't run
      MultiPlay at all), with the 2026-10-30 probes;
    - 7.2 multiboot in Normal mode (the Reference facts table and the vector);
    - 7.3 the Normal wire (D2, D3);
    - 7.4 the helper's Normal transport (D5);
    - 7.5 the wire choice at Connect (D6).
  - In §1, add one line pointing to §7.
  - Add M23–M27 to "Uncertain":
    - M23: the data reply is the next word's offset;
    - M24: SO goes high at transfer end (forced anyway);
    - M25: boot mode `0xC4` = 2 in Normal mode;
    - M26: the BIOS and helper sample our 256 kHz bit-banged clock reliably;
    - M27: the BIOS signals ready with SO low in multiboot.
- [ ] **Step 2: `hardware-test-3.md`.**
  - Add a note at the top: *"Cable: with a Game Link module whose SD line is connected (KBEmbedded: SD jumper bridged),
    use the GBA link cable, small plug in the Flipper. Otherwise use a Game Boy Color link cable, either end. The log's
    `wire` line says which link Connect used."*
  - Rewrite A.5 for D6: a reversed or missing cable shows `Waiting for GBA`, then the power-on hint, and the log says
    `wire normal`.
  - Add `## N. Game Boy Color cable`:
    1. Connect as in A.3 with the GBC cable. The log has `wire normal` and `mb done`.
    2. Run B.1, C.1, D.1–D.4 and D.6 over it.
    3. Time the `Backing up save` step.
  - Add rows to the Results table.
- [ ] **Step 3: `hardware-test-1.md`, `-2.md`, `-frlg.md`.** In "You need", add: the Game Link module's SD line must be
  connected (KBEmbedded module: the two SD jumper pads bridged). Without it Ruby never answers, and the log shows
  `R:FFFF` with a timeout on every frame.
- [ ] **Step 4: `README.md`.** In the Gen III section, explain:
  - trading needs a GBA link cable and an SD-connected module;
  - Cartridge also works with a GBC link cable on a module without SD.

  **`changelog.md` 3.1:** add `- Cartridge also works over a Game Boy Color link cable (GBA Normal mode), for Game Link
  modules whose SD line isn't connected`.
- [ ] **Step 5: Commit** the seven files with `docs: Gen III cartridge over a GBC cable (Normal mode); the SD jumper for trades`.

---

## After the last task

- Run `make -C tests`, `make -C gba/helper check`, the dev build and then the release build. Report the two
  `text + data` sizes against `BASE`.
- Push `gen3-save-access`. Then install both apps on the user's Flipper (`ufbt launch APPID=pokemon_cart`, then
  `ufbt launch APPID=pokemon`).
- The user runs `docs/gen3/hardware-test-3.md` with the GBC cable, starting with section N.1. If multiboot fails, the
  log's `mb failed … want … got …` line says which step and word (M23–M27).
