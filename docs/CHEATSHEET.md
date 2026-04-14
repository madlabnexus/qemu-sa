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

## System Maintenance

```bash
# Check active maintenance timers
systemctl list-timers --all | grep -E "fstrim|paccache|reflector"

# Manual pacman cache cleanup (keep last 3 versions)
sudo paccache -r

# Manual mirror ranking
sudo reflector --age 12 --protocol https --sort rate --fastest 20 --save /etc/pacman.d/mirrorlist

# Check journal disk usage
journalctl --disk-usage

# Vacuum journal manually (force cap)
sudo journalctl --vacuum-size=500M
```

**Enabled timers:**
- `fstrim.timer` — weekly NVMe TRIM
- `paccache.timer` — weekly pacman cache cleanup (3 versions kept)
- `reflector.timer` — weekly mirror ranking

---

## Firewall (firewalld)

```bash
# Check active zones
sudo firewall-cmd --get-active-zones

# List rules for home zone
sudo firewall-cmd --zone=home --list-all

# Add a service (e.g., for Samba server)
sudo firewall-cmd --zone=home --add-service=samba --permanent
sudo firewall-cmd --reload

# Open a specific port
sudo firewall-cmd --zone=home --add-port=8080/tcp --permanent
sudo firewall-cmd --reload

# After reinstall: reassign interfaces to home zone
sudo firewall-cmd --zone=home --change-interface=br0 --permanent
sudo firewall-cmd --zone=home --change-interface=eth0 --permanent
sudo firewall-cmd --zone=home --change-interface=enp0s31f6 --permanent
sudo firewall-cmd --reload
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

## GNOME Tips

```bash
# Ordenar apps alfabeticamente no menu (logout/login depois)
gsettings set org.gnome.shell app-picker-layout "[]"

# Listar extensões instaladas
gnome-extensions list

# Reiniciar GNOME Shell (X11 only)
Alt+F2 → digitar "r" → Enter

# Abrir Extension Manager
extension-manager
```

---

## NTFS Partitions (Windows)

```bash
# Se NTFS não monta ("Windows is hibernated")
sudo mount -t ntfs-3g -o remove_hiberfile /dev/nvme0n1p2 /mnt && sudo umount /mnt
sudo mount -t ntfs-3g -o remove_hiberfile /dev/nvme0n1p5 /mnt && sudo umount /mnt

# Verificar estado NTFS
sudo ntfsfix /dev/nvme0n1p2

# Fix definitivo: no Windows desativar Fast Startup
# Control Panel → Power Options → Choose what the power buttons do
# → desmarcar "Turn on fast startup"
# OU CMD admin: powercfg /h off
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

# Check interfaces and connections
ip link show
nmcli device status
nmcli connection show
```

---

## Network Bridge (br0) — QEMU Bridged LAN

```bash
# Check bridge status
bridge link show
ip addr show br0

# Recreate bridge after reinstall
sudo nmcli connection add type bridge con-name br0 ifname br0 \
  ipv4.method auto ipv6.method auto stp no
sudo nmcli connection add type ethernet con-name br0-dock ifname eth0 master br0
sudo nmcli connection add type ethernet con-name br0-builtin ifname enp0s31f6 master br0
sudo nmcli connection up br0

# Restore bridge.conf from dotfiles
sudo mkdir -p /etc/qemu
sudo cp ~/dotfiles/etc/qemu/bridge.conf /etc/qemu/
cp ~/dotfiles/etc/qemu/bridge.conf ~/Applications/qemu-3dfx/install/etc/qemu/
mkdir -p ~/Projects/qemu-build/qemu-bundle/usr/local/etc/qemu
cp ~/dotfiles/etc/qemu/bridge.conf ~/Projects/qemu-build/qemu-bundle/usr/local/etc/qemu/

# Restore setuid on bridge helpers
sudo chown root:root ~/Applications/qemu-3dfx/install/libexec/qemu-bridge-helper
sudo chmod u+s ~/Applications/qemu-3dfx/install/libexec/qemu-bridge-helper
sudo chown root:root ~/Projects/qemu-build/qemu-bridge-helper
sudo chmod u+s ~/Projects/qemu-build/qemu-bridge-helper

# QEMU bridged networking flags
# Win98: -netdev bridge,id=net0,br=br0 -device rtl8139,netdev=net0
# WinXP: -netdev bridge,id=net0,br=br0 -device e1000,netdev=net0
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

## Git — Resolver Conflitos de Sincronismo

```bash
# Push rejeitado (remote tem commits que não tens)
git pull --rebase && git push

# Ver ficheiros em conflito
git status

# Aceitar versão remota
git checkout --theirs path/to/file
git add path/to/file
git rebase --continue

# Aceitar versão local
git checkout --ours path/to/file
git add path/to/file
git rebase --continue

# Abortar rebase
git rebase --abort
```

---

## NordVPN

```bash
# Conectar (servidor mais rápido)
nordvpn connect

# Conectar a país específico
nordvpn connect br
nordvpn connect us

# Desconectar
nordvpn disconnect

# Ver status
nordvpn status

# Kill switch (bloqueia se VPN cair)
nordvpn set killswitch on
nordvpn set killswitch off
```

---

## LibreOffice

```bash
# Abrir Writer
libreoffice --writer

# Abrir Calc
libreoffice --calc

# Abrir Impress (apresentações)
libreoffice --impress

# Verificar dicionários instalados
ls /usr/share/hunspell/

# Instalar dicionário PT-BR manualmente:
# Tools → Extension Manager → Get more extensions → procurar "VERO"
```

---

## Wireless Cracking

```bash
# Modo monitor ON
sudo airmon-ng start wlan0

# Modo monitor OFF
sudo airmon-ng stop wlan0mon

# Scan de redes
sudo airodump-ng wlan0mon

# Capturar handshake (PMKID + 4-way)
sudo hcxdumptool -i wlan0mon -o capture.pcapng

# Converter para hashcat
hcxpcapngtool -o hash.hc22000 capture.pcapng

# Crack com GPU (WPA2)
hashcat -m 22000 hash.hc22000 wordlist.txt -d 1

# Crack com GPU (status)
hashcat -m 22000 hash.hc22000 --show

# Spoof MAC
sudo macchanger -r wlan0
sudo macchanger -p wlan0   # restaurar original
```

---

## Screenshots (GNOME 49 Wayland)

```bash
# Workflow: PrtSc → guarda em ~/Pictures/Screenshots/ → abre no Gradia para anotar

# Atalhos nativos GNOME
PrtSc                    # captura ecrã completo
Shift+PrtSc              # captura área selecionada
Alt+PrtSc                # captura janela ativa
Ctrl+PrtSc               # copia para clipboard em vez de salvar

# Gradia (annotation tool nativo GTK4/GNOME)
gradia                   # abre Gradia manualmente
# Ferramentas: Pen, Texto, Linha, Seta, Retângulo, Círculo, Marcador, Censor, Número
# Screenshots guardados em: ~/Pictures/Screenshots/

# OCR instalado: inglês (eng) + português (por)
# Para adicionar mais idiomas:
sudo pacman -S tesseract-data-deu   # alemão
sudo pacman -S tesseract-data-fra   # francês
```

---

*This file lives at `docs/CHEATSHEET.md` in the qemu-sa repo.*
