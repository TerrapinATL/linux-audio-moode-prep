### linux-audio-moode-cleanup-guide — v28 Change Log

**Version: v28 — FINAL** — Current version; supersedes v27. The full pipeline
has been run end-to-end against the production library and verified clean.
Remaining user action: SHA-512 checksum generation (separate repo).
(Prose and script corrections applied 2026-09-14 — see the last entry below.)

Main guide: [linux-audio-moode-cleanup-guide-v28.md](linux-audio-moode-cleanup-guide-v28.md)

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
