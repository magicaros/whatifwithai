# What-If Scenario: C64 CP/M 2.2 REU RAM-Disk Driver (and Why Real Hardware Hates It)

**Topic:** Evaluating the feasibility of writing a 512 KB Commodore REU RAM-disk driver (`B:` drive) for Commodore 64 CP/M 2.2 in the VICE emulator, bypassing Commodore's notorious 1983 dual-drive buffer bug, and examining why plugging both cartridges into a real C64 port expander fails electrically.

---

## 1. Initial Concept: Dual-Cartridge REU RAM-Disk for C64 CP/M
**Question:** *In the VICE Commodore 64 emulator, you can attach the CP/M Z80 cartridge and a Commodore REU (RAM Expansion Unit) simultaneously. What if we wrote a RAM-drive driver (`B:` drive) for C64 CP/M backed by the REU—and would this ever work on a real C64 using a cartridge port expander?*

**Key Takeaways:**
* **No I/O Register Collision:** The Commodore CP/M cartridge uses `IO1` (`$DE00` in 6510 space / `$CE00` in Z80 space) to toggle bus control between the Z80 and 6510 CPUs, while the REU uses `IO2` (`$DF00`–`$DF0A`). Their control registers do not overlap.
* **The Z80-to-6510 DMA Trampoline:** Because the Z80 cannot trigger REU DMA transfers directly, a custom CP/M BIOS disk driver writes target sector/bank parameters into shared C64 RAM, flips `$DE00` to wake the 6510, executes an ultra-fast REU DMA copy (`$DF01`–`$DF0A`) into a 128-byte buffer, and hands control right back to the Z80.
* **Emulation vs. Real Hardware Verdict:**
  * **In VICE (or FPGA cartridges like 1541 Ultimate-II+):** 100% feasible and yields a blazing-fast `B:` drive that eliminates agonizing 1541 floppy wait times.
  * **On Real 1983 Hardware (Graveyard Material):** Destined to crash or corrupt data. Both the CP/M cartridge and the REU are **bus-mastering DMA devices** that assert the `/DMA` line. Putting both on an unbuffered passive port expander (like an X-Pander 3) introduces trace capacitance, `/DMA` bus contention, and heavy current draw on the stock C64 power supply.

---

## 2. Fixing Commodore's Infamous 1983 Dual-Drive Bug (`PIP B:=A:*.*`)
**Question:** *Since Commodore's C64 CP/M BIOS was originally built for IEEE-488 dual drives (where `B:` was Drive 1) and lazily adapted for the single-drive 1541—causing infamous directory corruption whenever users ran `PIP B:=A:*.*`—how do we prevent that bug in a custom REU `B:` driver?*

**Key Takeaways:**
* **Root Cause of the 1983 Bug:** Commodore's hasty 1541 adaptation mapped Drive `B:` to the exact same Disk Parameter Header (`DPH`) scratchpad and allocation buffers as Drive `A:`, causing cross-drive `PIP` copies to overwrite directory blocks.
* **Clean BIOS Jump Table Hooking:** Rather than patching inside Commodore's messy 1983 1541 IEC serial routines, a runtime `.COM` installer (`REUDISK.COM`) intercepts the standard **CP/M BIOS Jump Table** (`SELDSK`, `SETTRK`, `SETSEC`, `SETDMA`, `READ`, `WRITE`) at the top of memory:
  * **Drive `A:` (`C = 0`):** Passes straight through to Commodore's original BIOS routines untouched.
  * **Drive `B:` (`C = 1`):** Points `HL` to a completely isolated **Disk Parameter Header (`DPH`)**, **Disk Parameter Block (`DPB`)**, and **64-byte Allocation Vector (`ALV`)** allocated in free high RAM, routing all I/O exclusively to the Z80/6510 REU DMA trampoline.
* **Result:** Completely bypasses the 1541 serial bus and fixes a 43-year-old Commodore bug while giving C64 CP/M a 512 KB solid-state drive.

---

## 3. Proof of Concept Confirmed (`REUDISK.COM` in VICE)
* **Working Implementation:** Tested in `VICE (C64SC)` with both the CP/M cartridge and a 512 KB REU enabled simultaneously. Running `REUDISK.COM` cleanly mounts `B:` without interfering with `A:`.
* **Verified `STAT B:DSK:` Output:**
  * **4096:** 128-byte record capacity
  * **512:** Kilobyte drive capacity
  * **128:** 32-byte directory entries (`0` checked directory entries, `0` reserved tracks)
  * **16:** Records/Block (`256` records/extent, `128` sectors/track)
* **Zero-Corruption Cross-Drive Copies:** Core CP/M system utilities (`MOVCPM.COM`, `PIP.COM`, `SUBMIT.COM`, `XSUB.COM`, `ED.COM`, `ASM.COM`, `DDT.COM`, `LOAD.COM`, `STAT.COM`, `SYSGEN.COM`, `DUMP.COM`, `COPY.COM`, `CONFIG.COM`, and `REUDISK.COM`) copy cleanly from `A:` to `B:` with zero directory scrambling.
* **Persistent Solid-State Mode (`REUDISK K`):** Passing the `K` (*Keep*) parameter (`REUDISK K`) skips formatting the directory track on initialization. When paired with a saved VICE REU image file, `B:` retains all files across cold reboots—turning the REU into a persistent 512 KB solid-state drive for C64 CP/M.

---

## 4. Architecture Summary & Graveyard Epitaph
* **Memory Offset Rule:** All shared RAM pointers account for the CP/M cartridge's `+$1000` address translation (`Z80 $0000` = `6510 $1000`, `Z80 $CE00` = `6510 $DE00`).
* **Interrupt Safety:** The 6510 trampoline disables interrupts (`SEI`) during REU DMA transfers (`$DF01`) and restores status cleanly before returning the bus to the Z80.
* **Graveyard Tombstone (Real Hardware Edition):**
  > **Here Lies the Dual-Cartridge C64 CP/M + REU Rig (1983–2026)**  
  > *"Died of two bus masters fighting over one `/DMA` pin and a 40-year-old power brick."*
