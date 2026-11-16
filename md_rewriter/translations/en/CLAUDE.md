# CLAUDE.md: guide for the agent that accompanies the person on the MD Rewriter project / Mega Drive cartridge reader-writer

This file is for you, the AI agent (Claude Code). Read all of it before acting.

The person who opened this folder has a **factory MD Rewriter 1.0** (PCB V1.0, firmware 1.0) and access to two public repositories: the **MD Rewriter** one (`https://github.com/Epicjoy/md_rewriter`: hardware, Keil/CubeMX firmware, Qt application) and the one for the **KY cartridge reader/programmer** (FlashKit MD family, with the C# programmer code and the cartridge PCB). Ask for the exact addresses if you need them.

The goal is to lead the person, step by step, along a path that has already been walked successfully on a real bench: power up the board, solder the programming connector, back up the factory firmware, install a better firmware (1.1), set up the USB interface to the computer, and read/write Mega Drive cartridges safely. This file gathers what was discovered along the way, including the traps. You may use it to develop a similar project (another dumper, another firmware, another application).

Always answer in the person's language.

---

## 0. How you must behave

### 0.1 Safety rules (always apply, even if the person asks otherwise without understanding the risk)

1. **Never write to or erase an original game cartridge** (mask ROM). It cannot be written; the attempt can only cause damage. For originals: read only. If `detect` does not recognise a flash chip, treat the cartridge as an original.
2. **Before erasing or writing a flash cartridge**, read its current content and save it to a file, unless the person explicitly says the content does not matter. Before writing the board's firmware, back up the factory firmware (section 4.3).
3. **Every destructive action needs the person's "go" at that moment**: writing firmware to the microcontroller (ST-Link), erasing a cartridge, writing a cartridge, writing a save. An earlier "go" for something else does not count. Say what is about to happen in one sentence and wait.
4. **One power source at a time** on the board: either the board's USB cable, or the ST-Link's 3.3 V wire, never both.
5. **Cartridges go in and out only with the USB cable unplugged.** Cartridge and board share the same lines; hot-plugging can destroy either.
6. A soldering iron at about 350 °C burns. Ask the person to work in a ventilated place, wash their hands afterwards (solder may contain lead) and let the iron cool. Discharge static by touching a grounded metal object before picking up the board, and do not hold the board by the cartridge slot.
7. Never ask for or record passwords, tokens or personal data of the person. Never publish anything (commit, push, issue) unless they ask.

### 0.2 Working style

- The person may be a beginner in electronics. Give **one step at a time**, say what they should see at the end of the step and move on only when they confirm. Use short lists and ready-to-copy commands.
- Ask them to describe or photograph what they see (a picture says more than text) and to measure with the multimeter before concluding that a connection is right.
- Always distinguish **measured fact** from **hypothesis**. This file marks what was proven on the bench and what was not. If something on their board differs from this file, **what they measure wins over what is written here**.
- If a step fails, stop and diagnose (section 10). Do not improvise writes to the microcontroller "to see if it works".
- Write tests and test scripts that **cannot** write or erase (see 7.4). Destructive operations only behind an explicit confirmation (`--yes`).

### 0.3 Opening questions (ask before anything else)

1. The computer's operating system (Linux was the tested one; Windows works, with manual commands).
2. Do they want just to **use** the board as it came (firmware 1.0, nothing to solder) or to **install firmware 1.1** (needs 4 pins soldered and an ST-Link V2)?
3. What they have at hand: ST-Link V2 (original or "pen drive" clone), soldering iron, multimeter, 2.54 mm 4-pin header, female Dupont jumpers, cartridges (which ones? flash? originals?).
4. Which flash cartridge: brand, label, photo of the cartridge's PCB. Remember section 8.

---

## 1. What the board is and what it does

The MD Rewriter is a dumper/writer for Mega Drive (Genesis) cartridges: an STM32 microcontroller drives the cartridge slot lines (address, data, control) directly and talks to the computer over USB as a virtual serial port (CDC). The application on the PC sends commands; the firmware executes bus cycles.

Facts about the V1.0 board (EasyEDA Pro project, 2 layers, 108.4 × 56.5 mm, no published schematic; the netlist was read from the PCB):

| Item | Fact |
|------|------|
| Microcontroller | the project files say STM32F103VB (128 KiB flash, 20 KiB RAM). **The first unit measured had an STM32F103xE** (ID `0x414`, 512 KiB flash, 64 KiB RAM). Firmware 1.1 was linked for 128 KiB/20 KiB and runs on the larger chip. **Always probe the person's chip** (4.2) and do not assume. |
| Crystal | 8 MHz, PLL ×9 = 72 MHz, USB derived from the PLL |
| USB | PA11 (D-) and PA12 (D+) with 22 Ω in series; 1.5 kΩ pull-up on D+. Enumerates as `0483:5740` "STM32 Virtual ComPort" (CDC). **There is no DFU.** |
| Power | VBUS → PTC (polyfuse) → 5 V → AMS1117 regulator → 3.3 V (MCU) and 5 V on the cartridge slot |
| LEDs | POWER (3.3 V) and ACT (PC13) |
| Button | RESET (NRST with pull-up) |
| SWD connector | **4 loose holes with no silkscreen**, in a column, under capacitor C10, to the right of chip U1 and above the USB-C connector |
| NRST/BOOT0 | there is no NRST pad; there is no BOOT0 pad (BOOT0 goes to GND through R3, 10 kΩ); PA9/PA10 (the ROM bootloader UART) have no pads |

### Pin map MCU → cartridge slot (confirmed by the firmware for the pins it uses)

| MCU pins | Signal |
|----------|--------|
| `PE0..PE15` | D0..D15 (input with pull-up) |
| `PD0..PD7` | A1..A8 |
| `PD8..PD15` | A9..A16 |
| `PA0..PA6` | A17..A23 |
| `PB4` | /CAS0 (read, `MD_RD`) |
| `PB6` | /CE0 (`MD_CS`) |
| `PB14` | /LWR (write, `MD_WR`) |
| `PB15` | /VRES (cartridge reset) |
| `PB12`, `PB13` | /TIME, M3 |
| `PB5`, `PB7`, `PB8`, `PB9`, `PB11` | /ASEL, /AS, VCLK, /MRES, /CART |
| `PA13`, `PA14` | SWDIO, SWCLK |
| `PC13` | ACT LED |

Not connected to the MCU: `/UWR`, `/CAS2`, `/DTACK`, EDCLK, YS, HSYNC, VSYNC, SL1, SR1. Consequences: the dumper never drives /UWR (FlashKit-type flash cartridges decode only /LWR for word writes, so this works; cartridges that depend on /UWR or /CAS2, such as I2C serial EEPROM, are not served by this board).

There is no level shifter: the MCU is 3.3 V and the cartridge is 5 V. It worked on every cartridge tested (FT data inputs and address outputs), but it is a known design risk.

### Known defects of the V1.0 board (warn about them)

- **The USB-C has no 5.1 kΩ Rd resistors on the CC1/CC2 lines.** A USB-C to USB-C cable plugged into a charger or a modern laptop may **not supply power**. **Use a USB-A to USB-C cable with data wires.** Advanced fix: solder two 5.1 kΩ 0603 resistors between the A5 and B5 pads of the connector and GND (only for people who know how to solder 0603).
- The polyfuse may be 0.1 A on some batches (the BOM says 0.5 A): check on the board; 0.1 A may be too little for MCU + cartridge.
- No TVS/ESD on USB or on the slot, no series resistors on the cartridge lines, no decoupling near the slot's 5 V pins. Handle with care.
- The ACT LED is on PC13 (a pin the datasheet advises against for driving LEDs): it works, dimly.

---

## 2. Required material (check with the person)

| Item | What for | Required? |
|------|----------|-----------|
| MD Rewriter board V1.0 | the device | yes |
| USB cable with data wires, preferably A→C | connect to the computer | yes |
| Computer (Linux tested; Windows possible) | run the application | yes |
| Flash cartridge (S29GL064N, KY/FlashKit type) | write games | to write |
| Original game cartridge | back it up | to read |
| ST-Link V2 programmer (original or USB clone) | install firmware 1.1 | optional |
| 1×4 pin header, 2.54 mm | solder to the SWD pads | with the ST-Link |
| 4 female Dupont jumpers | connect the ST-Link | with the ST-Link |
| Soldering iron 25-40 W, fine tip, 0.6-0.8 mm flux-core solder | soldering | with the ST-Link |
| Desoldering wick, flux pen, isopropyl alcohol, brush | fix and clean | helps a lot |
| Multimeter with continuity test | check pads and joints | with the ST-Link |
| Tape or a "third hand" | hold the board | helps |

With no ST-Link and no soldering the board **already works** with firmware 1.0 to read cartridges and to write some flash ones (with limitations, section 6.1).

---

## 3. Soldering the SWD connector (only to install firmware 1.1)

### 3.1 Where the 4 pads are

Top view, USB-C at the bottom of the board. From top to bottom:

```
   ●  3V3
   ●  SWCLK   (PA14, U1 pin 76)
   ●  SWDIO   (PA13, U1 pin 72)
   ●  GND
 [USB-C]
```

Seen from behind the vertical order is the same (3V3 on top); it only looks mirrored. The pad order was read from the project file; **confirm it with the multimeter before soldering and before connecting the ST-Link**:

1. Board unpowered, continuity mode: one probe on the bottom pad and the other on the metal shell of the USB-C connector. It must beep (that is GND). If it does not, the pad or the board (upside down?) is wrong.
2. Later, with USB plugged in, the top pad measures 3.3 V against GND.
3. SWCLK and SWDIO signals: inferred from the file and later accepted by the ST-Link on the bench. If `probe` (4.2) finds nothing, **swap SWCLK and SWDIO** before suspecting anything else.

### 3.2 Soldering step by step

1. Tape the board down, top side up.
2. Push the 4-pin header through from the top, short pins down, plastic resting on the board, pins straight and perpendicular.
3. Turn the board over. Iron at 330-350 °C, tip clean and tinned.
4. Touch the tip to the pin and the pad at the same time (about 1 s); feed solder to the joint (not the tip) for 1-2 s; remove the solder, then the iron. **Solder only one pin first.**
5. A good joint is shiny, a small volcano shape covering pad and pin. Dull, grey, a ball or with a gap: reheat with a little fresh solder, or remove with the wick and redo it.
6. Check that the header is still straight. Fix it by reheating the first pin while pushing. Then solder the other three (about 3 s each; too much heat lifts the pad).
7. Clean with isopropyl alcohol and a brush; inspect with a magnifier or phone camera: no bridges between neighbours and no gaps.
8. **Multimeter, board unpowered:**
   - GND: beeps to the USB-C shell (as before).
   - Between neighbouring pins (3V3-SWCLK, SWCLK-SWDIO, SWDIO-GND): it must **not** beep. A beep means a solder bridge; remove it with the wick.
   - Push each pin with a fingernail: none may wobble. **A loose GND pin cost hours on the bench**: the link to the ST-Link dropped every few minutes and it looked like a software fault. If anything wobbles, reheat that pin.

### 3.3 Optional USB-C fix (CC resistors)

Two 5.1 kΩ 0603 resistors, one from A5 to GND and one from B5 to GND. Only if the person knows how to solder 0603. Otherwise an A→C cable solves it.

### 3.4 Removing the connector afterwards (if wanted)

Reheat a pin and pull it with pliers, or cut the plastic with side cutters and remove the pins one by one, cleaning the holes with the wick. Never heat a pad for more than a few seconds: it lifts. The connector can stay on the board with no problem.

---

## 4. Installing firmware 1.1 over SWD (about 1 hour)

### 4.1 Why SWD is the only way

The bench proved there is **no USB update**: the board is plain CDC with no DFU interface; the STM32F103 ROM bootloader does not speak USB (UART only, on PA9/PA10, which have no pads), and firmware 1.0 has no "enter bootloader" command. So an ST-Link and the connector of section 3 are required. (Anyone who wants USB updates in the future would have to flash a bootloader once over SWD: this was not done.)

### 4.2 Connect the ST-Link and probe

**Unplug everything first**: board USB cable disconnected and slot **empty**. Connect the wires **by the name printed on the ST-Link**, never by colour or position (clones differ; the "pen drive" ST-Link with 4/6/10 pins does not follow the numbering of the 20-pin original):

| Pad (top to bottom) | Signal | ST-Link wire |
|---------------------|--------|--------------|
| 1 | 3V3 | `3.3V` (only if the board cannot go on USB; see below) |
| 2 | SWCLK | `SWCLK` |
| 3 | SWDIO | `SWDIO` (some clones print `SWIO`) |
| 4 | GND | `GND` |
| none | NRST | leave `RST` loose (the board has no pad) |

The ST-Link `5V` stays unconnected. On the original 20-pin ST-Link: SWCLK = pin 9, SWDIO = pin 7, GND = pin 4 (and 6, 8, 10...), 3V3 = pin 1 (voltage reference only: it does not power the board).

**Power:** the preferred way, which was stable: only `GND`, `SWCLK` and `SWDIO` on the ST-Link, **without** the 3.3 V wire, and the board powered by its own USB cable. If the board cannot go on USB, use the ST-Link's `3.3V` wire with USB unplugged and the slot empty; clones' 3.3 V output is weak, keep tests short. Never both.

Press every Dupont connector with a finger: a loose connection is the most common problem. Only then plug the ST-Link into the computer.

Probe (Linux: `sudo apt install stlink-tools`):

```
st-info --probe
```

Expected on the first unit: `chipid 0x414`, `F1xx_HD`, 512 KiB flash (`0x0200` in register `0x1FFFF7E0`), 64 KiB SRAM. If it is `0x410` with 128 KiB, also valid. If another ID or size shows up, **stop and understand it before continuing**.

If it finds nothing (`chipid 0x000`, `Failed to connect`, "0 programmers"):

1. **GND at the connector first** (a loose pin was the real cause of hours of intermittent failures).
2. Swap SWCLK and SWDIO.
3. Reseat each Dupont; try another ST-Link cable/port.
4. If the link drops now and then, repeat the read/write in a loop with retries, but **always demand the verification read-back**.

### 4.3 Back up the factory firmware (mandatory before writing)

```
st-flash read fw-factory.bin 0x08000000 0x80000     # 512 KiB; use 0x20000 if st-info says 128 KiB
sha256sum fw-factory.bin > fw-factory.bin.sha256
```

Read **twice** and compare. Validate:

- A read that is **all 0xFF or all 0x00** means read protection: **stop**. Do not erase or write. Removing the protection erases the chip and the factory firmware would be lost; the decision belongs to the person, with that clearly warned.
- Healthy firmware has a valid vector table: the first word (initial SP) points into RAM (`0x2000xxxx`) and the second (reset) into flash (`0x0800xxxx`, odd).

On the first unit: identical reads, no protection, 524,288 bytes read, about 18 KiB used, SP `0x20003C30`, reset `0x08000105`. Keep the backup in **two places** (another disk, a USB stick).

### 4.4 Getting firmware 1.1

The original MD Rewriter repository only ships firmware **1.0** (Keil source in `Firmware`/`legacy_firmware` and the `.hex` on the releases page). Two routes:

- **Route A, recommended:** get a ready-made 1.1 binary, if the person has one (with its `.sha256`; check with `sha256sum -c`).
- **Route B:** build firmware 1.1 from the original repository's code by applying the changes of section 6.2, with `arm-none-eabi-gcc` (`sudo apt install gcc-arm-none-eabi make`). Cautions:
  - 1.0 was compiled with Keil ARMCC5 `-O3`. Bus waits are done by software delay (`Delay_nop`), so **timing changes with the compiler and the optimisation**. The GCC build with `-O1` worked on the bench for reading and writing; `-O3` was not measured. Measure (logic analyser) if you change the optimisation, and validate by reading the same cartridge twice.
  - The linker must respect the chip's flash (128 KiB if `0x410`; the 30 KiB 1.1 file fits either), the 20 KiB of RAM and start at `0x08000000` with no bootloader.
  - Do not regenerate the code from CubeMX over the modified code (it wipes the changes).

### 4.5 Writing

Only after the backup is validated, with the person's "go", and with the board's USB plugged in (power from USB only):

```
st-flash write md-rewriter-firmware-1.1.0.bin 0x08000000
st-flash read check.bin 0x08000000 <size of the .bin>      # and compare with the .bin
```

Rules that held on the bench:

- **Verify by reading back** the flash and comparing it with the image. Only then reset.
- With no NRST the reset is done by software (AIRCR). **After writing, unplug and plug the board's USB cable again**: the software reset does not always make the board enumerate on USB.
- If the chip holds something different from the backup you made (another unit? another image?), **stop and make a new backup** before writing. Never write without a backup of what is there.
- To go back to the factory firmware: write `fw-factory.bin` back and read it back/compare.
- If the board stops answering on SWD (it did not happen), the way back is the ROM bootloader with BOOT0 high (take the BOOT0 side of R3 to 3.3 V), which speaks only UART on PA9/PA10, with no pads (pins 68/69 of the LQFP100): it needs fine-pitch wires. That is why backup and verification are non-negotiable.

### 4.6 Check

```
lsusb | grep 0483:5740           # the board shows up
```

and use the connect command of the application/CLI (section 7). Firmware 1.1 prints `FWVER:1.1.0` right after the connect banner.

---

## 5. USB interface to the computer

### 5.1 Permissions and ports

- Linux: the board becomes `/dev/ttyACM0` (or similar). Grant access: `sudo usermod -aG dialout $USER` and **log out and in again**, or install a udev rule (VID `0483`, PID `5740`, `MODE="0660"`, `GROUP="dialout"`, and `ENV{ID_MM_DEVICE_IGNORE}="1"` so ModemManager leaves the port alone). Then re-plug the cable.
- Windows: shows up as a COM port. Install the "STM32 Virtual ComPort" driver if the system asks.
- "permission denied" is serial permission. If the board does not show up: another cable (with data wires) and another port; the POWER LED must be on.

### 5.2 Serial protocol (firmware 1.0, and what 1.1 adds)

Observed on the factory firmware and kept in 1.1 (1.1 only adds). Command packet from the PC: `[cmd] AA 55 AA BB [arguments...]`.

| Command | Meaning |
|---------|---------|
| `0C` | connect. 1.1 adds a second line `FWVER:<version>` to the banner |
| `0D` | detect: the firmware prints `FLASHID:` and a line with the chip (1.1: `FLASHID:%02X%02X`, `CHIPINFO:`) |
| `0A` | read the ROM (argument = size: `01` 512K, `02` 1M, `03` 2M, `04` 4M; 1.1 also `05` = 8M) |
| `1A` | read the save RAM (argument `01` = 32K, otherwise 8K) |
| `0B` | write a ROM block (arguments: block index within the bank, bank) |
| `1B` | write a save block |
| `0E` | erase everything (chip and SRAM) |
| `1E` | erase by size (argument `01`..`06`, or `00` plus a 3-byte address) |
| `2E` | erase one sector |
| `0F` | clear buffers |
| `3A` | 1.1 only: read through the SSF2 mapper (argument = number of 512 KiB pages); **not tested on a cartridge** |

Formats:

- **Read:** the firmware sends text `... DUMP START!!!\r\n`, then N raw 1024-byte blocks (512, 1024, 2048 or 4096 blocks for 512K to 4M; 32 or 8 for save; 8192 for 8M in 1.1), and finally `DUMPER ROM FINISH!!!\r\n` (or `DUMPER RAM FINISH!!!\r\n`).
- **Block write:** the PC sends **1024 raw bytes and then the 7-byte `0B`/`1B` command**. The firmware answers `ADD:0x... WRITE OK\r\n` (ROM) or `... WRITE GK\r\n` (save). 1.1 answers `ADD:0x<word address> WRITE TIMEOUT` when a word was not programmed.
- **Source of truth for the fine semantics** (exact meaning of each argument, bank size, `1E` codes): the 1.0 firmware code in the original repository (`main.c` and `MD.c`). **Check there before implementing**; this table comes from bench observation.

### 5.3 Firmware quirks (learned on the bench, valid for 1.0 and 1.1)

1. **After answering `detect`, the firmware waits 100 ms and only then clears its input buffer.** Bytes sent in that window are lost. In practice: the first write block sent right after `detect` vanishes and the PC waits forever for a confirmation that never comes (`The device did not confirm block 1`). **Fix on the PC side: hold the next command 250 ms after `detect`.** Any tool that talks to the dumper suffers from this.
2. **A `connect` that arrives right after the port was reopened can be ignored.** Retry `connect` once, or wait 0.3 s between consecutive process runs.
3. After flashing firmware, re-plug the USB (section 4.5).
4. Measured speed: 2 MiB read in 10-12.5 s (about 166-214 KiB/s); 8 MiB in about 48 s.

---

## 6. What changed from firmware 1.0 to 1.1 and why

### 6.1 The 1.0 defect with fast flash cartridges

On the bench, with the S29GL064N cartridge: **firmware 1.0 programs only every other word**, because it does not wait for the chip to finish writing each word. No word comes out wrong; half stay blank (`0xFFFF`). Symptom in the application: the write finishes and the verification fails (`Verification failed: ... bytes differ`). It can be worked around by repeating the write of only the missing words (8 passes, about 2 minutes for 2 MiB), but the right fix is the firmware.

### 6.2 Changes in 1.1 (the minimum for a good result)

Start from the 1.0 source and apply:

1. **DQ7 data polling after every programmed word.** AMD programming sequence: `AA`@`555`, `55`@`2AA`, `A0`@`555`, then data@address (word addresses). After each word, read the programmed address until DQ7 equals bit 7 of the written data; if DQ5 rises (and DQ7 still does not match), the chip gave up: send reset (`F0`), stop and answer `WRITE TIMEOUT`. Bound the number of reads. Switch the bus between output and input quickly (direct register access) so polling is not slow. Measured result: correct write in a single pass.
2. **Chip identification (autoselect)**: read words `00`, `01`, `0E`, `0F` in one autoselect session, print `FLASHID:` with 4 hex digits (`017E` on the S29GL064N) and map it in a name table.
3. **Erase in small sectors**: erase in 8 KiB steps (command `30` per sector) below 64 KiB and then in 64 KiB sectors. See the reason in 8.3.
4. 8 MiB read (code `05`), `FWVER:`, `WRITE TIMEOUT` output and, if you want, a CFI query for unknown chips.
5. Optional: write-buffer programming (`25`/`29`, 16 words per operation) was implemented but **not tested on a cartridge** (off by default).
6. Nothing else: keep the rest equal to 1.0 so the old application keeps working.

Times measured with 1.1 (S29GL064N): erase 2 MiB in 6-9 s; write 2 MiB in about 79 s (about 90 s with the verification read).

### 6.3 Testing the firmware without a board

It is worth compiling the real `MD.c` on the PC against a **model of an AMD flash chip** (autoselect, programming with DQ7/DQ6/DQ5, sector erase, a chip that ignores commands while busy, a chip that lies about its layout). It caught real errors before the board arrived (for example DQ5 misread on `0xFF` when an erase finished between two status reads). But the model is only a hypothesis: **only what the board proves counts**.

---

## 7. The PC application

### 7.1 What exists

The original MD Rewriter ships a Qt application (`Software/`). The reference project improved it: it accepts even-sized ROMs up to 4 MiB, converts `.smd`, verifies the write by reading back, shows the chip name, handles `WRITE TIMEOUT`, offers 8 MiB only when the firmware says `FWVER:`, and has a **command line** (`mdrewriter-cli`) made for scripts and agents.

Linux, Qt 6 (tested on Linux Mint 22 with Qt 6.4):

```
sudo apt install build-essential qt6-base-dev qt6-serialport-dev qt6-tools-dev qt6-l10n-tools
cd Software && qmake6 MDRewriter.pro && make -j"$(nproc)" && ./MDRewriter
```

Windows: install Qt and open the `.pro` in Qt Creator, or use the ready-made executable from the releases page.

### 7.2 If the person does not want Qt

A simple Python client with `pyserial` is enough to read: open the port, send `0C AA 55 AA BB` (connect), wait 0.3 s, send `0D AA 55 AA BB` (detect), **wait 250 ms**, send `0A AA 55 AA BB 03` (2 MiB), consume up to `DUMP START!!!`, read `2048 × 1024` bytes and look for `DUMPER ROM FINISH!!!`. Find the banner and the exact text by printing what the board answers (do not guess). To write, follow the format in 5.2 and **only after erasing**.

### 7.3 Recommended CLI contract (use it in the person's project)

- Read-only commands: `ports`, `info`, `detect`, `read-rom`, `read-save`, `verify`, `compare`, `rominfo`.
- Commands that change the cartridge (`write-rom`, `write-save`, `erase`) **refuse** without `--yes` (exit code 4) and have `--dry-run`. Before writing, the CLI checks the chip (`connect`, `detect`, then write): it refuses if the firmware printed no chip ID, if the ROM is larger than the chip, or if the chip needs a command set that only newer firmware sends.
- `read-*` does not overwrite an existing file without `--yes`; it writes atomically.
- `--json` output and distinct exit codes: 0 ok, 1 operation failed, 2 usage, 3 no device, 4 `--yes` missing, 5 file error.
- Write by default **reading back and comparing**. On the bench the verification caught one wrongly read bit once (byte `0xAAB`, `0x04` read as `0x00`, not reproduced): never turn verification off without a reason.
- A **fake firmware** (simulator) for automated tests, with options to reproduce the quirks of section 5.3.

### 7.4 Read-only bench suite

Write (or ask the agent to write) a script that calls the CLI **only through a whitelist of read commands** and refuses `--yes`. Checks that proved useful:

- No cartridge: the board enumerates as `0483:5740`; 10 reconnects in a row; `detect` answers `FFFF` and names no chip; two 512K reads of the empty slot are identical and **all `0xFF`** (proof that the D0-D15 pull-ups work).
- With a cartridge: two identical reads; size; `SEGA` header; header ROM size; header checksum (information, not a failure); the smaller read equals the prefix of the larger; data bits (each bit should be 1 in ~30-40% of the words: none stuck); distinct 32 KiB blocks (a stuck address line gives repeats); two identical save reads.

Reference result: on every unit the empty slot should give a read of all `0xFF` bytes. If it gives anything else, suspect the pull-ups, a short in the solder or a cartridge forgotten in the slot.

---

## 8. Discoveries about the cartridges (read before writing anything)

Tested: one original mask-ROM cartridge (Rolling Thunder 2, Namco), two S29GL064N flash cartridges (one of them from KY Technology) and a 2 MiB game (Metal Gear) written and verified on both. The person who ran the tests confirmed that **the written cartridges ran on a real Mega Drive**.

### 8.1 Original cartridges (mask ROM)

- `detect` shows `FLASHID` `0000`/`00`/`FFFF` and no chip: that is normal. **Never write.**
- A read larger than the cartridge is a **mirror**: reading 2 MiB from a 1 MiB game gives the same image twice (the upper address lines are not decoded). The application does not detect size: the person chooses; a larger read repeats the content. Use the header size (`ROM size`) as a guide.
- The header checksum may not match even on an original (it happened on Rolling Thunder 2: header `BEB6`, sum `11B1`). Two identical reads, healthy data bits and no address aliasing indicate the read is right, but only a reference dump (No-Intro) settles the doubt: **left open**.

### 8.2 KY / FlashKit-type flash cartridges: **the 2 MiB limit**

**Most important result:** the two flash cartridges tested hold **at most 2 MiB of game**, although the chip (S29GL064N) has 8 MiB. Symptoms:

- Erasing or writing above 2 MiB fails: `erase 4M` stops with `A sector did not finish erasing in time` after about 23 s; erasing a sector at a byte address ≥ 2 MiB gives `ERASE TIMEOUT!`.
- Reading above 2 MiB shows the **save SRAM**, not the flash: every other byte is `0xFF` and the others repeat the save text (on the KY cartridge the game name showed up: `ff 52 ff 41 ff 47 ff 4e ...`). Reading 8 MiB shows the same 4 MiB twice.

**Cause (from the FlashKit cartridge reference design, `cart-md-v1.2.pcb`, and confirmed by behaviour):** the cartridge uses a 74HC139 decoder in which the flash /CE is only active when the 2 MiB address line is low; with it high the same decoder selects the SRAM. The flash's A21 pin is tied to GND and the connector's A21/A22 lines go nowhere. **It is how the cartridge is built, not a defect of the dumper or the firmware.**

We tried to map the flash over the SRAM window with the original programmer's register writes (`$A13000` = `0x0000`/`0x0001`/`0xFFFF`, `$A130F1` = `0`/`1`), with a mapper write cycle (`/TIME` and `/LWR` low, `/CE0` high): **nothing changed** (only one RAM byte changed, as a RAM write would do). So there is no register in these cartridges, and **2 MiB is the limit**.

What fits: almost every Mega Drive game (most are 512 KiB to 2 MiB). What does not fit: games larger than 2 MiB (for example Super Street Fighter II, 4 MiB, with a mapper). For those another cartridge is needed: a plain 4 MiB flash cartridge without that decoder (for example the open project `MD-Flash-Cart-4MB`, with MX29LV320 or MX29L3211) or a cartridge with a mapper. **Your agent must warn before writing or erasing above 2 MiB on these cartridges.**

### 8.3 Two cartridges with the same ID and different sector layouts

The two cartridges answer `FLASHID:017E` and the same CFI query, and look alike in everything, but:

- One has a **uniform first 64 KiB sector** (a single sector-erase command at `0x1000` words clears the 64 KiB).
- The other (KY) has **eight 8 KiB sectors** at the start (boot sectors): to clear the same 64 KiB you need eight erases. If the firmware erases the 64 KiB at once, 56,544 bytes stay programmed, and the next write gives `WRITE TIMEOUT` at the first dirty sector.
- The extra autoselect codes (cycles `0E/0F`) gave `017E1001` and `017E1000` on the two, but a rule based on them was wrong, and the CFI sector words (`2C..34h`) gave the same value on both and **changed between sessions** (`...7F...` and then `...7E...`). **Practical conclusion: do not trust ID, autoselect or CFI to choose the erase layout. Always erase in 8 KiB steps in the first 64 KiB.** It works on both, at the cost of a little time.
- Show the user the chip's extra codes (`Chip codes 017E1001`) and the CFI query as **information**, never as a decision.

### 8.4 Flash writing rules

- Flash only changes from 1 to 0. **Erase before writing.** Writing over data without erasing gives `WRITE TIMEOUT` in about 0.4 s and the board keeps answering: it is the safe, expected failure.
- Writing **identical data** over already-written data gives no error (nothing to change). To prove timeout detection, write different data (for example the inverted image).
- A 2 MiB erase does not erase the rest of the chip. What is above stays there (on one of the cartridges there was old data).
- The S29GL064N cartridges tested have **save RAM with no battery** on one of them (and FRAM on PCB 1.4): `read-save` works; **do not test `write-save` on a flash cartridge** without understanding the map, because the save window may coincide with flash space and a write there would be a bus cycle inside the chip.
- A dump of the cartridge before erasing is the safety net. Keep it.

### 8.5 Chips and commands (summary useful for extending support)

| Chip | ID (manufacturer+device) | Commands |
|------|--------------------------|----------|
| S29GL064N (Spansion/Cypress) | `01 7E` | AMD, unlock `555h/2AAh` |
| MX29LV640EB/ET | `C2 CB` / `C2 C9` (`C9` not confirmed) | AMD 555h |
| M29W640D/G | `20 DF`, `20 DE`, `20 7E` | AMD 555h |
| SST39VF6401B/6402B | `BF 236D` | AMD 555h, block erase with `50h` |
| M29W320DT/DB, M29W320ET/EB | `20 CA/CB/56/57` | AMD 555h; "T" = boot at the top (last 64 KiB in eight 8 KiB sectors) |
| MX29LV320T/B | `C2 A7/A8` | AMD 555h |
| MX29L3211 | `C2 F9` | AMD with unlock `5555h/2AAAh`; no page program |
| Intel PA28F400 | `89 70/71` | Intel/Sharp family (`FFh`, `90h`, `40h`, `20h`+`D0h`, `70h`/`50h`) |

Only the S29GL064N was **proven on hardware**. The rest is in the PC model and in datasheet reading: **read the chip's datasheet before the first write**.

### 8.6 Not tested (tell the person)

SSF2 mapper read (command `3A`), Sega Lock-on, Open EverDrive, EverDrive v2, Magistr-16, MX26L6420, serial I2C/SPI saves (`/UWR` and `/CAS2` missing on the board), Master System/Game Gear (different connector), writing with a power cut in the middle, write-buffer programming, flash cartridges of other brands, `write-save` on a flash cartridge, writing above 2 MiB on cartridges that lack the FlashKit decoder.

---

## 9. Full script for the agent to lead

Advance only when the person confirms each stage.

**Stage 1. Get to know them (0.3).** Ask the opening questions.

**Stage 2. Use it as it came (no soldering).**
1. Install the application (7.1) and the serial permission (5.1).
2. Power the board (A→C cable) with no cartridge: the POWER LED lights; `lsusb` shows `0483:5740`.
3. `Connect` in the application (or `mdrewriter-cli info`): "Device identified" and firmware `1.0`.
4. **Empty slot:** `detect` finds no chip; reading the empty slot gives only `0xFF`.
5. Unplug USB, insert an original cartridge (label toward the "CART FRONT" arrow, straight, no rocking), plug USB back, `detect`, read 2 MiB twice and compare. Different results = bad contact: clean the cartridge contacts with isopropyl alcohol and reseat.

**Stage 3. Soldering and checking (section 3)**, only if installing firmware 1.1.

**Stage 4. SWD: probe, back up, flash 1.1, check (section 4).**

**Stage 5. Test the unit (table below), in this order.** If one fails, stop and go to section 10.

| # | Test | What should appear |
|---|------|--------------------|
| 1 | Version, no cartridge | `Device identified.` and firmware `1.1.0` (or `1.0`) |
| 2 | Empty slot | no chip; read with only `0xFF` |
| 3 | Original, read only, 2 MiB | dump made in about 12.5 s; `detect` shows ID `0000`/unknown (not flash) |
| 4 | Read again and compare | no differences |
| 5 | Flash cartridge: detect | `S29GL064N ... Supported` and `Chip codes ...` |
| 6 | **Back up what is on the flash** | ROM 2 MiB and save 32K in a file |
| 7 | Erase 2 MiB | `Erase complete` in 6-9 s; new read only `0xFF` |
| 8 | Write ROM (≤ 2 MiB) with verification | `Game written` and `Verified` (about 90 s for 2 MiB) |
| 9 | Read and compare with the original file | identical |
| 10 | Save: read twice and compare | identical |

**Stage 6. Test on a real Mega Drive** (it is the final proof).

**Stage 7. Record.** Help the person write down what they measured (backup, SHA-256, times, what failed), in the style of a bench report, so they can share it.

---

## 10. Diagnosis

| Symptom | What to do |
|---------|------------|
| The board does not show up on the computer | USB cable with data wires (A→C); another port; POWER LED on? Without Rd on the USB-C, C→C cables may not power it (section 1) |
| `permission denied` | `dialout` group + new login, or udev rule (5.1) |
| `st-info` does not find the chip (`chipid 0x000`, `Failed to connect`) | GND at the connector first; swap SWCLK/SWDIO; reseat the Dupont connectors; change ST-Link cable/port; check continuity and solder bridges (3.2) |
| Backup read all `0xFF`/`0x00` | Read protection. **Stop.** Do not erase or write (4.3) |
| After flashing the board does not show on USB | Unplug and plug the board's USB cable again |
| First write block is not confirmed | Hold 250 ms after `detect` (5.3) |
| `connect` gets no answer right after another command | Repeat `connect` once / wait 0.3 s |
| `detect` says unknown chip | Probably an original (mask ROM) or a chip outside the table: **do not write** |
| `did not finish erasing` / `ERASE TIMEOUT` | Not flash, damaged, **or more than 2 MiB was requested** (8.2), or the erase did not walk through the 8 KiB sectors (8.3) |
| `WRITE TIMEOUT` | The area was not erased first; erase and write again (8.4) |
| Verification fails | Clean contacts, another USB cable, repeat erase+write. With firmware 1.0 on an S29GL064N it is the defect of section 6.1 |
| Two reads of the same cartridge differ | Bad contact: clean with isopropyl alcohol and reseat |
| A read larger than the cartridge repeats content | It is a mirror (8.1) or the flash cartridge's SRAM window (8.2): normal |
| Written game does not start on the console | Confirm it is a valid ROM (`SEGA` header), at most 2 MiB, and that verification passed |

---

## 11. If you will develop a similar project (another dumper, another firmware)

Lessons that hold regardless of the hardware:

1. **Read first, write afterwards.** Make the dumper read with 100% repeatability (two identical reads, equal SHA-256, empty slot all `0xFF`) before any write.
2. **Back up the firmware before anything**, double read, check read protection, copy in two places.
3. **The bench rules.** The PC model catches a lot, but it got the sector layout, the 2 MiB limit and the board's real chip wrong. Write "not tested on hardware" until you measure.
4. **Never decide by ID** what can be done safely in any case (such as erasing in 8 KiB sectors).
5. **Always verify by reading back.** It is cheap and caught real errors.
6. **Probe before assuming** (MCU ID, flash size, firmware version).
7. **Separate what reads from what writes**, in tools and test scripts.
8. **An intermittent connection is almost always GND or solder.** Before blaming software, wiggle the pins.
9. When the hardware does not allow it (2 MiB limit, USB-C without Rd), **document the limit** instead of insisting in software.
10. Keep every measurement (times, hashes, IDs) in a report. Whoever comes next saves weeks.
11. If you design a new board: put 5.1 kΩ Rd resistors on the USB-C, labelled pads and NRST and BOOT0 pads, TVS on USB, series resistors on the cartridge lines, decoupling near the slot's 5 V, `/UWR` and `/CAS2` connected, a 5 V/3.3 V level shifter if the cartridge requires it, and a USB bootloader (DFU) so the ST-Link is not needed.

---

## 12. Reference repositories that were useful

Clone the ones you need (in their own folder, outside the project) and **read the code before trusting this file**: where they disagree, what the person's bench shows wins. Each licence is different (GPL-3.0, MIT, CERN OHL): respect them when reusing code or designs.

| Repository | What it was useful for |
|------------|------------------------|
| `https://github.com/Epicjoy/md_rewriter` | The original MD Rewriter: board project (EasyEDA), firmware 1.0 (Keil/CubeMX), Qt application, release 1.0 with the factory `.hex`. Starting point |
| `https://github.com/krikzz/flashkit` | The original FlashKit MD programmer (Krikzz): C# source (`Cart.cs`, `Device.cs`, `Form1.cs`), the cartridge PCB (`cart-md-v1.2.pcb`, P-CAD) and the CPLD. It was the source of the 74HC139 decoder that explains the 2 MiB limit and of the `$A13000` SRAM-enable register |
| `https://github.com/MiGeRA/FlashKit-MD-Plus` | FlashKit fork with more chips (Intel PA28F400, MX29LV320, MX29L3211 with `5555h/2AAAh` unlock), timings and mappers (Magistr-16, EverDrive-MD v2). Basis for the alternative command routines |
| `https://github.com/MiGeRA/MD-Flash-Cart-4MB` | Open 4 MiB flash cartridge (KiCad, MX29L3211/MX29LV320, CERN OHL): an alternative without the 2 MiB decoder and a test cartridge |
| `https://github.com/joeyparrish/flashkit-md-py` | Python client (pyserial) for the FlashKit: separates image size from chip size, uses CFI to validate communication. A good model for a CLI and for the CFI query |
| `https://github.com/grantek/fkmd` | Independent Go implementation (read, ROM write, RAM): a cross-check of sequences and protocol |
| `https://github.com/X-death25/MD_Dumper` | Open USB-C dumper with STM32 (firmware, GUI/CLI software, `gameslist.csv`, chip tables): the best source of per-chip sequences (SST `50h`, Sharp, Toshiba), mappers (SSF2, Lock-on, WaterMelon, Open-EverDrive) and serial saves. Older variants: `https://github.com/X-death25/MD-Dumper_old_stm32F4`, `https://github.com/leolobato/MD_Dumper_V2`; graphical interface: `https://github.com/X-death25/PyMD-Dumper` |
| `https://github.com/krikzz/open-ed` | Open EverDrive cartridge: mapper (`A130E0h`, `CTRL_REG` at `A13000h`), and the warning that the programmer needs a 10 kΩ pull-up on /VRES (pin B27) |
| `https://gitlab.com/doragasu/mw-prog` (plus `mw-mdma-bl`, `mw-mdma-fw`, `mw-mdma-cli`) | Programmer for MegaWiFi cartridges: a reference for cartridges with their own mapper. Not cloned or tested |
| `https://github.com/sanni/cartreader` (Open Source Cartridge Reader) | Multi-console cartridge reader on Arduino Mega; basis of the sister project below and a source of ideas for flash chip detection and a games database |
| `https://github.com/legendaryredfox/oscr_minimal` | Sister project of the author of this material: reader/writer of SNES, NES, Famicom, Mega Drive and GBA cartridges on an Arduino Mega shield, based on the OSCR; collects the bench findings about Mega Drive cartridges and plans support for cartridges up to 8 MiB |

About the KY cartridge: Krikzz sells the FlashKit; KY Technology sells a programmer and compatible cartridges. No source showed a KY-exclusive protocol: "KY support" means writing and reading the chips and mappers that appear in those cartridges. KY's own programmer (FT245R + CPLD) is a different device and out of scope.

## 13. What this project improved over the original MD Rewriter

If the person asks what changes compared with `Epicjoy/md_rewriter`, summarise it like this (details, with what was tested on the unit and what was not: `https://github.com/legendaryredfox/md_rewriter/blob/main/docs/improvements/en.md`):

- **Application** rebuilt in layers: ROM of any even size, `.smd` conversion, read-back verification after writing, chip name, Cancel button, warnings before destructive operations, light or dark theme and a built-in guide (F1).
- **Command line `mdrewriter-cli`**: `--json` output, fixed exit codes, mandatory `--yes` to write or erase.
- **Firmware 1.1** (GCC and `make`): writes the S29GL064N in one pass (1.0 wrote only every other word), erases in 8 KiB sectors, names more chips, prepares 8 MiB.
- **Tests**: about 150 for the application, the real `MD.c` against a flash chip model and a read-only bench suite.
- **Tools**: SWD with backup and verification, a repair tool for cartridges written by firmware 1.0, a `.deb` package.
- **Documentation** in six languages (pt-BR, en, ru, zh-Hans, hi, ur): README, step-by-step guide, protocol, CLI, cartridge support, bench results.
- **Findings**: the chip is an STM32F103xE, there is no USB update, KY / FlashKit flash cartridges hold at most 2 MiB.
- **Did not change**: the board is V1.0; several extra chips have only passed PC tests (◇ mark in the full list).

Good work. When in doubt, ask the person and measure.
