# Pre-flight Validation

## Overview

The **pre-flight validation** is a self-check the `mule-developer` agent runs **after** generating code but **before** declaring the work done and handing off to the user (or to the `code-reviewer` skill).

It exists because specific issues keep slipping past the agent and getting flagged in code review:

1. ❌ Outbound HTTP requests not propagating `x-correlation-id`
2. ❌ `<error-handler>` references in flows missing the `description` (or the `<on-error-*>` blocks missing `doc:name`)

This checklist is **mandatory**. The agent must run all checks, fix any failures, then re-run until all pass before declaring done.

---

## When to Run

- After generating any new flow, sub-flow, error handler, or DWL file
- After modifying an existing flow (CR/bugfix path)
- **Before** invoking the `code-reviewer` skill or telling the user the task is complete

If a check fails, the agent must:
1. Identify the offending file + line
2. Apply the fix
3. Re-run the full checklist (fixes can introduce new issues)
4. Only declare done when **all checks pass**

---

## Self-Validation Procedure

The skill invokes this procedure at Step ND-15.5 (new development) or CR-4.5 (change request) — the mandatory gate before handoff.

1. **Run the full pre-flight checklist** from this file — all 8 categories. This is the single source of truth for validation — no parallel checklists exist.
2. **Auto-fix** failures where possible (missing `doc:name` on `<on-error-*>`, inline CDATA that should be external DWL, missing `doc:id`, hardcoded URLs, etc.).
3. **Re-run the checklist** after fixes — fixes can introduce new issues.
4. **Cross-check** with the broader compliance rules in `cross-skill-integration.md` § "Cross-Skill Rules NOT Covered by Pre-flight" for code-review-side rules not covered by pre-flight.
5. **Flag to user** items that need input (secure properties, shared DWL compatibility, hardcoded value decisions).
6. **Report results** in the final summary using the Output Format below (category-by-category PASS/FAIL → REMEDIATION list).
7. **Only declare done when OVERALL: PASS** — never hand off failing code to the user or the code-reviewer skill.

---

## Validation Categories

### 1. Logger Checks

| # | Check | How to Verify |
|---|---|---|
| L1 | Every endpoint flow has an INFO logger near the top (start of the flow) | First `<logger>` in flow body has `level="INFO"` |
| L2 | Every endpoint flow has an INFO logger near the bottom (end of the flow, before any `<error-handler>`) | Last `<logger>` in flow body has `level="INFO"` |
| L3 | Every logger has both `level` and `doc:name` attributes | Scan all `<logger>` elements |
| L4 | All loggers use the standard Mule `<logger>` — never any custom logger module | grep for `logger>` namespace-prefixed (e.g., `xxx-logger:`) — must return zero matches |
| L5 | Logger `message` attributes are plain text (inline `#[…]` is fine; no multi-line DataWeave) | grep for `message="#[` followed by `output application/` — must return zero matches |
| L6 | Every `<on-error-*>` scope contains at least one `<logger level="ERROR">` | Scan all `<on-error-*>` children |
| L7 | `vars.correlationId` is set in the main flow from the inbound `x-correlation-id` header | grep main flow for `variableName="correlationId"` |

### 2. Error Handler Checks (covers known fix #3)

| # | Check | How to Verify |
|---|---|---|
| E1 | Every endpoint flow has either an inline `<error-handler>` or a `<error-handler ref="..."/>` reference | Scan `<flow>` children for `<error-handler>` element |
| E2 | Every `<on-error-propagate>` and `<on-error-continue>` has a `doc:name` attribute | grep XML for `<on-error-` and ensure `doc:name=` follows |
| E3 | Every `<error-handler name="...">` definition has a `doc:name` | grep error handler definitions |
| E4 | Every `<raise-error>` has both `type` and `description` attributes | grep `<raise-error` and check attributes |
| E5 | All `type=` values in `<raise-error>` and `<on-error-propagate>` are listed in [error-types.md](../reference/error-types.md) | Cross-check against catalog |
| E6 | Error response payloads come from `dwl/error/message/*.dwl` files, never inline DWL | Scan `<on-error-propagate>` for `<ee:set-payload>` — must use `resource="..."`, not `<![CDATA[...]]>` |
| E7 | Every error handler sets `vars.httpStatus` explicitly | Scan `<on-error-propagate>` for `<ee:set-variable variableName="httpStatus">` |
| E8 | Reprocess handlers (`type=HTTP:CONNECTIVITY,HTTP:TIMEOUT,...`) only appear in PAPI projects | If layer = SAPI or EAPI, fail if reprocess handlers exist |
| E9 | Error response payload schema is `{error: {code, message}}` | Inspect every error-handler `<ee:set-payload>` (or its referenced DWL file) — payload must be a single `error` object with `code` (string) + `message` (string), NOT an array |

### 3. DWL File Checks

| # | Check | How to Verify |
|---|---|---|
| D1 | DWL files producing payloads are under `src/main/resources/dwl/payloads/...` | File path inspection |
| D2 | DWL files producing variable values are under `src/main/resources/dwl/vars/...` | File path inspection |
| D3 | Error message DWL files are under `dwl/error/message/...` | File path inspection |
| D4 | Status code mapping DWL files are under `dwl/error/statusCodes/...` | File path inspection |
| D5 | Reprocess DWL files are under `dwl/error/reprocess/...` | File path inspection |
| D6 | No DWL files at the root of `dwl/` | `ls src/main/resources/dwl/*.dwl` returns nothing |
| D7 | Multi-line DWL is not inlined in `<![CDATA[...]]>` — must reference a `.dwl` file | Scan `<ee:set-payload>` and `<ee:set-variable>` for multi-line CDATA blocks |
| D8 | DWL filenames are camelCase | File listing inspection |

### 4. Naming Checks

Cross-reference [naming-conventions.md](../standards/naming-conventions.md):

| # | Check |
|---|---|
| N1 | Sub-flow names end with `-sub-flow` or `-subflow` and are kebab-case |
| N2 | Variable names are camelCase and use reserved standard names where applicable (`correlationId`, `httpStatus`) |
| N3 | Connector configs use `{System}_{Type}_Config` PascalCase + underscores |
| N4 | APIKit config uses `{api-name}-config` kebab-case |
| N5 | Every processor has a meaningful `doc:name` (not just the type name like `"Logger"` or `"Transform"`) |
| N6 | Property keys use `dot.notation`, lowercase |
| N7 | Every processor has a `doc:id` attribute (auto-generated UUID by Studio — do not author manually, do not remove) |

### 5. Property Checks

| # | Check | How to Verify |
|---|---|---|
| P1 | No hardcoded URLs in `<http:request>` `path=` for backend hosts | grep for `https?://` in XML files |
| P2 | No hardcoded timeouts (e.g., `responseTimeout="30000"`) — must reference property | grep for numeric `Timeout=` attributes |
| P3 | No hardcoded credentials anywhere | grep for `password=`, `apiKey=`, `secret=` with non-`${...}` values |
| P4 | All property keys referenced in XML exist in `config-{env}.yaml` | Cross-check `${...}` references against YAML files |
| P5 | All property keys exist in **every** env file (dev/qa/prod) | Diff key sets across env files |
| P6 | Secret references use `${secure::...}` prefix | grep for known secret keys without `secure::` |
| P7 | No `default` values in property references (except optional feature flags) | grep `${[a-z.]+:[^}]+}` |

### 6. Reuse Check

| # | Check | How to Verify |
|---|---|---|
| R1 | Before creating a new sub-flow, scanned existing project for similar sub-flows | Agent self-attests; lists candidates considered |
| R2 | Before creating a new connector config, scanned existing `global.xml` for one already covering this backend | Cross-check connector configs |
| R3 | Before creating a new DWL file, scanned `dwl/` for one producing the same shape | Filename + output type cross-check |
| R4 | If a candidate was found and rejected, the reason is documented in commit message or PR description | Agent self-attests |

### 7. Layer-Specific Checks

| Layer | Additional Checks |
|---|---|
| **SAPI** | No reprocess handler. No EAPI-style consumer response shaping. |
| **PAPI** | If operation is a backend write (POST/PUT/PATCH), reprocess handler must be referenced. `originalPayload` saved before first backend call when orchestrating 2+ backends. |
| **EAPI** | Strict input validation present (raises `APP:VALIDATION` on failure). Response shaped for consumer (no raw PAPI passthrough). |

### 8. Dependency & Cleanup Checks

| # | Check | How to Verify |
|---|---|---|
| X1 | New connector dependencies in `pom.xml` use approved versions from `reference/connectors.md` | Cross-check `<dependency>` blocks against the connectors reference — fail on version drift |
| X2 | No unused DWL files, sub-flows, variables, or imports introduced | Every `.dwl` file must be referenced by at least one XML transform; every sub-flow must be `flow-ref`'d; every `vars.X` set must be read somewhere; every `import` must be used |

---

## Output Format

After running the checklist, produce a structured report:

```
PRE-FLIGHT VALIDATION REPORT
============================
Project: example-orders-papi
Layer: PAPI
Files modified: src/main/mule/implementation/orders.xml,
                src/main/resources/dwl/payloads/orders/backendRequest.dwl

[1] Logger Checks                       PASS (7/7)
[2] Error Handler Checks                FAIL (8/9)
    └─ E2 FAIL: <on-error-propagate type="APIKIT:BAD_REQUEST"> in
                src/main/mule/common/error-handler.xml:14 missing doc:name
[3] DWL File Checks                     PASS (8/8)
[4] Naming Checks                       PASS (7/7)
[5] Property Checks                     PASS (7/7)
[6] Reuse Check                         PASS (4/4)
[7] Layer-Specific Checks (PAPI)        PASS (3/3)
[8] Dependency & Cleanup Checks         PASS (2/2)

OVERALL: FAIL (1 issue)

REMEDIATION:
- Fix E2: add doc:name="On APIKit Bad Request" to src/main/mule/common/error-handler.xml:14
```

After remediation, re-run the full checklist. Only when **OVERALL: PASS** can the agent declare done.

---

## Integration with code-reviewer Skill

The `code-reviewer` skill assumes pre-flight validation has already passed. If pre-flight is skipped:

- Code review will catch the same issues — but later, costing a round-trip
- The user has explicitly signaled (via `project-context.md` or feedback) that these issues are recurring; failing to pre-validate is a process failure

The agent should explicitly mention: *"Pre-flight validation passed (X categories, Y checks)"* before invoking `code-reviewer`.

---

## What This Document Doesn't Cover

The pre-flight validation is for **structural / mechanical** checks. It does NOT replace:

- **Functional correctness** — does the flow actually do what the requirement says?
- **Code review** — handled by the `code-reviewer` skill
- **Test coverage** — handled downstream by the `munit-generator` skill (out of scope here)
- **Definition of Done** — see [definition-of-done.md](./definition-of-done.md) (broader gate)

The pre-flight is the cheapest, fastest filter — it should never miss a known-recurring issue.

---

## References

- [Logging Standards](../standards/logging.md) — standard Mule logger usage
- [Error Handling Standards](../standards/error-handling.md)
- [Error Types Reference](../reference/error-types.md)
- [Error Handling Usage](../usage/error-handling.md)
- [Naming Conventions](../standards/naming-conventions.md)
- [Configuration Properties](../standards/configuration-properties.md)
- [Reference Flows](../usage/reference-flows.md)

---
Last updated: 2026-04-13
Owner: Integration Team
