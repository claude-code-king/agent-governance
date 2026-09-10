# Architecture

This file holds the full description of the governance engine: the three layers it is built
from, and the flow a task follows through them. The README gives the short version; read
this one when you want the rules themselves and the reasoning behind them.

## The solution

Three layers, each doing what the layer above cannot.

**1. Policy — split the roles by cost.**
The orchestrator plans, writes briefs, audits diffs, reports. It does not write code over
~20 lines, does not run commands with large output, does not read whole files. Everything
else runs in a disposable subagent context on the cheapest model that can do the job. Only
conclusions cross back.

There are four executors. `implementer` (Opus 5, low effort, 100 calls) is the DEFAULT for
any brief, logic included. `implementer-complex` (Opus 5, medium effort, 100 calls) is chosen
at plan time for multi-file logic, a non-trivial verifier, or declared debugging.
`implementer-sonnet` (Sonnet 5, high effort, 100 calls) is chosen
at plan time only for briefs with a cheap checker (a `verifica-*.mjs` script, build, test, a
grep that catches failure) and no cross-file JS/TS debugging. `implementer-max` (Opus, high
effort, 120 calls) is an escalation, not a default: only a re-send after a failed audit on
the same brief, or debugging declared at plan time with a written reason; a logic deviation
at audit on Sonnet work goes to `implementer-max`, not a second Sonnet run. Worker turns and
tokens are not a cost to save: the worker's
context dies at the end of the run and only a ≤1,500-character report reaches the
orchestrator. The plan file holds the briefs as `## Brief N` sections; the `Agent` prompt
sent to the worker is ≤10 lines — the path to the plan file, "execute only section N," no
commit — so the orchestrator writes the brief once, not twice. Before reporting, the worker
runs a mandatory self-verify loop (build/tests/type-check/regression as applicable) and
lists anything it could not run, and why, on a `NOT RUN` line of the fixed report.

History: v1.3 (2026-08-30 00:55) added `implementer-sonnet`; v1.4 (2026-08-30 08:53) moved
the default to `implementer` at medium effort and capped a brief at ~150k agent context, on
evidence from 10 large runs (tool errors 0.9% → 5.1% from the first to the last quarter of a
run, cost per call doubled) and Anthropic's published effort curve (medium ≈ −2 points at
half the cost on long-horizon coding); the auditor now fixes mechanical deviations itself
(35/37 audits reported deviations, median fix 526 characters); verification runs once at the
end (12% of 507 verification calls led to a fix). Expected effect: measured on ≥5 v1.4
sessions against v1.2's $23.13/session, not claimed in advance. v1.4b added `scripter`
(cheap model, high effort) and `scripter-complex` (worker model, medium effort) for
repetitive edits before straight-to-worker briefs — same medium-vs-high reasoning as above
(medium ≈ high accuracy at 70–85% cost), on evidence from a $79/6-run day of manual
find/replace and hand-run verification. It also relaxed parallelism to a hard cap of 4 live
agents of any type (see "Parallelism" below). v1.4.1 (2026-08-30 evening) fixed
`context-agent.sh` to measure the sub-agent's own transcript instead of main's (it had never
actually enforced the 150k/220k ceiling); tested `explorer` and `auditor` against a
lower-cost cell each — both kept their model/effort, `explorer` on tied accuracy and
`auditor` because the cheaper cell missed silent-deletion deviations; and added a
decision-dossier pattern (`explorer` writes `docs/dosar/<slug>.md` before a >300-line-read
brief) plus two analyzer flags, `tool_results_read` and `late_first_edit`. v1.5 (2026-08-30
evening) routed main's own reading through `explorer`/`explorer-max` instead of direct
`Read`/grep, added `explorer-max` (Sonnet 5 medium, 6k report cap) and four analyzer flags
for the new rules (`main_read_before_first_agent`, `max_without_sendmessage`,
`agent_read_plan_whole`, `edit_via_bash`). v1.5.1 split orchestration out of
`~/.claude/CLAUDE.md` into `~/.claude/orchestrare.md`, injected only into the main session
by `hooks/session-start.sh`, so subagents no longer inherit it. v1.5.2 (2026-08-30 night)
split the SessionStart injection into two hook calls (`rules`, `handoff`) after finding the
combined 12KB output was persisted to a file with only a 2KB preview in context — sessions
had been starting with truncated rules and no handoff; verified on a test session, both
blocks now arrive whole. Same night: killed sessions get no SessionEnd, so their metrics are
recovered by running the analyzer manually. v1.5.3 (2026-09-01) redefined "narration
avoidable" in `tools/session_metrics.py` as a text-only call with no live agent and no
question to the user, adding a `residual_poll` subtype; across 12 sessions this cut avoidable
turns from 62 to 11 (5 of them residual). v1.6 (2026-09-02) added `bash-mare.sh`,
`write-mare.sh`, `commit-gate.sh`, and `agenti-vii.sh`, and gave `context-agent.sh` a
`--scope main` mode plus opt-in per-type thresholds (`--praguri-tip`) — see internal decision
notes, not published. Smoke-tested live 2026-09-02: deny hooks
enforce, ask hooks (commit gate, live-agent cap) are advisory under auto permission mode by
design; v1.6 is tuned for Fable 5.1 as orchestrator. v1.6.1 (2026-09-02) moved the default
implementer to Opus 5 low effort after the "simplu" experiment — see `docs/experiments.md`
«r2–r4 results»: opus-low audit 4/4/4 and eval 11/0 in all three lots at a mean $2.35 per
lot, vs opus-medium 4/4/3, eval 9–10/11, $2.52 (confounds listed in the same section); added
`implementer-complex` (Opus medium) as the plan-time choice for multi-file logic.

**2. Enforcement — hooks, not good intentions.**
A `SubagentStop` hook measures the final report and blocks it once if it exceeds 2,000
characters, demanding the compressed fixed format — it fires for background agents too (verified 2026-08-30; the earlier version was silently ignored because it emitted a top-level `decision` instead of `hookSpecificOutput`); the cap is enforced by the brief's report format, and the metric (`long_agent_report`) shows how often it holds. A `PreToolUse` hook on `Read`
(`read-mare.sh`) denies re-reading a file already read in main, and denies a 300+ line read
without `offset`/`limit` (plans and images stay a warning). Another on `Agent` (`brief-mare.sh`)
warns when a brief exceeds 7,000 characters — the signal that one brief is really three. A third,
`context-agent.sh`, tracks a subagent's own context: a reminder at 150k to wrap up, a deny
on every tool but `Bash` at 220k. A `PostToolUse` hook on `Edit`/`Write`, the only one active in
main and in every worker, flags comment blocks the call just added — a new comment is one
pointer line, the explanation belongs in `PATTERNS`/`DECIZII` — without ever blocking, and
logs them for `/handoff` (`comment_bloat` in the analyzer).

**3. Telemetry — offline, zero tokens.**
A `SessionEnd` hook (`session-metrics.sh`) runs a plain-Python analyzer over the session's
JSONL transcript and writes a JSON report to a gitignored directory. No model call, no tokens, no network. It
reports tokens and cost per model, the share of output produced in subagents, per-agent
final-report lengths, the largest tool results, and files read more than once.

## The flow

```
  operator
     │  approves
     ▼
┌──────────────────────────────────────────────────────────┐
│  ORCHESTRATOR  (expensive model, the scarce context)     │
│                                                          │
│   plan ──► brief ─────────┐        ┌────── audit ──┐     │
│    ▲                      │        │               │     │
│    │                      ▼        │               ▼     │
│    │              ╔═══════════════════╗    ╔═════════════╗
│    │              ║   implementer     ║    ║   auditor   ║
│    │              ║  (own context)    ║    ║ (own ctx,   ║
│    │              ╚═══════════════════╝    ║  read-only) ║
│    │                      │                ╚═════════════╝
│    │       report ≤1.5k   │  diff                │
│    │       [raport-lung]  ▼                      │ ≤1.5k
│    │              ┌───────────────┐              │
│    └──── repair ◄─┤   re-audit    │◄─────────────┘
│                   └───────────────┘
│                          │ clean
└──────────────────────────┼───────────────────────────────┘
                           ▼
                    final report ──► operator

  every session end ──► session_metrics.py ──► metrics-local/*.json  (0 tokens)
```

The dashed budget on every arrow crossing back into the orchestrator is 1,500 characters.
That is the whole trick: the diff can be 4,000 lines, but what the expensive model reads
about it is a page.

Audit itself is 3 fixed commands run by the orchestrator: `git diff --stat`, a diff against
the earlier briefs on the same file, and one grep per "unclear/risky" line in the worker's
report. The `auditor` agent is used above a threshold — `git diff --stat` over 150 lines
changed, more than 3 files, or any `.js/.ts/.mjs/.astro` file with new logic; below it the
orchestrator reads the diff itself, once, no re-reading. The auditor fixes mechanical
deviations directly (≤20 lines/file, ≤3 files, no new logic) and reports them as `FIXED`
with the hunk; the rest go back to the still-alive implementer via
`SendMessage` — one message, near-zero bootstrap — instead of spawning a new run.

**/polish (v1.2)**: design-lead writes the plan to a file (≤8k chars), orchestrator reads
only its ≤1.5k report; a router picks the lead — `design-lead` (expensive model) for
components with clear paths, `design-lead-expert` (orchestrator-tier model, Opus 5 xhigh)
for targets needing direction/taste. The long route (2a explorer → 2b implementer ∥ 2c
expert Phase A → 2d SendMessage Phase B): explorer writes a dossier (paths + line ranges);
implementer writes and runs the measurement script, in parallel with the expert's Phase A,
which writes 3 fixed concepts without data (C1/C2/C3, so "safe" is not an option); Phase B
resumes the same agent via `SendMessage` (no second launch) for synthesis — a backbone
concept plus borrowed parts, with numbers from the measurement script. The orchestrator
reviews the plan against DECISIONS/budget and does the steering by text, not the design.
History: v1.1 (2026-08-28) added the router and the long route; v1.2 (2026-08-29) moved
the expert to Opus 5 xhigh and split it into the two phases above.

**/refine (v1.8)**: same shape as `/polish` but scoped to one page or section, not a
redesign. A router picks `refiner` or `refiner-complex`; an explorer writes a dossier and a
scripter writes/runs the measurement script plus ≤3 screenshots in parallel; the refiner
writes the plan to `docs/refine/<slug>.md`, the orchestrator reviews it adversarially (one
round, against DECISIONS and the target's spec sections) before claude_code_king picks items to
implement. `docs/refine/` is never committed.

**Scripter before repetitive work (v1.4b)**: when a brief has the same edit repeated across
many files, or a check that will run more than once, the first brief goes to `scripter`
(cheap model, high effort) instead of straight to `implementer`. It writes a script under
`scripts/`, runs it dry-run → one-file proof → full run → idempotency check, and logs it in
`scripts/SCRIPTS.md` so the next brief can reuse or adapt it instead of writing a new one.
Motive: one day's transcripts showed 6 repetitive runs costing $79 — manual find/replace
by hand across a CSS file, and a verification script run by hand 21 times in one brief.
v1.4 also (2026-08-30 afternoon) made the comment policy explicit — a new comment is a
one-line pointer to a PATTERNS/DECISIONS section, never a block — after measuring 49%
comment lines and 234 multi-line blocks in one project's `src/`.

**Parallelism (v1.4)**: explorers can run in parallel without asking; the auditor runs on
brief N while the worker runs brief N+1, when the two briefs' file lists are disjoint. Hard
cap of 4 live agents of any type; the analyzer flags a session that goes over it as
`parallel_over_cap`. Reason: parallelism does not change tokens, only wall-clock time.

**Comments (v1.4)**: a new code comment is a one-line pointer, `🔴 <constraint> —
PATTERNS/DECISIONS «section»`, never an explanatory block — the explanation lives in the
doc section, not the code. Enforced by `hooks/comentarii-cod.sh` (PreToolUse on
`Edit|Write|MultiEdit`, main and every subagent; sub-agents are denied on a block or an
over-long comment line, main only gets a warning),
which logs every flagged Edit/Write to a JSONL file read by `/handoff`; the analyzer's
`comment_bloat` flag caught it in 178 of 257 past sessions.

**Decision dossier (v1.4.1)**: a brief that needs >300 lines of decision material read
(docs excerpts, config, comments, transcripts) before the first `Edit` sends `explorer`
first — it writes `docs/dosar/<slug>.md` (Bash `cat >`, works under `permissionMode: plan`)
and the implementer gets the path plus line ranges instead of whole files.
`late_first_edit` flags an implementer/scripter whose first write comes late anyway (≥100k
context or ≥15 reading calls); `tool_results_read` flags any worker reading a
`tool-results/` file instead of re-running the command narrower. Reason: on one postmortem,
the first `Edit` landed at call 29 of 41 (197k context, 242k characters read first).

**Reading through explorers (v1.5)**: main reads only `git diff --stat`, agent reports and
one dossier — facts over 3k characters go to `explorer`, table-shaped answers to
`explorer-max` (Sonnet 5 medium, 6k report cap, per-agent cap in `raport-lung.sh`). Each
agent gets its brief as its own file, never the whole plan. After a non-compliant audit the
order is auditor-fix → `SendMessage` to the live implementer → `implementer-max` only for
logic deviations with a written reason. The analyzer flags `main_read_before_first_agent`,
`max_without_sendmessage`, `agent_read_plan_whole`, `edit_via_bash`. Reason: on one
postmortem (s10) main read 64k characters before the first agent because "the table didn't
fit in 2k"; on another (s1) two `implementer-max` runs cost $7.8 for 9 of 10 deviations that
were mechanical.

