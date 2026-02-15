# Architecture Decision Records

This document tracks significant technical and strategic decisions for QEMU-SA. Each decision records the context, options considered, and rationale.

---

## ADR-001: Fork QEMU Instead of Building Standalone

**Date:** 2025-02 (project inception)
**Status:** Accepted

**Context:** We need a platform for retro PC gaming that spans DOS through Windows XP with proper sound, 3D graphics, and CD audio.

**Options:**
1. Fork QEMU and add features to its device model
2. Build a standalone emulator from scratch
3. Create a wrapper/orchestrator around existing emulators (86Box, DOSBox, QEMU)

**Decision:** Fork QEMU.

**Rationale:**
- QEMU already has KVM support (critical for Win98/XP performance)
- QEMU already has VFIO (GPU passthrough for XP)
- QEMU's device model architecture makes it straightforward to add ISA/PCI devices
- qemu-3dfx patches already exist for QEMU
- Building from scratch would take years with no advantage
- A wrapper adds complexity without integration (sound can't mix across emulators)

---

## ADR-002: Nuked-OPL3 Over Stock OPL2

**Date:** 2025-02
**Status:** Accepted

**Context:** QEMU's built-in OPL implementation is OPL2-only (YM3812 via fmopl.c from MAME). Games from 1992+ expect OPL3.

**Options:**
1. Improve stock fmopl.c to add OPL3 features
2. Replace with Nuked-OPL3 library
3. Use MAME's YMF262 implementation

**Decision:** Nuked-OPL3.

**Rationale:**
- Transistor-level accurate — reverse-engineered from die photos
- Same library used by DOSBox-Staging and 86Box (proven)
- Small, self-contained (2 files: opl3.c, opl3.h)
- LGPL 2.1 — compatible with QEMU's GPL v2
- Active maintenance by nukeykt

---

## ADR-003: MPU-401 UART Mode Only

**Date:** 2025-02
**Status:** Accepted

**Context:** The MPU-401 has two modes: Intelligent and UART. Intelligent mode includes hardware-based MIDI recording, timestamping, and complex handshaking.

**Decision:** Implement UART mode only.

**Rationale:**
- Every DOS game that uses MPU-401 works in UART mode
- Intelligent mode was used for professional MIDI sequencing, not gaming
- UART mode is simple: port 330h data, port 331h status, minimal protocol
- DOSBox, 86Box, and all other emulators also only implement UART mode

---

## ADR-004: Separate MIDI Backends (Munt vs FluidSynth vs Host)

**Date:** 2025-02
**Status:** Accepted

**Context:** Different games target different MIDI devices. We need to support MT-32 (1987-1993 games), General MIDI (1993+ games), and potentially real hardware.

**Decision:** Implement a backend interface with pluggable implementations.

**Rationale:**
- MT-32 and GM produce very different results — wrong device sounds terrible
- Users need to choose per-game which MIDI device to emulate
- Backend abstraction makes it easy to add more devices later
- Each backend has different dependencies (Munt needs ROMs, FluidSynth needs SoundFont)

---

## ADR-005: KVM for Win98/XP, TCG for DOS

**Date:** 2025-02
**Status:** Accepted

**Context:** DOS games need approximate speed control (can't run at full host CPU speed). Win98/XP games want maximum performance.

**Decision:**
- DOS/Win3.11 profiles use TCG (software emulation) with icount for speed control
- Win98/XP profiles use KVM (hardware virtualization) for native speed

**Rationale:**
- KVM runs guest code directly on the CPU — approaching native performance
- TCG with icount provides a crude speed throttle for older software
- This mirrors what the games actually need: slow CPU for old games, fast CPU for newer ones
- Splitting by profile keeps it simple

---

## ADR-006: Accept TCG Limitations (Not Cycle-Accurate)

**Date:** 2025-02
**Status:** Accepted

**Context:** QEMU's TCG counts instructions, not CPU cycles. A NOP and a DIV take the same "time." This means ~10-15% of DOS games with cycle-sensitive code may not work.

**Decision:** Accept this limitation and document it honestly. Recommend 86Box for affected games.

**Rationale:**
- Making TCG cycle-accurate would require rewriting QEMU's entire CPU emulation engine
- This is a multi-year, possibly multi-decade effort
- 85-90% of DOS games work fine without cycle accuracy
- 86Box already solves this problem — no need to duplicate their work
- Honest documentation builds trust with the community

---

## ADR-007: QEMU 9.2.x as Fork Base

**Date:** 2025-02
**Status:** Accepted

**Context:** We need a stable QEMU version to fork from. qemu-3dfx patches target specific QEMU versions.

**Decision:** Fork from QEMU 9.2.2 (latest stable at project start).

**Rationale:**
- 9.2.x is current stable release
- qemu-3dfx maintains patches for recent QEMU versions
- Newer than 9.2 risks instability; older misses fixes
- We track QEMU releases and rebase periodically

---

## ADR-008: GPL v2.0 License

**Date:** 2025-02
**Status:** Accepted (no choice)

**Context:** QEMU is GPL v2.0. qemu-3dfx is GPL v2.0. Combined work inherits GPL v2.0.

**Decision:** GPL v2.0 for the entire QEMU-SA project.

**Rationale:**
- Legal requirement — not a choice
- All integrated libraries are compatible (LGPL 2.1 can link with GPL 2.0)
- Monetization via convenience/services, not proprietary licensing

---

## ADR-009: Arch Linux for Development Environment

**Date:** 2025-02
**Status:** Accepted

**Context:** Need a Linux distribution for both the development notebook (P16v) and the build/test server (TD350).

**Options:**
1. Arch Linux (rolling release)
2. Fedora (semi-rolling, Red Hat ecosystem)
3. Ubuntu LTS (stable, well-documented)

**Decision:** Arch Linux on both machines.

**Rationale:**
- Rolling release = always latest QEMU dependencies, kernel, NVIDIA drivers
- AUR provides bleeding-edge packages (munt-mt32emu, etc.)
- Minimal base — install only what's needed
- Arch Wiki is the best Linux documentation
- Developer is familiar with Arch
- Same distro on both machines simplifies toolchain management

---

## ADR-010: Anti-Cheat Evasion Approach

**Date:** 2025-02
**Status:** Proposed

**Context:** Some XP-era games detect VMs and refuse to run (anti-cheat, DRM). QEMU exposes VM indicators via CPUID, SMBIOS, ACPI, and device names.

**Options:**
1. Provide VM masking as built-in feature
2. Document masking techniques but don't automate
3. Ignore the problem

**Decision:** Document techniques in machine profiles. Don't build specific anti-cheat bypass tools.

**Rationale:**
- VM masking is a cat-and-mouse game with diminishing returns
- QEMU already supports CPUID masking, SMBIOS strings, Hyper-V enlightenments
- We document what works and let users configure per-game
- Building "anti-detection" tools sends the wrong message (we're not a cheat tool)
- VFIO passthrough inherently avoids most detection (real hardware, real drivers)

---

*Add new decisions above this line.*
