# Mule Core Components Reference

Attribute reference for Mule 4 core components (routers, scopes, validators, error handling). For XML examples, see [Component Examples](../usage/component.md).

---

## Choice Router

Conditional routing based on DataWeave expressions.

| Element | Description |
|---------|-------------|
| `<choice>` | Container for conditional routes |
| `<when expression="...">` | Route evaluated when expression is true |
| `<otherwise>` | Default route when no `when` matches |

### Rules

- ALL routes inside Choice routers must have **at least one processor** — an empty `<otherwise>` is not allowed
- If no action is needed in `<otherwise>`, add a logger explaining why
- Never set `httpStatus` to 4xx/5xx inside a Choice — use `raise-error` and let the error handler set the status

---

## Foreach

Iterates over a collection sequentially. Each iteration gets its own `payload` and `vars.counter`.

| Attribute | Description | Default |
|-----------|-------------|---------|
| `collection` | Expression returning the collection to iterate | `#[payload]` |
| `batchSize` | Number of items per batch | `1` |
| `rootMessageVariableName` | Variable name to access the root message | `rootMessage` |

### Available Variables Inside Loop

| Variable | Description |
|----------|-------------|
| `payload` | Current item in the iteration |
| `vars.counter` | Current iteration index (starts at 1) |
| `vars.rootMessage` | Original message before foreach started |

### Rules

- Wrap in `<try>` with `<on-error-continue>` when individual item failures should not stop the loop
- Log errors per item with `vars.counter` for traceability

---

## Parallel Foreach

Iterates over a collection in parallel. Same as Foreach but concurrent.

| Attribute | Description | Default |
|-----------|-------------|---------|
| `collection` | Expression returning the collection | `#[payload]` |
| `maxConcurrency` | Maximum parallel threads | Number of cores x 2 |
| `timeout` | Max time to wait for all threads | No timeout |
| `timeoutUnit` | Timeout unit | `MILLISECONDS` |

### Rules

- Always set `maxConcurrency` to prevent thread exhaustion (recommended: `5` for HTTP calls)
- Wrap HTTP calls in `<try>` with `<on-error-continue>` to prevent one failure from cancelling all threads
- Results are returned in the same order as the input collection

---

## Scatter-Gather

Executes multiple routes **in parallel** and aggregates results.

| Attribute | Description | Default |
|-----------|-------------|---------|
| `maxConcurrency` | Maximum parallel routes | All routes |
| `timeout` | Max time to wait for all routes | No timeout |
| `timeoutUnit` | Timeout unit | `MILLISECONDS` |

### Structure

| Element | Description |
|---------|-------------|
| `<scatter-gather>` | Container for parallel routes |
| `<route>` | Individual parallel execution branch |

### Result Access

After scatter-gather completes, payload is a map of route results:

| Expression | Description |
|------------|-------------|
| `payload.0.payload` | Result from first route |
| `payload.1.payload` | Result from second route |
| `payload.N.attributes` | Attributes from route N |

### Rules

- Use `target` on HTTP requests inside routes to store responses in variables
- Handle individual route errors with `<try>` inside each `<route>` if partial failure is acceptable

---

## Try Scope

Error handling boundary within a flow. Catches errors without propagating to the flow-level error handler.

| Element | Description |
|---------|-------------|
| `<try>` | Scope that wraps processors and catches errors |
| `<error-handler>` | Error handling block inside try |
| `<on-error-continue>` | Catch error, continue flow execution |
| `<on-error-propagate>` | Catch error, re-throw after handling |

### On-Error-Continue vs On-Error-Propagate

| Type | Flow continues? | Payload after | Use when |
|------|----------------|---------------|----------|
| `on-error-continue` | Yes | Set by the error handler | Individual item failure should not stop processing |
| `on-error-propagate` | No (re-throws) | Set by the error handler | Error must be handled AND propagated to caller |

### Attributes

| Attribute | Description | Example |
|-----------|-------------|---------|
| `type` | Error type(s) to catch | `HTTP:CONNECTIVITY`, `ANY` |
| `when` | Additional condition | `#[error.errorType.identifier == 'TIMEOUT']` |
| `enableNotifications` | Send error notification | `true` |
| `logException` | Log the exception | `true` |

### Error Mapping

Use `<error-mapping>` inside connectors to remap error types:

| Attribute | Description | Example |
|-----------|-------------|---------|
| `sourceType` | Original error type | `HTTP:NOT_FOUND` |
| `targetType` | Custom error type | `BUSINESS:RECORD_NOT_FOUND` |

---

## Async Scope

Executes processors asynchronously (fire-and-forget). The main flow continues immediately without waiting.

| Attribute | Description | Default |
|-----------|-------------|---------|
| `maxConcurrency` | Max concurrent async executions | No limit |

### Rules

- Variables set inside `<async>` are NOT visible outside it
- The main flow's payload is NOT affected by async processing
- Use for notifications, logging to external systems, or non-critical side effects
- Always add logging inside async — errors are silent otherwise

---

## Raise Error

Throws a custom error that can be caught by error handlers.

| Attribute | Description | Example |
|-----------|-------------|---------|
| `type` | Custom error type (NAMESPACE:IDENTIFIER) | `VALIDATION:DATE_RANGE` |
| `description` | Error message | `"Date range cannot exceed 90 days"` |

### Rules

- Use `raise-error` instead of setting `httpStatus` to 4xx/5xx in the main flow
- Let the error handler map the error type to the correct HTTP status
- Namespace must be custom (not a reserved Mule namespace like `HTTP` or `APIKIT`)

---

## Validation Module

**Namespace prefix:** `validation:`

Declarative validation that throws `VALIDATION:INVALID_*` errors on failure.

### Configuration

```xml
<validation:config name="Validation_Config" doc:name="Validation Config" doc:id="GENERATE-UUID"/>
```

> Multiple configs can exist per project (e.g., `PO_Validation_Config`, `Inventory_Config`) but a single `Validation_Config` is typical.

### Operations

| Operation | Description | Key Attributes |
|-----------|-------------|----------------|
| `validation:is-true` | Fails if expression is false | `expression`, `message`, `config-ref` |
| `validation:is-false` | Fails if expression is true | `expression`, `message`, `config-ref` |
| `validation:is-not-null` | Fails if value is null | `value`, `message`, `config-ref` |
| `validation:is-not-empty` | Fails if collection/string is empty | `value`, `message`, `config-ref` |
| `validation:matches-regex` | Fails if value doesn't match regex | `value`, `regex`, `message`, `config-ref` |

### Error Mapping with Validation

Use `<error-mapping>` to convert validation errors to business errors:

| Attribute | Description |
|-----------|-------------|
| `targetType` | Custom error type, e.g., `BUSINESS:BACKEND_INVENTORY_FAILED` |

---

## Set Variable / Set Payload / Remove Variable

### Set Variable

| Attribute | Description |
|-----------|-------------|
| `variableName` | Name of the variable |
| `value` | Expression or literal value |

### Remove Variable

| Attribute | Description |
|-----------|-------------|
| `variableName` | Name of the variable to remove |

---

## Flow Reference

Calls another flow or sub-flow.

| Attribute | Description |
|-----------|-------------|
| `name` | Name of the target flow/sub-flow |

> Dynamic flow references are supported: `name="#[vars.scope ++ '-main-flow']"`

---

## References

- [Component Examples](../usage/component.md) — real-world patterns with full XML
- [Connector Examples](../usage/connector.md) — connector-specific patterns
- [Error Handling Standards](../standards/error-handling.md)

---
Last updated: 2026-04-10
Owner: Integration Team
