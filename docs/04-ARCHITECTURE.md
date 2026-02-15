# 04 — Architecture

## Overview

QEMU-SA extends QEMU's device model framework with new sound devices, MIDI backends, a CUE/BIN CD-ROM parser, and machine profile presets. It integrates qemu-3dfx patches for 3Dfx Glide/OpenGL passthrough. All changes live within QEMU's existing architecture — no engine modifications.

## QEMU Architecture (Relevant Parts)

```
┌──────────────────────────────────────────────────────────┐
│                      QEMU Process                        │
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌───────────────────┐   │
│  │ CPU      │    │ Memory   │    │ Device Model      │   │
│  │ TCG/KVM  │    │ Manager  │    │                   │   │
│  │          │    │          │    │ ┌───────────────┐  │   │
│  │          │    │          │    │ │ ISA Bus       │  │   │
│  │          │    │          │    │ │  SB16 + OPL3  │  │   │
│  │          │    │          │    │ │  MPU-401      │  │   │
│  │          │    │          │    │ └───────────────┘  │   │
│  │          │    │          │    │ ┌───────────────┐  │   │
│  │          │    │          │    │ │ PCI Bus       │  │   │
│  │          │    │          │    │ │  AC97         │  │   │
│  │          │    │          │    │ │  VGA/3dfx     │  │   │
│  │          │    │          │    │ │  VFIO         │  │   │
│  │          │    │          │    │ └───────────────┘  │   │
│  │          │    │          │    │ ┌───────────────┐  │   │
│  │          │    │          │    │ │ IDE/ATAPI     │  │   │
│  │          │    │          │    │ │  CD-ROM       │  │   │
│  │          │    │          │    │ └───────────────┘  │   │
│  └──────────┘    └──────────┘    └───────────────────┘   │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Audio Subsystem                                   │    │
│  │  QEMU audio mixer → PulseAudio/ALSA → speakers   │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

## QEMU-SA Modifications Map

### Layer 1: ISA Sound Devices (hw/audio/)

**File: `hw/audio/sb16.c`** — existing QEMU SB16 device

Changes:
- Replace YM3812 (OPL2) calls with Nuked-OPL3 (YMF262) API
- Add OPL3 register file (388h-38Bh for AdLib, 220h-22Fh for SB)
- Wire DMA channel for PCM playback (already exists)
- Add MPU-401 UART sub-device at 330h
- Route MPU-401 MIDI events to configurable backend

**New file: `hw/audio/nuked-opl3.c`** — OPL3 integration

- Wraps nukeykt's `opl3.c` and `opl3.h`
- Implements ISA I/O port handlers (port 388h-38Bh)
- Generates PCM samples at 49716 Hz (OPL3 native rate)
- Feeds samples into QEMU audio mixer
- Configurable: `opl3=nuked` (default) or `opl3=compat` (stock OPL2)

**New file: `hw/audio/mpu401.c`** — MPU-401 MIDI interface

- ISA device at port 330h-331h
- Implements MPU-401 UART mode (Intelligent mode not needed for games)
- Receives MIDI byte stream from guest
- Routes to selected MIDI backend (Munt, FluidSynth, or host MIDI)

### Layer 2: MIDI Backends (audio/)

**New file: `audio/midi-munt.c`** — MT-32/CM-32L backend

- Links against libmt32emu
- Receives MIDI events from MPU-401 device
- Renders audio via MT-32 emulation
- Outputs PCM to QEMU audio mixer
- Configurable ROM path: `-midi mt32,romdir=/path/to/mt32-roms`

**New file: `audio/midi-fluidsynth.c`** — General MIDI backend

- Links against libfluidsynth
- Loads SoundFont file
- Renders MIDI to PCM
- Configurable: `-midi gm,soundfont=/path/to/FluidR3_GM.sf2`

**New file: `audio/midi-host.c`** — Host MIDI passthrough

- Sends raw MIDI to host ALSA/CoreMIDI sequencer
- For users with real hardware (MT-32, SC-55)
- Configurable: `-midi host,port=hw:1,0`

### Layer 3: CD-ROM Enhancement (hw/ide/)

**Modified: `hw/ide/atapi.c`** — ATAPI CD-ROM

Changes:
- Add CUE/BIN image format support alongside ISO
- Implement READ CD, READ TOC with audio track awareness
- Audio play commands: PLAY AUDIO, PAUSE/RESUME, READ SUB-CHANNEL

**New file: `hw/ide/cue-parser.c`** — CUE sheet parser

- Uses libcue to parse CUE files
- Maps data tracks (MODE1/2352, MODE2/2352)
- Maps audio tracks (AUDIO)
- Handles pregaps, postgaps, INDEX 00/01
- Supports referenced audio formats: raw PCM, FLAC, OGG, WAV

**New file: `hw/ide/cdda-player.c`** — CD Digital Audio playback

- Reads audio track data from BIN/WAV/FLAC/OGG files
- Converts to PCM
- Feeds into QEMU audio mixer as separate stream
- Handles play/pause/stop/seek from ATAPI commands

### Layer 4: qemu-3dfx (hw/3dfx/ — from patches)

From kjliew/qemu-3dfx patches:
- Virtual PCI device that intercepts Glide/OpenGL calls
- Serializes GPU commands to host
- Host-side wrapper translates to real OpenGL on host GPU
- Works with Win98/XP Glide and OpenGL games

### Layer 5: Machine Profiles (configs/)

**New: `configs/machines/`**

Pre-configured machine definitions:
```
configs/machines/dos-gaming.cfg
configs/machines/win311.cfg
configs/machines/win98-3dfx.cfg
configs/machines/winxp-vfio.cfg
```

Each profile specifies: machine type, CPU model, RAM, bus configuration, sound devices, display adapter, boot order, and recommended command-line.

## Data Flow: Sound

### OPL3 (FM Synthesis)

```
Guest writes to port 388h/389h
        │
        ▼
nuked-opl3.c (ISA I/O handler)
        │
        ▼
opl3_write_reg() — Nuked-OPL3 library
        │
        ▼
opl3_generate() — produces 49716 Hz stereo PCM
        │
        ▼
QEMU audio mixer (resamples to host rate)
        │
        ▼
PulseAudio/ALSA → Speakers
```

### MIDI (MT-32 or General MIDI)

```
Guest writes MIDI byte to port 330h
        │
        ▼
mpu401.c (ISA I/O handler, UART mode)
        │
  Assembles MIDI message
        │
        ▼
midi-backend dispatch (based on -midi flag)
        │
   ┌────┴────┐
   ▼         ▼
midi-munt   midi-fluidsynth
(MT-32)     (GM SoundFont)
   │         │
   ▼         ▼
PCM output  PCM output
   │         │
   └────┬────┘
        ▼
QEMU audio mixer → PulseAudio/ALSA → Speakers
```

### CD Audio

```
Guest sends ATAPI PLAY AUDIO (MSF) command
        │
        ▼
atapi.c → cue-parser.c (find audio track)
        │
        ▼
cdda-player.c (read audio data, decode if needed)
        │
        ▼
PCM stream → QEMU audio mixer → speakers
```

## Command-Line Interface (Target)

```bash
# DOS Gaming PC
qemu-system-i386 \
  -machine dos-gaming \
  -cpu 486-v1 \
  -m 32M \
  -soundhw sb16-sa \
  -opl3 nuked \
  -midi mt32,romdir=~/vms/roms/mt32 \
  -cdrom ~/vms/iso/game.cue \
  -hda ~/vms/images/dos622.qcow2

# Windows 98 with 3Dfx
qemu-system-i386 \
  -machine win98-3dfx \
  -enable-kvm \
  -cpu host \
  -m 512M \
  -soundhw ac97 \
  -device 3dfx-glide \
  -cdrom ~/vms/iso/game.cue \
  -hda ~/vms/images/win98se.qcow2
```

## File Tree (QEMU-SA additions)

```
qemu/
├── hw/
│   ├── audio/
│   │   ├── sb16.c            (modified — OPL3 + MPU-401)
│   │   ├── nuked-opl3.c     (NEW — Nuked-OPL3 wrapper)
│   │   ├── nuked-opl3.h     (NEW)
│   │   ├── mpu401.c         (NEW — MPU-401 UART device)
│   │   └── mpu401.h         (NEW)
│   ├── ide/
│   │   ├── atapi.c           (modified — CUE/BIN + audio)
│   │   ├── cue-parser.c     (NEW — libcue integration)
│   │   ├── cue-parser.h     (NEW)
│   │   ├── cdda-player.c    (NEW — CD audio playback)
│   │   └── cdda-player.h    (NEW)
│   └── 3dfx/                 (from qemu-3dfx patches)
├── audio/
│   ├── midi-munt.c          (NEW — MT-32 backend)
│   ├── midi-fluidsynth.c    (NEW — GM SoundFont backend)
│   ├── midi-host.c          (NEW — host MIDI passthrough)
│   └── midi.h               (NEW — common MIDI interface)
├── configs/
│   └── machines/
│       ├── dos-gaming.cfg    (NEW)
│       ├── win311.cfg        (NEW)
│       ├── win98-3dfx.cfg   (NEW)
│       └── winxp-vfio.cfg   (NEW)
├── third-party/
│   ├── nuked-opl3/           (vendored — opl3.c, opl3.h)
│   └── libmt32emu/           (or system library)
└── meson.build               (modified — new build options)
```

## Design Principles

1. **Minimal QEMU core changes.** New functionality lives in new files. Existing files get surgical modifications at well-defined hook points.

2. **Backend abstraction.** MIDI backends share a common interface. The MPU-401 device doesn't know or care whether Munt or FluidSynth is rendering. Adding a new backend means implementing ~5 functions.

3. **Graceful degradation.** Missing libraries disable features, not the build. No Munt? MIDI plays through FluidSynth. No FluidSynth? MIDI is silent but the game runs. No libcue? CUE files rejected, ISO still works.

4. **Upstream compatibility.** We rebase on QEMU releases, not a permanent fork divergence. Changes are structured as potential upstream patches.

5. **Profile over configure.** Machine profiles encapsulate complexity. Users run `--machine dos-gaming`, not 15 separate flags.
