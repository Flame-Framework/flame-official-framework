# Sections B.3 + B.4 — Resource Definition + ResourceType Parameters

> **Agent scope**: Root RAML file (resource definitions).

> **Common mistakes to avoid**:
> - Marking "resourceTypes used" as PASS without checking parameter names → verify names match common-data: `patchRequestObject`, `objectResponse`, etc.
> - Flagging `collectionType` for item endpoints → common-data only has `collectionType`, no `itemType`. It can be used for BOTH collections and items
> - Expecting `displayName`/`description` at resource level → these are method attributes (get, post, etc.), NOT resource attributes
> - Flagging grouping resources for missing `collectionType` → resources without HTTP methods are just namespace organizers, they don't need it

## B.3. Resource Definition Validation

### B.3.1. Resource Structure <Critical>
All resources that declare HTTP methods (get, post, put, patch, delete) MUST include:
- `type` - Using `lib.collectionType` with proper parameters

**Exception**: Resources that exist solely as parent/grouping paths to nest other resources underneath them (i.e., they have NO HTTP methods declared directly on them) do NOT need `lib.collectionType`. These resources serve only as namespace organizers.

**IMPORTANT**: `displayName` and `description` are attributes of **HTTP methods** (GET, POST, PATCH, etc.), NOT of resources/endpoints. Do NOT expect these at the resource level.

Example:
```raml
/materials:
  type:
    lib.collectionType:
      errorResponse: lib.errorResponse
      objectResponse: lib.get-material-response[]
      createObject: lib.material-request
      createResponse: lib.post-material-response
  get:
    displayName: Get Materials           # Belongs to the method
    description: Retrieve all materials  # Belongs to the method
  post:
    displayName: Create Material
    description: Create a new material
```

**Grouping Resource Example** (no `lib.collectionType` needed):
```raml
# CORRECT - /items has no HTTP methods, only nested resources
/items:
  /pumpings:
    type:
      lib.collectionType:
        errorResponse: lib.errorResponse
        createObject: lib.items-pumpings-request
        createResponse: lib.post-items-pumpings-response
    post:
      displayName: Post Items Pumpings data
      description: Insert ItemPumping data
  /products:
    type:
      lib.collectionType:
        errorResponse: lib.errorResponse
        createObject: lib.items-products-request
        createResponse: lib.post-items-products-response
    post:
      displayName: Post Items Products data
      description: Insert ItemProduct data
```

### B.3.2. HTTP Method Documentation <Medium>
Each HTTP method (get, post, patch, put, delete) MUST have:
- `displayName` - Short descriptive name for the operation
- `description` - What the operation does

**Note**: These attributes belong to the method level, NOT to the resource/endpoint level.

Example:
```raml
/materials:
  type:
    lib.collectionType:
      errorResponse: lib.errorResponse
      objectResponse: lib.get-material-response[]
  get:
    displayName: Get Materials
    description: Retrieve all materials from the system
  post:
    displayName: Create Material
    description: Create a new material in the system
```

### B.3.3. URI Parameters Documentation <Medium>
All URI parameters MUST be documented with:
- `type` - Data type
- `description` - Clear description
- `example` - Sample value

Example:
```raml
/{materialCode}:
  uriParameters:
    materialCode:
      type: string
      description: Unique material identifier
      example: "MAT001"
```

---

## B.4. ResourceType Parameter Mapping <Critical>

### B.4.1. Correct Parameter Names <Critical>
MUST use correct parameter names per common-data `default-collection.raml`:

| Parameter | HTTP Method | Purpose |
|-----------|-------------|---------|
| `errorResponse` | ALL | Error response type (lib.errorResponse) |
| `objectResponse` | GET | Response for retrieving data |
| `createObject` | POST | Request body for creating |
| `createResponse` | POST | Response after creation (typically ID response) |
| `putRequestObject` | PUT | Request body for full update |
| `patchRequestObject` | PATCH | Request body for partial update |

**Example**:
```raml
type:
  lib.collectionType:
    errorResponse: lib.errorResponse
    objectResponse: lib.get-material-response[]    # GET returns array
    createObject: lib.material-request             # POST request
    createResponse: lib.post-material-response     # POST response (ID type)
```

### B.4.2. Different Response Types <Medium>
- GET and POST MUST use DIFFERENT response types
- POST typically returns an ID response (e.g., `material-id-response`), NOT the same object as GET

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 5
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (B.3.1, B.3.2, B.3.3, B.4.1, B.4.2), identify the missing ones, go back and validate them, then update your counts before returning.**
