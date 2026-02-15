# 09 — Business Model & Licensing

## License Reality

QEMU is licensed under **GPL v2.0**. The qemu-3dfx patches are **GPL v2.0**. When combined, the result MUST be GPL v2.0. This is not optional — it's a legal requirement.

### What GPL v2.0 Means

- Anyone who receives a binary of QEMU-SA has the right to receive the source code
- Anyone who receives the source can modify and redistribute it
- You cannot add restrictions beyond GPL (no DRM, no "no commercial use" clauses)
- You CAN sell GPL software — but you must provide source to buyers

### What We CANNOT Do

- ❌ Sell QEMU-SA as proprietary / closed-source software
- ❌ Prevent anyone from redistributing our binaries
- ❌ Bundle copyrighted BIOS ROMs or OS images (DOS, Windows)
- ❌ Bundle copyrighted MT-32 ROMs
- ❌ Add telemetry or licensing checks to the GPL codebase
- ❌ Use "open core" where core features are proprietary

## Monetization Strategies (GPL-Compatible)

### Strategy 1: Sell Convenience (Red Hat Model)

**Concept:** The source code is free. Pre-built, tested, ready-to-use packages cost money.

**Products:**
- Pre-built binaries with installers (Windows, Linux)
- Pre-configured VM images (just add your own OS install disc)
- One-click setup that takes 5 minutes vs 4 hours of DIY
- Automatic updates and version management

**Price point:** $15-30 one-time or $5-10/month

**Why it works:** Most retro gaming enthusiasts value their time. Building QEMU from source, applying patches, configuring audio devices, setting up VFIO — this takes expertise. Convenience has clear value.

### Strategy 2: Cloud Gaming / SaaS (STRONGEST)

**Concept:** Run QEMU-SA on servers. Users access retro games via browser streaming. They never receive a binary — they receive a video stream.

**GPL compliance:** GPLv2 only requires source distribution to recipients of the binary. Streaming users don't receive the binary. This is the same model as Google (who runs heavily modified Linux without distributing source to users).

**Products:**
- Monthly subscription: $10-20/month
- Retro game streaming service
- Bring-your-own-games or curated library (licensing dependent)
- Low-latency streaming via WebRTC or Parsec-like protocol

**Why it works:** No setup, no hardware requirements, instant access from any device. The server handles VFIO, GPU passthrough, audio — everything.

**Challenges:** Latency (retro games are often fast-paced), content licensing, server costs (GPU servers are expensive).

### Strategy 3: Hardware Appliance

**Concept:** Physical device pre-loaded with QEMU-SA. Plug into TV, add your own game discs/images.

**Products:**
- Raspberry Pi-scale device with QEMU-SA pre-configured
- "Retro PC in a box" — x86 mini-PC with GPU for 3Dfx passthrough
- Retro arcade cabinet with QEMU-SA backend

**GPL compliance:** Source code provided on device or downloadable.

**Price point:** $200-500 depending on hardware

### Strategy 4: Dual Product

**Concept:** Free GPL QEMU-SA core + separate proprietary launcher/manager application.

**The launcher is NOT a derivative work of QEMU.** It's a separate program that:
- Manages VM images and configurations
- Provides a game library UI (think Steam for retro games)
- Downloads and applies updates
- Manages ROM and SoundFont files
- Handles cloud save sync
- Invokes QEMU-SA via command line (not linked against it)

**Legal basis:** Programs that invoke GPL software via exec/command-line are NOT derivative works. GCC is GPL — IDEs that invoke GCC are not required to be GPL.

**Price point:** Free basic launcher, $20-40 for premium features

### Strategy 5: Support & Consulting

**Concept:** Enterprise customers pay for setup, customization, and support.

**Customers:**
- Retro gaming cafés and bars
- Museums and exhibitions
- Game preservation organizations
- Companies with legacy software dependencies
- Educational institutions

**Services:**
- Custom QEMU-SA deployment
- Hardware specification and setup
- Game compatibility testing
- Ongoing maintenance contracts
- Training

**Price point:** $500-5000+ per engagement

## Recommended Strategy

### Phase 1: Build Community (Months 1-12)

- Fully open source, no monetization
- Build reputation on Vogons, Reddit, YouTube
- Attract contributors
- Establish QEMU-SA as THE retro PC gaming hypervisor

### Phase 2: Monetize Convenience (Months 12-18)

- Release proprietary launcher (Strategy 4)
- Pre-built binary packages (Strategy 1)
- Patreon/Sponsors for ongoing development

### Phase 3: Scale (Months 18+)

- Cloud gaming service (Strategy 2)
- Hardware appliance (Strategy 3)
- Consulting for institutions (Strategy 5)

## Content Licensing Considerations

### What We Can Bundle

- ✅ SeaBIOS (LGPL) — QEMU's default BIOS
- ✅ OVMF/TianoCore (BSD) — UEFI firmware
- ✅ FreeDOS (GPL) — free DOS
- ✅ FluidR3_GM.sf2 (MIT) — General MIDI SoundFont
- ✅ SoftGPU (MIT) — Windows 9x guest drivers

### What Users Must Provide

- ❌ MS-DOS (copyrighted by Microsoft)
- ❌ Windows 3.11, 98, XP (copyrighted)
- ❌ MT-32/CM-32L ROMs (copyrighted by Roland)
- ❌ Games (copyrighted by publishers)
- ❌ NVIDIA/AMD drivers (proprietary)
- ❌ AC97 drivers (some are copyrighted)

### Legal Safe Harbor

QEMU-SA is a tool. It does not include any copyrighted content. How users obtain their OS images, ROMs, and games is their responsibility. This is the same legal position as QEMU, VirtualBox, and VMware.
