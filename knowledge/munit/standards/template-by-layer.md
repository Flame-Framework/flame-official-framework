# MUnit Templates by Layer (Reference)

> **Purpose:** Layer-specific guidance, quick-reference tables, requirement checklists, anti-pattern catalog, and payload-inference rules for SAPI / PAPI / EAPI MUnit tests. For the actual XML templates and complete working examples, see [usage/template-sapi.md](../usage/template-sapi.md), [usage/template-papi.md](../usage/template-papi.md), [usage/template-eapi.md](../usage/template-eapi.md), and [usage/template-common.md](../usage/template-common.md) (cross-layer patterns).

## Executive Summary

Layer-specific MUnit templates for MuleSoft 4 APIs across three architectural layers: System (SAPI), Process (PAPI), and Experience (EAPI). Each template follows established MuleSoft patterns to ensure consistency and prevent common errors.

### Which template to use

`{prefix}` is read from `config/framework.yaml` → `organization.project-prefix`. If empty, match by `*-{layer}` suffix only.

| API Layer | Pattern | Mock count | Primary focus | Complexity |
|-----------|---------|------------|---------------|------------|
| **System (SAPI)** | `{prefix}-*-sapi` | 1–2 mocks | Backend integration, OData/REST | Low |
| **Process (PAPI)** | `{prefix}-*-papi` | 6–10 mocks | Orchestration, aggregation | High |
| **Experience (EAPI)** | `{prefix}-*-eapi` | 1–3 mocks | UI transformation, async | Medium |

### Test naming conventions

See [naming-conventions.md § 3 Test-Case Names](./naming-conventions.md) for the full layer-specific name patterns.

### Common variables across all layers

| Variable | Layer | Purpose | Example |
|----------|-------|---------|---------|
| `httpStatus` | All | Store HTTP status for assertions | `200`, `404`, `500` |
| `storeError` | All | Capture error details for validation | `{code: "ERR001", message: "..."}` |
| `aggregatedData` | PAPI | Combined results from multiple SAPIs | `[{...}, {...}]` |
| `topic` | PAPI | Kafka/JMS topic name (async) | `orders-topic` |
| `processStatus` | PAPI | Orchestration state | `completed`, `failed` |
| `apiResponse` | All | Full HTTP response for complex asserts | Response object |
| `correlationId` | All | Request correlation across systems | UUID string |

When mocking a processor that uses a `target` attribute, always also mock the returned variable via `<munit-tools:variables>`.

### Required variables for ALL layers (EAPI, PAPI, SAPI)

Every test event must set:

| Variable | Value | MediaType | Required |
|----------|-------|-----------|----------|
| `correlationId` | `'"test-correlation-id-12345"'` | `application/java` | ALL LAYERS |

- `<munit:set-event>` must be the **first element inside `<munit:execution>`**, not inside `<munit:behavior>`.

---

## APIKit Router Mocking

**Mocking `apikit:router` is OPTIONAL** and rarely needed:

- Only mock when testing the router logic itself, bypassing routing behavior, or exercising routing-error edge cases.
- Normally, call the main flow directly (e.g., `get:suppliers:<your-prefix>-<backend-1>-sapi-config`) and let the router route internally. Focus on mocking external dependencies (`http:request`, Object Store, SAP, etc.). The standard Mule `<logger>` is a no-op in tests — do not mock it.

See the usage file for the optional mock snippet.

---

## Critical Requirements Checklists

### SAPI

**Must have:**
- [ ] Mock all external HTTP requests with the correct `doc:id`.
- [ ] Include `statusCode` in HTTP mock attributes.
- [ ] Use the `MunitTools::` prefix for all assertions.
- [ ] Test 200, 404, 500 scenarios minimum.

**Should have:**
- [ ] Mock bearer-token flow if the API uses OAuth2.
- [ ] Use `MunitTools::getResourceAsString()` for mock payloads.
- [ ] Verify the HTTP request was called with the correct `doc:id`.
- [ ] Assert field-level transformations.
- [ ] Test empty / null backend responses.
- [ ] Assert the transformed response structure.

#### External-system response shapes (SAPI-specific)

Fixtures must match what the real backend returns — not a generic JSON envelope. Using the wrong shape makes the test invalid even when it passes. Source fixture shapes from the real spec (Anypoint Exchange for the upstream system, or confirm with the user).

##### OData backend — OData error format

```json
{
  "odata.error": {
    "code": "",
    "message": { "lang": "en-US", "value": "Asset not found." }
  }
}
```

**Wrong** (standard JSON error — OData backends do not return this):
```json
{ "error": { "code": "BadRequest", "message": "Invalid request" } }
```

##### OData backend — "empty" response (nested-table envelope)

```json
{
  "d": {
    "results": [
      {
        "__metadata": { "id": "...", "uri": "...", "type": "..." },
        "PrimaryKey": "",
        "INNER_TABLE_1_T": { "results": [] },
        "INNER_TABLE_2_T": { "results": [] },
        "INNER_TABLE_3_T": { "results": [] }
      }
    ]
  }
}
```

Never use `{"d": {"results": []}}` alone — DWL transforms that access `payload.d.results[0].INNER_TABLE_1_T.results` will NPE.

##### DateTime query parameters

If the SAPI's DWL coerces to `LocalDateTime { format: "yyyy-MM-dd'T'HH:mm:ss" }`, **strip the `Z` suffix** in the test's `queryParams` values. Error symptom: `Cannot coerce String ("...Z") to LocalDateTime`.

### PAPI

**Must have:**
- [ ] Mock all SAPI HTTP calls (typically 4–6, minimum 2).
- [ ] Mock `<broker>` / queue operations when present.
- [ ] Use `<munit-tools:spy>` for orchestration validation.
- [ ] Initialize complex variables (`correlationId`, `scope`, `companyId`).
- [ ] Verify call counts for iterative operations.
- [ ] Test negative scenarios with `times="0"`.
- [ ] Mock Object Store for error flows.

**Should have:**
- [ ] Test scheduler flows with the `scope` variable set.
- [ ] Use inline payloads for complex data.
- [ ] Verify data-aggregation logic.
- [ ] Test partial failure scenarios.

### EAPI

**Must have:**
- [ ] Mock PAPI HTTP calls.
- [ ] Set `correlationId` in `<munit:set-event>`.
- [ ] Verify `ee:transform` components.
- [ ] Include `x-correlation-id` in headers.
- [ ] Assert UI-specific response fields.

**Should have:**
- [ ] Test async operations (202 responses).
- [ ] Assert client-friendly error messages.
- [ ] Verify correlation-ID propagation.
- [ ] Test experience-layer transformations.

#### EAPI logger handling

EAPI flows contain several standard Mule `<logger>` calls (start, end, debug points). They are no-ops at runtime in the test JVM and require no mocking. If a logger's `message` references a variable, set that variable in `<munit:set-event>` (e.g., `correlationId`).

---

## Iteration Scopes (`foreach` / `parallel-foreach`)

When a flow uses `<foreach>` or `<parallel-foreach>` to iterate over an array, each iteration receives a **single item** as payload. Mocks inside the scope that need to preserve the current item require a separate single-item fixture.

### Required fixtures

| Fixture | Purpose | Example path |
|---------|---------|--------------|
| Array file | Initial response returned by the upstream call | `test-data/quotations/<backend-1>-get-quotations.json` |
| Single-item file | Preserves payload inside the iteration | `test-data/quotations/<backend-1>-get-quotations-single.json` |

The single-item file must match one element of the array. Mocks inside the iteration that use a `target` attribute must reference the single-item file as payload so the current iteration item isn't overwritten.

See the usage file for a flow example and the single-item mock snippet.

---

## Anti-Patterns

Anti-patterns (compilation killers, runtime failures, logic errors) are consolidated in **[common-errors.md § 3](./common-errors.md)**. See the usage file ([usage/template-common.md § 5](../usage/template-common.md#5-anti-pattern-examples)) for wrong/correct XML snippets illustrating each.

---

## Payload Structure Inference from Final Transform

Analyze the **last** `ee:transform` in the flow to decide mock input structure and assertion shape. Five rules:

### Rule 1 — Array vs Object

- Transform uses `payload map` or returns `[...]` → **ARRAY** response → assert via `payload[0].field`.
- Transform returns `{...}` without `map` → **OBJECT** response → assert via `payload.field`.

### Rule 2 — Wrapper structure

- OData-style wrapper `payload.d.results` → mock with `{"d": {"results": [...]}}`.
- REST-style `payload.value` → mock with `{"value": [...]}`.
- No wrapper → mock the direct array/object.

### Rule 3 — Field mapping

For each field in the transform's output, generate an assertion for the **target** field name using the expected value derived from the mock's source field.

### Rule 4 — Conditional / empty responses

- Transform uses `if isEmpty(...)` or `default []` → create **two** scenarios: happy path (200 with data) and empty (typically 204).
- Transform sets `vars.httpStatus` conditionally → assert the variable to match the scenario.

### Rule 5 — Match mock to transform INPUT

Mock response structure must match what the transform **reads**, not what the final API returns. The transform handles the conversion.

### Decision flow

1. Locate the final transform (last `ee:transform` before the listener response).
2. Analyze output pattern → array vs object.
3. Check for a wrapper.
4. Map target field names.
5. Configure the mock to match the transform's input expectation.

### Layer-specific transform patterns

| Layer | Common pattern | Mock structure | Assertion focus |
|-------|----------------|----------------|-----------------|
| **SAPI** | `payload.d.results map {...}` | `{"d": {"results": [...]}}` | Field-level mapping validation |
| **PAPI** | `vars.aggregatedData` / complex merges | Multiple mocks per SAPI | Aggregation / orchestration results |
| **EAPI** | `{status: "SUCCESS", data: payload, message: "..."}` | PAPI response structure | UI-friendly wrapper fields |

---

## Quick Reference — Cross-Layer Tables

### Mock patterns by layer

| Mock Type | SAPI | PAPI | EAPI |
|-----------|------|------|------|
| HTTP Request | 1 (backend) | 2–6 (multiple SAPIs) | 1–2 (PAPI) |
| Bearer Token Flow | Optional | Never | Never |
| `logger` (standard Mule) | Never mocked | Never | Never |
| Object Store | Never | Common (errors) | Rare |
| Spy processor | Rare | Common (orchestration) | Rare |
| `ee:transform` | Never | Rare | Common |
| `<broker>` API | Never | Common | Never |

### Variable requirements by layer

| Variable | SAPI | PAPI | EAPI |
|----------|------|------|------|
| `correlationId` | Optional | Required | Required |
| `scope` | Never | Required | Optional |
| `companyId` | Optional | Common | Rare |
| `filter` / `id` | Common (OData) | Rare | Never |

### Assertion focus by layer

| Assertion Type | SAPI | PAPI | EAPI |
|----------------|------|------|------|
| Field-level values | Primary | Secondary | Secondary |
| Status codes | Always | Always | Always |
| Call counts | Simple (1) | Complex (0, 1, 2+) | Simple (1) |
| Data structure | Detailed | High-level | UI-focused |
| Error codes | Backend errors | Orchestration errors | User messages |

---

## Usage Instructions for Agents

1. Identify the API layer from the project-name pattern.
2. Select the appropriate template section (SAPI / PAPI / EAPI) in the usage file.
3. Copy the relevant template for the operation (GET, POST, PATCH).
4. Replace placeholders (`{entity}`, `{doc-id}`, etc.) with actual values.
5. Follow the requirements checklist for the layer.
6. Avoid the anti-patterns above.
7. Use the complete working examples as reference for complex scenarios.

**Document version:** 1.0.0 — validated against MuleSoft 4 MUnit test files across SAPI, PAPI, and EAPI layers.
