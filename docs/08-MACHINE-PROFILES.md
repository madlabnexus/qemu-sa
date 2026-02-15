# 08 — Machine Profiles

## Concept

Machine profiles are pre-configured hardware combinations that replicate the ideal gaming PC for each era. Instead of memorizing 15+ QEMU flags, users select a profile:

```bash
qemu-system-i386 --machine dos-gaming -hda dos.qcow2
```

## Profile 1: DOS Gaming PC (`dos-gaming`)

**Era:** 1987–1996
**Target:** DOOM, Duke Nukem 3D, Monkey Island, Wing Commander, TIE Fighter

### Hardware Emulated

| Component | Setting | Real Equivalent |
|-----------|---------|-----------------|
| Machine | pc-i440fx (PIIX3) | Generic 486 motherboard |
| CPU | 486-v1 via TCG | Intel 486DX2-66 |
| CPU Speed | icount shift=4 (tunable) | ~33-66 MHz equivalent |
| RAM | 32M | 32MB EDO DRAM |
| Sound | SB16 + Nuked-OPL3 + MPU-401 | Sound Blaster AWE32 |
| MIDI Backend | Munt (MT-32) or FluidSynth (GM) | Roland MT-32 / SC-55 |
| Display | Cirrus GD5446 | Tseng ET4000 / S3 Trio |
| CD-ROM | ATAPI + CUE/BIN | 2x-8x CD-ROM drive |
| Floppy | 1.44MB FDD | 3.5" floppy |
| HDD | IDE (qcow2) | 500MB-2GB IDE drive |
| Mouse | PS/2 serial mouse | Microsoft/Logitech serial |
| Joystick | — (future) | Gravis GamePad |

### Equivalent Command Line

```bash
qemu-system-i386 \
  -machine pc-i440fx-9.2,accel=tcg \
  -cpu 486-v1 \
  -icount shift=4,align=off,sleep=on \
  -m 32M \
  -device sb16,audiodev=audio0 \
  -audiodev pa,id=audio0 \
  -opl3 nuked \
  -midi mt32,romdir=~/vms/roms/mt32 \
  -vga cirrus \
  -drive file=dos622.qcow2,format=qcow2,if=ide \
  -cdrom game.cue \
  -boot order=c \
  -display sdl
```

### Guest Setup Notes

- DOS 6.22 with MSCDEX for CD-ROM access
- `SET BLASTER=A220 I5 D1 T6` in AUTOEXEC.BAT
- CTMOUSE or CuteMouse for mouse driver
- SBMIDI utility to select MPU-401 in games

### icount Tuning

The `shift` value controls approximate CPU speed:
- `shift=6` ≈ slow 386 (8-12 MHz feel)
- `shift=5` ≈ 386DX-33 to 486SX-25
- `shift=4` ≈ 486DX2-66 (recommended default)
- `shift=3` ≈ Pentium 75-90
- `shift=2` ≈ Pentium 133+

**Warning:** icount is NOT cycle-accurate. It counts instructions, not CPU cycles. Games with hardware timing loops or ISA bus timing dependencies may behave incorrectly. For those games, use 86Box.

## Profile 2: Windows 3.11 PC (`win311`)

**Era:** 1990–1995
**Target:** SimCity 2000, Myst, SkiFree, Lemmings for Windows

### Hardware Emulated

| Component | Setting | Real Equivalent |
|-----------|---------|-----------------|
| Machine | pc-i440fx | Pentium motherboard |
| CPU | pentium-v1 via TCG | Intel Pentium 75 |
| RAM | 32M | 32MB SIMM |
| Sound | SB16 + Nuked-OPL3 + MPU-401 | Sound Blaster 16 |
| Display | Cirrus GD5446 | S3 Trio64V+ |
| CD-ROM | ATAPI + CUE/BIN | 4x CD-ROM |
| HDD | IDE (qcow2) | 1GB IDE |
| Network | ne2k_pci | NE2000 compatible |

### Guest Setup Notes

- Windows for Workgroups 3.11
- Install Cirrus VGA drivers for SVGA modes
- SB16 drivers from Creative
- WinG for early Windows games

## Profile 3: Windows 98 + 3Dfx (`win98-3dfx`)

**Era:** 1996–2001 — THE CROWN JEWEL
**Target:** Unreal, Half-Life, Quake II, StarCraft, Diablo II, Need for Speed

### Hardware Emulated

| Component | Setting | Real Equivalent |
|-----------|---------|-----------------|
| Machine | pc-i440fx, **KVM** | Intel i440BX |
| CPU | host (KVM passthrough) | Pentium III 800MHz+ |
| RAM | 512M | 512MB SDRAM |
| Sound | AC97 (ich9-intel-hda) | Realtek AC97 |
| Graphics | **qemu-3dfx** | 3Dfx Voodoo3 / Banshee |
| CD-ROM | ATAPI + CUE/BIN | 24x CD-ROM |
| HDD | IDE (qcow2) | 20GB IDE |
| Network | rtl8139 | Realtek 8139 |
| USB | UHCI | USB 1.1 |

### Equivalent Command Line

```bash
qemu-system-i386 \
  -machine pc-i440fx-9.2,accel=kvm \
  -cpu host \
  -m 512M \
  -device AC97,audiodev=audio0 \
  -audiodev pa,id=audio0 \
  -device 3dfx-glide \
  -vga std \
  -drive file=win98se.qcow2,format=qcow2,if=ide \
  -cdrom game.cue \
  -netdev user,id=net0 \
  -device rtl8139,netdev=net0 \
  -usb \
  -boot order=c \
  -display sdl,gl=on
```

### Guest Setup Notes

1. Install Windows 98 SE (Second Edition — required for USB, better drivers)
2. Install SigmaTel AC97 WDM driver for audio
3. Install qemu-3dfx guest drivers (Voodoo + OpenGL ICD)
4. Install SoftGPU for 2D/DirectDraw acceleration (optional)
5. Install DirectX 9.0c (June 2010 redistributable for max compat)

### Performance

KVM gives native CPU speed. The qemu-3dfx passthrough handles Glide and OpenGL at excellent framerates for era-appropriate resolutions. This configuration has **no competition** — nothing else provides KVM + 3Dfx for Win98.

## Profile 4: Windows XP + VFIO (`winxp-vfio`)

**Era:** 2001–2008
**Target:** Morrowind, GTA Vice City, Far Cry, Doom 3, Half-Life 2, WoW vanilla

### Hardware Emulated

| Component | Setting | Real Equivalent |
|-----------|---------|-----------------|
| Machine | pc-q35, **KVM** | Intel ICH9 |
| CPU | host (KVM) | Athlon XP / P4 equiv |
| RAM | 4G | 4GB DDR400 |
| Sound | Intel HDA (ich9) | Realtek HD Audio |
| Graphics | **VFIO passthrough** | Physical GPU |
| CD/DVD | ATAPI + CUE/BIN | DVD-ROM |
| HDD | VirtIO (qcow2) | SATA drive |
| Network | VirtIO | Gigabit NIC |
| USB | EHCI + xHCI | USB 2.0 |

### Equivalent Command Line

```bash
qemu-system-x86_64 \
  -machine q35,accel=kvm \
  -cpu host,kvm=on,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic \
  -m 4G \
  -device intel-hda -device hda-duplex,audiodev=audio0 \
  -audiodev pa,id=audio0 \
  -device vfio-pci,host=XX:00.0,multifunction=on \
  -device vfio-pci,host=XX:00.1 \
  -vga none \
  -drive file=winxp.qcow2,format=qcow2,if=virtio \
  -cdrom game.cue \
  -netdev user,id=net0 \
  -device virtio-net-pci,netdev=net0 \
  -usb -device usb-ehci \
  -boot order=c
```

### Guest Setup Notes

1. Install Windows XP SP3
2. Install VirtIO drivers (Red Hat, for disk and network)
3. Install GPU drivers (NVIDIA ForceWare or AMD Catalyst — depends on passed GPU)
4. Install VDMSound for DOS games within XP
5. Install DirectX 9.0c

### Anti-Cheat / VM Detection

Some XP-era games with anti-cheat detect VMs and refuse to run. Mitigations:
- Hyper-V enlightenments mask some VM indicators
- CPU model spoofing via `-cpu host,kvm=on,-hypervisor`
- SMBIOS strings: `-smbios type=1,manufacturer=Dell,product=OptiPlex`
- ACPI table customization

See [DECISIONS.md](DECISIONS.md) for anti-cheat policy discussion.

## Profile Config Files

Profiles are stored in `configs/machines/` as QEMU configuration files:

```
configs/machines/
├── dos-gaming.cfg
├── win311.cfg
├── win98-3dfx.cfg
└── winxp-vfio.cfg
```

Format follows QEMU's `-readconfig` syntax:

```ini
# dos-gaming.cfg
[machine]
  type = "pc-i440fx-9.2"
  accel = "tcg"

[cpu]
  model = "486-v1"

[memory]
  size = "32M"

[device "sb16"]
  driver = "sb16"
  audiodev = "audio0"

[audiodev "audio0"]
  driver = "pa"
```

## Per-Game Overrides

Some games need specific tweaks. A game database (CSV or JSON) maps game names to profile overrides:

```json
{
  "doom": {
    "profile": "dos-gaming",
    "icount_shift": 4,
    "opl3": "nuked",
    "midi": "none",
    "notes": "Pure OPL3 music, no MIDI needed"
  },
  "monkey-island-2": {
    "profile": "dos-gaming",
    "midi": "mt32",
    "notes": "MT-32 required for intended soundtrack"
  },
  "half-life": {
    "profile": "win98-3dfx",
    "ram": "256M",
    "notes": "OpenGL mode via qemu-3dfx"
  }
}
```

This is Phase 4 functionality. Start simple, iterate.
