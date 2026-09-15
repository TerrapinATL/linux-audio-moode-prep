### linux-audio-moode-cleanup-guide — Change Log

All version changes are appended to this file, newest last, one `## vX.Y Change Log` section per version.

**Update rule:** before writing to the version-less main guide file, the current content must first be saved as a versioned copy (e.g. `linux-audio-moode-cleanup-guide-v28.md`) so every published version stays retrievable.

**Suite convention (auto-purge):** every guide/repo with error logging must purge its log directory at the START of the workflow (first step), so the previous run's logs remain reviewable until the next run replaces them. This applies to all current and future repositories.

**Current version: v29** — supersedes v28. Adds live progress counters to
the long-running scripts that previously ran silently (Step 1 cache
pre-warm, Steps 2C.2–2C.5 and 2D). See the v29 entries below.

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
