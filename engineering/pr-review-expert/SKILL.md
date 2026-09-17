---
name: "pr-review-expert"
description: "Orchestrated pull request review. Maps the PR's shape, runs /code-review high and /adversarial-reviewer as subagents, checks the diff against the project's own standards, executes the PR's tests, migrations, and deployment steps against production-state data, validates every finding against the code, discards weak findings, and reports a PR strength assessment with a fix / pending / ignore recommendation per finding. Use when the user asks to review a pull request, a branch, or a diff."
argument-hint: "[PR number | branch name | empty for the current branch]"
---

# PR Review Expert

The session that loads this skill is the **orchestrator**. The orchestrator maps
the PR, runs two reviewer subagents, runs its own project-standards pass,
executes the PR, and validates every candidate finding. Only the orchestrator
writes the report.

A review that only reads code accepts the author's test results, migration
results, and deployment claims secondhand. This skill runs them.

Target: `$ARGUMENTS`

## Rules for every run

1. **Evidence or discard.** A finding reaches the report only with a `file:line`
   at the PR head, lines the orchestrator read itself, and a concrete failure
   scenario.
2. **Docs before code.** Read the project instructions and the reference docs for
   touched modules before reading changed code.
3. **Reviewer output is candidate input.** No subagent verdict is final. Phase 6
   decides.
4. **Severity comes from impact.** Agreement between reviewers raises confidence.
   Agreement does not raise severity.
5. **No outward writes.** Never push, approve, comment, merge, or edit the PR's
   files. Never check out the PR in the user's working tree. Never create a local
   branch. Never run a command that writes to a production host. Phase 4 writes
   only to local containers and local databases, through the commands the project
   documents.
6. **Ask before a gated command.** When the project instructions require approval
   for a command, such as a production data sync, ask once before running it. Name
   what the command overwrites.
7. **Never pass `--comment` or `--fix`** to `/code-review`.
8. **No filler.** State a strength only when evidence supports it. Never add a
   finding to fill a section.
9. **Stable verdicts.** After the report, follow "Answering follow-up questions".

## Candidate finding format

Every reviewer, the standards pass, and the execution phase emit candidates in
this format:

```
ID: <CR-n | ADV-n | STD-n | EXE-n>
Severity: critical | warning | note
Location: path/to/file.py:123
Claim: <one sentence>
Failure scenario: <inputs or state> -> <wrong result>
Evidence: <what the reviewer read or ran>
Source verdict: CONFIRMED | PLAUSIBLE | none
Raised by: <reviewer or persona>
Quota finding: yes | no
```

## Phase 0 — Resolve the target

For a PR number `<N>`:

```bash
gh pr view <N> --json number,title,body,url,state,baseRefName,baseRefOid,headRefName,headRefOid,additions,deletions,changedFiles,commits,labels
git fetch origin <baseRefName>
git fetch origin pull/<N>/head
git rev-parse FETCH_HEAD        # must equal headRefOid; stop and report a mismatch
git worktree add --detach <scratchpad>/pr-<N> <headRefOid>
git -C <scratchpad>/pr-<N> diff <BASE_REF>...<headRefOid> > <scratchpad>/pr-<N>.diff
```

Set `BASE_REF` from the table below before writing the diff. Stop and report if
the diff is empty.

For a branch name, or no argument, run `gh pr view [<branch>] --json ...` to find
the open PR. If no PR exists, use the PR base branch that the project
instructions name. Otherwise use the repository default branch.

`git worktree add --detach` creates no branch and no upstream. If `headRefOid`
equals `git rev-parse HEAD` in the user's checkout, and the checkout is clean,
review the checkout directly and create no worktree.

For a GitLab merge request, use the `glab mr view` and `glab mr diff`
equivalents.

Record these values and use them in every later phase:

| Value | Meaning |
|---|---|
| `TARGET` | PR number, or branch name |
| `BASE_REF` | `origin/<baseRefName>` for an open PR. `baseRefOid` for a merged PR, because the base branch already contains a merge-committed head and the merge-base diff is then empty. |
| `HEAD_SHA` | `headRefOid` |
| `DIFF_RANGE` | `BASE_REF...HEAD_SHA` (diff from the merge base) |
| `REVIEW_ROOT` | absolute path of the worktree, or of the user's checkout |
| `DIFF_FILE` | absolute path of the saved diff |
| `PR_STATE` | `open` or `merged`, from the `state` field. Phases 7 and 8 use it. |

## Phase 1 — Map the shape

Build the Shape block from the PR metadata, `git diff --name-status DIFF_RANGE`,
and `DIFF_FILE`.

1. **Purpose.** State the intent of the PR in one sentence. List every spec,
   feature brief, or decision record that the PR body or the diff references.
2. **Size.** Count files, added lines, removed lines, and commits. Record the
   count of changed source lines separately from tests, docs, and migrations.
3. **Layers.** Classify each changed file: API router, worker, domain, model or
   schema, migration, integration, test, doc, config or infrastructure, other.
   Use the project's layout doc to map directories to layers.
4. **Blast radius.** For each changed non-test module, find its importers with
   Grep. Name the production entry points that reach it. Rate the radius:
   - CRITICAL: shared model, auth, payments, API contract, DB schema.
   - HIGH: module with more than three importers, shared config, env vars.
   - MEDIUM: internal change inside one module.
   - LOW: tests, docs, isolated UI component.
5. **Contract surfaces.** List changes to API routes, response schemas, DB schema,
   migrations, queue payloads, WebSocket or event frames, env vars, config keys.
6. **Orchestration flag.** Set the flag when the diff touches workers, queues,
   scheduled jobs, entry points, route wiring, loop ordering, or feature gates,
   or when it replaces a flow. The flag activates the project's required review
   format, if the project defines one.
7. **Tests and docs touched.** Count changed test files and changed doc files
   against changed source files.
8. **Scan leads.** Search `DIFF_FILE` with the pattern sets in
   [references/scan_patterns.md](references/scan_patterns.md) for each language
   in the diff. A match is a lead for Phase 3. A match is never a finding.
9. **CI status.** Record `gh pr checks <N>` as information. Raise no finding for
   missing or failing CI when project instructions or memory mark CI as
   non-gating.
10. **Execution scope.** Record whether the PR adds a migration, ships a release
    note or runbook, or changes deployment steps. Phase 4 uses these facts.

## Phase 2 — Launch the adversarial reviewer (background)

Launch this agent immediately after Phase 1. Use the Agent tool with
`subagent_type: general-purpose` and `run_in_background: true`. The
adversarial-reviewer skill needs no subagents, so the background tool set is
sufficient.

Prompt:

```
You are a reviewer subagent. You review code. You do not change code.

Target: <TARGET>. Head: <HEAD_SHA>. Base: <BASE_REF>.
The PR head is checked out at <REVIEW_ROOT>. Your working directory holds a
different revision. Read every file from <REVIEW_ROOT>. Run git as
`git -C <REVIEW_ROOT> ...`.

1. Read <REVIEW_ROOT>/CLAUDE.md, if it exists, for project conventions.
2. Invoke the adversarial-reviewer skill with the Skill tool. Args: --diff <DIFF_RANGE>
3. Run all three personas as that skill instructs.
4. Do not edit files, commit, push, or post comments.
5. Do not run containers, migrations, or tests. The orchestrator runs them
   against a database it is changing, so a parallel run corrupts both results.

Your final message is the only output the orchestrator receives. List every
finding in the candidate format below, with ID prefix ADV-. Set
"Quota finding: yes" on any finding you raised to meet a persona's minimum of
one finding, rather than because you found a defect. End with the skill's
verdict.

<paste the candidate finding format>
```

## Phase 3 — Project standards pass (orchestrator)

Run this pass while the adversarial reviewer works and while Phase 4's long steps
run in the background. Finish it before you read any reviewer output, so
reviewer findings do not bias it.

### 3a. Load the standards

Read these sources in order:

1. `CLAUDE.md` at the repository root, and any `CLAUDE.md` in a directory the diff
   touches.
2. `AGENTS.md`, and the documentation index that these files name.
3. Every per-module reference doc for a module the diff touches, end to end.
4. The docs those files link for architecture, change discipline, testing,
   documentation, schema changes, and code review format.
5. The memory files whose index description bears on review findings.

Write the Standards table: one row per rule, with its source `file:line`. Cover
every category below. For a category the project does not define, write
"no project rule found" and apply general practice.

| ID | Category | Extract |
|---|---|---|
| S1 | Layer boundaries | Which layer owns transactions, commits, HTTP errors, and business logic. Enforcement artifacts: ratchet baselines, sanctioned-exception lists, and who approves an exception. |
| S2 | Function reuse | Rules against parallel implementations, the generalize-first rule, rename rules. |
| S3 | Tests | When tests must ship, required test infrastructure, production-entry regression tests, rules against removing or weakening tests. |
| S4 | Documentation | When docs must ship, which doc tree holds which content, spec and brief updates. |
| S5 | Change safety | Deprecation policy, preservation maps, TODO policy, required review format and headings. |
| S6 | Data and schema | Migration workflow, idempotency, append-only tables, DDL location, DB constraints. |
| S7 | Execution | The full test command, the migration command, the container lifecycle commands, the procedure that provisions production-state data and its approval rule, the parallel-stack procedure for a worktree, and where release runbooks live. |

Read S7 first when Phase 1 set any execution scope, because Phase 4 needs it
before the rest of this pass.

### 3b. Check the diff against the standards

Read every changed source file in full at `REVIEW_ROOT`. Read the callers of
changed functions.

- **S1 boundaries.** For each changed file in a boundary-governed layer, compare
  it with `git show BASE_REF:<path>`. Record each net-new commit, HTTP error, or
  business rule in the wrong layer. Record each change to an enforcement baseline
  or sanctioned list, and whether the PR cites approval.
- **S2 reuse.** For each new function, search for an existing function that
  queries the same tables or performs the same operation. Grep for the key
  table, column, and verb names. Record a parallel implementation, and name the
  existing function. Record each renamed function or changed import alias.
- **S3 tests.** Map each behavior change to the test that exercises it. Record
  each behavior change with no test. Record each deleted, skipped, or narrowed
  test, and whether the PR cites approval. When the orchestration flag is set,
  require a test against the production entry point.
- **S4 docs.** Map each behavior change to the reference doc for that surface.
  Record each doc that now contradicts the code. Record each new doc in the wrong
  tree.
- **S5 change safety.** List every call that the diff removes from a router,
  worker, scheduled job, queue consumer, or top-level orchestrator. For each
  call, find the new production path, or record "none found". Record each new
  TODO or FIXME line that the project policy forbids.
- **S6 data and schema.** Check migrations against the project's migration
  workflow. Check new worker writes for retry idempotency. Check DB constraints
  in migrations, not only in ORM models.
- **Scan leads.** Resolve each Phase 1 lead to a candidate or to "no issue".

Write each result in the candidate format with ID prefix `STD-`.

## Phase 4 — Execute the PR (orchestrator)

Start this phase as soon as Phase 2 launches. Run every long step (a database
restore, a test suite) in the background, and continue Phase 3 while it runs.
Use the full commands that S7 records. Never substitute a shorthand the project
marks unavailable.

### 4a. Prepare the environment

1. **Run the code at `HEAD_SHA`.** When the project's containers mount the user's
   checkout and `REVIEW_ROOT` is a worktree, start a separate stack for the
   worktree with the project's parallel-stack procedure. When the project
   documents none, ask the user before you change what the containers mount.
2. **Record the starting state:** the database's migration revision, the running
   services, and the checked-out commit. Phase 4 restores none of them. The report
   states what changed.
3. **Provision production-state data** when the PR adds a migration or ships a
   runbook: production's schema, data, and migration revision. Use the project's
   documented procedure, and follow rule 6. A database built only from the
   repository's migrations lacks every object production gained outside them, so
   it cannot stand in for production.

### 4b. Rehearse the deployment

Run this step when the PR ships a release note or runbook, or changes deployment
steps.

1. Execute the documented steps in their documented order against the
   production-state database. Apply only the substitutions the local environment
   needs, such as a local compose overlay, and record each one.
2. Run every pre-flight and post-release check the runbook lists. Compare each
   result with the expected value the runbook states.
3. Each failed step, each result that differs from its expected value, and each
   step whose order exposes a code and schema mismatch is a candidate.

When the PR adds a migration and ships no runbook, run the project's migration
command from production's revision to the head at `HEAD_SHA`.

### 4c. Round-trip the migration

Run this step when the PR adds a migration. Upgrade to head, downgrade to
production's revision, and upgrade again. Take a data fingerprint before and
after each direction: row counts and aggregate sums for the tables the migration
touches. A failed direction, or a fingerprint that changes, is a candidate.

### 4d. Run the tests

1. Run the full suite at `HEAD_SHA` against the migrated production-state
   database.
2. When a test fails, run it at `BASE_REF` against the same data at the base
   revision. A test that fails at `HEAD_SHA` and passes at `BASE_REF` is a
   candidate. A test that fails at both is pre-existing.
3. When the PR description claims a mutation check, reproduce one.

### 4e. Record the results

Write each failure or mismatch as a candidate with ID prefix `EXE-`. Its evidence
is the command, the output lines, and the expected value. A reproduced failure
carries source verdict CONFIRMED.

Record every step as ran, failed, or skipped. Give the reason for each skip, and
the approval that would unblock it. The report's Execution section uses this
record.

## Phase 5 — Run /code-review high (foreground)

Use the Agent tool with `subagent_type: general-purpose` and
`run_in_background: false`. A foreground subagent keeps the Agent tool, so
`/code-review` can run its own subagents.

Prompt:

```
You are a reviewer subagent. You review code. You do not change code.

Target: <TARGET>. Head: <HEAD_SHA>. Base: <BASE_REF>.
The PR head is checked out at <REVIEW_ROOT>. Your working directory holds a
different revision. When you read a file, read it from <REVIEW_ROOT>.

1. Invoke the code-review skill with the Skill tool. Args: high <TARGET>
2. Do not pass --comment or --fix. Do not edit files, commit, push, or post
   comments.
3. Do not run containers, migrations, or tests. The orchestrator runs them.
4. If the Skill tool cannot load code-review, stop. Report
   "code-review unavailable" with the error text.

Your final message is the only output the orchestrator receives. The findings
display that /code-review renders does not reach the orchestrator. After
/code-review finishes, restate every finding it reported in the candidate
format below, with ID prefix CR-. Keep the CONFIRMED or PLAUSIBLE verdict that
/code-review assigned.

<paste the candidate finding format>
```

**Fallback.** If the subagent reports "code-review unavailable", invoke
`code-review` in the orchestrator with the Skill tool and args `high <TARGET>`.
Record the run mode in the report.

## Phase 6 — Validate every candidate

Wait for the completion notification of the adversarial reviewer and of every
Phase 4 background step. Do not poll. If a reviewer failed, record it as
"not run" with its error, and continue.

Pool all `CR-`, `ADV-`, `STD-`, and `EXE-` candidates. Apply these steps to each
one:

1. **Merge duplicates.** Merge candidates with one root cause into one finding.
   Keep every source ID.
2. **Check the citation.** Read the cited lines at `REVIEW_ROOT`, and quote them
   in the finding. Discard with D1 if the lines do not contain the claimed code.
   Discard with D2 if no concrete failure scenario can be written.
3. **Trace reachability.** Name the production entry point that reaches the code:
   route, worker, scheduled job, CLI command, or deployment step. Discard with D3
   if no entry point reaches the failure path.
4. **Search for guards.** Check validation in callers, DB constraints in
   migrations, unique indexes, locks, `ON CONFLICT` clauses, transaction scope,
   and tests that already cover the scenario. Discard with D4 if a guard prevents
   the failure.
5. **Check intent.** Read the spec, reference doc, or decision record for the
   behavior. Discard with D5 if the docs state that the behavior is intended. If
   the docs and the code disagree, convert the candidate to an Intent Question.
6. **Check standing rulings.** Compare the candidate with the project instructions
   and memory feedback. Discard with D6 if a ruling excludes the candidate, and
   cite the ruling.
7. **Separate introduced from pre-existing.** Compare with
   `git show BASE_REF:<path>`. Discard with D7 a defect that exists identically
   at base and that the PR does not extend. Keep it as a PENDING candidate only
   if it is material and the pending register does not already track it.
8. **Settle the claim by execution.** Use the Phase 4 environment to run the
   query, the targeted test, or the deployment step that decides the claim.
   Record the command and its output as evidence. Discard with D11 when the
   command ran and the failure did not occur. Never state production user
   behavior from local data when the project forbids that inference.

Assign one verdict:

- **CONFIRMED** — steps 2 to 8 pass, and the failure scenario traces end to end.
- **PLAUSIBLE** — the code confirms the mechanism, but one trigger condition is
  unverified and no local execution can reach it. Write that condition as a
  testable check. Keep a PLAUSIBLE finding only at severity critical or warning.
  Discard a PLAUSIBLE note with D8.
- **DISCARDED** — any discard code applies.

| Code | Discard reason |
|---|---|
| D1 | The cited lines do not show the claimed code. |
| D2 | No concrete failure scenario exists. |
| D3 | No production entry point reaches the failure path. |
| D4 | An existing guard prevents the failure. |
| D5 | The docs state that the behavior is intended. |
| D6 | A standing project ruling excludes the finding. |
| D7 | The defect is pre-existing, and the PR does not extend it. |
| D8 | The finding is a note-level concern with an unverified trigger. |
| D9 | The finding is a style or lint issue that the project's linter owns. |
| D10 | The finding is a quota finding with no defect behind it. |
| D11 | Execution refutes the failure scenario: the command ran and the failure did not occur. |

Re-rate the severity of every kept finding from its impact:

- **critical**: data loss, a wrong balance or ledger entry, a security breach, a
  production outage, a migration that fails on production-state data, or
  unapproved removal of live behavior.
- **warning**: wrong behavior in a reachable edge case, a missing required test
  or doc, a net-new boundary violation, or a parallel implementation.
- **note**: a maintainability issue with no behavior impact.

## Phase 7 — Assign a disposition

| Disposition | Use when |
|---|---|
| **FIX** | A critical or warning finding that the PR introduces. A test or doc that the project requires in the same change. A net-new boundary violation. A parallel implementation. Fix before merge, or on a follow-up branch when `PR_STATE` is `merged`. |
| **PENDING** | The finding is real and material, but its fix carries its own risk, needs a lead-developer decision, or is outside the PR's scope. Pre-existing debt the review surfaced belongs here. |
| **IGNORE** | The finding is confirmed but immaterial: the cost of the change exceeds the risk. |
| **QUESTION** | The finding depends on intent that the code and docs cannot settle. |

Apply these rules:

- A defect that this PR introduces is FIX. Scope is not a reason to defer it.
- A doc or test that the project requires in the same change is FIX. It is never
  PENDING.
- Every IGNORE names the evidence that makes the finding immaterial.
- Never change a FIX to PENDING to make the PR look finished.
- **Merged PRs.** When `PR_STATE` is `merged`, a FIX means a change on a
  follow-up branch from the base branch, before the next release. The merge
  does not lower a FIX to PENDING. The defect is live, so the fix is more urgent,
  not less.
- Find the pending register through the project instructions, memory, or a Glob
  for `**/pending_items*.md`. Draft each PENDING entry in the register's tier
  structure, numbering, and entry style. Separate confirmed facts from unverified
  conditions. Include "Raised on the PR #<N> review (<YYYY-MM-DD>)". Do not write
  the entry until the user approves it.

## Phase 8 — Write the report

Assign every kept finding to exactly one dimension:

| Dimension | Scope |
|---|---|
| Correctness | Product behavior: code that a production entry point reaches (routes, workers, scheduled jobs, domain, migrations). |
| Boundaries and reuse | Layer boundaries and duplicated implementations (standards S1, S2). |
| Tests | Test presence and test quality for product code (S3). |
| Docs | Reference docs, specs, and briefs (S4). |
| Change safety | Deprecation, contract changes, migrations, deployment runbooks, preservation maps (S5, S6). |
| Scope and shape | PR size, bundled unrelated changes, split recommendations. |
| Tooling | Developer tooling that no production entry point reaches: hooks, scripts, CI configuration, local dev config, and the tests of that tooling. |

A tooling defect counts toward Tooling, never toward Correctness or Tests.

Rate each dimension with the first rule that matches:

- **N/A**: the diff does not touch the dimension's scope.
- **Failing**: at least one FIX critical.
- **Weak**: at least one FIX warning, or at least one PENDING finding.
- **Adequate**: only IGNORE, QUESTION, or note-level findings; or no kept
  finding and no positive evidence.
- **Strong**: no kept finding, and positive evidence exists. Name the evidence.
  Correctness and Tests need a passing suite at `HEAD_SHA` from Phase 4. Change
  safety needs a rehearsed migration or runbook from Phase 4 when the PR has one.

Rate the PR overall with the first rule that matches. Use the label for
`PR_STATE`:

| Rule | Open PR | Merged PR |
|---|---|---|
| At least one FIX critical, or a removed production call with status "none found" or "unclear" | **Not mergeable** | **Urgent follow-up** |
| At least one FIX | **Needs changes** | **Needs follow-up changes** |
| No FIX, and at least one PENDING | **Mergeable with tracked follow-ups** | **Sound, with tracked follow-ups** |
| No FIX, and no PENDING | **Strong** | **Strong** |

When Phase 4 skipped a step that the execution scope requires (the suite, a
migration, a runbook rehearsal), append "unverified: <step>" to the overall
rating.

In the overall sentence, state how many FIX findings are product findings and
how many are Tooling findings.

Report template:

````markdown
## PR Review: <title> (#<N>)

**Overall: <rating>.** <One sentence: the deciding reason, with the product / Tooling FIX counts.>
PR state: <open | merged — each FIX is follow-up-branch work>.
Head `<short HEAD_SHA>` against `<BASE_REF>`. <files> files, +<added> / -<removed>.
Reviewers: /code-review high (<subagent | inline | not run>), /adversarial-reviewer (<ran | not run>), standards pass (ran), execution (<ran | partial | not run>).
Candidates: <raised> raised, <kept> kept, <discarded> discarded.

### Shape
- **Purpose:** ...
- **Layers:** ...
- **Blast radius:** <rating>. <reason>
- **Contract surfaces:** ...
- **Tests / docs touched:** ...
- **Orchestration flag:** <set | not set>
- **Execution scope:** <migration | runbook | deployment steps | none>

### Strength assessment
| Dimension | Rating | Evidence |
|---|---|---|
| Correctness | | |
| Boundaries and reuse | | |
| Tests | | |
| Docs | | |
| Change safety | | |
| Scope and shape | | |
| Tooling | | |

### Execution
| Step | Command | Result | Evidence |
|---|---|---|---|
| Production-state data | | | |
| Deployment rehearsal | | | |
| Migration round trip | | | |
| Test suite at HEAD_SHA | | | |

Local state left: <database and its migration revision, stopped or started services, checked-out commit>.

### Findings
#### F1 · FIX · critical — <claim>
- **Location:** `path:line`
- **Code:** <quoted lines>
- **Failure scenario:** ...
- **Validation:** <verdict>. Entry point: ... Guards checked: ... Execution: ... Base comparison: introduced.
- **Sources:** CR-2, ADV-1, EXE-1
- **Recommendation:** <the fix, in one to three sentences>

<Order the findings: FIX critical, FIX warning, PENDING, QUESTION, IGNORE.>

### Required review sections
<Include only when the orchestration flag is set and the project defines a
review format. Include every required heading, even when it is empty. Refer to
findings by F-number. Do not repeat their text.>

### Intent Questions

### Draft pending entries
<One block per PENDING finding. Not written to the register.>

### Standards applied
| ID | Rule | Source | Result |
|---|---|---|---|

### Discarded candidates
| Candidate | Claim | Code | Reason |
|---|---|---|---|

### Not verified
<Each check that the review could not run, the reason, and the approval that would unblock it.>

Worktree: `<REVIEW_ROOT>` (kept for follow-up questions).
````

## Answering follow-up questions

When the user questions a finding after the report:

1. Re-open the recorded evidence and the code at `HEAD_SHA` before answering.
2. Identify what the question adds: new evidence, a missed guard, a ruling, or a
   different reading of intent.
3. When a command in the Phase 4 environment can settle the question, run it
   before answering.
4. Change a verdict or disposition only when new evidence contradicts the
   recorded evidence. Cite that evidence as `file:line`, a doc line, or command
   output.
5. When the review missed something, name the validation step that missed it.
6. When the evidence still supports a finding, keep the finding. State what
   evidence would change it.
7. Record each change on one line:
   `Revised F3: FIX -> IGNORE. Evidence: <file:line>. Missed at: step 4.`

A question is not evidence. Pushback alone never changes a verdict. New evidence
always does.

## Cleanup

Keep the worktree, and any separate stack Phase 4 started, while follow-up
questions continue. When the user closes the review or starts a different task,
tear down that stack with the project's documented command, then run:

```bash
git worktree remove --force <REVIEW_ROOT>
git worktree prune
```

Never remove `REVIEW_ROOT` when it is the user's checkout. Phase 4 does not
restore the local database. Tell the user its final state before cleanup.
