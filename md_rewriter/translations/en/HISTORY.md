# HISTORY.md: how this project was developed

A summary of the development process of the project that adds support to the **MD Rewriter** (an STM32 Mega Drive cartridge dumper/writer) and to **KY / FlashKit** flash cartridges, from 31 October to 14 November 2026. The work was done by one person with the help of an AI agent (Claude Code): the person decided, the agent analysed and wrote code and documents, and the bench judged. Dates come from the repository history and from the notes of each session.

The technical details and practical instructions are in the `CLAUDE.md` that accompanies this file. Here is the story: what was done, in what order, what went wrong and what was learned.

---

## Starting point (October 2025)

The original project (`Epicjoy/md_rewriter`) was published on 10 October 2025: board project in EasyEDA, firmware 1.0 (STM32F103VB, Keil/CubeMX project), a Qt application and a release with the factory `.hex`. It had little documentation, no tests, no schematic, and the application and firmware wrote only a few flash chips well.

The person who led this work bought the cartridges and the dumper and, while the board had not yet arrived, decided to prepare all the software and firmware **on paper and on the PC**, so that the bench would only be needed to confirm.

## Phase 1: review and quality base (31 October 2026)

- Firmware review: vector bounds, `volatile` where it was missing, unbounded waits replaced by bounded waits, compiler warnings eliminated, a static check script (`arm-none-eabi-gcc` with warnings and `cppcheck`).
- Application rebuilt in layers: protocol, device session, interface, translations (Portuguese, English, Russian, Chinese) and a test suite (unit, integration against a simulated firmware, and end-to-end with a pseudo-terminal serial port).
- User guide built into the application, README in four languages, specification and plan, MIT licence.
- Cancel, unplug and "busy" tests made deterministic.

**Lesson:** a simulated firmware inside the tests made it possible to exercise real failures (fragmented replies, unplugged cable) long before any hardware existed.

## Phase 2: freeze the old, build the new (2 and 3 November)

- `Firmware/` became `legacy_firmware/` (frozen, review fixes only) and a new firmware was born with a GCC build and a `Makefile`, later named 1.1. The Keil one was not carried forward.
- Hardware review of PCB V1.0, read only, extracting the netlist from the `.eprj` file (a SQLite database with base64+gzip data). Result: the main defect is the missing 5.1 kΩ resistors on the USB-C; also wrong metadata, a stale BOM, a divergent polyfuse, unlabelled SWD with no NRST, no ESD protection.
- Application: even-sized ROMs of any size up to 4 MiB, `.smd` conversion, verification by reading back, chip names from `FLASHID`, `WRITE TIMEOUT` handling, 8 MiB reads offered only when the firmware identifies itself with `FWVER:`. Version 1.2.0.
- Firmware: S29GL064N identification from the Cypress datasheet, autoselect reading, erase by size, **DQ7 polling after every programmed word**, and the decision to **test the firmware code on the PC**: the real `MD.c` compiled against an AMD flash chip model (the suite reached more than 200 checks). SST39VF chips and an optional write-buffer programming (off by default) were also added.
- The PC model caught a real erase bug (DQ5 misread on `0xFF` when an erase finished between two status reads).

**Lesson:** the PC model is a well-tested hypothesis, not a fact. Everything that depended on it was marked "not tested on hardware" until the bench confirmed it.

## Phase 3: KY / FlashKit cartridges (4 November)

- Research in open sources (the repositories listed in section 12 of `CLAUDE.md`): it became clear that there is no KY-exclusive protocol. "KY support" came to mean "write and read, with our dumper, the chips and mappers found in FlashKit/KY cartridges".
- Firmware 1.1 for the current PCB: 32 Mbit chips (M29W320, MX29LV320), and two new command sets (AMD with `5555h/2AAAh` unlock and Intel/Sharp).
- `mdrewriter-cli`: the same session as the application, for scripts and agents, with `--yes` mandatory for anything that changes the cartridge, `--json` and defined exit codes.
- A **read-only** bench suite (`hwtest.py`): a whitelist of read commands, refuses `--yes`.
- **Decision:** the rework of the MD Rewriter PCB was cancelled. The hardware ideas moved to the sister project `oscr_minimal` (an Arduino Mega shield); the MD Rewriter V1.0 keeps being used as it came, with better firmware.

## Phase 4: more coverage on paper (5 to 10 November)

- `50h` block erase for SST39VF320x and writes of up to 8 MiB.
- CFI query for chips outside the table (information only) and ROM read with the SSF2 mapper (command `3A`, model only).
- README and in-application help updated; documentation of how to wire the ST-Link V2 to the SWD pads (including the pin map of the original ST-Link and of the clones).
- Items that were **declined on purpose** for lack of confirmed data: Sharp LH28F320BJE and Toshiba TC58FVT160F.
- Global plan with a log (`GLOBAL_PLAN.md`): each step done, each step postponed, so the next session does not redo the analysis.

## Phase 5: the board arrives (13 November)

The busiest day. In order:

1. **First contact (firmware 1.0, empty slot).** The board enumerates as `0483:5740`; `hwtest.py host` and `nocart` pass. The empty slot reads only `0xFF`. Finding 1: firmware 1.0 ignores a `connect` that arrives right after the port is reopened (the CLI now tries twice). Finding 2: **there is no USB update** (plain CDC, no DFU, ROM bootloader UART only): SWD is required.
2. **Original cartridge (Rolling Thunder 2), read only.** Repeatable reads, no stuck bits, a read larger than the cartridge becomes a mirror. The header checksum does not match; without a reference dump the doubt stayed open. The person decided: **this cartridge will never be written**.
3. **S29GL064N flash cartridge with firmware 1.0.** Two findings that changed course: (a) the firmware discards bytes sent in the 100 ms after `detect` (the application now holds 250 ms); (b) 1.0 **programs only every other word**: no wrong word, half blank. Workaround: a repair tool that writes only the missing words, in 8 passes. Metal Gear (2 MiB) was written and verified this way.
4. **SWD session.** The 4-pin connector was soldered. The link to the ST-Link dropped every few minutes for hours; the cause was **a loose GND pin**. The unit's chip is an **STM32F103xE** (512 KiB), not the F103VB of the project files. Backup of the factory firmware (no read protection), firmware 1.1 written, checked and running. The `swd.sh` script gained retries for dropping links, always demanding the verification read-back.
5. **Firmware 1.1 on the cartridges.** Correct write in **a single pass** (about 79 s for 2 MiB). Writing without erasing gives `WRITE TIMEOUT` in 0.4 s and the board keeps answering.
6. **The KY cartridge played a trick.** Same ID (`017E`) as the other cartridge, but the 2 MiB erase left 56,544 bytes dirty: it has **eight 8 KiB sectors** at the start, while the other has a uniform first sector. Neither autoselect nor CFI tells them apart (and the CFI changed value between sessions). Design decision: **always erase in 8 KiB steps and treat ID, autoselect and CFI as information**.
7. **The 2 MiB limit.** Erasing or writing above 2 MiB times out; reading above 2 MiB shows the save SRAM; reading 8 MiB repeats 4 MiB. The diagnosis came from reading the FlashKit cartridge design (74HC139: flash only below 2 MiB, SRAM above, flash A21 to GND). Experiments with writes to `$A13000` and `$A130F1` (through an experimental firmware command, later removed) did not map the flash over the SRAM. Conclusion: **2 MiB is the limit of these cartridges**; it is how they are built, not the dumper.
8. **The written cartridges ran on a real Mega Drive.**
9. **Step-by-step guide** in four languages (soldering, assembly, firmware 1.1, tests, the 2 MiB limit), application 1.3.0 and firmware 1.1.0 published (30,112 bytes) in a release, with the image on the bench unit.
10. Last commit of the day: a warning before writing or erasing above 2 MiB on these cartridges, a clean window title, trimmed documentation (not released yet: it would be 1.3.1).

## Phase 6: passing it on (14 November)

Preparing this material for a person who only has the factory MD Rewriter and the public repositories: the `CLAUDE.md` (instructions for an agent to lead the process, with the findings about the cartridges), this `HISTORY.md`, and translations of both plus the step-by-step guide into Hindi and Urdu.

---

## Project numbers

| Item | Value |
|------|-------|
| Period of the work | 31 October to 14 November 2026 |
| Commits of the work (after the original) | 43 (31 October to 13 November) |
| First contact with the board | 13 November 2026 |
| Firmware 1.1.0 | 30,112 bytes |
| Application | 1.3.0 (1.2.0 on 3 November) |
| 2 MiB read | 10 to 12.5 s |
| 2 MiB erase | 6 to 9 s |
| 2 MiB write with firmware 1.1 | about 79 s (about 90 s with the verification read-back) |
| Cartridges used | 1 original mask ROM (read only), 2 S29GL064N flash (one of them KY) |

## How the work was conducted

- **On paper first, on the bench afterwards.** Almost all the code was born before the board arrived, with a simulated firmware, a chip model and documentation of every decision. When the board arrived, tests were done in increasing order of risk: no cartridge, original read only, flash with firmware 1.0, and only then writing new firmware.
- **Every destructive action needed the "go" at that moment**: writing the microcontroller, erasing or writing a cartridge. No automatic test could write or erase.
- **Backup before everything**: factory firmware read twice and kept in two places; flash cartridge content read before being overwritten.
- **Every finding became a document and a test**: a bench report with hashes and times, a regression in the PC model, a line in the global plan. What was not tested on hardware stayed marked as such.
- **Hypotheses checked with an experiment**: the 2 MiB limit had a probable explanation (the cartridge design) and was checked with reads, register writes and erases before being asserted.
- **Authorship and publication**: the person is the sole author of the commits; publication only when they asked.

## Mistakes and fixes worth remembering

| What went wrong | How it was solved |
|-----------------|-------------------|
| `connect` right after another command ignored | The CLI retries `connect` once; the test waits 0.3 s |
| First write block lost after `detect` | The application holds 250 ms; a firmware fix was proposed (NFW-19), not applied |
| Firmware 1.0 writes half the words on S29GL064N | Firmware 1.1 does DQ7 polling after each word |
| Erasing 64 KiB at once leaves 8 KiB sectors dirty on the KY | Always erase in 8 KiB steps below 64 KiB |
| The rule by autoselect and by CFI got the layout wrong | Autoselect and CFI became information only |
| The SWD link dropped every minute | A loose GND pin on the connector; `swd.sh` repeats the operation and demands a read-back |
| The unit's chip differs from the project (F103xE instead of VB) | `swd.sh` accepts both IDs, backs up the whole flash |
| The suite's "mirror" check failed a flash cartridge with old data above the game | The test fails only when the mirror period is smaller than the game size |
| A blank read counted as a "repeated image" | An erased image never counts as a mirror |
| One bit read wrongly once (not reproduced) | The read-back verification caught it; it stays on by default |
| Assumption that the mapper register would unlock 4 MiB | An experiment showed it does not; the 2 MiB limit was documented and the application warns |

## What stayed open

- Writing or erasing above 2 MiB: only on a cartridge without the FlashKit decoder (for example `MD-Flash-Cart-4MB`).
- SSF2 mapper read, Lock-on, Open EverDrive, EverDrive v2, Magistr-16, MX26L6420: never ran on a cartridge.
- Write-buffer programming (`FLASH_FAST`), save writing on a flash cartridge, a power cut in the middle of a write, unplugging the cable during a write, chips of other brands.
- Fixing the discarded bytes after `detect` in the firmware (NFW-19).
- Header checksum of Rolling Thunder 2 (needs a reference dump).
- Firmware update over USB (needs a bootloader flashed once over SWD).
- A PCB revision (Rd resistors on the USB-C, labelled pads, NRST and BOOT0, ESD, `/UWR` and `/CAS2`) was left out: the new hardware work went to the sister project `oscr_minimal`.

## For whoever repeats this

1. Start with no soldering and no writing: read the empty slot, then an original, twice each.
2. Only then solder the connector, probe, back up, write. In that order.
3. Always measure; this history shows the real unit contradicted the project files (the chip), the model (the sector layout) and the first hypothesis (the 2 MiB limit) at least once each.
4. Record what you measure, with hashes and times. It is what makes the next step safe.
