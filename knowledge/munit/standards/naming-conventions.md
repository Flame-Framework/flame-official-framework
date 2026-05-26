# MUnit Naming Conventions (Standards)

> **Purpose:** Single source of truth for every naming rule in unit-testing — file names, directory structure, test-case names, fixture names, variable names, `doc:id`/`doc:name` conventions, and flow-name patterns used for layer detection.

---

## 1. Directory Structure

Test code and fixtures live under `src/test/` in the project, mirroring the structure of the flows under test. **One folder per domain entity** — folder names match the entity names used under `src/main/mule/`.

```
src/test/
├── munit/
│   ├── common/                              # shared test suites (optional)
│   └── {entity}/                            # one folder per domain entity
│       ├── get-{entity}-test-suite.xml
│       ├── post-{entity}-test-suite.xml
│       ├── patch-{entity}-test-suite.xml
│       └── scheduler/                       # optional: scheduler-flow tests
│           └── scheduler-{entity}-test-suite.xml
└── resources/
    ├── test-data/
    │   ├── common/                          # fixtures shared across entities
    │   │   └── apikit-event-base.json
    │   └── {entity}/                        # one folder per domain entity
    │       ├── <backend-1>-get-{entity}.json        # response mock (happy path)
    │       ├── <backend-1>-get-{entity}-empty.json  # variant
    │       ├── <backend-1>-post-{entity}.json       # create response
    │       ├── apikit-payload-{entity}.json # set-event body (UI / caller input)
    │       ├── apikit-event-{entity}-new.json # set-event attributes (variant)
    │       └── scheduler/                   # optional: scheduler-only fixtures
    │           └── <broker>-post-{entity}.json
    └── application.localtest.properties     # test-environment overrides
```

Rules:
- Suite files always end in `-test-suite.xml`. Never `-suite.xml` or any other variant.
- **All folders and filenames under `src/test/munit/` and `src/test/resources/test-data/` MUST be lowercase kebab-case** — no camelCase, no snake_case, no PascalCase. This applies to every segment of every name:
  - Folders: `exchange-rates/` ✓, `human-resources/` ✓ — never `exchangeRates/` ✗ or `human_resources/` ✗.
  - Files: `put-purchase-orders-test-suite.xml` ✓, `suppliers-fiscal-number-country-test-suite.xml` ✓ — never `put-purchaseOrders-test-suite.xml` ✗ or `suppliers-fiscalNumber-country-test-suite.xml` ✗.
  - When the implementation flow name contains camelCase (e.g., `purchaseOrders`, `fiscalNumber`, `exchangeRates`), split it into kebab-case for the test suite filename/folder: `purchase-orders`, `fiscal-number`, `exchange-rates`. The test suite name does NOT have to match the implementation flow name character-for-character — it must describe the same subject in kebab-case.
- All fixtures — response mocks, set-event payloads, variable-state fixtures — live in **one** `src/test/resources/test-data/{entity}/` folder per entity. The filename's verb (`get`/`post`/`payload`/`event`/`vars`) disambiguates. There is no separate `payloads/` folder.
- `{entity}` is kebab-case (e.g., `suppliers`, `work-orders`).
- Use a `common/` folder (sibling to entity folders) for fixtures shared across entities — mirrors `src/main/mule/common/` in the project.
- Add a `scheduler/` subfolder inside `munit/{entity}/` or `test-data/{entity}/` when the entity has scheduler-triggered flows that need their own suites or fixtures.

---

## 2. Test Suite File Names

Pattern: `{flow-name}-test-suite.xml` — where `{flow-name}` mirrors the flow name without namespace colons.

| Flow name | Suite file |
|---|---|
| `get:\suppliers:<your-prefix>-<backend>-sapi-config` | `get-suppliers-test-suite.xml` |
| `post:\work-orders:application\json:<your-prefix>-eapi-config` | `post-work-orders-test-suite.xml` |
| `purchase-orders-orchestration-flow` | `purchase-orders-orchestration-test-suite.xml` |

One suite per endpoint flow. Keep the `<munit:config name="...">` attribute equal to the filename — after any rename, update this attribute in lock-step with the file rename, otherwise test discovery / MUnit reporting will reference a stale name.

The entire filename must be kebab-case end-to-end. If any segment comes from a camelCase flow name, split it (`purchaseOrders` → `purchase-orders`, `fiscalNumber` → `fiscal-number`). Checklist before saving a new suite:
1. Folder is kebab-case.
2. Filename is kebab-case and ends in `-test-suite.xml`.
3. `<munit:config name="...">` equals the filename exactly.

---

## 3. Test-Case Names

### Pattern (all layers)

All layers — SAPI, PAPI, EAPI — use the same test-case name pattern:

```
{method}-{entity}-test-{statusCode}-{scenario}
```

| Layer | Example |
|---|---|
| **SAPI** | `get-suppliers-test-200-ok` |
| **PAPI** | `post-purchase-orders-test-201-created` |
| **EAPI** | `post-work-orders-test-201-created` |

`{scenario}` is the HTTP reason phrase in kebab-case (e.g., `200-ok`, `204-no-content`, `404-not-found`, `500-internal-server-error`). Use a more specific descriptor only when the extra precision is material (e.g., `503-connectivity`, `504-timeout`).

**Non-HTTP flows** (scheduler-triggered, VM-listener-triggered, etc.): drop `{statusCode}` and replace `{method}` with the trigger type:

```
{trigger}-{entity}-test-{scenario}
```

| Trigger type | Example |
|---|---|
| Scheduler | `scheduler-purchase-orders-test-execution`, `scheduler-{entity}-test-orchestration` |
| VM listener | `vm-{entity}-test-message-processed` |

### Standard scenario names per HTTP method

Full test-case name tables (GET / POST / PUT / PATCH / DELETE scenarios with status codes) live in [test-scenarios.md § 1](./test-scenarios.md). Use those names verbatim unless the Solution Design overrides.

Quick reference:
- GET → `get-{entity}-test-200-ok`, `get-{entity}-test-204-no-content`, `get-{entity}-test-500-internal-server-error`
- POST → `post-{entity}-test-201-created`, `post-{entity}-test-400-bad-request`, `post-{entity}-test-500-internal-server-error`
- PATCH/PUT → `patch-{entity}-test-204-no-content`, `patch-{entity}-test-404-not-found`, `patch-{entity}-test-500-internal-server-error`
- DELETE → `delete-{entity}-test-200-ok`, `delete-{entity}-test-404-not-found`, `delete-{entity}-test-500-internal-server-error`

### Test `description` attribute

Every `<munit:test>` carries a human-readable `description=` that names the scenario, not the mechanics: `"Test PATCH {entity} updates resource"`, not `"Test mock-when http:request returns 204"`.

---

## 4. Mock Fixture File Names

Pattern:

```
{source}-{verb}-{subject}[-variant].json
```

| Slot | Allowed values |
|---|---|
| `{source}` | System / origin of the payload: `<backend-1>`, `<backend-2>`, `<broker>`, `os`, `apikit` (use short kebab-case identifiers per backend; `os` for Object Store, `apikit` for caller payloads). Add new sources as systems are integrated. |
| `{verb}` | **Response mocks**: `get`, `post`, `patch`, `put`, `delete` — **Set-event inputs**: `payload` (body only), `event` (attributes / full event) — **Variable-state fixtures**: `vars` |
| `{subject}` | The entity the fixture targets (usually the entity name matching the folder, e.g., `suppliers`, `work-orders`); a finer-grained subject is allowed when an entity has multiple shapes (e.g., `supplier-header`, `work-order-items`) |
| `{variant}` *(optional)* | Scenario discriminator: `exists`, `not-found`, `empty`, `single`, `multiple`, `no-allocation`, `new`, etc. |

| Kind | Example |
|---|---|
| Backend GET response (happy path) | `<backend-1>-get-suppliers.json` |
| GET variant — empty collection | `<backend-1>-get-suppliers-empty.json` |
| GET variant — single record (for foreach) | `<backend-1>-get-suppliers-single.json` |
| GET variant — not found | `<backend-1>-get-suppliers-not-found.json` |
| Backend POST response | `<backend-1>-post-work-orders.json` |
| Backend POST variant — new record | `<backend-1>-post-work-orders-new.json` |
| Backend error response | `<backend-1>-get-suppliers-service-unavailable.json` |
| Set-event body (caller / UI input) | `apikit-payload-work-orders.json` |
| Set-event body variant | `apikit-payload-work-orders-new.json` |
| Set-event attributes / full event | `apikit-event-work-orders.json` |
| Variable-state fixture | `os-vars-correlation.json` |

Rules:
- **Lowercase kebab-case end-to-end** — both the fixture filename AND its containing folder. Every segment kebab-case, never mixed:
  - Folder: `test-data/exchange-rates/` ✓ — never `test-data/exchangeRates/` ✗ or `test-data/exchange_rates/` ✗.
  - Filename: `<backend-1>-get-exchange-rates.json` ✓, `root-message-empty.json` ✓ — never `rootMessage-empty.json` ✗ or `<backend-1>_get_suppliers.json` ✗.
  - When a fixture feeds a variable whose name is camelCase (e.g., `<munit:variable key="rootMessage" value="#[readUrl(...)]">`), the **variable name stays camelCase** (per §5) but the **filename is kebab-case**: `root-message-empty.json` read into variable `rootMessage`. File naming and variable naming are independent axes.
- **Do not** append `-response`, `-200`, or other status codes — the verb already implies the response context (`get` → 200-shaped response body; variant carries the scenario).
- The old mocks/ + payloads/ split is gone: request payloads now live in `test-data/{entity}/` with verb `payload` (body) or `event` (full event).
- Every fixture must be valid JSON (`jq .` parses without error) — see [workflow/pre-flight-validation.md § 8 F4](../workflow/pre-flight-validation.md).

---

## 5. Variable Names (Test-Internal)

Variables the agent sets inside `<munit:set-event>` or `<munit-tools:variables>` follow the same camelCase convention the implementation uses — and reuse the same reserved names.

| Variable | Type | Purpose | Required when |
|---|---|---|---|
| `correlationId` | String | Request-correlation identifier | PAPI/EAPI always; SAPI recommended |
| `httpStatus` | Integer | Expected HTTP status (assertions) | When the flow sets it via `set-variable`. **Always assert as Integer** (`equalTo(400)`), not string — see [workflow.md Do #23](../workflow/workflow.md) |
| `scope` | String | Scope / orchestration identifier | PAPI always |
| `companyId` | String | Multi-tenant identifier | PAPI commonly |
| `inputPayload` | Object | Original input preserved in `vars` before any transform | Pattern fix for "Cannot coerce Null to String" — see [workflow.md Don't #6](../workflow/workflow.md) |
| `bearerToken` | Object | OAuth2 bearer token set by `bearer-token` flow-ref mock | SAPI with OAuth2 |
| `filter` / `id` | String | OData query params | SAPI OData |

Test-only helper variables must also be camelCase and descriptive — avoid `tmp`, `x`, `data2`.

---

## 6. `doc:id` and `doc:name`

### `doc:id`

- Every processor in the **flow under test** must have a unique `doc:id` (UUID format — Anypoint Studio auto-generates).
- Mocks target processors by `doc:id`, not by `doc:name`, because multiple processors may share a `doc:name`.
- Copy the `doc:id` **verbatim** from the flow XML into the test's `<munit-tools:with-attribute whereValue="..." attributeName="doc:id"/>` — typos fail silently (mock never binds, test still passes on accident).
- Never author or mutate `doc:id` values by hand in tests — if a flow processor lacks `doc:id`, escalate to `mule-developer` rather than adding one yourself.

### `doc:name`

- Every processor must carry a meaningful `doc:name` (e.g., `"Get Suppliers from <backend-1>"`), not a generic placeholder (`"Logger"`, `"Transform"`).
- For `<munit-tools:mock-when>`, use `doc:name="Mock {what}"` (e.g., `Mock HTTP Request`, `Mock OS Retrieve`).
- For `<munit:test>`, `doc:name` is optional — the `name=` attribute is authoritative.

### `<flow-ref>`

Flow-refs target the sub-flow by `name`, not `doc:id`, so mock-when on `flow-ref` uses `attributeName="name"`:

```xml
<munit-tools:with-attribute whereValue="process-data-subflow" attributeName="name"/>
```

---

## 7. Flow-Name Patterns for Layer Detection

The agent detects the API layer from the project / flow name. `{prefix}` is read from `config/framework.yaml` → `organization.project-prefix`. If empty, match by `*-{layer}` suffix only.

| Pattern | Layer | Typical flow-name shape |
|---|---|---|
| `{prefix}-*-sapi` (project) | **SAPI** | `get:\resources:{config}-config`, `{entity}-subflow`, `bearer-token` |
| `{prefix}-*-papi` (project) | **PAPI** | `{entity}-orchestration-flow`, `{entity}-scheduler-flow`, `process-{entity}-subflow` |
| `{prefix}-*-eapi` (project) | **EAPI** | `post:\{resource}:application\json:{config}-config`, `{entity}-request-flow` |

Within a project, flow-name prefix drives the scenario approach — see [test-scenarios.md § 6 Flow Type → Test Approach](./test-scenarios.md).

---

## 8. Namespaces in Test XML

Required namespaces (always):
- `xmlns:munit`
- `xmlns:munit-tools`
- `xmlns:doc` (for `doc:name` / `doc:id`)
- `xmlns` (default = `mule:core`)
- `xmlns:xsi`

Conditional (add only when mocking those connectors):
- `xmlns:ee` — when mocking or spying `<ee:transform>`
- `xmlns:os` — when mocking Object Store operations

Full namespace example: [usage/allowed-components-list.md § Required MUnit Namespace Declarations](../usage/allowed-components-list.md).

---

## 9. Test Data / Configuration File Names

| File | Purpose |
|---|---|
| `application.localtest.properties` | Test-environment overrides in `src/test/resources/` |
| `config-{env}.yaml` (e.g. `config-local.yaml`, `config-dev.yaml`, `config-qa.yaml`, `config-prod.yaml` — environment names vary per organization) | Main app properties (not test-specific) |
| `secure-config-*.yaml` | Encrypted secure properties |

The `mule.env=localtest` global property in test suites routes the app to `application.localtest.properties`.

---

## References

- [test-scenarios.md](./test-scenarios.md) — canonical test-case names and scenarios
- [workflow.md](../workflow/workflow.md) — the Do's and Don'ts this file references
- [template-by-layer.md](./template-by-layer.md) — layer-specific naming checklists
- [../usage/allowed-components-list.md](../usage/allowed-components-list.md) — namespace & template examples
- [../workflow/pre-flight-validation.md](../workflow/pre-flight-validation.md) — self-check that validates these conventions
