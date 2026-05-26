# ares Emulator and the MMI Format
## Packaging an MLD Disc for Emulation

> New to MLD disc structure, CAV/CLV, or the LaserActive? Read the [Technical Overview](TECHNICAL_OVERVIEW.md) first.
> The ld-decode pipeline that produces the source files is documented in [Fun with ld-decode](LD_DECODE_PIPELINE.md).

---

## Table of Contents

1. [MLD Disc Structure — Where the Game Code Lives](#1-mld-disc-structure--where-the-game-code-lives)
2. [Current Challenges — Towards an MMI for ares](#2-current-challenges--towards-an-mmi-for-ares)
   - [2.1 AnalogVideo — Why QON](#21-analogvideo--why-qon-not-a-standard-video-format)
   - [2.2 AnalogVideo — Dual-Program Complication](#22-analogvideo--the-dual-program-complication)
   - [2.3 AnalogAudio — Status](#23-analogaudio--status)
   - [2.4 DigitalAudio — EFM Corruption in This Capture](#24-digitalaudio--efm-corruption-in-this-capture)
   - [2.5 How to Do a Correct Capture](#25-how-to-do-a-correct-capture)
   - [2.6 Summary of Open Questions](#26-summary-of-open-questions)
3. [Software and Tools](#3-software-and-tools)
   - [3.1 Emulation](#31-emulation)
   - [3.2 Original Tools](#32-original-tools)
   - [3.3 Community Tools](#33-community-tools)

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

## 2. Current Challenges — Towards an MMI for ares

The [ld-decode pipeline](LD_DECODE_PIPELINE.md) produces clean, archival-quality video files (`VC_prog1_final.mov`, `VC_prog2_final.mov`). Creating an MMI package for the [ares emulator](https://ares-emu.net) requires a further conversion step for each of the three MMI streams. This section documents the known challenges and open questions as of the time of writing.

**Catalogue IDs.** The Exodus title catalogue lists two *Virtual Cameraman* MegaLD titles for Japan:

| Title | Catalogue ID |
|-------|-------------|
| Virtual Cameraman | PEASJ5015 |
| Virtual Cameraman 2 | PEASJ5020 |

The archive.org capture is confirmed as *Virtual Cameraman* **PEASJ5015** — this is the catalogue ID to use in MediaInfo.json.

**Running ares.** Before loading any MegaLD title, ares requires BIOS files which are available from archive.org. Without them the emulator will not boot games regardless of MMI completeness.

### 2.1 AnalogVideo — Why QON, Not a Standard Video Format

A natural question is: why not use the ProRes `.mov` files produced by the pipeline directly as the video stream?

The answer is frame addressability. ares cannot use standard video container formats (MOV, MP4, MKV, etc.) for LaserDisc video because those formats are built around sequential playback. Their codecs use inter-frame compression — P-frames and B-frames that encode only the *difference* from neighbouring frames — which means decoding any given frame requires first decoding the frames before it. That is fundamentally incompatible with how a LaserDisc player operates: the hardware must be able to jump instantly to any arbitrary frame number, freeze on a single frame indefinitely, and play backwards or at variable speed. Games such as *Rocket Coaster* exercise all of these capabilities continuously during gameplay.

The **QON format** (from [github.com/RogerSanders/qon](https://github.com/RogerSanders/qon), a LaserActive-specialised fork of the QOI image format) solves this by storing every frame as an independently compressed, individually addressable unit — effectively a flat array of frames with no inter-frame dependencies. Any frame can be sought to and decoded in isolation, in constant time.

The source for the QON encode is the TBC file (`VC.tbc`), not the finished `.mov` files. The `.mov` files have already had the dual-program fields separated and scan-doubled; the TBC contains the raw interlaced output of ld-decode, closer to what the original hardware would have read off the disc.

### 2.2 AnalogVideo — The Dual-Program Complication

*Virtual Cameraman* is a dual-program disc: odd fields carry Program A and even fields carry Program B. This raises an open question for the QON stream:

**Should the QON contain the raw interleaved frames as they appear in the TBC (both programs on alternating fields), or should it contain two separate QON files — one per program?**

The answer depends on how the LaserActive hardware (and therefore the ares emulator) selects between programs. If the hardware simply plays back the raw field stream and the add-on module selects odd or even fields at display time, the QON should contain the unmodified interleaved output of ld-chroma-decoder. If the hardware or software expects pre-separated streams, two QON files would be needed.

This question is currently unresolved and is being discussed with the ares development team.

### 2.3 AnalogAudio — Status

The analog audio source (`VC.pcm`) is already in the correct format for the `AnalogAudio` MMI stream: raw s16le PCM, 44100 Hz, stereo. No conversion is required beyond confirming the channel-to-program mapping (left channel → one program, right channel → the other) and whether the MMI expects a split mono file or the full stereo PCM.

### 2.4 DigitalAudio — EFM Corruption in This Capture

The digital audio pipeline as far as WAV is documented in [LD_DECODE_PIPELINE.md Section 7.4](LD_DECODE_PIPELINE.md#74-efm-digital-audio-advanced-optional):

```
VC.efm → efm-decoder-f2 → VC.f2 → efm-decoder-audio → VC_digital.wav
```

However, the EFM data in this particular archive.org capture is known to be **corrupted at the locations that store the game code**. Nemesis (ares emulator author) examined this capture and reported the following (18 August 2025):

> *"I had a go at converting the virtual cameraman ldf rip that's on archive.org, but the rip wasn't done properly. They just ran the DdD mode that pushes through picture stop codes by repeatedly forcing the player to resume every time it stops. Sure that gets a picture out, but it corrupts the EFM at the point where it loops, which is where the digital data stores the game code, so it's scrambled and unusable."*

**What this means in practice:**

- The **analog video** and **analog audio** extractions documented in [LD_DECODE_PIPELINE.md](LD_DECODE_PIPELINE.md) are unaffected — they do not rely on the EFM track.
- The `VC.efm` file from this capture contains corrupted data at the picture-stop-code loop points. Decoding it will produce a WAV file, but the game code data tracks within the EFM are scrambled and cannot be used to build a working bin/cue.
- A complete MMI package for *Virtual Cameraman* requires a **new RF capture** performed without DdD mode — one that allows the disc to be read past picture stop codes without forcing repeated resumes.

The toolchain question (EFM → bin/cue) remains valid for future properly-captured MLD titles; it is documented as open below.

Nemesis has confirmed (May 2026) that a working decode-to-MMI pipeline exists, but it currently relies on an unpublished EFM decoder and a modified version of `ld-analyse` that dumps frames as a PNG stack — neither in a state for general use. He intends to tidy these up and contribute changes back to the ld-decode toolchain so the pipeline can be properly reproduced by others. The Exodus techdocs [decoding section](https://techdocs.exodusemulator.com/Console/PioneerLaserActive/Archiving.html) is marked TODO and will document the official procedure when it is ready.

A further update (May 2026) reports that a batch of new titles is ready to release, but two additional blockers have emerged:

- **ares mainline audio desync**: A recent cdrom layer refactor introduced an audio sync regression for MegaLD titles. The issue is present in current mainline but not in the build hosted on his website. Nemesis intends to fix this before releasing new titles, as the regression makes testing against mainline unreliable.
- **Emulation bugs under verification**: Two emulation bugs surfaced during testing of the new rips. One has been fixed; the second fix is complete but needs re-verification across all titles — work that depends on the audio sync issue being resolved first.

Progress is temporarily paused due to personal commitments. Nemesis estimates at least another month before he can return to this. The toolchain and titles exist; it is a matter of timing.

### 2.5 How to Do a Correct Capture

The authoritative procedure for archiving LaserActive titles is documented by the Exodus emulator team at:

```
https://techdocs.exodusemulator.com/Console/PioneerLaserActive/Archiving.html
```

#### Understanding the Tools — A Beginner's Guide

Before getting into the correct procedure it helps to understand what the various tools actually are, since they are often mentioned together and can be confused.

**DomesDay Duplicator (DdD)** is an FPGA-based hardware capture board developed by the Domesday86 project. It connects to a LaserDisc player's test header via BNC cable and records the raw FM signal at 40 Msps, producing an LDF file. It is the standard hardware for LaserDisc RF capture, and it is what Tanks of Kineko Video used to make the *Virtual Cameraman* archive.org item.

**Tanks of Kineko Video** is a professional Japanese video transfer service — not a tool. They performed the capture using the DomesDay Duplicator and uploaded the result to archive.org.

**DumpMegaLD** is a MegaCD ROM (software) written by Nemesis that runs on the LaserActive itself and uses the PAC-S1 hardware to read digital sector data directly off an MLD disc. It predates RF capture and is now considered obsolete — the DomesDay Duplicator approach produces a more complete result.

**MegaLDRegEditor** is also a MegaCD ROM by Nemesis, but it is not a capture tool at all. It is a diagnostic utility for reading and writing registers on the PD6103A IC — the chip that bridges the MegaCD and the LaserDisc mechanism — used for hardware research and development.

---

The archive.org capture and a correct capture both use the same hardware (DomesDay Duplicator) and tap the same raw RF signal. The entire difference comes down to two things: **player choice** and **how picture stop codes are handled**.

MLD discs embed picture stop codes at points where the game needs to pause the disc and interact with the player hardware. The EFM stream — which carries the game code — must remain uninterrupted across these points. The DomesDay Duplicator offers two ways to deal with them:

| | Archive.org capture | Correct procedure |
|---|---|---|
| **Who performed it** | Tanks of Kineko Video | — |
| **Capture hardware** | DomesDay Duplicator | DomesDay Duplicator |
| **Player** | Likely CLD-A100 (LaserActive itself) ✗ | LD-V4300D, LD-V4400, or LD-V8000 ✓ |
| **Stop code handling** | DdD mode — forces player to resume repeatedly ✗ | `1PS` serial command — disables stop codes before playback ✓ |
| **Analog video / audio** | ✓ Clean — this guide's pipeline works | ✓ Clean |
| **EFM / game code** | ✗ Corrupted at every stop-code loop point | ✓ Intact across full disc |
| **Suitable for video archive** | ✓ Yes | ✓ Yes |
| **Suitable for MMI / emulation** | ✗ No — game code unusable | ✓ Yes |

The `1PS` serial command requirement is MLD-specific and not widely known outside the specialist community. This was a knowledge gap, not negligence on the part of Tanks of Kineko Video.

---

Key requirements for a capture that will yield a usable EFM track:

**Hardware**

| Item | Requirement |
|------|-------------|
| LaserDisc player | LD-V4300D, LD-V4400, or LD-V8000 |
| Capture hardware | DomesDay Duplicator 3.0 (DE0-Nano FPGA + Cypress FX3) |
| Player connection | Serial cable to PC (required for test mode control) |
| RF connection | BNC cable from player test header |

> **Critical:** The Pioneer CLD-A100/A200 (the LaserActive itself) **cannot be used** for RF capture — it has an interference design that risks physical damage to the capture hardware. A separate compatible player is required.

**Picture stop code handling**

The correct procedure sends the serial command `1PS` to the player before capture to disable picture stop codes entirely, allowing uninterrupted playback from lead-in to lead-out. This is what preserves EFM integrity across the full disc.

The broken approach — using the "Multi Fwd" button (or equivalent software bypass) to force the player past each stop code as it is encountered — repeatedly interrupts and resumes the EFM stream, corrupting the data at every loop point.

**Capture procedure (summary)**

1. Enter service/diagnostic mode via front panel or serial
2. Enable open tracking (unstable still-frame mode)
3. Seek backward to the innermost lead-in ring (~53 mm radius)
4. Disable picture stop codes via serial command `1PS`
5. Initiate DomesDay Duplicator capture (34 minutes for CAV, 64 minutes for CLV)
6. Start playback in test mode; allow it to run to lead-out
7. Repeat the full capture 5 times per disc side for error correction

**Storage**

A single CAV disc side produces approximately 150 GB of raw RF data. The Exodus guide recommends retaining the raw RF files permanently rather than relying on the decoded outputs alone.

### 2.6 Summary of Open Questions

| # | Question | Status |
|---|----------|--------|
| 1 | Should the QON contain raw interleaved fields or pre-separated per-program streams? | Open — pending ares team input |
| 2 | Does the `AnalogAudio` stream expect stereo PCM or split mono files? | Open |
| 3 | What is the correct toolchain and track layout for building a Redbook bin/cue from decoded EFM output? | Open (toolchain); **blocked on new capture** for this disc. A working pipeline exists (Nemesis, May 2026) but is not yet publicly documented — upstream contributions to ld-decode are planned. |
| 4 | How are lead-in and lead-out frame counts determined for `framesInLeadInRegion` / `framesInLeadOutRegion` in MediaInfo.json — from `VC.tbc.db`, or counted manually? | Open |

---

## 3. Software and Tools

### 3.1 Emulation

**[ares](https://ares-emu.net)** (ares team; MegaLD/LD-ROM² support by Nemesis) is the primary — and currently only — emulator capable of running LaserActive titles. It supports both the Sega PAC (PAC-S1/S10) and NEC PAC (PAC-N1) variants and loads games via the `.mmi` (Mixed Media Image) format documented in this guide. BIOS files are required and are available from archive.org.

### 3.2 Original Tools

**PAC-PC1 Program Editor** (Pioneer) — the original SDK tool used by Pioneer and licensed developers to author MegaLD titles. Available via the Exodus techdocs [Software page](https://techdocs.exodusemulator.com/Console/PioneerLaserActive/Software.html).

### 3.3 Community Tools

All three community tools were developed by Nemesis (ares author):

**DumpMegaLD** — a Mode 1 MegaCD ROM that initialises the LaserDisc hardware from the PAC-S1/S10 and extracts full raw sector data and subcode from MegaLD/LD-ROM² discs. Now considered **obsolete** — RF capture via the DomesDay Duplicator produces a more complete and higher-quality source.

**MegaLDRegEditor** — a Mode 1 MegaCD ROM for interactive reading and writing of the PD6103A IC register block. The PD6103A is the chip that bridges the MegaCD and LaserActive hardware, distinguishing the PAC-S1/S10 from a standard Mega Drive/MegaCD combination. Useful for low-level hardware research and debugging.

**LDBIOS.INC** — reverse-engineered BIOS routine entry points for MegaLD systems in SDK-compatible format, with detailed notes on the PD6103A reverse-engineering effort. Now partially outdated; the most current information is in the ares source code.

All three are available from the Exodus techdocs [Software page](https://techdocs.exodusemulator.com/Console/PioneerLaserActive/Software.html).

---

*Pipeline design, filtergraph derivation, and manual written with assistance from **Claude Sonnet 4.6** (Anthropic), via [Claude Code](https://claude.ai/code).*
