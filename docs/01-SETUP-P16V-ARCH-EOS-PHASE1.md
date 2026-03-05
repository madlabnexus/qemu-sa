# 01 — P16v Workstation Setup: Phase 1 — Base System

**Hardware:** Lenovo ThinkPad P16v Gen 2  
**CPU:** Intel Core Ultra (MTL) with Intel Arc integrated + NVIDIA RTX 3000 Ada discrete  
**RAM:** 64GB  
**Storage:** Dual NVMe — nvme1 (Linux) · nvme0 (Windows 11)  
**Display:** Thunderbolt 4 dock → Lenovo 45" ultrawide (DP-8)  
**OS:** EndeavourOS (Arch Linux) · GNOME · Wayland · systemd-boot  
**Completed:** 2026-03-04

---

## Phase 1 Scope

Foundation layer — everything that must work before any dev tooling is installed:

| Step | Item | Status |
|------|------|--------|
| 00 | btrfs subvolume layout verified | ✅ |
| 01 | Mirrors optimized (Arch + EndeavourOS) | ✅ |
| 02 | System fully updated | ✅ |
| 03 | Timeshift snapshots (BTRFS mode) | ✅ |
| 04 | Git + SSH key → GitHub | ✅ |
| 05 | dotfiles repo `archp16v` (private) | ✅ |
| 06 | NVIDIA drivers (open-dkms 590.48.01) | ✅ |
| 07 | PRIME render offload verified | ✅ |
| 08 | Power management (lid + suspend) | ✅ |
| 09 | Bluetooth | ✅ |
| 10 | GNOME Extensions | ✅ |
| 11 | Ghostty terminal | ✅ |
| 12 | Claude Desktop | ✅ |
| 13 | Claude Code CLI | ✅ |
| 14 | Windows 11 dual boot (systemd-boot) | ✅ |
| 15 | RTC/timezone dual boot fix | ✅ |

---

## 00 — btrfs Layout

```bash
sudo btrfs subvolume list /
```

**Result:**
```
ID 256  path @           ← root filesystem (wiped on reinstall)
ID 257  path @home       ← home directory  (SURVIVES reinstall) ✅
ID 258  path @cache      ← /var/cache
ID 259  path @log        ← /var/log
ID 260  path var/lib/portables
ID 261  path var/lib/machines
```

**Partition layout:**
```
nvme1n1p1  vfat   /efi          ← Linux EFI (systemd-boot)
nvme1n1p2  btrfs  /             ← subvol=@
nvme1n1p2  btrfs  /home         ← subvol=@home  ✅ survives reinstall
nvme1n1p2  btrfs  /var/cache    ← subvol=@cache
nvme1n1p2  btrfs  /var/log      ← subvol=@log
nvme1n1p3  swap   [SWAP]

nvme0n1p2  ntfs                 ← Windows 11 system
nvme0n1p3  vfat   EFI           ← Windows EFI
nvme0n1p4  ntfs                 ← Windows Recovery
nvme0n1p5  ntfs   Data          ← Windows Data
```

> **Reinstall rule:** The installer wipes only `@`. The `@home` subvolume is
> preserved. All configs, dotfiles, `~/Applications`, `~/.nvm`, and project
> files survive.

---

## 01 — Mirrors

### Arch Linux mirrors (fastest globally)

```bash
sudo reflector --age 12 --protocol https --sort rate --fastest 20 --save /etc/pacman.d/mirrorlist
```

Top mirrors selected (2026-03-05):
- mirrors.gigenet.com (US) — 0.43s
- fastly.mirror.pkgbuild.com
- br.mirrors.cicku.me

### EndeavourOS mirrors

```bash
eos-rankmirrors
```

Top mirror selected: `mirrors.gigenet.com/endeavouros` — 0.43s

---

## 02 — System Update

```bash
yay -Syu
```

Run before every install session. Also update npm:

```bash
npm install -g npm@latest
```

---

## 03 — Timeshift Snapshots

```bash
sudo pacman -S timeshift
```

**Configuration:**
- Type: **BTRFS** (not RSYNC)
- Location: nvme1n1p2
- Schedule: manual (create before risky operations)
- Includes: `@` and `@home`

**Create snapshot:**
```bash
sudo timeshift --create --comments "description" --tags D
```

**List snapshots:**
```bash
sudo timeshift --list
```

**Restore:**
```bash
sudo timeshift --restore
```

**Snapshots created in Phase 1:**

| Snapshot | Description |
|----------|-------------|
| `00-clean-install` | Fresh EndeavourOS, mirrors updated |
| `01-claude-desktop-ok` | Claude Desktop installed and working |
| `02-nvidia-ok` | NVIDIA 590.48.01 + PRIME working |
| `03-nvidia-power-ok` | Power management configured |
| `04-bluetooth-ok` | Bluetooth working |
| `05-gnome-extensions-ghostty-ok` | Extensions + Ghostty configured |
| `06-claude-code-ok` | Claude Code 2.1.69 authenticated |
| `07-dualboot-ok` | Windows 11 dual boot verified |

> **TTY note:** On this machine, GNOME runs on **TTY3**. Emergency TTY is
> `Ctrl+Alt+F2` or `F4`. Use `Ctrl+Alt+F3` to return to desktop.

---

## 04 — Git + SSH Key

```bash
git config --global user.name "MADLAB NEXUS"
git config --global user.email "madlabnexus@gmail.com"
ssh-keygen -t ed25519 -C "madlabnexus@gmail.com"
cat ~/.ssh/id_ed25519.pub
```

Add public key at: **github.com/settings/keys** → New SSH key → title: `archp16v`

**Test:**
```bash
ssh -T git@github.com
# Hi madlabnexus! You've successfully authenticated...
```

---

## 05 — Dotfiles Repo

Private repo: `github.com/madlabnexus/archp16v`

```bash
mkdir -p ~/dotfiles ~/Applications ~/Projects ~/.local/bin
cd ~/dotfiles
git init
git branch -M main
git remote add origin git@github.com:madlabnexus/archp16v.git
```

**Save package list:**
```bash
pacman -Qqe > ~/dotfiles/pkglist.txt
```

**Push after each install step:**
```bash
pacman -Qqe > ~/dotfiles/pkglist.txt
cd ~/dotfiles
git add pkglist.txt
git commit -m "step-description"
git push
```

**Restore after reinstall:**
```bash
sudo pacman -S --needed - < ~/dotfiles/pkglist.txt
```

**Directory structure (all in `~`, survives reinstall):**
```
~/Applications/       ← AppImages and manual installs
~/Projects/           ← all code projects
~/dotfiles/           ← git-tracked configs + pkglist
~/.local/bin/         ← user scripts
~/.local/share/applications/  ← .desktop launchers
~/.nvm/               ← Node.js versions
~/.config/            ← app configs
```

---

## 06 — NVIDIA Drivers

```bash
sudo pacman -S nvidia-inst
nvidia-inst
```

**Installed packages:**
- `nvidia-open-dkms` 590.48.01 (open source kernel module)
- `nvidia-utils` 590.48.01
- `nvidia-settings` 590.48.01
- `nvidia-hook` (auto-rebuilds on kernel updates)
- `dkms` (kernel module manager)

**Additional:**
```bash
sudo pacman -S nvidia-prime
```

**Verification:**
```bash
nvidia-smi
# NVIDIA RTX 3000 Ada · Driver 590.48.01 · CUDA 13.1

glxinfo | grep "OpenGL renderer"
# Mesa Intel Arc (MTL) ← display GPU, correct

prime-run glxinfo | grep "OpenGL renderer"
# NVIDIA RTX 3000 Ada Generation ← discrete GPU via PRIME ✅

echo $XDG_SESSION_TYPE
# wayland ✅
```

**How GPU switching works on P16v:**
- Intel Arc → manages display (battery efficient, always active)
- NVIDIA RTX 3000 Ada → activated on demand via `prime-run`
- For QEMU-SA dev builds: `prime-run make` to use NVIDIA compute

---

## 07 — Power Management

### Suspend policy (via GNOME Settings → Power → Power Saving)

- On Battery: suspend after 15 minutes ✅
- When Plugged In: never suspend ✅

### Lid close (via `/etc/systemd/logind.conf`)

```ini
HandleLidSwitchExternalPower=ignore   ← lid closed + AC = no sleep ✅
HandleLidSwitchDocked=ignore          ← lid closed + dock = no sleep ✅
# HandleLidSwitch=suspend             ← lid closed + battery = suspend (default) ✅
```

```bash
sudo systemctl restart systemd-logind
```

> After restarting logind, screen may go black briefly. Press any key or
> use `Ctrl+Alt+F3` to return to GNOME session.

---

## 08 — Bluetooth

```bash
sudo systemctl enable bluetooth
sudo systemctl start bluetooth
sudo pacman -S blueman
```

Bluetooth management: GNOME top-right menu → Bluetooth toggle.
Blueman applet available for advanced pairing.

---

## 09 — GNOME Extensions

**Install Extension Manager:**
```bash
sudo pacman -S gnome-browser-connector
yay -S extension-manager
```

**Installed extensions:**

| Extension | Purpose |
|-----------|---------|
| AppIndicator and KStatusNotifierItem | System tray icons |
| Caffeine | Prevent sleep on demand |
| Clipboard Indicator | Clipboard history |
| Compiz windows effect | Window animations |
| Dash to Dock | Dock at bottom, autohide, centered icons |
| Just Perfection | Fine-tune GNOME UI |
| Removable Drive Menu | USB/drive access in topbar |
| Tiling Assistant | Window snapping for ultrawide |
| Transparent Window Moving | Transparency when dragging windows |
| Vitals | CPU/GPU/RAM temps in topbar |

**Dash to Dock config:**
- Position: Bottom
- Intelligent autohide: ON
- Place icons to center: ON
- Icon size: 48px
- Dock size limit: 90%
- Opacity: Fixed 75%
- Show overview on startup: OFF
- Click action: Cycle through windows
- Scroll action: Cycle through windows

---

## 10 — Ghostty Terminal

```bash
sudo pacman -S ghostty
sudo pacman -S ttf-jetbrains-mono-nerd
```

**Config at `~/.config/ghostty/config`:**
```ini
theme = Flexoki Dark
background-opacity = 0.85
font-family = JetBrains Mono
font-size = 13
window-padding-x = 8
window-padding-y = 8
scrollback-limit = 0
window-width = 160
window-height = 40
```

> Theme chosen for long dev sessions: Flexoki Dark — warm earth tones,
> low eye strain for C/kernel development.

---

## 11 — Claude Desktop

```bash
yay -S claude-desktop-bin
```

Version: 1.1.4498-7  
Launch: app grid (Super) → "Claude" or `claude-desktop` in terminal.

---

## 12 — Claude Code CLI

**Node.js via nvm (lives in `~/.nvm`, survives reinstall):**
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts
```

Node version: v24.14.0 LTS

**Claude Code:**
```bash
npm install -g @anthropic-ai/claude-code
```

Version: 2.1.69  
Model: Opus 4.6 · Claude Max

**Usage:**
```bash
cd ~/Projects/qemu-sa   # always cd to project first
claude                  # opens in that project context
```

> Use **Opus 4.6** for QEMU-SA development (complex C/kernel work).
> Sonnet for quick tasks and documentation.

---

## 13 — Windows 11 Dual Boot

Windows 11 is on `nvme0n1`. EndeavourOS automatically detected it during
installation and added it to systemd-boot.

**systemd-boot config (`/efi/loader/loader.conf`):**
```ini
default  11e21772c8b04f93afb1bed47c7803c6*
timeout  5
console-mode auto
reboot-for-bitlocker 1
```

**EFI structure (`/efi/EFI/`):**
```
BOOT/        ← generic fallback bootloader
Linux/        ← EndeavourOS kernel
Microsoft/    ← Windows 11 bootloader (bootmgfw.efi) ✅
systemd/      ← systemd-boot
```

**Boot entries:**
- `11e21772...6.18.13-arch1-1.conf` → EndeavourOS (default)
- `11e21772...6.18.13-arch1-1-fallback.conf` → EndeavourOS fallback
- Windows 11 → auto-detected via `/efi/EFI/Microsoft/Boot/bootmgfw.efi`

**Kernel boot options:**
```
nvme_load=YES nowatchdog rw rootflags=subvol=/@
root=UUID=da0e0dba-141c-4549-8ba1-80b02f836a15
resume=UUID=dcab91ec-378c-419a-a048-814a8ae09edb
```

**After reinstall:** systemd-boot will auto-detect Windows again as long
as `/efi/EFI/Microsoft/` is present on nvme0n1p3.

---

## 15 — RTC / Timezone (Dual Boot Fix)

**Problem:** Windows reads the hardware clock (RTC) as local time. Linux
reads it as UTC by default. In dual boot, each OS shifts the clock when
booting, causing a 3-hour offset (Brasília = UTC-3).

**Solution:** Force Linux to also read RTC as local time.

```bash
sudo timedatectl set-local-rtc 1 --adjust-system-clock
sudo hwclock --systohc --localtime
```

**Verify:**
```bash
timedatectl
```

**Expected output:**
```
               Local time: qui 2026-03-05 12:33:16 -03
           Universal time: qui 2026-03-05 15:33:16 UTC
                 RTC time: qui 2026-03-05 12:33:16
                Time zone: America/Sao_Paulo (-03, -0300)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: yes
```

> **Warning expected:** `"The system is configured to read the RTC time in
> the local time zone"` — this warning is normal and harmless in a dual-boot
> setup. NTP keeps the clock accurate automatically.

> **Do NOT** run `timedatectl set-local-rtc 0` — that would revert the fix
> and break Windows clock again.

---

## Phase 1 — Installed Packages Summary

Packages added beyond EndeavourOS base:

```
timeshift
nvidia-inst → nvidia-open-dkms nvidia-utils nvidia-settings nvidia-hook dkms
nvidia-prime
blueman
gnome-browser-connector
extension-manager (AUR)
ghostty
ttf-jetbrains-mono-nerd
claude-desktop-bin (AUR)
# nvm → ~/.nvm (not pacman)
# claude-code → npm global (not pacman)
```

Full list maintained at: `github.com/madlabnexus/archp16v` → `pkglist.txt`

---

*Phase 2 continues in `01-SETUP-P16V-ARCH-EOS-PHASE2.md`*
