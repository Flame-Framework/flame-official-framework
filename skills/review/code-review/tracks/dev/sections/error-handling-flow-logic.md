# Section B-I — Error Handling & Flow Logic

> **ALL-file search required**: Rule B-I.1 requires searching ALL XML files in `src/main/mule/` to ensure no error handling exists outside `error-handler.xml`, but only report issues if the violation is in a **modified file** or if the **modified file introduced** the violation.

## B-I.1. Error Handling Location <Blocking>
- Validate that ALL error handling implementations are located in `error-handler.xml`
- No error handling should be scattered in other XML files

## B-I.2. HTTP Error Status Codes Must Only Be Set Inside Error Handlers <Blocking>
- Any `httpStatus` variable set to a 4xx or 5xx value MUST only exist inside an error handler (`on-error-propagate` or `on-error-continue`), never in the main flow
- If a flow needs to return an HTTP error response, it MUST use a `raise-error` component (e.g., `<your-namespace>:BAD_REQUEST`, `<your-namespace>:VALIDATION`, `<your-namespace>:NOT_FOUND`) so that the error handler is responsible for building the error response and setting the httpStatus
- **How to detect**: Look for `ee:transform` or `set-variable` components that assign 4xx or 5xx values to `httpStatus` outside of an error handler (e.g., inside a `choice` branch in the main flow). Any occurrence is a violation
- **Why this matters**: Setting an error httpStatus outside of an error handler creates a divergence between the Mule runtime state (success) and the HTTP response (error). This causes error handlers to be skipped, logs to record the request as successful, and monitoring/metrics tools to undercount errors — distorting dashboards, alerting, and SLA reporting

## B-I.3. Unreachable or Impossible Logic Expressions <Blocking>
- Validate that conditional expressions (Choice routers, When conditions, etc.) can actually evaluate to all possible outcomes
- **How to detect**: Analyze the data type and structure produced by preceding processors and verify that the condition can logically be true or false

> **CRITICAL**: Before flagging an `isEmpty(payload)` check as a violation, you **MUST read and analyze the actual DWL file** referenced in the preceding Transform Message. Do NOT assume the output structure based on the file name alone.

- **When `isEmpty(payload)` IS a violation**:
  - When the DWL file **always** returns an object with a fixed structure (e.g., `{ quotations: payload map ... }`)
  - In this case, even with empty data, the result is `{ quotations: [] }` - an object that is never empty
  - `isEmpty({ quotations: [] })` returns `false`, so the condition will never be true

- **When `isEmpty(payload)` is VALID** (not a violation):
  - When the DWL file has **conditional logic** that can return an empty object `{}`
  - Example: `if (isEmpty(someCondition)) {} else { ... full object ... }`
  - In this case, `isEmpty({})` returns `true`, so the condition CAN be true
  - **Always check for `if/else` statements in the DWL that explicitly return `{}`**

- **Example of violation** (DWL always returns object with structure):
  ```dataweave
  // transform.dwl - ALWAYS returns { quotations: ... }
  %dw 2.0
  output application/json
  ---
  {
    quotations: payload map (...)  // Always returns object, never empty
  }
  ```
  ```xml
  <ee:transform doc:name="Final Transform">
    <ee:set-payload resource="dwl/transform.dwl" />
  </ee:transform>
  <choice>
    <when expression="#[isEmpty(payload)]">  <!-- VIOLATION - payload is always an object -->
      <set-variable variableName="httpStatus" value="204" />
    </when>
  </choice>
  ```

- **Example of VALID usage** (DWL can return empty object):
  ```dataweave
  // transform-by-id.dwl - CAN return {} when no data
  %dw 2.0
  output application/json
  ---
  if (isEmpty(uniqueIds))
      {}  // Returns empty object when no data
  else
      { ... full object structure ... }
  ```
  ```xml
  <ee:transform doc:name="Final Transform">
    <ee:set-payload resource="dwl/transform-by-id.dwl" />
  </ee:transform>
  <choice>
    <when expression="#[isEmpty(payload)]">  <!-- VALID - DWL can return {} -->
      <set-variable variableName="httpStatus" value="204" />
    </when>
  </choice>
  ```

- **Why this matters**: Unreachable code paths indicate logic errors that will cause unexpected behavior at runtime (e.g., never returning 204 status when there's no data)

## B-I.4. Empty Choice Routes <Attention Point — 2 pts>
- Validate that ALL routes inside a Choice router have at least one processor implemented
- If any `<when>` or `<otherwise>` branch is empty (contains no processors), it is an attention point
- **How to detect**:
  1. Look for `<when>` or `<otherwise>` elements inside `<choice>` that have no child processor elements
  2. Look for `<choice>` routers that have **no `<otherwise>` branch at all** — a missing `<otherwise>` is an implicit empty route (when no `<when>` condition matches, nothing executes)
- **Example 1 — explicitly empty branch**:
  ```xml
  <choice>
    <when expression="#[vars.action == 'create']">
      <flow-ref name="create-record" />
    </when>
    <when expression="#[vars.action == 'update']">
      <!-- VIOLATION - empty route, no processors -->
    </when>
    <otherwise>
      <flow-ref name="default-handler" />
    </otherwise>
  </choice>
  ```
- **Example 2 — missing `<otherwise>` (implicit empty route)**:
  ```xml
  <choice>
    <when expression="#[isEmpty(payload)]">
      <logger level="INFO" doc:name="Logger" />
    </when>
    <!-- VIOLATION - no <otherwise> branch, implicit empty route when condition is false -->
  </choice>
  ```
- **Why this matters**: Empty routes indicate incomplete implementation — the condition is evaluated but nothing happens, which typically means the logic was either forgotten or left as a placeholder that was never completed

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 4
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (B-I.1, B-I.2, B-I.3, B-I.4), identify the missing ones, go back and validate them, then update your counts before returning.**
