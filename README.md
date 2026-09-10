# agent-governance

Governance for Claude Code agent sessions: policies, enforcement hooks, and offline
telemetry. Cheap models do the work, the expensive model only plans and audits, and hooks
stop verbose agents from flooding the orchestrator's context.

Current version: **v1.8.2**. The governance rules are frozen at v1.8; v1.8.1 and v1.8.2
changed only the analyzer and the effort gate hook. v1.8 is the current stable ruleset,
validated on 19 product sessions from 2026-09-05 onward; see Measured results. Per-version
history is in [`CHANGELOG.md`](CHANGELOG.md).

## The problem

In a multi-agent Claude Code setup, the expensive orchestrator model burns its budget on
things that are not thinking:

- **Verbose agent reports.** A subagent finishes and hands back a 5,000–11,000 character
  essay. All of it lands in the orchestrator's context, permanently.
- **Fat tool results.** One `Bash` call that runs `git diff` and `cat`s a whole file: 21,000
  characters. One `Read` of a `tool-results/` directory: 31,000 characters — re-paying for
  a result already seen once. (The analyzer's own worst `Read` payload in that corpus is
  624,414 characters, mostly image data.)
- **Everything is re-paid.** Context is re-sent as cache reads on every following message.
  In the measured corpus, cache-read tokens ran ~155× the output tokens. A large payload
  read once is charged for the rest of the session.

Telling a model "keep it to 25 lines" does not work. It was in the instructions and it was
ignored systematically, because nothing enforced it.

## How it works

The orchestrator plans, writes briefs, audits diffs and reports. It does not write code over
~20 lines, does not run commands with large output, does not read whole files. Everything
else runs in a disposable subagent context, on the cheapest model that can do the job, and
only a ≤1,500-character report crosses back. The diagram of one full round —
brief → implementer → audit → repair → report — is in
[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) «The flow».

Three mechanisms carry it:

- **Policy** — `templates/`: who does what, which agent for which brief, escalation rules,
  the caps on reading and parallelism. Installed as `~/.claude/CLAUDE.md` and
  `~/.claude/orchestrare.md`.
- **Enforcement** — `hooks/`: hooks that block instead of asking. A report over 2,000
  characters is refused, a 300+ line read without `offset`/`limit` is denied, a subagent's
  own context is capped, comment blocks are flagged on every write.
- **Telemetry** — `tools/`: a `SessionEnd` hook runs a plain-Python analyzer over the
  session transcript and writes a local JSON/Markdown report. No model call, no tokens, no
  network.

Full rules and the reasoning: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## Measured results

From `metrics/trends-2026-09.md`, generated 2026-09-10 over the product split, 67
sessions, from a 125-session corpus. `older` = sessions before v1.0, no governance
rules applied yet.

| version | sessions | $/session | saved % (vs Fable-only) | wasted tok % | issues/session (H/M/L) | quality | effort |
|---|---:|---:|---:|---:|---:|---:|---|
| older | 13 | $23.54 | 73.8% | 4.5% | 19.2 (2.5/11.2/5.5) | — · 0/13 | high 13 |
| v1.0 | 7 | $31.12 | 79.5% | 5.3% | 23.7 (5.0/13.4/5.3) | — · 0/7 | high 7 |
| v1.1 | 10 | $28.03 | 79.6% | 4.7% | 16.7 (3.2/8.6/4.9) | 4.5 · 2/10 | high 10 |
| v1.2 | 10 | $24.19 | 81.4% | 4.5% | 17.7 (3.5/10.2/4.0) | 4.2 · 6/10 | high 10 |
| v1.4 | 2 | $28.12 | 88.3% | 10.2% | 22.5 (5.5/15.0/2.0) | 5.0 · 1/2 | high 2 |
| v1.5.2 | 1 | $3.85 | 8.3% | 3.1% | 3.0 (2.0/1.0/0.0) | — · 0/1 | high 1 |
| v1.7 | 4 | $25.31 | 87.7% | 3.3% | 19.2 (3.2/11.5/4.5) | 4.3 · 3/4 | medium 2 · low 1 · high 1 |
| v1.7.5 | 1 | $57.72 | 89.6% | 0.8% | 27.0 (7.0/18.0/2.0) | 4.0 · 1/1 | medium 1 |
| v1.8 | 19 | $15.62 | 84.9% | 1.4% | 9.2 (1.3/6.4/1.5) | 4.9 · 11/19 | low 16 · medium 3 |

Older vs v1.8: `$/session` $23.54 → $15.62; `wasted tok %` 4.5% → 1.4%; `issues/session`
19.2 → 9.2, with the high-severity share down (2.5 → 1.3); `saved %` 73.8% → 84.9%;
`quality` unrated (0/13) at `older` vs 4.9 mean on 11/19 rated sessions at v1.8.

Across the same 125 kept sessions, actual cost sums to **$2,398.36** against a realistic
Fable-only counterfactual of **$13,907.21** — **82.8%** saved ($11,508.85). It is a cost
counterfactual computed from the real per-call usage, not a quality claim. Sessions are
grouped by workflow version (`tools/versions.json`); from v1.1 (2026-08-28) each session
also gets a manual 1–5 quality rating (`/rate`), and the 35% context threshold is the
operator's, not Anthropic's. Full tables:
[`metrics/trends-2026-09.md`](metrics/trends-2026-09.md); earlier snapshot:
[`metrics/baseline-2026-08.md`](metrics/baseline-2026-08.md).

## Why these models

The default implementer runs Opus 5 at low effort. In the "simplu" experiment, opus-low beat
opus-medium, sonnet-low and sonnet-medium: audit 4/4/4 and eval 11/0 in all three lots at a
mean $2.35 per lot, vs opus-medium 4/4/3, eval 9–10/11, $2.52. The `explorer` and the
`auditor` were each tested against a cheaper cell and both kept their model and effort —
`explorer` on tied accuracy, `auditor` because the cheaper cell missed silent-deletion
deviations. Protocol, lots and confounds: [`docs/experiments.md`](docs/experiments.md).

## Install

```sh
git clone https://github.com/claude-code-king/agent-governance.git ~/agent-governance
bash ~/agent-governance/easy_install.sh
```

It backs up your current `~/.claude` first and asks for confirmation; `--dry-run` only
prints the plan. See [`HOW_TO_USE.md`](HOW_TO_USE.md) for what it changes, what
`--dry-run`/`--restore` do, the manual install, and settings that break the engine.

## Docs

- [`HOW_TO_USE.md`](HOW_TO_USE.md) — install, restore, per-project onboarding, platform
  notes, and how to run the analyzer.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — the three layers in full, the flow
  diagram, and the reasoning behind each rule.
- [`docs/experiments.md`](docs/experiments.md) — model/effort A-B tests for the agents.
- [`docs/PATTERNS.md`](docs/PATTERNS.md) — recurring technical traps.
- [`docs/RECIPES.md`](docs/RECIPES.md) — step-by-step procedures for repeated jobs.
- [`docs/postmortem-2026-09-01-fable-5-1.md`](docs/postmortem-2026-09-01-fable-5-1.md) —
  incident writeup on the Fable 5.1 sessions.
- [`metrics/trends-2026-09.md`](metrics/trends-2026-09.md) — current numbers, both splits,
  all versions.
- [`metrics/baseline-2026-08.md`](metrics/baseline-2026-08.md) — earlier snapshot, with
  method and caveats.
- [`CHANGELOG.md`](CHANGELOG.md) — what changed per version, newest first.

## Layout

```
agents/     agent definitions (explorer, implementer + complex/max/sonnet, scripter,
            scripter-complex, scribe, auditor, design-lead, design-lead-expert, refiner,
            refiner-complex) — model, effort, maxTurns, allowed tools, fixed report format
commands/   slash commands (polish, refine, rate) — mirrors ~/.claude/commands/
hooks/      the 14 enforcement hooks + 10 tests + settings.example.json
templates/  CLAUDE.global.md, CLAUDE.project.md, SCRIPTS.md
tools/      session_metrics.py (offline transcript analyzer), pricing.json, versions.json
docs/       ARCHITECTURE.md, PATTERNS.md, RECIPES.md, experiments.md, postmortem
metrics/    trends-2026-09.md, baseline-2026-08.md
CHANGELOG.md  per-version history
```

## License

MIT. See `LICENSE`.
