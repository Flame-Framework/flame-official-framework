# Deviation Loader — DEV Track

> Used by the code-review orchestrator and its sub-agents to validate `<PROJECT_ROOT>/project-context.md`, match findings against approved deviations, and emit the final cross-referenced report.
>
> This file is consulted ONLY when `--dev` flag is set. The RAML track ignores deviations.

## Why this file exists
The `project-context.md` file documents intentional deviations from FLAME standards, with hard expiry, evidence, and signature. Without strict load-time validation, the file would become a vector for silently waiving real failures. This loader enforces the contract.

## Load procedure (run ONCE during code-review pre-flight)

### 1. File presence
- If `<PROJECT_ROOT>/project-context.md` does not exist:
  - Continue review with no deviations honored and no project context.
- If file exists, proceed.

### 2. Schema check
- If frontmatter `schema-version != 1` → ignore the entire file, emit WARN: "Unknown project-context schema version; deviations not honored."
- If frontmatter is malformed (cannot parse YAML) → ignore the entire file, emit WARN.

### 3. Status check
- `status: incomplete` → emit WARN: "project-context.md has incomplete fields. Some context may be missing."

### 5. Per-deviation validation
For each `DEV-NNN` entry, set `review-status` according to the first failing rule:

| Check | Failure → status | Action |
|---|---|---|
| `expires < today` | `expired` | Ignored. Emit Claude Action listing expired DEVs to renew. |
| `approved-by` empty / contains `TODO` / contains `me` / contains the current session user | `invalid-approver` | Ignored. WARN. |
| `approved-by` not present in `config/approvers.yaml` | `unknown-approver` | Ignored. WARN. |
| `severity-cap != acknowledged` | `corrupt` | **Strong WARN**: "deviation file appears tampered. Ignoring all deviations." Stop processing further deviations and continue review with no deviations honored. |
| `builder-signature` missing or fails verification | `possibly-fabricated` | Ignored. WARN: "deviation `<DEV-NNN>` has invalid builder signature — possibly fabricated manually." |
| `proof-of-finding.file` does not exist | `stale-by-code-change` | Ignored. WARN: "deviation `<DEV-NNN>` references missing file." |
| Recomputed `snippet-hash` ≠ stored hash | `stale-by-code-change` | Ignored. WARN: "code under deviation `<DEV-NNN>` has changed since approval." |
| `rule-violated` does not resolve to a known KB rule | `unknown-rule` | Ignored. WARN. |
| `rule-violated` matches the **non-waivable list** below | `non-waivable` | Ignored. The finding it would have softened is reported at original severity. WARN. |
| All checks pass | `active` | Honored — cap the matching finding at `Acknowledged Deviation`. |

### 6. Build the active deviation map
For each deviation with `review-status: active`, store an entry keyed by `(rule-id, normalized-file-glob)` so sub-agent findings can be matched in O(1).

## Non-waivable rule list

A deviation referencing any of these rules is automatically `non-waivable` regardless of approver / signature / proof:

```
dev/sections/security-testing#*
dev/sections/secure-config-code-quality#secrets-*
dev/sections/logging-correlationid#correlation-id
dev/sections/error-handling-flow-logic#B-I.2
* containing token: tls | secret | password | credential | cipher
```

## Matching algorithm (run after sub-agents return findings)

For each finding `F` returned by a DEV sub-agent:

1. Extract `F.rule-id`, `F.file`, `F.flow` (when applicable).
2. Lookup the active deviation map: any entry where `entry.rule-id == F.rule-id` AND `F.file` matches at least one glob in `entry.scope.files` AND (if `entry.scope.flows` is set) `F.flow ∈ entry.scope.flows`.
3. If at least one match: rewrite `F.severity` to `Acknowledged Deviation`, attach `acknowledged-deviation: <DEV-NNN>` annotation. Keep the finding in the report.
4. If no match: leave `F.severity` unchanged.

**Crucially:** matching never **deletes** a finding. Acknowledged findings still appear in the report, just in a different section, and they do not consume the Attention Points budget or trigger FAIL.

## Verdict computation (replaces the standard PASS/FAIL step)

After matching:
- Count `Blocking` findings (after deviation rewrites). Acknowledged findings DO NOT count.
- Sum `Attention Point` values (only for findings still at Attention Point severity).
- Apply the existing rules from `tracks/dev/context.md`:
  - Any `Blocking` (≥ 1) → FAIL
  - Total Attention Points > 6 → FAIL
  - Otherwise → PASS

Apply staleness verdict modifier from §3 (12-month rule).

## Final report sections (orchestrator only)

The orchestrator output (per `tracks/dev/context.md`) gains two new sections, BEFORE the existing "Best Practice Suggestions":

### Acknowledged Deviations
For each finding rewritten to `Acknowledged Deviation`:
- **Rule ID** • **DEV ID** • **File / Line** • Description • Original severity
- One row per finding

### Known Open Issues
For each `OI-NNN` in the project-context file:
- **OI ID** • **Rule** • **Scope** • Notes • Decision required

These are informational. They do NOT change PASS/FAIL.

## Loader integrity self-check

Before processing any deviations:
1. Verify the loader can read `config/approvers.yaml`. If missing → all deviations get `unknown-approver` status. Emit WARN: "Create `config/approvers.yaml` to enable deviation honoring."
2. Verify the salt file `~/.flame/builder-salt` exists. If missing → all deviations get `possibly-fabricated`. Emit WARN: "Salt file missing; deviations cannot be signature-verified."

## What this loader will NEVER do
- Honor a deviation with `severity-cap` other than `acknowledged`
- Delete a finding from the final report
- Accept a deviation whose approver matches the session user
- Accept a deviation older than 12 months (`expires`) or where the underlying code has changed
- Honor a deviation against a non-waivable rule
- Modify `project-context.md` (read-only consumer)
