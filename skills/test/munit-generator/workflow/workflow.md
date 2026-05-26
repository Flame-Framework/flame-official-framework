# MUnit Agent — Workflow Instructions

## Overview

Two supported modes:

- **Mode A — New test generation:** build MUnit tests for endpoints that don't have them yet. The agent analyzes the flow, plans scenarios, generates mocks, and writes the test suite.
- **Mode B — Fix failing tests:** repair tests that are currently failing. The agent diagnoses the failure using only the **test-layer** (fixtures, mocks, expected values, test XML) and never modifies `src/main/` code.

Both modes apply across System (SAPI), Process (PAPI), and Experience (EAPI) API layers and follow the organization's established patterns.

---

## Mode Selection (first step — ALWAYS)

Before doing anything else, ask the user which mode applies (via `AskUserQuestion` — never plain-text choices):

- If **new tests** → follow **Mode A** below.
- If **fix failing tests** → skip Mode A and follow **Mode B — Fix Workflow** at the end of this document.

Never assume; always confirm the mode with the user.

---

## Three-Phase Pattern (both modes)

Every task — Mode A or Mode B — follows the same three-phase gated pattern:

```
Phase 1: Analysis      — gather context (read-only); ask the user any ambiguous question
      ↓
Phase 2: Planning      — present a structured plan to the user; WAIT for explicit approval
      ↓
Phase 3: Implementation — only after approval; apply changes; pre-flight validate; report
```

**Rules that apply to both modes:**

- **Never flow silently from Phase 1 to Phase 2, or from Phase 2 to Phase 3.** Approval gates are explicit (user answers via `AskUserQuestion`).
- **Ask for clarity** whenever the flow structure, test scope, mock source, or root cause is ambiguous. Don't guess.
- **Never run `mvn test`** without an explicit authorization popup.
- Every user question uses `AskUserQuestion` (no plain-text prompts).
- The SKILL.md (`skills/test/munit-generator/SKILL.md`) is the entry point that sequences each step; this file contains the technical reference for each step's content and requirements.

---

## Mandatory Requirements

### Coverage targets

- Coverage reports must be generated and validated before deployment.
- All flows and sub-flows must have corresponding test cases.

Default thresholds (configurable in `config/framework.yaml` under `test.coverage.*-percent`):

| Metric | Target |
|--------|--------|
| Application Coverage | 80% |
| Flow Coverage | 90% |
| Processor Coverage | 80% |

### Mock coverage rule

**All external calls in the flow under test MUST be mocked.** Every HTTP request, database call, SAP connection, ObjectStore operation, and any other external dependency must have a corresponding `mock-when` processor. No external call should be left unmocked.

---

## Testing Rules — Do's and Don'ts

These are the organization-specific rules the agent enforces. Cross-references elsewhere in the KB use "workflow.md Do #N" to point back here.

### Do's

1. Mock all external dependencies (HTTP, database, SAP, ObjectStore).
2. Use test data files stored in `src/test/resources/test-data/`.
3. Test error scenarios, not just happy paths.
4. Validate HTTP status codes in addition to payload content — verify actual API behavior (e.g., `POST /requests` returns `201`, not `200`; empty GETs return `204`, not `200 []`).
5. Use meaningful assertion messages for debugging.
6. Run MUnit tests before pushing to QA.
7. Create test scenarios for any code change in legacy APIs.
8. Use `doc:id` for mocking — always target processors via `doc:id`.
9. Call endpoint flows directly (e.g., `get:\suppliers:<your-config-name>`) instead of going through the main flow + APIKit router mock.
10. Set every variable referenced by the flow under test in `<munit:set-event>` (e.g., `correlationId`). Loggers in the flow are standard Mule `<logger>` no-ops at runtime, but any variable they reference must be set or the expression will fail.
11. Use inline DataWeave with CDATA for complex payloads — avoids stream compatibility issues with external DWL files.
12. Store input data in variables (`vars.inputPayload`) to preserve original data during transformations.
13. Do not mock `<logger>`. The standard Mule logger is a no-op at runtime in tests and never needs mocking.
14. Use XML entities for quotes (`&quot;` instead of `\"`) in XML attribute values.
15. Never expose `encrypt.key`. Use a placeholder in commands shown to the user: `mvn clean test -Dencrypt.key=<YOUR_ENCRYPT_KEY> -Dmule.env=<YOUR_ENV>`.
16. Always ask the user before running tests. Never execute `mvn test` autonomously.
17. **Always add `encoding="UTF-8"`** to every `<munit-tools:payload>` and `<munit-tools:variables>` in mock returns. Without it, MUnit returns a `CursorStream` that can only be read once; Studio tolerates re-reads (forgiving buffer), CI/CD (stricter memory) fails with `Cannot open a new cursor on a closed stream`. The `encoding` attribute forces String conversion so the payload re-reads safely.
18. **Get response structures from specs, not from DWL.** DWL shows how data is *used*, not how it's *structured*. EAPI/PAPI: pull via Anypoint Exchange (`anypoint_get_api_specification` / `anypoint_get_api_endpoints`, exposed by the `anypoint-mcp-server` MCP — requires the MCP configured with valid org credentials). SAPI: ask the user for the backend system's actual response.
19. For HTTP requests using the `target` attribute: the mock's `then-return` payload must preserve the data the downstream code expects AND set the target variable explicitly via `<munit-tools:variables>`. A naive mock overwrites the payload.
20. For HTTP requests wrapped in `<ee:cache>`: use `atLeast="0"` in `verify-call`. Cached requests may not execute on every run; exact counts are unreliable.
21. For `foreach` / `parallel-foreach` scopes: create a **single-item** fixture JSON file for mocks inside the iteration that need to preserve the current item. The array fixture is for the initial response; the single-item fixture matches one element from the array.
22. Analyze the complete flow — every `http:request`, `flow-ref`, transform, `target` attribute, `foreach` / `parallel-foreach`, `ee:cache` — before writing any mock.
23. **Type-match assertions correctly.** `vars.httpStatus` is set as an **Integer** in flows (e.g., `set-variable value="400"`) — assert with Integer: `equalTo(400)`. `payload.errors[0].code` is typically a **String** — assert with `equalTo('400')`. String-vs-Integer mismatches fail with `Expected: '400' but: 400 as Number`.
24. **When testing flows routed through APIkit**, set `vars.correlationId` in `<munit:set-event>` — it's normally initialized by the main flow before the router from the inbound `x-correlation-id` header, and tests that call the endpoint flow directly skip that initialization.

### Don'ts

1. Don't skip error handling tests.
2. Don't hardcode test data in XML — use external files.
3. Don't assert `vars.httpStatus` unless the flow explicitly sets it. SAPI flows typically don't.
4. Don't pre-set variables to make tests pass. If assertions fail on null variables, investigate why the flow doesn't set them.
5. Don't use the `value` attribute for complex payloads when using external DWL files — use inline DataWeave with CDATA instead.
6. Don't extract variables in the same `ee:transform` as the payload. The payload transforms first, making original values unavailable.

---

## AI Assistant Instructions

When generating MUnit tests:

1. Always request the Solution Design document to understand expected test scenarios.
2. Follow the documented naming conventions ([naming-conventions.md](../standards/naming-conventions.md)) for test files and test names.
3. Include assertions for both payload and HTTP status codes.
4. Add error-scenario tests based on the HTTP method being tested.
5. Use external test data files rather than inline data.

---

## Final Checklist

Before declaring any MUnit test suite done:

- [ ] Happy path scenario implemented
- [ ] No content / empty response scenario implemented (where applicable)
- [ ] Error scenarios implemented (400, 404, 500, etc.)
- [ ] All external calls are mocked
- [ ] Test data files are in `src/test/resources/`
- [ ] Solution Design test scenarios are covered
- [ ] Meaningful test names and descriptions
- [ ] All tests pass locally
- [ ] **`encoding="UTF-8"` on every mock payload** (CI/CD stream fix — Do #17)
- [ ] Every variable referenced by the flow's loggers (e.g., `correlationId`) is set in `<munit:set-event>` (Do #10)
- [ ] `<logger>` is NOT mocked anywhere in the suite (Do #13)
- [ ] `doc:id` present on every HTTP request and flow-ref in the implementation (Do #8)
- [ ] For HTTP requests with `target`: payload preserved AND target variable set explicitly (Do #19)
- [ ] For `foreach` / `parallel-foreach`: single-item fixture JSON created (Do #21)
- [ ] For `<ee:cache>`-wrapped requests: `atLeast="0"` in `verify-call` (Do #20)
- [ ] DateTime `queryParams` match the DWL-expected format (strip `Z` when DWL uses `LocalDateTime { yyyy-MM-dd'T'HH:mm:ss }`)
- [ ] EAPI/PAPI: response structures pulled from Anypoint Exchange, not inferred from DWL (Do #18)
- [ ] SAPI: backend response structure confirmed with user (Do #18)
- [ ] Pre-flight validation passed — see [pre-flight-validation.md](./pre-flight-validation.md).

---

# Mode A — New Test Generation

## Mode A — Phase Map

| Phase | Steps | Output | Gate |
|---|---|---|---|
| **Phase 1A — Analysis** | §1 Agent Capabilities, §2 Input Requirements, §4 Step 1, §5 Step 2, §6 Step 3, §7 Step 4 (plan only) | `PROJECT_ROOT`, `TARGET_FLOWS`, `LAYER`, `TARGET_ENDPOINTS`, `SD_SCENARIOS` (if any), `MOCK_SOURCE`, `FLOW_MAP`, `SCENARIO_PLAN` | — |
| **Phase 2A — Planning** | Present `SCENARIO_PLAN` + `MOCK_SOURCE` summary for approval | `APPROVED_PLAN` | ⚠️ Wait for user approval via `AskUserQuestion` |
| **Phase 3A — Implementation** | §7 Step 4 (generate fixtures), §8 Step 5 (generate XML), §9 Step 6 (pre-flight + execute) | Test suite files, fixtures, coverage report | — |

**Never enter Phase 3A without explicit "Approve" from the user.** The skill template walks the user through each step interactively; this file defines what each step must produce.

## Mock Data Source (ask early in Phase 1A)

Mocks are the load-bearing part of an MUnit suite. Ask the user via `AskUserQuestion` how to source fixture content. Four options:

| Option | How it works | Best when | Caveat |
|---|---|---|---|
| **Infer from implemented code** | Analyze the flow's DWL transforms + existing fixtures in the project + upstream API specs on disk; generate mock JSON that matches the transform's input shape | Fast iteration; DWL is well-commented; upstream specs are available locally | The inferred shape may not match the real backend — flag uncertainty in the plan |
| **User provides files / folder** | User supplies fixture JSON, captured backend responses, or Solution Design sample payloads; the agent copies them into `src/test/resources/test-data/...` renamed per `naming-conventions.md` §4 | Real backend responses already captured; Solution Design has concrete samples | User must provide the files; errors in user-provided data pollute the test |
| **Use `postman-mcp-server`** | Agent calls the Postman MCP server to fetch sample responses from saved collections; uses returned bodies as fixture content | Team already maintains a Postman collection for the backend; real responses | Requires MCP server configured and collection IDs available |
| **Use Anypoint Exchange** | Agent calls `anypoint_get_api_specification` / `anypoint_get_api_endpoints` (exposed by the **`anypoint-mcp-server`** MCP) to fetch the consumed-API spec; derives fixture shapes from published examples and schemas | EAPI/PAPI consuming a published API; the upstream team maintains the Exchange asset with examples | Requires `anypoint-mcp-server` configured with valid org credentials. EAPI/PAPI only — there is no upstream API to fetch for SAPI; spec must include response examples |

After the user picks one primary option, the agent MAY offer to **combine** with Anypoint Exchange (for EAPI/PAPI) or with user-provided files, when that gives a more accurate mock (e.g., infer shape from Exchange spec, values from user samples).

Record the user's choice as `MOCK_SOURCE` and any follow-up inputs (path, collection ID, Exchange asset ID) before building the plan.

---

## MANDATORY: Required Reading Before Writing Any Test

**Before writing any MUnit test, you MUST read these files in full using the Read tool. Do NOT use Explore agents or summaries - critical rules will be missed.**

1. **The Testing Rules (Do's / Don'ts) above** — the 24 Do's and 6 Don'ts are foundational to every test.
2. **[standards/test-scenarios.md](../standards/test-scenarios.md)** — canonical scenarios per HTTP method + flow-triggered extras.
3. **[standards/common-errors.md](../standards/common-errors.md)** — error classification + fixes (especially for Mode B).
4. **[standards/naming-conventions.md](../standards/naming-conventions.md)** — file, test-case, fixture, and variable naming.
5. **[standards/template-by-layer.md](../standards/template-by-layer.md)** — per-layer requirements, anti-patterns, external-system shapes.

This is non-negotiable. Summaries lose critical details that cause incorrect tests.

---

## Reference Documentation

The supporting documentation is split between `reference/` (rules, tables, priorities, checklists) and `usage/` (XML/DWL examples). Consult both for each topic:

| Document | Reference | Usage | Purpose |
|----------|-----------|-------|---------|
| **Testing Rules (Do's / Don'ts)** ⚠️ | This file — § Testing Rules above | — | **MUST READ FIRST** — 24 Do's, 6 Don'ts, coverage targets, mock coverage rule |
| **Test Scenarios** ⚠️ | [standards/test-scenarios.md](../standards/test-scenarios.md) | — | **MUST READ FIRST** — canonical scenarios, per-layer counts, flow-triggered extras |
| **Common Errors** | [standards/common-errors.md](../standards/common-errors.md) | — | Error message → cause → fix; Mode B classification |
| **Naming Conventions** | [standards/naming-conventions.md](../standards/naming-conventions.md) | — | Files, test cases, fixtures, variables, doc:id |
| **Allowed Components** | [reference/allowed-components-list.md](../reference/allowed-components-list.md) | [usage/allowed-components-list.md](../usage/allowed-components-list.md) | Approved connectors, processors, matchers, patterns |
| **MUnit Templates by Layer** | [standards/template-by-layer.md](../standards/template-by-layer.md) | [usage/template-sapi.md](../usage/template-sapi.md), [template-papi.md](../usage/template-papi.md), [template-eapi.md](../usage/template-eapi.md), [template-common.md](../usage/template-common.md) | SAPI/PAPI/EAPI templates with complete examples (common patterns in `template-common.md`) |
| **MUnit Official Guidelines** | [reference/official-guidelines.md](../reference/official-guidelines.md) | [usage/official-guidelines.md](../usage/official-guidelines.md) | Official MUnit methodology, structure, best practices |
| **DataWeave** | [reference/dataweave.md](../reference/dataweave.md) | [usage/dataweave.md](../usage/dataweave.md) | DataWeave 2.x reference and transformation patterns |

### When to Consult Each Document

- **For DataWeave expressions in mocks/assertions** → `reference/dataweave.md` + `usage/dataweave.md`
- **For the list of allowed components, priorities, and error types** → `reference/allowed-components-list.md`
- **For XML/DWL usage examples of those components** → `usage/allowed-components-list.md`
- **For test structure and naming** → `reference/official-guidelines.md` + `usage/official-guidelines.md`
- **For layer-specific templates** → `standards/template-by-layer.md` + `usage/template-sapi.md` / `template-papi.md` / `template-eapi.md` (layer-specific) + `usage/template-common.md` (cross-layer)
- **For coverage targets and testing rules (Do's / Don'ts)** → this file § Mandatory Requirements + § Testing Rules

---

## Table of Contents

1. [Agent Capabilities](#1-agent-capabilities)
2. [Input Requirements](#2-input-requirements)
3. [Workflow Overview](#3-workflow-overview)
4. [Step 1: Project Analysis](#4-step-1-project-analysis)
5. [Step 2: Flow Analysis](#5-step-2-flow-analysis)
6. [Step 3: Test Scenario Planning](#6-step-3-test-scenario-planning)
7. [Step 4: Mock Data Generation](#7-step-4-mock-data-generation)
8. [Step 5: MUnit Test Generation](#8-step-5-munit-test-generation)
9. [Step 6: Test Execution & Validation](#9-step-6-test-execution--validation)
10. [Layer-Specific Instructions](#10-layer-specific-instructions)
11. [Reference Catalogs (moved out of this file)](#reference-catalogs-moved-out-of-this-file)
12. [Appendix B: Agent Execution Checklist](#appendix-b-agent-execution-checklist)
13. [Mode B — Fix Failing Tests](#mode-b--fix-failing-tests)

---

## 1. Agent Capabilities

### What This Agent Does

- **Analyzes MuleSoft flow XML files** to extract processor information, `doc:id` values, and flow structure
- **Auto-detects API layer** (SAPI/PAPI/EAPI) from project naming conventions
- **Generates MUnit test files** following the organization's established patterns
- **Creates mock data JSON files** in the appropriate test resources directory
- **Executes tests** and validates code coverage
- **Reports coverage status** and flags tests needing review

### Scope

- Generates tests for **one flow at a time** as specified by the user
- Creates both **MUnit XML test files** and **mock JSON data files**
- Writes files **directly to the filesystem** in the appropriate locations
- **Appends new scenarios** to existing test files or asks user preference

---

## 2. Input Requirements

### Required Information

| Input | Source | Description |
|-------|--------|-------------|
| **Flow XML File Path** | User provides | Path to the flow XML file to test |
| **Project Root Path** | User provides or detect | Root directory of the MuleSoft project |

### Optional Information

| Input | Source | Fallback |
|-------|--------|----------|
| **Solution Design Document** | User provides | Generate tests based on standard HTTP patterns |
| **Specific Test Scenarios** | User provides | Auto-generate based on HTTP method |
| **API Layer** | Auto-detect from project name | Ask user if detection fails |

### Project Name Patterns for Layer Detection

```
*-sapi  → System Layer (SAPI)
*-papi  → Process Layer (PAPI)
*-eapi  → Experience Layer (EAPI)
```

---

## 3. Workflow Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    MUnit Agent Workflow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. PROJECT ANALYSIS                                             │
│     ├── Detect API layer from project name                       │
│     ├── Locate pom.xml and validate MUnit dependencies           │
│     └── Identify src/test/munit and src/test/resources paths     │
│                                                                  │
│  2. FLOW ANALYSIS                                                │
│     ├── Parse flow XML file                                      │
│     ├── Extract all processor doc:id values                      │
│     ├── Identify HTTP requests, transforms, loggers              │
│     └── Map flow structure and dependencies                      │
│                                                                  │
│  3. TEST SCENARIO PLANNING                                       │
│     ├── Check for Solution Design document scenarios             │
│     ├── Generate standard scenarios based on HTTP method         │
│     └── Plan minimum 3 test scenarios per endpoint               │
│                                                                  │
│  4. MOCK DATA GENERATION                                         │
│     ├── Create mock response JSON files                          │
│     ├── Create request payload JSON files                        │
│     └── Store in src/test/resources/test-data/{entity}/          │
│                                                                  │
│  5. MUNIT TEST GENERATION                                        │
│     ├── Generate test file following layer template              │
│     ├── Include all required mocks with correct doc:id           │
│     ├── Add assertions for payload and status codes              │
│     └── Write to src/test/munit/{flow-name}-test-suite.xml       │
│                                                                  │
│  6. TEST EXECUTION & VALIDATION                                  │
│     ├── Run: mvn clean test -Dencrypt.key=... -Dmule.env=...    │
│     ├── Check compilation and test pass/fail status              │
│     ├── Validate code coverage                            │
│     └── Report results and flag issues                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Step 1: Project Analysis

### 4.1 Detect API Layer

**Action:** Analyze project directory name to determine API layer.

```
Pattern Matching:
- Contains "-sapi" → SAPI (System Layer)
- Contains "-papi" → PAPI (Process Layer)
- Contains "-eapi" → EAPI (Experience Layer)
```

**If detection fails:** Ask user to specify the layer.

### 4.2 Validate Project Structure

**Required directories:**
```
{project-root}/
├── src/
│   ├── main/
│   │   └── mule/           # Flow XML files
│   └── test/
│       ├── munit/          # MUnit test files (create if missing)
│       └── resources/      # Test data and mocks (create if missing)
│           └── mocks/      # Mock response files
└── pom.xml                 # Project dependencies
```

**Action:** Create missing test directories if they don't exist.

### 4.3 Validate MUnit Dependencies

**Action:** Check `pom.xml` for required dependencies:

```xml
<!-- Required MUnit Dependencies -->
<dependency>
    <groupId>com.mulesoft.munit</groupId>
    <artifactId>munit-runner</artifactId>
    <classifier>mule-plugin</classifier>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>com.mulesoft.munit</groupId>
    <artifactId>munit-tools</artifactId>
    <classifier>mule-plugin</classifier>
    <scope>test</scope>
</dependency>
```

**If missing:** Notify user that dependencies need to be added.

---

## 5. Step 2: Flow Analysis

### 5.1 Parse Flow XML

**Action:** Read and parse the specified flow XML file.

**Extract the following information:**

| Element | Attribute | Purpose |
|---------|-----------|---------|
| `<flow>` or `<sub-flow>` | `name` | Flow name for test naming |
| `<http:request>` | `doc:id`, `doc:name`, `method`, `path`, `config-ref` | Mock configuration |
| `<http:listener>` | `doc:id`, `path`, `config-ref` | Endpoint path |
| `<ee:transform>` | `doc:id`, `doc:name` | Transform verification |
| `<os:store>`, `<os:retrieve>` | `doc:id` | Object Store mocking |
| `<flow-ref>` | `name`, `doc:id` | Sub-flow mocking |
| `<sap:sync-rfc>` | `doc:id`, `doc:name` | SAP mocking |
| `<apikit:router>` | `doc:id` | APIKit router mocking (optional) |
| `<choice>` | Children | Branch testing requirements |

### 5.2 Create Processor Map

**Action:** Build a map of all processors requiring mocks:

```json
{
  "httpRequests": [
    {
      "docId": "abc-123-def",
      "docName": "Call Backend API",
      "method": "GET",
      "path": "/api/endpoint",
      "configRef": "HTTP_Request_Config"
    }
  ],
  "transforms": [
    {
      "docId": "xyz-789",
      "docName": "Transform Response"
    }
  ],
  "loggers": [
    {
      "docId": "log-001",
      "docName": "Log start",
      "level": "INFO"
    }
  ],
  "objectStores": [],
  "flowRefs": [],
  "choices": []
}
```

### 5.3 Identify Flow Type

Match the flow name against the Flow Type → Test Approach table in [standards/test-scenarios.md §6](../standards/test-scenarios.md) to pick the initial scenario shape. Refine with flow-element extras (§4 of the same file).

---

## 6. Step 3: Test Scenario Planning

### 6.1 Check for Solution Design Document

**Action:** Ask user if Solution Design document is available.

**If available:** Extract test scenarios from "Use Case Scenarios for Testing" section.

**If not available:** Generate standard scenarios based on HTTP method.

### 6.2 Scenario catalog

Canonical scenarios, test-case names, counts per layer, and flow-triggered extras (Choice, Scatter-Gather, First-Successful, `foreach`/`parallel-foreach`, `ee:cache`, `target`, async, batch, VM, try-scope, error propagation) all live in **[../standards/test-scenarios.md](../standards/test-scenarios.md)**. Read it here — do not duplicate.

Summary of what to plan:
- **Minimum 3 scenarios per endpoint** (see test-scenarios.md §1).
- Apply **layer-specific naming** (test-scenarios.md §2).
- For each flow element listed in test-scenarios.md §4, add the extra scenarios it triggers.
- Match **external-system error shapes** (OData backends, empty-result envelopes, datetime `Z`) per test-scenarios.md §5.

---

## Phase 2A gate — Present the plan, wait for approval

Before generating any file, present the assembled plan (`SCENARIO_PLAN` + `MOCK_SOURCE` + fixture list + test-suite list) to the user via `AskUserQuestion`:

```
PLAN SUMMARY
============
Project:      {PROJECT_ROOT}
Layer:        {LAYER}
Flows:        {count} flows, {count} endpoints
Mock source:  {MOCK_SOURCE}   (infer / user-provided / postman-mcp-server / anypoint-exchange)

Per-endpoint scenarios (follows test-scenarios.md §1):
  - get-{entity}     → 3 scenarios (200 / 204 / 500)
  - post-{entity}    → 3 scenarios (201 / 400 / 500)
  - ...

Additional scenarios (from flow structure — test-scenarios.md §4):
  - <choice> in get-{entity}       → 3 branch tests
  - <foreach> in post-orders       → single-item fixture + iteration count tests
  - <ee:cache> on <backend-1>-request → verify-call atLeast="0"
  - ...

Fixtures to create: {N} files under src/test/resources/test-data/{entity}/
  - {source}-get-{entity}.json          (from {MOCK_SOURCE}, e.g. <backend-1> / <backend-2>)
  - {source}-get-{entity}-empty.json
  - {source}-post-{entity}.json
  - ...

Test suites to create: {M} files under src/test/munit/{entity}/
  - get-{entity}-test-suite.xml
  - post-{entity}-test-suite.xml
  - ...
```

Options for the user:
- **Approve** → proceed to Step 7 (Mock Data Generation) and Step 8 (MUnit Test Generation).
- **Modify** → loop back to the relevant Phase 1A step (endpoints, mock source, scenarios).
- **Cancel** → stop gracefully.

**Do not enter Phase 3A without explicit "Approve".** Store as `APPROVED_PLAN`.

---

## 7. Step 4: Mock Data Generation

**Action:** Create fixture files under `src/test/resources/` mirroring the suite structure. The agent **composes** fixtures based on the backend's real response shape.

- **Directory layout & naming conventions:** see [standards/naming-conventions.md](../standards/naming-conventions.md).
- **External-system response shapes** (OData `d.results`, OData `odata.error`, empty-result envelopes, DateTime `Z` stripping): see [standards/template-by-layer.md § SAPI — External-system response shapes](../standards/template-by-layer.md).
- **Where to get the real shape:** EAPI/PAPI → Anypoint Exchange consumed-API spec; SAPI → ask the user ([workflow.md Do #18](./workflow.md)).
- **`foreach` / `parallel-foreach`:** create a single-item fixture alongside the array fixture ([workflow.md Do #21](./workflow.md)).

**Do not infer fixture shape from DWL** — DWL shows how data is used, not how it's structured.

---

## 8. Step 5: MUnit Test Generation

**Action:** Generate the MUnit XML test suite file. Every concrete template, namespace block, pattern, and complete example lives in the KB — the workflow only tracks procedure.

### 8.1 File naming

Suite file: `{flow-name}-test-suite.xml` under `src/test/munit/{feature}/`. Test-case names follow [standards/test-scenarios.md § 1–§ 2](../standards/test-scenarios.md).

### 8.2 Where templates live

| Need | File |
|---|---|
| Required namespace declarations | [usage/allowed-components-list.md § Required MUnit Namespace Declarations](../usage/allowed-components-list.md) |
| Mandatory test structure skeleton | [usage/allowed-components-list.md § Mandatory Test Structure](../usage/allowed-components-list.md) |
| Mock patterns (HTTP, OS, flow-ref, SAP, errors) | [usage/allowed-components-list.md § Mocking Patterns](../usage/allowed-components-list.md) |
| Event setup (`<munit:set-event>`) with variables and attributes | [usage/allowed-components-list.md § Event Setup](../usage/allowed-components-list.md) |
| Assertion patterns (8 patterns) | [usage/allowed-components-list.md § Assertion Patterns](../usage/allowed-components-list.md) |
| Verification patterns (3 patterns) | [usage/allowed-components-list.md § Verification Patterns](../usage/allowed-components-list.md) |
| Layer-specific templates + complete SAPI/PAPI/EAPI test suites | [usage/template-sapi.md](../usage/template-sapi.md), [template-papi.md](../usage/template-papi.md), [template-eapi.md](../usage/template-eapi.md) |
| Cross-layer patterns (target-attr, APIKit router, foreach fixtures, error-handling, anti-patterns) | [usage/template-common.md](../usage/template-common.md) |
| Bearer-token flow-ref mock (SAPI OAuth2) | [usage/template-sapi.md § 1.0](../usage/template-sapi.md#10-bearer-token-flow-ref-mock-sapi-oauth2) |
| Methodology examples (choice, scatter-gather, async, batch, hooks) | [usage/official-guidelines.md](../usage/official-guidelines.md) |

### 8.3 Agent checklist during generation

- `encoding="UTF-8"` on **every** `<munit-tools:payload>` and `<munit-tools:variables>` in mock returns ([workflow.md Do #17](./workflow.md)).
- Every variable referenced by the flow's loggers (e.g., `correlationId`) is set in `<munit:set-event>` ([workflow.md Do #10](./workflow.md)).
- **Never** mock `<logger>` — zero `mock-when` and zero `verify-call` for it ([workflow.md Do #13](./workflow.md)).
- Call the MAIN flow via `<flow-ref>` (e.g., `get:\suppliers:<your-config-name>`), never the sub-flow directly — the main flow initializes variables via transforms before delegating.
- When generation is done, run the pre-flight checklist before declaring the tests ready ([workflow/pre-flight-validation.md](./pre-flight-validation.md)).

### 8.4 Handling existing test files

Before writing, check if the target test file exists:

- **File exists:** parse it, extract existing test names, then ask the user via `AskUserQuestion`: "Test file already exists with N tests. Append new scenarios or replace the entire file?"
  - **Append:** add only the new scenarios (those whose name is not already present).
  - **Replace:** overwrite the entire file.
- **File does not exist:** create it.

Never silently overwrite.

---


## 9. Step 6: Test Execution & Validation

### 9.1 Execute Tests

**Command:**
```bash
mvn clean test -Dencrypt.key=<YOUR_ENCRYPT_KEY> -Dmule.env=<YOUR_ENV>
```

**Run from:** Project root directory

### 9.2 Analyze Results

**Check for:**

1. **Compilation Errors:**
   - XML syntax errors
   - Missing namespace declarations
   - Invalid DataWeave expressions

2. **Test Failures:**
   - Mock configuration issues (wrong `doc:id`)
   - Assertion failures
   - Missing variables

3. **Coverage Report:**
   - Location: `target/munit-reports/coverage/`

### 9.3 Report Results

**Output Format:**

```
═══════════════════════════════════════════════════════════════
                    MUnit Test Execution Report
═══════════════════════════════════════════════════════════════

Test Suite: {flow-name}-test-suite.xml
Layer: {SAPI|PAPI|EAPI}

TESTS EXECUTED:
  ✓ {test-name-1} - PASSED
  ✓ {test-name-2} - PASSED
  ✗ {test-name-3} - FAILED: {error message}

SUMMARY:
  Total Tests: 3
  Passed: 2
  Failed: 1

CODE COVERAGE:
  Application Coverage: {XX}%
  Flow Coverage: {XX}%
  Status: {PASS|FAIL - Below threshold}

═══════════════════════════════════════════════════════════════
```

2. **Suggest additional tests:**
   - Uncovered error handlers
   - Uncovered choice branches
   - Uncovered processors
3. **Offer to generate additional test scenarios**

---

## 10. Layer-Specific Instructions

Layer-specific requirements, required mocks, required variables, checklists, and templates live in the knowledge base. From the workflow, the agent needs only to detect the layer (SAPI / PAPI / EAPI) and load the right file:

| Layer | Requirements (what to check) | Templates (what to write) |
|---|---|---|
| **SAPI** | [standards/template-by-layer.md § SAPI](../standards/template-by-layer.md) | [usage/template-sapi.md](../usage/template-sapi.md) (templates §1.0–§1.4 + Complete SAPI Example) |
| **PAPI** | [standards/template-by-layer.md § PAPI](../standards/template-by-layer.md) | [usage/template-papi.md](../usage/template-papi.md) (templates §2.1–§2.2 + Complete PAPI Example) |
| **EAPI** | [standards/template-by-layer.md § EAPI](../standards/template-by-layer.md) | [usage/template-eapi.md](../usage/template-eapi.md) (template §3.1 + Complete EAPI Example) |

Workflow-side considerations:

- **Calling endpoint flows directly (via APIkit-routed flow names):** set `vars.correlationId` in `<munit:set-event>`. Normally it's initialized by the main flow from the inbound `x-correlation-id` header before the APIkit router. See Do #24 above.
- **SAPI with OAuth2:** mock the bearer-token flow-ref. Example in [usage/template-sapi.md § 1.0](../usage/template-sapi.md#10-bearer-token-flow-ref-mock-sapi-oauth2).
- **EAPI loggers:** standard Mule `<logger>` calls are no-ops in tests — never mock them. Set any variable they reference (e.g., `correlationId`) in `<munit:set-event>`. See [standards/template-by-layer.md § EAPI](../standards/template-by-layer.md) "EAPI logger handling".
- **Cross-layer scenario counts, required variables, assertion focus:** see the Quick Reference tables in [standards/template-by-layer.md](../standards/template-by-layer.md).

---
---

## Reference Catalogs (moved out of this file)

Previously this file duplicated content from other KB files. Those sections have been removed; consult the canonical homes:

| Topic | Canonical location |
|---|---|
| Allowed MUnit components, processors to mock, matcher functions | [reference/allowed-components-list.md](../reference/allowed-components-list.md) + [usage/allowed-components-list.md](../usage/allowed-components-list.md) |
| Anti-patterns (compilation / runtime / logic errors) | [standards/template-by-layer.md](../standards/template-by-layer.md) §Anti-Patterns |
| Common errors and their solutions | [standards/common-errors.md](../standards/common-errors.md) |
| Troubleshooting guide | [standards/common-errors.md](../standards/common-errors.md) + [usage/official-guidelines.md](../usage/official-guidelines.md) examples |
| Quick-reference cross-layer tables (mock count, required variables, assertion focus) | [standards/template-by-layer.md](../standards/template-by-layer.md) §Quick Reference |
| Complete SAPI / PAPI / EAPI test examples | [usage/template-sapi.md](../usage/template-sapi.md), [usage/template-papi.md](../usage/template-papi.md), [usage/template-eapi.md](../usage/template-eapi.md) — each has a "Complete Working Example" section |

---

## Appendix B: Agent Execution Checklist

Use this checklist for each test generation:

```
□ Step 1: Project Analysis
  □ Detected API layer: _______
  □ Project root verified
  □ Test directories exist/created
  □ MUnit dependencies present

□ Step 2: Flow Analysis
  □ Flow XML parsed successfully
  □ Extracted doc:id values: _______
  □ Identified processors to mock: _______
  □ Flow type determined: _______

□ Step 3: Test Scenario Planning
  □ Solution Design available: YES / NO
  □ Scenarios planned (minimum 3):
    □ 1. _______________________
    □ 2. _______________________
    □ 3. _______________________

□ Step 4: Mock Data Generation
  □ Mock directory created
  □ Response JSON files created: _______
  □ Request JSON files created: _______
  □ DateTime queryParams use correct format (no 'Z' suffix for LocalDateTime)
  □ OData backend empty responses maintain nested table structure

□ Step 5: Test Generation
  □ Test file path: _______
  □ **ALL mock payloads include encoding="UTF-8"** (CRITICAL for CI/CD)
  □ All required namespaces declared:
    □ xmlns:munit
    □ xmlns:munit-tools
    □ xmlns:doc (for doc:name and doc:id attributes)
  □ Existing file handling: APPEND / REPLACE / NEW
  □ All mocks include correct doc:id
  □ Bearer token mock uses doc:id from general.xml sub-flow definition (NOT from flow-ref)
  □ Calling main flow (e.g., get:\suppliers:<your-config-name>), NOT sub-flow directly
  □ Every variable the flow's loggers reference is set (e.g., correlationId)
  □ All assertions use MunitTools:: prefix

□ Step 6: Execution & Validation
  □ Tests compiled successfully
  □ Tests executed: PASS / FAIL
  □ Coverage: _______%
  □ Coverage threshold met: YES / NO

□ Issues Flagged for Review:
  □ _______________________
  □ _______________________
```

---

---

# Mode B — Fix Failing Tests

## Absolute Rule: `src/main/` is OUT OF BOUNDS

The implementation under `src/main/` (flows, DataWeave scripts, configs, properties, global elements, XML) is the source of truth. **Never edit any file under `src/main/` to make a failing test pass.** All fixes must live in the test layer:

- `src/test/munit/` (test XML)
- `src/test/resources/` (fixtures, mocks, sample payloads, expected-result files)
- Test event variables, attributes, expected assertion values

If diagnosis reveals the implementation is genuinely wrong (buggy flow, bad DataWeave, wrong mapping), **STOP**. Report the finding with evidence to the user and wait for explicit instruction before any `src/main/` change. Never silently "fix" the implementation to make a test green.

### Things that are NEVER a legitimate Mode B fix

- Editing any file under `src/main/`.
- Adding a `mock-when` for `<logger>`.
- Deleting or weakening the assertion to silence a failure.
- Stubbing `expectedErrorType` on a test that should genuinely assert on a payload.
- Skipping the planning phase (Phase 2B).

## Mandatory required reading (fix mode)

Before diagnosing any failing test:

1. **this file — § Testing Rules** — especially the Common Errors table (maps observed errors to fixes).
2. **`standards/template-by-layer.md`** — anti-pattern reference + layer-specific requirements (helps spot what's missing in the failing test).
3. **`reference/official-guidelines.md`** — processor identification, mock contract rules, payload-overwrite & variable-timing fixes.
4. The failing test file itself, plus the flow under `src/main/` it targets (read-only).

## Mode B — Phase Map

| Phase | Steps | Output | Gate |
|---|---|---|---|
| **Phase 1B — Diagnosis** | F1 Collect failure evidence, F2 Classify failure, F3 Locate root cause | `FAILING_TEST`, `ERROR_TEXT`, `FAILURE_DIAGNOSIS`, `ROOT_CAUSE` | — |
| **Phase 2B — Fix Planning** | Build before/after proposal for every change; list only test-layer files | `PROPOSED_FIX_PLAN` | ⚠️ Wait for user approval via `AskUserQuestion` |
| **Phase 3B — Implementation** | F4 Apply minimal test-side fix, F5 Re-run, F6 Report | Fixed test suite/fixtures, passing test, change summary | — |

**Never enter Phase 3B without explicit "Approve" from the user.** If F3 concludes the failure is implementation-side (`src/main/` is wrong), STOP — report evidence to the user, skip Phase 2B/3B, and wait for explicit instruction.

## Fix workflow — steps

### Step F1. Collect failure evidence

Use `AskUserQuestion` to collect from the user:

- **Which MUnit to fix** (required): the failing test suite file path, or the test name within it, or both. Offer the tests the agent can discover under `src/test/munit/` as quick options; allow "Other" for free-text.
- **The error message** (optional): paste the exact error text + stack trace if available. If the user doesn't have it, run the test (after authorization) to capture it; never paraphrase or recall errors from memory.

Then read (read-only):

- The `src/main/` flow XML the test targets.
- The current mocks / fixtures / `<munit:set-event>` contents for that test.
- Any fixtures under `src/test/resources/` the test references.

Record:

- Test suite file path + failing test name.
- Exact error message and stack trace.
- The flow under test.

### Step F2. Classify the failure

Match the error message against the error-message table in [standards/common-errors.md section 1](../standards/common-errors.md) and the category table in [section 2](../standards/common-errors.md). If the symptom doesn't map to any category, re-read the failing test carefully before concluding the implementation is wrong.

### Step F3. Locate the true root cause

- Re-read the flow (read-only) to confirm what the implementation **does**.
- Compare against what the test **expects**.
- Determine whether the divergence is a test-side error (fix it) or an implementation-side error (stop and report — see the Absolute Rule above).

### Step F3.5 — Phase 2B gate: Present the fix plan, wait for approval

Before applying any change, present a before/after proposal to the user via `AskUserQuestion`:

```
FIX PLAN
========
Failing test:    {FAILING_TEST}
Error category:  {FAILURE_DIAGNOSIS.category}
Root cause:      {ROOT_CAUSE}

Proposed changes (ALL under src/test/):
  1. src/test/munit/.../xxx-test-suite.xml line 42
     Before: (no correlationId variable set)
     After:  <munit:variable key="correlationId" value='"test-correlation-id-12345"' mediaType="application/java"/>
     Why:    Do #10 — set every variable the flow's loggers reference

  2. src/test/resources/test-data/xxx/response.json
     Before: {"error": {"code": "BadRequest", ...}}
     After:  {"odata.error": {"code": "", "message": {...}}}
     Why:    Backend uses OData error format (template-by-layer.md § SAPI)

No files under src/main/ will be modified.
```

Options for the user:
- **Approve** → go to Step F4.
- **Modify** → loop back to F2 / F3 with the user's adjustments.
- **Cancel** → stop gracefully.

**Do not enter Step F4 without explicit "Approve".**

### Step F4. Apply the minimal test-side fix

Edit ONLY the test-layer files. Typical legitimate fixes:

- Update an expected payload (JSON) to match what the implementation actually produces.
- Adjust a mock's `then-return` so its payload / attributes / variables match the real contract.
- Fix `<munit:attributes>` (correct `queryParams`, strip timezone `Z` where DWL expects `LocalDateTime`).
- Fix `<munit:variables>` — set `correlationId` and any other domain-specific variables the flow reads.
- Fix `doc:id` on `<munit-tools:with-attribute>` to match the exact `doc:id` on the target processor in the flow.
- Add `encoding="UTF-8"` to every mock payload (CI/CD fix for cursor-closed errors).
- Change `verify-call times="1"` → `atLeast="0"` for `ee:cache`-wrapped requests.
- Replace `\"` with `&quot;` in XML attribute values.

### Step F5. Re-run the test

Ask the user before running tests. When authorized, run:

```bash
mvn clean test -Dencrypt.key=<YOUR_ENCRYPT_KEY> -Dmule.env=<YOUR_ENV>
```

- Green: report what changed and why.
- Still failing: return to Step F2 with the new error.

### Step F6. Report

Summarize what failed, what you changed, and why. List the files touched (all under `src/test/`). Confirm nothing under `src/main/` was modified.

## Fix-mode checklist

- [ ] Failure reproduced and error captured verbatim
- [ ] Error classified against the Common Errors table
- [ ] Flow (under `src/main/`) read and understood — but **not edited**
- [ ] Root cause determined (test-side vs implementation-side)
- [ ] If implementation-side: STOPPED and reported to user
- [ ] If test-side: minimal fix applied to `src/test/*` only
- [ ] Test re-run (after user authorization) and now green
- [ ] Change summary delivered; confirmed no `src/main/` edits

---

## Document Information

**Version:** 2.0.0
**Purpose:** AI agent instructions for MUnit test generation (Mode A) and failing-test repair (Mode B)
**Scope:** MuleSoft 4 Projects (SAPI, PAPI, EAPI)

