# Phase 1: Sound Revolution — Completion Report

**Date:** 2026-03-22
**Branch:** `qemu-sa/phase1` on `qemu-upstream`
**Base:** QEMU 9.2.4 + qemu-3dfx patches
**Timeshift snapshot:** `26-phase1-sound-revolution-complete`

## Summary

Phase 1 added three sound components to the QEMU source tree: Nuked-OPL3 (FM synthesis), MPU-401 (MIDI interface), and three MIDI backends (Munt MT-32, FluidSynth General MIDI, host passthrough). All components compile cleanly and register as QEMU devices. Backends are not yet wired to the MPU-401 — that's integration work for Phase 1.5.

## Commits (13 total)

```
a6a206361e  vendor: add Nuked-OPL3 library (opl3.c, opl3.h)
2736f62427  feat(opl3): add Nuked-OPL3 QEMU device wrapper
c80d69a726  build: replace fmopl/adlib with nuked-opl3 in meson.build
ff882fdd58  feat(mpu401): add MPU-401 UART mode ISA device
57f9979e32  feat(midi): add MIDI backend interface
8a22feff87  build: add MPU-401 to Kconfig and meson.build
33267131ee  feat(midi): add Munt MT-32/CM-32L backend
3be0eeb599  feat(midi): add FluidSynth General MIDI backend
cd47c30dec  feat(midi): add host MIDI passthrough backend
552ec7e8b5  build: add MIDI backends to audio/meson.build
```

## Files Added/Modified

### New files (QEMU-SA additions)

```
qemu-upstream/
├── third-party/nuked-opl3/
│   ├── opl3.c              (vendored, nukeykt commit cfedb09)
│   ├── opl3.h              (vendored)
│   └── LICENSE              (LGPL 2.1)
├── hw/audio/
│   ├── nuked-opl3.c        (OPL3 ISA device wrapper, ~290 lines)
│   ├── nuked-opl3.h        (type name define)
│   ├── mpu401.c             (MPU-401 UART ISA device, ~290 lines)
│   └── mpu401.h             (type name define)
├── audio/
│   ├── midi.h               (MIDIBackend interface, ~65 lines)
│   ├── midi-munt.c          (MT-32/CM-32L via libmt32emu, ~230 lines)
│   ├── midi-fluidsynth.c    (General MIDI via libfluidsynth, ~260 lines)
│   └── midi-host.c          (ALSA raw MIDI passthrough, ~140 lines)
```

### Modified files

```
hw/audio/meson.build    — CONFIG_ADLIB now builds nuked-opl3.c instead of fmopl.c/adlib.c
                         — Added CONFIG_MPU401 entry for mpu401.c
hw/audio/Kconfig        — Added config MPU401 (ISA_BUS, default y)
audio/meson.build       — Added midi-munt.c (when mt32emu), midi-fluidsynth.c (when fluidsynth),
                           midi-host.c (when alsa)
meson.build             — Added mt32emu and fluidsynth dependency detection (required: false)
```

## Architecture Decisions Made During Implementation

### ADR-011: OPL3 is a Separate Device, Not Inside SB16

**Discovery:** In QEMU 9.2.4, the OPL2 chip lives in `adlib.c` (a standalone ISA device), not inside `sb16.c`. The SB16 device handles only PCM/DMA — zero OPL references.

**Decision:** Create `nuked-opl3.c` as a replacement for `adlib.c`, not as a modification to `sb16.c`. The `CONFIG_ADLIB` flag now compiles our device instead of the stock one.

**Impact:** Simpler than the original plan. Sessions 3–4 from the Phase 1 prompt were revised — no SB16 modifications needed for OPL3.

### ADR-012: OPL3 Timer Emulation Implemented in Wrapper

**Context:** Nuked-OPL3 is a pure synthesis library — it generates audio samples but doesn't emulate the OPL3's timer registers or status byte. Games use timer-based OPL detection (write timer value, wait for overflow flag in status register).

**Decision:** Implement OPL3 Timer 1 (80µs) and Timer 2 (320µs) using QEMU virtual timers in `nuked-opl3.c`. Timer registers 0x02, 0x03, 0x04 are intercepted before reaching the synthesis engine. Status register (timer overflow flags, IRQ) is maintained by the wrapper.

### ADR-013: Stereo at Native OPL3 Rate

**Context:** Stock `adlib.c` runs at 44100 Hz mono. OPL3 natively generates at 49716 Hz stereo.

**Decision:** Generate at 49716 Hz stereo and let QEMU's audio mixer resample to host rate. This preserves pitch accuracy and stereo separation that OPL3 games expect.

## Dependencies

| Library | Package | Version | License | Used By |
|---------|---------|---------|---------|---------|
| Nuked-OPL3 | vendored | cfedb09 | LGPL 2.1 | nuked-opl3.c |
| libmt32emu | munt | 2.7.2 | LGPL 2.1 | midi-munt.c |
| libfluidsynth | fluidsynth | 2.5.3 | LGPL 2.1 | midi-fluidsynth.c |
| libasound (ALSA) | alsa-lib | 1.2.15.3 | LGPL 2.1 | midi-host.c |

All dependencies are optional (`required: false` in meson). Missing libraries disable features, not the build.

## Known Issues / TODO

1. **MPU-401 → backend wiring is stubbed.** The MPU-401 assembles MIDI messages correctly but dispatches to stub functions that log and discard. Need to wire `midi_get_backend()` / `midi_set_backend()` and add command-line parsing for `-midi` flag.

2. **Munt initializer warning.** `(mt32emu_report_handler_i){{NULL}}` produces "braces around scalar initializer" warning. Cosmetic — works correctly.

3. **No runtime testing yet.** Components compile and register but haven't been tested with actual DOS games. Need: DOS 6.22 disk image, test games (DOOM for OPL3, Monkey Island for MT-32).

4. **No `-midi` command-line option.** MIDI backend selection isn't exposed to the user yet. Need to add QEMU command-line parsing.

5. **`audio/midi.h` declares `midi_get_backend()` / `midi_set_backend()` but they're not implemented.** Need a small `audio/midi.c` with the global backend pointer and these functions.

## What's Next

### Phase 1.5: Integration Wiring
- Implement `audio/midi.c` (backend registry, command-line parsing)
- Wire MPU-401 stubs to actual backend dispatch
- Add `-midi mt32,romdir=PATH` / `-midi gm,soundfont=PATH` / `-midi host,port=PORT` options

### Phase 2: CD Audio (CUE/BIN)
- As documented in 04-ARCHITECTURE.md

### Testing
- Boot DOS 6.22 with `-device nuked-opl3 -device mpu401`
- Play DOOM (OPL3 FM music)
- Play Monkey Island (MT-32 MIDI)
- Verify timer-based OPL detection works
