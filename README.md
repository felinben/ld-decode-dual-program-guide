# MLD RF Capture — *Virtual Cameraman*
## A Guide to Decoding and Preserving a Pioneer LaserActive Dual-Program Disc

This repository documents the process of decoding an RF capture of the *Virtual Cameraman* MegaLD disc — a dual-program Japanese LaserDisc title for the Pioneer LaserActive — from raw signal to archival video files and (in progress) an ares emulator MMI package.

The source RF capture is publicly archived at:
```
https://archive.org/details/virtual-cameraman-pioneer-laseractive-discdump-hiresscans
```

---

## Documents

### 1. [Technical Overview](TECHNICAL_OVERVIEW.md)
Background concepts shared by both documents below: what an RF capture is, how ld-decode works, TBC files, dual-program discs, the Pioneer LaserActive, and how to visually identify CAV vs CLV discs.

### 2. [Fun with ld-decode — Extracting Video](LD_DECODE_PIPELINE.md)
The complete step-by-step pipeline from LDF source file to archival ProRes video: RF decode, VBI processing, chroma decode, dual-program field separation, audio channel splitting, and troubleshooting. Two clean videos were successfully extracted from this capture.

### 3. [ares Emulator and the MMI Format](ARES_MMI.md)
MLD disc structure, the MediaInfo.json stream layout, why QON is required over standard video formats, the EFM corruption in this particular capture, and the requirements for a correct future capture.

## External References

The Exodus emulator team maintains a comprehensive set of technical documentation for the LaserActive:

| Page | Content |
|------|---------|
| [Archiving](https://techdocs.exodusemulator.com/Console/PioneerLaserActive/Archiving.html) | Authoritative capture procedure: hardware requirements, stop-code handling, 5-pass workflow |
| [LaserDisc](https://techdocs.exodusemulator.com/Console/PioneerLaserActive/Laserdisc.html) | Signal encoding, RF format details, selective interlacing, lead-in/out structure |
| [Titles](https://techdocs.exodusemulator.com/Console/PioneerLaserActive/Titles.html) | Full MegaLD/LD-ROM² title catalogue with archive status and download links |
| [Software](https://techdocs.exodusemulator.com/Console/PioneerLaserActive/Software.html) | ares emulator, original Pioneer tools, and community tools by Nemesis |

---

*Written with assistance from **Claude Sonnet 4.6** (Anthropic), via [Claude Code](https://claude.ai/code).*
