---
name: "init"
description: "Create a new AgentHub collaboration session with task, agent count, and evaluation criteria."
command: /hub:init
---

# /hub:init — Create New Session

Initialize an AgentHub collaboration session. Creates the `.agenthub/` directory structure, generates a session ID, and configures evaluation criteria.

## Usage

```
/hub:init                                                    # Interactive mode
/hub:init --task "Optimize API" --agents 3 --eval "pytest bench.py" --metric p50_ms --direction lower
/hub:init --task "Refactor auth" --agents 2                  # No eval (LLM judge mode)
/hub:init --task "..." --skip-benefit-check                  # Bypass the gate below
```

---

## Benefit Evaluation — DO THIS FIRST, BEFORE CREATING ANYTHING

**Run this gate before `hub_init.py`, before writing `environment.md`, before any
research.** A parallel run costs roughly N× the tokens of doing the work once, plus
orchestration, plus the review of N−1 branches that get discarded. That is worth paying
only when the extra branches buy something. Frequently they do not, and the cheapest
moment to discover that is now — a "don't" verdict here costs one short analysis, while
the same verdict discovered at `/hub:merge` has already burned the whole run.

Do **not** skip this because the user asked for `/hub:init` directly. Asking for the tool
is not a claim that the tool is right for the task; the user is asking for a session
*and* for your judgement about whether to want one. Skip it only on
`--skip-benefit-check`.

### What actually creates value here

AgentHub pays off when **several defensible approaches exist and you cannot tell in
advance which one wins**. Its product is a *comparison*. Everything else it does —
worktrees, per-agent databases, ranked evaluation — is machinery for making that
comparison fair.

So the question is never "is this task big?" or "is this task hard?". It is:

> **If I ran three competent agents on this, would I get three meaningfully different
> answers?**

If the answer is no, you are paying 3× to receive the same branch three times.

### The five checks

Answer each honestly, in one line, citing the actual task — not in the abstract.

1. **Approach variance.** Is the *design* open, or is it already pinned? Count what
   constrains it: a spec that gives exact DDL, a fixed error-to-status mapping, a
   required ordering, a named file to edit, an explicit out-of-scope list. **A
   well-specified task is the classic false positive** — it looks big and important,
   which is exactly what makes people reach for parallelism, but a spec that pins the
   decisions has already done the exploring. Heavily specified ⇒ counts **against**.

2. **Discriminator.** When three branches land, can you actually pick a winner? A real
   eval metric is the strong form. A short, readable artifact you can judge side by side
   is the weak form. "I'll read three large diffs and form an opinion" is **not** a
   discriminator — it is three code reviews, and it costs more than writing the code
   once. No discriminator ⇒ counts **against**.

3. **Independent failure modes.** Would the agents fail *differently*? If the hard part
   is one shared unknown — an undocumented API, a flaky environment, a data question —
   all three hit the same wall simultaneously and you have bought three copies of one
   blocker. Shared single point of failure ⇒ counts **against**.

4. **Self-contained scope.** Can one agent finish a complete, mergeable unit without
   coordinating mid-flight? Sequential stages, or work that must interleave with another
   agent's output, do not parallelise — they serialise with extra steps. Not
   self-contained ⇒ counts **against**.

5. **Cost floor.** Is the task large enough that orchestration overhead disappears into
   it? Setup, per-agent environment docs, provisioning, spawn, monitoring, ranking and
   merge are a fixed cost paid whether the task takes ten minutes or ten hours. Small
   task ⇒ counts **against**.

### Verdict

| Checks favouring a run | Verdict |
|---|---|
| 4–5 | **PROCEED** — create the session |
| 3 | **PROCEED WITH CHANGES** — name the specific change that would fix the weak checks (usually: fewer agents, a narrower task, or adding an eval command) and get the user's answer before creating anything |
| 0–2 | **DO NOT RUN** — recommend doing the work directly, and say what would change the verdict |

**Report the verdict to the user and stop there.** On PROCEED, say so and create the
session. On the other two, create **nothing** — no session directory, no `environment.md`
— and wait. A session left at `init` is clutter that a later orchestrator has to
interpret, and its per-agent databases are gigabytes each.

Deliver it in this shape, short enough to read at a glance:

```
Benefit evaluation — <task in a few words>

  Approach variance    ✗  spec pins the DDL, the error mapping and the ordering
  Discriminator        ✗  no eval metric; judging = reading 3 large diffs
  Independent failures ✓  three distinct FRs, no shared unknown
  Self-contained       ✓  one mergeable unit
  Cost floor           ✓  ~7 files, real size

  2/5 — DO NOT RUN.

  Three agents would converge on near-identical implementations because the
  spec has already made the decisions. Recommend implementing directly.

  Would change the verdict: an open design question, or an eval command that
  ranks the branches without a human reading them.
```

### Salvage the research either way

Front-loading shared facts — schema quirks, measured row counts, exact line numbers — is
genuinely useful, and it is the one part of the AgentHub workflow that keeps its value
when the verdict is DO NOT RUN. If you have already gathered such facts, **do not
discard them with the session.** Put them where the work will actually happen: the
implementation notes, the spec, or the relevant `.github/docs/` page. They stop the next
person re-deriving them regardless of how the work gets done.

---

## What It Does

Everything below runs **only after** the benefit evaluation returns PROCEED (or the user
overrode a weaker verdict).

### If arguments provided

Pass them to the init script:

```bash
python {skill_path}/scripts/hub_init.py \
  --task "{task}" --agents {N} \
  [--eval "{eval_cmd}"] [--metric {metric}] [--direction {direction}] \
  [--base-branch {branch}]
```

### If no arguments (interactive mode)

Collect each parameter:

1. **Task** — What should the agents do? (required)
2. **Agent count** — How many parallel agents? (default: 3)
3. **Eval command** — Command to measure results (optional — skip for LLM judge mode)
4. **Metric name** — What metric to extract from eval output (required if eval command given)
5. **Direction** — Is lower or higher better? (required if metric given)
6. **Base branch** — Branch to fork from (default: current branch)

### Output

Lead with the verdict line, so the record of *why* this run exists sits next to the run:

```
Benefit evaluation: 5/5 — PROCEED

AgentHub session initialized
  Session ID: 20260317-143022
  Task: Optimize API response time below 100ms
  Agents: 3
  Eval: pytest bench.py --json
  Metric: p50_ms (lower is better)
  Base branch: dev
  State: init

Next step: Run /hub:spawn to launch 3 agents
```

For content or research tasks (no eval command → LLM judge mode):

```
AgentHub session initialized
  Session ID: 20260317-151200
  Task: Draft 3 competing taglines for product launch
  Agents: 3
  Eval: LLM judge (no eval command)
  Base branch: dev
  State: init

Next step: Run /hub:spawn to launch 3 agents
```

## Baseline Capture

If `--eval` was provided, capture a baseline measurement after session creation:

1. Run the eval command in the current working directory
2. Extract the metric value from stdout
3. Append `baseline: {value}` to `.agenthub/sessions/{session-id}/config.yaml`
4. Display: `Baseline captured: {metric} = {value}`

This baseline is used by `result_ranker.py --baseline` during evaluation to show deltas. If the eval command fails at this stage, warn the user but continue — baseline is optional.

## After Init

Tell the user:
- Session created with ID `{session-id}`
- Baseline metric (if captured)
- Next step: `/hub:spawn` to launch agents
- Or `/hub:spawn {session-id}` if multiple sessions exist

## If the run is abandoned before `/hub:spawn`

A session that never spawns still leaves cruft. Clean it up in the same turn you decide
to abandon it, rather than leaving it for a future orchestrator to interpret:

- **Delete `.agenthub/sessions/{session-id}/`** — including `state.json`, which is
  untracked and survives a `git rm`. A directory sitting at `state: init` reads as
  pending work.
- **Drop any per-agent databases that were provisioned** (`agent_1`, `agent_2`, …).
  They are template clones and each is gigabytes; `provision_hub_dbs.sh --reset`
  recreates them in seconds when they are next needed.
- **Leave `hub-postgres` and its pristine template database running.** That container
  is what lets a future run skip the ~950 MB restore whose silent progress trips the
  600-second stall watchdog. Removing it is not cleanup — it re-arms the failure the
  shared-template design exists to prevent.
