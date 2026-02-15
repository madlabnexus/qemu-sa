# 06 — Graphics Design

## Graphics Strategy by Machine

| Machine | Display Adapter | Acceleration | How |
|---------|----------------|--------------|-----|
| DOS Gaming | Cirrus Logic GD5446 | None (software) | Stock QEMU `-vga cirrus` |
| Win 3.11 | Cirrus Logic GD5446 | None (software) | Stock QEMU `-vga cirrus` |
| Win 98 | **qemu-3dfx virtual device** | **Glide + OpenGL passthrough** | Patches from kjliew |
| Win XP | **VFIO passthrough GPU** | **Native GPU drivers** | Stock QEMU VFIO |

## DOS/Win3.11: Software Rendering

### Cirrus Logic GD5446

QEMU's built-in Cirrus VGA provides:
- VGA standard modes (320×200, 640×480, etc.)
- SVGA modes up to 1280×1024×16bpp
- Hardware cursor
- BitBlt acceleration (limited, used by Win3.11 drivers)

This covers all DOS games (VGA/SVGA) and Windows 3.11. No 3D acceleration needed — DOS games are software-rendered.

### Alternative: Standard VGA (`-vga std`)

For games that don't work with Cirrus quirks, standard VGA provides a cleaner, simpler implementation. Try this first, fall back to Cirrus if SVGA modes are needed.

### SoftGPU (Supplementary for Win9x)

SoftGPU is a Windows 9x guest driver project that provides:
- DirectDraw acceleration via software rendering
- Faster GDI operations
- Better SVGA mode support

Not critical for QEMU-SA but can improve Win98 desktop experience. Works alongside qemu-3dfx for 3D.

## Windows 98: qemu-3dfx (THE CROWN JEWEL)

### What It Is

qemu-3dfx by kjliew creates a virtual PCI device inside QEMU that intercepts 3Dfx Glide API calls and MESA OpenGL calls from the Windows guest, serializes them, and executes them on the host GPU using the host's OpenGL/Vulkan stack.

### How It Works

```
┌─────────────────────────────────┐
│ Windows 98 Guest                │
│                                 │
│ Game → Glide API / OpenGL       │
│           │                     │
│           ▼                     │
│ qemu-3dfx guest driver          │
│ (intercepts 3D calls)           │
│           │                     │
│     MMIO/PIO to virtual PCI     │
└───────────┼─────────────────────┘
            │ VM Exit (KVM)
            ▼
┌─────────────────────────────────┐
│ QEMU Host                       │
│                                 │
│ qemu-3dfx host wrapper          │
│ (deserializes 3D commands)      │
│           │                     │
│           ▼                     │
│ Host OpenGL / Vulkan            │
│           │                     │
│           ▼                     │
│ RTX 4090 (real GPU)             │
└─────────────────────────────────┘
```

### Supported APIs

| API | Version | Status |
|-----|---------|--------|
| 3Dfx Glide 2.x | 2.11-2.60 | ✅ Full support |
| 3Dfx Glide 3.x | 3.10 | ✅ Full support |
| MESA OpenGL | 1.1-2.1 | ✅ Good support |
| DirectDraw | 1-7 | Partial (via SoftGPU) |
| Direct3D | N/A | ❌ Not supported |

### Guest Drivers Required

Inside the Win98 VM, install:
1. **3dfx Voodoo driver** — qemu-3dfx provides a virtual Voodoo-compatible device
2. **MESA OpenGL32.dll** — qemu-3dfx's custom OpenGL ICD
3. **SoftGPU** (optional) — for better 2D/DirectDraw

### Integration with QEMU-SA

qemu-3dfx exists as a patch set against QEMU. Integration steps:

1. Apply patches to QEMU 9.2.2 source
2. Build with 3dfx support enabled
3. Verify: Boot Win98, install guest drivers, run a Glide game (e.g., Unreal, Tomb Raider)

The patches modify:
- `hw/` — new PCI device for 3dfx
- `ui/` — display output integration
- Build system — new compile flags

### Performance Expectations

With KVM + qemu-3dfx on RTX 4090:
- CPU: Near-native speed (KVM hardware virtualization)
- 3D: Very good for era-appropriate resolutions (640×480 to 1024×768)
- Glide games: Excellent (direct API translation)
- OpenGL games: Good (MESA 2.1 calls mapped to modern GL)

This is QEMU-SA's unique selling point. No other emulator provides this combination.

### Test Games

| Game | API | Era |
|------|-----|-----|
| Unreal (1998) | Glide 2.x | Peak Glide |
| Tomb Raider (1996) | Glide 2.x | Early Glide |
| Quake II (1997) | OpenGL 1.1 | OpenGL classic |
| Half-Life (1998) | OpenGL 1.1 | Must-work title |
| Diablo II (2000) | DirectDraw + Glide | Both paths |
| StarCraft (1998) | DirectDraw | 2D + SoftGPU |
| Need for Speed III (1998) | Glide 3.x | Late Glide |

## Windows XP: VFIO GPU Passthrough

### What It Is

VFIO (Virtual Function I/O) passes a physical PCI device directly to a VM. The guest OS sees the real GPU and uses native drivers. This provides full hardware acceleration — DirectX 9, OpenGL, everything.

### How It Works

```
┌──────────────────────────────┐
│ Windows XP Guest             │
│                              │
│ Game → DirectX 9 / OpenGL    │
│           │                  │
│           ▼                  │
│ NVIDIA ForceWare drivers     │
│ (native, real hardware)      │
│           │                  │
│     Direct PCI access        │
└───────────┼──────────────────┘
            │ IOMMU maps directly
            ▼
┌──────────────────────────────┐
│ Physical RTX 4090            │
│ (passed through via VFIO)    │
└──────────────────────────────┘
```

### Configuration

Already covered in [02-SETUP-SERVER.md](02-SETUP-SERVER.md) for VFIO binding. QEMU command:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -cpu host,kvm=on \
  -m 4G \
  -device vfio-pci,host=XX:00.0,multifunction=on \
  -device vfio-pci,host=XX:00.1 \
  -vga none \
  -nographic \
  ...
```

### Driver Compatibility

| GPU | XP Driver | Status |
|-----|-----------|--------|
| RTX 4090 | ForceWare ~391.xx (last XP) | ⚠️ May not support Ada arch |
| RX 6650 XT | Catalyst ~15.7 (last XP AMD) | ⚠️ May not support RDNA2 |

**Reality check:** Modern GPUs may not have XP-compatible drivers. If neither card works natively in XP, options include:
1. Use qemu-3dfx OpenGL path instead of VFIO
2. Acquire an older GPU (GTX 970, R9 380) for XP VFIO
3. Run XP games in Win98 VM where possible

This needs testing in Phase 0.

### Display Output

With VFIO, the GPU outputs to a physical monitor. Connect the RTX 4090's DisplayPort/HDMI to a monitor. The XP desktop appears on that monitor.

For remote/headless operation: Looking Glass can mirror the VFIO display to a window on the host, but adds complexity.

## Multi-Monitor Setup

```
Monitor 1: Aspeed BMC → TD350 server console (BIOS, emergency)
Monitor 2: RTX 4090 → VFIO primary VM (Win98 3dfx or WinXP)
Monitor 3: RX 6650 XT → VFIO secondary VM (or host display via amdgpu)
```

The P16v notebook has its own display for coding. Three external monitors on the server handle all VM output.
