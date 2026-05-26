# MUnit Official Guidelines (Reference)

> **Purpose:** Official MUnit testing methodology — planning, organization, component concepts, processor identification strategy, coverage, TDD, best practices. For code examples that accompany these guidelines, see [usage/official-guidelines.md](../usage/official-guidelines.md).

## Overview

MUnit is MuleSoft's application testing framework for integration and unit tests, fully integrated with Maven and Anypoint Studio.

### What is unit testing?

Unit testing validates the correctness of a single smallest testable part of an application. For Mule applications, the smallest testable part is a **flow or sub-flow**.

### Goals

- Assure that all functions and features of a single unit of code perform as specified.
- Test in isolation using mocked interfaces for external systems/resources.
- Ensure code changes don't break existing functionality.
- Provide documentation through test cases.

### Test suite

A MUnit Test Suite is a collection of MUnit tests meant to run independently of other suites, both on-demand during development and in CI regression runs.

---

## 3. Unit Test Preparation and Planning

### Identify units of code

Identify the flows or sub-flows that need to be tested. Each is a testable unit.

### Positive scenarios

For each positive scenario:
- State the condition (use as the MUnit test title).
- Identify input data, including all required ranges and discrete values.
- Identify output data, and define ranges/values that designate success.
- Identify all message processors that require mocking.
- Define the mock data that represents a successful execution of each mocked processor.

**A test case must exist for every condition or branch** — every path out of a `Choice` must have a test.

### Negative scenarios

For each negative scenario:
- State the error condition (use as the MUnit test title).
- Identify invalid inputs or error conditions.
- Identify output data and channels.
- Define ranges/values that designate an unsuccessful test.
- Define tests to ensure error conditions are correctly captured and published.
- Identify all processors that require mocking.

### Special attention areas

- **Routing components** (XPath or MEL expressions): place asserts *before* and *after* the routing component.
- **Data transformations** (XSLT / DataWeave): place asserts before and after. Note that DataWeave code inside a transform is **not** included in code coverage.

---

## 4. Test Suite Organization Strategies

| Strategy | Best for | Description |
|----------|----------|-------------|
| **By resource** | System APIs | One suite per API resource (Product, Order, Customer). |
| **By functionality** | Process / Experience APIs | One suite per business function; may include multiple units. |
| **By flow** | Modularity | One suite per flow — improves readability and maintainability. |
| **By convenience** | End-to-end tests | A suite composed of other suites to test the API as a whole. |

### Creating test files

- **Auto-generate:** Right-click the flow in Anypoint Studio → **MUnit → Create new MUnit suite**. File is created in `src/test/munit`.
- **Manual:** Right-click `src/test/munit` → **New → MUnit Test** → configure the suite name.

---

## 5. Common Connector Processor Reference

### Processor name format

`<namespace>:<operation>` — exactly matching what appears in the flow XML.

### Reference table

| Connector | Namespace | Common Operations | Example |
|-----------|-----------|-------------------|---------|
| HTTP | `http` | `request`, `listener` | `processor="http:request"` |
| Database | `db` | `select`, `insert`, `update`, `delete`, `bulk-insert` | `processor="db:select"` |
| Salesforce | `salesforce` | `query`, `create`, `update`, `upsert`, `delete` | `processor="salesforce:query"` |
| SAP | `sap` | `sync-rfc`, `async-rfc`, `outbound-delivery` | `processor="sap:sync-rfc"` |
| Amazon SQS | `sqs` | `send-message`, `receive-messages`, `delete-message` | `processor="sqs:send-message"` |
| Amazon S3 | `s3` | `put-object`, `get-object`, `delete-object`, `list-objects` | `processor="s3:put-object"` |
| File | `file` | `read`, `write`, `list`, `delete` | `processor="file:read"` |
| FTP / SFTP | `ftp` / `sftp` | `read`, `write`, `list`, `delete` | `processor="sftp:read"` |
| Email | `email` | `send`, `list-imap`, `list-pop3` | `processor="email:send"` |
| JMS | `jms` | `publish`, `consume`, `publish-consume` | `processor="jms:publish"` |
| VM | `vm` | `publish`, `consume`, `publish-consume` | `processor="vm:publish"` |
| ObjectStore | `os` | `store`, `retrieve`, `contains`, `remove` | `processor="os:store"` |
| Anypoint MQ | `anypoint-mq` | `publish`, `consume`, `ack`, `nack` | `processor="anypoint-mq:publish"` |
| Web Service Consumer | `wsc` | `consume` | `processor="wsc:consume"` |
| Mule Core | `mule` | `flow-ref`, `logger`, `set-variable`, `set-payload` | `processor="mule:flow-ref"` |

### Finding the processor name

1. Open the flow in Anypoint Studio.
2. Switch to **Configuration XML** view.
3. Read the element name (e.g., `<http:request>`, `<db:select>`).
4. The processor name is `namespace:element` (e.g., `http:request`, `db:select`).

---

## 6. Processor Identification for Mocks

### Priority order

When uniquely identifying a processor for a mock, use:

1. **`doc:name`** (primary) — when available and unique.
2. **`config-ref`** — when multiple processors of the same type exist with different configs.
3. **Operation-specific attributes** — `method`, `path`, `name`, etc.
4. **Combinations** — when a single attribute isn't unique.

### Decision tree

```
Is doc:name unique in the flow?
├── YES → Use doc:name only
└── NO  → Does config-ref differentiate them?
    ├── YES → Use doc:name + config-ref
    └── NO  → Add operation-specific attributes (method, path, …)
```

### Common attributes by connector

| Connector | Primary | Secondary |
|-----------|---------|-----------|
| `http:request` | `doc:name` | `method`, `path`, `config-ref` |
| `db:select` / `insert` / `update` | `doc:name` | `config-ref` |
| `salesforce:*` | `doc:name` | `config-ref` |
| `mule:flow-ref` | `name` | — |
| `sqs:send-message` | `doc:name` | `config-ref` |
| `file:read` / `write` | `doc:name` | `config-ref`, `path` |
| `vm:publish` | `doc:name` | `queueName`, `config-ref` |
| `jms:publish` | `doc:name` | `destination`, `config-ref` |

---

## 7. Key MUnit Components

- **`<munit:set-event>`** — defines the initial message (payload, attributes, variables) for the test.
- **`<munit-tools:mock-when>`** — returns a scripted response for a processor instead of executing it. Always set `mediaType` to match the real connector.
- **`<munit-tools:assert-that>`** — validates the state of the Mule event after execution. Prefer asserting full payloads as objects (`output application/java`), not as JSON strings. Assert media types and flow variables too. For timestamps, assert fields individually or check `notNullValue()`.
- **`<munit-tools:verify-call>`** — confirms a processor was invoked the expected number of times. Also helps prove the mock is wired correctly.
- **`<munit-tools:spy>`** — observes a processor before/after invocation without modifying it.

---

## 8. Why Mock External Dependencies

Unit tests must be:

- **Isolated** — no dependency on external systems, databases, REST services.
- **Stateless** — no state between runs.
- **Repeatable** — same result every time.
- **Independent** — no dependency on the execution environment.

All external resources must be mocked. Typical targets: HTTP, Database, Flow-ref, and error injections.

---

## 9. Testing Routing Components

### Choice

Create a separate test per branch. Each test sets the payload so exactly one `when`/`otherwise` branch triggers. Mock the sub-flow that branch calls. Validate with `verify-call times="1"` for the expected branch and `times="0"` for the others.

### Scatter-Gather

Executes routes in parallel. Mock each route's underlying processor independently (use `doc:name` to differentiate). Verify the total number of calls and the aggregated payload.

### First Successful

Mock the primary route to succeed; verify the fallback's processor was called `times="0"`.

---

## 10. Handling Complex Payloads

### String vs stream helpers

| Method | Use Case | Returns |
|--------|----------|---------|
| `getResourceAsString` | Small payloads, JSON, XML that fit in memory | String |
| `getResourceAsStream` | Large files, binary data, streaming behavior tests | InputStream |

Applies to XML (inline or file-backed, with or without namespaces), streaming payloads, binary payloads (e.g. PDFs), and Java-object payloads.

---

## 11. Testing Error Handling

Error-handling test strategies (expect-an-error, test handler logic, propagation through `flow-ref`, global handlers) and the catalog of error types (HTTP / DB / Salesforce / File / Validation / Mule Core / SAP / APIkit / custom) live in **[standards/test-scenarios.md § Error type reference](../standards/test-scenarios.md)**. XML examples for each strategy: [usage/official-guidelines.md § Testing Error Handling](../usage/official-guidelines.md).

---

## 12. Testing Cached Requests (`ee:cache`)

When an HTTP request is wrapped in `<ee:cache>`, the first test populates the cache; subsequent tests get cached results and the underlying processor never executes. `verify-call` with `times="1"` fails even though the mock is correctly wired.

- Use `atLeast="0"` in `verify-call` for cached requests.
- The mock still returns data; only the call count is unreliable.

See the usage file for the `verify-call` snippet.

---

## 13. Testing Async, Batch, VM

### Async scopes

- Mock the processors inside the async scope — use `verify-call` with a `timeout` to confirm the async processor ran.
- Alternatively, extract async logic into a sub-flow and test that sub-flow directly.

### VM

- Mock `vm:publish` and `vm:consume` with empty or scripted responses.
- Test VM-listener flows by calling the flow directly via `flow-ref` and providing a `<munit:set-event>` payload.

### Batch

- **Test each phase separately** — batch jobs return batch-job instance info; assert the return is non-null.
- **Extract step logic into sub-flows** and test those directly with a single-record payload.
- Enable the batch job's source in test config via `<munit:enable-flow-sources>`.

---

## 14. Before / After Test Hooks

| Hook | Scope | Use |
|------|-------|-----|
| `<munit:before-test>` | Each test | Shared mocks, per-test setup |
| `<munit:after-test>` | Each test | Per-test cleanup |
| `<munit:before-suite>` | Suite | Global setup |
| `<munit:after-suite>` | Suite | Global cleanup |

**Best practice:** define common mocks in `<munit:before-test>` to avoid duplication across tests.

---

## 15. Loading Test Resources

- `MunitTools::getResourceAsString('path')` — as String (most common).
- `MunitTools::getResourceAsStream('path')` — as InputStream (for large / binary data or streaming).

---

## 16. Common Matchers

- `MunitTools::equalTo(value)`
- `MunitTools::not(MunitTools::equalTo(value))` — negation
- `MunitTools::nullValue()` / `MunitTools::notNullValue()`
- `MunitTools::containsString('substring')`
- `MunitTools::greaterThan(n)` / `MunitTools::lessThan(n)`
- `MunitTools::hasSize(n)` — collection size
- `MunitTools::empty()` — empty collection

---

## 17. Testing with Environment Properties

Unit tests must be **independent of the execution environment**. Override `mule.env` globally in test suites (typically to `localtest`) and place environment-specific properties in `src/test/resources/application.localtest.properties`.

- Studio default: `mule.env=local`.
- MUnit override: `mule.env=localtest` via `<global-property>` in any test suite (declared once, applies to all suites).
- Runtime: value driven by the deployment environment.

Alternatively, use `<munit:parameterizations>` to pass per-suite property overrides.

---

## 18. Execution Procedures

### Developer

1. Run the unit-test suite **before every commit**.
2. If results disagree with expectations, fix the code (or update the test) and re-run.
3. Once all tests pass, commit.

From Studio: right-click the test file → **Run As → MUnit** → check the MUnit tab.

### DevOps / CI/CD

- Unit tests run as a regression suite against **every release build**.
- Failures open a defect against the API owner; if unresolvable within the DevOps timespan, the build is held.

### Coverage reports

- Studio: MUnit tab.
- Maven: `<app-home>/target/munit-reports/coverage/` — HTML and JSON formats.

---

## 19. Code Coverage

### Metrics

MUnit tracks three levels:

1. **Application Coverage** — average of resource/flow coverage.
2. **Resource Coverage** — per `src/main/mule` configuration file.
3. **Flow Coverage** — per Flow / Sub-flow / Batch job.

### DataWeave limitation

DataWeave code inside transforms is **not** included in coverage. High coverage does not guarantee all logic is tested.

### Ignoring files

Use the `<ignoreFiles>` plugin configuration for:
- Reusable Mule components (libraries).
- Global error handlers.
- Common utility flows tested indirectly.

### Best practices

- Aim for comprehensive coverage on critical business logic.
- High coverage is meaningless without proper assertions — use TDD.

---

## 20. Test Driven Development

### Process

1. Add a test for the feature you want to implement.
2. Run all tests and watch the new test **fail**.
3. Write just enough code to make it pass.
4. Run tests; confirm all pass.
5. Refactor while keeping tests green.
6. Repeat.

### Why make the test fail first?

Proves the test is actually testing something, is properly configured, and can detect failures.

### Benefits

Better coverage from the start; tests double as documentation; less debugging; simpler code; earlier bug detection.

---

## 21. Best Practices Summary

### Testing strategy

1. One condition = one test case — cover every branch.
2. Test positive AND negative scenarios.
3. Include missing-data scenarios.
4. Descriptive test names.
5. Keep tests independent.

### Mocking and isolation

6. Mock all external dependencies.
7. Set `mediaType` in mocks.
8. Do not depend on the execution environment.
9. Define common mocks in `before-test`.

### Assertions

10. Compare full payloads as objects, not JSON strings.
11. Assert media types.
12. Assert flow variables.
13. Use `verify-call` to prove mocks are invoked.
14. Handle timestamps carefully — field-by-field or `notNullValue()`.

### Organization

15. One test suite per flow.
16. Store test data in `src/test/resources`.
17. Organize suites consistently.
18. Clean up with `after-test` when needed.

### Coverage and quality

19. Aim for comprehensive coverage on critical logic.
20. Follow TDD.
21. Make tests fail first.
22. Test error propagation.
23. Test connectivity errors.

### CI/CD

24. Run tests before commits.
25. Run tests in CI/CD pipelines.
26. Configure coverage reports (HTML + JSON).
27. Ignore library/utility files where appropriate.

---

## 22. Integration Testing with MUnit

### Integration vs unit

| | Integration | Unit |
|---|-------------|------|
| Scope | End-to-end; multiple components | Individual units in isolation |
| External deps | Often real | All mocked |
| Speed | Slower | Fast |
| Focus | Complex scenarios, high-level ops | Specific logic branches |

### APIKit scaffold

Generates an entire test suite per API endpoint, configured for integration testing:
- HTTP Listener source is **enabled** (disabled by default in unit tests).
- Tests call the API via HTTP Request to simulate real usage.
- Exercises the full request/response cycle.

Enable flow sources via `<munit:enable-flow-sources>` — see the usage file for the exact snippet.

**Docs:**
- [Enable Flow Sources](https://docs.mulesoft.com/munit/2.1/enable-flow-sources-concept)
- [MUnit Scaffold Test Task](https://docs.mulesoft.com/munit/2.1/munit-scaffold-test-task)
