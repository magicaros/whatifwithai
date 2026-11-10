# Mini What-If Follow-Up: PowerPC Macintosh Emulation via SheepShaver on Core 1 vs. Native Emu68 PPC

**Topic:** Evaluating the feasibility and trade-offs of running SheepShaver (PowerPC Mac OS 8.5–9.0.4) inside the isolated ARM64 Linux core versus waiting for native Amiga-side PowerPC Mac emulation via Emu68's upcoming PowerPC engine.

---

## 1. Pivoting from x86 Bridgeboard to PowerPC Macintosh (SheepShaver)
**Question:** *Using the same asymmetric multiprocessing (AMP) architecture with an isolated ARM64 Linux core, what if we run SheepShaver instead of x86 emulation—allowing Classic PowerPC Mac emulation instead of being capped at 68k Mac emulation via native Amiga ShapeShifter?*

**Key Takeaways:**
* **Breaking the 68k Ceiling:** Native Amiga Mac emulators like ShapeShifter and Fusion rely on Emu68's 68k JIT, capping them at 68040 Macintosh hardware (Mac OS 8.1 max). Offloading **SheepShaver** to Core 1 under Linux unlocks PowerPC 604/G4 emulation capable of running **Mac OS 8.5 through Mac OS 9.0.4**.
* **Zero Performance Drain on AmigaOS:** Running SheepShaver on Core 1 with 512 MB to 1 GB of the Raspberry Pi 4's upper physical RAM (above the 2 GB boundary) leaves 100% of Core 0 and Amiga Fast RAM untouched.
* **Unified Media & Networking:**
  * **Display:** SheepShaver renders to a virtual framebuffer in shared memory mapped directly into Emu68's VideoCore RTG screen list for instant `Left Amiga + M` switching.
  * **Audio & Files:** Linux handles ALSA/PulseAudio mixing (streamed to Paula/AHI), `slirp`/TAP networking over Pi Wi-Fi/Ethernet, and seamless file sharing via SheepShaver's "Unix Root" folder bridged to AmigaOS.
* **Practical Constraints:**
  * SheepShaver lacks MMU emulation, capping guest OS support at **Mac OS 9.0.4** (Mac OS 9.1, 9.2.2, and Mac OS X require heavier full-system emulation via `qemu-system-ppc`).
  * Requires an ARM64-enabled SheepShaver fork (such as `kanjitalk755/macemu`) so dynamic recompilation (JIT) is active rather than slow CPU interpretation.
  * Requires graceful shutdown signaling over the shared memory bridge before Amiga power-off to prevent HFS/HFS+ volume corruption.
* **Community Appeal:** Recreates the coveted Phase5 BlizzardPPC / CyberStormPPC dual-OS workstation experience (AmigaOS + PowerMac) purely in software on existing PiStorm hardware.

---

## 2. Architectural Comparison: Linux Core SheepShaver vs. Native Emu68 PPC
**Question:** *What are the pros and cons of running SheepShaver on the isolated Linux core versus running native PowerPC Mac emulation on the Amiga side once Emu68's PowerPC emulation core is completed and public?*

### Side-by-Side Comparison

| Feature / Dimension | SheepShaver on Isolated ARM64 Linux (Core 1) | Native Amiga-Side PPC Mac Emulation (Emu68 PPC) |
| :--- | :--- | :--- |
| **Software Readiness** | **Ready today.** Mature ARM64 Linux builds exist and boot Mac OS 7.5.2–9.0.4 immediately. | **Requires new Amiga ports.** No WarpOS-native PowerPC Mac emulator exists; one must be ported or written first. |
| **CPU Execution Speed** | **Very fast.** Uses SheepShaver's PowerPC-to-ARM64 JIT engine on an isolated Cortex-A72 core. | **Blazing fast.** Emu68's bare-metal PPC JIT (~1.8 PPC instructions/cycle) runs with zero OS kernel overhead. |
| **Memory Capacity** | **Unrestricted (2 GB+ on Pi 4).** Uses upper physical RAM completely invisible to AmigaOS. | **Capped by 32-bit Amiga space.** Shares the lower ~1.5–2 GB Zorro III Fast RAM pool with AmigaOS. |
| **AmigaOS Integration** | **Bridge-based.** Needs custom shared-memory RTG framebuffer, AHI audio stream, and filesystem bridge. | **Native.** Uses standard Picasso96/CyberGraphX RTG screens, AHI audio, and AmigaDOS volumes directly. |
| **System Stability** | **Crash-proof isolation.** A Mac OS 9 freeze only affects Core 1; AmigaOS keeps running untouched. | **Shared address space risk.** Without AmigaOS memory protection, a rogue Mac pointer can crash the Amiga. |
| **Setup Complexity** | **Higher upfront setup.** Requires an AMP Device Tree, minimal Linux rootfs, and custom bridge drivers. | **Plug-and-play.** Launches directly from Workbench alongside standard WarpOS/PowerUP software. |

---

### Pros & Cons Breakdown

#### Option A: SheepShaver on Isolated ARM64 Linux (Core 1)
* **Pros:**
  * **Immediate availability:** Compiles from existing open-source ARM64 repositories without waiting for a WarpOS port.
  * **Access to unreachable Pi 4 RAM:** Claims 512 MB–1 GB above the 2 GB mark without shrinking Amiga Fast RAM.
  * **Hardware crash isolation:** Cooperative-multitasking Mac OS 9 crashes cannot take down AmigaOS.
* **Cons:**
  * **Custom bridge engineering:** Requires maintaining shared-memory video, audio, input, and file bridges between Linux and Emu68.
  * **Boot overhead:** Adds a few seconds of Linux boot latency and a second OS filesystem to maintain on the SD card.

#### Option B: Native Amiga-Side PPC Mac Emulation (via Emu68 PPC)
* **Pros:**
  * **Bare-metal efficiency:** Runs on a secondary ARM core via Emu68's `powerpc.library` with hardware cache coherency and zero Linux syscall overhead.
  * **Seamless Workbench workflow:** Uses native `VideoCore.card` RTG, AHI, Amiga TCP/IP stacks, and hardfiles out of the box.
* **Cons:**
  * **No existing WarpOS Mac emulator:** Unlike 68k ShapeShifter, no PowerPC Mac emulator was ever released for WarpOS (`powerpc.library`).
  * **Still capped at Mac OS 9.0.4:** Emu68's WarpOS engine targets user-space PPC execution without full Mac hardware/MMU virtualization, so it shares SheepShaver's OS ceiling.
  * **Competes for 32-bit Fast RAM:** Every megabyte given to the virtual PowerMac reduces RAM available to AmigaOS, WHDLoad, and Emu68's JIT cache.

---

## 3. Verdict
* **Choose Isolated Linux + SheepShaver** if you prioritize **immediate software readiness, crash isolation, and utilizing excess upper RAM** on a 4 GB or 8 GB Raspberry Pi 4.
* **Choose Native Amiga Emu68 PPC** if a developer ports SheepShaver to **WarpOS (`powerpc.library`)**, trading upper-RAM access and crash isolation for **simpler setup, bare-metal speed, and native Workbench integration**.

---

## 4. Hardware Survival Verdict & Graveyard Epitaph
* **Real Hardware Verdict:** **Hardware Safe, Filesystem Fragile.** Your Amiga motherboard and Raspberry Pi are completely safe, and a Classic Mac OS 9 freeze stays quarantined inside Core 1 while Workbench keeps humming along on Core 0. However, yanking the Amiga's power switch without signaling Linux to unmount the HFS+ volume first will leave your virtual Macintosh begging for Disk First Aid on the next boot.
* **Graveyard Tombstone (Classic Mac Edition):**
  > **Here Lies a Virtual PowerMac 9500 Trapped Inside Core 1 (2026)**  
  > *"Died of a cooperative-multitasking Type 11 Bomb in Mac OS 9.0.4—leaving a smiling Finder icon frozen in upper RAM while AmigaOS didn't even notice."*

