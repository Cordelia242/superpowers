# SDD Orca-Dispatch Mode — Design Spec

**Status:** Approved design (brainstormed 2026-09-10); implementation plan to follow.
**Scope:** personal/BancoSol fork only (`Cordelia242/prawn`). Not intended for
upstream PR — per this repo's own CONTRIBUTING rules (`CLAUDE.md`), a
change that depends on a third-party CLI (Orca) and only benefits users of
that tool does not belong in core Superpowers. This spec documents a
fork-local addition.
**Objective:** add a third plan-execution path — "Orca-Dispatch" — that runs
the same implementer → review → fix-loop → ledger process as
`subagent-driven-development` (SDD), but dispatches implementer workers
through Orca's orchestration layer instead of the native `Agent` tool. This
lets tasks run on whichever provider (Claude, Codex, Cursor, ...) Orca has
configured, and automatically falls back to a different provider when the
current one hits its own plan/quota limit mid-task.
**Hard invariant:** SDD's existing process (spec/quality review, the 5-round
fix loop, ledger discipline, model selection by task tier) is untouched.
Orca-Dispatch only changes *how a worker is dispatched and how its result is
retrieved* — two steps, not the whole skill.

## Problem

Today SDD dispatches every implementer as a native subagent (the `Agent`
tool), in this same Claude Code session. That's fine when the session has
one provider. Using Orca, tasks can instead run on independent provider
terminals (Claude, Codex, Cursor, ...) that Orca supervises — which also
means a task in progress can survive one provider's usage cap by resuming on
another, instead of stalling until a plan-limit resets.

## Research Findings (grounding this design)

Pulled from the version-matched Orca orchestration guide
(`orca skills get orchestration --json`) and live probes against this
BancoSol Orca install, not from the marketing docs page:

- **Dispatch model:** Run (namespace) → Task (work item) → Dispatch (one
  attempt on a terminal). `worker-start --task <id> --agent <provider>
  --worktree current` creates the worker; `--model`/`--effort` only apply to
  fresh Claude/Codex/Cursor launches.
- **Completion contract:** a worker sends exactly one
  `orchestration send --type worker_done --outcome succeeded|failed
  --task-id <id> --dispatch-id <id>`. Outcome is structured; the *reason*
  for a failure is free text only (`--body`/`--payload`).
- **No structured rate-limit signal in the dispatch protocol.** Confirmed by
  reading the full guide, not just the summary: `worker_done`, `escalation`,
  and `ask` all carry free-text bodies, nothing schema'd for "quota
  exhausted."
- **But `orca account list --json` does carry structured quota data** —
  this is the actual mechanism this design relies on, discovered by probing
  the CLI directly:
  ```json
  "rateLimits": {
    "claude": {"session": {"usedPercent": 20, ...}, "weekly": {"usedPercent": 35, ...}, "status": "ok"},
    "codex":  {"session": {"usedPercent": 17, ...}, "weekly": {"usedPercent": 64, ...}, "status": "ok"},
    "gemini": {"status": "unavailable", "error": "Gemini CLI OAuth is disabled..."},
    "grok":   {"status": "unavailable", "error": "Not signed in to Grok..."}
  }
  ```
  The key set under `rateLimits` **is** the discoverable provider pool — no
  provider list is hardcoded anywhere in this design. `cursor`, `omp`, and
  `pi` are valid `--agent` values per the orchestration guide but do **not**
  appear under `rateLimits` — Orca does not track their quota, so they are
  out of scope for automatic, quota-based selection (see Non-Goals).
- **Retry is manual and explicit:** `worker-start --task <id> --retry-of
  <dispatch_id> --agent <new_provider>` does not inherit placement or
  provider — the caller chooses both every time. This is the primitive the
  fallback mechanism is built on.
- **Workers cannot self-nest** (default depth 1) — consistent with SDD's
  existing "implementer never dispatches subagents" contract.

## Design Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Same skill file, parametrized dispatch mode (`native` \| `orca`), not a cloned skill. | The review/fix-loop/ledger process is the valuable, tuned part of SDD. Cloning it risks two copies drifting apart. Only "how do I dispatch and read a result" actually differs. |
| 2 | Provider pool is discovered at runtime from `orca account list --json`, never hardcoded. | Requested explicitly: the pool must reflect whatever accounts are actually configured on this Orca install, and follow Orca as it adds providers. |
| 3 | Task "size" reuses SDD's existing Model Selection tiers (mechanical/standard/architecture) rather than a new taxonomy. | That classification already exists and is used for model choice; reusing it for provider-headroom thresholds is one fewer concept to hold. |
| 4 | Provider fallback triggers **only** on confirmed quota exhaustion, never as a generic retry strategy. | A bug in the implementer's code is not fixed by trying a different provider. Conflating the two would silently mask real failures as "just try someone else." |
| 5 | Fallback hop count is capped by the discovered pool size, not a fixed constant. | The pool varies per install; capping at "every available provider once" is the natural bound, not a magic number. |
| 6 | No visualizer/dashboard. Quota check is an internal step before each dispatch decision, not a UI. | Explicitly requested — the coordinator needs the data to decide, nothing more. |

## Component: `pick-provider` (new script)

`skills/subagent-driven-development/scripts/pick-provider <tier> [--exclude p1,p2,...]`

Bash entrypoint (matching this skill's existing script style —
`sdd-workspace`, `task-brief`, `review-package`), with the JSON
arithmetic done via an inline `python3` block (stdlib `json` only — no new
dependency, consistent with Superpowers' zero-third-party-dependency
policy).

**Algorithm:**
1. Run `orca account list --json`, read `.result.rateLimits`.
2. For every provider with `status == "ok"` and not in `--exclude`:
   `headroom = 100 - max(session.usedPercent ?? 0, weekly.usedPercent ?? 0)`.
   Using the max of both windows means whichever window is closer to its
   cap is the one that determines eligibility — the binding constraint.
3. Providers with `status != "ok"`, or absent from `rateLimits` entirely
   (cursor/omp/pi today), are excluded from the automatic pool. Real Orca
   output also nests non-provider sibling keys directly inside
   `rateLimits` (`minimaxCookieConfigured`, `claudeTarget`, `codexTarget`,
   `inactiveClaudeAccounts`, `inactiveCodexAccounts`, ...) — anything that
   isn't a dict with a `status` key is skipped as not a provider record.
   (Found live during Task 1's implementation: the unguarded version
   crashes with `AttributeError: 'bool' object has no attribute 'get'` the
   first time it runs against real `orca account list --json` output.)
4. Minimum headroom threshold by tier (a safety floor, not a provider
   list): `architecture ≥ 40`, `standard ≥ 15`, `mechanical ≥ 5`. A large
   task starting on an almost-exhausted provider is a task that dies
   mid-way and forces a fallback anyway — better to route it to headroom
   from the start.
5. Sort eligible candidates by headroom descending. Print JSON: the chosen
   provider, whether it met the tier's threshold, the full ranked list, and
   what was excluded and why. The script only measures and ranks — it does
   not decide policy for the zero-eligible case; that's the calling skill
   step's job (see below).

**Where it's called:** before every fresh Task dispatch, and before every
provider-fallback retry (never before an ordinary fix-loop round — those
keep resuming the same implementer/provider as SDD already does).

## Dispatch Flow (Orca mode)

Replaces SDD's "1. Dispatch the implementer" step when in `orca` mode:

1. Classify task tier (existing Model Selection rules).
2. `pick-provider <tier>` → chosen provider.
3. Ensure a Run is bound for this plan (`orchestration run-current`, or
   `run-create` once if none exists yet — recorded in the plan's SDD
   workspace alongside the ledger).
4. `orchestration task-create --spec <task-brief path>` → task id.
5. `orchestration worker-start --task <id> --worktree current --agent
   <chosen> [--model/--effort if claude/codex/cursor]` → dispatch id.
6. Ledger records: task id, dispatch id, chosen provider, headroom at
   dispatch time — alongside the brief/report paths SDD already logs.

Replaces SDD's "waiting on dispatched subagents" guidance in `orca` mode:

7. `orchestration check --wait --types worker_done,escalation,question
   --timeout-ms <n>` in the same bounded 5-10 minute stretches SDD already
   uses for waiting. Reply to any `question` immediately, same as today.
8. On an empty window: liveness checkpoint (`worker-show`, `terminal read`,
   `terminal wait --for tui-idle`). Visible activity → keep waiting
   (activity is not completion, per the orchestration guide). Two
   consecutive empty windows with no activity and no `worker_done` →
   treat as a **stalled dispatch** and enter failure handling below. This
   step doesn't exist in native mode — a native `Agent` tool call doesn't
   go silently unresponsive the way an async, provider-hosted terminal can
   when that provider's process dies mid-task.

## Failure & Fallback Flow (Orca mode)

Triggered by `worker_done --outcome failed` **or** a stalled dispatch.
Replaces / extends SDD's "2. Handle the report" step's BLOCKED handling:

1. Re-run `pick-provider` to read the *current* headroom of the provider
   that was in use.
2. **Classify:**
   - Headroom now ~0, or `status` no longer `ok` (or, as a secondary,
     corroborating signal only, the `worker_done` body / raw terminal
     output contains known limit/quota phrasing) → **confirmed quota
     exhaustion.**
   - Otherwise → an ordinary failure. Falls through to SDD's existing
     BLOCKED handling unchanged (more context / more capable model / break
     into pieces / rule on a plan defect). **No provider hop for a failure
     that isn't quota-related.**
3. **On confirmed quota exhaustion:**
   - `pick-provider <tier> --exclude <providers already tried for this
     task>` → next candidate.
   - None eligible → pool exhausted: stop hopping. This is the breaker
     condition — ledger it and adjudicate exactly like SDD's existing
     5-round breaker (park with a ruling, or escalate if load-bearing).
     Never silently give up.
   - Eligible candidate found → `worker-start --task <id> --retry-of
     <dispatch_id> --agent <next>`. Framed like SDD's existing round-4/5
     takeover ("a prior attempt hit its provider's usage limit on
     `<old>`; you own this task now on `<new>` — read the report file for
     what was tried"), except the reason is quota, not difficulty.
4. **Ledger line (new format, additive — doesn't change existing line
   types):**
   ```
   Task <N>: provider-fallback <old>→<new> (headroom old=<x>%, new=<y>%) — reason: quota exhausted (dispatch <old_id>→<new_id>)
   ```

This is orthogonal to the existing fix-loop: findings from task review still
go through the unchanged 5-round loop (resume same implementer/provider
rounds 1-3, fresh implementer more-capable-model rounds 4-5). Provider
fallback only fires on confirmed quota exhaustion, at any point in a task's
lifecycle — before or during a fix round.

## Integration Points (exact edits, this fork only)

- **`skills/subagent-driven-development/scripts/pick-provider`** — new file,
  as specified above.
- **`skills/subagent-driven-development/SKILL.md`:**
  - Near the top: state the two dispatch modes (`native` default, `orca`),
    and that the invoking instruction (from `writing-plans`, or the human
    partner directly) states which one applies.
  - "1. Dispatch the implementer" (~lines 246-284): add the Orca-mode
    dispatch flow above as an alternate branch; native-mode text unchanged.
  - "2. Handle the report" (~lines 286-306): add the quota classification +
    fallback branch above; DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT and the
    non-quota BLOCKED path are untouched.
  - "Waiting on dispatched subagents" (~lines 235-244): add the
    liveness-checkpoint / stalled-dispatch paragraph, scoped to `orca` mode.
- **`skills/writing-plans/SKILL.md`** (~lines 153-172, "Execution
  Handoff"): add a third menu item:

  > **3. Orca-Dispatch** — Same as Subagent-Driven, but dispatches workers
  > through Orca across providers, with automatic fallback if one hits its
  > plan limit mid-task.

  Routes to `superpowers:subagent-driven-development`, stating Orca-Dispatch
  mode explicitly in the hand-off instruction.

## Testing / Verification

- **`pick-provider` gets a ponytail-style self-check:** a small
  assert-based script feeding it fixture JSON (healthy provider, exhausted
  provider, unconfigured provider) in place of a live `account list` call,
  asserting ranking, threshold, and exclude behavior. No framework, no live
  Orca dependency for this check.
- **One manual end-to-end dry run** before considering this done: a real
  1-2 task plan run through Orca-Dispatch mode against this BancoSol Orca
  install, to observe actual behavior rather than assume it — see Open
  Questions below for exactly what's unverified.

## Open Questions / Risks

1. **RESOLVED (2026-09-10 dry run):** `rateLimits[provider].status` when a
   provider's session window hits 100% used — observed live, not assumed.
   During Task 4's dry run, `codex` naturally hit `session.usedPercent:
   100` mid-session. `status` stayed `"ok"` and `error` stayed `null`;
   nothing flips. Orca only sets `status: "unavailable"`/a populated
   `error` for auth/config problems (no session, OAuth disabled, not
   signed in) — never for quota exhaustion. This confirms the design's
   headroom-based check is the correct primary signal: a `status`-only
   check would have incorrectly kept reporting a fully session-exhausted
   provider as available. `pick-provider`'s output on this real data:
   `{"chosen": "claude", "ranked": [{"provider":"claude","headroom":51},
   {"provider":"codex","headroom":0}], ...}` — codex correctly demoted to
   the bottom (headroom 0), still technically "eligible" per `status`,
   which is exactly why headroom (not status alone) drives ranking.
2. **RESOLVED (2026-09-10 dry run):** the core `worker-start` →
   `check --wait` → `worker_done` mechanism (Task 2's actual design, not
   the Antigravity workaround below) was run for real against `claude` and
   worked end-to-end on the first try: `worker-start --agent claude`
   returned `state: "ready"`; the dispatched worker did the task
   autonomously and sent a structured `worker_done` with
   `outcome: "succeeded"` and `filesModified`; `check --wait` received it
   cleanly. No liveness-checkpoint path was exercised (nothing stalled).
3. **NEW — found during the dry run, not anticipated in this spec:**
   `orca orchestration worker-start --agent antigravity` fails at
   `agent_readiness` with `lastError: "Agent startup blocked:
   codex-trust-workspace"` — even after manually confirming the CLI's
   one-time workspace-trust prompt on the reused terminal and verifying
   the agent process was genuinely idle and ready. This appears to be an
   Orca-side readiness-detection bug specific to the `antigravity`
   launcher, not something this design can work around — Task 1-3 of the
   implementation plan were dispatched by sending prompts directly to the
   terminal via `orca terminal send` instead (unsupervised — no
   `worker_done`, manual polling of rendered terminal screen output).
   `pick-provider` never selects `antigravity` automatically anyway (it
   has no `rateLimits` entry), so this doesn't block the design as
   specified, but it does mean Antigravity is currently unusable through
   the supervised `worker-start` path at all, manual or automatic.
4. **NEW — found during the dry run:** `orca orchestration worker-release`
   failed with an internal error (`Error invoking remote method
   'session:set': TypeError: Cannot convert undefined or null to object`),
   reproducibly, even after following the exact recovery instruction in
   the error response (`worker-show` then retry with `--retry-request`).
   The dispatch/task themselves were already `completed`/`succeeded` —
   this only affects terminal cleanup, not correctness — but "run
   `worker-release` after every worker_done" (Global Constraints / Agent
   Guidance) cannot be treated as infallible; a release failure should not
   be mistaken for a dispatch failure.
5. **Still open, genuinely unverified:** whether a rate-limited provider
   CLI hangs vs. exits cleanly on its own quota exhaustion — the dry run
   observed a provider *approach* exhaustion (codex, session headroom hit
   0) but not a dispatch actually failing because of it. The
   stalled-dispatch liveness-checkpoint path (Task 2 Step 2) remains
   untested against a real exhaustion-triggered failure.
6. **Still open:** text-pattern corroboration (the secondary signal in
   Failure Flow step 2) is inherently fragile across providers and will
   drift as provider CLIs change their own error copy. Treat it as
   corroboration only, never as the primary trigger (already reflected in
   the design), and expect to revisit the pattern list periodically.

## Non-Goals

- No visualizer/dashboard — explicitly ruled out.
- No automatic fallback for providers Orca doesn't track quota for
  (cursor/omp/pi) — they remain usable via explicit manual dispatch only.
- No changes to `executing-plans` (inline mode) or `requesting-code-review`.
- No upstream PR to `superpowers` core — this is a fork-local addition per
  `CLAUDE.md`'s own contribution rules.
