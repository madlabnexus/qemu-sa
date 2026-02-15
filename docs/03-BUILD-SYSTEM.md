# 03 — Build System

## Prerequisites

See [02-SETUP-SERVER.md](02-SETUP-SERVER.md) for dependency installation.

## Clone & Setup

```bash
# On the TD350 server
cd ~/projects

# Clone our fork (once the QEMU fork is ready)
git clone git@github.com:madlabnexus/qemu-sa.git
cd qemu-sa

# Add upstream QEMU as remote for rebasing
git remote add upstream https://gitlab.com/qemu-project/qemu.git
git fetch upstream
```

## Initial Fork Process

Phase 0 starts by forking QEMU 9.2.2:

```bash
# Start from QEMU 9.2.2 release tag
git clone https://gitlab.com/qemu-project/qemu.git qemu-sa-build
cd qemu-sa-build
git checkout v9.2.2
git submodule update --init --recursive

# Create our branch
git checkout -b qemu-sa/main

# Apply qemu-3dfx patches (from kjliew/qemu-3dfx)
# Detailed instructions TBD after Phase 0 setup
```

## Configure

```bash
mkdir build && cd build

../configure \
  --prefix=/usr/local \
  --target-list=i386-softmmu,x86_64-softmmu \
  --enable-kvm \
  --enable-vfio \
  --enable-spice \
  --enable-usb-redir \
  --enable-libusb \
  --enable-virtiofsd \
  --enable-opengl \
  --enable-gtk \
  --enable-sdl \
  --enable-pulseaudio \
  --enable-alsa \
  --audio-drv-list=pa,alsa \
  --enable-slirp \
  --enable-curl \
  --enable-linux-aio \
  --enable-linux-io-uring \
  --enable-numa \
  --enable-seccomp \
  --enable-cap-ng \
  --extra-cflags="-O2 -march=native" \
  --extra-ldflags=""
```

### QEMU-SA Specific Configure Flags (after integration)

These will be added as we integrate components:

```bash
# Phase 1: Sound
--enable-nuked-opl3      # Nuked-OPL3 (replaces stock YM3812)
--enable-munt            # MT-32/CM-32L emulation
--enable-fluidsynth      # General MIDI SoundFont synthesis

# Phase 2: CD Audio
--enable-libcue          # CUE/BIN disc image support
```

## Build

```bash
make -j$(nproc)
```

On the TD350 with 24 threads: ~5-8 minutes for a full build.

## Install (optional)

```bash
sudo make install
```

Or run directly from the build directory:

```bash
./build/qemu-system-i386 --version
./build/qemu-system-x86_64 --version
```

## Quick Test: Stock QEMU

Before any modifications, verify stock QEMU works:

```bash
# Simple boot test (should show SeaBIOS splash, then "No bootable device")
./build/qemu-system-i386 \
  -machine pc-i440fx-9.2 \
  -cpu pentium3 \
  -m 256M \
  -vga cirrus \
  -soundhw sb16 \
  -display sdl
```

## Development Workflow

```
P16v Notebook                    TD350 Server
┌─────────────┐                 ┌──────────────────┐
│ VS Code/Kate│ ──── SSH ────→  │ ~/projects/      │
│ Claude Code │                 │   qemu-sa/       │
│ Git push    │                 │     src/         │
└─────────────┘                 │     build/       │
      │                         │                  │
      └── git push ──→ GitHub ──→ git pull         │
                                │ make -j24        │
                                │ Test VMs         │
                                └──────────────────┘
```

### Option A: Edit on notebook, build on server

```bash
# On P16v: edit, commit, push
cd ~/projects/qemu-sa
vim src/hw/audio/nuked-opl3.c
git add -A && git commit -m "hw/audio: add OPL3 register write handler"
git push

# On TD350: pull and build
ssh td350
cd ~/projects/qemu-sa
git pull
cd build && make -j$(nproc)
```

### Option B: Edit directly on server via SSH

```bash
# On P16v: SSH with VS Code Remote
code --remote ssh-remote+td350 ~/projects/qemu-sa
```

### Option C: Claude Code (preferred for new development)

```bash
# On P16v
cd ~/projects/qemu-sa
claude
# Claude Code edits files, commits, you review
git push

# On TD350
ssh td350 "cd ~/projects/qemu-sa && git pull && cd build && make -j\$(nproc)"
```

## Build Targets

```bash
# Full rebuild
make -j$(nproc)

# Rebuild only after audio changes
make -j$(nproc) hw/audio/

# Clean rebuild
make clean && make -j$(nproc)

# Run QEMU's built-in tests
make check
```

## Packaging (Phase 4)

### Arch Linux (AUR)

```bash
# PKGBUILD will be in packaging/arch/
makepkg -si
```

### Debian/Ubuntu (.deb)

```bash
# packaging/debian/ with dpkg-buildpackage
```

### Flatpak

```bash
# packaging/flatpak/com.madlabnexus.qemu-sa.yml
```

Details TBD when we reach Phase 4.
