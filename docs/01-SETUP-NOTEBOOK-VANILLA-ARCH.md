# 01 — P16v Notebook Setup (Dev Workstation)

## Hardware

- **Model:** Lenovo ThinkPad P16v Gen 2
- **CPU:** Intel Core Ultra 9 185H (6P+8E+2LP cores, 22 threads, 5.1GHz turbo)
- **RAM:** 32GB DDR5 5600MHz (2×16GB SODIMM)
- **GPU 1:** Intel Arc Pro Graphics (integrated)
- **GPU 2:** NVIDIA RTX 3000 Ada Laptop, 8GB GDDR6
- **NVMe 1:** UMIS 1TB → **Arch Linux (dev environment)**
- **NVMe 2:** BIWIN 4TB → Windows 11 (untouched)
- **Dock:** Lenovo ThinkPad Thunderbolt 4 Dock

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

Expected output (drive names **may be swapped** — verify by model name):
```
nvme0n1   931.5G  UMIS RPETJ1T24MKP2QDQ    ← WIPE THIS (or nvme1n1)
nvme1n1   3.6T    BIWIN NV7400 4TB          ← DO NOT TOUCH
```

> **⚠ WARNING:** NVMe device numbering (`nvme0n1` vs `nvme1n1`) varies between boots and BIOS versions. Always identify by SIZE and MODEL, not by number. The commands below use `nvme0n1` — **adjust every command** if your 1TB UMIS drive appears as `nvme1n1`.

### 3.3 Partition

```bash
sgdisk --zap-all /dev/nvme0n1

sgdisk -n 1:0:+1G -t 1:ef00 -c 1:"EFI" /dev/nvme0n1
sgdisk -n 2:0:+32G -t 2:8200 -c 2:"Swap" /dev/nvme0n1
sgdisk -n 3:0:0 -t 3:8300 -c 3:"Arch" /dev/nvme0n1

lsblk /dev/nvme0n1
```

### 3.4 Format and Mount

```bash
mkfs.fat -F 32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2

# Btrfs with compression
mkfs.btrfs -f /dev/nvme0n1p3

# Create subvolumes
mount /dev/nvme0n1p3 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
umount /mnt

# Mount subvolumes with optimized options
mount -o compress=zstd,noatime,ssd,discard=async,subvol=@ /dev/nvme0n1p3 /mnt
mkdir -p /mnt/{home,boot,.snapshots}
mount -o compress=zstd,noatime,ssd,discard=async,subvol=@home /dev/nvme0n1p3 /mnt/home
mount -o compress=zstd,noatime,ssd,discard=async,subvol=@snapshots /dev/nvme0n1p3 /mnt/.snapshots
mount /dev/nvme0n1p1 /mnt/boot
swapon /dev/nvme0n1p2
```

### 3.5 Install Base System

```bash
# Prevent vconsole.conf warning during pacstrap
mkdir -p /mnt/etc
echo "KEYMAP=us" > /mnt/etc/vconsole.conf

pacstrap -K /mnt \
  base linux linux-firmware \
  intel-ucode \
  btrfs-progs \
  nvidia-open nvidia-utils nvidia-settings \
  networkmanager wpa_supplicant iw wireless-regdb \
  bluez bluez-utils \
  sof-firmware \
  efibootmgr power-profiles-daemon \
  sudo vim nano \
  base-devel git \
  openssh dhcpcd iperf3 \
  man-db man-pages \
  bash-completion

genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

**Package notes:**
- `nvidia-open` — required for RTX 20xx+ since Dec 2025 Arch change (replaces `nvidia`)
- `btrfs-progs` — required for btrfs filesystem
- `sof-firmware` — Intel Sound Open Firmware (P16v audio hardware)
- `wpa_supplicant iw wireless-regdb` — WiFi stack (NetworkManager needs these)
- `bluez bluez-utils` — Bluetooth stack
- `dhcpcd` — wired DHCP client (NetworkManager alone may not get DHCP on ethernet)
- `efibootmgr` — UEFI boot entry management
- `power-profiles-daemon` — laptop power management
- `bash-completion` — tab completion for commands (including after `sudo`)

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
systemctl enable bluetooth
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

# Write boot entry with UUID expanded inline (unquoted EOF lets $(…) expand)
cat > /boot/loader/entries/arch.conf << EOF
title   Arch Linux (QEMU-SA Dev)
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p3) rw quiet rootflags=subvol=@ nvidia_drm.modeset=1 intel_iommu=on iommu=pt
EOF

# Verify — the options line must show the actual UUID, not a blank
cat /boot/loader/entries/arch.conf
```

> **⚠ IMPORTANT:** The second `cat` uses unquoted `EOF` (no single quotes) so that `$(blkid ...)` expands to the real UUID. If you see a blank UUID or literal `$(blkid ...)` in the output, delete the file and redo this step.

### 3.8 NVIDIA Configuration

```bash
# IMPORTANT: nvidia_drm (not nvidia_drv — common typo that breaks boot)
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

# Set WiFi to auto-connect on boot
nmcli connection modify "YOUR_SSID" connection.autoconnect yes

# Verify
ping -c 3 archlinux.org
```

### 4.2 KDE Plasma Desktop

```bash
sudo pacman -S \
  plasma-meta \
  sddm \
  plasma-x11-session kwin-x11 \
  xorg-xrandr \
  konsole dolphin kate ark spectacle \
  pipewire pipewire-alsa pipewire-pulse wireplumber \
  xdg-desktop-portal-kde \
  firefox \
  ttf-fira-code ttf-liberation noto-fonts \
  libreoffice-fresh \
  okular gwenview filelight

sudo systemctl enable sddm
```

**Package prompts — select these when asked:**
- JACK provider: `pipewire-jack` (option 2)
- qt6-multimedia-backend: `qt6-multimedia-ffmpeg` (option 1)

**Why `plasma-meta` instead of `plasma-desktop`:**

`plasma-desktop` is the bare minimum — no WiFi applet, no display management, no audio controls, no Bluetooth, no Thunderbolt support. `plasma-meta` pulls in everything needed for a functional desktop:

- `plasma-nm` — WiFi/network system tray applet
- `plasma-pa` — audio volume control in system tray
- `kscreen` — display/monitor management (arrangement, resolution, Compositor)
- `bluedevil` — Bluetooth management
- `plasma-thunderbolt` — Thunderbolt device management
- `plasma-disks` — disk health monitoring
- `plasma-systemmonitor` — system resource monitor
- `kinfocenter` — system information
- `breeze` + `breeze-gtk` — consistent theming across Qt/GTK apps
- `kde-gtk-config` — GTK application theming
- `kdeplasma-addons` — desktop widgets
- `plasma-browser-integration` — browser integration
- `discover` — software center
- `milou` — desktop search
- And more (~50 packages total)

**Why `plasma-x11-session kwin-x11`:**

Since Plasma 6.4, Wayland is the default and X11 is a separate package. NVIDIA + Wayland + Thunderbolt 4 dock causes complete system freezes. X11 is required for dock stability.

**Why `xorg-xrandr`:**

Not included in plasma-meta. Needed for display management from the command line (e.g., initial external monitor activation).

#### Configure SDDM to default to X11

```bash
sudo mkdir -p /etc/sddm.conf.d
sudo tee /etc/sddm.conf.d/session.conf << 'EOF'
[Desktop]
Session=plasmax11.desktop
EOF
```

#### Docked mode (lid closed, external monitors)

Thunderbolt dock displays don't enumerate until after the KDE session starts. This means the SDDM login screen will always appear on the built-in display (eDP-1) regardless of dock state. After login, KDE's kscreen takes over and switches to your configured external monitor layout.

**With lid closed:** Type your password blind and press Enter. KDE will activate external monitors and disable eDP-1 within seconds.

#### Enroll Thunderbolt dock for auto-authorization

By default, the Thunderbolt dock must be re-authorized each boot. Enrolling it makes authorization instant:

```bash
# Find the dock UUID
sudo boltctl list

# Enroll it (replace UUID with yours)
sudo boltctl enroll YOUR-DOCK-UUID
```

#### Prevent suspend on lid close (docked mode)

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
sudo tee /etc/systemd/logind.conf.d/lid.conf << 'EOF'
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
EOF
sudo systemctl restart systemd-logind
```

#### Reboot

```bash
sudo reboot
```

> **⚠ THUNDERBOLT DOCK:** If using a Thunderbolt dock, **disconnect it for the first login** after installing KDE. Log in once without the dock, then reconnect it. This avoids a known freeze on first SDDM boot with NVIDIA + dock.

After reboot, SDDM should present the login screen and default to the X11 session. Log in as `savant`.

#### After first login — enable external displays

With the Thunderbolt dock connected, external monitors may need initial activation:

```bash
# List detected displays
xrandr

# Enable detected displays (adjust output names to match your hardware)
xrandr --output DP-4-1-6 --auto    # Example: 4K monitor
xrandr --output DP-4-2 --auto       # Example: ultrawide
```

After initial xrandr activation, **System Settings → Display & Monitor** will manage them going forward (arrangement, resolution, primary display).

### 4.3 Development Tools

After logging into KDE, open Konsole:

```bash
# AUR helper
cd /tmp
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
cd ~
```

> **yay prompts explained:** yay asks up to three questions when installing AUR packages:
> - "Packages to install (eg: 1 2 3, 1-3 or ^4)" → press **Enter** (default — installs all)
> - "Packages to cleanBuild?" → type **N** (None)
> - "Diffs to show?" → type **N** (None)

```bash
# npm global prefix (avoids permission errors with npm install -g)
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Node.js + Claude Code
sudo pacman -S nodejs npm
npm install -g @anthropic-ai/claude-code

# Verify Claude Code
claude --version

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

# Claude Desktop App (unofficial AUR package — supports MCP)
yay -S claude-desktop-bin
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

---

## Known Issues & Workarounds

### NVIDIA + Wayland + Thunderbolt 4 Dock = Freeze

**Symptom:** Complete system freeze (keyboard, mouse, trackpad all unresponsive) when Thunderbolt dock is connected under Wayland.

**Cause:** NVIDIA driver + Wayland compositor + Thunderbolt dock display outputs.

**Solution:** Use X11 session exclusively. This is configured in Step 4.2 via SDDM session default and the `plasma-x11-session` package.

### Wired Ethernet DHCP

**Symptom:** Ethernet through dock or built-in port doesn't get an IP address.

**Cause:** NetworkManager may not acquire DHCP on wired connections without `dhcpcd`.

**Solution:** `dhcpcd` is included in the pacstrap packages (Step 3.5).

### External Monitors Not Showing

**Symptom:** Dock is working (USB, ethernet) but external monitors are black.

**Cause:** X11 doesn't always auto-activate new display outputs.

**Solution:** Run `xrandr --output <output-name> --auto` for each display. After first activation, KDE Display Settings manages them.

### SDDM Login Shows on Built-in Display When Docked

**Symptom:** With lid closed and dock connected, login screen appears on the laptop's built-in display instead of external monitors.

**Cause:** Thunderbolt dock displays don't enumerate until after the desktop session starts. SDDM's Xsetup script runs before the dock is ready, so it only sees eDP-1. This is a Thunderbolt initialization timing issue — not fixable with scripts or delays.

**Workaround:** Type your password blind and press Enter. KDE will switch to external monitors within seconds of login. This is cosmetic only — no functionality is lost.

---

## Verification Checklist

- [ ] Arch boots from 1TB NVMe (btrfs with subvolumes)
- [ ] Windows boots from 4TB NVMe via F12
- [ ] KDE Plasma desktop working (X11 session)
- [ ] NVIDIA drivers working (`nvidia-smi`)
- [ ] WiFi working (system tray applet visible)
- [ ] Audio working (volume control in system tray)
- [ ] Bluetooth available in settings
- [ ] Thunderbolt dock functional (USB, ethernet, displays)
- [ ] External monitors detected and configurable
- [ ] Display & Monitor settings show arrangement panel
- [ ] Lid close does not suspend when docked
- [ ] After login, KDE switches to external monitors (built-in disabled)
- [ ] Thunderbolt dock enrolled (`sudo boltctl list` shows `stored: yes`)
- [ ] Tab completion working (including after `sudo`)
- [ ] Claude Code installed (`claude --version`)
- [ ] Claude Desktop app available in application menu
- [ ] Git configured with SSH key on GitHub
- [ ] VS Code installed
- [ ] Docker running
