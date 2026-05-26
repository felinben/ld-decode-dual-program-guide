# Technical Overview
## LaserDisc, ld-decode, and the Pioneer LaserActive

---

## Table of Contents

1. [What Is a LaserDisc RF Capture?](#1-what-is-a-laserdisc-rf-capture)
2. [What Is ld-decode?](#2-what-is-ld-decode)
3. [What Is a TBC File?](#3-what-is-a-tbc-file)
4. [What Is a Dual-Program LaserDisc?](#4-what-is-a-dual-program-laserdisc)
5. [What Is the Pioneer LaserActive?](#5-what-is-the-pioneer-laseractive)
6. [CAV vs CLV — Visual Identification](#6-cav-vs-clv--visual-identification)

---

## 1. What Is a LaserDisc RF Capture?

Inside a LaserDisc player, an analog signal chain converts the optical signal read from the disc into a standard composite video output. Most LaserDisc "rips" go through this chain and are recorded at the composite output — meaning the signal has already passed through the player's internal circuits, accumulating noise, chroma crosstalk, and time-base jitter.

An **RF capture** goes further upstream. It taps the raw **FM (Frequency Modulated) signal** directly from the disc mechanism, before the player does any demodulation. This raw signal — containing luma, chroma, sync, audio, and metadata all multiplexed together as FM sidebands — is digitised at high sample rate and stored to disk. The result is a perfect digital copy of the signal as it came off the disc surface, entirely independent of the player's aging analog circuits.

The file format used here is `.ldf` (**LaserDisc Frequency capture**), a raw PCM recording of the RF signal at approximately 40 Msps. A single disc side typically produces 300–500 GB of data.

---

## 2. What Is ld-decode?

**ld-decode** is an open-source, software-defined LaserDisc RF decoder. It replicates — entirely in software — the signal processing that a LaserDisc player's hardware would perform:

- FM demodulation of the luma (brightness) channel
- Time-base correction (TBC) to remove mechanical jitter
- Dropout detection and concealment
- EFM digital audio extraction
- Analog FM audio demodulation
- VBI (Vertical Blanking Interval) metadata decoding

Because it operates on a perfect digital recording of the raw signal, ld-decode can produce output quality that exceeds what any real player hardware can achieve, particularly for discs with surface wear or pressing defects.

ld-decode is the core of a broader ecosystem of tools — `ld-chroma-decoder`, `ld-process-vbi`, `ld-analyse`, and others — all sharing the same TBC file format.

---

## 3. What Is a TBC File?

The primary output of ld-decode is a `.tbc` (**Time-Base Corrected**) file. This is a raw, uncompressed video file containing every field of the LaserDisc, stored as 16-bit luma samples at the disc's native resolution:

- **910 samples wide × 263 lines per field** for NTSC (including active picture and blanking)
- Fields are stored sequentially: field 1, field 2, field 3, …
- No compression, no colour information — luma only

Colour information is encoded in the luma signal itself (as the NTSC colour subcarrier) and is extracted separately by `ld-chroma-decoder`.

A companion **`.tbc.db`** SQLite database stores per-field metadata: signal-to-noise ratio, detected dropouts, VBI content (frame numbers, timecodes, chapter markers), and sync quality metrics.

---

## 4. What Is a Dual-Program LaserDisc?

Standard NTSC video is **interlaced**: each video frame consists of two **fields** captured at slightly different moments in time.

- The **top field** (also called the odd field) contains lines 1, 3, 5, 7, … of the picture
- The **bottom field** (also called the even field) contains lines 2, 4, 6, 8, …
- Fields alternate at **59.94 Hz**; two fields make one complete frame at **29.97 fps**

On some Japanese LaserDiscs — particularly idol and gravure video titles — a technique was used to encode **two entirely different programs** on alternating fields:

- Odd fields → Program A
- Even fields → Program B

A specialised player (or a player with a hardware switch) could select which field set to display, effectively choosing between the two programs. The analog FM stereo audio tracks were correspondingly split: the **left channel** carries Program A's audio, and the **right channel** carries Program B's audio.

When decoded naively — treating both fields as a normal interlaced pair — the result looks like severe combing: each "frame" contains one field from Program A interleaved with one field from Program B, creating a picture with half-resolution horizontal stripes of two completely different scenes. Correct extraction requires separating the two field sets entirely.

The Exodus emulator technical documentation refers to this technique as **"selective interlacing"** — displaying only odd or even fields rather than the traditional interlaced pair — and notes it as a known archival challenge that standard capture hardware gets wrong, as it assumes conventional NTSC interlacing.

---

## 5. What Is the Pioneer LaserActive?

The Pioneer LaserActive (CLD-A100/CLD-A200) was a hybrid LaserDisc/Mega Drive/PC Engine console released in 1993. It played standard LaserDiscs as well as **MLD (Mega LaserDisc)** titles — LaserDiscs that contained game data or extended content accessible only through the console's add-on modules. The `RF-MLD_` prefix in the filename indicates this is an MLD disc captured from a LaserActive player.

---

## 6. CAV vs CLV — Visual Identification

LaserDiscs were mastered in one of two formats:

- **CAV (Constant Angular Velocity)**: The disc rotates at a fixed speed (1,800 rpm for NTSC; 1,500 rpm for PAL). Each complete revolution stores exactly one video frame. This 1:1 relationship between revolution and frame enables still-frame, slow-motion, and random frame access — features used extensively by MLD and interactive titles. CAV discs hold up to 30 minutes of video per side (36 minutes for PAL).

- **CLV (Constant Linear Velocity)**: The disc spins faster near the centre and slower toward the edge, maintaining a constant track speed. This allows up to 60 minutes per side for NTSC (64 minutes for PAL) but loses the fixed frame-per-revolution relationship, making still-frame and random frame access impossible without special hardware.

You can distinguish the two formats visually by inspecting the disc surface under a light source:

- **CAV discs** show prominent radial streaks — "spokes" — radiating outward from the hub. These appear because every adjacent track shares the same angular frame-start position, creating a consistent radial zone of slightly different reflectivity across the entire playing area.

- **CLV discs** have a uniform, smooth iridescent appearance with no radial patterning. The track is a true continuous spiral with no fixed angular frame boundaries.

![CAV vs CLV diagram](cav_clv_diagram.svg)

The photograph below shows a CAV LaserDisc. The spoke pattern is clearly visible as bright radial streaks in the iridescent reflection. Note the disc label explicitly reads "(CAV)":

![CAV LaserDisc showing spoke pattern](Laserdisc_CAV.jpg)

*Photo: [Autopilot](https://commons.wikimedia.org/wiki/File:Laserdisc_CAV.jpg), [CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/)*

All MLD titles are NTSC — the LaserActive was only sold in Japan and North America, both NTSC markets. Most MLD titles are CAV. The `"format"` field in MediaInfo.json (`"NTSC-CAV"` or `"NTSC-CLV"`) should reflect what you observe on the disc itself.

---

*Pipeline design, filtergraph derivation, and manual written with assistance from **Claude Sonnet 4.6** (Anthropic), via [Claude Code](https://claude.ai/code).*
