# Section A-III — Secure Config & Code Quality <Blocking>

> **Agent scope**: All project files (Section A-III is NOT limited to modified files).

All of the following must be present and correctly structured:

## A.5.5. Secure Config Local
- Must include a local-environment secure-config file (e.g., `secure-config-local.yaml`) in the properties folder

## A.5.6. Secure Config Dev
- Must include a development-environment secure-config file (e.g., `secure-config-dev.yaml`) in the properties folder

## A.5.7. Secure Config Prod
- Must include a production-environment secure-config file (e.g., `secure-config-prod.yaml`) in the properties folder

## A.7. POM Properties Validation
- Must NOT have any unused entries inside the `<properties>` tag
- **How to detect**: For EACH property defined in `<properties>`, search for `${property.name}` in both the `pom.xml` and all property `.yaml` files under `src/main/resources/properties/`. If a property has **zero references** across these files, it is unused.
- **Do NOT assume** a property is used by the parent POM, CI/CD pipelines, or external tools — only count explicit `${...}` references found within the project's own `pom.xml` and property files

## A.8. Unused DWL Files <Blocking>
- Validate that ALL `.dwl` files in the project are referenced by at least one consumer
- **How to detect**:
  1. List all `.dwl` files under `src/main/resources/` (typically in `dwl/` or similar folders)
  2. For each `.dwl` file, search for references in:
     - **XML files** (`src/main/mule/**/*.xml`): look for `resource="dwl/..."` attributes in `<ee:set-payload>`, `<ee:set-variable>`, `<ee:set-attributes>` that match the DWL file path
     - **Other DWL files**: look for `readUrl("classpath://dwl/...")` or similar import patterns that reference the file
  3. If a `.dwl` file has **zero references** across XML and DWL files, it is unused
- **Why this matters**: Unused DWL files add dead code to the project, increase maintenance burden, and can cause confusion about which transformations are active. They may also indicate incomplete refactoring where a transformation was replaced but the old file was not removed

## A.9. MUnit External Call Mock Coverage <Blocking>
- For each MUnit test suite, validate that ALL external calls (HTTP requests, SAP connector calls, database calls, or any other outbound connector operation) present in the flow under test are covered by a corresponding `mock:when` processor in the MUnit test
- **How to detect**:
  1. Identify the flow being tested (from `munit:test` attribute referencing the flow)
  2. Find all external call processors in that flow (e.g., `http:request`, `sap:function-*`, `sap:execute-synchronous-remote-function-call`, `db:*`, `flow-ref` to sub-flows containing external calls)
  3. Verify that each external call has a matching `munit-tools:mock-when` with the correct `processor` attribute
- **Why this matters**: Missing mocks cause MUnit tests to attempt real external calls, leading to test failures, timeouts, and potentially unintended data modifications in external systems

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 6
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (A.5.5, A.5.6, A.5.7, A.7, A.8, A.9), identify the missing ones, go back and validate them, then update your counts before returning.**
