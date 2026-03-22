# QEMU-SA Testing Guide: DOS & Win98 VMs

**Last updated:** 2026-03-22
**Branch:** `qemu-sa/phase1` on `qemu-upstream`
**Binary:** `~/Projects/qemu-build/qemu-system-i386`

---

## Prerequisites

### Software Already Installed

| Item | Location | Version |
|------|----------|---------|
| QEMU-SA binary | `~/Projects/qemu-build/qemu-system-i386` | 9.2.4 + qemu-3dfx + QEMU-SA sound |
| FluidR3 SoundFont | `/usr/share/soundfonts/FluidR3_GM.sf2` | 3.1 (pacman: soundfont-fluid) |
| qemu-3dfx guest wrappers | `~/Projects/qemu-3dfx/` | Already cloned |
| libmt32emu | System library | 2.7.2 (pacman: munt) |
| libfluidsynth | System library | 2.5.3 (pacman: fluidsynth) |

### Media You Must Provide

| Item | Notes |
|------|-------|
| DOS 6.22 install floppies | 3x 1.44MB .img files (disk1.img, disk2.img, disk3.img) |
| Windows 98 SE ISO | Must be Second Edition for AC97 driver support |
| DOOM (Shareware or Full) | OPL3 FM music test — get `doom19s.zip` from archive.org if needed |
| Pinball Fantasies | Your existing game files — SB16 PCM baseline test |
| MT-32 ROMs (optional) | `MT32_CONTROL.ROM` + `MT32_PCM.ROM` — copyrighted, dump from hardware you own |

### Drivers to Download for Win98

| Driver | Purpose | Where to Find |
|--------|---------|---------------|
| SigmaTel AC97 WDM (`stac9708.exe`) | Win98 audio (AC97 codec) | Search: "SigmaTel C-Major AC97 WDM driver" |
| DirectX 7.0a (`dx70a_full.exe`) | Game compatibility | archive.org |
| qemu-3dfx guest DLLs | Glide/OpenGL passthrough | `~/Projects/qemu-3dfx/wrappers/` |

**Note:** DOS does not need separate drivers. Our QEMU-SA devices (SB16, OPL3, MPU-401) are configured via `AUTOEXEC.BAT` environment variables and detected by games at their standard I/O ports.

---

## Important: qemu-3dfx Is Always Active

The qemu-3dfx Glide and Mesa/OpenGL passthrough devices are **hardwired into QEMU's PC machine board** (`hw/i386/pc.c`). They are always present in every i386 VM — no `-device` flag needed. The only requirement is installing the **guest-side wrapper DLLs** inside the VM (for Win98).

---

## Part 1: DOS 6.22 VM

### 1.1 Create VM Directory and Disk Image

```bash
mkdir -p ~/vms/dos622
cd ~/vms/dos622

# Create 504MB hard disk (max for DOS without LBA)
~/Projects/qemu-build/qemu-img create -f qcow2 dos622.qcow2 504M
```

### 1.2 Place Floppy Images

Copy your DOS 6.22 floppy images:

```bash
# Name them consistently
cp /path/to/your/disk1.img ~/vms/dos622/disk1.img
cp /path/to/your/disk2.img ~/vms/dos622/disk2.img
cp /path/to/your/disk3.img ~/vms/dos622/disk3.img
```

### 1.3 Install DOS 6.22

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -fda ~/vms/dos622/disk1.img \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot a
```

**Step-by-step inside the VM:**

1. DOS Setup starts. Choose to configure hard disk.
2. **FDISK:**
   - Select "1. Create DOS partition"
   - Select "1. Create Primary DOS Partition"
   - "Do you wish to use the maximum available size?" → **Y**
   - "Do you want to make the partition active?" → **Y**
   - FDISK will ask to reboot — press any key
3. **Reboot with same command** (close QEMU, run the command again)
4. DOS Setup will say the disk isn't formatted:
   - FORMAT C: /S → **Y** to proceed
   - This formats and makes it bootable
5. DOS setup continues — it copies files from Disk 1
6. **When it asks for Disk 2:**
   - Press **Ctrl+Alt+2** in QEMU window (switches to monitor console)
   - Type: `change floppy0 /home/madlabn/vms/dos622/disk2.img`
   - Press **Ctrl+Alt+1** to switch back to guest
   - Press Enter in the guest to continue
7. **When it asks for Disk 3:** repeat step 6 with `disk3.img`
8. Setup completes. Remove floppy and reboot:
   - Ctrl+Alt+2 → type `eject floppy0` → Ctrl+Alt+1
   - Let it reboot (or close QEMU and boot from HD)

### 1.4 Configure DOS (CONFIG.SYS and AUTOEXEC.BAT)

Boot into DOS (from hard disk):

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot c
```

**Edit CONFIG.SYS:**

```
C:\> EDIT CONFIG.SYS
```

Enter these contents:

```
DEVICE=C:\DOS\HIMEM.SYS
DEVICE=C:\DOS\EMM386.EXE RAM
DOS=HIGH,UMB
FILES=40
BUFFERS=20
LASTDRIVE=Z
```

What each line does:
- `HIMEM.SYS` — enables extended memory (XMS), required by most games
- `EMM386.EXE RAM` — enables expanded memory (EMS) and upper memory blocks (UMB)
- `DOS=HIGH,UMB` — loads DOS into high memory, freeing conventional memory for games
- `FILES=40` — max open file handles (some games need more than default 8)
- `BUFFERS=20` — disk cache buffers
- `LASTDRIVE=Z` — allows drive letters up to Z (needed for CD-ROM)

Save: Alt+F → Save → Alt+F → Exit

**Edit AUTOEXEC.BAT:**

```
C:\> EDIT AUTOEXEC.BAT
```

Enter these contents:

```
@ECHO OFF
PATH=C:\DOS;C:\GAMES
PROMPT $P$G
SET BLASTER=A220 I5 D1 H5 T6
SET MIDI=SYNTH:1 MAP:E
MKDIR C:\GAMES 2>NUL
```

What each line does:
- `PATH` — where DOS looks for programs; includes C:\GAMES for convenience
- `PROMPT $P$G` — shows current directory in prompt (e.g., `C:\GAMES>`)
- `SET BLASTER=A220 I5 D1 H5 T6` — tells games where the Sound Blaster is:
  - `A220` = base I/O port 0x220
  - `I5` = IRQ 5
  - `D1` = 8-bit DMA channel 1
  - `H5` = 16-bit DMA channel 5
  - `T6` = card type 6 (SB16)
- `SET MIDI=SYNTH:1 MAP:E` — MIDI synthesizer configuration for General MIDI
- `MKDIR C:\GAMES` — create games directory (2>NUL suppresses "already exists" error)

Save and exit. Reboot: `Ctrl+Alt+Del` in the guest.

### 1.5 Create Game Transfer Disk

This is how you get game files into the DOS VM:

```bash
# Create a 200MB FAT16 transfer disk
~/Projects/qemu-build/qemu-img create -f raw ~/vms/dos622/transfer.img 200M

# Format as FAT16
mkfs.vfat -F 16 ~/vms/dos622/transfer.img

# Mount and copy game files
sudo mkdir -p /mnt/transfer
sudo mount -o loop ~/vms/dos622/transfer.img /mnt/transfer

# Copy DOOM
sudo mkdir -p /mnt/transfer/DOOM
sudo cp /path/to/doom/files/* /mnt/transfer/DOOM/

# Copy Pinball Fantasies
sudo mkdir -p /mnt/transfer/PINBALL
sudo cp /path/to/pinball/files/* /mnt/transfer/PINBALL/

# Unmount
sudo umount /mnt/transfer
```

### 1.6 Install Games Inside DOS

Boot DOS with the transfer disk as drive D:

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -device sb16,iobase=0x220,irq=5,dma=1,hdma=5 \
  -device nuked-opl3 \
  -hda ~/vms/dos622/dos622.qcow2 \
  -hdb ~/vms/dos622/transfer.img \
  -boot c
```

Inside DOS:

```
C:\> MKDIR C:\GAMES\DOOM
C:\> MKDIR C:\GAMES\PINBALL
C:\> XCOPY D:\DOOM C:\GAMES\DOOM /S /E
C:\> XCOPY D:\PINBALL C:\GAMES\PINBALL /S /E
```

For DOOM, if you have the shareware ZIP, you may need to run the installer:
```
C:\> D:
D:\> CD DOOM
D:\DOOM> INSTALL.EXE
```
Follow the installer — set install directory to `C:\GAMES\DOOM`.

### 1.7 Launch Scripts

Create these scripts on your host machine:

**Baseline (no QEMU-SA sound devices):**

```bash
cat > ~/vms/dos622/run-baseline.sh << 'EOF'
#!/bin/bash
cd ~/Projects/qemu-build
./qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot c
EOF
chmod +x ~/vms/dos622/run-baseline.sh
```

**With OPL3 + SB16 (for DOOM FM music, Pinball Fantasies):**

```bash
cat > ~/vms/dos622/run-opl3.sh << 'EOF'
#!/bin/bash
cd ~/Projects/qemu-build
./qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -device sb16,iobase=0x220,irq=5,dma=1,hdma=5 \
  -device nuked-opl3 \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot c
EOF
chmod +x ~/vms/dos622/run-opl3.sh
```

**With OPL3 + SB16 + General MIDI via FluidSynth:**

```bash
cat > ~/vms/dos622/run-gm.sh << 'EOF'
#!/bin/bash
cd ~/Projects/qemu-build
./qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -device sb16,iobase=0x220,irq=5,dma=1,hdma=5 \
  -device nuked-opl3 \
  -device mpu401,backend=gm,config=/usr/share/soundfonts/FluidR3_GM.sf2 \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot c
EOF
chmod +x ~/vms/dos622/run-gm.sh
```

**With OPL3 + SB16 + MT-32 via Munt (requires ROMs):**

```bash
cat > ~/vms/dos622/run-mt32.sh << 'EOF'
#!/bin/bash
cd ~/Projects/qemu-build
./qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu 486-v1 \
  -m 32M \
  -display gtk \
  -vga cirrus \
  -device sb16,iobase=0x220,irq=5,dma=1,hdma=5 \
  -device nuked-opl3 \
  -device mpu401,backend=mt32,config=~/vms/roms/mt32 \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot c
EOF
chmod +x ~/vms/dos622/run-mt32.sh
```

### 1.8 Testing DOS Games

#### Test 1: Pinball Fantasies (SB16 PCM Baseline)

Launch: `~/vms/dos622/run-opl3.sh`

```
C:\> CD \GAMES\PINBALL
C:\GAMES\PINBALL> PINBALL.EXE
```

In sound setup: select **Sound Blaster 16**, port 220, IRQ 5, DMA 1.

**What this tests:** Stock QEMU SB16 PCM playback (DMA). Not our code — but confirms the VM and audio pipeline work.

**Expected:** Digital music and sound effects play.

#### Test 2: DOOM with OPL3 FM Music (Tests Nuked-OPL3!)

Launch: `~/vms/dos622/run-opl3.sh`

```
C:\> CD \GAMES\DOOM
C:\GAMES\DOOM> SETUP.EXE
```

In DOOM SETUP:
1. **Music card:** Select "Sound Blaster" (this uses OPL3 FM synthesis)
2. **Sound FX card:** Select "Sound Blaster"
3. **Port:** 220h, **IRQ:** 5, **DMA:** 1
4. Save and Launch Game

**What this tests:** Our Nuked-OPL3 device. The game writes to port 388h, our `nuked-opl3.c` receives the register writes, Nuked-OPL3 library generates the audio.

**Expected:** Bobby Prince's FM music in E1M1 ("At Doom's Gate"). If you hear music, **Nuked-OPL3 is working.**

#### Test 3: DOOM with General MIDI (Tests MPU-401 + FluidSynth!)

Launch: `~/vms/dos622/run-gm.sh`

In DOOM SETUP:
1. **Music card:** Select "General MIDI" (routes through MPU-401)
2. **Sound FX card:** Keep "Sound Blaster"
3. Save and Launch Game

**What this tests:** MPU-401 UART device → MIDI message assembly → FluidSynth backend → PCM audio.

**Expected:** Same melodies as Test 2 but with richer instrument sounds (piano, strings, brass instead of FM synthesis bleeps). If you hear MIDI instruments, **the entire MPU-401 → FluidSynth pipeline is working.**

---

## Part 2: Windows 98 SE VM

### 2.1 Create VM Directory and Disk Image

```bash
mkdir -p ~/vms/win98
cd ~/vms/win98

# Create 4GB disk
~/Projects/qemu-build/qemu-img create -f qcow2 win98se.qcow2 4G
```

### 2.2 Install Windows 98 SE

Place your Win98 SE ISO as `~/vms/win98/win98se.iso`.

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu pentium3-v1 \
  -m 512M \
  -display gtk \
  -vga cirrus \
  -device ac97 \
  -hda ~/vms/win98/win98se.qcow2 \
  -cdrom ~/vms/win98/win98se.iso \
  -boot d
```

**Step-by-step:**

1. Boot from CD → "Start Windows 98 Setup from CD-ROM"
2. Setup creates partition and formats (FAT32) — accept defaults
3. Copies files, reboots
4. **After first reboot**, change boot order: close QEMU, relaunch with `-boot c` instead of `-boot d` (or just let it boot from HD if it does automatically)
5. GUI setup: enter product key, set timezone, computer name
6. Let it finish and reboot to desktop

### 2.3 Install Win98 Drivers

Boot Win98:

```bash
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu pentium3-v1 \
  -m 512M \
  -display gtk \
  -vga cirrus \
  -device ac97 \
  -hda ~/vms/win98/win98se.qcow2 \
  -cdrom ~/vms/win98/win98se.iso \
  -net nic,model=rtl8139 \
  -net user \
  -boot c
```

**Driver install order:**

1. **AC97 Audio (SigmaTel):**
   - Transfer `stac9708.exe` into the VM (via transfer disk or shared folder)
   - Run the installer
   - Reboot
   - Verify: Start → Settings → Control Panel → System → Device Manager → "Sound, video and game controllers" should show "SigmaTel AC97"
   - Test: Start → Programs → Accessories → Entertainment → Sound Recorder → record and play back

2. **DirectX 7.0a (if needed for games):**
   - Transfer `dx70a_full.exe` into VM
   - Run installer, reboot

3. **qemu-3dfx guest wrappers (for Glide/OpenGL games):**
   - Check what is available: `ls ~/Projects/qemu-3dfx/wrappers/`
   - Transfer the appropriate DLLs into the VM
   - `glide2x.dll` → `C:\Windows\System\` (for Glide 2.x games)
   - `glide3x.dll` → `C:\Windows\System\` (for Glide 3.x games)
   - `opengl32.dll` → **game directory only** (NEVER put in System folder — breaks Windows)

### 2.4 Win98 Launch Script

```bash
cat > ~/vms/win98/run.sh << 'EOF'
#!/bin/bash
cd ~/Projects/qemu-build
./qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -enable-kvm \
  -cpu host \
  -m 512M \
  -display gtk,gl=on \
  -vga cirrus \
  -device ac97 \
  -hda ~/vms/win98/win98se.qcow2 \
  -cdrom ~/vms/win98/win98se.iso \
  -net nic,model=rtl8139 \
  -net user \
  -boot c
EOF
chmod +x ~/vms/win98/run.sh
```

**Note:** qemu-3dfx is always active — no extra flags needed. The `-display gtk,gl=on` enables OpenGL on the host side which qemu-3dfx needs for passthrough.

### 2.5 Testing Win98

1. **Audio test:** Play a WAV file or check Windows startup sound
2. **3dfx test:** Install a Glide game, copy wrapper DLLs, select 3Dfx renderer

---

## Test Matrix

| # | VM | Game | Sound Config | What It Tests | Expected |
|---|-----|------|-------------|---------------|----------|
| 1 | DOS | Pinball Fantasies | SB16 PCM | Stock SB16 DMA (baseline) | Digital music + SFX |
| 2 | DOS | DOOM (SB music) | nuked-opl3 | **Nuked-OPL3 FM synthesis** | Bobby Prince FM music |
| 3 | DOS | DOOM (GM music) | mpu401 + FluidSynth | **MPU-401 → FluidSynth** | Rich MIDI instruments |
| 4 | DOS | Monkey Island (opt) | mpu401 + Munt | **MPU-401 → MT-32** | LA synthesis |
| 5 | Win98 | Desktop sounds | AC97 | AC97 audio driver | Windows sounds play |
| 6 | Win98 | 3dfx game | qemu-3dfx + AC97 | Glide passthrough | 3D accelerated graphics |

---

## QEMU-SA Registered Devices

```
nuked-opl3  — ISA, "Yamaha YMF262 (OPL3) - Nuked-OPL3"
mpu401      — ISA, "MPU-401 UART MIDI Interface"
              Properties: backend= (mt32|gm|host), config= (path)
sb16        — ISA, stock QEMU SB16 (PCM/DMA only, no OPL)
ac97        — PCI, stock QEMU Intel AC97
3dfx/mesa   — always active (hardwired in pc.c), no -device flag
```

---

## Troubleshooting

### No sound at all (DOS or Win98)
- Check PulseAudio: `pactl info`
- Try explicit audio backend: add `-audiodev pa,id=pa0 -device sb16,audiodev=pa0`

### OPL3: game detects AdLib but no music plays
- Check QEMU console (Ctrl+Alt+2) for error messages
- Verify `nuked-opl3` device is in the command line

### MPU-401: game does not detect General MIDI
- Run the game's SETUP.EXE and manually select "MPU-401" or "General MIDI" at port 330
- Verify `mpu401` device is in the command line with `backend=` set

### FluidSynth: MPU-401 detected but no MIDI sound
- Check SoundFont path: `ls /usr/share/soundfonts/FluidR3_GM.sf2`
- Check QEMU startup output for "midi-fluid:" messages

### MT-32: ROM file errors
- ROM directory must contain exactly: `MT32_CONTROL.ROM` and `MT32_PCM.ROM`
- File names are case-sensitive

### Win98: blue screen on boot with KVM
- Replace `-cpu host` with `-cpu pentium3-v1`
- Try adding `-no-acpi`

### Win98: no 3dfx in games
- Verify guest DLLs are installed (glide2x.dll in System or game dir)
- Verify `-display gtk,gl=on` is in launch command

### QEMU monitor commands (Ctrl+Alt+2)
- `change floppy0 /path/to/disk.img` — swap floppy disk
- `eject floppy0` — remove floppy
- `info block` — show attached drives
- `info qtree` — show device tree (verify our devices are present)
- Switch back to guest: Ctrl+Alt+1
