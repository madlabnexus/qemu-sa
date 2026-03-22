# 06 — VM Reference Guide

## QEMU Command-Line Options Reference

This document explains every QEMU option used in QEMU-SA launch commands, why each is chosen, and what alternatives exist. It serves as both a decision record and a quick reference for building new VM configurations.

---

## Option Reference Table

### Machine & CPU

| Option | What It Does | Our Choice | Why | Alternatives |
|--------|-------------|------------|-----|-------------|
| `-machine` | Selects the virtual motherboard/chipset | `pc-i440fx-9.2` | i440FX is the classic Pentium-era chipset; version suffix pins to our QEMU build so behavior doesn't change across upgrades | `pc` (latest i440FX), `q35` (ICH9 — modern, not period-correct for DOS/Win98) |
| `-cpu` | Selects the emulated CPU model | `486-v1` (DOS/Win3.11), `host` (Win98/XP) | 486 gives period-correct instruction set and prevents speed-sensitive games from breaking; `host` with KVM gives native performance | `pentium`, `pentium2`, `pentium3`, `qemu64`, `max` |
| `-m` | RAM size | `32M` (DOS), `512M` (Win98), `1G`–`2G` (XP) | Matches period-correct RAM; DOS rarely needs >32MB, Win98 sweet spot is 512MB | Any power of 2; DOS max useful is 64M |
| `-enable-kvm` | Use hardware virtualization (KVM) | Win98/XP only | Native CPU speed — critical for 3D games; NOT used for DOS because games need speed throttling | Omit for TCG (software emulation) |
| `-smp` | Number of CPU cores | 1 (DOS), 1 (Win98), 1–2 (XP) | DOS is single-threaded; Win98 has limited SMP support; XP can use 2 cores | Up to host core count with KVM |

### Display

| Option | What It Does | Our Choice | Why | Alternatives |
|--------|-------------|------------|-----|-------------|
| `-display` | Host-side display backend | `gtk` (testing), `gtk,gl=on` (3dfx) | GTK works on Wayland (our desktop); `gl=on` required for qemu-3dfx OpenGL passthrough | `sdl`, `spice-app`, `vnc`, `none` |
| `-vga` | Guest-side video adapter | `cirrus` (DOS/Win98), `std` (XP) | Cirrus CL-GD5446 has excellent DOS/Win9x driver support; `std` (bochs-display) for XP VFIO setups | `virtio` (fastest but no DOS support), `vmware` (VMware SVGA), `qxl` (SPICE) |

### Storage

| Option | What It Does | Our Choice | Why | Alternatives |
|--------|-------------|------------|-----|-------------|
| `-hda` | Primary IDE hard disk (master on IDE0) | qcow2 image | qcow2 supports snapshots, compression, thin provisioning | `-drive file=X,format=raw` for raw images; `-drive if=virtio` for VirtIO (XP with driver) |
| `-fda` | Floppy drive A: | 1.44MB floppy image | Required for DOS install; floppy images are raw sector dumps | `-fdb` for drive B: |
| `-cdrom` | IDE CD-ROM (slave on IDE1) | ISO or CUE file | CUE/BIN support is a QEMU-SA addition for CD audio; stock QEMU only handles ISO | `-drive file=X,media=cdrom` for more control |
| `-boot` | Boot device order | `a` (floppy), `c` (hard disk), `d` (CD-ROM) | `a` for install from floppy, `c` for normal boot, `d` for CD install (Win98) | `order=dc` (try CD first, then disk) |

### Sound Devices

| Option | What It Does | Our Choice | Why | Alternatives |
|--------|-------------|------------|-----|-------------|
| `-device sb16` | Sound Blaster 16 ISA card | DOS/Win3.11 machines | SB16 is the universal DOS sound standard; provides PCM/DMA playback | `-device sb16,iobase=0x220,irq=5,dma=1,hdma=5` (explicit settings matching SET BLASTER) |
| `-device nuked-opl3` | Yamaha YMF262 OPL3 FM synth (QEMU-SA) | DOS/Win3.11 machines | Transistor-level accurate FM synthesis; replaces stock OPL2 | Omit to use stock `adlib` (OPL2 only) |
| `-device mpu401` | MPU-401 UART MIDI interface (QEMU-SA) | DOS/Win3.11 machines | Routes MIDI to Munt or FluidSynth backends | Properties: `backend=mt32\|gm\|host`, `config=<path>` |
| `-device ac97` | Intel ICH AC97 codec | Win98/XP machines | Standard PCI audio; Win98 needs SigmaTel WDM driver for SB emulation in DOS box | `-device intel-hda -device hda-duplex` (HD Audio — XP/later) |
| `-audiodev` | Host audio backend | `pa,id=snd0` (PulseAudio) | PulseAudio is the default on Arch/GNOME; PipeWire's PulseAudio layer also works | `alsa,id=snd0`, `sdl,id=snd0`, `pipewire,id=snd0` |

### Sound Device Properties (QEMU-SA Specific)

| Device | Property | Values | Default | Notes |
|--------|----------|--------|---------|-------|
| `sb16` | `iobase` | `0x220`, `0x240`, `0x260`, `0x280` | `0x220` | Must match `SET BLASTER=A220` in DOS |
| `sb16` | `irq` | `5`, `7`, `9`, `10` | `5` | Must match `SET BLASTER=I5` |
| `sb16` | `dma` | `0`, `1`, `3` | `1` | 8-bit DMA, must match `SET BLASTER=D1` |
| `sb16` | `hdma` | `5`, `6`, `7` | `5` | 16-bit DMA, must match `SET BLASTER=H5` |
| `nuked-opl3` | *(none yet)* | — | — | Uses fixed AdLib ports 388h–38Bh |
| `mpu401` | `backend` | `mt32`, `gm`, `host` | *(required)* | Selects MIDI synthesis engine |
| `mpu401` | `config` | file path | *(required for gm/mt32)* | SoundFont path (gm) or ROM dir (mt32) |

### Network

| Option | What It Does | Our Choice | Why | Alternatives |
|--------|-------------|------------|-----|-------------|
| `-net nic,model=` | Guest network adapter | `rtl8139` (Win98), `e1000` (XP) | RTL8139 has native Win98 drivers; e1000 has native XP drivers | `virtio-net-pci` (fastest, needs VirtIO driver), `ne2k_pci`, `pcnet` |
| `-net user` | SLIRP user-mode networking | All machines | Simple NAT — guest can reach internet, no host config needed | `-netdev bridge` (bridged, requires host setup), `-netdev tap` (advanced) |

### Miscellaneous

| Option | What It Does | Our Choice | Why | Alternatives |
|--------|-------------|------------|-----|-------------|
| `-rtc base=localtime` | Set real-time clock to local time | All machines | DOS and Windows expect local time, not UTC; without this, clock is wrong | `base=utc` (Linux guests) |
| `-no-reboot` | Exit QEMU on guest reboot | During install | Prevents accidental reboot loop during OS setup | Omit for normal use |
| `-snapshot` | Write changes to temp file, discard on exit | Testing | Safe experimentation — disk image stays clean | Omit to persist changes |
| `-monitor stdio` | QEMU monitor on terminal | Development | Allows floppy swap, device hotplug, debugging from terminal | `-monitor telnet::4444,server,nowait` |

---

## Launch Scenarios

### Scenario 1: DOS 6.22 Install (from floppy)

**Goal:** Install MS-DOS 6.22 from three floppy disk images onto a fresh qcow2 disk.

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -rtc base=localtime \
  -hda ~/vms/dos622/dos622.qcow2 \
  -fda ~/vms/dos622/disk1.img \
  -boot a
```

**Why these options:**
- `486-v1` — period-correct CPU for DOS; prevents timing issues
- `32M` — more than enough for DOS; some games need expanded/extended memory
- `cirrus` — CL-GD5446 has good DOS VESA support
- `rtc base=localtime` — DOS reads RTC directly, expects local time
- `-boot a` — boot from floppy to start the installer

**Floppy swap during install:** Use the GTK menu (Machine → Removable Media) to swap disk1.img → disk2.img → disk3.img when prompted.

### Scenario 2: DOS Baseline (stock QEMU sound — no QEMU-SA devices)

**Goal:** Verify DOS games work with stock SB16 before testing our custom devices.

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -rtc base=localtime \
  -device sb16,iobase=0x220,irq=5,dma=1,hdma=5 \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot c
```

**Why these options:**
- Stock `sb16` only — no nuked-opl3 or mpu401
- This uses QEMU's built-in OPL2 (fmopl.c) through the SB16 device
- If a game produces PCM sound (Pinball Fantasies), it works here
- OPL music will play but sound thinner (OPL2 vs OPL3)

### Scenario 3: DOS with Nuked-OPL3 (QEMU-SA FM synthesis)

**Goal:** Test our Nuked-OPL3 device — FM music should sound fuller with stereo and OPL3 waveforms.

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -rtc base=localtime \
  -device sb16,iobase=0x220,irq=5,dma=1,hdma=5 \
  -device nuked-opl3 \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot c
```

**What changed:** Added `-device nuked-opl3`. This registers a separate ISA device on ports 388h–38Bh that intercepts AdLib writes and routes them through Nuked-OPL3 instead of the stock OPL2 emulation.

### Scenario 4: DOS with OPL3 + General MIDI (full QEMU-SA sound)

**Goal:** Test the complete Phase 1 audio pipeline — OPL3 for FM, MPU-401 → FluidSynth for MIDI.

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -rtc base=localtime \
  -device sb16,iobase=0x220,irq=5,dma=1,hdma=5 \
  -device nuked-opl3 \
  -device mpu401,backend=gm,config=/usr/share/soundfonts/FluidR3_GM.sf2 \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot c
```

**What changed:** Added `-device mpu401,backend=gm,config=...`. Games configured for General MIDI will send MIDI data to port 330h, the MPU-401 device assembles messages, and FluidSynth renders them using the FluidR3 SoundFont.

### Scenario 5: DOS with OPL3 + MT-32 (Roland MT-32 emulation)

**Goal:** Test Munt MT-32 backend — for games composed specifically for the MT-32 (1987–1993).

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -rtc base=localtime \
  -device sb16,iobase=0x220,irq=5,dma=1,hdma=5 \
  -device nuked-opl3 \
  -device mpu401,backend=mt32,config=~/vms/roms/mt32 \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot c
```

**What changed:** `backend=mt32` and `config=` points to the directory containing MT-32 ROM files (`MT32_CONTROL.ROM` + `MT32_PCM.ROM`). ROM files are copyrighted by Roland — user must provide their own.

### Scenario 6: Windows 98 SE Install (from CD)

**Goal:** Install Win98 SE with KVM acceleration and AC97 audio.

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -enable-kvm \
  -cpu host \
  -m 512M \
  -display gtk,gl=on \
  -vga cirrus \
  -rtc base=localtime \
  -device ac97 \
  -hda ~/vms/win98/win98se.qcow2 \
  -cdrom ~/vms/win98/win98se.iso \
  -net nic,model=rtl8139 -net user \
  -boot d
```

**Why these options:**
- `-enable-kvm -cpu host` — native CPU speed, essential for Win98/XP gaming
- `gl=on` — required for qemu-3dfx OpenGL passthrough to host GPU
- `ac97` — Intel ICH audio; needs SigmaTel WDM driver for SB emulation in DOS box
- `rtl8139` — Win98 has native RTL8139 drivers (no driver disk needed)
- `-boot d` — boot from CD-ROM for installation

### Scenario 7: Windows 98 SE Normal Boot (post-install)

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -enable-kvm \
  -cpu host \
  -m 512M \
  -display gtk,gl=on \
  -vga cirrus \
  -rtc base=localtime \
  -device ac97 \
  -hda ~/vms/win98/win98se.qcow2 \
  -cdrom ~/vms/win98/win98se.iso \
  -net nic,model=rtl8139 -net user \
  -boot c
```

**What changed:** `-boot c` (hard disk) instead of `-boot d` (CD-ROM). CD-ROM kept attached for driver installs.

---

## SB16 Environment Variable

DOS games detect the Sound Blaster through the `BLASTER` environment variable. This must match the `-device sb16` settings exactly.

```
SET BLASTER=A220 I5 D1 H5 T6
```

| Field | Meaning | Value | Maps to `-device sb16,...` |
|-------|---------|-------|--------------------------|
| `A` | Base I/O address | `220` | `iobase=0x220` |
| `I` | IRQ | `5` | `irq=5` |
| `D` | 8-bit DMA | `1` | `dma=1` |
| `H` | 16-bit DMA | `5` | `hdma=5` |
| `T` | Card type | `6` | SB16 (informational only) |

---

## Disk Image Formats

| Format | Pros | Cons | Use When |
|--------|------|------|----------|
| `qcow2` | Snapshots, thin provisioning, compression | Slightly slower than raw | Default for all VMs |
| `raw` | Maximum I/O performance | No snapshots, allocates full size | Performance-critical XP VMs |
| `vhd`/`vhdx` | Compatible with 86Box, Hyper-V | No QEMU-native snapshots | Sharing images with 86Box |

### Creating Disk Images

```bash
# DOS VM (504MB is plenty)
qemu-img create -f qcow2 ~/vms/dos622/dos622.qcow2 504M

# Win98 VM (4GB — Win98 + games + drivers)
qemu-img create -f qcow2 ~/vms/win98/win98se.qcow2 4G

# WinXP VM (20GB — XP + games + VFIO overhead)
qemu-img create -f qcow2 ~/vms/winxp/winxp.qcow2 20G
```

---

## Audio Backend Configuration

QEMU-SA sound devices need a host audio backend. If audio doesn't work, check this first.

```bash
# Check what audio backends the build supports
~/Projects/qemu-build/qemu-system-i386 -audio help

# Explicit PulseAudio backend (usually auto-detected)
-audiodev pa,id=snd0 -device sb16,audiodev=snd0

# ALSA backend (if PulseAudio isn't running)
-audiodev alsa,id=snd0 -device sb16,audiodev=snd0
```

On our Arch + GNOME + PipeWire setup, QEMU auto-detects PulseAudio through PipeWire's compatibility layer. No explicit `-audiodev` should be needed, but it's the first thing to add if sound is silent.

---

## Transfer Disk (Getting Files Into DOS)

DOS VMs can't use network shares easily. We use a FAT16 floppy image as a transfer mechanism.

```bash
# Create a 32MB FAT16 disk image
dd if=/dev/zero of=~/vms/dos622/transfer.img bs=1M count=32
mkfs.fat -F 16 ~/vms/dos622/transfer.img

# Mount on host, copy files in
sudo mkdir -p /mnt/transfer
sudo mount -o loop ~/vms/dos622/transfer.img /mnt/transfer
sudo cp -r /path/to/game/* /mnt/transfer/
sudo umount /mnt/transfer

# Attach to VM as second floppy or second hard disk
-fdb ~/vms/dos622/transfer.img    # appears as B: in DOS
# or
-hdb ~/vms/dos622/transfer.img    # appears as D: in DOS
```

**Size limit:** FAT16 floppy images max out at ~32MB. For larger transfers, use `-hdb` which presents it as a second hard disk (no FAT16 size limit on hard disks, but DOS may need a partition table).

---

## Quick Copy-Paste Commands

### Check build is working
```bash
~/Projects/qemu-build/qemu-system-i386 --version
~/Projects/qemu-build/qemu-system-i386 -device help 2>&1 | grep -i 'mpu401\|nuked\|sb16\|ac97'
```

### Check audio support
```bash
~/Projects/qemu-build/qemu-system-i386 -audio help
```

### Snapshot management
```bash
# Take a snapshot inside a running VM (QEMU monitor: Ctrl+Alt+2)
savevm clean-install

# Restore snapshot
loadvm clean-install

# List snapshots
qemu-img info ~/vms/dos622/dos622.qcow2
```
