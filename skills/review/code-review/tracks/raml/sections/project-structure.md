# Section A — Project Structure Validation <Critical>

> **Agent scope**: All project files.

> **Common mistakes to avoid**:
> - Flagging incomplete exchange.json as Critical → mark as Low severity, fields can be passed as publish tool parameters
> - Expecting `library.raml` in root → must be `library/library.raml`
> - Ignoring inline trait/type definitions → constructs defined outside their designated folders break project structure (traits must be in `traits/`, types in `dataTypes/`, resourceTypes and securitySchemes imported from common-data)

All of the following must be present and correctly structured:

## A.1. Root API File Naming <Critical>
- Root RAML file MUST follow naming convention: `{project-prefix}-{name}-{sapi|papi|eapi}.raml` where `{project-prefix}` is read from `config/framework.yaml` → `organization.project-prefix`. If the prefix is empty, the convention is `{name}-{sapi|papi|eapi}.raml`.
- The `{sapi|papi|eapi}` suffix is mandatory; `{name}` MUST be kebab-case.
- **Examples** (assuming `organization.project-prefix: "myorg"`):
  - `myorg-orders-sapi.raml` ✅
  - `myorg-order-management-papi.raml` ✅
  - `myorg-mobile-eapi.raml` ✅
  - `api.raml` ❌
  - `myorg-example.raml` ❌ (missing layer suffix)

## A.2. exchange.json File <Critical>
- MUST exist in the project root
- **A.2.1**: `main` field MUST match the actual root file name exactly
  ```json
  {
    "main": "<your-root-raml-filename>"  // Must match actual file name
  }
  ```
- **A.2.2**: MUST include the organization's `common-data` dependency. Values are read from `config/framework.yaml` → `raml.common-data.{group-id, asset-id, version}`. The `dependencies` entry MUST match exactly.
  ```json
  {
    "dependencies": [
      {
        "groupId": "<raml.common-data.group-id>",
        "assetId": "<raml.common-data.asset-id>",
        "version": "<raml.common-data.version>"
      }
    ]
  }
  ```

## A.3. Library Folder Structure <Critical>
- MUST have `library/` folder (not in root)
- MUST contain `library/library.raml` file
- Library file MUST centralize types, resourceTypes, traits, and security schemes

## A.4. DataTypes Folder Structure <Critical>
- MUST have `dataTypes/` folder or `datatypes/` folder
- **A.4.1**: MUST contain `dataTypes/requests/` subfolder or MUST contain `datatypes/requests/` subfolder
  - **Exception**: The `requests/` subfolder is NOT required if the API does not contain any resources that require request payloads (e.g., APIs with only GET or DELETE operations that do not accept request bodies)
- **A.4.2**: MUST contain `dataTypes/responses/` subfolder or MUST contain `datatypes/responses/` subfolder
  - **Exception**: The `responses/` subfolder is NOT required if the API does not contain any resources that return custom response payloads (e.g., APIs that only return standard responses like 204 No Content or use only common-data types like errorResponse)
- **A.4.3**: DataType files MUST use kebab-case naming: `material-request.raml`, `get-material-response.raml`

### A.4. DataTypes Folder Naming <Low>
- MUST have `dataTypes/` folder (camelCase, NOT "datatypes" or "types")

## A.5. Traits Folder Structure <Critical>
- If the API uses custom traits, they MUST be defined in a dedicated `traits/` folder
- Trait files MUST be organized into subfolders by the type of content they define:
  - `traits/queryParams/` — for traits that define `queryParameters`
  - `traits/headers/` — for traits that define custom `headers`
- Each trait file MUST use the `#%RAML 1.0 Trait` header
- Traits MUST be referenced via `!include` from `library/library.raml` under the `traits:` section
- Traits MUST NOT be defined inline in the root RAML file, in `library.raml` directly, or in any other location

**Note**: Only create the subfolders that the API actually needs. For example, if the API only uses query parameter traits, only `traits/queryParams/` is required.

**Expected Structure**:
```
traits/
├── queryParams/          # queryParameters traits (e.g., filter.raml, id.raml)
└── headers/              # headers traits (e.g., correlation-id.raml)
```

**Example** (`traits/queryParams/filter.raml`):
```raml
#%RAML 1.0 Trait
queryParameters:
  filter?:
    description: Filter
    type: string
    example: "PA00"
```

Traits are referenced from `library/library.raml` via `!include`: `filter: !include ../traits/queryParams/filter.raml`

## A.6. Unused RAML Artifacts <Critical>
Validate that ALL dataType files, trait files, and library references are actually used. Unused artifacts are dead code and MUST be removed.

### A.6.1. Unused DataType Files <Critical>
- For each `.raml` file in `dataTypes/requests/` and `dataTypes/responses/`, verify it is referenced via `!include` in `library/library.raml` under the `types:` section
- If a dataType file exists but is NOT included in `library/library.raml`, it is unused
- **How to detect**:
  1. List all `.raml` files in `dataTypes/requests/` and `dataTypes/responses/`
  2. Read `library/library.raml` and collect all `!include` paths under `types:`
  3. Any dataType file not present in the include list is unused

### A.6.2. Unused Trait Files <Critical>
- For each `.raml` file in `traits/` (and its subfolders), verify it is referenced via `!include` in `library/library.raml` under the `traits:` section
- If a trait file exists but is NOT included in `library/library.raml`, it is unused
- **How to detect**:
  1. List all `.raml` files in `traits/` and its subfolders (`traits/queryParams/`, `traits/headers/`, etc.)
  2. Read `library/library.raml` and collect all `!include` paths under `traits:`
  3. Any trait file not present in the include list is unused

### A.6.3. Unused Library References <Critical>
- For each type, trait, or resourceType declared in `library/library.raml`, verify it is actually used in the root RAML file
- If a library reference (e.g., `lib.some-type`, `lib.some-trait`) is declared in `library/library.raml` but never appears in the root RAML file, it is unused
- **How to detect**:
  1. Read `library/library.raml` and collect all declared names under `types:`, `traits:`, and `resourceTypes:`
  2. Read the root RAML file and search for `lib.<name>` references (in `type:`, `is:`, `securedBy:`, etc.)
  3. Any declared name that has **zero references** in the root RAML file is unused
  4. **Exception**: `errorResponse` imported from common-data is typically used indirectly via `collectionType` — verify before flagging

- **Why this matters**: Unused artifacts clutter the API specification, increase maintenance burden, and can cause confusion about which types and traits are actually part of the API contract

## A.7. Exchange Modules <Critical>
- MUST have `exchange_modules/` folder containing the common-data dependency
- Path MUST be: `exchange_modules/<raml.common-data.group-id>/<raml.common-data.asset-id>/<raml.common-data.version>/` — values read from `config/framework.yaml` → `raml.common-data.*`

## A.8. RAML Construct Placement <Critical>
Every RAML construct MUST be defined in its designated location. Defining constructs outside their proper folder breaks the project structure and makes the API specification harder to maintain.

| Construct | Designated Location | Referenced From |
|-----------|-------------------|-----------------|
| **Types** (dataTypes) | `dataTypes/requests/` and `dataTypes/responses/` | `library/library.raml` under `types:` |
| **Traits** | `traits/queryParams/` and/or `traits/headers/` | `library/library.raml` under `traits:` |
| **ResourceTypes** | Imported from `exchange_modules/` (common-data) | `library/library.raml` under `resourceTypes:` |
| **SecuritySchemes** | Imported from `exchange_modules/` (common-data) | `library/library.raml` under `securitySchemes:` |

**Violations to flag**:
- Traits defined inline in the root RAML file or inside `library.raml` instead of in the `traits/` folder
- DataType definitions placed outside `dataTypes/requests/` or `dataTypes/responses/`
- ResourceTypes or SecuritySchemes redefined locally instead of imported from common-data
- Any RAML construct defined in an arbitrary or non-standard location

**Correct pattern**: Traits in `traits/` folder, referenced via `!include` from `library.raml`, applied in root RAML via `is: [lib.traitName]`

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 16
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (A.1, A.2, A.2.1, A.2.2, A.3, A.4, A.4.1, A.4.2, A.4.3, A.5, A.6, A.6.1, A.6.2, A.6.3, A.7, A.8), identify the missing ones, go back and validate them, then update your counts before returning.**
