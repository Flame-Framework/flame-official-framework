# Cross-Skill Integration Guide

How the `munit-generator` skill integrates with other skills in the pipeline. Follow these conventions to ensure smooth handoffs.

## Skill Pipeline

```
Solution Designer → RAML Designer → Mule Developer → MUnit Generator → Code Review
```

The MUnit Generator sits **between implementation and review** in the construction pipeline. It consumes what upstream skills produced (flow XML with `doc:id`s, RAML spec, solution design) and hands off to `code-review`. CI/CD then runs the generated test suite.

**Supported modes:**

- **Mode A — New test generation**
- **Mode B — Fix failing tests**

Most of the guidance below applies to both; differences are called out explicitly.

---

## Upstream: Mule Developer → MUnit Generator

The MUnit agent consumes what `mule-developer` produced. For the handoff to work, the dev agent must have produced code that is **mockable**.

### What the MUnit agent expects from the dev agent

| Dev-agent output | Why it matters for MUnit |
|---|---|
| Every component has a unique `doc:id` (UUID) | MUnit targets mocks by `doc:id` — without it, the mock can't bind |
| Every external call (`http:request`, `sap:*`, `db:*`, `os:*`, `wsc:*`, `jms:*`, `vm:*`, etc.) has a `doc:id` | Each external dependency must be individually mockable |
| Sub-flow names follow `naming-conventions.md` (kebab-case, `-subflow` suffix) | MUnit infers test-suite filenames from flow names |
| External connectors are isolated — one per processor block, not buried inside large flows | Each external dependency must be mockable in isolation |
| `vars.correlationId` is captured in the main flow from the inbound `x-correlation-id` header | MUnit relies on this being settable via `<munit:set-event>` in tests |
| Flow reads input from well-defined sources (`attributes.queryParams.X`, `vars.inputPayload.Y`, etc.) | Tests must be able to stage these values in `<munit:set-event>` |
| Every `<on-error-propagate>` uses a cataloged error `type` (`HTTP:CONNECTIVITY`, `APIKIT:BAD_REQUEST`, `APP:VALIDATION`, etc.) | MUnit error-scenario tests use the same `typeId` in `<munit-tools:error>` |

### What the MUnit agent does NOT expect

The dev agent does **not** write the mocks, fixtures, or assertions — that's entirely the MUnit agent's job. If something is missing for testability, escalate to the dev agent (or to the user) rather than modifying `src/main/`.

### When dev-agent output is insufficient for MUnit

If pre-flight validation fails because:

- An `<http:request>` has no `doc:id` → cannot mock → **escalate to `mule-developer`**
- A flow inlines multi-step logic with no sub-flow boundaries → cannot test in isolation → **escalate to `mule-developer`**
- Error types are not cataloged → cannot write negative-path tests → **escalate to `mule-developer`**

Never "fix" these by editing `src/main/`. The MUnit skill is test-layer only.

---

## Upstream: RAML Designer → MUnit Generator

The RAML specification drives the set of test scenarios to generate.

| RAML Element | What MUnit agent does |
|---|---|
| Resource path `/resources` | Maps to the endpoint flow `get:\resources:config-name` — test suite filename uses the entity (`get-resources-test-suite.xml`) |
| HTTP method (`GET`, `POST`, `PATCH`, `DELETE`) | Determines minimum scenarios per [standards/test-scenarios.md § 1](../standards/test-scenarios.md) (200/204/400/404/500 as applicable) + test-case naming pattern |
| URI parameter `/{id}` | `<munit:attributes>` sets `uriParams: { id: '…' }` |
| Query parameter `?status=` | `<munit:attributes>` sets `queryParams: { status: '…' }`. If DWL coerces to `LocalDateTime`, strip `Z` suffix |
| Request body type | Request payload fixture JSON in `src/test/resources/test-data/{entity}/` matching the RAML type |
| Response type | Happy-path mock fixture matching the RAML response type (or the real backend spec if different from the API response — see below) |
| Traits (e.g., `client-credentials`) | Triggers mocking of the bearer-token flow-ref in SAPI tests |
| Status codes in responses: block | Each declared status code becomes a test scenario |

### RAML response type vs. backend response

**Important:** The RAML response type is the **API's** response, but the mocked `http:request` simulates the **backend**. These are often different.

- **Mock input** must match what the DWL transform **reads** (backend shape).
- **Assertion** target matches what the DWL transform **produces** (RAML shape).
- For EAPI/PAPI: pull backend response shape from Anypoint Exchange (the consumed API's spec) via the `anypoint-mcp-server` MCP. For SAPI: ask the user for the backend system's actual response shape.
- Never infer the backend shape from the DWL file — DWL shows *usage*, not *structure*.

---

## Upstream: Solution Designer → MUnit Generator

When a solution design document is available, it is the **authoritative source** of test scenarios.

| Solution Design Section | How MUnit agent uses it |
|---|---|
| **Use Case Scenarios for Testing** | Authoritative list of required test cases — generate exactly these, no less |
| API Design tables | Cross-check endpoint definitions against RAML |
| Data mappings | Cross-check mock fixture shape against mapping source and target |
| Sequence diagram | Identify which sub-flows/external calls fire in which order — confirms mock targets |
| External system requests | Confirms which connectors need mocking |
| NFRs (performance, SLA) | Informs whether timeouts / retries need explicit tests |
| Error handling scenarios | Drives negative-path test generation (error types, expected error payloads) |

If no Solution Design exists for the project, the agent falls back to the standard scenarios in [standards/test-scenarios.md](../standards/test-scenarios.md) (minimum 3 per endpoint per HTTP method) and asks the user about edge cases.

---

## Downstream: MUnit Generator → code-review

The `code-review` skill may review the generated/fixed test suite. Use the compliance checklist below during pre-flight self-validation to ensure the tests pass review.

> **Sync Notice**: This checklist should map to the code-review rules for test files. If review rules change, update corresponding items here.

### Compliance Checklist — Test Quality

| Rule | Severity | Verification | KB Reference |
|---|---|---|---|
| T-I.1 | Blocking | Never mock the standard Mule `<logger>` on any layer. It's a no-op in tests | [workflow.md Do #13](./workflow.md) |
| T-I.2 | Blocking | Every variable referenced by the flow's loggers (e.g., `correlationId`) is set in `<munit:set-event>` | [workflow.md Do #10](./workflow.md) |
| T-I.3 | Blocking | Every mock payload has `encoding="UTF-8"` | [workflow.md Do #17](./workflow.md) |
| T-I.4 | Blocking | Every external call in the flow under test has a `mock-when` in every test that exercises that path | [workflow.md § Mock coverage rule](./workflow.md) |
| T-I.5 | Blocking | `doc:id` on `<munit-tools:with-attribute>` exactly matches a `doc:id` in the flow XML | [pre-flight-validation.md](./pre-flight-validation.md) Category 4 |
| T-I.6 | Blocking | For `<http:request target="X">`: mock preserves payload AND sets target variable | [workflow.md Do #19](./workflow.md) |
| T-I.7 | Blocking | For `<ee:cache>`-wrapped requests: `verify-call` uses `atLeast="0"` | [workflow.md Do #20](./workflow.md) |
| T-I.8 | Attention | Every endpoint has at least 3 scenarios per [standards/test-scenarios.md § 1](../standards/test-scenarios.md) | [standards/test-scenarios.md](../standards/test-scenarios.md) |
| T-I.9 | Attention | Test names follow the pattern `{method}-{entity}-test-{statusCode}-{scenario}` for all layers (e.g., `get-{entity}-test-200-ok`) | [standards/naming-conventions.md § 3](../standards/naming-conventions.md) |
| T-I.10 | Attention | Assertions use the `MunitTools::` prefix (never bare matcher calls) | [reference/allowed-components-list.md](../reference/allowed-components-list.md) |
| T-I.12 | Attention | External-format fixtures match the real API shape (OData backends, etc.) | [standards/template-by-layer.md § SAPI](../standards/template-by-layer.md) |

### Compliance Checklist — Fix Mode

| Rule | Severity | Verification | KB Reference |
|---|---|---|---|
| T-II.1 | **Blocking** | No files under `src/main/` were modified to make a test pass | [workflow.md](./workflow.md) Mode B absolute rule |
| T-II.2 | Blocking | All edits are under `src/test/munit/` or `src/test/resources/` | Mode B scope definition |
| T-II.3 | Attention | If the implementation was found to be wrong, the agent stopped and reported — did not silently patch the test to mask the bug | Mode B step F3 |

### Existing-test Compliance Scan (Mode B)

When repairing a failing test, scan the rest of the suite file for pre-existing violations of T-I.1 through T-I.12. Report them separately from the fix — do not silently "improve" unrelated tests without the user's approval.

---

## Downstream: MUnit Generator → CI/CD

There is no explicit "handoff" to CI/CD — but the generated tests run there. Items the MUnit agent must guarantee for CI/CD compatibility:

| Requirement | Why |
|---|---|
| **Every mock payload has `encoding="UTF-8"`** | Without it, `CursorStream` closes after first read — Studio tolerates, CI fails |
| **Tests don't depend on order** | CI may parallelize or reorder |
| **No real network calls** — every external dep mocked | CI has no access to backend systems |
| **`encrypt.key` is a placeholder** in any documentation or scripts shown to the user | Real keys never committed |
| **Coverage plugin config in `pom.xml`** with `failBuild>true` | CI gate on coverage target miss |
| **Tests are deterministic** — no `now()` compared with equality, no random seeds without control | Flaky tests poison the pipeline |

---

## What MUnit Will Mock vs. Test for Real

| Component the dev agent uses | MUnit treatment |
|---|---|
| `<http:request>` | **Mocked** |
| `<sap:sync-rfc>`, `<sap:*>` | **Mocked** |
| `<db:select>`, `<db:insert>`, etc. | **Mocked** |
| `<os:store>`, `<os:retrieve>`, `<os:retrieve-all>`, etc. | **Mocked** or spied with verify |
| `<mqtt:*>`, `<jms:*>`, `<vm:*>`, `<sqs:*>`, `<s3:*>`, `<salesforce:*>`, `<wsc:*>`, `<email:*>` | **Mocked** |
| `<ee:transform>` | **Tested for real** (no mock). Can be spied for verification. |
| `<logger>` (standard Mule) | Not mocked — no-op in tests. Set any referenced variable in `<munit:set-event>` |
| `<flow-ref>` (internal sub-flow) | Tested for real by default. May be mocked to isolate a single sub-flow. |
| `<flow-ref name="bearer-token">` | **Mocked** in SAPI OAuth2 tests |
| `<validation:*>` | **Tested for real** |
| `<apikit:router>` | Usually NOT mocked — call endpoint flow directly (see [standards/template-by-layer.md](../standards/template-by-layer.md) APIKit Router Mocking) |
| `<choice>`, `<foreach>`, `<parallel-foreach>`, `<scatter-gather>`, `<try>` | Tested for real — control flow is part of what we test |

The MUnit agent does NOT write mocks for components not in this list without explicit user approval.

---

## Checklist Before Handoff

Before declaring Mode A (new tests) or Mode B (fix) complete, verify:

- [ ] All items in [pre-flight-validation.md](./pre-flight-validation.md) PASS (coverage item pending test execution)
- [ ] `mvn clean test -Dencrypt.key=<YOUR_ENCRYPT_KEY> -Dmule.env=<YOUR_ENV>` run (after user authorization) and all tests green
- [ ] Coverage targets met (defaults: Application ≥ 80%, Flow ≥ 90%, Processor ≥ 80%; configurable via `config/framework.yaml` → `test.coverage.*-percent`)
- [ ] For Mode B: no files under `src/main/` modified
- [ ] Summary delivered to user: files created / modified, coverage achieved, anti-patterns caught and fixed
- [ ] If handing off to `code-review`: explicitly state "MUnit pre-flight validation passed"

---

## References

- [Workflow](./workflow.md) — Mode A and Mode B step procedures
- [Pre-flight Validation](./pre-flight-validation.md) — self-check categories and output format
- [MUnit Rules](./workflow.md) — do's/don'ts, HTTP scenarios, common errors
- [Templates by Layer](../standards/template-by-layer.md) — SAPI/PAPI/EAPI checklists
- [Allowed Components](../reference/allowed-components-list.md) — approved components catalog
