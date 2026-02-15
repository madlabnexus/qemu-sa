# 05 — Sound Design

## Sound Capabilities by Machine

| Feature | DOS/Win3.11 (TCG) | Win98 (KVM) | WinXP (KVM+VFIO) |
|---------|-------------------|-------------|-------------------|
| FM Synthesis (OPL3) | Nuked-OPL3 (ISA SB16) | N/A (AC97) | N/A (AC97/HDA) |
| PCM/WAV | SB16 DMA | AC97 native | AC97/HDA native |
| MIDI (MT-32) | MPU-401 → Munt | N/A | N/A |
| MIDI (General) | MPU-401 → FluidSynth | N/A | N/A |
| AWE32 Wavetable | Phase 3 (EMU8000) | N/A | N/A |
| CD Audio | CUE/BIN playback | CUE/BIN playback | CUE/BIN playback |
| DOS in Win9x | — | VDMSound / SB compat | VDMSound |
| GUS (Gravis) | Stock QEMU GUS | N/A | N/A |

## Component 1: Nuked-OPL3 (FM Synthesis)

### What It Is

Nuked-OPL3 by nukeykt is a transistor-level accurate emulation of the Yamaha YMF262 (OPL3) chip. It was reverse-engineered by decapping the actual chip and reading the die. This is the same library used by DOSBox-Staging and 86Box.

### Why QEMU Needs It

Stock QEMU only has an OPL2 (YM3812) implementation via the `fmopl.c` file from MAME. OPL2 provides 9 channels of 2-operator FM synthesis. OPL3 doubles that to 18 channels with 4-operator modes, stereo output, and new waveforms. Games from 1992+ expect OPL3.

### Integration Plan

The Nuked-OPL3 library is two files: `opl3.c` and `opl3.h`. We vendor them into `third-party/nuked-opl3/`.

New file `hw/audio/nuked-opl3.c`:
- Registers ISA I/O ports 388h-38Bh (AdLib / OPL3)
- On port write: calls `OPL3_WriteRegBuffered()`
- On timer tick: calls `OPL3_GenerateStream()` at 49716 Hz native rate
- Feeds stereo PCM into QEMU's audio mixer
- Replaces the old `fmopl.c` path when `--opl3=nuked` is set

### OPL3 Register Map

```
Port 388h: AdLib address register (bank 0)
Port 389h: AdLib data register (bank 0)
Port 38Ah: OPL3 address register (bank 1) — OPL3 only
Port 38Bh: OPL3 data register (bank 1) — OPL3 only
```

Key registers:
- 0x01: Waveform select enable
- 0x04: Timer/IRQ control
- 0x05: OPL3 mode enable (bit 0 = 1 for OPL3 mode)
- 0x08: CSW/Note-Sel
- 0x20-0x35: Tremolo/Vibrato/Sustain/KSR/Freq multiplier
- 0xA0-0xB8: Frequency/key-on/octave
- 0xC0-0xC8: Feedback/connection/output
- 0xE0-0xF5: Waveform select

### Sample Rate Considerations

OPL3 native rate is 49716 Hz. QEMU's audio subsystem typically runs at 44100 or 48000 Hz. Options:

1. **Generate at 49716, resample to host rate** — most accurate, slight CPU cost
2. **Generate at 48000 directly** — Nuked-OPL3 supports arbitrary rates, minimal pitch error

Recommendation: Option 1 for accuracy. The resampling cost is trivial on modern CPUs.

### Test Games

| Game | What to Listen For |
|------|-------------------|
| DOOM (1993) | OPL3 music — Bobby Prince's FM compositions |
| Dune II (1992) | OPL3 music with stereo separation |
| Wolfenstein 3D (1992) | AdLib music (OPL2 mode) |
| Duke Nukem 3D (1996) | OPL3 music and sound effects |
| Tyrian (1995) | Complex OPL3 with all waveforms |

## Component 2: MPU-401 MIDI Interface

### What It Is

The MPU-401 (MIDI Processing Unit) was Roland's MIDI interface standard for PCs. Nearly every Sound Blaster card from the SB16 onward included MPU-401 UART mode at I/O port 330h.

### Why QEMU Needs It

QEMU's SB16 device has no MIDI output. Games that use MIDI (Sierra, LucasArts, Origin, MicroProse) either get no music or fall back to OPL3 FM. For games designed around the MT-32 or General MIDI, this is a severe quality loss.

### Integration Plan

New file `hw/audio/mpu401.c`:
- ISA device at port 330h (data) and 331h (status/command)
- Implements MPU-401 UART mode only (not Intelligent mode — no game needs it)
- Receives MIDI bytes from guest, assembles into MIDI messages
- Routes complete messages to the selected MIDI backend
- Supports SysEx passthrough (needed for MT-32 custom patches)

### MPU-401 UART Protocol

```
Port 330h (read):  MIDI data byte from device (not used for TX-only)
Port 330h (write): MIDI data byte to device
Port 331h (read):  Status register
                   Bit 6: DSR (data set ready, 0 = ready)
                   Bit 7: DRR (data receive ready, 0 = data available)
Port 331h (write): Command register
                   0x3F: Enter UART mode
                   0xFF: Reset (returns 0xFE ACK)
```

### MIDI Message Assembly

MIDI bytes arrive one at a time. The MPU-401 device must assemble them:

```
Status byte (0x80-0xEF): Note on/off, CC, program change, etc.
  - 2 or 3 bytes depending on message type
SysEx (0xF0 ... 0xF7): Variable length, terminated by 0xF7
  - Must buffer entire message before sending to backend
```

## Component 3: MIDI Backends

### Backend Interface

```c
typedef struct MIDIBackend {
    void (*init)(const char *config);
    void (*send_short)(uint8_t status, uint8_t data1, uint8_t data2);
    void (*send_sysex)(const uint8_t *data, size_t len);
    void (*generate)(int16_t *buf, int frames);  /* render audio */
    void (*close)(void);
} MIDIBackend;
```

### Backend: Munt (MT-32 / CM-32L)

**Library:** libmt32emu (from munt project)

The Roland MT-32 was the gold standard for PC game music from 1987-1993. Games like Monkey Island, King's Quest, Space Quest, Ultima, and Wing Commander were composed specifically for the MT-32's unique LA synthesis engine with custom instrument patches.

Configuration:
```
-midi mt32,romdir=/path/to/mt32-roms
-midi cm32l,romdir=/path/to/cm32l-roms
```

Required ROM files:
- MT-32: `MT32_CONTROL.ROM` + `MT32_PCM.ROM`
- CM-32L: `CM32L_CONTROL.ROM` + `CM32L_PCM.ROM`

**Note:** ROM files are copyrighted by Roland. Users must provide their own (dumped from hardware). QEMU-SA does NOT bundle ROMs.

### Backend: FluidSynth (General MIDI)

**Library:** libfluidsynth

For games that target General MIDI (GM) standard — most games from 1993 onward. FluidSynth loads a SoundFont file (.sf2) and renders MIDI to audio.

Configuration:
```
-midi gm,soundfont=/path/to/FluidR3_GM.sf2
```

Recommended SoundFonts:
- **FluidR3_GM.sf2** — good all-around, 142MB
- **GeneralUser_GS.sf2** — excellent quality, 30MB
- **Arachno SoundFont** — great for gaming, 150MB
- **SC-55 SoundFont** — captures Roland SC-55 sound (many 90s games targeted this)

### Backend: Host MIDI

For users with real Roland hardware connected to their PC.

Configuration:
```
-midi host,port=hw:1,0
```

Passes raw MIDI bytes through ALSA sequencer to the specified port.

### MIDI Priority / Hierarchy

Games typically support multiple MIDI devices and auto-detect what's available. The setup order in AUTOEXEC.BAT or game config determines priority:

1. **MT-32** — if the game supports it, use this (richest sound, custom patches)
2. **General MIDI** — nearly universal from 1993+, good quality
3. **OPL3 FM** — always available as fallback

QEMU-SA lets users choose via `-midi` flag. The guest game sees an MPU-401 regardless.

## Component 4: AC97 (Windows 98/XP)

### Already in QEMU

The Intel AC97 (ICH) audio codec is already implemented in QEMU (`hw/audio/intel-hda.c` and `hw/audio/ac97.c`). It works out of the box with:

- Windows 98SE (SigmaTel AC97 WDM drivers provide SB emulation in DOS box)
- Windows XP (native AC97 support)

### SigmaTel AC97 Driver for Win98

This is critical for running DOS games inside the Win98 DOS box. The SigmaTel C-Major AC97 WDM driver provides:
- Native Windows audio
- SoundBlaster emulation for DOS applications
- FM synthesis (basic) for DOS games

Driver: `stac9708.exe` or similar — must be obtained separately.

## Component 5: AWE32 EMU8000 Wavetable (Phase 3 — Stretch Goal)

### What It Is

The EMU8000 is the wavetable synthesis chip on the Sound Blaster AWE32/AWE64. It provides:
- 32-voice wavetable synthesis
- SoundFont loading (the format AWE32 invented)
- Hardware reverb and chorus effects
- 1MB onboard RAM (expandable to 28MB) for sample data

### Why It Matters

Some DOS games have specific AWE32 music: Warcraft II, System Shock, Epic Pinball, Jazz Jackrabbit. The AWE32 mode often uses unique instrument patches not available in General MIDI.

### Complexity Warning

The EMU8000 is significantly more complex than OPL3 or MPU-401:
- 32 independent voices with individual pitch, volume, pan, filter
- Custom DSP for reverb/chorus (needs reverse engineering or approximation)
- SoundFont loading via ISA DMA (RAM management)
- Multiple register banks at ports 620h, A20h, E20h

This is Phase 3 at earliest. DOSBox-X has a partial implementation that could serve as reference.

## Component 6: GUS (Gravis UltraSound)

### Already in QEMU

QEMU has a GUS implementation (`hw/audio/gus.c` via GUSemu library). It works but has known limitations:
- Sample playback timing may not be accurate
- Some games with custom GUS patches may not work

### Improvement Plans

Low priority — GUS support in QEMU is "good enough" for most games. Improvements if time permits:
- Better DMA timing
- Interwave (GUS PnP) support
- Verify compatibility with top GUS games (Doom, Epic Pinball, Tyrian)

## Audio Mixing Architecture

```
                    QEMU Audio Mixer
                    ┌─────────────────┐
OPL3 PCM ─────────→│                 │
SB16 PCM (DMA) ───→│  Mix + Volume   │──→ PulseAudio/ALSA
MIDI PCM (Munt) ──→│  Control        │       │
MIDI PCM (FS) ────→│                 │       ▼
CD Audio PCM ─────→│                 │    Speakers
GUS PCM ──────────→│                 │
                    └─────────────────┘
```

Each source has independent volume control. The mixer handles sample rate conversion and channel mapping.

## Audio Latency Targets

| Component | Target Latency | Notes |
|-----------|---------------|-------|
| OPL3 | < 10ms | Buffer: 512 samples at 49716 Hz |
| SB16 PCM | < 20ms | DMA buffer dependent |
| MIDI (Munt) | < 20ms | MT-32 emulation adds ~5ms |
| MIDI (FluidSynth) | < 15ms | Configurable buffer |
| CD Audio | < 50ms | Acceptable for background music |
