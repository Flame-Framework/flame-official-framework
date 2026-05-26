# Project Context Builder — Workflow

> Read this file FIRST. It orchestrates the discovery flow that produces `<PROJECT_ROOT>/project-context.md`.

## Pre-flight

1. **Resolve project root.** Use the supplied `<project-path>`. If absent, use cwd.
2. **Refuse if invalid:**
   - No `pom.xml` at root → abort: "`<path>` is not a Mule application root (no `pom.xml` found). If you meant to start a new project, scaffold it first in Anypoint Studio (File → New → Mule Project → Generate from local file / Exchange), then re-run against the generated folder."
   - `pom.xml` packaging is not `mule-application` → abort: "`<path>` contains a `pom.xml` but its `<packaging>` is not `mule-application`. FLAME only operates on Mule applications. If this should be a Mule app, fix the packaging in `pom.xml`; otherwise scaffold a new Mule project in Anypoint Studio."
   - Resolved target path for the output file contains `CLAUDE.md` → abort: "project-context.md cannot be written next to CLAUDE.md."
   - Multiple `pom.xml` with `mule-application` packaging in subdirs → abort: "Mono-repos out of scope. Run from the specific project root."
3. **Detect existing file.** If `<PROJECT_ROOT>/project-context.md` exists, read it for diff context and notify the user: "Existing file found. I will regenerate it; the result will be shown for confirmation before writing."

---

## Phase 1 — Facts (read + confirm)

Goal: capture what the project IS. Auto-detect what's detectable, ask the user to confirm or correct, prompt for the rest.

### 1a. Auto-detect

Read (no writes):
- `pom.xml` — `<artifactId>`, `<groupId>`, `<version>`, `<parent>`, `<jdk.version>`, dependencies (api-spec dep matching `<artifactId>` → `raml-artifact` / `raml-version`)
- `mule-artifact.json` — `minMuleVersion`
- `src/main/mule/**/*.xml` — flow names; extract every `<http:request-config>`, `<db:config>`, `<sftp:config>`, `<sap:config>`, `<jms:config>`, `<vm:config>`, `<salesforce:*>`, `<wsc:config>`, `<email:*>`, `<os:config>`, `<anypoint-mq:config>` for **Integrations**; also scan for `<scatter-gather>`, `<parallel-foreach>`, `<ee:cache>`, `<scheduler>`, rate-limiting policies for **Implementation Footprint**
- `src/main/mule/common/error-handler.xml` (or equivalent) — detect presence of standard error handler, reprocess (ObjectStore-based), scheduler error handling, broker email notification, and custom error types (any `<raise-error type="...">` not in the cataloged set)
- `src/main/resources/properties/*.yaml` — **keys only** (never decrypt or read values from `secure-config-*.yaml`)
- `src/main/resources/api/` or `src/main/resources/*.raml` — RAML title / version if present
- `git config --get remote.origin.url` (best effort)

Infer:
- **Project identity** (artifactId, groupId, version, runtime, java, parent-pom)
- **Project layer** — SAPI / PAPI / EAPI from project name suffix or `pom.xml`. **Layer code** derived from layer: SAPI→`sys`, PAPI→`prc`, EAPI→`exp`, Broker→`broker`, Unknown→`unknown`.
- **Integrations** — one row per `<*-config>` found; system name from config name, protocol from connector type, direction inferred (listener = inbound, request = outbound). Also capture the connector element name (e.g. `http:request-config`) and the Mule config `name=` attribute as `Config Name`.
- **Domain entities** — RAML resource names, or file names in `src/main/mule/implementation/`
- **Related APIs** — hostnames in `<http:request-config>`
- **External constraints (detectable)** — timeouts in HTTP configs, content-types in RAML examples
- **Implementation Footprint** — one row per feature in the template's table. For each: `yes` if detected at least once; otherwise `no`. For `Scheduler`, capture the cron expression(s) in Notes. For `Custom error types`, list the type identifiers.

### 1b. User confirmation

For each auto-detected block (frontmatter, Integrations, Domain Entities, Related APIs, detected External Constraints, Implementation Footprint): present the candidate list with `Confirm all / Edit individual / Remove` options. Edits captured verbatim.

For **Project layer**: present inferred value with `Confirm / Override`. If the user can't tell, accept `Unknown` and mark frontmatter accordingly. `layer-code` follows the chosen layer automatically.

### 1c. User-supplied fields

Ask each as a single prompt with `skip` allowed:

- **External Constraints (free text)** — "Are there constraints not visible in code? (regulatory facts, blackout windows, retention, GDPR scope, batch SLAs, etc.) — skip if none."
- **Business Context** — "Does this project have business rules not visible in code? (multi-tenant binding, special domain logic, etc.) — skip if none. Only fill this if it would meaningfully help a developer understand WHY decisions were made."

Any skipped field is omitted from the file (no TODOs).

---

## Phase 2 — Pattern detection + triage

Goal: identify recurring patterns the project uses consistently, confirm them with the user, and route each confirmed pattern to either Project Conventions or Documented Deviations.

### 2a. Detection (read-only)

Scan for recurring patterns across these categories. A candidate qualifies if it recurs in **≥ 3 instances** OR **≥ 50% of relevant files** for the category. Below threshold = discard.

| Category | What to scan for |
|---|---|
| **DataWeave conventions** | Custom DW function modules (`src/main/resources/functions/`, `src/main/resources/dwl/functions/`); recurring imports across `.dwl` files; recurring function calls in transforms |
| **Flow structure conventions** | Recurring sub-flow templates, recurring error-handler shapes (specific `type=` patterns, specific `on-error-*` combinations), retry/reprocess patterns, scheduler-driven flows |
| **Configuration conventions** | Recurring HTTP request configs targeting the same downstream system, recurring header sets, recurring property-key prefix patterns |
| **Connector usage conventions** | Object Store usage patterns (e.g., error replay via OS), queue patterns, multi-system integration patterns (multiple downstream system APIs) |
| **Naming conventions** | Variable naming style consistently applied (≥ 50% of vars), folder partitioning schemes inside `src/main/resources/dwl/` |
| **Logging conventions** | Use of non-standard logger modules (e.g., custom loggers, json-logger) instead of the standard Mule `<logger>`; recurring log message templates or fields included consistently across loggers |
| **Domain organization** | Implementation folder partitioning (sub-folders for domain groups, files-per-entity, etc.) |

**Important:**
- One-off occurrences are NOT patterns. Discard.
- Inconsistencies and missing fields are NOT patterns. Discard. (Code-review will flag these at review time.)
- Patterns already aligned with FLAME standards are NOT recorded. After confirmation, if a pattern matches a known KB rule, mark it as "matches FLAME standard" and skip writing it to the file.

### 2b. Triage (one pattern at a time)

For each candidate pattern, present to the user:

```
**Pattern [N of M]** — <category>: <short description>

Evidence: <recurrence count and file:line references — e.g., "found in 12 of 15 transform XMLs after HTTP requests">

KB cross-reference: <one of>
  • Matches FLAME standard <rule-id> → if confirmed, we discard (standard already covers it)
  • Violates FLAME standard <rule-id> → if confirmed, we capture as a Documented Deviation (interview follows)
  • No KB position → if confirmed, we capture as a Project Convention (advisory)

Is this a real convention used in this project?
1. YES — keep it
2. NO — false positive, discard
```

**On user response:**

- **NO** → discard, move to next pattern.
- **YES + matches FLAME standard** → discard from this file (standard documents it), move on.
- **YES + no KB position** → record as Project Convention (Phase 2c).
- **YES + violates FLAME standard** → inform the user: "This convention violates `<rule-id>`. To keep doing it intentionally, we need to record it as a Documented Deviation — this requires an approver (from `config/approvers.yaml`) with a ticket reference, expiry ≤ 12 months, and a ≥ 2-sentence justification. Proceed?" → if yes, route to the deviation interview (Phase 2d); if no, record as Open Issue with `decision-required: code-fix`.

### 2c. Convention recording

For each pattern recorded as a convention, stage an entry with:
```yaml
name: "<short label>"
category: <DataWeave | Flow | Config | Naming | Connector | Logging | Domain>
description: "<what the convention is>"
evidence: <file:line refs or summary, e.g., "8 of 12 transforms import functions::sapToDate">
enforcement: advisory
```

### 2d. Deviation interview (only when 2b routes here)

For each finding routed to deviation flow:

1. **Non-waivable check.** If the violated rule matches the non-waivable list (security-testing, secrets, correlationId, TLS, B-I.2, or contains tokens `tls | secret | password | credential | cipher`), inform the user that this finding **cannot** be waived under any circumstance. Skip the interview. Stage as Open Issue with `decision-required: code-fix`.
2. **Re-validate evidence:** open the source file, slice the relevant lines, compute `sha256` → this becomes `proof-of-finding.snippet-hash`.
3. **Interview:**
   - `approved-by` — must contain name + ticket-ref (e.g. `M. Silva (ACME-1234)`). Reject session user's name. Reject `me`, `TODO`, anonymous handles. Validate against `config/approvers.yaml` — if not found, fall back to Open Issue with note "approver not in approvers.yaml".
   - `approved-on` — date ≤ today.
   - `expires` — date ≤ approved-on + 12 months. If user requests longer, fall back to Open Issue.
   - `justification` — ≥ 2 sentences. Reject one-liners.
4. **Compute `builder-signature`:** `sha256(salt || rule-id || canonical-scope || approved-by || approved-on || expires || snippet-hash)` where `salt` is from `~/.flame/builder-salt` (created if missing).
5. Assign next `DEV-NNN` (read existing IDs from current file, +1).

---

## Phase 3 — Ask for missed patterns

After auto-detected patterns are triaged, prompt the user:

> "I scanned and found N patterns. Are there other recurring conventions in this API that I might have missed?
>
> Examples: Object Store usage strategy, queue strategy, custom error envelope shape, idempotency key handling, retry/circuit-breaker behavior, custom subflow templates, common HTTP header sets, multi-tenant binding rules, scheduler frequency strategy, ..."

User can free-text any number of additions. For each:
- Capture name + description.
- `evidence` is `user-described` (no file:line refs).
- Ask the user: "Does this convention align with FLAME standards, or could it conflict with any?"
  - If aligned/neutral → record as Project Convention.
  - If user says it might conflict → ask which rule, route to the deviation interview.

---

## Phase 4 — Write

1. Render the file from `templates/project-context-template.md` with all collected data:
   - Frontmatter (project identity + `layer-code` + `last-confirmed-on: <today>` + `status`).
   - Phase 1 sections: Integrations (with Connector + Config Name columns), Domain Entities, Related APIs, External Constraints, Implementation Footprint (always included; empty list / all-`no` row set if nothing detected). Business Context only if filled.
   - Phase 2/3 sections: Project Conventions (list of convention entries). Documented Deviations (list of DEV entries). Open Issues (list of OI entries).
   - Change Log row: `<today> | project-context-builder | initial generation` (or summary of diff vs previous file).
2. Re-validate: `schema-version: 1` present; every deviation has all required fields; every `expires` ≤ approved-on + 12 months; every `approved-by` is in `config/approvers.yaml`; no deviation against non-waivable rules. Any failure → drop the offending deviation to Open Issues, surface the change to the user.
3. Compute `status`: `complete` if Phase 1 confirmation completed for every detected fact; otherwise `incomplete`.
4. Show the user the rendered file as a preview. Ask: "Confirm write?" (yes / cancel).
5. On `yes` → single `Write` to `<PROJECT_ROOT>/project-context.md`. On `cancel` → discard, no file changes.

## Post-write

- Print a summary: counts of confirmed integrations, domain entities, project conventions, documented deviations, open issues.
- Remind the user: "Other FLAME skills will read this file on next invocation. To regenerate, re-run discovery via mule-developer."
