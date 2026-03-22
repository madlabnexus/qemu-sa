# QEMU-SA Phase 1: Sound Revolution — Implementation Guide

## Who I Am

I'm Savant (username: madlabn). I'm the project director for QEMU-SA, not a C programmer. I understand the architecture and can follow along, but I need you to do the actual coding. I'll review your work, approve changes, and verify builds. You write the code, I tell you what to build.

## Project

QEMU-SA (Sound & Acceleration) — a QEMU fork for retro PC gaming, DOS through Windows XP.
- **Repo:** github.com/madlabnexus/qemu-sa (project docs)
- **Source:** ~/Projects/qemu-upstream (QEMU 9.2.4 + qemu-3dfx patches applied)
- **Build dir:** ~/Projects/qemu-build (out-of-tree, binaries here)
- **License:** GPL v2.0

## Machine

- **Notebook:** Lenovo ThinkPad P16v Gen 2 ("archp16v")
- **OS:** EndeavourOS (Arch Linux), GNOME 49.5, Wayland, kernel 6.19.8-arch1-1
- **GPU:** NVIDIA RTX 3000 Ada (proprietary 590.48.01) + Intel Arc iGPU
- **Terminal:** Ghostty
- **Editor/AI:** Claude Code CLI + VS Code

## What's Already Done

- Phases 1–3 of notebook setup complete (KVM, IOMMU, build toolchain, all deps)
- QEMU 9.2.4 cloned and built successfully with qemu-3dfx patches
- All QEMU-SA audio dependencies installed: fluidsynth, munt/libmt32emu, libcue
- Last Timeshift snapshot: `23-qemu-sa-first-build-ok`

**Build verified:**
```
./qemu-system-i386 --version
QEMU emulator version 9.2.4 (v9.2.4-dirty)
  featuring qemu-3dfx@b2995afed2-18:11:05 Mar 18 2026 build
```

**Configure command that works:**
```bash
cd ~/Projects/qemu-build
../qemu-upstream/configure \
  --target-list=i386-softmmu,x86_64-softmmu \
  --enable-kvm --enable-opengl --enable-sdl --enable-gtk \
  --enable-alsa --enable-slirp --audio-drv-list=pa,alsa,jack \
  --extra-cflags="-O2" --disable-werror
make -j$(nproc)
```

**Notes:**
- GCC 15.2.1 requires `--disable-werror` (harmless const-qualifier warnings)
- `python-distlib` needed for QEMU's meson venv setup
- qemu-3dfx applied via: `rsync -r` overlay + `patch -p0`

## Directory Layout

```
~/Projects/
├── qemu-sa/           ← Project docs repo (github.com/madlabnexus/qemu-sa)
├── qemu-upstream/     ← QEMU 9.2.4 source + qemu-3dfx patches (WE EDIT HERE)
├── qemu-build/        ← Out-of-tree build directory (WE BUILD HERE)
└── qemu-3dfx/         ← kjliew/qemu-3dfx repo (patches + wrappers, reference only)
```

---

## Phase 1 Plan: 12 Sessions

We're implementing three components: Nuked-OPL3 (FM synthesis), MPU-401 (MIDI interface), and MIDI backends (Munt, FluidSynth, host passthrough). Each session = one focused task = one commit. Walk me through each session step by step — explain what you're doing before you do it, show me diffs before applying, and always verify compile.

### Workflow Rules

1. **One device, one file, one commit.** Never mix components.
2. **Explain before editing.** Tell me what you're about to change and why before touching any file.
3. **Show diffs before applying.** I review, then approve.
4. **Build after every change.** `cd ~/Projects/qemu-build && make -j$(nproc)` — must compile clean.
5. **Commit convention:** `feat(opl3):`, `feat(mpu401):`, `feat(midi):`, `build:`, `vendor:` prefixes.
6. **Never `git add .`** — always add specific files.
7. **Timeshift snapshots** after each component is complete (I'll do these manually).
8. **If something breaks,** explain what went wrong before trying to fix it.

### Commit Messages

```
vendor: add Nuked-OPL3 library (opl3.c, opl3.h)
feat(opl3): add Nuked-OPL3 QEMU device wrapper
feat(opl3): replace YM3812 with Nuked-OPL3 in SB16 device
build: add nuked-opl3 to hw/audio/meson.build
feat(mpu401): add MPU-401 UART mode ISA device
feat(midi): add MIDI backend interface
build: add mpu401 to hw/audio/meson.build
feat(midi): add Munt MT-32/CM-32L backend
feat(midi): add FluidSynth General MIDI backend
feat(midi): add host MIDI passthrough backend
build: add MIDI backends to audio/meson.build
```

---

### SESSION 1 — Vendor Nuked-OPL3

**Goal:** Download Nuked-OPL3 source files and place them in the QEMU source tree.

**What to do:**
1. Clone or download nukeykt/Nuked-OPL3 from GitHub
2. Copy only `opl3.c` and `opl3.h` into `~/Projects/qemu-upstream/third-party/nuked-opl3/`
3. Create the `third-party/nuked-opl3/` directory if it doesn't exist
4. Verify the files are LGPL 2.1 (compatible with QEMU's GPL v2)
5. Commit: `vendor: add Nuked-OPL3 library (opl3.c, opl3.h)`

**Risk:** Zero. Just file copying.

---

### SESSION 2 — Explore SB16 (No Changes)

**Goal:** Read `hw/audio/sb16.c` and produce a mapping of the OPL2 integration points.

**What to do:**
1. Read `hw/audio/sb16.c` completely
2. Find and list every reference to: fmopl, YM3812, Adlib, OPL
3. Map the I/O port registrations (portio arrays, `isa_register_portio_list()`)
4. Find the audio voice creation (`AUD_open_out()`) for OPL output
5. Find the timer setup that drives the OPL chip clock
6. Find `fmopl.c` location and its meson.build entry
7. Produce a replacement table: each YM3812_* function → equivalent OPL3_* function from Nuked-OPL3

**Deliverable:** A summary report I can review. No files changed. No commit.

---

### SESSION 3 — Write OPL3 Wrapper

**Goal:** Create `hw/audio/nuked-opl3.c` — the QEMU device wrapper around Nuked-OPL3.

**What to do:**
1. Create `hw/audio/nuked-opl3.c` following QEMU's QOM device pattern (as seen in SB16)
2. Register ISA I/O ports 0x388–0x38B (AdLib/OPL3)
3. On port write: call `OPL3_WriteRegBuffered()`
4. On timer tick: call `OPL3_GenerateStream()` at 49716 Hz native rate
5. Create an audio voice via `AUD_open_out()` to feed stereo PCM into QEMU's mixer
6. Also create `hw/audio/nuked-opl3.h` with the public interface
7. Commit: `feat(opl3): add Nuked-OPL3 QEMU device wrapper`

**Reference:** The OPL3 register map and integration plan are in 05-SOUND-DESIGN.md (in the qemu-sa docs repo). Key points:
- Port 388h: AdLib address register (bank 0)
- Port 389h: AdLib data register (bank 0)
- Port 38Ah: OPL3 address register (bank 1)
- Port 38Bh: OPL3 data register (bank 1)
- Native sample rate: 49716 Hz
- Generate at native rate, let QEMU's mixer resample to host rate

---

### SESSION 4 — Swap OPL2 → OPL3 in SB16

**Goal:** Modify `hw/audio/sb16.c` to use Nuked-OPL3 instead of stock OPL2.

**What to do:**
1. Using the replacement table from Session 2, replace every YM3812* call with the OPL3 equivalent
2. Update the device state struct to hold OPL3 chip state instead of YM3812 pointer
3. Update timer callbacks to drive OPL3 at the correct rate
4. Keep PCM/DMA playback completely untouched
5. Keep mixer registers completely untouched
6. Add a configurable option: `opl3=nuked` (default) vs `opl3=compat` (stock OPL2 fallback)
7. Commit: `feat(opl3): replace YM3812 with Nuked-OPL3 in SB16 device`

---

### SESSION 5 — Build + Verify (OPL3)

**Goal:** Update build system, compile, verify binary starts.

**What to do:**
1. Edit `hw/audio/meson.build` to add `nuked-opl3.c` and the vendored library path
2. Run: `cd ~/Projects/qemu-build && make -j$(nproc)`
3. If compile errors: fix them one at a time, explain each fix
4. Run: `./qemu-system-i386 --version` to verify binary works
5. Run: `./qemu-system-i386 -soundhw help` to verify SB16 is still listed
6. Commit: `build: add nuked-opl3 to hw/audio/meson.build`

**After this session:** I take Timeshift snapshot `24-nuked-opl3-integrated`

---

### SESSION 6 — Write MPU-401 Device

**Goal:** Create `hw/audio/mpu401.c` and `mpu401.h` — MIDI interface at ports 0x330–0x331.

**What to do:**
1. Create ISA device following QOM pattern
2. Register I/O ports 0x330 (data) and 0x331 (status/command)
3. Implement UART mode only (not Intelligent mode — no game needs it)
4. UART protocol: 0x3F = enter UART mode, 0xFF = reset (returns 0xFE ACK)
5. Receive MIDI bytes from guest, assemble into complete MIDI messages
6. Handle SysEx passthrough (0xF0...0xF7, variable length, buffer entire message)
7. Route assembled messages to the MIDI backend interface (Session 7)
8. Commit: `feat(mpu401): add MPU-401 UART mode ISA device`

**Reference:** MPU-401 UART protocol details are in 05-SOUND-DESIGN.md.

---

### SESSION 7 — MIDI Backend Interface

**Goal:** Create `audio/midi.h` — the common interface all MIDI backends implement.

**What to do:**
1. Define the `MIDIBackend` struct with function pointers:
   - `init(const char *config)` — initialize backend
   - `send_short(uint8_t status, uint8_t data1, uint8_t data2)` — note on/off, CC, etc.
   - `send_sysex(const uint8_t *data, size_t len)` — SysEx messages
   - `generate(int16_t *buf, int frames)` — render audio into buffer
   - `close()` — cleanup
2. This is a small header file, maybe 30–40 lines
3. Commit: `feat(midi): add MIDI backend interface`

---

### SESSION 8 — Build + Verify (MPU-401)

**Goal:** Update build system for MPU-401, compile, verify.

**What to do:**
1. Edit `hw/audio/meson.build` to add `mpu401.c`
2. Run: `cd ~/Projects/qemu-build && make -j$(nproc)`
3. Fix any compile errors
4. Verify binary starts
5. Commit: `build: add mpu401 to hw/audio/meson.build`

**After this session:** I take Timeshift snapshot `25-mpu401-added`

---

### SESSION 9 — Munt MT-32 Backend

**Goal:** Create `audio/midi-munt.c` — MT-32/CM-32L emulation via libmt32emu.

**What to do:**
1. Implement the `MIDIBackend` interface from midi.h
2. Link against libmt32emu (already installed: `pacman -Q munt-mt32emu`)
3. Accept ROM path configuration: `-midi mt32,romdir=/path/to/mt32-roms`
4. Receive MIDI events from MPU-401, render via MT-32 emulation, output PCM to QEMU mixer
5. Handle graceful failure if ROM files are missing (warn, don't crash)
6. Commit: `feat(midi): add Munt MT-32/CM-32L backend`

**Note:** ROM files (MT32_CONTROL.ROM, MT32_PCM.ROM) are copyrighted by Roland. We never bundle them. Users provide their own.

---

### SESSION 10 — FluidSynth General MIDI Backend

**Goal:** Create `audio/midi-fluidsynth.c` — General MIDI via libfluidsynth.

**What to do:**
1. Implement the `MIDIBackend` interface from midi.h
2. Link against libfluidsynth (already installed: `pacman -Q fluidsynth`)
3. Accept SoundFont configuration: `-midi gm,soundfont=/path/to/FluidR3_GM.sf2`
4. Load SoundFont, receive MIDI events, render to PCM, output to QEMU mixer
5. Handle graceful failure if SoundFont file is missing
6. Commit: `feat(midi): add FluidSynth General MIDI backend`

---

### SESSION 11 — Host MIDI Passthrough Backend

**Goal:** Create `audio/midi-host.c` — raw MIDI passthrough to ALSA sequencer.

**What to do:**
1. Implement the `MIDIBackend` interface from midi.h
2. Send raw MIDI bytes through ALSA sequencer to specified port
3. Configuration: `-midi host,port=hw:1,0`
4. The `generate()` function is a no-op (real hardware produces its own audio)
5. Handle graceful failure if ALSA port doesn't exist
6. Commit: `feat(midi): add host MIDI passthrough backend`

---

### SESSION 12 — Build + Integration Test

**Goal:** Update build system for all MIDI backends, compile, full verification.

**What to do:**
1. Edit `audio/meson.build` to add all three MIDI backend files
2. Add meson options for optional dependencies (munt, fluidsynth)
3. Implement graceful degradation: missing libs disable features, not the build
4. Run: `cd ~/Projects/qemu-build && make -j$(nproc)`
5. Fix any compile errors
6. Verify: `./qemu-system-i386 --version`
7. Verify: the new `-midi` flag is recognized (or at least doesn't crash)
8. Commit: `build: add MIDI backends to audio/meson.build`

**After this session:** I take Timeshift snapshot `26-phase1-sound-revolution-complete`

---

## Reference Architecture

From 04-ARCHITECTURE.md — the files we're creating:

```
qemu-upstream/
├── hw/audio/
│   ├── sb16.c            (MODIFIED — OPL3 + MPU-401 integration)
│   ├── nuked-opl3.c      (NEW — Sessions 3-4)
│   ├── nuked-opl3.h      (NEW — Session 3)
│   ├── mpu401.c           (NEW — Session 6)
│   ├── mpu401.h           (NEW — Session 6)
│   └── meson.build        (MODIFIED — Sessions 5, 8)
├── audio/
│   ├── midi.h             (NEW — Session 7)
│   ├── midi-munt.c        (NEW — Session 9)
│   ├── midi-fluidsynth.c  (NEW — Session 10)
│   ├── midi-host.c        (NEW — Session 11)
│   └── meson.build        (MODIFIED — Session 12)
└── third-party/
    └── nuked-opl3/
        ├── opl3.c         (VENDORED — Session 1)
        └── opl3.h         (VENDORED — Session 1)
```

## Sound Data Flow (What We're Building)

```
Guest writes port 388h → nuked-opl3.c → OPL3_WriteRegBuffered() → OPL3_GenerateStream()
                                          → 49716 Hz stereo PCM → QEMU mixer → speakers

Guest writes port 330h → mpu401.c → assemble MIDI message → midi backend dispatch
                                      → midi-munt.c (MT-32 PCM) → QEMU mixer → speakers
                                      → midi-fluidsynth.c (GM PCM) → QEMU mixer → speakers
                                      → midi-host.c (raw MIDI → ALSA → real hardware)
```

## Important: Start with Session 1

When I say "let's start," begin with Session 1 (vendoring Nuked-OPL3). Walk me through every step, explain what you're doing, and wait for my approval before committing. We go one session at a time — I'll tell you when to proceed to the next.

Let's start with Session 1.
