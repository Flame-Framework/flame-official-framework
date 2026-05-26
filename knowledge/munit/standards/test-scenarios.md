# MUnit Test Scenarios (Standards)

> **Purpose:** Single source of truth for required test scenarios, test-case name patterns, and scenario counts across MuleSoft projects. Consult this before generating any test suite.

> Every endpoint must have **at least 3 test scenarios**. A Solution Design document, when available, overrides these defaults — follow the Solution Design's "Use Case Scenarios for Testing" section verbatim.

---

## 1. Standard Scenarios per HTTP Method

Combines status-code expectations with canonical test-case names.

### GET endpoints (minimum 3)

| Test-case name | HTTP Status | Scenario |
|---|---|---|
| `get-{entity}-test-200-ok` | `200 OK` | Happy path — data returned |
| `get-{entity}-test-204-no-content` | `204 No Content` | Backend returns empty / no data found |
| `get-{entity}-test-500-internal-server-error` | `500 Internal Server Error` | Backend system failure |

> **Do not** add a `get-{entity}-test-400-bad-request` scenario unless the RAML validation explicitly declares path/query constraints that can fail — otherwise APIkit handles `400` before the flow runs and the test is untestable.

### POST endpoints (minimum 3)

| Test-case name | HTTP Status | Scenario |
|---|---|---|
| `post-{entity}-test-201-created` | `201 Created` | Resource successfully created |
| `post-{entity}-test-400-bad-request` | `400 Bad Request` | Invalid payload / business-rule validation failure |
| `post-{entity}-test-500-internal-server-error` | `500 Internal Server Error` | Backend system failure |

### PUT / PATCH endpoints (minimum 3)

| Test-case name | HTTP Status | Scenario |
|---|---|---|
| `patch-{entity}-test-204-no-content` | `204 No Content` | Resource successfully updated |
| `patch-{entity}-test-404-not-found` | `404 Not Found` | Resource to update does not exist |
| `patch-{entity}-test-500-internal-server-error` | `500 Internal Server Error` | Backend system failure |

> For PUT endpoints use `put-` prefix with the same scenarios. If the API contract returns `200 OK` with the updated resource, swap `204-no-content` for `200-ok`.

### DELETE endpoints (minimum 3)

| Test-case name | HTTP Status | Scenario |
|---|---|---|
| `delete-{entity}-test-200-ok` | `200 OK` or `204 No Content` | Resource successfully deleted |
| `delete-{entity}-test-404-not-found` | `404 Not Found` | Resource to delete does not exist |
| `delete-{entity}-test-500-internal-server-error` | `500 Internal Server Error` | Backend system failure |

---

## 2. Test-Case Naming Convention (all layers)

All layers — SAPI, PAPI, EAPI — use the same pattern: `{method}-{entity}-test-{statusCode}-{scenario}`, with `{scenario}` as the HTTP reason phrase in kebab-case (e.g., `200-ok`, `204-no-content`, `404-not-found`, `500-internal-server-error`). The §1 tables above apply unchanged to every layer. See [naming-conventions.md § 3 Test-Case Names](./naming-conventions.md).

---

## 3. Scenario Counts by Layer

| Aspect | SAPI | PAPI | EAPI |
|---|---|---|---|
| **Mock count** | 1–2 (backend + optional bearer token) | 6–10 (multiple SAPI calls + broker/queue) | 1–3 (PAPI + transform) |
| **Minimum scenarios per endpoint** | 3 | 3 + one per orchestration branch | 3 + one per UI path |
| **Primary focus** | Backend integration, OData/REST shape | Orchestration, aggregation, error persistence | UI transformation, async operations, error envelopes |
| **Complexity** | Low | High | Medium |

---

## 4. Additional Scenarios (Triggered by Flow Structure)

When the flow contains any of these elements, add extra scenarios beyond the minimum:

### Choice router (`<choice>`)

One test per `<when>` condition + one test for `<otherwise>`. Payload in `<munit:set-event>` drives which branch fires. Verify with `verify-call times="1"` on the target sub-flow / processor of the branch; `times="0"` on the others.

### Scatter-Gather (`<scatter-gather>`)

One "all routes succeed" test + one "one route fails" test per route. Mock each route independently via `doc:name`. Assert the aggregated payload shape.

### First-Successful router

One "primary succeeds" test (mock primary OK; verify fallback `times="0"`) + one "primary fails, fallback succeeds" test.

### `foreach` / `parallel-foreach`

- Array fixture for the upstream call that returns the collection.
- **Single-item fixture** for mocks inside the iteration that need to preserve the current item.
- Additional scenario: one iteration, multiple iterations, empty collection. Verify call counts multiply correctly.

### `ee:cache`-wrapped HTTP request

All `verify-call` on the wrapped request uses `atLeast="0"` instead of `times="1"` (cache may short-circuit the call).

### HTTP request with `target` attribute

The mock's `then-return` must **preserve the upstream payload** AND set the target variable via `<munit-tools:variables>`. Otherwise downstream transforms see an overwritten payload.

### Async scope (`<async>`)

Either: mock the processor inside the async scope with `verify-call timeout="5000"` to confirm it eventually ran; OR extract the async logic into a sub-flow and test that sub-flow directly.

### VM listener flow

Call the listener flow directly via `<flow-ref>` providing a `<munit:set-event>` payload. Mock `vm:publish` and `vm:consume`.

### Batch job

Test the **input phase** (assert payload is non-null) and/or extract batch-step logic into a sub-flow to test a single record directly.

### Error propagation through `<flow-ref>`

For a flow that calls a sub-flow, one scenario must mock the sub-flow to throw (via `<munit-tools:error typeId="..."/>`) and assert the caller's error handler fires.

### Try-scope error handling

One scenario must trigger each `<on-error-*>` handler inside the `<try>`. Use `expectedErrorType` on the test and omit the `<munit:validation>` section.

### Error type reference (for `expectedErrorType` and `<munit-tools:error typeId=...>`)

When writing negative-path scenarios or mocking errors, use a real error type from MuleSoft's catalog — not an invented one.

#### APIkit

| Error type | HTTP status | Meaning |
|---|---|---|
| `APIKIT:BAD_REQUEST` | 400 | Invalid request format |
| `APIKIT:NOT_FOUND` | 404 | Resource not found |
| `APIKIT:METHOD_NOT_ALLOWED` | 405 | HTTP method not allowed |
| `APIKIT:UNSUPPORTED_MEDIA_TYPE` | 415 | Content-Type not supported |

#### HTTP

| Error type | HTTP status (when mapped) |
|---|---|
| `HTTP:CONNECTIVITY` | 503 |
| `HTTP:TIMEOUT` | 504 |
| `HTTP:BAD_REQUEST` | 400 |
| `HTTP:UNAUTHORIZED` | 401 |
| `HTTP:FORBIDDEN` | 403 |
| `HTTP:NOT_FOUND` | 404 |
| `HTTP:INTERNAL_SERVER_ERROR` | 500 |
| `HTTP:SERVICE_UNAVAILABLE` | 503 |

#### Database

- `DB:CONNECTIVITY`, `DB:QUERY_EXECUTION`, `DB:BAD_SQL_SYNTAX`

#### Salesforce

- `SALESFORCE:CONNECTIVITY`, `SALESFORCE:INVALID_SESSION`

#### File / SFTP

- `FILE:CONNECTIVITY`, `FILE:FILE_NOT_FOUND`, `FILE:ACCESS_DENIED`

#### Validation

- `VALIDATION:INVALID_BOOLEAN`, `VALIDATION:INVALID_NUMBER`, `VALIDATION:BLANK_STRING`, `VALIDATION:NULL`

#### SAP

- `SAP:CONNECTIVITY`, `SAP:EXECUTION`

#### Mule Core

- `MULE:EXPRESSION`, `MULE:ROUTING`, `MULE:CONNECTIVITY`, `MULE:TIMEOUT`

#### Custom (defined in project `error-types`)

- `APP:CUSTOM_ERROR`, `ORDER:VALIDATION_ERROR`, etc. — only use types declared in the project's `error-types` catalog.

### Error-handling test strategies

How to write scenarios that exercise error paths. For XML examples of each strategy, see [usage/official-guidelines.md § Testing Error Handling](../usage/official-guidelines.md).

| Goal | Strategy |
|---|---|
| Confirm a downstream failure propagates | Mock the processor to return `<munit-tools:error typeId="..."/>`; attach `expectedErrorType` to the `<munit:test>`; omit the `<munit:validation>` section |
| Confirm error-handler logic runs | Trigger the error through a mock; in `<munit:validation>`, `verify-call` the handler components (logger, transform, etc.) |
| Confirm error propagation via `flow-ref` | Mock `mule:flow-ref` to return a typed error; assert the caller's error handler fires |
| Confirm global error handler fires | Trigger the error; verify logger calls + resulting error-response payload |

---

## 5. Flow Type → Test Approach

Use the flow name to pick the initial scenario approach before drilling into §4 flow-element extras. External-system fixture shapes (OData backends, empty-result envelopes, DateTime `Z` stripping) live in [template-by-layer.md § SAPI — External-system response shapes](./template-by-layer.md).

| Flow name pattern | Type | Primary scenarios |
|---|---|---|
| `get:\*` or `get-*` | GET endpoint | Query params, 200 / 204 / 500 (404 if `{id}` path) |
| `post:\*` or `post-*` | POST endpoint | Request body, 201 / 400 / 500 |
| `put:\*` / `patch:\*` (or `put-*` / `patch-*`) | Update endpoint | URI params, 204-no-content (or 200-ok) / 404 / 500 |
| `delete:\*` or `delete-*` | DELETE endpoint | URI params, 200 or 204 / 404 / 500 |
| `*-scheduler-*` | Scheduler flow | No HTTP `<munit:attributes>` needed; set `scope` variable; verify iteration counts |
| `*-subflow` / `*-sub-flow` | Sub-flow | Called via `flow-ref`; stage required `vars.*` the sub-flow reads |

---

## 6. Required Scenario Sources

The agent decides which scenarios to generate using this priority:

1. **Solution Design document** — if available, its "Use Case Scenarios for Testing" section is authoritative. Generate exactly these scenarios, no less.
2. **RAML specification** — each declared response status code becomes a candidate scenario.
3. **Flow structure** — each `<choice>`, `<foreach>`, `<ee:cache>`, `target` attribute, `<try>` triggers additional scenarios per §4.
4. **This document** — defaults above if 1–3 are silent.

Never skip scenarios declared in the Solution Design. If the agent cannot generate a scenario (e.g., missing fixture data), STOP and escalate to the user.

---

## References

- [workflow.md](../workflow/workflow.md) — coverage targets, do's/don'ts, common errors
- [template-by-layer.md](./template-by-layer.md) — layer-specific requirements checklists
- [../usage/template-sapi.md](../usage/template-sapi.md) / [template-papi.md](../usage/template-papi.md) / [template-eapi.md](../usage/template-eapi.md) — XML templates per layer + complete working examples
- [../usage/template-common.md](../usage/template-common.md) — cross-layer patterns (target-attr, APIKit router, foreach fixtures, generic error handling, anti-patterns)
- [../usage/official-guidelines.md](../usage/official-guidelines.md) — XML examples for choice, scatter-gather, async, batch
- [../workflow/workflow.md](../workflow/workflow.md) — Mode A / Mode B step procedures
- [../workflow/pre-flight-validation.md](../workflow/pre-flight-validation.md) — self-check before declaring done
