# MuleSoft API Design Best Practices

## Overview

This document outlines best practices for designing APIs using RAML in MuleSoft's Anypoint Platform. Following these guidelines ensures consistent, maintainable, and reusable API specifications across the team.

---

## FIRST STEPS: Before Creating or Modifying an API

**ALWAYS check Anypoint Exchange first before starting any API work.**

### Workflow

1. **Search Exchange** - Use the Anypoint Exchange search tools to check if an API already exists for your use case
2. **If API exists** → Download the existing specification and add/modify only what's needed
3. **If API doesn't exist** → Create a new API following the naming conventions and patterns in this document

### Why This Matters

- Avoids duplicate APIs in the organization
- Ensures you're building on existing work rather than recreating from scratch
- Maintains consistency across API versions
- Reduces review cycles by following established patterns

### How to Check Exchange

```
# Using MCP tools:
1. Search for assets: anypoint_search_assets with search term (e.g., "customer", "sap")
2. Get asset details: anypoint_get_asset with groupId and assetId
3. Download project: Download the asset ZIP from Exchange to modify locally
```

---

## MANDATORY: Common-Data Fragment

**ALL new RAML specifications MUST use the `common-data` fragment as a dependency.**

See [common-data-fragment.md](./common-data-fragment.md) for complete documentation.

### Quick Reference

```raml
#%RAML 1.0
title: <org>-example-sapi
description: System API for Example integration
version: 1.0
baseUri: <org>-example-sapi/api/

uses:
  lib: library/library.raml

# Apply standard security from library
securedBy: lib.client_credentials

# Use resourceTypes and traits from library
/resources:
  type:
    lib.collectionType:
      errorResponse: lib.errorResponse
      createObject: lib.resource-request
      createResponse: lib.post-resource-response
  get:
    is: [lib.dateRangeFilter]
```

### Exchange Dependency

Add to `exchange.json`:
```json
{
  "dependencies": [
    {
      "groupId": "<ANYPOINT_ORG_ID>",
      "assetId": "common-data",
      "version": "<COMMON_DATA_VERSION>"
    }
  ]
}
```

### exchange.json Requirements

#### Root File Naming Convention

**The root RAML file MUST follow the same API naming convention** (see [API/Project Naming](#apiproject-naming-mandatory)) with `.raml` extension:

| API Type | Root File Name |
|----------|----------------|
| **System API** | `<org>-{system}-sapi.raml` |
| **Process API** | `<org>-{business-process}-papi.raml` |
| **Experience API** | `<org>-{channel/purpose}-eapi.raml` |

**CRITICAL - exchange.json alignment:**
- The `main` field in `exchange.json` **MUST match the actual root file name**
- If they don't match, **publishing to Exchange will fail**

**CRITICAL - When updating existing APIs:**
- **NEVER rename the existing root file**
- **NEVER create a new root file**
- **Keep the original root file name** exactly as it is in Exchange
- Only modify the content of the existing root file

#### Required for Publishing (MINIMUM)
These fields are needed for the publish tool, but can be passed as parameters:
```json
{
  "main": "<org>-example-sapi.raml",
  "name": "your-api-name",
  "classifier": "raml",
  "dependencies": [...]
}
```

#### Recommended for Completeness (Best Practice)
These fields improve tooling support (Design Center, IDE integrations):
```json
{
  "main": "<org>-example-sapi.raml",
  "name": "<org>-example-sapi",
  "assetId": "<org>-example-sapi",
  "groupId": "<ANYPOINT_ORG_ID>",
  "organizationId": "<ANYPOINT_ORG_ID>",
  "version": "1.0.0",
  "apiVersion": "v1",
  "classifier": "raml",
  "descriptorVersion": "1.0.0",
  "originalFormatVersion": "1.0",
  "dependencies": [...]
}
```

**Note:** When publishing via MCP tools (`anypoint_publish_asset_project`), fields like `assetId`, `version`, `apiVersion` are passed as tool parameters - they do NOT need to be in exchange.json for publishing to succeed. However, having them in exchange.json helps with:
- Design Center synchronization
- Local development tooling
- IDE integrations (Anypoint Code Builder)

---

## Guidelines

### Design-First Approach

- Always design APIs before implementation using RAML specifications
- Use Anypoint Design Center to create and validate specifications
- Enable the mocking service to test your design live in the browser before development starts
- This allows consumers to test early in the development cycle using stubs/mocks

### Naming Conventions

#### API/Project Naming (MANDATORY)

All APIs MUST follow this naming pattern based on the API-Led Connectivity layer:

| API Type | Pattern | Example |
|----------|---------|---------|
| **System API** | `<org>-{system}-sapi` | `<org>-sap-sapi`, `<org>-salesforce-sapi` |
| **Process API** | `<org>-{business-process}-papi` | `<org>-order-management-papi` |
| **Experience API** | `<org>-{channel/purpose}-eapi` | `<org>-mobile-eapi`, `<org>-portal-eapi` |

Where:
- `{system}` = the backend system being connected (e.g., sap, salesforce, oracle, dynamics)
- `{business-process}` = the business process being orchestrated (e.g., order-management, customer-onboarding)
- `{channel/purpose}` = the channel or purpose of the experience API (e.g., mobile, web, portal)

#### Resource Naming

- Use **lowercase with hyphens** for resource names (e.g., `/users`, `/order-items`)
- Use **plural nouns** for resource names: `/orders`, `/users`
- Resource names should be **nouns representing entities**, not actions
- Avoid verbs in resource names: Use `/orders/{orderId}` instead of `/getOrder/{orderId}`

### API-Led Connectivity

- Structure APIs following the three-layer architecture:
  - **System APIs**: Connect to backend systems (SAP, databases, legacy systems)
  - **Process APIs**: Orchestrate and transform data between systems
  - **Experience APIs**: Expose data tailored to specific channels (web, mobile)
- This approach maximizes reuse and governance

### Versioning

#### API Version (Base URI)
- Document changes between versions in the API documentation
- Maintain a changelog or release notes to help developers understand migrations

#### Asset Version (Semantic Versioning)

Follow **Semantic Versioning (SemVer)** for Exchange asset versions: `MAJOR.MINOR.PATCH`

| Change Type | Version Bump | Example | When to Use |
|-------------|--------------|---------|-------------|
| **MAJOR** | X.0.0 | 1.0.0 → 2.0.0 | Breaking changes: removing endpoints, changing response structure, removing fields |
| **MINOR** | X.Y.0 | 1.0.0 → 1.1.0 | New features: adding endpoints, adding new optional fields, adding new types |
| **PATCH** | X.Y.Z | 1.0.1 → 1.0.2 | Bug fixes: fixing examples, correcting descriptions, typos |

**Examples:**
- Adding a new `/customers` endpoint → **MINOR** bump (1.0.0 → 1.1.0)
- Adding optional `contractDate` field to existing type → **MINOR** bump
- Removing a field from response → **MAJOR** bump (breaking change)
- Fixing typo in description → **PATCH** bump (1.0.0 → 1.0.1)
- Changing required field to optional → **MINOR** bump
- Changing optional field to required → **MAJOR** bump (breaking change)

#### Publishing to Exchange

When publishing a new version:
1. **Always increment the version** according to SemVer rules above
2. **Publish as stable (published)** - new versions should be the latest stable by default
3. **Update exchange.json with the new version before publishing** - The `version` field in `exchange.json` MUST match the version being published, otherwise you'll get: `"version should be X instead of Y"` error
4. **Keep apiVersion consistent** - For version `1.x.x` assets, `apiVersion` must be `"1.0"`. Changing `apiVersion` requires a MAJOR version bump (e.g., to `2.0.0`)
5. **Never overwrite existing versions** - always create a new version

**Common Publishing Errors:**

| Error | Cause | Fix |
|-------|-------|-----|
| `version should be "1.14.0" instead of "1.13.5"` | `exchange.json` has old version | Update `version` field in `exchange.json` to match new version |
| `Assets with version 1.x.x should have API version 1.0` | `apiVersion` mismatch | Use `apiVersion: "1.0"` for 1.x.x versions, or bump to 2.0.0 if changing API version |

### RAML Fragments & Modularization

- Split your RAML into multiple files using `!include` for better organization
- Import fragments locally before publishing to Anypoint Exchange
- Reuse data types as declarations
- Use inheritance only when necessary
- Employ short unions rather than verbose type definitions
- Maintain simple traits and avoid combining traits with resource type properties
- Keep total project size below 500 KB for optimal parsing

### RAML Library Pattern (MANDATORY)

**Always create a `library.raml` file to centralize datatypes, resourceTypes, and traits, keeping the root `api.raml` file clean.**

This approach:
- Prevents polluting the root file with type definitions
- Improves maintainability and readability
- Promotes reusability across the specification
- Makes it easier to navigate the API structure
- **Ensures consistent success/error responses across all endpoints**
- **Avoids inline queryParams definitions**

#### Recommended Project Structure

```
<org>-{name}-{sapi|papi|eapi}/
├── <org>-{name}-{sapi|papi|eapi}.raml  # Root file (named after API)
├── exchange.json                        # main field MUST match root file name
├── library/
│   └── library.raml            # Centralizes datatypes and traits (inside library folder)
├── dataTypes/                  # camelCase "dataTypes" (NOT "datatypes" or "types")
│   ├── requests/               # Request body datatypes
│   │   ├── entity-one.raml
│   │   └── entity-two.raml
│   └── responses/              # Response body datatypes
│       ├── get-entity-one.raml
│       └── post-entity-one.raml
└── exchange_modules/           # Dependencies from Exchange (auto-managed)
    └── {orgId}/
        └── common-data/
```

**IMPORTANT folder naming:**
- Use `dataTypes/` (camelCase) NOT `datatypes/` or `types/`
- Use `library/` folder to contain `library.raml`
- Separate `requests/` and `responses/` subfolders inside dataTypes

#### Library File Example (COMPLETE PATTERN)

**File location:** `library/library.raml`

```raml
#%RAML 1.0 Library

securitySchemes:
  client_credentials: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/securitySchemas/client-credentials.raml

types:
  errorResponse: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/dataTypes/error-response-data-type.raml
  generic-request: !include ../dataTypes/requests/generic-request.raml
  generic-response: !include ../dataTypes/responses/generic-response.raml

resourceTypes:
  collectionType: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/collections/default-collection.raml
```

**Key Conventions (from template):**
- **Security scheme**: Use `client_credentials` (underscore, not camelCase)
- **Type names**: Use **kebab-case** for type names: `generic-request`, `get-material-response`
- **Paths**: Use `../dataTypes/` (camelCase folder name)
- **No `usage:` field** required (optional)

**Adding custom traits** (when needed):
```raml
# NOTE: Traits can ONLY contain queryParameters, headers, responses - NOT uriParameters
traits:
  dateRangeFilter:
    queryParameters:
      fromDate:
        type: date-only
        required: false
        description: Filter records from this date
        example: "2024-01-01"
      toDate:
        type: date-only
        required: false
        description: Filter records until this date
        example: "2024-12-31"
```

#### Merging Duplicate DataTypes (DRY Principle)

**When POST and PATCH operations use the same data structure, merge them into a single datatype.**

**Problem - Duplicate datatypes:**
```
dataTypes/requests/
├── post-clients.raml      # Has externalId field
└── patch-clients.raml     # Identical but without externalId
```

**Solution - Single datatype with optional method-specific fields:**
```raml
#%RAML 1.0 DataType
# dataTypes/requests/clients.raml

displayName: Clients
description: Represents client data for POST and PATCH operations
type: object
properties:
  externalId:
    type: string
    description: External identifier (required for POST, not used in PATCH as it's in URI)
    required: false  # Optional because PATCH doesn't need it
    example: "435900"
  name:
    type: string
    description: Client name
    required: true
    example: "Acme Corp"
  # ... other fields
```

**Then in library.raml:**
```raml
types:
  # BEFORE (duplicate):
  # post-clients-request: !include ../dataTypes/requests/post-clients.raml
  # patch-clients-request: !include ../dataTypes/requests/patch-clients.raml

  # AFTER (merged):
  clients-request: !include ../dataTypes/requests/clients.raml
```

**And in api.raml:**
```raml
/clients:
  type:
    lib.collectionType:
      createObject: lib.clients-request       # Same type
      createResponse: lib.post-clients-response

  /{externalId}:
    type:
      lib.collectionType:
        patchRequestObject: lib.clients-request  # Same type
        objectResponse: lib.get-clients-response
```

**When to merge:**
- POST and PATCH have identical or nearly identical structure
- Only difference is fields used in URI (like `externalId` in PATCH)
- Fields can be made optional with clear documentation

**When NOT to merge:**
- POST and PATCH have significantly different required fields
- Different validation rules per operation
- Business logic requires distinct types

#### Using ResourceTypes and Traits in api.raml

```raml
#%RAML 1.0
title: <org>-example-sapi
description: System API for Example integration
version: 1.0
baseUri: <org>-example-sapi/api/
protocols:
  - HTTPS

uses:
  lib: library/library.raml

securedBy: lib.client_credentials

/resources:
  displayName: Resources
  description: Resource collection
  type:
    lib.collectionType:
      errorResponse: lib.errorResponse
      createObject: lib.resource-request
      createResponse: lib.post-resource-response

  post:
    displayName: Create Resource
    description: Create a new resource

  /{resourceId}:
    uriParameters:
      resourceId:
        type: string
        description: Unique resource identifier
        example: "RES-001"
    type:
      lib.collectionType:
        errorResponse: lib.errorResponse
        objectResponse: lib.get-resource-response
        patchRequestObject: lib.resource-request

    get:
      displayName: Get Resource
      description: Retrieve a specific resource by ID

    patch:
      displayName: Update Resource
      description: Update a resource
```

**Key Benefits of this pattern:**
- ✅ Responses (200, 201, 400, 404, 500) are defined ONCE in resourceTypes
- ✅ QueryParams are defined ONCE in traits and reused across endpoints
- ✅ The `api.raml` is clean, readable, and focused on endpoint structure
- ✅ All error responses consistently use `common.error-response`
- ✅ Adding new endpoints follows the same pattern automatically

#### CRITICAL: ResourceType Parameter Mapping by HTTP Method

**Each HTTP method MUST use a DIFFERENT response type.** Never use the same type for GET and POST responses.

| Parameter | HTTP Method | Purpose |
|-----------|-------------|---------|
| `objectResponse` | GET | Response for retrieving data |
| `createObject` | POST | Request body for creating |
| `createResponse` | POST | Response after creation (typically an ID response) |
| `putRequestObject` | PUT | Request body for full update |
| `patchRequestObject` | PATCH | Request body for partial update |

**Note:** These parameter names come from `common-data/collections/default-collection.raml`. The resourceType returns 204 No Content for PUT/PATCH, so no separate response type is needed.

**Example - CORRECT usage:**
```raml
/companies:
  type:
    lib.collectionType:
      objectResponse: lib.Company[]        # GET returns array of companies
      createObject: lib.Company            # POST receives a company
      createResponse: lib.CompanyIdResponse # POST returns ID response (DIFFERENT from GET!)
```

**Example - INCORRECT usage (same response for GET and POST):**
```raml
/companies:
  type:
    lib.collectionType:
      objectResponse: lib.Company[]
      createObject: lib.Company
      createResponse: lib.Company          # WRONG! POST should return CompanyIdResponse
```

#### MANDATORY: Add errorResponse Trait to All Endpoints

All endpoints MUST include the `lib.errorResponse` trait for consistent error handling:

```raml
/companies:
  get:
    is: [common.pageable-trait, common.correlation-id-trait, lib.errorResponse]
  post:
    is: [common.content-type-required-trait, common.correlation-id-trait, lib.errorResponse]

  /{companyId}:
    get:
      is: [common.correlation-id-trait, lib.errorResponse]
    put:
      is: [common.content-type-required-trait, common.correlation-id-trait, lib.errorResponse]
    delete:
      is: [common.correlation-id-trait, lib.errorResponse]
```

### Data Types

- Define data types in separate files for reusability
- Use clear, descriptive type names
- Include field descriptions
- Specify required vs optional fields explicitly
- **MANDATORY: Include examples at PROPERTY LEVEL in datatype files** (this removes the need for example includes in api.raml)

#### Inline Examples in DataTypes (MANDATORY)

Add `example` to each property in your datatypes. This makes the RAML cleaner and removes the need for `!include examples/` in api.raml.

**Note:** Property-level examples are sufficient. Type-level examples (at the bottom of the datatype) are optional and not required if all properties already have examples.

```raml
#%RAML 1.0 DataType

displayName: Company
description: Represents a company entity in the system
type: object
properties:
  id:
    type: string
    description: Unique identifier for the company
    required: false
    example: "COMP-001"
  name:
    type: string
    description: Company legal name
    required: true
    example: "Acme Manufacturing Ltd"
  nif:
    type: string
    description: Tax identification number (NIF)
    required: true
    pattern: "^[0-9]{9}$"
    example: "500243512"
  status:
    type: string
    description: Company status
    required: false
    enum: [ACTIVE, INACTIVE, PENDING]
    example: "ACTIVE"
  createdAt:
    type: datetime
    description: Record creation timestamp
    required: false
    example: "2024-01-15T10:30:00Z"
example:
  id: "COMP-001"
  name: "Acme Manufacturing Ltd"
  nif: "500243512"
  status: "ACTIVE"
  createdAt: "2024-01-15T10:30:00Z"
```

**Benefits:**
- ✅ Examples are visible when viewing the datatype definition
- ✅ No need for `!include examples/` in api.raml responses
- ✅ Examples are automatically used when the type is referenced
- ✅ Cleaner api.raml file

#### Date/Time Fields (MANDATORY - Use Format Patterns)

**CRITICAL**: All date and time fields in dataTypes MUST use the standardized format patterns from common-data. Do NOT use plain `type: string` or `type: datetime` for date/time fields.

See [common-data-fragment.md](./common-data-fragment.md) for complete format pattern reference.

**Step 1**: Import format patterns in your dataType:
```raml
#%RAML 1.0 DataType

uses:
  format: ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/dataTypes/commons/patterns/format-pattern.raml

type: object
properties:
  # ...
```

**Step 2**: Use format types for date/time properties:
```raml
properties:
  creationDate:
    type: format.dateFormat_aaaa-MM-dd
    description: Record creation date
    required: true
    example: "2025-01-15"

  createdAt:
    type: format.dateTimeFormat_aaaa-MM-ddThhmmssZ
    description: Record creation timestamp
    required: false
    example: "2025-01-15T10:30:00Z"
```

**Available Format Types:**

| Format Type | Pattern | Example | Recommended |
|-------------|---------|---------|-------------|
| `dateFormat_aaaa-MM-dd` | yyyy-MM-dd | `"2023-12-31"` | **YES** |
| `dateFormat_ddmmaaaa` | dd/MM/yyyy | `"29/02/2024"` | |
| `dateTimeFormat_aaaa-MM-ddThhmmssZ` | ISO 8601 UTC | `"2023-12-31T23:59:59Z"` | **YES** |
| `dateTimeFormat_aaaa-MM-ddThhmmss` | ISO 8601 | `"2023-12-31T23:59:59"` | |
| `timeFormat_hhmmss` | HH:mm:ss | `"23:59:59"` | |

**WRONG vs CORRECT:**
```raml
# WRONG - plain string type
validFrom:
  type: string
  example: "15-01-2021"

# CORRECT - using format pattern
validFrom:
  type: format.dateFormat_aaaa-MM-dd
  description: Valid from date
  required: true
  example: "2021-01-15"
```

### Security Schemes

- Define security schemes globally at the root level to centralize authentication logic
- Apply security schemes globally for APIs requiring authentication everywhere
- Apply locally for specific endpoints when needed
- Store complex security schemes in separate files using `!include`

### Documentation & Examples

- Include sample requests and responses for all endpoints
- Provide meaningful descriptions for all resources, methods, and parameters
- Document error codes and their meanings
- Include tutorials and use cases when applicable

## Examples

### Example 1: Resource Naming

```raml
#%RAML 1.0
title: <org>-order-management-papi
version: 1.0
baseUri: <org>-order-management-papi/api/

/orders:
  get:
    description: Retrieve all orders
  post:
    description: Create a new order
  /{orderId}:
    get:
      description: Retrieve a specific order
    patch:
      description: Update an order
    /order-items:
      get:
        description: Retrieve items for an order
```

### Example 2: Using Fragments

```raml
#%RAML 1.0
title: Customer API
version: v1

types:
  Customer: !include types/customer.raml
  Address: !include types/address.raml

traits:
  paginated: !include traits/pagination.raml
  searchable: !include traits/search.raml

/customers:
  get:
    is: [paginated, searchable]
```

### Example 3: Security Scheme

```raml
#%RAML 1.0
title: Secure API
version: v1

securitySchemes:
  oauth_2_0: !include security/oauth2.raml

securedBy: [oauth_2_0]
```

### Example 4: DataType with Embedded Example

```raml
#%RAML 1.0 DataType

uses:
  format: ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/dataTypes/commons/patterns/format-pattern.raml

displayName: Company
description: Represents a company entity in the system
type: object
properties:
  id:
    type: string
    description: Unique identifier for the company
    required: false
    example: "COMP-001"
  name:
    type: string
    description: Company legal name
    required: true
    example: "Acme Manufacturing Ltd"
  nif:
    type: string
    description: Tax identification number (NIF)
    required: true
    pattern: "^[0-9]{9}$"
    example: "500243512"
  status:
    type: string
    description: Company status
    required: false
    enum: [ACTIVE, INACTIVE, PENDING]
    example: "ACTIVE"
  createdAt:
    type: format.dateTimeFormat_aaaa-MM-ddThhmmssZ
    description: Record creation timestamp
    required: false
    example: "2024-01-15T10:30:00Z"
```

**Key points:**
- Server-generated fields (`id`, `createdAt`) are `required: false`
- Date/time fields use format patterns from common-data (NOT `type: datetime`)
- All properties have descriptions and property-level examples
- Enums are used for fields with fixed values

## Modifying Existing API Specifications

When adding features to an existing API specification from Anypoint Exchange:

1. **Download original files exactly as they are** - Never recreate or rewrite existing files manually
2. **Only add new code** - Create only new files (types, examples) for new features
3. **Never modify existing code without explicit user request** - Existing types, examples, and endpoints must remain untouched
4. **Be 100% faithful to the original** - Preserve original structure, formatting, and content exactly

### Workflow

1. Fetch API specification from Exchange using anypoint MCP tools
2. Download all original files to local folder
3. Add only new files for new features
4. Update main RAML by appending new endpoints/references - never modify existing sections
5. User reviews before publishing

## Common Mistakes

| Mistake | Why It's a Problem | Correct Approach |
|---------|-------------------|------------------|
| Using verbs in resource names | Violates REST conventions | Use nouns: `/orders` not `/getOrders` |
| Duplicating type definitions | Hard to maintain, inconsistent | Use `!include` and fragments |
| No versioning | Breaking changes affect all consumers | Always version: `/v1/resource` |
| Overly complex inheritance | Difficult to understand and maintain | Keep inheritance simple, prefer composition |
| Missing examples | Consumers can't understand expected data | Include realistic examples for all types |
| Large monolithic RAML files | Poor performance, hard to navigate | Split into modular fragments |
| Defining types directly in api.raml | Pollutes root file, hard to maintain | Use `library.raml` to centralize datatypes |
| Examples only in API responses | Datatypes lack documentation | Include examples in datatype files using `!include` |
| **Inline response definitions** | Inconsistent responses, hard to maintain | Use `resourceTypes` (collectionType from common-data) |
| **Inline queryParams** | Code duplication, inconsistent params | Define as `traits` in library.raml |
| **Not using common.error-response** | Inconsistent error formats | Always use `common.error-response` for errors |
| **Same response type for GET and POST** | GET and POST should return different data | Use `objectResponse` for GET, `createResponse` (IdResponse type) for POST |
| **Missing errorResponse trait** | Inconsistent error handling | Add `lib.errorResponse` trait to ALL endpoints |
| **Missing property-level examples** | Less visible documentation | Add `example` to each property in datatypes (type-level example is optional) |
| **Missing uriParameters documentation** | Path parameters lack type/description | Document all `{param}` with type, description, and example |
| **Direct `!include` for common-data** | Bypasses library pattern | Import via `uses:` pattern then reference as `common.type-name` |
| **Missing `is: [lib.errorResponse]` on methods** | Error handling not applied | Add trait to ALL endpoint methods, not just resourceType |
| **Properties missing `description` or `required`** | Incomplete datatype documentation | Every property MUST have `description` and explicit `required: true/false` |
| **Trying to use uriParameters in traits** | NOT supported in RAML 1.0 | Define uriParameters inline at each resource - traits only support queryParameters |
| **Separate datatypes for POST and PATCH** | Unnecessary duplication when structure is identical | Merge into single datatype with optional fields for method-specific fields |
| **Using "types" or "datatypes" folder** | Inconsistent with template | Use `dataTypes/` (camelCase) folder with `requests/` and `responses/` subfolders |
| **Root file named `api.raml`** | Doesn't follow naming convention | Name root file after API: `<org>-{name}-{sapi|papi|eapi}.raml` |
| **`exchange.json` main field mismatch** | Publishing to Exchange will fail | Ensure `main` field matches actual root file name |
| **Renaming/creating new root file when updating API** | Breaks existing references and publishing | Keep original root file name, only modify content |
| **Type names using camelCase or PascalCase** | Inconsistent with template | Use kebab-case: `generic-request`, `get-material-response` |
| **Using `clientCredentials` for security** | Inconsistent with template | Use `client_credentials` (underscore): `securedBy: lib.client_credentials` |
| **library.raml in root folder** | Inconsistent project structure | Place inside `library/` folder: `library/library.raml` |
| **Redefining resourceTypes in library.raml** | Duplication - they come from common-data | Import resourceTypes from common-data fragment, do NOT redefine |
| **Using `type: string` for date/time fields** | No validation, inconsistent formats | Use common-data format patterns: `format.dateFormat_aaaa-MM-dd` |
| **Date fields without format pattern import** | Missing regex validation, non-standard formats | Import `format-pattern.raml` and use format types in dataTypes |

## Checklist

### MANDATORY (Must Have)

#### Naming & Structure
- [ ] **API name follows naming convention**: `<org>-{name}-sapi/papi/eapi`
- [ ] **Root file named after API**: `<org>-{name}-{sapi|papi|eapi}.raml`
- [ ] **exchange.json `main` matches root file name**
- [ ] **baseUri format**: `{api-name}/api/` (e.g., `<org>-example-sapi/api/`)
- [ ] **library.raml inside `library/` folder** (NOT in root)
- [ ] **`dataTypes/` folder** (camelCase) with `requests/` and `responses/` subfolders
- [ ] **Type names use kebab-case**: `generic-request`, `get-material-response`
- [ ] **Security scheme**: `client_credentials` (underscore, NOT camelCase)
- [ ] **ResourceTypes imported from common-data** (NOT redefined in library.raml)

#### Common-Data Import (STRICT)
- [ ] **common-data imported via `uses:` pattern** (NOT direct `!include` to exchange_modules)
```raml
# CORRECT:
uses:
  common: exchange_modules/.../common-data/<COMMON_DATA_VERSION>/common-data.raml
securitySchemes:
  client_credentials: common.client-credentials

# WRONG:
securitySchemes:
  client_credentials: !include exchange_modules/.../client-credentials.raml
```

#### ResourceTypes (STRICT)
- [ ] **`collectionType` imported from common-data** (used for both collections and item endpoints)
- [ ] **Correct parameter names used** (per common-data `default-collection.raml`):
  - `objectResponse` (GET response)
  - `createObject` / `createResponse` (POST)
  - `putRequestObject` (PUT request body)
  - `patchRequestObject` (PATCH request body)
  - `errorResponse` (error responses)

#### Error Handling (STRICT)
- [ ] **errorResponse trait defined** in library.raml
- [ ] **errorResponse trait applied to ALL endpoint methods** via `is: [lib.errorResponse]`
- [ ] **common.error-response used** for all error responses (400, 404, 500)

#### Data Types
- [ ] **Examples at property level** in datatype files
- [ ] **All properties have `description`** field
- [ ] **All properties have explicit `required`** field (true or false)
- [ ] **No `!include examples/` in api.raml** (examples come from datatypes)
- [ ] **Date/time fields use format patterns** from common-data (NOT `type: string`)
- [ ] **Format patterns imported** in dataTypes with date fields via `uses: format:`

#### Documentation
- [ ] **All resources have `displayName` and `description`**
- [ ] **All methods have `displayName` and `description`**
- [ ] **uriParameters documented** for ALL path parameters with type, description, and example
- [ ] **Traits used for queryParams** (no inline queryParams definitions)

#### DRY Principle (Avoid Duplication)
- [ ] **Duplicate datatypes merged** when POST/PATCH use same structure (e.g., `clients.raml` instead of `post-clients.raml` + `patch-clients.raml`)

#### Response Types
- [ ] **Different response types per HTTP method**:
  - GET uses `objectResponse`
  - POST uses `createResponse` (typically an ID response, NOT same as GET)

### Best Practices
- [ ] Standard traits applied (pageable, sortable, correlation-id)
- [ ] API follows design-first approach
- [ ] Resource names use lowercase with hyphens
- [ ] Resource names are plural nouns
- [ ] Resources have displayName and description
- [ ] API version included in base URI
- [ ] RAML is modularized using fragments
- [ ] Data types defined in separate files under `types/` folder
- [ ] Examples defined in separate files under `examples/` folder
- [ ] Server-generated fields (e.g., `id`) marked as `required: false`
- [ ] Security schemes defined globally (using common-data schemes)
- [ ] All endpoints have descriptions
- [ ] Request/response examples provided
- [ ] Error responses documented
- [ ] API follows API-led connectivity pattern (System/Process/Experience)
- [ ] Project size is under 500 KB
- [ ] Root `api.raml` file is clean (uses `lib.TypeName` references)

---

## Strict Review Requirements

**IMPORTANT: All RAML reviews MUST be strict and comprehensive. Surface-level checks are NOT sufficient.**

### Review Depth Levels

| Level | What It Checks | Acceptable? |
|-------|----------------|-------------|
| **Surface** | Typos, missing displayName, missing descriptions | ❌ NOT SUFFICIENT |
| **Structural** | ResourceTypes usage, parameter names, trait application | ✅ MINIMUM REQUIRED |
| **Deep** | Import patterns, datatype completeness, response type correctness | ✅ RECOMMENDED |

### What a Strict Review MUST Check

#### 1. ResourceType Parameter Names (Per common-data)
```raml
# CHECK: Are parameter names correct per common-data default-collection.raml?
objectResponse: lib.get-response          # ✅ Correct (GET)
createObject: lib.create-request          # ✅ Correct (POST body)
createResponse: lib.post-response         # ✅ Correct (POST response)
patchRequestObject: lib.patch-request     # ✅ Correct (PATCH body)
putRequestObject: lib.put-request         # ✅ Correct (PUT body)

# NOTE: collectionType can be used for BOTH collections and item endpoints
# The common-data fragment does NOT have a separate itemType
```

#### 2. Trait Application (Not Just Definition)
```raml
# CHECK: Is errorResponse trait APPLIED to every method?
get:
  displayName: Get Resource
  # ❌ Missing: is: [lib.errorResponse]

get:
  displayName: Get Resource
  is: [lib.errorResponse]  # ✅ Correct
```

#### 3. Import Pattern (Not Just Imported)
```raml
# CHECK: Is common-data imported via uses: pattern?

# ❌ WRONG - Direct include
errorResponse: !include ../exchange_modules/.../error-response-data-type.raml

# ✅ CORRECT - Via uses pattern
uses:
  common: ../exchange_modules/.../common-data.raml
types:
  errorResponse: common.error-response
```

#### 4. DataType Completeness
```raml
# CHECK: Does EVERY property have description AND required?

# ❌ INCOMPLETE
code:
  type: number
  example: 123

# ✅ COMPLETE
code:
  type: number
  description: Internal system code
  required: true
  example: 123
```

### Review Output Format

All reviews MUST produce a report with:

1. **Summary Table** - Pass/Fail status per category
2. **Critical Issues** - Structural problems that MUST be fixed
3. **Medium Issues** - Important but non-blocking
4. **Minor Issues** - Nice-to-have improvements
5. **Checklist Status** - Each mandatory item explicitly marked

### Issue Severity Classification

**IMPORTANT:** Correctly classify issue severity to avoid blocking publishing unnecessarily.

| Severity | Definition | Examples |
|----------|------------|----------|
| **Critical** | Blocks publishing or causes runtime errors | Wrong resourceType parameters, missing required files, RAML syntax errors |
| **Medium** | Should fix but doesn't block publishing | Missing `displayName` on methods, incomplete descriptions |
| **Low** | Nice-to-have, best practice | Incomplete exchange.json fields, minor documentation gaps |

**NOT Critical (common false positives):**
- Incomplete `exchange.json` - fields can be passed as tool parameters when publishing
- Missing `displayName` on methods - doesn't block publishing
- Missing type-level examples (if property-level examples exist)

### Common Review Mistakes to Avoid

| Mistake | Why It's Wrong | Correct Approach |
|---------|----------------|------------------|
| Marking "resourceTypes used" as PASS without checking parameters | Parameter names might be wrong | Verify parameter names match common-data: `patchRequestObject`, `putRequestObject`, etc. |
| Marking "common-data imported" as PASS | Doesn't verify import pattern | Check for `uses:` pattern, not direct `!include` |
| Marking "errorResponse used" as PASS | Might be in resourceType but not applied as trait | Verify `is: [lib.errorResponse]` on ALL methods |
| Only checking for typos/missing fields | Surface-level review | Check structural patterns and correctness |
| Flagging `patchRequestObject` as wrong | This IS the correct parameter name per common-data | `patchRequestObject` and `putRequestObject` are CORRECT (check `default-collection.raml`) |
| Flagging `collectionType` for item endpoints | common-data only has `collectionType`, no `itemType` | `collectionType` can be used for BOTH collections and items |
| Marking incomplete exchange.json as Critical | Fields can be passed as publish tool parameters | Mark as Low severity - best practice, not blocker |

---

## References

- [Best Practices when using API Designer - MuleSoft Documentation](https://docs.mulesoft.com/design-center/design-best-practices)
- [Best practices to design your first API Specification - MuleSoft Developers](https://developer.mulesoft.com/tutorials-and-howtos/getting-started/best-practices-first-api-spec/)
- [Step 3. Develop the API - MuleSoft Documentation](https://docs.mulesoft.com/general/api-led-develop)

---
Last updated: 2025-12-30 (Added: Root file naming convention - must match API name; exchange.json main field alignment critical for publishing)
Owner: Development Team
