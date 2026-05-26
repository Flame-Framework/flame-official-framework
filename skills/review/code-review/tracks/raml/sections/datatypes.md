# Section B.5 — DataType Validation

> **Agent scope**: All dataType files in `dataTypes/requests/` and `dataTypes/responses/`.

## B.5.1. Required Property Fields <Critical>
Every property in a datatype MUST have:
- `type` - Data type
- `example` - Sample value at property level

## B.5.2. Required Property Fields <Low>

**Recommended fields** (suggest adding if missing):
- `description` - Clear description of the field
- `required` - Explicit true/false (not implicit)

**WRONG Example**:
```raml
# WRONG - missing type and example (Critical)
properties:
  materialCode:
    description: Unique material identifier
```

**ACCEPTABLE Example** (but suggest improvements):
```raml
# ACCEPTABLE - has type and example, but missing description and required
# Suggest adding: description and required fields
properties:
  materialCode:
    type: string
    example: "MAT001"
```

## B.5.3. Server-Generated Fields <Medium>
Server-generated fields (id, createdAt, updatedAt) should be marked as `required: false`

Example:
```raml
properties:
  id:
    type: string
    description: System-generated unique identifier
    required: false    # Server generates this
    example: "MAT001"
  createdAt:
    type: format.dateTimeFormat_aaaa-MM-ddThhmmssZ
    description: Record creation timestamp
    required: false    # Server generates this
    example: "2024-01-15T10:30:00Z"
```

## B.5.4. Date/Time Fields Format Patterns <Critical>
All date and time fields in dataTypes MUST use the standardized format patterns from common-data.

**MUST import format patterns** in dataType files containing date/time fields:
```raml
#%RAML 1.0 DataType

uses:
  format: ../exchange_modules/<raml.common-data.group-id>/<raml.common-data.asset-id>/<raml.common-data.version>/dataTypes/commons/patterns/format-pattern.raml

type: object
properties:
  # ...
```

**Available format types**:
| Format Type | Pattern | Example |
|-------------|---------|---------|
| `format.dateFormat_aaaa-MM-dd` | yyyy-MM-dd | `"2023-12-31"` |
| `format.dateFormat_ddmmaaaa` | dd/MM/yyyy | `"29/02/2024"` |
| `format.dateTimeFormat_aaaa-MM-ddThhmmssZ` | ISO 8601 UTC | `"2023-12-31T23:59:59Z"` |
| `format.dateTimeFormat_aaaa-MM-ddThhmmss` | ISO 8601 | `"2023-12-31T23:59:59"` |
| `format.timeFormat_hhmmss` | HH:mm:ss | `"23:59:59"` |

**WRONG Example**:
```raml
# WRONG - using plain string for dates
properties:
  validFrom:
    type: string                    # WRONG!
    description: Valid from date
    example: "15-01-2021"           # Non-standard format
  orderDate:
    type: datetime                  # WRONG! Use format patterns instead
    example: "2024-01-15T10:30:00Z"
```

**CORRECT Example**:
```raml
# CORRECT - using format patterns
uses:
  format: ../exchange_modules/<raml.common-data.group-id>/<raml.common-data.asset-id>/<raml.common-data.version>/dataTypes/commons/patterns/format-pattern.raml

type: object
properties:
  validFrom:
    type: format.dateFormat_aaaa-MM-dd
    description: Valid from date
    required: true
    example: "2021-01-15"
  createdAt:
    type: format.dateTimeFormat_aaaa-MM-ddThhmmssZ
    description: Record creation timestamp
    required: false
    example: "2024-01-15T10:30:00Z"
```

**Why This Matters**:
- Format patterns include regex validation
- Ensures consistency across all APIs
- Standard formats reduce integration issues

## B.5.5. DataType File Structure <Critical>
DataType files MUST include:
- `#%RAML 1.0 DataType` header
- `displayName` (optional but recommended)
- `description` (optional but recommended)
- `type: object` (when defining object types)
- `properties` section with all fields documented

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 5
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (B.5.1, B.5.2, B.5.3, B.5.4, B.5.5), identify the missing ones, go back and validate them, then update your counts before returning.**
