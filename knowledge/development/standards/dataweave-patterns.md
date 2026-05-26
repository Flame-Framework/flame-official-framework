# DataWeave Patterns

## Overview

This document provides DataWeave rules and best practices for data transformations in MuleSoft APIs. All data transformations should be implemented using DataWeave 2.0. For code examples, see [DataWeave Cookbook](../usage/dataweave.md).

## File Organization

### External DWL Files

Store all DataWeave files in the `dwl/` folder under `src/main/resources/`, organized by purpose:

```
src/main/resources/
└── dwl/
    ├── payloads/                     # Files used in ee:set-payload
    │   ├── alive_check_payload.dwl
    │   ├── {entity}/                 # Group by entity when multiple files
    │   │   ├── entityMapping.dwl
    │   │   └── entityResponse.dwl
    │   └── reprocess/                # Reprocess-specific payloads
    ├── vars/                         # Files used in ee:set-variable
    │   ├── {entity}/                 # Group by entity context
    │   ├── scheduler/                # Scheduler-specific variables
    │   ├── httpStatus/               # HTTP status code variables
    │   ├── reprocess/                # Reprocess-specific variables
    │   └── scopes/                   # Scope-specific variables
    └── error/                        # Error handling DWL files
        ├── message/                  # API error response payloads
        ├── statusCodes/              # HTTP status code mappings
        └── reprocess/                # ObjectStore error handling
```

### Folder Convention

| Folder | Usage | Determined By |
|--------|-------|---------------|
| `payloads/` | Files used in `ee:set-payload` | Referenced in `<ee:set-payload resource="..."/>` |
| `vars/` | Files used in `ee:set-variable` | Referenced in `<ee:set-variable resource="..."/>` |
| `error/message/` | API error response payloads | Error response body transformations |
| `error/statusCodes/` | HTTP status code mappings | Status code set-variable in error handlers |
| `error/reprocess/` | ObjectStore error handling | Reprocess/retry error patterns |

### Naming & Organization Rules

- **Root folder**: Always `dwl/` (not `dw/`)
- **Subfolder naming**: `vars/` (not `variables/`), `payloads/` (not `payload/`)
- **camelCase** for subfolder names (e.g., `orderCosts/`, not `order-costs/`)
- **Group related files** in subfolders when they share a context (scheduler, orders, reprocess, etc.)
- **Naming conflicts**: Resolve with subfolders (e.g., `vars/scheduler/beginDateTime.dwl` vs `vars/orders/beginDateTime.dwl`)

### Common Misplacements

| Misplacement | Correct Location |
|-------------|-----------------|
| `set-payload` DWL files in `vars/` | Move to `payloads/` |
| `set-variable` DWL files in `payloads/` | Move to `vars/` |
| Error-related DWL in root `error/` | Move to `error/message/`, `error/statusCodes/`, or `error/reprocess/` |
| Mixing `vars/` and `variables/` | Standardize on `vars/` |
| DWL files in `dwl/` root | Move to appropriate subfolder |

### Using External DWL Files

```xml
<ee:transform>
    <ee:message>
        <ee:set-payload resource="dwl/payloads/orders/orderMapping.dwl"/>
    </ee:message>
</ee:transform>
```

## Language Rules & Constraints

DataWeave 2.0 is a **purely functional, expression-oriented** language. Every transformation is a single expression that evaluates to a value. Understanding what DataWeave is NOT is critical to writing correct code.

### Core Principles

1. **Declarative, not imperative** — describe the output shape, don't write step-by-step procedures
2. **Immutable** — no variable reassignment; use `var` in the header for computed values
3. **Expression-based** — everything returns a value; there are no "statements"
4. **No side effects** — DataWeave cannot call external services, log, or mutate state

### Prohibited Constructs (Common Hallucinations)

The following patterns are **frequently hallucinated** by AI but are either invalid or strongly discouraged in DataWeave. **Never generate these.**

#### `do` blocks — PROHIBITED

`do` blocks exist in DataWeave but are **almost never needed** in MuleSoft integration context. They add unnecessary complexity and produce imperative-looking code. Use `var` declarations in the header instead.

```dataweave
// WRONG — do block for local variables
%dw 2.0
output application/json
---
payload map (item) -> do {
    var fullName = item.firstName ++ " " ++ item.lastName
    var isActive = item.status == "active"
    ---
    {
        name: fullName,
        active: isActive
    }
}

// CORRECT — inline expressions
%dw 2.0
output application/json
---
payload map (item) -> {
    name: item.firstName ++ " " ++ item.lastName,
    active: item.status == "active"
}

// CORRECT — use header var when a value is reused across the body
%dw 2.0
output application/json
var items = payload.value default []
---
{
    data: items map (item) -> {
        name: item.firstName ++ " " ++ item.lastName
    },
    total: sizeOf(items)
}
```

#### Unnecessary `fun` definitions — AVOID

Only use `fun` for genuinely reusable logic (e.g., shared error response format). Do not create functions for one-time operations.

```dataweave
// WRONG — function for a one-time mapping
%dw 2.0
output application/json

fun transformItem(item) = {
    id: item.item_id,
    name: item.item_name
}

fun buildResponse(items) = {
    data: items map (item) -> transformItem(item),
    count: sizeOf(items)
}
---
buildResponse(payload.value default [])

// CORRECT — direct expression
%dw 2.0
output application/json
var items = payload.value default []
---
{
    data: items map (item) -> {
        id: item.item_id,
        name: item.item_name
    },
    count: sizeOf(items)
}
```

#### Imperative/Java-style patterns — INVALID

These constructs do **NOT** exist in DataWeave:

```dataweave
// WRONG                              // CORRECT
payload.toString()                    // payload as String
payload.length()                      // sizeOf(payload)
items.stream().filter()               // items filter (condition)
for (item in payload) {}              // payload map (item) -> ...
try { ... } catch { ... }            // Error handling at Mule flow level, not in DWL
var x = 1; x = 2;                    // Variables are immutable — no reassignment
payload.get("key")                    // payload."key" or payload.key
if (x) { return y }                  // if (x) y else z (expression, not statement)
```

#### Invented/Non-existent functions — INVALID

Do not invent functions. Only use functions from:
- DataWeave core (`map`, `filter`, `reduce`, `flatMap`, `groupBy`, `distinctBy`, `orderBy`, `sizeOf`, `isEmpty`, `upper`, `lower`, `trim`, `now`, `sum`, `avg`, `min`, `max`, etc.)
- Explicit imports (`dw::core::Strings`, `dw::core::Arrays`, `dw::core::Dates`, `dw::core::Objects`, etc.)

```dataweave
// WRONG — these functions do NOT exist   // CORRECT equivalents
payload.collect()                         // payload map (item) -> ...
payload.stream()                          // payload map / filter / reduce
payload.forEach()                         // payload map (item) -> ...
payload.toList()                          // payload (already an Array)
payload.entries()                         // payload pluck ((v, k) -> ...)
String.format(...)                        // string interpolation: "Hello $(name)"
Arrays.sort(payload)                      // payload orderBy $.field
payload.split(",").toArray()              // payload splitBy "," (already returns Array)
```

#### Overly complex patterns — SIMPLIFY

```dataweave
// WRONG — unnecessary match/case for simple conditions
payload.status match {
    case "active" -> true
    case "inactive" -> false
    else -> false
}

// CORRECT — simple comparison
payload.status == "active"

// WRONG — recursive function for what reduce/sum does
fun sumAmounts(items, acc = 0) =
    if (isEmpty(items)) acc
    else sumAmounts(items[1 to -1], acc + items[0].amount)

// CORRECT — use sum or reduce
sum(payload.amount)
// or
payload reduce ((item, acc = 0) -> acc + item.amount)
```

### Valid DataWeave Operators Reference

| Operation | Operator | Example |
|-----------|----------|---------|
| Transform each | `map` | `payload map (item) -> { ... }` |
| Filter | `filter` | `payload filter ($.active == true)` |
| Accumulate | `reduce` | `payload reduce ((item, acc) -> ...)` |
| Flatten + map | `flatMap` | `payload flatMap ($.items)` |
| Group | `groupBy` | `payload groupBy $.category` |
| Deduplicate | `distinctBy` | `payload distinctBy $.id` |
| Sort | `orderBy` | `payload orderBy $.name` |
| Object keys/values | `pluck` | `payload pluck ((v, k) -> ...)` |
| Concatenate | `++` | `obj1 ++ obj2` or `str1 ++ str2` |
| Null coalesce | `default` | `payload.name default "N/A"` |
| Null check | `?` / `??` | `payload.field?` / `a ?? b` |
| Conditional field | `if` | `(field: val) if condition` |
| Type coercion | `as` | `payload.id as String` |
| Size | `sizeOf()` | `sizeOf(payload)` |
| Check empty | `isEmpty()` | `isEmpty(payload.name)` |

## Best Practices

### Do's

- Use external `.dwl` files for reusable transformations
- Leverage null-safe operators (`?`, `??`, `default`)
- Use meaningful variable names in lambdas
- Add comments for complex transformations
- Use typed parameters where possible
- **Extract `payload.value` to a variable** for reusability and readability
- **Add `default []`** to handle null/empty responses gracefully
- **Keep transformations declarative** — describe the output shape directly
- **Use `var` in the header** for values that are reused in the body
- **Prefer inline expressions** over `fun` for one-time logic

### Don'ts

- Don't use complex DataWeave in Set Payload (use Transform Message)
- Don't hardcode values that should be configurable
- Don't ignore null/empty checks
- Don't create deeply nested inline transformations
- Don't process large datasets without pagination
- **Don't use inline CDATA in Transform Message** — use external `.dwl` files. Trivial one-line expressions are exempt
- **Don't hardcode empty strings or static values** where a field mapping should exist
- **Don't modify shared DWL files** without verifying all usages remain compatible
- **Don't leave unused DWL files** — every `.dwl` must be referenced by at least one consumer
- **Don't use `do` blocks** — use `var` in the header or inline expressions
- **Don't create `fun` definitions** for one-time operations — write the expression directly
- **Don't write imperative/procedural code** — no loops, no mutation, no step-by-step procedures
- **Don't invent functions** — only use documented DataWeave core functions and explicit imports
- **Don't use Java/JavaScript syntax** — no `.toString()`, `.length()`, `.get()`, `for`, `try/catch`

## Error Response Format

All APIs must return errors as `{error: {code, message}}`. For SAPI, include error type: `error.type ++ " - " ++ backend error`.

## Checklist

- [ ] Transformations use DataWeave 2.0
- [ ] DWL files in correct `dwl/` subfolder (payloads, vars, error)
- [ ] Null safety handled (`default []` for arrays)
- [ ] Response data extracted to variables for reusability
- [ ] Date formats consistent
- [ ] Error response format: `{error: {code, message}}`
- [ ] Complex transformations documented
- [ ] Performance considered for large datasets
- [ ] No unused DWL files
- [ ] No shared DWL modifications without compatibility check

## References

- [DataWeave Cookbook](../usage/dataweave.md) — 15 code patterns with examples
- [DataWeave 2.0 Documentation](https://docs.mulesoft.com/dataweave/2.4/)
- [Main Development Guide](./api-development-best-practices.md)

---
Last updated: 2026-04-09
Owner: Integration Team
