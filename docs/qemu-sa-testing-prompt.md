# QEMU-SA Phase 1 Testing — Session Handoff Prompt

## Who I Am

I'm Savant (username: madlabn). I'm the project director for QEMU-SA, not a C programmer. I understand the architecture and can follow along, but I need step-by-step guidance for setting up VMs and testing.

## Project

QEMU-SA (Sound & Acceleration) — a QEMU fork for retro PC gaming, DOS through Windows XP.
- **Repo:** github.com/madlabnexus/qemu-sa (project docs)
- **Source:** ~/Projects/qemu-upstream (QEMU 9.2.4 + qemu-3dfx + QEMU-SA sound patches)
- **Build dir:** ~/Projects/qemu-build (out-of-tree, binaries here)
- **Branch:** `qemu-sa/phase1` (14 commits)
- **License:** GPL v2.0

## Machine

- **Notebook:** Lenovo ThinkPad P16v Gen 2 ("archp16v")
- **OS:** EndeavourOS (Arch Linux), GNOME 49.5, Wayland, kernel 6.19.8-arch1-1
- **GPU:** NVIDIA RTX 3000 Ada (proprietary 590.48.01) + Intel Arc iGPU
- **Terminal:** Ghostty
- **Editor/AI:** Claude Code CLI + VS Code

## What's Already Done (Phase 1 + 1.5 Complete)

All code is written, compiled, and linked. The binary works. No runtime testing yet.

### QEMU-SA Devices Built

| Device | Type | File | Status |
|--------|------|------|--------|
| Nuked-OPL3 | ISA | `hw/audio/nuked-opl3.c` | Compiled, registered as `nuked-opl3` |
| MPU-401 UART | ISA | `hw/audio/mpu401.c` | Compiled, registered as `mpu401` |
| MIDI backend registry | — | `audio/midi.c`, `audio/midi.h` | Compiled |
| Munt MT-32 backend | — | `audio/midi-munt.c` | Compiled, links libmt32emu 2.7.2 |
| FluidSynth GM backend | — | `audio/midi-fluidsynth.c` | Compiled, links libfluidsynth 2.5.3 |
| Host MIDI passthrough | — | `audio/midi-host.c` | Compiled, links ALSA |

### Key Facts Discovered During Implementation

1. **OPL lives in `adlib.c`, not `sb16.c`.** In QEMU 9.2.4, OPL2 is a separate ISA device. We replaced `adlib.c`/`fmopl.c` with `nuked-opl3.c` under the same `CONFIG_ADLIB` flag.
2. **qemu-3dfx is always-on.** The Glide and Mesa passthrough devices are hardwired into `hw/i386/pc.c` — no `-device` flag needed. Just install guest-side DLLs.
3. **SoundFont installed:** `/usr/share/soundfonts/FluidR3_GM.sf2` (pacman: soundfont-fluid)
4. **MPU-401 device properties:** `-device mpu401,backend=gm,config=/usr/share/soundfonts/FluidR3_GM.sf2`

### Binary Verified

```
./qemu-system-i386 --version
QEMU emulator version 9.2.4 (v9.2.4-9-gcd47c30dec-dirty)
  featuring qemu-3dfx@b2995afed2-15:44:36 Mar 22 2026 build

./qemu-system-i386 -device help 2>&1 | grep -i 'mpu401\|nuked'
name "mpu401", bus ISA, desc "MPU-401 UART MIDI Interface"
name "nuked-opl3", bus ISA, desc "Yamaha YMF262 (OPL3) - Nuked-OPL3"
```

### Snapshots

- `26-phase1-sound-revolution-complete` — after all 12 sessions
- `27-phase1.5-midi-wiring-complete` — after MPU-401 → backend wiring

## What We're Doing Now: Testing

The testing guide is in `~/Projects/qemu-sa/docs/qemu-sa-testing-guide.md` and contains complete step-by-step instructions. We need to:

### Task 1: Set Up DOS 6.22 VM
1. Create disk image at `~/vms/dos622/dos622.qcow2` (504MB)
2. Install DOS 6.22 from floppy images
3. Configure CONFIG.SYS and AUTOEXEC.BAT (memory managers, BLASTER env)
4. Create transfer disk and copy DOOM + Pinball Fantasies into VM
5. Create launch scripts (baseline, opl3, gm, mt32)

### Task 2: Test DOS Games
1. **Pinball Fantasies** — SB16 PCM baseline (stock QEMU, not our code)
2. **DOOM with Sound Blaster music** — tests Nuked-OPL3 FM synthesis (our code!)
3. **DOOM with General MIDI** — tests MPU-401 → FluidSynth pipeline (our code!)

### Task 3: Set Up Windows 98 SE VM
1. Create disk image at `~/vms/win98/win98se.qcow2` (4GB)
2. Install Win98 SE from ISO
3. Install drivers: AC97 (SigmaTel), DirectX 7, qemu-3dfx guest DLLs
4. Create launch script with KVM + gl=on

### Task 4: Test Win98
1. Windows audio (AC97 + SigmaTel driver)
2. 3dfx Glide passthrough with a game

## Launch Commands Quick Reference

**DOS with OPL3:**
```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 -cpu 486-v1 -m 32M \
  -display gtk -vga cirrus \
  -device sb16,iobase=0x220,irq=5,dma=1,hdma=5 \
  -device nuked-opl3 \
  -hda ~/vms/dos622/dos622.qcow2 -boot c
```

**DOS with OPL3 + General MIDI:**
```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 -cpu 486-v1 -m 32M \
  -display gtk -vga cirrus \
  -device sb16,iobase=0x220,irq=5,dma=1,hdma=5 \
  -device nuked-opl3 \
  -device mpu401,backend=gm,config=/usr/share/soundfonts/FluidR3_GM.sf2 \
  -hda ~/vms/dos622/dos622.qcow2 -boot c
```

**Win98 with KVM + 3dfx:**
```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 -enable-kvm -cpu host -m 512M \
  -display gtk,gl=on -vga cirrus \
  -device ac97 \
  -hda ~/vms/win98/win98se.qcow2 \
  -cdrom ~/vms/win98/win98se.iso \
  -net nic,model=rtl8139 -net user -boot c
```

## Directory Layout

```
~/Projects/
├── qemu-sa/           ← Project docs repo (github.com/madlabnexus/qemu-sa)
├── qemu-upstream/     ← QEMU 9.2.4 source + qemu-3dfx + QEMU-SA (branch: qemu-sa/phase1)
├── qemu-build/        ← Out-of-tree build directory (binaries here)
└── qemu-3dfx/         ← kjliew/qemu-3dfx repo (guest wrappers here)

~/vms/
├── dos622/            ← DOS VM (to be created)
│   ├── dos622.qcow2
│   ├── disk1.img, disk2.img, disk3.img (DOS install floppies)
│   ├── transfer.img   (FAT16 disk for transferring game files)
│   └── run-*.sh       (launch scripts)
├── win98/             ← Win98 VM (to be created)
│   ├── win98se.qcow2
│   ├── win98se.iso
│   └── run.sh
├── soundfonts/        ← (symlink to /usr/share/soundfonts/ or copy)
└── roms/
    └── mt32/          ← MT-32 ROMs (user-provided, optional)
```

## How to Work With Me

1. Walk me through each step — I'll run commands and paste output
2. If something breaks, explain what went wrong before trying to fix
3. I'm not a C programmer but I understand the architecture
4. I have Claude Code CLI installed if you need me to run complex commands
5. Document everything — update the testing guide and qemu-sa repo as we go

## Start Here

Start with Task 1 — setting up the DOS 6.22 VM. I have the floppy images ready. Walk me through it step by step.
