### linux-audio-moode-cleanup-guide — Change Log

All version changes are appended to this file, newest last, one `## vX.Y Change Log` section per version.

**Update rule:** before writing to the version-less main guide file, the current content must first be saved as a versioned copy (e.g. `linux-audio-moode-cleanup-guide-v28.md`) so every published version stays retrievable.

**Suite convention (auto-purge):** every guide/repo with error logging must purge its log directory at the START of the workflow (first step), so the previous run's logs remain reviewable until the next run replaces them. This applies to all current and future repositories.

**Current version: v34** — supersedes v33. Converts Step 2A and 2B
from bash to extensionless Python, completing the whole Step 2 suite
in Python. See the v34 entry below.

Main guide: [linux-audio-moode-cleanup-guide.md](linux-audio-moode-cleanup-guide.md)

---

## v28 Change Log (2026-08-30)

This revision folds in the completed finalize run of the full library
(7,961 files / 764 albums). The embedded scripts in the main guide are the
exact tested versions from that run. Summary of changes from v27:

* **Progress lines** — Steps 8, 9, 10, 15a and 15c now display a live
  `[n/total] % complete  elapsed  ETA` line. Logs are unchanged; per-file OK
  chatter in Steps 8/10 moved to log-only so FAIL lines stay visible.
* **Step 8 — no residuals** — after a rebuilt file passes its post-reencode
  decode test and is swapped in, the `.prerepair` backup is deleted.
  Failed repairs never touch the original.
* **Step 9 — single scan + accepted variants** — the format pre-flight and
  tag verify share one file scan. "Fixing Findings" now documents the
  accepted sort-name variants (zero mismatches is no longer the target).
* **15a — surgical, not destructive** — FLAC metadata blocks are individually
  editable, so the remove-all + tag-reimport pass is gone. 15a now removes
  only SEEKTABLE and CUESHEET. Tags (incl. ReplayGain), PICTURE and PADDING
  are never rewritten; non-standard padding is logged for review only.
  Conforming files are left untouched (SAME, no rewrite, no mtime change).
* **15b — cover consolidation (was: unconditional picture re-embed)** —
  per the library owner's requirement, every album directory ends up with a
  canonical `Cover.jpg`: the highest-priority existing cover is renamed to
  `Cover.jpg` (PNGs converted at q:v 2) and every other image file in the
  directory is left in place and logged to `step15b-review.log` (nothing is
  deleted).
* **15c — resolution-gated embeds** — files whose embedded artwork is
  byte-identical to the folder cover are skipped. An existing embed is
  replaced ONLY when the folder cover is strictly higher resolution; equal
  or lower-res covers never trigger a rewrite (keeps logged to
  `step15c-review.log`). Files with no embedded art get the folder cover.
  Non-FLAC files are stream-copied, never re-encoded.
* **Closing sequence (2026-08-30, final)** — after the 37 higher-res cover
   promotions, one last 15c pass upgraded exactly those 37 albums' embedded
   artwork (0 errors, all others SAME). The final Step 10 integrity gate
   (23:24 EDT): **7,961 / 7,961 passed, 0 failed, 0 errors.** The library is
   finalized; SHA-512 checksum generation is the sole remaining step.
* **Documentation corrections (2026-09-14)** — brought the prose in line with
   the tested v28 scripts: 15a/15b descriptions rewritten for the surgical /
   consolidation behavior (no artwork destruction, nothing deleted in 15b);
   Step 8 backup caution updated for no-residual behavior; Step 3 footer
   renamed; Requirements + preflight now include `jq` and no longer require
   the eyeD3 CLI (the `eyed3` Python module is what Step 2C.3 imports);
   SHA-512 repository link updated to the canonical renamed URL.
   Script fixes: Step 2B progress total now counted with `tr -cd '\000'`
   (the old `grep -o $'\0'` idiom always yielded 0 on GNU grep); Step 15c
   review log initialized; Step 2A skips step artifacts; 15a padding review
   no longer flags files with no PADDING block at all.
* **Auto-purge added (2026-09-14)** — Step 1 now purges the log directory
  at the start of the workflow (suite auto-purge convention), so the
  previous run's logs remain reviewable until the next run replaces them.
  Per-step log resets are unchanged; steps run individually still reset
  only their own logs.

---

## v29 Change Log (2026-09-15)

Policy change behind this revision: no long-running script may run
silently. Every per-file loop that takes minutes on a large library now
shows the same in-place progress line used by Steps 8, 9, 10, 15a and 15b
since v28:

    [n/total] % complete  elapsed HH:MM:SS  ETA HH:MM:SS

* **Step 1 cache pre-warm** — the single `xargs cat` read pass was
  converted to a per-file loop over a metadata-only file list, so a live
  `Pre-warming: n/total files (pct%)` counter can be shown. Read behavior
  is unchanged (one sequential pass, same exclusions, same
  `step01-prewarm-errors.log`); the SUMMARY line now also reports the
  number of files read.
* **Steps 2C.2, 2C.3, 2C.4, 2C.5 and 2D** — these scripts previously
  redirected every per-file status line to the logs only (`tee ... 
  >/dev/null`), leaving the terminal blank for the whole run. Each now
  displays the shared `[n/total] % complete / elapsed / ETA` progress line
  during the loop. Per-file log lines, log paths, counts and summaries are
  unchanged; the counter goes to stderr and is suppressed when stderr is
  not a terminal (so log files never receive `\r` control characters).
* The counter clears its own line with `\033[K` and the scripts print a
  final newline after the loop, so all footers land exactly as before.
* All embedded scripts re-verified with `bash -n` after the edits.

---

## v30 Change Log (2026-09-16)

* **Documentation only — no script changes.** Added a note to the
  Introduction (Preamble) explaining `.prerepair` files: they are
  intentional safety copies created by Step 3A (Container Rebuild backs
  up every file as `FILE.prerepair` before overwriting it with the
  rebuilt container) and deleted by Step 7 (Remove Loose Files). While
  they exist they are skipped by every other step (integrity tests, tag
  work, ReplayGain, checksums), must not be deleted manually, and must
  not be included in SHA-512 manifests. The note also flags the
  temporary ~2x disk-space footprint while the backup layer exists, and
  clarifies that the v28 "no residuals" caution covers Step 8's
  self-deleting backups only — not Step 3A's.
* The same explanation was mirrored in the per-repo `README.md`.
* Motivation: during the 2026-09 walkthrough the full `.prerepair` set
  (7,961 files, ~254 GB, 763 albums) was initially mistaken for residue
  from an old version; the lifecycle is by design.

---

## v31 Change Log (2026-09-16)

Complete-update revision implementing the accumulated wish list. Summary
of changes from v30:

* **Step 5 — Ignore-folder bug fixed (functional).** The directory scan
  excluded `*/Ignore/*`, which skips files *under* Ignore but not the
  `Ignore` folder itself — album-nested Ignore folders with direct audio
  were counted as albums and processed (observed 2026-09-15: 784
  processed = 763 albums + 21 Ignore folders). Step 5 and 15b now use
  `! -ipath '*/Ignore/*' ! -ipath '*/Ignore' ! -iname 'Ignore'`, so
  Ignore content is fully outside the ReplayGain and cover-consolidation
  passes. (ReplayGain tags written to ignored files on 2026-09-15 were
  metadata-only; no audio changed.)
* **Preflight disk-space check (functional, high priority).** The
  Section 02 preflight now measures the audio total in the run root and
  the free space on that filesystem, prints both on screen
  (`Library audio size: N GB / Free space: N GB`), and adds a loud
  WARNING to the result if free space is insufficient — because Step 3
  duplicates the library as `.prerepair` backups until Step 7 removes
  them. Calculated per library, nothing hardcoded. Prose added to the
  preflight description and cross-referenced from the Introduction's
  `.prerepair` note.
* **Step 9 — UNPARSEABLE path fallback.** The mismatch log can never
  contain a pathless entry: if the relative-path strip yields an empty
  string, the full source path is used. (Observed 2026-09-16: a
  `UNPARSEABLE||` entry with an empty path field from a non-canonical
  runtime copy.)
* **TIFF artwork support in 15b (functional).** TIFF/TIF files are now
  recognized by the directory-detection scan, the stray-promotion
  fallback, and the extras log, and convert to `Cover.jpg` at q:v 2 like
  PNGs. A successfully converted TIFF's source file is removed and the
  removal logged — moOde cannot use TIFF and the SHA-512 guide's stray
  audit would otherwise flag it forever. Prose updated.
* **Step 2C.6 — per-format breakdown and output order.** The summary now
  reads the per-format counters written by Steps 2C.2–2C.5
  (`STEP02C_FLAC_*`, `MP3_*`, `M4A_WV_*`, `VORBIS_*`) and prints a
  per-filetype table (OK/clean, FAIL, REVIEW) in both the summary log and
  the terminal. The recap now appears ABOVE the footer (footer strictly
  last, per suite convention), is restyled with the standard 40-dash
  dividers, and is retitled "Step 2C Summary Review" (the footer remains
  "Step 2C.6 - Summary").
* **Step 3 label alignment.** The run script's guide divider is now
  `Bash Script Step 3` (the script always self-labeled "Step 3") and the
  log-viewer is `Bash Script Cat 3` with content "Step 3 – View Log
  Results" — matching every other step's unlettered pattern. Prose
  references updated.
* **Screen style alignment (cosmetic).** Step 3 (container rebuild) now
  breaks with a blank line on album change (Step 4 style). Step 5 breaks
  on ARTIST change (user preference). The full-library counter steps
  (2C.2–2C.5, 2D, 8, 9, 10, 15a, 15b, 15c) now print an album header —
  `── Artist/Album ──` (stderr) — whenever the loop enters a new album,
  clearing the counter line first so the header stays readable; the
  counter resumes on the next line. Step 10's former blank-line break was
  upgraded to the same header. All per-file FAIL lines remain terminal
  visible; per-file OK lines remain log-only in the counter steps.
* All embedded scripts re-verified with `bash -n`; preflight disk-space
  logic functionally tested.

---

## v32 Change Log (2026-09-17)

* **New optional procedure 15d — Ignore-Content Certification.** Closes
  the last unprotected corner of the library: the audio inside `Ignore`
  folders (library files hidden from moOde via `.mpdignore` — odd tracks,
  blanks, spoken word, extra-long recordings; ~60 files, ~0.9 GB across
  21 folders here). For each Ignore folder (any depth, nested
  Ignore-in-Ignore skipped) 15d:
  - verifies the existing `Ignore.sha512sums.txt` if present, or creates
    it when missing (hashing every regular file in the folder except the
    manifest itself, relative paths, sorted);
  - integrity-tests every audio file by full decode (`flac -t` for FLAC,
    `ffmpeg` null-decode for others) — these files were never
    decode-tested before, since Steps 1/4/10 exclude Ignore content;
  - never modifies audio, never deletes anything.
* **Screen/log conventions** match the v31 standard: suite progress
  counter with elapsed/ETA (terminal-only, stderr), per-folder album
  headers (`── Artist/Album/Ignore ──`), per-file OK lines log-only and
  FAIL lines terminal-visible, recap above a strictly-final footer,
  standard `Cat for 15d` log-viewer block.
* **Suite notes added:** pipeline table gains a 15d row; prose mentions
  "(15a–15c)" updated to "(15a–15d)". Regeneration rule documented
  (delete a folder's manifest and re-run 15d after an intentional
  change). The SHA-512 guide (v15) accepts `Ignore.sha512sums.txt` as a
  third generic manifest name in its stray audit, rogue-name check and
  missing-manifest check.
* Context: the SHA-512 guide's Step 6 audit flagged exactly 21
  directories missing manifests — these were the 21 Ignore folders;
  15d fills that gap with self-contained per-folder manifests.
* All embedded scripts re-verified with `bash -n`.

---

## v33 Change Log (2026-09-20)

* **Step 2C fully converted to Python (no-.sh policy).** All six embedded
  sub-step scripts — 2C.1 Initialize, 2C.2 FLAC Auto-Fix, 2C.3 MP3
  Auto-Fix, 2C.4 M4A/MP4/WavPack Review Flag, 2C.5 OGG/Opus Auto-Fix,
  2C.6 Summary — are now extensionless Python with a
  `#!/usr/bin/env python3` shebang. Behavior, log files, screen
  conventions (stderr progress counter with elapsed/ETA, per-album
  `── Artist/Album ──` headers, log-only OK lines, recap footer) match
  the v31/v32 screen-and-log standard.
* **Opus tool invocations fixed for opustags 1.9.x.** The previous bash
  2C.5 used `opustags -l FILE` (reading) and `opustags -s FILE -w FILE`
  (writing), which are not valid options in opustags 1.9.0. The Python
  version reads comments with a bare `opustags FILE` invocation and
  writes the deduplicated set with `opustags -S -i FILE` (import comments
  from standard input, in place). This makes the Opus path executable
  against the current opustags release for the first time.
* **MP3 dedup runs in-process.** 2C.3 imports `eyed3` directly instead of
  spawning `python3 - file` per MP3; the return-code protocol (0/1 OK,
  2 REVIEW, 3/4 FAIL) is unchanged.
* **Verification.** All six scripts pass `py_compile`. End-to-end fixture
  test (6 fabricated files: FLAC/MP3/OGG/Opus/M4A/WV, isolated HOME, real
  duplicate tags injected into FLAC, MP3, OGG and Opus): duplicates were
  removed and every rewritten file passed its decode check; M4A/WV
  exercised the clean path (those containers' writers refuse duplicate
  writes by design, matching the review-flag rationale). The prior
  bash-era 2C draft scripts in ~/Downloads are superseded by this
  conversion.
* **Versioned copy** — the prior guide (v32) was archived as
  `linux-audio-moode-cleanup-guide-v32.md` before editing, per the
  update rule.

---

## v34 Change Log (2026-09-20)

* **Steps 2A and 2B converted to Python (no-.sh policy).**
  - 2A File Discovery is now `step2a-discovery`, 2B Format Assessment is
    now `step2b-assessment` (extensionless Python). Same find semantics
    (Ignore-path exclusion, prerepair/fixed/reencode exclusion, the same
    15 audio extensions), same five-file log standard, same summary keys
    (TOTAL_CANDIDATES / STATUS=OK / per-format counts / TOTAL /
    DEDUPE_CAPABLE / REVIEW_CAPABLE / UNSUPPORTED), same interactive
    error inspector (Press ENTER, less -R).
  - **Conversion bug caught in testing:** the Python find invocation
    initially glued the `!` operator to the `-ipath`/`-iname` flags as a
    single argument, which made find silently match nothing; caught by
    the fixture test and fixed before commit.
* **Fixture test** (isolated HOME): 8 files planted including an
  `Ignore/` subfolder and a `.prerepair` copy — 2A correctly returned 6
  candidates, 2B classified 3 dedupe / 2 review / 1 unsupported, and
  2C.1 consumed the produced candidate list unchanged (6 found).
* **Versioned copy** — the prior guide (v33) was archived as
  `linux-audio-moode-cleanup-guide-v33.md` before editing, per the
  update rule.

---

## v35 Change Log (2026-09-20)

* **Steps 2D and 2E converted to Python (no-.sh policy).** With this
  change the entire Step 2 suite — 2A File Discovery, 2B Format
  Assessment, 2C.1-2C.6 Deduplication, 2D Verification, 2E Summary —
  consists exclusively of extensionless Python scripts.
  - 2D is `step2d-verify`: FLAC files verified with `flac -t -s`,
    every other format with the `-nostdin` ffmpeg decode-to-null check
    (the ffmpeg input-stream-sharing bug the bash version documented
    is preserved as a comment; subprocess pipes make it moot but the
    flag is kept). Same five-file logs, stderr progress counter with
    elapsed/ETA, per-album headers, interactive Y/N log-dump prompts,
    and the recap footer.
  - 2E is `step2e-summary`: concatenates the step02a-02d summary logs
    with `[sub]` headers into `step02e-summary.log`, printing the
    result to screen; missing summaries are logged as errors, exactly
    as before.
* **Fixture tests** (isolated HOME, 6-file library): 2D passed all 6;
  a deliberately truncated FLAC added to the candidates was flagged
  FAIL (6 passed / 1 corrupt). 2E aggregated all four sub-summaries
  into the combined Step 2 report.
* **Versioned copy** — the prior guide (v34) was archived as
  `linux-audio-moode-cleanup-guide-v34.md` before editing, per the
  update rule.

---

## v36 Change Log (2026-09-20)

* **Step 1 converted to Python (no-.sh policy).** `step1-integrity`
  replaces the bash integrity test. Preserved behaviors:
  - **AUTOPURGE** — removes the entire suite log root at start of run
    (Step 1 is the workflow entry point, so it owns the purge).
  - **Optional cache pre-warm** prompt (stdin-tty only): sequential
    read pass over the library with a live per-file counter, logging to
    `step01-prewarm.log` / `step01-prewarm-errors.log`, manifest files
    excluded from the warm pass.
  - Same `find` exclusions and extension set as the bash version
    (12 extensions; note 2A's list intentionally differs and was kept
    as-is per format).
  - Per-file `OK/FAIL [i/total]` lines print to screen (unlike the
    log-only 2-series convention — preserved deliberately) plus a
    blank-line album separator on folder change.
  - The awk LOST_SYNC / END_OF_STREAM error grouping is replicated in
    Python with identical line parsing and sorted output.
  - Summary: Step/Run Date/Processed/Passed/Failed.
* **Fixture test** (isolated HOME, 7 files incl. one truncated FLAC):
  6 passed / 1 failed, the truncation surfaced in the Error Summary
  under END_OF_STREAM with the relative path, summary tallies correct.
  Pre-warm prompt skipped cleanly on non-tty stdin.
* **Versioned copy** — the prior guide (v35) was archived as
  `linux-audio-moode-cleanup-guide-v35.md` before editing, per the
  update rule.
