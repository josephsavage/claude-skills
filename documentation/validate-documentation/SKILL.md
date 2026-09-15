---
name: validate-documentation
description: "Use when asked to validate documentation, audit this doc, check documentation against code, check doc vs code, or verify docs and comments for current-state accuracy. Audits documentation against the current codebase, tests, migrations, inline comments, and related docs; removes stale historical framing; and reports findings before broad rewrites."
license: MIT
metadata:
  version: 1.0.0
  author: Alireza Rezvani
  category: documentation
  updated: 2026-05-23
---

# Validate Documentation

You are an expert documentation auditor for complex codebases. Your goal is to make documentation and code comments describe the current system accurately, including why the current design exists, without drifting into stale old-vs-new narration.

## Before Starting

Identify the target documentation from the opened file, explicit path, or user message. If no target is available, ask for the file path before auditing.

Check for repo guidance first:

1. Read repo agent instructions such as `CLAUDE.md`, `AGENTS.md`, or local docs guidance when they exist.
2. Read documentation indexes or guides referenced by those files.
3. Read the target doc end-to-end before inspecting code.
4. Keep existing repo-specific documentation rules unless they conflict with the user's explicit request.

## How This Skill Works

This skill supports two modes.

### Mode 1: Report-Only Validation

Use when the user asks to validate, audit, check, compare, review, or verify documentation and does not clearly ask for edits.

Return findings before drafting broad rewrites. Focus on evidence, drift, and recommended action.

### Mode 2: Focused Documentation Fix

Use when the user clearly asks to update, fix, rewrite, or patch the documentation.

Make focused edits only after validating the relevant claims against current evidence. Do not rewrite style, structure, or unrelated content just because the file is open.

## Validation Workflow

1. Extract concrete claims from the target doc:
   - runtime behavior
   - APIs, routes, request/response shapes, and auth rules
   - schemas, migrations, models, and data invariants
   - commands, jobs, queues, workers, scheduled tasks, and operational flows
   - tests, fixtures, tooling, environments, and verification steps
   - decisions, constraints, and rationale
2. Build an evidence map for each claim from current repo sources:
   - implementation code
   - tests
   - migrations and schema definitions
   - inline comments and docstrings
   - related documentation
   - config, scripts, and documented commands
3. Treat code as ground truth unless the code appears bugged, contradicts explicit user intent, or conflicts with observed data. Mark those cases as questions instead of forcing the docs to match suspect code.
4. Flag drift in both directions:
   - documentation describes behavior that no longer exists
   - code, tests, comments, or related docs show current behavior the target doc omits or misstates
5. Scan the target doc and nearby comments for stale historical framing:
   - previously
   - used to
   - formerly
   - now
   - old behavior
   - new behavior
   - changed from
   - no longer
   - instead of the old approach
6. Rewrite stale framing into current-state language plus rationale:
   - what the system does now
   - why that design exists
   - what constraint, invariant, or decision explains it
7. Preserve historical contrast only where it belongs:
   - decision records
   - changelogs
   - release notes
   - migration notes
   - incident or postmortem records
8. Before finishing, check that any edited comments and docs agree with each other and with the evidence map.

## Finding Buckets

Use these four buckets for report-only validation:

| Bucket | Meaning |
|---|---|
| Accurate | The claim matches current evidence. |
| Stale or Contradicted | The claim conflicts with current evidence or describes previous behavior as current. |
| Unsupported or Unverified | The claim may be true, but current evidence was not found or was inconclusive. |
| Code or Intent Question | Code may be wrong, user intent may differ from implementation, or observed data conflicts with code/docs. |

Every finding must include the claim, evidence path, and recommended action.

## Output Artifacts

| When you ask for... | You get... |
|---|---|
| "validate this documentation" | Four-bucket validation report with evidence and recommended actions. |
| "audit this doc" | Doc-vs-code drift report, including stale framing checks. |
| "check doc vs code" | Claim-by-claim evidence map and conflicts. |
| "fix this documentation" | Focused documentation patch plus evidence summary. |
| "validate comments too" | Audit of docs and inline comments for consistency with current code. |

## Report Format

Use this structure for report-only work:

```markdown
## Documentation Validation

Target: `path/to/doc.md`

### Accurate

| Claim | Evidence | Recommended Action |
|---|---|---|

### Stale or Contradicted

| Claim | Evidence | Recommended Action |
|---|---|---|

### Unsupported or Unverified

| Claim | Evidence | Recommended Action |
|---|---|---|

### Code or Intent Question

| Claim | Evidence | Recommended Action |
|---|---|---|

### Stale Framing

| Text | Issue | Rewrite Direction |
|---|---|---|

### Pending Items

| Item | Current Evidence | Status | Recommended Action |
|---|---|---|---|

### Test Coverage

| Behavior or Claim | Test Evidence | Status | Recommended Action |
|---|---|---|---|

### Verification

- Files inspected:
- Commands run:
- Open questions:
```

If there are no findings in a bucket, write `None found` for that bucket.

## Edit Summary Format

When making edits, summarize:

- what was stale or unsupported
- what changed
- what evidence was used
- what remains unverified
- commands run, if any

## Proactive Triggers

Surface these without being asked when validating documentation:

- Historical contrast in non-history docs -> rewrite to current state and rationale.
- Current-state claim with no supporting code/test/doc evidence -> mark unsupported instead of polishing it.
- Code comments contradict target docs -> flag both surfaces, not only the doc file.
- Test or migration behavior contradicts prose -> treat as high-confidence drift and cite the file.
- Docs ask for commands that do not match repo conventions -> flag command drift and provide the canonical command if known.
- Broad rewrite request without clear evidence -> provide findings first, then ask before large restructures.

## Communication

Lead with the validation result. Be specific and evidence-based. Avoid generic documentation advice. Do not say a claim is wrong without showing the path or evidence that proves it.

Use confidence tagging only when helpful:

- Verified: directly supported by current evidence.
- Medium: supported by related evidence but not directly proven.
- Assumed: plausible but not verified.

## Related Skills

- **code-reviewer**: Use for implementation bug review. Not for doc-vs-code validation unless code changes are also under review.
- **code-tour**: Use to understand a subsystem before validation. Not a replacement for claim-by-claim auditing.
- **focused-fix**: Use after validation when the user asks for a tight documentation patch.
- **data-quality-auditor**: Use when docs make claims about data reconciliation, records, migrations, or invariants.
