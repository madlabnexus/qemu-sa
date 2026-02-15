# 01 — P16v Notebook Setup (Dev Workstation)

## Hardware

- **Model:** Lenovo ThinkPad P16v Gen 2
- **CPU:** Intel Core Ultra 9 185H (6P+8E+2LP cores, 22 threads, 5.1GHz turbo)
- **RAM:** 32GB DDR5 5600MHz (2×16GB SODIMM)
- **GPU 1:** Intel Arc Pro Graphics (integrated)
- **GPU 2:** NVIDIA RTX 3000 Ada Laptop, 8GB GDDR6
- **NVMe 1:** UMIS 1TB → **Arch Linux (dev environment)**
- **NVMe 2:** BIWIN 4TB → Windows 11 (untouched)

## Role

This machine is the cockpit. It runs the desktop environment, code editor, Claude Code, git, and SSH sessions to the TD350 server. It does NOT build QEMU or run retro VMs.

---

## Step 1: BIOS Configuration

Reboot, press **F1** during Lenovo splash to enter BIOS.

1. **Security → Secure Boot:** Disabled
2. **Config → USB → USB UEFI BIOS Support:** Enabled
3. **Config → Display → Graphics Device:** Discrete Graphics (forces NVIDIA as primary, simpler Linux driver setup)
4. **Startup → UEFI/Legacy Boot:** UEFI Only
5. **Config → Thunderbolt → Security Level:** No Security

Save and exit (F10).

## Step 2: Prepare Arch Linux USB

On Windows 11 (booted from the 4TB drive):

1. Download latest Arch ISO: https://archlinux.org/download/
2. Download Ventoy: https://ventoy.net/
3. Install Ventoy on a USB drive
4. Copy the Arch ISO onto the Ventoy USB
5. Reboot, press **F12**, select USB device

## Step 3: Arch Installation

### 3.1 Network

```bash
# Ethernet (if available)
ip addr

# WiFi
iwctl
station wlan0 scan
station wlan0 get-networks
station wlan0 connect YOUR_SSID
exit

ping -c 3 archlinux.org
```

### 3.2 Identify the 1TB NVMe

```bash
lsblk -d -o NAME,SIZE,MODEL
```

Expected output:
```
nvme0n1   931.5G  UMIS RPETJ1T24MKP2QDQ    ← WIPE THIS
nvme1n1   3.6T    BIWIN NV7400 4TB          ← DO NOT TOUCH
```

**TRIPLE CHECK before proceeding.** The commands below use `nvme0n1` — adjust if your 1TB drive has a different name.

### 3.3 Partition

```bash
sgdisk --zap-all /dev/nvme0n1

sgdisk -n 1:0:+1G -t 1:ef00 -c 1:"EFI" /dev/nvme0n1
sgdisk -n 2:0:+32G -t 2:8200 -c 2:"Swap" /dev/nvme0n1
sgdisk -n 3:0:0 -t 3:8300 -c 3:"Arch" /dev/nvme0n1

lsblk /dev/nvme0n1
```

### 3.4 Format

```bash
mkfs.fat -F 32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3
```

### 3.5 Install Base System

```bash
mount /dev/nvme0n1p3 /mnt
mount --mkdir /dev/nvme0n1p1 /mnt/boot
swapon /dev/nvme0n1p2

pacstrap -K /mnt base linux linux-firmware \
  intel-ucode \
  nvidia nvidia-utils nvidia-settings \
  networkmanager \
  sudo vim nano \
  base-devel git \
  openssh \
  man-db man-pages

genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

### 3.6 System Configuration

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
echo "p16v" > /etc/hostname
cat > /etc/hosts << 'EOF'
127.0.0.1   localhost
::1         localhost
127.0.1.1   p16v.localdomain p16v
EOF

# Passwords
passwd
useradd -m -G wheel,video,audio,input -s /bin/bash savant
passwd savant

# Sudo
EDITOR=nano visudo
# Uncomment: %wheel ALL=(ALL:ALL) ALL

# Services
systemctl enable NetworkManager
systemctl enable sshd
```

### 3.7 Boot Loader

```bash
bootctl install

cat > /boot/loader/loader.conf << 'EOF'
default arch.conf
timeout 5
console-mode max
editor no
EOF

# Get root partition UUID
blkid -s UUID -o value /dev/nvme0n1p3
# Note the output — you need it below

cat > /boot/loader/entries/arch.conf << 'EOF'
title   Arch Linux (QEMU-SA Dev)
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=UUID=REPLACE_WITH_ACTUAL_UUID rw quiet nvidia_drm.modeset=1
EOF

# EDIT the file and replace REPLACE_WITH_ACTUAL_UUID with the real UUID:
nano /boot/loader/entries/arch.conf
```

### 3.8 NVIDIA Configuration

```bash
cat > /etc/mkinitcpio.conf.d/nvidia.conf << 'EOF'
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
EOF

mkinitcpio -P
```

### 3.9 Reboot

```bash
exit
umount -R /mnt
swapoff /dev/nvme0n1p2
reboot
```

Remove USB. Press **F12** to boot from the 1TB NVMe.

## Step 4: First Boot — Desktop & Tools

Log in as `savant` at the text console.

### 4.1 Network

```bash
# WiFi
nmcli device wifi connect "YOUR_SSID" password "YOUR_PASSWORD"

# Or ethernet auto-connects
nmcli connection show
```

### 4.2 KDE Plasma Desktop

```bash
sudo pacman -S plasma-desktop sddm \
  konsole dolphin kate ark spectacle \
  pipewire pipewire-alsa pipewire-pulse wireplumber \
  xdg-desktop-portal-kde \
  firefox \
  ttf-fira-code ttf-liberation noto-fonts

sudo systemctl enable sddm
sudo reboot
```

### 4.3 Development Tools

After logging into KDE, open Konsole:

```bash
# AUR helper
cd /tmp
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
cd ~

# Node.js + Claude Code
sudo pacman -S nodejs npm
npm install -g @anthropic-ai/claude-code

# Dev tools
sudo pacman -S \
  git git-lfs \
  python python-pip \
  cmake meson ninja \
  gcc clang \
  gdb valgrind strace \
  htop btop \
  tmux \
  rsync \
  jq ripgrep fd \
  docker docker-compose

sudo usermod -aG docker savant
sudo systemctl enable docker

# VS Code
yay -S visual-studio-code-bin
```

### 4.4 SSH Keys & Git

```bash
ssh-keygen -t ed25519 -C "savant@p16v"
cat ~/.ssh/id_ed25519.pub
# Add this key to your GitHub account: github.com → Settings → SSH Keys

git config --global user.name "Savant"
git config --global user.email "your-email@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase true
```

### 4.5 Clone QEMU-SA Repo

```bash
mkdir -p ~/projects
cd ~/projects
git clone git@github.com:madlabnexus/qemu-sa.git
```

### 4.6 SSH Config for Server

```bash
cat >> ~/.ssh/config << 'EOF'

Host td350
    HostName 192.168.1.100
    User savant
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes
EOF
```

Adjust the IP address after the server is set up.

## Step 5: Dual Boot Verification

Reboot, press **F12**:
- Select UMIS 1TB → Arch Linux ✓
- Select BIWIN 4TB → Windows 11 ✓

Set the Arch NVMe as default in BIOS (F1 → Startup → Boot Priority).

## Verification Checklist

- [ ] Arch boots from 1TB NVMe
- [ ] Windows boots from 4TB NVMe via F12
- [ ] KDE Plasma desktop working
- [ ] NVIDIA drivers working (`nvidia-smi`)
- [ ] Network working
- [ ] Git configured with SSH key on GitHub
- [ ] Claude Code installed (`claude --version`)
- [ ] VS Code or Kate installed
- [ ] Docker running
