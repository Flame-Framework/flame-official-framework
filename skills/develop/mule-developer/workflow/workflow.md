# Mule Developer — Workflow Details

This file contains the detailed procedures for each step in the mule-developer skill workflow. The skill file defines the step sequence and branching logic; this file provides the "how" for each step.

---

## Project Root Resolution Protocol

**Runs first, before the Checkpoint Resume Protocol.** Converts the `<project-name>` argument into an absolute `PROJECT_ROOT` path, with a user dialogue when nothing matches.

1. **Read workspaces.** Load `workspaces:` from `config/framework.yaml`. Expand `~` to the user's home directory.
2. **Search each workspace.** For each workspace path, look for a direct child folder named `<project-name>`. If `organization.project-prefix` is configured and `<project-name>` does not already start with it, also try `<prefix><project-name>`.
3. **One match** → set `PROJECT_ROOT` to that absolute path and continue.
4. **Multiple matches** → use `AskUserQuestion` to ask which workspace the user means; each candidate path is an option. Set `PROJECT_ROOT` to the chosen path.
5. **No match** → use `AskUserQuestion` with three options:
   - **New project I want to set up** → invoke the `scaffold` skill per the "Scaffold Invocation Contract" below, then continue with the Checkpoint Resume Protocol and Step 0 using the `PROJECT_ROOT` it returns.
   - **Existing project at a different location** → ask the user for the full absolute path. Validate it exists, contains a `pom.xml` with `packaging` = `mule-application`, then set `PROJECT_ROOT` and continue. If the path is invalid, re-ask up to twice; on the third failure, stop.
   - **I mistyped the name** → ask for the correct name and restart this protocol from step 2 with the new name.

### Scaffold Invocation Contract (new-project branch)

Invoke `scaffold` with the same `<project-name>` argument. `scaffold` runs autonomously, deriving everything it can from `config/framework.yaml`, the project-name suffix (layer), and MCP (Exchange asset, group ID, RAML endpoints) — and asks the user only for genuine ambiguities (multiple workspaces, multiple Exchange matches, MCP disabled, etc.). See `skills/develop/scaffold/workflow/workflow.md` for the full procedure.

Wait for `scaffold` to finish. Parse its final marker line:

- **`SCAFFOLD_OK PROJECT_ROOT=<absolute-path>`** → set `PROJECT_ROOT` to that path and continue (no user prompt — proceed straight to the Checkpoint Resume Protocol and Step 0).
- **`SCAFFOLD_ABORTED reason="..."`** → stop the workflow cleanly. Surface `scaffold`'s reason to the user verbatim; do NOT proceed to the Checkpoint Resume Protocol or Step 0.

The scaffold returns a compile-ready, KB-compliant project skeleton with the RAML imported, APIkit configured, properties files in place, and per-endpoint stub flows ready for ND-x to fill in. `project-context-builder` (invoked in Step 0) will then discover the just-scaffolded structure and produce `project-context.md`.

---

## Checkpoint Resume Protocol

When starting a session, check for an existing `.mule-dev-checkpoint.md` in `PROJECT_ROOT`:

1. **If checkpoint exists**: Read it and extract `last_completed_step`, `task_type`, and stored variables (`REQUIREMENTS`, `APPROVED_PLAN`, etc.). Project facts are not stored in the checkpoint — they are re-read from `project-context.md` at Step 0 every session.
2. **Identify resume point**: The next step after `last_completed_step`.
3. **Load only needed KB batches**: Skip batches for already-completed steps. Load only batches from the resume point forward.
4. **Confirm with user**: Show the checkpoint summary and ask: "Resume from Step X, or start fresh?"
5. **If user says start fresh**: Delete the checkpoint and begin from Step 1.

If the checkpoint file is corrupted or references unknown steps, delete it and start fresh.

---

## Error Recovery

When something goes wrong during implementation, follow these guidelines:

| Situation | Action |
|-----------|--------|
| **Generated XML is invalid** | Re-read the layer reference flow example. Compare structure element by element. Fix namespace declarations, closing tags, and attribute syntax. |
| **Connector config doesn't match KB** | Re-read `reference/connectors.md` and `usage/connector.md`. Never guess connector attributes — if not documented, ask user. |
| **User rejects the plan** | Ask what specifically needs to change. Do NOT regenerate the entire plan — make targeted adjustments and re-present only the changed sections. |
| **DataWeave compilation error** | Check DWL syntax against `usage/dataweave.md`. Common issues: missing `output` directive, wrong MIME type, unescaped special characters. |
| **Missing dependency** | Check `reference/connectors.md` for correct groupId/artifactId/version before adding to pom.xml. |
| **Config property not found** | Check `standards/configuration-properties.md` for correct YAML structure and property path. Verify all 4 env files (local/dev/qa/prod) have the property. |
| **Unsure about a pattern** | STOP and ask the user. Never invent patterns not in the KB. |
| **Unsure about logic, sequence, or orchestration** | STOP and ask the user via `AskUserQuestion`. Never assume execution order, branching, aggregation, or error-compensation behavior. For PAPI, prefer asking over assuming even when the requirement "seems clear". |

---

## Checkpoint File Format

**Location**: `PROJECT_ROOT/.mule-dev-checkpoint.md`

```markdown
# Mule Developer — Session Checkpoint

## Session
- Project: {PROJECT_ROOT}
- Last completed step: {N}

## Variables
- PROJECT_ROOT: {value}
- TASK_TYPE: {NEW_DEV|CR_BUGFIX}
- RAML_VERSION: {value}
- REQUIREMENTS_PATH: {path to requirements files}

## KB Files Loaded
- [x/  ] api-development-best-practices.md
- [x/  ] api-layers.md
- [x/  ] cross-skill-integration.md
- [x/  ] pre-flight-validation.md
- [x/  ] naming-conventions.md
- [x/  ] dataweave-patterns.md
- [x/  ] logging.md
- [x/  ] usage/logging.md
- [x/  ] error-handling.md
- [x/  ] configuration-properties.md
- [x/  ] dataweave-functions.md
- [x/  ] error-types.md
- [x/  ] error-handling.md
- [x/  ] reference-flows.md

## Completed Steps
- [x] Step 1: {brief outcome}
- [x] Step 2: {brief outcome}
- ...
```

---

## Step 0 — Verify Project Context

**Mandatory gate. Runs before ND-1 / CR-1 / Step 3, regardless of task type.**

1. Check for `<PROJECT_ROOT>/project-context.md`.
2. **If missing** → invoke `project-context-builder` with `<PROJECT_ROOT>` as argument. Wait for it to complete. Then re-check:
   - If now present → proceed.
   - If still missing → the user cancelled the builder. Abort with: "project-context.md is required to develop in this project. Re-run when you are ready to capture context."
3. **If present** → read its frontmatter and the sections `Business Context`, `External Constraints`, `Integrations`, `Related APIs`, `Domain Entities`, `Implementation Footprint`, `Project Conventions`. Do NOT read `Documented Deviations` or `Open Issues` — those are review-only.
4. **If `status: incomplete`** → invoke `project-context-builder` again to finish the missing fields, then re-read.
5. From this point on, `project-context.md` is read-only — only `project-context-builder` writes to it.

---

## Step ND-1 — Fetch RAML from Exchange

1. Extract asset name from `PROJECT_ROOT` folder name.
2. **Check `pom.xml` first** for an existing RAML asset dependency (a `<dependency>` whose `<artifactId>` matches the asset name AND `<classifier>raml</classifier>`):
   - **Match found** (most common when scaffold or a previous run already declared it):
     - Capture the declared version as `RAML_VERSION_LOCAL`.
     - **`[USER INPUT]`** — "RAML `<asset>` is already pinned at `<RAML_VERSION_LOCAL>` in pom.xml. Use this version, or fetch a newer one from Exchange?" — options: *Use the pinned version* / *Fetch latest from Exchange* / *Specify a version*.
     - If user picks *Use the pinned version* → set `RAML_VERSION = RAML_VERSION_LOCAL`, jump to step 7.
     - If user picks *Fetch latest from Exchange* → fall through to step 3.
     - If user picks *Specify a version* → ask for the version string, set `RAML_VERSION`, update `pom.xml`, jump to step 7.
   - **No match** (greenfield without scaffold, or pom.xml lost the dep somehow) → fall through to step 3.
3. Call `mcp__anypoint__anypoint_search_assets` with `search` = asset name, `types` = `["raml"]`.
4. If not found → STOP, inform user. (For scaffolded projects sourced from a local RAML, this branch should never be reached because step 2 will have matched.)
5. Call `mcp__anypoint__anypoint_get_asset` with `groupId` and `assetId` (omit version for latest).
6. If MCP fails → STOP, inform user about Anypoint MCP server.
7. Store `RAML_VERSION`, update `pom.xml` only if the new version differs from what's already declared.
8. **`[USER INPUT]`** — Show user the resolved RAML version and ask: "Is this the correct version to implement? Any endpoints from this RAML you want to skip?"

---

## Step ND-2 — Gather Technical Requirements

**`[USER INPUT]`** — Ask the user:
> "Please provide the path(s) to your requirements files (solution design, Jira ticket, specs document, etc.)"

Read all provided files. For each endpoint found, extract and present to the user for confirmation:

1. **Endpoints summary** — Show a table:
   | # | Method | Path | Description |
   Ask: "Are these all the endpoints to implement? Any missing or to be excluded?"

2. **Data mappings** — For each endpoint, show source → target field mapping.
   Ask: "Are these mappings correct? Any fields missing, renamed, or with special transformation logic?"

3. **Business rules & validations** — List extracted rules.
   Ask: "Any additional validation rules or business logic not captured in the requirements?"

4. **Error scenarios** — List error cases found (e.g., not found, timeout, validation failure).
   Ask: "Any additional error scenarios to handle?"

5. **External system dependencies** — List downstream APIs/systems identified.
   Ask: "Are these the correct target systems? Any additional systems involved?"

After all confirmations, store as `REQUIREMENTS`.

---

## Step ND-3 — Update Configuration Properties

> Reference: `standards/configuration-properties.md` (file layout, env-specific YAML, secure properties Blowfish+CBC, CloudHub DNS, externalization rules) — load if not yet in context.

1. Analyze `REQUIREMENTS` for config needs (URLs, paths, timeouts, credentials)
2. Confirm the standard file layout exists: `config-common.yaml`, `config-{local,dev,qa,prod}.yaml`, `secure-config-{local,dev,qa,prod}.yaml`. This file layout is mandatory for all API projects in this framework.
3. **`[USER INPUT]`** — Present proposed config changes as a table:
   | File | Property | Value | Action |
   Ask: "Do these config changes look correct? Any properties missing?"

4. For each downstream system, ask:
   > "What are the host/port/basePath values for {system} in local, dev, qa, and prod?"

5. For secure configs (`secure-config-*.yaml`):
   - Ask: "Do you have the encrypted values for these credentials? (must be `![encryptedValue]` format, encrypted with Blowfish CBC)"
   - Validate every value matches `![...]` pattern — STOP if plaintext detected
   - Verify the project's `<secure-properties:encrypt>` config uses `algorithm="Blowfish" mode="CBC"`

6. Verify all property keys use `dot.notation` lowercase (per `standards/configuration-properties.md` and `standards/naming-conventions.md`)

7. Verify the same key set exists across **all** env files (dev/qa/prod) — never have a key in one env without the others

8. **`[USER INPUT]`** — Present full summary of all config changes across all env files.
   Ask: "Confirm these config changes before I proceed?"

---

## Step ND-4 — Scan for Reusable Components

**KB required**: `standards/api-development-best-practices.md`, `standards/api-layers.md`, `standards/naming-conventions.md`

1. Scan sub-flows in all XML files under `src/main/mule/` — apply naming patterns from `naming-conventions.md` to spot conventional sub-flows
2. Scan connector configs in `global.xml` — match against the `{System}_{Type}_Config` naming pattern
3. Scan DWL files under `src/main/resources/dwl/` (correct subfolders: `payloads/`, `vars/`, `error/{message,statusCodes,reprocess}/`)
4. Search Anypoint Exchange for existing SAPIs/PAPIs that could fulfill requirements
5. **`[USER INPUT]`** — Present reuse report as a table:
   | Component | Type | Status | Recommendation |
   - Status: **Reusable as-is** / **Reusable with modification** / **Must create new**
   Ask: "Do you agree with the reuse assessment? Any components you'd prefer to create new instead of reusing?"
6. Store as `REUSE_PLAN`.

---

## Step ND-4.5 — Compliance Scan of Reusable Components

**KB required**: `cross-skill-integration.md` (loaded in Batch 2)

1. Load the compliance checklist from `cross-skill-integration.md` (section "Downstream: Mule Developer → Code Reviewer")
2. For each component in `REUSE_PLAN` marked as "Reusable as-is" or "Reusable with modification":
   - Scan for B-I.1 violations (error handling outside `error-handler.xml`)
   - Scan for B-I.2 violations (httpStatus 4xx/5xx outside error handlers)
   - Scan for B-I.3 violations (unreachable `isEmpty` checks)
   - Scan for B-II.2.1 violations (missing correlationId in HTTP requests)
   - Scan for B-III.1 violations (global configs in non-global.xml files)
   - Scan for B-IV.4 violations (inline CDATA where external DWL should be used)
3. If violations found in "Reusable as-is" components → reclassify as "Reusable with modification" (modification = compliance fix)
4. Compile `REUSE_VIOLATIONS` list with: file, rule ID, description, proposed fix
5. These will be included in the ND-5 implementation plan under "Compliance Fixes on Reused Components"

> **Performance note**: This is a lightweight scan using only the compliance checklist tables against files already read in ND-4. No additional file reads, no sub-agents, no interactive questions.

---

## Step ND-5 — Implementation Plan

Compile a plan based on all information gathered in ND-1 through ND-4. Present **progressively by phase** for complex implementations (4+ endpoints or 3+ sub-flows). For simpler implementations, a single plan is fine.

### Phase 1: Endpoints & Routing
0. **Compliance Fixes on Reused Components** (only if `REUSE_VIOLATIONS` is non-empty): list each violation with file, rule ID, and proposed fix. These are part of the plan for user approval.
1. API Layer confirmation and layer-specific patterns
2. Endpoints to create (method, path, flow name)
3. Implementation files (new or reuse existing)
4. Sub-flows (name, purpose, layer pattern: SAPI auth→request→map / PAPI orchestrate→aggregate→logic / EAPI validate→call→shape)
5. Connector configs (new or reuse from `REUSE_PLAN`)
6. DWL files (new or reuse, correct `dwl/` subfolder)
7. Data mappings (source → target per transformation)
8. Config properties added in ND-3
9. Loggers per flow (level + message)
10. Error handling patterns (from `project-context.md` Implementation Footprint)
11. Layer-specific special patterns
12. Parallelizable work

**`[USER INPUT]`** — Ask: "Confirm the endpoint structure before I continue to sub-flow design?"

### Phase 2: Sub-flows & Connectors
Present:
- Sub-flows (name, purpose, layer pattern)
- Connector configs (new or reuse from `REUSE_PLAN`)
- External system call sequence

**`[USER INPUT]`** — Ask: "Confirm the sub-flow structure and connector usage?"

### Phase 3: Data Transformations
Present:
- DWL files (new or reuse, correct `dwl/` subfolder)
- Data mappings (source → target per transformation)
- Variable extractions

**`[USER INPUT]`** — Ask: "Confirm the data mappings and DWL file structure?"

### Phase 4: Logging & Error Handling
Present:
- Loggers per flow (level + message)
- Error handling patterns (from `project-context.md` Implementation Footprint)
- Error response format per endpoint

**`[USER INPUT]`** — Ask: "Confirm logging and error handling before I start implementation?"

Store full approved plan as `APPROVED_PLAN`.

---

## Step ND-6 — Create Endpoints in Main XML

**KB required**: `api-development-best-practices.md`, `api-layers.md`

1. Extract endpoints from `APPROVED_PLAN`
2. Locate main XML (the file with `<apikit:config>` or `<http:listener-config>`)
3. New endpoints → create flow stubs per `api-layers.md`
4. Existing endpoints → do NOT touch without user approval
5. Unmentioned endpoints → leave untouched

---

## Steps ND-7 through ND-16

| Step | Action | Key Rules |
|------|--------|-----------|
| ND-7 | Create transform for query/URI params | Skip if endpoint has no params |
| ND-8 | Create implementation XML file(s) | Ask user if reusing existing file. Naming per `naming-conventions.md` |
| ND-9 | Create sub-flow stubs | Layer pattern: SAPI auth→req→map, PAPI orch→agg→logic, EAPI val→call→shape. **Use `usage/reference-flows.md` as scaffold** — copy the matching layer flow and adapt |
| ND-10 | Wire flow references | Follow execution order from `REQUIREMENTS` |
| ND-11 | Create connectors & outbound requests | Consult `usage/connector.md` and `usage/reference-flows.md`. Never hardcode URLs. DataWeave: use functions from `reference/dataweave-functions.md` |
| ND-12 | Apply data mappings | CRITICAL: never guess mappings. Ask user if ambiguous |
| ND-13 | Apply logging | KB: `standards/logging.md` (level + plain-text rules) + `usage/logging.md` (XML examples). Use the standard Mule `<logger>` everywhere. INFO at flow start/end; DEBUG around outbound calls when troubleshooting; ERROR inside `<on-error-*>` scopes. Propagate `vars.correlationId` as the `x-correlation-id` header on every outbound HTTP request |
| ND-14 | Apply error handling | KB: `standards/error-handling.md` + `reference/error-types.md` (only raise types from the catalog) + `usage/error-handling.md` (copy real handler patterns). Every `<on-error-*>` needs `doc:name`. Every `<raise-error>` needs `description` |
| ND-15 | Update pom.xml version | New feature = minor bump |
| **ND-15.5** | **Pre-flight validation** | Run full checklist from `workflow/pre-flight-validation.md` (7 categories). Auto-fix failures, re-run until OVERALL: PASS |
| ND-16 | Summary of changes | Present checklist + pre-flight report. Delete checkpoint file |

---

## Step CR-1 — Gather Change Request Details

Ask user for: description, affected endpoints/flows, requirement files/tickets. Summarize and confirm. Store as `CR_DESCRIPTION`.

---

## Step CR-2 — Analyze Existing Code

**KB required**: all dev KB files.

1. Identify affected files from `CR_DESCRIPTION` + `project-context.md` (Integrations, Implementation Footprint)
2. Read affected flows — document what each does, connectors, transforms, error handling, logging
3. Trace upstream/downstream dependencies
4. Check DWL files — note if shared
5. Check config properties used

Store as `IMPACT_ANALYSIS`.

---

## Step CR-2.5 — Compliance Scan of Existing Code

**KB required**: `cross-skill-integration.md` (loaded in Batch 2)

1. Load the compliance checklist from `cross-skill-integration.md` (section "Downstream: Mule Developer → Code Reviewer")
2. For each file in `IMPACT_ANALYSIS` that will be modified:
   - Scan for B-I.1 violations (error handling outside `error-handler.xml`)
   - Scan for B-I.2 violations (httpStatus 4xx/5xx outside error handlers)
   - Scan for B-I.3 violations (unreachable `isEmpty` checks)
   - Scan for B-I.4 violations (empty choice routes, missing `<otherwise>`)
   - Scan for B-II.2.1 violations (missing correlationId in HTTP requests)
   - Scan for B-III.1 violations (global configs in non-global.xml files)
   - Scan for B-IV.4 violations (inline CDATA where external DWL should be used)
3. Compile `EXISTING_VIOLATIONS` list with: file, rule ID, description, proposed fix
4. These will be presented as optional compliance fixes in CR-3

> **Performance note**: This is a lightweight scan using only the compliance checklist tables against files already read in CR-2. No additional file reads, no sub-agents, no interactive questions.
>
> **Purpose**: Prevents the developer from replicating non-compliant patterns when modifying existing files. Gives the user visibility into technical debt.

---

## Step CR-3 — Plan Targeted Changes

1. Determine minimum changes needed
2. Assess impact: shared DWL files, shared sub-flows, logging/error handling implications
3. If `EXISTING_VIOLATIONS` is non-empty, add a **"Compliance Fixes"** section to the plan:
   - List each violation with file, rule ID, and proposed fix
   - Mark these as "Optional — pre-existing compliance issues"
   - Present separately from functional changes — user can approve or decline independently
   - If user declines compliance fixes, note them in the final summary as "Known pre-existing violations"
4. Present full plan to user: functional changes (files to modify, what changes and why, new files, config changes) + compliance fixes (if any) + impact assessment
5. Wait for user approval. Store as `APPROVED_CR_PLAN`.

---

## Step CR-4 — Apply Code Changes

1. For each file: read current state → show before/after → wait for user confirmation → apply
2. New DWL files → correct `dwl/` subfolder
3. After all changes: verify XML references, DWL references, logging standards, error handling standards

---

## Steps CR-4.5, CR-5 and CR-6

| Step | Action |
|------|--------|
| **CR-4.5** | **Pre-flight validation** — Run full checklist from `workflow/pre-flight-validation.md` against every modified file. Auto-fix failures, re-run until OVERALL: PASS |
| CR-5 | Update pom.xml — bugfix = patch bump, CR = minor bump |
| CR-6 | Summary: CR description, files changed, new files, config changes, impact, version bump, pre-flight report. Delete checkpoint |

---
Last updated: 2026-04-13
Owner: Integration Team
