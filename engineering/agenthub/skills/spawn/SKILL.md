---
name: "spawn"
description: "Launch N parallel subagents in isolated git worktrees to compete on the session task."
command: /hub:spawn
---

# /hub:spawn — Launch Parallel Agents

Spawn N subagents that work on the same task in parallel, each in an isolated git worktree.

## Usage

```
/hub:spawn                                    # Spawn agents for the latest session
/hub:spawn 20260317-143022                    # Spawn agents for a specific session
/hub:spawn --template optimizer               # Use optimizer template for dispatch prompts
/hub:spawn --template refactorer              # Use refactorer template
```

## Templates

When `--template <name>` is provided, use the dispatch prompt from `references/agent-templates.md` instead of the default prompt below. Available templates:

| Template | Pattern | Use Case |
|----------|---------|----------|
| `optimizer` | Edit → eval → keep/discard → repeat x10 | Performance, latency, size reduction |
| `refactorer` | Restructure → test → iterate until green | Code quality, tech debt |
| `test-writer` | Write tests → measure coverage → repeat | Test coverage gaps |
| `bug-fixer` | Reproduce → diagnose → fix → verify | Bug fix with competing approaches |

When using a template, replace all `{variables}` with values from the session config. Assign each agent a **different strategy** appropriate to the template and task — diverse strategies maximize the value of parallel exploration.

## Before Spawning: Requirements First, Then Model Selection

**The orchestrator thinks once; agents execute.** Parallel agents multiply every token
the orchestrator failed to provide up front — N agents each re-researching the same API
or re-deriving the same requirements is the fastest way to burn through session limits.

**1. Front-load complete requirements into each dispatch post:**
- Exact task plus acceptance criteria — what "done" means and what the judge will reward
- Pre-researched facts (API endpoints, SDK/package names, known gotchas, domain rules
  like sign conventions or formulas) — researched ONCE by the orchestrator, stated as
  facts the agents can trust without re-verification
- Hard constraints: schemas, env var names, invariants that must not change
- The agent's assigned strategy and how it differs from the other agents
- Absolute paths for anything outside the worktree (the dispatch/results board, any
  gitignored docs — worktrees only materialize committed files)

If requirements are ambiguous, resolve them with the user BEFORE spawning — a wrong
assumption baked into N dispatch posts is N times as expensive to fix.

**2. Pick the lowest capable model for each agent** and pass it via the Agent tool's
`model` parameter. Subagents inherit the orchestrator's model by default, which is
usually the most expensive option:

| Model | Use for |
|-------|---------|
| `haiku` | Mechanical work: applying a spelled-out recipe, renames, config changes, test scaffolding |
| `sonnet` | **Default.** Standard implementation: migrations, refactors, feature code with clear requirements |
| inherit / `opus` | Only when the task genuinely needs deep reasoning (novel architecture, subtle concurrency) — and prefer ONE strong agent over N |

The better the dispatch post, the cheaper the model that can execute it.

## What It Does

1. Load session config from `.agenthub/sessions/{session-id}/config.yaml`
2. For each agent 1..N:
   - Write task assignment to `.agenthub/board/dispatch/`
   - Build agent prompt with task, constraints, and board write instructions
3. Launch ALL agents in a **single message** with multiple Agent tool calls:

```
Agent(
  prompt: "You are agent-{i} in hub session {session-id}.

Your task: {task}

Read your full assignment at .agenthub/board/dispatch/{seq}-agent-{i}.md

Instructions:
1. Work in your worktree — make changes, run tests, iterate
2. Commit all changes with descriptive messages
3. Write your result summary to .agenthub/board/results/agent-{i}-result.md
   Include: approach taken, files changed, metric if available, confidence level
4. Exit when done

Constraints:
- Do NOT read or modify other agents' work
- Do NOT access .agenthub/board/results/ for other agents
- Commit early and often with descriptive messages
- If you hit a dead end, commit what you have and explain in your result",
  isolation: "worktree",
  model: "sonnet"  // lowest capable model — see "Before Spawning" above
)
```

4. Update session state to `running` via:
```bash
python {skill_path}/scripts/session_manager.py --update {session-id} --state running
```

## Critical Rules

- **All agents in ONE message** — spawn all Agent tool calls simultaneously for true parallelism
- **isolation: "worktree"** is mandatory — each agent needs its own filesystem
- **Set `model` explicitly on every Agent call** — lowest capable model for the dispatch (default `sonnet`); never silently inherit the orchestrator's model
- **Requirements are settled before spawn** — dispatch posts carry acceptance criteria and pre-researched facts; agents should never have to re-research what the orchestrator already knows
- **Never modify session config** after spawn — agents rely on stable configuration
- **Each agent gets a unique board post** — dispatch posts are numbered sequentially

## After Spawn

Tell the user:
- {N} agents launched in parallel
- Each working in an isolated worktree
- Monitor with `/hub:status`
- Evaluate when done with `/hub:eval`
