# 01 — P16v Notebook Setup — EndeavourOS (Dev Workstation)

## Hardware

- **Model:** Lenovo ThinkPad P16v Gen 2
- **CPU:** Intel Core Ultra 9 185H (6P+8E+2LP cores, 22 threads, 5.1GHz turbo)
- **RAM:** 32GB DDR5 5600MHz (2×16GB SODIMM)
- **GPU 1:** Intel Arc Pro Graphics (integrated)
- **GPU 2:** NVIDIA RTX 3000 Ada Laptop, 8GB GDDR6
- **NVMe 1:** UMIS 1TB → **EndeavourOS (dev environment)**
- **NVMe 2:** BIWIN 4TB → Windows 11 (untouched)
- **Dock:** Lenovo ThinkPad Thunderbolt 4 Dock

## Role

This machine is the cockpit. It runs the desktop environment, code editor, Claude Code, git, and SSH sessions to the TD350 server. It does NOT build QEMU or run retro VMs.

## Why EndeavourOS + GNOME

**EndeavourOS** is a thin layer on top of Arch Linux that provides a graphical installer (Calamares) while keeping everything else pure Arch — same repos, same pacman, same AUR.

**GNOME** was chosen after testing KDE Plasma and Cinnamon:
- **Wayland + NVIDIA + Thunderbolt dock** works out of the box — no freezes, no scripts
- **GDM login screen** displays on all monitors natively — no display manager hacks
- **Lid close + dock/undock** handled seamlessly — no custom scripts needed
- Zero workarounds required for basic desktop functionality

**Other DEs tested and rejected:**
- **KDE Plasma:** Froze completely with Wayland + NVIDIA + Thunderbolt dock. Required X11 fallback, SDDM display scripts, manual window placement config, lid close overrides.
- **Cinnamon:** Worked well with Wayland + NVIDIA, but SDDM login screen failed to display on external monitors reliably. Mirror scripts broke when desktop display settings changed.
- **CachyOS:** Blank screen on boot — incompatible with this hardware regardless of DE.

---

## Step 1: BIOS Configuration

Reboot, press **F1** during Lenovo splash to enter BIOS.

1. **Security → Secure Boot:** Disabled
2. **Config → USB → USB UEFI BIOS Support:** Enabled
3. **Config → Display → Graphics Device:** Discrete Graphics (forces NVIDIA as primary, simpler Linux driver setup)
4. **Startup → UEFI/Legacy Boot:** UEFI Only
5. **Config → Thunderbolt → Security Level:** No Security

Save and exit (F10).

## Step 2: Prepare EndeavourOS USB

On Windows 11 (booted from the 4TB drive):

1. Download latest EndeavourOS ISO: https://endeavouros.com/download/
2. Download Ventoy: https://ventoy.net/
3. Install Ventoy on a USB drive
4. Copy the EndeavourOS ISO onto the Ventoy USB
5. Reboot, press **F12**, select USB device

## Step 3: EndeavourOS Installation (Calamares)

### 3.1 Boot the Live Environment

When the EndeavourOS boot menu appears, select the **default** entry. Connect to WiFi using the network applet in the taskbar if not on ethernet.

### 3.2 Launch Installer

The Welcome app should appear automatically. Click **Install EndeavourOS** to launch Calamares.

### 3.3 Installer Screens

#### Location & Keyboard
- **Location:** Your timezone (America/Sao_Paulo)
- **Keyboard:** Portuguese (Brazil) — `pt-BR` layout

#### Desktop Environment
- **Select:** GNOME

#### Packages
- Keep defaults. EndeavourOS will install base Arch system, GNOME desktop, GDM display manager, Firefox, NetworkManager, PipeWire, and yay (AUR helper).

#### Partitioning — ⚠ CRITICAL STEP

Select **Manual Partitioning** (not "Erase Disk" — that could wipe the wrong drive).

**Identify the correct drive by size and model:**
```
UMIS 1TB  (~931 GB)  ← INSTALL HERE
BIWIN 4TB (~3.6 TB)  ← DO NOT TOUCH
```

> **⚠ WARNING:** NVMe device numbering (`nvme0n1` vs `nvme1n1`) varies between boots. Always identify by SIZE and MODEL, never by device name.

**Create these partitions on the 1TB UMIS drive:**

| # | Size | Type | Mount | Filesystem | Flags |
|---|------|------|-------|------------|-------|
| 1 | 1 GB | EFI System Partition | `/efi` | FAT32 | boot, esp |
| 2 | ~34 GB | Linux swap | swap | linuxswap | — |
| 3 | Remaining (~898 GB) | Linux filesystem | `/` | **btrfs** | — |

**Btrfs subvolume note:** Calamares automatically creates subvolumes: `@` (root), `@home`, `@cache` (var/cache), `@log` (var/log) — all with zstd compression.

**Swap for hibernation:** Swap size matches RAM (32GB), enabling hibernation later if needed.

#### Boot Loader
- **systemd-boot** (EndeavourOS default for UEFI)
- Calamares handles the boot entries automatically

#### User Account
- **Username:** `madlabnexus`
- **Hostname:** `p16v`
- **Password:** your choice
- **Auto-login:** No (security)

### 3.4 Install

Click **Install** and wait. Calamares handles everything: formatting, base install, bootloader, fstab, locale, user creation, initramfs (dracut).

Reboot when prompted. Remove the USB.

---

## Step 4: First Boot — Post-Install Configuration

Log into GNOME. Open a terminal (press `Super` key, type "Terminal").

### 4.1 Verify Basics

```bash
hostnamectl
# Should show: hostname=p16v, OS=EndeavourOS

mount | grep btrfs
# Should show subvol=/@, compress=zstd (plus @home, @cache, @log)

swapon --show
# Should show swap partition

echo $XDG_SESSION_TYPE
# Should show: wayland

ping -c 3 archlinux.org
```

### 4.2 Optimize Mirrors and Update

```bash
eos-rankmirrors
sudo pacman -Syu

# Backup optimized mirror lists (restore with sudo cp *.backup to original filenames)
sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
sudo cp /etc/pacman.d/endeavouros-mirrorlist /etc/pacman.d/endeavouros-mirrorlist.backup
```

### 4.3 Install Claude Desktop (Early — For Copy/Paste Convenience)

```bash
yay -S claude-desktop-bin
```

> **yay prompts:** When asked:
> - "Packages to install (eg: 1 2 3)" → press **Enter** (default)
> - "Packages to cleanBuild?" → type **N**
> - "Diffs to show?" → type **N**

### 4.4 NVIDIA Driver Setup

Check if NVIDIA drivers were installed during Calamares:

```bash
nvidia-smi
```

**If nvidia-smi works:** Skip to NVIDIA Initramfs below.

**If nvidia-smi fails:**

```bash
sudo pacman -S nvidia-open nvidia-utils nvidia-settings
# nvidia-open is required for RTX 20xx+ since Dec 2025 Arch change
```

#### NVIDIA Initramfs (dracut)

EndeavourOS uses **dracut** (not mkinitcpio). Verify with `which dracut`. Add NVIDIA modules:

```bash
sudo tee /etc/dracut.conf.d/nvidia.conf << 'EOF'
force_drivers+=" nvidia nvidia_modeset nvidia_uvm nvidia_drm "
EOF

sudo dracut --force
```

> **IMPORTANT:** `nvidia_drm` not `nvidia_drv` — common typo that breaks boot.

#### NVIDIA Kernel Parameters

Check if `nvidia_drm.modeset=1` is already in the kernel command line:

```bash
cat /proc/cmdline
```

If missing, add it persistently via `/etc/kernel/cmdline` (survives kernel updates). Copy your existing cmdline and append the new params:

```bash
# First, see your current options
cat /proc/cmdline

# Then write them with the additions (replace UUIDs with YOUR values from above)
sudo tee /etc/kernel/cmdline << 'EOF'
nvme_load=YES nowatchdog rw rootflags=subvol=/@ root=UUID=YOUR-ROOT-UUID resume=UUID=YOUR-SWAP-UUID systemd.machine_id=YOUR-MACHINE-ID nvidia_drm.modeset=1 intel_iommu=on iommu=pt
EOF
```

> **⚠ IMPORTANT:** Replace `YOUR-ROOT-UUID`, `YOUR-SWAP-UUID`, and `YOUR-MACHINE-ID` with the actual values from your `cat /proc/cmdline` output. Do NOT copy placeholders literally.

Reinstall the kernel entry and verify:

```bash
sudo kernel-install add $(uname -r) /usr/lib/modules/$(uname -r)/vmlinuz

# Find the boot entry files
sudo find /efi -name "*.conf" 2>/dev/null

# Verify the main entry (adjust filename to match your machine-id and kernel)
sudo cat /efi/loader/entries/YOUR-MACHINE-ID-$(uname -r).conf
# options line should include nvidia_drm.modeset=1 intel_iommu=on iommu=pt
```

Reboot and confirm:

```bash
sudo reboot
# After reboot:
nvidia-smi
cat /proc/cmdline
```

### 4.5 GNOME Enhancements

#### Terminal with Transparency

GNOME Console (kgx) doesn't support transparency. Install **Ghostty** — a modern GPU-accelerated terminal with transparency, ligatures, and splits:

```bash
sudo pacman -S ghostty
```

Configure transparency in Ghostty's config file (`~/.config/ghostty/config`):
```
background-opacity = 0.85
```

#### Tweaks and Extensions

```bash
sudo pacman -S gnome-tweaks gnome-shell-extension-appindicator gnome-browser-connector
yay -S gnome-shell-extension-blur-my-shell gnome-shell-extension-dash-to-dock gnome-shell-extension-clipboard-indicator gnome-shell-extension-caffeine
```

**Log out and back in** (Wayland requires a full session restart for new extensions), then enable them:

```bash
gnome-extensions enable blur-my-shell@aunetx
gnome-extensions enable dash-to-dock@micxgx.gmail.com
gnome-extensions enable clipboard-indicator@tudmotu.com
gnome-extensions enable appindicatorsupport@rgcjonas.gmail.com
gnome-extensions enable caffeine@patapon.info
```

**Extension descriptions:**
- **Blur my Shell** — transparency + blur on top bar, overview, dash, and app windows
- **Dash to Dock** — persistent dock at bottom (like macOS) instead of hidden sidebar
- **Clipboard Indicator** — clipboard history manager
- **AppIndicator** — system tray support (needed for VS Code, Claude Desktop tray icons)
- **Caffeine** — quick toggle to prevent screen sleep

#### Install extensions from the web

`gnome-browser-connector` enables installing extensions directly from Firefox. Browse https://extensions.gnome.org and toggle extensions on/off from the website.

Recommended additional extensions (install via the website):
- **User Themes** — apply custom GNOME Shell themes from `~/.themes`: https://extensions.gnome.org/extension/19/user-themes/
- **Transparent Window Moving** — windows become transparent while dragging/resizing: https://extensions.gnome.org/extension/1446/transparent-window-moving/

Configure all extensions via the built-in **GNOME Extensions** app (search "Extensions" in Activities).

### 4.6 Prevent Suspend on Lid Close

GNOME 49+ removed the lid close setting from the GUI entirely. Use systemd-logind:

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

This works both before and after login. No additional GUI config needed.

### 4.7 Thunderbolt Dock Setup

Enroll the dock for auto-authorization (no re-auth on each boot):

```bash
sudo boltctl list
sudo boltctl enroll YOUR-DOCK-UUID
```

### 4.8 Reboot and Verify

```bash
sudo reboot
```

Test all scenarios:

1. **Lid closed + dock:** GDM login screen appears on external monitors. Type password, log in. GNOME starts with external monitors.
2. **Undock:** Built-in display activates automatically.
3. **Redock:** External monitors reconnect, GNOME restores layout.
4. **Lid close after login:** System stays awake, external monitors active.

---

## Step 5: Btrfs Snapshots (Timeshift)

Take a snapshot now — the base system is working and any mistakes from here can be rolled back.

### 5.1 Install and Configure Timeshift

```bash
sudo pacman -S timeshift
sudo systemctl enable --now cronie.service
```

> **Note:** Timeshift requires cronie for scheduled snapshots. Without it, only manual snapshots work.

Open Timeshift from Activities:

1. **Snapshot Type:** Select **BTRFS** (not rsync)
2. **Snapshot Location:** Auto-detects your btrfs root
3. Click **Create** to take a snapshot
4. Add a comment: "Base install — NVIDIA + GNOME + dock working"

### 5.2 Restore from a Snapshot

If something breaks: open Timeshift, select a snapshot, click **Restore**, reboot.

If the system won't boot: boot from the EndeavourOS USB, install Timeshift in the live session, and restore from there.

### 5.3 Recommended Snapshot Schedule

In Timeshift **Settings**, configure automatic snapshots: keep 5 daily, 3 weekly, 1 monthly.

Good habit: take a manual snapshot before kernel updates, NVIDIA driver changes, or major package installs.

---

## Step 6: Development Tools

### 6.1 Additional Desktop Packages

```bash
sudo pacman -S \
  evince eog \
  libreoffice-fresh \
  ttf-fira-code ttf-liberation noto-fonts
```

**Package notes:**
- `evince` — GNOME PDF/document viewer
- `eog` — GNOME image viewer (Eye of GNOME)

(Skip any already installed — pacman will tell you.)

### 6.2 Core Dev Tools

```bash
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
  docker docker-compose \
  openssh \
  nmap \
  iperf3 \
  net-tools \
  base-devel \
  bash-completion

sudo usermod -aG docker madlabnexus
sudo systemctl enable --now docker
sudo systemctl enable --now sshd
```

### 6.3 AUR Packages

```bash
# VS Code
yay -S visual-studio-code-bin

# Claude Desktop already installed in Step 4.3
```

#### Set VS Code as Default Text Editor

```bash
for type in text/plain text/x-python application/x-shellscript text/markdown text/x-c text/x-c++ text/x-java application/json application/xml text/css text/html text/javascript; do
    xdg-mime default code.desktop "$type"
done
```

### 6.4 Node.js + Claude Code

```bash
sudo pacman -S nodejs npm

# npm global prefix (avoids permission errors)
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

npm install -g @anthropic-ai/claude-code
claude --version
```

### 6.5 SSH Keys & Git

```bash
ssh-keygen -t ed25519 -C "madlabnexus@p16v"
cat ~/.ssh/id_ed25519.pub
# Add this key to GitHub: github.com → Settings → SSH Keys

git config --global user.name "Madlab Nexus"
git config --global user.email "madlabnexus@gmail.com"
git config --global init.defaultBranch main
git config --global pull.rebase true
```

### 6.6 Clone QEMU-SA Repo

```bash
mkdir -p ~/projects
cd ~/projects
git clone git@github.com:madlabnexus/qemu-sa.git
```

### 6.7 SSH Config for Server

```bash
cat >> ~/.ssh/config << 'EOF'

Host td350
    HostName 192.168.1.100
    User madlabnexus
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes
EOF
```

Adjust the IP address after the server is set up.

### 6.8 Desktop Productivity & Media

```bash
sudo pacman -S \
  vlc \
  wireguard-tools \
  chromium \
  thunderbird \
  obsidian \
  transmission-gtk \
  unrar p7zip unzip

yay -S spotify
```

**Package descriptions:**
- **vlc** — plays any video/audio format
- **spotify** — music streaming (AUR)
- **wireguard-tools** — VPN client (configure later with your router)
- **chromium** — second browser for testing, separate profiles
- **thunderbird** — email client
- **obsidian** — markdown notes/knowledge base (great for project docs)
- **transmission-gtk** — lightweight torrent client
- **unrar / p7zip / unzip** — extract any archive format

> **Note:** Firefox and LibreOffice were already installed in earlier steps.

### 6.9 Final Snapshot

Open Timeshift, create a new snapshot with comment: "Full setup complete — all dev tools + apps installed".

---

## Step 7: Dual Boot & Boot Menu

### Verify Dual Boot

Reboot, press **F12** (BIOS boot menu):
- Select UMIS 1TB → EndeavourOS ✓
- Select BIWIN 4TB → Windows 11 ✓

### systemd-boot Menu

systemd-boot auto-detects Windows Boot Manager on the shared EFI partition. To see available entries:

```bash
bootctl list
```

Set a 5-second timeout so you can choose Windows at boot without F12:

```bash
sudo tee /efi/loader/loader.conf << 'EOF'
default de5c1a4f5b284e64a4a19027e26d638d-*
timeout 5
console-mode auto
EOF
```

On reboot, systemd-boot shows a menu for 5 seconds with EndeavourOS (default) and Windows Boot Manager. Press arrow keys to select, Enter to boot.

**If Windows is NOT auto-detected** (doesn't appear in `bootctl list`), create a manual entry:

```bash
sudo tee /efi/loader/entries/windows.conf << 'EOF'
title   Windows 11
efi     /EFI/Microsoft/Boot/bootmgfw.efi
EOF
```

> **Note:** This only works if both OSes share the same EFI partition. If Windows has its own EFI partition on the 4TB drive, use F12 at BIOS to select it.

**To make Windows the default** (not recommended for a dev machine):

```bash
sudo tee /efi/loader/loader.conf << 'EOF'
default auto-windows
timeout 5
console-mode auto
EOF
```

---

## Known Issues & Workarounds

### CachyOS Doesn't Work on This Hardware

**Symptom:** Blank screen on boot, no video output.

**Cause:** CachyOS kernel/driver configuration incompatible with P16v Gen 2 hardware (NVIDIA RTX 3000 Ada + Intel Arc iGPU combo).

**Solution:** Use EndeavourOS instead.

### KDE Plasma + Wayland + NVIDIA + Thunderbolt = Freeze

**Symptom:** Complete system freeze with KDE Plasma when Thunderbolt dock is connected under Wayland.

**Solution:** Use GNOME instead. Tested and works flawlessly with Wayland + NVIDIA + Thunderbolt dock.

### Cinnamon + SDDM Login Screen Issues

**Symptom:** Cinnamon desktop worked well, but SDDM login screen failed to display on external monitors reliably. Mirror scripts broke when desktop display settings changed.

**Solution:** Use GNOME instead. GDM handles multi-monitor login natively without scripts.

### GNOME 49+ Removed Lid Close Setting

**Symptom:** No GUI option to prevent suspend on lid close in GNOME Settings or Tweaks.

**Cause:** GNOME 49 removed `lid-close-ac-action` from gsettings. Lid behavior is delegated entirely to systemd-logind.

**Solution:** systemd-logind `lid.conf` override (configured in Step 4.6).

---

## Key Differences from Vanilla Arch Guide

| What | Vanilla Arch | EndeavourOS |
|------|-------------|-------------|
| Installer | Manual CLI (pacstrap, chroot, etc.) | Calamares GUI |
| Partitioning | sgdisk CLI | Calamares partition editor |
| Boot loader | Manual systemd-boot setup | Auto-configured |
| Initramfs | mkinitcpio | dracut |
| NVIDIA in initramfs | `/etc/mkinitcpio.conf.d/nvidia.conf` | `/etc/dracut.conf.d/nvidia.conf` |
| Kernel params | Manual boot entry editing | `/etc/kernel/cmdline` + `kernel-install` |
| EFI mount point | `/boot` | `/efi` |
| Boot entries | `/boot/loader/entries/` | `/efi/loader/entries/` (auto-managed) |
| Desktop | KDE Plasma (X11, forced) | GNOME (Wayland, native) |
| Display manager | SDDM (with mirror scripts) | GDM (works out of the box) |
| Lid close config | systemd-logind + KDE GUI | systemd-logind only |
| Multi-monitor login | Script workaround | Native |
| AUR helper | Manual yay build | Pre-installed |
| PipeWire | Manual pacman install | Installed by default |
| Username | `savant` | `madlabnexus` |
| Time to install | ~45 minutes | ~15 minutes |

---

## Verification Checklist

- [ ] EndeavourOS boots from 1TB NVMe (btrfs with subvolumes)
- [ ] Windows boots from 4TB NVMe (F12 or systemd-boot menu)
- [ ] systemd-boot shows 5-second menu with EndeavourOS + Windows
- [ ] GNOME desktop working (Wayland session)
- [ ] NVIDIA drivers working (`nvidia-smi`)
- [ ] GDM login screen displays on external monitors
- [ ] Lid close: system stays awake (login screen and desktop)
- [ ] Dock: external monitors work, undock/redock seamless
- [ ] WiFi working
- [ ] Audio working
- [ ] Bluetooth available
- [ ] Thunderbolt dock enrolled (`sudo boltctl list` shows `stored: yes`)
- [ ] GNOME extensions active (Blur my Shell, Dash to Dock, AppIndicator, Clipboard Indicator, Caffeine)
- [ ] Btrfs snapshots created (Timeshift) — base + full setup
- [ ] Cronie enabled for scheduled snapshots
- [ ] Tab completion working (including after `sudo`)
- [ ] Claude Code installed (`claude --version`)
- [ ] Claude Desktop app available in Activities
- [ ] Git configured with SSH key on GitHub
- [ ] QEMU-SA repo cloned (`~/projects/qemu-sa`)
- [ ] VS Code installed
- [ ] Docker running
- [ ] SSH config for td350 server
- [ ] Spotify, VLC, Chromium, Thunderbird, Obsidian available in Activities
- [ ] WireGuard tools installed (`wg --version`)
