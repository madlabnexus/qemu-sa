# 01 — P16v Workstation Setup: Phase 3 — System Optimizations & First QEMU-SA Build

**Hardware:** Lenovo ThinkPad P16v Gen 2  
**OS:** EndeavourOS (Arch Linux) · GNOME 49.5 · Wayland · systemd-boot  
**Kernel:** 6.19.8-arch1-1  
**Completed:** 2026-03-18

---

## Phase 3 Scope

System optimizations, QEMU upstream source clone, qemu-3dfx patch integration, and first successful QEMU-SA build:

| Step | Item | Status |
|------|------|--------|
| 00 | System update (272 packages, kernel 6.19.8, QEMU 10.2.2) | ✅ |
| 01 | zram compressed swap (zram-generator, zstd, 7.7GB) | ✅ |
| 02 | sysctl tuning (swappiness, dirty ratio, inotify, file-max) | ✅ |
| 03 | I/O scheduler verified (none — optimal for NVMe) | ✅ |
| 04 | QEMU 9.2.4 upstream cloned with submodules | ✅ |
| 05 | qemu-3dfx repo cloned (kjliew/qemu-3dfx) | ✅ |
| 06 | qemu-3dfx patches applied (overlay + patch) | ✅ |
| 07 | First QEMU-SA build — toolchain validation | ✅ |
| 08 | SSH config for TD350 server | ✅ |
| 09 | Network bridge (br0) for QEMU bridged LAN | ✅ |

---

## 00 — System Update

```bash
yay -Syu
```

272 packages updated including:

| Package | Old | New | Notes |
|---------|-----|-----|-------|
| linux | 6.19.6-arch1-1 | 6.19.8-arch1-1 | NVIDIA DKMS rebuilt automatically |
| qemu-full | 10.2.0 | 10.2.2 | System QEMU (separate from QEMU-SA build) |
| llvm/clang | 21.1.8 | 22.1.1 | Major toolchain bump |
| meson | 1.10.1 | 1.10.2 | Build system |
| gnome-shell | 49.4 | 49.5 | Desktop |
| ghostty | 1.2.3 | 1.3.1 | Terminal |

NVIDIA DKMS output confirmed:
```
==> dkms install --no-depmod nvidia/590.48.01 -k 6.19.8-arch1-1
```

Reboot required after update (kernel + NVIDIA).

---

## 01 — zram Compressed Swap

```bash
sudo pacman -S zram-generator
```

**Config at `/etc/systemd/zram-generator.conf`:**
```ini
[zram0]
zram-size = ram / 4
compression-algorithm = zstd
swap-priority = 100
```

**Result:**
```
NAME           TYPE       SIZE USED PRIO
/dev/nvme1n1p3 partition 33.9G   0B   -1
/dev/zram0     partition  7.7G   0B  100
```

zram (priority 100) is used before NVMe swap (priority -1). With ~31GB RAM, `ram / 4 = 7.7GB` zram device using zstd compression (best ratio-to-speed tradeoff).

**Start service:**
```bash
sudo systemctl daemon-reexec
sudo systemctl start systemd-zram-setup@zram0
```

> **Reinstall note:** Config lives on `@` subvolume (wiped on reinstall). Backed up to `~/dotfiles/etc/systemd/zram-generator.conf`. Restore with: `sudo cp ~/dotfiles/etc/systemd/zram-generator.conf /etc/systemd/`

---

## 02 — sysctl Tuning

**Config at `/etc/sysctl.d/99-qemu-sa-dev.conf`:**
```ini
# Prefer zram over NVMe swap (default 60 is too aggressive for 31GB RAM)
vm.swappiness = 10

# Flush dirty pages less aggressively (better for large builds)
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5

# VS Code / QEMU source tree needs high inotify watches (30k+ files)
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 1024

# More file handles for parallel builds
fs.file-max = 2097152
```

**Apply:**
```bash
sudo sysctl --system
```

> **Reinstall note:** Backed up to `~/dotfiles/etc/99-qemu-sa-dev.conf`. Restore with: `sudo cp ~/dotfiles/etc/99-qemu-sa-dev.conf /etc/sysctl.d/`

---

## 03 — I/O Scheduler

```bash
cat /sys/block/nvme1n1/queue/scheduler
# [none] mq-deadline kyber bfq

cat /sys/block/nvme0n1/queue/scheduler
# [none] mq-deadline kyber bfq
```

Both NVMe drives already on `none` — optimal (NVMe controller handles its own scheduling). No changes needed.

---

## 04 — QEMU 9.2.4 Upstream Clone

```bash
cd ~/Projects
git clone https://gitlab.com/qemu-project/qemu.git qemu-upstream
cd qemu-upstream
git checkout v9.2.4
git submodule update --init --recursive
```

- Commit: `b2995afed2` (Update version for 9.2.4 release)
- 15 submodules including SeaBIOS, edk2, ipxe, OpenBIOS
- Size: ~500MB with all submodules

> **Why 9.2.4 and not 10.x:** qemu-3dfx patches (`00-qemu92x-mesa-glide.patch`) target QEMU 9.2.x only. No 10.x patch exists. 9.2.4 is the latest bugfix release in the 9.2.x stable branch — updated from the originally planned 9.2.2 (ADR-007).

---

## 05 — qemu-3dfx Clone

```bash
cd ~/Projects
git clone https://github.com/kjliew/qemu-3dfx.git
```

Patch files available:
- `00-qemu92x-mesa-glide.patch` — QEMU 9.2.x (our target)
- `01-qemu82x-mesa-glide.patch` — QEMU 8.2.x
- `02-qemu72x-mesa-glide.patch` — QEMU 7.2.x

---

## 06 — Apply qemu-3dfx Patches

```bash
cd ~/Projects/qemu-upstream

# Copy 3dfx/MESA overlay files into QEMU source tree
rsync -r ../qemu-3dfx/qemu-0/hw/3dfx ../qemu-3dfx/qemu-1/hw/mesa ./hw/

# Apply the patch
patch -p0 -i ../qemu-3dfx/00-qemu92x-mesa-glide.patch

# Sign the commit (stamps build with commit ID)
bash ../qemu-3dfx/scripts/sign_commit
```

**Patch result:** All files patched cleanly. One minor offset on `meson.build` (expected for 9.2.4 vs 9.2.2):
```
patching file ./meson.build
Hunk #2 succeeded at 3698 (offset -9 lines).
Hunk #3 succeeded at 4118 (offset -9 lines).
```

**Files modified by qemu-3dfx:**
- `accel/kvm/kvm-all.c` — guest PA range update function
- `hw/i386/pc.c`, `include/hw/i386/pc.h` — PC machine hooks
- `include/sysemu/kvm.h`, `include/sysemu/whpx.h` — hypervisor headers
- `include/ui/console.h`, `ui/console.c`, `ui/sdl2.c` — display integration
- `meson.build` — build system (3dfx/mesa subdirs)
- `system/vl.c` — main QEMU loop
- `target/i386/whpx/whpx-all.c` — WHPX support

**New files added (overlay):**
- `hw/3dfx/` — 3Dfx Glide pass-through device model
- `hw/mesa/` — MESA GL pass-through device model

---

## 07 — First QEMU-SA Build

### Additional build dependency discovered

```bash
sudo pacman -S python-distlib
```

> **Note:** QEMU 9.2.4's build system requires `python-distlib` for its virtual environment setup. This was not in the Phase 2 dependency list — discovered during first build attempt.

### GCC 15 compatibility note

QEMU 9.2.4 has `const` qualifier warnings in utility code (`util/log.c`, `util/qemu-sockets.c`) that GCC 15.2.1 promotes to errors with `-Werror`. Solution: `--disable-werror` in configure. These are harmless const-correctness issues that don't affect functionality.

### Configure

```bash
mkdir -p ~/Projects/qemu-build && cd ~/Projects/qemu-build

../qemu-upstream/configure \
  --target-list=i386-softmmu,x86_64-softmmu \
  --enable-kvm \
  --enable-opengl \
  --enable-sdl \
  --enable-gtk \
  --enable-alsa \
  --enable-slirp \
  --audio-drv-list=pa,alsa,jack \
  --extra-cflags="-O2" \
  --disable-werror
```

**Key configure output:**
- Audio backends: PulseAudio ✅, ALSA ✅, JACK ✅, PipeWire ✅
- KVM support: YES
- TCG support: YES (native x86_64)
- OpenGL (epoxy): YES
- SDL: YES 2.32.64
- GTK: YES
- Target list: i386-softmmu, x86_64-softmmu

> **Note:** `--enable-pulseaudio` is not a valid flag in QEMU 9.2.x. Use `--audio-drv-list=pa,alsa,jack` instead.

### Build

```bash
make -j$(nproc)
```

Build completed: 3145/3145 targets, using 22 threads on the P16v. Build time: ~5 minutes.

### Verify

```bash
./qemu-system-i386 --version
# QEMU emulator version 9.2.4 (v9.2.4-dirty)
# Copyright (c) 2003-2024 Fabrice Bellard and the QEMU Project developers
#   featuring qemu-3dfx@b2995afed2-18:11:05 Mar 18 2026 build

./qemu-system-x86_64 --version
# QEMU emulator version 9.2.4 (v9.2.4-dirty)
# Copyright (c) 2003-2024 Fabrice Bellard and the QEMU Project developers
#   featuring qemu-3dfx@b2995afed2-18:11:05 Mar 18 2026 build
```

The `featuring qemu-3dfx@b2995afed2` line confirms 3dfx patches are integrated.

---

## 08 — SSH Config for TD350

```bash
cat >> ~/.ssh/config << 'EOF'

# QEMU-SA Build Server
Host td350
    HostName 192.168.1.100
    User savant
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes
EOF
chmod 600 ~/.ssh/config
```

> **Note:** IP address `192.168.1.100` is placeholder — update after TD350 server setup.

---

## 09 — Network Bridge (br0) for QEMU Bridged LAN

QEMU VMs default to SLIRP (user-mode NAT) which isolates the guest from the LAN. A bridge exposes VMs directly on the physical network — they get their own DHCP address and are reachable from other machines on the LAN.

### Bridge topology

```
                    br0 (bridge)
                   ╱            ╲
           eth0 (dock)    enp0s31f6 (built-in)
           Thunderbolt      RJ45 on laptop
```

Both physical Ethernet interfaces are slaved to the bridge. Whichever has a cable provides connectivity. Both can be active simultaneously.

### Create bridge + slaves (NetworkManager)

```bash
# Create bridge
sudo nmcli connection add type bridge con-name br0 ifname br0 \
  ipv4.method auto ipv6.method auto stp no

# Slave 1: Thunderbolt dock Ethernet
sudo nmcli connection add type ethernet con-name br0-dock ifname eth0 master br0

# Slave 2: Built-in Ethernet
sudo nmcli connection add type ethernet con-name br0-builtin ifname enp0s31f6 master br0

# Activate (drops network briefly during switchover)
sudo nmcli connection down "Wired connection 2"
sudo nmcli connection up br0

# Delete old direct connections
sudo nmcli connection delete "Wired connection 1"
sudo nmcli connection delete "Wired connection 2"
```

### Verify

```bash
nmcli device status
# br0        bridge    connected    br0
# eth0       ethernet  connected    br0-dock
# enp0s31f6  ethernet  unavailable  --  (joins when cable plugged)

ip addr show br0
# inet 172.16.80.160/16  (DHCP assigned to bridge)

bridge link show
# eth0: master br0 state forwarding
```

### QEMU bridge helper configuration

Each QEMU build has its own bridge helper that looks for `bridge.conf` relative to its install prefix:

```bash
# System-wide config (also serves as backup in dotfiles)
sudo mkdir -p /etc/qemu
echo "allow br0" | sudo tee /etc/qemu/bridge.conf

# QEMU-SA (qemu-3dfx) build — prefix-relative config
mkdir -p ~/Applications/qemu-3dfx/install/etc/qemu
cp /etc/qemu/bridge.conf ~/Applications/qemu-3dfx/install/etc/qemu/

# Projects build — prefix-relative config
mkdir -p ~/Projects/qemu-build/qemu-bundle/usr/local/etc/qemu
cp /etc/qemu/bridge.conf ~/Projects/qemu-build/qemu-bundle/usr/local/etc/qemu/
```

### Setuid on bridge helpers

The bridge helper needs to run as root to create tap interfaces:

```bash
# QEMU-SA (qemu-3dfx) build
sudo chown root:root ~/Applications/qemu-3dfx/install/libexec/qemu-bridge-helper
sudo chmod u+s ~/Applications/qemu-3dfx/install/libexec/qemu-bridge-helper

# Projects build
sudo chown root:root ~/Projects/qemu-build/qemu-bridge-helper
sudo chmod u+s ~/Projects/qemu-build/qemu-bridge-helper
```

> **Reinstall note:** Bridge connections survive on `@home` (NetworkManager profiles in `~/.local/`... no, NM stores in `/etc/NetworkManager/`). After reinstall: re-create bridge via nmcli commands above, copy `bridge.conf` from `~/dotfiles/etc/qemu/`, re-apply setuid on helpers.

### QEMU usage with bridge

```bash
# Bridged networking (VM gets its own LAN IP via DHCP)
-netdev bridge,id=net0,br=br0 -device rtl8139,netdev=net0    # Win98
-netdev bridge,id=net0,br=br0 -device e1000,netdev=net0      # WinXP
```

---

## Phase 3 — Snapshots

| Snapshot | Description |
|----------|-------------|
| `22-pre-phase3-optimizations` | Pre-Phase 3 (clean post-update) |
| `23-qemu-sa-first-build-ok` | First QEMU-SA 9.2.4 build with qemu-3dfx |
| `28-network-bridge-br0` | Network bridge for QEMU bridged LAN |

---

## Phase 3 — Directory Layout

```
~/Projects/
├── qemu-sa/           ← Project docs repo (github.com/madlabnexus/qemu-sa)
├── qemu-upstream/     ← QEMU 9.2.4 source + qemu-3dfx patches (working copy)
├── qemu-build/        ← Out-of-tree build directory
└── qemu-3dfx/         ← kjliew/qemu-3dfx repo (patches + wrappers)
```

---

## QEMU-SA Build Dependencies — Updated Summary

Everything needed to compile QEMU-SA from source on Arch Linux:

```
# Core build (Phase 2)
base-devel git meson ninja cmake python-sphinx dtc
clang llvm attr curl

# QEMU core (Phase 2)
glib2 pixman libslirp libcap-ng libseccomp libepoxy
virglrenderer spice-protocol usbredir libusb fuse3
bzip2 lzo snappy numactl libiscsi
sdl2 sdl2_image gtk3 vte3 spice spice-gtk
libpng libjpeg-turbo libdrm mesa libepoxy

# QEMU-SA audio (Phase 2)
fluidsynth alsa-lib libpulse pipewire-jack
libsamplerate libsndfile opus
munt  ← AUR (libmt32emu)
libcue

# Discovered in Phase 3
python-distlib  ← Required by QEMU's meson virtual environment setup

# Virtualization (Phase 2)
qemu-full libvirt dnsmasq iptables-nft edk2-ovmf
```

---

## Next: TD350 Server Setup & Claude Code Tutorial

- TD350 server installation (Arch Linux, VFIO, build environment)
- Claude Code workflow for QEMU-SA development
- First smoke test: SeaBIOS boot on P16v build

---

*Phase 4 continues with QEMU-SA sound integration (Nuked-OPL3, MPU-401, MIDI backends)*
