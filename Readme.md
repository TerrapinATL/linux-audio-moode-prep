# [Linux Audio Utilities](https://github.com/TerrapinATL/linux.audio.flac-clean-up/edit/main/Readme.md)

**Guide version: v26** — The full moOde library cleanup guide (`linux-audio-moode-cleanup-guide.md`). Current version; supersedes v25.

A collection of lightweight command-line workflows and utilities designed for validating, cleaning, managing, and backing up local music libraries on Linux.

## Overview

This repository contains tools and documentation to help maintain a pristine, standardized digital music library. It focuses on batch processing, metadata management, loudness normalization, integrity verification, and efficient backups.

## Library Naming Convention (Recommended for Best Results)

This guide works best — and its verification layers work *only* at full strength — when the library follows this layout:

```
Parent/
└── Artist/
    └── YYYY Album Name/
        ├── 01 Track One.flac
        ├── 02 Track Two.flac
        └── ...
```

* Album folders start with a 4-digit year then a space then the album name: `2004 Album Name`.
* Track files are named with a zero-padded track number then the title, no dash: `01 Track One.ext` (padded `01`–`09`, then `10` and up).
* A hyphenated variant (`NN - Title`) is accepted by the tag tools; the padded number is what matters.

Why it matters:

* **The 13e failsafe verifies embedded tags against folder and file names.** Artist comes from the parent folder, Album Year/Album name from the album folder, Track number/Title from the filename. The filesystem becomes the reference schema for what each file must be tagged.
* **Tags can be rebuilt from names alone** if metadata is ever lost or corrupted — losslessly, with audio untouched.
* **Zero-padded track numbers sort correctly** in moOde, mpd, SHA-512 manifests, and file managers.
* **The checksum and recertification guides use the same layout**, so album- and artist-level manifests stay consistent with what moOde displays.

Steps 1–8 (integrity, deduplication, rebuild, ReplayGain) process any folder layout. But the confirmation layers — 13e, Write Tags, checksum tools — are built around this convention.

## Included Tools & Guides

* **FLAC Cleanup & Validation:** Step-by-step guidance and command workflows for verifying file integrity, stripping unnecessary tags, standardizing metadata, and applying proper checksums.
* **Audio Processing Workflows:** Efficient use of command-line utilities for audio management.
* **Permission Check:** Automated checks and guidance to ensure your user has the correct read/write ownership and access to the library directory before executing mass batch operations.

## Prerequisites

Ensure you have the following command-line utilities installed on your system:

* `ffmpeg`
* `flac`
* `loudgain`
* `rsync`

**Directory Permissions:** Before running any batch processing scripts, verify that your current user has full read and write permissions to the target library directory to prevent "Permission Denied" failures during execution.

## Recommended Workflow

A four part series to clean, verify, and lockdown securely the integrity of an audio file library. 

1. linux-audio-moode-prep: https://github.com/TerrapinATL/linux-audio-moode-prep

2. linux-audio-sha512-checksums: https://github.com/TerrapinATL/linux-audio-sha512-checksums

3. linux-os-nemo-sha512-shortcut:https://github.com/TerrapinATL/linux-os-nemo-sha512-shortcut

4. linux-audio-folder-recertification: https://github.com/TerrapinATL/linux-audio-folder-recertification

## Disclaimer

This file was created as a mix of AI generated content, user input, and user editing. It was a cooperative effort between Claude, Gemini, ChatGPT, and user.

## IMPORTANT

Your Original Library should be treated as immutable.

You should only work on a COPY of your Original Library when processing these scripts. The workflow is designed around creating a validated secondary copy, testing the results, and only then promoting that copy to become a replacement.

Before promotion, files should be cleaned, verified with flac -t, and protected with two layers of SHA-512 checksums.

The purpose is to ensure you have a verifiable library that can be copied, backed up, and restored repeatedly while still matching the validated cleaned copy.

## Recommendations

For those not familiar with using a Raspberry Pi as a Music Server, I have been using a Raspberry Pi 5 with Moode Audio and it has made a wonderful audio file server for my studio recordings. Moode is Freeware and is an amazing well constructed program optimized for a Raspberry Pi. It turns any web browser into a media interface to control playback and can be optimized to personal preferences. My setup is like a radio station with zero commercials, endless music.

I had no need of a DAC chip, since one is integrated into my receiver. I do recommend one if your receiver/amp is not so equipped. More info on that is in the Moode Forum.

-- Moode Audio

Moode Software: https://moodeaudio.org/

Moode Forum: https://moodeaudio.org/forum/member.php?action=register&referrer=13448

-- My hardware set-up:

Raspberry Pi 5 motherboard: https://vilros.com/products/raspberry-pi-5?variant=40065551269982

Vilros Pi 5 Case: https://vilros.com/products/vilros-raspberry-pi-5-compatible-aluminum-alloy-case-with-passive-active-cooling-insert

Vilros 5v/5a Power Supply: https://vilros.com/products/vilros-27w-5v-5a-raspberry-pi-5-compatible-usb-c-power-supply

SanDisk 128gb MicroSD Card: https://www.amazon.com/SANDISK-128GB-Extreme-microSD-UHS-I/dp/B0G8LLXFJH/

HDMI Cable: https://www.amazon.com/UGREEN-Certified-Aluminum-Compatible-Blu-ray/dp/B0CFFFSFFN


Note: These are recommendations based upon my experiences. I receive zero compensation for them. Please consider them a guide, nothing more. Your configuration may vary. For me, this worked quite well. 



