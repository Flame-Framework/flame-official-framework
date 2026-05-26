# Pre-flight Validation

## Overview

The **pre-flight validation** is a self-check the `munit-generator` agent runs **after** generating or fixing MUnit tests but **before** declaring the work done and handing off to the user (or to the `code-review` skill).

It exists because specific issues keep slipping past the agent and only surface when tests are run (often in CI/CD, where the feedback loop is slow):

1. ❌ Mock payloads missing `encoding="UTF-8"` (tests pass locally, fail in CI with `Cannot open a new cursor on a closed stream`)
2. ❌ External calls (`http:request`, `sap:*`, `db:*`, `os:*`, etc.) left unmocked
3. ❌ Mock `doc:id` values don't match the flow's actual `doc:id` (mock silently never binds)
4. ❌ Variables referenced in the flow's logger messages (e.g., `correlationId`) not set in `<munit:set-event>`
5. ❌ Fix mode: implementation file under `src/main/` edited to make a test pass

This checklist is **mandatory** for both Mode A (new tests) and Mode B (fix failing tests). The agent must run all checks, fix any failures, then re-run until all pass before declaring done.

---

## When to Run

- **Mode A:** after generating the full test suite for a project / endpoints, before running `mvn test` and before reporting back to the user.
- **Mode B:** after applying the minimal test-side fix and before re-running tests.
- **Before** invoking the `code-review` skill or telling the user the task is complete.

If a check fails, the agent must:

1. Identify the offending file + line (test XML, fixture JSON, mock definition, etc.)
2. Apply the fix (test-layer only — never `src/main/`)
3. Re-run the full checklist (fixes can introduce new issues)
4. Only declare done when **all checks pass**

---

## Validation Categories

### 1. Logger Mocking — absolute rule

| # | Check | How to Verify |
|---|---|---|
| J1 | NO `<munit-tools:mock-when processor="logger">` anywhere in any test file | `grep 'processor="logger"'` across `src/test/munit/**/*.xml` must return **zero** matches |
| J2 | NO `<munit-tools:verify-call processor="logger">` anywhere | Same grep against `verify-call` — zero matches |

The standard Mule `<logger>` is a no-op in tests — there is no reason to mock or verify it. If a logger references a variable, set that variable in `<munit:set-event>` instead.

### 2. Variable-Setup Checks

| # | Check | How to Verify |
|---|---|---|
| L1 | Every variable referenced by the flow under test (in `<logger message=…>`, in DataWeave expressions, in `<set-payload>`, etc.) is set in `<munit:set-event>` | Cross-check `vars.*` references in the flow against `<munit:variable key=…/>` |
| L2 | `correlationId` is set in `<munit:set-event>` whenever the flow's main loggers or outbound HTTP requests reference it | Grep flow for `vars.correlationId`; cross-check the test sets it |

### 3. Mock Coverage — every external call mocked

| # | Check | How to Verify |
|---|---|---|
| M1 | Every `<http:request>` in the flow under test has a corresponding `<munit-tools:mock-when processor="http:request" …>` in the test's `<munit:behavior>` | Parse flow XML, list every `<http:request>` with its `doc:id`; cross-check each is mocked |
| M2 | Every `<sap:sync-rfc>` / `<sap:*>` in the flow has a mock | Same cross-check for SAP |
| M3 | Every `<db:select>` / `<db:insert>` / `<db:*>` in the flow has a mock | Same for Database |
| M4 | Every `<os:store>` / `<os:retrieve>` / `<os:retrieve-all>` / `<os:contains>` / `<os:remove>` used by the flow has a mock | Same for ObjectStore (mock-when OR spy with verify) |
| M5 | Every `<flow-ref name="bearer-token">` (or any OAuth2 token sub-flow) is mocked in SAPI tests | Grep flow for bearer-token refs + cross-check test mocks |
| M6 | Any `<wsc:consume>`, `<jms:publish>`, `<vm:publish>`, `<email:*>`, `<sqs:*>`, `<s3:*>`, `<salesforce:*>` present in the flow has a mock | Same cross-check for all external connectors |
| M7 | No external call is left unmocked. If one is, the test will hit real external systems (or fail trying) | Final sweep |

### 4. Mock `doc:id` matching

| # | Check | How to Verify |
|---|---|---|
| D1 | Every `<munit-tools:with-attribute attributeName="doc:id" whereValue="…"/>` in the test references a `doc:id` that **actually exists** on a processor in the flow XML | Grep the exact `doc:id` string in the flow file — fail if zero matches |
| D2 | No `{placeholder}` or `{entity}` style literals remain in `whereValue=` | Grep for `whereValue="{` — must return zero |
| D3 | Every mocked processor type matches the actual processor type for that `doc:id` (e.g., `processor="http:request"` with a `doc:id` that belongs to an `<http:request>` — not a `<flow-ref>`) | Cross-check processor name vs the tag in the flow XML |

### 5. `encoding="UTF-8"` Checks (CI/CD stream fix)

| # | Check | How to Verify |
|---|---|---|
| E1 | Every `<munit-tools:payload …/>` in a mock return has `encoding="UTF-8"` | Grep `<munit-tools:payload` blocks — fail any without `encoding="UTF-8"` |
| E2 | Every `<munit-tools:variable …/>` inside `<munit-tools:variables>` in a mock return has `encoding="UTF-8"` | Same grep on mock variables |
| E3 | Same applies to mocks using `readUrl(...)` or `MunitTools::getResourceAsString(...)` — `encoding` attribute is mandatory | Confirm attribute is present |

### 6. Test Structure Checks

| # | Check | How to Verify |
|---|---|---|
| T1 | `<munit:set-event>` is the **first element inside `<munit:execution>`**, not `<munit:behavior>` | Parse every `<munit:test>` |
| T2 | Tests using `expectedErrorType="…"` **omit** the `<munit:validation>` section entirely — do not leave it empty with only comments | Scan tests with `expectedErrorType` — validation block must be absent |
| T3 | Every `<munit:test>` has both `name` and `description` attributes | Grep `<munit:test` elements |
| T4 | Test name follows `{method}-{entity}-test-{statusCode}-{scenario}` (or `{trigger}-{entity}-test-{scenario}` for non-HTTP flows) — see `standards/naming-conventions.md § 3` and `standards/test-scenarios.md § 1` | Match test name against pattern |
| T5 | Every assertion uses the `MunitTools::` prefix (e.g., `MunitTools::notNullValue()` — not `notNullValue()`) | Grep `is="#[` in assert-that for bare matcher calls |

### 7. Anti-Pattern Checks (compilation + logic)

| # | Check | How to Verify |
|---|---|---|
| A1 | No HTML entities (`&quot;`) inside DataWeave expressions in `value=` attributes — use plain quotes | Grep for `value='#\[.*&quot;` |
| A2 | XML attribute values use `&quot;` for double quotes, never `\"` | Grep for `\\"` in XML attributes |
| A3 | `<munit-tools:mock-when>` always has `<munit-tools:with-attributes>` (or is explicitly unscoped) | Fail any mock-when without attributes unless intentional |
| A4 | Mock response structure matches what the flow's transforms **read** (e.g., `payload.d.results` for OData means mock returns `{"d": {"results": [...]}}`) | Cross-check the DWL file's input expectations against fixture shape |

### 8. Fixture Checks (mock data files)

| # | Check | How to Verify |
|---|---|---|
| F1 | Every fixture JSON referenced by a mock exists at the cited path | `ls` each path from `getResourceAsString(...)` / `readUrl(...)` |
| F2 | Fixtures are under `src/test/resources/test-data/{entity}/` (mirroring test-suite structure) | File path inspection |
| F3 | For any `<foreach>` / `<parallel-foreach>` in the flow, a **single-item** fixture file exists alongside the array fixture | Check `*_single_*.json` pairs `*_response.json` |
| F4 | Fixture JSON is valid (parses) | Run `jq . file.json` on each fixture |
| F5 | Fixtures for external APIs were obtained from the correct source (EAPI/PAPI: Anypoint Exchange spec; SAPI: confirmed with the user) — NOT inferred from DWL | Agent self-attests: record the source for each fixture |

### 9. Target-Attribute + `ee:cache` Checks

| # | Check | How to Verify |
|---|---|---|
| TC1 | For every `<http:request target="X">` in the flow, its mock `<munit-tools:then-return>` **preserves the pre-request payload** AND sets `<munit-tools:variable key="X" …/>` | Inspect mock return content |
| TC2 | For every `<http:request>` wrapped in `<ee:cache>`, the corresponding `<munit-tools:verify-call>` uses `atLeast="0"` (never `times="1"`) | Grep verify-call blocks + cross-check against cache-wrapped requests |

### 10. Datetime / External Format Checks

| # | Check | How to Verify |
|---|---|---|
| DT1 | If the flow's DWL coerces a queryParam to `LocalDateTime { format: "yyyy-MM-dd'T'HH:mm:ss" }`, the test's `queryParams` values have **no** trailing `Z` | Cross-check DWL format strings against test queryParams |
| DT2 | OData backend error mocks use `odata.error` structure (not standard JSON `error`) | Grep OData backend 4xx/5xx fixtures |
| DT3 | OData backend "empty" responses follow the backend-specific empty-shape convention (e.g., one outer result with empty inner tables, not `{"d": {"results": []}}` directly) | Grep OData backend empty-result fixtures |

### 11. Layer-Specific Checks

Cross-reference [standards/template-by-layer.md](../standards/template-by-layer.md):

| Layer | Additional checks |
|---|---|
| **SAPI** | Bearer-token flow mocked if the API uses OAuth2. Mock response structure matches backend exactly (OData `d.results`, REST `value`, etc.). |
| **PAPI** | All SAPI calls mocked (typically 4–6 distinct HTTP requests). Message broker / queue operations mocked when present. `correlationId`, `scope`, `companyId` set in `<munit:set-event>`. For `foreach`/`parallel-foreach`: single-item fixtures present. Error-flow Object Store writes mocked. |
| **EAPI** | `correlationId` set in `<munit:set-event>`. PAPI call mocked. `x-correlation-id` in test request headers. UI-specific response fields asserted (`displayMessage`, `status`, etc.). |

### 12. Coverage Checks

| # | Check | How to Verify |
|---|---|---|
| C1 | Every endpoint (flow) covered by at least 3 test scenarios | Count `<munit:test>` per endpoint flow |
| C2 | Every Choice router branch has a test that triggers it | Map each `<when>`/`<otherwise>` to a test scenario |
| C3 | `pom.xml` has MUnit coverage plugin with `runCoverage>true` and `failBuild>true` | Inspect `pom.xml` |
| C4 | Coverage targets met (defaults: Application ≥ 80%, Flow ≥ 90%, Processor ≥ 80%; configurable via `config/framework.yaml` → `test.coverage.*-percent`) | Run `mvn test` (during F5/execution step) and read coverage report |

### 13. Mode B — Fix-Mode Checks (blocking when in fix mode)

| # | Check | How to Verify |
|---|---|---|
| FX1 | **No file under `src/main/`** was modified | `git diff --stat src/main/` must be empty; agent self-attests |
| FX2 | Only files under `src/test/munit/` or `src/test/resources/` were modified | `git diff --stat src/test/` shows the fix scope |
| FX3 | If diagnosis indicated the implementation was wrong, the agent STOPPED and reported — did not silently patch | Agent self-attests with evidence |
| FX4 | The original failing error message has been reproduced and classified (Common Errors table in `standards/common-errors.md`) before applying the fix | Agent records: error text → classified category → applied fix |

Failure of FX1 or FX3 is **blocking** and indicates a process violation — escalate to the user immediately.

---

## Output Format

After running the checklist, produce a structured report:

```
MUNIT PRE-FLIGHT VALIDATION REPORT
==================================
Project: my-suppliers-sapi
Layer: SAPI
Mode: A (new tests)
Files generated:
  - src/test/munit/suppliers/get-suppliers-test-suite.xml
  - src/test/resources/test-data/suppliers/backend-get-suppliers.json
  - src/test/resources/test-data/suppliers/backend-get-suppliers-empty.json

[1]  Logger Mocking Rule                 PASS (2/2)
[2]  Variable-Setup Checks               PASS (2/2)
[3]  Mock Coverage                        FAIL (6/7)
     └─ M1 FAIL: <http:request doc:id="e5f6a7b8-c9d0-1234-efgh-567890123456"> in
                 implementation.xml:42 is NOT mocked in any test
[4]  doc:id Matching                      PASS (3/3)
[5]  encoding="UTF-8"                     PASS (3/3)
[6]  Test Structure                       PASS (5/5)
[7]  Anti-Patterns                        PASS (5/5)
[8]  Fixtures                             PASS (5/5)
[9]  Target-Attribute + ee:cache          PASS (2/2)
[10] Datetime / External Formats          PASS (3/3)
[11] Layer-Specific (SAPI)                PASS
[12] Coverage (not yet run)               PENDING (run after mvn test)
[13] Fix-Mode Checks                      N/A (Mode A)

OVERALL: FAIL (1 issue)

REMEDIATION:
- Fix M1: add a <munit-tools:mock-when processor="http:request"> targeting
          doc:id="e5f6a7b8-c9d0-1234-efgh-567890123456" in every test
          (this is the bearer-token validation call and must be mocked
          to avoid real backend contact)
```

After remediation, re-run the full checklist. Only when **OVERALL: PASS** (excluding the coverage check, which runs after `mvn test`) can the agent ask the user for permission to run tests and then report results.

---

## Integration with Other Skills

### code-review

If the user invokes `code-review` on the generated / fixed tests, it assumes pre-flight validation has passed. Explicitly mention: *"MUnit pre-flight validation passed (13 categories, N checks)"* before handing off.

### mule-developer

If pre-flight validation fails on Category 3 (Mock Coverage) because a flow component lacks a `doc:id`, the fix is **NOT** to edit `src/main/`. Instead, escalate to the `mule-developer` skill: the flow needs a `doc:id` on that processor, and it is their skill's responsibility to add it.

---

## What This Document Doesn't Cover

The pre-flight validation is for **structural / mechanical** test-quality checks. It does NOT replace:

- **Functional correctness** — do the assertions actually validate the business logic, or only presence/shape?
- **Mock data realism** — is the mock response *representative* of real backend data (field values, edge cases), or just a syntactically valid blob?
- **Test naming semantics** — does the test name describe what's actually being tested, not just the HTTP status?
- **Coverage quality** — coverage metrics measure executed processors, not assertion quality (see [workflow.md § Coverage targets](./workflow.md))

The pre-flight is the cheapest, fastest filter — it should never miss a known-recurring issue.

---

## References

- [MUnit Rules](./workflow.md) — do's/don'ts, HTTP scenarios, common errors table
- [MUnit Templates by Layer](../standards/template-by-layer.md) — SAPI/PAPI/EAPI checklists + anti-patterns
- [Allowed Components](../reference/allowed-components-list.md) — approved components, mock priorities
- [Workflow](./workflow.md) — Mode A and Mode B step procedures
- [Cross-Skill Integration](./cross-skill-integration.md) — handoffs with mule-developer, code-review, RAML/solution designers
