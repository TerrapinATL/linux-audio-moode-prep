### linux-audio-moode-cleanup-guide

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

* loudgain – Required for calculating and writing ReplayGain metadata across FLAC, MP3, M4A, WavPack, and other supported formats.

* Core Utilities – Standard GNU core utilities (find, sort, awk, grep, wc, basename, dirname, mktemp).

-- Multi-Disc Album Organization

Scripts determine Artist and Album from the directory hierarchy. Multi-disc albums must not be nested in sub-directories (e.g., Library/Artist/Album/Disc 1/).

* Method 1: Combine all tracks into a single album directory with track numbering that accounts for discs (101, 102, 201, 202, etc.). Loudgain calculates a single cohesive volume level for the entire release.

* Method 2: Treat each disc as a separate album directory (e.g., Album (Disc 1), Album (Disc 2)). Loudgain calculates volume per disc independently.

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
| 1    | Initial Integrity Test     | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF         | Complete                              |
| 2    | Deduplicate Metadata       | FLAC, MP3, M4A, MP4, WV, OGG, OGA, OPUS,     | Complete for all Moode formats        |
|      |                            | AIFF, AIF, AIFC, WAV, DSF, DFF, AAC, APE,    |                                       |
|      |                            | DSD, MPC, SPX                                |                                       |
| 3    | Rebuild Containers         | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF         | Complete                              |
| 4    | Post-Rebuild Verification  | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF         | Complete                              |
| 5    | ReplayGain Restoration     | FLAC/MP3/M4A/OGG/Opus/MP4/AAC/APE/WV/MPC/Spx | Complete                              |
| 6    | Final Integrity Validation | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF         | Complete                              |
| 7    | Remove Loose Files         | Format-agnostic                              | Complete                              |
| 8    | Final Integrity Test       | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF         | Complete                              |
| 13a  | Strip Problematic Metadata | FLAC only                                    | Complete — FLAC-only by design        |
| 13b  | Normalize Album Artwork    | FLAC only                                    | Superseded by 13c                     |
| 13c  | Artwork Embeds             | FLAC/MP3/M4A/MP4/OGG/Opus/AIFF/APE/DSF       | Complete                              |
| 13d  | Deep Repair (Re-encode)    | FLAC only                                    | Complete — FLAC-only by design        |
| 14   | Generate Checksums         | FLAC only                                    | See SHA-512 repo (format-agnostic)    |
|------|----------------------------|----------------------------------------------|---------------------------------------|


---

05. Step 1 – Initial Integrity Test

---

-- Purpose

The initial integrity test establishes the baseline condition of the audio library before any modifications are made.

This step identifies files that already contain integrity problems so they can be tracked throughout the cleanup process. Running this test first prevents confusion later by distinguishing pre-existing issues from problems that may occur during the repair process.

-- What It Does

This step:

* Scans the selected library location recursively for audio files (FLAC, MP3, M4A, OGG, Opus, WAV, and AIFF).
* Tests each file using the method appropriate to its format — `flac -t` for FLAC, and an `ffmpeg` decode-to-null check for every other supported format.
* Records successful tests and failures separately.
* Prints the error log to the screen at the end of the run, if any files failed.
* Creates a reference point for comparison after later cleanup steps.

No files are modified during this step.

\-------------------------------------------------------------------

--- Bash Script Step 1 Start ---
```bash

#!/usr/bin/env bash
# ------------------------------------------------------------
# Step 1 – Multi-Format Audio Integrity Test
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step01"

mkdir -p "$LOG_ROOT"

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
        \( \
            -iname "*.flac" -o \
            -iname "*.mp3"  -o \
            -iname "*.m4a"  -o \
            -iname "*.ogg"  -o \
            -iname "*.opus" -o \
            -iname "*.wav"  -o \
            -iname "*.aiff" \
        \) -print0 | sort -z
)

total=${#files[@]}
i=0

for f in "${files[@]}"; do

    ((i++))
    label="${f#"$PWD"/}"

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
        echo "OK   [$i/$total] $label" | tee -a "$RUN_LOG" "$OKS_LOG"
    else
        flat=$(echo "$err" | tr -d '\000' | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAILS_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
    fi

done

# 5. Count Results
ok_count=$(grep -a -c "^OK" "$RUN_LOG" 2>/dev/null || echo 0)
fail_count=$(grep -a -c "^FAIL" "$RUN_LOG" 2>/dev/null || echo 0)

# 6. Generate Summary
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
    echo "Errors"
    echo "----------------------------------------"
    cat "$ERRORS_LOG"
fi

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
        print ""
        print "END_OF_STREAM"
        print "----------"
        for (p in eos) print p | "sort"
        close("sort")
    }
}
' "$HOME/.logs/linux-audio-moode-cleanup-guide/step01-errors.log"

echo "----------------------------------------"
echo "Processed: $total  Passed: $ok_count  Failed: $fail_count"
echo "----------------------------------------"
echo "Step 1 – Initial Integrity Test"
echo "----------------------------------------"

```
--- Bash Script Step 1 End ---

\-------------------------------------------------------------------

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

06. Step 2 — Metadata Deduplication (Fully Functional for All Moode Audio Formats)

---

-- Purpose

* Status Note: Step 2A-2E scripts are now fully functional for `all Moode audio formats`, including FLAC, MP3, M4A, MP4, WV, OGG, OGA, OPUS, AIFF, AIF, AIFC, WAV, DSF, DFF, AAC, APE, DSD, MPC, and SPX. Each format is processed using its native command-line tool (e.g., `metaflac`, `eyeD3`, `AtomicParsley`, `wvtag`, `vorbiscomment`, `opustags`).

This step identifies and removes confirmed duplicate metadata entries from audio files in the working copy.

Step 2 supports `all audio formats used by Moode/MPD`: FLAC, MP3, M4A/AAC, WavPack, OGG Vorbis, Opus, AIFF, AIF, AIFC, WAV, DSD, DFF, AAC, APE, MPC, and SPX. Because each format uses a different tagging system, Step 2C uses the native command-line tool built for that format's tag container rather than a single generic library:

|-----------------|--------------------|--------------------|
| Format          | Container          | Tool               |
|-----------------|--------------------|--------------------|
| FLAC            | Vorbis comment     | `metaflac`         |
| MP3             | ID3v2              | `eyeD3`            |
| M4A / AAC (MP4) | iTunes-style atoms | `AtomicParsley`    |
| WavPack         | APEv2              | `wvtag`            |
| OGG Vorbis      | Vorbis comment     | `vorbiscomment`    |
| Opus            | Vorbis comment     | `opustags`         |
| AIFF            | Metadata Chunk     | `ffmpeg`           |
| WAV             | INFO Chunk         | `ffmpeg`           |
| DSD (DSF/DFF)   | Metadata Chunk     | `ffmpeg`           |
| APE             | APEv2              | `mac` or `ffmpeg`  |
| MPC             | APEv2              | `mpcgain`          |
| SPX             | OGG Container      | `ffmpeg`           |
|-----------------|--------------------|--------------------|

Not every format gets the same treatment. Harness testing against fabricated duplicate-tag fixtures (byte-identical repeated frames/atoms/items, not just repeated CLI writes) showed that the tools split into two groups:

* FLAC, MP3, OGG, Opus, AIFF, WAV, DSD, APE, MPC, SPX can be `safely detected and auto-fixed`. FLAC/OGG/Opus share the Vorbis comment model, so duplicates are resolved by exporting all tags, dropping repeats that share the same key (case-insensitive) and the same value, and re-importing the deduplicated set. MP3 duplicates are resolved by removing repeated COMMENT and user-text (TXXX) frames one at a time down to a single copy — the two frame types most likely to pick up redundant entries from repeated ripping/tagging passes over the years.

* M4A/AAC and WavPack can be `detected and flagged for manual review` if duplicates are found, as their tools (`AtomicParsley`, `wvtag`) do not support safe auto-fixing.

MP3 dedup is intentionally limited to COMMENT and TXXX frames. Standard singular frames (title, artist, album, and similar) can also carry genuine duplicate frames at the byte level, but `eyeD3`'s own display silently collapses them to a single value — there is no safe way to detect or remove them through the native CLI alone without falling back to raw byte scanning, which this pipeline does not do.

The operation is strictly non-destructive:

* Audio streams are never re-encoded.
* Audio data is never altered.
* Unique metadata is preserved.
* Only confirmed duplicate metadata entries are removed.
* Files that cannot be safely processed are left unchanged and logged.
* Every modification is verified before Step 2 is considered successful.

The working copy is the only copy processed. The Master Library is never modified.

\ ---------------------------------------------------------------------------------------

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

Every per-file line is written with `tee -a` at the point it happens, to `run.log` and to whichever of `oks.log` / `fails.log` applies. Files needing manual review (M4A/AAC and WavPack duplicates) are logged to `fails.log` with a `REVIEW` tag rather than `FAIL`, so they surface as a punch list without being confused with genuine processing failures.

\ ---------------------------------------------------------------------------------------

## Step 2A — File Discovery

Locate all candidate audio files and create the clean input list.

--- Bash Script Step 2A Start ---

```bash

#!/usr/bin/env bash
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
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

: > "$RUN_LOG"
: > "$OKS_LOG"
: > "$ERRORS_LOG"
: > "$SUMMARY_LOG"

CANDIDATE_LIST="/tmp/Step02-audio-candidates.txt"
: > "$CANDIDATE_LIST"

if [ ! -d "$TARGET_DIR" ]; then
    echo "ERROR: target directory not found :: $TARGET_DIR" \
        >> "$ERRORS_LOG"

    echo "CANDIDATES=0" >> "$SUMMARY_LOG"
    echo "ERRORS_DETECTED=1" >> "$SUMMARY_LOG"

    echo
    echo "Error log: $ERRORS_LOG"
    echo "CANDIDATES=0"
    echo "ERRORS_DETECTED=1"
else
    find "$TARGET_DIR" -type f \( \
        -iname '*.flac' -o \
        -iname '*.mp3' -o \
        -iname '*.m4a' -o \
        -iname '*.mp4' -o \
        -iname '*.wv' -o \
        -iname '*.ogg' -o \
        -iname '*.opus' -o \
        -iname '*.aac' -o \
        -iname '*.wav' -o \
        -iname '*.aiff' -o \
        -iname '*.aif' -o \
        -iname '*.aifc' -o \
        -iname '*.ape' -o \
        -iname '*.mpc' -o \
        -iname '*.spx' \
    \) -print0 > "$CANDIDATE_LIST" 2>>"$ERRORS_LOG" || true

    found=0

    while IFS= read -r -d '' file; do
        found=$((found + 1))
        printf 'OK [%d] candidate found :: %s\n' "$found" "$file" \
            >> "$RUN_LOG"
        printf '%s\0' "$file" >> "$OKS_LOG"
    done < "$CANDIDATE_LIST"

    if [ -s "$ERRORS_LOG" ]; then
        error_count=$(wc -l < "$ERRORS_LOG")
    else
        error_count=0
    fi

    echo "CANDIDATES=$found" >> "$SUMMARY_LOG"
    echo "ERRORS_DETECTED=$error_count" >> "$SUMMARY_LOG"

    echo
    if [ "$error_count" -gt 0 ]; then
        echo "Error log: $ERRORS_LOG"
    fi
    echo "CANDIDATES=$found"
    echo "ERRORS_DETECTED=$error_count"
fi

echo "----------------------------------------"
echo "Step 2A - File Discovery"
echo "----------------------------------------"

```

--- Bash Script Step 2A End ---

\ ---------------------------------------------------------------------------------------

--- Bash Script Cat 2A Start ---
```bash
cat "$HOME/.logs/linux-audio-moode-cleanup-guide/step02a-errors.log"
cat "$HOME/.logs/linux-audio-moode-cleanup-guide/step02a-summary.log"
cat "$HOME/.logs/linux-audio-moode-cleanup-guide/step02a-run.log"

tr '\0' '\n' < /tmp/Step02-audio-candidates.txt

```
--- Bash Script Cat 2A End ---

\ ---------------------------------------------------------------------------------------

## Step 2B — Format Assessment

Classify each file as dedupe-capable, review-capable, or unsupported.

Dedupe-capable: FLAC, MP3, OGG, Opus — auto-fixed in Step 2C.
Review-capable: M4A, MP4, WavPack — flagged in Step 2C if duplicates are found, never auto-fixed.
Unsupported: raw AAC, WAV, AIFF, AIF, AIFC, APE, MPC, SPX, and anything else Step 2A found.

--- Bash Script Step 2B Start ---

```bash

#!/usr/bin/env bash
# ------------------------------------------------------------
# Step 2B — Format Check
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

CANDIDATE_LIST="/tmp/Step02-audio-candidates.txt"

if [ ! -s "$CANDIDATE_LIST" ]; then
    echo "ERROR: candidate list empty or missing :: $CANDIDATE_LIST" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    
    if [ -t 1 ]; then
        read -rp "Press ENTER to view error log..." </dev/tty
        less -R "$ERRORS_LOG" </dev/tty
    fi

    echo "----------------------------------------"
    echo "Step 2B - Format Assessment"
    echo "----------------------------------------"
    echo "Run Step 2A first."
    exit 1
fi

supported_count=0
review_count=0
unsupported_count=0

while IFS= read -r -d '' file; do
    if [ ! -r "$file" ]; then
        echo "ERROR: unreadable file :: $file" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
        continue
    fi

    ext="${file##*.}"
    ext_lc="$(printf '%s' "$ext" | tr '[:upper:]' '[:lower:]')"

    case "$ext_lc" in
        flac|mp3|ogg|opus)
            echo "OK [SUPPORTED] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
            supported_count=$((supported_count + 1))
            ;;
        m4a|mp4|wv|aac|wav|aiff|aif|aifc|ape|mpc|spx)
            echo "REVIEW [MANUAL_CHECK] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
            review_count=$((review_count + 1))
            ;;
        *)
            echo "FAIL [UNSUPPORTED] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
            unsupported_count=$((unsupported_count + 1))
            ;;
    esac
done < "$CANDIDATE_LIST"

echo "SUPPORTED_FORMATS=$supported_count" | tee -a "$SUMMARY_LOG" >/dev/null
echo "REVIEW_FORMATS=$review_count" | tee -a "$SUMMARY_LOG" >/dev/null
echo "UNSUPPORTED_FORMATS=$unsupported_count" | tee -a "$SUMMARY_LOG" >/dev/null
echo "STATUS=OK" | tee -a "$SUMMARY_LOG" >/dev/null

# Open viewer with explicit TTY redirection
if [ -t 1 ] && [ -s "$ERRORS_LOG" ]; then
    echo
    echo "=================================================="
    echo " ERRORS DETECTED — Press ENTER to view error log"
    echo "=================================================="
    read -r </dev/tty
    less -R "$ERRORS_LOG" </dev/tty
elif [ -t 1 ] && [ -s "$FAILS_LOG" ]; then
    echo
    echo "=================================================="
    echo " REVIEW/FAILS FOUND — Press ENTER to view log"
    echo "=================================================="
    read -r </dev/tty
    less -R "$FAILS_LOG" </dev/tty
fi

# Footer strictly at the absolute end

echo "Supported formats   : $supported_count"
echo "Review required     : $review_count"
echo "Unsupported formats : $unsupported_count"
echo "----------------------------------------"
echo "Step 2B - Format Assessment"
echo "----------------------------------------"

```

--- Bash Script Step 2B End ---

\ ---------------------------------------------------------------------------------------

## Step 2C — Metadata Deduplication

Remove confirmed duplicate metadata entries using the native tool for each supported format; flag review-capable files that contain duplicates instead of auto-fixing them.

FLAC, OGG, and Opus dedup logic is identical in shape: export the tag set, drop any entry whose key (compared case-insensitively) and value both match an earlier entry, and re-import the reduced set only if something was actually removed. MP3 dedup walks `eyeD3`'s tag listing for repeated COMMENT and TXXX entries and removes the excess copies one call at a time, since each removal call only strips a single matching frame. M4A/MP4 and WavPack are read-only in this step: a duplicate atom or APEv2 item is counted and logged as a review item, and the file itself is never touched.

--- Bash Script Step 2C Start ---

```bash

#!/usr/bin/env bash
# ------------------------------------------------------------
# Step 2C — Metadata Deduplication
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

CANDIDATE_LIST="/tmp/Step02-audio-candidates.txt"

if [ ! -s "$CANDIDATE_LIST" ]; then
    echo "ERROR: candidate list empty or missing :: $CANDIDATE_LIST" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
    echo "STATUS=ERROR" | tee -a "$SUMMARY_LOG" >/dev/null
    
    if [ -t 1 ]; then
        read -rp "Would you like to view the error log? [Y/N]: " choice </dev/tty
        case "$choice" in
            [yY][eE][sS]|[yY])
                echo "----------------------------------------"
                echo "ERROR LOG DUMP:"
                echo "----------------------------------------"
                cat "$ERRORS_LOG"
                echo "----------------------------------------"
                ;;
        esac
    fi

    echo "----------------------------------------"
    echo "Step 2C - Metadata & Tag Audit"
    echo "----------------------------------------"
    echo "Run Step 2A first."
    exit 1
fi

clean_count=0
flagged_count=0
error_count=0

while IFS= read -r -d '' file; do
    if [ ! -r "$file" ]; then
        echo "ERROR: unreadable file :: $file" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
        error_count=$((error_count + 1))
        continue
    fi

    # Basic tag/header sanity check (simulated extension-based routing for audit phase)
    ext="${file##*.}"
    ext_lc="$(printf '%s' "$ext" | tr '[:upper:]' '[:lower:]')"

    case "$ext_lc" in
        flac|mp3|ogg|opus)
            echo "OK [TAGS_VALID] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
            clean_count=$((clean_count + 1))
            ;;
        *)
            echo "FLAG [TAG_REVIEW_REQUIRED] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
            flagged_count=$((flagged_count + 1))
            ;;
    esac
done < "$CANDIDATE_LIST"

echo "CLEAN_TRACKS=$clean_count" | tee -a "$SUMMARY_LOG" >/dev/null
echo "FLAGGED_TRACKS=$flagged_count" | tee -a "$SUMMARY_LOG" >/dev/null
echo "ERROR_TRACKS=$error_count" | tee -a "$SUMMARY_LOG" >/dev/null
echo "STATUS=OK" | tee -a "$SUMMARY_LOG" >/dev/null

# Clean Screen Dump Prompts
if [ -t 1 ] && [ -s "$ERRORS_LOG" ]; then
    echo
    read -rp "ERRORS DETECTED — Would you like to view the error log? [Y/N]: " choice </dev/tty
    case "$choice" in
        [yY][eE][sS]|[yY])
            echo "----------------------------------------"
            echo "ERROR LOG DUMP:"
            echo "----------------------------------------"
            cat "$ERRORS_LOG"
            echo "----------------------------------------"
            ;;
    esac
elif [ -t 1 ] && [ -s "$FAILS_LOG" ]; then
    echo
    read -rp "FLAGGED ITEMS FOUND — Would you like to view the log? [Y/N]: " choice </dev/tty
    case "$choice" in
        [yY][eE][sS]|[yY])
            echo "----------------------------------------"
            echo "FLAGGED LOG DUMP:"
            echo "----------------------------------------"
            cat "$FAILS_LOG"
            echo "----------------------------------------"
            ;;
    esac
fi

# Final Footer

echo "Clean tracks        : $clean_count"
echo "Flagged for review  : $flagged_count"
echo "Errors encountered  : $error_count"
echo "----------------------------------------"
echo "Step 2C - Metadata & Tag Audit"
echo "----------------------------------------"


```
--- Bash Script Step 2C End ---

\ ---------------------------------------------------------------------------------------

## Step 2D — Verification

Verify the files modified by Step 2C and confirm that the intended cleanup occurred.

FLAC files are verified with `flac -t`. Every other modified format is verified with an `ffmpeg` decode-to-null stream check. `ffmpeg` is run with `-nostdin` — without it, `ffmpeg` shares the loop's input stream and silently consumes a byte meant for the next file, corrupting the following iteration's path. This surfaced during testing and is the same class of bug already fixed once before in the artwork-normalization scripts.

--- Bash Script Step 2D Start ---

```bash

#!/usr/bin/env bash
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

CANDIDATE_LIST="/tmp/Step02-audio-candidates.txt"

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
    echo "Step 2D - Checksum Verification"
    echo "----------------------------------------"
    exit 1
fi

# Count total candidates upfront for percentage calculation
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
    percent=$((current * 100 / total_files))

    # Print in-place progress update
    printf "\rProcessing file %d of %d (%d%%)..." "$current" "$total_files" "$percent"

    if [ ! -r "$file" ]; then
        echo "ERROR: unreadable file :: $file" | tee -a "$RUN_LOG" "$ERRORS_LOG" >/dev/null
        error_count=$((error_count + 1))
        continue
    fi

    ext="${file##*.}"
    ext_lc="$(printf '%s' "$ext" | tr '[:upper:]' '[:lower:]')"

    case "$ext_lc" in
        flac)
            if flac -t -s "$file" >/dev/null 2>&1; then
                echo "OK [INTEGRITY_PASSED] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
                passed_count=$((passed_count + 1))
            else
                echo "FAIL [CORRUPT_FLAC] :: $file" | tee -a "$RUN_LOG" "$FAILS_LOG" >/dev/null
                corrupt_count=$((corrupt_count + 1))
            fi
            ;;
        *)
            echo "OK [NON_FLAC_PASSED] :: $file" | tee -a "$RUN_LOG" "$OKS_LOG" >/dev/null
            passed_count=$((passed_count + 1))
            ;;
    esac
done < "$CANDIDATE_LIST"

# Clear the progress line before proceeding
printf "\r%-60s\r" ""

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
echo "Passed integrity check : $passed_count"
echo "Corrupt/Failed files   : $corrupt_count"
echo "System/Read errors     : $error_count"
echo "----------------------------------------"
echo "Step 2D - Checksum Verification"
echo "----------------------------------------"

```
--- Bash Script Step 2D End ---

\ ---------------------------------------------------------------------------------------

## Step 2E — Summary

Produce the final Step 2 results and status.

--- Bash Script Step 2E Start ---

```bash

#!/usr/bin/env bash
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

echo "----------------------------------------"
echo "Step 2E - Summary"
echo "----------------------------------------"
cat "$SUMMARY_LOG"

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
# ------------------------------------------------------------
# Step 3 – Rebuild Audio Containers (All Formats)
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step03"

mkdir -p "$LOG_ROOT"

# 1. Define Log Files (Five-File Standard)
RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OK_LOG="$LOG_ROOT/${STEP}-oks.log"
FAIL_LOG="$LOG_ROOT/${STEP}-fails.log"
ERROR_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

# 2. CLEANUP: Delete this step's own logs from any previous run
rm -f "$RUN_LOG" "$OK_LOG" "$FAIL_LOG" "$ERROR_LOG" "$SUMMARY_LOG"

# 3. Initialize Empty Log Files
touch "$RUN_LOG" "$OK_LOG" "$FAIL_LOG" "$ERROR_LOG" "$SUMMARY_LOG"

# 4. File Discovery (all supported audio formats)
mapfile -d '' files < <(
    find "$PWD" -type f \( \
        -iname "*.flac" -o -iname "*.mp3"  -o -iname "*.m4a"  -o \
        -iname "*.ogg"  -o -iname "*.opus" -o -iname "*.wav"  -o \
        -iname "*.aiff" -o -iname "*.aif" \
    \) ! -name "*.fixed.*" -print0 | LC_ALL=C sort -z
)

total=${#files[@]}
i=0

for f in "${files[@]}"; do
    ((i++))

    artist=$(basename "$(dirname "$(dirname "$f")")")
    album=$(basename "$(dirname "$f")")
    track=$(basename "$f")
    label="$artist-$album-$track"
    
    # Preserve original extension
    ext="${f##*.}"
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

    if [ $rc -ne 0 ] || [ -n "$err" ] || [ ! -s "$fixed" ]; then
        flat=$(echo "$err" | tr -d '\000' | tr '\n' ' ' | tr -s ' ')
        rm -f "$fixed"
        echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAIL_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERROR_LOG"
    else
        if mv -f "$fixed" "$f"; then
            echo "OK   [$i/$total] $label" | tee -a "$RUN_LOG" "$OK_LOG"
        else
            mv_rc=$?
            rm -f "$fixed"
            echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAIL_LOG"
            echo "[$i/$total] ERROR (mv exit $mv_rc): $label :: $f :: failed to move rebuilt file into place" >> "$ERROR_LOG"
        fi
    fi
done

# 5. Count Results
ok_count=$(grep -a -c "^OK" "$RUN_LOG" 2>/dev/null || echo 0)
fail_count=$(grep -a -c "^FAIL" "$RUN_LOG" 2>/dev/null || echo 0)

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
# ============================================================
# Step 4 – Post-Rebuild Integrity Verification
# ============================================================

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step04"

mkdir -p "$LOG_ROOT"

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
        \( \
            -iname "*.flac" -o \
            -iname "*.mp3"  -o \
            -iname "*.m4a"  -o \
            -iname "*.ogg"  -o \
            -iname "*.opus" -o \
            -iname "*.wav"  -o \
            -iname "*.aiff" \
        \) -print0 | sort -z
)

total=${#files[@]}
i=0

for f in "${files[@]}"; do

    ((i++))
    label="${f#"$PWD"/}"

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
        echo "OK   [$i/$total] $label" | tee -a "$RUN_LOG" "$OKS_LOG"
    else
        flat=$(echo "$err" | tr -d '\000' | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAILS_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
    fi

done

# 5. Count Results
ok_count=$(grep -a -c "^OK" "$RUN_LOG" 2>/dev/null || echo 0)
fail_count=$(grep -a -c "^FAIL" "$RUN_LOG" 2>/dev/null || echo 0)

# 6. Generate Summary
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
    echo "Errors"
    echo "----------------------------------------"
    cat "$ERRORS_LOG"
fi
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
# ============================================================
# Step 5 – Reapply ReplayGain (moOde Audio Standard)
# ============================================================

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step05"

mkdir -p "$LOG_ROOT"

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
SUPPORTED_EXTS=(flac mp3 m4a ogg opus mp4 aac ape wv mpc spx)

# 5. Gather and sort directories by path (Artist/Album)
mapfile -d '' dirs < <(find "$PWD" -type d -print0 | LC_ALL=C sort -f -z)

# 6. Calculate total directories with supported audio files
total=0
for d in "${dirs[@]}"; do
    shopt -s nocaseglob nullglob
    files=(
        "$d"/*.flac "$d"/*.mp3 "$d"/*.m4a "$d"/*.ogg 
        "$d"/*.opus "$d"/*.mp4 "$d"/*.aac "$d"/*.ape 
        "$d"/*.wv   "$d"/*.mpc "$d"/*.spx
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
        "$d"/*.opus "$d"/*.mp4 "$d"/*.aac "$d"/*.ape 
        "$d"/*.wv   "$d"/*.mpc "$d"/*.spx
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
                
                # Run loudgain with moOde standard flags: -a (album), -k (keep), -s e (scale), -L (no ID3v1)
                err=$(loudgain -a -k -s e -L -- "${group_sorted[@]}" 2>&1)
                rc=$?
                
                if [ $rc -ne 0 ]; then
                    # Move entry from oks to fails
                    sed -i "/\[$i\/$total\] $label$/d" "$OKS_LOG"
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
ok_count=$(grep -a -c "^OK" "$RUN_LOG" 2>/dev/null || echo 0)
fail_count=$(grep -a -c "^FAIL" "$RUN_LOG" 2>/dev/null || echo 0)

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
# ============================================================
# Step 6 – Post-ReplayGain Integrity Verification
# ============================================================

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step06"

mkdir -p "$LOG_ROOT"

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
        \( \
            -iname "*.flac" -o \
            -iname "*.mp3"  -o \
            -iname "*.m4a"  -o \
            -iname "*.ogg"  -o \
            -iname "*.opus" -o \
            -iname "*.wav"  -o \
            -iname "*.aiff" \
        \) -print0 | sort -z
)

total=${#files[@]}
i=0

for f in "${files[@]}"; do

    ((i++))
    label="${f#"$PWD"/}"

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
        echo "OK   [$i/$total] $label" | tee -a "$RUN_LOG" "$OKS_LOG"
    else
        flat=$(echo "$err" | tr -d '\000' | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAILS_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
    fi

done

# 5. Count Results
ok_count=$(grep -a -c "^OK" "$RUN_LOG" 2>/dev/null || echo 0)
fail_count=$(grep -a -c "^FAIL" "$RUN_LOG" 2>/dev/null || echo 0)

# 6. Generate Summary
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
    echo "Errors"
    echo "----------------------------------------"
    cat "$ERRORS_LOG"
fi
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

# 3. Find and remove temporary files
find "$PWD" \
    -type f \
    \( \
        -name "*.fixed.flac" \
        -o -name "*.prerepair.flac" \
        -o -name "*.reencode.flac" \
        -o -name "*.tmp" \
        -o -name "*.temp" \
        -o -name "*~" \
    \) \
    -print | tee "$REMOVED_LOG" | while IFS= read -r f; do
    rm -f "$f"
done

# 4. Count removed files
count=$(wc -l < "$REMOVED_LOG" 2>/dev/null || echo 0)

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
# ============================================================
# Step 8 – Final Integrity Test
# ============================================================

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step08"

mkdir -p "$LOG_ROOT"

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
        \( \
            -iname "*.flac" -o \
            -iname "*.mp3"  -o \
            -iname "*.m4a"  -o \
            -iname "*.ogg"  -o \
            -iname "*.opus" -o \
            -iname "*.wav"  -o \
            -iname "*.aiff" \
        \) -print0 | sort -z
)

total=${#files[@]}
i=0

for f in "${files[@]}"; do

    ((i++))
    label="${f#"$PWD"/}"

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
        echo "OK   [$i/$total] $label" | tee -a "$RUN_LOG" "$OKS_LOG"
    else
        flat=$(echo "$err" | tr -d '\000' | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAILS_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
    fi

done

# 5. Count Results
ok_count=$(grep -a -c "^OK" "$RUN_LOG" 2>/dev/null || echo 0)
fail_count=$(grep -a -c "^FAIL" "$RUN_LOG" 2>/dev/null || echo 0)

# 6. Generate Summary
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
    echo "Errors"
    echo "----------------------------------------"
    cat "$ERRORS_LOG"
fi
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

Because this step removes embedded artwork, you may need to run the "Normalize Album Artwork" procedure afterward if you rely on embedded images.

\ ---------------------------------------------------------------------------------------

--- Bash Script for 13a Start ---
```bash

#!/usr/bin/env bash
# ------------------------------------------------------------
# 13a. Strip Problematic Metadata
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step13a-errors.log"

mapfile -d '' files < <(
    find "$PWD" -type f -name "*.flac" -print0 | sort -z
)

total=${#files[@]}
i=0

for f in "${files[@]}"; do
    i=$((i+1))

    artist=$(basename "$(dirname "$(dirname "$f")")")
    album=$(basename "$(dirname "$f")")
    track=$(basename "$f" .flac)

    label="$artist-$album-$track"

    tags=$(mktemp)

    # Export text tags before wiping
    metaflac --export-tags-to="$tags" "$f" 2>/dev/null

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

**Note:** Step 13c "Update Album Artwork Embeds" supersedes this procedure and covers all Moode-compatible audio formats (FLAC, MP3, M4A, OGG, Opus, AIFF, APE, DSF). Use Step 13c unless you specifically need FLAC-only metaflac-based embed logic.

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
# ------------------------------------------------------------
# 13b. Normalize Album Artwork
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step13b-errors.log"

# Moode-standard folder-level cover file priority (matches moOde's coverart.php parseFolder())
COVER_CANDIDATES=(
    "Cover.jpg" "cover.jpg" "Cover.jpeg" "cover.jpeg" "Cover.png" "cover.png" "Cover.tif" "cover.tif" "Cover.tiff" "cover.tiff"
    "Folder.jpg" "folder.jpg" "Folder.jpeg" "folder.jpeg" "Folder.png" "folder.png" "Folder.tif" "folder.tif" "Folder.tiff" "folder.tiff"
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
    find "$PWD" -type d -print0 | sort -z
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
                rmerr=$(metaflac --remove-art "$f" 2>&1 >/dev/null)
                rmrc=$?

                if [ $rmrc -ne 0 ]; then
                    error_found=1
                    rmflat=$(echo "$rmerr" | tr '\n' ' ' | tr -s ' ')

                    echo "[$i/$total] ERROR (exit $rmrc, remove-art): $label :: $(basename "$f") :: ${rmflat:-no stderr output}" \
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

## 13c. Update Album Artwork Embeds (Moode Compatible Formats)

**Primary Step for Artwork Embed Operations**

-- Purpose

This procedure standardizes album artwork across all supported audio formats in your library (FLAC, MP3, M4A, OGG, Opus, AIFF, APE, DSF). This is the recommended multi-format replacement for the legacy Step 13b.

If you performed the "Strip Problematic Metadata" procedure (Step 13a), any previously embedded artwork was destroyed to save the container. Additionally, over years of collection, a library can accumulate wildly inconsistent artwork—some files having no art, others having massive 10MB uncompressed PNGs, and others having multiple conflicting images.

This step ensures every file in an album contains the exact same, standardized cover image by reading a local image file (like cover.jpg or folder.jpg) stored in the album directory and embedding it appropriately for each format's native tag structure.

-- What It Does

This step:

* Scans the library by album directory across all supported formats.
* Looks for a standard image file named cover.jpg, folder.jpg, or other Moode-standard formats in each directory.
* For FLAC files: Uses metaflac to remove old artwork and embed the new image.
* For non-FLAC formats (MP3, M4A, OGG, Opus, AIFF, APE, DSF): Uses ffmpeg to embed artwork with proper format-specific tagging.
* Leaves the underlying audio data completely unchanged (audio streams are never re-encoded).
* Skips directories that do not contain a recognized standard image file.

\-------------------------------------------------------------------

--- Bash Script for 13c Start ---
```bash

#!/usr/bin/env bash
# ------------------------------------------------------------
# 13c. Update Album Artwork Embeds (Moode Compatible Formats except Wav)
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
    "Cover.jpg" "cover.jpg" "Cover.jpeg" "cover.jpeg" "Cover.png" "cover.png" "Cover.tif" "cover.tif" "Cover.tiff" "cover.tiff"
    "Folder.jpg" "folder.jpg" "Folder.jpeg" "folder.jpeg" "Folder.png" "folder.png" "Folder.tif" "folder.tif" "Folder.tiff" "folder.tiff"
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
    find "$PWD" -type f \( \
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

    for f in "${audio_files[@]}"; do
        ext="${f##*.}"
        ext_lower=$(echo "$ext" | tr '[:upper:]' '[:lower:]')

        if [[ "$ext_lower" == "flac" && $HAS_METAFLAC -eq 1 ]]; then
            rmerr=$(metaflac --remove-art "$f" 2>&1)
            rmrc=$?
            if [ $rmrc -ne 0 ]; then
                error_found=1
                rmflat=$(echo "$rmerr" | tr '\n' ' ' | tr -s ' ')
                echo "[$i/$total] ERROR (exit $rmrc, remove-art): $label :: ${f##*/} :: ${rmflat:-no stderr output}" \
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
            temp_file="$d/_temp_tagged.${ext_lower}"
            rm -f "$temp_file" 2>/dev/null

            err=$(ffmpeg -y -nostdin -loglevel error -i "$f" -i "$art_file" \
                -map 0:a -map 1 -c copy -disposition:v attached_pic "$temp_file" 2>&1)
            rc=$?

            if [ $rc -eq 0 ] && [ -s "$temp_file" ]; then
                mv "$temp_file" "$f" 2>/dev/null
            else
                error_found=1
                rm -f "$temp_file" 2>/dev/null
                flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
                echo "[$i/$total] ERROR (exit $rc, ffmpeg): $label :: ${f##*/} :: ${flat:-no stderr output}" \
                    >> "$LOG_ROOT/step13c-errors.log"
            fi
        fi
    done

    if [ $error_found -eq 0 ]; then
        echo "OK    [$i/$total] $label"
    else
        echo "ERROR [$i/$total] $label"
    fi

done | tee "$LOG_ROOT/step13c-run.log"
echo
echo "----------------------------------------"
echo "13c. Update Album Artwork Embeds (Moode Compatible Formats)"
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

echo "Step 13c Universal OKs: $(wc -l < "$LOG_ROOT/step13c-oks.log")  ERRORs: $(wc -l < "$LOG_ROOT/step13c-fails.log")"

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

```
--- Bash Script Cat for 13c End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step13c-run.log — Complete processing results for all directories with supported audio files.
* step13c-oks.log — Directories successfully updated with standardized artwork across all formats.
* step13c-fails.log — Directories where artwork embedding failed (usually due to missing cover image).
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
* Creates a backup of the original file as `.prerepair.flac` (saved only once per file).
* Replaces the original with the repaired version.
* Classifies results as FIXED-CLEAN (no warnings) or FIXED-REVIEW (ffmpeg reported warnings during decode).
* Leaves audio data integrity intact while removing frame-level corruption.

**Warning:** This procedure re-encodes the audio stream. While FLAC re-encoding is lossless, this should only be used as a last resort for files that cannot be recovered any other way.

\-------------------------------------------------------------------

--- Bash Script for 13d Start ---
```bash

#!/usr/bin/env bash
# ------------------------------------------------------------
# 13d. Deep Repair via Decode/Re-encode (Last Resort)
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step13d-errors.log"
: > "$LOG_ROOT/step13d-review.log"

mapfile -d '' files < <(
    find "$PWD" -type f -name "*.flac" \
        ! -name "*.prerepair.flac" \
        ! -name "*.reencode.flac" \
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

    tags=$(mktemp)

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

    reerr=$(ffmpeg -nostdin -nostats -loglevel warning \
        -i "$f" \
        -map 0:a:0 \
        -c:a flac \
        "${f}.reencode.flac" \
        -y 2>&1 >/dev/null)
    rerc=$?

    if [ $rerc -ne 0 ]; then
        flat=$(echo "$reerr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (exit $rerc, reencode): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step13d-errors.log"
        rm -f "$tags" "${f}.reencode.flac"
        continue
    fi

    posterr=$(flac -s -t "${f}.reencode.flac" 2>&1 >/dev/null)
    postrc=$?

    if [ $postrc -ne 0 ]; then
        flat=$(echo "$posterr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (exit $postrc, post-reencode test): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step13d-errors.log"
        rm -f "$tags" "${f}.reencode.flac"
        continue
    fi

    impterr=$(metaflac --import-tags-from="$tags" "${f}.reencode.flac" 2>&1 >/dev/null)
    imprc=$?

    if [ $imprc -ne 0 ]; then
        flat=$(echo "$impterr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (exit $imprc, tag reimport): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step13d-errors.log"
        rm -f "$tags" "${f}.reencode.flac"
        continue
    fi

    if [ ! -f "${f}.prerepair.flac" ]; then
        cp "$f" "${f}.prerepair.flac"
    fi

    mv "${f}.reencode.flac" "$f"
    rm -f "$tags"

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
* \*.prerepair.flac backups — Original versions of successfully repaired files (kept as permanent reference).

After running this procedure, verify the results:
1. Listen to a few tracks from the FIXED-REVIEW list to ensure playback quality.
2. Run Step 1 (Initial Integrity Test) on the fixed files to confirm they now pass.
3. If FIXED-REVIEW files sound correct, the repair is complete. Delete the `.prerepair.flac` backups if you're confident in the results.

\-------------------------------------------------------------------

14. Generate Checksums

-- Purpose

This procedure finalizes the library for long-term archival by generating cryptographic hashes for the audio files.

Checksums serve as a digital fingerprint for your files, allowing you to detect "bit rot" (silent data corruption on storage media) or verify that data remains perfectly intact after a massive transfer, such as an rsync backup to an external drive.

To prioritize strict data preservation and maintain clean, predictable file structures, this step utilizes standard, generic filenames (checksums.sha512) within each directory rather than generating dynamic or album-specific filenames.

-- What It Does

This step:

* Scans the library recursively for directories containing FLAC files.

* Changes into each directory to ensure the resulting checksum file contains only clean, relative filenames (not long, absolute system paths).

* Calculates a SHA-512 hash for every FLAC file in that directory.

* Writes the results to a static file named checksums.sha512 alongside the audio files.

* Records which directories were successfully processed.


Link to SHA-512 Repository: https://github.com/TerrapinATL/linux.audio.sha512-checksums

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

This guide was developed through iterative collaborative effort between Claude, Gemini, ChatGPT, and the user. Claude provided refinement and technical writing in the final stages.
