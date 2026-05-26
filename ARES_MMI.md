# Ares Emulator and the MMI Format
## Packaging an MLD Disc for Emulation

> New to MLD disc structure, CAV/CLV, or the LaserActive? Read the [Technical Overview](TECHNICAL_OVERVIEW.md) first.
> The ld-decode pipeline that produces the source files is documented in [Fun with ld-decode](LD_DECODE_PIPELINE.md).

---

## Table of Contents

1. [MLD Disc Structure — Where the Game Code Lives](#1-mld-disc-structure--where-the-game-code-lives)
2. [Current Challenges — Towards an MMI for Ares](#2-current-challenges--towards-an-mmi-for-ares)

---

## 1. MLD Disc Structure — Where the Game Code Lives

An MLD disc carries three distinct content types, which map directly to the three streams in MediaInfo.json:

| Stream | Type | Content |
|--------|------|---------|
| `AnalogVideo` | QON | FMV frames (frame-addressable video) |
| `AnalogAudio` | Raw PCM | Analog FM stereo audio |
| `DigitalAudio` | Redbook bin/cue | Game code **and** digital audio |

The `DigitalAudio` label is slightly misleading. A Redbook bin/cue is not just audio — it is a complete CD disc image, and CD images can contain **data tracks** (Mode 1 or Mode 2/XA) alongside audio tracks. On an MLD disc the EFM digital track is a mixed disc: the data tracks carry the Mega Drive or PC Engine game ROM and assets; the audio tracks carry the digital stereo soundtrack.

The LaserActive's add-on module (Mega Drive PAC-S1 or PC Engine PAC-N1) reads the game code from those data tracks exactly as it would from a normal Mega CD or PC Engine CD-ROM disc. There is no separate "code" stream in the MMI JSON because the bin/cue is already the complete disc image that the emulator's CD-ROM layer consumes.

From the emulator's perspective:

- **QON** → video renderer (frame-addressable, required for sync'd FMV, reverse playback, and still-frame)
- **PCM** → analog audio output
- **bin/cue** → CD-ROM drive emulation (game logic, data assets, and digital audio tracks)

---

## 2. Current Challenges — Towards an MMI for Ares

The [ld-decode pipeline](LD_DECODE_PIPELINE.md) produces clean, archival-quality video files (`VC_prog1_final.mov`, `VC_prog2_final.mov`). Creating an MMI package for the [Ares emulator](https://ares-emu.net) requires a further conversion step for each of the three MMI streams. This section documents the known challenges and open questions as of the time of writing.

### 2.1 AnalogVideo — Why QON, Not a Standard Video Format

A natural question is: why not use the ProRes `.mov` files produced by the pipeline directly as the video stream?

The answer is frame addressability. Ares cannot use standard video container formats (MOV, MP4, MKV, etc.) for LaserDisc video because those formats are built around sequential playback. Their codecs use inter-frame compression — P-frames and B-frames that encode only the *difference* from neighbouring frames — which means decoding any given frame requires first decoding the frames before it. That is fundamentally incompatible with how a LaserDisc player operates: the hardware must be able to jump instantly to any arbitrary frame number, freeze on a single frame indefinitely, and play backwards or at variable speed. Games such as *Rocket Coaster* exercise all of these capabilities continuously during gameplay.

The **QON format** (from [github.com/RogerSanders/qon](https://github.com/RogerSanders/qon), a LaserActive-specialised fork of the QOI image format) solves this by storing every frame as an independently compressed, individually addressable unit — effectively a flat array of frames with no inter-frame dependencies. Any frame can be sought to and decoded in isolation, in constant time.

The source for the QON encode is the TBC file (`VC.tbc`), not the finished `.mov` files. The `.mov` files have already had the dual-program fields separated and scan-doubled; the TBC contains the raw interlaced output of ld-decode, closer to what the original hardware would have read off the disc.

### 2.2 AnalogVideo — The Dual-Program Complication

*Virtual Cameraman* is a dual-program disc: odd fields carry Program A and even fields carry Program B. This raises an open question for the QON stream:

**Should the QON contain the raw interleaved frames as they appear in the TBC (both programs on alternating fields), or should it contain two separate QON files — one per program?**

The answer depends on how the LaserActive hardware (and therefore the Ares emulator) selects between programs. If the hardware simply plays back the raw field stream and the add-on module selects odd or even fields at display time, the QON should contain the unmodified interleaved output of ld-chroma-decoder. If the hardware or software expects pre-separated streams, two QON files would be needed.

This question is currently unresolved and is being discussed with the Ares development team.

### 2.3 AnalogAudio — Status

The analog audio source (`VC.pcm`) is already in the correct format for the `AnalogAudio` MMI stream: raw s16le PCM, 44100 Hz, stereo. No conversion is required beyond confirming the channel-to-program mapping (left channel → one program, right channel → the other) and whether the MMI expects a split mono file or the full stereo PCM.

### 2.4 DigitalAudio — EFM to Redbook bin/cue

The digital audio pipeline as far as WAV is documented in [LD_DECODE_PIPELINE.md Section 7.4](LD_DECODE_PIPELINE.md#74-efm-digital-audio-advanced-optional):

```
VC.efm → efm-decoder-f2 → VC.f2 → efm-decoder-audio → VC_digital.wav
```

What remains undocumented is the step from `VC_digital.wav` to a properly structured **Redbook bin/cue** suitable for the `DigitalAudio` MMI stream. For a standard MLD title this bin/cue is a mixed disc image: data tracks carry the game ROM and assets, audio tracks carry the digital soundtrack. The correct toolchain and track layout for assembling this from the decoded EFM output has not yet been established for this disc.

### 2.5 Summary of Open Questions

| # | Question | Status |
|---|----------|--------|
| 1 | Should the QON contain raw interleaved fields or pre-separated per-program streams? | Open — pending Ares team input |
| 2 | Does the `AnalogAudio` stream expect stereo PCM or split mono files? | Open |
| 3 | What is the correct toolchain and track layout for building a Redbook bin/cue from the decoded EFM output? | Open |
| 4 | How are lead-in and lead-out frame counts determined for `framesInLeadInRegion` / `framesInLeadOutRegion` in MediaInfo.json — from `VC.tbc.db`, or counted manually? | Open |

---

*Pipeline design, filtergraph derivation, and manual written with assistance from **Claude Sonnet 4.6** (Anthropic), via [Claude Code](https://claude.ai/code).*
