# CHEATSHEET — P16v Daily Commands

Quick reference for the most used commands on the archp16v workstation.

---

## System Update

```bash
# Update everything (pacman + AUR)
yay -Syu

# Update npm
npm install -g npm@latest

# Update mirrors (Arch)
sudo reflector --age 12 --protocol https --sort rate --fastest 20 --save /etc/pacman.d/mirrorlist

# Update mirrors (EndeavourOS)
eos-rankmirrors
```

---

## Package Management

```bash
# Install official package
sudo pacman -S package

# Install AUR package
yay -S package

# Search official
sudo pacman -Ss package

# Search AUR
yay -Ss package

# Remove package + dependencies
sudo pacman -Rns package

# Clean orphan packages
sudo pacman -Rns $(pacman -Qdtq)

# Install Node package globally
npm install -g package
```

---

## Timeshift Snapshots

```bash
# Create snapshot
sudo timeshift --create --comments "description" --tags D

# List snapshots
sudo timeshift --list

# Restore snapshot
sudo timeshift --restore

# Open GUI
sudo timeshift-gtk
```

**Convention:** always create snapshot + push GitHub after each major install.

---

## Dotfiles / GitHub Sync

```bash
# Save package list + push
pacman -Qqe > ~/dotfiles/pkglist.txt
cd ~/dotfiles
git add pkglist.txt
git commit -m "step-description"
git push

# Restore packages after reinstall
sudo pacman -S --needed - < ~/dotfiles/pkglist.txt
```

---

## Git

```bash
# Clone qemu-sa
cd ~/Projects
git clone git@github.com:madlabnexus/qemu-sa.git

# Clone dotfiles
git clone git@github.com:madlabnexus/archp16v.git ~/dotfiles

# Status
git status

# Add + commit + push
git add .
git commit -m "message"
git push

# Pull latest
git pull
```

---

## NVIDIA / GPU

```bash
# GPU status
nvidia-smi

# Check active session
echo $XDG_SESSION_TYPE   # wayland

# Check display GPU
glxinfo | grep "OpenGL renderer"   # Intel Arc

# Run app on NVIDIA
prime-run app

# Run glxinfo on NVIDIA
prime-run glxinfo | grep "OpenGL renderer"   # NVIDIA RTX 3000 Ada
```

---

## Claude Code

```bash
# Open in current project
cd ~/Projects/qemu-sa
claude

# Check version
claude --version

# Model: Opus 4.6 for QEMU-SA dev, Sonnet for quick tasks
```

---

## Bluetooth

```bash
# Status
systemctl status bluetooth

# Pair device
bluetoothctl
> power on
> scan on
> pair XX:XX:XX:XX:XX:XX
> trust XX:XX:XX:XX:XX:XX
> connect XX:XX:XX:XX:XX:XX
> quit
```

---

## Systemd

```bash
# Check service status
systemctl status service

# Enable + start service
sudo systemctl enable --now service

# Restart service
sudo systemctl restart service

# Check logs
journalctl -u service -f
```

---

## TTY / Session

```bash
# Return to GNOME desktop (this machine)
Ctrl+Alt+F3

# Emergency TTY (text login)
Ctrl+Alt+F2  or  Ctrl+Alt+F4

# Check active sessions
loginctl list-sessions

# Kill extra session
loginctl terminate-session SESSION_ID
```

---

## Disk / btrfs

```bash
# List partitions
lsblk -f

# List btrfs subvolumes
sudo btrfs subvolume list /

# Check disk usage
df -h
```

---

## Network

```bash
# Test download speed / bandwidth
iperf3 -c server-ip

# Check open ports
ss -tulpn

# DNS lookup
nslookup domain.com
```

---

## Ghostty Config

```bash
# Edit config
nano ~/.config/ghostty/config

# List available themes
ghostty +list-themes

# Current theme: Flexoki Dark
```

---

## Power / Battery

```bash
# Check power profile
powerprofilesctl status

# Set performance mode
powerprofilesctl set performance

# Check battery
upower -i /org/freedesktop/UPower/devices/battery_BAT0

# Check RTC timezone (dual boot)
timedatectl
```

---

## Useful One-liners

```bash
# Find which package owns a file
pacman -Qo /path/to/file

# Find package content
pacman -Ql package

# Check what's eating disk
du -sh ~/* | sort -rh | head -20

# Watch GPU usage live
watch -n 1 nvidia-smi

# Check system temps
sensors

# Monitor system resources
btop
```

---

*This file lives at `docs/CHEATSHEET.md` in the qemu-sa repo.*
