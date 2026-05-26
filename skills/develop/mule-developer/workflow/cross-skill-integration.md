# Cross-Skill Integration Guide

How the mule-developer skill integrates with other skills in the pipeline. Follow these conventions to ensure smooth handoffs.

## Skill Pipeline

```
RAML Designer → Mule Developer → Code Reviewer → MUnit Generator
                       ↑
       Solution Designer
```

The Mule Developer skill sits in the middle. Upstream skills (RAML Designer, Solution Designer) feed it specs; downstream skills (Code Reviewer, MUnit Generator) consume what it produces. **Implementation of those downstream skills is out of scope here** — this guide only covers what the dev agent must produce so the handoff works.

## Upstream: RAML Designer → Mule Developer

The RAML specification drives the flow structure. When the dev agent receives a RAML:

| RAML Element | What dev agent generates |
|--------------|------------------------|
| Resource path `/resources` | APIkit-routed flow `get:\resources:config-name` |
| HTTP method `GET`, `POST`, etc. | Corresponding flow name prefix |
| URI parameter `/{resourceId}` | Variable extraction in sub-flow: `attributes.uriParams.resourceId` |
| Query parameter `?status=` | DWL in `dwl/vars/` to extract query params |
| Request body type | DWL transform in `dwl/payloads/` matching the type structure |
| Response type | DWL response transform matching the type structure |
| Traits (e.g., `client-credentials`) | Auth configuration in global.xml |

**Convention**: The RAML `displayName` for each method should map to the `doc:name` of the implementation sub-flow.

## Downstream: Mule Developer → Code Reviewer

The code reviewer validates what the dev agent produces. Use the compliance checklist below during self-validation (ND-15.5 / CR-4.5) and during compliance scans (ND-4.5 / CR-2.5) to ensure code passes review.

> **Sync Notice**: This checklist maps to code-review sections v2.0.0 (2026-04-07). If review rules change, update corresponding items here.

### Compliance Checklist — Error Handling & Flow Logic (B-I)

| Rule | Severity | Verification | Dev KB Reference |
|------|----------|-------------|-----------------|
| B-I.1 | Blocking | ALL error handling (on-error-continue, on-error-propagate) must be ONLY in `error-handler.xml`. Try-scope handlers in flow files are the only exception | `error-handling.md` |
| B-I.2 | Blocking | NEVER set httpStatus to 4xx/5xx outside an error handler. Use `raise-error` in main flow to trigger the error handler | `error-handling.md` |
| B-I.3 | Blocking | Every condition in Choice routers must be logically reachable. Do NOT use `isEmpty(payload)` after a transform that always returns a non-empty object — read the actual DWL first | `api-development-best-practices.md` |
| B-I.4 | Attention | ALL Choice router branches must have at least one processor. ALWAYS include an `<otherwise>` branch | `api-development-best-practices.md` |

### Compliance Checklist — CorrelationId (B-II)

| Rule | Severity | Verification | Dev KB Reference |
|------|----------|-------------|-----------------|
| B-II.2.1 | Blocking | correlationId MUST be sent in ALL HTTP requests and connectors that support it | `logging.md` |
| B-II.2.2 | Attention | Use `vars.correlationId` format (preferred over plain `correlationId`) | `logging.md` |
| B-II.2.3 | Attention | Sending `correlationId` without `vars.` prefix is acceptable but sub-optimal — auto-fixed as part of B-II.2.2 check | `logging.md` |

### Compliance Checklist — Configuration Management (B-III)

| Rule | Severity | Verification | Dev KB Reference |
|------|----------|-------------|-----------------|
| B-III.1 | Blocking | ALL global config elements (http:listener-config, http:request-config, db:config, os:object-store, vm:config, etc.) must be ONLY in `global.xml` | `project-structure.md` |
| B-III.2 | Attention | config-local/dev/prod YAML files must have identical key structures (values may differ) | `configuration-properties.md` |
| B-III.3 | Blocking | No hardcoded URLs, hosts, ports, or config-like values in flow XML — use property references | `configuration-properties.md` |

### Compliance Checklist — DataWeave Quality (B-IV)

| Rule | Severity | Verification | Dev KB Reference |
|------|----------|-------------|-----------------|
| B-IV.1 | Blocking | Do NOT modify DWL files used by multiple Transform Messages without verifying all usages remain compatible | `dataweave-patterns.md` |
| B-IV.4 | Attention | Use external `.dwl` files (`resource="dwl/..."`) instead of inline CDATA DataWeave. Exception: simple single-value assignments | `dataweave-patterns.md` |

### Compliance Checklist — Security (B-V)

| Rule | Severity | Verification | Dev KB Reference |
|------|----------|-------------|-----------------|
| B-V.1 | Blocking | ALL sensitive data (passwords, API keys, tokens, secrets) must be in secure-config files with `![encrypted]` format | `configuration-properties.md` |

### Compliance Checklist — Project Structure & Testability (A)

| Rule | Severity | Verification | Dev KB Reference |
|------|----------|-------------|-----------------|
| A.8 | Blocking | Every `.dwl` file must be referenced by at least one XML transform or another DWL file — no unused DWL files | `project-structure.md` |
| A.9 | Blocking | Every external call (`http:request`, `db:*`) must be mockable — i.e., have a unique `doc:id` so the downstream MUnit Generator can target it | This file (Downstream: MUnit Generator) |

### Cross-Skill Rules NOT Covered by Pre-flight

Pre-flight validation (see `pre-flight-validation.md`) is the primary self-check. The rules below are the residual code-reviewer rules that pre-flight does **not** cover — the dev agent must scan for them separately during self-validation (Step ND-15.5 / CR-4.5). Rules that overlap with pre-flight have been removed; pre-flight is authoritative for those.

| Rule | What to scan for | Auto-fixable? |
|------|-----------------|---------------|
| B-I.1 | Any error handling outside `error-handler.xml` (except try-scope) | Yes — move to error-handler.xml |
| B-I.2 | Any httpStatus 4xx/5xx set-variable outside error handlers | Yes — replace with raise-error |
| B-I.3 | `isEmpty()` on always-non-empty transform outputs | No — flag to user |
| B-I.4 | Choice routers with empty branches or missing `<otherwise>` | Yes — add otherwise branch |
| B-II.2.1 | HTTP requests missing correlationId header | Yes — add correlationId header |
| B-II.2.2 | correlationId sent as plain `correlationId` instead of `vars.correlationId` | Yes — update format |
| B-III.1 | Global config elements in files other than `global.xml` | Yes — move to global.xml |
| B-IV.1 | Modified DWL files referenced by multiple transforms | No — flag to user |

> Removed because already in pre-flight: B-III.2 (P5), B-III.3 (P1), B-IV.4 (D7), B-V.1 (P6), A.8 (X2).

**If any check fails**: fix silently (for auto-fixable items) or flag to the user (for items needing input). Include a "Self-validation results" section in the final summary showing all checks, results, and any fixes applied.

### Compliance Checklist — Code Quality (self-validation only)

These are checked during self-validation but do not have a specific code-review rule ID:

| Check | Verification | Dev KB Reference |
|-------|-------------|-----------------|
| Error response format | Must use `{error: {code, message}}` structure | `error-handling.md` |
| doc:id | Every component MUST have a unique `doc:id` (UUID format) | `naming-conventions.md` |
| doc:name | All `doc:name` attributes must be descriptive (not "Transform Message") | `naming-conventions.md` |
| Connector versions | New connectors must use versions from `reference/connectors.md` | `reference/connectors.md` |
| DWL subfolder | DWL files in correct subfolder: `dwl/payloads/`, `dwl/vars/`, `dwl/error/` | `dataweave-patterns.md` |
| No unused code | No unused imports, variables, or sub-flows introduced | N/A — scan project |

### Existing Code Compliance Scan

When working on a project that already has existing code (**both PATH A and PATH B**), scan existing code against the compliance checklist above. This is a **lightweight check** — it uses only the tables above against files already in context. No sub-agents, no interactive questions, no additional file reads.

**PATH A — New Development (Step ND-4.5, after reuse scan)**:
1. For each component in `REUSE_PLAN` marked as "Reusable as-is" or "Reusable with modification":
   - Scan the component's code against the blocking rules in the compliance checklist
   - If violations found: reclassify "Reusable as-is" → "Reusable with modification" (modification = compliance fix)
2. Store violations as `REUSE_VIOLATIONS` with: file, rule ID, description, proposed fix
3. Include `REUSE_VIOLATIONS` in the ND-5 implementation plan under "Compliance Fixes on Reused Components"

**PATH B — CR/Bugfix (Step CR-2.5, after code analysis)**:
1. For each file in `IMPACT_ANALYSIS` that will be modified:
   - Scan the file's code against the blocking rules in the compliance checklist
   - Flag pre-existing violations that should not be replicated or preserved
2. Store violations as `EXISTING_VIOLATIONS` with: file, rule ID, description, proposed fix
3. Include `EXISTING_VIOLATIONS` in the CR-3 change plan as a separate "Compliance Fixes" section
4. Present compliance fixes separately from functional changes — user can approve or decline independently
5. If user declines compliance fixes, note them in the summary as "Known pre-existing violations"

## Downstream: Mule Developer → MUnit Generator

> **Scope**: This section describes only what the **dev agent must produce** so the downstream MUnit Generator can do its job. Writing the actual MUnit tests (suites, assertions, mocks) is the MUnit Generator skill's responsibility — not this skill's.

### What the dev agent must produce

| Requirement | Why it matters for MUnit |
|---|---|
| Every component has a unique `doc:id` (UUID) | MUnit references components by `doc:id` to mock them |
| Every external call (`http:request`, `db:*`, etc.) has a `doc:id` | Required mock target |
| Sub-flow names follow `naming-conventions.md` | MUnit infers test-suite filenames from flow names |
| External connectors are isolated (not buried inside large flows) | Each external dependency must be individually mockable |

### What MUnit will mock vs. test for real

| Component the dev agent uses | MUnit treatment |
|---|---|
| `http:request` | Mocked |
| `mqtt:*`, `db:*`, other external connectors | Mocked |
| `ee:transform` | Tested for real (no mock) |
| `logger` (standard Mule) | Not mocked (no-op in tests) |
| `flow-ref` (internal sub-flow) | Tested for real, may be mocked when isolating a single sub-flow |
| `validation:*` | Tested for real |

The dev agent does NOT write the mocks — it just produces code structured so that mocks are possible.

## Upstream: Solution Designer → Mule Developer

When working from a solution design document:

| Solution Design Section | How dev agent uses it |
|------------------------|-----------------------|
| API Design tables | Drives endpoint creation and sub-flow structure |
| Data mappings | Drives DWL transform logic |
| Sequence diagram | Drives sub-flow execution order and async patterns |
| External system requests | Drives connector configuration |
| Test scenarios | Informs what the code should handle (happy path + error cases) |
| NFRs (performance, SLA) | Informs timeout and retry configuration |

## Checklist Before Handoff

Before finishing development (Step ND-16 or CR-6), verify:

- [ ] All `doc:id` attributes are present and unique
- [ ] All `doc:name` attributes are descriptive
- [ ] Error format follows `{error: {code, message}}`
- [ ] Logging covers START, END, BEFORE/AFTER_REQUEST, BEFORE/AFTER_TRANSFORM
- [ ] Config properties exist in all 4 env files with same structure
- [ ] DWL files are in correct subfolders
- [ ] No hardcoded values in flow XML
- [ ] Secure properties use `![...]` format
- [ ] New dependencies use versions from reference/connectors.md
