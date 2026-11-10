# What-If Scenario: Bare-Metal Amiga ROM Replacement with Stripped-Down EmuTOS & ANSI TUI

**Topic:** Exploring the feasibility of replacing the stock Amiga Kickstart ROM with an open-source, command-line operating system featuring a FAT ROMdisk, ANSI/VT100 TUI support, and direct hardware takeover for games.

---

## 1. Initial Concept: Custom ROM Booting CP/M-68K from a ROMdisk
**Question:** *What if we wanted to make an Amiga alternative ROM replacement with CP/M-68K booting straight from a ROMdisk?*

**Key Takeaways:**
* **Technical Feasibility:** Completely feasible. A standard 512 KB Kickstart EPROM replacement (such as a 27C400) easily holds the bootstrap code, CP/M-68K kernel (<32 KB), and ~480 KB of read-only ROMdisk storage.
* **Cold-Boot Hardware Bring-Up:** Without Kickstart libraries, the custom ROM must handle early 68000 initialization—jumping to high ROM space (`$F80000`–`$FFFFFF`), toggling the `OVL` bit on CIA-A (`$BFE001`) to expose Chip RAM at `$000000`, and configuring Agnus/Denise for a basic monochrome bitplane display.
* **Custom CP/M BIOS:** Requires implementing minimal 68000 routines for console output (software 8×8 bitmap font blitting), CIA-A keyboard reading, and 128-byte logical sector reads mapped directly to ROM addresses.
* **Precedents:** Projects like **DiagROM** and **EmuTOS** already prove that third-party bare-metal ROMs can initialize stock Amiga hardware reliably from a cold boot.

---

## 2. Pivoting to a Stripped-Down, GUI-Less EmuTOS
**Question:** *Respecting licensing, what if we built our custom Amiga ROM replacement on a stripped-down EmuTOS that retains the GEMDOS part (without the GUI) instead of CP/M-68K, giving us a more MS-DOS-like OS base?*

**Key Takeaways:**
* **Built-In "No-GUI" Build Target:** EmuTOS already supports compile-time flags (`localconf.h`) to completely exclude the **AES** (window manager), **VDI**, and **EmuDesk** GUI, booting straight into **EmuCON**—a fast, MS-DOS-style command-line shell (~128 KB total footprint).
* **True Hierarchical Filesystem (FAT):** Unlike CP/M-68K's flat filesystem, GEMDOS natively speaks **FAT12 and FAT16**. The ROMdisk image (`A:` or `C:`) can be generated on any modern PC using `mkfs.fat` and `mtools`, then wired into EmuTOS's `Rwabs` BIOS block driver.
* **Existing Amiga Hardware Layer:** EmuTOS maintains an official `amiga` target that already includes cold-boot video initialization, CIA-A keyboard handling, and A600/A1200 IDE support.
* **Clean Open-Source Licensing:** Licensed under **GPLv2** with zero proprietary Commodore or Atari code. Custom ROM binaries can be freely distributed or sold on physical EPROMs as long as modified EmuTOS source code is made available, while standalone tools on the ROMdisk remain separate works ("mere aggregation").

---

## 3. Adding a VT100/ANSI Terminal Layer for TUI Applications
**Question:** *Could we eventually pair that with a VT100/ANSI terminal emulator running the command line so we can build TUI applications easily?*

**Key Takeaways:**
* **Upgrading from VT52 to ANSI:** Extending EmuTOS's existing VT52 console state machine (`bios/vt52.c`) to parse standard **VT100/ANSI `ESC [` sequences** unlocks cursor positioning, inverse video, and color attributes without heavy GUI overhead.
* **4-Color Bitplane Mode:** Configuring Denise/Agnus for a 2-bitplane 640×200 or 640×256 screen provides **4 distinct colors** using only ~32 KB of Chip RAM, paired with an embedded **CP437 font** for crisp single- and double-line box-drawing frames.
* **Lightweight TUI Ecosystem:** Developers can compile lightweight C TUI libraries, dual-pane file managers (Norton Commander style), hex editors, or text editors (like `MicroEMACS` or `Kilo`) into tiny `.TOS` executables.
* **Serial Mirroring:** Console I/O can also be routed to the Amiga's RS-232 port (`AUX:`), turning the system into an instant-on retro BBS terminal or headless serial workstation.

---

## 4. Direct Hardware Takeover for Games & Demos
**Question:** *Could we still have software launch from the command line, bypass the OS, and talk to the Amiga hardware directly for things like games?*

**Key Takeaways:**
* **Unrestricted Custom Chip Access:** Because GEMDOS is single-tasking and runs without memory protection, any `.TOS` executable launched from EmuCON can directly read and write the Amiga's custom chip registers (`$DFF000`–`$DFF1FE` for Agnus, Denise, and Paula).
* **Zero Multitasking Overhead:** Unlike standard AmigaOS (`exec.library`), there are no background tasks competing for DMA or interrupts. Disabling OS timer/keyboard interrupts gives a game 100% control of the machine.
* **Clean Return to the Shell:** By saving the initial Copper list pointer, interrupt vectors, and DMA flags (`DMACONR`) on startup, games and demos can restore them upon exit and drop cleanly back to the ANSI command prompt.
* **Maximum Free Chip RAM:** With the minimal OS and console using only a few dozen kilobytes, virtually all Chip RAM remains available for game graphics, sprites, and ProTracker `.MOD` audio, while still letting developers use GEMDOS (`Fopen`, `Fread`, `Malloc`) to load assets before taking over the hardware.

---

## 5. Hardware Survival Verdict & Graveyard Epitaph
* **Real Hardware Verdict:** **100% Real-Hardware Safe.** Unlike dual-cartridge bus fights, a single 27C400 EPROM sitting in the Amiga's Kickstart socket draws standard power and plays nicely with Gary, Agnus, and Denise—provided you remember to byte-swap the binary before burning.
* **Graveyard Tombstone (User Error Edition):**
  > **Here Lies a Freshly Burned 27C400 Kickstart EPROM (2026)**  
  > *"Died of a solid black screen because someone forgot to byte-swap `ABCD` to `BADC` before hitting Program."*

