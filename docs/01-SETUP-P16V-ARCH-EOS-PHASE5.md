# 01 — P16v Workstation Setup: Phase 5 — Gaming (Steam, GOG, Proton)

**Hardware:** Lenovo ThinkPad P16v Gen 2  
**OS:** EndeavourOS (Arch Linux) · GNOME · Wayland · systemd-boot  
**GPU:** NVIDIA RTX 3000 Ada (proprietary driver) + Intel Arc iGPU  
**Completed:** 2026-04-15

---

## Phase 5 Scope

Gaming platform setup — Steam, GOG (Heroic), Proton/Proton-GE, MangoHud, and performance tools:

| Step | Item | Status |
|------|------|--------|
| 00 | Fix GDM auto-login (restore password prompt) | ✅ |
| 01 | Steam (multilib) + lib32-nvidia-utils (Vulkan 32-bit) | ✅ |
| 02 | MangoHud 0.8.2 (64-bit + 32-bit) — FPS/performance overlay | ✅ |
| 03 | ProtonUp-Qt 2.15.0 — Proton version manager GUI | ✅ |
| 04 | Heroic Games Launcher 2.20.1 — GOG/Epic/Amazon launcher | ✅ |
| 05 | Steam configuration — Steam Play + Proton Experimental | ✅ |
| 06 | GE-Proton installed via ProtonUp-Qt | ✅ |
| 07 | Heroic — GOG account login | ✅ |

---

## 00 — Fix GDM Auto-Login

O auto-login havia sido configurado acidentalmente em uma sessão anterior. Para restaurar a tela de login com senha:

```bash
sudo tee /etc/gdm/custom.conf << 'EOF'
# GDM configuration storage

[daemon]
# WaylandEnable=false

[security]

[xdmcp]

[chooser]

[debug]
# Enable=true
EOF
```

Reboot para confirmar que pede senha:

```bash
sudo reboot
```

**Explicação:** O GDM lê `/etc/gdm/custom.conf` no boot. Se `AutomaticLogin=username` estiver presente (mesmo com `AutomaticLoginEnable=false`), algumas versões do GDM fazem auto-login de qualquer forma. A solução é remover completamente as linhas `AutomaticLogin*`.

---

## 01 — Steam

Steam vem do repositório **multilib** (já habilitado no pacman.conf):

```bash
sudo pacman -S steam
```

Quando perguntar sobre `lib32-vulkan-driver`, selecionar **1) lib32-nvidia-utils** — é o driver Vulkan 32-bit da NVIDIA, necessário para Proton rodar jogos DirectX.

**76 pacotes instalados** incluindo toda a stack lib32 (glibc, mesa, nvidia-utils 595.58.03, pipewire, Vulkan ICD loader, NSS, etc.).

---

## 02 — MangoHud (FPS Overlay)

MangoHud mostra FPS, temperaturas, uso de CPU/GPU em tempo real dentro do jogo — equivalente ao MSI Afterburner no Windows:

```bash
sudo pacman -S mangohud lib32-mangohud
```

| Pacote | Versão | Uso |
|--------|--------|-----|
| `mangohud` | 0.8.2 | Overlay 64-bit |
| `lib32-mangohud` | 0.8.2 | Overlay 32-bit (necessário para jogos 32-bit via Proton) |

**Uso no Steam:** Em um jogo, botão direito → Properties → Launch Options:
```
mangohud %command%
```

Com GameMode (já instalado na Phase 3) + MangoHud:
```
gamemoderun mangohud %command%
```

**Para laptop NVIDIA Optimus** (forçar GPU dedicada):
```
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia __VK_LAYER_NV_optimus=NVIDIA_only gamemoderun mangohud %command%
```

---

## 03 — ProtonUp-Qt (Gerenciador de Proton)

ProtonUp-Qt é uma GUI para instalar e gerenciar versões do GE-Proton, DXVK e Wine-GE:

```bash
yay -S protonup-qt
```

**Nota:** Os warnings de teste (`python-steam` 10 failed) são incompatibilidades com Python 3.14 nos testes, NÃO na funcionalidade. O ProtonUp-Qt 2.15.0 funciona normalmente.

---

## 04 — Heroic Games Launcher (GOG / Epic / Amazon)

Heroic é o launcher open-source para GOG, Epic Games Store e Amazon Games no Linux:

```bash
yay -S heroic-games-launcher-bin
```

**Importante:** Usar o pacote `-bin` (binário pré-compilado). O pacote `heroic-games-launcher` (compilação do source) tem problemas de build conhecidos no AUR com fontawesome/npm.

Heroic 2.20.1 inclui:
- `legendary` — backend para Epic Games
- `gogdl` — backend para GOG
- `nile` — backend para Amazon Games
- `comet` — online features para GOG
- Wine/Proton manager integrado

---

## 05 — Configuração do Steam

Abrir Steam → fazer login → configurar Steam Play:

1. **Steam → Settings → Compatibility**
2. Ativar **"Enable Steam Play for all other titles"**
3. Selecionar **Proton Experimental** como versão default
4. Reiniciar Steam quando solicitado

Isso permite que todos os jogos Windows da biblioteca rodem via Proton automaticamente.

---

## 06 — Instalar GE-Proton via ProtonUp-Qt

GE-Proton (GloriousEggroll) é uma versão do Proton com patches extras de compatibilidade, codecs de mídia e fixes que ainda não estão no Proton oficial:

1. Abrir **ProtonUp-Qt** (Activities → "ProtonUp-Qt")
2. Em **"Install for"**, selecionar **Steam**
3. Clicar **"Add version"**
4. Selecionar **GE-Proton** e a versão mais recente
5. Clicar **Install**
6. Aguardar download e extração

Depois de instalado, o GE-Proton aparece como opção em Steam → jogo → Properties → Compatibility → Force the use of a specific Steam Play compatibility tool.

**Quando usar GE-Proton vs Proton Experimental:**
- Default: Proton Experimental (oficial da Valve, mais estável)
- Se um jogo não funciona: tentar GE-Proton (mais patches, melhor compatibilidade com vídeos/cutscenes)
- Verificar **ProtonDB** (protondb.com) para recomendações por jogo

---

## 07 — Heroic: Login GOG

1. Abrir **Heroic** (Activities → "Heroic")
2. Na tela inicial, clicar **"Log in"** na seção GOG
3. Autenticar com as credenciais GOG no browser
4. A biblioteca GOG aparece automaticamente no Heroic

**Para instalar jogos GOG:**
- Clicar em um jogo → Install
- Heroic usa Wine/Proton automaticamente para jogos Windows
- O Wine Manager do Heroic permite instalar versões de Wine-GE/Proton-GE independentes do Steam

---

## Referência: Launch Options por Cenário

| Cenário | Launch Options |
|---------|---------------|
| Básico | `%command%` |
| Com FPS overlay | `mangohud %command%` |
| Com performance boost | `gamemoderun %command%` |
| FPS + performance | `gamemoderun mangohud %command%` |
| Forçar NVIDIA dGPU | `__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia __VK_LAYER_NV_optimus=NVIDIA_only gamemoderun mangohud %command%` |

---

## Phase 5 — Snapshots

| Snapshot | Description |
|----------|-------------|
| `29-pre-phase5-gaming` | Pre-Phase 5 baseline |
| `30-phase5-gaming-complete` | Steam, Heroic, MangoHud, ProtonUp-Qt, GE-Proton |

---

## Phase 5 — Packages Installed

### Pacman (multilib)

```
steam
mangohud
lib32-mangohud
lib32-nvidia-utils (+ ~73 lib32 dependencies)
```

### AUR (yay)

```
protonup-qt
heroic-games-launcher-bin
```

### Already Installed (previous phases)

```
gamemode (Phase 3)
wine-staging wine-mono winetricks zenity (Phase 2)
```

---

*Phase 5 completa. O P16v agora é uma estação de gaming completa com Steam + Proton, GOG via Heroic, e ferramentas de performance.*
