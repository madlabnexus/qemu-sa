# QEMU-SA

**Sound & Acceleration enhanced QEMU fork for retro PC gaming**

QEMU-SA is a QEMU fork that integrates cycle-accurate sound emulation, 3Dfx Glide/OpenGL passthrough, CUE/BIN CD-ROM audio, and pre-configured machine profiles into a single hypervisor for retro PC gaming — from DOS through Windows XP.

## What It Does

QEMU-SA combines components that already exist but have never been assembled together:

- **Nuked-OPL3** — transistor-level accurate Yamaha YMF262 FM synthesis (replaces QEMU's OPL2-only AdLib)
- **Munt** — Roland MT-32/CM-32L emulation for Sierra/LucasArts era MIDI
- **FluidSynth** — General MIDI via SoundFonts
- **qemu-3dfx** — 3Dfx Glide and MESA OpenGL passthrough to host GPU
- **CUE/BIN CD-ROM** — Red Book audio track support (QEMU only supports ISO natively)
- **Machine Profiles** — pre-configured hardware combos for each gaming era
- **VFIO GPU Passthrough** — native GPU acceleration for Windows 98/XP VMs

## Target Machines

| Profile | Era | CPU Mode | Sound | Graphics |
|---------|-----|----------|-------|----------|
| DOS Gaming PC | 1987–1996 | TCG + icount | SB16 + OPL3 + MPU-401 | Cirrus/SVGA |
| Windows 3.11 | 1990–1995 | TCG + icount | SB16 + OPL3 + MPU-401 | Cirrus/SVGA |
| Windows 98 | 1996–2001 | **KVM** | AC97 | **qemu-3dfx passthrough** |
| Windows XP | 2001–2008 | **KVM** | AC97/HDA | **VFIO GPU passthrough** |

## Why This Exists

Stock QEMU has no OPL3, no MIDI, no CD audio, and no 3D acceleration for retro games. You need 4-5 separate emulators (86Box, DOSBox-Staging, QEMU+qemu-3dfx, standalone VFIO) to cover the full DOS-to-XP range. QEMU-SA brings them under one roof.

The **crown jewel** is the Windows 98 experience: KVM gives you native CPU speed while qemu-3dfx provides real 3D acceleration — something 86Box and DOSBox cannot offer.

## Honest Limitations

QEMU-SA does **not** replace 86Box for cycle-accurate 8088/286/386 emulation. QEMU's TCG engine counts instructions, not CPU cycles. Games that depend on exact CPU timing (speed-sensitive loops, ISA bus timing tricks) may not work correctly. For those ~10-15% of DOS games, we recommend 86Box — and we document how to run it alongside QEMU-SA.

See [docs/00-PROJECT-OVERVIEW.md](docs/00-PROJECT-OVERVIEW.md) for the full feasibility analysis.

## Status

🚧 **Early development** — project infrastructure being set up.

## Building

See [docs/03-BUILD-SYSTEM.md](docs/03-BUILD-SYSTEM.md) for build instructions.

## Documentation

| Document | Description |
|----------|-------------|
| [Project Overview](docs/00-PROJECT-OVERVIEW.md) | Architecture, feasibility, honest limitations |
| [Notebook Setup](docs/01-SETUP-NOTEBOOK.md) | P16v Arch Linux dev workstation |
| [Server Setup](docs/02-SETUP-SERVER.md) | TD350 build & test server with VFIO |
| [Build System](docs/03-BUILD-SYSTEM.md) | How to compile QEMU-SA |
| [Architecture](docs/04-ARCHITECTURE.md) | Technical design decisions |
| [Sound Design](docs/05-SOUND-DESIGN.md) | OPL3, MIDI, AWE32 integration |
| [Graphics Design](docs/06-GRAPHICS-DESIGN.md) | qemu-3dfx, VFIO, display strategy |
| [CD-ROM Design](docs/07-CDROM-DESIGN.md) | CUE/BIN parser, audio tracks |
| [Machine Profiles](docs/08-MACHINE-PROFILES.md) | The four target PCs |
| [Business](docs/09-BUSINESS.md) | Licensing and monetization |
| [Decisions](docs/DECISIONS.md) | Architecture Decision Records |
| [Changelog](docs/CHANGELOG.md) | Version history |

## License

GPL v2.0 (inherits from QEMU). See [LICENSE](LICENSE).

All integrated components are GPL-compatible:
- QEMU: GPL v2.0
- qemu-3dfx patches: GPL v2.0
- Nuked-OPL3: LGPL v2.1
- Munt: GPL v2.0+ / LGPL v2.1
- FluidSynth: LGPL v2.1
- libcue: GPL v2.0

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Credits

QEMU-SA builds on the work of:
- [QEMU Project](https://www.qemu.org/) — the foundation
- [kjliew/qemu-3dfx](https://github.com/kjliew/qemu-3dfx) — 3Dfx Glide/OpenGL passthrough
- [nukeykt/Nuked-OPL3](https://github.com/nukeykt/Nuked-OPL3) — YMF262 emulation
- [munt/munt](https://github.com/munt/munt) — Roland MT-32/CM-32L emulation
- [FluidSynth](https://www.fluidsynth.org/) — SoundFont MIDI synthesis
- [86Box](https://86box.net/) — inspiration for machine accuracy
- [DOSBox-Staging](https://dosbox-staging.github.io/) — inspiration for sound integration
