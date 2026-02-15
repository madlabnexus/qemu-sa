# 02 — TD350 Server Setup (Build & Test)

## Hardware

- **Model:** Lenovo ThinkServer TD350
- **CPU:** 2× Intel Xeon E5-2643 v4 (6C/12T each, 3.4GHz base, 3.7GHz turbo)
- **RAM:** 256GB DDR4 ECC (16×16GB)
- **GPU 1:** NVIDIA GeForce RTX 4090 24GB → VFIO passthrough (Win98/XP VMs)
- **GPU 2:** AMD Radeon RX 6650 XT 8GB → VFIO passthrough (secondary VMs)
- **GPU 3:** Aspeed BMC → Host console (headless management)
- **Storage:** TBD (OS drive + VM storage)
- **Network:** Onboard 1GbE (+ optional 10GbE if available)

## Role

This machine compiles QEMU-SA from source, runs retro gaming VMs with GPU passthrough, and serves as the integration test environment. It runs headless — managed entirely via SSH from the P16v notebook.

---

## Step 1: BIOS Configuration

Enter BIOS (F1 at boot).

### Critical Settings

1. **Advanced → Processor Configuration:**
   - Intel Virtualization Technology (VT-x): **Enabled**
   - Intel VT for Directed I/O (VT-d): **Enabled**
   - Hyper-Threading: **Enabled**

2. **Advanced → PCI Configuration:**
   - SR-IOV Support: **Enabled** (if available)
   - MMIO High Base: **56T** (if option exists — needed for RTX 4090's large BAR)
   - Above 4G Decoding: **Enabled**

3. **Advanced → Integrated Devices:**
   - Aspeed Video: **Enabled** (this is the server console)
   - Primary Display: **Aspeed / Onboard** (keep GPUs free for passthrough)

4. **Advanced → USB Configuration:**
   - USB UEFI Support: **Enabled**

5. **Security → Secure Boot:**
   - Secure Boot: **Disabled**

6. **Boot:**
   - Boot Mode: **UEFI Only**

Save and exit (F10).

## Step 2: Arch Installation

Boot from Ventoy USB (F12), select Arch ISO.

### 2.1 Network

```bash
# Ethernet should auto-connect
ip addr show
ping -c 3 archlinux.org
```

### 2.2 Identify OS Drive

```bash
lsblk -d -o NAME,SIZE,MODEL
```

Pick the drive for the OS (SSD recommended). We'll call it `/dev/sdX` — **replace with your actual device name**.

### 2.3 Partition

```bash
sgdisk --zap-all /dev/sdX

sgdisk -n 1:0:+1G -t 1:ef00 -c 1:"EFI" /dev/sdX
sgdisk -n 2:0:+32G -t 2:8200 -c 2:"Swap" /dev/sdX
sgdisk -n 3:0:0 -t 3:8300 -c 3:"Arch" /dev/sdX

lsblk /dev/sdX
```

### 2.4 Format & Install

```bash
mkfs.fat -F 32 /dev/sdX1
mkswap /dev/sdX2
mkfs.ext4 /dev/sdX3

mount /dev/sdX3 /mnt
mount --mkdir /dev/sdX1 /mnt/boot
swapon /dev/sdX2

pacstrap -K /mnt base linux linux-firmware \
  intel-ucode \
  networkmanager \
  sudo vim nano \
  base-devel git \
  openssh \
  man-db man-pages

genfstab -U /mnt >> /mnt/etc/fstab
```

### 2.5 System Configuration

```bash
arch-chroot /mnt

# Timezone
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
hwclock --systohc

# Locale
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "pt_BR.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Hostname
echo "td350" > /etc/hostname
cat > /etc/hosts << 'EOF'
127.0.0.1   localhost
::1         localhost
127.0.1.1   td350.localdomain td350
EOF

# Users
passwd
useradd -m -G wheel,video,audio,kvm,input -s /bin/bash savant
passwd savant
EDITOR=nano visudo
# Uncomment: %wheel ALL=(ALL:ALL) ALL

# Services
systemctl enable NetworkManager
systemctl enable sshd
```

### 2.6 Boot Loader

```bash
bootctl install

cat > /boot/loader/loader.conf << 'EOF'
default arch.conf
timeout 3
console-mode max
editor no
EOF

blkid -s UUID -o value /dev/sdX3
# Note the UUID

cat > /boot/loader/entries/arch.conf << 'EOF'
title   Arch Linux (QEMU-SA Server)
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=UUID=REPLACE_WITH_UUID rw quiet intel_iommu=on iommu=pt
EOF

nano /boot/loader/entries/arch.conf
# Replace REPLACE_WITH_UUID with actual UUID
```

**Key kernel parameters:**
- `intel_iommu=on` — enable IOMMU for VFIO
- `iommu=pt` — passthrough mode (performance, only devices bound to VFIO use IOMMU)

### 2.7 Reboot

```bash
exit
umount -R /mnt
swapoff /dev/sdX2
reboot
```

## Step 3: VFIO GPU Passthrough Configuration

### 3.1 Verify IOMMU

```bash
dmesg | grep -i iommu
# Should see: "DMAR: IOMMU enabled" or similar

# List IOMMU groups
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=$(echo $d | cut -d/ -f5)
    echo "IOMMU Group $n: $(lspci -nns ${d##*/})"
done
```

### 3.2 Identify GPU PCI IDs

```bash
lspci -nn | grep -E "(VGA|Audio)" | grep -v Aspeed
```

Expected output (IDs will vary):
```
XX:00.0 VGA compatible controller: NVIDIA Corporation AD102 [GeForce RTX 4090] [10de:2684]
XX:00.1 Audio device: NVIDIA Corporation AD102 High Definition Audio [10de:22ba]
YY:00.0 VGA compatible controller: AMD/ATI Navi 23 [Radeon RX 6650 XT] [1002:73ef]
YY:00.1 Audio device: AMD/ATI Navi 23 [1002:ab28]
```

**Write down all four PCI IDs** (GPU + audio for each card). You need both — if you only pass the GPU without its audio device, IOMMU group isolation breaks.

### 3.3 Bind GPUs to vfio-pci

```bash
# Replace with YOUR actual PCI IDs
sudo tee /etc/modprobe.d/vfio.conf << 'EOF'
options vfio-pci ids=10de:2684,10de:22ba,1002:73ef,1002:ab28
softdep nvidia pre: vfio-pci
softdep amdgpu pre: vfio-pci
EOF
```

The `softdep` lines ensure vfio-pci claims the GPUs before nvidia/amdgpu drivers try to.

### 3.4 Update initramfs

```bash
cat > /etc/mkinitcpio.conf.d/vfio.conf << 'EOF'
MODULES=(vfio_pci vfio vfio_iommu_type1)
EOF

sudo mkinitcpio -P
```

### 3.5 Reboot and Verify

```bash
sudo reboot

# After reboot, verify GPUs are bound to vfio-pci:
lspci -nnk | grep -A 3 "RTX 4090"
lspci -nnk | grep -A 3 "6650"
```

Both should show `Kernel driver in use: vfio-pci`.

The Aspeed BMC GPU remains as the host console for emergency access (monitor + keyboard plugged into server).

## Step 4: Build Dependencies

```bash
# QEMU build dependencies
sudo pacman -S \
  python python-pip \
  cmake meson ninja \
  gcc clang \
  glib2 pixman \
  libepoxy virglrenderer \
  spice spice-protocol \
  libusb usbredir \
  libcap-ng \
  libpulse pipewire pipewire-alsa pipewire-pulse wireplumber \
  alsa-lib \
  dtc \
  libseccomp \
  libaio \
  liburing \
  curl \
  gdb valgrind strace \
  ovmf \
  numactl

# Sound libraries for QEMU-SA
sudo pacman -S \
  fluidsynth \
  libsndfile \
  libsamplerate

# Munt (MT-32 emulator)
yay -S munt-mt32emu

# AUR helper (if not installed)
cd /tmp
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
cd ~

# VM management
sudo pacman -S \
  libvirt virt-install \
  qemu-base \
  dnsmasq iptables-nft \
  bridge-utils \
  tmux htop btop \
  rsync jq ripgrep fd

sudo systemctl enable libvirtd
sudo usermod -aG libvirt savant
```

## Step 5: VM Storage Structure

```bash
mkdir -p ~/vms/{images,iso,drivers,roms,soundfonts,shared}

# images/     — qcow2 disk images for VMs
# iso/        — OS installation ISOs (DOS, Win3.11, Win98, WinXP)
# drivers/    — Guest drivers (AC97, VDMSound, SoftGPU, etc.)
# roms/       — BIOS ROMs, option ROMs
# soundfonts/ — GM SoundFonts (e.g., FluidR3_GM.sf2)
# shared/     — 9p/virtiofs shared folder with guest
```

### SoundFont Setup

```bash
# Install a good General MIDI SoundFont
cd ~/vms/soundfonts
curl -LO https://member.keymusician.com/Member/FluidR3_GM/FluidR3_GM.sf2
# Or use the Arch package:
sudo pacman -S soundfont-fluid
# Installed to: /usr/share/soundfonts/FluidR3_GM.sf2
```

## Step 6: Network Setup

### Static IP (recommended for server)

```bash
# Find your connection name
nmcli connection show

# Set static IP (adjust for your network)
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses "192.168.1.100/24" \
  ipv4.gateway "192.168.1.1" \
  ipv4.dns "1.1.1.1,8.8.8.8"

sudo nmcli connection up "Wired connection 1"
```

### SSH Hardening (optional)

```bash
sudo nano /etc/ssh/sshd_config
# PasswordAuthentication no
# PubkeyAuthentication yes
# PermitRootLogin no

sudo systemctl restart sshd
```

## Step 7: Accept SSH Key from Notebook

On the P16v notebook:
```bash
ssh-copy-id savant@192.168.1.100
ssh td350   # should connect without password
```

## Verification Checklist

- [ ] Arch boots with `intel_iommu=on iommu=pt`
- [ ] IOMMU enabled: `dmesg | grep -i iommu`
- [ ] RTX 4090 bound to vfio-pci
- [ ] RX 6650 XT bound to vfio-pci
- [ ] Aspeed BMC available as host console
- [ ] SSH accessible from notebook
- [ ] QEMU build dependencies installed
- [ ] FluidSynth and SoundFont installed
- [ ] libvirtd running
- [ ] VM storage directories created
- [ ] Static IP configured
