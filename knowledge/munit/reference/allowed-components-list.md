# MuleSoft MUnit — Allowed Components List (Reference)

> **Purpose:** Catalog of approved connectors, components, processors, matchers, and error types for MuleSoft projects. Use this file as the authoritative *"what is allowed"* reference. For XML and DWL code examples, see the companion file [usage/allowed-components-list.md](../usage/allowed-components-list.md).

---

## AI Context — Quick Reference

When generating MUnit tests, follow these priorities:

### Primary Components to Use (ALWAYS PREFER)

| Category | Component | Priority |
|----------|-----------|----------|
| **Mocking** | `munit-tools:mock-when` with `processor` attribute | HIGHEST |
| **Assertion** | `munit-tools:assert-that` with `MunitTools::notNullValue()` | HIGHEST |
| **Assertion** | `munit-tools:assert-that` with `MunitTools::equalTo()` | HIGHEST |
| **Verification** | `munit-tools:verify-call` with `times` attribute | HIGH |
| **Event Setup** | `munit:set-event` with `payload`, `attributes`, `variables` | HIGH |

### Processors to Mock (Priority Order)

1. `apikit:router` — ALWAYS mock for API flow tests
2. `http:request` — ALWAYS mock for external HTTP calls
3. `os:retrieve-all` / `os:retrieve` — Mock Object Store operations

The standard Mule `<logger>` is a no-op in tests — do not mock it. If a logger's `message` references a variable, set that variable in `<munit:set-event>`.

### Standard Test Structure

```
munit:test
├── munit:behavior      → Mock processors here
├── munit:execution     → Set event and call flow-ref
└── munit:validation    → Assert and verify here
```

### Key Matchers

| Matcher | Purpose |
|---------|---------|
| `MunitTools::notNullValue()` | Most common assertion |
| `MunitTools::equalTo(value)` | Value comparison |
| `MunitTools::nullValue()` | Null check |

Attribute matching always uses `doc:id`.

---

## Approved Connectors & Dependencies

### Primary Connectors

| Connector | Group ID | Artifact ID | Version | Usage |
|-----------|----------|-------------|---------|-------|
| **HTTP Connector** | org.mule.connectors | mule-http-connector | `<your-version>` | REST APIs, HTTP calls |
| **SAP Connector** | com.mulesoft.connectors | mule-sap-connector | `<your-version>` | SAP RFC/BAPI integration |
| **Object Store Connector** | org.mule.connectors | mule-objectstore-connector | `<your-version>` | Caching, error persistence |
| **Validation Module** | org.mule.modules | mule-validation-module | `<your-version>` | Input/business validation |
| **APIkit** | org.mule.modules | mule-apikit-module | `<your-version>` | REST API routing |

### SAP-Specific Libraries

| Library | Group ID | Artifact ID | Version |
|---------|----------|-------------|---------|
| SAP IDoc | com.sap.conn.idoc | sapidoc3 | `<your-version>` |
| SAP JCo | com.sap.conn.jco | sapjco3 | `<your-version>` |
| SAP JCo Native | com.sap.conn.jco | sapjco3 | `<your-version>` (DLL) |

### MUnit Testing Dependencies

| Dependency | Group ID | Artifact ID | Scope |
|------------|----------|-------------|-------|
| MUnit Runner | com.mulesoft.munit | munit-runner | test |
| MUnit Tools | com.mulesoft.munit | munit-tools | test |

---

## Core Mule Components Catalog

### HTTP / APIkit / Logging — always used

- `http:listener` — REST endpoint entry point.
- `http:request` — outbound HTTP calls.
- `apikit:router` — routes requests based on RAML spec.
- `apikit:console` — development API console.
- `logger` — standard Mule logger (level + plain-text message).

### Flow Control Components

| Component | XML Element | Purpose |
|-----------|-------------|---------|
| Choice | `<choice>` | Conditional routing |
| Foreach | `<foreach>` | Sequential iteration |
| Parallel Foreach | `<parallel-foreach>` | Parallel processing |
| Scatter-Gather | `<scatter-gather>` | Parallel execution with aggregation |
| Try-Catch | `<try>` | Error handling scope |
| Flow Reference | `<flow-ref>` | Modular flow composition |

### Data Components

| Component | XML Element | Purpose |
|-----------|-------------|---------|
| Transform Message | `<ee:transform>` | DataWeave transformations |
| Set Variable | `<set-variable>` | Variable assignment |
| Set Payload | `<set-payload>` | Payload manipulation |
| Remove Variable | `<remove-variable>` | Variable cleanup |

### SAP / Object Store / Validation

- SAP: `sap:sync-rfc` for BAPI/RFC calls.
- Object Store: `os:store`, `os:retrieve`, `os:retrieve-all`, `os:contains`, `os:remove`.
- Validation: `validation:is-null`, `validation:is-false`, and related validators from `mule-validation-module`.

---

## MUnit Testing Components

### Required Namespaces

Every MUnit test file must declare the `munit`, `munit-tools`, and core namespaces. Add `ee` and `os` namespaces *only* when mocking those connectors.

### MUnit Core (`munit:` namespace)

#### Test suite configuration

| Component | Required | Usage |
|-----------|----------|-------|
| `<munit:config>` | YES | Once per test file, configures the test suite |

#### Test lifecycle

| Component | Usage | Description |
|-----------|-------|-------------|
| `<munit:test>` | HIGH | Individual test case |
| `<munit:before-test>` | MEDIUM | Setup before each test |
| `<munit:before-suite>` | LOW | Setup once before all tests |

#### Test sections

| Component | Purpose | Required |
|-----------|---------|----------|
| `<munit:behavior>` | Mock and spy setup | YES |
| `<munit:execution>` | Set event and call flow | YES |
| `<munit:validation>` | Assertions and verifications | CONDITIONAL* |

**\* Validation section rules:**
- Required for tests that assert on results.
- **Omit entirely** for tests using `expectedErrorType` — do not include an empty `<munit:validation>` with only comments (MUnit requires at least one child).

#### Event setup

| Component | Usage | Description |
|-----------|-------|-------------|
| `<munit:set-event>` | HIGH | Sets up the test event |
| `<munit:payload>` | HIGH | Defines test payload content |
| `<munit:attributes>` | HIGH | Defines HTTP attributes (headers, uriParams, queryParams) |
| `<munit:variables>` | MEDIUM | Container for test variables |
| `<munit:variable>` | MEDIUM | Individual variable with key-value |

### MUnit-Tools (`munit-tools:` namespace)

#### Mocking

| Component | Priority | Description |
|-----------|----------|-------------|
| `<munit-tools:mock-when>` | PRIMARY | Main mocking component |
| `<munit-tools:with-attributes>` | PRIMARY | Container for attribute matchers |
| `<munit-tools:with-attribute>` | PRIMARY | Match by `doc:id` or `doc:name` |
| `<munit-tools:then-return>` | PRIMARY | Define mock return value |
| `<munit-tools:then-call>` | SECONDARY | Call another flow instead |
| `<munit-tools:payload>` | PRIMARY | Mock response payload |
| `<munit-tools:attributes>` | PRIMARY | Mock response HTTP attributes |

#### Verification

| Component | Priority | Description |
|-----------|----------|-------------|
| `<munit-tools:verify-call>` | PRIMARY | Verify processor was called |

#### Spying

| Component | Priority | Description |
|-----------|----------|-------------|
| `<munit-tools:spy>` | SECONDARY | Observe without modifying |
| `<munit-tools:before-call>` | SECONDARY | Capture data before execution |
| `<munit-tools:after-call>` | SECONDARY | Capture data after execution |

#### Assertions

| Component | Priority | Description |
|-----------|----------|-------------|
| `<munit-tools:assert-that>` | PRIMARY | Assert with matchers |
| `<munit-tools:assert-equals>` | PRIMARY | Simple equality check |
| `<munit-tools:assert>` | SECONDARY | Custom DataWeave assertion |
| `<munit-tools:that>` | SECONDARY | Contains CDATA expression |

---

## Allowed Processors for Mocking

### Primary targets (most common)

| Processor | Namespace | Priority | Description |
|-----------|-----------|----------|-------------|
| `apikit:router` | APIKit | HIGHEST | Main API router — most mocked |
| `http:request` | HTTP | HIGHEST | Outbound HTTP requests |
| `os:retrieve-all` | Object Store | HIGH | Retrieve all items |
| `os:retrieve` | Object Store | HIGH | Retrieve single item |
| `os:store` | Object Store | HIGH | Store item |
| `os:remove` | Object Store | MEDIUM | Remove item |
| `os:contains` | Object Store | MEDIUM | Check key existence |

> The standard Mule `<logger>` is a no-op in tests — do not mock it. Set any variable a logger message references in `<munit:set-event>`.

### Secondary targets

| Processor | Namespace | Description |
|-----------|-----------|-------------|
| `flow-ref` | Core | Flow references |
| `ee:transform` | Enterprise Edition | DataWeave transforms (usually spied) |
| `sap:sync-rfc` | SAP | SAP RFC calls |

---

## MunitTools Matcher Functions

### By priority

| Function | Priority | Usage Frequency | Description |
|----------|----------|-----------------|-------------|
| `MunitTools::notNullValue()` | PRIMARY | VERY HIGH | Assert value is not null |
| `MunitTools::equalTo(value)` | PRIMARY | VERY HIGH | Assert equals specific value |
| `MunitTools::nullValue()` | PRIMARY | MEDIUM | Assert value is null |
| `MunitTools::containsString(str)` | SECONDARY | MEDIUM | Assert contains substring |
| `MunitTools::greaterThan(num)` | SECONDARY | LOW | Assert greater than |
| `MunitTools::lessThan(num)` | SECONDARY | LOW | Assert less than |
| `MunitTools::anyOf(matchers)` | SECONDARY | LOW | Assert any matcher passes |
| `MunitTools::not(matcher)` | SECONDARY | LOW | Negate a matcher |
| `MunitTools::hasSize(size)` | SECONDARY | LOW | Collection size assertion |
| `MunitTools::isEmptyString()` | SECONDARY | LOW | Empty string assertion |

### Utility functions

| Function | Priority | Description |
|----------|----------|-------------|
| `MunitTools::getResourceAsString(path)` | PRIMARY | Load test resource files — heavily used |

### DataWeave test module (`dw::test::Asserts`)

| Function | Description |
|----------|-------------|
| `eachItem()` | Assert on each item in collection |
| `haveKey()` | Assert object has key |
| `notBe()` | Negation assertion |

> For MUnit assertion type-matching rules (`vars.httpStatus` Integer vs `payload.errors[0].code` String), see [workflow.md Testing Rules](../workflow/workflow.md) Do #23.

---

## Standard Error Types

The full error-type catalog (APIkit, HTTP, Database, Salesforce, File, Validation, SAP, Mule Core, Custom) lives in **[standards/test-scenarios.md § Error type reference](../standards/test-scenarios.md)**.

---

## Component Priority Tiers

### Tier 1 — ALWAYS use (project standard)

- `logger` — for ALL logging (level + plain-text message)
- `http:listener` / `http:request` — for HTTP operations
- `apikit:router` — for REST API routing
- `ee:transform` — for data transformations
- `flow-ref` — for flow composition
- `set-variable` — for variable management

### Tier 2 — use when needed (approved)

- `sap:sync-rfc` — SAP integration
- `os:store` / `os:retrieve` — object store operations
- `validation:*` — validation
- `choice` — conditional routing
- `foreach` / `parallel-foreach` — iteration
- `try` with error-handler — error handling

### Tier 3 — use with caution

- `scatter-gather` — parallel processing
- `until-successful` — retries
- `async` — async processing

### MUnit components to avoid

| Component | Reason |
|-----------|--------|
| `munit-tools:sleep` | Avoid artificial delays |
| `munit-tools:queue` | Not used in current tests |
| `munit-tools:dequeue` | Not used in current tests |
| `munit-tools:store` | Use Object Store mocking instead |
| `munit-tools:fail` | Prefer `expectedErrorType` attribute |
| `munit:after-test` | Not commonly used |
| `munit:after-suite` | Not commonly used |
| Third-party testing libraries | Stick to MunitTools and `dw::test::Asserts` |

---

## File Naming Conventions

For all MUnit naming rules (test suite files, test-case names, mock fixtures, directory structure, variable names, `doc:id` / `doc:name` conventions, namespace declarations), see **[standards/naming-conventions.md](../standards/naming-conventions.md)** — single source of truth.

---

