# Gen III Direct Save Access Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** With a genuine Ruby/Sapphire or FireRed/LeafGreen cartridge in a GBA held at Start+Select, Gen III → Cartridge
sends a helper program over the link cable, backs the save up to the SD card and verifies it, and lets the user browse
the party and all 14 boxes, edit any Pokemon in the plan 2 editor, put Pokemon into empty slots, and write the result
back, verified. Restore from a backup and copy the ROM are also there. Emerald is read-only. Writes go only to carts the
helper positively identifies.

**Architecture:** The plan 1 driver gains a free-running stream mode (one transfer per 1073 µs tick, 932 transfers/s,
and idle ticks with no transfer). Pure-C state machines run in its frame callback: first a BIOS multiboot sender, then
a packet pump that speaks the helper protocol. A pure client, save-layout module and job module sit on top of the pump
and do backup, verify, the safe write and restore through an abstract SD store. They are host-tested against the real
helper's portable C core, compiled for the host on a fake GBA (ROM, flash-chip and SRAM models). The helper is C plus a
small ARM crt0, built with the ufbt toolchain. Its image is committed to `files/`. A worker thread runs the jobs, and
thin scenes drive them.

**Tech Stack:** C11. Flipper SDK via `ufbt` (release SDK 1.5.1, API 88.2 per plan 1). STM32 LL (TIM2). For the helper:
`arm-none-eabi-gcc` 12.3.1 from ufbt (`-mcpu=arm7tdmi`), GNU make, and Python 3 standard library (header fix only).
Host tests use `make` and host `cc`. CI is GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-10-25-gen3-gba-design.md` (Phase 2; Goals → Phase 2; Anti-goals; calls #5, #10).
Technical reference, cited as §n: `docs/gen3/multiboot-and-save.md`. Its open questions are M1–M22.

## Plan sequence

This is plan 5 of 5.

1. Link connection test (done).
2. Gen III data core and editor (done before this plan).
3. Cable Club trade with Ruby (done before this plan).
4. FireRed/LeafGreen trades, v3.0 (done before this plan).
5. **This plan:** direct save access, v3.1.

**Branch:** before Task 1, check out `gen3-gba-design` and check that plans 2–4 are in it (`src/gen3/gen3_edit.h`,
`src/gen3/gen3_pk3_furi.h` and `src/scenes/pokemon_gen3_edit.c` exist, and `application.fam` says `fap_version=[3,0]`).
Then run `git switch -c gen3-save-access`. Record `BASE=$(git rev-parse HEAD)` in your report; the size and diff checks
use it. The PR targets `gen3-gba-design`, stacked on it.

## Global Constraints

- Commits are authored by the user (LawlessNPC, Aleksandr Kofman) with **no** AI-attribution trailers: no `Co-Authored-By`, no `Claude-Session`, no "Generated with". After every commit, `git log -1 --format='%(trailers)'` must print nothing.
- Stage only the files your task lists, by explicit path. Never `git add -A`, `git add .`, a directory, or `git commit -a`.
- Both SDK builds must finish with 0 warnings: `ufbt` (release) and `UFBT_HOME=/Users/kyvo/.claude/jobs/72354b36/tmp/ufbt-dev ufbt` (dev). `make -C tests` passes after every task.
- **Pure-module rule:** every file in `src/gen3/` whose name does not end in `_furi.c`/`_furi.h` includes only `<stdint.h> <stdbool.h> <stddef.h> <string.h> <stdio.h>` and other pure `src/gen3/` headers.
- **Helper portable rule:** `gba/helper/src/*.c` and `gba/helper/include/*.h` include only `<stdint.h> <stdbool.h> <stddef.h> <string.h>` and each other. Only `gba/helper/gba/*` and `crt0.S` touch GBA registers. Every helper symbol starts with `hp_`/`Hp`/`HP_`.
- **ISR rule** (from plans 1 and 2): the driver frame callback runs in interrupt context, and so does everything it calls (`gen3_mb_frame`, `gen3_hlink_frame`). No `malloc`/`free`, no `FURI_LOG_*`, no furi calls, and it returns in under 200 µs.
- **Frame mode unchanged:** `mlink_hw_start` and every frame-mode code path in `mlink_hw_furi.c` behave exactly as before (Task 1 adds a mode, nothing else).
- **MIT, no copied code.** pret, GBATEK, afska/gba-link-connection, FIX94's dumper and PKHeX are references for behaviour, constants and layouts only. The 156-byte boot logo in `fix_header.py` is interop data (Task 3).
- Gen I/II stay untouched (the plan 2 list of files). Never modify `lib/flipper-gblink`. Western games only.
- **Save safety (spec anti-goals, call #10):**
  - Never write without a verified SD backup of the cart's current state.
  - Never write the slot the game loads.
  - Never touch sectors 28–31.
  - Never chip-erase.
  - Write only carts with game codes `AXVE AXPE BPRE BPGE`, a flash ID of `0x09C2` or `0x1362`, and `FLASH1M_V` with no other save library string. Emerald and non-English carts are read-only.
  - The helper enforces every rule on its own; the Flipper checks them first.
- **RAM:**
  - A 128 KB save never sits in RAM; work in 4 KB sectors.
  - The Cartridge session allocates one buffer of `max(image size, 4096)` bytes (≤ 8192), on its first Connect. Multiboot holds the helper image in it; after that it is the 4 KB sector buffer. It is kept until the session ends, so peak heap stays one buffer. The worker stack is 2048 bytes.
  - No scene or view keeps a local buffer over 128 bytes (the app stack is 2 KB).
  - `text + data` of `dist/debug/pokemon_d.elf` may grow by at most 24,000 bytes over `BASE` (FAPs load into RAM whole). Measure with `$(ls -d ~/.ufbt/toolchain/*/bin | head -1)/arm-none-eabi-size dist/debug/pokemon_d.elf`.
- **Helper image:** `files/gen3_helper.mb` is built only by `make -C gba/helper install`; never edit it by hand. Any change under `gba/helper/` rebuilds and recommits it in the same commit, and `make -C gba/helper check` must pass. The image is ≤ 8192 bytes.
- Use plan 2's names exactly, as declared in its headers: `Gen3Pkm`, `GEN3_BOX_SIZE`, `GEN3_PARTY_SIZE`, `gen3_pkm_get/set/encrypt/decrypt/fix_checksum`, `Gen3Field*`, `gen3_recalc_party`, `gen3_level_for_exp`, `gen3_species`, `gen3_species_name`, `gen3_national_from_internal`, `Gen3Rng`, `gen3_rng_seed/next`, `gen3_is_shiny`, `Gen3Editor`, `gen3_edit_init/load/set_ot_name/set_ot_id`, `gen3_pkm_check`, `gen3_check_text`, `gen3_pk3_dir`, `gen3_pk3_load_file`, `pokemon_fap->gen3_editor`, `PokemonSceneGen3Edit`.
- Includes use the repo's root-relative style (`#include <src/gen3/gen3_save.h>`). Follow `.clang-format`. Use snake_case functions and CamelCase typedefs. Write multi-byte fields byte by byte, little-endian, with no struct-packing tricks.
- Shared registration points change by appending only: `src/scenes/include/pokemon_scene_config.h` (new lines at the end), `src/include/pokemon_app.h`, `src/pokemon_app.c`, `src/scenes/pokemon_gen3_menu.c`.
- `fap_version` becomes `[3,1]` in Task 15 only.
- Implementers can't run on-device checks. UI tasks write their steps into `docs/gen3/hardware-test-3.md`, and the user runs it at the gate (Task 15).

## Review Focus

1. **The link drops, or the GBA loses power, in the middle of a write.** A cable knocked loose, the Flipper's battery, or the GBA's AA batteries can all cause this. At every moment the game must load either the old save or the complete new one. Retry must finish the job without erasing a slot that is already finished. *(Task 5: `test_hp_dedupe_write_resend`; Task 7: `test_hcli_write_resend_not_reexecuted`; Task 10: `test_write_link_drop_sweep`, `test_write_retry_after_commit`, `test_write_power_cut_sweep`; Task 13 checklist D.6.)*
2. **A reproduction or unknown save chip.** This covers SRAM and batteryless repros, 64 KB chips, no save chip at all, and ROMs patched to save to SRAM. No erase, program or bank-switch command may reach such a chip, not even for reading, and writes stay off. *(Task 4: `test_hp_probe_sram_repro`, `test_hp_probe_no_id_change`, `test_hp_probe_64k`; Task 5: `test_hp_read_refused_unless_flash128`, `test_hp_arm_refused_carts`; Task 8: `test_cart_identify_refusals`; Task 10: `test_write_refused_no_flash_ops`.)*
3. **A corrupt or unusual save slot.** One slot is damaged, both are, the cart was never saved, or a slot's counters disagree. The Flipper must pick the slot the game loads, write only the other one, and refuse whatever it can't prove. *(Task 8: `test_slots_*`; Task 10: `test_write_over_corrupt_inactive`, `test_write_refused_not_strict`, `test_restore_onto_corrupt_save`, `test_restore_onto_empty_cart`.)*
4. **The SD card fills up or fails during a backup, or the backup reads back differently.** Nothing is written to the cartridge without a verified backup of its current state. No partial backup is ever left looking like a real one. *(Task 9: `test_job_backup_sd_full_precheck`, `test_job_backup_sd_full_midway`, `test_job_backup_readback_mismatch`, `test_job_backup_flash_unstable`; Task 10: `test_write_requires_backup`, `test_post_write_backup_sd_full`.)*
5. **The wrong game or region.** Examples: a Japanese or German cart, Emerald, or a FireRed backup restored onto Ruby. Never parse a save with the wrong layout, and never write a layout we haven't tested. *(Task 5: `test_hp_arm_refused_carts`; Task 8: `test_cart_identify_codes`, `test_cart_identify_regions`, `test_save_layout_mismatch`; Task 9: `test_job_backup_japanese`; Task 10: `test_write_refused_emerald`, `test_restore_wrong_game_backup`.)*

## Design decisions

**D1. Stream mode with idle ticks.**
- Multiboot and the helper protocol run in a new `MlinkModeStream`. There is no frame-rate gate: a frame starts on the tick after the previous frame's callback.
- `MlinkTx.count` may be 0 in stream mode only. That is an idle tick: no transfer, and the callback runs again on the next tick.
- Idle ticks give multiboot its required silent waits (≥ 1/16 s before the length word, §2.2 step 9) and let the link go quiet when no request is pending.
- Pure state machines take `elapsed_us` per frame and do every timeout in time, never in tries. A frame takes `max(count, 1) × 1073 µs`.
- Frame mode keeps its clamp of 1..9.

**D2. The helper is portable C with a HAL.**
- `gba/helper/src/` holds the protocol, the commands, the cart checks and the flash command sequences. It is plain C on top of `hp_hal.h`.
- `gba/helper/gba/` implements the HAL with GBA registers. `tests/fake_gba.c` implements it with a ROM model and a JEDEC flash-chip model, and `tests/fake_helper.c` wraps it as the word-level partner for the Flipper's tests.
- The Flipper client is therefore tested against the helper's real refusal and arming rules, not a parallel model.
- Flash and hash routines are marked `HP_IWRAM` (`.iwram` section, ARM state on the GBA, nothing on the host).

**D3. Protocol additions to §6.**
- **`H_RECV = 0xB0C0`:** H sends it while it is receiving a packet. P sends a request only after a valid `H_READY`.
- **Early packet check:** H checks `code` and `n` at word w2 against the command's fixed length (table in Task 5). On a mismatch it resumes hunting, so a false `SOF` swallows at most two words.
- **Error responses:** a non-OK status response always carries `n = 2` (fail code, fail offset).
- **Duplicates (replaces §6.5's seq rule):** H remembers the CRC-32 and the result of the last packet it executed. A byte-identical packet that follows it is answered from memory, without executing again, when that command is write-class. Any other valid packet replaces the memory, so a seq that wraps can never alias.
- **`ARM_WRITE` gains a `mode` word:** 0 keeps slot 1−T (§6.3). 1 is allowed only when neither slot is valid, for restoring onto an empty or corrupt save.
- **`ARM_WRITE` also refuses** when slot T is the slot the game loads (§5.5 over both slots).
- **`ROM_CRC32`** is limited to 0x40000 bytes per call (its `H_BUSY` stays under 6 s).
- **Not implemented:** `ROM_SHA1`.

**D4. The write path is §5.8 with three additions.**
- **Fast copies:** unchanged sections use `COPY_SECTOR`. A party edit uploads one sector.
- **Interrupted writes:** a write interrupted after its last commit is recognised as already written, so Retry doesn't redo it. That is the case where slot T is active with counter N+1 and its CRCs match.
- **Post-write backup:** every verified write ends with a new backup of the cart's new state. It is composed from the previous backup plus the 14 sectors just written, then verified against the cart with `SECTOR_CRC32` for all 32 sectors and an SD read-back. So the hub always holds a verified backup of what is on the cart.

**D5. Working copy on SD.**
- Edits go into `apps_data/pokemon/gen3/cart_work.bin`: the active slot's 14 sectors in id order (57,344 bytes), rebuilt from the backup.
- RAM keeps only a 14-bit dirty mask per section and a 426-bit changed-Pokemon map (6 party + 420 box slots).

**D6. Restore = the safe write path with a backup file as the source.**
- On a cart with a valid save, the backup's active slot goes into the cart's inactive slot with counter N+1. The cart's current save stays as the fallback, and the composed backup (D4) already holds it.
- On an empty or corrupt cart (no valid slot), arming uses mode 1 with the backup's own counter and rotation.
- Sectors 28–31 are never restored. The UI says so.

**D7. Backups.**
- **Format:** raw 131,072-byte `.sav` files, all 32 sectors, which load in emulators and PKHeX.
- **Name and place:** `APP_DATA_PATH("gen3/backups/<code>_<tid>_<stamp>.sav")`, where `<code>` is the 4-char game code, `<tid>` is `%05u` or `nosave`, and `<stamp>` is `YYYYMMDD-HHMMSS` from the RTC. Example: `AXVE_12345_20261025-143005.sav`. Written as `.tmp` and renamed only after it verifies.
- **Verified means three things:** the packet CRC on every sector, the helper re-reads every sector (`SECTOR_CRC32`) and the CRC matches, and the SD read-back CRC matches too.

**D8. ROM copy.**
- **Name:** `APP_DATA_PATH("gen3/roms/<code>_v<version>.gba")`.
- **In 256 KiB regions:** each region is received in 4 KB `ROM_READ`s, checked against the helper's `ROM_CRC32` of that region and against an SD read-back, and only then kept. A bad region is re-read once.
- **Resume:** the `.part` file grows only by verified regions, so a cancelled or failed copy resumes at its length.

**D9. Region and game policy (§3.2).**
- **Japanese (`xxxJ`):** backup and ROM copy only; no browse (M6).
- **`D/F/I/S` and Emerald:** browse only.
- **Writes:** `AXVE AXPE BPRE BPGE` only.
- **How the game is told:** from the cart header, never the save. The section-0 checksum with the right size cross-checks the layout (§5.6).

**D10. The editor is shared.**
- Entering Cartridge saves a copy of `*pokemon_fap->gen3_editor`. Leaving puts it back.
- A box or party Pokemon is decrypted into the editor (`gen3_edit_load`) and opened with `PokemonSceneGen3Edit`. On return its bytes are compared, and only a real change is staged.
- New Pokemon get the save's trainer: the raw OT name bytes, TID/SID, OT gender, and the met game of the cart (§5.6 trainer table).

## File map

| File | Responsibility |
|---|---|
| `src/gen3/mlink_types.h`, `mlink_hw_furi.{h,c}` | modify: `MlinkMode`, stream mode, tick constants |
| `src/gen3/gen3_mb.{h,c}` | multiboot encryption, CRC, image check, sender state machine |
| `gba/helper/Makefile`, `helper.ld`, `crt0.S`, `fix_header.py` | helper build; header and logo fix |
| `gba/helper/include/hp_*.h`, `gba/helper/src/hp_*.c` | portable helper: link, core/commands, cart, flash, save checks |
| `gba/helper/gba/hp_hal_gba.c`, `gba/helper/gba/main.c` | GBA registers, link loop |
| `files/gen3_helper.mb` | committed helper image (shipped to `apps_assets/pokemon/`) |
| `src/gen3/gen3_hlink.{h,c}` | Flipper packet framing, CRC-32, ISR pump |
| `src/gen3/gen3_hcli.{h,c}` | helper commands, resends, response parsing |
| `src/gen3/gen3_save.{h,c}` | Gen III save layout: footers, checksums, slots, spans, write plan, Pokemon summary |
| `src/gen3/gen3_cart_id.{h,c}` | cart identification and write policy; new-Pokemon defaults |
| `src/gen3/gen3_cart.{h,c}` | jobs: connect, backup, verify, working copy, write, restore, ROM copy |
| `src/gen3/gen3_store_furi.{h,c}` | `Gen3Store` over the SD card |
| `src/gen3/gen3_cart_furi.{h,c}` | session: driver, ISR dispatch, worker thread, transport, log |
| `src/views/gen3_cart_progress.{h,c}` | progress view |
| `src/scenes/include/pokemon_gen3_cart.h`, `src/scenes/pokemon_gen3_cart*.c` | Cartridge scenes |
| `tests/fake_bios.*`, `fake_gba.*`, `fake_helper.*`, `fake_save.*`, `fake_store.*`, `test_*.c` | host fakes and tests |
| `docs/gen3/hardware-test-3.md`, `README.md`, `changelog.md`, `application.fam`, `.gitignore`, `tests/Makefile` | docs, version, build |

---

### Task 1: Driver stream mode

**Files:**
- Modify: `src/gen3/mlink_types.h`, `src/gen3/mlink_hw_furi.h`, `src/gen3/mlink_hw_furi.c`

**Interfaces:**
- Produces (`mlink_types.h`, appended; existing lines unchanged except `MlinkTx.count`'s comment):
  ```c
  typedef enum { MlinkModeFrame, MlinkModeStream } MlinkMode;
  #define MLINK_TICK_US 1073u   /* one tick, start to start (mlink_hw_furi.c MLINK_TICK_DECI_US / 10) */
  #define MLINK_FRAME_US 16740u /* frame mode: frame start to frame start */
  /* Time one frame takes, for protocol timeouts. Stream: max(count, 1) ticks. Frame: MLINK_FRAME_US. */
  static inline uint32_t mlink_frame_us(MlinkMode mode, uint8_t count);
  /* MlinkTx.count: frame mode 1..9; stream mode 0..9, 0 = idle tick (no transfer). */
  ```
- Produces (`mlink_hw_furi.h`):
  ```c
  /* mlink_hw_start with a mode. MlinkModeFrame is exactly mlink_hw_start. MlinkModeStream: a frame starts on the
   * first tick after start and on every tick after a frame's callback (no frame-rate gate); transfer t on tick t;
   * count 0 makes an idle tick whose callback gets rx.count 0, masks 0. The cable guard still reads SI at every
   * frame start (there are no idle ticks to latch on). Same failure rules as mlink_hw_start. */
  bool mlink_hw_start_mode(MlinkHw* hw, MlinkMode mode, MlinkFrameCallback cb, void* ctx, const MlinkTx* first);
  ```

Implementation constraints:
- `mlink_hw_start` becomes a one-line call of `mlink_hw_start_mode(hw, MlinkModeFrame, …)`.
- `struct MlinkHw` gains `MlinkMode mode`.
- `mlink_load_tx` clamps to 1..9 in frame mode and to 0..9 in stream mode.
- The ISR's accumulator block runs only in frame mode. In stream mode, the `!in_frame` path starts a frame at once, running the same cable-guard code.
  - **count 0:** it calls `mlink_frame_done` without moving any line.
  - **Otherwise:** it falls through to the existing transfer code.
- `frame_acc` starts at 0 in both modes.
- Add `_Static_assert(MLINK_TICK_DECI_US == MLINK_TICK_US * 10u)` and the same for `MLINK_FRAME_DECI_US`/`MLINK_FRAME_US`.
- Update the header comment's description of frames. Nothing else in the file changes.

- [ ] **Step 1: Record the baseline.** Run `ufbt`, then `arm-none-eabi-size dist/debug/pokemon_d.elf` (path in Global Constraints). Report `text + data` as `SIZE_BASE`.
- [ ] **Step 2: Implement** as specified.
- [ ] **Step 3: Build** with both SDKs. Expected: 0 warnings.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS (no behaviour change for existing tests).
- [ ] **Step 5: Self-review the diff.** `git diff -- src/gen3/mlink_hw_furi.c` shows only:
  - the mode field;
  - the start split;
  - the clamp parameter;
  - the mode guard around the accumulator and the count-0 branch;
  - the asserts;
  - comments.

  The on-device regression check (Link Test still works) is in Task 15, section R.
- [ ] **Step 6: Commit** `gen3: mlink driver: add a free-running stream mode with idle ticks`.

---

### Task 2: Multiboot sender (pure) and fake BIOS

**Files:**
- Create: `src/gen3/gen3_mb.h`, `src/gen3/gen3_mb.c`, `tests/fake_bios.h`, `tests/fake_bios.c`
- Test: `tests/test_gen3_mb.c`

**Interfaces:**
- Consumes: `MlinkTx`, `MlinkRx`, `MLINK_NO_DATA` (`mlink_types.h`); `Gen3Rng`, `gen3_rng_seed`, `gen3_rng_next` (plan 2 `gen3_rules.h`).
- Produces:
  ```c
  #define GEN3_MB_CLIENT_BIT 0x02u
  #define GEN3_MB_PALETTE 0x93u
  #define GEN3_MB_HEADER_LEN 0xC0u
  #define GEN3_MB_MIN_LEN 0x1C0u
  #define GEN3_MB_MAX_LEN 0x2000u          /* our helper cap; the BIOS allows 0x40000 */
  #define GEN3_MB_MAX_RESTARTS 3u          /* then Failed: ask for a power cycle (§2.6, M4) */
  #define GEN3_MB_RESTART_MIN_US 70000u    /* restart wait = 70 ms + rng % 170001 us (§2.4) */
  #define GEN3_MB_JUNK_LIMIT 16u           /* detection replies that are neither 0x0000, 0xFFFF nor 0x72x2 */
  #define GEN3_MB_PALETTE_US 268000u       /* 16 tries at the GBA frame rate (§2.2 step 7), as time */
  #define GEN3_MB_LENGTH_DELAY_US 100000u  /* silent wait before the length word (§2.2 step 9, M20) */
  #define GEN3_MB_FINAL_US 5000000u        /* waiting for 0x0075 (§2.2 step 12) */
  uint32_t gen3_mb_crc_step(uint32_t crc, uint32_t data);   /* 32 rounds, poly 0xA517 (§2.3) */
  uint8_t gen3_mb_hh(uint8_t cc);                           /* (0x11 + cc + 0xFF + 0xFF) & 0xFF */
  typedef struct { uint32_t seed; uint32_t crc; } Gen3MbCrypt;
  void gen3_mb_crypt_init(Gen3MbCrypt* k, uint8_t pp, uint8_t cc);         /* seed pp|cc<<8|0xFFFF0000, crc 0xFFF8 */
  uint32_t gen3_mb_crypt_word(Gen3MbCrypt* k, uint32_t ptr, uint32_t plain); /* CRC, seed step, returns e (§2.3) */
  uint16_t gen3_mb_crypt_final(const Gen3MbCrypt* k, uint8_t hh, uint8_t rr); /* crc_step(crc, hh|rr<<8|0xFFFF0000) */
  typedef enum { Gen3MbImageOk, Gen3MbImageBadLen, Gen3MbImageBadHeader, Gen3MbImageBadVersion } Gen3MbImageCheck;
  /* len in [MIN, MAX] and % 16 == 0; le32(img+4) == 0x51AEFF24; img[0xB2] == 0x96; img[0xBD] == the §2.1
   * complement of 0xA0..0xBC; img[0xBC] == version. */
  Gen3MbImageCheck gen3_mb_image_check(const uint8_t* img, uint32_t len, uint8_t version);
  typedef enum { Gen3MbPhaseWait, Gen3MbPhaseDetect, Gen3MbPhaseHello, Gen3MbPhaseHeader, Gen3MbPhasePalette,
                 Gen3MbPhaseHandshake, Gen3MbPhaseDelay, Gen3MbPhaseLength, Gen3MbPhaseData, Gen3MbPhaseFinal,
                 Gen3MbPhaseCrc, Gen3MbPhaseDone, Gen3MbPhaseFailed } Gen3MbPhase;
  typedef enum { Gen3MbFailNone, Gen3MbFailHello, Gen3MbFailHeader, Gen3MbFailPalette, Gen3MbFailHandshake,
                 Gen3MbFailLength, Gen3MbFailData, Gen3MbFailFinal, Gen3MbFailCrc } Gen3MbFail;
  typedef struct {
      Gen3MbPhase phase; Gen3MbFail last_fail;
      uint8_t restarts;      /* counted failures after detection succeeded */
      uint32_t sent, total;  /* payload bytes acknowledged / to send */
      uint32_t waiting_us;   /* time in Detect without a 0x72x2 reply (UI hint after 3 s) */
  } Gen3MbStatus;
  typedef struct Gen3Mb Gen3Mb; /* full struct in the header; holds a pointer to the caller's image, no heap */
  void gen3_mb_init(Gen3Mb* mb, const uint8_t* img, uint32_t len, uint32_t seed); /* image must stay valid until Done/Failed */
  /* One call per driver frame (ISR). rx: result of the previous tx (NULL on the first call). elapsed_us: that frame's time. */
  void gen3_mb_frame(Gen3Mb* mb, const MlinkRx* rx, MlinkTx* tx, uint32_t elapsed_us);
  void gen3_mb_status(const Gen3Mb* mb, Gen3MbStatus* out); /* phase is a single field: safe to poll from a thread */
  ```

State machine. It follows §2.2 and §2.4. Each expected reply is checked against the same transfer's rx word.
- **Invalid words:** a word with its `timeout_mask` or `framing_mask` bit set is invalid.
  - In a polling phase it is ignored.
  - Anywhere else it fails the phase.
- **Counted failure:** `restarts++`, record `last_fail`, then Wait. Once `restarts > GEN3_MB_MAX_RESTARTS` the machine goes to Failed instead. Failed and Done send idle ticks forever.

| Phase | tx | Leaves when |
|---|---|---|
| Wait | count 0 | the wait elapsed → Detect (first call starts in Detect) |
| Detect | 1 word `0x6200` | valid `(r & 0xFFF0) == 0x7200 && (r & 0xF) == 2` → Hello. `GEN3_MB_JUNK_LIMIT` other valid replies (not 0x0000/0xFFFF) → Wait (not counted) |
| Hello | 1 word `0x6102` | reply `0x7202` → Header, else counted failure |
| Header | open-loop, 9 per frame: 96 header halfwords, then `0x6200`, `0x6202` | replies `((0x60-k)<<8)\|2` for k = 0..95, then `0x0002`, `0x7202`; any mismatch → counted |
| Palette | 1 word `0x6393` | valid `(r & 0xFF00) == 0x7300` → cc = r & 0xFF → Handshake; `GEN3_MB_PALETTE_US` without → counted |
| Handshake | 1 word `0x6400 \| gen3_mb_hh(cc)` | `(r & 0xFF00) == 0x7300` → Delay, else counted |
| Delay | count 0 | `GEN3_MB_LENGTH_DELAY_US` elapsed → Length |
| Length | 1 word `(len - 0x190) / 4` | `(r & 0xFF00) == 0x7300` → rr = r & 0xFF, crypt init → Data, else counted |
| Data | open-loop, 9 per frame: per 32-bit word at `ptr`, `e & 0xFFFF` then `e >> 16` | replies `ptr & 0xFFFF`, `(ptr+2) & 0xFFFF`; mismatch → counted |
| Final | 1 word `0x0065` | valid `0x0075` → Crc; `GEN3_MB_FINAL_US` without → counted |
| Crc | open-loop: `0x0066`, then `crypt_final(hh, rr)` | replies `0x0075`, then the CRC itself → Done; else counted |

`tests/fake_bios.{h,c}` (test only) models the BIOS slave as child 1, at the word level, with the hardware's one-transfer
reply delay: each transfer replies with the current SEND word, then handles P's word.
- **Fidelity:** every reply value comes from §2.2. The fake implements the decryption and the CRC on its own, written from §2.3, not by calling `gen3_mb.c`.
- **Bad words:** any word that breaks the sequence sets `fake_bios_error` and sends the fake back to detection (SEND `0x0000`), which models M4.
```c
typedef enum { FakeBiosOff, FakeBiosWaiting, FakeBiosBooted } FakeBiosPhase;
typedef struct FakeBios FakeBios;  /* full struct in the header */
void fake_bios_init(FakeBios* b, uint8_t cc, uint8_t rr);      /* starts Off: every transfer times out */
void fake_bios_power_on(FakeBios* b);                           /* Start+Select: 2 transfers time out, then SEND 0x0000 */
void fake_bios_set_busy(FakeBios* b, uint16_t palette, uint16_t final); /* replies 0x7202 / 0x0074 that many times */
void fake_bios_corrupt_reply(FakeBios* b, uint32_t transfer);  /* xor 0x0100 into the reply of that transfer (from power-on) */
void fake_bios_force_bad_crc(FakeBios* b, bool on);            /* its own CRC is off by one: in its reply and its check */
void fake_bios_frame(FakeBios* b, const MlinkTx* tx, MlinkRx* rx); /* count 0: no transfer */
FakeBiosPhase fake_bios_phase(const FakeBios* b);
const uint8_t* fake_bios_image(const FakeBios* b, uint32_t* len); /* header + decrypted payload it received */
uint32_t fake_bios_boots(const FakeBios* b);
const char* fake_bios_error(const FakeBios* b);                 /* last protocol violation, or NULL */
```

- [ ] **Step 1: Write the failing tests.**
  - **Loop helper:** a test-local `run(mb, bios, max_frames)` alternates `gen3_mb_frame` and `fake_bios_frame`, passing `elapsed = max(tx.count,1) * MLINK_TICK_US`, and records every tx. It stops at Done or Failed. `max_frames` is 100,000 unless a test says otherwise. A glitch leaves the fake mid-sequence, and it only resyncs after its own violation check, so the glitch tests need the room.
  - **Image builder:** a test-local `make_image(buf, len)` writes a valid image:
    - bytes 0x04..0x07 = `24 FF AE 51`, and the rest of 0x04..0x9F zero;
    - `[0xB2] = 0x96`, `[0xBC] = 1`, and the complement;
    - payload bytes from an LCG.

```c
TEST(test_mb_crc_step)      /* crc_step(0xFFF8, 0) == 0x0749; crc_step(0xFFF8, 0xFFFFFFFF) == 0x8723 */
TEST(test_mb_doc_vector)    /* §2.3 vector: header 0xC0 zeros, payload 00..FF, len 0x1C0, pp 0x93, cc 0x5A, rr 0xC3:
                               hh 0x69; (len-0x190)/4 == 0x000C; seed after init 0xFFFF5A93;
                               halfwords 0..7 == 3525 4645 0D28 FD10 CF51 4E63 C51C 2615;
                               124..127 == DAF9 5C4D 4DC4 A30A; final CRC 0x2A28 */
TEST(test_mb_image_check)   /* make_image 0x1C0 -> Ok; len 0x1B0, 0x1C8, 0x2010 -> BadLen; [0xB2]=0 -> BadHeader;
                               [0xBD]+1 -> BadHeader; [4]^=1 -> BadHeader; version 2 -> BadVersion */
TEST(test_mb_boot)          /* image 0x1000 B, cc 0x5A rr 0xC3, power_on, run <= 20000 frames: phase Done, restarts 0,
                               fake Booted, boots 1, fake_bios_image == image (all 0x1000 bytes), every tx.count <= 9 */
TEST(test_mb_waits_for_gba) /* 4000 frames Off: phase Detect, restarts 0, waiting_us >= 3000000; then power_on -> Done */
TEST(test_mb_length_delay)  /* between the transfer carrying 0x64hh and the one carrying the length word: only count-0
                               frames, totalling >= 100000 us */
TEST(test_mb_busy_slave)    /* set_busy(40, 200) -> Done, restarts 0 */
TEST(test_mb_header_glitch) /* corrupt the reply of the 50th transfer after power-on (a header transfer):
                               restarts 1, last_fail Header, then Done, boots 1 */
TEST(test_mb_data_glitch)   /* corrupt the reply of the 1000th transfer after power-on (data phase of a 0x1000 image):
                               restarts 1, last_fail Data, then Done */
TEST(test_mb_bad_crc_gives_up) /* force_bad_crc: 4 counted failures -> phase Failed, restarts 4, last_fail Crc, boots 0 */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL (missing `gen3_mb.h`).
- [ ] **Step 3: Implement** `gen3_mb.{h,c}` and the fake.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS, and the whole `test_gen3_mb` binary runs in under 2 s.
- [ ] **Step 5: Commit** `gen3: add the GBA multiboot sender`.

---

### Task 3: GBA helper: build, header, HAL, packet layer, HELLO

**Files:**
- Create in `gba/helper/`:
  - `Makefile`, `helper.ld`, `crt0.S`, `fix_header.py`;
  - `include/hp_version.h`, `include/hp_hal.h`, `include/hp_link.h`, `include/hp_core.h`;
  - `src/hp_link.c`, `src/hp_core.c`;
  - `gba/hp_hal_gba.c`, `gba/main.c`.
- Create: `tests/fake_gba.h`, `tests/fake_gba.c` (own header, timer and status only in this task), `tests/test_hp_link.c`
- Modify:
  - `application.fam`: `sources=["*.c*", "!tests", "!gba"]`.
  - `.gitignore`: add `gba/helper/build/`.
  - `tests/Makefile`:
    - `HELPER_SRCS = $(wildcard ../gba/helper/src/*.c)` is added to `SRCS`;
    - `CFLAGS += -I../gba/helper/include`;
    - the `$(BUILD)/%` prerequisites gain `$(wildcard ../gba/helper/include/*.h)`.

**Interfaces:**
- Produces:
  ```c
  /* hp_version.h */
  #define HP_PROTO_VERSION 1   /* also the header's version byte 0xBC (crt0.S includes this file) */
  #define HP_BUILD_ID 1        /* bump whenever files/gen3_helper.mb changes */
  /* hp_hal.h: implemented by gba/hp_hal_gba.c (GBA) and tests/fake_gba.c (host) */
  #ifdef HP_GBA
  #define HP_IWRAM __attribute__((section(".iwram"), target("arm"), noinline))
  #else
  #define HP_IWRAM
  #endif
  typedef enum { HpStatusReady, HpStatusBusy, HpStatusWriting, HpStatusRefused } HpStatus; /* green, yellow, red, blue (§4.8 rule 4) */
  uint16_t hp_hal_rom16(uint32_t off);           /* cart ROM at 0x08000000 + off; off even, < 0x02000000 */
  uint32_t hp_hal_rom32(uint32_t off);           /* off % 4 == 0 */
  uint8_t hp_hal_flash_read(uint16_t off);       /* 0x0E000000 + off, 8-bit access only */
  void hp_hal_flash_write(uint16_t off, uint8_t v);
  void hp_hal_timer_start(void);                 /* TM2/TM3 cascade (§4.7) */
  uint16_t hp_hal_timer_units(void);             /* 1.0224 ms units since hp_hal_timer_start */
  void hp_hal_id_delay(void);                    /* pret DELAY(): 20000-iteration volatile loop */
  const uint8_t* hp_hal_own_header(void);        /* the helper's 0xC0-byte header; [0xC4]/[0xC5] = BIOS boot mode / slave id */
  void hp_hal_status(HpStatus s);
  /* hp_link.h */
  #define HP_SOF 0x55AAu
  #define HP_READY 0xB0A0u
  #define HP_BUSY 0xB0B0u
  #define HP_RECV 0xB0C0u
  #define HP_MAX_PAYLOAD 2064u
  uint32_t hp_crc32_word(uint32_t reg, uint16_t w);                  /* low byte, then high byte */
  uint32_t hp_crc32_bytes(uint32_t reg, const uint8_t* p, uint32_t n); /* reflected 0xEDB88320; caller inits 0xFFFFFFFF, finishes ~reg */
  /* hp_core.h */
  #define HP_STEP_ERR 1u      /* SIOCNT error bit set for this word */
  #define HP_STEP_TIMEOUT 2u  /* no transfer for 200 ms (TM3 >= 196): `in` is not a word */
  typedef struct Hp Hp;       /* full struct in the header: rx state, payload buffer, response, 4 KB sector buffer, caches */
  void hp_init(Hp* hp);
  uint16_t hp_step(Hp* hp, uint16_t in, uint8_t flags); /* P's word of the transfer just ended -> H's word for the next */
  bool hp_exec_pending(const Hp* hp);
  uint16_t hp_execute(Hp* hp);  /* runs the pending command; returns the response's first word (HP_SOF) */
  ```
  Status codes `HP_ST_OK … HP_ST_NOT_ERASED` = §6.4 values 0x00–0x09, and command codes `HP_CMD_*` = §6.3 values, also go in `hp_core.h`.

Core link behaviour (§6.1, §6.2, D3):
- `hp_step` returns `HP_READY` while hunting and `HP_RECV` from the word after `SOF` to the packet's last word.
- After a valid packet, `hp_exec_pending` becomes true, and the caller must call `hp_execute` before the next `hp_step`.
- While a response is going out, `hp_step` ignores `in` and returns its words: `SOF`, `seq<<8|status`, `n`, payload, CRC low, CRC high. Then it returns `HP_READY`.
- **Packet checks at w2:** the code must be known, and `n` must equal the command's fixed request length. Otherwise the packet is dropped and hunting resumes at the next word.
- **Bad CRC, error word, or timeout:** the packet is dropped silently.
- **HELLO:** reply `[HP_PROTO_VERSION, HP_BUILD_ID, HP_MAX_PAYLOAD, own_header[0xC4] | own_header[0xC5] << 8]`. In later tasks it also disarms.
- **Fixed request lengths:**

  | Command | n |
  |---|---|
  | HELLO, CART_INFO, SAVE_PROBE, DISARM, READ_FOOTERS | 0 |
  | READ_SECTOR, SECTOR_CRC32, ERASE, COMMIT | 1 |
  | ROM_READ | 3 |
  | ROM_CRC32 | 4 |
  | COPY_SECTOR | 5 |
  | ARM_WRITE | 6 |
  | WRITE_SECTOR | 2050 |

  Codes other than HELLO answer `HP_ST_UNSUPPORTED` until their task adds them.

GBA side (§4.3–§4.8):
- **crt0.S:** the §4.3 crt0, with header title `FLIPPERGEN3 `, code `ZFGE`, maker `00`, `0xB2 = 0x96` and `0xBC = HP_PROTO_VERSION`.
- **helper.ld:** the §4.4 linker script.
- **Start-up (`main.c`):**
  - IME 0, WAITCNT `0x4317` (SRAM field 3 = 8 cycles; `#define HP_WAITCNT` tunable, fall back to `0x0003`);
  - RCNT 0, SIOCNT `0x6003`, IE `0x0080`, DISPCNT 0;
  - status green, SEND = `HP_READY`.
- **Main loop (`main.c`):**
  1. `in = xfer(out, &flags)`. This writes SEND and waits for the transfer end with the TM3 200 ms timeout. `#define HP_POLL_IF 1` polls `IF` bit 7 and acknowledges it; 0 polls `SIOCNT` bit 7 going 1→0 instead (M16). Then it reads `SIOMULTI0` and the error bit (`SIOCNT` bit 6).
  2. `out = hp_step(...)`.
  3. If exec is pending:
     - SEND = `HP_BUSY` and status yellow;
     - `out = hp_execute(...)`;
     - acknowledge the stale `IF` and wait for one fresh one (§4.8 rule 2);
     - status green.
- `hp_hal_gba.c` implements the HAL with the registers in §4.7/§4.8. The flash accessors are `HP_IWRAM`.

Build (`gba/helper/Makefile`):
- **Toolchain:** `CROSS ?= $(HOME)/.ufbt/toolchain/current/bin/arm-none-eabi-`.
- **Flags:** §4.2, plus `-DHP_GBA -Iinclude`.
- **Libraries:** `LIBS = -lc_nano -lgcc`.
- **Targets:**
  - `all` → `build/helper.mb`;
  - `size` prints `size build/helper.elf` and the `.mb` byte count, and fails over 8192;
  - `install` copies the image to `../../files/gen3_helper.mb`;
  - `check` builds and `cmp`s it against the committed file;
  - `clean`.

`fix_header.py` (`python3 -I fix_header.py in.bin out.mb`) runs §4.5 steps 1–4. It also fails unless all of these hold:
- the 156 logo bytes have CRC-32 `0xD0BEB55E`;
- `[0xB2] == 0x96`;
- the top byte of `le32([0xC0])` is `0xEA`;
- the output is ≤ 8192 bytes.

Logo, as interop data with a comment saying so (it is the fixed bitmap every GBA cartridge carries and the BIOS checks;
transcribed here, not code from any project):

```
24FFAE51699AA2213D84820A84E409AD11248B98C0817F21A352BE199309CE2010464A4AF82731EC58C7E83382E3CEBF85F4DF94
CE4B09C194568AC01372A7FC9F844D73A3CA9A615897A327FC039876231DC7610304AE56BF38840040A70EFDFF52FE036F9530F1
97FBC08560D68025A963BE03014E38E2F9A234FFBB3E0344780090CB88113A9465C07C6387F03CAFD625E48B380AAC7221D4F807
```

`tests/fake_gba.{h,c}`, this task's part:
```c
typedef struct FakeGba FakeGba;  /* full struct in the header; extended by Tasks 4-5 */
void fake_gba_init(FakeGba* g);  /* own header: 0xC0 bytes with a test logo (any 156-byte pattern), [0xC4]=3, [0xC5]=1 */
FakeGba* fake_gba_current(void); /* the HAL functions act on the most recently initialised FakeGba */
```

- [ ] **Step 1: Write the failing tests** in `tests/test_hp_link.c`. A test-local `feed(hp, words, n)` steps the core word by word.
  - Request words for HELLO seq 1: `55AA 0101 0000 D24E 983A` (CRC-32 `0x983AD24E`).

```c
TEST(test_hp_crc32_check)    /* ~hp_crc32_bytes(0xFFFFFFFF, "123456789", 9) == 0xCBF43926;
                                ~(hp_crc32_word(hp_crc32_word(0xFFFFFFFF, 0x0101), 0x0000)) == 0x983AD24E */
TEST(test_hp_idle_ready)     /* 10 idle words -> each hp_step returns 0xB0A0; never pending */
TEST(test_hp_recv_word)      /* after 0x55AA, steps return 0xB0C0 until the packet's last word */
TEST(test_hp_hello)          /* feed HELLO -> pending after the last word; hp_execute == 0x55AA; then steps return
                                0100 0004 0001 0001 0810 0103 0F9C B666, then B0A0 (HP_BUILD_ID 1; CRC 0xB6660F9C) */
TEST(test_hp_bad_crc_dropped)/* HELLO with CRC low ^1 -> never pending, B0A0 after; then a good HELLO is served */
TEST(test_hp_bad_length)     /* 55AA 0101 0001 ... -> dropped at w2: the next return is B0A0 */
TEST(test_hp_unknown_code)   /* code 0x7F -> dropped at w2 */
TEST(test_hp_rx_timeout)     /* 55AA 0101, then HP_STEP_TIMEOUT -> B0A0; a full HELLO after is served */
TEST(test_hp_err_word)       /* a word with HP_STEP_ERR inside a packet -> dropped */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement** the files above.
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Build the helper.** Run `make -C gba/helper` and `make -C gba/helper size`. Expected: 0 warnings, `build/helper.mb` ≤ 8192 bytes. `xxd -s 0xA0 -l 0x20 build/helper.mb` shows `FLIPPERGEN3 ZFGE00`, `96` at 0xB2, and `01` at 0xBC. Report the size.
- [ ] **Step 6: Build** with both SDKs. Expected: 0 warnings, and no `gba/` path in the `CC` lines.
- [ ] **Step 7: Commit** `gen3: add the GBA helper skeleton (build, header, packet layer, HELLO)`. Don't commit `files/gen3_helper.mb` yet.

---

### Task 4: Helper: cart identification, save probe, flash routines

**Files:**
- Create: `gba/helper/include/hp_cart.h`, `gba/helper/src/hp_cart.c`, `gba/helper/include/hp_flash.h`, `gba/helper/src/hp_flash.c`
- Modify: `gba/helper/src/hp_core.c` (CART_INFO, SAVE_PROBE), `tests/fake_gba.{h,c}` (ROM, flash chip, SRAM models)
- Test: `tests/test_hp_cart.c`, `tests/test_hp_flash.c`

**Interfaces:**
- Produces:
  ```c
  typedef enum { HpLibNone, HpLibFlash1M, HpLibFlash, HpLibFlash512, HpLibSram, HpLibEeprom } HpLib; /* wire values 0..5 */
  typedef struct {
      bool logo_ok, fixed_ok, complement_ok; /* logo == own header logo; [0xB2] == 0x96; §2.1 complement */
      uint32_t rom_size;    /* §3.7; 0 if rom32(4) != 0x51AEFF24 */
      HpLib lib; uint32_t lib_off;  /* first FLASH1M_V hit, else the first other hit */
      bool other_lib;       /* any non-FLASH1M library string found */
      bool write_game;      /* game code AXVE, AXPE, BPRE or BPGE */
  } HpCart;
  void hp_cart_scan(HpCart* out);  /* header + whole-ROM scan, HP_IWRAM inner loop */
  typedef enum { HpChipFlash128, HpChipFlash64, HpChipSram, HpChipNone } HpChip; /* SAVE_PROBE class 0..3 */
  typedef struct { uint16_t id; HpChip cls; uint8_t pre0, pre1, id0, id1; bool sram; } HpProbe;
  void hp_flash_probe(HpProbe* out, const uint8_t code[4], uint8_t version);
  typedef struct { uint8_t status; uint16_t code; uint16_t off; } HpFlashResult; /* HP_ST_*, pret fail code, offset */
  HpFlashResult hp_flash_erase(uint8_t sector);
  HpFlashResult hp_flash_program(uint8_t sector, const uint8_t img[4096], bool defer_commit);
  HpFlashResult hp_flash_commit(uint8_t sector);
  void hp_flash_read(uint8_t sector, uint16_t off, uint8_t* out, uint16_t len);
  uint32_t hp_flash_crc32(uint8_t sector);   /* CRC-32 (IEEE) of the 4096 bytes */
  ```

Behaviour:
- **Library scan** (§3.3 step 1, D9): 32-bit aligned steps over `[0, rom_size)`. These patterns are matched; the first is FLASH1M, all others set `other_lib`:

  | Kind | Bytes |
  |---|---|
  | FLASH1M | `FLASH1M_V` |
  | Flash | `FLASH_V` |
  | Flash512 | `FLASH512_V` |
  | Sram | `SRAM_V` or `SRAM_F_V` |
  | Eeprom | `EEPROM_V` |

  The scan reads through `hp_hal_rom32`/`rom16`, and never writes in the ROM area.
- **ROM size:** §3.7 (open-bus halfwords `(addr/2) & 0xFFFF` at a power of two from 1 MiB up, or a mirror of offset 0); 32 MiB if none.
- **Probe:** §3.3 steps 2–3 and §3.5 exactly, in this order:
  1. SRAM probe. If positive, `sram = true` and class Sram, and no flash command is ever issued.
  2. Read `pre0, pre1`.
  3. The ID sequence, with `hp_hal_id_delay` before and after reading.

  Classification:

  | Class | When |
  |---|---|
  | Flash128 | `0x09C2` or `0x1362` with the ID bytes ≠ `pre`; or `0xD4BF` when code `AXPD` and version 0 |
  | Flash64 | the other §3.4 IDs (`0xD4BF`, `0x1CC2`, `0x1B32`, `0x3D1F`) with changed bytes |
  | None | anything else |
- **Erase, program, commit, read:** copy §4.8's primitives and §3.4's rules.
  - **Bank:** select the bank before every access (M21); sector n is bank `n >> 4` at window `(n & 15) << 12`.
  - **Polling:** `wait()` with DQ5 and timer timeouts, and `0xF0` on failure.
  - **Erase:** up to 5 tries.
  - **Program:** erase first if any byte ≠ 0xFF; program ascending, skipping 0xFF8 if deferred; verify; up to 3 tries.
  - **Commit:** program `0x25` at 0xFF8 only if it reads `0xFF` and 0xFF9..0xFFB read `20 01 08`. If 0xFF8 is already programmed: `NOT_ERASED`. If the signature bytes are wrong: `BAD_ARG`. Then verify that 0xFF8..0xFFB read `0x08012025` (else `VERIFY_FAIL`).
  - **Status colour:** red during erase, program and commit.
  - Never chip-erase.
  - **Placement:** every function in `hp_flash.c` is `HP_IWRAM`, and so are the scan and CRC inner loops. Check after the build with `arm-none-eabi-nm build/helper.elf`: their addresses are in `0x03000000–0x03005FFF`.
- **CART_INFO:** runs `hp_cart_scan` and caches it. The response (n = 102) is:
  - 96 header words;
  - flags (bit 0 logo_ok, 1 fixed_ok, 2 complement_ok, 3 other_lib, 4 write_game);
  - rom_size (2 words);
  - lib (1 word);
  - lib_off (2 words).
- **SAVE_PROBE:** runs once and caches. The response (n = 5) is `[id, cls, pre0|pre1<<8, id0|id1<<8, sram]`. It reads the code and version from the header bytes `rom16(0xAC..0xBC)`.

`tests/fake_gba` additions:
```c
typedef enum { FakeSaveFlash, FakeSaveSram, FakeSaveNone, FakeSaveRomLike } FakeSaveKind; /* RomLike: ID mode changes nothing */
void fake_gba_rom(FakeGba* g, const uint8_t header[0xC0], uint32_t rom_size);  /* sparse: header, then a pattern */
void fake_gba_rom_string(FakeGba* g, uint32_t off, const char* s);             /* place a library string */
void fake_gba_save(FakeGba* g, FakeSaveKind kind, uint16_t id);                /* chip id for Flash */
uint8_t* fake_gba_flash(FakeGba* g);        /* 131072 bytes (Flash) or 65536 (Sram) */
void fake_gba_fail_erase(FakeGba* g, uint8_t sector, uint8_t times);
void fake_gba_fail_program(FakeGba* g, uint8_t sector, uint16_t off, uint8_t times);
void fake_gba_power_cut_after(FakeGba* g, uint32_t ops); /* freeze the chip after that many erases + byte programs */
typedef struct { uint32_t erases, programs, commits, bank_selects, chip_erases, sram_writes, id_entries; } FakeGbaCounts;
void fake_gba_counts(const FakeGba* g, FakeGbaCounts* out);
uint32_t fake_gba_take_busy_us(FakeGba* g); /* erase 300000, program 25/byte, scan 1500000, crc 1/byte; resets to 0 */
```
Fake ROM details:
- Pattern halfword `((off * 31) >> 3) & 0xFFFF`.
- Open-bus `(off/2) & 0xFFFF` past `rom_size`.

Fake flash chip (a JEDEC state machine, M8):
- `AA@5555, 55@2AAA, cmd@5555` with cmd 0x90 (ID), 0xF0, 0x80 (then `AA, 55, 30@sector` or `10@5555`), 0xA0 (program: `new = old & v`) and 0xB0 (bank, then `bank@0000`).
- A write of 0xF0 anywhere returns it to read mode. Any other write outside a sequence is ignored.
- Erase and program complete at once.
- An injected failure leaves the bytes unchanged. Status reads then return the old value with bit 5 (DQ5) set, so the helper's `wait()` reports `phase | 0xA000`.
- `hp_hal_timer_units()` = flash reads since `hp_hal_timer_start` / 8.
- SRAM is a plain 64 KB array.

- [ ] **Step 1: Write the failing tests.**

```c
/* test_hp_flash.c */
TEST(test_hp_probe_mx)        /* id 0x09C2, flash[0..1] = 12 34: cls Flash128, id 0x09C2, pre 12/34, id bytes C2/09,
                                 sram false; chip back in read mode (hp_hal_flash_read(0) == 0x12); chip_erases 0 */
TEST(test_hp_probe_sanyo)     /* 0x1362 -> Flash128 */
TEST(test_hp_probe_sram_repro)/* Sram: cls Sram, sram true; SRAM byte 0x5555 restored; the other 65535 bytes unchanged;
                                 id_entries 0, bank_selects 0 */
TEST(test_hp_probe_no_id_change) /* RomLike with id bytes equal to the array: cls None */
TEST(test_hp_probe_64k)       /* 0x1CC2 -> Flash64; 0xD4BF + AXVE -> Flash64; 0xD4BF + AXPD v0 -> Flash128; AXPD v1 -> Flash64 */
TEST(test_hp_probe_open_bus)  /* None: cls None, sram false */
TEST(test_hp_erase)           /* sector 20 all 0x00 -> erase OK -> 4096 x 0xFF; other 31 sectors unchanged; bank 1 selected */
TEST(test_hp_program_defer)   /* blank sector 3, image i*7 with bytes 0xFF8..0xFFB = 25 20 01 08: program(defer) OK ->
                                 equal except [0xFF8] == 0xFF; commit OK -> [0xFF8] == 0x25 */
TEST(test_hp_commit_rules)    /* 0xFF8 already 0x25 -> NOT_ERASED; 0xFF9..0xFFB = 00 00 00 -> BAD_ARG; no byte programmed */
TEST(test_hp_program_erases_first) /* non-blank sector, program(no defer) -> equals image; erases 1 */
TEST(test_hp_erase_retry)     /* fail_erase(5, 2) -> OK on try 3; fail_erase(5, 5) -> ERASE_FAIL, code 0xA002, last write F0@5555 */
TEST(test_hp_program_fail)    /* fail_program(7, 0x123, 99) -> PROGRAM_FAIL, code 0xA001, off 0x123 */
TEST(test_hp_crc32_sector)    /* 0xFF sector -> 0xF154670A; 0x00 sector -> 0xC71C0011 */
TEST(test_hp_bank_explicit)   /* chip left in bank 1, read sector 0 -> bank 0 data (M21) */
/* test_hp_cart.c */
TEST(test_hp_cart_ruby)       /* header POKEMON RUBY / AXVE / 01, logo = own logo, 16 MiB, FLASH1M_V103 at 0x3B0000:
                                 logo/fixed/complement ok, rom_size 0x1000000, lib Flash1M, lib_off 0x3B0000, !other_lib, write_game */
TEST(test_hp_cart_patched)    /* plus SRAM_V113 at 0x500000 -> other_lib */
TEST(test_hp_cart_eeprom_only)/* only EEPROM_V124 -> lib Eeprom, other_lib */
TEST(test_hp_cart_size_8m)    /* rom_size 0x800000 -> 0x800000 */
TEST(test_hp_cart_bad_logo)   /* logo byte 0x10 flipped -> logo_ok false; rom_size still 0x1000000 */
TEST(test_hp_cart_codes)      /* AXPE, BPRE, BPGE write_game; BPEE, AXVJ, AXVD, AXPD not */
TEST(test_hp_cart_info_cmd)   /* CART_INFO through hp_step/hp_execute: OK, n 102, header words == header, flags 0x17 */
TEST(test_hp_save_probe_cmd)  /* SAVE_PROBE twice: same payload {0x09C2, 0, ...}; id_entries stays 1 */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Build the helper** with `make -C gba/helper size`. Expected: 0 warnings, ≤ 8192. Report the size.
- [ ] **Step 6: Commit** `gen3: helper: identify the cart, probe the save chip, flash routines`.

---

### Task 5: Helper: save commands, arming, duplicates; commit the image

**Files:**
- Create: `gba/helper/include/hp_save.h`, `gba/helper/src/hp_save.c`, `tests/fake_save.h`, `tests/fake_save.c`, `files/gen3_helper.mb` (via `make install`)
- Modify: `gba/helper/src/hp_core.c`
- Test: `tests/test_hp_cmd.c`, `tests/test_helper_image.c`

**Interfaces:**
- Produces (`hp_save.h`):
  ```c
  /* Slot s (0/1) holds 14 sectors with signature 0x08012025, full-length checksum ok (§5.3 over 0xF80 bytes),
   * ids 0..13 once each, one counter. Writes it to *counter. */
  bool hp_save_slot_ok(uint8_t slot, uint32_t* counter);
  /* §5.5 over both slots with full-length checksums: 0/1 = the slot the game loads, -1 = none valid. */
  int hp_save_active_slot(void);
  ```
- Command table:

  | Code | Args (words) | OK response (n) | Class |
  |---|---|---|---|
  | `0x04` ARM_WRITE | magic `0x5752`, flash id, T, mode, N lo, N hi | 0 | write |
  | `0x05` DISARM | — | 0 | — |
  | `0x10` READ_SECTOR | sector | 2048 | flash read |
  | `0x11` READ_FOOTERS | — | 256: per sector `id, checksum, sig lo, sig hi, ctr lo, ctr hi, full checksum, 0` | flash read |
  | `0x12` SECTOR_CRC32 | sector | 2 | flash read |
  | `0x18` ERASE_SECTOR | sector | 0 | write |
  | `0x19` WRITE_SECTOR | sector, flags (bit 0 DEFER_COMMIT), 2048 image words | 0 | write |
  | `0x1A` COPY_SECTOR | src, dst, flags, ctr lo, ctr hi | 0 | write |
  | `0x1B` COMMIT_SECTOR | sector | 0 | write |
  | `0x20` ROM_READ | addr lo, addr hi, words (1..2048) | words | — |
  | `0x22` ROM_CRC32 | addr lo, addr hi, len lo, len hi (2..0x40000, even) | 2 | — |

  Non-OK responses carry `[fail code, fail offset]` (n = 2).

Rules (§6.3, §3.6, D3):
1. **Flash reads** answer `UNSUPPORTED` unless SAVE_PROBE ran and classed Flash128 with `sram` false. This keeps bank commands away from SRAM.
2. **Write-class commands** answer `REFUSED` unless all of these hold:
   - CART_INFO ran, with logo, fixed byte and complement ok, `write_game`, lib Flash1M and no other_lib;
   - SAVE_PROBE gave Flash128.
3. **ARM_WRITE:**
   - magic wrong → `BAD_ARG`;
   - flash id ≠ the probed id → `REFUSED`;
   - mode 0: `hp_save_slot_ok(1-T, &n)` with `n == N`, `N % 2 == 1-T` and `hp_save_active_slot() == 1-T`, else `REFUSED`;
   - mode 1: `hp_save_active_slot() == -1` and neither slot ok, else `REFUSED`;
   - mode > 1 or T > 1 → `BAD_ARG`;
   - OK arms sectors `14T .. 14T+13`.
4. **ERASE, WRITE, COPY, COMMIT:**
   - sector ≥ 28 or outside slot T → `REFUSED`; not armed → `NOT_ARMED`.
   - COPY's src must be in slot 1−T, else `BAD_ARG`. COPY reads src into the sector buffer, patches 0xFFC..0xFFF with the new counter, and then works like WRITE.
   - WRITE checks the packet CRC before any flash access (that is automatic: no execution without a valid packet).
5. **Disarm:** on DISARM, HELLO, any flash error, or once all 14 sectors of T have been committed.
6. **ROM_READ:** the address must be in `0x08000000..0x09FFFFFF` with the whole range inside, else `BAD_ARG`. **ROM_CRC32:** the same range rule, and len over 0x40000 → `BAD_ARG`.
7. **Duplicates (D3):** after executing a packet, remember its CRC-32 and its response. A byte-identical next packet whose command is write-class is answered from memory without touching flash. Any other valid packet clears the memory.

`tests/fake_save.{h,c}`. It builds synthetic 128 KB flash images. It has its own copy of the §5.4 size tables and §5.3
checksum, written from the doc and independent of `gen3_save.c`.
```c
typedef enum { FakeGameRS, FakeGameFRLG, FakeGameE } FakeGame;
typedef struct {
    FakeGame game;
    bool slot_present[2]; uint32_t counter[2]; uint8_t first[2]; /* physical index of id 0 */
    uint32_t seed;            /* data bytes within size[id] from an LCG; past size[id] zero */
} FakeSaveSpec;
void fake_save_build(uint8_t img[131072], const FakeSaveSpec* spec); /* absent slots and sectors 28..31 all 0xFF */
uint32_t fake_save_phys(const FakeSaveSpec* spec, uint8_t slot, uint8_t id); /* sector number 0..27 */
void fake_save_fix(uint8_t img[131072], uint8_t sector, FakeGame game);     /* recompute that sector's checksum */
void fake_save_set_counter(uint8_t img[131072], uint8_t sector, uint32_t counter);
void fake_save_set_id(uint8_t img[131072], uint8_t sector, uint16_t id);
void fake_save_break_checksum(uint8_t img[131072], uint8_t sector);
```

- [ ] **Step 1: Write the failing tests.**
  - **Helpers:** a test-local `req(hp, code, seq, args, n, data, resp, &status, &resp_n)` builds a packet with `hp_crc32_word` and runs it through `hp_step`/`hp_execute`.
  - **Default cart:** Ruby AXVE, MX 0x09C2, FLASH1M_V103, and `fake_save_build` with `{RS, A present counter 4 first 0, B absent}`. CART_INFO and SAVE_PROBE are done first unless the test says otherwise.

```c
TEST(test_hp_read_sector)     /* READ_SECTOR 17 -> 2048 words == flash bytes 17*4096.. LE */
TEST(test_hp_read_footers)    /* entry 5 == {id 5, checksum, 0x2025, 0x0801, 4, 0, full checksum (== checksum), 0} */
TEST(test_hp_sector_crc32)    /* == CRC-32 of sector 9 */
TEST(test_hp_read_refused_unless_flash128) /* Sram cart: READ_SECTOR/READ_FOOTERS/SECTOR_CRC32 -> UNSUPPORTED;
                                 SRAM unchanged; bank_selects 0. Same before SAVE_PROBE ran. */
TEST(test_hp_arm_ok)          /* ARM_WRITE(0x09C2, T 1, mode 0, N 4) -> OK */
TEST(test_hp_arm_refused_matrix) /* each REFUSED, and a following ERASE 14 -> NOT_ARMED: T 0; N 5; id 0x1362;
                                 slot A one checksum broken; slot A one counter 3; slot A id 7 replaced by 6;
                                 A 4 and B 5 both valid (B active) with T 1; mode 1 while A valid.
                                 Magic 0x1234 -> BAD_ARG; mode 2 -> BAD_ARG */
TEST(test_hp_arm_refused_carts) /* each: ARM_WRITE REFUSED; ERASE 14 REFUSED; counts erases/programs/commits 0:
                                 BPEE, AXVJ, AXVD, other_lib (SRAM_V string added), chip 0x1CC2, logo bad,
                                 SAVE_PROBE never sent */
TEST(test_hp_write_gating)    /* armed T 1: ERASE 3 REFUSED; ERASE 28 REFUSED; ERASE 14 OK; DISARM; ERASE 15 NOT_ARMED */
TEST(test_hp_write_full_path) /* ERASE 20; WRITE 20 defer image X (valid footer) -> flash 20 == X except [0xFF8] 0xFF;
                                 COMMIT 20 -> == X */
TEST(test_hp_copy_sector)     /* COPY 3 -> 17, counter 0x12345679, defer -> flash 17 == flash 3 except 0xFFC..0xFFF
                                 = 79 56 34 12 and [0xFF8] 0xFF; COMMIT 17 -> [0xFF8] 0x25 */
TEST(test_hp_disarm_after_14_commits) /* ERASE/WRITE/COMMIT all 14 of slot B -> next ERASE 14 NOT_ARMED */
TEST(test_hp_dedupe_write_resend) /* the same ERASE packet twice -> 2nd answered OK, erases 1;
                                 ERASE 15 (different), then the first packet again -> executed, erases 3 */
TEST(test_hp_flash_error_disarms) /* fail_program(21, 0, 99): WRITE 21 -> PROGRAM_FAIL, n 2; ERASE 22 -> NOT_ARMED */
TEST(test_hp_mode1_blank)     /* all 0xFF flash: mode 1 T 0 -> OK; mode 0 -> REFUSED */
TEST(test_hp_rom_read)        /* ROM_READ(0x08000100, 4) == ROM; (0x0A000000, 1) BAD_ARG; (0x09FFFFFE, 2) BAD_ARG */
TEST(test_hp_rom_crc32)       /* (0x08000000, 0x40000) == CRC-32 of those ROM bytes; len 0x40002 -> BAD_ARG */
/* test_helper_image.c (reads ../files/gen3_helper.mb) */
TEST(test_helper_image)       /* file exists; len <= 8192; gen3_mb_image_check(img, len, HP_PROTO_VERSION) == Ok;
                                 img[0xA0..0xAB] == "FLIPPERGEN3 " */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL, with `test_helper_image` failing because the file is missing.
- [ ] **Step 3: Implement** the commands and the fake.
- [ ] **Step 4: Build and install the image.** Run `make -C gba/helper size install check`. Expected: 0 warnings, ≤ 8192 bytes, and `check` reports identical. Report the size; the target is ≤ 6 KB.
- [ ] **Step 5: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 6: Build** with both SDKs. Expected: 0 warnings, and the image is packaged: `strings dist/pokemon.fap | grep -c gen3_helper.mb` is ≥ 1, like `all_sprites.fxbm`.
- [ ] **Step 7: Commit** `gen3: helper: save commands with arming and duplicate safety; add the image`, staging the helper sources, the tests, the fakes and `files/gen3_helper.mb`.

---

### Task 6: Flipper packet pump (pure)

**Files:**
- Create: `src/gen3/gen3_hlink.h`, `src/gen3/gen3_hlink.c`
- Test: `tests/test_gen3_hlink.c`

**Interfaces:**
- Produces:
  ```c
  #define GEN3_HLINK_SOF 0x55AAu
  #define GEN3_HLINK_READY 0xB0A0u
  #define GEN3_HLINK_BUSY 0xB0B0u
  #define GEN3_HLINK_RECV 0xB0C0u
  #define GEN3_HLINK_IDLE 0x0000u
  #define GEN3_HLINK_MAX_PAYLOAD 2064u
  #define GEN3_HLINK_MAX_ARGS 6u
  #define GEN3_HLINK_NO_RESPONSE_XFERS 32u  /* §6.5 */
  #define GEN3_HLINK_NOT_READY_XFERS 64u    /* words that are neither READY nor BUSY before a request */
  #define GEN3_HLINK_BUSY_US 6000000u       /* §6.5 */
  uint32_t gen3_crc32_update(uint32_t reg, const uint8_t* p, size_t n); /* reflected 0xEDB88320, no init/xorout */
  uint32_t gen3_crc32(const uint8_t* p, size_t n);                      /* IEEE: init and xorout 0xFFFFFFFF */
  typedef enum { Gen3HlinkIdle, Gen3HlinkSync, Gen3HlinkSending, Gen3HlinkWaiting, Gen3HlinkReceiving,
                 Gen3HlinkDone, Gen3HlinkFailed } Gen3HlinkState;
  typedef enum { Gen3HlinkFailNone, Gen3HlinkFailNoReady, Gen3HlinkFailNoResponse, Gen3HlinkFailBusyTimeout,
                 Gen3HlinkFailBadCrc, Gen3HlinkFailWire, Gen3HlinkFailSeq, Gen3HlinkFailTooLong } Gen3HlinkFail;
  typedef struct {
      uint8_t code, seq;
      uint16_t args[GEN3_HLINK_MAX_ARGS]; uint8_t nargs;
      const uint8_t* data; uint16_t data_words;    /* payload after args, LE bytes (WRITE_SECTOR's 4 KB image) */
      uint8_t* resp; uint16_t resp_max_words;      /* response payload lands here as LE bytes */
  } Gen3HlinkReq;
  typedef struct Gen3Hlink Gen3Hlink; /* full struct in the header; `state` is volatile */
  void gen3_hlink_init(Gen3Hlink* h);
  /* App side, only while the state is Idle, Done or Failed. The pump owns req's buffers until Done/Failed. */
  void gen3_hlink_submit(Gen3Hlink* h, const Gen3HlinkReq* req);
  void gen3_hlink_sync(Gen3Hlink* h, uint8_t need, uint32_t timeout_us); /* Done after `need` consecutive valid READY */
  /* ISR side: one call per driver frame. rx NULL on the first call. */
  void gen3_hlink_frame(Gen3Hlink* h, const MlinkRx* rx, MlinkTx* tx, uint32_t elapsed_us);
  Gen3HlinkState gen3_hlink_state(const Gen3Hlink* h);
  void gen3_hlink_result(const Gen3Hlink* h, uint8_t* status, uint16_t* resp_words, Gen3HlinkFail* fail);
  ```

Behaviour (§6.1, §6.2, §6.5, D3):
- **Idle, Done, Failed:** `tx.count = 0`.
- **Sync, Sending, Waiting, Receiving:** `tx.count = 9`. Request words come first, then `0x0000`.
- **Valid words:** an rx word is valid when its mask bits are clear. Words from the transfers after the request's last word go, in order, to the response logic, including later words of the same frame.
- **Before a request:**
  - one valid `READY` starts Sending;
  - `BUSY` waits, up to `GEN3_HLINK_BUSY_US`;
  - `GEN3_HLINK_NOT_READY_XFERS` other words (invalid ones count) → `NoReady`.
- **Sync:** `need` consecutive valid `READY`s → Done. Any other word resets the run. `timeout_us` → Failed `NoReady`.
- **Request words:** `SOF`, `seq<<8|code`, `n = nargs + data_words`, the args, the data words, then CRC-32 over w1..w(2+n) as bytes, low half then high half.
- **Waiting:**
  - `BUSY` adds elapsed time; beyond `GEN3_HLINK_BUSY_US` → `BusyTimeout`.
  - `SOF` → Receiving.
  - `GEN3_HLINK_NO_RESPONSE_XFERS` words that are neither (invalid ones count) → `NoResponse`.
- **Receiving:**
  - an invalid word → `Wire`;
  - `n > resp_max_words` → `TooLong`, decided at w2;
  - CRC mismatch → `BadCrc`;
  - echoed seq ≠ the request's → `Seq`, checked after the CRC;
  - else Done with `status = w1 & 0xFF`.
- **ISR budget:** under 200 µs per frame: one CRC update per word, no loops over the payload.

- [ ] **Step 1: Write the failing tests.** Scripted rx words only, with no fake helper. A test-local `frame(h, rx_words, n, masks, elapsed)` runs one frame and returns the tx.

```c
TEST(test_crc32_check)        /* gen3_crc32("123456789") == 0xCBF43926; gen3_crc32({01 01 00 00}) == 0x983AD24E;
                                 4096 x 0xFF -> 0xF154670A */
TEST(test_hlink_idle)         /* after init: tx.count 0 */
TEST(test_hlink_hello_words)  /* submit HELLO seq 1; rx READY -> next frames carry 55AA 0101 0000 D24E 983A then 0000s */
TEST(test_hlink_read_sector_words) /* READ_SECTOR seq 2 arg 5 -> 55AA 0210 0001 0005 6378 0D1F */
TEST(test_hlink_data_payload) /* args {7, 8}, data {01 02 03 04} -> n 4, payload 0007 0008 0201 0403 */
TEST(test_hlink_response_ok)  /* after HELLO: BUSY x3, 55AA 0100 0004 0001 0001 0810 0103 0F9C B666, READY ->
                                 Done, status 0, resp_words 4, resp bytes 01 00 01 00 10 08 03 01 */
TEST(test_hlink_response_same_frame) /* the response starts in the frame that carried the request's last word -> Done */
TEST(test_hlink_bad_crc)      /* last word ^1 -> Failed BadCrc */
TEST(test_hlink_seq_mismatch) /* seq 2 echoed for seq 1 (CRC valid) -> Failed Seq */
TEST(test_hlink_too_long)     /* resp_max 2, n 4 -> Failed TooLong */
TEST(test_hlink_wire_error)   /* a timeout_mask bit on a payload word -> Failed Wire */
TEST(test_hlink_no_response)  /* 33 READY after the request -> Failed NoResponse */
TEST(test_hlink_busy_timeout) /* BUSY for 6.1 s -> BusyTimeout; BUSY for 5.9 s then the response -> Done */
TEST(test_hlink_waits_for_ready) /* RECV x20 then READY: the first SOF goes out only in the frame after the READY */
TEST(test_hlink_no_ready)     /* RECV x65 -> Failed NoReady */
TEST(test_hlink_sync4)        /* READY READY BUSY READY READY READY READY -> Done at the 7th word; a timeout-masked word
                                 resets the run */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add the helper link packet pump`.

---

### Task 7: Helper client and the fake helper

**Files:**
- Create: `src/gen3/gen3_hcli.h`, `src/gen3/gen3_hcli.c`, `tests/fake_helper.h`, `tests/fake_helper.c`
- Test: `tests/test_gen3_hcli.c`

**Interfaces:**
- Consumes: Task 6; for the tests only, the helper core (Tasks 3–5).
- Produces:
  ```c
  #define GEN3_HELPER_PROTO 1u
  #define GEN3_HELPER_ASSET "gen3_helper.mb"   /* APP_ASSETS_PATH name */
  #define GEN3_HCLI_RESENDS 3u
  /* GEN3_HCMD_* (§6.3 codes) and GEN3_HST_* (§6.4) */
  typedef struct {
      void* ctx;
      Gen3HlinkFail (*exchange)(void* ctx, const Gen3HlinkReq* req, uint8_t* status, uint16_t* resp_words);
      bool (*sync)(void* ctx, uint8_t need, uint32_t timeout_us);
  } Gen3HTransport;
  typedef enum { Gen3HcliOk, Gen3HcliStatusError, Gen3HcliLinkLost, Gen3HcliBusyTimeout, Gen3HcliBadResponse } Gen3HcliResult;
  typedef struct {
      Gen3HTransport t; uint8_t seq; uint32_t resends; uint8_t last_status; uint16_t last_fail_code, last_fail_off;
      uint8_t resp[512];  /* raw small responses (READ_FOOTERS, the largest, is 512 B); sector and ROM data go to the caller */
  } Gen3Hcli;
  typedef struct { uint16_t proto, build, max_payload; uint8_t boot_mode, slave_id; } Gen3HHello;
  typedef struct { uint8_t header[0xC0]; bool logo_ok, fixed_ok, complement_ok, other_lib, write_game;
                   uint32_t rom_size; uint8_t lib; uint32_t lib_off; } Gen3HCartInfo;
  typedef struct { uint16_t flash_id; uint8_t cls; uint8_t pre0, pre1, id0, id1; bool sram; } Gen3HProbe;
  typedef struct { uint16_t id, checksum; uint32_t signature, counter; uint16_t full_checksum; } Gen3HFooter;
  void gen3_hcli_init(Gen3Hcli* c, const Gen3HTransport* t);
  Gen3HcliResult gen3_hcli_connect(Gen3Hcli* c, Gen3HHello* out);   /* sync(4, 2 s) + HELLO; BadResponse unless proto matches */
  Gen3HcliResult gen3_hcli_cart_info(Gen3Hcli* c, Gen3HCartInfo* out);
  Gen3HcliResult gen3_hcli_save_probe(Gen3Hcli* c, Gen3HProbe* out);
  Gen3HcliResult gen3_hcli_arm_write(Gen3Hcli* c, uint16_t flash_id, uint8_t target, uint8_t mode, uint32_t counter);
  Gen3HcliResult gen3_hcli_disarm(Gen3Hcli* c);
  Gen3HcliResult gen3_hcli_read_sector(Gen3Hcli* c, uint8_t sector, uint8_t out[4096]);
  Gen3HcliResult gen3_hcli_read_footers(Gen3Hcli* c, Gen3HFooter out[32]);
  Gen3HcliResult gen3_hcli_sector_crc32(Gen3Hcli* c, uint8_t sector, uint32_t* crc);
  Gen3HcliResult gen3_hcli_erase_sector(Gen3Hcli* c, uint8_t sector);
  Gen3HcliResult gen3_hcli_write_sector(Gen3Hcli* c, uint8_t sector, bool defer_commit, const uint8_t img[4096]);
  Gen3HcliResult gen3_hcli_copy_sector(Gen3Hcli* c, uint8_t src, uint8_t dst, bool defer_commit, uint32_t counter);
  Gen3HcliResult gen3_hcli_commit_sector(Gen3Hcli* c, uint8_t sector);
  Gen3HcliResult gen3_hcli_rom_read(Gen3Hcli* c, uint32_t addr, uint16_t words, uint8_t* out);
  Gen3HcliResult gen3_hcli_rom_crc32(Gen3Hcli* c, uint32_t addr, uint32_t len, uint32_t* crc);
  ```

Rules:
- **Sequence numbers:** each call takes the next `seq`. A resend reuses it and the same request, so the helper's duplicate memory applies.
- **Resends:** on `NoReady`, `NoResponse`, `BadCrc`, `Wire` or `Seq`, resend up to `GEN3_HCLI_RESENDS` times (`resends++`), then `LinkLost`.
- **Other failures:** `BusyTimeout` is returned at once. `TooLong`, or an OK response whose `n` isn't the Task 5 table's, → `BadResponse`.
- **Non-OK status:** `StatusError`, with `last_status`, `last_fail_code` and `last_fail_off` set.
- **No link resets:** the client never resets the link on its own. The job layer reconnects (Task 10).

`tests/fake_helper.{h,c}`. It wraps `Hp` over a `FakeGba` as child 1 with the one-transfer reply delay:
- **Each transfer:** the reply is SEND; then P's word goes to `hp_step`.
- **When exec becomes pending:** call `hp_execute` at once (the flash changes now). Then reply `HP_BUSY` for `max(1, fake_gba_take_busy_us(gba) / MLINK_TICK_US)` transfers, ignoring P's words, and then send the response.
```c
typedef struct FakeHelper FakeHelper;  /* full struct in the header */
void fake_helper_init(FakeHelper* f, FakeGba* gba);
void fake_helper_frame(FakeHelper* f, const MlinkTx* tx, MlinkRx* rx, uint32_t elapsed_us); /* 200 ms silent -> HP_STEP_TIMEOUT */
/* Pulled cable: from P's transfer number `transfer` (counted by this fake), `len` transfers never reach the helper.
 * P gets no reply (timeout bit), and the helper is not stepped (time still passes). */
void fake_helper_unplug_at(FakeHelper* f, uint32_t transfer, uint32_t len);
void fake_helper_replug(FakeHelper* f);                                      /* ends any unplug window now */
void fake_helper_corrupt_at(FakeHelper* f, uint32_t transfer);              /* xor 0x0001 into that reply */
void fake_helper_restart(FakeHelper* f); /* GBA power-cycled and helper re-sent: hp_init, caches cleared, flash kept */
uint32_t fake_helper_transfers(const FakeHelper* f);
/* After each hp_execute: the command code and its first argument word. */
void fake_helper_on_command(FakeHelper* f, void (*cb)(void* ctx, uint8_t code, uint16_t arg0), void* ctx);
/* Transport that runs `pump` against this fake with stream timing; gives up after 50,000,000 transfers. */
Gen3HTransport fake_helper_transport(FakeHelper* f, Gen3Hlink* pump);
```
A power cut (`fake_gba_power_cut_after`) makes the fake stop replying for good.

- [ ] **Step 1: Write the failing tests.** The default cart is Task 5's.

```c
TEST(test_hcli_codes_match_helper) /* _Static_assert-style checks: GEN3_HCMD_* == HP_CMD_*, GEN3_HST_* == HP_ST_*,
                                      GEN3_HELPER_PROTO == HP_PROTO_VERSION, GEN3_HLINK_{SOF,READY,BUSY,RECV} == HP_* */
TEST(test_hcli_connect)       /* Ok; proto 1, build HP_BUILD_ID, max 2064, boot 3, slave 1 */
TEST(test_hcli_cart_info)     /* header == the fake header, rom_size 0x1000000, lib 1, write_game */
TEST(test_hcli_probe)         /* flash_id 0x09C2, cls 0 */
TEST(test_hcli_read_sector)   /* == fake flash sector 9 */
TEST(test_hcli_read_footers)  /* entry 3 matches the fake save's sector 3 footer */
TEST(test_hcli_resend_after_corrupt_reply) /* corrupt one response word -> Ok, resends 1 */
TEST(test_hcli_write_resend_not_reexecuted) /* armed; unplug 40 transfers right after the ERASE request's last word ->
                                 Ok, resends >= 1, erases 1 */
TEST(test_hcli_link_lost)     /* unplug 10,000,000 transfers -> LinkLost after 1 + 3 attempts */
TEST(test_hcli_status_error)  /* ERASE without ARM -> StatusError, last_status GEN3_HST_NOT_ARMED */
TEST(test_hcli_rom_read)      /* 2048 words at 0x08001000 == fake ROM */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add the helper client`.

---

### Task 8: Save layout and cart identification (pure)

**Files:**
- Create: `src/gen3/gen3_save.h`, `src/gen3/gen3_save.c`, `src/gen3/gen3_cart_id.h`, `src/gen3/gen3_cart_id.c`
- Modify: `tests/fake_save.{h,c}` (trainer, party and box placement)
- Test: `tests/test_gen3_save.c`, `tests/test_gen3_cart_id.c`

**Interfaces:**
- Consumes: plan 2's `Gen3Pkm`, `gen3_pkm_decrypt/encrypt/get/set/fix_checksum`, `gen3_recalc_party`, `gen3_level_for_exp`, `gen3_species`, `gen3_national_from_internal`, `gen3_pkm_nickname`, `gen3_text_decode`, `gen3_is_shiny`, and `Gen3Editor` with `gen3_edit_init`/`gen3_edit_set_ot_id`; `Gen3HCartInfo`, `Gen3HProbe` (Task 7).
- Produces (`gen3_save.h`):
  ```c
  #define GEN3_SAVE_SECTOR_SIZE 4096u
  #define GEN3_SAVE_SECTOR_DATA 0xF80u
  #define GEN3_SAVE_SECTORS 32u
  #define GEN3_SAVE_SLOT_SECTORS 14u
  #define GEN3_SAVE_SIZE 131072u
  #define GEN3_SAVE_SIGNATURE 0x08012025u
  #define GEN3_SAVE_BOXES 14u
  #define GEN3_SAVE_BOX_SLOTS 30u
  #define GEN3_SAVE_PARTY_MAX 6u
  #define GEN3_SAVE_PARTY 0xFFu      /* "box" value for the party */
  typedef enum { Gen3GameUnknown, Gen3GameRS, Gen3GameFRLG, Gen3GameE } Gen3Game;
  uint16_t gen3_save_section_size(Gen3Game g, uint16_t id);       /* §5.4; 0 if unknown or id > 13 */
  uint16_t gen3_save_checksum(const uint8_t* data, uint16_t size);/* §5.3 */
  typedef struct { uint16_t id, checksum; uint32_t signature, counter; } Gen3SaveFooter;
  void gen3_save_footer_get(const uint8_t sector[4096], Gen3SaveFooter* out);
  void gen3_save_footer_set(uint8_t sector[4096], const Gen3SaveFooter* f);
  typedef struct { Gen3SaveFooter f; bool valid; } Gen3SaveSectorInfo; /* valid: signature ok, id <= 13, checksum ok */
  void gen3_save_sector_info(Gen3Game g, const uint8_t sector[4096], Gen3SaveSectorInfo* out); /* exact size[id] */
  typedef enum { Gen3SlotEmpty, Gen3SlotOk, Gen3SlotError } Gen3SlotStatus;
  typedef enum { Gen3SaveOk, Gen3SaveError, Gen3SaveEmpty, Gen3SaveCorrupt } Gen3SaveResult; /* the game's view */
  typedef struct {
      Gen3SlotStatus slot[2]; uint32_t counter[2];  /* counter of each slot's last valid sector (game rule) */
      Gen3SaveResult result; uint32_t save_counter; uint8_t active; /* save_counter % 2 */
      bool strict_ok;       /* active slot Ok, all 14 counters == save_counter, parity, ids once each (§5.5) */
      uint8_t first;        /* physical index 0..13 of id 0 in the active slot */
      uint8_t phys[14];     /* id -> sector 0..27: the last valid one in physical order, as the game loads */
  } Gen3SaveSlots;
  void gen3_save_analyze(const Gen3SaveSectorInfo info[28], Gen3SaveSlots* out); /* §5.5 exactly */
  typedef struct { uint8_t target; uint32_t counter; uint8_t first; uint8_t dst[14]; uint8_t src[14]; } Gen3SaveWritePlan;
  bool gen3_save_plan_write(const Gen3SaveSlots* s, Gen3SaveWritePlan* out); /* §5.8 step 2; false unless strict_ok */
  void gen3_save_plan_fresh(uint32_t counter, uint8_t first, Gen3SaveWritePlan* out); /* mode 1: target counter%2; src unused */
  typedef struct { uint8_t id; uint16_t off; uint8_t len; } Gen3SaveSpan;
  /* box 0..13 slot 0..29 (80 B in ids 5..13), or box GEN3_SAVE_PARTY slot 0..5 (100 B in id 1). 1 or 2 spans; 0 if invalid. */
  uint8_t gen3_save_mon_spans(Gen3Game g, uint8_t box, uint8_t slot, Gen3SaveSpan out[2]);
  Gen3SaveSpan gen3_save_party_count_span(Gen3Game g);  /* RS/E {1, 0x234, 1}; FRLG {1, 0x034, 1} */
  Gen3SaveSpan gen3_save_box_name_span(uint8_t box);    /* {13, 0x744 + 9*box, 9} */
  typedef struct { char name[8]; uint8_t name_raw[8]; uint8_t gender; uint16_t tid, sid; uint16_t hours; uint8_t minutes; } Gen3SaveTrainer;
  void gen3_save_trainer(const uint8_t sec0[4096], Gen3SaveTrainer* out);  /* §5.6 trainer table */
  typedef enum { Gen3MonEmpty, Gen3MonOk, Gen3MonEgg, Gen3MonBad } Gen3MonKind;
  typedef struct { Gen3MonKind kind; uint16_t national; uint8_t level; char nickname[11]; } Gen3MonSummary;
  /* len 80 or 100, game form. Empty: all bytes 0, or species 0. Bad: checksum fails, BadEgg, or an unknown species.
   * Level from exp and growth. */
  void gen3_save_mon_summary(const uint8_t* mon, size_t len, Gen3MonSummary* out);
  bool gen3_save_mon_to_pkm(const uint8_t* mon, size_t len, Gen3Pkm* out); /* decrypt; len 80: Status 0, Mail 0xFF, recalc */
  void gen3_save_pkm_to_mon(const Gen3Pkm* p, size_t len, uint8_t* out);  /* encrypt, first len bytes */
  ```
- Produces (`gen3_cart_id.h`):
  ```c
  typedef enum { Gen3CartRefuseNone, Gen3CartRefuseHeader, Gen3CartRefuseNotPokemon, Gen3CartRefuseJapanese,
                 Gen3CartRefuseSram, Gen3CartRefuseChip, Gen3CartRefuseLibrary, Gen3CartRefuseRegion,
                 Gen3CartRefuseEmerald, Gen3CartRefuseSave } Gen3CartRefuse; /* checked in this order */
  typedef struct {
      char code[5]; char title[13]; uint8_t version; Gen3Game game; uint8_t met_game; /* 1 S, 2 R, 3 E, 4 FR, 5 LG */
      uint16_t flash_id; bool can_read_save, can_browse, can_write; Gen3CartRefuse why;
  } Gen3CartId;
  void gen3_cart_identify(const Gen3HCartInfo* info, const Gen3HProbe* probe, Gen3CartId* out);
  const char* gen3_cart_refuse_text(Gen3CartRefuse r);
  /* gen3_edit_init(seed), then OT name = t->name_raw (7 bytes copied raw), TID/SID via gen3_edit_set_ot_id,
   * OtGender = t->gender, MetGame = met_game, checksum fixed. */
  void gen3_cart_new_mon(Gen3Editor* ed, const Gen3SaveTrainer* t, uint8_t met_game, uint32_t seed);
  ```

Identification (D9):
- `can_read_save` = header ok (logo, fixed byte, complement), one of the five games (`AXV AXP BPR BPG BPE`), not SRAM, and probe class 0.
- `can_browse` = `can_read_save` and not region `J`.
- `can_write` = `can_browse`, lib Flash1M, no other_lib, region `E`, and not `BPE`.
- `why` is the first failed check in enum order. `Gen3CartRefuseSave` is set later by the job layer (Task 9).
- Texts (exact):

  | Reason | Text |
  |---|---|
  | None | `Writing allowed` |
  | Header | `Bad cartridge header` |
  | NotPokemon | `Not a Gen III Pokemon game` |
  | Japanese | `Japanese saves not supported` |
  | Sram | `SRAM save (reproduction cart)` |
  | Chip | `Unknown save chip` |
  | Library | `ROM patched for another save type` |
  | Region | `Writing: English carts only` |
  | Emerald | `Emerald is read-only for now` |
  | Save | `Save layout not recognised` |

`fake_save` additions:
```c
void fake_save_put_trainer(uint8_t img[131072], const FakeSaveSpec* s, uint8_t slot, const char* name, uint8_t gender, uint16_t tid, uint16_t sid);
void fake_save_put_party(uint8_t img[131072], const FakeSaveSpec* s, uint8_t slot, uint8_t count, const uint8_t mons[][100]);
void fake_save_put_box_mon(uint8_t img[131072], const FakeSaveSpec* s, uint8_t slot, uint8_t box, uint8_t pos, const uint8_t mon[80]);
void fake_save_put_box_name(uint8_t img[131072], const FakeSaveSpec* s, uint8_t slot, uint8_t box, const char* name);
```
Each puts the bytes and refixes the touched checksums. Section 1 party offsets follow §5.6, and boxes follow the §5.6
straddle rule. The builder zeroes the party and box areas before placing data.

- [ ] **Step 1: Write the failing tests.**

```c
/* test_gen3_save.c */
TEST(test_save_checksum)      /* data[i] = i & 0xFF: size 0xF80 -> 0xCA26, 0x890 -> 0x5101, 0xF24 -> 0xFF90; all 0xFF:
                                 0xF80 -> 0xFC1F, 0x7D0 -> 0xFE0B; zeros -> 0; LE words 12345678 9ABCDEF0 0F0F0F0F
                                 FFFFFFFF (16 B) -> 0x0076 */
TEST(test_section_sizes)      /* RS 0 0x890, 4 0xC40, 13 0x7D0; FRLG 0 0xF24, 4 0xEE8; E 0 0xF2C, 4 0xF08;
                                 1-3 and 5-12 0xF80 in all; id 14 -> 0; Unknown -> 0 */
TEST(test_footer_roundtrip)   /* {7, 0xBEEF, 0x08012025, 0x01020304} -> bytes FF4..FFF 07 00 EF BE 25 20 01 08 04 03 02 01 */
TEST(test_slots_both_ok_b_newer) /* A 4 first 0, B 5 first 3: result Ok, save_counter 5, active 1, strict, first 3,
                                 phys[0] 17, phys[13] 16 */
TEST(test_slots_a_newer)      /* A 6, B 5 -> active 0 */
TEST(test_slots_wrap)         /* A 0, B 0xFFFFFFFF -> save_counter 0, active 0, strict */
TEST(test_slots_wrap_odd)     /* A 0xFFFFFFFF, B 0 -> save_counter 0, active 0, strict_ok false */
TEST(test_slots_one_bad)      /* A 4 ok, B 5 with id 6's checksum broken -> slot[1] Error, active 0, result Error, strict */
TEST(test_slots_missing_id)   /* A: id 4's sector relabelled 3 (checksum refixed) -> slot[0] Error */
TEST(test_slots_empty)        /* all 0xFF -> result Empty, save_counter 0 */
TEST(test_slots_corrupt)      /* both slots Error -> Corrupt */
TEST(test_slots_counter_mismatch) /* A all 4 except physical sector 13 = 2 -> counter[0] 2, active 0, strict false */
TEST(test_slots_bad_id)       /* a sector of A with id 20, valid signature -> not valid, slot Error, no crash (ASan clean) */
TEST(test_slots_sig_only)     /* signature ok, checksum wrong -> slot Error, not Empty */
TEST(test_save_layout_mismatch) /* FRLG-built save analysed with RS sizes -> id 0 sectors invalid, strict false;
                                   with FRLG sizes -> strict */
TEST(test_plan_write)         /* A active, first 5, N 4 -> target 1, counter 5, first 6, dst[0] 20, dst[8] 14, dst[13] 19,
                                 src[k] == phys[k] */
TEST(test_plan_write_wrap)    /* B active with 0xFFFFFFFF -> target 0, counter 0 */
TEST(test_plan_refused)       /* strict false -> false */
TEST(test_plan_fresh)         /* counter 7, first 2 -> target 1, dst[0] 16, dst[12] 14 */
TEST(test_mon_spans)          /* (0,0) -> {5,4,80}; (1,19) -> {5,0xF54,44},{6,0,36}; (3,9) -> {6,0xF74,12},{7,0,68};
                                 (13,29) -> {13,0x6F4,80}; the 8 straddlers of §5.6 give 2 spans with first lengths
                                 44 12 60 28 76 44 12 60; RS party 2 -> {1,0x300,100}; FRLG party 0 -> {1,0x38,100};
                                 box 14 -> 0; party 6 -> 0 */
TEST(test_count_and_name_spans) /* party count RS {1,0x234,1}, FRLG {1,0x34,1}; box name 13 -> {13,0x7B9,9} */
TEST(test_trainer)            /* "BRENDAN", gender 0, TID 12345, SID 54321, hours 0x0102, min 3 -> decoded; name_raw ==
                                 the 8 bytes at 0x00 */
TEST(test_mon_summary)        /* GOLDEN_GAME (100) -> Ok, 252, level 5, "TREECKO"; first 80 bytes -> Ok, level 5;
                                 80 zeros -> Empty; golden [0x30]^=1 -> Bad; IsEgg set and re-encrypted -> Egg */
TEST(test_mon_pkm_roundtrip)  /* mon_to_pkm(GOLDEN_GAME, 80) -> raw[0x50..] == GOLDEN_PK3[0x50..]; pkm_to_mon(80) ==
                                 GOLDEN_GAME[0..79]; mon_to_pkm of a broken checksum -> false */
/* test_gen3_cart_id.c */
TEST(test_cart_identify_ruby) /* AXVE, Flash128 0x09C2, Flash1M: read, browse, write, why None, RS, met 2, "POKEMON RUBY" */
TEST(test_cart_identify_codes)/* AXPE RS met 1 write; BPRE FRLG met 4 write; BPGE FRLG met 5 write;
                                 BPEE E met 3: browse, no write, why Emerald */
TEST(test_cart_identify_regions) /* AXVJ: read, no browse, why Japanese; AXVD/AXVF/AXVI/AXVS: browse, no write, why Region */
TEST(test_cart_identify_refusals) /* logo bad -> Header, nothing allowed; "AGBE" -> NotPokemon; sram -> Sram, no read;
                                 cls 1 or 3 -> Chip, no read; lib Eeprom or other_lib -> Library, read+browse, no write */
TEST(test_cart_refuse_text)   /* every reason non-empty; Emerald's text contains "Emerald" */
TEST(test_cart_new_mon)       /* trainer "MAY" raw, 1234/4321, female, met 4: OT bytes 0x14..0x1A == name_raw[0..6],
                                 OtId 0x10E104D2, OtGender 1, MetGame 4, gen3_pkm_check Ok, not shiny */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add the Gen III save layout and cartridge identification`.

---

### Task 9: Cart jobs I: connect, backup, working copy, ROM copy

**Files:**
- Create: `src/gen3/gen3_cart.h`, `src/gen3/gen3_cart.c`, `tests/fake_store.h`, `tests/fake_store.c`
- Modify: `tests/fake_gba.{h,c}`: add `void fake_gba_rom_glitch_once(FakeGba* g, uint32_t off);`, which xors 0x0001 into the halfword at `off` on its next read only.
- Test: `tests/test_gen3_cart.c`

**Interfaces:**
- Consumes: Tasks 7, 8.
- Produces:
  ```c
  #define GEN3_CART_BUF_SIZE 4096u
  #define GEN3_CART_NAME_MAX 48u
  #define GEN3_CART_WORK_NAME "cart_work.bin"
  #define GEN3_CART_BACKUP_DIR "backups"
  #define GEN3_CART_ROM_DIR "roms"
  #define GEN3_CART_ROM_REGION 0x40000u
  #define GEN3_CART_BACKUP_SPACE (131072u + 57344u + 4096u)
  typedef enum { Gen3StoreRead, Gen3StoreCreate, Gen3StoreUpdate } Gen3StoreMode;
  typedef struct {
      void* ctx;   /* names are relative to the gen3 data dir; up to 2 files open at once */
      void* (*open)(void* ctx, const char* name, Gen3StoreMode mode);   /* Create truncates and makes subdirs */
      bool (*read)(void* ctx, void* f, uint32_t off, void* buf, uint32_t len);
      bool (*write)(void* ctx, void* f, uint32_t off, const void* buf, uint32_t len);
      bool (*truncate)(void* ctx, void* f, uint32_t len);
      bool (*close)(void* ctx, void* f);                                /* false: the flush failed */
      bool (*remove)(void* ctx, const char* name);
      bool (*rename)(void* ctx, const char* from, const char* to);
      bool (*size)(void* ctx, const char* name, uint32_t* size);        /* false if absent */
      uint64_t (*free_bytes)(void* ctx);
  } Gen3Store;
  typedef enum { Gen3CartStepNone, Gen3CartStepMbWait, Gen3CartStepMbSend, Gen3CartStepIdentify, Gen3CartStepBackupRead,
                 Gen3CartStepBackupVerify, Gen3CartStepAnalyze, Gen3CartStepPrepare, Gen3CartStepErase,
                 Gen3CartStepProgram, Gen3CartStepCommit, Gen3CartStepVerify, Gen3CartStepPostBackup,
                 Gen3CartStepRestoreCheck, Gen3CartStepRomRead, Gen3CartStepRomVerify } Gen3CartStep;
  typedef struct { void (*cb)(void* ctx, Gen3CartStep step, uint32_t done, uint32_t total); void* ctx; volatile bool* cancel; } Gen3CartProgress;
  typedef enum { Gen3CartOk, Gen3CartWrittenNoBackup, Gen3CartNothing, Gen3CartCancelled, Gen3CartLinkLost,
                 Gen3CartBusyTimeout, Gen3CartHelperError, Gen3CartRefused, Gen3CartChanged, Gen3CartSdFull,
                 Gen3CartSdError, Gen3CartVerifyFailed, Gen3CartFlashError, Gen3CartBadBackup, Gen3CartBadArg,
                 Gen3CartNoHelper, Gen3CartMultibootFailed, Gen3CartLinkBusy } Gen3CartResult;
  const char* gen3_cart_result_text(Gen3CartResult r);
  typedef struct Gen3Cart Gen3Cart; /* full struct in the header: id, header copy, slots, sector infos, sector CRCs,
                                       trainer, backup name, flags, dirty mask, changed map; no heap */
  void gen3_cart_init(Gen3Cart* c, Gen3Hcli* cli, const Gen3Store* store, uint8_t* buf, const Gen3CartProgress* p);
  Gen3CartResult gen3_cart_connect(Gen3Cart* c);   /* gen3_hcli_connect, CART_INFO, SAVE_PROBE, identify */
  /* Keeps the current backup and working copy if this cart's header, flash id and all 28 footers still match them;
   * otherwise runs gen3_cart_backup and sets *dropped = (changes were pending). */
  Gen3CartResult gen3_cart_attach(Gen3Cart* c, const char* stamp, bool* dropped);
  Gen3CartResult gen3_cart_backup(Gen3Cart* c, const char* stamp);
  Gen3CartResult gen3_cart_discard(Gen3Cart* c);   /* rebuild the working copy from the backup */
  const Gen3CartId* gen3_cart_id(const Gen3Cart* c);
  const Gen3SaveSlots* gen3_cart_slots(const Gen3Cart* c);
  bool gen3_cart_trainer(const Gen3Cart* c, Gen3SaveTrainer* out);   /* false: no valid save */
  const char* gen3_cart_backup_name(const Gen3Cart* c);
  bool gen3_cart_backup_current(const Gen3Cart* c);
  bool gen3_cart_browsable(const Gen3Cart* c);     /* can_browse, the save has a slot the game loads, backup current */
  bool gen3_cart_writable(const Gen3Cart* c, Gen3CartRefuse* why); /* can_write, strict_ok, backup current */
  uint8_t gen3_cart_party_count(Gen3Cart* c);
  bool gen3_cart_box_name(Gen3Cart* c, uint8_t box, char out[9]);
  Gen3CartResult gen3_cart_mon_get(Gen3Cart* c, uint8_t box, uint8_t slot, uint8_t out[GEN3_PARTY_SIZE], size_t* len);
  Gen3CartResult gen3_cart_mon_put(Gen3Cart* c, uint8_t box, uint8_t slot, const Gen3Pkm* p); /* BadArg if not browsable,
                                                     out of range, or a party slot > count */
  bool gen3_cart_mon_changed(const Gen3Cart* c, uint8_t box, uint8_t slot);
  uint16_t gen3_cart_changes(const Gen3Cart* c);
  uint32_t gen3_cart_rom_size(const Gen3Cart* c);                            /* CART_INFO rom_size; 0: no ROM copy */
  void gen3_cart_rom_name(const Gen3Cart* c, char out[GEN3_CART_NAME_MAX]);  /* "roms/AXVE_v1.gba" */
  bool gen3_cart_rom_part(Gen3Cart* c, uint32_t* done_bytes);                /* a resumable .part exists */
  Gen3CartResult gen3_cart_rom_dump(Gen3Cart* c, bool resume);
  ```

Stack: the jobs run on the 2048-byte worker stack, and storage calls need about 1 KB of it. So the large working arrays live in `Gen3Cart`, not on the stack: 32 footers, 32 sector CRCs, 28 sector infos and 14 expected CRCs. No job function keeps more than 256 bytes of locals.

Backup (D7), in order. Any failure removes every file this run created.
1. **Precondition:** `can_read_save`, else `Refused`. Then set `backup_current = false`; it becomes true only at step 8.
2. **Space:** `free_bytes() < GEN3_CART_BACKUP_SPACE` → `SdFull` before any link traffic.
3. **Temp file:** create `backups/<code>_<stamp>.tmp`.
4. **Read:** for s = 0..31:
   - `read_sector` into `buf`, then `crc[s] = gen3_crc32(buf)`;
   - write at `s*4096`; a failed write → `SdFull` if `free_bytes() < 4096`, else `SdError`;
   - report progress `(BackupRead, s+1, 32)`;
   - cancel → `Cancelled`.
5. **Second read on the cart:** `sector_crc32(s) == crc[s]` for all s, else `VerifyFailed`.
6. **SD read-back:** close; a failed close → `SdError`. Reopen for reading, and every sector's CRC must equal `crc[s]`, else `VerifyFailed` (`BackupVerify`).
7. **Analyze**, if `can_browse` (`Analyze`):
   - sector infos with `id.game`, then `gen3_save_analyze`;
   - the trainer from the active slot's id 0, when `result` is Ok or Error;
   - `strict_ok` false while `why` is None → `why = Gen3CartRefuseSave`.
8. **Rename** to `backups/<code>_<%05u tid | nosave>_<stamp>.sav`, then `backup_current = true`. If that name exists, append `-2`, `-3`, … before `.sav`.
9. **Working copy**, if browsable: `cart_work.bin` = the 14 sectors `phys[k]` in id order. `dirty = 0`; changed cleared.

Working copy (D5):
- `mon_get` and `mon_put` go through `gen3_save_mon_spans` on `cart_work.bin` at `id*4096 + off`, with buffers of ≤ 100 bytes. Nothing touches the backup file.
- `mon_put` encrypts with `gen3_save_pkm_to_mon`, then sets the dirty bit for each span id and the changed bit.
  - A party slot equal to the count appends: it writes the count byte too, and sets id 1 dirty.
  - A party slot > count → `BadArg`.

ROM copy (D8):
- **Free space:** needs `rom_size − part + 4096` free, else `SdFull` before any `ROM_READ`.
- **Each region** (`GEN3_CART_ROM_REGION` bytes; the last one may be shorter):
  1. `ROM_READ`s of 2048 words (64 per full region), appended to the `.part`, with the region CRC updated as it goes (`RomRead`, progress in bytes).
  2. `ROM_CRC32` of the region must equal it, and so must the CRC of an SD read-back (`RomVerify`).
  3. On a mismatch, truncate to the region start and re-read once. A second mismatch → `VerifyFailed`, keeping the `.part`.
- **Cancel:** checked between `ROM_READ`s. It truncates to the last verified region and returns `Cancelled`.
- **Resume** starts at the `.part` length rounded down to a region.
- **End:** rename the `.part` to the final name.

`tests/fake_store.{h,c}` (RAM files):
```c
typedef struct FakeStore FakeStore;
void fake_store_init(FakeStore* s, uint64_t capacity);   /* bytes across all files */
void fake_store_report_free(FakeStore* s, uint64_t bytes); /* free_bytes() returns this instead of the truth */
Gen3Store fake_store_api(FakeStore* s);
void fake_store_fail_write_after(FakeStore* s, uint32_t writes);
void fake_store_corrupt_read(FakeStore* s, const char* name, uint32_t off); /* xor 0x01 on reads of that byte */
const uint8_t* fake_store_file(FakeStore* s, const char* name, uint32_t* size); /* NULL if absent */
uint32_t fake_store_count(FakeStore* s, const char* suffix);  /* files ending with suffix */
```

- [ ] **Step 1: Write the failing tests.**
  - **Default setup:** Ruby, MX, a save with A counter 4 first 0 and B absent; trainer `BRENDAN` 12345/54321; party of 3 (`GOLDEN_GAME` and two edits); box 0 named `BOX 1`, and Treecko at box 1 slot 19.
  - **Fixtures:** stamp `20261025-143005`, store capacity 64 MiB, and a fake ROM of 1 MiB.

```c
TEST(test_job_connect)        /* Ok; id.can_write; code "AXVE" */
TEST(test_job_backup_ok)      /* Ok; backups/AXVE_12345_20261025-143005.sav, 131072 B == fake flash; no .tmp;
                                 cart_work.bin 57344 B == slot A sectors in id order; trainer BRENDAN 12345; backup_current */
TEST(test_job_backup_nosave)  /* blank flash -> backups/AXVE_nosave_20261025-143005.sav; result Empty; not browsable;
                                 no cart_work.bin */
TEST(test_job_backup_sd_full_precheck) /* capacity 150000 -> SdFull; no files; on_command saw no READ_SECTOR */
TEST(test_job_backup_sd_full_midway)   /* capacity 70000, report_free 64 MiB -> SdFull; no .sav, no .tmp;
                                          backup_current false */
TEST(test_job_backup_sd_write_error)   /* fail_write_after(5) -> SdError; no files */
TEST(test_job_backup_readback_mismatch)/* corrupt_read(tmp, 7*4096+9) -> VerifyFailed; no .sav */
TEST(test_job_backup_flash_unstable)   /* on_command hook: after READ_SECTOR with arg0 4, flip a flash byte of
                                          sector 4 -> VerifyFailed */
TEST(test_job_backup_link_lost)        /* unplug for good at transfer 25000 -> LinkLost; no files */
TEST(test_job_backup_cancel)           /* cancel at BackupRead 3 -> Cancelled; no files */
TEST(test_job_backup_japanese)         /* AXVJ -> Ok, .sav written, not browsable, no cart_work.bin */
TEST(test_job_backup_refused_sram)     /* Sram cart -> Refused; no READ_SECTOR sent */
TEST(test_job_attach_keeps_changes)    /* backup, mon_put, fake_helper_restart, connect, attach -> Ok, dropped false,
                                          changes 1, no new .sav */
TEST(test_job_attach_after_game_save)  /* backup, mon_put, then a valid slot B (counter 5) copied into the fake flash from a
                                          second fake_save_build -> attach -> new .sav, dropped true, changes 0 */
TEST(test_job_mon_get_put)    /* party 0 get == GOLDEN_GAME; put it with level 50 -> only id 1 differs in cart_work.bin,
                                 dirty 0x0002, changes 1; put (1,19) -> dirty has bits 5 and 6; get returns the new bytes;
                                 the .sav unchanged */
TEST(test_job_party_append)   /* count 3: put party 3 -> count 4 at the count span; put party 5 -> BadArg */
TEST(test_job_discard)        /* after puts, discard -> cart_work.bin == slot A, changes 0 */
TEST(test_job_box_name)       /* box 0 -> "BOX 1" */
TEST(test_job_rom_dump)       /* 1 MiB ROM -> roms/AXVE_v0.gba == ROM bytes; 4 ROM_CRC32 calls; no .part */
TEST(test_job_rom_resume)     /* cancel once 300 KiB are read -> Cancelled, .part 262144 B; rom_part true, 262144;
                                 resume -> final == ROM; the resumed run sends exactly 192 ROM_READs */
TEST(test_job_rom_region_retry) /* fake_gba_rom_glitch_once(0x40010), armed after connect (the CART_INFO scan reads the ROM
                                   too) -> Ok, file == ROM; region 1 read twice (320 ROM_READs) */
TEST(test_job_rom_sd_full)    /* capacity 500000 -> SdFull before any ROM_READ */
TEST(test_result_texts)       /* every Gen3CartResult has non-empty text */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement.** `gen3_cart_result_text` strings (exact):

  | Result | Text |
  |---|---|
  | Ok | `Done.` |
  | WrittenNoBackup | `Written and verified, but the new backup could not be saved. Back up again before writing more.` |
  | Nothing | `No changes to write.` |
  | Cancelled | `Stopped.` |
  | LinkLost | `Link lost. The cartridge still loads its last good save.` |
  | BusyTimeout | `The GBA stopped answering during a flash step. The cartridge still loads its last good save.` |
  | HelperError | `The helper reported an error. See gen3_cart.log.` |
  | Refused | `Not allowed for this cartridge or save.` |
  | Changed | `The cartridge changed since the backup. Reconnect to back it up again.` |
  | SdFull | `SD card full.` |
  | SdError | `SD card error.` |
  | VerifyFailed | `Check failed: the data read back differs.` |
  | FlashError | `Flash write failed. The cartridge still loads its last good save.` |
  | BadBackup | `Not a valid backup for this game.` |
  | BadArg | `Invalid request.` |
  | NoHelper | `Helper file missing or damaged. Reinstall the app.` |
  | MultibootFailed | `Could not start the helper. Power the GBA off, hold START+SELECT, power it on, then Retry.` |
  | LinkBusy | `The link is in use. Leave the Trade or Link Test screen first.` |

  Map client results:

  | Client result | Cart result |
  |---|---|
  | `LinkLost` | `LinkLost` |
  | `BusyTimeout` | `BusyTimeout` |
  | `StatusError` on a flash step | `FlashError` |
  | `StatusError` `REFUSED` | `Refused` |
  | any other `StatusError`, `BadResponse` | `HelperError` |
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 5: Commit** `gen3: add cartridge backup, working copy and ROM copy jobs`.

---

### Task 10: Cart jobs II: safe write, post-write backup, restore, reconnect

**Files:**
- Modify: `src/gen3/gen3_cart.h`, `src/gen3/gen3_cart.c`
- Test: `tests/test_gen3_cart_write.c`

**Interfaces:**
- Produces:
  ```c
  Gen3CartResult gen3_cart_write(Gen3Cart* c, const char* stamp);
  /* Reads a backup file and checks it: 131072 bytes, strict with this cart's layout. Fills the backup's trainer. */
  Gen3CartResult gen3_cart_restore_check(Gen3Cart* c, const char* name, Gen3SaveTrainer* trainer, bool* has_trainer);
  Gen3CartResult gen3_cart_restore(Gen3Cart* c, const char* name, const char* stamp);
  /* After LinkLost: gen3_hcli_connect, CART_INFO header == stored, SAVE_PROBE id == stored; else Changed. */
  Gen3CartResult gen3_cart_reconnect(Gen3Cart* c);
  ```

`gen3_cart_write` (§5.8 and D4), in order:
1. **Preconditions,** checked before any link traffic:
   - `gen3_cart_writable`, else `Refused`;
   - no dirty bits → `Nothing`;
   - `free_bytes() < 131072 + 4096` → `SdFull`, so that the post-write backup has room.
2. **Plan:** `plan = gen3_save_plan_write(&slots)`.
3. **Prepare** (`Prepare`): for k = 0..13, read work sector k; set `counter = plan.counter`; if dirty, set `checksum = gen3_save_checksum(data, size[k])`. Write it back to the work file, and set `exp_crc[k] = gen3_crc32(sector)`.
4. **Footers:** `READ_FOOTERS`, then analyze with `valid = sig ok && id <= 13 && checksum == full_checksum`.
   - **Fresh:** the 14 footers of the slot that is not `plan.target` equal the current backup's, and the game's choice is unchanged (slot S in mode 0, no valid slot in mode 1) → step 5.
   - **Already written:** the active slot is `plan.target` with `plan.counter`, strict, and `SECTOR_CRC32(dst[k]) == exp_crc[k]` for all k → skip to step 9.
   - Anything else → `Changed`.
5. **Arm:** `arm_write(flash_id, plan.target, mode, N)`. Mode is 0 for a write, and 0 or 1 for a restore (below).
6. **Erase** `dst[k]` for k = 0..13 (`Erase`).
7. **Program** (`Program`): for k = 0..13, if dirty, `write_sector(dst[k], defer, work sector k)`; else `copy_sector(plan.src[k], dst[k], defer, plan.counter)`.
8. **Commit** `dst[k]` for k = 0..13 (`Commit`).
9. **Verify** (`Verify`): `SECTOR_CRC32(dst[k]) == exp_crc[k]` for all k, and a fresh `READ_FOOTERS` analysis shows active `plan.target`, `save_counter == plan.counter`, strict. Otherwise `VerifyFailed`.
10. **Post-write backup** (`PostBackup`, D4):
    - copy the current `.sav` to `backups/<code>_<tid>_<stamp>.tmp`, replacing each sector `dst[k]` with the image written there (the patched source of id k);
    - SD read-back CRC and `SECTOR_CRC32` must match for all 32 sectors;
    - rename the `.tmp` to `.sav`;
    - re-analyze from it, rebuild `cart_work.bin`, and clear dirty and changed.
    - If the space or a write fails → `WrittenNoBackup` with `backup_current = false`. Dirty and changed are still cleared, and `cart_work.bin` is kept: it already equals the new slot. Browsing and writing stay off until a backup succeeds (`gen3_cart_browsable`), so "Back up again" can't drop edits.
- **Failures in steps 5–9:** best-effort `disarm` (ignore its result), and return the mapped result. `backup_current` stays true, because slot S is untouched.

`gen3_cart_restore` (D6):
- **Preconditions:** `restore_check` Ok; `gen3_cart_id(c)->can_write`; `backup_current`; the same free-space check as step 1.
- **Plan, by the cart's save:**
  - strict → mode 0 with `gen3_save_plan_write`, sources = the backup's `phys_b[k]`;
  - `result` Empty or Corrupt → mode 1 with `gen3_save_plan_fresh(N_b, first_b)`;
  - otherwise → `Refused`.
- **Then:** steps 3–10 with every sector uploaded (no copies) from the backup file, and the counter patched to `plan.counter`.
- **The backup file is never written.** Step 3 patches each source sector in the buffer only, to get `exp_crc`, and step 7 reads and patches it again for the upload.

- [ ] **Step 1: Write the failing tests.**
  - **Setup:** Task 9's default setup and a backup taken first.
  - **Invariant checker:** a test-local `game_view(FakeGba*)`, run from `fake_helper_on_command` after every helper command. It analyzes the fake flash with exact checksums and asserts that `result` is Ok or Error, and that the CRCs of the active slot's 14 data areas equal either `orig_crc[]` (before the write) or `new_crc[]` (the expected edit). The checker counts violations, and each test asserts the count is 0.
  - **Sweeps** may snapshot the fake and store state after the backup and reuse it for each run.

```c
TEST(test_write_party_edit)   /* put party 0 at level 50; write -> Ok; slot B: 14 sectors, counter 5, sector for id k at
                                 14 + (k+1)%14; slot A and sectors 28-31 byte-identical to before; analyze: active 1, strict;
                                 party 0 level 50; WRITE_SECTOR 1, COPY_SECTOR 13; a second .sav equals the fake flash;
                                 cart_work.bin rebuilt; changes 0; backup_current */
TEST(test_write_box_straddle) /* put (1,19) -> WRITE_SECTOR 2 (ids 5, 6), COPY 12; reads back */
TEST(test_write_twice)        /* edit, write, edit, write -> 2nd targets slot A with counter 6; 3 .sav files */
TEST(test_write_nothing)      /* no changes -> Nothing; no link traffic */
TEST(test_write_sd_full_precheck) /* free below 135168 after connect -> SdFull; no link traffic, no flash ops */
TEST(test_write_requires_backup) /* a second gen3_cart_backup failing (fail_write_after) leaves backup_current false ->
                                 write Refused; erases/programs/commits 0 */
TEST(test_write_refused_not_strict) /* save with slot A's physical sector 13 at counter 2 -> backup Ok, why Save;
                                 write -> Refused; no flash ops */
TEST(test_write_refused_emerald) /* BPEE -> browse Ok, write Refused, no flash ops */
TEST(test_write_refused_no_flash_ops) /* Sram, chip None, other_lib, AXVD: backup and write fail or refuse; counts
                                 erases/programs/commits/bank_selects 0, Sram: sram_writes == the probe's 2 */
TEST(test_write_over_corrupt_inactive) /* slot B present with one broken sector, A active -> write Ok; B active, result Ok */
TEST(test_write_link_drop_sweep) /* for d = 400, 800, ... up to the reference run's transfer count: fresh state; unplug
                                 at d (from gen3_cart_write's start) for 10,000,000 transfers -> LinkLost or Ok; invariant
                                 holds; then fake_helper_replug, gen3_cart_reconnect Ok, gen3_cart_write Ok (or Nothing if
                                 the first run finished); final view == new */
TEST(test_write_retry_after_commit) /* unplug for good right after the 14th COMMIT request -> LinkLost; reconnect;
                                 write -> Ok through "already written": erases unchanged by the 2nd run */
TEST(test_write_short_glitch) /* unplug 30 transfers in the middle of the WRITE_SECTOR upload -> Ok, resends >= 1 */
TEST(test_write_power_cut_sweep) /* for k = 0, 1500, 3000, ... over the reference run's flash ops: power cut after k ops
                                 -> LinkLost; the fake flash's game view is the original or the new save, never another;
                                 result Ok or Error */
TEST(test_write_flash_error)  /* fail_program on dst[1] x99 -> FlashError; view original; helper disarmed (ERASE ->
                                 NOT_ARMED); clear, write -> Ok */
TEST(test_write_cart_changed) /* after backup, change one data byte of slot A in the fake (checksum fixed) -> Changed;
                                 no flash ops */
TEST(test_restore_ok)         /* backup X; edit+write (Y); restore X's .sav -> Ok; view == X's data, counter 6; new .sav;
                                 cart_work.bin == X's slot data */
TEST(test_restore_onto_corrupt_save) /* cart with both slots broken; a valid backup file (counter 9, first 4) in the store ->
                                 mode 1 -> Ok; view == backup data; active 1, counter 9, phys[0] == 18 */
TEST(test_restore_onto_empty_cart)   /* all 0xFF cart -> mode 1 Ok */
TEST(test_restore_wrong_game_backup) /* FRLG-built .sav onto Ruby -> restore_check BadBackup; restore Refused; no flash ops */
TEST(test_restore_bad_size)   /* 65536-byte file -> BadBackup */
TEST(test_restore_check_trainer) /* returns the backup's BRENDAN 12345 */
TEST(test_post_write_backup_sd_full) /* real capacity leaves < 131072 free after connect, report_free 64 MiB ->
                                 write WrittenNoBackup; view new; backup_current false; next write Refused */
```

- [ ] **Step 2: Run** `make -C tests`. Expected: FAIL.
- [ ] **Step 3: Implement.** Write and restore share one internal slot writer with a per-sector source: the work file (copy allowed when the sector is clean) or a backup file (always upload).
- [ ] **Step 4: Run** `make -C tests`. Expected: PASS; the whole suite runs in under 60 s.
- [ ] **Step 5: Commit** `gen3: add the safe cartridge write, post-write backup and restore`.

---

### Task 11: Cartridge session and SD store (Flipper glue)

**Files:**
- Create: `src/gen3/gen3_store_furi.h`, `src/gen3/gen3_store_furi.c`, `src/gen3/gen3_cart_furi.h`, `src/gen3/gen3_cart_furi.c`

**Interfaces:**
- Produces:
  ```c
  /* gen3_store_furi.h: Gen3Store rooted at APP_DATA_PATH("gen3"), resolved with
   * storage_common_resolve_path_and_ensure_app_directory; subdirs made on Create; free via storage_common_fs_info. */
  typedef struct Gen3StoreFuri Gen3StoreFuri;
  Gen3StoreFuri* gen3_store_furi_alloc(void);
  void gen3_store_furi_free(Gen3StoreFuri* s);
  const Gen3Store* gen3_store_furi_api(Gen3StoreFuri* s);
  /* gen3_cart_furi.h */
  typedef enum { Gen3CartJobConnect, Gen3CartJobBackup, Gen3CartJobWrite, Gen3CartJobDiscard, Gen3CartJobRestoreCheck,
                 Gen3CartJobRestore, Gen3CartJobRom, Gen3CartJobReconnect } Gen3CartJob;
  typedef struct {
      bool busy; Gen3CartJob job; Gen3CartResult result;   /* result valid when !busy */
      Gen3CartStep step; uint32_t done, total;
      bool cable_ok; Gen3MbStatus mb; bool link_up;        /* link_up: the helper answered the last command */
      bool dropped;                                        /* Connect: pending edits were dropped (gen3_cart_attach) */
      Gen3SaveTrainer restore_trainer; bool restore_has_trainer; /* RestoreCheck */
  } Gen3CartSessionStatus;
  typedef struct Gen3CartSession Gen3CartSession;
  Gen3CartSession* gen3_cart_session_alloc(void* gblink_handle); /* pins as gen3_session_alloc (plan 1) */
  void gen3_cart_session_free(Gen3CartSession* s);  /* cancels, joins the worker, stops the driver, frees buffers */
  /* arg: Restore/RestoreCheck file name (relative); Rom: "resume" or NULL. false if busy. */
  bool gen3_cart_session_run(Gen3CartSession* s, Gen3CartJob job, const char* arg);
  void gen3_cart_session_cancel(Gen3CartSession* s);
  void gen3_cart_session_status(Gen3CartSession* s, Gen3CartSessionStatus* out);
  Gen3Cart* gen3_cart_session_cart(Gen3CartSession* s);  /* GUI thread, only while !busy: mon_get/put, names, ids */
  ```

Behaviour:
- **Worker thread** `"Gen3Cart"`, stack 2048. It waits on thread flags for a job. Status is guarded by a `FuriMutex` and written only by the worker. It logs its free stack on exit (`FURI_LOG_D`, as plan 1's log thread does).
- **Connect job.** It is also the "send the helper again" path, so it works on a session that already ran.
  1. Stop the driver if it is running.
  2. On the first Connect, allocate the session buffer: `max(size of APP_ASSETS_PATH(GEN3_HELPER_ASSET), 4096)` bytes, capped at `GEN3_MB_MAX_LEN`. Then run `gen3_hcli_init` and `gen3_cart_init` with it. Later Connects reuse the buffer and keep the `Gen3Cart` state, so `gen3_cart_attach` can keep the backup and the edits.
  3. Load the image into the buffer. `gen3_mb_image_check(…, GEN3_HELPER_PROTO)` must be Ok, else `NoHelper` (a missing file is `NoHelper` too).
  4. Start the driver with `mlink_hw_start_mode(MlinkModeStream, …)`.
     - While `mlink_hw_check_cable` reads Reversed: `cable_ok = false`, retry every 250 ms, honour cancel.
     - Once running, `cable_ok` follows `mlink_hw_cable_ok`.
     - TIM2 busy → `LinkBusy`.
  5. Run multiboot: `gen3_mb_init(…, furi_hal_random_get())`, with `MbWait`/`MbSend` progress from `gen3_mb_status`. Failed → `MultibootFailed`.
  6. Switch the ISR core to the pump: `gen3_hlink_init`, then `__DMB()`, then set the core selector. The buffer is now the sector buffer.
  7. `gen3_cart_connect`, then `gen3_cart_attach`.
- **Frame callback (ISR).** It does exactly this:
  - `elapsed = mlink_frame_us(MlinkModeStream, rx ? rx->count : 0)`;
  - dispatch to `gen3_mb_frame` or `gen3_hlink_frame` by the selector;
  - if `GEN3_CART_GAP_TICKS` (default 0, tunable for M18) is nonzero, insert that many count-0 frames after each frame that transferred, adding their time to the next call's `elapsed`.
- **Transport:**
  - `exchange` submits to the pump and polls `gen3_hlink_state` with `furi_delay_ms(1)` until Done or Failed. The ISR sends no signal.
  - `sync` does the same with `gen3_hlink_sync`.
  - `link_up` = the last exchange was Done.
- **Other jobs** call the matching `gen3_cart_*`. Reconnect runs `gen3_cart_reconnect`.
- **Stamp** `YYYYMMDD-HHMMSS` from `furi_hal_rtc_get_datetime`.
- **Log** `APP_DATA_PATH("gen3_cart.log")`: created at alloc (`FSOM_CREATE_ALWAYS`); first line `# gen3 cart log v1`. Only the worker writes it, one line per event, each `"%08lu "` with `furi_get_tick()`. Events:
  - job start and end, with the result;
  - multiboot restarts with `last_fail`, `cc` and `rr`;
  - hello proto and build;
  - cart code, flash id, why;
  - the slot analysis;
  - `resends` per job;
  - `last_status`/`fail_code`/`fail_off` on every StatusError.

  No SD card → no log, and the job still runs.

- [ ] **Step 1: Implement** the two modules.
- [ ] **Step 2: Build** with both SDKs. Expected: 0 warnings.
- [ ] **Step 3: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 4: Size.** Report `text + data` against `SIZE_BASE`. If the growth is already above 18,000, stop and report before the UI tasks.
- [ ] **Step 5: Self-review against the ISR rule.** The callback and everything it calls make no furi calls and no allocations.
- [ ] **Step 6: Commit** `gen3: add the cartridge session and SD store`.

---

### Task 12: UI I: Cartridge entry, connect, cart info

**Files:**
- Create:
  - `src/views/gen3_cart_progress.h`, `src/views/gen3_cart_progress.c`;
  - `src/scenes/include/pokemon_gen3_cart.h`;
  - `src/scenes/pokemon_gen3_cart_connect.c`, `pokemon_gen3_cart.c`, `pokemon_gen3_cart_info.c`;
  - `pokemon_gen3_cart_confirm.c`, `pokemon_gen3_cart_job.c`, `pokemon_gen3_cart_result.c`;
  - `docs/gen3/hardware-test-3.md`.
- Modify:
  - `src/scenes/include/pokemon_scene_config.h`;
  - `src/include/pokemon_app.h`: append `AppViewGen3CartProgress` to `AppView`, plus fields `void* gen3_cart_progress;` and `struct Gen3CartUi* gen3_cart_ui;`;
  - `src/pokemon_app.c`: set both to NULL; `free_app` calls `gen3_cart_ui_end` before `view_dispatcher_free`;
  - `src/scenes/pokemon_gen3_menu.c`.

**Interfaces:**
- Produces:
  ```
  ADD_SCENE(pokemon, gen3_cart_connect, Gen3CartConnect)
  ADD_SCENE(pokemon, gen3_cart, Gen3Cart)
  ADD_SCENE(pokemon, gen3_cart_info, Gen3CartInfo)
  ADD_SCENE(pokemon, gen3_cart_confirm, Gen3CartLeave)
  ADD_SCENE(pokemon, gen3_cart_job, Gen3CartJobRun)
  ADD_SCENE(pokemon, gen3_cart_result, Gen3CartResult)
  ```
  ```c
  /* pokemon_gen3_cart.h */
  typedef enum { Gen3CartEditNone, Gen3CartEditExisting, Gen3CartEditNew } Gen3CartEditMode;
  typedef struct Gen3CartUi {
      Gen3CartSession* session;
      Gen3Editor saved_editor;            /* *gen3_editor before Cartridge; put back on leave (D10) */
      uint8_t box, slot; Gen3CartEditMode edit_mode;
      uint8_t orig[GEN3_PARTY_SIZE]; size_t orig_len;
      Gen3CartJob job; char arg[GEN3_CART_NAME_MAX]; /* job for Gen3CartJobRun */
      Gen3CartResult result; char result_header[24]; /* for Gen3CartResult */
      const char* result_text;  /* NULL: gen3_cart_result_text(result) */
      uint32_t result_back;     /* scene that OK searches back to; default PokemonSceneGen3Cart */
  } Gen3CartUi;
  /* Allocs on first use: the ui, the session, the saved editor copy, and the progress view
   * (app->gen3_cart_progress, view id AppViewGen3CartProgress). */
  Gen3CartUi* gen3_cart_ui_get(PokemonFap* app);
  /* Frees the session, the progress view and the ui, and restores the editor. Safe if nothing was allocated.
   * Callers switch away from the progress view first (the Gen III menu's on_enter does, as for the Link Test);
   * free_app calls it before view_dispatcher_free. */
  void gen3_cart_ui_end(PokemonFap* app);
  /* views/gen3_cart_progress.h */
  void* gen3_cart_progress_alloc(ViewDispatcher* vd, uint32_t view_id);
  void gen3_cart_progress_free(ViewDispatcher* vd, uint32_t view_id, void* p);
  void gen3_cart_progress_set(void* p, const char* title, const char* line1, const char* line2, int8_t percent); /* -1: no bar */
  void gen3_cart_progress_set_back(void* p, void (*cb)(void* ctx), void* ctx); /* Back key */
  ```

Behaviour:
- **Gen III menu:**
  - Append the item `Cartridge` → `PokemonSceneGen3CartConnect`; `on_event` accepts it.
  - `on_enter` also calls `gen3_cart_ui_end(app)`, after switching to the submenu view, the same way it ends the Link Test.
  - Before allocating the session, end any other Gen III link session the app holds:
    - plan 1's Link Test is already ended by the menu's `on_enter`;
    - if plan 3 keeps a background trade session, call its stop/free exactly as `pokemon_gen3_menu.c` does when leaving Gen III, and name the call in your report.
- **Progress view:**
  - Layout: title in FontPrimary at y 10, two FontSecondary lines at y 26 and 38, `elements_progress_bar(4, 46, 120, p)` when percent ≥ 0, and a hint `Back: stop` or `Do not power off` at y 62.
  - The model is `ViewModelTypeLocking` with fixed char arrays (title 22, lines 26).
- **`Gen3CartConnect`:**
  - **Instructions** (DialogEx): header `Cartridge`. Text: `GBA off, cart in.`, `Small plug: Flipper.`, `Hold START+SELECT`, `and power on.` Left `Back`, right `Start`.
  - **Start** runs the Connect job and switches to the progress view, refreshed by a 250 ms `FuriTimer` that sends a custom event.
  - **Lines while connecting:**
    - `!cable_ok` → `Cable: plug in/swap ends`;
    - MbWait → `Waiting for GBA`, and after `waiting_us ≥ 3 s` line 2 `Hold START+SELECT at power on`;
    - MbSend → `Sending helper` with %;
    - Identify → `Reading cartridge`;
    - BackupRead → `Backing up save` with % and `about 70 s`;
    - BackupVerify → `Checking backup`.
  - **Back,** on the instructions or while busy → cancel any job, then search back to Gen3Menu. If `gen3_cart_changes > 0`, go through `Gen3CartLeave` first.
  - **Done Ok** → `Gen3Cart`. If `dropped`, show first a result: `Edits dropped` / `The save on the cartridge changed.`
  - **Error** → `Gen3CartResult` with header `Connect failed` and a `Retry` button. Retry searches back to `Gen3CartConnect`, which shows the instructions again.
- **`Gen3Cart`** (submenu hub):
  - The header is the cart title.
  - Items in this task: `Cart info`; `Reconnect` when `!link_up`; `Back up again` when `!backup_current` and `can_read_save`.
  - Back → if `gen3_cart_changes > 0`, open `Gen3CartLeave` (DialogEx `Leave?` / `<n> changes not written.`, left `Stay`, right `Leave`); else search back to Gen3Menu.
- **`Gen3CartInfo`** (DialogEx, center `OK`): header `<code> v<version>`. Lines:
  - `Flash <id hex4> <Macronix|Sanyo|?>`;
  - `Save: slot A|B #<counter>`, `Save: empty` or `Save: corrupt`;
  - `<trainer> <tid %05u>`;
  - `Writing on` or `<gen3_cart_refuse_text>`;
  - `Backup <stamp>`.
- **`Gen3CartJobRun`:**
  - Runs `ui->job` once per entry. It is guarded by a `started` flag so that coming back from an overlay doesn't restart it.
  - Progress texts per step: Erase/Program/Commit `Writing`, hint `Do not power off`; Verify `Verifying`; PostBackup `Saving new backup`; RomRead `Copying ROM`; RomVerify `Checking ROM`; Analyze `Reading save`; Prepare `Preparing`; RestoreCheck `Checking backup file`.
  - On done → `Gen3CartResult`.
- **`Gen3CartResult`** (DialogEx): header `ui->result_header`, text `ui->result_text` or `gen3_cart_result_text`.
  - Center `OK` → search back to `ui->result_back`, then reset it to Gen3Cart.
  - For `LinkLost` and `BusyTimeout`: right `Retry` runs the Reconnect job in `Gen3CartJobRun`. If Reconnect itself fails, its result's `Retry` searches back to `Gen3CartConnect`, which sends the helper again; `gen3_cart_attach` keeps the edits if the save is unchanged.

- [ ] **Step 1: Implement** the view, scenes and wiring.
- [ ] **Step 2: Build** with both SDKs. Expected: 0 warnings.
- [ ] **Step 3: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 4: Write `docs/gen3/hardware-test-3.md`.**
  - **Intro:**
    - Goal.
    - What you need: the Ruby and FireRed carts, cable and module.
    - Build: `ufbt` release; install `dist/pokemon.fap` with qFlipper or `ufbt launch`. Make sure `apps_assets/pokemon/gen3_helper.mb` exists after install.
    - Keep `apps_data/pokemon/gen3_cart.log` from every failed run.
  - **A. Connect (Ruby):**
    1. Gen III → Cartridge shows the instructions.
    2. With the small plug in the Flipper, the GBA off and the cart in, press Start. The Flipper shows `Waiting for GBA`; after 3 s line 2 shows `Hold START+SELECT at power on`.
    3. Hold START+SELECT and power on. You should see `Sending helper` (about 3–5 s), then the GBA screen turns green, then `Reading cartridge`, then `Backing up save` (about 70 s), then the hub `POKEMON RUBY`.
    4. Power on without the keys: the game starts and the Flipper keeps waiting. Power off, then retry with the keys.
    5. The wrong plug shows `Cable: plug in/swap ends`.
    6. Back during `Sending helper` returns to the Gen III menu.
  - **B. Cart info and backup:**
    1. Cart info shows `AXVE` and its version, `Flash 09C2 Macronix` or `1362 Sanyo`, the save slot and counter, your trainer name and TID (compare the in-game trainer card), and `Writing on`.
    2. `apps_data/pokemon/gen3/backups/AXVE_<tid>_<stamp>.sav` is 131,072 bytes.
    3. Optional: it loads in mGBA or PKHeX and shows your party.
- [ ] **Step 5: Commit** `gen3: add the Cartridge menu, connect screen and cart info`.

---

### Task 13: UI II: party and box browser, editing, putting Pokemon, writing

**Files:**
- Create: `src/scenes/pokemon_gen3_cart_boxes.c`, `pokemon_gen3_cart_slots.c`, `pokemon_gen3_cart_slot.c`
- Modify:
  - `src/scenes/include/pokemon_scene_config.h`;
  - `src/scenes/pokemon_gen3_cart.c`: items;
  - `src/scenes/pokemon_gen3_cart_confirm.c`: the new states;
  - `docs/gen3/hardware-test-3.md`.

**Interfaces:**
- Produces:
  ```
  ADD_SCENE(pokemon, gen3_cart_boxes, Gen3CartBoxes)
  ADD_SCENE(pokemon, gen3_cart_slots, Gen3CartSlots)
  ADD_SCENE(pokemon, gen3_cart_slot, Gen3CartSlot)
  ADD_SCENE(pokemon, gen3_cart_confirm, Gen3CartPut)
  ADD_SCENE(pokemon, gen3_cart_confirm, Gen3CartWriteAsk)
  ADD_SCENE(pokemon, gen3_cart_confirm, Gen3CartDiscardAsk)
  ```
- **Alias rule:** before `next_scene` to any `gen3_cart_confirm` alias, the caller writes the target scene id into `Gen3CartLeave`'s scene state (plan 2's alias pattern).
- **Precondition:** this task relies on `Gen3Edit`'s Back being the scene manager's default previous-scene. If plan 2's code searches back to `Gen3Menu` instead, change it to the default; that is the same behaviour for plan 2's own entry from the Gen III menu. Say so in your report.

Behaviour:
- **Hub items:**
  - `Party and boxes` when browsable.
  - `Write changes (<n>)` when `gen3_cart_writable` and n > 0.
  - `Writing off: <refuse text>` when not writable → Cart info.
  - `Discard changes` when n > 0.
- **`Gen3CartBoxes`:** `Party (<count>/6)`, then 14 rows of the box names from the save.
- **`Gen3CartSlots`:** the header is the box name. The scene state keeps the highlight.
  - **Party:** rows 1..count, then `-- empty --` if count < 6.
  - **Box:** 30 rows.
  - **Row text:** from `gen3_cart_mon_get` and `gen3_save_mon_summary`. `%02u <SPECIES> Lv<n>` with a trailing `*` when changed; `-- empty --`; `EGG`; `BAD EGG`. Read row by row with ≤ 100-byte buffers.
- **`Gen3CartSlot`** (router; selecting a row lands here). It decides in `on_enter`, then switches scenes from `on_event` on a custom event it sends itself:
  - **Occupied Ok:** save `orig`, `gen3_save_mon_to_pkm`, `gen3_edit_load`, set `edit_mode = Existing`, then `next_scene(Gen3Edit)`. On re-entry with Existing: if `gen3_save_pkm_to_mon(ed->pkm, orig_len)` ≠ `orig` and `gen3_pkm_check == Gen3CheckOk`, call `gen3_cart_mon_put`. Set mode None and go back to the previous scene.
  - **Empty:** submenu `New Pokemon`, `Load .pk3`.
    - New: `gen3_cart_new_mon(ed, trainer, met_game, furi_hal_random_get())`, set mode New, then Gen3Edit.
    - Load: plan 2's `gen3_pk3_dir` and `dialog_file_browser_show` (`.pk3`), then `gen3_pk3_load_file`. Ok → `gen3_edit_load`, mode New, Gen3Edit. Error → result `Load failed` / `gen3_check_text`.
    - Re-entry with New → `Gen3CartPut`: `Put <SPECIES> Lv<n>` / `in <box name> slot <k>?`, left `Cancel`, right `Put` → `gen3_cart_mon_put`.
  - **Egg:** result `Eggs can't be edited`. **Bad:** result `Bad Egg: left as is`.
  - These slot-level results set `result_back = PokemonSceneGen3CartSlots`, so OK returns to the list.
- **`Gen3CartWriteAsk`:** `Write <n> changes?` / `The save is backed up.\nKeep the GBA on.` Left `Cancel`, right `Write` → `Gen3CartJobRun` with Write.
  - Ok → result header `Written`, text `Written and verified.\nPower-cycle the GBA to play.` `WrittenNoBackup` uses its own text.
  - Errors → header `Write failed`.
- **`Gen3CartDiscardAsk`:** `Discard <n> changes?` → the Discard job.

- [ ] **Step 1: Implement** the scenes and items.
- [ ] **Step 2: Build** with both SDKs. Expected: 0 warnings.
- [ ] **Step 3: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 4: Add sections C–E** to `hardware-test-3.md`:
  - **C. Browse:**
    1. Party lists your party with the right levels.
    2. The boxes show your box names.
    3. A box lists its Pokemon; empty slots show `-- empty --`.
  - **D. Edit and write (Ruby):**
    1. Open party slot 1. The editor shows its species, level, nickname and OT as in the game.
    2. Set the level to 50, then Back. The row shows `*` and the hub shows `Write changes (1)`.
    3. Write: you should see `Writing` (about 10 s), then `Written and verified`.
    4. Power-cycle the GBA without keys. Continue: no "save corrupt" message, and party slot 1 is Lv 50. Save in-game and reload: still fine.
    5. Reconnect (A.3). The hub shows the new state. Its connect backup is a new `.sav`.
    6. **Link drop:** edit any Pokemon, choose Write, and pull the cable while `Writing` shows. You should get `Link lost …`. Plug it back and press Retry: the Flipper reconnects. Choose Write again: `Written and verified`. The game loads the edit, with no corrupt-save message (Review Focus 1).
  - **E. Put Pokemon:**
    1. In box 2's first empty slot, choose New Pokemon, set TREECKO Lv 10, then Back: you should see `Put TREECKO Lv10 in <box> slot 1?` → Put.
    2. In another empty slot, Load .pk3 a file saved with plan 2 → Put.
    3. In the party's empty slot, New → Put.
    4. Write. In the game the boxes show them; their OT is your name and ID, and they obey. The party has the new member.
    5. Discard: make a change, choose Discard changes; the `*` is gone.
- [ ] **Step 5: Commit** `gen3: add the cartridge party and box browser and writing`.

---

### Task 14: UI III: restore and copy ROM

**Files:**
- Create: `src/scenes/pokemon_gen3_cart_restore.c`
- Modify:
  - `src/scenes/include/pokemon_scene_config.h`;
  - `src/scenes/pokemon_gen3_cart.c`: items;
  - `src/scenes/pokemon_gen3_cart_confirm.c`: states;
  - `src/scenes/pokemon_gen3_cart_job.c`: Back handling for the ROM copy;
  - `docs/gen3/hardware-test-3.md`.

**Interfaces:**
- Produces:
  ```
  ADD_SCENE(pokemon, gen3_cart_restore, Gen3CartRestore)
  ADD_SCENE(pokemon, gen3_cart_confirm, Gen3CartRestoreAsk)
  ADD_SCENE(pokemon, gen3_cart_confirm, Gen3CartRomAsk)
  ADD_SCENE(pokemon, gen3_cart_confirm, Gen3CartRomStop)
  ```

Behaviour:
- **Hub items:** `Restore backup` when `gen3_cart_id(c)->can_write` and `gen3_cart_backup_current`. This also covers an empty or corrupt save, through mode 1. `Copy ROM` when `gen3_cart_rom_size(c) > 0`.
- **`Gen3CartRestore`:**
  1. The file browser in `<gen3 dir>/backups` (`.sav`). Cancel → previous scene.
  2. A picked file runs `RestoreCheck`. BadBackup → result `Can't restore` / its text.
  3. Ok → `Gen3CartRestoreAsk`: `Restore backup?` / `Cart: <name> <tid>` (or `Cart: no save`) / `File: <name> <tid>` / `Hall of Fame is kept.` Left `Cancel`, right `Restore`.
  4. Restore runs the Restore job. The result is `Restored` / `Restored and verified.\nPower-cycle the GBA to play.`
- **Copy ROM:**
  - If a `.part` exists, `Gen3CartRomAsk` shows `Resume copy?` / `<pct>% done`, left `Restart`, right `Resume`. Restart removes the part first.
  - Otherwise: `Copy ROM?` / `<MB> MB, about <h> h <m> min.\nKeep both powered.` Left `Cancel`, right `Copy`. MB = `gen3_cart_rom_size / 1048576`; time = rom_size / 1850 bytes per second (§6.6 stream rate).
  - Progress: `Copying ROM` with % and line 2 `<h> h <m> min left`, from the measured rate.
  - Back during the job → `Gen3CartRomStop`: `Stop copying?` / `You can resume later.` Left `Keep going`, right `Stop` → cancel.
  - The job keeps running under the dialog.
  - Done: `ROM copied` / `Verified.\n<roms/... name>`. Cancelled: `Stopped` / `Copy ROM again to resume.`

- [ ] **Step 1: Implement.**
- [ ] **Step 2: Build** with both SDKs. Expected: 0 warnings.
- [ ] **Step 3: Run** `make -C tests`. Expected: PASS.
- [ ] **Step 4: Add sections F and G** to `hardware-test-3.md`:
  - **F. Restore:**
    1. Restore backup, and pick the `.sav` from A. The confirmation shows both trainers.
    2. Restore (about 40 s) → `Restored and verified`.
    3. The game loads the save as it was at A: the party levels from before D.
  - **G. Copy ROM:**
    1. Copy ROM shows `16 MB, about 2 h 30 min`.
    2. Copy for 1 min, then Back → Stop → `Stopped`.
    3. Copy ROM again → `Resume copy?` → Resume. The percent continues.
    4. Optional: let it finish. `roms/AXVE_v<n>.gba` is 16,777,216 bytes, and on a computer `shasum` of it matches your revision in §3.6's table.
- [ ] **Step 5: Commit** `gen3: add cartridge restore and ROM copy screens`.

---

### Task 15: Docs, version 3.1, final checks, user gate

**Files:**
- Modify: `README.md`, `changelog.md`, `application.fam` (`fap_version=[3,1]` only), `docs/gen3/hardware-test-3.md`

README section "Gen III (experimental): cartridge save access" covers:
- the Start+Select boot and the small plug in the Flipper;
- what the backup is: a raw `.sav` in `apps_data/pokemon/gen3/backups/`, loadable in emulators and PKHeX, plus a new one after every write;
- how writing works: the save slot the game is not using, with the old one kept, and verification;
- which carts can be written:
  - genuine English Ruby, Sapphire, FireRed and LeafGreen are writable;
  - Emerald and other Western languages are read-only;
  - Japanese carts get backup and ROM copy only;
  - reproduction carts are refused;
- that restore keeps the Hall of Fame and e-Reader/Trainer Tower sectors;
- that the Pokedex isn't updated for Pokemon you add;
- that the ROM copy takes about 2.5 h and can resume;
- that the helper's source is in `gba/helper/`, rebuilt with `make -C gba/helper install`.

`changelog.md`: a new top section `## Version 3.1` with the bullet "Gen III cartridge save access (experimental): back up, browse and edit the party and boxes, write back with verify, restore, copy ROM". Point to `docs/gen3/hardware-test-3.md`.

- [ ] **Step 1: Write** the README section, the changelog section and the version bump.
- [ ] **Step 2: Add the closing sections** to `hardware-test-3.md`:
  - **H. FireRed:** repeat A–E with FireRed. Cart info shows `POKEMON FIRE`/`BPRE`; the party, boxes, edit and write work; the game loads.
  - **R. Regression:** the Gen III Link Test still reaches `in Trade Center` (hardware-test-1 steps 4–8; frame mode is unchanged by Task 1). Gen I and Gen II still open their editors.
  - **M. Memory:** during `Backing up save`, run `free` in `ufbt cli` and write down `Free heap size`. The app must not crash anywhere in A–G.
  - **Results:** a table with pass/fail per step. On a failure, report the step, both screens, and `gen3_cart.log`.
  - **If it fails (for whoever reads the log):**

    | Symptom | Likely cause | Try |
    |---|---|---|
    | Never leaves `Waiting for GBA` | the plug, or the keys at power on | — |
    | Log shows multiboot restarts with `Header` or `Crc` | M3, M4 | — |
    | Multiboot fails 4× | power-cycle needed (M4) | — |
    | `Link lost` often during long jobs | stream-mode load (M18) | raise `GEN3_CART_GAP_TICKS` in `gen3_cart_furi.c` |
    | Flash steps time out | M10 | — |
    | `ROM patched for another save type` on a genuine cart | a false hit in the library-string scan | report the log's `lib_off` |
    | `Unknown save chip` on a genuine cart | — | report the flash id line |
    | ROM reads unstable | WAITCNT | set `HP_WAITCNT 0x0003` |
- [ ] **Step 3: Final checks.** Put each output in the report.
  - `make -C tests` passes.
  - `make -C gba/helper check` passes, and `make -C gba/helper size` is ≤ 8192.
  - Both SDK builds have 0 warnings.
  - The `text + data` growth over `SIZE_BASE` is ≤ 24,000.
  - These diffs print nothing:
    - `git diff $BASE -- lib/flipper-gblink`;
    - the plan 2 list of Gen I/II files: `git diff $BASE --stat -- src/pokemon_data.c src/pokemon_data_i.h src/pokemon_table.c src/pokemon_attribute.c src/pokemon_char_encode.c src/patch_list.c src/named_list.c src/move_nl.c src/item_nl.c src/stat_nl.c src/type_nl.c src/views/select_pokemon.c src/views/trade.c src/scenes/pokemon_menu.c src/scenes/pokemon_gen.c src/scenes/pokemon_select.c src/scenes/pokemon_name_input.c src/scenes/pokemon_number_input.c src/scenes/pokemon_move.c src/scenes/pokemon_item.c src/scenes/pokemon_stats.c src/scenes/pokemon_shiny.c src/scenes/pokemon_gender.c src/scenes/pokemon_pokerus.c src/scenes/pokemon_type.c src/scenes/pokemon_trade.c src/scenes/pokemon_exit_confirm.c src/scenes/pokemon_trade_reset_confirm.c`.
  - `git diff $BASE -- src/gen3/mlink_hw_furi.c` is confined to Task 1's list.
- [ ] **Step 4: Commit** `docs: Gen III cartridge save access; version 3.1`.
- [ ] **Step 5: Hand-off.** Push `gen3-save-access`, and open a draft PR against `gen3-gba-design` on `LawlessNPC/Flipper-Zero-Game-Boy-Pokemon-Trading`. Give the user `dist/pokemon.fap` and `docs/gen3/hardware-test-3.md`. **Stop here until the user reports results.**
  - **Memory:** if `free` reads under 4,000, or the app won't start, report the numbers. The fallback is plan 2's D2 (move the tables to an SD asset, about 18 KB), as a follow-up.
  - **Other failures:** read `gen3_cart.log` against the table in the checklist, fix, and retest only the failed sections.

---

## Spec coverage

| Requirement (spec) | Task |
|---|---|
| Send the helper to the GBA (multiboot) | 2, 11, 12 |
| Helper source in the repo, built binary committed to `files/` (call #5) | 3, 4, 5 |
| Back up first, verify | 9, 12 |
| Write the save safely, verify what was written | 10, 13 |
| Edit the party and all 14 boxes in place, in the shared editor | 8, 9, 13 |
| Put Pokemon in empty slots (new, or from `.pk3`) | 8, 9, 13 |
| Restore from an SD backup | 10, 14 |
| Copy the ROM to the SD card (hours; said in the UI; resumable) | 9, 14 |
| UI: Cartridge item, Start+Select walk-through, helper progress, cart info (game, save type, trainer) | 12 |
| UI: party/box browser, open in the plan 2 editor, put new or `.pk3` Pokemon, write with confirm, backup and verify, clear errors | 13 |
| UI: Restore backup, Copy ROM | 14 |
| Driver free-running mode, frame mode unchanged | 1, 15 (R) |
| Docs: `hardware-test-3.md`, README, changelog, v3.1 | 12–15 |
| RS layouts required, FRLG next, Emerald later (read-only) | 8, 10, 15 (H) |
| Only write positively identified save types (call #10); refuse repros | 4, 5, 8, 10 |
| Anti-goal: never write without a verified backup | 9, 10 (D4, D7) |
| Anti-goal: never damage the save | 5, 10 (sweeps), Global Constraints |
| Japanese games out of scope | 8, 9 (D9) |
| Non-goal: no ROM modification; no trainer, bag or money editing | ROM read-only (5); only party and box spans are written (8, 9) |
| 4 KB pieces (RAM) | Global Constraints, 9, 11 |
| Gen I/II untouched | Global Constraints, 15 |
| Host-testable core | 2–10 |
| Phase 2 ships as 3.x | 15 |
