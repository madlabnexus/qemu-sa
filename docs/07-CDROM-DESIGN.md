# 07 — CD-ROM Design

## Problem

QEMU only supports ISO 9660 disc images. Many retro games ship as CUE/BIN images that include:
- Data tracks (ISO 9660 filesystem)
- Audio tracks (Red Book CD-DA)

Games like Descent, Command & Conquer, Wing Commander III, Quake, and hundreds of others play music directly from CD audio tracks. With ISO-only support, these games run in silence.

## Solution

Integrate libcue (CUE sheet parser) and build a CD Digital Audio playback engine into QEMU's ATAPI CD-ROM device.

## CUE/BIN Format Overview

A CUE file describes the disc layout:

```
FILE "game.bin" BINARY
  TRACK 01 MODE1/2352
    INDEX 01 00:00:00
  TRACK 02 AUDIO
    PREGAP 00:02:00
    INDEX 01 04:32:15
  TRACK 03 AUDIO
    INDEX 01 09:15:42
```

- **TRACK 01 MODE1/2352**: Data track — 2352 bytes per sector including headers/ECC
- **TRACK 02+ AUDIO**: Red Book audio — 2352 bytes per sector = 588 stereo 16-bit samples at 44100 Hz
- **INDEX 01**: Start position in MSF (Minutes:Seconds:Frames, 75 frames/sec)
- **PREGAP**: Silence before track starts

### Variant Formats

Some rips split audio into separate files:

```
FILE "game.bin" BINARY
  TRACK 01 MODE1/2352
    INDEX 01 00:00:00
FILE "track02.wav" WAVE
  TRACK 02 AUDIO
    INDEX 01 00:00:00
FILE "track03.flac" FLAC
  TRACK 03 AUDIO
    INDEX 01 00:00:00
```

QEMU-SA must support both single-BIN and multi-file layouts.

## Architecture

### Component: CUE Parser (`hw/ide/cue-parser.c`)

Uses libcue to parse CUE sheets:

```c
typedef struct CueDisc {
    int num_tracks;
    CueTrack tracks[];
} CueDisc;

typedef struct CueTrack {
    int number;
    TrackType type;          /* DATA_MODE1, DATA_MODE2, AUDIO */
    char *filename;          /* source file path */
    uint64_t file_offset;    /* byte offset in source file */
    uint64_t length_bytes;   /* track length in bytes */
    int pregap_frames;       /* pregap in CD frames (75/sec) */
    int start_lba;           /* logical block address */
    AudioFormat audio_fmt;   /* RAW_PCM, WAV, FLAC, OGG */
} CueTrack;
```

### Component: CDDA Player (`hw/ide/cdda-player.c`)

Reads and decodes audio tracks, feeds PCM to QEMU audio mixer:

```c
typedef struct CDDAPlayer {
    CueDisc *disc;
    int current_track;
    uint64_t current_frame;      /* position within track */
    bool playing;
    bool paused;
    QEMUAudioOutputStream *audio_out;
    /* Decoder state for non-PCM formats */
    void *decoder;
} CDDAPlayer;
```

Supported audio formats:
- **Raw PCM** (in BIN file) — direct read, no decoding
- **WAV** — trivial header parsing + raw PCM
- **FLAC** — via libFLAC (lossless, common for CD rips)
- **OGG Vorbis** — via libvorbis (lossy, smaller files)

### Component: ATAPI Modifications (`hw/ide/atapi.c`)

Extend existing ATAPI handler with CD audio commands:

| ATAPI Command | Opcode | Function |
|---------------|--------|----------|
| READ TOC | 0x43 | Return track list with types and positions |
| PLAY AUDIO MSF | 0x47 | Start playing from M:S:F position |
| PLAY AUDIO (10) | 0x45 | Start playing from LBA |
| PAUSE/RESUME | 0x4B | Pause or resume audio playback |
| STOP | 0x4E | Stop audio playback |
| READ SUB-CHANNEL | 0x42 | Return current play position, track, state |
| READ CD | 0xBE | Read raw sectors (data + audio) |
| GET CONFIGURATION | 0x46 | Report CD-ROM capabilities |

### Data Flow

```
Guest sends ATAPI PLAY AUDIO MSF (start=track 02, index 01)
        │
        ▼
atapi.c → identifies command, extracts MSF position
        │
        ▼
cue-parser.c → maps MSF to track number + byte offset
        │
        ▼
cdda-player.c → opens source file, seeks to offset
        │
  ┌─────┴─────┐
  │ RAW PCM?  │ → direct read
  │ WAV?      │ → skip header, read PCM
  │ FLAC?     │ → libFLAC decode to PCM
  │ OGG?      │ → libvorbis decode to PCM
  └─────┬─────┘
        ▼
44100 Hz stereo 16-bit PCM
        │
        ▼
QEMU audio mixer → PulseAudio/ALSA → speakers

Meanwhile, atapi.c responds to READ SUB-CHANNEL
with current position (updated per-frame)
```

## Command Line Interface

```bash
# CUE/BIN disc image
qemu-system-i386 ... -cdrom /path/to/game.cue

# QEMU-SA detects .cue extension and uses CUE parser
# instead of standard ISO handler
```

The detection is automatic: if the file ends in `.cue`, use CUE parser. If `.iso`, use stock ISO handler.

## Edge Cases

### Mixed Mode Discs

Track 1 is data (game files), tracks 2+ are audio (music). This is the most common retro game format. The guest reads data from track 1 via normal file I/O and plays audio tracks via ATAPI audio commands.

### Multi-Session Discs

Rare in gaming. Not planned for Phase 2.

### Copy Protection

Some games check for specific subchannel data (SafeDisc, SecuROM). CUE/BIN format doesn't capture subchannel data. This affects a small number of titles. Workaround: use cracked executables (user's responsibility) or specific subchannel-preserving formats.

### CD Swap

Games that span multiple CDs (e.g., Final Fantasy VII) need disc changing. QEMU supports media change via QMP:

```bash
# Via QEMU monitor
change ide1-cd0 /path/to/disc2.cue
```

Can be automated with a launcher UI in Phase 5.

## Build Dependencies

```bash
# Arch Linux
sudo pacman -S libcue flac libvorbis libogg libsndfile

# The build system links these when --enable-libcue is set
```

## Test Games

| Game | Tracks | Audio Format | Test Focus |
|------|--------|-------------|------------|
| Descent (1995) | Data + 10 audio | PCM | Basic CD audio play |
| Command & Conquer (1995) | Data + ~20 audio | PCM | Many tracks, fast switching |
| Quake (1996) | Data + 10 audio | PCM | Track 1 data, 2-11 audio |
| Wing Commander III (1994) | Data + FMV audio | PCM | Large disc, seeking |
| Myst (1993) | Data + audio | PCM | Ambient audio loops |
| Red Alert (1996) | Data + audio | PCM | Music during gameplay |
| Wipeout (1995) | Data + licensed music | PCM | Audio track selection |
