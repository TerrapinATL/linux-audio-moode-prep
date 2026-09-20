### linux-audio-moode-cleanup-guide

**Version: v38** — Current version; supersedes v37. Step 1, Step 1B,
the complete Step 2 suite (2A, 2B, 2C.1-2C.6, 2D, 2E) and Step 3 are
now extensionless Python scripts (no-.sh policy); Opus tool
invocations corrected for opustags 1.9.x.
(Conversions 2026-09-20.)

Change log and version history are maintained separately:
[linux-audio-moode-cleanup-guide-changelog.md](linux-audio-moode-cleanup-guide-changelog.md)

---

01. Introduction

---

This document is a technical guide and automated script workflow for auditing, repairing, and standardizing Moode Audio–compatible music libraries on Linux systems.

It covers FLAC, MP3, M4A/AAC, WavPack, OGG, and Opus formats. Not all steps support all formats; see the Format Support Status table in Section 04 for details.

Note: Always work off a backup library copy until a clean copy has been secured by sha512 checksums.

Note: `*.prerepair` files are intentional safety copies, not corruption or residue. Step 3 (Container Rebuild) backs up every file as `FILE.prerepair` before overwriting it with the rebuilt container; Step 7 (Remove Loose Files) deletes them once their purpose is served. While they exist they are skipped by every other step (integrity tests, tag work, ReplayGain, checksums), so do not delete them manually and do not include them in SHA-512 manifests. While this backup layer exists the library temporarily occupies roughly twice its audio size, so ensure sufficient free disk space before running (see the disk-space note in the Preflight section). (The "Since v28" caution about automatic deletion refers to Step 8's own backups, which are self-deleting; Step 3's are removed by Step 7.)

---

02. Requirements

---

To successfully execute the scripts and workflows in this guide, your system must have the following command-line tools installed and available in your shell's PATH:

* flac / metaflac – Required for FLAC integrity testing and Vorbis comment metadata handling.

* ffmpeg – Required for multi-format integrity testing (Step 1), container rebuilding, and artwork embedding across Moode-compatible formats.

* loudgain – Required for calculating and writing ReplayGain metadata across FLAC, MP3, M4A, OGG, Opus, MP4, WavPack, APE, and SPX.

* eyeD3 – Not used as a command; the `eyed3` Python module it ships is required for MP3 metadata deduplication (Step 2C.3). eyeD3 0.9+ renamed the import to lowercase `eyed3`; if it is missing, install it with `sudo apt install python3-eyed3` (Debian family; disabled PEP 668) or `python3 -m pip install --user eyeD3` (where allowed; add `--break-system-packages` only as a last resort). The module is not always installed alongside the `eyeD3` command, and the CLI itself is not required by this guide.

* vorbiscomment – Required for OGG Vorbis metadata deduplication (Step 2C.5).

* opustags – Required for Opus metadata deduplication (Step 2C.5).

* AtomicParsley – Required for M4A/MP4 metadata review-flagging (Step 2C.4).

* wvtag – Required for WavPack metadata review-flagging (Step 2C.4).

* jq – Required for tag reading in Step 9 (Verify Tags Against Filenames).

* python3 – Required for MP3 metadata deduplication (Step 2C.3). The `eyed3` Python module must be importable; see the `eyeD3` entry above if it is missing.

* Core Utilities – Standard GNU core utilities (find, sort, awk, grep, wc, basename, dirname, mktemp).

-- Software Preflight

Run the preflight diagnostic below BEFORE starting any cleanup step. It verifies that every tool and Python module the guide requires is installed and importable, and fails loudly with install hints if anything is missing. This prevents silent mid-run failures and dead ends. It writes a diagnostic report to `~/.logs/linux-audio-moode-cleanup-guide/preflight.log` and does not modify any audio files.

**Disk space:** the preflight also performs a dynamic disk-space check, calculated per library — nothing is hardcoded. Step 3 backs up every file as `FILE.prerepair` before overwriting it (Step 7 removes those backups afterwards), so from Step 3 until Step 7 the library needs free space roughly equal to its own audio size. The preflight measures the audio total in the run root and the free space on that filesystem, prints both on screen (e.g. `Library audio size: 254 GB / Free space: 180 GB`), and fails loudly if free space is insufficient.

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

# 1. Steps 1-10 command-line tools
for tool in flac metaflac ffmpeg ffprobe loudgain python3 vorbiscomment opustags AtomicParsley wvtag jq; do
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
echo "Software - Pass: $pass   Missing: $missing" | tee -a "$PREFLIGHT_LOG"

# --- DISK-SPACE CHECK (dynamic, per library) -------------------------------
# Step 3 backs up every file as FILE.prerepair before overwriting it and
# Step 7 removes those backups, so the library needs free space roughly
# equal to its own audio size from Step 3 until Step 7. Measure it here,
# before anything runs — every library is different, so nothing is hardcoded.
if [ "$missing" -eq 0 ]; then
    echo "" | tee -a "$PREFLIGHT_LOG"
    echo "Preflight - Disk-Space Check" | tee -a "$PREFLIGHT_LOG"
    audio_bytes=$(find "$PWD" -type f \
        ! -ipath '*/Ignore/*' \
        ! -iname '*.prerepair*' ! -iname '*.fixed.*' ! -iname '*.reencode.*' \
        \( -iname '*.flac' -o -iname '*.mp3'  -o -iname '*.m4a' -o \
           -iname '*.ogg'  -o -iname '*.opus' -o -iname '*.wav'  -o \
           -iname '*.aiff' -o -iname '*.aif'  -o -iname '*.mp4'  -o \
           -iname '*.ape'  -o -iname '*.wv'   -o -iname '*.spx' \
        \) -printf '%s\n' 2>/dev/null | awk '{s+=$1} END {printf "%.0f", s+0.5}')
    if [ "${audio_bytes:-0}" -gt 0 ]; then
        audio_gb=$(( audio_bytes / 1073741824 ))
        avail_kb=$(df -Pk "$PWD" | awk 'NR==2 {print $4}')
        avail_gb=$(( avail_kb / 1048576 ))
        echo "Library audio size : ${audio_gb} GB" | tee -a "$PREFLIGHT_LOG"
        echo "Free space         : ${avail_gb} GB on this filesystem" | tee -a "$PREFLIGHT_LOG"
        if [ "$avail_gb" -ge "$audio_gb" ]; then
            echo "DISK SPACE: OK - the Step 3 backup layer will fit (Needed: ${audio_gb} GB / Available: ${avail_gb} GB)." | tee -a "$PREFLIGHT_LOG"
        else
            echo "" | tee -a "$PREFLIGHT_LOG"
            echo "****************************************************************" | tee -a "$PREFLIGHT_LOG"
            echo "WARNING: INSUFFICIENT DISK SPACE" | tee -a "$PREFLIGHT_LOG"
            echo "Needed: ${audio_gb} GB (Step 3 duplicates the library as" | tee -a "$PREFLIGHT_LOG"
            echo "FILE.prerepair backups until Step 7 removes them)." | tee -a "$PREFLIGHT_LOG"
            echo "Available: ${avail_gb} GB on this filesystem." | tee -a "$PREFLIGHT_LOG"
            echo "Free up space or point the run at a larger filesystem." | tee -a "$PREFLIGHT_LOG"
            echo "****************************************************************" | tee -a "$PREFLIGHT_LOG"
            missing=$((missing + 1))
            printf "%-22s : FAILED (needed ${audio_gb} GB, available ${avail_gb} GB)\n" "disk-space" >> "$PREFLIGHT_LOG"
        fi
    fi
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
        ├── 01 Track One.flac
        ├── 02 Track Two.flac
        └── ...
```

* Album folders start with a 4-digit year followed by a space and the album name: `2004 Album Name`.
* Track files are named with a zero-padded track number, then the title, no dash: `01 Track One.ext` (padded `01` through `09`, then `10` and up).
* A hyphenated variant (`NN - Title`) is also accepted by the tag tools; the padded number is what matters.

This layout is the single strongest confirmation layer in the suite:

* **Step 9 (Verify Tags Against Filenames) derives the expected tags from the folder and file names** — Artist (parent folder), Album Year/Album name (album folder), and Track number/Title (filename). If the tags disagree with the names, they are flagged as mismatches, so the filesystem itself becomes the reference schema for what a file must be tagged.
* **Tags can be rebuilt from names alone.** If embedded metadata is ever lost or corrupted, the "Write Tags from Folder/File Names" Nemo action (and the same logic used manually) can restore Artist/Album/Year/Title/TrackNumber losslessly — never re-encoding, so the audio is untouched.
* **Zero-padded track numbers sort correctly** everywhere the library is used: moOde, mpd, the SHA-512 manifests, and any file manager.
* **Checksum and recertification tools use the same layout**, so album-level and artist-level manifests stay consistent with what moOde displays.

The cleanup pipeline itself (Steps 1–10, including Step 1B) processes any folder layout: integrity testing, naming enforcement, metadata deduplication, container rebuild, and ReplayGain do not require this naming pattern. But the verification and failsafe layers — Step 9, the Write Tags action, and the checksum tools — are built around it. Following the layout gives a clean, confirmable result; deviating from it means some of those checks cannot run at full strength.

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
1b. Enforce Naming Convention
2. Deduplicate Metadata
3. Container Rebuild
4. Post-Rebuild Verification
5. ReplayGain Restoration
6. Final Integrity Validation
7. Remove Loose Files
8. Deep Repair via Decode/Re-encode
9. Verify Tags Against Filenames
10. Final Integrity Test

Optional procedures (Steps 15a–15d) handle edge cases, including Ignore-content certification (15d). Files failing all steps should be replaced.

-- Format Support Status

|------|----------------------------|----------------------------------------------|---------------------------------------|
| Step | Step Name                  | Formats                                      | Status                                |
|------|----------------------------|----------------------------------------------|---------------------------------------|
| 0    | Software Preflight         | All required tools + Python module             | Prerequisite                           |
| 1    | Initial Integrity Test     | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF, MP4, APE, WV, SPX | Complete                              |
| 1b   | Enforce Naming Convention   | Format-agnostic (renames files)                             | Complete                              |
| 2    | Deduplicate Metadata       | FLAC/MP3/M4A/MP4/WV/OGG/OPUS; other formats   | Complete for supported formats        |
|      |                            | detected, counted, reported, not modified    |                                       |
| 3    | Rebuild Containers         | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF    | Complete                              |
| 4    | Post-Rebuild Verification  | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF, MP4, APE, WV, SPX | Complete                              |
| 5    | ReplayGain Restoration     | FLAC/MP3/M4A/OGG/Opus/MP4/APE/WV/SPX          | Complete                              |
| 6    | Final Integrity Validation | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF, MP4, APE, WV, SPX | Complete                              |
| 7    | Remove Loose Files         | Format-agnostic                              | Complete                              |
| 8    | Deep Repair (Re-encode)    | FLAC only                                    | Complete — FLAC-only by design        |
| 9    | Verify Tags                | FLAC/MP3/M4A                                 | Complete — read-only failsafe         |
| 10   | Final Integrity Test       | FLAC, MP3, M4A, OGG, Opus, WAV, AIFF, AIF, MP4, APE, WV, SPX | Complete                              |
| 15a  | Strip Problematic Metadata | FLAC only                                    | Complete — FLAC-only by design        |
| 15b  | Cover Consolidation        | Format-agnostic (folder images)               | Complete — renames/converts covers, never deletes |
| 15c  | Artwork Embeds             | FLAC/MP3/M4A/MP4 (OGG/Opus/AIFF/APE/DSF skip) | Complete                              |
| 15d  | Ignore-Content Certification | Ignore folders (all formats)                | Complete                              |
| 16   | Generate Checksums         | Format-agnostic                              | See separate SHA-512 repo             |
|------|----------------------------|----------------------------------------------|---------------------------------------|


\---------------------------------------------------------------------------------------

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

--- Script Step 1 Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 1 – Multi-Format Audio Integrity Test
# ------------------------------------------------------------
import os
import re
import shutil
import subprocess
import sys
import time

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step01"

# AUTOPURGE: remove all logs from any previous run of this workflow.
# This runs at the START of the workflow, so the previous run's logs remain
# on disk for review until the next run replaces them.
shutil.rmtree(LOG_ROOT, ignore_errors=True)
os.makedirs(LOG_ROOT, exist_ok=True)


def which(name):
    return shutil.which(name) is not None


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


def keep_open_on_error(code):
    if code != 0 and sys.stdout.isatty():
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


# Software Preflight: fail loudly if a required tool is missing
for tool in ("flac", "ffmpeg"):
    if not which(tool):
        print(f"ERROR: {tool} is not installed. Install it and re-run "
              "(see Requirements).", file=sys.stderr)
        keep_open_on_error(1)

# 1. Define Log Files (Five-File Standard)
RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

# 2/3. CLEANUP + Initialize (files already gone with AUTOPURGE, touch to be sure)
for path in (RUN_LOG, OKS_LOG, FAILS_LOG, ERRORS_LOG, SUMMARY_LOG):
    open(path, "w").close()

# 3b. Optional: Pre-warm the filesystem cache (read-only)
# Asking at the start of Step 1 lets later flac -t / ffmpeg passes
# (Step 1, 4, 6, 10) read from RAM instead of a mechanical HDD.
EXTS = ["flac", "mp3", "m4a", "ogg", "opus", "wav", "aiff", "aif", "mp4",
        "ape", "wv", "spx"]
EXCLUDES = ["!", "-ipath", "*/Ignore/*",
            "!", "-iname", "*.prerepair*",
            "!", "-iname", "*.fixed.*",
            "!", "-iname", "*.reencode.*"]


def build_find(extra_exclude_manifests=False):
    args = ["find", os.getcwd(), "-type", "f", *EXCLUDES]
    if extra_exclude_manifests:
        args += ["!", "-iname", "*.sha512sums.txt"]
    args.append("(")
    for i, e in enumerate(EXTS):
        if i:
            args.append("-o")
        args += ["-iname", f"*.{e}"]
    args += [")", "-print0"]
    r = subprocess.run(args, stdout=subprocess.PIPE,
                       stderr=subprocess.DEVNULL, check=False)
    files = [p.decode("utf-8", "surrogateescape")
             for p in r.stdout.split(b"\0") if p]
    # sort -z equivalent: byte-wise sort of the NUL-separated list
    files.sort(key=lambda s: s.encode("utf-8", "surrogateescape"))
    return files


def prompt_yn(prompt):
    try:
        answer = input(prompt)
    except EOFError:
        return False
    return answer.strip().lower() in ("y", "yes")


if sys.stdin.isatty() and prompt_yn(
        "\nPre-warm the library into the page cache before testing? [y/N] "):
    PREWARM_LOG = os.path.join(LOG_ROOT, "step01-prewarm.log")
    PREWARM_ERRORS = os.path.join(LOG_ROOT, "step01-prewarm-errors.log")
    open(PREWARM_LOG, "w").close()
    with open(PREWARM_LOG, "a") as f:
        f.write("=== Step 1 Cache Pre-Warm (read-only) ===\n")
        f.write(f"Started: {time.strftime('%c')}\n")
    start = time.time()
    pw_files = build_find(extra_exclude_manifests=True)
    pw_total = len(pw_files)
    err_sink = open(PREWARM_ERRORS, "a")
    try:
        for i, path in enumerate(pw_files, 1):
            print(f"\rPre-warming: {i}/{pw_total} files "
                  f"({i * 100 // max(1, pw_total)}%)", end="", flush=True)
            with open(path, "rb") as fh:
                while fh.read(1 << 20):
                    pass
    except OSError as e:
        err_sink.write(f"{path}: {e}\n")
    finally:
        err_sink.close()
    sys.stdout.write("\r\x1b[K")
    print(f"SUMMARY: pre-warm read {pw_total} files in "
          f"{int(time.time() - start)}s.")
else:
    if sys.stdin.isatty():
        print("Skipping cache pre-warm.")

# 4. File Discovery
files = build_find(extra_exclude_manifests=False)
total = len(files)
last_dir = ""

for i, path in enumerate(files, 1):
    label = os.path.relpath(path, os.getcwd())
    current_dir = os.path.dirname(label) or "."

    # Insert a blank line on terminal screen when moving to a new folder/album
    if last_dir and current_dir != last_dir:
        print()
    last_dir = current_dir

    if path.lower().endswith(".flac"):
        r = subprocess.run(["flac", "-s", "-t", path],
                           stdout=subprocess.DEVNULL,
                           stderr=subprocess.PIPE, text=True, check=False)
    else:
        r = subprocess.run(
            ["ffmpeg", "-nostdin", "-v", "error", "-i", path, "-f", "null", "-"],
            stdout=subprocess.DEVNULL, stderr=subprocess.PIPE, text=True,
            check=False)
    rc = r.returncode

    if rc == 0:
        out_msg = f"OK   [{i}/{total}] {label}"
        print(out_msg)
        append(RUN_LOG, out_msg)
        append(OKS_LOG, out_msg)
    else:
        flat = re.sub(r"\s+", " ", r.stderr.replace("\r", " ").replace("\n", " ")).strip()
        out_msg = f"FAIL [{i}/{total}] {label}"
        print(out_msg)
        append(RUN_LOG, out_msg)
        append(FAILS_LOG, out_msg)
        append(ERRORS_LOG,
               f"[{i}/{total}] ERROR (exit {rc}): {label} :: {path} :: "
               f"{flat or 'no stderr output'}")

# 5. Count Results
with open(RUN_LOG) as f:
    run_text = f.read()
ok_count = sum(1 for l in run_text.splitlines() if l.startswith("OK"))
fail_count = sum(1 for l in run_text.splitlines() if l.startswith("FAIL"))

# 6. Generate Summary Log
with open(SUMMARY_LOG, "w") as f:
    f.write("Step 1 Summary\n")
    f.write("==============\n\n")
    f.write(f"Step       : {STEP}\n")
    f.write(f"Run Date   : {time.strftime('%c')}\n\n")
    f.write(f"Processed  : {total}\n")
    f.write(f"Passed     : {ok_count}\n")
    f.write(f"Failed     : {fail_count}\n")

# 7. Terminal Output
print()
if os.path.getsize(ERRORS_LOG) > 0:
    print("----------------------------------------")
    print("Error Summary")
    print("----------------------------------------")
    # Group error lines: LOST_SYNC vs END_OF_STREAM, path extracted from
    # the [i/total] ERROR (exit rc): label :: file :: detail line
    lost_sync, eos = {}, {}
    with open(ERRORS_LOG, encoding="utf-8", errors="replace") as f:
        for line in f:
            if "LOST_SYNC" in line:
                idx = line.find(" :: ")
                if idx > 0:
                    temp = line[:idx]
                    pos = temp.find("): ") + 3
                    lost_sync[temp[pos:]] = True
            elif "END_OF_STREAM" in line:
                idx = line.find(" :: ")
                if idx > 0:
                    temp = line[:idx]
                    pos = temp.find("): ") + 3
                    eos[temp[pos:]] = True
    if lost_sync:
        print("LOST_SYNC")
        print("----------")
        for p in sorted(lost_sync):
            print(p)
    if eos:
        if lost_sync:
            print()
        print("END_OF_STREAM")
        print("----------")
        for p in sorted(eos):
            print(p)

print()
print("----------------------------------------")
print(f"Processed: {total}  Passed: {ok_count}  Failed: {fail_count}")
print("----------------------------------------")
print("Step 1 – Initial Integrity Test")
print("----------------------------------------")

```
--- Script Step 1 End ---

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

05b. Step 1B – Enforce Naming Convention

---

-- Purpose

This step standardizes every audio file name to the `NN TrackName.ext` convention (`01`–`09` zero-padded, `10` and up natural, no dash) before any metadata work begins. It runs right after Step 1 because a rename can only be trusted once the file has passed the integrity test, and because Steps 2, 4, 5, 6, and 10 all re-scan the tree — a stable, standard file name set means those checks and every re-run of Step 1B produce provably clean logs.

It is the enforcement layer for the Library Naming Convention (Section 02b). Doing this by hand is error-prone; the script applies the rule uniformly, logs every change, and safely reports anything it cannot decide.

-- What It Does

This step:

* Scans the selected library location recursively for audio files (same extension set as Step 1).
* Skips any folder named `Ignore` at any depth (case-insensitive), and skips `.prerepair`, `.fixed.*`, and `.reencode.*` artifacts.
* Splits each file name into a leading track number + title using ordered, explicit separator rules so a title that legitimately begins with `.` or `-` (e.g. `01 ...And The Gods Made Love.flac`) is preserved, not eaten.
* Normalizes every accepted variant to the strict `NN Title` form:
  * `NN - Title`, `NN. Title`, `NN-Title`, `NN.Title`, `NN_Title` → `NN Title`
  * Single-digit numbers are zero-padded: `1 Title` → `01 Title`
  * Collapsed inner whitespace and stripped trailing spaces (e.g. `05 Track Name .flac` → `05 Track Name.flac`)
* Extracts the trailing track segment from Bandcamp-style bundled names (`Artist – Album – NN Title.ext`) and renames to just `NN Title.ext`, so mile-long download names are tamed automatically.
* Runs in DRY-RUN by default — it reports what it would rename and touches nothing. Re-run with `--apply` to commit.
* Reports anything it cannot safely decide (no usable track-number prefix) to the fails log for a hand rename rather than guessing.
* Writes a `SUMMARY:` tally of processed / conforming / to-rename / skipped files.

-- How to Run

Run from the terminal against the library root (or an artist folder for a targeted check):

--- Script Step 1B Start ---
```python

#!/usr/bin/env python3
# ============================================================
# Step 1B – Enforce Naming Convention (NN TrackName)
#   Usage: step1b-enforce /path/to/music/root [--apply]
#   Default is DRY-RUN (reports the renames it would make).
#   Pass --apply to actually rename. A verified --apply run,
#   followed by a re-run, reports 0 to-rename.
# ============================================================
import os
import re
import shutil
import subprocess
import sys
import time

TARGET = sys.argv[1] if len(sys.argv) > 1 else None
if not TARGET:
    print("Usage: step1b-enforce /path/to/music/root [--apply]", file=sys.stderr)
    sys.exit(1)
APPLY = "--apply" in sys.argv[2:]

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step1b"
MODE = "APPLY" if APPLY else "DRYRUN"

os.makedirs(LOG_ROOT, exist_ok=True)


def keep_open_on_error(code):
    if code != 0 and sys.stdout.isatty():
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


# Software Preflight: fail loudly if a required tool is missing
if shutil.which("find") is None:
    print("ERROR: find is not installed. Install it and re-run "
          "(see Requirements).", file=sys.stderr)
    keep_open_on_error(1)

# 1. Define Log Files (Five-File Standard + rename map)
RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")
RENAMES_LOG = os.path.join(LOG_ROOT, f"{STEP}-renames.log")


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


# 2/3. CLEANUP + Initialize this step's logs from any previous run
for path in (RUN_LOG, OKS_LOG, FAILS_LOG, ERRORS_LOG, SUMMARY_LOG, RENAMES_LOG):
    open(path, "w").close()


def banner(msg):
    print(msg)
    append(RUN_LOG, msg)


banner(f"========== Step 1B: Enforce Naming Convention ({MODE}) ==========")
banner(f"Root: {TARGET}")
banner(f"Started: {time.strftime('%c')}")
banner("Convention: 'NN TrackName.ext' - zero-padded number, NO dash")
banner("Folders named 'Ignore' are skipped.")
print()

# 4. File Discovery (audio extensions; skip Ignore dirs and step artifacts)
EXTS = ["flac", "mp3", "m4a", "ogg", "opus", "wav", "aiff", "aif", "mp4",
        "ape", "wv", "spx"]
find_args = ["find", TARGET, "-type", "f",
             "!", "-ipath", "*/Ignore/*",
             "!", "-iname", "*.prerepair*",
             "!", "-iname", "*.fixed.*",
             "!", "-iname", "*.reencode.*", "("]
for i, e in enumerate(EXTS):
    if i:
        find_args.append("-o")
    find_args += ["-iname", f"*.{e}"]
find_args += [")", "-print0"]

err_sink = open(ERRORS_LOG, "a")
try:
    r = subprocess.run(find_args, stdout=subprocess.PIPE,
                       stderr=err_sink, check=False)
finally:
    err_sink.close()
files = [p.decode("utf-8", "surrogateescape") for p in r.stdout.split(b"\0") if p]
files.sort(key=lambda s: s.encode("utf-8", "surrogateescape"))  # sort -z

total = len(files)
conforming = renamed = skipped = 0

# Ordered separators - explicit checks so titles beginning with '.'
# or '-' (e.g. '01 ...Moves On') are preserved instead of eaten by a
# greedy separator class:
#   1) NN - Title   2) NN. Title   3) NN-Title
#   4) NN.Title     5) NN_Title    6) NN Title
# In strict convention, all of these normalize to "NN Title".
#
# Bandcamp-style names ("Artist - Album - NN Title.ext") carry no
# leading track number. The trailing "NN Title" segment is extracted
# and the Artist/Album prefix dropped. All remaining names report to
# the fails log for a hand rename.
PATTERNS = [
    re.compile(r"^(\d+)\s+-\s+(.*)$"),
    re.compile(r"^(\d+)\.\s+(.*)$"),
    re.compile(r"^(\d+)-(.*)$"),
    re.compile(r"^(\d+)\.(.*)$"),
    re.compile(r"^(\d+)_(.*)$"),
    re.compile(r"^(\d+)\s+(.*)$"),
    re.compile(r"^.*\s-\s+(\d{1,3})(\s+)(.+)$"),
]

for i, path in enumerate(files, 1):
    label = os.path.relpath(path, TARGET) if path.startswith(
        os.path.abspath(TARGET) + os.sep) else path
    directory = os.path.dirname(path)
    base = os.path.basename(path)
    stem, ext = os.path.splitext(base)
    ext = ext[1:]  # strip leading dot

    num = rest = None
    for pi, pattern in enumerate(PATTERNS):
        m = pattern.match(stem)
        if not m:
            continue
        if pi == 6:  # Bandcamp: num and title are groups 1 and 3
            num, rest = m.group(1), m.group(3)
        else:
            num, rest = m.group(1), m.group(2)
        break

    if num is None:
        msg = f"MANUAL [{i}/{total}] {label}"
        print(msg)
        append(RUN_LOG, msg)
        append(FAILS_LOG, msg)
        append(ERRORS_LOG, f"[{i}/{total}] no leading track number available - "
                           "rename needed by hand")
        skipped += 1
        continue

    # Zero-pad track number to 2 digits
    newnum = f"{int(num):02d}"

    # Collapse internal whitespace runs, strip leading/trailing spaces
    rest = re.sub(r"\s+", " ", rest).strip()

    newbase = f"{newnum} {rest}{os.path.splitext(base)[1]}"
    target = os.path.join(directory, newbase)

    if stem == f"{newnum} {rest}":
        msg = f"OK    [{i}/{total}] {label}"
        print(msg)
        append(RUN_LOG, msg)
        append(OKS_LOG, msg)
        conforming += 1
        continue

    if os.path.exists(target) or os.path.islink(target):
        msg = f"COLLIDE [{i}/{total}] {label} -> {newbase} (target exists; left untouched)"
        print(msg)
        append(RUN_LOG, msg)
        append(FAILS_LOG, msg)
        skipped += 1
        continue

    if APPLY:
        try:
            os.rename(path, target)
            msg = f"RENAME  [{i}/{total}] {label} -> {newbase}"
        except OSError:
            msg = f"RENAME-FAIL [{i}/{total}] {label}"
            print(msg)
            append(RUN_LOG, msg)
            append(FAILS_LOG, msg)
            skipped += 1
            continue
    else:
        msg = f"WOULD-RENAME [{i}/{total}] {label} -> {newbase}"
    print(msg)
    append(RUN_LOG, msg)
    append(RENAMES_LOG, f"{label} -> {target}")
    renamed += 1

# 5. Generate Summary Log
with open(SUMMARY_LOG, "w") as f:
    f.write(f"Step 1B Summary ({MODE})\n")
    f.write("========================\n\n")
    f.write(f"Step       : {STEP}\n")
    f.write(f"Run Date   : {time.strftime('%c')}\n")
    f.write(f"Root       : {TARGET}\n")
    f.write(f"Mode       : {MODE}\n\n")
    f.write(f"Processed  : {total}\n")
    f.write(f"Conforming : {conforming}\n")
    f.write(f"To Rename  : {renamed}\n")
    f.write(f"Skipped    : {skipped}\n\n")
    if APPLY:
        f.write("Renames performed. Re-run this script (dry-run) to confirm 0 to-rename.\n")
    else:
        f.write("DRY-RUN: no files were changed. Re-run with --apply to rename.\n")

# 6. Terminal Output
print()
print("----------------------------------------")
print(f"Processed: {total}  Conforming: {conforming}  To Rename: {renamed}  "
      f"Skipped: {skipped}")
print("----------------------------------------")
print(f"Step 1B - Enforce Naming Convention ({MODE})")
print("----------------------------------------")
if os.path.getsize(RENAMES_LOG) > 0 and not APPLY:
    with open(RENAMES_LOG) as f:
        n = len(f.read().splitlines())
    print()
    print(f"Dry-run rename plan ({n} line(s)):")
    print("----------")
    with open(RENAMES_LOG) as f:
        print(f.read(), end="")

```
--- Script Step 1B End ---

Run it with your music root as the argument, e.g.:

--- Bash Script Start ---
```bash

bash /path/to/where/you/saved/this /media/youruser/music --apply

```
--- Bash Script End ---

-- Expected Results

A successful first run reports its plan in DRY-RUN; a `--apply` run performs the renames; the next DRY-RUN reports 0 to-rename and all conforming. The five standard logs plus the rename map are produced:

* step1b-run.log — Full scan record, including every file and its disposition.
* step1b-oks.log — Conforming file names.
* step1b-fails.log — Files the script could not safely decide (reported for a hand rename) and any name collisions.
* step1b-errors.log — Run-level errors (missing tools, missing input list, per-file MANUAL reasons).
* step1b-summary.log — Final tallies and status.
* step1b-renames.log — One `old -> new` line per planned or performed rename (this is the provable change record).

A `SUMMARY:` line of `Processed: N  Conforming: N  To Rename: 0  Skipped: 0` means the tree matches the convention.

\---------------------------------------------------------------------------------------

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

--- Script Step 2A Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 2A — File Discovery
# ------------------------------------------------------------
import os
import subprocess
import sys

TARGET_DIR = sys.argv[1] if len(sys.argv) > 1 else "."
LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step02a"
os.makedirs(LOG_ROOT, exist_ok=True)

RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

for path in (RUN_LOG, OKS_LOG, FAILS_LOG, ERRORS_LOG, SUMMARY_LOG):
    open(path, "w").close()

CANDIDATE_LIST = os.path.join(LOG_ROOT, "step02-candidates.txt")


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


def keep_open_on_error(code):
    if code != 0 and sys.stdout.isatty():
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


if not os.path.isdir(TARGET_DIR):
    msg = f"ERROR: target directory not found :: {TARGET_DIR}"
    append(RUN_LOG, msg)
    append(ERRORS_LOG, msg)
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    print("----------------------------------------")
    print("Step 2A - File Discovery")
    print("----------------------------------------")
    if sys.stdout.isatty() and os.path.getsize(ERRORS_LOG) > 0:
        print()
        print("==================================================")
        print(" ERRORS DETECTED — Press ENTER to view error log")
        print(" (Use arrow keys to scroll, press 'q' to exit)")
        print("==================================================")
        try:
            input()
        except EOFError:
            pass
        subprocess.run(["less", "-R", ERRORS_LOG], check=False)
    keep_open_on_error(1)

# Extensions covered by Step 2A (matches the bash find command)
EXTS = ["flac", "mp3", "m4a", "mp4", "wv", "ogg", "opus", "aac", "wav",
        "aiff", "aif", "aifc", "ape", "mpc", "spx"]

find_args = ["find", TARGET_DIR, "-type", "f",
             "!", "-ipath", "*/Ignore/*",
             "!", "-iname", "*.prerepair*",
             "!", "-iname", "*.fixed.*",
             "!", "-iname", "*.reencode*",
             "("]
for i, e in enumerate(EXTS):
    if i:
        find_args.append("-o")
    find_args += ["-iname", f"*.{e}"]
find_args += [")", "-print0"]

r = subprocess.run(find_args, stdout=subprocess.PIPE, check=False)
files = [p.decode("utf-8", "surrogateescape") for p in r.stdout.split(b"\0") if p]

with open(CANDIDATE_LIST, "wb") as f:
    for path in files:
        f.write(path.encode("utf-8", "surrogateescape") + b"\0")

found = len(files)
for i, path in enumerate(files, 1):
    append(RUN_LOG, f"OK   [{i}/{found}] :: {path}")
    append(OKS_LOG, f"OK   [{i}/{found}] :: {path}")

with open(SUMMARY_LOG, "a") as f:
    f.write(f"TOTAL_CANDIDATES={found}\n")
    f.write("STATUS=OK\n")

if sys.stdout.isatty() and os.path.getsize(ERRORS_LOG) > 0:
    print()
    print("==================================================")
    print(" ERRORS DETECTED — Press ENTER to view error log")
    print(" (Use arrow keys to scroll, press 'q' to exit)")
    print("==================================================")
    try:
        input()
    except EOFError:
        pass
    subprocess.run(["less", "-R", ERRORS_LOG], check=False)

print()
print("----------------------------------------")
print(f"Candidates found : {found}")
print("----------------------------------------")
print("Step 2A - File Discovery")
print("----------------------------------------")

```
--- Script Step 2A End ---

\ ---------------------------------------------------------------------------------------

## Step 2B — Format Assessment

Classify each file as dedupe-capable, review-capable, or unsupported.

Dedupe-capable: FLAC, MP3, OGG, Opus — auto-fixed in Step 2C.
Review-capable: M4A, MP4, WavPack — flagged in Step 2C if duplicates are found, never auto-fixed.
Unsupported: raw AAC, WAV, AIFF, AIF, AIFC, APE, MPC, SPX, and anything else Step 2A found.

--- Script Step 2B Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 2B — Format Assessment
# ------------------------------------------------------------
import os
import sys

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step02b"
os.makedirs(LOG_ROOT, exist_ok=True)

RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

for path in (RUN_LOG, OKS_LOG, FAILS_LOG, ERRORS_LOG, SUMMARY_LOG):
    open(path, "w").close()

CANDIDATE_LIST = os.path.join(LOG_ROOT, "step02-candidates.txt")


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


def keep_open_on_error(code):
    if code != 0:
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


if not (os.path.isfile(CANDIDATE_LIST) and os.path.getsize(CANDIDATE_LIST) > 0):
    msg = f"ERROR: candidate list empty or missing :: {CANDIDATE_LIST}"
    append(ERRORS_LOG, msg)
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    print(msg)
    print("Run Step 2A first.")
    keep_open_on_error(1)

DEDUPE = {"flac", "mp3", "ogg", "opus"}
REVIEW = {"m4a", "mp4", "wv"}

with open(CANDIDATE_LIST, "rb") as f:
    candidates = [p.decode("utf-8", "surrogateescape") for p in f.read().split(b"\0") if p]
total_count = len(candidates)

format_count = {}
dedupe_count = review_count = unsupported_count = 0

for i, path in enumerate(candidates, 1):
    if not os.access(path, os.R_OK):
        log_msg = f"ERROR: unreadable file :: {path}"
        append(ERRORS_LOG, log_msg)
        continue
    ext_lc = os.path.splitext(os.path.basename(path))[1].lower().lstrip(".")
    append(RUN_LOG, f"{ext_lc} [FOUND] :: {path}")
    format_count[ext_lc] = format_count.get(ext_lc, 0) + 1

    if ext_lc in DEDUPE:
        append(OKS_LOG, f"OK   [{i}/{total_count}] :: {path}")
        dedupe_count += 1
    elif ext_lc in REVIEW:
        append(FAILS_LOG, f"REVIEW [{i}/{total_count}] :: {path}")
        review_count += 1
    else:
        unsupported_count += 1

with open(SUMMARY_LOG, "a") as f:
    f.write("STATUS=OK\n")
    for fmt in sorted(format_count):
        f.write(f"{fmt}={format_count[fmt]}\n")
    f.write(f"TOTAL={total_count}\n")
    f.write(f"DEDUPE_CAPABLE={dedupe_count}\n")
    f.write(f"REVIEW_CAPABLE={review_count}\n")
    f.write(f"UNSUPPORTED={unsupported_count}\n")

print("Format Breakdown:")
for fmt in sorted(format_count):
    print(f"  {fmt.upper():<18} : {format_count[fmt]}")

print()
print("----------------------------------------")
print(f"Total: {total_count}  Dedupe: {dedupe_count}  Review: {review_count}  "
      f"Unsupported: {unsupported_count}")
print("----------------------------------------")
print("Step 2B - Format Assessment")
print("----------------------------------------")

```
--- Script Step 2B End ---
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

--- Script Step 2C.1 Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 2C.1 — Initialize & Clean Logs
# ------------------------------------------------------------
import os
import sys

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step02c"
os.makedirs(LOG_ROOT, exist_ok=True)

RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

CANDIDATE_LIST = os.path.join(LOG_ROOT, "step02-candidates.txt")

for path in (RUN_LOG, OKS_LOG, FAILS_LOG, ERRORS_LOG, SUMMARY_LOG):
    open(path, "w").close()

def bail(msg):
    for path in (RUN_LOG, ERRORS_LOG):
        with open(path, "a") as f:
            f.write(msg + "\n")
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    print(msg)
    print("Run Step 2A first.")
    print("----------------------------------------")
    print("Step 2C.1 - Initialize & Clean Logs")
    print("----------------------------------------")
    sys.exit(1)

if not (os.path.isfile(CANDIDATE_LIST) and os.path.getsize(CANDIDATE_LIST) > 0):
    bail(f"ERROR: candidate list empty or missing :: {CANDIDATE_LIST}")

with open(CANDIDATE_LIST, "rb") as f:
    total = f.read().count(b"\0")

with open(SUMMARY_LOG, "a") as f:
    f.write(f"TOTAL_CANDIDATES={total}\n")
    f.write("STATUS=OK\n")

print()
print("----------------------------------------")
print(f"Candidates found : {total}")
print("----------------------------------------")
print("Step 2C.1 - Initialize & Clean Logs")
print("----------------------------------------")

```
--- Script Step 2C.1 End ---

\---------------------------------------------------------------------------------------

## Step 2C.2 — FLAC Auto-Fix (Vorbis Comment Deduplication)

Exports all Vorbis comments with `metaflac`, drops duplicate entries that share the same key (case-insensitive) and the same value, and re-imports the deduplicated set. Audio data and non-comment metadata blocks (artwork, seek tables, padding) are left untouched. Every file is verified with `flac -t` before it is counted as OK.

--- Script Step 2C.2 Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 2C.2 — FLAC Auto-Fix (Vorbis Comment Deduplication)
# ------------------------------------------------------------
import os
import shutil
import subprocess
import sys
import time

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step02c"
os.makedirs(LOG_ROOT, exist_ok=True)

RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

CANDIDATE_LIST = os.path.join(LOG_ROOT, "step02-candidates.txt")
WORK_DIR = os.path.join(LOG_ROOT, "step02c-work")
os.makedirs(WORK_DIR, exist_ok=True)


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


def log_error(msg):
    with open(ERRORS_LOG, "a") as f:
        f.write(msg + "\n")


def run(cmd, stdin=None):
    """Run a command; append stderr to the error log; return (rc, stdout)."""
    err_sink = open(ERRORS_LOG, "a")
    try:
        r = subprocess.run(cmd, stdin=stdin, stdout=subprocess.PIPE,
                           stderr=err_sink, check=False)
    finally:
        err_sink.close()
    return r.returncode, r.stdout


def which(name):
    return shutil.which(name) is not None


def keep_open_on_error(code, stderr=sys.stderr):
    if code != 0:
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


with open(CANDIDATE_LIST, "rb") as f:
    candidates = [p.decode("utf-8", "surrogateescape") for p in f.read().split(b"\0") if p]
files = [p for p in candidates if p.lower().endswith(".flac")]
count_total = len(files)

if count_total == 0:
    with open(SUMMARY_LOG, "a") as f:
        f.write("STEP02C_FLAC_OK=0\nSTEP02C_FLAC_FAIL=0\nSTATUS=OK\n")
    print()
    print("----------------------------------------")
    print("FLAC : 0 files to process")
    print("----------------------------------------")
    print("Step 2C.2 - FLAC Auto-Fix")
    print("----------------------------------------")
    sys.exit(0)

if not (which("metaflac") and which("flac")):
    msg = ("ERROR: metaflac and flac are required for FLAC deduplication.\n"
           "       Install the flac package and re-run Step 2C.2.")
    append(RUN_LOG, msg)
    append(ERRORS_LOG, msg)
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    print("----------------------------------------")
    print("Step 2C.2 - FLAC Auto-Fix")
    print("----------------------------------------")
    keep_open_on_error(1)

start_ts = time.time()
is_tty = sys.stderr.isatty()


def progress(done_n, total_n):
    if not is_tty or total_n <= 0:
        return
    el = int(time.time() - start_ts)
    pct = done_n * 100 // total_n
    eta = el * (total_n - done_n) // done_n if done_n else 0
    sys.stderr.write(
        "\r\x1b[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   "
        % (done_n, total_n, pct, el // 3600, (el // 60) % 60, el % 60,
           eta // 3600, (eta // 60) % 60, eta % 60))
    sys.stderr.flush()


def album_header(hdr):
    if is_tty:
        sys.stderr.write("\r\x1b[K── %s ──\n" % hdr)


def flac_ok(path):
    return subprocess.run(
        ["flac", "-t", "-s", path],
        stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
        check=False).returncode == 0


def dedup_lines(lines):
    """Drop repeats sharing the same key (case-insensitive) and value."""
    seen, out = set(), []
    for line in lines:
        if "=" in line:
            key, val = line.split("=", 1)
            ident = (key.lower(), val)
        else:
            ident = line.lower()
        if ident not in seen:
            seen.add(ident)
            out.append(line)
    return out


count_ok = count_fail = 0
last_dir = ""

try:
    for i, path in enumerate(files, 1):
        progress(i, count_total)
        hdr = os.path.dirname(path)
        if last_dir and hdr != last_dir and is_tty:
            sys.stderr.write("\r\x1b[K── %s ──\n" % hdr)
        last_dir = hdr

        tmp_tags = os.path.join(WORK_DIR, f"tags.{i}")
        open(tmp_tags, "w").close()

        rc, _ = run(["metaflac", f"--export-tags-to={tmp_tags}", path])
        if rc != 0:
            if os.path.getsize(tmp_tags) == 0:
                # No comment block present — nothing to deduplicate
                if flac_ok(path):
                    count_ok += 1
                    append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
                else:
                    count_fail += 1
                    append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path}")
            else:
                count_fail += 1
                append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path}")
            append(RUN_LOG, "processed")
            os.unlink(tmp_tags)
            continue

        with open(tmp_tags, encoding="utf-8", errors="replace") as f:
            lines = f.read().splitlines()
        dedup = dedup_lines(lines)

        tmp_dedup = tmp_tags + ".dedup"
        with open(tmp_dedup, "w", encoding="utf-8") as f:
            f.write("\n".join(dedup) + ("\n" if dedup else ""))

        if dedup == lines:
            # Already clean — verify and count OK without touching the file
            if flac_ok(path):
                count_ok += 1
                append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
            else:
                count_fail += 1
                append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path}")
            append(RUN_LOG, "processed")
            os.unlink(tmp_tags)
            os.unlink(tmp_dedup)
            continue

        rc1, _ = run(["metaflac", "--remove-all-tags", path])
        rc2, _ = run(["metaflac", f"--import-tags-from={tmp_dedup}", path])
        if rc1 == 0 and rc2 == 0 and flac_ok(path):
            count_ok += 1
            append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
        else:
            # Failed replacement: restore the original tag set before it was removed
            run(["metaflac", "--remove-all-tags", path])
            rcr, _ = run(["metaflac", f"--import-tags-from={tmp_tags}", path])
            if rcr == 0 and flac_ok(path):
                count_fail += 1
                append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path} "
                                  "(metadata replacement failed; original tags restored)")
            else:
                count_fail += 1
                append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path} "
                                  "(metadata replacement AND restore failed - REVIEW)")
        append(RUN_LOG, "processed")
        os.unlink(tmp_tags)
        os.unlink(tmp_dedup)
finally:
    if is_tty:
        sys.stderr.write("\n")
    shutil.rmtree(WORK_DIR, ignore_errors=True)

with open(SUMMARY_LOG, "a") as f:
    f.write(f"STEP02C_FLAC_OK={count_ok}\n")
    f.write(f"STEP02C_FLAC_FAIL={count_fail}\n")
    f.write("STATUS=OK\n")

print()
print("----------------------------------------")
print(f"FLAC : {count_ok} OK  {count_fail} FAIL")
print("----------------------------------------")
print("Step 2C.2 - FLAC Auto-Fix")
print("----------------------------------------")

```
--- Script Step 2C.2 End ---

\---------------------------------------------------------------------------------------

## Step 2C.3 — MP3 Auto-Fix (COMMENT & TXXX Deduplication)

Deduplication is intentionally limited to COMMENT and user-text (TXXX) frames — the two frame types most likely to pick up redundant entries from years of repeated ripping and tagging passes. Standard singular frames (title, artist, album, and similar) are never touched. The check runs through the `eyed3` Python module, which exposes the individual comment and user-text frames that the CLI display silently collapses. Requires `python3` with the `eyed3` Python module (note: eyeD3 0.9+ renamed the import to lowercase `eyed3`; if the module is missing, install it with `sudo apt install python3-eyed3` or `python3 -m pip install --user eyeD3`).

--- Script Step 2C.3 Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 2C.3 — MP3 Auto-Fix (COMMENT & TXXX Deduplication)
# ------------------------------------------------------------
import os
import sys
import time

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step02c"
os.makedirs(LOG_ROOT, exist_ok=True)

RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

CANDIDATE_LIST = os.path.join(LOG_ROOT, "step02-candidates.txt")


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


def log_error(msg):
    with open(ERRORS_LOG, "a") as f:
        f.write(msg + "\n")


def keep_open_on_error(code):
    if code != 0:
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


def mp3_dedup(path):
    """Returns: 0 already clean/unreadable, 1 duplicates removed,
    2 duplicates present but could not be saved safely, 3 engine error."""
    try:
        import eyed3
        import eyed3.id3 as ID3
        import eyed3.id3.frames as FRAMES
    except Exception as e:
        log_error(f"eyed3 module unavailable: {e}")
        return 3

    try:
        audio = eyed3.load(path)
    except Exception as e:
        log_error(f"unable to load file: {e}")
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
            log_error(f"unable to save changes: {e}")
            return 2  # duplicates present but could not be saved safely
        return 1  # duplicates removed
    return 0  # already clean


if not sys.executable:
    keep_open_on_error(1)

with open(CANDIDATE_LIST, "rb") as f:
    candidates = [p.decode("utf-8", "surrogateescape") for p in f.read().split(b"\0") if p]
files = [p for p in candidates if p.lower().endswith(".mp3")]
count_total = len(files)

if count_total == 0:
    with open(SUMMARY_LOG, "a") as f:
        f.write("STEP02C_MP3_OK=0\nSTEP02C_MP3_FAIL=0\nSTEP02C_MP3_REVIEW=0\nSTATUS=OK\n")
    print()
    print("----------------------------------------")
    print("MP3 : 0 files to process")
    print("----------------------------------------")
    print("Step 2C.3 - MP3 Auto-Fix")
    print("----------------------------------------")
    sys.exit(0)

try:
    import eyed3  # noqa: F401
except Exception:
    msg = ("ERROR: the eyed3 Python module is not importable.\n"
           "       Install it with:  sudo apt install python3-eyed3  "
           "(or: python3 -m pip install --user eyeD3)")
    append(RUN_LOG, msg)
    append(ERRORS_LOG, msg)
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    print("----------------------------------------")
    print("Step 2C.3 - MP3 Auto-Fix")
    print("----------------------------------------")
    keep_open_on_error(1)

start_ts = time.time()
is_tty = sys.stderr.isatty()


def progress(done_n, total_n):
    if not is_tty or total_n <= 0:
        return
    el = int(time.time() - start_ts)
    pct = done_n * 100 // total_n
    eta = el * (total_n - done_n) // done_n if done_n else 0
    sys.stderr.write(
        "\r\x1b[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   "
        % (done_n, total_n, pct, el // 3600, (el // 60) % 60, el % 60,
           eta // 3600, (eta // 60) % 60, eta % 60))
    sys.stderr.flush()


count_ok = count_fail = count_review = 0
last_dir = ""

for i, path in enumerate(files, 1):
    progress(i, count_total)
    hdr = os.path.dirname(path)
    if last_dir and hdr != last_dir and is_tty:
        sys.stderr.write("\r\x1b[K── %s ──\n" % hdr)
    last_dir = hdr

    try:
        rc = mp3_dedup(path)
    except Exception as e:
        import traceback
        log_error(f"UNEXPECTED ERROR: {e}\n{traceback.format_exc()}")
        rc = 4

    if rc in (0, 1):
        count_ok += 1
        append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
    elif rc == 2:
        count_review += 1
        append(FAILS_LOG, f"REVIEW [{i}/{count_total}] :: {path} "
                          "(duplicate frames detected but could not be removed safely)")
    else:
        count_fail += 1
        append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path} "
                          "(dedup engine error - see step02c-errors.log)")
    append(RUN_LOG, "processed")

if is_tty:
    sys.stderr.write("\n")

with open(SUMMARY_LOG, "a") as f:
    f.write(f"STEP02C_MP3_OK={count_ok}\n")
    f.write(f"STEP02C_MP3_FAIL={count_fail}\n")
    f.write(f"STEP02C_MP3_REVIEW={count_review}\n")
    f.write("STATUS=OK\n")

print()
print("----------------------------------------")
print(f"MP3 : {count_ok} OK  {count_review} REVIEW  {count_fail} FAIL")
print("----------------------------------------")
print("Step 2C.3 - MP3 Auto-Fix")
print("----------------------------------------")

```
--- Script Step 2C.3 End ---

\---------------------------------------------------------------------------------------

## Step 2C.4 — M4A/MP4 & WavPack Review Flag

The native tools for these containers (`AtomicParsley`, `wvtag`) do not support safe auto-fixing, so Step 2C.4 detects confirmed duplicate metadata entries and flags the affected files for manual review. Files are never modified by this sub-step and the audio stream is never re-encoded.

--- Script Step 2C.4 Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 2C.4 — M4A/MP4 & WavPack Review Flag
# ------------------------------------------------------------
import os
import shutil
import subprocess
import sys
import time
from collections import Counter

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step02c"
os.makedirs(LOG_ROOT, exist_ok=True)

RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

CANDIDATE_LIST = os.path.join(LOG_ROOT, "step02-candidates.txt")
WORK_DIR = os.path.join(LOG_ROOT, "step02c-work")
os.makedirs(WORK_DIR, exist_ok=True)


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


def log_error(msg):
    with open(ERRORS_LOG, "a") as f:
        f.write(msg + "\n")


def run(cmd):
    err_sink = open(ERRORS_LOG, "a")
    try:
        r = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=err_sink, text=True,
                           check=False)
    finally:
        err_sink.close()
    return r.returncode, r.stdout


def which(name):
    return shutil.which(name) is not None


def keep_open_on_error(code):
    if code != 0:
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


with open(CANDIDATE_LIST, "rb") as f:
    candidates = [p.decode("utf-8", "surrogateescape") for p in f.read().split(b"\0") if p]
ext = lambda p: os.path.splitext(p)[1].lower()
files = [p for p in candidates if ext(p) in (".m4a", ".mp4", ".wv")]
count_total = len(files)

if count_total == 0:
    with open(SUMMARY_LOG, "a") as f:
        f.write("STEP02C_M4A_WV_CLEAN=0\nSTEP02C_M4A_WV_REVIEW=0\n"
                "STEP02C_M4A_WV_FAIL=0\nSTATUS=OK\n")
    print()
    print("----------------------------------------")
    print("M4A/MP4/WV : 0 files to review")
    print("----------------------------------------")
    print("Step 2C.4 - M4A/MP4 & WavPack Review Flag")
    print("----------------------------------------")
    sys.exit(0)

if any(ext(p) in (".m4a", ".mp4") for p in files) and not which("AtomicParsley"):
    msg = "ERROR: AtomicParsley is missing but M4A/MP4 files were found.\n" \
          "       Install AtomicParsley and re-run Step 2C.4."
    append(RUN_LOG, msg)
    append(ERRORS_LOG, msg)
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    print("----------------------------------------")
    print("Step 2C.4 - M4A/MP4 & WavPack Review Flag")
    print("----------------------------------------")
    keep_open_on_error(1)

if any(ext(p) == ".wv" for p in files) and not which("wvtag"):
    msg = "ERROR: wvtag is missing but WavPack files were found.\n" \
          "       Install wavpack and re-run Step 2C.4."
    append(RUN_LOG, msg)
    append(ERRORS_LOG, msg)
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    print("----------------------------------------")
    print("Step 2C.4 - M4A/MP4 & WavPack Review Flag")
    print("----------------------------------------")
    keep_open_on_error(1)

start_ts = time.time()
is_tty = sys.stderr.isatty()


def progress(done_n, total_n):
    if not is_tty or total_n <= 0:
        return
    el = int(time.time() - start_ts)
    pct = done_n * 100 // total_n
    eta = el * (total_n - done_n) // done_n if done_n else 0
    sys.stderr.write(
        "\r\x1b[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   "
        % (done_n, total_n, pct, el // 3600, (el // 60) % 60, el % 60,
           eta // 3600, (eta // 60) % 60, eta % 60))
    sys.stderr.flush()


def has_duplicates(text):
    """True if any (trailing-space-stripped) line repeats."""
    lines = [l.rstrip() for l in text.splitlines()]
    return any(n > 1 for n in Counter(lines).values())


count_clean = count_review = count_fail = 0
last_dir = ""

try:
    for i, path in enumerate(files, 1):
        progress(i, count_total)
        hdr = os.path.dirname(path)
        if last_dir and hdr != last_dir and is_tty:
            sys.stderr.write("\r\x1b[K── %s ──\n" % hdr)
        last_dir = hdr

        kind = ext(path)

        if kind in (".m4a", ".mp4"):
            rc, atoms = run(["AtomicParsley", path, "-t"])
            if rc != 0:
                count_fail += 1
                append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path} "
                                  "(AtomicParsley could not read metadata)")
            elif not atoms.strip():
                count_clean += 1
                append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
            elif has_duplicates(atoms):
                count_review += 1
                append(FAILS_LOG, f"REVIEW [{i}/{count_total}] :: {path} "
                                  "(duplicate metadata atoms detected)")
            else:
                count_clean += 1
                append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
            append(RUN_LOG, "processed")
            continue

        if kind == ".wv":
            rc, out = run(["wvtag", "-l", path])
            if rc != 0:
                count_review += 1
                append(FAILS_LOG, f"REVIEW [{i}/{count_total}] :: {path} "
                                  "(wvtag could not parse the tag block)")
            else:
                lines = [l.rstrip() for l in out.splitlines()]
                if any(n > 1 for n in Counter(lines).values()):
                    count_review += 1
                    append(FAILS_LOG, f"REVIEW [{i}/{count_total}] :: {path} "
                                      "(duplicate tag items detected)")
                else:
                    count_clean += 1
                    append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
            append(RUN_LOG, "processed")
finally:
    if is_tty:
        sys.stderr.write("\n")
    shutil.rmtree(WORK_DIR, ignore_errors=True)

with open(SUMMARY_LOG, "a") as f:
    f.write(f"STEP02C_M4A_WV_CLEAN={count_clean}\n")
    f.write(f"STEP02C_M4A_WV_REVIEW={count_review}\n")
    f.write(f"STEP02C_M4A_WV_FAIL={count_fail}\n")
    f.write("STATUS=OK\n")

print()
print("----------------------------------------")
print(f"M4A/MP4/WV : {count_clean} OK  {count_review} REVIEW  {count_fail} FAIL")
print("----------------------------------------")
print("Step 2C.4 - M4A/MP4 & WavPack Review Flag")
print("----------------------------------------")

```
--- Script Step 2C.4 End ---

\---------------------------------------------------------------------------------------

## Step 2C.5 — OGG & Opus Auto-Fix (Vorbis Comment Deduplication)

OGG Vorbis and Opus share the Vorbis comment model, so duplicates are resolved the same way as FLAC: export every comment, drop repeats that share the same key (case-insensitive) and the same value, then re-import the deduplicated set with the format's native writer. Audio streams are never re-encoded. Every modified file is verified with an `ffmpeg` decode-to-null check.

--- Script Step 2C.5 Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 2C.5 — OGG & Opus Auto-Fix (Vorbis Comment Deduplication)
# ------------------------------------------------------------
import os
import shutil
import subprocess
import sys
import time
from collections import Counter

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step02c"
os.makedirs(LOG_ROOT, exist_ok=True)

RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

CANDIDATE_LIST = os.path.join(LOG_ROOT, "step02-candidates.txt")
WORK_DIR = os.path.join(LOG_ROOT, "step02c-work")
os.makedirs(WORK_DIR, exist_ok=True)


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


def run(cmd, stdin=None):
    """Run a command; append stderr to ERRORS_LOG; return (rc, stdout bytes)."""
    err_sink = open(ERRORS_LOG, "a")
    try:
        r = subprocess.run(cmd, stdin=stdin, stdout=subprocess.PIPE,
                           stderr=err_sink, check=False)
    finally:
        err_sink.close()
    return r.returncode, r.stdout


def which(name):
    return shutil.which(name) is not None


def keep_open_on_error(code):
    if code != 0:
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


def dedup_lines(lines):
    """Drop repeats that share the same key (case-insensitive) and value."""
    seen, out = set(), []
    for line in lines:
        if "=" in line:
            key, val = line.split("=", 1)
            ident = (key.lower(), val)
        else:
            ident = line.lower()
        if ident not in seen:
            seen.add(ident)
            out.append(line)
    return out


def ffmpeg_decodes(path):
    r = subprocess.run(
        ["ffmpeg", "-nostdin", "-v", "error", "-i", path, "-f", "null", "-"],
        stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, check=False)
    return r.returncode == 0


with open(CANDIDATE_LIST, "rb") as f:
    candidates = [p.decode("utf-8", "surrogateescape") for p in f.read().split(b"\0") if p]
ext = lambda p: os.path.splitext(p)[1].lower()
files = [p for p in candidates if ext(p) in (".ogg", ".opus")]
count_total = len(files)

if count_total == 0:
    with open(SUMMARY_LOG, "a") as f:
        f.write("STEP02C_VORBIS_OK=0\nSTEP02C_VORBIS_FAIL=0\nSTATUS=OK\n")
    print()
    print("----------------------------------------")
    print("OGG/OPUS : 0 files to process")
    print("----------------------------------------")
    print("Step 2C.5 - OGG & Opus Auto-Fix")
    print("----------------------------------------")
    sys.exit(0)

if any(ext(p) == ".ogg" for p in files) and not which("vorbiscomment"):
    msg = ("ERROR: vorbiscomment is missing but OGG files were found.\n"
           "       Install vorbis-tools and re-run Step 2C.5.")
    append(RUN_LOG, msg)
    append(ERRORS_LOG, msg)
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    print("----------------------------------------")
    print("Step 2C.5 - OGG & Opus Auto-Fix")
    print("----------------------------------------")
    keep_open_on_error(1)

if any(ext(p) == ".opus" for p in files) and not which("opustags"):
    msg = ("ERROR: opustags is missing but Opus files were found.\n"
           "       Install opustags and re-run Step 2C.5.")
    append(RUN_LOG, msg)
    append(ERRORS_LOG, msg)
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    print("----------------------------------------")
    print("Step 2C.5 - OGG & Opus Auto-Fix")
    print("----------------------------------------")
    keep_open_on_error(1)

if not which("ffmpeg"):
    msg = ("ERROR: ffmpeg is missing but is used to verify OGG/Opus rewrites.\n"
           "       Install ffmpeg and re-run Step 2C.5.")
    append(RUN_LOG, msg)
    append(ERRORS_LOG, msg)
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    print("----------------------------------------")
    print("Step 2C.5 - OGG & Opus Auto-Fix")
    print("----------------------------------------")
    keep_open_on_error(1)

start_ts = time.time()
is_tty = sys.stderr.isatty()


def progress(done_n, total_n):
    if not is_tty or total_n <= 0:
        return
    el = int(time.time() - start_ts)
    pct = done_n * 100 // total_n
    eta = el * (total_n - done_n) // done_n if done_n else 0
    sys.stderr.write(
        "\r\x1b[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   "
        % (done_n, total_n, pct, el // 3600, (el // 60) % 60, el % 60,
           eta // 3600, (eta // 60) % 60, eta % 60))
    sys.stderr.flush()


count_ok = count_fail = 0
last_dir = ""

try:
    for i, path in enumerate(files, 1):
        progress(i, count_total)
        hdr = os.path.dirname(path)
        if last_dir and hdr != last_dir and is_tty:
            sys.stderr.write("\r\x1b[K── %s ──\n" % hdr)
        last_dir = hdr

        kind = ext(path)
        result = "fail"
        if kind == ".ogg":
            # Export all comments; fail hard if the tool cannot read the file
            rc, raw = run(["vorbiscomment", "-l", path])
            if rc != 0:
                append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path} "
                                  "(vorbiscomment could not read the file)")
                append(RUN_LOG, "processed")
                count_fail += 1
                continue
            lines = [l for l in raw.decode("utf-8", "replace").splitlines()
                     if "=" in l]
            if not lines:
                count_ok += 1
                append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
                append(RUN_LOG, "processed")
                continue
            dedup = dedup_lines(lines)
            if dedup == lines:
                count_ok += 1
                append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
                append(RUN_LOG, "processed")
                continue
            dedup_text = "\n".join(dedup) + "\n"
            r = subprocess.run(["vorbiscomment", "-w", path],
                               input=dedup_text.encode(),
                               stdout=subprocess.DEVNULL,
                               stderr=subprocess.DEVNULL, check=False)
            if r.returncode == 0 and ffmpeg_decodes(path):
                count_ok += 1
                append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
            else:
                count_fail += 1
                append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path}")
            append(RUN_LOG, "processed")
            continue

        if kind == ".opus":
            # opustags 1.9.0: bare `opustags FILE` prints the comment list
            # (there is no -l option; the guide's bash version predates 1.9)
            rc, raw = run(["opustags", path])
            if rc != 0:
                count_fail += 1
                append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path} "
                                  "(opustags could not read the file)")
                append(RUN_LOG, "processed")
                continue
            lines = [l for l in raw.decode("utf-8", "replace").splitlines()
                     if "=" in l]
            if not lines:
                count_ok += 1
                append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
                append(RUN_LOG, "processed")
                continue
            dedup = dedup_lines(lines)
            if dedup == lines:
                count_ok += 1
                append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
                append(RUN_LOG, "processed")
                continue
            dedup_text = "\n".join(dedup) + "\n"
            # -S imports comments from standard input; -i rewrites in place
            r = subprocess.run(["opustags", "-S", "-i", path],
                               input=dedup_text.encode(),
                               stdout=subprocess.DEVNULL,
                               stderr=subprocess.DEVNULL, check=False)
            if r.returncode == 0 and ffmpeg_decodes(path):
                count_ok += 1
                append(OKS_LOG, f"OK   [{i}/{count_total}] :: {path}")
            else:
                count_fail += 1
                append(FAILS_LOG, f"FAIL [{i}/{count_total}] :: {path}")
            append(RUN_LOG, "processed")
            continue
finally:
    if is_tty:
        sys.stderr.write("\n")
    shutil.rmtree(WORK_DIR, ignore_errors=True)

with open(SUMMARY_LOG, "a") as f:
    f.write(f"STEP02C_VORBIS_OK={count_ok}\n")
    f.write(f"STEP02C_VORBIS_FAIL={count_fail}\n")
    f.write("STATUS=OK\n")

print()
print("----------------------------------------")
print(f"OGG/OPUS : {count_ok} OK  {count_fail} FAIL")
print("----------------------------------------")
print("Step 2C.5 - OGG & Opus Auto-Fix")
print("----------------------------------------")

```
--- Script Step 2C.5 End ---

\---------------------------------------------------------------------------------------

## Step 2C.6 — Summary

Aggregates the Step 2C sub-step results into `step02c-summary.log` and prints the final Step 2C status. Step 2E reads this summary file.

--- Script Step 2C.6 Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 2C.6 — Summary
# ------------------------------------------------------------
import os
import re
import sys
import time

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step02c"
os.makedirs(LOG_ROOT, exist_ok=True)

OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")


def read(path):
    try:
        with open(path, encoding="utf-8", errors="replace") as f:
            return f.read()
    except FileNotFoundError:
        return ""


def count_prefix(text, prefix):
    return sum(1 for l in text.splitlines() if l.startswith(prefix))


def kv(text, key):
    """Per-format counter: last value wins, since repeat runs append."""
    vals = re.findall(rf"^{key}=(.*)$", text, flags=re.MULTILINE)
    return vals[-1] if vals else "0"


oks = read(OKS_LOG)
fails = read(FAILS_LOG)
summary = read(SUMMARY_LOG)

ok_count = sum(1 for l in oks.splitlines() if l.startswith("OK"))
fail_count = sum(1 for l in fails.splitlines() if l.startswith("FAIL"))
review_count = sum(1 for l in fails.splitlines() if l.startswith("REVIEW"))

total = kv(summary, "TOTAL_CANDIDATES")
flac_ok = kv(summary, "STEP02C_FLAC_OK")
flac_fail = kv(summary, "STEP02C_FLAC_FAIL")
mp3_ok = kv(summary, "STEP02C_MP3_OK")
mp3_fail = kv(summary, "STEP02C_MP3_FAIL")
mp3_review = kv(summary, "STEP02C_MP3_REVIEW")
m4a_clean = kv(summary, "STEP02C_M4A_WV_CLEAN")
m4a_fail = kv(summary, "STEP02C_M4A_WV_FAIL")
m4a_review = kv(summary, "STEP02C_M4A_WV_REVIEW")
ogg_ok = kv(summary, "STEP02C_VORBIS_OK")
ogg_fail = kv(summary, "STEP02C_VORBIS_FAIL")

rows = [
    ("FLAC", flac_ok, flac_fail, "-"),
    ("MP3", mp3_ok, mp3_fail, mp3_review),
    ("M4A/MP4/WV", m4a_clean, m4a_fail, m4a_review),
    ("OGG/OPUS", ogg_ok, ogg_fail, "-"),
]

report = "\n".join(
    [f"Step 2C Summary",
     f"==============",
     "",
     f"Step       : step02c",
     f"Run Date   : {time.strftime('%c')}",
     "",
     f"Processed  : {total}",
     f"OK         : {ok_count}",
     f"FAIL       : {fail_count}",
     f"REVIEW     : {review_count}",
     "",
     f"Per-format breakdown (OK/clean, FAIL, REVIEW):",
     *[f"  {name:<12} {a:>10} {b:>6} {c:>7}" for name, a, b, c in rows],
 ]) + "\n"

with open(SUMMARY_LOG, "w") as f:
    f.write(report)

# Terminal output: the summary review comes FIRST; the footer is
# strictly the final output before the shell prompt returns.
print()
print("----------------------------------------")
print("Step 2C Summary Review")
print("----------------------------------------")
print(f"Processed  : {total}")
print(f"OK         : {ok_count}")
print(f"FAIL       : {fail_count}")
print(f"REVIEW     : {review_count}")
print()
print("Format        OK/clean  FAIL  REVIEW")
for name, a, b, c in rows:
    print(f"  {name:<12} {a:>10} {b:>6} {c:>7}")
print()
print(f"Summary written to : {SUMMARY_LOG}")
print()
print("----------------------------------------")
print("Step 2C.6 - Summary")
print("----------------------------------------")

```
--- Script Step 2C.6 End ---

\---------------------------------------------------------------------------------------

## Step 2D — Verification

Verify the files modified by Step 2C and confirm that the intended cleanup occurred.

FLAC files are verified with `flac -t`. Every other modified format is verified with an `ffmpeg` decode-to-null stream check. `ffmpeg` is run with `-nostdin` — without it, `ffmpeg` shares the loop's input stream and silently consumes a byte meant for the next file, corrupting the following iteration's path. This surfaced during testing and is the same class of bug already fixed once before in the artwork-normalization scripts.

--- Script Step 2D Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 2D — Verification
# ------------------------------------------------------------
import os
import shutil
import subprocess
import sys
import time

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step02d"
os.makedirs(LOG_ROOT, exist_ok=True)

RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

for path in (RUN_LOG, OKS_LOG, FAILS_LOG, ERRORS_LOG, SUMMARY_LOG):
    open(path, "w").close()

CANDIDATE_LIST = os.path.join(LOG_ROOT, "step02-candidates.txt")


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


def keep_open_on_error(code):
    if code != 0 and sys.stdout.isatty():
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


def ask_yn(prompt):
    if not sys.stdin.isatty():
        return False
    try:
        choice = input(prompt)
    except EOFError:
        return False
    return choice.strip().lower() in ("y", "yes")


# Software Preflight: fail loudly if a required tool is missing
for tool in ("flac", "ffmpeg", "metaflac"):
    if shutil.which(tool) is None:
        print(f"ERROR: {tool} is not installed. Install it and re-run "
              "(see Requirements).", file=sys.stderr)
        keep_open_on_error(1)

if not (os.path.isfile(CANDIDATE_LIST) and os.path.getsize(CANDIDATE_LIST) > 0):
    msg = f"ERROR: candidate list empty or missing :: {CANDIDATE_LIST}"
    append(RUN_LOG, msg)
    append(ERRORS_LOG, msg)
    with open(SUMMARY_LOG, "a") as f:
        f.write("STATUS=ERROR\n")
    if sys.stdout.isatty() and os.path.getsize(ERRORS_LOG) > 0:
        print()
        if ask_yn("Would you like to view the error log? [Y/N]: "):
            print("----------------------------------------")
            print("ERROR LOG DUMP:")
            print("----------------------------------------")
            with open(ERRORS_LOG) as f:
                print(f.read())
    print("Status: FAILED (Run Step 2A first)")
    print("----------------------------------------")
    print("Step 2D - Verification")
    print("----------------------------------------")
    keep_open_on_error(1)

with open(CANDIDATE_LIST, "rb") as f:
    candidates = [p.decode("utf-8", "surrogateescape") for p in f.read().split(b"\0") if p]
total_files = len(candidates)

start_ts = time.time()
is_tty = sys.stderr.isatty()


def progress(done_n, total_n):
    if not is_tty or total_n <= 0:
        return
    el = int(time.time() - start_ts)
    pct = done_n * 100 // total_n
    eta = el * (total_n - done_n) // done_n if done_n else 0
    sys.stderr.write(
        "\r\x1b[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   "
        % (done_n, total_n, pct, el // 3600, (el // 60) % 60, el % 60,
           eta // 3600, (eta // 60) % 60, eta % 60))
    sys.stderr.flush()


print("Notice: Integrity checking in progress.")
print(f"Total files to process: {total_files}")
print()

passed_count = corrupt_count = error_count = 0
last_dir = ""

for current, path in enumerate(candidates, 1):
    progress(current, total_files)
    hdr = os.path.dirname(path)
    if last_dir and hdr != last_dir and is_tty:
        sys.stderr.write("\r\x1b[K── %s ──\n" % hdr)
    last_dir = hdr

    if not os.access(path, os.R_OK):
        append(RUN_LOG, f"ERROR [{current}/{total_files}] :: {path} (unreadable)")
        append(ERRORS_LOG, f"ERROR [{current}/{total_files}] :: {path} (unreadable)")
        error_count += 1
        continue

    ext_lc = os.path.splitext(os.path.basename(path))[1].lower().lstrip(".")
    if ext_lc == "flac":
        cmd = ["flac", "-t", "-s", path]
    else:
        # -nostdin: without it ffmpeg shares the loop's input stream and
        # silently consumes bytes meant for the next file
        cmd = ["ffmpeg", "-nostdin", "-v", "error", "-i", path, "-f", "null", "-"]

    r = subprocess.run(cmd, stdout=subprocess.DEVNULL,
                       stderr=subprocess.DEVNULL, check=False)
    if r.returncode == 0:
        passed_count += 1
        append(OKS_LOG, f"OK   [{current}/{total_files}] :: {path}")
    else:
        corrupt_count += 1
        append(FAILS_LOG, f"FAIL [{current}/{total_files}] :: {path}")
    append(RUN_LOG, "processed")

if is_tty:
    sys.stderr.write("\n")

with open(SUMMARY_LOG, "a") as f:
    f.write(f"PASSED_FILES={passed_count}\n")
    f.write(f"CORRUPT_FILES={corrupt_count}\n")
    f.write(f"ERROR_FILES={error_count}\n")
    f.write("STATUS=OK\n")

# Interactive Screen Dump Prompts
if sys.stdout.isatty() and os.path.getsize(ERRORS_LOG) > 0:
    print()
    if ask_yn("ERRORS DETECTED — Would you like to view the error log? [Y/N]: "):
        print("----------------------------------------")
        print("ERROR LOG DUMP:")
        print("----------------------------------------")
        with open(ERRORS_LOG) as f:
            print(f.read())
elif sys.stdout.isatty() and os.path.getsize(FAILS_LOG) > 0:
    print()
    if ask_yn("CORRUPT/FAILED FILES DETECTED — Would you like to view the log? [Y/N]: "):
        print("----------------------------------------")
        print("CORRUPT FILES LOG DUMP:")
        print("----------------------------------------")
        with open(FAILS_LOG) as f:
            print(f.read())

# Footer — Strictly the final output before shell prompt returns
print()
print("----------------------------------------")
print(f"Passed integrity check : {passed_count}")
print(f"Corrupt/Failed files   : {corrupt_count}")
print(f"System/Read errors     : {error_count}")
print("----------------------------------------")
print("Step 2D - Verification")
print("----------------------------------------")

```
--- Script Step 2D End ---

\ ---------------------------------------------------------------------------------------

## Step 2E — Summary

Produce the final Step 2 results and status.

--- Script Step 2E Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 2E — Summary
# ------------------------------------------------------------
import os
import sys

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step02e"
os.makedirs(LOG_ROOT, exist_ok=True)

RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

for path in (RUN_LOG, OKS_LOG, FAILS_LOG, ERRORS_LOG, SUMMARY_LOG):
    open(path, "w").close()


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


for sub in ("step02a", "step02b", "step02c", "step02d"):
    src = os.path.join(LOG_ROOT, f"{sub}-summary.log")
    if os.path.isfile(src):
        append(RUN_LOG, f"OK found summary :: {sub}")
        append(OKS_LOG, f"OK found summary :: {sub}")
        with open(src) as f, open(SUMMARY_LOG, "a") as out:
            out.write(f"[{sub}]\n")
            out.write(f.read())
            out.write("\n")
    else:
        append(RUN_LOG, f"ERROR missing summary :: {sub}")
        append(ERRORS_LOG, f"ERROR missing summary :: {sub}")

with open(SUMMARY_LOG) as f:
    print(f.read(), end="")

print()
print("----------------------------------------")
print(f"Summary written to : {SUMMARY_LOG}")
print("----------------------------------------")
print("Step 2E - Summary")
print("----------------------------------------")

```
--- Script Step 2E End ---

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

--- Script Step 3 Start ---
```python

#!/usr/bin/env python3
# ------------------------------------------------------------
# Step 3 – Rebuild Audio Containers (All Formats)
# ------------------------------------------------------------
import os
import re
import shutil
import subprocess
import sys
import time

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step03"
os.makedirs(LOG_ROOT, exist_ok=True)


def which(name):
    return shutil.which(name) is not None


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


def keep_open_on_error(code):
    if code != 0 and sys.stdout.isatty():
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


# Software Preflight: fail loudly if a required tool is missing
for tool in ("ffmpeg", "ffprobe"):
    if not which(tool):
        print(f"ERROR: {tool} is not installed. Install it and re-run "
              "(see Requirements).", file=sys.stderr)
        keep_open_on_error(1)

# 1. Define Log Files (Five-File Standard + warnings)
RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OK_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAIL_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERROR_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
WARN_LOG = os.path.join(LOG_ROOT, f"{STEP}-warnings.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

# 2/3. CLEANUP + Initialize this step's logs
for path in (RUN_LOG, OK_LOG, FAIL_LOG, ERROR_LOG, WARN_LOG, SUMMARY_LOG):
    open(path, "w").close()


def decode_real(path):
    """Measure the real decodable duration of a file in seconds (None if unreadable).

    Header/format durations lie when a file is padded with junk (e.g. 0xFF fill),
    so a full decode is the only truthful measure of what a player will hear.
    """
    r = subprocess.run(
        ["ffmpeg", "-nostdin", "-v", "info", "-i", path, "-f", "null", "-"],
        stdout=subprocess.DEVNULL, stderr=subprocess.PIPE, text=True, check=False)
    times = re.findall(r"time=(\d{2}:\d{2}:\d{2}(?:\.\d{1,3})?)", r.stderr)
    if not times:
        return None
    h, m, s = times[-1].split(":")
    if "." in s:
        sec, frac = s.split(".")
        return int(h) * 3600 + int(m) * 60 + int(sec) + int(frac) / (10 ** len(frac))
    return int(h) * 3600 + int(m) * 60 + int(s)


# 4. File Discovery (all supported audio formats)
#    Optional override: pass $1 = a NUL-terminated file list to re-process only
#    those files (e.g. the failures from a previous run), instead of a full scan.
if len(sys.argv) > 1 and os.access(sys.argv[1], os.R_OK):
    with open(sys.argv[1], "rb") as f:
        files = [p.decode("utf-8", "surrogateescape")
                 for p in f.read().split(b"\0") if p]
else:
    EXTS = ["flac", "mp3", "m4a", "ogg", "opus", "wav", "aiff", "aif"]
    find_args = ["find", os.getcwd(), "-type", "f",
                 "!", "-ipath", "*/Ignore/*",
                 "!", "-iname", "*.prerepair",
                 "!", "-iname", "*.fixed.*",
                 "!", "-iname", "*.reencode.*", "("]
    for i, e in enumerate(EXTS):
        if i:
            find_args.append("-o")
        find_args += ["-iname", f"*.{e}"]
    find_args += [")", "-print0"]
    err_sink = open(ERROR_LOG, "a")
    try:
        r = subprocess.run(find_args, stdout=subprocess.PIPE,
                           stderr=err_sink, check=False)
    finally:
        err_sink.close()
    files = [p.decode("utf-8", "surrogateescape") for p in r.stdout.split(b"\0") if p]
    files.sort(key=lambda s: s.encode("utf-8", "surrogateescape"))  # LC_ALL=C sort -z

total = len(files)
last_album = ""

for i, path in enumerate(files, 1):
    artist = os.path.basename(os.path.dirname(os.path.dirname(path)))
    album = os.path.basename(os.path.dirname(path))
    track = os.path.basename(path)
    label = f"{artist}-{album}-{track}"

    # Insert a blank line on the terminal screen when moving to a new album
    # (Step 4 screen style: album-broken, readable per-file output)
    if last_album and album != last_album:
        print()
    last_album = album

    # Preserve original extension
    fixed = re.sub(r"\.[^.]*$", "", path) + ".fixed." + track.rsplit(".", 1)[1]

    r = subprocess.run(
        ["ffmpeg", "-nostdin", "-nostats", "-loglevel", "error",
         "-i", path, "-map_metadata", "0", "-c", "copy", fixed, "-y"],
        stdout=subprocess.DEVNULL, stderr=subprocess.PIPE, text=True,
        check=False)
    rc = r.returncode
    err_text = r.stderr

    # Non-fatal MJPEG "unable to decode APP fields" warnings come from corrupt
    # embedded JPEG artwork; they must not fail a valid container rebuild.
    warn = [l for l in err_text.splitlines()
            if re.search(r"unable to decode APP fields|"
                         "Invalid data found when processing input", l)]
    err_lines = [l for l in err_text.splitlines()
                 if not re.search(r"unable to decode APP fields|"
                                  "Invalid data found when processing input", l)
                 and l.strip()]

    # Duration sanity check: a silent partial copy must never replace the source.
    # 1) Cheap header check first; 2) if headers disagree by >2%, decode both sides
    # fully and compare their true decodable audio — sources padded with junk
    # (0xFF fill) lie about their real duration and must not be treated as lost.
    def probe_duration(p):
        r = subprocess.run(
            ["ffprobe", "-v", "error", "-show_entries", "format=duration",
             "-of", "default=noprint_wrappers=1:nokey=1", p],
            stdout=subprocess.PIPE, stderr=subprocess.DEVNULL, text=True,
            check=False)
        return r.stdout.strip()

    in_dur = probe_duration(path)
    out_dur = probe_duration(fixed)
    junk_note = ""
    dur_ok = True
    if in_dur and out_dur:
        a, b = float(in_dur), float(out_dur)
        if abs(a - b) > 0.02 * a:
            # Headers disagree by more than 2%: decode both and compare truthfully.
            src_real = decode_real(path)
            out_real = decode_real(fixed)
            if src_real is not None and out_real is not None:
                if abs(src_real - out_real) > 0.5:
                    dur_ok = False
                elif out_real - float(in_dur) > 45:
                    junk_note = (f"source claimed {in_dur}s but only {src_real}s "
                                 "decodable; junk tail removed")
            else:
                dur_ok = False

    if rc != 0 or err_lines or os.path.getsize(fixed) == 0 or not dur_ok:
        flat = re.sub(r"\s+", " ", err_text.replace("\0", " ")).strip()
        if os.path.exists(fixed):
            os.unlink(fixed)
        print(f"FAIL [{i}/{total}] {label}")
        append(RUN_LOG, f"FAIL [{i}/{total}] {label}")
        append(FAIL_LOG, f"FAIL [{i}/{total}] {label}")
        detail = flat or "no stderr output"
        if not dur_ok:
            append(ERROR_LOG,
                   f"[{i}/{total}] ERROR (exit {rc}, duration mismatch verified by "
                   f"full decode: in={in_dur} out={out_dur}): {label} :: {path} :: {detail}")
        else:
            append(ERROR_LOG,
                   f"[{i}/{total}] ERROR (exit {rc}): {label} :: {path} :: {detail}")
        continue

    # Back up the original once before overwriting (removed later by Step 7)
    prerepair = path + ".prerepair"
    if not os.path.exists(prerepair):
        try:
            shutil.copy2(path, prerepair)
        except OSError:
            os.unlink(fixed)
            print(f"FAIL [{i}/{total}] {label}")
            append(RUN_LOG, f"FAIL [{i}/{total}] {label}")
            append(ERROR_LOG, f"[{i}/{total}] ERROR (backup failed): {label} :: "
                              f"{path} :: could not create {prerepair}")
            continue
    try:
        os.replace(fixed, path)
        print(f"OK   [{i}/{total}] {label}")
        append(RUN_LOG, f"OK   [{i}/{total}] {label}")
        append(OK_LOG, f"OK   [{i}/{total}] {label}")
        if warn:
            append(WARN_LOG, f"[{i}/{total}] WARN: {label} :: embedded artwork "
                             f"warnings: {' '.join(warn)}")
        if junk_note:
            append(WARN_LOG, f"[{i}/{total}] WARN: {label} :: {junk_note}")
    except OSError as exc:
        if os.path.exists(fixed):
            os.unlink(fixed)
        print(f"FAIL [{i}/{total}] {label}")
        append(RUN_LOG, f"FAIL [{i}/{total}] {label}")
        append(ERROR_LOG, f"[{i}/{total}] ERROR (move failed): {label} :: {path} :: "
                          f"failed to move rebuilt file into place ({exc})")

# 5. Count Results
with open(RUN_LOG) as f:
    run_text = f.read()
ok_count = sum(1 for l in run_text.splitlines() if l.startswith("OK"))
fail_count = sum(1 for l in run_text.splitlines() if l.startswith("FAIL"))

# 6. Generate Summary
with open(SUMMARY_LOG, "w") as f:
    f.write("Step 3 Summary\n==============\n\n")
    f.write(f"Step       : {STEP}\n")
    f.write(f"Run Date   : {time.strftime('%c')}\n\n")
    f.write(f"Processed  : {total}\n")
    f.write(f"Passed     : {ok_count}\n")
    f.write(f"Failed     : {fail_count}\n")

# 7. Terminal Output
print()
print("----------------------------------------")
print(f"Processed: {total}  Passed: {ok_count}  Failed: {fail_count}")
print("----------------------------------------")
print("Step 3 – Rebuild Audio Containers")
print("----------------------------------------")

```
--- Script Step 3 End ---

\ ---------------------------------------------------------------------------------------

-- Step 3: View Log Files

--- Bash Script Cat 3 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 3 – View Log Results
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
echo "Step 3 – View Log Results"
echo "----------------------------------------"

```
--- Bash Script Cat 3 End ---

---

08. Step 4 – Repeat Step 1 Integrity Test

---

-- Purpose

This step repeats the integrity test performed in Step 1, but now `after` Step 3's container rebuilding. It verifies all audio files to confirm that the container rebuild was successful and that files remain structurally valid.

Running this verification after container work provides a direct comparison against the original Step 1 baseline, identifying files that were repaired and files that continue to report problems.

No files are modified during this step. This is a verification step only.

\ ---------------------------------------------------------------------------------------

--- Script Step 4 Start ---
```python

#!/usr/bin/env python3
# ============================================================
# Step 4 – Post-Rebuild Integrity Verification
# ============================================================
import os
import re
import shutil
import subprocess
import sys
import time

LOG_ROOT = os.path.join(os.path.expanduser("~"), ".logs", "linux-audio-moode-cleanup-guide")
STEP = "step04"

os.makedirs(LOG_ROOT, exist_ok=True)


def which(name):
    return shutil.which(name) is not None


def append(path, msg):
    with open(path, "a") as f:
        f.write(msg + "\n")


def keep_open_on_error(code):
    if code != 0 and sys.stdout.isatty():
        print(f"\nScript exited with status {code}. "
              "Press ENTER to close this terminal.")
        try:
            input()
        except EOFError:
            pass
    sys.exit(code)


# Software Preflight: fail loudly if a required tool is missing
for tool in ("flac", "ffmpeg"):
    if not which(tool):
        print(f"ERROR: {tool} is not installed. Install it and re-run "
              "(see Requirements).", file=sys.stderr)
        keep_open_on_error(1)

# 1. Define Log Files (Five-File Standard)
RUN_LOG = os.path.join(LOG_ROOT, f"{STEP}-run.log")
OKS_LOG = os.path.join(LOG_ROOT, f"{STEP}-oks.log")
FAILS_LOG = os.path.join(LOG_ROOT, f"{STEP}-fails.log")
ERRORS_LOG = os.path.join(LOG_ROOT, f"{STEP}-errors.log")
SUMMARY_LOG = os.path.join(LOG_ROOT, f"{STEP}-summary.log")

# 2/3. CLEANUP + Initialize this step's logs from any previous run
for path in (RUN_LOG, OKS_LOG, FAILS_LOG, ERRORS_LOG, SUMMARY_LOG):
    open(path, "w").close()

# 4. File Discovery
EXTS = ["flac", "mp3", "m4a", "ogg", "opus", "wav", "aiff", "aif", "mp4",
        "ape", "wv", "spx"]
EXCLUDES = ["!", "-ipath", "*/Ignore/*",
            "!", "-iname", "*.prerepair*",
            "!", "-iname", "*.fixed.*",
            "!", "-iname", "*.reencode.*"]

args = ["find", os.getcwd(), "-type", "f", *EXCLUDES, "("]
for i, e in enumerate(EXTS):
    if i:
        args.append("-o")
    args += ["-iname", f"*.{e}"]
args += [")", "-print0"]
err_sink = open(ERRORS_LOG, "a")
try:
    r = subprocess.run(args, stdout=subprocess.PIPE, stderr=err_sink, check=False)
finally:
    err_sink.close()
files = [p.decode("utf-8", "surrogateescape") for p in r.stdout.split(b"\0") if p]
files.sort(key=lambda s: s.encode("utf-8", "surrogateescape"))  # sort -z

total = len(files)
last_dir = ""

for i, path in enumerate(files, 1):
    label = os.path.relpath(path, os.getcwd())
    current_dir = os.path.dirname(label) or "."

    # Insert a blank line on terminal screen when moving to a new folder/album
    if last_dir and current_dir != last_dir:
        print()
    last_dir = current_dir

    if path.lower().endswith(".flac"):
        res = subprocess.run(["flac", "-s", "-t", path],
                             stdout=subprocess.DEVNULL,
                             stderr=subprocess.PIPE, text=True, check=False)
    else:
        res = subprocess.run(
            ["ffmpeg", "-nostdin", "-v", "error", "-i", path, "-f", "null", "-"],
            stdout=subprocess.DEVNULL, stderr=subprocess.PIPE, text=True,
            check=False)
    rc = res.returncode

    if rc == 0:
        out_msg = f"OK   [{i}/{total}] {label}"
        print(out_msg)
        append(RUN_LOG, out_msg)
        append(OKS_LOG, out_msg)
    else:
        flat = re.sub(r"\s+", " ",
                      res.stderr.replace("\r", " ").replace("\n", " ")).strip()
        out_msg = f"FAIL [{i}/{total}] {label}"
        print(out_msg)
        append(RUN_LOG, out_msg)
        append(FAILS_LOG, out_msg)
        append(ERRORS_LOG,
               f"[{i}/{total}] ERROR (exit {rc}): {label} :: {path} :: "
               f"{flat or 'no stderr output'}")

# 5. Count Results
with open(RUN_LOG) as f:
    run_text = f.read()
ok_count = sum(1 for l in run_text.splitlines() if l.startswith("OK"))
fail_count = sum(1 for l in run_text.splitlines() if l.startswith("FAIL"))

# 6. Generate Summary Log
with open(SUMMARY_LOG, "w") as f:
    f.write("Step 4 Summary\n==============\n\n")
    f.write(f"Step       : {STEP}\n")
    f.write(f"Run Date   : {time.strftime('%c')}\n\n")
    f.write(f"Processed  : {total}\n")
    f.write(f"Passed     : {ok_count}\n")
    f.write(f"Failed     : {fail_count}\n")

# 7. Terminal Output
print()
if os.path.getsize(ERRORS_LOG) > 0:
    print("----------------------------------------")
    print("Error Summary")
    print("----------------------------------------")
    lost_sync, eos = {}, {}
    with open(ERRORS_LOG, encoding="utf-8", errors="replace") as f:
        for line in f:
            if "LOST_SYNC" in line:
                idx = line.find(" :: ")
                if idx > 0:
                    temp = line[:idx]
                    pos = temp.find("): ") + 3
                    lost_sync[temp[pos:]] = True
            elif "END_OF_STREAM" in line:
                idx = line.find(" :: ")
                if idx > 0:
                    temp = line[:idx]
                    pos = temp.find("): ") + 3
                    eos[temp[pos:]] = True
    if lost_sync:
        print("LOST_SYNC")
        print("----------")
        for p in sorted(lost_sync):
            print(p)
    if eos:
        if lost_sync:
            print()
        print("END_OF_STREAM")
        print("----------")
        for p in sorted(eos):
            print(p)

print()
print("----------------------------------------")
print(f"Processed: {total}  Passed: {ok_count}  Failed: {fail_count}")
print("----------------------------------------")
print("Step 4 – Post-Rebuild Integrity Verification")
print("----------------------------------------")

```
--- Script Step 4 End ---

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
mapfile -d '' dirs < <(find "$PWD" -type d \
    ! -ipath '*/Ignore/*' ! -ipath '*/Ignore' ! -iname 'Ignore' \
    -print0 | LC_ALL=C sort -f -z)

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
last_artist=""

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

        # Insert a blank line on the terminal screen when moving to a new
        # ARTIST (user preference: ReplayGain output breaks per artist)
        if [[ -n "$last_artist" && "$artist" != "$last_artist" ]]; then
            echo ""
        fi
        last_artist="$artist"

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

12. Step 8 – Deep Repair via Decode/Re-encode (Last Resort)

-- Purpose

This step is a rescue option for FLAC files that continue to fail integrity testing after the earlier cleanup steps (Steps 1-7, including Step 1B) and optional procedures (15a-15c) have been completed.

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

**Caution:** Since v28, the `.prerepair` backup is deleted automatically as soon as the rebuilt file passes its post-reencode decode test and is swapped in, so no residuals are left behind. A failed repair never touches the original.

\---------------------------------------------------------------------------------------

--- Bash Script Step 8 Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 8 – Deep Repair via Decode/Re-encode (Last Resort)
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step08-errors.log"
: > "$LOG_ROOT/step08-review.log"

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
last_dir=""

# Progress line: [done/total] % complete, elapsed and ETA (terminal only)
start_ts=$(date +%s)
progress() {
    local done_n=$1 total_n=$2 now el pct eta
    [ "$total_n" -gt 0 ] || return 0
    [ -t 2 ] || return 0
    now=$(date +%s)
    el=$((now - start_ts))
    pct=$((done_n * 100 / total_n))
    eta=0
    [ "$done_n" -gt 0 ] && eta=$((el * (total_n - done_n) / done_n))
    printf '\r\033[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   ' \
        "$done_n" "$total_n" "$pct" \
        $((el/3600)) $(((el/60)%60)) $((el%60)) \
        $((eta/3600)) $(((eta/60)%60)) $((eta%60)) >&2
}

for f in "${files[@]}"; do
    i=$((i+1))
    progress "$i" "$total"
    # Album header on folder change: clear the counter line, print the
    # album path, let the counter resume on the next line (stderr only)
    hdr="$(dirname "${f#"$PWD"/}")"
    if [[ -n "$last_dir" && "$hdr" != "$last_dir" ]]; then
        printf '\r\033[K── %s ──\n' "$hdr" >&2
    fi
    last_dir="$hdr"

    artist=$(basename "$(dirname "$(dirname "$f")")")
    album=$(basename "$(dirname "$f")")
    track=$(basename "$f" .flac)

    label="$artist-$album-$track"

    testerr=$(flac -s -t "$f" 2>&1 >/dev/null)
    testrc=$?

    if [ $testrc -eq 0 ]; then
        echo "OK [$i/$total] $label" >> "$LOG_ROOT/step08-run.log"
        continue
    fi

    tags=$(mktemp "$LOG_ROOT/step08-tags.XXXXXX")

    tagerr=$(metaflac --export-tags-to="$tags" "$f" 2>&1 >/dev/null)
    tagrc=$?

    if [ $tagrc -ne 0 ]; then
        flat=$(echo "$tagerr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label" | tee -a "$LOG_ROOT/step08-run.log"
        echo "[$i/$total] ERROR (exit $tagrc, tag export): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step08-errors.log"
        rm -f "$tags"
        continue
    fi

    # Save the first embedded picture so it can be restored after the re-encode
    pic=$(mktemp "$LOG_ROOT/step08-pic.XXXXXX")
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
        echo "FAIL [$i/$total] $label" | tee -a "$LOG_ROOT/step08-run.log"
        echo "[$i/$total] ERROR (exit $rerc, reencode): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step08-errors.log"
        rm -f "$tags" "$pic" "${f}.reencode"
        continue
    fi

    posterr=$(flac -s -t "${f}.reencode" 2>&1 >/dev/null)
    postrc=$?

    if [ $postrc -ne 0 ]; then
        flat=$(echo "$posterr" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label" | tee -a "$LOG_ROOT/step08-run.log"
        echo "[$i/$total] ERROR (exit $postrc, post-reencode test): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step08-errors.log"
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
        echo "FAIL [$i/$total] $label" | tee -a "$LOG_ROOT/step08-run.log"
        echo "[$i/$total] ERROR (exit $imprc, tag/picture reimport): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step08-errors.log"
        rm -f "$tags" "$pic" "${f}.reencode"
        continue
    fi

    # Preserve the original once; suffix is non-FLAC so moOde never indexes it
    if [ ! -e "${f}.prerepair" ]; then
        if ! cp "$f" "${f}.prerepair" >/dev/null 2>&1; then
            echo "FAIL [$i/$total] $label" | tee -a "$LOG_ROOT/step08-run.log"
            echo "[$i/$total] ERROR (backup failed): $label :: $f :: could not create ${f}.prerepair" \
                >> "$LOG_ROOT/step08-errors.log"
            rm -f "$tags" "$pic" "${f}.reencode"
            continue
        fi
    fi

    if ! mv -f "${f}.reencode" "$f" >/dev/null 2>&1; then
        echo "FAIL [$i/$total] $label" | tee -a "$LOG_ROOT/step08-run.log"
        echo "[$i/$total] ERROR (mv failed): $label :: $f :: could not move rebuilt file into place" \
            >> "$LOG_ROOT/step08-errors.log"
        rm -f "$tags" "$pic" "${f}.reencode"
        continue
    fi
    rm -f "$tags" "$pic"

    # The rebuild already passed a full decode test before the swap, so the
    # backup has served its purpose — no residuals left behind.
    rm -f "${f}.prerepair"

    if [ -n "$reerr" ]; then
        flat=$(echo "$reerr" | tr '\n' ' ' | tr -s ' ')
        echo "FIXED-REVIEW [$i/$total] $label" | tee -a "$LOG_ROOT/step08-run.log"
        echo "[$i/$total] REVIEW $label :: $f :: ffmpeg reported during decode: $flat" \
            >> "$LOG_ROOT/step08-review.log"
    else
        echo "FIXED-CLEAN [$i/$total] $label" | tee -a "$LOG_ROOT/step08-run.log"
    fi

done
printf '\n' >&2
echo
echo "----------------------------------------"
echo "Step 8 – Deep Repair via Decode/Re-encode (Last Resort)"
echo "----------------------------------------"

```
--- Bash Script Step 8 End ---

\---------------------------------------------------------------------------------------

-- Separate Results

--- Bash Script Step 8 Results Start ---
```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"

grep '^OK' "$LOG_ROOT/step08-run.log" \
    > "$LOG_ROOT/step08-oks.log"

grep '^FIXED-CLEAN' "$LOG_ROOT/step08-run.log" \
    > "$LOG_ROOT/step08-fixed-clean.log"

grep '^FIXED-REVIEW' "$LOG_ROOT/step08-run.log" \
    > "$LOG_ROOT/step08-fixed-review.log"

grep '^FAIL' "$LOG_ROOT/step08-run.log" \
    > "$LOG_ROOT/step08-fails.log"

echo "Step 8 OKs: $(wc -l < "$LOG_ROOT/step08-oks.log")  FIXED-CLEAN: $(wc -l < "$LOG_ROOT/step08-fixed-clean.log")  FIXED-REVIEW: $(wc -l < "$LOG_ROOT/step08-fixed-review.log")  FAILs: $(wc -l < "$LOG_ROOT/step08-fails.log")"

```
--- Bash Script Step 8 Results End ---

\---------------------------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat 8 Start ---
```bash

cat "$LOG_ROOT/step08-errors.log"
cat "$LOG_ROOT/step08-review.log"
cat "$LOG_ROOT/step08-run.log"
cat "$LOG_ROOT/step08-oks.log"
cat "$LOG_ROOT/step08-fixed-clean.log"
cat "$LOG_ROOT/step08-fixed-review.log"
cat "$LOG_ROOT/step08-fails.log"

```
--- Bash Script Cat 8 End ---

\---------------------------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step08-run.log — Complete processing results for all FLAC files.
* step08-oks.log — Files that already passed `flac -t` (no repair needed).
* step08-fixed-clean.log — Files successfully repaired with no warnings during re-encode.
* step08-fixed-review.log — Files successfully repaired but ffmpeg reported warnings; these warrant manual playback testing.
* step08-fails.log — Files that could not be repaired and remain unchanged.
* step08-errors.log — Detailed error output from failed repair attempts.
* step08-review.log — Decode-time warnings from ffmpeg for FIXED-REVIEW files.

Since v28, successfully repaired files leave no `.prerepair` backups behind — the backup is deleted once the rebuild passes its decode test and is swapped in (a backup can only remain if the final move into place failed).

After running this procedure, verify the results:
1. Listen to a few tracks from the FIXED-REVIEW list to ensure playback quality.
2. Run Step 1 (Initial Integrity Test) on the fixed files to confirm they now pass.
3. If FIXED-REVIEW files sound correct, the repair is complete. No `.prerepair` backups remain to clean up.

\---------------------------------------------------------------------------------------

13. Step 9 – Verify Tags Against Filenames (Failsafe)

-- Purpose

This step is a failsafe verification that confirms every tag was updated correctly after the cleanup pipeline (Steps 1–8, including Step 1B) has run. It catches any file whose embedded metadata still disagrees with the folder and filename convention — for example, a title that was not written, a track number placed on the wrong file, or an artist/album tag that was left stale.

It is read-only: it never modifies any file. It only reports. Any findings are fixed separately (see "Fixing Findings" below), after which you re-run Step 16 to regenerate the checksums that the tag edits invalidate.

-- What It Does

This step:

* Runs a pre-flight format check: each FLAC/MP3/M4A file's container is confirmed against its extension, and any mismatch is reported to `step09-format-errors.log` before verification begins.
* Scans the given music root recursively for FLAC, MP3, and M4A files; folders named `Ignore` are skipped at any depth.
* Derives the expected Artist (parent folder), Album Year / Album name (album folder name, `YYYY Album`), and Track number / Title (filename, `NN Title`) from the naming convention.
* Reads the actual embedded tags from each file with ffprobe.
* Compares them with a lenient normalizer: tags are lowercased and reduced to their alphanumeric core, so smart quotes, punctuation-only differences, and spacing variants are treated as "close enough" and are NOT flagged. Only genuinely different words or characters — or tags that are entirely missing — are reported.
* Flags any file where a tag is missing or disagrees with the filename/folder on a real word level, writing one line per finding.
* Files whose name has no `NN Title` prefix are reported as UNPARSEABLE rather than guessed (Step 1B enforces the naming convention, so this should be empty after the pipeline).
* Writes a `SUMMARY:` tally of files checked vs. mismatches found.

-- How to Run

Run from the terminal against the library root (or an artist folder for a targeted check):

--- Bash Script Step 9 Start ---
```bash

#!/usr/bin/env bash

trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# Step 9 – Verify Tags Against Filenames (Failsafe)
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"

TARGET="${1:?Usage: $0 /path/to/music/root}"

RUN_LOG="$LOG_ROOT/step09-run.log"
MISMATCH_LOG="$LOG_ROOT/step09-mismatches.log"

: > "$RUN_LOG"
: > "$MISMATCH_LOG"

# Progress line: [done/total] % complete, elapsed and ETA (terminal only)
start_ts=$(date +%s)
progress() {
    local done_n=$1 total_n=$2 now el pct eta
    [ "$total_n" -gt 0 ] || return 0
    [ -t 2 ] || return 0
    now=$(date +%s)
    el=$((now - start_ts))
    pct=$((done_n * 100 / total_n))
    eta=0
    [ "$done_n" -gt 0 ] && eta=$((el * (total_n - done_n) / done_n))
    printf '\r\033[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   ' \
        "$done_n" "$total_n" "$pct" \
        $((el/3600)) $(((el/60)%60)) $((el%60)) \
        $((eta/3600)) $(((eta/60)%60)) $((eta%60)) >&2
}

# "Close" tags are fine, so the comparator folds cosmetic differences:
# lowercase, then keep ONLY alphanumeric characters. Smart quotes
# ("Ain't" vs "Ain't"), punctuation-only drift ("Name?" vs "Name"),
# and spacing variants all compare equal. A genuinely different word
# set still differs and flags.
norm() { tr '[:upper:]' '[:lower:]' | tr -cd '[:alnum:]'; }

echo "========== Step 9: Verify Tags Against Filenames ==========" | tee -a "$RUN_LOG"
echo "Root: $TARGET" | tee -a "$RUN_LOG"
echo "Started: $(date)" | tee -a "$RUN_LOG"
echo

# --- Pre-flight: format check --------------------------------------
# Confirm every audio file's container actually matches its extension
# before trusting any tag comparison. Wrong-container/mislabeled files
# are reported to step09-format-errors.log but still verified below.
FORMAT_LOG="$LOG_ROOT/step09-format-errors.log"
: > "$FORMAT_LOG"
mapfile -d '' fmt_files < <(find "$TARGET" -type f ! -ipath '*/Ignore/*' \( -iname "*.flac" -o -iname "*.mp3" -o -iname "*.m4a" \) -print0 | sort -z)
fmt_total=${#fmt_files[@]}
fmt_done=0
start_ts=$(date +%s)
while IFS= read -r -d '' filepath; do
    fmt_done=$((fmt_done+1))
    progress "$fmt_done" "$fmt_total"
    ext="${filepath##*.}"
    ext="${ext,,}"
    fmt=$(ffprobe -v error -show_entries format=format_name -of default=nw=1:nk=1 "$filepath" 2>/dev/null)
    case "$ext" in
        flac) want="flac" ;;
        mp3)  want="mp3" ;;
        m4a)  want="mov mp4 m4a" ;;
        *)    want="" ;;
    esac
    if [ -n "$want" ] && [ -n "$fmt" ]; then
        fmt_ok=0
        IFS=',' read -ra parts <<< "$fmt"
        for p in "${parts[@]}"; do
            case " $want " in *" $p "*) fmt_ok=1 ;; esac
        done
        if [ "$fmt_ok" -eq 0 ]; then
            echo "FORMAT|${filepath#"$TARGET"/}|extension=.$ext but ffprobe reports: $fmt" >> "$FORMAT_LOG"
        fi
    fi
done < <(printf '%s\0' "${fmt_files[@]}")
printf '\n' >&2
if [ -s "$FORMAT_LOG" ]; then
    echo "Pre-flight found possible format/extension mismatches in $(wc -l < "$FORMAT_LOG") file(s) — see $FORMAT_LOG" | tee -a "$RUN_LOG"
else
    echo "Pre-flight: all containers match their extensions." | tee -a "$RUN_LOG"
fi
echo

checked=0
mismatched=0
last_dir=""
start_ts=$(date +%s)

while IFS= read -r -d '' filepath; do
    checked=$((checked+1))
    progress "$checked" "$fmt_total"
    # Album header on folder change (stderr only)
    hdr="$(dirname "${filepath#"$TARGET"/}")"; [ "$filepath" != "${filepath#"$TARGET"/}" ] || hdr="$(dirname "$filepath")"
    hdr="${hdr#./}"
    if [[ -n "$last_dir" && "$hdr" != "$last_dir" ]]; then
        printf '\r\033[K── %s ──\n' "$hdr" >&2
    fi
    last_dir="$hdr"
    filename=$(basename "$filepath")
    rel="${filepath#"$TARGET"/}"
    [ -n "$rel" ] || rel="$filepath"
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

    if [[ "$name_no_ext" =~ ^([0-9]+)[[:space:]]+(-[[:space:]]+)?(.+)$ ]]; then
        want_track_str="${BASH_REMATCH[1]}"
        want_track=$(( 10#${want_track_str} ))
        want_title="${BASH_REMATCH[3]}"
    else
        echo "UNPARSEABLE|$rel|filename has no \"NN Title\"" >> "$MISMATCH_LOG"
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
done < <(printf '%s\0' "${fmt_files[@]}")
printf '\n' >&2

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
echo "Names' Nemo action, then re-run Step 16 to regenerate checksums."

```
--- Bash Script Step 9 End ---

Run it with your music root as the argument, e.g.:

--- Bash Script Start ---
```bash

bash /path/to/where/you/saved/this /media/youruser/music

```
--- Bash Script End ---

-- Expected Results

A successful run produces:

* step09-run.log — Full scan record, including the final `SUMMARY:` tally.
* step09-mismatches.log — One line per file needing attention, in `UNPARSEABLE` or `MISMATCH|path|reason` form. Empty when the library is clean.

A `SUMMARY:` line of `N file(s) checked, 0 had tag/filename mismatches.` means the failsafe passed.

-- Fixing Findings

Fixing findings:

* Missing tags and tag drift that is a genuine error should be fixed with the **"Write Tags from Folder/File Names"** Nemo action (from the linux-audio-nemo-actions guide), or individually with EasyTAG. Both are lossless — audio is never re-encoded.
* **Accepted variants are not errors.** The naming convention uses sort-friendly folder names while tags carry official display names, and Step 9 flags these by design. Verified on 2026-08-30 for this library (793 files, decision recorded in `Scripts/moode-run/FINALIZE/03-step9-accepted-variants-decision.md`):
  - Folder "String Cheese Incident" vs tag "The String Cheese Incident"
  - Folder "Bob Marley" vs tag "Bob Marley & The Wailers"
  - Folder "Jimi Hendrix" vs tag "The Jimi Hendrix Experience"
  - Folder "Saucerful Of Secrets" vs tag "A Saucerful Of Secrets" (leading-article sorting)
  - "Various Artists" folders where each track's ARTIST is the real performer
  Do NOT "fix" these by rewriting tags to match folders — that degrades what moOde displays.
* After any tag fix, the checksums for the affected album(s)/artist(s) are stale. Re-run Step 16 (Generate Checksums) — or the "Regenerate ALBUM/ARTIST Checksum" Nemo action — to record the corrected state.
* Optionally re-run Step 9 afterward. A clean Step 9 result now includes accepted variants; zero is no longer the target.

\---------------------------------------------------------------------------------------

14. Step 10 – Final Integrity Test

---

-- Purpose

This step performs the final integrity verification of the library after all standard cleanup operations have been completed. It confirms that the complete workflow — metadata cleanup, container rebuilding, ReplayGain restoration, and loose file removal — has resulted in a stable and valid library.

This final verification provides the archival baseline for the repaired library.

No files are modified during this step.

\ ---------------------------------------------------------------------------------------

--- Bash Script Step 10 Start ---

```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ============================================================
# Step 10 – Final Integrity Test
# ============================================================

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step10"

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

# Progress line: [done/total] % complete, elapsed and ETA (terminal only)
start_ts=$(date +%s)
progress() {
    local done_n=$1 total_n=$2 now el pct eta
    [ "$total_n" -gt 0 ] || return 0
    [ -t 2 ] || return 0
    now=$(date +%s)
    el=$((now - start_ts))
    pct=$((done_n * 100 / total_n))
    eta=0
    [ "$done_n" -gt 0 ] && eta=$((el * (total_n - done_n) / done_n))
    printf '\r\033[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   ' \
        "$done_n" "$total_n" "$pct" \
        $((el/3600)) $(((el/60)%60)) $((el%60)) \
        $((eta/3600)) $(((eta/60)%60)) $((eta%60)) >&2
}

for f in "${files[@]}"; do

    ((i++))
    progress "$i" "$total"
    label="${f#"$PWD"/}"
    current_dir="$(dirname "$label")"

    # Album header on album change: clear the counter line, print the
    # album path, let the counter resume on the next line (stderr only)
    if [[ -n "$last_dir" && "$current_dir" != "$last_dir" ]]; then
        printf '\r\033[K── %s ──\n' "$current_dir" >&2
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
        echo "$out_msg" >> "$RUN_LOG"
        echo "$out_msg" >> "$OKS_LOG"
    else
        flat=$(printf '%s\n' "$err" | tr '\r\n' ' ' | tr -s ' ')
        out_msg="FAIL [$i/$total] $label"
        echo ""
        echo "$out_msg"
        echo "$out_msg" >> "$RUN_LOG"
        echo "$out_msg" >> "$FAILS_LOG"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
    fi

done
printf '\n' >&2

# 5. Count Results
ok_count=$(grep -a "^OK" "$RUN_LOG" 2>/dev/null | wc -l)
fail_count=$(grep -a "^FAIL" "$RUN_LOG" 2>/dev/null | wc -l)

# 6. Generate Summary Log
{
echo "Step 10 Summary"
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
echo "Step 10 – Final Integrity Test"
echo "----------------------------------------"

```
--- Bash Script Step 10 End ---

\ ---------------------------------------------------------------------------------------


-- Review Results

--- Bash Script Cat 10 Start ---

```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step10"

cat "$LOG_ROOT/${STEP}-summary.log"
cat "$LOG_ROOT/${STEP}-errors.log"
cat "$LOG_ROOT/${STEP}-run.log"
cat "$LOG_ROOT/${STEP}-oks.log"
cat "$LOG_ROOT/${STEP}-fails.log"

```
--- Bash Script Cat 10 End ---

---

15. Optional Procedures

---

The following procedures are not required for a standard library cleanup but may be necessary for resolving stubborn errors, standardizing visual presentation, or preparing the library for long-term preservation.

Note: Run these optional procedures before Step 7 if you want Step 7 to clean up their working files (Step 7 removes `*.prerepair`, `*.reencode`, `*.fixed.*`, `*.tmp`, `*.temp`, and `*~`, plus legacy `*.prerepair.flac`/`*.reencode.flac` names), or simply re-run Step 7 after them.

\ ---------------------------------------------------------------------------------------

## 15a. Strip Problematic Metadata

This procedure is a "nuclear option" for files that continue to fail integrity testing even after the standard deduplication and container rebuilding steps.

Sometimes, FLAC files contain deeply corrupted non-text metadata blocks—such as broken seek tables or damaged cue sheets—that prevent standard tools from reading the file correctly. Since v28 this step is surgical rather than destructive: FLAC metadata blocks are individually editable in place, so only the problem blocks are removed.

-- What It Does

This step:

* Scans the selected location for FLAC files.

* Removes only SEEKTABLE blocks (stale after any tag edit; moOde never uses them) and CUESHEET blocks (which confuse some players) from affected files.

* Leaves VORBIS_COMMENT (all text tags, including ReplayGain), PICTURE, and PADDING blocks untouched — tags and artwork are never rewritten, and excess/missing padding is only logged for review.

* Leaves conforming files completely unmodified (no rewrite, no mtime change) and records them as SAME.

* Leaves the underlying audio stream completely untouched.

* Flags ID3-prefixed FLACs and APPLICATION metadata blocks for review instead of modifying them.

Because artwork and tags are preserved, no follow-up artwork step is needed after 15a itself; run 15c only if you want to standardize artwork independently.

\ ---------------------------------------------------------------------------------------

--- Bash Script for 15a Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# 15a. Strip Problematic Metadata (SURGICAL — FLAC only)
#   FLAC metadata blocks are individually editable in place.
#   We only touch files that actually contain problem blocks:
#     - SEEKTABLE  (stale after any tag edit; moOde never uses it)
#     - CUESHEET   (confuses some players)
#   Padding, VORBIS_COMMENT (tags incl. ReplayGain) and PICTURE are NEVER
#   touched — excess/missing padding is flagged to the review log only.
#   Conforming files are left completely unmodified (no rewrite, no mtime).
#   ID3-prefixed FLACs and APPLICATION blocks are flagged for review.
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step15a-errors.log"
: > "$LOG_ROOT/step15a-review.log"
: > "$LOG_ROOT/step15a-run.log"

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
last_dir=""
changed=0
skipped=0
failed=0

# Progress line: [done/total] % complete, elapsed and ETA (terminal only)
start_ts=$(date +%s)
progress() {
    local done_n=$1 total_n=$2 now el pct eta
    [ "$total_n" -gt 0 ] || return 0
    [ -t 2 ] || return 0
    now=$(date +%s)
    el=$((now - start_ts))
    pct=$((done_n * 100 / total_n))
    eta=0
    [ "$done_n" -gt 0 ] && eta=$((el * (total_n - done_n) / done_n))
    printf '\r\033[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   ' \
        "$done_n" "$total_n" "$pct" \
        $((el/3600)) $(((el/60)%60)) $((el%60)) \
        $((eta/3600)) $(((eta/60)%60)) $((eta%60)) >&2
}

for f in "${files[@]}"; do
    i=$((i+1))
    progress "$i" "$total"
    # Album header on folder change: clear the counter line, print the
    # album path, let the counter resume on the next line (stderr only)
    hdr="$(dirname "${f#"$PWD"/}")"
    if [[ -n "$last_dir" && "$hdr" != "$last_dir" ]]; then
        printf '\r\033[K── %s ──\n' "$hdr" >&2
    fi
    last_dir="$hdr"

    artist=$(basename "$(dirname "$(dirname "$f")")")
    album=$(basename "$(dirname "$f")")
    track=$(basename "$f" .flac)
    label="$artist-$album-$track"

    # ID3 junk prefix check (moOde/flac tooling dislike ID3 on FLAC)
    if [ "$(head -c 3 "$f" 2>/dev/null)" = "ID3" ]; then
        echo "REVIEW [$i/$total] $label :: ID3-prefixed FLAC" | tee -a "$LOG_ROOT/step15a-run.log"
        echo "[$i/$total] REVIEW: $label :: $f :: ID3v2 prefix detected — strip manually if moOde misbehaves" \
            >> "$LOG_ROOT/step15a-review.log"
    fi

    listing=$(metaflac --list "$f" 2>/dev/null)
    if [ -z "$listing" ]; then
        echo "FAIL [$i/$total] $label" | tee -a "$LOG_ROOT/step15a-run.log"
        echo "[$i/$total] ERROR: $label :: $f :: metaflac could not read block list" \
            >> "$LOG_ROOT/step15a-errors.log"
        failed=$((failed+1))
        continue
    fi

    has_seek=$(grep -qc 'type: [0-9]* (SEEKTABLE)' <<< "$listing" && echo 1 || echo 0)
    has_cuesheet=$(grep -qc 'type: [0-9]* (CUESHEET)' <<< "$listing" && echo 1 || echo 0)
    has_application=$(grep -qc 'type: [0-9]* (APPLICATION)' <<< "$listing" && echo 1 || echo 0)
    pad_total=$(awk '/\(PADDING\)/{p=1; next} p && /length:/{gsub(/[^0-9]/,"",$2); s+=$2; p=0} END{print s+0}' <<< "$listing")

    [ "$has_application" -eq 1 ] && \
        echo "[$i/$total] REVIEW: $label :: $f :: APPLICATION metadata block present (left in place)" \
            >> "$LOG_ROOT/step15a-review.log"

    # Padding is informational only — never rewritten. Excess or missing
    # padding is harmless; it just costs a little disk space or future edit
    # speed. Only flag padding that is PRESENT but non-standard (not the
    # 8192 bytes metaflac writes when editing); a file with no PADDING block
    # at all is normal and stays silent.
    if [ "$pad_total" -ne 0 ] && [ "$pad_total" -ne 8192 ]; then
        echo "[$i/$total] REVIEW: $label :: $f :: padding is $pad_total bytes (left in place)" \
            >> "$LOG_ROOT/step15a-review.log"
    fi

    cmds=()
    [ "$has_seek" -eq 1 ]      && cmds+=(SEEKTABLE)
    [ "$has_cuesheet" -eq 1 ]  && cmds+=(CUESHEET)

    if [ ${#cmds[@]} -eq 0 ]; then
        skipped=$((skipped+1))
        echo "SAME [$i/$total] $label" >> "$LOG_ROOT/step15a-run.log"
        continue
    fi

    detail=""
    [ "$has_seek" -eq 1 ] && detail="seektable"
    [ "$has_cuesheet" -eq 1 ] && detail="${detail:+$detail+}cuesheet"

    # metaflac forbids mixing major (--remove) and shorthand (--add-padding)
    # operations in one call, so removals run as their own single call.
    err=""
    rc=0
    err=$(metaflac --remove "${cmds[@]/#/--block-type=}" "$f" 2>&1 >/dev/null) || rc=$?
    if [ $rc -ne 0 ] || [ -z "$(metaflac --list "$f" 2>/dev/null)" ]; then
        flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
        echo "FAIL [$i/$total] $label" | tee -a "$LOG_ROOT/step15a-run.log"
        echo "[$i/$total] ERROR (exit $rc): $label :: $f :: ${flat:-no stderr output}" \
            >> "$LOG_ROOT/step15a-errors.log"
        failed=$((failed+1))
        continue
    fi

    changed=$((changed+1))
    echo "FIXED [$i/$total] $label :: $detail" | tee -a "$LOG_ROOT/step15a-run.log"

done

printf '\n' >&2
echo
echo "----------------------------------------"
echo "15a. Strip Problematic Metadata (surgical)"
echo "Total: $total   Fixed: $changed   Already clean: $skipped   Failed: $failed"
echo "Review flags: $( [ -s "$LOG_ROOT/step15a-review.log" ] && wc -l < "$LOG_ROOT/step15a-review.log" || echo 0 )  (see $LOG_ROOT/step15a-review.log)"
echo "----------------------------------------"
```
--- Bash Script for 15a End ---

\---------------------------------------------------------------------------------------

-- Separate Results

After the stripping process completes, separate successful and failed results:

--- Bash Script Results for 15a Start ---
```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"

grep '^FIXED' "$LOG_ROOT/step15a-run.log" \
    > "$LOG_ROOT/step15a-fixed.log"

grep '^FAIL' "$LOG_ROOT/step15a-run.log" \
    > "$LOG_ROOT/step15a-fails.log"

echo "Step 15a FIXED: $(wc -l < "$LOG_ROOT/step15a-fixed.log")  FAILs: $(wc -l < "$LOG_ROOT/step15a-fails.log")"

```
--- Bash Script Results for 15a End ---

\---------------------------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 15a Start ---
```bash

cat "$LOG_ROOT/step15a-errors.log"

cat "$LOG_ROOT/step15a-run.log"

cat "$LOG_ROOT/step15a-fixed.log"

cat "$LOG_ROOT/step15a-fails.log"

```
--- Bash Script Cat for 15a End ---

\---------------------------------------------------------------------------------------

-- Expected Results

A successful run produces:

    step15a-run.log — Complete processing results.

    step15a-fixed.log — Files whose SEEKTABLE and/or CUESHEET blocks were removed.

    step15a-fails.log — Files that could not be processed.

    step15a-errors.log — Detailed error output.

After running this on stubborn files, you should run the integrity test (flac -t) on them again. If they pass, the corruption was isolated to a non-audio metadata block.

\---------------------------------------------------------------------------------------
## 15b: Cover Consolidation → Cover.jpg (Reference / Legacy)

**Note:** Step 15c "Update Album Artwork Embeds" supersedes this procedure for everything embedded inside audio files, and covers FLAC, MP3, M4A, and MP4. 15b only standardizes the folder-level cover image; use Step 15c unless you specifically need to rename/convert folder covers to a single `Cover.jpg`.

-- Purpose

This procedure standardizes which image file each album directory uses as its cover, matching moOde's coverart.php priority order.

Over years of collection, an album directory can accumulate wildly inconsistent artwork — `cover.jpg`, `folder.jpg`, stray PNGs, and so on — which makes cover display unpredictable.

This step ensures every album directory has exactly one canonical `Cover.jpg` by renaming the highest-priority existing cover (PNGs and TIFFs are converted at q:v 2) and leaving every other image file in place, logged for review. TIFF sources are consolidated — once a TIFF has been converted into `Cover.jpg`, the source `.tiff`/`.tif` file is removed (moOde cannot use TIFF, and the SHA-512 guide's stray audit would otherwise keep flagging it); the removal is logged in `step15b-run.log`.

-- What It Does

This step (folder covers only — audio files are never touched):

* Scans the library by album directory for any image file.
* Picks the highest-priority existing cover using moOde's coverart.php order (Cover.jpg, cover.jpg, Cover.jpeg, cover.jpeg, Cover.png, cover.png, Folder.* variants). TIFF/TIF files are recognized as cover sources (converted to JPEG at q:v 2, source removed after a successful conversion) and by the stray-promotion fallback.
* If no standard candidate exists, promotes the alphabetically-first stray image and logs the promotion for review.
* Renames JPEG covers to `Cover.jpg`; converts PNG covers to JPEG at q:v 2.
* Logs every other image file in the directory to step15b-review.log — nothing is deleted.
* Audio files are never read or modified.

\---------------------------------------------------------------------------------------

--- Bash Script for 15b Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# 15b. Consolidate Album Artwork -> one Cover.jpg per directory
#   Per-directory rule (moOde coverart.php priority order):
#     - The highest-priority existing cover becomes Cover.jpg
#       (renamed if needed; PNG and TIFF converted to JPEG at high quality)
#     - Every OTHER image file in the dir is left in place and
#       logged to step15b-review.log (nothing is deleted)
#   Byte-identical renames never invalidate existing embeds.
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step15b-errors.log"
: > "$LOG_ROOT/step15b-review.log"
: > "$LOG_ROOT/step15b-run.log"

# Software Preflight: fail loudly if a required tool is missing
for tool in ffmpeg find; do
    if ! command -v "$tool" >/dev/null 2>&1; then
        echo "ERROR: $tool is not installed. Install it and re-run (see Requirements)." >&2
        exit 1
    fi
done

# Moode-standard folder-level cover priority (matches moOde's coverart.php parseFolder())
COVER_CANDIDATES=(
    "Cover.jpg" "cover.jpg" "Cover.jpeg" "cover.jpeg" "Cover.png" "cover.png"
    "Folder.jpg" "folder.jpg" "Folder.jpeg" "folder.jpeg" "Folder.png" "folder.png"
)

# Progress line: [done/total] % complete, elapsed and ETA (terminal only)
start_ts=$(date +%s)
progress() {
    local done_n=$1 total_n=$2 now el pct eta
    [ "$total_n" -gt 0 ] || return 0
    [ -t 2 ] || return 0
    now=$(date +%s)
    el=$((now - start_ts))
    pct=$((done_n * 100 / total_n))
    eta=0
    [ "$done_n" -gt 0 ] && eta=$((el * (total_n - done_n) / done_n))
    printf '\r\033[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   ' \
        "$done_n" "$total_n" "$pct" \
        $((el/3600)) $(((el/60)%60)) $((el%60)) \
        $((eta/3600)) $(((eta/60)%60)) $((eta%60)) >&2
}

# Dirs that contain any image file (Ignore dirs excluded)
mapfile -d '' dirs < <(
    find "$PWD" -type d \
        ! -ipath '*/Ignore/*' ! -ipath '*/Ignore' ! -iname 'Ignore' \
        -print0 | while IFS= read -r -d '' d; do
        if compgen -G "$d/*.jpg" >/dev/null || compgen -G "$d/*.jpeg" >/dev/null || \
           compgen -G "$d/*.JPG" >/dev/null || compgen -G "$d/*.JPEG" >/dev/null || \
           compgen -G "$d/*.png" >/dev/null || compgen -G "$d/*.PNG" >/dev/null || \
           compgen -G "$d/*.tiff" >/dev/null || compgen -G "$d/*.tif" >/dev/null || \
           compgen -G "$d/*.TIFF" >/dev/null || compgen -G "$d/*.TIF" >/dev/null; then
            printf '%s\0' "$d"
        fi
    done
)

total=${#dirs[@]}
i=0
renamed=0
converted=0
already_ok=0
failed=0

for d in "${dirs[@]}"; do
    i=$((i+1))
    printf '\r\033[K── %s ──\n' "${d#"$PWD"/}" >&2
    progress "$i" "$total"

    parent_dir="${d%/*}"
    artist="${parent_dir##*/}"
    album="${d##*/}"
    label="$artist-$album"
    target="$d/Cover.jpg"

    # Pick the highest-priority existing cover (moOde order)
    best=""
    for name in "${COVER_CANDIDATES[@]}"; do
        if [ -s "$d/$name" ]; then
            best="$d/$name"
            break
        fi
    done

    # No standard candidate? Promote the alphabetically-first stray image.
    if [ -z "$best" ]; then
        stray=$(find "$d" -maxdepth 1 -type f \( -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.png' -o -iname '*.tiff' -o -iname '*.tif' \) | sort | head -1)
        if [ -n "$stray" ]; then
            best="$stray"
            echo "[$i/$total] REVIEW: $label :: promoting non-standard cover $(basename "$stray")" \
                >> "$LOG_ROOT/step15b-review.log"
        else
            continue
        fi
    fi

    if [ "$best" = "$target" ]; then
        already_ok=$((already_ok+1))
        echo "SAME  [$i/$total] $label" >> "$LOG_ROOT/step15b-run.log"
    else
        case "${best##*.}" in
            jpg|jpeg|JPG|JPEG)
                if [ "$(basename "$best")" = "$(basename "$target")" ] || [ "$(basename "$best")" = "cover.jpg" ]; then
                    mv -f "$best" "$d/.cover-tmp.jpg" && mv -f "$d/.cover-tmp.jpg" "$target" || { failed=$((failed+1)); echo "[$i/$total] ERROR: $label :: rename failed" >> "$LOG_ROOT/step15b-errors.log"; continue; }
                else
                    mv -f "$best" "$target" || { failed=$((failed+1)); echo "[$i/$total] ERROR: $label :: rename failed" >> "$LOG_ROOT/step15b-errors.log"; continue; }
                fi
                renamed=$((renamed+1))
                echo "RENAMED [$i/$total] $label :: $(basename "$best") -> Cover.jpg" | tee -a "$LOG_ROOT/step15b-run.log"
                ;;
            png|PNG)
                err=$(ffmpeg -y -nostdin -v error -i "$best" -q:v 2 "$target" 2>&1)
                rc=$?
                if [ $rc -ne 0 ] || [ ! -s "$target" ]; then
                    flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
                    echo "FAIL [$i/$total] $label" | tee -a "$LOG_ROOT/step15b-run.log"
                    echo "[$i/$total] ERROR (exit $rc, png->jpg convert): $label :: $best :: ${flat:-no stderr output}" \
                        >> "$LOG_ROOT/step15b-errors.log"
                    failed=$((failed+1))
                    rm -f "$target"
                    continue
                fi
                converted=$((converted+1))
                echo "CONVERTED [$i/$total] $label :: $(basename "$best") -> Cover.jpg (png->jpg, q:v 2)" | tee -a "$LOG_ROOT/step15b-run.log"
                ;;
            tiff|tif|TIFF|TIF)
                err=$(ffmpeg -y -nostdin -v error -i "$best" -q:v 2 "$target" 2>&1)
                rc=$?
                if [ $rc -ne 0 ] || [ ! -s "$target" ]; then
                    flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
                    echo "FAIL [$i/$total] $label" | tee -a "$LOG_ROOT/step15b-run.log"
                    echo "[$i/$total] ERROR (exit $rc, tiff->jpg convert): $label :: $best :: ${flat:-no stderr output}" \
                        >> "$LOG_ROOT/step15b-errors.log"
                    failed=$((failed+1))
                    rm -f "$target"
                    continue
                fi
                # TIFF cannot be used by moOde and would be flagged as a stray
                # by the SHA-512 guide's audit, so the source is consolidated
                # (removed) once its content lives in Cover.jpg.
                rm -f "$best"
                converted=$((converted+1))
                echo "CONVERTED [$i/$total] $label :: $(basename "$best") -> Cover.jpg (tiff->jpg, q:v 2; source tiff removed)" | tee -a "$LOG_ROOT/step15b-run.log"
                ;;
        esac
    fi

    # Every other image in the dir is logged, never deleted
    while IFS= read -r extra; do
        [ "$extra" = "$target" ] && continue
        echo "[$i/$total] REVIEW: $label :: extra cover left in place: $(basename "$extra")" \
            >> "$LOG_ROOT/step15b-review.log"
    done < <(find "$d" -maxdepth 1 -type f \( -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.png' -o -iname '*.tiff' -o -iname '*.tif' \) | sort)

done

printf '\n' >&2
echo
echo "----------------------------------------"
echo "15b. Consolidate Album Artwork -> Cover.jpg"
echo "Dirs: $total   Already Cover.jpg: $already_ok   Renamed: $renamed   Converted: $converted   Failed: $failed"
echo "Extras left in place: $(wc -l < "$LOG_ROOT/step15b-review.log")  (see $LOG_ROOT/step15b-review.log)"
echo "----------------------------------------"
```
--- Bash Script for 15b End ---

\---------------------------------------------------------------------------------------

-- Separate Results

After the artwork normalization completes, separate successful and failed results:

--- Bash Script Results for 15b Start ---
```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"

grep -E '^(RENAMED|CONVERTED)' "$LOG_ROOT/step15b-run.log" > "$LOG_ROOT/step15b-oks.log"
grep '^FAIL' "$LOG_ROOT/step15b-run.log" > "$LOG_ROOT/step15b-fails.log"

echo "Step 15b Renamed/Converted: $(wc -l < "$LOG_ROOT/step15b-oks.log")  FAILs: $(wc -l < "$LOG_ROOT/step15b-fails.log")"

```
--- Bash Script Results for 15b End ---

\---------------------------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 15b Start ---
```bash

cat "$LOG_ROOT/step15b-errors.log"
cat "$LOG_ROOT/step15b-run.log"
cat "$LOG_ROOT/step15b-oks.log"
cat "$LOG_ROOT/step15b-fails.log"

```
--- Bash Script Cat for 15b End ---

\---------------------------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step15b-run.log — Complete processing results for directories containing artwork (RENAMED, CONVERTED, SAME, and FAIL lines).
* step15b-oks.log — Cover files renamed to Cover.jpg or converted from PNG (RENAMED/CONVERTED lines).
* step15b-fails.log — Cover operations that failed.
* step15b-review.log — Non-standard covers promoted, plus every extra image file left in place.
* step15b-errors.log — Detailed error output from failed operations.

This step never embeds artwork into audio files — it only standardizes the folder-level cover to `Cover.jpg`. To update the artwork embedded inside the audio files themselves, run Step 15c afterward. Albums without any image file are simply ignored by this script. To process them, place an appropriately sized JPEG in their directory and re-run the script.

\---------------------------------------------------------------------------------------

## 15c. Update Album Artwork Embeds (FLAC, MP3, M4A, MP4)

**Primary Step for Artwork Embed Operations**

-- Purpose

This procedure standardizes album artwork across the formats this script can safely embed: FLAC, MP3, M4A, and MP4. This is the recommended multi-format replacement for the legacy Step 15b.

OGG, Opus, AIFF, AIF, APE, and DSF are NOT embedded by this script: ffmpeg cannot attach cover art to OGG/Opus (their native containers store art as a `METADATA_BLOCK_PICTURE` Vorbis comment instead of a picture stream), and the AIFF/APE/DSF muxers in ffmpeg silently drop or reject picture streams. Files in those formats are detected and skipped with a `SKIP` line — never modified.

Over years of collection, a library can accumulate wildly inconsistent artwork—some files having no art, others having massive 10MB uncompressed PNGs, and others having multiple conflicting images.

This step ensures every file in an album contains the exact same, standardized cover image by reading a local image file (like cover.jpg or folder.jpg) stored in the album directory and embedding it appropriately for each format's native tag structure.

-- What It Does

This step:

* Scans the library by album directory across all supported formats.
* Looks for a standard image file named cover.jpg, folder.jpg, or other Moode-standard formats in each directory.
* For FLAC files: Uses metaflac to remove old artwork and embed the new image.
* For non-FLAC formats (MP3, M4A, MP4): Uses ffmpeg to attach the artwork as a cover picture stream without re-encoding audio. OGG, Opus, AIFF, APE, and DSF files are logged as `SKIP` and left unchanged.
* Leaves the underlying audio data completely unchanged (audio streams are never re-encoded).
* Skips directories that do not contain a recognized standard image file.

\---------------------------------------------------------------------------------------

--- Bash Script for 15c Start ---
```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# 15c. Update Album Artwork Embeds (FLAC, MP3, M4A, MP4)
# ------------------------------------------------------------

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
mkdir -p "$LOG_ROOT"
: > "$LOG_ROOT/step15c-errors.log"
: > "$LOG_ROOT/step15c-review.log"

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
    printf '── %s ──\n' "${d#"$PWD"/}" >&2

    parent_dir="${d%/*}"
    artist="${parent_dir##*/}"
    album="${d##*/}"
    label="$artist - $album"
    error_found=0

    art_file=$(find_cover_art "$d")

    if [ -z "$art_file" ]; then
        echo "ERROR [$i/$total] $label :: Missing standard image file"
        echo "[$i/$total] ERROR: $label :: No Moode-standard cover image found (checked: ${COVER_CANDIDATES[*]})" >> "$LOG_ROOT/step15c-errors.log"
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
        echo "[$i/$total] ERROR: $label :: Directory has no supported audio files" >> "$LOG_ROOT/step15c-errors.log"
        continue
    fi

    processed_any=0
    unchanged=0
    kept=0
    upgraded=0

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

        # Extract the currently embedded artwork (if any) to a temp file
        emb=$(mktemp "$LOG_ROOT/step15c-emb.XXXXXX.${art_file##*.}")
        rm -f "$emb" 2>/dev/null
        has_emb=0
        if [[ "$ext_lower" == "flac" && $HAS_METAFLAC -eq 1 ]]; then
            if metaflac --export-picture-to="$emb" "$f" >/dev/null 2>&1 && [ -s "$emb" ]; then
                has_emb=1
            fi
        else
            if ffmpeg -y -nostdin -v error -i "$f" -map 0:v:0 -c copy -update 1 -f image2 "$emb" >/dev/null 2>&1 && [ -s "$emb" ]; then
                has_emb=1
            fi
        fi

        # Resolution gate: only a STRICTLY higher-resolution folder cover may
        # replace an existing embed. Byte-identical or equal/lower-res covers
        # leave the file untouched.
        do_embed=0
        if [ $has_emb -eq 0 ]; then
            do_embed=1
        elif cmp -s "$emb" "$art_file"; then
            :
        else
            emb_dims=$(ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of csv=s=x:p=0 "$emb" 2>/dev/null)
            art_dims=$(ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of csv=s=x:p=0 "$art_file" 2>/dev/null)
            emb_w=${emb_dims%x*};      emb_h=${emb_dims#*x}
            art_w=${art_dims%x*};      art_h=${art_dims#*x}
            case "$emb_w$emb_h$art_w$art_h" in *[!0-9]*|"") 
                echo "[$i/$total] REVIEW: $label :: ${f##*/} :: could not compare resolutions (emb=$emb_dims cover=$art_dims) — kept existing" \
                    >> "$LOG_ROOT/step15c-review.log"
                kept=$((kept+1))
                ;;
            *)  if [ $((art_w * art_h)) -gt $((emb_w * emb_h)) ]; then
                    do_embed=1
                else
                    kept=$((kept+1))
                    echo "[$i/$total] KEPT: $label :: ${f##*/} :: embedded ${emb_dims} >= cover ${art_dims} — existing artwork kept" \
                        >> "$LOG_ROOT/step15c-review.log"
                fi
                ;;
            esac
        fi
        rm -f "$emb" 2>/dev/null

        if [ $do_embed -eq 0 ]; then
            unchanged=$((unchanged+1))
            continue
        fi

        if [[ "$ext_lower" == "flac" && $HAS_METAFLAC -eq 1 ]]; then
            rmerr=$(metaflac --remove --block-type=PICTURE "$f" 2>&1)
            rmrc=$?
            if [ $rmrc -ne 0 ]; then
                error_found=1
                rmflat=$(echo "$rmerr" | tr '\n' ' ' | tr -s ' ')
                echo "[$i/$total] ERROR (exit $rmrc, remove-picture): $label :: ${f##*/} :: ${rmflat:-no stderr output}" \
                    >> "$LOG_ROOT/step15c-errors.log"
            fi

            err=$(metaflac --import-picture-from="$art_file" "$f" 2>&1)
            rc=$?
            if [ $rc -ne 0 ]; then
                error_found=1
                flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
                echo "[$i/$total] ERROR (exit $rc, import-art): $label :: ${f##*/} :: ${flat:-no stderr output}" \
                    >> "$LOG_ROOT/step15c-errors.log"
            else
                upgraded=$((upgraded+1))
            fi
        else
            temp_file=$(mktemp "$LOG_ROOT/step15c-tagged.XXXXXX.${ext_lower}")
            rm -f "$temp_file" 2>/dev/null

            err=$(ffmpeg -y -nostdin -loglevel error -i "$f" -i "$art_file" \
                -map 0:a -map 1 -c copy -disposition:v attached_pic "$temp_file" 2>&1)
            rc=$?

            if [ $rc -eq 0 ] && [ -s "$temp_file" ]; then
                mv -f "$temp_file" "$f" 2>/dev/null
                upgraded=$((upgraded+1))
            else
                error_found=1
                rm -f "$temp_file" 2>/dev/null
                flat=$(echo "$err" | tr '\n' ' ' | tr -s ' ')
                echo "[$i/$total] ERROR (exit $rc, ffmpeg): $label :: ${f##*/} :: ${flat:-no stderr output}" \
                    >> "$LOG_ROOT/step15c-errors.log"
            fi
        fi
    done

    if [ $error_found -eq 0 ] && [ $upgraded -gt 0 ]; then
        echo "OK    [$i/$total] $label (upgraded: $upgraded, kept: $kept, unchanged: $unchanged)"
    elif [ $error_found -eq 0 ] && [ $processed_any -gt 0 ]; then
        echo "SAME  [$i/$total] $label (kept: $kept, unchanged: $unchanged — nothing needed a higher-res cover)"
    elif [ $processed_any -eq 0 ]; then
        echo "SKIP  [$i/$total] $label"
        echo "[$i/$total] SKIP: $label :: no embeddable audio files (FLAC/MP3/M4A/MP4) in this directory" \
            >> "$LOG_ROOT/step15c-errors.log"
    else
        echo "ERROR [$i/$total] $label"
    fi

done | tee "$LOG_ROOT/step15c-run.log"
echo
echo "----------------------------------------"
echo "15c. Update Album Artwork Embeds (FLAC, MP3, M4A, MP4)"
echo "----------------------------------------"

```
--- Bash Script for 15c End ---

\---------------------------------------------------------------------------------------

-- Separate Results

--- Bash Script Results for 15c Start ---
```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"

grep '^OK' "$LOG_ROOT/step15c-run.log" > "$LOG_ROOT/step15c-oks.log"
grep '^ERROR' "$LOG_ROOT/step15c-run.log" > "$LOG_ROOT/step15c-fails.log"
grep '^SKIP' "$LOG_ROOT/step15c-run.log" > "$LOG_ROOT/step15c-skips.log"

echo "Step 15c Universal OKs: $(wc -l < "$LOG_ROOT/step15c-oks.log")  SKIPs: $(wc -l < "$LOG_ROOT/step15c-skips.log")  ERRORs: $(wc -l < "$LOG_ROOT/step15c-fails.log")"

```
--- Bash Script Results for 15c End ---

\---------------------------------------------------------------------------------------

-- Review Results

View the generated reports:

--- Bash Script Cat for 15c Start ---
```bash

cat "$LOG_ROOT/step15c-errors.log"
cat "$LOG_ROOT/step15c-run.log"
cat "$LOG_ROOT/step15c-oks.log"
cat "$LOG_ROOT/step15c-fails.log"
cat "$LOG_ROOT/step15c-skips.log"

```
--- Bash Script Cat for 15c End ---

\---------------------------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step15c-run.log — Complete processing results for all directories with supported audio files.
* step15c-oks.log — Directories successfully updated with standardized artwork across all formats.
* step15c-fails.log — Directories where artwork embedding failed (usually due to missing cover image).
* step15c-skips.log — Files or directories skipped because their format cannot carry embedded artwork through this script (OGG, Opus, AIFF, APE, DSF).
* step15c-errors.log — Detailed error output from metaflac (FLAC) or ffmpeg (other formats).

Directories without a standard cover image (Cover.jpg, cover.jpg, Folder.jpg, folder.jpg, etc.) are logged as errors and require manual intervention—place an appropriately sized JPEG in the directory and re-run the script.

\---------------------------------------------------------------------------------------

15d. Ignore-Content Certification

---

-- Purpose

The `Ignore` folders hold library songs that moOde never plays (hidden via
`.mpdignore` markers) — odd tracks, blanks, spoken word, or extra-long
recordings — but they are still part of the collection and need protection
from bit rot. Two facts define this step's job:

* The SHA-512 guide's artist-level manifests already fingerprint the
  contents of each album folder's Ignore subfolder inside the aggregate
  hash, so corruption there will fail an artist-level verification. What
  the aggregate cannot do is say *which* file rotted, and nothing has
  ever decode-tested these files.
* This step closes both gaps: it integrity-tests every audio file in
  every `Ignore` folder and writes a self-contained
  `Ignore.sha512sums.txt` manifest inside each folder, so per-file
  detection and localized re-verification work without touching the
  album/artist manifests.

Ignore content stays OUT of the regular pipeline (no dedup, rebuild, or
ReplayGain) and OUT of the album/artist manifests; `Ignore.sha512sums.txt`
is a third accepted generic manifest name (the SHA-512 guide's stray
audit and manifest-name checker accept it).

-- What It Does

For each `Ignore` folder (any depth, nested Ignore-in-Ignore skipped):

* Verifies the existing `Ignore.sha512sums.txt` if one is present.
* Creates the manifest when it is missing — hashing every regular file in
  the folder (audio, artwork, notes, `.mpdignore`) except the manifest
  itself.
* Integrity-tests every audio file by full decode (`flac -t` for FLAC,
  `ffmpeg` null-decode for the other formats) — these files have never
  been decode-tested by Steps 1/4/10, which exclude Ignore content.
* Reports failures loudly; never modifies audio or deletes anything.

-- Regeneration

If a verification fails and you have corrected or accepted the files
(after investigation), delete that folder's `Ignore.sha512sums.txt` and
re-run 15d to rebuild it.

-- Logging

Writes `step15d-run.log`, `step15d-oks.log`, `step15d-fails.log`,
`step15d-errors.log`, and `step15d-summary.log` to
`$HOME/.logs/linux-audio-moode-cleanup-guide`. Per-file OK lines are
log-only; FAIL lines print to the terminal under the album header,
per the suite screen convention.

--- Bash Script for 15d Start ---

```bash

#!/usr/bin/env bash

# Keep the terminal open on any failure so the error cause stays visible
trap 'rc=$?; if [ "$rc" -ne 0 ]; then trap - EXIT; echo; echo "Script exited with status $rc. Press ENTER to close this terminal."; read -r _; exit "$rc"; fi' EXIT
# ------------------------------------------------------------
# 15d. Ignore-Content Certification — decode-test + per-folder SHA-512
# ------------------------------------------------------------

set -u

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step15d"
mkdir -p "$LOG_ROOT"

RUN_LOG="$LOG_ROOT/${STEP}-run.log"
OKS_LOG="$LOG_ROOT/${STEP}-oks.log"
FAILS_LOG="$LOG_ROOT/${STEP}-fails.log"
ERRORS_LOG="$LOG_ROOT/${STEP}-errors.log"
SUMMARY_LOG="$LOG_ROOT/${STEP}-summary.log"

: > "$RUN_LOG"; : > "$OKS_LOG"; : > "$FAILS_LOG"; : > "$ERRORS_LOG"; : > "$SUMMARY_LOG"

# --- Preflight: required tools
for tool in flac ffmpeg sha512sum; do
    if ! command -v "$tool" >/dev/null 2>&1; then
        echo "ERROR: $tool is not installed. Install it and re-run (see Requirements)." >&2
        exit 1
    fi
done

# --- Discover Ignore folders (any depth, case-insensitive;
#     nested Ignore-in-Ignore skipped)
mapfile -d '' idirs < <(find "$PWD" -type d -iname 'Ignore' \
    ! -ipath '*/Ignore/*' -print0 | LC_ALL=C sort -z)

if [ "${#idirs[@]}" -eq 0 ]; then
    echo
    echo "----------------------------------------"
    echo "15d. Ignore-Content Certification"
    echo "No Ignore folders found - nothing to certify."
    echo "----------------------------------------"
    exit 0
fi

# --- Progress line: [done/total] % complete, elapsed and ETA (terminal only)
start_ts=$(date +%s)
progress() {
    local done_n=$1 total_n=$2 now el pct eta
    [ "$total_n" -gt 0 ] || return 0
    [ -t 2 ] || return 0
    now=$(date +%s)
    el=$((now - start_ts))
    pct=$((done_n * 100 / total_n))
    eta=0
    [ "$done_n" -gt 0 ] && eta=$((el * (total_n - done_n) / done_n))
    printf '\r\033[K[%d/%d] %3d%% complete  elapsed %02d:%02d:%02d  ETA %02d:%02d:%02d   ' \
        "$done_n" "$total_n" "$pct" \
        $((el/3600)) $(((el/60)%60)) $((el%60)) \
        $((eta/3600)) $(((eta/60)%60)) $((eta%60)) >&2
}

MANIFEST_NAME="Ignore.sha512sums.txt"
total_dirs=${#idirs[@]}

mapfile -d '' all_audio < <(
    find "${idirs[@]}" -maxdepth 1 -type f \( \
        -iname "*.flac" -o -iname "*.mp3" -o -iname "*.m4a" -o \
        -iname "*.ogg"  -o -iname "*.opus" -o -iname "*.wav"  -o \
        -iname "*.aiff" -o -iname "*.aif"  -o -iname "*.ape"  -o \
        -iname "*.wv"   -o -iname "*.spx" \
    \) -print0 | LC_ALL=C sort -z
)
total_files=${#all_audio[@]}

echo "========== 15d: Ignore-Content Certification ==========" | tee -a "$RUN_LOG"
echo "Root: $PWD" | tee -a "$RUN_LOG"
echo "Started: $(date)" | tee -a "$RUN_LOG"
echo "Ignore folders: $total_dirs   Audio files: $total_files" | tee -a "$RUN_LOG"
echo | tee -a "$RUN_LOG"

manifests_created=0
manifests_verified=0
tested_ok=0
tested_fail=0
dir_idx=0
j=0

for d in "${idirs[@]}"; do
    dir_idx=$((dir_idx + 1))
    rel_dir="${d#"$PWD"/}"
    printf '\r\033[K── %s ──\n' "$rel_dir" >&2
    label="$(basename "$(dirname "$d")") - $(basename "$d") [Ignore]"

    shopt -s nocaseglob nullglob
    audio=("$d"/*.flac "$d"/*.mp3 "$d"/*.m4a "$d"/*.ogg "$d"/*.opus "$d"/*.wav \
           "$d"/*.aiff "$d"/*.aif "$d"/*.ape "$d"/*.wv "$d"/*.spx)
    shopt -u nocaseglob nullglob

    manifest="$d/$MANIFEST_NAME"

    # 1. Manifest: verify if present, create if missing
    if [ -f "$manifest" ]; then
        vout=$(cd "$d" && sha512sum -c --strict "$MANIFEST_NAME" 2>&1)
        vrc=$?
        if [ "$vrc" -eq 0 ]; then
            manifests_verified=$((manifests_verified + 1))
            echo "OK   [$dir_idx/$total_dirs] $rel_dir :: manifest verified ($(wc -l < "$manifest") entries)" >> "$RUN_LOG"
            echo "OK   [$dir_idx/$total_dirs] $rel_dir" >> "$OKS_LOG"
        else
            flat=$(printf '%s\n' "$vout" | tr '\r\n' '  ' | tr -s ' ')
            echo "FAIL [$dir_idx/$total_dirs] $rel_dir"
            echo "FAIL [$dir_idx/$total_dirs] $rel_dir" >> "$RUN_LOG"
            echo "FAIL [$dir_idx/$total_dirs] $rel_dir" >> "$FAILS_LOG"
            echo "[$dir_idx/$total_dirs] ERROR (manifest verify): $rel_dir :: ${flat:-no output}" >> "$ERRORS_LOG"
        fi
    else
        if (cd "$d" && find . -maxdepth 1 -type f ! -name "$MANIFEST_NAME" -print0 \
                | LC_ALL=C sort -z | xargs -0 -r sha512sum > "$MANIFEST_NAME" 2>>"$ERRORS_LOG"); then
            manifests_created=$((manifests_created + 1))
            echo "CREATED [$dir_idx/$total_dirs] $rel_dir :: $MANIFEST_NAME" >> "$RUN_LOG"
            echo "CREATED [$dir_idx/$total_dirs] $rel_dir" >> "$OKS_LOG"
        else
            echo "FAIL [$dir_idx/$total_dirs] $rel_dir (manifest creation failed)"
            echo "FAIL [$dir_idx/$total_dirs] $rel_dir" >> "$RUN_LOG"
            echo "FAIL [$dir_idx/$total_dirs] $rel_dir" >> "$FAILS_LOG"
        fi
    fi

    # 2. Integrity-test every audio file in this Ignore folder
    for f in "${audio[@]}"; do
        j=$((j + 1))
        progress "$j" "$total_files"
        case "${f,,}" in
            *.flac) err=$(flac -s -t "$f" 2>&1); rc=$? ;;
            *)      err=$(ffmpeg -nostdin -v error -i "$f" -f null - 2>&1); rc=$? ;;
        esac
        rel_f="${f#"$PWD"/}"
        if [ "$rc" -eq 0 ]; then
            tested_ok=$((tested_ok + 1))
            echo "OK   [$j/$total_files] $rel_f" >> "$RUN_LOG"
            echo "OK   [$j/$total_files] $rel_f" >> "$OKS_LOG"
        else
            flat=$(printf '%s\n' "$err" | tr '\r\n' '  ' | tr -s ' ')
            tested_fail=$((tested_fail + 1))
            echo "FAIL [$j/$total_files] $rel_f"
            echo "FAIL [$j/$total_files] $rel_f" >> "$RUN_LOG"
            echo "FAIL [$j/$total_files] $rel_f" >> "$FAILS_LOG"
            echo "[$j/$total_files] ERROR (exit $rc): $rel_f :: ${flat:-no stderr output}" >> "$ERRORS_LOG"
        fi
    done
done
printf '\n' >&2

{
echo "Step 15d Summary"
echo "=============="
echo
echo "Step               : step15d"
echo "Run Date           : $(date)"
echo
echo "Ignore folders     : $total_dirs"
echo "Manifests created  : $manifests_created"
echo "Manifests verified : $manifests_verified"
echo "Audio files tested : $j (OK: $tested_ok  FAIL: $tested_fail)"
} > "$SUMMARY_LOG"

echo
echo "----------------------------------------"
echo "Step 15d Summary Review"
echo "----------------------------------------"
echo "Ignore folders     : $total_dirs"
echo "Manifests created  : $manifests_created"
echo "Manifests verified : $manifests_verified"
echo "Audio files tested : $j (OK: $tested_ok  FAIL: $tested_fail)"
echo
echo "Summary written to : $SUMMARY_LOG"
echo
echo "----------------------------------------"
echo "15d – Ignore-Content Certification"
echo "----------------------------------------"

```
--- Bash Script for 15d End ---

\---------------------------------------------------------------------------------------

-- Review Results

View the generated reports by running:

--- Bash Script Cat for 15d Start ---

```bash

LOG_ROOT="$HOME/.logs/linux-audio-moode-cleanup-guide"
STEP="step15d"

cat "$LOG_ROOT/${STEP}-run.log"
cat "$LOG_ROOT/${STEP}-oks.log"
cat "$LOG_ROOT/${STEP}-fails.log"
cat "$LOG_ROOT/${STEP}-errors.log"
cat "$LOG_ROOT/${STEP}-summary.log"

```
--- Bash Script Cat for 15d End ---

\---------------------------------------------------------------------------------------

-- Expected Results

A successful run produces:

* step15d-run.log — Full transcript: one `── Ignore folder ──` header per
  folder, `OK`/`CREATED`/`FAIL` lines per folder and per file.
* step15d-oks.log — Folders whose manifest verified, and folders whose
  manifest was newly created.
* step15d-fails.log — Folders whose manifest verification failed or whose
  creation failed, and any audio file that failed its decode test.
* step15d-errors.log — Detailed verify/creation stderr for failed folders.
* step15d-summary.log — Folder, manifest and test counts.

First run: 21 manifests created (one per Ignore folder). Re-runs: all
manifests verify. If verification fails after an intentional change
(e.g., a track was replaced inside an Ignore folder), delete that
folder's `Ignore.sha512sums.txt` and re-run 15d to rebuild it.

\---------------------------------------------------------------------------------------

16. Generate Checksums

-- Purpose

Checksums finalize the library for long-term archival by generating cryptographic hashes for the audio files. They serve as a digital fingerprint, allowing you to detect "bit rot" (silent data corruption on storage media) or verify that data remains perfectly intact after a massive transfer, such as an rsync backup to an external drive.

-- Separate Repository

SHA-512 checksum generation is maintained in a dedicated, standalone repository rather than in this guide. It was once included here, but it grew to be too much for a single project and serves users better on its own — especially those who only want to secure what they have without running the rest of the cleanup workflow.

All scripts, usage instructions, and the resulting checksums.sha512 format live in the repository:

Link to SHA-512 Repository: https://github.com/TerrapinATL/linux-audio-sha512-checksums

-- When to Generate Checksums

Generate checksums on the working copy only after all cleanup steps (Steps 1-10 including Step 1B, plus any optional Steps 15a-15d you intend to run) are complete. SHA-512 hashes capture the file contents at the moment they are generated, so any later modification of the audio files will break previously generated checksum files.

After generating the checksum files, copy them along with the audio files to the Master Library and to your rsync backup drive, so the Master Library and every backup carries its own trusted fingerprint. Verify periodically with `sha512sum -c checksums.sha512` in each directory to catch bit rot early.

---

17. Troubleshooting & Reference Information

---

This section provides supplementary materials to support the primary workflow. It includes solutions for common errors encountered during library processing and a detailed breakdown of the log files generated by the automated scripts, ensuring you can effectively troubleshoot issues and verify your results.

-- Common Issues and Fixes:

\---------------------------------------------------------------------------------------

   1. FLAC Decoding Errors (e.g., Unknown Total Samples)

When validating audio files with flac -t, you might encounter failures like ERROR: FLAC input has STREAMINFO with unknown total samples. This is often caused by the software that originally extracted or encoded the files generating an incorrect or missing metadata header. Because FLAC is a lossless format, you can safely repair the file structure without degrading the audio quality by re-encoding the file with ffmpeg.

Run this command on the broken file:

--- Bash Script Start ---
```bash

ffmpeg -i "broken.flac" -c:a flac "fixed.flac"

```
--- Bash Script End ---

Once you verify that fixed.flac plays correctly and passes a flac -t check, you can replace the broken original.

\---------------------------------------------------------------------------------------

   2. Permission Denied Errors

If tools like metaflac, loudgain, or sha512sum fail to write data to a directory, it usually indicates a file ownership or permission issue. This is common when copying files from different filesystems, running tools in a virtual machine, or moving data from external drives.

To grant your current user full read and write access to the library, run:

--- Bash Script Start ---
```bash

sudo chown -R $USER:$USER /path/to/your/music/library
chmod -R u+rw /path/to/your/music/library

```
--- Bash Script End ---

\---------------------------------------------------------------------------------------

   3. Checksum Verification Failures

If you ever run sha512sum -c checksums.sha512 in an album directory and it reports a mismatch, it means the audio file has been modified or corrupted (bit rot) since the original checksum was generated. Do not re-run the checksum generation script to "fix" the error — that will just validate the corrupted state. Instead, delete the corrupted file from your master library and restore a pristine copy from your rsync backup drive.

\---------------------------------------------------------------------------------------

-- Log File Reference & Understanding Your Log Files

Throughout this workflow, all scripts direct their tracking data to a dedicated log directory created in your home directory ($HOME/.logs/linux-audio-moode-cleanup-guide), regardless of which library folder you're working in. This ensures your music directories remain completely free of random text files and gives you a single, centralized place to review the results of your mass operations across every run.

Every log file lives directly in that one directory and is named for the step that produced it, using the pattern stepNN-logname.log (e.g. step01-run.log, step02a-fails.log, step08-errors.log). The naming convention is consistent across all processing steps:

  1. stepNN-run.log
    The complete, raw output of the script. It lists every directory sequentially as it is processed, showing the OK or FAIL status for each one.

  2. stepNN-oks.log
    A filtered list containing only the albums that were processed successfully.

  3. stepNN-fails.log
    A filtered list of albums that encountered an issue. This is your primary "to-do" list for manual troubleshooting. If this file is empty, the step was a 100% success.

  4. stepNN-errors.log
    Contains the detailed standard error (stderr) output from the specific command-line utilities (like ffmpeg, loudgain, or metaflac). When an album shows up in the fails.log, you can check this error file to see exactly why it failed (e.g., "file not found," "malformed metadata," "permission denied").

\---------------------------------------------------------------------------------------

-- General Cleanup

Once you have reviewed the final logs, verified that your master library is fully processed, and completed your rsync transfer to the external backup drive, you can safely delete the entire $HOME/.logs/linux-audio-moode-cleanup-guide directory. It is completely independent of the audio files and is no longer needed once the project is finished.

\---------------------------------------------------------------------------------------

-- Disclaimer

This guide was developed through iterative collaborative effort between ChatGPT, Claude, Gemini, Mistral and the user. I cannot thank OpenCode project enough. I was about to give up on the other four (well, actually I did) when I came across OpenCode. I run a 10+ year old laptop yet OpenCode ran perfectly well, offloading the heaving lifting to an offsite server. 

https://opencode.ai/






