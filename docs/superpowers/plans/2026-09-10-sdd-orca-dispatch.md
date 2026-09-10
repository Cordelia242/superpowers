# SDD Orca-Dispatch Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a third `subagent-driven-development` dispatch mode — `orca` —
that dispatches implementer workers through Orca orchestration across
whatever providers this Orca install has configured, choosing the
healthiest one by quota headroom and task size, and automatically retrying
on a different provider when the current one hits its own plan limit
mid-task.

**Architecture:** One new script (`pick-provider`) does the only new
deterministic logic — rank Orca-tracked providers by remaining quota,
filtered by a per-task-tier headroom floor. Everything else is targeted
prose edits to two existing skill files: `subagent-driven-development`
gets a stated dispatch mode plus Orca-mode branches at its two dispatch/
report touchpoints, and `writing-plans` gets a third Execution Handoff
option that routes into it. No new skill, no cloned process — the review
loop, fix loop, ledger, and model selection stay exactly as they are today.

**Tech Stack:** Bash (`set -euo pipefail`, matching this skill's existing
scripts) with inline `python3` (stdlib `json` only) for the JSON ranking
logic. Markdown for the skill-prose edits.

**Spec:** `docs/superpowers/specs/2026-09-10-sdd-orca-dispatch-design.md`.
Read it before starting any task — this plan implements its decisions
without re-deriving them.

## Global Constraints

- **Fork-local only.** This branch (`Cordelia242/explore-multi-provider-agent`)
  is a personal/BancoSol fork of `superpowers`. No task in this plan is
  destined for an upstream PR — see the spec's framing and `CLAUDE.md`'s own
  contribution rules.
- **`pick-provider` CLI contract (exact, later tasks depend on it):**
  `pick-provider TIER [--exclude p1,p2,...] [--input FILE]`
  - `TIER` ∈ `mechanical | standard | architecture` (SDD's existing Model
    Selection tiers).
  - `--exclude`: comma-separated provider names to skip.
  - `--input FILE`: read account/quota data from this file instead of
    calling `orca account list --json` live (used by the self-check test).
  - Prints one JSON object to stdout:
    `{"tier", "threshold", "chosen", "headroom", "meets_threshold", "ranked": [{"provider","headroom"}...], "excluded": [{"provider","reason"}...]}`.
    `chosen` is `null` only when no provider is eligible — that is a valid
    result, not an error.
  - Headroom formula: `100 - max(session.usedPercent, weekly.usedPercent)`,
    treating a missing `session` or `weekly` block as `0` used.
  - Thresholds: `mechanical: 5`, `standard: 15`, `architecture: 40`.
  - A provider is eligible only if its `rateLimits[provider].status == "ok"`
    and it is not in `--exclude`. Providers absent from `rateLimits`
    entirely (e.g. `cursor`, `omp`, `pi` today) are never eligible for
    automatic selection.
  - Exit 2 on a usage error (bad tier, unknown flag). Otherwise exits 0 and
    propagates `orca`'s exit code only if the live `orca account list` call
    itself fails.
- **Ledger line format (new, additive — existing SDD ledger line formats
  are unchanged):**
  `Task <N>: provider-fallback <old>→<new> (headroom old=<x>%, new=<y>%) — reason: quota exhausted (dispatch <old_id>→<new_id>)`
- **Dispatch-mode framing:** `native` (today's behavior) is the default;
  `orca` is opt-in, stated by whoever invokes `subagent-driven-development`.
  Only "1. Dispatch the implementer," "2. Handle the report," and the
  waiting guidance branch by mode — every other section applies to both.

## File Structure

- Create: `skills/subagent-driven-development/scripts/pick-provider` — the
  provider-ranking script (Task 1)
- Create: `skills/subagent-driven-development/scripts/test-pick-provider` —
  its fixture-based self-check (Task 1)
- Modify: `skills/subagent-driven-development/SKILL.md` — Dispatch Mode
  section, Orca-mode branches in the waiting guidance, Step 1, and Step 2
  (Task 2)
- Modify: `skills/writing-plans/SKILL.md` — third Execution Handoff option
  (Task 3)

---

### Task 1: `pick-provider` script and its self-check

**Files:**
- Create: `skills/subagent-driven-development/scripts/pick-provider`
- Create: `skills/subagent-driven-development/scripts/test-pick-provider`

**Interfaces:**
- Produces: the `pick-provider` CLI contract from Global Constraints above.
  Task 2 shells out to it as `scripts/pick-provider <tier> [--exclude ...]`
  from the SDD skill directory and reads the printed JSON.

- [ ] **Step 1: Write the self-check test (it will fail — the script doesn't exist yet)**

Create `skills/subagent-driven-development/scripts/test-pick-provider`:

```bash
#!/usr/bin/env bash
# Fixture-based self-check for pick-provider. No live Orca dependency.
# Usage: ./test-pick-provider
set -euo pipefail

dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
tmp=$(mktemp -d)
trap 'rm -rf "$tmp"' EXIT
fail_count=0

check() {
  local name=$1 out_file=$2 expected_chosen=$3 expected_meets=$4
  if ! python3 - "$name" "$out_file" "$expected_chosen" "$expected_meets" <<'PY'
import json
import sys

name, out_file, expected_chosen, expected_meets = sys.argv[1:5]
with open(out_file) as f:
    d = json.load(f)

expected_chosen = None if expected_chosen == "null" else expected_chosen
expected_meets = expected_meets == "true"

ok = True
if d.get("chosen") != expected_chosen:
    print(f"{name}: expected chosen={expected_chosen!r}, got {d.get('chosen')!r}", file=sys.stderr)
    ok = False
if d.get("meets_threshold") != expected_meets:
    print(f"{name}: expected meets_threshold={expected_meets}, got {d.get('meets_threshold')!r}", file=sys.stderr)
    ok = False
sys.exit(0 if ok else 1)
PY
  then
    fail_count=$((fail_count + 1))
  fi
}

# Fixture 1: two healthy providers, one unavailable.
cat > "$tmp/healthy.json" <<'JSON'
{"id":"x","ok":true,"result":{"rateLimits":{
  "claude":{"session":{"usedPercent":20},"weekly":{"usedPercent":35},"status":"ok"},
  "codex":{"session":{"usedPercent":17},"weekly":{"usedPercent":64},"status":"ok"},
  "gemini":{"status":"unavailable","error":"Gemini CLI OAuth is disabled"}
}}}
JSON

"$dir/pick-provider" architecture --input "$tmp/healthy.json" > "$tmp/out1.json"
check "healthy/architecture picks highest headroom" "$tmp/out1.json" "claude" "true"

"$dir/pick-provider" architecture --exclude claude --input "$tmp/healthy.json" > "$tmp/out2.json"
check "excluding claude falls to codex, below architecture threshold" "$tmp/out2.json" "codex" "false"

"$dir/pick-provider" standard --exclude claude --input "$tmp/healthy.json" > "$tmp/out3.json"
check "excluding claude still meets standard threshold" "$tmp/out3.json" "codex" "true"

# Fixture 2: one provider effectively exhausted (100% weekly).
cat > "$tmp/exhausted.json" <<'JSON'
{"id":"y","ok":true,"result":{"rateLimits":{
  "claude":{"session":{"usedPercent":95},"weekly":{"usedPercent":100},"status":"ok"},
  "codex":{"session":{"usedPercent":10},"weekly":{"usedPercent":12},"status":"ok"}
}}}
JSON

"$dir/pick-provider" mechanical --input "$tmp/exhausted.json" > "$tmp/out4.json"
check "exhausted provider is ranked last, healthy one chosen" "$tmp/out4.json" "codex" "true"

# Fixture 3: no providers tracked at all.
cat > "$tmp/empty.json" <<'JSON'
{"id":"z","ok":true,"result":{"rateLimits":{}}}
JSON

"$dir/pick-provider" mechanical --input "$tmp/empty.json" > "$tmp/out5.json"
check "no eligible providers yields chosen=null" "$tmp/out5.json" "null" "false"

if [ "$fail_count" -gt 0 ]; then
  echo "$fail_count check(s) failed" >&2
  exit 1
fi
echo "All pick-provider checks passed."
```

Make it executable: `chmod +x skills/subagent-driven-development/scripts/test-pick-provider`

- [ ] **Step 2: Run it to confirm it fails**

Run: `./skills/subagent-driven-development/scripts/test-pick-provider`
Expected: FAIL — `pick-provider: No such file or directory` (or similar),
since the script under test doesn't exist yet.

- [ ] **Step 3: Write `pick-provider`**

Create `skills/subagent-driven-development/scripts/pick-provider`:

```bash
#!/usr/bin/env bash
# Rank Orca-managed providers by remaining quota headroom for one task tier,
# and choose the healthiest one not already excluded. Provider pool is
# whatever `orca account list --json` reports under .result.rateLimits —
# nothing hardcoded here.
#
# Usage: pick-provider TIER [--exclude p1,p2,...] [--input FILE]
#   TIER: mechanical | standard | architecture (SDD's existing task tiers)
#   --exclude: comma-separated provider names to skip (already tried for
#              this task's fallback chain)
#   --input: path to a JSON file shaped like `orca account list --json`
#            output, used instead of calling the live Orca runtime
#            (for testing; see test-pick-provider)
#
# Prints one JSON object to stdout:
#   {"tier", "threshold", "chosen", "headroom", "meets_threshold",
#    "ranked": [{"provider","headroom"}, ...],
#    "excluded": [{"provider","reason"}, ...]}
# "chosen" is null only when no provider has status "ok" and is not
# excluded — that is a valid result, not an error. Exit 2 on a usage
# error; otherwise exit 0 (propagating `orca`'s exit code only if the
# live account-list call itself fails).
set -euo pipefail

if [ $# -lt 1 ]; then
  echo "usage: pick-provider TIER [--exclude p1,p2,...] [--input FILE]" >&2
  exit 2
fi

tier=$1
shift

exclude=""
input=""
while [ $# -gt 0 ]; do
  case "$1" in
    --exclude) exclude=$2; shift 2 ;;
    --input) input=$2; shift 2 ;;
    *) echo "unknown argument: $1" >&2; exit 2 ;;
  esac
done

case "$tier" in
  mechanical|standard|architecture) ;;
  *) echo "unknown tier: $tier (expected mechanical|standard|architecture)" >&2; exit 2 ;;
esac

if [ -n "$input" ]; then
  account_file="$input"
else
  account_file=$(mktemp)
  trap 'rm -f "$account_file"' EXIT
  orca account list --json > "$account_file"
fi

python3 - "$tier" "$exclude" "$account_file" <<'PY'
import json
import sys

tier, exclude_arg, account_file = sys.argv[1], sys.argv[2], sys.argv[3]
exclude = {p for p in exclude_arg.split(",") if p}

THRESHOLDS = {"mechanical": 5, "standard": 15, "architecture": 40}

with open(account_file) as f:
    data = json.load(f)

rate_limits = data.get("result", {}).get("rateLimits", {})

ranked = []
excluded = []
for provider, info in rate_limits.items():
    if provider in exclude:
        excluded.append({"provider": provider, "reason": "excluded"})
        continue
    if info.get("status") != "ok":
        excluded.append({
            "provider": provider,
            "reason": info.get("error") or info.get("status") or "not ok",
        })
        continue
    session_pct = (info.get("session") or {}).get("usedPercent") or 0
    weekly_pct = (info.get("weekly") or {}).get("usedPercent") or 0
    headroom = 100 - max(session_pct, weekly_pct)
    ranked.append({"provider": provider, "headroom": headroom})

ranked.sort(key=lambda r: r["headroom"], reverse=True)

threshold = THRESHOLDS[tier]
chosen = ranked[0] if ranked else None

result = {
    "tier": tier,
    "threshold": threshold,
    "chosen": chosen["provider"] if chosen else None,
    "headroom": chosen["headroom"] if chosen else None,
    "meets_threshold": bool(chosen and chosen["headroom"] >= threshold),
    "ranked": ranked,
    "excluded": excluded,
}
print(json.dumps(result))
PY
```

Make it executable: `chmod +x skills/subagent-driven-development/scripts/pick-provider`

- [ ] **Step 4: Run the self-check to confirm it passes**

Run: `./skills/subagent-driven-development/scripts/test-pick-provider`
Expected: `All pick-provider checks passed.` with exit code 0.

- [ ] **Step 5: Commit**

```bash
git add skills/subagent-driven-development/scripts/pick-provider skills/subagent-driven-development/scripts/test-pick-provider
git commit -m "feat(sdd): add pick-provider quota-aware provider ranking script"
```

---

### Task 2: Orca-Dispatch mode in `subagent-driven-development`

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`

**Interfaces:**
- Consumes: `scripts/pick-provider` CLI contract (Task 1).
- Produces: a "Dispatch Mode" section (`native` | `orca`) and the
  provider-fallback ledger line format from Global Constraints. Task 3
  references this mode by name ("orca dispatch mode").

- [ ] **Step 1: Insert the Dispatch Mode section**

In `skills/subagent-driven-development/SKILL.md`, find this exact
paragraph (it currently ends the intro, right before `## When to Use`):

```markdown
Four things stop you, and only these: an irreversible or destructive
operation; a security-sensitive action; a side effect outside this worktree
that norms say you ask about first (a merge, a push to a shared branch, a
publish); and a plan so broken that every path forward is a guess. For those,
stop and ask.

## When to Use
```

Replace it with:

```markdown
Four things stop you, and only these: an irreversible or destructive
operation; a security-sensitive action; a side effect outside this worktree
that norms say you ask about first (a merge, a push to a shared branch, a
publish); and a plan so broken that every path forward is a guess. For those,
stop and ask.

## Dispatch Mode

This skill runs in one of two dispatch modes, stated by whoever invokes it
(`writing-plans`'s Execution Handoff, or your human partner directly):

- **native (default):** dispatch implementers with the `Agent` tool, in
  this session. Everything below assumes this mode unless stated otherwise.
- **orca:** dispatch implementers through Orca orchestration, across
  whichever providers this Orca install has configured, with automatic
  fallback to a different provider if one hits its own plan/quota limit
  mid-task. Changes only "1. Dispatch the implementer," "2. Handle the
  report," and the waiting guidance below — everything else (review, the
  fix loop, the ledger, model selection) is identical in both modes.

## When to Use
```

- [ ] **Step 2: Add the Orca-mode waiting/liveness paragraph**

In the same file, find this exact paragraph (end of the "Waiting on
dispatched subagents" guidance, immediately before `### 1. Dispatch the
implementer`):

```markdown
**Waiting on dispatched subagents:** never poll a wait interface with
short timeouts, and never sit in one silent, open-ended wait either.
While you have local work — ledger updates, packaging the next review,
reading reports — keep working; child results arrive on their own.
When you are genuinely idle, wait in bounded stretches (five to ten
minutes, where your platform allows), and between stretches post one
line of status and reconcile your live children: list them, and chase
any that finished without reporting. A bounded stretch keeps nearly
all of a long wait's efficiency while guaranteeing a stuck or lost
child is noticed within minutes, not at the end of the session.

### 1. Dispatch the implementer
```

Replace it with:

```markdown
**Waiting on dispatched subagents:** never poll a wait interface with
short timeouts, and never sit in one silent, open-ended wait either.
While you have local work — ledger updates, packaging the next review,
reading reports — keep working; child results arrive on their own.
When you are genuinely idle, wait in bounded stretches (five to ten
minutes, where your platform allows), and between stretches post one
line of status and reconcile your live children: list them, and chase
any that finished without reporting. A bounded stretch keeps nearly
all of a long wait's efficiency while guaranteeing a stuck or lost
child is noticed within minutes, not at the end of the session.

**Orca mode — waiting and liveness:** wait with `orca orchestration check
--wait --types worker_done,escalation,question --timeout-ms <n>` in the
same bounded stretches. Reply to any `question` immediately. If a wait
window returns nothing, run a liveness checkpoint (`orca orchestration
worker-show --dispatch <id>`, `orca terminal read --terminal <handle>`, or
`orca terminal wait --terminal <handle> --for tui-idle`) — visible activity
means keep waiting, it is not completion. Two consecutive empty windows
with no activity and no `worker_done` is a **stalled dispatch**: treat it
exactly like a `worker_done --outcome failed` in "2. Handle the report"
below. A native `Agent` tool call cannot go silently unresponsive this way;
an Orca-hosted provider terminal can, most often because that provider's
own process died on its usage limit before it could report anything.

### 1. Dispatch the implementer
```

- [ ] **Step 3: Add the Orca-mode dispatch branch to Step 1**

In the same file, find this exact line (the last bullet of "1. Dispatch
the implementer," immediately before the `Template:` line):

```markdown
- Never dispatch multiple implementation subagents in parallel (conflicts).

Template: [implementer-prompt.md](implementer-prompt.md)
```

Replace it with:

```markdown
- Never dispatch multiple implementation subagents in parallel (conflicts).

**Orca mode:** classify the task's tier (see Model Selection), then run
`scripts/pick-provider <tier>` to choose a provider. Ensure a Run is bound
for this plan (`orca orchestration run-current`, or `run-create` once if
none exists yet — record its id as a `Run: <run_id>` ledger line the first
time). Run `orca orchestration task-create --spec <task-brief path>` to get
a task id, then `orca orchestration worker-start --task <task id>
--worktree current --agent <chosen provider> [--model <id> --effort
<level>, only for claude/codex/cursor]` to dispatch. Record the task id,
dispatch id, chosen provider, and its headroom at dispatch time in the
ledger, alongside the brief/report paths. The dispatch prompt content is
unchanged — Orca delivers the same brief/report/context described above;
only the delivery mechanism differs.

Template: [implementer-prompt.md](implementer-prompt.md)
```

- [ ] **Step 4: Add the Orca-mode fallback branch to Step 2**

In the same file, find this exact block (the start of "2. Handle the
report"'s BLOCKED case):

```markdown
**BLOCKED:** The implementer cannot complete the task. Assess the blocker:
1. If it's a context problem, provide more context and re-dispatch with the same model
2. If the task requires more reasoning, re-dispatch with a more capable model
3. If the task is too large, break it into smaller pieces
4. If the plan itself is wrong, rule on the correction, ledger it, and re-dispatch with the ruling carried in the dispatch

**Never** ignore an escalation or force the same model to retry without changes. If the implementer said it's stuck, something needs to change.
```

Replace it with:

```markdown
**BLOCKED (or, in Orca mode, a stalled dispatch — see Waiting above):** the
implementer cannot complete the task.

**Orca mode only — classify before anything else:** run
`scripts/pick-provider <tier>` again and read the current provider's own
entry in its output. If its headroom is now ~0 or it's no longer eligible
(or, only as corroboration, the report/terminal output names a usage/rate
limit) — this is confirmed quota exhaustion, not an implementation problem:
- Run `scripts/pick-provider <tier> --exclude <providers already tried for
  this task>`. No eligible candidate → the fallback pool is exhausted: stop
  hopping, ledger it, and adjudicate exactly like the fix loop's breaker
  (park with a ruling, or escalate if load-bearing) — never silently give up.
- Eligible candidate → `orca orchestration worker-start --task <task id>
  --retry-of <dispatch id> --agent <next provider>`, framed like the fix
  loop's round-4/5 takeover ("a prior attempt hit its provider's usage
  limit on `<old>`; you own this task now on `<new>` — read the report
  file for what was tried"), except the reason is quota, not difficulty.
- Ledger: `Task <N>: provider-fallback <old>→<new> (headroom old=<x>%,
  new=<y>%) — reason: quota exhausted (dispatch <old_id>→<new_id>)`.
- This never applies to fix-loop rounds — those keep resuming the same
  implementer/provider exactly as described in the fix loop below.

If the provider is not quota-exhausted (or you're in native mode), assess
the blocker normally:
1. If it's a context problem, provide more context and re-dispatch with the same model
2. If the task requires more reasoning, re-dispatch with a more capable model
3. If the task is too large, break it into smaller pieces
4. If the plan itself is wrong, rule on the correction, ledger it, and re-dispatch with the ruling carried in the dispatch

**Never** ignore an escalation or force the same model to retry without changes. If the implementer said it's stuck, something needs to change.
```

- [ ] **Step 5: Verify all four edits landed and nothing else changed**

Run: `git diff skills/subagent-driven-development/SKILL.md`
Expected: four additions matching Steps 1-4 above, no other lines touched.
Run: `grep -c "Orca mode" skills/subagent-driven-development/SKILL.md`
Expected: `4` — one in Step 2's waiting paragraph, one in Step 3's dispatch
branch, and two in Step 4 (the BLOCKED parenthetical plus the "Orca mode
only — classify" line). The Dispatch Mode heading itself uses lowercase
`orca` as a bullet label, not the literal string "Orca mode," so it does
not add to this count.

- [ ] **Step 6: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat(sdd): add orca dispatch mode with quota-based provider fallback"
```

---

### Task 3: Third Execution Handoff option in `writing-plans`

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

**Interfaces:**
- Consumes: the "orca" dispatch mode named in Task 2's Dispatch Mode
  section of `subagent-driven-development`.

- [ ] **Step 1: Replace the Execution Handoff section**

In `skills/writing-plans/SKILL.md`, find this exact section:

```markdown
## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Fresh subagent per task + two-stage review

**If Inline Execution chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:executing-plans
- Batch execution with checkpoints for review
```

Replace it with:

```markdown
## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Three execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**3. Orca-Dispatch** - Same as Subagent-Driven, but dispatches workers through Orca orchestration across providers, with automatic fallback to another provider if one hits its plan limit mid-task

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Fresh subagent per task + two-stage review

**If Inline Execution chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:executing-plans
- Batch execution with checkpoints for review

**If Orca-Dispatch chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development in **orca** dispatch mode (see that skill's Dispatch Mode section)
- Fresh implementer per task, dispatched via Orca across providers + two-stage review
```

- [ ] **Step 2: Verify**

Run: `grep -n "Orca-Dispatch" skills/writing-plans/SKILL.md`
Expected: two matches — the menu item and the "If Orca-Dispatch chosen"
branch.

- [ ] **Step 3: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat(writing-plans): add Orca-Dispatch as a third execution option"
```

---

### Task 4: Manual end-to-end dry run (flagged — consumes real provider quota)

This task is exploratory verification, not code. It creates real Orca Run/
Task/Dispatch state and spends real usage on whichever provider gets
chosen — run it only with your human partner's go-ahead, the same as any
other side effect outside this worktree.

**Files:**
- Modify: `docs/superpowers/specs/2026-09-10-sdd-orca-dispatch-design.md`
  (Open Questions section — record what got verified)

- [ ] **Step 1: Write a throwaway 1-2 task plan**

Any small plan works — e.g. a one-task plan that adds a one-line comment to
a scratch file in this worktree. It only needs to be small and mechanical
(tier: `mechanical`) so the dry run is cheap.

- [ ] **Step 2: Run it through `subagent-driven-development` in orca mode**

Follow `skills/subagent-driven-development/SKILL.md`'s Task Loop with
Dispatch Mode = `orca`. Confirm, and note the actual observed result for
each:
- `pick-provider mechanical` (live, no `--input`) prints a real provider
  choice from `orca account list --json`.
- `orca orchestration worker-start` succeeds and the ledger records task
  id, dispatch id, provider, and headroom as Task 2's Step 3 specifies.
- `orca orchestration check --wait` returns `worker_done` and the task
  completes normally through the existing review flow.

- [ ] **Step 3: Update the spec's Open Questions section with real findings**

Edit `docs/superpowers/specs/2026-09-10-sdd-orca-dispatch-design.md`: under
"Open Questions / Risks," add what this dry run actually confirmed (e.g.
the exact shape of `worker-start`'s receipt, whether the ledger format
round-trips cleanly) and what remains genuinely unverified — this dry run
cannot safely force a real provider into quota exhaustion, so items 1 and 2
in that section (what an exhausted `status`/`error` looks like, whether a
rate-limited provider CLI hangs vs. exits) stay open until one occurs
naturally. Do not mark them resolved without having actually observed one.

- [ ] **Step 4: Commit the spec update**

```bash
git add docs/superpowers/specs/2026-09-10-sdd-orca-dispatch-design.md
git commit -m "docs: record orca-dispatch dry-run findings in open questions"
```

## Self-Review Notes

- **Spec coverage:** provider discovery/ranking (Task 1), dispatch flow +
  ledger fields (Task 2 Step 3), waiting/liveness (Task 2 Step 2),
  failure classification + fallback + ledger line (Task 2 Step 4),
  writing-plans menu (Task 3), verification (Task 4). No visualizer, no
  cursor/omp/pi auto-fallback, no executing-plans/requesting-code-review
  changes — matches the spec's Non-Goals; no task builds any of them.
- **Type/name consistency:** `pick-provider`'s flags (`--exclude`,
  `--input`), tiers (`mechanical|standard|architecture`), and JSON field
  names (`chosen`, `meets_threshold`, `headroom`, `ranked`, `excluded`)
  match verbatim between Task 1's code and Task 2's SKILL.md prose. The
  ledger line format in Global Constraints matches Task 2 Step 4 exactly.
