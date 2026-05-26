# DataWeave (Usage Examples)

> **Purpose:** DataWeave code examples — script structure, selectors, flow control, pattern matching, functions, imports, and common transformation patterns. For the language reference (types, operators, module function lists, format properties, configuration), see [reference/dataweave.md](../reference/dataweave.md).

---

## Script Structure

```dataweave
%dw 2.0
output application/json
---
payload
```

## String Interpolation

```dataweave
"Hello $(payload.name), your total is $(payload.amount)"
```

---

## Selectors

```dataweave
payload.name                   // single-value
payload.*item                  // multi-value — all matching values
payload..id                    // descendants
payload[(key)]                 // dynamic
payload[0]                     // index
payload[-1]                    // last element
payload[0 to 2]                // range
payload.user.@id               // XML attribute
payload.root.#                 // namespace
payload.name?                  // presence check → Boolean
payload.name!                  // assert present or throw
payload[?($.age > 18)]         // filter
payload.^mimeType              // metadata
```

### Mule runtime variables

```dataweave
payload
attributes
vars.myVariable
error
flow.name
```

---

## Update Operator

```dataweave
myObject update {
  case age at .age -> age + 1
  case name at .name if (name == "Ken") -> "Kenneth"
}
```

## Metadata Assignment

```dataweave
{name: "Ken"} <~ {class: "Person"}
```

---

## Flow Control

### Conditional

```dataweave
if (payload.age >= 18) "Adult"
else if (payload.age >= 13) "Teen"
else "Child"
```

### `do` scope

```dataweave
do {
  var name = "DataWeave"
  var version = 2.0
  ---
  "$(name) $(version)"
}
```

---

## Pattern Matching

### Expression patterns (with guards)

```dataweave
payload.number match {
  case num if num > 0 -> "positive"
  case num if num < 0 -> "negative"
  else -> "zero"
}
```

### Literal value patterns

```dataweave
payload.status match {
  case "active" -> true
  case "inactive" -> false
  else -> null
}
```

### Data type patterns

```dataweave
payload.value match {
  case is String -> "STRING"
  case is Number -> "NUMBER"
  case is Boolean -> "BOOLEAN"
  case is Array -> "ARRAY"
  case is Object -> "OBJECT"
  case is Null -> "NULL"
}
```

### Regex patterns

```dataweave
payload.phone match {
  case phone matches /\+(\d+)\s\((\d+)\)\s(\d+\-\d+)/ ->
    { country: phone[1], area: phone[2], number: phone[3] }
  else -> { error: "Invalid format" }
}
```

### Array deconstruction

```dataweave
[1, 2, 3, 4] match {
  case [head ~ tail] -> { first: head, rest: tail }
  case [] -> { empty: true }
}
```

### Object deconstruction

```dataweave
{a: 1, b: 2} match {
  case {headKey: headValue ~ tail} -> headValue
  case {} -> null
}
```

---

## Functions

### Definition with default args

```dataweave
%dw 2.0
output application/json

fun greet(name) = "Hello, $(name)!"
fun greetWithDefault(name = "World") = "Hello, $(name)!"
---
greet("DataWeave")
```

### Type constraints

```dataweave
fun toUser(id: Number, name: String): Object = {
  userId: id,
  userName: name
}
```

### Generics

```dataweave
fun toArray<T>(x: T): Array<T> = [x]

fun getName<T <: { name: String }>(x: T): String = x.name
```

### Overloading

```dataweave
fun process(a: String) = upper(a)
fun process(a: Number) = a * 2
fun process(a: Array) = sizeOf(a)
```

### Lambdas

```dataweave
var double = (n) -> n * 2
var add = (a, b) -> a + b
```

Used with higher-order functions:

```dataweave
payload map ((item, index) -> item.name)
payload filter ((item) -> item.active)
payload reduce ((item, acc) -> acc + item.value)
```

---

## Variables

```dataweave
%dw 2.0
output application/json

var greeting = "Hello"
var items = [1, 2, 3]
var transform = (x) -> x * 2
---
{
  message: greeting,
  doubled: items map transform
}
```

### Local variables in `do`

```dataweave
do {
  var localVar = "only here"
  ---
  localVar
}
```

---

## Imports

```dataweave
// Import entire module (requires ModuleName:: prefix)
import dw::core::Strings

// Import all from module (direct access)
import * from dw::core::Strings

// Import specific functions
import camelize, capitalize from dw::core::Strings

// Import with alias
import camelize as toCamel from dw::core::Strings
```

### `dw::Runtime` error handling

```dataweave
import try, orElse from dw::Runtime

try(() -> payload.value / 0) orElse "Division error"
```

### Custom module

`modules/MyUtils.dwl`:

```dataweave
%dw 2.0

fun double(n: Number): Number = n * 2
fun triple(n: Number): Number = n * 3
var PI = 3.14159
```

Usage:

```dataweave
import * from modules::MyUtils

double(5)  // 10
```

---

## Format-Specific Syntax

### JSON streaming output

```dataweave
output application/json streaming=true
```

### XML namespace declaration

```dataweave
%dw 2.0
ns ns0 http://example.com/schema
output application/xml
---
{
  ns0#root: {
    ns0#child: "value"
  }
}
```

### XML attribute syntax

```dataweave
{
  user @(id: "123", active: true): {
    name: "John"
  }
}
```

---

## Common Transformation Patterns

### JSON ↔ XML

```dataweave
// JSON → XML
%dw 2.0
output application/xml
---
{
  root: payload
}
```

```dataweave
// XML → JSON
%dw 2.0
output application/json
---
payload.root
```

### Map array elements

```dataweave
payload map ((item, index) -> {
  id: index,
  name: upper(item.name),
  value: item.amount * 1.1
})
```

### Filter and transform

```dataweave
payload
  filter ((item) -> item.status == "active")
  map ((item) -> item.name)
```

### Group by field

```dataweave
payload groupBy ((item) -> item.category)
```

### Flatten nested arrays

```dataweave
payload flatMap ((item) -> item.children)
```

### Reduce to single value

```dataweave
payload reduce ((item, acc = 0) -> acc + item.amount)
```

### Merge objects

```dataweave
payload.object1 ++ payload.object2
```

### Conditional fields

```dataweave
{
  name: payload.name,
  (email: payload.email) if payload.email?,
  (phone: payload.phone) if payload.phone?
}
```

### Dynamic keys

```dataweave
{
  (payload.keyName): payload.value
}
```

### Error handling with `try`

```dataweave
import try from dw::Runtime

try(() -> payload.value as Number) match {
  case result if result.success -> result.result
  else -> 0
}
```

### Date formatting

```dataweave
payload.date as String {format: "yyyy-MM-dd"}
```

### Null coalescing

```dataweave
payload.value default "N/A"
```
