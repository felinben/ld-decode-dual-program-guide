# MLD RF Capture — *Virtual Cameraman*
## A Guide to Decoding and Preserving a Pioneer LaserActive Dual-Program Disc

This repository documents the process of decoding an RF capture of the *Virtual Cameraman* MegaLD disc — a dual-program Japanese LaserDisc title for the Pioneer LaserActive — from raw signal to archival video files and (in progress) an Ares emulator MMI package.

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

### 3. [Ares Emulator and the MMI Format](ARES_MMI.md)
MLD disc structure, the MediaInfo.json stream layout, why QON is required over standard video formats, the EFM corruption in this particular capture, and the requirements for a correct future capture.

## External Reference

The authoritative procedure for archiving LaserActive titles (hardware requirements, correct stop-code handling, capture workflow) is maintained by the Exodus emulator team:

```
https://techdocs.exodusemulator.com/Console/PioneerLaserActive/Archiving.html
```

---

*Written with assistance from **Claude Sonnet 4.6** (Anthropic), via [Claude Code](https://claude.ai/code).*
