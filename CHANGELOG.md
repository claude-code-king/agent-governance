# Changelog

Newest first. The governance rules are frozen at v1.8; later versions change only tooling.

## v1.8.2 (2026-09-10)

settings.json does not hot-reload effort, and SessionStart writes the target too late for
the current session. Fix: SessionEnd writes medium for the next launch, and a new PreToolUse
gate denies tools until effective effort matches the `/effort <target>` claude_code_king ran, then
he types go. See `docs/DECIZII.md` «v1.8.2 — effort gate» (private decision log, see the
note in HOW_TO_USE.md §6).

## v1.8 (2026-09-05)

Phase-based effort is back: the `~/.claude/v17-effort-auto` flag is set again, since
changelog 2.1.260 states `/effort` no longer rewrites the prompt cache. claude_code_king still runs
`/effort` by hand; the hook only warns. Check the first main call after a switch — if
`cache_read` drops to ~17k and `cache_creation` jumps, remove the flag and go back to
medium constant. The advisor is now mandatory on triggers a-f (2+ JS/TS briefs, hooks or
live config or data migration, claude_code_king's own word, any complex/max brief, 3+ briefs, parallel
implementers or a worktree), and it re-reads the updated plan in round 2. `bash-mare.sh`
adds a main-side nudge on the 3rd consecutive small Bash call.

`subagentPromptCacheTtl` is deliberately left unset: measured on the 2026-09-03 blueprint,
agents already write 100% at 5m (2.21M tokens) and main 100% at 1h (293k). Fable 5.1
pricing is corrected across the analyzer ($10/$50, cache read $1 at 0.1x for calc — claude_code_king is
on subscription; $0.25 is only the informative API price, write 5m $12.50 / 1h $20;
`scripts/pricing-cache-5m.py`): the same blueprint goes $65.87 to $57.72 in total (the drop
comes only from agent 5m writes billed at 1.25x instead of 2x), main stays $19.34 (main
writes 1h cache). The Fable-only floor now runs on `claude-fable-5-1` too (cache read $1,
0.1x): floor $89.44 (1.5x), realistic $556.44 — read orchestration savings on `realistic`,
not floor. Narration was left without a hook: 27 of 88 main calls on that blueprint,
~$0.73, about 1.5% of the session.

New skill `/refine`: like `/polish` but scoped to one page or section, routed to `refiner`
or `refiner-complex` (both Fable 5.1) — see the `/polish` entry in docs/ARCHITECTURE.md
«The flow» for the shape. Separately,
`~/.claude/orchestrare.md` and `templates/orchestrare.md` are capped at 10 KB, since the
harness truncates the injected SessionStart block to 2 KB past that size; on 2026-09-05 the
margin was under 5 bytes, so any new line there needs a compensating cut.

Phase-based effort now also runs inside `/polish` and `/refine`: both commands call the
`effort-phase.sh` hook, at step 0/1 and again at step 5 after plan approval. The `/refine`
scripter brief (step 2b) asks for line ranges plus per-project measurement scripts. Metrics from
14:45 (local time) onward are the ones that count for the v1.8 verdict.

## v1.7.5 (stable, 2026-09-03)

New entry `v1.7.5` in `tools/versions.json` (from 2026-09-03T13:04 local): sessions from
now on are grouped separately in TRENDS; v1.7.1–v1.7.4 stay under `v1.7`. The three
v1.7.4 agent guards were validated live on a real agent (read-mare deny on re-reading its
own write, bash-mare additionalContext on the 3rd identical run, comentarii-cod deny on a
2-line comment block); all four offline suites pass (model-gate 25, read-mare 31,
bash-mare 8, comentarii-cod 12). The "verifier runs once, at the end" rule was already
present in the agent definitions; nothing added there.

## v1.7.4 (stable, 2026-09-03)

The effort-split experiment (plan=medium / implementation=low) is dropped; main now runs
Fable 5.1 medium constant. Reason: a real `/effort` rewrites the prompt cache — a
top-level effort change invalidates the cache per Anthropic's API docs, while Claude
Code's docs claimed otherwise (issue `anthropics/claude-code#61984`, unclear whether bug
or intended). Measured on promo-site 2026-09-02: two switches cost ~$3.2 in cache writes,
the low phase saved ~$2.0; low was only ~13% cheaper per main turn across 6 v1.7 sessions.

More enforcement: a model gate `hooks/main-model.sh` — on Opus (non-Fable) main, the
main-only guards (write-mare, read-mare, bash-mare, brief-mare, agenti-vii, commit-gate)
go silent and `SessionStart` no longer injects ORCHESTRATION; `comentarii-cod.sh` moved
to `PreToolUse` and now denies ≥2-line comment blocks in agents (warns in main).

`read-mare` now denies a re-read (agent Read on a path it just Wrote/Edited, no legitimate
Read/Bash-on-basename in between) and denies Read on `.png/.jpg/.jpeg/.webp/.gif` >200 KB in
main (agents unaffected); `bash-mare` warns in `additionalContext` on the 3rd identical
Bash command in an agent's transcript with no edit in between (main unaffected).

New test files: `hooks/test-model-gate.sh`, `hooks/test-comentarii-cod.sh`,
`hooks/test-bash-mare.sh`.

Bug fixes: `settings.json` `effortLevel` restored to medium (the phase hook had left it
on low); test suites no longer depend on live settings (`GOV_MODEL` env var instead).

## v1.7.3 EXPERIMENTAL (2026-09-02)

`/rate N [note]` now takes only the score (1-5) and one line of context; `advisor_score`
and `mistakes` are derived by the analyzer instead of typed by hand (`derive_quality` in
`tools/session_metrics.py`). Rough formula: start at 3, +1 if changes were actually
applied after the plan, +1 if the audit found zero deviations, -1 for ≥3 deviations, -1 if
a NO-GO verdict was implemented anyway, clamped 1-5; `mistakes` sums audit deviations,
low-effort-phase flags, and reruns. Both fields carry a `*_src` tag, `auto` or `manual`.

Bug fixes this round: `main=0` on 33 of 94 metrics records, because `inherited` compared
`session_id` — a process id that survives `/clear` — against the transcript file name; the
fix checks the foreign `session_id` AND that the line's `uuid` is present in the parent's
own transcript (a first attempt keyed on `sessionId` instead double-counted three
sessions, caught by the advisor before it shipped). `write_record` now writes `.json`/`.md`
through a temp file + atomic rename. `hooks/session-start.sh`: the v17 branch read
`settings.json` before the rules branch had written it, so a `/clear` could show a stale
effort — fixed with one shared reset function called from both branches.

The metrics corpus was regenerated: 21 records moved from `main=0` to their real cost,
so `TRENDS.md` and `V17.md` now attribute spend correctly.

## v1.7.1 EXPERIMENTAL (2026-09-02)

Two additions, both gated by a trigger, not always-on. An **advisor** agent (Fable 5.1
high, read-only) is called only for risky plans — see `orchestrare-v17.md` for the exact
a/b/c trigger conditions — and returns a fixed report (verdict, changes, risks, edge
cases, improvements); if it needs more information it asks for it via a `NEED` line
through the orchestrator, never by reading extra files itself. `orchestrare-v17.md` is
injected by a third, separate `SessionStart` hook call because the existing
`orchestrare.md` is already near the 10k-character cap per hook command. Second: effort
now switches by phase — medium while planning, low while implementing — flipped by hooks
around `EnterPlanMode`/`ExitPlanMode`, with a `WARN` line when the setting and the actual
effort disagree. Live-agent cap raised 4 → 6. Metrics: section «T-v17» in
`docs/experiments.md` + `metrics-local/V17.md`. Sessions are labeled `task_class`
(`governance-rd`/`product`), cost is read as `v17.cost_per_turn` (not counterfactual),
and `plan.echo` and `Summary` show up in the per-session report under `metrics-local/`.

**What we're testing**: claude_code_king's theory is that heavy reasoning earns its cost at plan time;
at implementation time governance (briefs, audit, hooks) already does the reasoning's job,
so the expensive second opinion (the advisor) is only worth it when the plan is risky.
Judged over ≥3 real sessions with `/rate`, $ spent by main per phase, and the flags from
`metrics-local`, compared against medium-throughout sessions; conclusion goes in
`docs/experiments.md` «T-v17».

**Known limitations**: switching effort currently needs a manual `/effort low` — the
running session's settings.json is not reloaded live, this was measured, not assumed; no
evidence yet either way for running main itself on low effort.

**v1.7.1 (2026-09-02, evening) — bugs found**: (1) `session-start.sh` reset effort to
medium on any `source` but `resume`; a fork (`source=fork`) hit it and WARN pointed the
wrong way — fixed (blacklist resume|fork|compact, 5 tests); (2) the analyzer saw only the
pre-fork part of a forked session (SessionEnd passes the origin path, fork messages carry
the new id) — fixed (`continued-in` chain, `forked_to`, one record); (3) Anthropic side:
Claude Code auto-forks main when it goes background with live subagents, undocumented,
rewrites the conversation cache (~63k tokens) — feedback filed 2026-09-02. Measured: a
real `/effort` rewrites the messages cache once (~52k); writing settings.json does not.


v1.3-v1.6.1: see the History block in docs/ARCHITECTURE.md.

Earlier versions (v1.0-v1.7): see [`metrics/trends-2026-09.md`](metrics/trends-2026-09.md) for per-version numbers.
