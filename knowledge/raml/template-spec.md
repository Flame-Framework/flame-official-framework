# Organization Template API Specification

> **MANDATORY**: This template MUST be followed for ALL new RAML API specifications.
> Source: Anypoint Exchange - `<template-asset-id>` (asset ID and version configured per organization)

## Template Structure

```
<org>-{name}-{sapi|papi|eapi}/
├── <org>-{name}-{sapi|papi|eapi}.raml  # Root API file (named after API)
├── exchange.json                        # main field MUST match root file name
├── library/
│   └── library.raml              # Centralizes types and resourceTypes; references traits via !include
├── dataTypes/                    # Note: "dataTypes" not "datatypes"
│   ├── requests/
│   │   └── {entity}-request.raml
│   └── responses/
│       └── {entity}-response.raml
├── traits/                       # Only create subfolders the API actually needs
│   ├── queryParams/              # Traits that define queryParameters
│   │   └── {trait-name}.raml
│   └── headers/                  # Traits that define custom headers
│       └── {trait-name}.raml
└── exchange_modules/             # Auto-managed dependencies
    └── <ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/
```

---

## Template Files

### 1. Root API File (`<org>-{name}-{sapi|papi|eapi}.raml`)

```raml
#%RAML 1.0
title: <org>-{name}-{sapi|papi|eapi}
version: 1.0.0

baseUri: <org>-{name}-{sapi|papi|eapi}/api/

description: |
  SET BRIEF DESCRIPTION OF API IN HERE

uses:
  lib: library/library.raml

securedBy: lib.client_credentials

/accounts:
  type:
    lib.collectionType:
      errorResponse: lib.errorResponse
      objectResponse: lib.generic-response

  get:
    displayName: Get Accounts
    description: Get all the accounts
```

**Key Rules:**
- **Root file name**: Must match API name with `.raml` extension (e.g., `<org>-orders-sapi.raml`)
- `title`: Must match folder/file name pattern `<org>-{name}-{sapi|papi|eapi}`
- `version`: Semantic version (e.g., `1.0.0`)
- `baseUri`: Format is `{api-name}/api/` (NO version placeholder, NO https://)
- `uses`: Reference library as `lib: library/library.raml`
- `securedBy`: Use `lib.client_credentials`
- All resources use `lib.collectionType` resourceType with `errorResponse` parameter

**CRITICAL - When updating existing APIs:**
- **NEVER rename the existing root file**
- **NEVER create a new root file**
- Keep the original root file name, only modify its content

---

### 2. Library File (`library/library.raml`)

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

traits:
  filter: !include ../traits/queryParams/filter.raml
  correlationId: !include ../traits/headers/correlation-id.raml
```

**Key Rules:**
- File location: MUST be inside `library/` folder
- Security schemes: Import from common-data (DO NOT redefine)
- Types: Use kebab-case for type names (e.g., `generic-request`, NOT `genericRequest`)
- resourceTypes: Import `collectionType` from common-data (DO NOT redefine)
- Traits: MUST be referenced via `!include` from the `traits/` folder — NEVER defined inline here. Omit the `traits:` block entirely if the API has no custom traits.
- Paths use `../` to go up from library folder

---

### 3. Request DataType (`dataTypes/requests/{entity}-request.raml`)

```raml
#%RAML 1.0 DataType

properties:
  message:
    description: Request Simple Message
    type: string
    example: "Sent"
```

**Key Rules:**
- Every property MUST have `description`, `type`, and `example`
- Use `required: true` or `required: false` explicitly when needed
- File naming: `{entity}-request.raml` (kebab-case)

---

### 4. Response DataType (`dataTypes/responses/{entity}-response.raml`)

```raml
#%RAML 1.0 DataType

properties:
  message:
    description: Response Simple Message
    type: string
    example: "Received"
```

**Key Rules:**
- Same rules as request datatypes
- File naming: `{entity}-response.raml` or `get-{entity}.raml`, `post-{entity}.raml` (kebab-case)

---

### 5. Trait Files (`traits/queryParams/{trait}.raml` and `traits/headers/{trait}.raml`)

Custom traits MUST live in their own files inside the `traits/` folder, organized by the type of content they define:

- `traits/queryParams/` — for traits that define `queryParameters`
- `traits/headers/` — for traits that define custom `headers`

Only create the subfolders the API actually needs (e.g., if the API has only query-parameter traits, do not create `traits/headers/`).

**Example** (`traits/queryParams/filter.raml`):

```raml
#%RAML 1.0 Trait

queryParameters:
  filter?:
    description: Filter
    type: string
    example: "PA00"
```

**Example** (`traits/headers/correlation-id.raml`):

```raml
#%RAML 1.0 Trait

headers:
  x-correlation-id?:
    description: Request correlation identifier
    type: string
    example: "abc-123-def-456"
```

**Key Rules:**
- Each trait file MUST start with `#%RAML 1.0 Trait`
- One trait per file
- File names use kebab-case (e.g., `correlation-id.raml`, NOT `correlationId.raml`)
- Traits MUST be referenced from `library/library.raml` via `!include` under the `traits:` section
- Traits MUST NOT be defined inline in the root RAML file, in `library.raml` directly, or in any other location

---

### 6. Exchange Metadata (`exchange.json`)

```json
{
  "dependencies": [
    {
      "version": "<COMMON_DATA_VERSION>",
      "assetId": "common-data",
      "groupId": "<ANYPOINT_ORG_ID>",
      "provided": false
    }
  ],
  "version": "1.0.0",
  "descriptorVersion": "1.0.0",
  "classifier": "raml",
  "main": "<org>-{name}-{sapi|papi|eapi}.raml",
  "assetId": "<org>-{name}-{sapi|papi|eapi}",
  "groupId": "<ANYPOINT_ORG_ID>",
  "name": "<org>-{name}-{sapi|papi|eapi}",
  "organizationId": "<ANYPOINT_ORG_ID>",
  "apiVersion": "v1",
  "tags": [],
  "originalFormatVersion": "1.0"
}
```

**Key Rules:**
- **CRITICAL: `main` MUST match the actual root file name** - If they don't match, publishing will fail
- `main`: Root file follows API naming convention (e.g., `<org>-orders-sapi.raml`)
- `assetId` and `name`: Must match the API title
- `apiVersion`: Use `v1` format (for version 1.x.x assets)
- `dependencies`: Include common-data fragment at the configured version

---

## Common-Data Dependency

The template uses the common-data fragment (configured version) which provides:

| Component | Path | Usage |
|-----------|------|-------|
| Client Credentials | `securitySchemas/client-credentials.raml` | OAuth 2.0 client credentials |
| Error Response | `dataTypes/error-response-data-type.raml` | Standard error format |
| Collection Type | `collections/default-collection.raml` | GET/POST/PATCH/DELETE patterns |

**Organization ID:** `<ANYPOINT_ORG_ID>`

---

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| API Title/Folder | `<org>-{name}-{sapi\|papi\|eapi}` | `<org>-cmms-sapi` |
| **Root file** | `<org>-{name}-{sapi\|papi\|eapi}.raml` | `<org>-cmms-sapi.raml` |
| baseUri | `{api-name}/api/` | `<org>-cmms-sapi/api/` |
| Type names in library | kebab-case | `generic-request`, `error-response` |
| DataType files | kebab-case | `material-request.raml` |
| Folder names | camelCase | `dataTypes`, NOT `datatypes` |

---

## Quick Start Checklist

When creating a new API spec:

1. [ ] Copy template structure
2. [ ] Rename root file to match API name: `<org>-{name}-{sapi|papi|eapi}.raml`
3. [ ] Update `title` to match `<org>-{name}-{sapi|papi|eapi}`
4. [ ] Update `baseUri` to `<org>-{name}-{sapi|papi|eapi}/api/`
5. [ ] Update `description` with API purpose
6. [ ] Add request/response datatypes in `dataTypes/requests/` and `dataTypes/responses/`
7. [ ] If the API needs custom traits, add them as separate files under `traits/queryParams/` and/or `traits/headers/` — never inline
8. [ ] Update `library/library.raml` to include new types and `!include` references to every custom trait
9. [ ] **Update `exchange.json`**: ensure `main` matches root file name, update `assetId` and `name`
10. [ ] Define resources using `lib.collectionType` with `errorResponse` parameter

**When updating an existing API:**

1. [ ] **NEVER rename the existing root file**
2. [ ] **NEVER create a new root file**
3. [ ] Only modify the content of the existing files
4. [ ] Ensure `exchange.json` `main` still matches the root file name

---

## Example: Adding a New Resource

**1. Create request datatype** (`dataTypes/requests/material.raml`):
```raml
#%RAML 1.0 DataType

displayName: Material
description: Material request data
type: object
properties:
  materialCode:
    type: string
    description: Unique material code
    required: true
    example: "MAT001"
  description:
    type: string
    description: Material description
    required: true
    example: "Steel plate 10mm"
```

**2. Create response datatype** (`dataTypes/responses/get-material.raml`):
```raml
#%RAML 1.0 DataType

displayName: Get Material Response
description: Material response from GET operations
type: object
properties:
  materialCode:
    type: string
    description: Unique material code
    required: true
    example: "MAT001"
  description:
    type: string
    description: Material description
    required: true
    example: "Steel plate 10mm"
  status:
    type: string
    description: Material status
    required: true
    enum: [ACTIVE, INACTIVE]
    example: "ACTIVE"
```

**3. Update library** (`library/library.raml`):
```raml
types:
  errorResponse: !include ../exchange_modules/.../error-response-data-type.raml
  material-request: !include ../dataTypes/requests/material.raml
  get-material-response: !include ../dataTypes/responses/get-material.raml
  post-material-response: !include ../dataTypes/responses/post-material.raml
```

**4. Add resource** (`api.raml`):
```raml
/materials:
  displayName: Materials
  description: Material management
  type:
    lib.collectionType:
      errorResponse: lib.errorResponse
      objectResponse: lib.get-material-response[]
      createObject: lib.material-request
      createResponse: lib.post-material-response

  get:
    displayName: Get Materials
    description: Retrieve all materials

  post:
    displayName: Create Material
    description: Create a new material

  /{materialCode}:
    uriParameters:
      materialCode:
        type: string
        description: Unique material identifier
        example: "MAT001"
    type:
      lib.collectionType:
        errorResponse: lib.errorResponse
        objectResponse: lib.get-material-response
        patchRequestObject: lib.material-request

    get:
      displayName: Get Material
      description: Retrieve a specific material

    patch:
      displayName: Update Material
      description: Update a material
```

---

*Source: Anypoint Exchange - `<template-asset-id>` (asset ID and version configured per organization)*
*Last updated: 2025-12-30 (Root file naming convention - must match API name)*
