# Scan patterns

Phase 1 of `pr-review-expert` searches the saved diff with these patterns. Search
added lines with the prefix `^\+` and removed lines with the prefix `^-`. Use the
Grep tool with `output_mode: content` and `-C 3` for context.

A match is a **lead**. Phase 3 reads the full file and decides whether the lead
is a candidate finding. Never report a match as a finding.

Patterns are regular expressions. Select the sets for the languages in the diff.

## Python, FastAPI, SQLAlchemy — added lines

- `text\(f["']` or `execute\(f["']` or `\.format\(.*(SELECT|INSERT|UPDATE|DELETE)` —
  SQL built by string interpolation.
- `\.commit\(` — a transaction commit. Check the file's layer against standard S1.
- `raise HTTPException` or `except HTTPException` — an HTTP error outside the
  router layer. Check against S1.
- `except Exception:` or `except:` — a broad catch. Check whether the handler
  swallows the error.
- `datetime\.utcnow\(` or `datetime\.now\(\)` — a naive timestamp.
- `float\(` — float arithmetic. Check whether the value is an amount, a balance,
  or a price that requires `Decimal`.
- `time\.sleep\(` or `requests\.` — a blocking call. Check whether it runs inside
  `async def`.
- `\.all\(\)` — an unbounded result set. Check for a limit on request paths.
- `while True` — a loop. Check its exit condition and backoff.
- `insert\(` — a new insert. Check worker and domain inserts for retry
  idempotency, such as `on_conflict_do_nothing`.
- `for .* in .*:` followed within three lines by `db\.|session\.|select\(` — a
  query inside a loop (N+1 query).
- `(password|secret|api_key|token|private_key)\s*=\s*["'][^"']{8,}` — a hardcoded
  secret.
- `\b5[HJK][1-9A-HJ-NP-Za-km-z]{49}\b` — a Wallet Import Format (WIF) private key,
  the format Hive keys use.
- `logger\.\w+\(.*(key|token|password|secret)` — a secret in a log line.
- `\beval\(|\bexec\(|pickle\.loads|yaml\.load\(|shell=True` — code execution.
- `os\.getenv\(|os\.environ` — a new config key. Check its documentation and its
  production value.
- `TODO|FIXME|XXX` — check against the project's TODO policy (S5).
- `create_engine\(["']sqlite` — SQLite in tests. Check against S3.
- `pytest\.mark\.skip|pytest\.skip\(|xfail` — a skipped test. Check against S3.

## Python — removed lines

- `^-\s*(async\s+)?def test_` — a deleted test. Check against S3.
- `^-.*@router\.(get|post|put|patch|delete)\(` — a removed route. Check the API
  contract.
- `^-\s+\w+\s*:\s*[\w\[]` in a schema file — a removed response field. Check the
  API contract.
- `^-.*\w+\(` in a router, worker, scheduler, or orchestrator file — a removed
  call. Trace the new production path under S5.
- `^-.*(os\.getenv|os\.environ)` — a removed config key.

## Alembic and SQL

- A migration file with status `M` in `git diff --name-status` — an edit to a
  migration already in history. Check against S6.
- `op\.drop_table|op\.drop_column|DROP (TABLE|COLUMN)` — a destructive schema
  change.
- `nullable=False` without `server_default` on an added column — this change
  fails on a populated table.
- `op\.execute\(.*CREATE (OR REPLACE )?VIEW` — view DDL inside a migration. Check
  the project's rule for DDL location.
- `drop_index|DROP INDEX` — a removed index. Check query plans that use it.
- `CheckConstraint|ck_` — a CHECK constraint. Check that the ORM model and the
  migration agree.

## TypeScript and JavaScript — added lines

- `dangerouslySetInnerHTML|innerHTML\s*=` — a cross-site scripting (XSS) vector.
- `\beval\(|new Function\(` — code execution.
- `__proto__|constructor\[` — prototype pollution.
- `localStorage\.setItem\(.*(token|session|key)` — a credential in web storage.
- `process\.env\.[A-Z_]+|import\.meta\.env\.[A-Z_]+` — a new config key.
- `:\s*any\b|as any\b` — a type escape.
- `useEffect\(` — an effect. Check its dependency array.
- `fetch\(|axios\.` — a network call. Check its error handling.
- `console\.log|debugger` — debug residue.

## TypeScript and JavaScript — removed lines

- `^-.*(export\s+)?(interface|type)\s` — a removed type. Check its consumers.
- `^-.*(process\.env|import\.meta\.env)\.[A-Z_]+` — a removed config key.
