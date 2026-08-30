### linux-audio-moode-cleanup-guide

**Version: v26** — Current version; supersedes v25.

---

01. Introduction

---

This document is a technical guide and automated script workflow for auditing, repairing, and standardizing Moode Audio–compatible music libraries on Linux systems.

It covers FLAC, MP3, M4A/AAC, WavPack, OGG, and Opus formats. Not all steps support all formats; see the Format Support Status table in Section 04 for details.

Note: Always work off a backup library copy until a clean copy has been secured by sha512 checksums.

---

02. Requirements

---

To successfully execute the scripts and workflows in this guide, your system must have the following command-line tools installed and available in your shell's PATH:

* flac / metaflac – Required for FLAC integrity testing and Vorbis comment metadata handling.

* ffmpeg – Required for multi-format integrity testing (Step 1), container rebuilding, and artwork embedding across Moode-compatible formats.

* loudgain – Required for calculating and writing ReplayGain metadata across FLAC, MP3, M4A, OGG, Opus, MP4, WavPack, APE, and SPX.

* eyeD3 – Required for MP3 metadata deduplication (Step 2C.3). The `eyed3` Python module must be importable (eyeD3 0.9+ renamed the import to lowercase `eyed3`); if it is missing, install it with `sudo apt install python3-eyed3` (Debian family; disabled PEP 668) or `python3 -m pip install --user eyeD3` (where allowed; add `--break-system-packages` only as a last resort). The module is not always installed alongside the `eyeD3` command.

* vorbiscomment – Required for OGG Vorbis metadata deduplication (Step 2C.5).

* opustags – Required for Opus metadata deduplication (Step 2C.5).

* AtomicParsley – Required for M4A/MP4 metadata review-flagging (Step 2C.4).

* wvtag – Required for WavPack metadata review-flagging (Step 2C.4).

* python3 – Required for MP3 metadata deduplication (Step 2C.3). The `eyed3` Python module must be importable; see the `eyeD3` entry above if it is missing.

* Core Utilities – Standard GNU core utilities (find, sort, awk, grep, wc, basename, dirname, mktemp).

-- Software Preflight

Run the preflight diagnostic below BEFORE starting any cleanup step. It verifies that every tool and Python module the guide requires is installed and importable, and fails loudly with install hints if anything is missing. This prevents silent mid-run failures and dead ends. It writes a diagnostic report to `~/.logs/linux-audio-moode-cleanup-guide/preflight.log` and does not modify any audio files.

--- Bash Script Preflight Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ---------------------------------------------------------------------------
# Software Preflight - Verify all required tools before any cleanup step runs
# ---------------------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
PREFLIGHT_LOG="$LOG_ROOT/preflight.log"

: > "$PREFLIGHT_LOG"

echo "================================================" | tee -a "$PREFLIGHT_LOG"
echo "Software Preflight - All Required Tools"         | tee -a "$PREFLIGHT_LOG"
echo "================================================" | tee -a "$PREFLIGHT_LOG"

missing=0
pass=0

check_cmd() {
    local tool="$1"
    if command -v "$tool" >/dev/null 2>&1; then
        printf "%-22s : OK\n" "$tool" >> "$PREFLIGHT_LOG"
        pass=$((pass + 1))
    else
        printf "%-22s : MISSING\n" "$tool" >> "$PREFLIGHT_LOG"
        missing=$((missing + 1))
    fi
}

# 1. Steps 1-8 command-line tools
for tool in flac metaflac ffmpeg ffprobe loudgain python3 eyeD3 vorbiscomment opustags AtomicParsley wvtag; do
    check_cmd "$tool"
done

# 2. Core utilities assumed present on any Linux system
for tool in find sort awk grep wc basename dirname mktemp sed tr cmp tee; do
    check_cmd "$tool"
done

# 3. Python module check: eyeD3 (required by Step 2C.3 for MP3 deduplication)
if command -v python3 >/dev/null 2>&1; then
    if python3 -c "import eyed3" >/dev/null 2>&1; then
        printf "%-22s : OK\n" "eyed3-python-module" >> "$PREFLIGHT_LOG"
        pass=$((pass + 1))
    else
        printf "%-22s : MISSING\n" "eyed3-python-module" >> "$PREFLIGHT_LOG"
        printf "%-22s : install with: sudo apt install python3-eyed3  (or: python3 -m pip install --user eyeD3)\n" "[hint]" >> "$PREFLIGHT_LOG"
        missing=$((missing + 1))
    fi
else
    printf "%-22s : MISSING (python3 not found; install the python3 package)\n" "eyed3-python-module" >> "$PREFLIGHT_LOG"
    missing=$((missing + 1))
fi

echo "----------------------------------------" | tee -a "$PREFLIGHT_LOG"
echo "Pass: $pass   Missing: $missing"         | tee -a "$PREFLIGHT_LOG"

if [ "$missing" -eq 0 ]; then
    echo "RESULT: ALL SOFTWARE PRESENT - ready to run." | tee -a "$PREFLIGHT_LOG"
    echo "----------------------------------------" | tee -a "$PREFLIGHT_LOG"
    echo "Preflight - Software Check" | tee -a "$PREFLIGHT_LOG"
    echo "----------------------------------------" | tee -a "$PREFLIGHT_LOG"
    exit 0
else
    echo "RESULT: $missing item(s) missing. Install them and re-run preflight." | tee -a "$PREFLIGHT_LOG"
    echo "----------------------------------------" | tee -a "$PREFLIGHT_LOG"
    echo "Preflight - Software Check" | tee -a "$PREFLIGHT_LOG"
    echo "----------------------------------------" | tee -a "$PREFLIGHT_LOG"
    exit 1
fi
```

-- Multi-Disc Album Organization

Scripts determine Artist and Album from the directory hierarchy. Multi-disc albums must not be nested in sub-directories (e.g., Library/Artist/Album/Disc 1/).

* Method 1: Combine all tracks into a single album directory with track numbering that accounts for discs (101, 102, 201, 202, etc.). Loudgain calculates a single cohesive volume level for the entire release.

* Method 2: Treat each disc as a separate album directory (e.g., Album (Disc 1), Album (Disc 2)). Loudgain calculates volume per disc independently.

---

02b. Library Naming Convention (Recommended for Best Results)

---

This guide is designed around the following layout:

```
Parent/
└── Artist/
    └── YYYY Album Name/
        ├── 01 - Track One.flac
        ├── 02 - Track Two.flac
        └── ...
```

* Album folders start with a 4-digit year followed by a space and the album name: `2004 Album Name`.
* Track files are named with a zero-padded track number, then the title: `01 - Track One.ext` (padded `01` through `09`, then `10` and up).
* Armlike variants (`NN Title` without a hyphen) are also accepted by the tag tools; the padded number is what matters.

This layout is the single strongest confirmation layer in the suite:

* **13e (Verify Tags Against Filenames) derives the expected tags from the folder and file names** — Artist (parent folder), Album Year/Album name (album folder), and Track number/Title (filename). If the tags disagree with the names, they are flagged as mismatches, so the filesystem itself becomes the reference schema for what a file must be tagged.
* **Tags can be rebuilt from names alone.** If embedded metadata is ever lost or corrupted, the "Write Tags from Folder/File Names" Nemo action (and the same logic used manually) can restore Artist/Album/Year/Title/TrackNumber losslessly — never re-encoding, so the audio is untouched.
* **Zero-padded track numbers sort correctly** everywhere the library is used: moOde, mpd, the SHA-512 manifests, and any file manager.
* **Checksum and recertification tools use the same layout**, so album-level and artist-level manifests stay consistent with what moOde displays.

The cleanup pipeline itself (Steps 1–8) processes any folder layout: integrity testing, metadata deduplication, container rebuild, and ReplayGain do not require this naming pattern. But the verification and failsafe layers — 13e, the Write Tags action, and the checksum tools — are built around it. Following the layout gives a clean, confirmable result; deviating from it means some of those checks cannot run at full strength.

---

03. Design Philosophy

---

This guide follows four core principles:

* Non-Destructive: Audio streams never re-encoded; zero generational loss.
* Explicit Verification: Every modification bracketed by validation tests.
* Traceable Execution: Clean logs and separate success/failure lists for auditability.
* Format-Aware: Native tooling per format; no single FLAC-centric approach forced across all types.

---

04. Workflow Overview

---

The cleanup process follows: Verify → Edit → Verify → Edit → Verify

Each operation begins with understanding the current state, then validates results before continuing.

-- Sequential Pipeline:

0. Software Preflight (see Requirements; verifies all tools before anything runs)
1. Initial Integrity Test
2. Deduplicate Metadata
3. Container Rebuild
4. Post-Rebuild Verification
5. ReplayGain Restoration
6. Final Integrity Validation
7. Remove Loose Files
8. Final Integrity Test

Optional procedures (Steps 13a–13d) handle edge cases. Files failing all steps should be replaced.

-- Format Support Status

|------|----------------------------|----------------------------------------------|---------------------------------------|
| Step | Step Name                  | Formats                                      | Status                                |
|------|----------------------------|----------------------------------------------|---------------------------------------|
| 0    | Software Preflight         | All required tools + Python module             | Prerequisite                           |
| 1    | Initial Integrity Test     | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF, MP4, APE, WV, SPX | Complete                              |
| 2    | Deduplicate Metadata       | FLAC/MP3/M4A/MP4/WV/OGG/OPUS; other formats   | Complete for supported formats        |
|      |                            | detected, counted, reported, not modified    |                                       |
| 3    | Rebuild Containers         | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF    | Complete                              |
| 4    | Post-Rebuild Verification  | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF, MP4, APE, WV, SPX | Complete                              |
| 5    | ReplayGain Restoration     | FLAC/MP3/M4A/OGG/Opus/MP4/APE/WV/SPX          | Complete                              |
| 6    | Final Integrity Validation | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF, MP4, APE, WV, SPX | Complete                              |
| 7    | Remove Loose Files         | Format-agnostic                              | Complete                              |
| 8    | Final Integrity Test       | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF, MP4, APE, WV, SPX | Complete                              |
| 13a  | Strip Problematic Metadata | FLAC only                                    | Complete — FLAC-only by design        |
| 13b  | Normalize Album Artwork    | FLAC only                                    | Superseded by 13c                     |
| 13c  | Artwork Embeds             | FLAC/MP3/M4A/MP4 (OGG/Opus/AIFF/APE/DSF skip) | Complete                              |
| 13d  | Deep Repair (Re-encode)    | FLAC only                                    | Complete — FLAC-only by design        |
| 14   | Generate Checksums         | Format-agnostic                              | See separate SHA-512 repo             |
|------|----------------------------|----------------------------------------------|---------------------------------------|


\-------------------------------------------------------------------

05. Step 1 – Initial Integrity Test

---

-- Purpose

The initial integrity test establishes the baseline condition of the audio library before any modifications are made.

This step identifies files that already contain integrity problems so they can be tracked throughout the cleanup process. Running this test first prevents confusion later by distinguishing pre-existing issues from problems that may occur during the repair process.

-- What It Does

This step:

* Scans the selected library location recursively for audio files (FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF, MP4, APE, WV, and SPX).
* Tests each file using the method appropriate to its format — `flac -t` for FLAC, and an `ffmpeg` decode-to-null check for every other supported format.
* Records successful tests and failures separately.
* Prints the error log to the screen at the end of the run, if any files failed.
* Creates a reference point for comparison after later cleanup steps.
* Skips any folder named `Ignore` at any depth (case-insensitive). Quarantine folders like these are excluded from every step of this guide so intentionally-set-aside files never add noise or get rewritten.

No files are modified during this step.

\---------------------------------------------------------------------------------------

--- Bash Script Step 1 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 1 – Multi-Format Audio Integrity Test
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step01"

mkdir -p "$LOG_ROOT"

# Software Preflight: fail loudly if a required tool is missing
for tool in flac ffmpeg; do
    if ! command -v "$tool" >/dev/null 2>&1; then
        echo "ERROR: $tool is not installed. Install it and re-run (see Requirements)." >&2
        exit 1
    fi
done

# 1. Define Log Files (Five-File Standard)
RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

# 2. CLEANUP: Delete this step's own logs from any previous run
rm -f "$RUN_LOG" "$OKS_LOG" "$FAILS_LOG" "$ERRORS_LOG" "$SUMMARY_LOG"

# 3. Initialize Empty Log Files
touch "$RUN_LOG" "$OKS_LOG" "$FAILS_LOG" "$ERRORS_LOG" "$SUMMARY_LOG"

# 3b. Optional: Pre-warm the filesystem cache (read-only)
# Asking at the start of Step 1 lets later flac -t / ffmpeg passes
# (Step 1, 4, 6, 8) read from RAM instead of a mechanical HDD.
if [ -t 0 ]; then
    printf '\nPre-warm the library into the page cache before testing? [y/N] '
    read -r WARM
    case "$WARM" in
        y|Y|yes|YES)
            PREWARM_LOG="$LOG_ROOT/step01-prewarm.log"
            : > "$PREWARM_LOG"
            echo "=== Step 1 Cache Pre-Warm (read-only) ===" | tee -a "$PREWARM_LOG"
            echo "Started: $(date)" | tee -a "$PREWARM_LOG"
            start=$(date +%s)
            # Single sequential read pass - friendliest for a mechanical HDD
            find "$PWD" -type f \
                ! -ipath '*/Ignore/*' \
                ! -iname '*.prerepair*' \
                ! -iname '*.fixed.*' \
                ! -iname '*.reencode.*' \
                ! -iname '*.sha512sums.txt' \
                \( \
                    -iname "*.flac" -o -iname "*.mp3" -o -iname "*.m4a" -o \
                    -iname "*.ogg"  -o -iname "*.opus" -o -iname "*.wav" -o \
                    -iname "*.aiff" -o -iname "*.aif"  -o -iname "*.mp4" -o \
                    -iname "*.ape"  -o -iname "*.wv"   -o -iname "*.spx" \
                \) -print0 \
                | xargs -0 -r cat >/dev/null 2>>"$LOG_ROOT/step01-prewarm-errors.log"
            end=$(date +%s)
            echo "SUMMARY: pre-warm finished in $((end-start))s." | tee -a "$PREWARM_LOG"
            ;;
        *)
            echo "Skipping cache pre-warm."
            ;;
    esac
fi

# 4. File Discovery
mapfile -d '' files < <(
    find "$PWD" -type f \
        ! -ipath '*/Ignore/*' \
        ! -iname "*.prerepair*" \
        ! -iname "*.fixed.*" \
        ! -iname "*.reencode.*" \
        \( \
            -iname "*.flac" -o \
            -iname "*.mp3"  -o \
            -iname "*.m4a"  -o \
            -iname "*.ogg"  -o \
            -iname "*.opus" -o \
            -iname "*.wav"  -o \
            -iname "*.aiff" -o \
            -iname "*.aif"  -o \
            -iname "*.mp4"  -o \
            -iname "*.ape"  -o \
            -iname "*.wv"   -o \
            -iname "*.spx" \
        \) -print0 2>>"$LOG_ROOT/${STEP}-errors.log" | sort -z
)

total=${#files[@]}
i=0
last_dir=""

for f in "${files[@]}"; do

    ((i++))
    label="${f#"$PWD"/}"
    current_dir="$(dirname "$label")"

    # Insert a blank line on terminal screen when moving to a new folder/album
    if [[ -n "$last_dir" && "$current_dir" != "$last_dir" ]]; then
        echo ""
    fi
    last_dir="$current_dir"

    case "${f,,}" in
        *.flac)
            err=$(flac -s -t "$f" 2>&1)
            rc=$?
            ;;
        *)
            err=$(ffmpeg -nostdin -v error -i "$f" -f null - 2>&1)
            rc=$?
            ;;
    esac

    if [ $rc -eq 0 ]; then
        out_msg="OK   [$i/$total] $label"
        echo "$out_msg"
        echo "$out_msg" >> "$RUN_LOG"
        echo "$out_msg" >> "$OKS_LOG"
    else
        flat=$(printf '%s\n' "$err" | tr '\r\n' ' ' | tr -s ' ')
        out_msg="FAIL [$i/$total] $label"
        echo "$out_msg"
        echo "$out_msg" >> "$RUN_LOG"
        echo "$out_msg" >> "$FAILS_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
    fi

done

# 5. Count Results
ok_count=$(grep -a "^OK" "$RUN_LOG" 2>/dev/null | wc -l)
fail_count=$(grep -a "^FAIL" "$RUN_LOG" 2>/dev/null | wc -l)

# 6. Generate Summary Log
{
echo "Step 1 Summary"
echo "=============="
echo
echo "Step       : $STEP"
echo "Run Date   : $(date)"
echo
echo "Processed  : $total"
echo "Passed     : $ok_count"
echo "Failed     : $fail_count"
} > "$SUMMARY_LOG"

# 7. Terminal Output
echo
if [ -s "$ERRORS_LOG" ]; then
    echo "----------------------------------------"
    echo "Error Summary"
    echo "----------------------------------------"
    awk '
    /LOST_SYNC/ {
        idx = index($0, " :: ")
        if (idx > 0) {
            temp = substr($0, 1, idx - 1)
            pos = index(temp, "): ") + 3
            path = substr(temp, pos)
            lost_sync[path] = 1
        }
    }
    /END_OF_STREAM/ && !/LOST_SYNC/ {
        idx = index($0, " :: ")
        if (idx > 0) {
            temp = substr($0, 1, idx - 1)
            pos = index(temp, "): ") + 3
            path = substr(temp, pos)
            eos[path] = 1
        }
    }
    END {
        if (length(lost_sync) > 0) {
            print "LOST_SYNC"
            print "----------"
            for (p in lost_sync) print p | "sort"
            close("sort")
        }
        if (length(eos) > 0) {
            if (length(lost_sync) > 0) print ""
            print "END_OF_STREAM"
            print "----------"
            for (p in eos) print p | "sort"
            close("sort")
        }
    }
    ' "$ERRORS_LOG"
fi

echo
echo "----------------------------------------"
echo "Processed: $total  Passed: $ok_count  Failed: $fail_count"
echo "----------------------------------------"
echo "Step 1 – Initial Integrity Test"
echo "----------------------------------------"

```
--- Bash Script Step 1 End ---

\---------------------------------------------------------------------------------------

--- Bash Script Cat Step 1 Start ---
```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"

cat "$LOG_ROOT/step01-run.log"
cat "$LOG_ROOT/step01-oks.log"
cat "$LOG_ROOT/step01-fails.log"
cat "$LOG_ROOT/step01-errors.log"
cat "$LOG_ROOT/step01-summary.log"

```
--- Bash Script Cat Step 1 End ---

---

06. Step 2 — Metadata Deduplication

---

-- Purpose

* Status Note: Step 2A-2E scripts are fully functional for FLAC, MP3, M4A/MP4, WavPack (WV), OGG/OGA, and Opus. FLAC, MP3, OGG, and Opus are auto-fixed with their native tools (`metaflac`, `eyeD3`, `vorbiscomment`, `opustags`); M4A/MP4 and WavPack are review-flagged with `AtomicParsley` and `wvtag`. All other formats (AIFF, AIF, AIFC, WAV, DSF, DFF, raw AAC, APE, DSD, MPC, SPX) are detected and counted, but are not auto-fixed by this step.

This step identifies and removes confirmed duplicate metadata entries from audio files in the working copy.

Step 2 supports the audio formats most commonly used by Moode/MPD: FLAC, MP3, M4A/MP4, WavPack, OGG Vorbis, and Opus. Step 2A discovers every audio format in the library and Step 2B classifies each; Step 2C then uses the native command-line tool built for each processed format's tag container rather than a single generic library:

|-----------------|--------------------|--------------------|
| Format          | Container          | Tool               |
|-----------------|--------------------|--------------------|
| FLAC            | Vorbis comment     | `metaflac`         |
| MP3             | ID3v2              | `eyeD3`            |
| M4A / MP4       | iTunes-style atoms | `AtomicParsley`    |
| WavPack         | APEv2              | `wvtag`            |
| OGG Vorbis      | Vorbis comment     | `vorbiscomment`    |
| Opus            | Vorbis comment     | `opustags`         |
|-----------------|--------------------|--------------------|

Not every format gets the same treatment. Harness testing against fabricated duplicate-tag fixtures (byte-identical repeated frames/atoms/items, not just repeated CLI writes) showed that the tools split into two groups:

* FLAC, MP3, OGG, and Opus are `safely detected and auto-fixed`. FLAC/OGG/Opus share the Vorbis comment model, so duplicates are resolved by exporting all tags, dropping repeats that share the same key (case-insensitive) and the same value, and re-importing the deduplicated set. MP3 duplicates are resolved by removing repeated COMMENT and user-text (TXXX) frames one at a time down to a single copy — the two frame types most likely to pick up redundant entries from repeated ripping/tagging passes over the years.

* M4A/MP4 and WavPack can be `detected and flagged for manual review` if duplicates are found, as their tools (`AtomicParsley`, `wvtag`) do not support safe auto-fixing.

MP3 dedup is intentionally limited to COMMENT and TXXX frames. Standard singular frames (title, artist, album, and similar) can also carry genuine duplicate frames at the byte level, but `eyeD3`'s own display silently collapses them to a single value — there is no safe way to detect or remove them through the native CLI alone without falling back to raw byte scanning, which this pipeline does not do.

The operation is strictly non-destructive:

* Audio streams are never re-encoded.
* Audio data is never altered.
* Unique metadata is preserved.
* Only confirmed duplicate metadata entries are removed.
* Files that cannot be safely processed are left unchanged and logged.
* Every modification is verified before Step 2 is considered successful.

The working copy is the only copy processed. The Master Library is never modified.

\---------------------------------------------------------------------------------------

-- Step 2 Workflow

* Step 2 is divided into small, purpose-driven scripts:

* Step 2A — File Discovery
  Locate all candidate audio files and create the clean input list.

* Step 2B — Format Assessment
  Classify each file as dedupe-capable, review-capable, or unsupported.

* Step 2C — Metadata Deduplication
  Remove confirmed duplicate metadata entries using the native tool for each supported format; flag review-capable files that contain duplicates instead of auto-fixing them.

* Step 2D — Verification
  Verify the files modified by Step 2C and confirm that the intended cleanup occurred.

* Step 2E — Summary
  Produce the final Step 2 results and status.

Each script starts by cleaning its own previous-run artifacts, then performs its assigned task and leaves a clean handoff for the next script.

Verify → Edit → Verify

No Step 2 script will proceed on the assumption that a previous run completed successfully.

-- Logging

Every Step 2 script writes to a common log root, using a fixed five-file set per script:

```
$HOME/.logs/linux-audio-moode-cleanup-guide/step02<letter>-run.log        full chronological transcript
$HOME/.logs/linux-audio-moode-cleanup-guide/step02<letter>-oks.log        successful outcomes only
$HOME/.logs/linux-audio-moode-cleanup-guide/step02<letter>-fails.log      per-file failures and review flags
$HOME/.logs/linux-audio-moode-cleanup-guide/step02<letter>-errors.log     run-level errors (missing tools, missing input lists)
$HOME/.logs/linux-audio-moode-cleanup-guide/step02<letter>-summary.log    final tallies and status for that script
```

Every per-file line is written with `tee -a` at the point it happens, to `run.log` and to whichever of `oks.log` / `fails.log` applies. Files needing manual review (M4A/MP4 and WavPack duplicates) are logged to `fails.log` with a `REVIEW` tag rather than `FAIL`, so they surface as a punch list without being confused with genuine processing failures.

\ ---------------------------------------------------------------------------------------

## Step 2A — File Discovery

Locate all candidate audio files and create the clean input list.

--- Bash Script Step 2A Start ---

```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 2A — File Discovery
# ------------------------------------------------------------

set -u

TARGET_DIR="${1:-.}"
LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step02a"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

: > "$RUN_LOG"
: > "$OKS_LOG"
: > "$FAILS_LOG"
: > "$ERRORS_LOG"
: > "$SUMMARY_LOG"

CANDIDATE_LIST="$LOG_ROOT/step02-candidates.txt"
: > "$CANDIDATE_LIST"

if [ ! -d "$TARGET_DIR" ]; then
    echo "ERROR: target directory not found :: $TARGET_DIR" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    echo "----------------------------------------"
    echo "Step 2A - File Discovery"
    echo "----------------------------------------"
    
    # Interactive view for directory target errors
    if [ -t 1 ] && [ -s "$ERRORS_LOG" ]; then
        echo
        echo "=================================================="
        echo " ERRORS DETECTED — Press ENTER to view error log"
        echo " (Use arrow keys to scroll, press 'q' to exit)"
        echo "=================================================="
        read -r
        less -R "$ERRORS_LOG"
    fi
    exit 1
fi

i=0

mapfile -d '' files < <(find "$TARGET_DIR" -type f ! -ipath '*/Ignore/*' \( \
    -iname '*.flac' -o -iname '*.mp3' -o -iname '*.m4a' -o -iname '*.mp4' -o \
    -iname '*.wv' -o -iname '*.ogg' -o -iname '*.opus' -o -iname '*.aac' -o \
    -iname '*.wav' -o -iname '*.aiff' -o -iname '*.aif' -o -iname '*.aifc' -o \
    -iname '*.ape' -o -iname '*.mpc' -o -iname '*.spx' \) -print0)

found=${#files[@]}
: > "$CANDIDATE_LIST"

for file in "${files[@]}"; do
    i=$((i + 1))
    printf '%s\0' "$file" >> "$CANDIDATE_LIST"
    echo "OK   [$i/$found] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
done

echo "TOTAL_CANDIDATES=$found" | tee -a "$SUMMARY_LOG" >/dev/null
echo "STATUS=OK" | tee -a "$SUMMARY_LOG" >/dev/null


# Interactive error inspector
if [ -t 1 ] && [ -s "$ERRORS_LOG" ]; then
    echo
    echo "=================================================="
    echo " ERRORS DETECTED — Press ENTER to view error log"
    echo " (Use arrow keys to scroll, press 'q' to exit)"
    echo "=================================================="
    read -r
    less -R "$ERRORS_LOG"
fi

echo
echo "----------------------------------------"
echo "Candidates found : $found"
echo "----------------------------------------"
echo "Step 2A - File Discovery"
echo "----------------------------------------"

```

--- Bash Script Step 2A End ---

\ ---------------------------------------------------------------------------------------

## Step 2B — Format Assessment

Classify each file as dedupe-capable, review-capable, or unsupported.

Dedupe-capable: FLAC, MP3, OGG, Opus — auto-fixed in Step 2C.
Review-capable: M4A, MP4, WavPack — flagged in Step 2C if duplicates are found, never auto-fixed.
Unsupported: raw AAC, WAV, AIFF, AIF, AIFC, APE, MPC, SPX, and anything else Step 2A found.

--- Bash Script Step 2B Start ---

```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 2B — Format Assessment
# ------------------------------------------------------------

set -u
LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step02b"
mkdir -p "$LOG_ROOT"
RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"
: > "$RUN_LOG"
: > "$OKS_LOG"
: > "$FAILS_LOG"
: > "$ERRORS_LOG"
: > "$SUMMARY_LOG"
CANDIDATE_LIST="$LOG_ROOT/step02-candidates.txt"
if [ ! -s "$CANDIDATE_LIST" ]; then
    echo "ERROR: candidate list empty or missing :: $CANDIDATE_LIST" >> "$ERRORS_LOG"
    echo "STATUS=ERROR" >> "$SUMMARY_LOG"
    echo "ERROR: candidate list empty or missing :: $CANDIDATE_LIST"
    echo "Run Step 2A first."
    exit 1
fi
declare -A format_count
total_count=$(grep -a -o $'\0' "$CANDIDATE_LIST" 2>/dev/null | wc -l)
dedupe_count=0
review_count=0
unsupported_count=0
i=0

while IFS= read -r -d '' file; do
    if [ ! -r "$file" ]; then
        echo "ERROR: unreadable file :: $file" >> "$ERRORS_LOG"
        continue
    fi
    i=$((i + 1))
    fname=$(basename "$file")
    ext="${fname##*.}"
    ext_lc="$(printf '%s' "$ext" | tr '[:upper:]' '[:lower:]')"
    echo "$ext_lc [FOUND] :: $file" >> "$RUN_LOG"
    format_count[$ext_lc]=$((${format_count[$ext_lc]:-0} + 1))

    case "$ext_lc" in
        flac|mp3|ogg|opus)
            echo "OK   [$i/$total_count] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
            dedupe_count=$((dedupe_count + 1))
            ;;
        m4a|mp4|wv)
            echo "REVIEW [$i/$total_count] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
            review_count=$((review_count + 1))
            ;;
        *)
            unsupported_count=$((unsupported_count + 1))
            ;;
    esac
done < "$CANDIDATE_LIST"
echo "STATUS=OK" >> "$SUMMARY_LOG"
for fmt in "${!format_count[@]}"; do
    echo "$fmt=${format_count[$fmt]}" >> "$SUMMARY_LOG"
done
echo "TOTAL=$total_count" >> "$SUMMARY_LOG"
echo "DEDUPE_CAPABLE=$dedupe_count" >> "$SUMMARY_LOG"
echo "REVIEW_CAPABLE=$review_count" >> "$SUMMARY_LOG"
echo "UNSUPPORTED=$unsupported_count" >> "$SUMMARY_LOG"

echo "Format Breakdown:"
for fmt in $(printf '%s\n' "${!format_count[@]}" | sort); do
    printf "  %-18s : %d\n" "$(printf '%s' "$fmt" | tr '[:lower:]' '[:upper:]')" "${format_count[$fmt]}"
done

echo
echo "----------------------------------------"
echo "Total: $total_count  Dedupe: $dedupe_count  Review: $review_count  Unsupported: $unsupported_count"
echo "----------------------------------------"
echo "Step 2B - Format Assessment"
echo "----------------------------------------"

```
--- Bash Script Step 2B End ---
## Step 2C — Metadata Deduplication

Removes confirmed duplicate metadata entries using the native tool for each supported format and flags review-capable files that contain duplicates instead of auto-fixing them.

Formats are handled in the three groups established by Step 2B:

* Dedupe-capable (auto-fixed): FLAC, MP3, OGG, Opus.
* Review-capable (flagged for manual review, never auto-fixed): M4A/MP4, WavPack.
* Unsupported (only counted, never processed by this step): raw AAC, WAV, AIFF, AIF, AIFC, APE, MPC, SPX.

The operation is strictly non-destructive:

* Audio streams are never re-encoded.
* Audio data is never altered.
* Unique metadata is preserved.
* Only confirmed duplicate metadata entries are removed.
* Files that cannot be safely processed are left unchanged and logged.

\---------------------------------------------------------------------------------------

## Step 2C.1 — Initialize & Clean Logs

Verifies the Step 2A candidate list exists, clears this step's previous-run artifacts, and records the starting totals.

--- Bash Script Step 2C.1 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 2C.1 — Initialize & Clean Logs
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step02c"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

: > "$RUN_LOG"
: > "$OKS_LOG"
: > "$FAILS_LOG"
: > "$ERRORS_LOG"
: > "$SUMMARY_LOG"

CANDIDATE_LIST="$LOG_ROOT/step02-candidates.txt"

if [ ! -s "$CANDIDATE_LIST" ]; then
    echo "ERROR: candidate list empty or missing :: $CANDIDATE_LIST" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    echo "Run Step 2A first."
    echo "----------------------------------------"
    echo "Step 2C.1 - Initialize & Clean Logs"
    echo "----------------------------------------"
    exit 1
fi

total=$(tr -cd '\000' < "$CANDIDATE_LIST" | wc -c)

echo "TOTAL_CANDIDATES=$total" | tee -a "$SUMMARY_LOG" >/dev/null
echo "STATUS=OK" | tee -a "$SUMMARY_LOG" >/dev/null

echo
echo "----------------------------------------"
echo "Candidates found : $total"
echo "----------------------------------------"
echo "Step 2C.1 - Initialize & Clean Logs"
echo "----------------------------------------"

```
--- Bash Script Step 2C.1 End ---

\---------------------------------------------------------------------------------------

## Step 2C.2 — FLAC Auto-Fix (Vorbis Comment Deduplication)

Exports all Vorbis comments with `metaflac`, drops duplicate entries that share the same key (case-insensitive) and the same value, and re-imports the deduplicated set. Audio data and non-comment metadata blocks (artwork, seek tables, padding) are left untouched. Every file is verified with `flac -t` before it is counted as OK.

--- Bash Script Step 2C.2 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 2C.2 — FLAC Auto-Fix
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step02c"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

CANDIDATE_LIST="$LOG_ROOT/step02-candidates.txt"
WORK_DIR="$LOG_ROOT/step02c-work"
mkdir -p "$WORK_DIR"

# First pass: count FLAC candidates for progress reporting
count_total=0
while IFS= read -r -d '' file; do
    [[ "${file,,}" == *.flac ]] && count_total=$((count_total + 1))
done < "$CANDIDATE_LIST"

if [ "$count_total" -eq 0 ]; then
    echo "STEP02C_FLAC_OK=0" >> "$SUMMARY_LOG"
    echo "STEP02C_FLAC_FAIL=0" >> "$SUMMARY_LOG"
    echo "STATUS=OK" >> "$SUMMARY_LOG"

    echo
    echo "----------------------------------------"
    echo "FLAC : 0 files to process"
    echo "----------------------------------------"
    echo "Step 2C.2 - FLAC Auto-Fix"
    echo "----------------------------------------"
    exit 0
fi

if ! command -v metaflac >/dev/null 2>&1 || ! command -v flac >/dev/null 2>&1; then
    echo "ERROR: metaflac and flac are required for FLAC deduplication." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "       Install the flac package and re-run Step 2C.2." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    echo "----------------------------------------"
    echo "Step 2C.2 - FLAC Auto-Fix"
    echo "----------------------------------------"
    exit 1
fi

count_ok=0
count_fail=0
i=0

while IFS= read -r -d '' file; do
    [[ "${file,,}" == *.flac ]] || continue
    i=$((i + 1))

    tmp_tags="$WORK_DIR/tags.$i"
    : > "$tmp_tags"

    if ! metaflac --export-tags-to="$tmp_tags" "$file" 2>>"$ERRORS_LOG"; then
        if [ ! -s "$tmp_tags" ]; then
            # No comment block present — nothing to deduplicate
            if flac -t -s "$file" >/dev/null 2>&1; then
                count_ok=$((count_ok + 1))
                echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
            else
                count_fail=$((count_fail + 1))
                echo "FAIL [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
            fi
        else
            count_fail=$((count_fail + 1))
            echo "FAIL [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
        fi
        rm -f "$tmp_tags"
        continue
    fi

    # Drop repeats that share the same key (case-insensitive) and the same value
    awk -F= '{ v = substr($0, index($0, "=") + 1); k = tolower($1); if (!seen[k "\034" v]++) print }' \
        "$tmp_tags" > "$tmp_tags.dedup"

    if cmp -s "$tmp_tags" "$tmp_tags.dedup"; then
        # Already clean — verify and count OK without touching the file
        if flac -t -s "$file" >/dev/null 2>&1; then
            count_ok=$((count_ok + 1))
            echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
        else
            count_fail=$((count_fail + 1))
            echo "FAIL [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
        fi
        rm -f "$tmp_tags" "$tmp_tags.dedup"
        continue
    fi

    if metaflac --remove-all-tags "$file" 2>>"$ERRORS_LOG" && \
       metaflac --import-tags-from="$tmp_tags.dedup" "$file" 2>>"$ERRORS_LOG" && \
       flac -t -s "$file" >/dev/null 2>&1; then
        count_ok=$((count_ok + 1))
        echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
    else
        # Failed replacement: restore the original tag set before it was removed
        metaflac --remove-all-tags "$file" 2>>"$ERRORS_LOG"
        if metaflac --import-tags-from="$tmp_tags" "$file" 2>>"$ERRORS_LOG" && flac -t -s "$file" >/dev/null 2>&1; then
            count_fail=$((count_fail + 1))
            echo "FAIL [$i/$count_total] :: $file (metadata replacement failed; original tags restored)" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
        else
            count_fail=$((count_fail + 1))
            echo "FAIL [$i/$count_total] :: $file (metadata replacement AND restore failed - REVIEW)" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
        fi
    fi

    rm -f "$tmp_tags" "$tmp_tags.dedup"
done < "$CANDIDATE_LIST"

rm -rf "$WORK_DIR"

echo "STEP02C_FLAC_OK=$count_ok" >> "$SUMMARY_LOG"
echo "STEP02C_FLAC_FAIL=$count_fail" >> "$SUMMARY_LOG"
echo "STATUS=OK" >> "$SUMMARY_LOG"

echo
echo "----------------------------------------"
echo "FLAC : $count_ok OK  $count_fail FAIL"
echo "----------------------------------------"
echo "Step 2C.2 - FLAC Auto-Fix"
echo "----------------------------------------"

```
--- Bash Script Step 2C.2 End ---

\---------------------------------------------------------------------------------------

## Step 2C.3 — MP3 Auto-Fix (COMMENT & TXXX Deduplication)

Deduplication is intentionally limited to COMMENT and user-text (TXXX) frames — the two frame types most likely to pick up redundant entries from years of repeated ripping and tagging passes. Standard singular frames (title, artist, album, and similar) are never touched. The check runs through the `eyed3` Python module, which exposes the individual comment and user-text frames that the CLI display silently collapses. Requires `python3` with the `eyed3` Python module (note: eyeD3 0.9+ renamed the import to lowercase `eyed3`; if the module is missing, install it with `sudo apt install python3-eyed3` or `python3 -m pip install --user eyeD3`).

--- Bash Script Step 2C.3 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 2C.3 — MP3 Auto-Fix
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step02c"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

CANDIDATE_LIST="$LOG_ROOT/step02-candidates.txt"

if ! command -v python3 >/dev/null 2>&1; then
    echo "ERROR: python3 is required for MP3 deduplication." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    exit 1
fi

# First pass: count MP3 candidates for progress reporting
count_total=0
while IFS= read -r -d '' file; do
    [[ "${file,,}" == *.mp3 ]] && count_total=$((count_total + 1))
done < "$CANDIDATE_LIST"

if [ "$count_total" -eq 0 ]; then
    echo "STEP02C_MP3_OK=0" >> "$SUMMARY_LOG"
    echo "STEP02C_MP3_FAIL=0" >> "$SUMMARY_LOG"
    echo "STEP02C_MP3_REVIEW=0" >> "$SUMMARY_LOG"
    echo "STATUS=OK" >> "$SUMMARY_LOG"

    echo
    echo "----------------------------------------"
    echo "MP3 : 0 files to process"
    echo "----------------------------------------"
    echo "Step 2C.3 - MP3 Auto-Fix"
    echo "----------------------------------------"
    exit 0
fi

if ! python3 -c "import eyed3" >/dev/null 2>&1; then
    echo "ERROR: the eyed3 Python module is not importable." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "       Install it with:  sudo apt install python3-eyed3  (or: python3 -m pip install --user eyeD3)" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    echo "----------------------------------------"
    echo "Step 2C.3 - MP3 Auto-Fix"
    echo "----------------------------------------"
    exit 1
fi

count_ok=0
count_fail=0
count_review=0
i=0

while IFS= read -r -d '' file; do
    [[ "${file,,}" == *.mp3 ]] || continue
    i=$((i + 1))

    python3 - "$file" <<'PYEOF' 2>>"$ERRORS_LOG"
import sys

def run():
    try:
        import eyed3
        import eyed3.id3 as ID3
        import eyed3.id3.frames as FRAMES
    except Exception as e:
        sys.stderr.write("eyed3 module unavailable: %s\n" % e)
        return 3

    path = sys.argv[1]
    try:
        audio = eyed3.load(path)
    except Exception as e:
        sys.stderr.write("unable to load file: %s\n" % e)
        return 0  # unreadable/corrupt -> leave for the other repair steps

    tag = audio.tag if audio is not None else None
    if tag is None:
        return 0  # no ID3 tag -> nothing to deduplicate

    removed = 0
    for fid in (FRAMES.COMMENT_FID, FRAMES.USERTEXT_FID):
        frames = tag.frame_set.get(fid)
        if not frames:
            continue
        seen = set()
        for frame in list(frames):
            key = ((getattr(frame, "text", "") or "") + "\x1f" +
                   (getattr(frame, "description", "") or ""))
            if key in seen:
                frames.remove(frame)
                removed += 1
            else:
                seen.add(key)

    if removed:
        try:
            tag.save(version=ID3.ID3_V2_4)
        except Exception as e:
            sys.stderr.write("unable to save changes: %s\n" % e)
            return 2  # duplicates present but could not be saved safely
        return 1  # duplicates removed
    return 0  # already clean

try:
    sys.exit(run())
except SystemExit:
    raise
except Exception as e:
    import traceback
    sys.stderr.write("UNEXPECTED ERROR: %s\n" % e)
    traceback.print_exc()
    sys.exit(4)
PYEOF
    rc=$?

    if [ "$rc" -eq 0 ] || [ "$rc" -eq 1 ]; then
        count_ok=$((count_ok + 1))
        echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
    elif [ "$rc" -eq 2 ]; then
        count_review=$((count_review + 1))
        echo "REVIEW [$i/$count_total] :: $file (duplicate frames detected but could not be removed safely)" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
    else
        count_fail=$((count_fail + 1))
        echo "FAIL [$i/$count_total] :: $file (dedup engine error - see step02c-errors.log)" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
    fi
done < "$CANDIDATE_LIST"

echo "STEP02C_MP3_OK=$count_ok" >> "$SUMMARY_LOG"
echo "STEP02C_MP3_FAIL=$count_fail" >> "$SUMMARY_LOG"
echo "STEP02C_MP3_REVIEW=$count_review" >> "$SUMMARY_LOG"
echo "STATUS=OK" >> "$SUMMARY_LOG"

echo
echo "----------------------------------------"
echo "MP3 : $count_ok OK  $count_review REVIEW  $count_fail FAIL"
echo "----------------------------------------"
echo "Step 2C.3 - MP3 Auto-Fix"
echo "----------------------------------------"

```
--- Bash Script Step 2C.3 End ---

\---------------------------------------------------------------------------------------

## Step 2C.4 — M4A/MP4 & WavPack Review Flag

The native tools for these containers (`AtomicParsley`, `wvtag`) do not support safe auto-fixing, so Step 2C.4 detects confirmed duplicate metadata entries and flags the affected files for manual review. Files are never modified by this sub-step and the audio stream is never re-encoded.

--- Bash Script Step 2C.4 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 2C.4 — M4A/MP4 & WavPack Review Flag
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step02c"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

CANDIDATE_LIST="$LOG_ROOT/step02-candidates.txt"
WORK_DIR="$LOG_ROOT/step02c-work"
mkdir -p "$WORK_DIR"

# First pass: count review-capable candidates for progress reporting
count_total=0
while IFS= read -r -d '' file; do
    fname="$(basename "$file")"
    ext_lc="$(printf '%s' "${fname##*.}" | tr '[:upper:]' '[:lower:]')"
    case "$ext_lc" in
        m4a|mp4|wv) count_total=$((count_total + 1)) ;;
    esac
done < "$CANDIDATE_LIST"

if [ "$count_total" -eq 0 ]; then
    echo "STEP02C_M4A_WV_CLEAN=0" >> "$SUMMARY_LOG"
    echo "STEP02C_M4A_WV_REVIEW=0" >> "$SUMMARY_LOG"
    echo "STEP02C_M4A_WV_FAIL=0" >> "$SUMMARY_LOG"
    echo "STATUS=OK" >> "$SUMMARY_LOG"

    echo
    echo "----------------------------------------"
    echo "M4A/MP4/WV : 0 files to review"
    echo "----------------------------------------"
    echo "Step 2C.4 - M4A/MP4 & WavPack Review Flag"
    echo "----------------------------------------"
    exit 0
fi

needs_atomic=0
needs_wv=0
while IFS= read -r -d '' file; do
    fname="$(basename "$file")"
    ext_lc="$(printf '%s' "${fname##*.}" | tr '[:upper:]' '[:lower:]')"
    case "$ext_lc" in
        m4a|mp4) needs_atomic=1 ;;
        wv) needs_wv=1 ;;
    esac
done < "$CANDIDATE_LIST"

if [ "$needs_atomic" -eq 1 ] && ! command -v AtomicParsley >/dev/null 2>&1; then
    echo "ERROR: AtomicParsley is missing but M4A/MP4 files were found." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "       Install AtomicParsley and re-run Step 2C.4." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    echo "----------------------------------------"
    echo "Step 2C.4 - M4A/MP4 & WavPack Review Flag"
    echo "----------------------------------------"
    exit 1
fi

if [ "$needs_wv" -eq 1 ] && ! command -v wvtag >/dev/null 2>&1; then
    echo "ERROR: wvtag is missing but WavPack files were found." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "       Install wavpack and re-run Step 2C.4." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    echo "----------------------------------------"
    echo "Step 2C.4 - M4A/MP4 & WavPack Review Flag"
    echo "----------------------------------------"
    exit 1
fi

count_clean=0
count_review=0
count_fail=0
i=0

while IFS= read -r -d '' file; do
    fname="$(basename "$file")"
    ext_lc="$(printf '%s' "${fname##*.}" | tr '[:upper:]' '[:lower:]')"

    case "$ext_lc" in
        m4a|mp4)
            i=$((i + 1))

            atoms=$(AtomicParsley "$file" -t 2>>"$ERRORS_LOG")
            rc=$?

            if [ $rc -ne 0 ]; then
                count_fail=$((count_fail + 1))
                echo "FAIL [$i/$count_total] :: $file (AtomicParsley could not read metadata)" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
                continue
            fi

            if [ -z "$atoms" ]; then
                count_clean=$((count_clean + 1))
                echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
                continue
            fi

            dup=$(printf '%s\n' "$atoms" | sed 's/[[:space:]]*$//' | sort | uniq -d)
            if [ -n "$dup" ]; then
                count_review=$((count_review + 1))
                echo "REVIEW [$i/$count_total] :: $file (duplicate metadata atoms detected)" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
            else
                count_clean=$((count_clean + 1))
                echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
            fi
            ;;
        wv)
            i=$((i + 1))

            wvtag -l "$file" > "$WORK_DIR/wvtag.$i" 2>>"$ERRORS_LOG"
            rc=$?

            if [ $rc -ne 0 ]; then
                count_review=$((count_review + 1))
                echo "REVIEW [$i/$count_total] :: $file (wvtag could not parse the tag block)" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
            else
                dup=$(sed 's/[[:space:]]*$//' "$WORK_DIR/wvtag.$i" | sort | uniq -d)
                if [ -n "$dup" ]; then
                    count_review=$((count_review + 1))
                    echo "REVIEW [$i/$count_total] :: $file (duplicate tag items detected)" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
                else
                    count_clean=$((count_clean + 1))
                    echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
                fi
            fi
            ;;
    esac
done < "$CANDIDATE_LIST"

rm -rf "$WORK_DIR"

echo "STEP02C_M4A_WV_CLEAN=$count_clean" >> "$SUMMARY_LOG"
echo "STEP02C_M4A_WV_REVIEW=$count_review" >> "$SUMMARY_LOG"
echo "STEP02C_M4A_WV_FAIL=$count_fail" >> "$SUMMARY_LOG"
echo "STATUS=OK" >> "$SUMMARY_LOG"

echo
echo "----------------------------------------"
echo "M4A/MP4/WV : $count_clean OK  $count_review REVIEW  $count_fail FAIL"
echo "----------------------------------------"
echo "Step 2C.4 - M4A/MP4 & WavPack Review Flag"
echo "----------------------------------------"

```
--- Bash Script Step 2C.4 End ---

\---------------------------------------------------------------------------------------

## Step 2C.5 — OGG & Opus Auto-Fix (Vorbis Comment Deduplication)

OGG Vorbis and Opus share the Vorbis comment model, so duplicates are resolved the same way as FLAC: export every comment, drop repeats that share the same key (case-insensitive) and the same value, then re-import the deduplicated set with the format's native writer. Audio streams are never re-encoded. Every modified file is verified with an `ffmpeg` decode-to-null check.

--- Bash Script Step 2C.5 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 2C.5 — OGG & Opus Auto-Fix
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step02c"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

CANDIDATE_LIST="$LOG_ROOT/step02-candidates.txt"
WORK_DIR="$LOG_ROOT/step02c-work"
mkdir -p "$WORK_DIR"

# First pass: count OGG/Opus candidates for progress reporting
count_total=0
while IFS= read -r -d '' file; do
    fname="$(basename "$file")"
    ext_lc="$(printf '%s' "${fname##*.}" | tr '[:upper:]' '[:lower:]')"
    case "$ext_lc" in
        ogg|opus) count_total=$((count_total + 1)) ;;
    esac
done < "$CANDIDATE_LIST"

if [ "$count_total" -eq 0 ]; then
    echo "STEP02C_VORBIS_OK=0" >> "$SUMMARY_LOG"
    echo "STEP02C_VORBIS_FAIL=0" >> "$SUMMARY_LOG"
    echo "STATUS=OK" >> "$SUMMARY_LOG"

    echo
    echo "----------------------------------------"
    echo "OGG/OPUS : 0 files to process"
    echo "----------------------------------------"
    echo "Step 2C.5 - OGG & Opus Auto-Fix"
    echo "----------------------------------------"
    exit 0
fi

needs_vorbis=0
needs_opus=0
while IFS= read -r -d '' file; do
    fname="$(basename "$file")"
    ext_lc="$(printf '%s' "${fname##*.}" | tr '[:upper:]' '[:lower:]')"
    case "$ext_lc" in
        ogg) needs_vorbis=1 ;;
        opus) needs_opus=1 ;;
    esac
done < "$CANDIDATE_LIST"

if [ "$needs_vorbis" -eq 1 ] && ! command -v vorbiscomment >/dev/null 2>&1; then
    echo "ERROR: vorbiscomment is missing but OGG files were found." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "       Install vorbis-tools and re-run Step 2C.5." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    echo "----------------------------------------"
    echo "Step 2C.5 - OGG & Opus Auto-Fix"
    echo "----------------------------------------"
    exit 1
fi

if [ "$needs_opus" -eq 1 ] && ! command -v opustags >/dev/null 2>&1; then
    echo "ERROR: opustags is missing but Opus files were found." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "       Install opustags and re-run Step 2C.5." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    echo "----------------------------------------"
    echo "Step 2C.5 - OGG & Opus Auto-Fix"
    echo "----------------------------------------"
    exit 1
fi

if ! command -v ffmpeg >/dev/null 2>&1; then
    echo "ERROR: ffmpeg is missing but is used to verify OGG/Opus rewrites." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "       Install ffmpeg and re-run Step 2C.5." | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    echo "----------------------------------------"
    echo "Step 2C.5 - OGG & Opus Auto-Fix"
    echo "----------------------------------------"
    exit 1
fi

count_ok=0
count_fail=0
i=0

while IFS= read -r -d '' file; do
    fname="$(basename "$file")"
    ext_lc="$(printf '%s' "${fname##*.}" | tr '[:upper:]' '[:lower:]')"
    case "$ext_lc" in
        ogg|opus) ;;
        *) continue ;;
    esac
    i=$((i + 1))

    tmp_tags="$WORK_DIR/tags.$i"
    : > "$tmp_tags"

    case "$ext_lc" in
        ogg)
            if ! vorbiscomment -l "$file" > "$tmp_tags" 2>>"$ERRORS_LOG"; then
                if [ ! -s "$tmp_tags" ]; then
                    count_fail=$((count_fail + 1))
                    echo "FAIL [$i/$count_total] :: $file (vorbiscomment could not read the file)" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
                else
                    count_fail=$((count_fail + 1))
                    echo "FAIL [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
                fi
                rm -f "$tmp_tags"
                continue
            fi
            grep '=' "$tmp_tags" > "$tmp_tags.filtered"
            if [ ! -s "$tmp_tags.filtered" ]; then
                count_ok=$((count_ok + 1))
                echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
                rm -f "$tmp_tags" "$tmp_tags.filtered"
                continue
            fi
            awk -F= '{ v = substr($0, index($0, "=") + 1); k = tolower($1); if (!seen[k "\034" v]++) print }' \
                "$tmp_tags.filtered" > "$tmp_tags.dedup"
            if cmp -s "$tmp_tags.filtered" "$tmp_tags.dedup"; then
                count_ok=$((count_ok + 1))
                echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
                rm -f "$tmp_tags" "$tmp_tags.filtered" "$tmp_tags.dedup"
                continue
            fi
            if vorbiscomment -w "$file" < "$tmp_tags.dedup" 2>>"$ERRORS_LOG" && \
               ffmpeg -nostdin -v error -i "$file" -f null - >/dev/null 2>&1; then
                count_ok=$((count_ok + 1))
                echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
            else
                count_fail=$((count_fail + 1))
                echo "FAIL [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
            fi
            rm -f "$tmp_tags" "$tmp_tags.filtered" "$tmp_tags.dedup"
            ;;
        opus)
            if ! opustags -l "$file" > "$tmp_tags" 2>>"$ERRORS_LOG"; then
                if [ ! -s "$tmp_tags" ]; then
                    count_fail=$((count_fail + 1))
                    echo "FAIL [$i/$count_total] :: $file (opustags could not read the file)" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
                else
                    count_fail=$((count_fail + 1))
                    echo "FAIL [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
                fi
                rm -f "$tmp_tags"
                continue
            fi
            grep '=' "$tmp_tags" > "$tmp_tags.filtered"
            if [ ! -s "$tmp_tags.filtered" ]; then
                count_ok=$((count_ok + 1))
                echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
                rm -f "$tmp_tags" "$tmp_tags.filtered"
                continue
            fi
            awk -F= '{ v = substr($0, index($0, "=") + 1); k = tolower($1); if (!seen[k "\034" v]++) print }' \
                "$tmp_tags.filtered" > "$tmp_tags.dedup"
            if cmp -s "$tmp_tags.filtered" "$tmp_tags.dedup"; then
                count_ok=$((count_ok + 1))
                echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
                rm -f "$tmp_tags" "$tmp_tags.filtered" "$tmp_tags.dedup"
                continue
            fi
            if opustags -s "$tmp_tags.dedup" -w "$file" 2>>"$ERRORS_LOG" && \
               ffmpeg -nostdin -v error -i "$file" -f null - >/dev/null 2>&1; then
                count_ok=$((count_ok + 1))
                echo "OK   [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
            else
                count_fail=$((count_fail + 1))
                echo "FAIL [$i/$count_total] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
            fi
            rm -f "$tmp_tags" "$tmp_tags.filtered" "$tmp_tags.dedup"
            ;;
    esac
done < "$CANDIDATE_LIST"

rm -rf "$WORK_DIR"

echo "STEP02C_VORBIS_OK=$count_ok" >> "$SUMMARY_LOG"
echo "STEP02C_VORBIS_FAIL=$count_fail" >> "$SUMMARY_LOG"
echo "STATUS=OK" >> "$SUMMARY_LOG"

echo
echo "----------------------------------------"
echo "OGG/OPUS : $count_ok OK  $count_fail FAIL"
echo "----------------------------------------"
echo "Step 2C.5 - OGG & Opus Auto-Fix"
echo "----------------------------------------"

```
--- Bash Script Step 2C.5 End ---

\---------------------------------------------------------------------------------------

## Step 2C.6 — Summary

Aggregates the Step 2C sub-step results into `step02c-summary.log` and prints the final Step 2C status. Step 2E reads this summary file.

--- Bash Script Step 2C.6 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 2C.6 — Summary
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step02c"
mkdir -p "$LOG_ROOT"

OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

ok_count=$(grep -a '^OK' "$OKS_LOG" 2>/dev/null | wc -l)
fail_count=$(grep -a '^FAIL' "$FAILS_LOG" 2>/dev/null | wc -l)
review_count=$(grep -a '^REVIEW' "$FAILS_LOG" 2>/dev/null | wc -l)
total=$(grep -a '^TOTAL_CANDIDATES=' "$SUMMARY_LOG" 2>/dev/null | cut -d= -f2)

{
echo "Step 2C Summary"
echo "=============="
echo
echo "Step       : step02c"
echo "Run Date   : $(date)"
echo
echo "Processed  : ${total:-0}"
echo "OK         : $ok_count"
echo "FAIL       : $fail_count"
echo "REVIEW     : $review_count"
} > "$SUMMARY_LOG"

echo
echo "----------------------------------------"
echo "Processed: ${total:-0}  OK: $ok_count  FAIL: $fail_count  REVIEW: $review_count"
echo "----------------------------------------"
echo "Step 2C.6 - Summary"
echo "----------------------------------------"
cat "$SUMMARY_LOG"

```
--- Bash Script Step 2C.6 End ---

\---------------------------------------------------------------------------------------

## Step 2D — Verification

Verify the files modified by Step 2C and confirm that the intended cleanup occurred.

FLAC files are verified with `flac -t`. Every other modified format is verified with an `ffmpeg` decode-to-null stream check. `ffmpeg` is run with `-nostdin` — without it, `ffmpeg` shares the loop's input stream and silently consumes a byte meant for the next file, corrupting the following iteration's path. This surfaced during testing and is the same class of bug already fixed once before in the artwork-normalization scripts.

--- Bash Script Step 2D Start ---

```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 2D — Verification
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step02d"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

: > "$RUN_LOG"
: > "$OKS_LOG"
: > "$FAILS_LOG"
: > "$ERRORS_LOG"
: > "$SUMMARY_LOG"

# Software Preflight: fail loudly if a required tool is missing
for tool in flac ffmpeg metaflac; do
    if ! command -v "$tool" >/dev/null 2>&1; then
        echo "ERROR: $tool is not installed. Install it and re-run (see Requirements)." >&2
        exit 1
    fi
done

CANDIDATE_LIST="$LOG_ROOT/step02-candidates.txt"

if [ ! -s "$CANDIDATE_LIST" ]; then
    echo "ERROR: candidate list empty or missing :: $CANDIDATE_LIST" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null

    if [ -t 1 ]; then
        echo
        read -rp "Would you like to view the error log? [Y/N]: " choice </dev/tty
        case "$choice" in
            [yY][eE][sS]|[yY])
                echo "----------------------------------------"
                echo "ERROR LOG DUMP:"
                echo "----------------------------------------"
                cat "$ERRORS_LOG"
                ;;
        esac
    fi

    echo "Status: FAILED (Run Step 2A first)"
    echo "----------------------------------------"
    echo "Step 2D - Verification"
    echo "----------------------------------------"
    exit 1
fi

# Count total candidates upfront for progress reporting
total_files=$(grep -c $'\0' "$CANDIDATE_LIST" || grep -c '^' "$CANDIDATE_LIST")

echo "Notice: Integrity checking in progress."
echo "Total files to process: $total_files"
echo

passed_count=0
corrupt_count=0
error_count=0
current=0

while IFS= read -r -d '' file; do
    current=$((current + 1))

    if [ ! -r "$file" ]; then
        echo "ERROR [$current/$total_files] :: $file (unreadable)" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
        error_count=$((error_count + 1))
        continue
    fi

    fname=$(basename "$file")
    ext="${fname##*.}"
    ext_lc="$(printf '%s' "$ext" | tr '[:upper:]' '[:lower:]')"

    if [ "$ext_lc" = flac ]; then
        verify_cmd=(flac -t -s "$file")
    else
        verify_cmd=(ffmpeg -nostdin -v error -i "$file" -f null -)
    fi

    if "${verify_cmd[@]}" >/dev/null 2>&1; then
        echo "OK   [$current/$total_files] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
        passed_count=$((passed_count + 1))
    else
        echo "FAIL [$current/$total_files] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
        corrupt_count=$((corrupt_count + 1))
    fi
done < "$CANDIDATE_LIST"

echo "PASSED_FILES=$passed_count" | tee -a "$SUMMARY_LOG" >/dev/null
echo "CORRUPT_FILES=$corrupt_count" | tee -a "$SUMMARY_LOG" >/dev/null
echo "ERROR_FILES=$error_count" | tee -a "$SUMMARY_LOG" >/dev/null
echo "STATUS=OK" | tee -a "$SUMMARY_LOG" >/dev/null

# Interactive Screen Dump Prompts
if [ -t 1 ] && [ -s "$ERRORS_LOG" ]; then
    echo
    read -rp "ERRORS DETECTED — Would you like to view the error log? [Y/N]: " choice </dev/tty
    case "$choice" in
        [yY][eE][sS]|[yY])
            echo "----------------------------------------"
            echo "ERROR LOG DUMP:"
            echo "----------------------------------------"
            cat "$ERRORS_LOG"
            ;;
    esac
elif [ -t 1 ] && [ -s "$FAILS_LOG" ]; then
    echo
    read -rp "CORRUPT/FAILED FILES DETECTED — Would you like to view the log? [Y/N]: " choice </dev/tty
    case "$choice" in
        [yY][eE][sS]|[yY])
            echo "----------------------------------------"
            echo "CORRUPT FILES LOG DUMP:"
            echo "----------------------------------------"
            cat "$FAILS_LOG"
            ;;
    esac
fi

# Footer — Strictly the final output before shell prompt returns
echo
echo "----------------------------------------"
echo "Passed integrity check : $passed_count"
echo "Corrupt/Failed files   : $corrupt_count"
echo "System/Read errors     : $error_count"
echo "----------------------------------------"
echo "Step 2D - Verification"
echo "----------------------------------------"

```
--- Bash Script Step 2D End ---

\ ---------------------------------------------------------------------------------------

## Step 2E — Summary

Produce the final Step 2 results and status.

--- Bash Script Step 2E Start ---

```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 2E — Summary
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step02e"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

: > "$RUN_LOG"
: > "$OKS_LOG"
: > "$FAILS_LOG"
: > "$ERRORS_LOG"
: > "$SUMMARY_LOG"

for sub in step02a step02b step02c step02d; do
    src="$LOG_ROOT/${sub}-summary.log"
    if [ -f "$src" ]; then
        echo "OK found summary :: $sub" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
        { echo "[$sub]"; cat "$src"; echo; } >> "$SUMMARY_LOG"
    else
        echo "ERROR missing summary :: $sub" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    fi
done

cat "$SUMMARY_LOG"

echo
echo "----------------------------------------"
echo "Summary written to : $SUMMARY_LOG"
echo "----------------------------------------"
echo "Step 2E - Summary"
echo "----------------------------------------"

```
--- Bash Script Step 2E End ---

\ ---------------------------------------------------------------------------------------

-- Testing Notes

Every dedup path in Step 2C (FLAC, MP3 COMMENT, MP3 TXXX, OGG, Opus) and the M4A review-flag path were validated end to end against fabricated fixtures with byte-identical duplicate tags, including a full 2A→2E pipeline run against a mixed twelve-format-plus test library. The WavPack review path relies on `wvtag -l` output; `wvtag` itself prevents duplicate keys through its own write path (case-insensitive collapse), so a genuine raw duplicate item was only reproducible through direct byte-level splicing, and `wvtag` failed to parse that fixture at all — the same class of failure seen with `AtomicParsley` on M4A. In practice this means `review_wv` will report clean far more often than `review_m4a`; that asymmetry is expected, not a bug.

\ ---------------------------------------------------------------------------------------

---

07. Step 3 – Rebuild Audio Containers (All Formats)

---

-- Purpose

This step rebuilds audio containers to remove invalid or incompatible metadata headers that may prevent proper reading by audio players and media applications.

Files may contain unexpected ID3v2 headers, malformed atoms, or other container-level issues. This process rebuilds the container while preserving the original audio stream (no re-encoding).

-- What It Does

This step:

* Scans the selected library location recursively for audio files (FLAC, MP3, M4A, OGG, Opus, WAV, AIFF).
* Uses ffmpeg to rebuild each file's container.
* Copies the existing audio stream without re-encoding.
* Removes problematic container headers and metadata issues.
* Replaces the original file only after successful rebuild verification.
* Records files that could not be processed.

No audio quality changes occur because the audio stream is copied rather than converted.

\ ---------------------------------------------------------------------------------------

--- Bash Script Step 3A Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 3 – Rebuild Audio Containers (All Formats)
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step03"

mkdir -p "$LOG_ROOT"

# Software Preflight: fail loudly if a required tool is missing
for tool in ffmpeg ffprobe; do
    if ! command -v "$tool" >/dev/null 2>&1; then
        echo "ERROR: $tool is not installed. Install it and re-run (see Requirements)." >&2
        exit 1
    fi
done

# 1. Define Log Files (Five-File Standard)
RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OK_LOG="$LOG_ROOT/${STEP}-oks.log"
FAIL_LOG="$LOG_ROOT/${STEP}-fails.log"
ERROR_LOG="$LOG_ROOT/${STEP}-errors.log"
WARN_LOG="$LOG_ROOT/${STEP}-warnings.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

# 2. CLEANUP: Delete this step's own logs from any previous run
rm -f "$RUN_LOG" "$OK_LOG" "$FAIL_LOG" "$ERROR_LOG" "$WARN_LOG" "$SUMMARY_LOG"

# 3. Initialize Empty Log Files
touch "$RUN_LOG" "$OK_LOG" "$FAIL_LOG" "$ERROR_LOG" "$WARN_LOG" "$SUMMARY_LOG"

# Measure the real decodable duration of a file in seconds (empty if unreadable).
# Header/format durations lie when a file is padded with junk (e.g. 0xFF fill),
# so a full decode is the only truthful measure of what a player will hear.
decode_real() {
    ffmpeg -nostdin -v info -i "$1" -f null - 2>&1 |
        grep -aoE 'time=[0-9]{2}:[0-9]{2}:[0-9]{2}(\.[0-9]{1,3})?' |
        tail -1 | sed 's/time=//' |
        awk -F'[:.]' '{f=$4; n=length(f); print ($1*3600)+($2*60)+$3+(f/(10^n))}'
}

# 4. File Discovery (all supported audio formats)
#    Optional override: pass $1 = a NUL-terminated file list to re-process only
#    those files (e.g. the failures from a previous run), instead of a full scan.
if [ -n "${1:-}" ] && [ -r "$1" ]; then
    mapfile -d '' files < "$1"
else
mapfile -d '' files < <(
    find "$PWD" -type f \
        ! -ipath '*/Ignore/*' \
        ! -iname "*.prerepair" \
        ! -iname "*.fixed.*" \
        ! -iname "*.reencode.*" \
        \( \
            -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a"  -o \
            -iname "*.ogg"  -o -iname "*.opus" -o -iname "*.wav"  -o \
            -iname "*.aiff" -o -iname "*.aif" \
        \) -print0 2>>"$LOG_ROOT/${STEP}-errors.log" | LC_ALL=C sort -z
)
fi

total=${#files[@]}
i=0

for f in "${files[@]}"; do
    ((i++))

    artist=$(basename "$(dirname "$(dirname "$f")")")
    album=$(basename "$(dirname "$f")")
    track=$(basename "$f")
    label="$artist-$album-$track"
    
    # Preserve original extension
    fname=$(basename "$f")
    ext="${fname##*.}"
    fixed="${f%.*}.fixed.${ext}"

    err=$(ffmpeg \
        -nostdin \
        -nostats \
        -loglevel error \
        -i "$f" \
        -map_metadata 0 \
        -c copy \
        "$fixed" \
        -y 2>&1)
    rc=$?

    # Non-fatal MJPEG "unable to decode APP fields" warnings come from corrupt
    # embedded JPEG artwork; they must not fail a valid container rebuild.
    warn="$(printf '%s\n' "$err" | grep -E 'unable to decode APP fields|Invalid data found when processing input' || true)"
    err="$(printf '%s\n' "$err" | grep -v -E 'unable to decode APP fields|Invalid data found when processing input' | grep -v '^[[:space:]]*$' || true)"

    # Duration sanity check: a silent partial copy must never replace the source.
    # 1) Cheap header check first; 2) if headers disagree by >2%, decode both sides
    # fully and compare their true decodable audio — sources padded with junk
    # (0xFF fill) lie about their real duration and must not be treated as lost.
    in_dur=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "$f" 2>/dev/null)
    out_dur=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "$fixed" 2>/dev/null)
    junk_note=""
    dur_ok=1
    if [ -n "$in_dur" ] && [ -n "$out_dur" ]; then
        if ! awk -v a="$in_dur" -v b="$out_dur" 'BEGIN { d=a-b; if (d<0) d=-d; exit (d<=0.02*a ? 0 : 1) }'; then
            # Headers disagree by more than 2%: decode both and compare truthfully.
            src_real=$(decode_real "$f")
            out_real=$(decode_real "$fixed")
            if [ -n "$src_real" ] && [ -n "$out_real" ]; then
                if ! awk -v a="$src_real" -v b="$out_real" 'BEGIN { d=a-b; if (d<0) d=-d; exit (d<=0.5 ? 0 : 1) }'; then
                    dur_ok=0
                elif awk -v a="$out_real" -v b="$in_dur" 'BEGIN { exit !(b-a>45) }'; then
                    junk_note="source claimed ${in_dur}s but only ${out_real}s decodable; junk tail removed"
                fi
            else
                dur_ok=0
            fi
        fi
    fi

    if [ $rc -ne 0 ] || [ -n "$err" ] || [ ! -s "$fixed" ] || [ "$dur_ok" -ne 1 ]; then
        flat=$(echo "$err" | tr -d '\000' | tr '\n' ' ' | tr -s ' ')
        rm -f "$fixed"
        echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAIL_LOG"
        if [ "$dur_ok" -ne 1 ]; then
            echo "[$i/$total] ERROR (exit $rc, duration mismatch verified by full decode: in=$in_dur out=$out_dur): $label :: $f :: ${flat:-no stderr output}" >> "$ERROR_LOG"
        else
            echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERROR_LOG"
        fi
    else
        # Back up the original once before overwriting (removed later by Step 7)
        if [ ! -e "${f}.prerepair" ]; then
            cp -p "$f" "${f}.prerepair" 2>>"$ERROR_LOG" || {
                rm -f "$fixed"
                echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAIL_LOG"
                echo "[$i/$total] ERROR (backup failed): $label :: $f :: could not create ${f}.prerepair" >> "$ERROR_LOG"
                continue
            }
        fi
        if mv -f "$fixed" "$f"; then
            echo "OK   [$i/$total] $label" | tee -a "$RUN_LOG" "$OK_LOG"
            if [ -n "$warn" ]; then
                echo "[$i/$total] WARN: $label :: embedded artwork warnings: $(printf '%s' "$warn" | tr '\n' ' ')" >> "$WARN_LOG"
            fi
            if [ -n "$junk_note" ]; then
                echo "[$i/$total] WARN: $label :: $junk_note" >> "$WARN_LOG"
            fi
        else
            mv_rc=$?
            rm -f "$fixed"
            echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAIL_LOG"
            echo "[$i/$total] ERROR (mv exit $mv_rc): $label :: $f :: failed to move rebuilt file into place" >> "$ERROR_LOG"
        fi
    fi
done

# 5. Count Results
ok_count=$(grep -a "^OK" "$RUN_LOG" 2>/dev/null | wc -l)
fail_count=$(grep -a "^FAIL" "$RUN_LOG" 2>/dev/null | wc -l)

# 6. Generate Summary
{
echo "Step 3 Summary"
echo "=============="
echo
echo "Step       : $STEP"
echo "Run Date   : $(date)"
echo
echo "Processed  : $total"
echo "Passed     : $ok_count"
echo "Failed     : $fail_count"
} > "$SUMMARY_LOG"

# 7. Terminal Output
echo
echo "----------------------------------------"
echo "Processed: $total  Passed: $ok_count  Failed: $fail_count"
echo "----------------------------------------"
echo "Step 3A – Strip Invalid Metadata Headers"
echo "----------------------------------------"

```
--- Bash Script Step 3A End ---

\ ---------------------------------------------------------------------------------------

-- Step 3B: View Log Files

--- Bash Script Cat 3B Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 3B – View Log Results
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step03"

echo "=== ${STEP}-summary.log ==="
cat "$LOG_ROOT/${STEP}-summary.log"

echo
echo "=== ${STEP}-errors.log ==="
cat "$LOG_ROOT/${STEP}-errors.log"

echo
echo "=== ${STEP}-run.log ==="
cat "$LOG_ROOT/${STEP}-run.log"

echo
echo "=== ${STEP}-oks.log ==="
cat "$LOG_ROOT/${STEP}-oks.log"

echo
echo "=== ${STEP}-fails.log ==="
cat "$LOG_ROOT/${STEP}-fails.log"
echo
echo "----------------------------------------"
echo "Step 3B – View Log Results"
echo "----------------------------------------"

```
--- Bash Script Cat 3B End ---

---

08. Step 4 – Repeat Step 1 Integrity Test

---

-- Purpose

This step repeats the integrity test performed in Step 1, but now `after` Step 3's container rebuilding. It verifies all audio files to confirm that the container rebuild was successful and that files remain structurally valid.

Running this verification after container work provides a direct comparison against the original Step 1 baseline, identifying files that were repaired and files that continue to report problems.

No files are modified during this step. This is a verification step only.

\ ---------------------------------------------------------------------------------------

--- Bash Script Step 4 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ============================================================
# Step 4 – Post-Rebuild Integrity Verification
# ============================================================

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step04"

mkdir -p "$LOG_ROOT"

# Software Preflight: fail loudly if a required tool is missing
for tool in flac ffmpeg; do
    if ! command -v "$tool" >/dev/null 2>&1; then
        echo "ERROR: $tool is not installed. Install it and re-run (see Requirements)." >&2
        exit 1
    fi
done

# 1. Define Log Files (Five-File Standard)
RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

# 2. CLEANUP: Delete this step's own logs from any previous run
rm -f "$RUN_LOG" "$OKS_LOG" "$FAILS_LOG" "$ERRORS_LOG" "$SUMMARY_LOG"

# 3. Initialize Empty Log Files
touch "$RUN_LOG" "$OKS_LOG" "$FAILS_LOG" "$ERRORS_LOG" "$SUMMARY_LOG"

# 4. File Discovery
mapfile -d '' files < <(
    find "$PWD" -type f \
        ! -ipath '*/Ignore/*' \
        ! -iname "*.prerepair*" \
        ! -iname "*.fixed.*" \
        ! -iname "*.reencode.*" \
        \( \
            -iname "*.flac" -o \
            -iname "*.mp3"  -o \
            -iname "*.m4a"  -o \
            -iname "*.ogg"  -o \
            -iname "*.opus" -o \
            -iname "*.wav"  -o \
            -iname "*.aiff" -o \
            -iname "*.aif"  -o \
            -iname "*.mp4"  -o \
            -iname "*.ape"  -o \
            -iname "*.wv"   -o \
            -iname "*.spx" \
        \) -print0 2>>"$LOG_ROOT/${STEP}-errors.log" | sort -z
)

total=${#files[@]}
i=0
last_dir=""

for f in "${files[@]}"; do

    ((i++))
    label="${f#"$PWD"/}"
    current_dir="$(dirname "$label")"

    # Insert a blank line on terminal screen when moving to a new folder/album
    if [[ -n "$last_dir" && "$current_dir" != "$last_dir" ]]; then
        echo ""
    fi
    last_dir="$current_dir"

    case "${f,,}" in
        *.flac)
            err=$(flac -s -t "$f" 2>&1)
            rc=$?
            ;;
        *)
            err=$(ffmpeg -nostdin -v error -i "$f" -f null - 2>&1)
            rc=$?
            ;;
    esac

    if [ $rc -eq 0 ]; then
        out_msg="OK   [$i/$total] $label"
        echo "$out_msg"
        echo "$out_msg" >> "$RUN_LOG"
        echo "$out_msg" >> "$OKS_LOG"
    else
        flat=$(printf '%s\n' "$err" | tr '\r\n' ' ' | tr -s ' ')
        out_msg="FAIL [$i/$total] $label"
        echo "$out_msg"
        echo "$out_msg" >> "$RUN_LOG"
        echo "$out_msg" >> "$FAILS_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
    fi

done

# 5. Count Results
ok_count=$(grep -a "^OK" "$RUN_LOG" 2>/dev/null | wc -l)
fail_count=$(grep -a "^FAIL" "$RUN_LOG" 2>/dev/null | wc -l)

# 6. Generate Summary Log
{
echo "Step 4 Summary"
echo "=============="
echo
echo "Step       : $STEP"
echo "Run Date   : $(date)"
echo
echo "Processed  : $total"
echo "Passed     : $ok_count"
echo "Failed     : $fail_count"
} > "$SUMMARY_LOG"

# 7. Terminal Output
echo
if [ -s "$ERRORS_LOG" ]; then
    echo "----------------------------------------"
    echo "Error Summary"
    echo "----------------------------------------"
    awk '
    /LOST_SYNC/ {
        idx = index($0, " :: ")
        if (idx > 0) {
            temp = substr($0, 1, idx - 1)
            pos = index(temp, "): ") + 3
            path = substr(temp, pos)
            lost_sync[path] = 1
        }
    }
    /END_OF_STREAM/ && !/LOST_SYNC/ {
        idx = index($0, " :: ")
        if (idx > 0) {
            temp = substr($0, 1, idx - 1)
            pos = index(temp, "): ") + 3
            path = substr(temp, pos)
            eos[path] = 1
        }
    }
    END {
        if (length(lost_sync) > 0) {
            print "LOST_SYNC"
            print "----------"
            for (p in lost_sync) print p | "sort"
            close("sort")
        }
        if (length(eos) > 0) {
            if (length(lost_sync) > 0) print ""
            print "END_OF_STREAM"
            print "----------"
            for (p in eos) print p | "sort"
            close("sort")
        }
    }
    ' "$ERRORS_LOG"
fi

echo
echo "----------------------------------------"
echo "Processed: $total  Passed: $ok_count  Failed: $fail_count"
echo "----------------------------------------"
echo "Step 4 – Post-Rebuild Integrity Verification"
echo "----------------------------------------"

```
--- Bash Script Step 4 End ---

\ ---------------------------------------------------------------------------------------

-- Review Results

--- Bash Script Cat 4 Start ---

```bash
LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step04"

echo "=== ${STEP}-summary.log ==="
cat "$LOG_ROOT/${STEP}-summary.log"

echo
echo "=== ${STEP}-errors.log ==="
cat "$LOG_ROOT/${STEP}-errors.log"

echo
echo "=== ${STEP}-run.log ==="
cat "$LOG_ROOT/${STEP}-run.log"

echo
echo "=== ${STEP}-oks.log ==="
cat "$LOG_ROOT/${STEP}-oks.log"

echo
echo "=== ${STEP}-fails.log ==="
cat "$LOG_ROOT/${STEP}-fails.log"

```
--- Bash Script Cat 4 End ---

---

09. Step 5 – Reapply ReplayGain

---

-- Purpose

This step restores ReplayGain metadata that may have been removed during the FLAC container rebuild performed in Step 3.

ReplayGain stores volume adjustment information in metadata tags so compatible music players can provide consistent playback volume between tracks and albums without changing the actual audio data.

This step recalculates and reapplies ReplayGain information after the FLAC files have been rebuilt and verified.

-- What It Does

This step:

* Scans the library by album directory.
* Evaluates the files in each album as a group.
* Calculates album-level ReplayGain values.
* Writes ReplayGain metadata back into the files.
* Records albums that were successfully processed and any failures.

This step recalculates and reapplies ReplayGain metadata after the FLAC container rebuilds in Step 3. ReplayGain stores volume adjustment so compatible players provide consistent playback levels without changing audio data.

No audio is modified or re-encoded. Calculation is performed at album level to preserve track relationships within each album.

\ ---------------------------------------------------------------------------------------

--- Bash Script Step 5 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ============================================================
# Step 5 – Reapply ReplayGain (moOde Audio Standard)
# ============================================================

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step05"

mkdir -p "$LOG_ROOT"

# Software Preflight: fail loudly if a required tool is missing
if ! command -v loudgain >/dev/null 2>&1; then
    echo "ERROR: loudgain is not installed. Install it and re-run (see Requirements)." >&2
    exit 1
fi

# 1. Define Log Files (Five-File Standard)
RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

# 2. CLEANUP: Delete this step's own logs from any previous run
rm -f "$RUN_LOG" "$OKS_LOG" "$FAILS_LOG" "$ERRORS_LOG" "$SUMMARY_LOG"

# 3. Initialize Empty Log Files
touch "$RUN_LOG" "$OKS_LOG" "$FAILS_LOG" "$ERRORS_LOG" "$SUMMARY_LOG"

# 4. Supported audio extensions
SUPPORTED_EXTS=(flac mp3 m4a ogg opus mp4 ape wv spx)

# 5. Gather and sort directories by path (Artist/Album)
mapfile -d '' dirs < <(find "$PWD" -type d ! -ipath '*/Ignore/*' -print0 | LC_ALL=C sort -f -z)

# 6. Calculate total directories with supported audio files
total=0
for d in "${dirs[@]}"; do
    shopt -s nocaseglob nullglob
    files=(
        "$d"/*.flac "$d"/*.mp3 "$d"/*.m4a "$d"/*.ogg
        "$d"/*.opus "$d"/*.mp4 "$d"/*.ape "$d"/*.wv "$d"/*.spx
    )
    shopt -u nocaseglob nullglob
    
    if [ ${#files[@]} -gt 0 ]; then
        total=$((total + 1))
    fi
done

i=0

# 7. Process each directory (album) in Artist/Album order
for d in "${dirs[@]}"; do
    shopt -s nocaseglob nullglob
    files=(
        "$d"/*.flac "$d"/*.mp3 "$d"/*.m4a "$d"/*.ogg
        "$d"/*.opus "$d"/*.mp4 "$d"/*.ape "$d"/*.wv "$d"/*.spx
    )
    shopt -u nocaseglob nullglob
    
    if [ ${#files[@]} -gt 0 ]; then
        i=$((i + 1))
        
        artist=$(basename "$(dirname "$d")")
        album=$(basename "$d")
        label="$artist - $album"
        
        # Write header (assume OK; mark FAIL if any format fails)
        echo "OK [$i/$total] $label" | tee -a "$RUN_LOG" "$OKS_LOG"
        
        # Process each audio format separately
        for ext in "${SUPPORTED_EXTS[@]}"; do
            shopt -s nocaseglob nullglob
            group=("$d"/*."$ext")
            shopt -u nocaseglob nullglob
            
            if [ ${#group[@]} -gt 0 ]; then
                # Sort format group naturally
                mapfile -d '' group_sorted < <(printf '%s\0' "${group[@]}" | LC_ALL=C sort -f -z -V)
                
                # Run loudgain with moOde standard flags: -a (album), -k (noclip), -s e (ReplayGain 2.0 + extra tags), -L (force lowercase tags)
                err=$(loudgain -a -k -s e -L -- "${group_sorted[@]}" 2>&1)
                rc=$?
                
                if [ $rc -ne 0 ]; then
                    # Move entry from oks to fails (fixed-string match; labels may contain regex metacharacters)
                    grep -vxF "OK [$i/$total] $label" "$OKS_LOG" > "$OKS_LOG.tmp" 2>/dev/null || true
                    if diff -q "$OKS_LOG" "$OKS_LOG.tmp" >/dev/null 2>&1; then
                        rm -f "$OKS_LOG.tmp"
                    else
                        mv -f "$OKS_LOG.tmp" "$OKS_LOG"
                    fi
                    echo "FAIL [$i/$total] $label" >> "$FAILS_LOG"
                    
                    # Log error details
                    flat=$(echo "$err" | tr -d '\000' | tr '\n' ' ' | tr -s ' ')
                    echo "[$i/$total] ERROR (exit $rc): $label [.$ext] :: $d :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
                fi
            fi
        done
    fi
done

# 8. Count Results
ok_count=$(sort -u "$OKS_LOG" 2>/dev/null | grep -a "^OK" | wc -l)
fail_count=$(sort -u "$FAILS_LOG" 2>/dev/null | grep -a "^FAIL" | wc -l)

# 9. Generate Summary
{
echo "Step 5 Summary"
echo "=============="
echo
echo "Step       : $STEP"
echo "Run Date   : $(date)"
echo
echo "Processed  : $total"
echo "Passed     : $ok_count"
echo "Failed     : $fail_count"
} > "$SUMMARY_LOG"

# 10. Terminal Output
echo
if [ -s "$ERRORS_LOG" ]; then
    echo "----------------------------------------"
    echo "Errors"
    echo "----------------------------------------"
    cat "$ERRORS_LOG"
fi
echo "----------------------------------------"
echo "Processed: $total  Passed: $ok_count  Failed: $fail_count"
echo "----------------------------------------"
echo "Step 5 – Reapply ReplayGain"
echo "----------------------------------------"

```
--- Bash Script Step 5 End ---

\ ---------------------------------------------------------------------------------------

-- Review Results

--- Bash Script Cat 5 Start ---

```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step05"

cat "$LOG_ROOT/${STEP}-summary.log"
cat "$LOG_ROOT/${STEP}-errors.log"
cat "$LOG_ROOT/${STEP}-run.log"
cat "$LOG_ROOT/${STEP}-oks.log"
cat "$LOG_ROOT/${STEP}-fails.log"

```
--- Bash Script Cat 5 End ---

---

10. Step 6 – Repeat Step 1 Integrity Test

---

-- Purpose

This step repeats the integrity test from Step 1 after ReplayGain metadata has been restored. It confirms that the ReplayGain process completed successfully without introducing new integrity problems.

Because ReplayGain only modifies metadata and does not modify the audio stream, this verification confirms the metadata update process preserved structural integrity.

No files are modified during this step.

\ ---------------------------------------------------------------------------------------

--- Bash Script Step 6 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ============================================================
# Step 6 – Post-ReplayGain Integrity Verification
# ============================================================

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step06"

mkdir -p "$LOG_ROOT"

# Software Preflight: fail loudly if a required tool is missing
for tool in flac ffmpeg; do
    if ! command -v "$tool" >/dev/null 2>&1; then
        echo "ERROR: $tool is not installed. Install it and re-run (see Requirements)." >&2
        exit 1
    fi
done

# 1. Define Log Files (Five-File Standard)
RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

# 2. CLEANUP: Delete this step's own logs from any previous run
rm -f "$RUN_LOG" "$OKS_LOG" "$FAILS_LOG" "$ERRORS_LOG" "$SUMMARY_LOG"

# 3. Initialize Empty Log Files
touch "$RUN_LOG" "$OKS_LOG" "$FAILS_LOG" "$ERRORS_LOG" "$SUMMARY_LOG"

# 4. File Discovery
mapfile -d '' files < <(
    find "$PWD" -type f \
        ! -ipath '*/Ignore/*' \
        ! -iname "*.prerepair*" \
        ! -iname "*.fixed.*" \
        ! -iname "*.reencode.*" \
        \( \
            -iname "*.flac" -o \
            -iname "*.mp3"  -o \
            -iname "*.m4a"  -o \
            -iname "*.ogg"  -o \
            -iname "*.opus" -o \
            -iname "*.wav"  -o \
            -iname "*.aiff" -o \
            -iname "*.aif"  -o \
            -iname "*.mp4"  -o \
            -iname "*.ape"  -o \
            -iname "*.wv"   -o \
            -iname "*.spx" \
        \) -print0 2>>"$LOG_ROOT/${STEP}-errors.log" | sort -z
)

total=${#files[@]}
i=0
last_dir=""

for f in "${files[@]}"; do

    ((i++))
    label="${f#"$PWD"/}"
    current_dir="$(dirname "$label")"

    # Insert a blank line on terminal screen when moving to a new folder/album
    if [[ -n "$last_dir" && "$current_dir" != "$last_dir" ]]; then
        echo ""
    fi
    last_dir="$current_dir"

    case "${f,,}" in
        *.flac)
            err=$(flac -s -t "$f" 2>&1)
            rc=$?
            ;;
        *)
            err=$(ffmpeg -nostdin -v error -i "$f" -f null - 2>&1)
            rc=$?
            ;;
    esac

    if [ $rc -eq 0 ]; then
        out_msg="OK   [$i/$total] $label"
        echo "$out_msg"
        echo "$out_msg" >> "$RUN_LOG"
        echo "$out_msg" >> "$OKS_LOG"
    else
        flat=$(printf '%s\n' "$err" | tr '\r\n' ' ' | tr -s ' ')
        out_msg="FAIL [$i/$total] $label"
        echo "$out_msg"
        echo "$out_msg" >> "$RUN_LOG"
        echo "$out_msg" >> "$FAILS_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
    fi

done

# 5. Count Results
ok_count=$(grep -a "^OK" "$RUN_LOG" 2>/dev/null | wc -l)
fail_count=$(grep -a "^FAIL" "$RUN_LOG" 2>/dev/null | wc -l)

# 6. Generate Summary Log
{
echo "Step 6 Summary"
echo "=============="
echo
echo "Step       : $STEP"
echo "Run Date   : $(date)"
echo
echo "Processed  : $total"
echo "Passed     : $ok_count"
echo "Failed     : $fail_count"
} > "$SUMMARY_LOG"

# 7. Terminal Output
echo
if [ -s "$ERRORS_LOG" ]; then
    echo "----------------------------------------"
    echo "Error Summary"
    echo "----------------------------------------"
    awk '
    /LOST_SYNC/ {
        idx = index($0, " :: ")
        if (idx > 0) {
            temp = substr($0, 1, idx - 1)
            pos = index(temp, "): ") + 3
            path = substr(temp, pos)
            lost_sync[path] = 1
        }
    }
    /END_OF_STREAM/ && !/LOST_SYNC/ {
        idx = index($0, " :: ")
        if (idx > 0) {
            temp = substr($0, 1, idx - 1)
            pos = index(temp, "): ") + 3
            path = substr(temp, pos)
            eos[path] = 1
        }
    }
    END {
        if (length(lost_sync) > 0) {
            print "LOST_SYNC"
            print "----------"
            for (p in lost_sync) print p | "sort"
            close("sort")
        }
        if (length(eos) > 0) {
            if (length(lost_sync) > 0) print ""
            print "END_OF_STREAM"
            print "----------"
            for (p in eos) print p | "sort"
            close("sort")
        }
    }
    ' "$ERRORS_LOG"
fi

echo
echo "----------------------------------------"
echo "Processed: $total  Passed: $ok_count  Failed: $fail_count"
echo "----------------------------------------"
echo "Step 6 – Post-ReplayGain Integrity Verification"
echo "----------------------------------------"

```
--- Bash Script Step 6 End ---

\ ---------------------------------------------------------------------------------------

-- Review Results

--- Bash Script Cat 6 Start ---

```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step06"

cat "$LOG_ROOT/${STEP}-summary.log"
cat "$LOG_ROOT/${STEP}-errors.log"
cat "$LOG_ROOT/${STEP}-run.log"
cat "$LOG_ROOT/${STEP}-oks.log"
cat "$LOG_ROOT/${STEP}-fails.log"

```
--- Bash Script Cat 6 End ---

---

11. Step 7 – Remove Loose Files

---

-- Purpose

This step removes temporary files and unwanted artifacts created during the cleanup process. Metadata repair and validation can create incomplete rebuilds, leftover test files, and other artifacts that should not remain in the final library.

This cleanup ensures only the intended music files and required metadata remain before archival preparation.

\ ---------------------------------------------------------------------------------------

--- Bash Script Step 7 Start ---

```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ============================================================
# Step 7 – Remove Loose Files
# ============================================================

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step07"

mkdir -p "$LOG_ROOT"

# 1. Define Log File
REMOVED_LOG="$LOG_ROOT/${STEP}-removed.log"

# 2. CLEANUP: Delete previous log
rm -f "$REMOVED_LOG"

# 3. Find temporary files
find "$PWD" \
    -type f \
    ! -ipath '*/Ignore/*' \
    \( \
        -iname "*.fixed.*" \
        -o -iname "*.prerepair" \
        -o -iname "*.prerepair.flac" \
        -o -iname "*.reencode" \
        -o -iname "*.reencode.flac" \
        -o -iname "*.tmp" \
        -o -iname "*.temp" \
        -o -iname "*~" \
    \) \
    -print > "$REMOVED_LOG"

# 4. Remove and count actually-removed files
count=0
while IFS= read -r f; do
    if rm -f "$f" 2>/dev/null; then
        count=$((count + 1))
    fi
done < "$REMOVED_LOG"

# 5. Terminal Output
echo
echo "----------------------------------------"
echo "Removed: $count files"
echo "----------------------------------------"
echo "Step 7 – Remove Loose Files"
echo "----------------------------------------"

```
--- Bash Script Step 7 End ---

\ ---------------------------------------------------------------------------------------

-- Review Results

--- Bash Script Cat 7 Start ---

```bash
LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step07"

cat "$LOG_ROOT/${STEP}-removed.log"

```
--- Bash Script Cat 7 End ---

---

12. Step 8 – Final Integrity Test

---

-- Purpose

This step performs the final integrity verification of the library after all standard cleanup operations have been completed. It confirms that the complete workflow — metadata cleanup, container rebuilding, ReplayGain restoration, and loose file removal — has resulted in a stable and valid library.

This final verification provides the archival baseline for the repaired library.

No files are modified during this step.

\ ---------------------------------------------------------------------------------------

--- Bash Script Step 8 Start ---

```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ============================================================
# Step 8 – Final Integrity Test
# ============================================================

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step08"

mkdir -p "$LOG_ROOT"

# Software Preflight: fail loudly if a required tool is missing
for tool in flac ffmpeg; do
    if ! command -v "$tool" >/dev/null 2>&1; then
        echo "ERROR: $tool is not installed. Install it and re-run (see Requirements)." >&2
        exit 1
    fi
done

# 1. Define Log Files (Five-File Standard)
RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

# 2. CLEANUP: Delete this step's own logs from any previous run
rm -f "$RUN_LOG" "$OKS_LOG" "$FAILS_LOG" "$ERRORS_LOG" "$SUMMARY_LOG"

# 3. Initialize Empty Log Files
touch "$RUN_LOG" "$OKS_LOG" "$FAILS_LOG" "$ERRORS_LOG" "$SUMMARY_LOG"

# 4. File Discovery
mapfile -d '' files < <(
    find "$PWD" -type f \
        ! -ipath '*/Ignore/*' \
        ! -iname "*.prerepair*" \
        ! -iname "*.fixed.*" \
        ! -iname "*.reencode.*" \
        \( \
            -iname "*.flac" -o \
            -iname "*.mp3"  -o \
            -iname "*.m4a"  -o \
            -iname "*.ogg"  -o \
            -iname "*.opus" -o \
            -iname "*.wav"  -o \
            -iname "*.aiff" -o \
            -iname "*.aif"  -o \
            -iname "*.mp4"  -o \
            -iname "*.ape"  -o \
            -iname "*.wv"   -o \
            -iname "*.spx" \
        \) -print0 2>>"$LOG_ROOT/${STEP}-errors.log" | sort -z
)

total=${#files[@]}
i=0
last_dir=""

for f in "${files[@]}"; do

    ((i++))
    label="${f#"$PWD"/}"
    current_dir="$(dirname "$label")"

    # Insert a blank line on terminal screen when moving to a new folder/album
    if [[ -n "$last_dir" && "$current_dir" != "$last_dir" ]]; then
        echo ""
    fi
    last_dir="$current_dir"

    case "${f,,}" in
        *.flac)
            err=$(flac -s -t "$f" 2>&1)
            rc=$?
            ;;
        *)
            err=$(ffmpeg -nostdin -v error -i "$f" -f null - 2>&1)
            rc=$?
            ;;
    esac

    if [ $rc -eq 0 ]; then
        out_msg="OK   [$i/$total] $label"
        echo "$out_msg"
        echo "$out_msg" >> "$RUN_LOG"
        echo "$out_msg" >> "$OKS_LOG"
    else
        flat=$(printf '%s\n' "$err" | tr '\r\n' ' ' | tr -s ' ')
        out_msg="FAIL [$i/$total] $label"
        echo "$out_msg"
        echo "$out_msg" >> "$RUN_LOG"
        echo "$out_msg" >> "$FAILS_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
    fi

done

# 5. Count Results
ok_count=$(grep -a "^OK" "$RUN_LOG" 2>/dev/null | wc -l)
fail_count=$(grep -a "^FAIL" "$RUN_LOG" 2>/dev/null | wc -l)

# 6. Generate Summary Log
{
echo "Step 8 Summary"
echo "=============="
echo
echo "Step       : $STEP"
echo "Run Date   : $(date)"
echo
echo "Processed  : $total"
echo "Passed     : $ok_count"
echo "Failed     : $fail_count"
} > "$SUMMARY_LOG"

# 7. Terminal Output
echo
if [ -s "$ERRORS_LOG" ]; then
    echo "----------------------------------------"
    echo "Error Summary"
    echo "----------------------------------------"
    awk '
    /LOST_SYNC/ {
        idx = index($0, " :: ")
        if (idx > 0) {
            temp = substr($0, 1, idx - 1)
            pos = index(temp, "): ") + 3
            path = substr(temp, pos)
            lost_sync[path] = 1
        }
    }
    /END_OF_STREAM/ && !/LOST_SYNC/ {
        idx = index($0, " :: ")
        if (idx > 0) {
            temp = substr($0, 1, idx - 1)
            pos = index(temp, "): ") + 3
            path = substr(temp, pos)
            eos[path] = 1
        }
    }
    END {
        if (length(lost_sync) > 0) {
            print "LOST_SYNC"
            print "----------"
            for (p in lost_sync) print p | "sort"
            close("sort")
        }
        if (length(eos) > 0) {
            if (length(lost_sync) > 0) print ""
            print "END_OF_STREAM"
            print "----------"
            for (p in eos) print p | "sort"
            close("sort")
        }
    }
    ' "$ERRORS_LOG"
fi

echo
echo "----------------------------------------"
echo "Processed: $total  Passed: $ok_count  Failed: $fail_count"
echo "----------------------------------------"
echo "Step 8 – Final Integrity Test"
echo "----------------------------------------"

```
--- Bash Script Step 8 End ---

\ ---------------------------------------------------------------------------------------


-- Review Results

--- Bash Script Cat 8 Start ---

```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step08"

cat "$LOG_ROOT/${STEP}-summary.log"
cat "$LOG_ROOT/${STEP}-errors.log"
cat "$LOG_ROOT/${STEP}-run.log"
cat "$LOG_ROOT/${STEP}-oks.log"
cat "$LOG_ROOT/${STEP}-fails.log"

```
--- Bash Script Cat 8 End ---

---

13. Optional Procedures

---

The following procedures are not required for a standard library cleanup but may be necessary for resolving stubborn errors, standardizing visual presentation, or preparing the library for long-term preservation.

Note: Run these optional procedures before Step 7 if you want Step 7 to clean up their working files (Step 7 removes `*.prerepair`, `*.reencode`, `*.fixed.*`, `*.tmp`, `*.temp`, and `*~`, plus legacy `*.prerepair.flac`/`*.reencode.flac` names), or simply re-run Step 7 after them.

\ ---------------------------------------------------------------------------------------

## 13a. Strip Problematic Metadata

This procedure is a "nuclear option" for files that continue to fail integrity testing even after the standard deduplication and container rebuilding steps.

Sometimes, FLAC files contain deeply corrupted non-text metadata blocks—such as malformed embedded images, corrupted padding blocks, or broken seek tables—that prevent standard tools from reading the file correctly. This step rescues the file by backing up the text tags, completely obliterating all metadata blocks, and then restoring only the clean text tags.

-- What It Does

This step:

* Scans the selected location for FLAC files.

* Exports all valid text-based Vorbis comments (Artist, Album, Track, etc.) to a temporary file.

* Uses metaflac --remove-all to strip every metadata block from the file (including embedded art, padding, and seek tables) except for the required STREAMINFO block.

* Re-imports the clean text tags back into the file.

* Leaves the underlying audio stream completely untouched.

Because this step removes embedded artwork, you should run Step 13c "Update Album Artwork Embeds" afterward if you rely on embedded images.

\ ---------------------------------------------------------------------------------------

--- Bash Script for 13a Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# 13a. Strip Problematic Metadata
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step13a-errors.log"

# Software Preflight: fail loudly if a required tool is missing
if ! command -v metaflac >/dev/null 2>&1; then
    echo "ERROR: metaflac is not installed. Install the flac package and re-run (see Requirements)." >&2
    exit 1
fi

mapfile -d '' files < <(
    find "$PWD" -type f ! -ipath '*/Ignore/*' -name "*.flac" -print0 | sort -z
)

total=${#files[@]}
i=0

for f in "${files[@]}"; do
    i=$((i+1))

    artist=$(basename "$(dirname "$(dirname "$f")")")
    album=$(basename "$(dirname "$f")")
    track=$(basename "$f" .flac)

    label="$artist-$album-$track"

    tags=$(mktemp "$LOG_ROOT/step13a-tags.XXXXXX")

    # Export text tags before wiping (fail loudly if the export fails)
    exp_err=$(metaflac --export-tags-to="$tags" "$f" 2>&1 >/dev/null)
    exp_rc=$?

    if [ $exp_rc -ne 0 ]; then
        flat=$(echo "$exp_err" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (exit $exp_rc, tag export): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step13a-errors.log"
        rm -f "$tags"
        continue
    fi

    # Strip ALL metadata blocks (pictures, padding, seektables, tags)
    err=$(metaflac --remove-all "$f" 2>&1 >/dev/null)
    rc=$?

    if [ $rc -ne 0 ]; then

        flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')

        echo "FAIL [$i/$total] $label"

        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step13a-errors.log"

    else
        # Re-import the clean text tags
        err2=$(metaflac --import-tags-from="$tags" "$f" 2>&1 >/dev/null)
        rc2=$?

        # Add a standard 8KB padding block back for future tag editing efficiency
        err3=$(metaflac --add-padding=8192 "$f" 2>&1 >/dev/null)
        rc3=$?

        if [ $rc2 -ne 0 ] || [ $rc3 -ne 0 ]; then

            flat=$(echo "$err2 $err3" | tr '\n' ' ' | tr -s ' ')

            echo "FAIL [$i/$total] $label"

            echo "[$i/$total] ERROR (re-import/padding): $label :: $f :: ${flat:-no stderr output}" \
                >> "$LOG_ROOT/step13a-errors.log"

        else

            echo "FIXED [$i/$total] $label"

        fi

    fi

    rm -f "$tags"

done | tee "$LOG_ROOT/step13a-run.log"
echo
echo "----------------------------------------"
echo "13a. Strip Problematic Metadata"
echo "----------------------------------------"

```
--- Bash Script for 13a End ---

\-------------------------------------------------------------------

-- Separate Results

After the stripping process completes, separate successful and failed results:

--- Bash Script Results for 13a Start ---
```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"

grep '^FIXED' "$LOG_ROOT/step13a-run.log" \
    > "$LOG_ROOT/step13a-fixed.log"

grep '^FAIL' "$LOG_ROOT/step13a-run.log" \
    > "$LOG_ROOT/step13a-fails.log"

echo "Step 13a FIXED: $(wc -l < "$LOG_ROOT/step13a-fixed.log")  FAILs: $(wc -l < "$LOG_ROOT/step13a-fails.log")"

```
--- Bash Script Results for 13a End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 13a Start ---
```bash

cat "$LOG_ROOT/step13a-errors.log"

cat "$LOG_ROOT/step13a-run.log"

cat "$LOG_ROOT/step13a-fixed.log"

cat "$LOG_ROOT/step13a-fails.log"

```
--- Bash Script Cat for 13a End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

    step13a-run.log — Complete processing results.

    step13a-fixed.log — Files that had their metadata completely rebuilt.

    step13a-fails.log — Files that could not be processed.

    step13a-errors.log — Detailed error output.

After running this on stubborn files, you should run the integrity test (flac -t) on them again. If they pass, the corruption was isolated to a non-audio metadata block.

\-------------------------------------------------------------------
## 13b: Normalize Album Artwork (FLAC Only—Reference / Legacy)

**Note:** Step 13c "Update Album Artwork Embeds" supersedes this procedure and covers FLAC, MP3, M4A, and MP4. Use Step 13c unless you specifically need FLAC-only metaflac-based embed logic.

-- Purpose

This procedure standardizes how album artwork is stored within your FLAC files using metaflac.

If you performed the "Strip Problematic Metadata" procedure (Step 13a), any previously embedded artwork was destroyed to save the container. Additionally, over years of collection, a library can accumulate wildly inconsistent artwork—some files having no art, others having massive 10MB uncompressed PNGs, and others having multiple conflicting images.

This step ensures every FLAC file in an album contains the exact same, standardized cover image by reading a local image file (like cover.jpg or folder.jpg) stored in the album directory and embedding it cleanly into the audio files.

-- What It Does

This step (FLAC only):

* Scans the library by album directory.
* Looks for a standard image file named cover.jpg or folder.jpg in each directory.
* If a standard image is found, it removes any existing, potentially corrupt or oversized artwork from the FLAC files in that directory.
* Embeds the standard image into each FLAC file using metaflac.
* Leaves the audio data completely unchanged.
* Skips directories that do not contain a recognized standard image file. 

\-------------------------------------------------------------------

--- Bash Script for 13b Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# 13b. Normalize Album Artwork
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step13b-errors.log"

# Software Preflight: fail loudly if a required tool is missing
if ! command -v metaflac >/dev/null 2>&1; then
    echo "ERROR: metaflac is not installed. Install the flac package and re-run (see Requirements)." >&2
    exit 1
fi

# Moode-standard folder-level cover file priority (matches moOde's coverart.php parseFolder())
COVER_CANDIDATES=(
    "Cover.jpg" "cover.jpg" "Cover.jpeg" "cover.jpeg" "Cover.png" "cover.png"
    "Folder.jpg" "folder.jpg" "Folder.jpeg" "folder.jpeg" "Folder.png" "folder.png"
)

find_cover_art() {
    local dir="$1"
    for name in "${COVER_CANDIDATES[@]}"; do
        if [ -s "$dir/$name" ]; then
            echo "$dir/$name"
            return 0
        fi
    done
    return 1
}

mapfile -d '' dirs < <(
    find "$PWD" -type d ! -ipath '*/Ignore/*' -print0 | sort -z
)

total=0

for d in "${dirs[@]}"; do
    art_file=$(find_cover_art "$d")

    if [ -n "$art_file" ]; then
        shopt -s nullglob
        flac_files=("$d"/*.flac)
        shopt -u nullglob

        if [ ${#flac_files[@]} -gt 0 ]; then
            total=$((total+1))
        fi
    fi
done

i=0

for d in "${dirs[@]}"; do
    art_file=$(find_cover_art "$d")

    if [ -n "$art_file" ]; then
        shopt -s nullglob
        flac_files=("$d"/*.flac)
        shopt -u nullglob

        if [ ${#flac_files[@]} -gt 0 ]; then
            i=$((i+1))

            artist=$(basename "$(dirname "$d")")
            album=$(basename "$d")
            label="$artist-$album"

            error_found=0

            for f in "${flac_files[@]}"; do
                rmerr=$(metaflac --remove --block-type=PICTURE "$f" 2>&1 >/dev/null)
                rmrc=$?

                if [ $rmrc -ne 0 ]; then
                    error_found=1
                    rmflat=$(echo "$rmerr" | tr '\n' ' ' | tr -s ' ')

                    echo "[$i/$total] ERROR (exit $rmrc, remove-picture): $label :: $(basename "$f") :: ${rmflat:-no stderr output}" \
                        >> "$LOG_ROOT/step13b-errors.log"
                fi

                err=$(metaflac --import-picture-from="$art_file" "$f" 2>&1 >/dev/null)
                rc=$?

                if [ $rc -ne 0 ]; then
                    error_found=1
                    flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')

                    echo "[$i/$total] ERROR (exit $rc): $label :: $(basename "$f") :: ${flat:-no stderr output}" \
                        >> "$LOG_ROOT/step13b-errors.log"
                fi
            done

            if [ $error_found -eq 0 ]; then
                echo "OK [$i/$total] $label (Embedded $(basename "$art_file"))"
            else
                echo "FAIL [$i/$total] $label"
            fi
        fi
    fi
done | tee "$LOG_ROOT/step13b-run.log"
echo
echo "----------------------------------------"
echo "13b. Normalize Album Artwork"
echo "----------------------------------------"

```
--- Bash Script for 13b End ---

\-------------------------------------------------------------------

-- Separate Results

After the artwork normalization completes, separate successful and failed results:

--- Bash Script Results for 13b Start ---
```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"

grep '^OK' "$LOG_ROOT/step13b-run.log" > "$LOG_ROOT/step13b-oks.log"
grep '^FAIL' "$LOG_ROOT/step13b-run.log" > "$LOG_ROOT/step13b-fails.log"

echo "Step 13b OKs: $(wc -l < "$LOG_ROOT/step13b-oks.log")  FAILs: $(wc -l < "$LOG_ROOT/step13b-fails.log")"

```
--- Bash Script Results for 13b End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 13b Start ---
```bash

cat "$LOG_ROOT/step13b-errors.log"
cat "$LOG_ROOT/step13b-run.log"
cat "$LOG_ROOT/step13b-oks.log"
cat "$LOG_ROOT/step13b-fails.log"

```
--- Bash Script Cat for 13b End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step13b-run.log — Complete processing results for directories containing artwork.
* step13b-oks.log — Albums successfully updated with standardized artwork.
* step13b-fails.log — Albums where metaflac encountered an error embedding the image.
* step13b-errors.log — Detailed error output from failed embeds.

Albums without a cover.jpg or folder.jpg are simply ignored by this script. To process them, place an appropriately sized JPEG in their directory and re-run the script.

\-------------------------------------------------------------------

## 13c. Update Album Artwork Embeds (FLAC, MP3, M4A, MP4)

**Primary Step for Artwork Embed Operations**

-- Purpose

This procedure standardizes album artwork across the formats this script can safely embed: FLAC, MP3, M4A, and MP4. This is the recommended multi-format replacement for the legacy Step 13b.

OGG, Opus, AIFF, AIF, APE, and DSF are NOT embedded by this script: ffmpeg cannot attach cover art to OGG/Opus (their native containers store art as a `METADATA_BLOCK_PICTURE` Vorbis comment instead of a picture stream), and the AIFF/APE/DSF muxers in ffmpeg silently drop or reject picture streams. Files in those formats are detected and skipped with a `SKIP` line — never modified.

If you performed the "Strip Problematic Metadata" procedure (Step 13a), any previously embedded artwork was destroyed to save the container. Additionally, over years of collection, a library can accumulate wildly inconsistent artwork—some files having no art, others having massive 10MB uncompressed PNGs, and others having multiple conflicting images.

This step ensures every file in an album contains the exact same, standardized cover image by reading a local image file (like cover.jpg or folder.jpg) stored in the album directory and embedding it appropriately for each format's native tag structure.

-- What It Does

This step:

* Scans the library by album directory across all supported formats.
* Looks for a standard image file named cover.jpg, folder.jpg, or other Moode-standard formats in each directory.
* For FLAC files: Uses metaflac to remove old artwork and embed the new image.
* For non-FLAC formats (MP3, M4A, MP4): Uses ffmpeg to attach the artwork as a cover picture stream without re-encoding audio. OGG, Opus, AIFF, APE, and DSF files are logged as `SKIP` and left unchanged.
* Leaves the underlying audio data completely unchanged (audio streams are never re-encoded).
* Skips directories that do not contain a recognized standard image file.

\-------------------------------------------------------------------

--- Bash Script for 13c Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# 13c. Update Album Artwork Embeds (FLAC, MP3, M4A, MP4)
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step13c-errors.log"

if ! command -v ffmpeg >/dev/null 2>&1; then
    echo "ERROR: ffmpeg is required to process non-FLAC formats."
    exit 1
fi

HAS_METAFLAC=0
if command -v metaflac >/dev/null 2>&1; then
    HAS_METAFLAC=1
fi

# Moode-standard folder-level cover file priority (matches moOde's coverart.php parseFolder())
COVER_CANDIDATES=(
    "Cover.jpg" "cover.jpg" "Cover.jpeg" "cover.jpeg" "Cover.png" "cover.png"
    "Folder.jpg" "folder.jpg" "Folder.jpeg" "folder.jpeg" "Folder.png" "folder.png"
)

find_cover_art() {
    local dir="$1"
    for name in "${COVER_CANDIDATES[@]}"; do
        if [ -s "$dir/$name" ]; then
            echo "$dir/$name"
            return 0
        fi
    done
    return 1
}

mapfile -d '' dirs < <(
    find "$PWD" -type f ! -ipath '*/Ignore/*' \( \
        -iname "*.flac" -o -iname "*.mp3" -o -iname "*.m4a" -o \
        -iname "*.mp4"  -o -iname "*.ogg" -o -iname "*.opus" -o \
        -iname "*.aiff" -o -iname "*.aif" -o -iname "*.ape"  -o \
        -iname "*.dsf" \
    \) -printf '%h\0' | sort -u -z
)

total=${#dirs[@]}
i=0

if [ "$total" -eq 0 ]; then
    echo "No directories with supported audio files found."
    exit 0
fi

for d in "${dirs[@]}"; do
    i=$((i + 1))

    parent_dir="${d%/*}"
    artist="${parent_dir##*/}"
    album="${d##*/}"
    label="$artist - $album"
    error_found=0

    art_file=$(find_cover_art "$d")

    if [ -z "$art_file" ]; then
        echo "ERROR [$i/$total] $label :: Missing standard image file"
        echo "[$i/$total] ERROR: $label :: No Moode-standard cover image found (checked: ${COVER_CANDIDATES[*]})" >> "$LOG_ROOT/step13c-errors.log"
        continue
    fi

    shopt -s nullglob nocaseglob
    audio_files=(
        "$d"/*.flac "$d"/*.mp3 "$d"/*.m4a "$d"/*.mp4 \
        "$d"/*.ogg  "$d"/*.opus "$d"/*.aiff "$d"/*.aif \
        "$d"/*.ape  "$d"/*.dsf
    )
    shopt -u nullglob nocaseglob

    mapfile -t audio_files < <(printf "%s\n" "${audio_files[@]}" | sort -u)

    if [ ${#audio_files[@]} -eq 0 ]; then
        echo "ERROR [$i/$total] $label :: No audio files found"
        echo "[$i/$total] ERROR: $label :: Directory has no supported audio files" >> "$LOG_ROOT/step13c-errors.log"
        continue
    fi

    processed_any=0

    for f in "${audio_files[@]}"; do
        fname=$(basename "$f")
        ext="${fname##*.}"
        ext_lower=$(echo "$ext" | tr '[:upper:]' '[:lower:]')

        case "$ext_lower" in
            flac|mp3|m4a|mp4) ;;
            *)
                echo "SKIP [$i/$total] $label :: ${f##*/} ($ext_lower artwork embed not supported; file left unchanged)"
                continue
                ;;
        esac
        processed_any=1

        if [[ "$ext_lower" == "flac" && $HAS_METAFLAC -eq 1 ]]; then
            rmerr=$(metaflac --remove --block-type=PICTURE "$f" 2>&1)
            rmrc=$?
            if [ $rmrc -ne 0 ]; then
                error_found=1
                rmflat=$(echo "$rmerr" | tr '\n' ' ' | tr -s ' ')
                echo "[$i/$total] ERROR (exit $rmrc, remove-picture): $label :: ${f##*/} :: ${rmflat:-no stderr output}" \
                    >> "$LOG_ROOT/step13c-errors.log"
            fi

            err=$(metaflac --import-picture-from="$art_file" "$f" 2>&1)
            rc=$?
            if [ $rc -ne 0 ]; then
                error_found=1
                flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
                echo "[$i/$total] ERROR (exit $rc, import-art): $label :: ${f##*/} :: ${flat:-no stderr output}" \
                    >> "$LOG_ROOT/step13c-errors.log"
            fi
        else
            temp_file=$(mktemp "$LOG_ROOT/step13c-tagged.XXXXXX.${ext_lower}")
            rm -f "$temp_file" 2>/dev/null

            err=$(ffmpeg -y -nostdin -loglevel error -i "$f" -i "$art_file" \
                -map 0:a -map 1 -c copy -disposition:v attached_pic "$temp_file" 2>&1)
            rc=$?

            if [ $rc -eq 0 ] && [ -s "$temp_file" ]; then
                mv -f "$temp_file" "$f" 2>/dev/null
            else
                error_found=1
                rm -f "$temp_file" 2>/dev/null
                flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
                echo "[$i/$total] ERROR (exit $rc, ffmpeg): $label :: ${f##*/} :: ${flat:-no stderr output}" \
                    >> "$LOG_ROOT/step13c-errors.log"
            fi
        fi
    done

    if [ $error_found -eq 0 ] && [ $processed_any -gt 0 ]; then
        echo "OK    [$i/$total] $label"
    elif [ $processed_any -eq 0 ]; then
        echo "SKIP  [$i/$total] $label"
        echo "[$i/$total] SKIP: $label :: no embeddable audio files (FLAC/MP3/M4A/MP4) in this directory" \
            >> "$LOG_ROOT/step13c-errors.log"
    else
        echo "ERROR [$i/$total] $label"
    fi

done | tee "$LOG_ROOT/step13c-run.log"
echo
echo "----------------------------------------"
echo "13c. Update Album Artwork Embeds (FLAC, MP3, M4A, MP4)"
echo "----------------------------------------"

```
--- Bash Script for 13c End ---

\-------------------------------------------------------------------

-- Separate Results

--- Bash Script Results for 13c Start ---
```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"

grep '^OK' "$LOG_ROOT/step13c-run.log" > "$LOG_ROOT/step13c-oks.log"
grep '^ERROR' "$LOG_ROOT/step13c-run.log" > "$LOG_ROOT/step13c-fails.log"
grep '^SKIP' "$LOG_ROOT/step13c-run.log" > "$LOG_ROOT/step13c-skips.log"

echo "Step 13c Universal OKs: $(wc -l < "$LOG_ROOT/step13c-oks.log")  SKIPs: $(wc -l < "$LOG_ROOT/step13c-skips.log")  ERRORs: $(wc -l < "$LOG_ROOT/step13c-fails.log")"

```
--- Bash Script Results for 13c End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 13c Start ---
```bash

cat "$LOG_ROOT/step13c-errors.log"
cat "$LOG_ROOT/step13c-run.log"
cat "$LOG_ROOT/step13c-oks.log"
cat "$LOG_ROOT/step13c-fails.log"
cat "$LOG_ROOT/step13c-skips.log"

```
--- Bash Script Cat for 13c End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step13c-run.log — Complete processing results for all directories with supported audio files.
* step13c-oks.log — Directories successfully updated with standardized artwork across all formats.
* step13c-fails.log — Directories where artwork embedding failed (usually due to missing cover image).
* step13c-skips.log — Files or directories skipped because their format cannot carry embedded artwork through this script (OGG, Opus, AIFF, APE, DSF).
* step13c-errors.log — Detailed error output from metaflac (FLAC) or ffmpeg (other formats).

Directories without a standard cover image (Cover.jpg, cover.jpg, Folder.jpg, folder.jpg, etc.) are logged as errors and require manual intervention—place an appropriately sized JPEG in the directory and re-run the script.

\-------------------------------------------------------------------

## 13d. Deep Repair via Decode/Re-encode (Last Resort)

-- Purpose

This procedure is a final rescue option for FLAC files that continue to fail integrity testing after all standard cleanup steps (Steps 1-8) and optional procedures (13a-13c) have been completed.

When a FLAC file contains deeply corrupted audio frames (not just metadata), the only way to salvage it is to decode the audio stream to PCM, then re-encode it as a new FLAC file. This process rebuilds the audio container from scratch, eliminating frame-level corruption while preserving the original audio data.

-- What It Does

This step:

* Tests each FLAC file for integrity using `flac -t`.
* Skips files that already pass the integrity test.
* For failing files: exports the Vorbis comment tags to a temporary file.
* Decodes the audio with ffmpeg and re-encodes as a fresh FLAC file.
* Verifies the newly encoded file passes `flac -t`.
* Re-imports the original tags into the repaired file.
* Creates a backup of the original file as `FILE.prerepair` (saved only once per file).
* Replaces the original with the repaired version.
* Classifies results as FIXED-CLEAN (no warnings) or FIXED-REVIEW (ffmpeg reported warnings during decode).
* Leaves audio data integrity intact while removing frame-level corruption.

**Warning:** This procedure re-encodes the audio stream. While FLAC re-encoding is lossless, this should only be used as a last resort for files that cannot be recovered any other way.

**Caution:** The `.prerepair` backup is a working copy, not archival. Step 7 removes `*.prerepair` files automatically as part of its loose-file cleanup, so the backup only survives until the next Step 7 run. Keep it only until you have verified the repaired file (Step 1 passes and playback is correct), then delete it or move it outside the library.

\-------------------------------------------------------------------

--- Bash Script for 13d Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# 13d. Deep Repair via Decode/Re-encode (Last Resort)
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step13d-errors.log"
: > "$LOG_ROOT/step13d-review.log"

# Software Preflight: fail loudly if a required tool is missing
for tool in flac metaflac ffmpeg; do
    if ! command -v "$tool" >/dev/null 2>&1; then
        echo "ERROR: $tool is not installed. Install it and re-run (see Requirements)." >&2
        exit 1
    fi
done

mapfile -d '' files < <(
    find "$PWD" -type f ! -ipath '*/Ignore/*' -name "*.flac" \
        ! -iname "*.prerepair*" \
        ! -iname "*.reencode*" \
        ! -iname "*.fixed.*" \
        -print0 | sort -z
)

total=${#files[@]}
i=0

for f in "${files[@]}"; do
    i=$((i+1))

    artist=$(basename "$(dirname "$(dirname "$f")")")
    album=$(basename "$(dirname "$f")")
    track=$(basename "$f" .flac)

    label="$artist-$album-$track"

    testerr=$(flac -s -t "$f" 2>&1 >/dev/null)
    testrc=$?

    if [ $testrc -eq 0 ]; then
        echo "OK [$i/$total] $label"
        continue
    fi

    tags=$(mktemp "$LOG_ROOT/step13d-tags.XXXXXX")

    tagerr=$(metaflac --export-tags-to="$tags" "$f" 2>&1 >/dev/null)
    tagrc=$?

    if [ $tagrc -ne 0 ]; then
        flat=$(echo "$tagerr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (exit $tagrc, tag export): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step13d-errors.log"
        rm -f "$tags"
        continue
    fi

    # Save the first embedded picture so it can be restored after the re-encode
    pic=$(mktemp "$LOG_ROOT/step13d-pic.XXXXXX")
    rm -f "$pic"
    if metaflac --export-picture-to="$pic" "$f" >/dev/null 2>&1 && [ -s "$pic" ]; then
        had_pic=1
    else
        had_pic=0
        rm -f "$pic"
    fi

    reerr=$(ffmpeg -nostdin -nostats -loglevel warning \
        -i "$f" \
        -map 0:a:0 \
        -c:a flac \
        -f flac \
        "${f}.reencode" \
        -y 2>&1 >/dev/null)
    rerc=$?

    if [ $rerc -ne 0 ]; then
        flat=$(echo "$reerr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (exit $rerc, reencode): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step13d-errors.log"
        rm -f "$tags" "$pic" "${f}.reencode"
        continue
    fi

    posterr=$(flac -s -t "${f}.reencode" 2>&1 >/dev/null)
    postrc=$?

    if [ $postrc -ne 0 ]; then
        flat=$(echo "$posterr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (exit $postrc, post-reencode test): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step13d-errors.log"
        rm -f "$tags" "$pic" "${f}.reencode"
        continue
    fi

    # Rebuild tags from scratch (--import-tags-from APPENDS; it will not replace)
    impterr=$(metaflac --remove-all-tags "${f}.reencode" 2>&1 >/dev/null)
    imprc=$?
    if [ $imprc -eq 0 ]; then
        impterr=$(metaflac --import-tags-from="$tags" "${f}.reencode" 2>&1 >/dev/null)
        imprc=$?
    fi
    if [ $imprc -eq 0 ] && [ "$had_pic" -eq 1 ]; then
        impterr=$(metaflac --import-picture-from="$pic" "${f}.reencode" 2>&1 >/dev/null)
        imprc=$?
    fi

    if [ $imprc -ne 0 ]; then
        flat=$(echo "$impterr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (exit $imprc, tag/picture reimport): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step13d-errors.log"
        rm -f "$tags" "$pic" "${f}.reencode"
        continue
    fi

    # Preserve the original once; suffix is non-FLAC so moOde never indexes it
    if [ ! -e "${f}.prerepair" ]; then
        if ! cp "$f" "${f}.prerepair" >/dev/null 2>&1; then
            echo "FAIL [$i/$total] $label"
            echo "[$i/$total] ERROR (backup failed): $label :: $f :: could not create ${f}.prerepair" \
                >> "$LOG_ROOT/step13d-errors.log"
            rm -f "$tags" "$pic" "${f}.reencode"
            continue
        fi
    fi

    if ! mv -f "${f}.reencode" "$f" >/dev/null 2>&1; then
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (mv failed): $label :: $f :: could not move rebuilt file into place" \
            >> "$LOG_ROOT/step13d-errors.log"
        rm -f "$tags" "$pic" "${f}.reencode"
        continue
    fi
    rm -f "$tags" "$pic"

    if [ -n "$reerr" ]; then
        flat=$(echo "$reerr" | tr '\n' ' ' | tr -s ' ')
        echo "FIXED-REVIEW [$i/$total] $label"
        echo "[$i/$total] REVIEW $label :: $f :: ffmpeg reported during decode: $flat" \
            >> "$LOG_ROOT/step13d-review.log"
    else
        echo "FIXED-CLEAN [$i/$total] $label"
    fi

done | tee "$LOG_ROOT/step13d-run.log"
echo
echo "----------------------------------------"
echo "13d. Deep Repair via Decode/Re-encode (Last Resort)"
echo "----------------------------------------"

```
--- Bash Script for 13d End ---

\-------------------------------------------------------------------

-- Separate Results

--- Bash Script Results for 13d Start ---
```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"

grep '^OK' "$LOG_ROOT/step13d-run.log" \
    > "$LOG_ROOT/step13d-oks.log"

grep '^FIXED-CLEAN' "$LOG_ROOT/step13d-run.log" \
    > "$LOG_ROOT/step13d-fixed-clean.log"

grep '^FIXED-REVIEW' "$LOG_ROOT/step13d-run.log" \
    > "$LOG_ROOT/step13d-fixed-review.log"

grep '^FAIL' "$LOG_ROOT/step13d-run.log" \
    > "$LOG_ROOT/step13d-fails.log"

echo "Step 13d OKs: $(wc -l < "$LOG_ROOT/step13d-oks.log")  FIXED-CLEAN: $(wc -l < "$LOG_ROOT/step13d-fixed-clean.log")  FIXED-REVIEW: $(wc -l < "$LOG_ROOT/step13d-fixed-review.log")  FAILs: $(wc -l < "$LOG_ROOT/step13d-fails.log")"

```
--- Bash Script Results for 13d End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 13d Start ---
```bash

cat "$LOG_ROOT/step13d-errors.log"
cat "$LOG_ROOT/step13d-review.log"
cat "$LOG_ROOT/step13d-run.log"
cat "$LOG_ROOT/step13d-oks.log"
cat "$LOG_ROOT/step13d-fixed-clean.log"
cat "$LOG_ROOT/step13d-fixed-review.log"
cat "$LOG_ROOT/step13d-fails.log"

```
--- Bash Script Cat for 13d End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step13d-run.log — Complete processing results for all FLAC files.
* step13d-oks.log — Files that already passed `flac -t` (no repair needed).
* step13d-fixed-clean.log — Files successfully repaired with no warnings during re-encode.
* step13d-fixed-review.log — Files successfully repaired but ffmpeg reported warnings; these warrant manual playback testing.
* step13d-fails.log — Files that could not be repaired and remain unchanged.
* step13d-errors.log — Detailed error output from failed repair attempts.
* step13d-review.log — Decode-time warnings from ffmpeg for FIXED-REVIEW files.
* `FILE.prerepair` backups — Original versions of successfully repaired files, kept for review. Note that Step 7 deletes `*.prerepair` automatically as loose files, so if you want to keep one, move it outside the library before running Step 7 (see the caution in the procedure above).

After running this procedure, verify the results:
1. Listen to a few tracks from the FIXED-REVIEW list to ensure playback quality.
2. Run Step 1 (Initial Integrity Test) on the fixed files to confirm they now pass.
3. If FIXED-REVIEW files sound correct, the repair is complete. Delete the `.prerepair` backups if you're confident in the results.

\-------------------------------------------------------------------

## 13e. Verify Tags Against Filenames (Failsafe)

-- Purpose

This procedure is a backup/failsafe verification that confirms every tag was updated correctly after the cleanup pipeline (Steps 1-8) and the other optional procedures have run. It catches any file whose embedded metadata still disagrees with the folder and filename convention — for example, a title that was not written, a track number placed on the wrong file, or an artist/album tag that was left stale.

It is read-only: it never modifies any file. It only reports. Any findings are fixed separately (see "Fixing Findings" below), after which you re-run Step 14 to regenerate the checksums that the tag edits invalidate.

-- What It Does

This step:

* Scans the given music root recursively for FLAC, MP3, and M4A files.
* Derives the expected Artist (parent folder), Album Year / Album name (album folder name, `YYYY Album`), and Track number / Title (filename, `NN - Title`) from the naming convention.
* Reads the actual embedded tags from each file with ffprobe.
* Compares them case-insensitively and with whitespace collapsed, so harmless capitalization differences are not flagged.
* Flags any file where a tag is missing or disagrees with the filename/folder, writing one line per finding.
* Files whose name has no `NN - Title` prefix are reported as UNPARSEABLE rather than guessed.
* Writes a `SUMMARY:` tally of files checked vs. mismatches found.

-- How to Run

Run from the terminal against the library root (or an artist folder for a targeted check):

--- Bash Script for 13e Start ---
```bash

#!/usr/bin/env bash

trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# 13e. Verify Tags Against Filenames (Failsafe)
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"

TARGET="${1:?Usage: $0 /path/to/music/root}"

RUN_LOG="$LOG_ROOT/step13e-run.log"
MISMATCH_LOG="$LOG_ROOT/step13e-mismatches.log"

: > "$RUN_LOG"
: > "$MISMATCH_LOG"

norm() { tr '[:upper:]' '[:lower:]' | tr -s '[:space:]' ' '; }

echo "========== Step 13e: Verify Tags Against Filenames ==========" | tee -a "$RUN_LOG"
echo "Root: $TARGET" | tee -a "$RUN_LOG"
echo "Started: $(date)" | tee -a "$RUN_LOG"
echo

checked=0
mismatched=0

while IFS= read -r -d '' filepath; do
    checked=$((checked+1))
    filename=$(basename "$filepath")
    rel="${filepath#"$TARGET"/}"
    name_no_ext="${filename%.*}"

    parent_dir=$(dirname "$filepath")
    album_dir=$(basename "$parent_dir")
    artist_dir=$(basename "$(dirname "$parent_dir")")

    album_year=""
    album_name="$album_dir"
    if [[ "$album_dir" =~ ^([0-9]{4})[[:space:]]+(.+)$ ]]; then
        album_year="${BASH_REMATCH[1]}"
        album_name="${BASH_REMATCH[2]}"
    fi

    if [[ "$name_no_ext" =~ ^([0-9]+)[[:space:]]+-?[[:space:]]+(.+)$ ]]; then
        want_track_str="${BASH_REMATCH[1]}"
        want_track=$(( 10#${want_track_str} ))
        want_title="${BASH_REMATCH[2]}"
    else
        echo "UNPARSEABLE|$rel|filename has no \"NN - Title\"" >> "$MISMATCH_LOG"
        mismatched=$((mismatched+1))
        continue
    fi

    json=$(ffprobe -v error -print_format json -show_format "$filepath" 2>/dev/null)
    [ -z "$json" ] && { echo "NOFFPROBE|$rel|ffprobe could not read tags" >> "$MISMATCH_LOG"; mismatched=$((mismatched+1)); continue; }

    t() { jq -r --arg k "$1" '.format.tags | to_entries[] | select(.key | ascii_downcase == $k) | .value' <<< "$json" | head -1; }
    got_track=$(t track)
    [ -z "$got_track" ] && got_track=$(t tracknumber)
    got_title=$(t title)
    got_artist=$(t artist)
    got_album_artist=$(t album_artist)
    got_album=$(t album)
    got_year=$(t date)
    [ -z "$got_year" ] && got_year=$(t year)

    got_track=${got_track%%/*}
    if [[ "$got_track" =~ ^[0-9]+$ ]]; then
        got_track=$(( 10#${got_track} ))
    else
        got_track=""
    fi

    issues=()
    [ -z "$got_artist" ] && issues+=("missing artist")
    [ -z "$got_album_artist" ] && issues+=("missing album_artist")
    [ -z "$got_album" ] && issues+=("missing album")
    [ -z "$got_year" ] && issues+=("missing year")
    [ -z "$got_title" ] && issues+=("missing title")
    [ -z "$got_track" ] && issues+=("missing tracknumber")

    [ -n "$got_artist" ] && [[ "$(norm <<< "$got_artist")" != "$(norm <<< "$artist_dir")" ]] && issues+=("artist tag differs")
    [ -n "$got_album" ] && [[ "$(norm <<< "$got_album")" != "$(norm <<< "$album_name")" ]] && issues+=("album tag differs")
    [ -n "$got_year" ] && [ -n "$album_year" ] && [[ "$(norm <<< "$got_year")" != "$(norm <<< "$album_year")" ]] && issues+=("year tag differs")
    [ -n "$got_title" ] && [[ "$(norm <<< "$got_title")" != "$(norm <<< "$want_title")" ]] && issues+=("title tag differs")
    [ -n "$got_track" ] && [ "$got_track" -ne "$want_track" ] && issues+=("tracknumber differs")

    if [ "${#issues[@]}" -gt 0 ]; then
        echo "MISMATCH|$rel|$(IFS='; '; echo "${issues[*]}")" >> "$MISMATCH_LOG"
        mismatched=$((mismatched+1))
    fi
done < <(find "$TARGET" -type f \( -iname "*.flac" -o -iname "*.mp3" -o -iname "*.m4a" \) -print0 | sort -z)

if [ -s "$MISMATCH_LOG" ]; then
    cat "$MISMATCH_LOG" | tee -a "$RUN_LOG"
else
    echo "No tag/filename mismatches found." | tee -a "$RUN_LOG"
fi

echo
echo "----------------------------------------"
echo "SUMMARY: $checked file(s) checked, $mismatched had tag/filename mismatches."
echo "Log: $MISMATCH_LOG"
echo "----------------------------------------"
echo "Fix findings interactively with the 'Write Tags from Folder/File"
echo "Names' Nemo action, then re-run Step 14 to regenerate checksums."

```
--- Bash Script for 13e End ---

Run it with your music root as the argument, e.g.:

--- Bash Script Start ---
```bash

bash /path/to/where/you/saved/this /media/youruser/music

```
--- Bash Script End ---

-- Expected Results

A successful run produces:

* step13e-run.log — Full scan record, including the final `SUMMARY:` tally.
* step13e-mismatches.log — One line per file needing attention, in `UNPARSEABLE` or `MISMATCH|path|reason` form. Empty when the library is clean.

A `SUMMARY:` line of `N file(s) checked, 0 had tag/filename mismatches.` means the failsafe passed.

-- Fixing Findings

The moOde guide is a batch pipeline and does not rewrite tags itself. To fix reported files:

* Use the **"Write Tags from Folder/File Names"** Nemo action (from the linux-audio-nemo-actions guide) to losslessly write correct Artist/Album/Year/Title/TrackNumber tags from the naming convention. It uses metaflac, eyeD3, and AtomicParsley — never re-encoding, so audio quality is untouched.
* Or edit individual files with EasyTAG (see the linux-audio-tags-mount-pi5-drive guide).
* After fixing the tags, the checksums for the affected album(s)/artist(s) are stale. Re-run Step 14 (Generate Checksums) — or the "Regenerate ALBUM/ARTIST Checksum" Nemo action — to record the corrected state.
* Optionally re-run 13e afterward to confirm the mismatches are resolved.

\-------------------------------------------------------------------

14. Generate Checksums

-- Purpose

Checksums finalize the library for long-term archival by generating cryptographic hashes for the audio files. They serve as a digital fingerprint, allowing you to detect "bit rot" (silent data corruption on storage media) or verify that data remains perfectly intact after a massive transfer, such as an rsync backup to an external drive.

-- Separate Repository

SHA-512 checksum generation is maintained in a dedicated, standalone repository rather than in this guide. It was once included here, but it grew to be too much for a single project and serves users better on its own — especially those who only want to secure what they have without running the rest of the cleanup workflow.

All scripts, usage instructions, and the resulting checksums.sha512 format live in the repository:

Link to SHA-512 Repository: https://github.com/TerrapinATL/linux.audio.sha512-checksums

-- When to Generate Checksums

Generate checksums on the working copy only after all cleanup steps (Steps 1-8, plus any optional Steps 13a-13d you intend to run) are complete. SHA-512 hashes capture the file contents at the moment they are generated, so any later modification of the audio files will break previously generated checksum files.

After generating the checksum files, copy them along with the audio files to the Master Library and to your rsync backup drive, so the Master Library and every backup carries its own trusted fingerprint. Verify periodically with `sha512sum -c checksums.sha512` in each directory to catch bit rot early.

---

15. Troubleshooting & Reference Information

---

This section provides supplementary materials to support the primary workflow. It includes solutions for common errors encountered during library processing and a detailed breakdown of the log files generated by the automated scripts, ensuring you can effectively troubleshoot issues and verify your results.

-- Common Issues and Fixes:

\-------------------------------------------------------------------

   1. FLAC Decoding Errors (e.g., Unknown Total Samples)

When validating audio files with flac -t, you might encounter failures like ERROR: FLAC input has STREAMINFO with unknown total samples. This is often caused by the software that originally extracted or encoded the files generating an incorrect or missing metadata header. Because FLAC is a lossless format, you can safely repair the file structure without degrading the audio quality by re-encoding the file with ffmpeg.

Run this command on the broken file:

--- Bash Script Start ---
```bash

ffmpeg -i "broken.flac" -c:a flac "fixed.flac"

```
--- Bash Script End ---

Once you verify that fixed.flac plays correctly and passes a flac -t check, you can replace the broken original.

\-------------------------------------------------------------------

   2. Permission Denied Errors

If tools like metaflac, loudgain, or sha512sum fail to write data to a directory, it usually indicates a file ownership or permission issue. This is common when copying files from different filesystems, running tools in a virtual machine, or moving data from external drives.

To grant your current user full read and write access to the library, run:

--- Bash Script Start ---
```bash

sudo chown -R $USER:$USER /path/to/your/music/library
chmod -R u+rw /path/to/your/music/library

```
--- Bash Script End ---

\-------------------------------------------------------------------

   3. Checksum Verification Failures

If you ever run sha512sum -c checksums.sha512 in an album directory and it reports a mismatch, it means the audio file has been modified or corrupted (bit rot) since the original checksum was generated. Do not re-run the checksum generation script to "fix" the error — that will just validate the corrupted state. Instead, delete the corrupted file from your master library and restore a pristine copy from your rsync backup drive.

\-------------------------------------------------------------------

-- Log File Reference & Understanding Your Log Files

Throughout this workflow, all scripts direct their tracking data to a dedicated log directory created in your home directory ($HOME/.logs/linux-audio-moode-cleanup-guide), regardless of which library folder you're working in. This ensures your music directories remain completely free of random text files and gives you a single, centralized place to review the results of your mass operations across every run.

Every log file lives directly in that one directory and is named for the step that produced it, using the pattern stepNN-logname.log (e.g. step01-run.log, step02a-fails.log, step13d-errors.log). The naming convention is consistent across all processing steps:

  1. stepNN-run.log
    The complete, raw output of the script. It lists every directory sequentially as it is processed, showing the OK or FAIL status for each one.

  2. stepNN-oks.log
    A filtered list containing only the albums that were processed successfully.

  3. stepNN-fails.log
    A filtered list of albums that encountered an issue. This is your primary "to-do" list for manual troubleshooting. If this file is empty, the step was a 100% success.

  4. stepNN-errors.log
    Contains the detailed standard error (stderr) output from the specific command-line utilities (like ffmpeg, loudgain, or metaflac). When an album shows up in the fails.log, you can check this error file to see exactly why it failed (e.g., "file not found," "malformed metadata," "permission denied").

\-------------------------------------------------------------------

-- General Cleanup

Once you have reviewed the final logs, verified that your master library is fully processed, and completed your rsync transfer to the external backup drive, you can safely delete the entire $HOME/.logs/linux-audio-moode-cleanup-guide directory. It is completely independent of the audio files and is no longer needed once the project is finished.

\-------------------------------------------------------------------

-- Disclaimer

This guide was developed through iterative collaborative effort between ChatGPT, Claude, Gemini, Mistral and the user. I cannot thank OpenCode project enough. I was about to give up on the other four (well, actually I did) when I came across OpenCode. I run a 10+ year old laptop yet OpenCode ran perfectly well, offloading the heaving lifting to an offsite server. 

https://opencode.ai/







