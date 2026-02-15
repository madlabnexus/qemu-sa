# 00 — Project Overview

## Vision

QEMU-SA (Sound & Acceleration) is a QEMU fork that turns QEMU into a retro PC gaming platform spanning DOS through Windows XP. It integrates proven open-source components — Nuked-OPL3, Munt, FluidSynth, qemu-3dfx, libcue — into QEMU's device model framework, creating a single hypervisor that handles four decades of PC gaming.

## The Four Target Machines

Each machine represents the ideal gaming PC for its era — the hardware you'd build if money was no object and game compatibility was the only goal.

### Machine 1: DOS Gaming PC (1987–1996)

**The real hardware:** 486DX2-66, 16-32MB RAM, Sound Blaster AWE32, Tseng ET4000 SVGA, 2x CD-ROM.

Why a 486DX2 and not a Pentium? Many DOS games have speed-sensitive timing loops that run too fast on faster CPUs. The 486DX2-66 was the sweet spot where almost everything ran correctly.

**QEMU-SA approach:** TCG (software CPU emulation) with icount for approximate speed control. Enhanced SB16 device with Nuked-OPL3, MPU-401 MIDI routed to Munt or FluidSynth, CUE/BIN CD-ROM support. Cirrus SVGA for display.

**Compatibility estimate:** ~85-90% of DOS games. The remaining 10-15% that need cycle-accurate ISA bus timing or exact CPU cycle counts require 86Box.

### Machine 2: Windows 3.11 PC (1990–1995)

**The real hardware:** 486DX2-66 or Pentium 75, 32MB RAM, SB AWE32, Matrox Millennium or S3 Trio64 for SVGA.

**QEMU-SA approach:** Same as Machine 1 but with emphasis on SVGA driver support. Win3.11 games are generally less timing-sensitive than raw DOS games.

**Compatibility estimate:** ~90%+ of Win3.11 games.

### Machine 3: Windows 98 PC (1996–2001) — THE CROWN JEWEL

**The real hardware:** Pentium III, i440BX motherboard, 256-512MB RAM, AC97 audio, Voodoo3 or GeForce2.

**QEMU-SA approach:** KVM acceleration (native CPU speed), AC97 audio with SigmaTel WDM drivers (provides SB emulation in DOS box), qemu-3dfx for Glide and OpenGL passthrough to host GPU. This is where QEMU-SA has zero competition — no other solution provides KVM-accelerated Win98 with real 3D graphics.

**Compatibility estimate:** ~95%+ of Windows 98 games that use Glide or OpenGL.

### Machine 4: Windows XP PC (2001–2008)

**The real hardware:** Athlon XP or Pentium 4, 1-2GB RAM, GeForce 6/7 or Radeon X-series.

**QEMU-SA approach:** KVM + VFIO GPU passthrough. The guest sees a real GPU with native drivers. AC97 or Intel HDA for audio, VDMSound for DOS games within XP.

**Compatibility estimate:** Near 100% for games that run on XP with the passed-through GPU.

## Feasibility Analysis

### What IS Feasible

| Component | Status | Effort | Risk |
|-----------|--------|--------|------|
| Nuked-OPL3 integration | Library exists, needs QEMU ISA device wrapper | ~300 lines glue code | Low |
| Munt MT-32 integration | Library exists, needs MPU-401 device + backend | Medium (new device) | Low |
| FluidSynth integration | Library exists, needs MIDI backend | Small | Low |
| qemu-3dfx patches | Patches exist for QEMU 9.2.x, actively maintained | Apply + maintain | Low |
| CUE/BIN CD-ROM | libcue exists, needs ATAPI CD audio integration | Medium | Medium |
| Machine profiles | Configuration work, SeaBIOS tuning | Small | Low |
| KVM for Win98/XP | Already works in stock QEMU | Configuration | None |
| VFIO GPU passthrough | Already works in stock QEMU | Configuration | None |

### What Is NOT Feasible

| Component | Why | Alternative |
|-----------|-----|-------------|
| Cycle-accurate CPU emulation | Would require rewriting QEMU's entire TCG engine. QEMU counts instructions, not cycles. A `NOP` and `DIV` cost the same. | Use 86Box for games needing cycle accuracy |
| ISA bus timing emulation | QEMU doesn't model ISA bus wait states, DMA timing at hardware level | Use 86Box |
| Exact CPU speed matching (e.g., "run at exactly 66MHz 486") | icount approximates but isn't cycle-accurate | icount + calibrated delay loops give "close enough" for most games |
| Full chipset emulation (430VX, PIIX, etc.) from 86Box | Massive engineering effort, 86Box has years of chipset work | Use QEMU's i440FX which works for Win9x/XP |
| AWE32 EMU8000 wavetable | Complex DSP (32-voice, SoundFont, reverb/chorus) | Stretch goal — Phase 3+ |

### License Compatibility (Verified)

All components are GPL-compatible with QEMU (GPLv2):

| Component | License | Compatible? |
|-----------|---------|-------------|
| QEMU | GPL v2.0 | Base |
| qemu-3dfx | GPL v2.0 | ✅ Same license |
| Nuked-OPL3 | LGPL v2.1 | ✅ Can link into GPL |
| Munt | GPL v2.0+ / LGPL v2.1 | ✅ Same or compatible |
| FluidSynth | LGPL v2.1 | ✅ Can link into GPL |
| libcue | GPL v2.0 | ✅ Same license |
| SoftGPU wrappers | MIT | ✅ Permissive |

## Competitive Landscape

| Feature | QEMU-SA | 86Box | DOSBox-Staging | Stock QEMU | VirtualBox |
|---------|---------|-------|----------------|------------|------------|
| DOS game sound (OPL3) | ✅ Nuked | ✅ Nuked | ✅ Nuked | ❌ OPL2 only | ❌ |
| MT-32 MIDI | ✅ Munt | ✅ Munt | ✅ Munt | ❌ | ❌ |
| CD Audio (CUE/BIN) | ✅ | ✅ | ✅ | ❌ ISO only | ❌ |
| Win98 KVM acceleration | ✅ | ❌ | ❌ | ✅ | ✅ |
| 3Dfx Glide passthrough | ✅ qemu-3dfx | ❌ | ❌ | ❌ | ❌ |
| VFIO GPU passthrough | ✅ | ❌ | ❌ | ✅ | Limited |
| Cycle-accurate 8088-486 | ❌ | ✅ | Partial | ❌ | ❌ |
| Runs on Proxmox/server | ✅ | ❌ | ❌ | ✅ | ❌ |
| AWE32 wavetable | Planned | ✅ | ❌ | ❌ | ❌ |

**QEMU-SA's unique position:** The only solution that spans DOS to XP in one hypervisor with KVM acceleration AND proper retro sound AND 3D graphics passthrough. Unmatched for Win98 gaming.

## Development Phases

### Phase 0: Foundation (Month 1-2)
- Fork QEMU 9.2.2
- Apply qemu-3dfx patches
- Verify Win98 boots with 3Dfx passthrough
- Set up CI/CD, packaging

### Phase 1: Sound Revolution (Month 2-4)
- Integrate Nuked-OPL3 as ISA device (replaces YM3812)
- Add MPU-401 MIDI to SB16 device
- Connect Munt (MT-32) and FluidSynth (GM) as MIDI backends
- Test: DOOM (OPL3), Monkey Island (MT-32), TIE Fighter (GM)

### Phase 2: CD Audio (Month 4-5)
- CUE/BIN parser via libcue
- ATAPI CD audio playback commands
- Audio decode: PCM, FLAC, OGG, WAV
- Test: Descent, C&C Red Alert, Wing Commander III

### Phase 3: Advanced Sound (Month 5-7)
- AWE32 EMU8000 wavetable synthesizer (stretch goal)
- GUS improvements
- Test: Warcraft II AWE32 mode, Epic Pinball GUS

### Phase 4: Machine Profiles & Polish (Month 7-9)
- Pre-configured machine profiles
- CPU throttle via icount tuning
- Per-game config system
- Packaging: AUR, Debian PPA, Flatpak
- Documentation and game compatibility database

### Phase 5: Ecosystem (Month 9-12)
- Optional Qt/web launcher GUI
- Proxmox integration
- 86Box accuracy-mode documentation
- Community building: Vogons, Reddit, YouTube
