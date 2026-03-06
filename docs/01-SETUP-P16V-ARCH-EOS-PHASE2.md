# 01 — P16v Workstation Setup: Phase 2 — KVM & QEMU-SA Dev Environment

**Hardware:** Lenovo ThinkPad P16v Gen 2  
**OS:** EndeavourOS (Arch Linux) · GNOME · Wayland · systemd-boot  
**Kernel:** 6.19.6-arch1-1  
**Completed:** 2026-03-05

---

## Phase 2 Scope

KVM virtualization, IOMMU/VFIO, complete QEMU-SA development toolchain, and additional dev tools:

| Step | Item | Status |
|------|------|--------|
| 00 | IOMMU enabled + kernel parameters optimized | ✅ |
| 01 | KVM / QEMU 10.2.0 + virt-manager installed | ✅ |
| 02 | libvirtd service enabled | ✅ |
| 03 | User added to kvm/libvirt/qemu groups | ✅ |
| 04 | virt-host-validate passed | ✅ |
| 05 | Build toolchain (gcc, meson, ninja, cmake, clang) | ✅ |
| 06 | QEMU-SA audio dependencies (fluidsynth, ALSA, PulseAudio, JACK) | ✅ |
| 07 | Munt / libmt32emu 2.7.2 compiled from AUR | ✅ |
| 08 | libcue 2.3.0 installed | ✅ |
| 09 | VS Code 1.110.0 + C/C++ extensions | ✅ |
| 10 | Android Studio 2025.3.1.8 (Panda) | ✅ |
| 11 | Network tools (iperf3, nmap, traceroute, etc.) | ✅ |
| 12 | 86Box v6.0 build 8509 (experimental) + ROMs | ✅ |
| 13 | Wine Staging 11.3 + winetricks | ✅ |
| 14 | Media apps (OBS, Audacity, VLC, Handbrake, Kdenlive, GIMP, Spotify) | ✅ |
| 15 | PDF tools (Master PDF Editor 5.9.96, Okular) | ✅ |
| 16 | RetroArch 1.22.2 + cores NES/SNES/Mega Drive | ✅ |
| 17 | Wallpapers ultrawide 5120x1440 | ✅ |
| 18 | Gradia 1.11.3 — screenshot annotation nativo GNOME Wayland | ✅ |

---

## 00 — IOMMU + Kernel Parameters

### Why

IOMMU (Intel VT-d) is required for:
- VFIO GPU passthrough (WinXP machine profile)
- Secure device isolation between VMs
- Better DMA performance for KVM guests

### Boot entry configuration

File: `/efi/loader/entries/11e21772c8b04f93afb1bed47c7803c6-6.19.6-arch1-1.conf`

```
options nvme_load=YES nowatchdog rw rootflags=subvol=/@ \
  root=UUID=da0e0dba-141c-4549-8ba1-80b02f836a15 \
  resume=UUID=dcab91ec-378c-419a-a048-814a8ae09edb \
  systemd.machine_id=11e21772c8b04f93afb1bed47c7803c6 \
  intel_iommu=on iommu=pt mitigations=off kvm.ignore_msrs=1 \
  transparent_hugepage=never
```

### Parameter explanation

| Parameter | Purpose |
|-----------|---------|
| `intel_iommu=on` | Enables Intel VT-d IOMMU — required for VFIO passthrough |
| `iommu=pt` | Passthrough mode — better native device performance |
| `mitigations=off` | Disables Spectre/Meltdown mitigations — real performance gain in VMs |
| `kvm.ignore_msrs=1` | Prevents Windows VM crashes on unknown MSR accesses |
| `transparent_hugepage=never` | Better memory latency for VMs — disables THP |

### Verification

```bash
sudo dmesg | grep -i iommu | head -10
```

Expected output:
```
[    0.000000] Command line: ... intel_iommu=on iommu=pt ...
[    0.049343] DMAR: IOMMU enabled
[    0.804932] iommu: Default domain type: Passthrough (set via kernel command line)
[    0.870141] pci 0000:00:02.0: Adding to iommu group 0
```

> **Note:** `mitigations=off` is a conscious trade-off. This machine is a
> dedicated dev workstation, not a multi-tenant server. The performance
> gain for QEMU-SA development and VM execution justifies the risk.

---

## 01 — KVM / QEMU 10.2.0 Installation

```bash
sudo pacman -S qemu-full virt-manager libvirt dnsmasq iptables-nft edk2-ovmf
```

**Installed packages:**
- `qemu-full` 10.2.0 — complete QEMU with all system emulators
- `virt-manager` 5.1.0 — GUI VM manager
- `libvirt` 12.1.0 — virtualization API daemon
- `dnsmasq` — NAT networking for VMs
- `iptables-nft` — firewall rules for VM networking
- `edk2-ovmf` — UEFI firmware for VMs (required for Win98/XP passthrough)

> **Note:** `qemu-full` from pacman is for running standard VMs and provides
> build dependencies. The QEMU-SA fork will be compiled separately from
> source in `~/Projects/qemu-sa` and runs independently.

---

## 02 — libvirtd Service

```bash
sudo systemctl enable --now libvirtd
```

**Verify:**
```bash
systemctl status libvirtd
```

---

## 03 — User Groups

```bash
sudo usermod -aG libvirt,kvm,qemu madlabn
newgrp libvirt
```

> Group changes require logout/login or `newgrp` to take effect in current session.

---

## 04 — KVM Validation

```bash
virt-host-validate
```

**Expected results:**

```
QEMU: Checking for hardware virtualization        : PASS (VMX)
QEMU: Checking if device '/dev/kvm' exists        : PASS
QEMU: Checking if device '/dev/kvm' is accessible : PASS
QEMU: Checking if device '/dev/vhost-net' exists  : PASS
QEMU: Checking for device assignment IOMMU support: PASS (DMAR)
QEMU: Checking if IOMMU is enabled by kernel      : PASS
```

**Expected warnings (harmless):**
- `cgroup 'devices' controller` — not used by modern QEMU
- `secure guest support (SEV/TDX)` — not available on this CPU, irrelevant for retro gaming

---

## 05 — Build Toolchain

All tools required to compile QEMU-SA from source:

```bash
sudo pacman -S base-devel git ninja python-pip pkg-config glib2 pixman \
  libslirp libcap-ng libseccomp libepoxy virglrenderer spice-protocol \
  libpulse alsa-lib libpipewire meson cmake python-sphinx dtc \
  attr curl llvm clang
```

**Note on Arch package naming:**
- No `libcurl` — Arch includes it in `curl`
- No `libattr` — Arch includes it in `attr`
- No `libjack` — use `pipewire-jack` (provides JACK via PipeWire)
- No `ceph-libs` as standalone — skip, not needed for QEMU-SA
- When asked about jack conflict: choose **N** (keep pipewire-jack)

| Tool | Version | Purpose |
|------|---------|---------|
| gcc | 15.2.1 | C/C++ compiler |
| clang/llvm | 21.1.8 | Alternative compiler, static analysis |
| meson | 1.10.1 | QEMU build system |
| ninja | 1.13.2 | Fast build executor |
| cmake | 4.2.3 | Dependency build system |
| python-sphinx | 8.2.3 | Documentation generator |
| dtc | 1.7.2 | Device tree compiler |

---

## 06 — Audio Dependencies

Libraries required for QEMU-SA sound features:

```bash
sudo pacman -S fluidsynth libpulse pipewire-jack alsa-lib \
  libsamplerate libsndfile opus
```

| Library | Version | QEMU-SA Use |
|---------|---------|-------------|
| fluidsynth | 2.5.3 | General MIDI backend (GM SoundFont) |
| alsa-lib | 1.2.15.3 | ALSA audio output |
| libpulse | 17.0 | PulseAudio output |
| pipewire-jack | 1.6.0 | JACK audio (via PipeWire) |
| libsamplerate | 0.2.2 | OPL3 49716Hz → host rate resampling |
| libsndfile | 1.2.2 | Audio file I/O (WAV/FLAC for CD audio) |
| opus | 1.6.1 | Compressed audio support |

---

## 07 — Munt / libmt32emu (MT-32 Emulation)

Roland MT-32 emulation library — essential for Sierra/LucasArts era MIDI.

```bash
yay -S munt
```

**During install:**
- Remove make dependencies: **N**
- Qt6 multimedia backend: **1** (ffmpeg)
- Jack conflict: **N** (keep pipewire-jack)

**Installed:**
- `libmt32emu.so.2.7.2` → `/usr/lib/`
- Headers → `/usr/include/mt32emu/`
- `mt32emu-qt` — standalone MT-32 player GUI
- `mt32emu-smf2wav` — MIDI to WAV converter

**Compiled with:**
- ALSA ✅
- PulseAudio ✅
- JACK (via PipeWire) ✅
- PortAudio ✅

> **ROMs required:** The MT-32 ROMs are proprietary Roland property and
> are NOT included. Users must provide their own:
> - `MT32_CONTROL.ROM` + `MT32_PCM.ROM` (MT-32)
> - `CM32L_CONTROL.ROM` + `CM32L_PCM.ROM` (CM-32L)

**Verify:**
```bash
ls /usr/lib/libmt32emu*
# /usr/lib/libmt32emu.so  /usr/lib/libmt32emu.so.2  /usr/lib/libmt32emu.so.2.7.2
```

---

## 08 — libcue (CUE/BIN Parser)

Required for CD-ROM audio track support in QEMU-SA:

```bash
sudo pacman -S libcue
```

Version: 2.3.0

---

## QEMU-SA Build Dependencies — Summary

Everything needed to compile QEMU-SA from source:

```
# Core build
base-devel git meson ninja cmake python-sphinx dtc
clang llvm attr curl

# QEMU core
glib2 pixman libslirp libcap-ng libseccomp libepoxy
virglrenderer spice-protocol usbredir libusb fuse3
bzip2 lzo snappy numactl libiscsi libnfs
sdl2 sdl2_image gtk3 vte3 spice spice-gtk
libpng libjpeg-turbo libdrm mesa libepoxy

# QEMU-SA audio (beyond stock QEMU)
fluidsynth alsa-lib libpulse pipewire-jack
libsamplerate libsndfile opus
munt  ← AUR (libmt32emu)
libcue

# Virtualization
qemu-full libvirt dnsmasq iptables-nft edk2-ovmf
```

Full package list maintained at: `github.com/madlabnexus/archp16v` → `pkglist.txt`

---

## 09 — VS Code 1.110.0

```bash
yay -S visual-studio-code-bin
```

**Extensions installed:**

| Extension | Version | Purpose |
|-----------|---------|---------|
| ms-vscode.cpptools | 1.30.5 | C/C++ IntelliSense, debugging |
| ms-vscode.cpptools-extension-pack | 1.5.1 | C/C++ full pack |
| ms-vscode.cmake-tools | 1.22.28 | CMake integration |
| twxs.cmake | 0.0.17 | CMake syntax highlighting |
| mhutchie.git-graph | 1.30.0 | Git history visualization |
| eamodio.gitlens | 17.11.0 | Git blame + insights |
| usernamehw.errorlens | 3.28.0 | Inline error display |
| streetsidesoftware.code-spell-checker | 4.5.6 | Spell check |

**Workspace config** (`~/Projects/qemu-sa/.vscode/settings.json`):
```json
{
    "editor.fontSize": 14,
    "editor.fontFamily": "JetBrains Mono, monospace",
    "editor.tabSize": 4,
    "editor.rulers": [80, 120],
    "C_Cpp.default.compilerPath": "/usr/bin/gcc",
    "C_Cpp.default.cStandard": "c11",
    "C_Cpp.default.cppStandard": "c++17",
    "C_Cpp.default.intelliSenseMode": "linux-gcc-x64",
    "cmake.buildDirectory": "${workspaceFolder}/build",
    "cmake.generator": "Ninja"
}
```

---

## 10 — Android Studio 2025.3.1.8 (Panda)

```bash
yay -S android-studio
yay -S ncurses5-compat-libs   # native debugger support
```

Version: 2025.3.1.8 (Panda patch 1) — 3.2GB installed.

---

## 11 — Network Tools

```bash
sudo pacman -S iperf3 nmap traceroute whois bind net-tools inetutils
```

| Tool | Purpose |
|------|---------|
| iperf3 3.20 | Bandwidth testing (TD350 server benchmarks) |
| nmap 7.98 | Network scanning |
| traceroute | Route tracing |
| whois | Domain lookups |
| bind (dig/nslookup) | DNS queries |
| net-tools (ifconfig/netstat) | Legacy network tools |
| inetutils (ping/hostname) | Basic network utilities |

---

## 12 — 86Box v6.0 build 8509 (Experimental)

86Box is the cycle-accurate DOS/Win9x emulator used alongside QEMU-SA for games that need exact CPU timing (the ~10-15% that QEMU-SA's TCG cannot handle).

### Download

Experimental builds from 86Box Jenkins — always use **Old Recompiler** (New Recompiler is beta):

```bash
mkdir -p ~/Applications/86Box
cd ~/Applications/86Box
wget "https://ci.86box.net/job/86Box/8509/artifact/Old%20Recompiler%20(recommended)/Linux%20-%20x64%20(64-bit)/86Box-Linux-x86_64-b8509.AppImage"
chmod +x 86Box-Linux-x86_64-b8509.AppImage
```

> To get the latest build number: go to `https://86box.net/builds`, select Linux x64, Old Recompiler.

### ROM Set

For experimental builds, clone the ROM repo directly (tagged releases may lag behind):

```bash
cd ~/Applications/86Box
git clone --depth=1 https://github.com/86Box/roms.git roms
```

To update ROMs later: `cd ~/Applications/86Box/roms && git pull`

### Desktop Entry

```bash
cat > ~/.local/share/applications/86box.desktop << 'EOF'
[Desktop Entry]
Name=86Box
Comment=Retro PC Emulator (build 8509)
Exec=/home/madlabn/Applications/86Box/86Box-Linux-x86_64-b8509.AppImage
Icon=86box
Terminal=false
Type=Application
Categories=Emulator;Game;
EOF
```

### Notes

- `gamemodeauto: dlopen failed` warning — harmless, install `gamemode` to suppress
- `Warning: Ignoring XDG_SESSION_TYPE=wayland` — normal, uses XWayland via Qt
- VMs stored at: `~/.local/share/86Box/Virtual Machines/`
- Config at: `~/.config/86Box/`

---

## 13 — Wine Staging 11.3

```bash
sudo pacman -S wine-staging wine-mono winetricks zenity
sudo systemctl restart systemd-binfmt
```

- `wine-staging` — patches extras de compatibilidade e performance vs wine stable
- `wine-mono` — runtime .NET para aplicações Windows
- `winetricks` — instala dependências Windows facilmente (vcredist, directx, etc.)
- `zenity` — GUI para winetricks no GNOME
- `systemd-binfmt` restart — habilita execução direta de `.exe` no terminal

**Verificação:**
```bash
wine --version
# wine-11.3 (Staging)
```

---

## 14 — Media Apps

```bash
sudo pacman -S obs-studio audacity vlc handbrake kdenlive gimp
yay -S spotify
```

| App | Versão | Uso |
|-----|--------|-----|
| OBS Studio | 32.0.4 | Streaming / gravação de ecrã |
| Audacity | 3.7.7 | Edição de áudio |
| VLC | 3.0.21 | Player de vídeo/áudio universal |
| Handbrake | 1.10.2 | Conversão e compressão de vídeo |
| Kdenlive | 25.12.3 | Edição de vídeo não-linear |
| GIMP | 3.0.8 | Edição de imagens (alternativa ao Photoshop) |
| Spotify | 1.2.82.428 | Streaming de música |

---

## 15 — PDF Tools

```bash
yay -S masterpdfeditor okular
```

| App | Versão | Uso |
|-----|--------|-----|
| Master PDF Editor | 5.9.96 | Edição completa de PDF (anotações, formulários, assinatura) |
| Okular | 25.12.3 | Visualizador universal (PDF, EPUB, MOBI, CBZ) |

> **Nota:** Adobe Acrobat Reader foi descontinuado para Linux em 2013. Master PDF Editor + Okular cobrem todos os casos de uso.

---

## 16 — RetroArch + Cores

```bash
sudo pacman -S retroarch libretro-nestopia libretro-snes9x libretro-genesis-plus-gx
sudo pacman -S retroarch-assets-ozone retroarch-assets-xmb
```

| Core | Sistema | Versão |
|------|---------|--------|
| Nestopia | NES / Famicom | 1.69 |
| Snes9x | SNES / Super Famicom | 2651 |
| Genesis Plus GX | Mega Drive / Game Gear / Master System | 2306 |

**ROMs:** colocar em `~/ROMs/<sistema>/` e configurar em RetroArch → Settings → Directory → File Browser.

---

## 17 — Wallpapers Ultrawide

**Monitores:**
- DP-8 — Lenovo 45" ultrawide `5120x1440` (principal)
- DP-9 — monitor secundário `3840x2160` (4K)

**Fontes recomendadas para 5120x1440:**
- [wallhaven.cc](https://wallhaven.cc/search?resolutions=5120x1440) — melhor acervo, filtra por resolução exata
- [wallpaperflare.com](https://www.wallpaperflare.com) — boa seleção ultrawide
- [ultrawidewallpapers.net](https://ultrawidewallpapers.net) — dedicado a ultrawide

**Definir via terminal:**
```bash
gsettings set org.gnome.desktop.background picture-uri "file:///home/madlabn/Pictures/Wallpapers/ultrawide/wallpaper.jpg"
gsettings set org.gnome.desktop.background picture-uri-dark "file:///home/madlabn/Pictures/Wallpapers/ultrawide/wallpaper.jpg"
```

## 18 — Gradia (Screenshot + Anotação)

**Por que Gradia e não Flameshot/Spectacle/ksnip?**

No GNOME 49 + Wayland, o GNOME removeu o acesso à API privada de screenshots (`gnome-screenshot` quebrado, Flameshot não captura, Spectacle é KDE). Gradia é a única solução nativa GTK4/libadwaita que integra com o portal xdg-desktop-portal do GNOME.

```bash
yay -S gradia
# Durante instalação: escolher tesseract-data-eng (30) para OCR em inglês
# Para adicionar português depois: sudo pacman -S tesseract-data-por
```

**Workflow:**
1. **PrtSc** — captura nativa do GNOME (guarda em `~/Pictures/Screenshots/`)
2. **Gradia abre automaticamente** com o screenshot pronto para anotar
3. Ferramentas: Pen, Texto, Linha, Seta, Retângulo, Círculo, Marcador, Censor, Número
4. Backgrounds: Sólido, Gradiente, Imagem
5. Exportar: PNG, JPG, WEBP

**OCR instalado:** tesseract-data-eng (inglês) + tesseract-data-por (português)

**Para configurar Gradia como app padrão para screenshots:**
Gradia → Preferences → definir pasta de screenshots para `~/Pictures/Screenshots`

---

## Phase 2 — Snapshots

| Snapshot | Description |
|----------|-------------|
| `10-kvm-qemu-ok` | KVM + QEMU 10.2.0 + virt-manager working |
| `11-qemu-sa-deps-ok` | All QEMU-SA build dependencies installed |
| `12-vscode-ok` | VS Code 1.110.0 + C/C++ extensions |
| `13-android-studio-network-tools-ok` | Android Studio + network tools |
| `14-86box-b8509-ok` | 86Box v6.0 build 8509 + ROMs |
| `15-media-apps-retroarch-wine-ok` | Wine, media apps, PDF tools, RetroArch |
| `16-gradia-phase2-complete` | Gradia screenshot tool — Phase 2 completa |

---

## Git — Resolver Conflitos de Sincronismo

Quando o push é rejeitado porque o remote tem commits que não tens localmente:

```bash
# Erro típico:
# ! [rejected] main -> main (fetch first)

# Solução: pull com rebase + push
git pull --rebase && git push
```

Se o rebase tiver conflitos:

```bash
# Ver ficheiros em conflito
git status

# Opção A: aceitar versão remota (descarta as tuas mudanças no ficheiro)
git checkout --theirs path/to/file.md
git add path/to/file.md
git rebase --continue

# Opção B: aceitar a tua versão (descarta as mudanças remotas)
git checkout --ours path/to/file.md
git add path/to/file.md
git rebase --continue

# Opção C: abortar e voltar ao estado anterior
git rebase --abort

# Após resolver todos os conflitos
git push
```

> **Nota:** `--theirs` = versão do remote. `--ours` = versão local.

---

## Next: Phase 3

- Sistema de otimizações (zram, sysctl, I/O scheduler)
- Clone QEMU upstream source
- Apply qemu-3dfx patches
- Primeiro build QEMU-SA

---

*Phase 3 continues in `01-SETUP-P16V-ARCH-EOS-PHASE3.md`*
