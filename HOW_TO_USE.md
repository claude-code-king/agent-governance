# How to use this repo

## 1. What `easy_install.sh` does

Run from a clone of this repo: `bash easy_install.sh [--dry-run] [--yes] [--restore <backup-dir>]`.

Steps, in order:

1. Checks the OS (`OSTYPE`). Linux is tested. On macOS it warns that `tac` may be
   missing (`brew install coreutils`); without it, `main-model.sh` fails open instead
   of failing hard. Windows Git Bash is untested. Anything else aborts.
2. Requires `python3` on PATH (used by the metrics analyzer and by the installer's
   own JSON check).
3. Verifies the directory the script lives in is a real clone of this repo (checks for
   `agents/`, `hooks/`, `commands/`, `templates/`, etc.); aborts otherwise.
4. Prints the plan (what gets replaced, how many agents/hooks/commands) and, unless
   `--dry-run`, asks `[y/N]` to continue (skipped with `--yes`).
5. Backs up `~/.claude/{agents,hooks,commands,settings.json,CLAUDE.md,
   orchestrare*.md}` into `~/.claude-backup-<timestamp>`. If that path already
   exists (same-second re-run), it appends `-2`, `-3`, ... Aborts if the backup
   ends up with fewer entries than the source.
6. Deletes and replaces `agents/`, `hooks/`, `commands/` under
   `~/.claude`, and copies `CLAUDE.md`, `orchestrare.md`, `orchestrare-v17.md`,
   `settings.json` from `templates/` and `hooks/settings.example.json`.
   `~/.claude/skills/` is not touched.
7. Rewrites the `REPO_DIR=` line inside the installed `hooks/session-metrics.sh`
   to point at this clone (or at `$AGENT_GOVERNANCE_DIR` if you set it), so
   `TRENDS.md` generation finds the right repo later.
8. Validates the installed `settings.json` is valid JSON.
9. Prints how many agents/hooks/commands landed, and warns if `orchestrare.md` is over
   10,240 bytes (above that, SessionStart truncates the injected block).

`--dry-run` only prints the plan, writes nothing. `--restore <backup-dir>` reverses
step 6: it asks `[y/N]` first (skipped with `--yes`), then deletes the current
`agents/hooks/commands` and top-level files under `~/.claude`, copies everything from
the backup dir back in, and exits — it does not run the rest of the install. A backup
made before this version may contain `skills/`; `--restore` does not put it back.

**Left untouched, always**: `~/.claude/memory/`, `~/.claude/projects/`,
`~/.claude/plans/`, any `history*` files, and anything outside the
`INSTALLED_FILES`/`INSTALLED_DIRS` list above.

## 2. Why STRICT: settings that break the engine

Some `~/.claude/settings.json` edits silently disable the governance hooks and
agents this repo installs:

- Pinning a specific model for the main session: the guard hooks that check for
  model drift (`main-model.sh`) never trigger, because there's no drift to see.
- `permissions.deny` on `Bash`: the hooks that run through Bash (commit-gate,
  session-metrics, etc.) never fire.
- `disableAllHooks`: turns off every hook installed by `easy_install.sh`, engine
  and all.
- `env` limits that cap tool output: hook stdout/stderr gets truncated, so
  ask/deny decisions and session-metrics can silently lose data.
- Foreign skills or hooks with overlapping triggers (e.g. another `SessionStart`
  or `Stop` hook): they can race or shadow the ones this repo installs.

To recover: restore permissions from the backup written by the installer, either
by hand (`cp ~/.claude-backup-<timestamp>/settings.json ~/.claude/settings.json`)
or with `bash easy_install.sh --restore ~/.claude-backup-<timestamp>`.

**Careful with `--restore`**: it deletes the *current* `~/.claude` install first,
including anything you changed there after the original install (custom hooks,
manual settings edits) — not just the engine's own files. It asks `[y/N]` before
doing so, unless you pass `--yes`.

## 2b. Don't move or delete the clone after install

`easy_install.sh` points `~/.claude/hooks/session-metrics.sh` at this clone's
path. `SessionEnd` runs `tools/session_metrics.py` from there to build
`TRENDS.md`. If you move, rename, or delete the clone, that hook fails silently
and metrics stop. If you must move it, re-run `easy_install.sh`, or set
`AGENT_GOVERNANCE_DIR` to the new path and re-run the installer so the hook
picks it up.

## 3. `/onboarding`

Run once per project, after the engine is installed. It never touches
`~/.claude`. It asks up to 7 questions (language, project type/commands, what
"verified" means, reader level, commit policy, which source-of-truth docs to
create, report length), then writes `./CLAUDE.md` with three sections: `# Rules`
(which docs to read instead of exploring the codebase), `# Preferences` (from
your answers), and `# Engine freeze` (see below). If `CLAUDE.md` already exists,
it asks before appending or writing `CLAUDE.local.md` instead.

## 4. Engine freeze

Once installed, leave `~/.claude` (hooks, agents, `orchestrare.md`) untouched
for the first 5–10 sessions. Change it only on evidence from `TRENDS.md`,
produced by `python3 tools/session_metrics.py`, and never mid-session.

## 5. Platform notes

- **Linux**: tested.
- **macOS**: should work; needs `python3`. The only external command the engine
  depends on that macOS may lack is `tac` (used by `main-model.sh`); without it,
  that hook fails open rather than blocking you. Installable via
  `brew install coreutils`.
- **Windows (Git Bash)**: untested; needs `python3` on PATH.
- **WSL**: works like Linux.

## 6. Requirements

Requirements: Claude Code, Python 3.6+ (no f-strings, no walrus, no `match`; 3.7+
recommended). No dependencies, no config file needed.

Comments shaped `🔴 … — DECIZII/PATTERNS «…»` are pointers to internal decision notes
not included in this repo. The regexes with Romanian diacritics target the Romanian
wording of agent reports from the author's own system.

## 7. Manual install, without `easy_install.sh`

**Install the agents and hooks**

```sh
git clone <this-repo> ~/agent-governance
cd ~/agent-governance

cp agents/*.md   ~/.claude/agents/
cp commands/*.md ~/.claude/commands/
cp hooks/*.sh    ~/.claude/hooks/
chmod +x         ~/.claude/hooks/*.sh
```

`jq` is only needed by `hooks/test-main-guards.sh`.

Merge the `hooks` block from `hooks/settings.example.json` into `~/.claude/settings.json`.
If you cloned somewhere other than `~/agent-governance`, point the metrics hook at it:

```sh
export AGENT_GOVERNANCE_DIR=/path/to/your/clone
```

**Install the policies**

```sh
cp templates/CLAUDE.global.md  ~/.claude/CLAUDE.md      # orchestration rules
cp templates/CLAUDE.project.md /your/project/CLAUDE.md  # then fill in the brackets
```

`~/.claude/CLAUDE.md` stays small — addressing, report format, conventions any agent needs.
The orchestration rules (which agent for which task, escalation, parallelism caps) live in
`~/.claude/orchestrare.md` instead, injected by `hooks/session-start.sh` only into the main
session, never into subagents. Install: `cp templates/orchestrare.md ~/.claude/orchestrare.md`,
the hook itself into `~/.claude/hooks/`, and the SessionStart entry from
`hooks/settings.example.json` into `~/.claude/settings.json`.

## 8. Run the analyzer

```sh
python3 tools/session_metrics.py ~/.claude/projects/<project-dir>/            # all sessions
python3 tools/session_metrics.py <session>.jsonl --md   --out report.md
python3 tools/session_metrics.py <session>.jsonl --json --out report.json
python3 tools/session_metrics.py ~/.claude/projects/<project-dir>/ --json --md --out-dir metrics-local/  # per-session files
python3 tools/session_metrics.py --rename metrics-local/          # rename old <uuid>.json/.md files
```

`--out-dir DIR` writes one `<name>.json` (with `--json`) and one `<name>.md` (with `--md`)
per session into `DIR`. `--rename DIR` renames old `<uuid>.json`/`.md` files in `DIR` to the
new naming scheme, skipping collisions unless `--force` is given. `--ctx-warn N` sets the
threshold for the `high_context_end` flag (default 150000). Files are named
`YYYY-MM-DD-HHMM-<project>.json`/`.md` (local start date and time, `HHMMSS` if another
session started the same minute, `<project>` = basename of the working directory);
`--migrate-names DIR` renames files left over from the old `-sN-` scheme (dry-run without
`--yes`).

Run it on a single transcript and check the tests:

```sh
python3 tools/session_metrics.py --md ~/.claude/projects/<project-dir>/<session>.jsonl
python3 tools/tests/test_v17.py     # expected: 0 failed
```

`tools/pricing.json` is optional: without it every cost is reported as `0.0`, one WARN line
goes to stderr, and all other metrics are unaffected. On a session with no subagents and no
governance hooks the `## v1.7` block (effort phases, advisor, low phase) is empty, while the
generic metrics — turns, tool calls, context, reads, inefficiencies, postmortem — are all
filled in.

The `.md` report includes
the summary plus the "Inefficiencies" list, an automatic "Postmortem" section with severity
and recommendations, and a Fable-only cost estimate. `--trends DIR` regenerates
`DIR/TRENDS.md`, a cross-session view of recurring inefficiencies. It opens with an
executive summary per corpus and per version: spend, savings against the Fable-only
realistic estimate ($ and %, cumulative across versions), estimated waste in tokens and as
% of main input volume, waste grouped into families (reads, agent overhead, orchestration
turns, discipline), and deltas against the previous version and against `older`. Sessions where at least
50% of the main session's tool calls are `mcp__claude-in-chrome__*` are listed separately
under "Excluded" and do not count toward the numbers (threshold: `--browser-threshold`,
default 0.5; version list: `--versions PATH`). `/rate N [note]` before closing a session
attaches a 1-5 quality rating to it (via the SessionEnd hook); `--rate NAME N` rates a
session after the fact; TRENDS shows the mean quality per version.

Nothing here calls a network service or a model. `metrics-local/` is gitignored so raw
session data never leaves the machine.

