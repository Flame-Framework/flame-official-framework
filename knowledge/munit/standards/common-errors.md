# Common MUnit Errors (Standards)

> **Purpose:** Single source of truth for MUnit-runtime errors encountered in MuleSoft projects — error message → root cause → fix, error classification for Mode B, and anti-pattern errors. Use this when a test fails, when classifying a failure (Mode B fix workflow), or before declaring new tests ready (pre-flight check).
>
> **Related:** Error-type catalog (`expectedErrorType` / `<munit-tools:error typeId=...>`) → [test-scenarios.md § Error type reference](./test-scenarios.md). External-system response shapes (OData backends, empty-result envelopes) → [template-by-layer.md § SAPI](./template-by-layer.md).

---

## 1. Error Message → Root Cause → Fix

The most-seen failures during MUnit development and CI/CD runs.

| Error / symptom | Root cause | Fix |
|---|---|---|
| `Stream Compatible content required` | Using `value="..."` with external DWL (`resource="..."`) that expects a JSON stream | Use inline DataWeave with CDATA — not the `value` attribute |
| `Cannot coerce Null to String` | Variable extraction happens in the same `ee:transform` as `set-payload` — `ee:set-payload` runs BEFORE `ee:set-variable` and overwrites what the variable reads | Store input in `vars.inputPayload` in `<munit:set-event>`; extract with `<set-variable>` BEFORE the transform; reference `vars.inputPayload` inside the transform |
| `Cannot coerce String ("...Z") to LocalDateTime` | DWL coerces to `LocalDateTime { format: "yyyy-MM-dd'T'HH:mm:ss" }`; test sent ISO with `Z` | Strip trailing `Z` from `queryParams` datetime values when the DWL uses `LocalDateTime` |
| `Cannot open a new cursor on a closed stream` (CI/CD only; Studio tolerates) | Mock payload without `encoding="UTF-8"` returned as `CursorStream` — single-read only | Always add `encoding="UTF-8"` to every `<munit-tools:payload>` and `<munit-tools:variables>` in mock returns ([workflow.md Do #17](../workflow/workflow.md)) |
| `verify-call: Expected 1 but got 0 calls` on a cached request | HTTP request wrapped in `<ee:cache>` — first run populates cache, subsequent runs skip the processor | Use `atLeast="0"` instead of `times="1"` for `ee:cache`-wrapped HTTP requests |
| XML parsing errors (attribute value rejected) | Using `\"` (escaped quotes) inside XML attribute values | Use the XML entity `&quot;` instead of `\"` |
| Assertion passes silently — `verify-call` never matches | Mock `doc:id` doesn't match a real `doc:id` in the flow XML | Copy the `doc:id` verbatim from the flow file; validate via grep |
| `vars.httpStatus` is null in assertion | SAPI flows typically don't set `vars.httpStatus` | Don't assert `httpStatus` — use `verify-call` on the HTTP request instead |
| `Invalid input, expected '}'` during test execution | DWL syntax error (missing brace, bracket, comma) | Validate DWL — every `{` and `[` has a closing counterpart |
| Assertion type mismatch: `Expected: '400' but: 400 as Number` | Comparing Integer `vars.httpStatus` against String `'400'`, or vice versa | Use `equalTo(400)` (Integer) for `vars.httpStatus`; `equalTo('400')` (String) for `payload.errors[0].code` ([workflow.md Do #23](../workflow/workflow.md)) |
| Mock payload overwrites upstream payload (downstream transform sees null) | HTTP request uses `target="..."` — real Mule preserves upstream payload; mock overwrites it | Mock `then-return` payload must preserve the upstream data AND set the target variable via `<munit-tools:variables>` |
| `foreach` iteration mock fails — iteration payload is the whole array | Mock inside the iteration returns the array, not the single item | Create a **single-item fixture** JSON alongside the array fixture; reference the single-item file in the iteration mock |

---

## 2. Error Classification (Mode B — fix workflow)

When an MUnit test fails, classify the failure before touching code. Use the table below; most symptoms map to one of these categories. Pair this with §1's specific error table for the exact fix.

| Category | Typical symptoms | Primary fix location |
|---|---|---|
| **Setup error** | `Stream Compatible content required`, XML parse errors | `<munit:set-event>`, payload `mediaType` |
| **Mock-contract error** | Assertions see `null`; mock response doesn't match DWL input shape; wrong `doc:id` | Fixture JSON shape, `<munit-tools:then-return>` structure, `doc:id` in `<munit-tools:with-attribute>` |
| **Verify-count error** | `Expected 1 but got 0 calls` on an `ee:cache`-wrapped request | Change `times="1"` → `atLeast="0"` |
| **External-format error** | OData vs standard JSON mismatch; empty-result envelope shape; datetime `Z` suffix vs `LocalDateTime` | Fixture JSON must match the real external format (§4 below) |
| **Target-attribute overwrite** | Variable extraction returns `null`; downstream transform sees wrong payload | Mock `then-return` preserves payload AND sets the target variable |
| **Foreach / parallel-foreach** | Iteration mock gets the whole array | Single-item fixture JSON for mocks inside the iteration |
| **Variable-timing error** | `Cannot coerce Null to String` when extracting in the same `ee:transform` as `set-payload` | `vars.inputPayload` pattern — extract BEFORE the transform |

If no symptom maps to these categories, re-read the failing test before concluding the implementation is wrong. Never edit `src/main/` to silence a test failure — see Mode B absolute rule in [workflow/workflow.md](../workflow/workflow.md).

---

## 3. Anti-Pattern Errors (compilation / runtime / logic)

Issues that appear only when you run or compile the test — the test suite won't even start, or it starts and does the wrong thing.

### Compilation killers

- **HTML entities inside DataWeave** — using `&quot;` instead of plain quotes inside `#[...]` expressions. Use `&quot;` only inside XML attribute values, never inside DWL.
- **Missing `MunitTools::` prefix** — assertions like `is="#[notNullValue()]"` silently fail to compile. Always qualify: `is="#[MunitTools::notNullValue()]"`.
- **`<munit-tools:mock-when>` without `<munit-tools:with-attributes>`** — the mock never binds to a processor. Every `mock-when` must scope its target.

### Runtime failures

- **Missing variable referenced in a logger message** — if a logger has `message="... #[vars.correlationId] ..."` and the test doesn't set `correlationId`, the expression evaluates to `null` and downstream concatenation fails. Set every referenced variable in `<munit:set-event>`.

### Logic errors (compile + run fine, but test is wrong)

- **`doc:id` mismatch between mock and `verify-call`** — the verify never matches the mock, count is 0, test silently passes wrong state.
- **Mock returns wrong shape** — test passes because assertions match the mock, but real backend has a different shape. Always source fixture shape from the real spec (Anypoint Exchange for EAPI/PAPI, user for SAPI).
- **Mocking `apikit:router` when calling endpoint flow directly** — mock never fires because the router isn't in the call path. Call the main flow via `<flow-ref>` instead.

---

---

## References

- [workflow.md](../workflow/workflow.md) — the 24 Do's & 6 Don'ts this file's fixes reference
- [naming-conventions.md](./naming-conventions.md) — `doc:id` / `doc:name` / variable naming rules
- [test-scenarios.md](./test-scenarios.md) — required scenarios (every endpoint must have negative-path tests)
- [template-by-layer.md](./template-by-layer.md) — per-layer requirements and anti-patterns
- [../workflow/workflow.md](../workflow/workflow.md) — Mode A / Mode B procedures
- [../workflow/pre-flight-validation.md](../workflow/pre-flight-validation.md) — self-check that catches most of these errors before CI
- [../usage/official-guidelines.md](../usage/official-guidelines.md) — XML examples for error-handling scenarios
