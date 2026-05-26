# DataWeave Functions Reference

Quick-reference for approved DataWeave 2.0 functions and custom modules in this framework. For full transformation examples see [DataWeave Usage](../usage/dataweave.md). For folder structure & styling rules see [DataWeave Patterns](../standards/dataweave-patterns.md).

---

## Approved Module Imports

Always import explicitly — never rely on transitive imports:

| Module | Import Statement | When to Use |
|---|---|---|
| Core arrays | `import * from dw::core::Arrays` | `divideBy`, `partition`, `sumBy`, `countBy`, `indexOf` |
| Core objects | `import * from dw::core::Objects` | `entrySet`, `keySet`, `valueSet`, `mergeWith` |
| Core strings | `import * from dw::core::Strings` | `capitalize`, `camelize`, `pluralize`, `substringBefore`, `leftPad` |
| Core periods | `import * from dw::core::Periods` | `period`, `between`, date arithmetic |
| Util timer | `import * from dw::util::Timer` | Performance measurement (DEBUG only) |
| URL | `import * from dw::core::URL` | `encodeURIComponent`, `decodeURIComponent` |
| Crypto | `import * from dw::Crypto` | `SHA1`, `MD5`, `HMACBinary` (avoid in business logic) |
| Runtime | `import * from dw::Runtime` | `wait`, `try` (rarely needed) |
| Mule | `import * from dw::Mule` | `p()` for property access, `causedBy()` for errors |

---

## Core Functions

### Array Operations

| Function | Signature | Example |
|---|---|---|
| `map` | `Array<T> map ((T, Number) -> R)` | `[1,2,3] map ((v, i) -> v * 2)` |
| `filter` | `Array<T> filter ((T, Number) -> Boolean)` | `orders filter ((o) -> o.status == "ACTIVE")` |
| `reduce` | `Array<T> reduce ((T, R) -> R)` | `[1,2,3] reduce ((v, acc=0) -> acc + v)` |
| `groupBy` | `Array<T> groupBy ((T) -> K) → { (K): Array<T> }` | `orders groupBy $.customerId` |
| `pluck` | `{ (K): V } pluck ((V, K) -> R) → Array<R>` | `obj pluck $$` (gets all keys) |
| `distinctBy` | `Array<T> distinctBy ((T) -> K)` | `orders distinctBy $.id` |
| `orderBy` | `Array<T> orderBy ((T) -> Comparable)` | `orders orderBy $.createdAt` |
| `flatten` | `Array<Array<T>> → Array<T>` | `[[1,2],[3]] flatten` |
| `joinBy` | `Array<String> joinBy String` | `["a","b"] joinBy "-"` |
| `sumBy` | `Array<T> sumBy ((T) -> Number) → Number` | `lines sumBy $.amount` |
| `countBy` | `Array<T> countBy ((T) -> Boolean) → Number` | `orders countBy $.status == "OPEN"` |

### Object Operations

| Function | Signature | Example |
|---|---|---|
| `mapObject` | `{ (K): V } mapObject ((V, K) -> { (K2): V2 })` | `obj mapObject ((v, k) -> { (upper(k)): v })` |
| `filterObject` | `{ (K): V } filterObject ((V, K) -> Boolean)` | `obj filterObject ((v, k) -> v != null)` |
| `update` | `obj update path with newValue` | `payload update "status" with "DONE"` |
| `++` (merge) | `Object ++ Object → Object` | `vars.request ++ { extra: "x" }` |
| `entrySet` | `Object → Array<{ key, value }>` | (from `dw::core::Objects`) |

### String Operations

| Function | Signature | Example |
|---|---|---|
| `upper`, `lower` | `String → String` | `upper("abc") // "ABC"` |
| `trim` | `String → String` | `trim(" x ") // "x"` |
| `replace` | `String replace String with String` | `"a-b" replace "-" with "_"` |
| `splitBy` | `String splitBy String → Array<String>` | `"a,b" splitBy ","` |
| `contains` | `String contains String → Boolean` | `"abc" contains "b"` |
| `substring` | `String[from to]` | `"hello"[0 to 1] // "he"` |
| `sizeOf` | `String → Number` | `sizeOf("abc") // 3` |
| `leftPad` | `(from `Strings`) String leftPad (size, char)` | `"5" leftPad (3, "0") // "005"` |
| `camelize`, `capitalize` | (from `Strings`) | `camelize("hello world") // "helloWorld"` |

### Date / Time

Standard date/time format: ISO-8601 with timezone offset — `yyyy-MM-dd'T'HH:mm:ssXXX`

| Function / Pattern | Example |
|---|---|
| `now()` | Current `DateTime` |
| `today()` | Current `Date` |
| `|2026-04-13|` | Date literal |
| `|2026-04-13T10:30:00Z|` | DateTime literal |
| Format | `now() as String { format: "yyyy-MM-dd'T'HH:mm:ssXXX" }` |
| Parse | `"2026-04-13" as Date { format: "yyyy-MM-dd" }` |
| Period add | `now() + |P1D|` (1 day) |
| Period subtract | `now() - |PT30M|` (30 min) |
| Timezone shift | `now() >> "UTC"` |

**Default timezone**: `UTC`. Always set explicitly when formatting for downstream systems.

### Type Coercion

| Pattern | Use For |
|---|---|
| `value as String` | Force string conversion |
| `value as Number` | Force number conversion |
| `value as Date { format: "..." }` | Parse string to date |
| `value default fallback` | Null-safe fallback (preferred over `if/else`) |

---

## Mule Property Access

```dwl
%dw 2.0
import * from dw::Mule
output application/json
---
{
    serviceName: p('logger.serviceName') default 'unknown',
    timeout: p('http.read.timeout') as Number
}
```

- `Mule::p('key')` → reads runtime property
- Returns `String` — coerce to `Number` / `Boolean` as needed
- Use `default` for optional properties

---

## Custom Modules

Project-specific reusable DWL lives in `src/main/resources/modules/` (when present).

### `norm.dwl` — Normalization (acme-erp-sapi)

Strips leading/trailing whitespace from strings, recursively for arrays.

```dwl
fun norm(v) =
  v match {
    case is Null   -> null
    case is Array  -> v map trim((it default "") as String)
    case is String -> trim(v)
    else           -> v
  }
```

**Usage:**
```dwl
import * from modules::norm
---
payload mapObject ((v, k) -> { (k): norm(v) })
```

> **Note**: There is no shared cross-project DWL module library yet. When you create a reusable function, place it in `src/main/resources/modules/{moduleName}.dwl` and document it here.

---

## Common Patterns

### Null-Safe Field Access

```dwl
{
    name: payload.customer.name default "Unknown",
    email: payload.customer.contact.email default null
}
```

### Conditional Field Inclusion

Use `skipNullOn` directive — preferred over `if/else`:

```dwl
%dw 2.0
output application/json skipNullOn="everywhere"
---
{
    id: payload.id,
    name: payload.name,
    optionalField: if (payload.includeOptional) payload.optional else null
    // → omitted entirely when null
}
```

### Safe Array Indexing

```dwl
{
    firstError: (payload.errors default [])[0].message default "no error"
}
```

### Building from RAML Schemas

Always write the output object to match the RAML schema exactly — use the schema's field order and casing.

---

## Forbidden / Discouraged Patterns

| ❌ Don't | ✅ Do |
|---|---|
| `if (payload.x != null) payload.x else "default"` | `payload.x default "default"` |
| Multiple sequential `do { ... }` blocks | Single transformation expression with helper functions |
| Dynamic field access without null guard: `payload[vars.key].value` | `(payload[vars.key] default {}).value default null` |
| Using `as` on potentially-null values: `payload.x as String` | `(payload.x default "") as String` |
| Logging via `log()` in DWL | Use `<logger level="…">` outside the transform |
| Importing `dw::*` (wildcard top-level) | Import specific modules: `import * from dw::core::Strings` |
| Hardcoding env-specific values in DWL | Read via `Mule::p('property.key')` |
| Inlining large transformations in XML | Move to `.dwl` file under `src/main/resources/dwl/...` |

---

## When to Externalize a DWL File

Inline DWL is acceptable for **one-line** transforms (e.g., `<ee:set-variable>` with a single expression). Anything multi-line must live in its own `.dwl` file under `src/main/resources/dwl/` (see [DataWeave Patterns](../standards/dataweave-patterns.md) for folder rules).

---

## References

- [DataWeave Patterns (standards)](../standards/dataweave-patterns.md)
- [DataWeave Usage Examples](../usage/dataweave.md)
- [Logging Standards](../standards/logging.md) — standard Mule logger usage
- [DataWeave 2.0 Documentation](https://docs.mulesoft.com/dataweave/latest/)

---
Last updated: 2026-04-13
Owner: Integration Team
