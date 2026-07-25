# Linux Audio Utilities

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

### Installation Example (Debian/Ubuntu/Linux Mint)
```bash
sudo apt update
sudo apt install ffmpeg flac rsync
