# QEMU-SA Testing Guide: DOS & Win98 VMs

## Prerequisites

### What You Need to Obtain

| Item | Where to Get It | Notes |
|------|-----------------|-------|
| DOS 6.22 install floppies | Your existing media (3x 1.44MB .img files) | Disk 1, 2, 3 |
| Windows 98 SE ISO | Your existing media | Must be Second Edition for AC97 support |
| DOOM 1.9 (Shareware or Full) | archive.org or original media | OPL3 FM music test |
| Monkey Island 1 or 2 | Original media | MT-32 MIDI test |
| Pinball Fantasies | Your existing files | Baseline SB16 PCM test |
| FluidR3_GM.sf2 | `https://member.keymusician.com/Member/FluidR3_GM/` or search | Free General MIDI SoundFont (~142MB) |
| MT-32 ROMs (optional) | Dump from real hardware you own | MT32_CONTROL.ROM + MT32_PCM.ROM |
| SigmaTel AC97 driver | Search: `stac9708.exe` or `SigmaTel C-Major AC97` | For Win98 audio |
| Scitech Display Doctor 7 | Search: `sdd-win-7.0` | For Win98 SVGA (optional, stock VGA works) |
| qemu-3dfx guest files | `~/Projects/qemu-3dfx/` (already cloned) | Glide/OpenGL wrappers for Win98 |

### What You Already Have

- QEMU-SA binary at `~/Projects/qemu-build/qemu-system-i386`
- qemu-3dfx patches applied
- All audio deps: mt32emu 2.7.2, fluidsynth 2.5.3, ALSA

---

## Part 1: DOS 6.22 VM

### Step 1: Create VM Directory and Disk Image

```bash
mkdir -p ~/vms/dos622
cd ~/vms/dos622

# Create 504MB hard disk (max for DOS without LBA)
~/Projects/qemu-build/qemu-img create -f qcow2 dos622.qcow2 504M
```

### Step 2: Install DOS 6.22

Place your DOS floppy images in `~/vms/dos622/` as `disk1.img`, `disk2.img`, `disk3.img`.

```bash
# Boot from floppy disk 1
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

During install:
1. FDISK → create primary partition, use entire disk, set active
2. Reboot (it will ask) — use same command above
3. FORMAT C: /S
4. Continue DOS setup
5. When prompted for Disk 2/3: from QEMU monitor (Ctrl+Alt+2), type:
   ```
   change floppy0 /full/path/to/disk2.img
   ```
   Then press Enter in the guest to continue. Repeat for disk3.

### Step 3: Create DOS Boot Script (No Sound Devices)

This boots DOS without our custom devices — baseline test:

```bash
cat > ~/vms/dos622/run-dos-baseline.sh << 'EOF'
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
chmod +x ~/vms/dos622/run-dos-baseline.sh
```

### Step 4: Create DOS Boot Script (With QEMU-SA Sound)

#### OPL3 Only (for DOOM)

```bash
cat > ~/vms/dos622/run-dos-opl3.sh << 'EOF'
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
chmod +x ~/vms/dos622/run-dos-opl3.sh
```

#### OPL3 + MIDI via FluidSynth (for games with General MIDI)

```bash
cat > ~/vms/dos622/run-dos-gm.sh << 'EOF'
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
  -device mpu401,backend=gm,config=~/vms/soundfonts/FluidR3_GM.sf2 \
  -hda ~/vms/dos622/dos622.qcow2 \
  -boot c
EOF
chmod +x ~/vms/dos622/run-dos-gm.sh
```

#### OPL3 + MIDI via Munt MT-32 (for Sierra/LucasArts games)

```bash
cat > ~/vms/dos622/run-dos-mt32.sh << 'EOF'
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
chmod +x ~/vms/dos622/run-dos-mt32.sh
```

### Step 5: DOS CONFIG.SYS and AUTOEXEC.BAT

After DOS is installed, boot with baseline script and edit these files:

**CONFIG.SYS:**
```
DEVICE=C:\DOS\HIMEM.SYS
DEVICE=C:\DOS\EMM386.EXE RAM
DOS=HIGH,UMB
FILES=40
BUFFERS=20
LASTDRIVE=Z
```

**AUTOEXEC.BAT:**
```
@ECHO OFF
PATH=C:\DOS;C:\GAMES
PROMPT $P$G
SET BLASTER=A220 I5 D1 H5 T6
SET MIDI=SYNTH:1 MAP:E
```

The `SET BLASTER` line tells games where to find the SB16. `A220`=base port, `I5`=IRQ, `D1`=DMA, `H5`=HDMA, `T6`=SB16 type.

### Step 6: Install Games in DOS

Boot with baseline script, then use QEMU to attach game files. Easiest method — create a FAT16 transfer disk:

```bash
# Create a 100MB transfer disk
~/Projects/qemu-build/qemu-img create -f raw ~/vms/dos622/transfer.img 100M

# Format it (from Linux)
mkfs.vfat -F 16 ~/vms/dos622/transfer.img

# Mount and copy game files
sudo mkdir -p /mnt/transfer
sudo mount -o loop ~/vms/dos622/transfer.img /mnt/transfer
sudo cp -r /path/to/doom/* /mnt/transfer/
sudo cp -r /path/to/pinball/* /mnt/transfer/
sudo umount /mnt/transfer
```

Then boot DOS with the transfer disk as a second drive:

```bash
# Add to any run script:
#   -hdb ~/vms/dos622/transfer.img
```

In DOS:
```
D:
XCOPY D:\DOOM C:\GAMES\DOOM /S /E
XCOPY D:\PINBALL C:\GAMES\PINBALL /S /E
```

### Step 7: Test Games

#### Test 1: Pinball Fantasies (SB16 PCM Baseline)

```bash
~/vms/dos622/run-dos-opl3.sh
```

In DOS:
```
CD \GAMES\PINBALL
PINBALL.EXE
```

Configure sound: select **Sound Blaster 16**, port 220, IRQ 5, DMA 1. This tests stock SB16 PCM — not our code, but confirms the VM works.

#### Test 2: DOOM (OPL3 FM — Tests Our Code!)

```bash
~/vms/dos622/run-dos-opl3.sh
```

In DOS:
```
CD \GAMES\DOOM
SETUP.EXE
```

In DOOM setup:
- Music: **Sound Blaster** (this uses OPL3 FM)
- SFX: **Sound Blaster**, port 220, IRQ 5, DMA 1
- Save and run

**What to listen for:** Bobby Prince's FM music in E1M1. If you hear music, Nuked-OPL3 is working. If silent, the OPL3 device isn't being detected.

#### Test 3: DOOM with General MIDI (Tests MPU-401 + FluidSynth)

```bash
~/vms/dos622/run-dos-gm.sh
```

In DOOM setup:
- Music: **General MIDI** (this routes through MPU-401)

**What to listen for:** Same melodies but richer instrument sounds (piano, strings, brass instead of FM bleeps).

---

## Part 2: Windows 98 SE VM

### Step 1: Create VM Directory and Disk Image

```bash
mkdir -p ~/vms/win98
cd ~/vms/win98

# Create 4GB disk (plenty for Win98 + games)
~/Projects/qemu-build/qemu-img create -f qcow2 win98se.qcow2 4G
```

### Step 2: Install Windows 98 SE

Place your Win98 SE ISO as `~/vms/win98/win98se.iso`.

```bash
# Boot from CD-ROM for installation
~/Projects/qemu-build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu pentium3-v1 \
  -m 512M \
  -display gtk \
  -vga cirrus \
  -device sb16 \
  -hda ~/vms/win98/win98se.qcow2 \
  -cdrom ~/vms/win98/win98se.iso \
  -boot d
```

During install:
1. Boot from CD, choose "Start Windows 98 Setup from CD-ROM"
2. It will FDISK and format — accept defaults (FAT32)
3. Restart when prompted (change `-boot d` to `-boot c` for subsequent boots, or just let it boot from HD)
4. Complete the GUI setup (enter product key, set timezone, etc.)
5. Let it finish and reboot to desktop

### Step 3: Install Drivers

Boot Win98 (from hard disk now):

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
  -boot c
```

**Drivers to install in order:**

1. **AC97 Audio:** SigmaTel C-Major AC97 WDM driver (`stac9708.exe`)
   - Copy to VM via transfer disk (same method as DOS) or shared folder
   - Run installer, reboot
   - Device Manager should show "SigmaTel AC97" under Sound

2. **SVGA (optional):** Cirrus VGA driver comes with Win98 and auto-detects. For higher resolutions, install Scitech Display Doctor 7 if needed.

3. **DirectX:** Win98SE ships with DirectX 6.1. For best compatibility, install DirectX 7.0a:
   - Download `dx70a_full.exe` from archive.org
   - Run inside the VM, reboot

### Step 4: Install qemu-3dfx Guest Wrappers

The qemu-3dfx guest wrappers make Glide/OpenGL calls work inside the VM. The files are in your cloned repo:

```bash
# Check what's in the guest wrappers directory
ls ~/Projects/qemu-3dfx/wrappers/
```

You need to copy the guest-side DLLs into the Win98 VM. The exact files depend on the qemu-3dfx version — typically:

- `glide2x.dll` → `C:\Windows\System\`
- `glide3x.dll` → `C:\Windows\System\`
- `opengl32.dll` → game directory (NOT System — would break Windows)

Transfer method: create a transfer ISO or use the FAT transfer disk method from the DOS section.

### Step 5: Create Win98 Boot Scripts

#### Win98 with AC97 Audio (Basic)

```bash
cat > ~/vms/win98/run-win98-basic.sh << 'EOF'
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
chmod +x ~/vms/win98/run-win98-basic.sh
```

#### Win98 with 3Dfx Glide Passthrough (THE CROWN JEWEL)

```bash
cat > ~/vms/win98/run-win98-3dfx.sh << 'EOF'
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
  -device 3dfx-glide \
  -hda ~/vms/win98/win98se.qcow2 \
  -cdrom ~/vms/win98/win98se.iso \
  -net nic,model=rtl8139 \
  -net user \
  -boot c
EOF
chmod +x ~/vms/win98/run-win98-3dfx.sh
```

**Note:** The `-device 3dfx-glide` flag comes from the qemu-3dfx patches. Verify it's available:

```bash
~/Projects/qemu-build/qemu-system-i386 -device help 2>&1 | grep -i 3dfx
```

### Step 6: Install and Test Win98 Games

For 3dfx games (Tomb Raider, NFS3, Quake II):
1. Install the game normally inside Win98
2. Copy the appropriate `glide2x.dll` or `glide3x.dll` from qemu-3dfx wrappers to the game directory
3. In game settings, select 3Dfx/Glide renderer
4. Launch with the `run-win98-3dfx.sh` script

---

## Test Matrix

After setup, run through this checklist:

| # | VM | Game | Sound Device | What Tests | Expected Result |
|---|-----|------|-------------|------------|-----------------|
| 1 | DOS | Pinball Fantasies | SB16 | PCM/DMA (baseline) | Digital music and SFX |
| 2 | DOS | DOOM (SB music) | nuked-opl3 | OPL3 FM synthesis | Bobby Prince FM music |
| 3 | DOS | DOOM (GM music) | mpu401+fluidsynth | MPU-401 → FluidSynth | MIDI instruments |
| 4 | DOS | Monkey Island | mpu401+mt32 | MPU-401 → Munt | MT-32 LA synthesis |
| 5 | Win98 | Desktop | AC97 | AC97 audio + SB emu | Windows sounds play |
| 6 | Win98 | 3dfx game | 3dfx-glide | Glide passthrough | 3D-accelerated graphics |

---

## Troubleshooting

### No sound at all
- Check PulseAudio is running: `pactl info`
- QEMU audio backend: add `-audiodev pa,id=snd0` if auto-detection fails

### OPL3 detected but no music
- Verify game is configured for Sound Blaster / AdLib
- Check `SET BLASTER=A220 I5 D1 H5 T6` in AUTOEXEC.BAT

### MPU-401 not detected by game
- Some games only scan port 330h during setup, not gameplay
- Run the game's SETUP/INSTALL program and manually configure MPU-401 at port 330

### DOOM says "No sound cards detected"
- Run SETUP.EXE first, don't launch DOOM.EXE directly
- Select sound card manually in setup

### MT-32 errors about ROM files
- Check path in `config=` parameter points to directory containing both ROM files
- Files must be named exactly: `MT32_CONTROL.ROM` and `MT32_PCM.ROM`

### Win98 blue screen on boot with KVM
- Try `-cpu pentium3-v1` instead of `-cpu host`
- Win98 is picky about CPU features — APIC can cause issues

### Win98 no 3Dfx
- Verify `glide2x.dll` is in the game directory (not System folder)
- Verify `-device 3dfx-glide` is in the launch command
- Check QEMU console for 3dfx-related messages
