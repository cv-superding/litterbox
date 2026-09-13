---
name: litterbox
description: |
  Workspace hygiene for coding agents: classify AI-generated junk files and old
  file versions, then MOVE them — never delete, you may not have permission —
  into -Delete/ and -Backup/ folders at the workspace root, each with a manifest
  for audit and restore. Keep dependency installs inside the project (venv,
  node_modules) and ledger anything that leaked into global environments with
  uninstall commands. Use when finishing any task that created scratch files,
  before committing, when the user mentions cleaning up, tidying, junk, temp
  files, a messy project, environment pollution, global installs, or venvs, or
  whenever you notice litter you created piling up — even if the user did not ask.
license: MIT
metadata:
  version: "1.2.0"
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

## Environments: install inside the project

Dependency installs are the most persistent litter an agent produces — they outlive every file in the workspace and pollute a machine the agent does not own. But not everything is a dependency: sometimes the runtime itself (Python, Node, a build toolchain) is missing. The two cases follow different rules.

### Packages: always project-local

1. **Create or use a project-local environment before installing any package.** Python: `python -m venv .venv` at the workspace root, then install into it. Node: install against the project's own `package.json` so packages land in its `node_modules/`. Never `pip install --user`, never `npm install -g`, never install into the interpreter or prefix you happened to find on the machine.

2. **Write the project's dependency manifest before installing.** Create `requirements.txt` / update `package.json` first, then install — the environment stays reproducible and can be deleted without losing information. An installed-but-unrecorded dependency is a landmine for the next person.

3. **If a normal local install is impossible** (read-only workspace, permission walls), fall back to `pip install --target ./.deps` — still inside the project, still one removable folder — and note the required `PYTHONPATH` in the report.

### Runtimes: detect, offer the choice, conflict-check

A missing runtime (Python on a machine that has none, Node, a compiler) cannot always live inside the project — and unlike a package, a wrong runtime install breaks other people's work too. So:

1. **Detect before proposing.** Check what already exists: `python` / `python3` / `py` and their versions, `node`, plus version managers (`pyenv`, `nvm`, `conda`, `asdf`) that own those names. A "missing Python" that is really a PATH-shadowed Python is a configuration fix, not an install.

2. **Offer the user the choice — one question, with your recommendation:**
   - **Project-local** (default recommendation): a portable/standalone runtime or version-managed environment inside or beside the workspace. Nothing on the machine changes; deleting the project removes it.
   - **Global** (system installer / package manager): for runtimes the user wants to reuse outside this project. Proceed only on an explicit yes.
   Do not proceed on silence. "It's the obvious choice" is not consent — it is exactly the guess this skill exists to stop.

3. **Conflict-check before any global install; if a conflict exists, project-local is the only option.** Look for: an existing runtime of the same name on PATH (a version a global install would shadow or upgrade), a version manager that owns the name, an ecosystem pinned to the current version. State what you found; when a conflict exists, say "global would conflict — installing project-locally" and do that. Two Pythons fighting over `python` is a machine-level bug you must not introduce.

4. **Ledger a global runtime install** like any other machine change: what, where, and the uninstall command.

## Regenerables: never archive, always record

`.venv/`, `node_modules/`, `__pycache__/`, `dist/`, `build/`, tool caches: too big to move, rebuilt from manifest files. Never archive them into the buckets. Instead: (1) confirm the lockfile or requirements file exists so they are truly regenerable, (2) add the matching `.gitignore` entry if missing, (3) report them with sizes as "regenerable — safe to wipe" and let the human decide. Wipe only when the user asks; a permission failure goes to the report, not to a force-flag retry.

## Secrets check

Debug dumps, `.env` copies, and log files often contain tokens — `sk-…`, `ghp_…`, connection strings, passwords. Before archiving such a file, scan it. If it holds secrets, mark the manifest line `CONTAINS SECRETS — empty bucket soon` and say so in the report. Never reproduce the secret values in the report.

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
   - `debug_env.txt` — **CONTAINS SECRETS** (1 token pattern) — empty bucket soon
   - installed outside the project: `httpie 3.1.3` — `pip uninstall -y httpie` — **needs manual uninstall**
   ```

5. **Report in this order: archived → backed up → needs manual deletion → left alone (and why).** Short table, no narration.

Shell tip: a leading dash confuses argument parsing. Use `./`: `mv scratch_test.py ./-Delete/`.

## Never move

`.git/` and other VCS internals · agent/tool config dirs (`.claude/`, `.cursor/`, `.agents/`) · `node_modules/`, `venv/`, `__pycache__/` and other regenerables (record, don't archive) · lockfiles (`package-lock.json`, `uv.lock`, `poetry.lock`) · `LICENSE`, `README.md`, CI workflows · shared caches outside the workspace (`~/.cache`, pip/npm caches — the machine shares them) · anything outside the workspace root — except litter you created there yourself this session (a stray download in `~/Downloads`), which you may move into `-Delete/` if permitted, else report the path · anything the user recently changed by hand. When in doubt: leave it, list it in the report as "left alone (why)".

## Anti-patterns

Numbered strongest first. §1, §4, and §6 justify stopping yourself on a single sighting.

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

### §6. The global install

**Watch for:** `pip install` into the system or user interpreter; `npm install -g`; installing into conda base; installing anything without a `requirements.txt` / `package.json` entry; treating a machine-level environment as personal scratch space; installing a missing runtime globally without checking what already owns that name.
**Problem:** This litter is invisible and outlives everything. The next project inherits your experiment; the machine's Python is now state nobody documented; the "temporary" tool stays forever because nobody remembers installing it.
**Before:**
> pip install httpie — ok, tool installed, moving on.
**After:**
> Created .venv, added httpie to requirements.txt, installed locally. (Missing runtime? Detect, conflict-check, offer the choice — see *Runtimes* above. Ledger anything that does go global.)

## Restore

Read `MANIFEST.md`, copy back: `cp -r "./-Backup/2026-09-13_1542/src/config.yaml" src/config.yaml`. For `-Delete/`, the manifest's original path is the destination. Restore is a normal file operation — no special tooling, no lock-in.

## What to return

**End-of-task sweep (default).** A short table: archived to -Delete (count + notable names) · backed up to -Backup (count + what was snapshotted) · installed outside the project (packages + uninstall commands) · regenerable and safe to wipe (names + sizes) · needs manual deletion (paths, permission errors) · left alone (names + one-line why). Nothing else.

**User asks for a restore.** The manifest lines matching their description, the exact copy commands, and a confirmation of what was restored.

**Nothing to clean.** Say "no litter found" — do not create empty buckets for the sake of it.
