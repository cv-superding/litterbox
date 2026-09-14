---
name: litterbox
description: |
  Workspace hygiene for coding agents: classify AI-generated junk files and old
  file versions, then MOVE them — never delete, you may not have permission —
  into -Delete/ and -Backup/ folders at the workspace root, each with a manifest
  for audit and restore. Keep dependency installs inside the project (venv,
  node_modules) and ledger anything that leaked into global environments with
  uninstall commands. Keep secrets (keys, cookies, tokens) from leaving the
  machine and keep shareable files free of absolute paths. Use when finishing
  any task that created scratch files, before committing or pushing, when the
  user mentions cleaning up, tidying, junk, temp files, a messy project,
  environment pollution, global installs, open-sourcing, multiple agent sessions
  in one project, or venvs, or whenever
  you notice litter you created piling up — including leftover dev servers and
  processes you started — even if the user did not ask.
license: MIT
metadata:
  version: "1.5.0"
---

# Litterbox: move, never delete

An agent's litter gets everywhere: scratch scripts, debug dumps, `output_final_v2.json`, superseded drafts. The sandbox decides *where* you may write, not *how tidy* you must be — and often you cannot delete at all. So don't. **Sort the litter into two buckets at the workspace root and let the human empty them.**

## The two buckets

**`-Delete/` — litter.** Files the agent created during work that the final result does not reference: one-off test scripts, debug dumps, experiment outputs, duplicated attempts (`v2`, `final`, `final2`), placeholder files you never filled.

**`-Backup/` — old versions.** A snapshot of any existing file *before* you overwrite or replace it: edited sources, replaced configs, superseded drafts. Keep the original relative path inside a session directory: `-Backup/2026-09-14_1530_a3f/src/config.yaml`.

The leading dash sorts both folders above every normal folder, so the human sees them first. If the workspace root is read-only, fall back to the nearest writable ancestor directory or `~/.litterbox/<project-name>/` — and say where the buckets ended up in the report. If the user prefers different bucket names, follow their names.

## The one rule

**Move, never delete.** Deletion is the human's decision; the manifest makes it reviewable; moving is reversible. If even a move fails with a permission error, do *not* retry with force or sudo tricks — record the path in the manifest under "needs manual deletion" and move on. A cleanup that says "done" while the litter is still in place is a lie.

## The backup habit

Backups happen **at edit time, not cleanup time**. Before an Edit/Write replaces an existing file you did not create in this session, copy the original into `-Backup/<session-id>/<original-relative-path>` first. Once you have overwritten it, the old version is gone — a cleanup pass cannot bring it back.

## Create litter in one place — per session

The cheapest cleanup is the one that needs no archaeology. IDEs run several agent sessions over the same project, and their litter ends up tangled in one directory — nobody can tell which window made what. Prevent that at birth:

- **Claim a session id at task start.** Use the harness's session id when there is one; otherwise make up a short stamp once and reuse it all session: `2026-09-14_1530_a3f`.
- **All throwaway artifacts go to your session scratch folder**: `./tmp/<session-id>/` (or the project's existing convention). Never into `src/`, never into the root, never into another session's folder.
- **Snapshots carry the id too**: `-Backup/<session-id>/<original-relative-path>` — two sessions running in the same hour stay perfectly separable.
- **At sweep time your folder is pre-classified**: it moves into `-Delete/<session-id>/` wholesale, no reference checks needed, and the manifest tags every entry with the session id so the human can see which window produced what — and empty one window's mess without touching another's.

Litter that never scattered — and never mixed across sessions — is the only kind this skill handles perfectly.

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

## Secrets: never leave the machine

Debug dumps, `.env` copies, and log files often contain tokens — `sk-…` API keys, `ghp_…` GitHub tokens, `Cookie:` / session strings, `AKIA…` AWS keys, JWTs (`eyJ…`), `-----BEGIN … PRIVATE KEY-----` blocks, connection strings with passwords. Before archiving such a file, scan it. If it holds secrets, mark the manifest line `CONTAINS SECRETS — empty bucket soon` and say so in the report. Never reproduce the secret values in the report.

The scan matters most at the **publish boundary** — the moment content leaves the machine:

- **Before every commit that will be pushed**, scan the staged diff for token patterns, cookie strings, private-key blocks, and passwords. A scan before `git push` is part of the sweep, not paranoia.
- `.env` is gitignored, always. What ships is `.env.example` with placeholder values (`API_KEY=your-key-here`), never the real ones.
- Secrets live in the environment or a local untracked file, never hardcoded in example code, README quickstarts, or test fixtures.
- **If a secret did get committed and pushed, treat it as burned.** Say it plainly: deleting the file afterwards does not help — history, forks, and scrapers keep it. Rotate/revoke the credential; that is the fix. Do not offer "I removed it from the repo" as if that solved anything.

## No absolute paths in anything that might leave the machine

`F:\Code\...`, `/Users/alice/...` — absolute paths leak the machine's layout, break on every other machine, and often leak usernames. In anything that could be published or shared — README, docs, examples, test fixtures, config samples, scripts meant for reuse — use paths relative to the project root or placeholders: `./data/seed.csv`, `${PROJECT_ROOT}/config.yaml`, `~`. Absolute paths are acceptable only in genuinely machine-bound files that stay local (a log, an untracked local config). When you catch an absolute path in a shareable file, replace it before the sweep ends — and if the script genuinely needs a machine-specific location, read it from an environment variable or a config file the user fills in.

## How to clean up

1. **Inventory your own litter first.** You know what you created this session — that is the highest-confidence list, and none of it needs a reference check. Everything else is a candidate that needs one.

2. **Classify.** Old version of something that still exists → `-Backup/`. Yours and unreferenced → `-Delete/`. Unsure → leave it and say so; a false positive costs more than a leftover file.

3. **Reference-check anything you did not create.** Search the workspace for the basename (imports, config keys, path strings, docs) before moving it. If it is referenced: leave it alone. "Looks unused" is not evidence — a file named `legacy_export.py` may be load-bearing.

4. **Move, verify, then manifest.** A move is complete only when the destination exists and the origin is gone — check both before writing anything. A directory move that dies halfway (permission errors, a locked file on Windows) must be finished or rolled back, never reported as done. Then remove parent directories the litter left empty — only ones created this session. Append one line per file to `MANIFEST.md` inside the bucket:

   ```markdown
   ## 2026-09-14 15:42 · session 2026-09-14_1530_a3f
   - `scratch_test.py` → -Delete/ — one-off test script from session work
   - `output_v1.json`, `output_final.json` → -Delete/ — superseded experiment outputs
   - `src/config.yaml` → -Backup/2026-09-13_1542/ — original before override edit
   - `/etc/nginx/conf.d/app.conf` — MOVE FAILED: permission denied — **needs manual deletion**
   - `debug_env.txt` — **CONTAINS SECRETS** (1 token pattern) — empty bucket soon
   - installed outside the project: `httpie 3.1.3` — `pip uninstall -y httpie` — **needs manual uninstall**
   ```

5. **Report in this order: archived → backed up → needs manual deletion → left alone (and why).** Short table, no narration.

Shell tip: a leading dash confuses argument parsing. Use `./`: `mv scratch_test.py ./-Delete/`.

Another session is active in the same workspace? Each of you has a session id and a scratch folder — sweep only yours and leave everything else. You cannot see the other agent's in-flight references, and with per-session folders you should never have to.

## Keep the buckets out of git

The buckets are a local safety net, not repo assets. During a project's first sweep, make sure `.gitignore` covers `-Delete/` and `-Backup/` — add the entries if missing. Committed litter is worse than litter: it lives in history even after the folder is emptied.

## Litter isn't only files

Sessions also leave *running* things behind: the dev server on :8000, the test database container, a watch process. At sweep time, report what **you started** that is still running, with the command to stop it — and stop them when the user asks. Things that were already running before your session are not yours: leave them alone and don't list them.

## Never move

`.git/` and other VCS internals · agent/tool config dirs (`.claude/`, `.cursor/`, `.agents/`) · `node_modules/`, `venv/`, `__pycache__/` and other regenerables (record, don't archive) · lockfiles (`package-lock.json`, `uv.lock`, `poetry.lock`) · `LICENSE`, `README.md`, CI workflows · shared caches outside the workspace (`~/.cache`, pip/npm caches — the machine shares them) · anything outside the workspace root — except litter you created there yourself this session (a stray download in `~/Downloads`), which you may move into `-Delete/` if permitted, else report the path · anything the user recently changed by hand. When in doubt: leave it, list it in the report as "left alone (why)".

## Anti-patterns

Numbered strongest first. §1, §4, §6, and §7 justify stopping yourself on a single sighting.

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

### §7. The published secret

**Watch for:** `git add -A && git push` with a `.env` or config holding live credentials; hardcoded `api_key="sk-…"` in example code; a real token pasted into a README quickstart; "I pushed it by mistake but deleted the file right away"; a cookie string copied from the browser into a debug script that gets committed.
**Problem:** A pushed secret is a burned secret. Deletion, force-push, and "it was only up for a minute" do not help — history, forks, and automated scrapers that watch new commits keep it. The fix is rotation, and the cost is downtime plus someone's quota.
**Before:**
> git add -A && git push  *(config.js still holds the live API key)*
**After:**
> Scanned the staged diff — caught the key. It moved to `.env` (gitignored), code reads `process.env.API_KEY`, README ships `.env.example` with a placeholder. Ready to push.

## Restore

Read `MANIFEST.md`, copy back: `cp -r "./-Backup/2026-09-14_1530_a3f/src/config.yaml" src/config.yaml`. For `-Delete/`, the manifest's original path is the destination.

**Check the destination first.** If the file has changed since the snapshot — newer edits, a newer backup, anything — snapshot the current version into `-Backup/<your-session-id>-restored/` before restoring, so a restore can never destroy newer work. Then verify what you restored: run it, open it, or diff it. Restore is a normal file operation — no special tooling, no lock-in.

**Backup aging.** If `-Backup/` holds many snapshots of the same file, or snapshots older than a month, say so in the sweep report and suggest pruning to the newest. The human decides; you never prune on your own.

## What to return

**End-of-task sweep (default).** A short table: archived to -Delete (count + notable names) · backed up to -Backup (count + what was snapshotted) · still running (processes you started + stop commands) · installed outside the project (packages + uninstall commands) · regenerable and safe to wipe (names + sizes) · needs manual deletion (paths, permission errors) · left alone (names + one-line why). Nothing else.

**User asks for a restore.** The manifest lines matching their description, the exact copy commands, and a confirmation of what was restored.

**Nothing to clean.** Say "no litter found" — do not create empty buckets for the sake of it.
