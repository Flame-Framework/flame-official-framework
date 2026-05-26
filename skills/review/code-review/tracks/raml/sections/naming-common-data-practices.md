# Sections B.6 + B.7 + B.8 + B.9 — Naming, Common-Data, Best Practices & Versioning

> **Agent scope**: All project files.

> **Common mistakes to avoid**:
> - Accepting camelCase for type names → enforce kebab-case: `material-request` not `materialRequest`

## B.6. Resource Naming Conventions

### B.6.1. Resource Path Naming <Critical>
- MUST use lowercase with hyphens
- MUST use plural nouns for collections
- MUST NOT use verbs

**Examples**: `/materials`, `/order-items`, `/customer-accounts`
**Violations**: verbs (`/getMaterials`), uppercase (`/OrderItems`), singular (`/material`)

### B.6.2. API Naming Pattern <Critical>
API name MUST follow the API-Led Connectivity pattern. `{project-prefix}` is read from `config/framework.yaml` → `organization.project-prefix`. If empty, omit the leading `{project-prefix}-`.

| API Type | Pattern | Example (with `organization.project-prefix: "myorg"`) |
|----------|---------|---------|
| **System API** | `{project-prefix}-{system}-sapi` | `myorg-orders-sapi` |
| **Process API** | `{project-prefix}-{business-process}-papi` | `myorg-order-management-papi` |
| **Experience API** | `{project-prefix}-{channel/purpose}-eapi` | `myorg-mobile-eapi` |

### B.6.3. Type Name Conventions <Medium>
- SHOULD use kebab-case for all type names in library.raml (RECOMMENDED)
- camelCase is also acceptable
- Examples:
  - kebab-case (recommended): `material-request`, `get-material-response`, `error-response`
  - camelCase (acceptable): `materialRequest`, `getMaterialResponse`, `errorResponse`

**Note**: If camelCase is used, suggest adjusting to kebab-case as a best practice recommendation.

---

## B.7. Common-Data Integration

### B.7.1. Import Pattern <Critical>
- common-data MUST be imported via `uses:` pattern or direct `!include` in `library.raml`
- Direct `!include` outside `library.raml` (e.g., in root RAML file) is a violation

### B.7.2. Error Response Usage <Medium>
- All error responses (400, 404, 500) SHOULD use `lib.errorResponse`
- This is typically handled by resourceTypes, but custom endpoints should also use it

---

## B.8. Best Practices Validation

### B.8.1. Avoid Duplicate DataTypes <Low>
- When POST and PATCH use identical structures, merge into a single datatype with optional fields (e.g., `material.raml` with `required: false` for fields only needed by one method)

### B.8.2. No Inline Definitions <Low>
- Avoid inline queryParameters definitions (use traits)
- Avoid inline response definitions (use resourceTypes)

### B.8.3. Documentation Completeness <Low>
- All endpoints should have meaningful descriptions
- Include examples where helpful
- Document any special behaviors or constraints

---

## B.9. Versioning Validation

### B.9.1. Semantic Versioning <Low>
Version should follow SemVer (MAJOR.MINOR.PATCH)
- Breaking changes: MAJOR bump
- New features: MINOR bump
- Bug fixes: PATCH bump

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 9
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (B.6.1, B.6.2, B.6.3, B.7.1, B.7.2, B.8.1, B.8.2, B.8.3, B.9.1), identify the missing ones, go back and validate them, then update your counts before returning.**
