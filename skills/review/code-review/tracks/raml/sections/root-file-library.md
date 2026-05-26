# Sections B.1 + B.2 — Root API File + Library Validation

> **Agent scope**: Root RAML file + `library/library.raml`.

> **Common mistakes to avoid**:
> - Accepting `clientCredentials` for security → must be `client_credentials` (underscore)
> - Accepting camelCase for type names → enforce kebab-case: `material-request` not `materialRequest`

## B.1. Root API File Structure

### B.1.1. Required Root Fields <Critical>
All root RAML files MUST include:
```raml
#%RAML 1.0
title: {project-prefix}-{name}-{sapi|papi|eapi}
version: 1.0.0
baseUri: {project-prefix}-{name}-{sapi|papi|eapi}/api/
description: |
  Clear description of API purpose
```
Where `{project-prefix}` is read from `config/framework.yaml` → `organization.project-prefix`. If the prefix is empty, the convention is `{name}-{sapi|papi|eapi}` (no leading prefix).

**Validation Rules**:
- `title` MUST match the A.1 naming convention
- `baseUri` MUST follow format: `{api-name}/api/` (NO https://, NO version placeholder)
- `description` MUST be present and meaningful (not placeholder text)

### B.1.2. Uses Declaration <Critical>
- MUST include library reference:
  ```raml
  uses:
    lib: library/library.raml
  ```

### B.1.3. Security Declaration <Critical>
- SHOULD apply security globally. Recommend `securedBy: lib.client_credentials`. Other naming conventions (`clientCredentials`, `client-credentials`) are accepted but suggest adjusting to the standard format.

---

## B.2. Library File Validation

### B.2.1. Security Schemes Import <Critical>
- MUST import security schemes from common-data via `!include` (NOT redefine them)
- All naming conventions are accepted (`client_credentials`, `clientCredentials`, `client-credentials`), but recommend `client_credentials` (underscore)
- Redefining the scheme inline instead of importing is a violation
  ```raml
  securitySchemes:
    client_credentials: !include ../exchange_modules/<raml.common-data.group-id>/<raml.common-data.asset-id>/<raml.common-data.version>/securitySchemas/client-credentials.raml
  ```

### B.2.2. Security Schemes Naming <Low>
- MUST use `client_credentials` naming (underscore). If `clientCredentials` or `client-credentials` is used, suggest adjusting to the standard format

### B.2.3. Types Import <Critical>
- MUST import `errorResponse` from common-data
- MUST reference dataTypes from correct paths:
  ```raml
  types:
    errorResponse: !include ../exchange_modules/<raml.common-data.group-id>/<raml.common-data.asset-id>/<raml.common-data.version>/dataTypes/error-response-data-type.raml
    material-request: !include ../dataTypes/requests/material-request.raml
    get-material-response: !include ../dataTypes/responses/get-material-response.raml
  ```

### B.2.4. Types Naming <Low>
- MUST use kebab-case for type names (not PascalCase or camelCase)
  ```raml
  types:
    errorResponse: !include ../exchange_modules/.../error-response-data-type.raml
    material-request: !include ../dataTypes/requests/material-request.raml
    get-material-response: !include ../dataTypes/responses/get-material-response.raml
  ```

### B.2.5. ResourceTypes Import <Critical>
- MUST import `collectionType` from common-data via `!include` (NOT redefine it inline)
- **collectionType can be used for BOTH collections and item endpoints** (there is no separate itemType)
  ```raml
  resourceTypes:
    collectionType: !include ../exchange_modules/<raml.common-data.group-id>/<raml.common-data.asset-id>/<raml.common-data.version>/collections/default-collection.raml
  ```

### B.2.6. Traits Definition <Medium>
- Should define custom traits for queryParameters (when needed)
- Traits can ONLY contain: `queryParameters`, `headers`, `responses` (NOT `uriParameters`)
- Traits MUST be defined as separate files in the `traits/queryParams/` folder and referenced via `!include` (see rule A.5)
- Traits MUST NOT be defined inline in `library.raml` or in the root RAML file
- Example:
  ```raml
  # In library/library.raml - reference external trait files
  traits:
    dateRangeFilter: !include ../traits/queryParams/date-range-filter.raml
    filter: !include ../traits/queryParams/filter.raml
  ```

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 9
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (B.1.1, B.1.2, B.1.3, B.2.1, B.2.2, B.2.3, B.2.4, B.2.5, B.2.6), identify the missing ones, go back and validate them, then update your counts before returning.**
