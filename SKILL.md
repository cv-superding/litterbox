---
name: litterbox
description: |
  Workspace hygiene for coding agents: classify AI-generated junk files and old
  file versions, then MOVE them — never delete, you may not have permission —
  into -Delete/ and -Backup/ folders at the workspace root, each with a manifest
  for audit and restore. Use when finishing any task that created scratch files,
  before committing, when the user mentions cleaning up, tidying, junk, temp
  files, or a messy project, or whenever you notice litter you created piling up —
  even if the user did not ask.
license: MIT
metadata:
  version: "1.0.0"
---

# Litterbox: move, never delete

An agent's litter gets everywhere: scratch scripts, debug dumps, `output_final_v2.json`, superseded drafts. The sandbox decides *where* you may write, not *how tidy* you must be — and often you cannot delete at all. So don't. **Sort the litter into two buckets at the workspace root and let the human empty them.**

## The two buckets

**`-Delete/` — litter.** Files the agent created during work that the final result does not reference: one-off test scripts, debug dumps, experiment outputs, duplicated attempts (`v2`, `final`, `final2`), placeholder files you never filled.

**`-Backup/` — old versions.** A snapshot of any existing file *before* you overwrite or replace it: edited sources, replaced configs, superseded drafts. Keep the original relative path inside a timestamp directory: `-Backup/2026-09-13_1542/src/config.yaml`.

The leading dash sorts both folders above every normal folder, so the human sees them first.

## The one rule

**Move, never delete.** Deletion is the human's decision; the manifest makes it reviewable; moving is reversible. If even a move fails with a permission error, do *not* retry with force or sudo tricks — record the path in the manifest under "needs manual deletion" and move on. A cleanup that says "done" while the litter is still in place is a lie.

## The backup habit

Backups happen **at edit time, not cleanup time**. Before an Edit/Write replaces an existing file you did not create in this session, copy the original into `-Backup/<timestamp>/<original-relative-path>` first. Once you have overwritten it, the old version is gone — a cleanup pass cannot bring it back.

## How to clean up

1. **Inventory your own litter first.** You know what you created this session — that is the highest-confidence list, and none of it needs a reference check. Everything else is a candidate that needs one.

2. **Classify.** Old version of something that still exists → `-Backup/`. Yours and unreferenced → `-Delete/`. Unsure → leave it and say so; a false positive costs more than a leftover file.

3. **Reference-check anything you did not create.** Search the workspace for the basename (imports, config keys, path strings, docs) before moving it. If it is referenced: leave it alone. "Looks unused" is not evidence — a file named `legacy_export.py` may be load-bearing.

4. **Move, then manifest.** Append one line per file to `MANIFEST.md` inside the bucket:

   ```markdown
   ## 2026-09-13 15:42
   - `scratch_test.py` → -Delete/ — one-off test script from session work
   - `output_v1.json`, `output_final.json` → -Delete/ — superseded experiment outputs
   - `src/config.yaml` → -Backup/2026-09-13_1542/ — original before override edit
   - `/etc/nginx/conf.d/app.conf` — MOVE FAILED: permission denied — **needs manual deletion**
   ```

5. **Report in this order: archived → backed up → needs manual deletion → left alone (and why).** Short table, no narration.

Shell tip: a leading dash confuses argument parsing. Use `./`: `mv scratch_test.py ./-Delete/`.

## Never move

`.git/` and other VCS internals · agent/tool config dirs (`.claude/`, `.cursor/`, `.agents/`) · `node_modules/`, `venv/`, `__pycache__/` · lockfiles (`package-lock.json`, `uv.lock`, `poetry.lock`) · `LICENSE`, `README.md`, CI workflows · anything outside the workspace root · anything the user recently changed by hand. When in doubt: leave it, list it in the report as "left alone (why)".

## Anti-patterns

Numbered strongest first. §1 and §4 justify stopping yourself on a single sighting.

### §1. The brave delete

**Watch for:** `rm -rf` on pattern guesses; `rm ... 2>/dev/null` that hides permission failures; declaring cleanup done when the error output says otherwise.
**Problem:** Deletion is irreversible and you are working from guesses about what matters. A hidden permission failure turns "done" into a lie the user discovers next week.
**Before:**
> rm -rf scratch* debug* *.bak 2>/dev/null && echo "cleaned up"
**After:**
> Moved 3 files to -Delete/, logged in MANIFEST.md. `debug_dump.txt` could not be moved (permission denied) — needs manual deletion.

### §2. The hoard-and-hope

**Watch for:** leaving litter scattered because "we might need it"; ending the task with `test_final2.py` sitting in the repo root; "the user can delete it".
**Problem:** The user should not have to archaeology your session to find the project. A leftover in `-Delete/` costs nothing — it is reviewable and reversible; a leftover in the root is noise forever.
**Before:**
> I'll leave those three test scripts in the root in case we need them later.
**After:**
> The three scripts are in -Delete/ with a manifest entry; restore any of them with one copy command.

### §3. The quiet overwrite

**Watch for:** writing a new `config.yaml` over the old one with no snapshot; replacing a whole file when a targeted edit would do; "the old version is in git" — it may not be committed, and this may not be a git repo.
**Problem:** Cleanup cannot restore what was never backed up. The pre-overwrite moment is the only chance.
**Before:**
> Rewrote src/config.yaml with the new settings.
**After:**
> Snapped the original to -Backup/2026-09-13_1542/src/config.yaml, then rewrote it.

### §4. The false positive

**Watch for:** moving a "weird-looking" file without checking references; classifying by name (`legacy_*`, `*_old`) instead of by usage; touching generated-but-imported code.
**Problem:** One moved-but-referenced file breaks the build, and the cleanup that caused it is the last place anyone looks. This is the most expensive mistake a cleanup can make.
**Before:**
> Moved legacy_export.py to -Delete/ — nothing should use that anymore.
**After:**
> legacy_export.py looks abandoned, but main.py imports it — left alone and noted in the report.

### §5. The unmarked grave

**Watch for:** moving files into `-Delete/` with no manifest; a bucket folder that becomes a second, worse mess; timestamps in the report but not next to the files.
**Problem:** Without a manifest the bucket is unauditable and unrestorable — you have converted litter into a black hole. The manifest is the whole point of moving instead of deleting.
**Before:**
> All cleaned up — check the -Delete folder.
**After:**
> All cleaned up — MANIFEST.md lists each file, its origin, and the reason; restore is one copy per line.

## Restore

Read `MANIFEST.md`, copy back: `cp -r "./-Backup/2026-09-13_1542/src/config.yaml" src/config.yaml`. For `-Delete/`, the manifest's original path is the destination. Restore is a normal file operation — no special tooling, no lock-in.

## What to return

**End-of-task sweep (default).** A short table: archived to -Delete (count + notable names) · backed up to -Backup (count + what was snapshotted) · needs manual deletion (paths, permission errors) · left alone (names + one-line why). Nothing else.

**User asks for a restore.** The manifest lines matching their description, the exact copy commands, and a confirmation of what was restored.

**Nothing to clean.** Say "no litter found" — do not create empty buckets for the sake of it.
