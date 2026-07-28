# [Linux Audio Utilities](https://github.com/TerrapinATL/linux.audio.flac-clean-up/edit/main/Readme.md)

A collection of lightweight command-line workflows and utilities designed for validating, cleaning, managing, and backing up local FLAC music libraries on Linux.

## Overview

This repository contains tools and documentation to help maintain a pristine, standardized digital music library. It focuses on batch processing, metadata management, loudness normalization, integrity verification, and efficient backups.

## Included Tools & Guides

* **FLAC Cleanup & Validation:** Step-by-step guidance and command workflows for verifying file integrity, stripping unnecessary tags, standardizing metadata, and applying proper checksums.
* **Audio Processing Workflows:** Efficient use of command-line utilities for audio management.

## Prerequisites

Ensure you have the following command-line utilities installed on your system:

* `ffmpeg`
* `flac`
* `loudgain`
* `rsync`

## Recommended Workflow

A three part series to clean, verify, and lockdown securely the integrity of an audio file library. 

1. linux.audio.flac-clean-up: https://github.com/TerrapinATL/linux.audio.flac-clean-up

2. linux.audio.sha512-checksums: https://github.com/TerrapinATL/linux.audio.sha512-checksums

3. linux.os.nemo.sha512-shortcut: https://github.com/TerrapinATL/linux.os.nemo.sha512-shortcut

## Disclaimer

This file was created as a mix of AI generated content, user input, and user editing. It was a cooperative effort between Claude, Gemini, ChatGPT, and user.

## IMPORTANT

Your Original Library should be treated as immutable.

You should only work on a COPY of your Original Library when processing these scripts. The workflow is designed around creating a validated secondary copy, testing the results, and only then promoting that copy to become a replacement.

Before promotion, files should be cleaned, verified with flac -t, and protected with two layers of SHA-512 checksums.

The purpose is to ensure you have a verifiable library that can be copied, backed up, and restored repeatedly while still matching the validated cleaned copy.
