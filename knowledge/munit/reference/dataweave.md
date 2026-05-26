# DataWeave Context (Reference)

> **Purpose:** DataWeave language reference — types, operators, selectors, modules, formats, memory configuration, best practices, and version compatibility. For DataWeave code examples and transformation patterns, see [usage/dataweave.md](../usage/dataweave.md).

## Overview

DataWeave is MuleSoft's functional programming language for data transformation within Mule applications. It is the primary expression language for the Mule runtime engine, enabling transformation between formats, data structure manipulation, and component configuration.

**Current Version:** DataWeave 2.10 (bundled with Mule 4.10).

---

## 1. Language Fundamentals

### Script structure

A DataWeave script has two parts separated by `---`:
- **Header:** directives (`%dw 2.0`), output format, imports, variable declarations, function definitions.
- **Body:** the expression that generates the result.

### Functional programming principles

| Concept | Description |
|---------|-------------|
| Pure Functions | Identical input → identical output, no side effects |
| Immutability | Variables cannot be reassigned after definition |
| First-Class Functions | Functions can be stored, passed, and returned |
| Higher-Order Functions | Functions that accept or return functions |
| Function Composition | Chain functions — one's output is another's input |
| Lazy Evaluation | Expressions evaluated only when needed |

### No traditional loops

No `for` / `while`. Use functional alternatives:
- `map` — apply transformation to each element.
- `filter` — remove specific elements.
- `reduce` — collapse array into a single value.

---

## 2. Data Types

### Simple types

| Type | Description | Example |
|------|-------------|---------|
| String | Text in double, single, or backtick quotes | `"hello"`, `'world'`, `` `text` `` |
| Number | Unified integers and floating-point | `42`, `3.14` |
| Boolean | `true` / `false` | |
| Null | Absence of value | `null` |
| Regex | Regular expression | `/\d+/` |

### Composite types

| Type | Syntax |
|------|--------|
| Array | `[1, 2, 3]` |
| Object | `{name: "John", age: 30}` |

### Date and time (ISO-8601)

| Type | Format | Example |
|------|--------|---------|
| Date | `\|uuuu-MM-dd\|` | `\|2024-01-15\|` |
| DateTime | Date + Time + TimeZone | `\|2024-01-15T10:30:00Z\|` |
| LocalDateTime | DateTime without timezone | `\|2024-01-15T10:30:00\|` |
| Time | `\|HH:mm:ss.SSS\|` | `\|10:30:00\|` |
| LocalTime | Time without timezone | `\|10:30:00\|` |
| TimeZone | GMT offset | `\|+05:00\|` |
| Period | ISO-8601 duration | `\|P1Y2M3D\|` (1y 2mo 3d) |

**Date selectors:** `.year`, `.month`, `.day`, `.hour`, `.minutes`, `.seconds`, `.milliseconds`, `.quarter`, `.dayOfWeek`, `.dayOfYear`.

### Special types

| Type | Description |
|------|-------------|
| CData | XML character data block (inherits from String) |
| Iterator | Java-based iterator (consumed after single use) |
| TryResult | Result object with `success`, `result`, and `error` fields |
| URI | Complex type with URL component fields |

---

## 3. Selectors

Selectors traverse objects and arrays to retrieve values.

| Selector | Syntax | Returns |
|----------|--------|---------|
| Single-value | `.keyName` | First matching key's value |
| Multi-value | `.*keyName` | Array of all matching values |
| Descendants | `..keyName` | Array of all descendant matches |
| Dynamic | `[(expression)]` | Value based on evaluated expression |
| Index | `[<index>]` | Value at array position (`-1` for last) |
| Range | `[<start> to <end>]` | Array slice |
| Attribute | `@keyName` | XML attribute value |
| Namespace | `keyName.#` | Namespace string |
| Key Present | `keyName?` | Boolean presence check |
| Assert Present | `keyName!` | Value or exception if absent |
| Filter | `[?(expression)]` | Conditionally filtered results |
| Metadata | `.^mimeType` | Payload metadata |

### Mule runtime variables

- `payload` — message content
- `attributes` — metadata attributes
- `vars.<name>` — Mule variables
- `error` — error objects
- `flow.name` — current flow name

---

## 4. Operators

### Mathematical

| Operator | Description |
|----------|-------------|
| `+` | Addition; array append; object merge |
| `-` | Subtraction; array/object removal |
| `*` | Multiplication |
| `/` | Division |

### Equality & relational

| Operator | Description |
|----------|-------------|
| `==` | Strict equality |
| `~=` | Coercing equality (attempts type conversion) |
| `<`, `>`, `<=`, `>=` | Magnitude comparisons |

### Logical

| Operator | Description | Note |
|----------|-------------|------|
| `and` | Logical AND | Both conditions must be true |
| `or` | Logical OR | At least one condition true |
| `not` | Logical negation | Requires space: `not (true)` |
| `!` | Logical negation | No space needed: `!true` |

**Precedence gotcha:** `not true or true` = `false` (applies to entire expression). `!true or true` = `true` (applies only to first value).

### Array

| Operator | Description | Example |
|----------|-------------|---------|
| `>>` | Prepend | `1 >> [2, 3]` → `[1, 2, 3]` |
| `<<` | Append | `[1, 2] << 3` → `[1, 2, 3]` |
| `+` | Append (array on left) | `[1] + 2` → `[1, 2]` |
| `-` | Remove | `[1, 2, 3] - 2` → `[1, 3]` |

### Update operator (Mule 4.3+)

Simplifies field updates without reconstructing the object. Features:
- Value selector: `.field`
- Index selector: `.items[0]`
- Attribute selector: `.user.@name`
- Conditional updates: `case x at .field if(condition)`
- Upsert with `!` — creates missing fields
- Sugar: `$` as default variable

### Metadata assignment operator (Mule 4.5+)

`<~` assigns metadata without type coercion (e.g., `{name: "Ken"} <~ {class: "Person"}`).

---

## 5–8. Flow Control, Pattern Matching, Functions, Variables

Summarized here; see [usage file](../usage/dataweave.md) for code examples.

### Flow control

- `if … else …` — every `if` requires a matching `else`; else-if chains allowed.
- `do { … --- … }` — scoped block with local declarations.

### Pattern matching

`match` is DataWeave's switch-like construct. Pattern forms:
- **Expression** with guards: `case n if n > 0 -> …`
- **Literal value**: `case "active" -> true`
- **Data type**: `case is String -> …`
- **Regex**: `case phone matches /…/ -> …`
- **Array deconstruction**: `case [head ~ tail] -> …`
- **Object deconstruction**: `case {k: v ~ tail} -> …`

### Functions

- Definition with `fun`: `fun greet(name) = …`.
- Default args: `fun f(x = "default") = …`.
- Type constraints: `fun f(id: Number, name: String): Object = …`.
- Generics: `fun toArray<T>(x: T): Array<T> = [x]`; bounded: `fun f<T <: {name: String}>(x: T) = x.name`.
- Overloading by arg type.
- Lambdas: `(x) -> x * 2`; used in `map`, `filter`, `reduce`.

### Variables

- Declared with `var` in header or `do` block.
- Immutable — no reassignment.
- Eager evaluation.
- Functions (`fun`) are syntactic sugar for lambda variables.
- Global (header) vs local (inside `do`).

---

## 9. Modules and Imports

Import forms:
- `import dw::core::Strings` — module-qualified (`Strings::camelize`)
- `import * from dw::core::Strings` — direct access
- `import camelize, capitalize from dw::core::Strings` — specific functions
- `import camelize as toCamel from dw::core::Strings` — aliased

### `dw::Core` (auto-imported)

**Strings:** `lower`, `upper`, `trim`, `startsWith`, `endsWith`, `splitBy`, `joinBy`, `replace`, `match`, `matches`, `scan`.

**Numeric:** `abs`, `ceil`, `floor`, `round`, `pow`, `sqrt`, `mod`, `avg`, `sum`, `max`, `min`, `isInteger`, `isDecimal`, `isEven`, `isOdd`, `random`, `randomInt`.

**Arrays/collections:** `map`, `filter`, `reduce`, `flatMap`, `flatten`, `zip`, `unzip`, `groupBy`, `orderBy`, `distinctBy`, `indexOf`, `find`, `sizeOf`, `isEmpty`, `pluck`, `maxBy`, `minBy`.

**Objects:** `keysOf`, `valuesOf`, `namesOf`, `entriesOf`, `mapObject`, `filterObject`.

**Type / data handling:** `typeOf`, `read`, `write`, `readUrl`, `now`, `uuid`, `log`.

### `dw::core::Strings`

| Function | Description |
|----------|-------------|
| `camelize` | Convert to camelCase |
| `capitalize` | Capitalize first letter of each word |
| `dasherize` | Replace spaces/underscores with hyphens |
| `underscore` | Replace with underscores |
| `leftPad`, `rightPad` | Pad strings |
| `pluralize`, `singularize` | Pluralization |
| `reverse` | Reverse string |
| `substring`, `substringAfter`, `substringBefore` | Extract substrings |
| `isAlpha`, `isNumeric`, `isAlphanumeric` | Validation |
| `levenshteinDistance`, `hammingDistance` | String distance |

### `dw::core::Arrays`

| Function | Description |
|----------|-------------|
| `countBy` | Count elements matching condition |
| `divideBy` | Split into sub-arrays |
| `drop`, `dropWhile` | Remove from start |
| `take`, `takeWhile` | Select from start |
| `every`, `some` | Boolean tests on all/any |
| `firstWith` | First element matching condition |
| `indexOf`, `indexWhere` | Find positions |
| `join`, `leftJoin`, `outerJoin` | Join by ID |
| `partition` | Split by condition |
| `slice`, `splitAt`, `splitWhere` | Slicing |
| `sumBy` | Sum with transformation |

### `dw::core::Objects`

| Function | Description |
|----------|-------------|
| `divideBy` | Split into sub-objects |
| `entrySet` | Array of key-value pairs |
| `keySet`, `nameSet` | Array of keys |
| `valueSet` | Array of values |
| `mergeWith` | Merge objects |
| `everyEntry`, `someEntry` | Boolean tests |
| `takeWhile` | Select entries by condition |

### `dw::Runtime` (error handling)

| Function | Description |
|----------|-------------|
| `try` | Evaluate with error capture |
| `fail` | Throw exception |
| `failIf` | Conditional exception |
| `orElse` | Default on failure |
| `orElseTry` | Chain try attempts |

### `dw::core::Dates`

| Function | Description |
|----------|-------------|
| `date`, `dateTime`, `time` | Create date/time values |
| `localDateTime`, `localTime` | Create local date/time |
| `today`, `tomorrow`, `yesterday` | Relative dates |
| `atBeginningOfDay`, `atBeginningOfHour` | Truncate time |
| `atBeginningOfMonth`, `atBeginningOfWeek`, `atBeginningOfYear` | Truncate date |

### `dw::core::URL`

| Function | Description |
|----------|-------------|
| `encodeURI`, `decodeURI` | Full URI encoding/decoding |
| `encodeURIComponent`, `decodeURIComponent` | Component encoding/decoding |
| `parseURI` | Parse URL into components |
| `compose` | Build URLs with interpolation |

### `dw::Crypto`

| Function | Description |
|----------|-------------|
| `MD5` | MD5 hash (hex) |
| `SHA1` | SHA1 hash (hex) |
| `hashWith` | Hash with specified algorithm |
| `HMACBinary`, `HMACWith` | HMAC operations |

### `dw::core::Binaries`

| Function | Description |
|----------|-------------|
| `toBase64`, `fromBase64` | Base64 encoding/decoding |
| `toHex`, `fromHex` | Hexadecimal encoding/decoding |
| `concatWith` | Concatenate binary values |
| `readLinesWith`, `writeLinesWith` | Line-based operations |

### Custom modules

- Place `.dwl` files in `src/main/resources/modules/`.
- Modules have no body expression — only declarations.
- Import: `import * from modules::MyUtils`.

---

## 10. Supported Data Formats

### Overview

| MIME Type | ID | Format |
|-----------|-----|--------|
| `application/json` | `json` | JSON |
| `application/xml` | `xml` | XML |
| `application/csv` | `csv` | CSV |
| `application/yaml` | `yaml` | YAML |
| `application/java` | `java` | Java Objects |
| `application/flatfile` | `flatfile` | Flat File, COBOL Copybook |
| `application/xlsx` | `excel` | Excel |
| `application/avro` | `avro` | Avro |
| `application/protobuf` | `protobuf` | Protocol Buffers |
| `text/plain` | `text` | Plain Text |
| `application/octet-stream` | `binary` | Binary |
| `application/x-ndjson` | `ndjson` | Newline Delimited JSON |
| `application/x-www-form-urlencoded` | `urlencoded` | URL Encoded |
| `multipart/form-data` | `multipart` | Multipart |
| `text/x-java-properties` | `properties` | Java Properties |

### Reading strategies

| Strategy | Description | Formats |
|----------|-------------|---------|
| In-Memory | Loads entire document | All |
| Indexed | Disk for large files (>1.5 MB) | CSV, JSON, XML |
| Streaming | Sequential access to partitions | CSV, JSON, Excel, XML |

### JSON

**Writer properties:**

| Property | Default | Description |
|----------|---------|-------------|
| `indent` | `true` | Pretty-print |
| `skipNullOn` | `null` | Skip nulls: `arrays`, `objects`, `everywhere` |
| `duplicateKeyAsArray` | `false` | Convert duplicate keys to arrays |
| `encoding` | `UTF-8` | Output encoding |

### XML

**Reader properties:**

| Property | Default | Description |
|----------|---------|-------------|
| `streaming` | `false` | Enable streaming |
| `nullValueOn` | `blank` | Treat empty/blank as null |
| `supportDtd` | `false` | Enable DTD (security risk) |
| `externalEntities` | `false` | Block XXE attacks |

**Writer properties:**

| Property | Default | Description |
|----------|---------|-------------|
| `indent` | `true` | Pretty-print |
| `writeDeclaration` | `true` | Include XML header |
| `inlineCloseOn` | `empty` | Self-closing tag behavior |
| `writeNilOnNull` | `false` | Add `nil` attribute for nulls |

### CSV

**Reader properties:**

| Property | Default | Description |
|----------|---------|-------------|
| `header` | `true` | First row is header |
| `separator` | `,` | Field delimiter |
| `quote` | `"` | Quote character |
| `escape` | `\` | Escape character |
| `streaming` | `false` | Enable streaming |

**Writer properties:**

| Property | Default | Description |
|----------|---------|-------------|
| `separator` | `,` | Field delimiter |
| `quoteValues` | `false` | Quote all values |
| `quoteHeader` | `false` | Quote header values |
| `lineSeparator` | `\n` | Line break |

---

## 11. Memory Management

### Buffer files

DataWeave creates temporary files when data exceeds memory thresholds.

| Buffer | Purpose | Trigger |
|--------|---------|---------|
| Output | Stores transformation results | > 1.5 MB output |
| Input | Stores input data | > 1.5 MB input |
| Index | Maintains index for fast access | Indexed parsing |

### Configuration properties

| Property | Default | Description |
|----------|---------|-------------|
| `com.mulesoft.dw.buffersize` | 8,192 | In-memory buffer size (bytes) |
| `com.mulesoft.dw.max_memory_allocation` | 1,572,864 | Max memory before disk |
| `com.mulesoft.dw.directbuffer.disable` | `false` | Use heap vs off-heap memory |

---

## 13. Best Practices

1. Use streaming for large files to avoid memory issues.
2. Prefer pure functions for predictability.
3. Leverage pattern matching for complex conditional logic.
4. Use type constraints for better error detection.
5. Organize reusable code in custom modules.
6. Handle errors explicitly with `try` and `orElse`.
7. Use meaningful variable names.
8. Avoid nested `map`s when `flatMap` suffices.
9. Use `default` for null safety.
10. Document functions with AsciiDoc comments.

---

## 14. Version Compatibility

| Mule | DataWeave |
|------|-----------|
| 4.10 | 2.10 |
| 4.9 | 2.9 |
| 4.8 | 2.8 |
| 4.7 | 2.7 |
| 4.6 | 2.6 |
| 4.5 | 2.5 |
| 4.4 | 2.4 |
| 4.3 | 2.3 |
| 3.x | 1.x |

---

## References

- [DataWeave Documentation](https://docs.mulesoft.com/dataweave/latest/)
- [DataWeave Playground](https://developer.mulesoft.com/learn/dataweave)
- [DataWeave Cookbook](https://docs.mulesoft.com/dataweave/latest/dataweave-cookbook)
