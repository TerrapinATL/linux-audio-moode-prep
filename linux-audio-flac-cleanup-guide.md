---
01. Introduction

---

This document is a technical guide and automated script workflow for auditing, repairing, and standardizing FLAC-based music libraries on Linux systems, focusing on integrity testing, metadata cleanup, container rebuilding, and volume normalization.

It is intended for users who maintain large music collections and want a consistent process for correcting metadata issues, verifying file integrity, and preparing libraries for long-term archival.

Note: Always work off a backup library copy until a clean copy has been secured by sha512 checksums.

---

02. Requirements

---

To successfully execute the scripts and workflows in this guide, your system must have the following command-line tools installed and available in your shell's PATH:

* flac – Required for testing file integrity and decoding/encoding validation.

* metaflac – Required for inspecting, exporting, and modifying Vorbis comment metadata tags.

* ffmpeg – Required for rebuilding corrupted FLAC containers without altering the underlying audio stream.

* loudgain – Required for calculating and writing album- and track-level ReplayGain metadata across audio formats.

* Core Utilities – Standard GNU core utilities (find, sort, awk, grep, wc, basename, dirname, mktemp) commonly available in Linux environments like Linux Mint.

-- Note on Handling Multi-Disc Albums

To maintain the strict Parent -> Artist -> Album directory structure required by these automated scripts, multi-disc albums must not be nested in sub-directories. 

    Parent/
    ├── Artist1/
    │   ├── artist-level files
    │   ├── Album1/
    │   │   ├── album files
    │   │   └── ...
    │   └── Album2/
    └── Artist2/

The bash scripts determine the Artist and Album metadata labels based on the directory hierarchy (using standard dirname and basename commands). If you nest discs (e.g., Library/Artist/Album/Disc 1/), the scripts will incorrectly parse the path, misidentifying "Album" as the Artist, and "Disc 1" as the Album.

To prevent structural errors and ensure tools like loudgain calculate album-level ReplayGain correctly, multi-disc albums should be organized using one of two methods:

Method 1: Single Unified Directory (Recommended)

Combine all tracks from all discs into a single album directory. Use a metadata track numbering scheme that accounts for multiple discs (e.g., Disc 1 tracks as 101, 102 and Disc 2 tracks as 201, 202, or continuous numbering from 01 onward). This preserves the flat hierarchy and guarantees loudgain calculates a single, cohesive volume normalization for the entire release.

Method 2: Discrete Album Directories

Treat each disc as its own separate album at the directory level. For example:

* Library/Artist/Album (Disc 1)/
* Library/Artist/Album (Disc 2)/

This preserves the script's strict structural pathing, though loudgain will calculate volume normalization for each disc independently rather than as a combined unit.

---

03. Design Philosophy

---

This guide is built on three core engineering principles:

* Non-Destructive Processing: Audio streams are never re-encoded during cleanup, ensuring zero generational loss or degradation of audio quality.

* Explicit Verification: Every automated modification is bracketed by strict validation tests to catch errors immediately rather than propagating them through the library.

* Traceable Execution: All operations generate clean, timestamped logs and separate success/failure lists, ensuring total transparency and auditability across large-scale library management.

---

04. Workflow Overview

---

The cleanup process follows a simple verification-based approach:

Verify → Edit → Verify → Edit → Verify

Each major operation begins with an understanding of the current state of the library and is followed by validation to confirm the expected results.

Verify — Establish the current condition of the files and identify problems before making changes.

Edit — Perform a specific repair or cleanup operation.

Verify — Confirm that the operation completed successfully and did not introduce new issues.

This approach ensures that problems are identified before modification, changes are traceable, and each stage of the cleanup process can be evaluated before continuing.

Sequential Pipeline Summary

1. Initial Integrity Test: Establish a baseline of existing issues using flac -t.
2. Deduplicate Metadata: Remove redundant Vorbis comment entries from FLAC tags.
3. Container Rebuild: Strip invalid metadata headers using ffmpeg without re-encoding audio streams.
4. Post-Rebuild Verification: Repeat the integrity test to validate structural container fixes.
5. ReplayGain Restoration: Recalculate and reapply album-level volume normalization.
6. Final Integrity Validation: Run a final structural test to confirm library stability.
7. Cleanup: Remove temporary files, incomplete rebuilds, and unwanted artifacts.
8. Archival Preparation: Finalize the library for long-term storage or checksum generation.

If files continue to fail integrity testing after the standard workflow, optional repair procedures may be performed on the affected files before repeating the validation process.

Additional optional procedures are available for standardizing album artwork and generating checksum files for long-term archival.

Files that continue to fail integrity testing should be considered for replacement with a clean copy that does.

---

05. Step 1 – Initial Integrity Test

---

-- Purpose

The initial integrity test establishes the baseline condition of the FLAC library before any modifications are made.

This step identifies files that already contain integrity problems so they can be tracked throughout the cleanup process. Running this test first prevents confusion later by distinguishing pre-existing issues from problems that may occur during the repair process.

-- What It Does

This step:

* Scans the selected library location recursively for FLAC files.
* Tests each file using flac -t.
* Records successful tests and failures separately.
* Creates a reference point for comparison after later cleanup steps.

No files are modified during this step.

\-------------------------------------------------------------------

--- Bash Script Step 1 Start ---
```bash

#!/usr/bin/env bash
# ------------------------------------------------------------
# Step 1 – Multi-Format Audio Integrity Test
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/Linux_Audio_Folder_Level"
GUIDE="Library_Cleanup"
STEP="Step01_Integrity_Test"

LOG_DIR="$LOG_ROOT/$GUIDE/$STEP"

# 1. Ensure Guide Root Exists
mkdir -p "$LOG_ROOT/$GUIDE"

# 2. CLEANUP: Delete all existing files in the STEP directory immediately
# This ensures no stale logs remain from previous runs
find "$LOG_DIR" -type f -delete 2>/dev/null || true

# 3. Re-create the specific STEP directory
mkdir -p "$LOG_DIR"

# 4. Define Log Files (Template Standard)
RUN_LOG="$LOG_DIR/step01_run.log"
OK_LOG="$LOG_DIR/step01_oks.log"
FAIL_LOG="$LOG_DIR/step01_fails.log"
ERROR_LOG="$LOG_DIR/step01_errors.log"
SUMMARY_LOG="$LOG_DIR/step01_summary.log"

# 5. Initialize Empty Log Files
touch "$RUN_LOG" "$OK_LOG" "$FAIL_LOG" "$ERROR_LOG"

# 6. File Discovery
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
            err=$(ffmpeg -v error -i "$f" -f null - 2>&1)
            rc=$?
            ;;
    esac

    if [ $rc -eq 0 ]; then
        echo "OK   [$i/$total] $label" | tee -a "$RUN_LOG" "$OK_LOG"
    else
        flat=$(echo "$err" | tr -d '\000' | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAIL_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERROR_LOG"
    fi

done

# 7. Count Results
ok_count=$(grep -a -c "^OK" "$RUN_LOG" 2>/dev/null || echo 0)
fail_count=$(grep -a -c "^FAIL" "$RUN_LOG" 2>/dev/null || echo 0)

# 8. Generate Summary
{
echo "Step 1 Summary"
echo "=============="
echo
echo "Guide      : $GUIDE"
echo "Step       : $STEP"
echo "Run Date   : $(date)"
echo
echo "Processed  : $total"
echo "Passed     : $ok_count"
echo "Failed     : $fail_count"
} > "$SUMMARY_LOG"

# 9. Terminal Output
echo
echo "----------------------------------------"
echo "Processed: $total  Passed: $ok_count  Failed: $fail_count"
echo "----------------------------------------"
echo "Step 1 – Initial Integrity Test"
echo "----------------------------------------"

```
--- Bash Script Step 1 End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat 1 Start ---
```bash

    cat "$LOGDIR/audio_test_errors.log"
    
    cat "$LOGDIR/step1_run.log"
    
    cat "$LOGDIR/step1_oks.log"
    
    cat "$LOGDIR/step1_fails.log"

```
--- Bash Script Cat 1 End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step1_run.log — Complete test results.
* step1_oks.log — Files that passed integrity testing.
* step1_fails.log — Files that failed integrity testing.
* audio_test_errors.log — Detailed error output from failed tests.

Files that fail this initial test should be reviewed, but they should not automatically be considered beyond repair. Later workflow steps may resolve some failures caused by metadata or container issues.

---

06. Step 2 – Deduplicate Vorbis Comments

---

-- Purpose

This step removes duplicate metadata entries from FLAC files caused by repeated tag editing.

FLAC files store metadata using Vorbis comments. Over time, repeated edits from different applications can create duplicate tag entries, which may cause inconsistent behavior between media players and library management software.

This step cleans up duplicate metadata entries while preserving the existing audio data.

-- What It Does

This step:

* Scans the selected library location recursively for FLAC files.
* Exports the existing Vorbis comment tags from each file.
* Removes duplicate tag lines while preserving unique entries.
* Reimports the cleaned metadata back into the FLAC file.
* Records files that were modified and files that could not be processed.

No audio is re-encoded during this process.

\-------------------------------------------------------------------

-- Step 2A: File Discovery & Log Initialization

--- Bash Script Step 2A Start ---
```bash

#!/usr/bin/env bash
# ------------------------------------------------------------
# Step 2A: File Discovery & Log Initialization
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/Linux_Audio_Folder_Level"
GUIDE="Library_Cleanup"
STEP="Step02A_Discovery"

LOG_DIR="$LOG_ROOT/$GUIDE/$STEP"

# 1. Ensure Guide Root Exists
mkdir -p "$LOG_ROOT/$GUIDE"

# 2. CLEANUP: Delete all existing files in the STEP directory immediately
# This ensures no stale logs remain from previous runs
find "$LOG_DIR" -type f -delete 2>/dev/null || true

# 3. Re-create the specific STEP directory
mkdir -p "$LOG_DIR"

# 4. Define Log Files (Template Standard)
RUN_LOG="$LOG_DIR/step02a_run.log"
OK_LOG="$LOG_DIR/step02a_oks.log"
FAIL_LOG="$LOG_DIR/step02a_fails.log"
ERROR_LOG="$LOG_DIR/step02a_errors.log"
SUMMARY_LOG="$LOG_DIR/step02a_summary.log"

# 5. Initialize Empty Log Files
touch "$RUN_LOG" "$OK_LOG" "$FAIL_LOG" "$ERROR_LOG"

# 6. Target List (Temp File for Discovery)
TARGET_LIST="/tmp/step2_targets.txt"

# 7. Discovery: Find FLAC files sorted safely
find "$PWD" -type f -iname "*.flac" -print0 | LC_ALL=C sort -z > "$TARGET_LIST"

# 8. Count Total Files (wc -z counts null terminators)
total=$(wc -z < "$TARGET_LIST" | tr -d ' ')

# 9. Initial Status Logging
if [ "$total" -gt 0 ]; then
    echo "OK   [1/1] Discovery Complete: Found $total .flac file(s)" | tee -a "$RUN_LOG" "$OK_LOG"
else
    echo "FAIL [1/1] Discovery Complete: No .flac files found" | tee -a "$RUN_LOG" "$FAIL_LOG"
fi

# 10. Summary Output
{
echo "Step 2A Summary"
echo "==============="
echo
echo "Guide      : $GUIDE"
echo "Step       : $STEP"
echo "Run Date   : $(date)"
echo
echo "Target Dir : $PWD"
echo "Total Found: $total"
echo "Log Dir    : $LOG_DIR"
} > "$SUMMARY_LOG"

# 11. Terminal Output
echo
echo "----------------------------------------"
echo "Discovery Complete: $total .flac file(s) found"
echo "----------------------------------------"
echo "Step 2A – File Discovery & Log Initialization"
echo "----------------------------------------"

```
--- Bash Script Step 2A End ---

\-------------------------------------------------------------------

-- Step 2B: Tag Processing & Deduplication

--- Bash Script Step 2B Start ---
```bash

#!/usr/bin/env bash
# ------------------------------------------------------------
# Step 2B: Tag Processing & Deduplication
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/Linux_Audio_Folder_Level"
GUIDE="Library_Cleanup"
STEP="Step02B_Dedup"

LOG_DIR="$LOG_ROOT/$GUIDE/$STEP"

# 1. Directory Setup & Cleanup (Template Standard)
mkdir -p "$LOG_ROOT/$GUIDE"
find "$LOG_DIR" -type f -delete 2>/dev/null || true
mkdir -p "$LOG_DIR"

# 2. Define Log Files (Template Standard)
RUN_LOG="$LOG_DIR/step02b_run.log"
OK_LOG="$LOG_DIR/step02b_oks.log"
FAIL_LOG="$LOG_DIR/step02b_fails.log"
ERROR_LOG="$LOG_DIR/step02b_errors.log"
SUMMARY_LOG="$LOG_DIR/step02b_summary.log"
FIX_LOG="$LOG_DIR/step02b_fixes.log"  # Optional: Additional log for fixes

# 3. Initialize Log Files
touch "$RUN_LOG" "$OK_LOG" "$FAIL_LOG" "$ERROR_LOG" "$FIX_LOG"

# 4. Target List
TARGET_LIST="/tmp/step2_targets.txt"

if [ ! -f "$TARGET_LIST" ] || [ ! -s "$TARGET_LIST" ]; then
    echo "FAIL [0/0] No target list found or empty." | tee -a "$RUN_LOG" "$FAIL_LOG"
    echo "Step 2B Summary" > "$SUMMARY_LOG"
    echo "===============" >> "$SUMMARY_LOG"
    echo "Error: No target files found in $TARGET_LIST" >> "$SUMMARY_LOG"
    exit 1
fi

# 5. Load Files into Array (Safe for null-delimited)
mapfile -d '' files < "$TARGET_LIST"
total=${#files[@]}

if [ "$total" -eq 0 ]; then
    echo "FAIL [0/0] No target files to process." | tee -a "$RUN_LOG" "$FAIL_LOG"
    {
    echo "Step 2B Summary"
    echo "==============="
    echo "Guide      : $GUIDE"
    echo "Step       : $STEP"
    echo "Run Date   : $(date)"
    echo "Total Found: 0"
    echo "Result     : No files to process"
    } > "$SUMMARY_LOG"
    exit 0
fi

# 6. Initialize Counters (Outside loop to avoid subshell issues)
i=0
fixed_count=0
ok_count=0
fail_count=0

# 7. Process Loop (No pipe to tee - use tee inside)
for f in "${files[@]}"; do
    ((i++))
    
    # Construct Label
    filename=$(basename "$f")
    track="${filename%.*}"
    dir_path=$(dirname "$f")
    album=$(basename "$dir_path")
    artist=$(basename "$(dirname "$dir_path")")
    
    if [ "$dir_path" = "$PWD" ]; then
        label="$album-$track"
    else
        label="$artist-$album-$track"
    fi
    
    # Temporary Files
    raw=$(mktemp)
    dedup=$(mktemp)
    
    # Export Tags
    if ! metaflac --export-tags-to="$raw" "$f" 2>/dev/null; then
        err_msg=$(tail -n 1 "$raw" 2>/dev/null || echo "Export failed")
        fail_count=$((fail_count + 1))
        echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAIL_LOG"
        echo "[$i/$total] ERROR (metaflac): $label :: $f :: ${err_msg:-Unknown error}" >> "$ERROR_LOG"
        rm -f "$raw" "$dedup"
        continue
    fi
    
    # Ensure file ends with newline
    [ -n "$(tail -c1 "$raw" 2>/dev/null)" ] && echo "" >> "$raw"
    
    before=$(wc -l < "$raw" | tr -d ' ')
    
    # Deduplicate tags (Case-insensitive key, keep first occurrence)
    # Note: Adjust logic if specific dedup rules are needed
    awk -F'=' '
        BEGIN { IGNORECASE = 1 }
        /^[^=]+=/ {
            key = toupper($1)
            val = substr($0, length($1) + 2)
            pair = key "=" val
            if (!seen[pair]++) {
                print $0
            }
            next
        }
        { print }
    ' "$raw" > "$dedup"
    
    after=$(wc -l < "$dedup" | tr -d ' ')
    
    if [ "$before" -ne "$after" ]; then
        if metaflac --preserve-modtime --remove-all-tags --import-tags-from="$dedup" "$f" 2>/dev/null; then
            diff=$((before - after))
            fixed_count=$((fixed_count + 1))
            echo "FIXED [$i/$total] $label ($diff duplicate tag line(s) removed)" | tee -a "$RUN_LOG" "$FIX_LOG"
        else
            fail_count=$((fail_count + 1))
            echo "FAIL [$i/$total] $label" | tee -a "$RUN_LOG" "$FAIL_LOG"
            echo "[$i/$total] ERROR (import): $label :: $f :: Tag import failed" >> "$ERROR_LOG"
        fi
    else
        ok_count=$((ok_count + 1))
        echo "OK   [$i/$total] $label" | tee -a "$RUN_LOG" "$OK_LOG"
    fi
    
    rm -f "$raw" "$dedup"
done

# 8. Generate Summary
{
echo "Step 2B Summary"
echo "==============="
echo
echo "Guide      : $GUIDE"
echo "Step       : $STEP"
echo "Run Date   : $(date)"
echo
echo "Processed  : $total"
echo "OK         : $ok_count"
echo "Fixed      : $fixed_count"
echo "Failed     : $fail_count"
echo
echo "Log Files:"
echo "  Run      : $RUN_LOG"
echo "  OKs      : $OK_LOG"
echo "  Fails    : $FAIL_LOG"
echo "  Errors   : $ERROR_LOG"
echo "  Fixes    : $FIX_LOG"
} > "$SUMMARY_LOG"

# 9. Terminal Output
echo
echo "----------------------------------------"
echo "Processed: $total  OK: $ok_count  Fixed: $fixed_count  Failed: $fail_count"
echo "----------------------------------------"
echo "Step 2B – Tag Processing & Deduplication"
echo "----------------------------------------"

```
--- Bash Script Step 2B End ---

\-------------------------------------------------------------------

-- Step 2C: Summary & Clean Up

--- Bash Script Step 2C Start ---
```bash

    #!/usr/bin/env bash
    # Step 2C: Summary & Clean Up

    bash << 'EOF'

    LOGDIR="$HOME/flac_logs"
    RUN_LOG="$LOGDIR/step2_run.log"
    
    if [ -f /tmp/step2_stats.txt ]; then
        source /tmp/step2_stats.txt
        echo "----------------------------------------" | tee -a "$RUN_LOG"
        echo "Step 2 Complete | Total: $total | OK: $ok_count | Fixed: $fixed_count | Failed: $fail_count" | tee -a "$RUN_LOG"
        rm -f /tmp/step2_targets.txt /tmp/step2_stats.txt
    else
        echo "Part C Error: No statistics found. Run Part B first."
    fi
    EOF

```
--- Bash Script Step 2C End ---

\-------------------------------------------------------------------

-- Separate Results

After the cleanup completes, separate successful, modified, and failed results:

--- Bash Script Results 2 Start ---
```bash

#!/usr/bin/env bash
# ------------------------------------------------------------
# Step 2C: Log Post-Processing & Summary
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/Linux_Audio_Folder_Level"
GUIDE="Library_Cleanup"
STEP="Step02C_Summary"

LOG_DIR="$LOG_ROOT/$GUIDE/$STEP"

# 1. Directory Setup & Cleanup (Template Standard)
mkdir -p "$LOG_ROOT/$GUIDE"
find "$LOG_DIR" -type f -delete 2>/dev/null || true
mkdir -p "$LOG_DIR"

# 2. Define Log Files (Template Standard)
RUN_LOG="$LOG_DIR/step02c_run.log"
OK_LOG="$LOG_DIR/step02c_oks.log"
FAIL_LOG="$LOG_DIR/step02c_fails.log"
ERROR_LOG="$LOG_DIR/step02c_errors.log"
SUMMARY_LOG="$LOG_DIR/step02c_summary.log"
FIX_LOG="$LOG_DIR/step02c_fixes.log"

# 3. Initialize Log Files
touch "$RUN_LOG" "$OK_LOG" "$FAIL_LOG" "$ERROR_LOG" "$FIX_LOG"

# 4. Source Run Log (From Step 2B)
# Note: Step 2B writes to step02b_run.log in its own directory
SOURCE_RUN_LOG="$LOG_ROOT/$GUIDE/Step02B_Dedup/step02b_run.log"

if [ ! -f "$SOURCE_RUN_LOG" ]; then
    echo "ERROR: Source log not found at $SOURCE_RUN_LOG" | tee -a "$RUN_LOG" "$ERROR_LOG"
    {
    echo "Step 2C Summary"
    echo "==============="
    echo "Error: Source log (Step 2B) not found."
    echo "Expected: $SOURCE_RUN_LOG"
    } > "$SUMMARY_LOG"
    exit 1
fi

# 5. Extract Results
# Note: Using -a flag to safely handle binary/null bytes if any
grep -a '^OK' "$SOURCE_RUN_LOG" > "$OK_LOG"
grep -a '^FIXED' "$SOURCE_RUN_LOG" > "$FIX_LOG"
grep -a '^FAIL' "$SOURCE_RUN_LOG" > "$FAIL_LOG"

# 6. Count Results
ok_count=$(wc -l < "$OK_LOG" | tr -d ' ')
fix_count=$(wc -l < "$FIX_LOG" | tr -d ' ')
fail_count=$(wc -l < "$FAIL_LOG" | tr -d ' ')
total_processed=$((ok_count + fix_count + fail_count))

# 7. Generate Summary
{
echo "Step 2C Summary"
echo "==============="
echo
echo "Guide      : $GUIDE"
echo "Step       : $STEP"
echo "Run Date   : $(date)"
echo
echo "Source Log : $SOURCE_RUN_LOG"
echo "Processed  : $total_processed"
echo "OK         : $ok_count"
echo "Fixed      : $fix_count"
echo "Failed     : $fail_count"
echo
echo "Output Files:"
echo "  OKs      : $OK_LOG"
echo "  Fixes    : $FIX_LOG"
echo "  Fails    : $FAIL_LOG"
} > "$SUMMARY_LOG"

# 8. Terminal Output
echo
echo "----------------------------------------"
echo "Step 2C: Log Post-Processing Complete"
echo "----------------------------------------"
echo "OKs      : $ok_count"
echo "FIXED    : $fix_count"
echo "FAILs    : $fail_count"
echo "Total    : $total_processed"
echo "----------------------------------------"
echo "Summary saved to: $SUMMARY_LOG"

```
--- Bash Script Results 2 End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat 2 Start ---
```bash

    cat "$LOGDIR/tag_dedup_errors.log"
    
    cat "$LOGDIR/tag_dedup.log"
    
    cat "$LOGDIR/step2_run.log"
    
    cat "$LOGDIR/step2_oks.log"
    
    cat "$LOGDIR/step2_fixed.log"
    
    cat "$LOGDIR/step2_fails.log"

```
--- Bash Script Cat 2 End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step2_run.log — Complete processing results.
* step2_oks.log — Files that did not require changes.
* step2_fixed.log — Files where duplicate metadata entries were removed.
* step2_fails.log — Files that could not be processed.
* tag_dedup.log — Details of metadata changes made.
* tag_dedup_errors.log — Detailed error output from failed metadata reads.

Files that fail this step should be reviewed. A failure here usually indicates that metaflac could not correctly read the file metadata and may require additional repair.

---

07. Step 3 – Strip Invalid Metadata Headers (Selective use for files that fail in above scripts).

---

-- Purpose

This step rebuilds the FLAC container to remove invalid or incompatible metadata headers that may prevent proper reading by FLAC tools and media applications.

Some FLAC files may contain unexpected ID3v2 headers or other container-level metadata issues introduced by previous software. These issues can cause inconsistent behavior even when the underlying audio data remains intact.

This process rebuilds the FLAC container while preserving the original audio stream.

-- What It Does

This step:

* Scans the selected library location recursively for FLAC files.
* Uses ffmpeg to rebuild the FLAC container.
* Copies the existing audio stream without re-encoding.
* Removes problematic container headers.
* Replaces the original file only after a successful rebuild.
* Records files that could not be processed.

No audio quality changes are introduced because the audio stream is copied rather than converted.

A warning related to embedded artwork may appear during this step. If a file reports an embedded-image issue, it may require the optional metadata cleanup procedure later in the workflow (Step 13a – Strip Problematic Metadata, followed by Step 13b – Normalize Album Artwork to restore clean cover art).

\-------------------------------------------------------------------

--- Bash Script Step 3 Start ---
```bash

    #!/usr/bin/env bash
    #Step 3 – Strip Invalid Metadata Headers
    
    LOGDIR="$HOME/flac_logs"
    mkdir -p "$LOGDIR"
    : > "$LOGDIR/ffmpeg_errors.log"
    
    mapfile -d '' files < <(
        find "$PWD" -type f -name "*.flac" ! -name "*.fixed.flac" -print0 | sort -z
    )
    
    total=${#files[@]}
    i=0
    
    for f in "${files[@]}"; do
        i=$((i+1))
    
        artist=$(basename "$(dirname "$(dirname "$f")")")
        album=$(basename "$(dirname "$f")")
        track=$(basename "$f" .flac)
    
        label="$artist-$album-$track"
    
        err=$(ffmpeg \
            -nostdin \
            -nostats \
            -loglevel error \
            -i "$f" \
            -map_metadata 0 \
            -c copy \
            "${f}.fixed.flac" \
            -y 2>&1 >/dev/null)
    
        rc=$?
    
        if [ $rc -ne 0 ] || [ -n "$err" ]; then
    
            flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
    
            rm -f "${f}.fixed.flac"
    
            echo "FAIL [$i/$total] $label"
    
            echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" \
                >> "$LOGDIR/ffmpeg_errors.log"
    
        else
    
            mv "${f}.fixed.flac" "$f"
    
            echo "OK [$i/$total] $label"
    
        fi
    
    done | tee "$LOGDIR/step3_run.log"

```
--- Bash Script Step 3 End ---

\-------------------------------------------------------------------

-- Separate Results

After the rebuild completes, separate successful and failed results:

--- Bash Script Results 3 Start ---
```bash

    LOGDIR="$HOME/flac_logs"
    
    grep '^OK' "$LOGDIR/step3_run.log" \
        > "$LOGDIR/step3_oks.log"
    
    grep '^FAIL' "$LOGDIR/step3_run.log" \
        > "$LOGDIR/step3_fails.log"
    
    echo "Step 3 OKs: $(wc -l < "$LOGDIR/step3_oks.log")  FAILs: $(wc -l < "$LOGDIR/step3_fails.log")"

```
--- Bash Script Results 3 End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat 3 Start ---
```bash

    cat "$LOGDIR/ffmpeg_errors.log"
    
    cat "$LOGDIR/step3_run.log"
    
    cat "$LOGDIR/step3_oks.log"
    
    cat "$LOGDIR/step3_fails.log"

```
--- Bash Script Cat 3 End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step3_run.log — Complete rebuild results.
* step3_oks.log — Files successfully rebuilt.
* step3_fails.log — Files that could not be rebuilt.
* ffmpeg_errors.log — Detailed error output from failed rebuilds.

After this step, files that previously contained invalid headers should have clean FLAC containers while preserving the original audio stream.

---

08. Step 4 – Repeat Step 1 Integrity Test

---

-- Purpose

This step repeats the initial integrity test performed in Step 1 after the FLAC container rebuild completed in Step 3.

The purpose of this verification is to confirm that the metadata header cleanup was successful and that the rebuilt FLAC files remain structurally valid.

Running the same integrity test again provides a direct comparison against the original baseline established before modifications were made.

-- What It Does

This step:

* Runs the same integrity test procedure used in Step 1.
* Tests each FLAC file using flac -t.
* Compares the results against the original Step 1 baseline.
* Identifies files that were repaired and files that continue to report problems.

No files are modified during this step.

This is a verification step only.

\-------------------------------------------------------------------

--- Bash Script Step 4 Start ---
```bash

    #!/usr/bin/env bash
    #Step 4 – Repeat Step 1 Integrity Test
    
    LOGDIR="$HOME/flac_logs"
    mkdir -p "$LOGDIR"
    : > "$LOGDIR/flac_test_errors_step4.log"
    
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
    
        err=$(flac -t "$f" 2>&1 >/dev/null)
        rc=$?
    
        if [ $rc -ne 0 ]; then
    
            flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
    
            echo "FAIL [$i/$total] $label"
    
            echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" \
                >> "$LOGDIR/flac_test_errors_step4.log"
    
        else
    
            echo "OK [$i/$total] $label"
    
        fi
    
    done | tee "$LOGDIR/step4_run.log"

```
--- Bash Script Step 4 End ---

\-------------------------------------------------------------------

-- Separate Results

After the integrity test completes, separate successful and failed results:

--- Bash Script Results 4 Start ---
```bash

    LOGDIR="$HOME/flac_logs"
    
    grep '^OK' "$LOGDIR/step4_run.log" \
        > "$LOGDIR/step4_oks.log"
    
    grep '^FAIL' "$LOGDIR/step4_run.log" \
        > "$LOGDIR/step4_fails.log"
    
    echo "Step 4 OKs: $(wc -l < "$LOGDIR/step4_oks.log")  FAILs: $(wc -l < "$LOGDIR/step4_fails.log")"

```
--- Bash Script Results 4 End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat 4 Start ---
```bash

    cat "$LOGDIR/flac_test_errors_step4.log"
    
    cat "$LOGDIR/step4_run.log"
    
    cat "$LOGDIR/step4_oks.log"
    
    cat "$LOGDIR/step4_fails.log"

```
--- Bash Script Cat 4 End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step4_run.log — Complete integrity test results after container rebuilding.
* step4_oks.log — Files that passed integrity testing.
* step4_fails.log — Files that still failed integrity testing.
* flac_test_errors_step4.log — Detailed error output from failed tests.

Files that failed Step 1 but pass Step 4 indicate that the container rebuild in Step 3 successfully corrected the original issue.

Files that continue to fail should be reviewed before continuing to the ReplayGain restoration step.

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

No audio is modified or re-encoded during this process.

The calculation is performed at the album level to preserve the intended relationship between tracks within an album.

\-------------------------------------------------------------------

--- Bash Script Step 5 Start ---
```bash

    #!/usr/bin/env bash
    # Step 5 – Reapply ReplayGain (moOde Audio Standard)
    
    LOGDIR="$HOME/flac_logs"
    mkdir -p "$LOGDIR"
    : > "$LOGDIR/loudgain_errors.log"
    : > "$LOGDIR/step5_run.log"
    
    # Supported audio extensions
    SUPPORTED_EXTS=(flac mp3 m4a ogg opus mp4 aac ape wv mpc spx)
    
    # 1. Gather and sort directories by path (Artist/Album) case-insensitively
    mapfile -d '' dirs < <(find "$PWD" -type d -print0 | LC_ALL=C sort -f -z)
    
    # 2. Calculate total folders containing supported audio files
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
    
    # 3. Process each directory in Artist/Album order
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
    
            # Line 1: Header output
            echo "OK [$i/$total] $label" | tee -a "$LOGDIR/step5_run.log"
    
            # Process each audio format separately to prevent TagLib container errors
            for ext in "${SUPPORTED_EXTS[@]}"; do
                shopt -s nocaseglob nullglob
                group=("$d"/*."$ext")
                shopt -u nocaseglob nullglob
    
                if [ ${#group[@]} -gt 0 ]; then
                    # Sort format group naturally
                    mapfile -d '' group_sorted < <(printf '%s\0' "${group[@]}" | LC_ALL=C sort -f -z -V)
    
                    # Option A: Run loudgain live directly to terminal (moOde standard -s e -L)
                    loudgain -a -k -s e -L -- "${group_sorted[@]}"
                    rc=$?
    
                    # Handle failure if loudgain returns a non-zero exit code
                    if [ $rc -ne 0 ]; then
                        echo "FAIL [$i/$total] $label [.$ext]" | tee -a "$LOGDIR/step5_run.log"
    
                        tracklist=""
                        n=0
                        for f in "${group_sorted[@]}"; do
                            n=$((n + 1))
                            tracklist="$tracklist $n=$(basename "$f")"
                        done
    
                        echo "[$i/$total] ERROR (exit $rc): $label [.$ext] :: $d :: tracks:$tracklist" \
                            >> "$LOGDIR/loudgain_errors.log"
                    fi
                fi
            done
            echo ""
        fi
    done

```
--- Bash Script Step 5 End ---

\-------------------------------------------------------------------

-- Separate Results

After ReplayGain processing completes, separate successful and failed results:

--- Bash Script Results 5 Start ---
```bash

    LOGDIR="$HOME/flac_logs"
    
    grep '^OK' "$LOGDIR/step5_run.log" \
        > "$LOGDIR/step5_oks.log"
    
    grep '^FAIL' "$LOGDIR/step5_run.log" \
        > "$LOGDIR/step5_fails.log"
    
    echo "Step 5 OKs: $(wc -l < "$LOGDIR/step5_oks.log")  FAILs: $(wc -l < "$LOGDIR/step5_fails.log")"

```
--- Bash Script Results 5 End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat 5 Start ---
```bash

    cat "$LOGDIR/loudgain_errors.log"
    
    cat "$LOGDIR/step5_run.log"
    
    cat "$LOGDIR/step5_oks.log"
    
    cat "$LOGDIR/step5_fails.log"

```
--- Bash Script Cat 5 End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step5_run.log — Complete ReplayGain processing results.
* step5_oks.log — Albums where ReplayGain was successfully applied.
* step5_fails.log — Albums that could not be processed.
* loudgain_errors.log — Detailed error output from failed ReplayGain calculations.

After this step, the FLAC files should contain restored ReplayGain metadata and remain unchanged from an audio-content perspective.

---

10. Step 6 – Repeat Step 1 Integrity Test

---

-- Purpose

This step repeats the integrity test from Step 1 after ReplayGain metadata has been restored.

The purpose of this verification is to confirm that the ReplayGain process completed successfully without introducing new FLAC integrity problems.

Because ReplayGain only modifies metadata and does not modify the audio stream, this test provides confirmation that the final metadata update process preserved the structural integrity of the files.

-- What It Does

This step:

* Runs the same integrity test procedure used in Step 1.
* Tests each FLAC file using flac -t.
* Confirms that files remain valid after ReplayGain processing.
* Identifies any files that developed integrity issues during metadata updates.

No files are modified during this step.

This is a verification step only.

\-------------------------------------------------------------------

--- Bash Script Step 6 Start ---
```bash

    #!/usr/bin/env bash
    # Step 6 – Repeat Step 1 Integrity Test
    
    LOGDIR="$HOME/flac_logs"
    mkdir -p "$LOGDIR"
    : > "$LOGDIR/flac_test_errors_step6.log"
    
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
    
        err=$(flac -t "$f" 2>&1 >/dev/null)
        rc=$?
    
        if [ $rc -ne 0 ]; then
    
            flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
    
            echo "FAIL [$i/$total] $label"
    
            echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" \
                >> "$LOGDIR/flac_test_errors_step6.log"
    
        else
    
            echo "OK [$i/$total] $label"
    
        fi
    
    done | tee "$LOGDIR/step6_run.log"

```
--- Bash Script Step 6 End ---

\-------------------------------------------------------------------

-- Separate Results

After the integrity test completes, separate successful and failed results:

--- Bash Script Results 6 Start ---
```bash

    LOGDIR="$HOME/flac_logs"
    
    grep '^OK' "$LOGDIR/step6_run.log" \
        > "$LOGDIR/step6_oks.log"
    
    grep '^FAIL' "$LOGDIR/step6_run.log" \
        > "$LOGDIR/step6_fails.log"
    
    echo "Step 6 OKs: $(wc -l < "$LOGDIR/step6_oks.log")  FAILs: $(wc -l < "$LOGDIR/step6_fails.log")"

```
--- Bash Script Results 6 End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat 6 Start ---
```bash

    cat "$LOGDIR/flac_test_errors_step6.log"
    
    cat "$LOGDIR/step6_run.log"
    
    cat "$LOGDIR/step6_oks.log"
    
    cat "$LOGDIR/step6_fails.log"

```
--- Bash Script Cat 6 End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step6_run.log — Complete integrity test results after ReplayGain processing.
* step6_oks.log — Files that passed integrity testing.
* step6_fails.log — Files that failed integrity testing.
* flac_test_errors_step6.log — Detailed error output from failed tests.

Files passing this test confirm that the standard repair workflow has completed without introducing new integrity problems.

Files that continue to fail should be reviewed before continuing to cleanup and archival preparation.

---

11. Step 7 – Remove Loose Files

---

-- Purpose

This step removes temporary files and unwanted artifacts created during the cleanup process.

During metadata repair and validation, temporary files may be created for testing, rebuilding, or storing intermediate results. These files are not part of the final FLAC library and should be removed before archival preparation.

This cleanup step ensures that only the intended music files and required metadata remain in the library.

-- What It Does

This step:

* Scans the selected library location recursively.
* Identifies temporary files created during processing.
* Removes incomplete rebuild files and leftover artifacts.
* Removes cleanup-related temporary files.
* Records files that were removed.

This step does not modify the audio data or FLAC metadata.

\-------------------------------------------------------------------

--- Bash Script Step 7 Start ---
```bash

    #!/usr/bin/env bash
    #Step 7 – Remove Loose Files
    
    LOGDIR="$HOME/flac_logs"
    mkdir -p "$LOGDIR"
    
    echo "Loose file cleanup started"
    
    find "$PWD" \
        -type f \
        \( \
            -name "*.fixed.flac" \
            -o -name "*.tmp" \
            -o -name "*.temp" \
            -o -name "*~" \
        \) \
        -print | tee "$LOGDIR/step7_removed_files.log"
    
    while IFS= read -r f; do
    
        rm -f "$f"
    
    done < "$LOGDIR/step7_removed_files.log"
    
    echo "Loose file cleanup completed"
    
```
--- Bash Script Step 7 End ---

\-------------------------------------------------------------------

-- Review Results

View the cleanup report:

--- Bash Script Cat 7 Start ---
```bash

LOGDIR="$HOME/flac_logs"

cat "$LOGDIR/step7_removed_files.log"

```
--- Bash Script Cat 7 End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step7_removed_files.log — List of temporary and unwanted files removed during cleanup.

If the report is empty, no loose files requiring removal were found.

After this step, the library should contain only the intended FLAC files and permanent supporting files required for archival preparation.

---

12. Step 8 – Final Integrity Test

---

-- Purpose

This step performs the final integrity verification of the FLAC library after all standard cleanup operations have been completed.

The final integrity test confirms that the complete workflow — metadata cleanup, container rebuilding, ReplayGain restoration, and loose file removal — has resulted in a stable and valid FLAC library.

This final verification provides the archival baseline for the repaired library.

-- What It Does

This step:

* Scans the selected library location recursively for FLAC files.
* Tests each file using flac -t.
* Records successful tests and failures.
* Provides the final integrity status of the library.

No files are modified during this step.

This is a verification step only.

\-------------------------------------------------------------------

--- Bash Script Step 8 Start ---
```bash

#!/usr/bin/env bash
#Step 8 – Final Integrity Test

LOGDIR="$HOME/flac_logs"
mkdir -p "$LOGDIR"
: > "$LOGDIR/flac_test_errors_final.log"

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

    err=$(flac -t "$f" 2>&1 >/dev/null)
    rc=$?

    if [ $rc -ne 0 ]; then

        flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')

        echo "FAIL [$i/$total] $label"

        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOGDIR/flac_test_errors_final.log"

    else

        echo "OK [$i/$total] $label"

    fi

done | tee "$LOGDIR/step8_run.log"

```
--- Bash Script Step 8 End ---

\-------------------------------------------------------------------

-- Separate Results

After the final integrity test completes, separate successful and failed results:

--- Bash Script Results 8 Start ---
```bash

LOGDIR="$HOME/flac_logs"

grep '^OK' "$LOGDIR/step8_run.log" \
    > "$LOGDIR/step8_oks.log"

grep '^FAIL' "$LOGDIR/step8_run.log" \
    > "$LOGDIR/step8_fails.log"

echo "Step 8 OKs: $(wc -l < "$LOGDIR/step8_oks.log")  FAILs: $(wc -l < "$LOGDIR/step8_fails.log")"

```
--- Bash Script Results 8 End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat 8 Start ---
```bash

cat "$LOGDIR/flac_test_errors_final.log"

cat "$LOGDIR/step8_run.log"

cat "$LOGDIR/step8_oks.log"

cat "$LOGDIR/step8_fails.log"

```
--- Bash Script Cat 8 End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step8_run.log — Complete final integrity test results.
* step8_oks.log — Files that passed final integrity testing.
* step8_fails.log — Files that failed final integrity testing.
* flac_test_errors_final.log — Detailed error output from failed tests.

A successful completion of this step confirms that the FLAC library has completed the standard cleanup workflow and is ready for archival preparation or additional optional maintenance procedures.

Files that continue to fail should be reviewed before archival storage. Additional troubleshooting or optional metadata repair procedures may be required.

---

13. Optional Procedures

---

The following procedures are not required for a standard library cleanup but may be necessary for resolving stubborn errors, standardizing visual presentation, or preparing the library for long-term preservation.

\-------------------------------------------------------------------

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

--- Bash Script for 13a Start ---
```bash

#!/usr/bin/env bash
#13a. Strip Problematic Metadata

LOGDIR="$HOME/flac_logs"
mkdir -p "$LOGDIR"
: > "$LOGDIR/strip_metadata_errors.log"

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
            >> "$LOGDIR/strip_metadata_errors.log"

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
                >> "$LOGDIR/strip_metadata_errors.log"

        else

            echo "FIXED [$i/$total] $label"

        fi

    fi

    rm -f "$tags"

done | tee "$LOGDIR/step13a_run.log"

```
--- Bash Script for 13a End ---

\-------------------------------------------------------------------

-- Separate Results

After the stripping process completes, separate successful and failed results:

--- Bash Script Results for 13a Start ---
```bash

LOGDIR="$HOME/flac_logs"

grep '^FIXED' "$LOGDIR/step13a_run.log" \
    > "$LOGDIR/step13a_fixed.log"

grep '^FAIL' "$LOGDIR/step13a_run.log" \
    > "$LOGDIR/step13a_fails.log"

echo "Step 13a FIXED: $(wc -l < "$LOGDIR/step13a_fixed.log")  FAILs: $(wc -l < "$LOGDIR/step13a_fails.log")"

```
--- Bash Script Results for 13a End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 13a Start ---
```bash

cat "$LOGDIR/strip_metadata_errors.log"

cat "$LOGDIR/step13a_run.log"

cat "$LOGDIR/step13a_fixed.log"

cat "$LOGDIR/step13a_fails.log"

```
--- Bash Script Cat for 13a End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

    step13a_run.log — Complete processing results.

    step13a_fixed.log — Files that had their metadata completely rebuilt.

    step13a_fails.log — Files that could not be processed.

    strip_metadata_errors.log — Detailed error output.

After running this on stubborn files, you should run the integrity test (flac -t) on them again. If they pass, the corruption was isolated to a non-audio metadata block.

\-------------------------------------------------------------------
## 13b. Normalize Album Artwork

-- Purpose

This procedure standardizes how album artwork is stored within your FLAC files.

If you performed the "Strip Problematic Metadata" procedure (13a), any previously embedded artwork was destroyed to save the container. Additionally, over years of collection, a library can accumulate wildly inconsistent artwork—some files having no art, others having massive 10MB uncompressed PNGs, and others having multiple conflicting images.

This step ensures every FLAC file in an album contains the exact same, standardized cover image by reading a local image file (like cover.jpg or folder.jpg) stored in the album directory and embedding it cleanly into the audio files.

-- What It Does

This step:

* Scans the library by album directory.
* Looks for a standard image file named cover.jpg or folder.jpg in each directory.
* If a standard image is found, it removes any existing, potentially corrupt or oversized artwork from the FLAC files in that directory.
* Embeds the standard image into each FLAC file.
* Leaves the audio data completely unchanged.
* Skips directories that do not contain a recognized standard image file.

\-------------------------------------------------------------------

--- Bash Script for 13b Start ---
```bash

#!/usr/bin/env bash
#13b. Normalize Album Artwork

LOGDIR="$HOME/flac_logs"
mkdir -p "$LOGDIR"
: > "$LOGDIR/artwork_errors.log"

mapfile -d '' dirs < <(
    find "$PWD" -type d -print0 | sort -z
)

total=0

for d in "${dirs[@]}"; do
    art_file=""

    if [ -f "$d/cover.jpg" ]; then
        art_file="$d/cover.jpg"
    elif [ -f "$d/folder.jpg" ]; then
        art_file="$d/folder.jpg"
    fi

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
    art_file=""

    if [ -f "$d/cover.jpg" ]; then
        art_file="$d/cover.jpg"
    elif [ -f "$d/folder.jpg" ]; then
        art_file="$d/folder.jpg"
    fi

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
                        >> "$LOGDIR/artwork_errors.log"
                fi

                err=$(metaflac --import-picture-from="$art_file" "$f" 2>&1 >/dev/null)
                rc=$?

                if [ $rc -ne 0 ]; then
                    error_found=1
                    flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')

                    echo "[$i/$total] ERROR (exit $rc): $label :: $(basename "$f") :: ${flat:-no stderr output}" \
                        >> "$LOGDIR/artwork_errors.log"
                fi
            done

            if [ $error_found -eq 0 ]; then
                echo "OK [$i/$total] $label (Embedded $(basename "$art_file"))"
            else
                echo "FAIL [$i/$total] $label"
            fi
        fi
    fi
done | tee "$LOGDIR/step13b_run.log"

```
--- Bash Script for 13b End ---

\-------------------------------------------------------------------

-- Separate Results

After the artwork normalization completes, separate successful and failed results:

--- Bash Script Results for 13b Start ---
```bash

LOGDIR="$HOME/flac_logs"

grep '^OK' "$LOGDIR/step13b_run.log" > "$LOGDIR/step13b_oks.log"
grep '^FAIL' "$LOGDIR/step13b_run.log" > "$LOGDIR/step13b_fails.log"

echo "Step 13b OKs: $(wc -l < "$LOGDIR/step13b_oks.log")  FAILs: $(wc -l < "$LOGDIR/step13b_fails.log")"

```
--- Bash Script Results for 13b End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 13b Start ---
```bash

cat "$LOGDIR/artwork_errors.log"
cat "$LOGDIR/step13b_run.log"
cat "$LOGDIR/step13b_oks.log"
cat "$LOGDIR/step13b_fails.log"

```
--- Bash Script Cat for 13b End ---

\-------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step13b_run.log — Complete processing results for directories containing artwork.
* step13b_oks.log — Albums successfully updated with standardized artwork.
* step13b_fails.log — Albums where metaflac encountered an error embedding the image.
* artwork_errors.log — Detailed error output from failed embeds.

Albums without a cover.jpg or folder.jpg are simply ignored by this script. To process them, place an appropriately sized JPEG in their directory and re-run the script.

\-------------------------------------------------------------------

## 13c. Update Album Artwork Embeds (Moode Compatible Formats except Wav)

--- Bash Script for 13c Start ---
```bash

#!/usr/bin/env bash
# 13c. Update Album Artwork Embeds (Moode Compatible Formats except Wav)

LOGDIR="$HOME/flac_logs"
mkdir -p "$LOGDIR"
: > "$LOGDIR/artwork_embedding_errors.log"

if ! command -v ffmpeg >/dev/null 2>&1; then
    echo "ERROR: ffmpeg is required to process non-FLAC formats."
    exit 1
fi

HAS_METAFLAC=0
if command -v metaflac >/dev/null 2>&1; then
    HAS_METAFLAC=1
fi

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

    art_file=""
    if [ -s "$d/Cover.jpg" ]; then
        art_file="$d/Cover.jpg"
    elif [ -s "$d/cover.jpg" ]; then
        art_file="$d/cover.jpg"
    elif [ -s "$d/folder.jpg" ]; then
        art_file="$d/folder.jpg"
    fi

    if [ -z "$art_file" ]; then
        echo "ERROR [$i/$total] $label :: Missing standard image file"
        echo "[$i/$total] ERROR: $label :: No Cover.jpg, cover.jpg, or folder.jpg found" >> "$LOGDIR/artwork_embedding_errors.log"
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
        echo "[$i/$total] ERROR: $label :: Directory has no supported audio files" >> "$LOGDIR/artwork_embedding_errors.log"
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
                    >> "$LOGDIR/artwork_embedding_errors.log"
            fi

            err=$(metaflac --import-picture-from="$art_file" "$f" 2>&1)
            rc=$?
            if [ $rc -ne 0 ]; then
                error_found=1
                flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
                echo "[$i/$total] ERROR (exit $rc, import-art): $label :: ${f##*/} :: ${flat:-no stderr output}" \
                    >> "$LOGDIR/artwork_embedding_errors.log"
            fi
        else
            temp_file="$d/_temp_tagged.${ext_lower}"
            rm -f "$temp_file" 2>/dev/null

            err=$(ffmpeg -y -loglevel error -i "$f" -i "$art_file" \
                -map 0:a -map 1 -c copy -disposition:v attached_pic "$temp_file" 2>&1)
            rc=$?

            if [ $rc -eq 0 ] && [ -s "$temp_file" ]; then
                mv "$temp_file" "$f" 2>/dev/null
            else
                error_found=1
                rm -f "$temp_file" 2>/dev/null
                flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
                echo "[$i/$total] ERROR (exit $rc, ffmpeg): $label :: ${f##*/} :: ${flat:-no stderr output}" \
                    >> "$LOGDIR/artwork_embedding_errors.log"
            fi
        fi
    done

    if [ $error_found -eq 0 ]; then
        echo "OK    [$i/$total] $label"
    else
        echo "ERROR [$i/$total] $label"
    fi

done | tee "$LOGDIR/step13c_all_formats_run.log"

```
--- Bash Script for 13c End ---

\-------------------------------------------------------------------

-- Separate Results

--- Bash Script Results for 13c Start ---
```bash

LOGDIR="$HOME/flac_logs"

grep '^OK' "$LOGDIR/step13c_all_formats_run.log" > "$LOGDIR/step13c_all_oks.log"
grep '^ERROR' "$LOGDIR/step13c_all_formats_run.log" > "$LOGDIR/step13c_all_fails.log"

echo "Step 13c Universal OKs: $(wc -l < "$LOGDIR/step13c_all_oks.log")  ERRORs: $(wc -l < "$LOGDIR/step13c_all_fails.log")"

```
--- Bash Script Results for 13c End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 13c Start ---
```bash

cat "$LOGDIR/artwork_embedding_errors.log"
cat "$LOGDIR/step13c_all_formats_run.log"
cat "$LOGDIR/step13c_all_oks.log"
cat "$LOGDIR/step13c_all_fails.log"

```
--- Bash Script Cat for 13c End ---

\-------------------------------------------------------------------

## 13d. Deep Repair via Decode/Re-encode (Last Resort)

--- Bash Script for 13c Start ---
```bash

#!/usr/bin/env bash
#13d. Deep Repair via Decode/Re-encode (Last Resort)

LOGDIR="$HOME/flac_logs"
mkdir -p "$LOGDIR"
: > "$LOGDIR/reencode_errors.log"
: > "$LOGDIR/reencode_review.log"

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
            >> "$LOGDIR/reencode_errors.log"
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
            >> "$LOGDIR/reencode_errors.log"
        rm -f "$tags" "${f}.reencode.flac"
        continue
    fi

    posterr=$(flac -s -t "${f}.reencode.flac" 2>&1 >/dev/null)
    postrc=$?

    if [ $postrc -ne 0 ]; then
        flat=$(echo "$posterr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (exit $postrc, post-reencode test): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOGDIR/reencode_errors.log"
        rm -f "$tags" "${f}.reencode.flac"
        continue
    fi

    impterr=$(metaflac --import-tags-from="$tags" "${f}.reencode.flac" 2>&1 >/dev/null)
    imprc=$?

    if [ $imprc -ne 0 ]; then
        flat=$(echo "$impterr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label"
        echo "[$i/$total] ERROR (exit $imprc, tag reimport): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOGDIR/reencode_errors.log"
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
            >> "$LOGDIR/reencode_review.log"
    else
        echo "FIXED-CLEAN [$i/$total] $label"
    fi

done | tee "$LOGDIR/step13d_run.log"

```
--- Bash Script for 13d End ---

\-------------------------------------------------------------------

-- Separate Results

--- Bash Script Results for 13d Start ---
```bash

LOGDIR="$HOME/flac_logs"

grep '^OK' "$LOGDIR/step13d_run.log" \
    > "$LOGDIR/step13d_oks.log"

grep '^FIXED-CLEAN' "$LOGDIR/step13d_run.log" \
    > "$LOGDIR/step13d_fixed_clean.log"

grep '^FIXED-REVIEW' "$LOGDIR/step13d_run.log" \
    > "$LOGDIR/step13d_fixed_review.log"

grep '^FAIL' "$LOGDIR/step13d_run.log" \
    > "$LOGDIR/step13d_fails.log"

echo "Step 13d OKs: $(wc -l < "$LOGDIR/step13d_oks.log")  FIXED-CLEAN: $(wc -l < "$LOGDIR/step13d_fixed_clean.log")  FIXED-REVIEW: $(wc -l < "$LOGDIR/step13d_fixed_review.log")  FAILs: $(wc -l < "$LOGDIR/step13d_fails.log")"

```
--- Bash Script Results for 13d End ---

\-------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 13d Start ---
```bash

cat "$LOGDIR/reencode_errors.log"
cat "$LOGDIR/reencode_review.log"
cat "$LOGDIR/step13d_run.log"
cat "$LOGDIR/step13d_oks.log"
cat "$LOGDIR/step13d_fixed_clean.log"
cat "$LOGDIR/step13d_fixed_review.log"
cat "$LOGDIR/step13d_fails.log"

```
--- Bash Script Cat for 13d End ---

\-------------------------------------------------------------------

14. Generate Checksums

-- Purpose

This procedure finalizes the library for long-term archival by generating cryptographic hashes for the audio files.

Checksums serve as a digital fingerprint for your files, allowing you to detect "bit rot" (silent data corruption on storage media) or verify that data remains perfectly intact after a massive transfer, such as an rsync backup to an external drive.

To prioritize strict data preservation and maintain clean, predictable file structures, this step utilizes standard, generic filenames (checksums.sha256) within each directory rather than generating dynamic or album-specific filenames.

-- What It Does

This step:

* Scans the library recursively for directories containing FLAC files.

* Changes into each directory to ensure the resulting checksum file contains only clean, relative filenames (not long, absolute system paths).

* Calculates a SHA-256 hash for every FLAC file in that directory.

* Writes the results to a static file named checksums.sha256 alongside the audio files.

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

If tools like metaflac, loudgain, or sha256sum fail to write data to a directory, it usually indicates a file ownership or permission issue. This is common when copying files from different filesystems, running tools in a virtual machine, or moving data from external drives.

To grant your current user full read and write access to the library, run:

--- Bash Script Start ---
```bash

sudo chown -R $USER:$USER /path/to/your/music/library
chmod -R u+rw /path/to/your/music/library

```
--- Bash Script End ---

\-------------------------------------------------------------------

   3. Checksum Verification Failures

If you ever run sha256sum -c checksums.sha256 in an album directory and it reports a mismatch, it means the audio file has been modified or corrupted (bit rot) since the original checksum was generated. Do not re-run the checksum generation script to "fix" the error — that will just validate the corrupted state. Instead, delete the corrupted file from your master library and restore a pristine copy from your rsync backup drive.

\-------------------------------------------------------------------

-- Log File Reference & Understanding Your Log Files

Throughout this workflow, all scripts direct their tracking data to a dedicated flac_logs directory created in your home directory ($HOME/flac_logs), regardless of which library folder you're working in. This ensures your music directories remain completely free of random text files and gives you a single, centralized place to review the results of your mass operations across every run.

The logging system uses a consistent naming convention across all processing steps:

  1. [step_name]_run.log
    The complete, raw output of the script. It lists every directory sequentially as it is processed, showing the OK or FAIL status for each one.

  2. [step_name]_oks.log
    A filtered list containing only the albums that were processed successfully.

  3. [step_name]_fails.log
    A filtered list of albums that encountered an issue. This is your primary "to-do" list for manual troubleshooting. If this file is empty, the step was a 100% success.

  4. [step_name]_errors.log
    Contains the detailed standard error (stderr) output from the specific command-line utilities (like ffmpeg, loudgain, or metaflac). When an album shows up in the fails.log, you can check this error file to see exactly why it failed (e.g., "file not found," "malformed metadata," "permission denied").

\-------------------------------------------------------------------

-- General Cleanup

Once you have reviewed the final logs, verified that your master library is fully processed, and completed your rsync transfer to the external backup drive, you can safely delete the entire flac_logs directory. It is completely independent of the audio files and is no longer needed once the project is finished.

\-------------------------------------------------------------------

-- Disclaimer

This file is generated as a mix of AI generated content, user input, and user editing. It was a cooperative effort between Claude, Gemini, ChatGPT, and user.


