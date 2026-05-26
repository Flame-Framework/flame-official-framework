# MuleSoft API Development Best Practices

## Overview

This document outlines the core conventions for developing APIs in MuleSoft's Anypoint Platform. For detailed standards on specific topics, see the dedicated files linked in Related Documentation.

## API-Led Connectivity Layers

Our architecture follows the three-tier API-led connectivity model:

| Layer | Purpose | Naming Convention | Layer Code | Example |
|-------|---------|-------------------|------------|---------|
| **System API (SAPI)** | Direct backend connectivity | `{org}-{system}-sapi` | `sys` | acme-erp-sapi |
| **Process API (PAPI)** | Business logic orchestration | `{org}-{domain}-papi` | `prc` | acme-orders-papi |
| **Experience API (EAPI)** | Consumer-facing interfaces | `{org}-{consumer}-eapi` | `exp` | acme-portal-eapi |

> **Layer Codes** identify the API layer in error responses (`error.message` prefix in SAPI) and in any layer-specific configuration.

## Project Setup

- If the current development relates to a new API in portfolio, always start by using the `{org}-template-api` as template
- **Configure Parent POM**: Inherit from `{org}-parent-pom`:
  ```xml
  <parent>
      <groupId>com.mulesoft</groupId>
      <artifactId>{org}-parent-pom</artifactId>
      <version>X.Y.Z</version>
  </parent>
  ```
- Add Anypoint Platform credentials in Studio once for all future projects
- Import API specifications from Exchange when creating new projects
- Enable scaffolding to auto-generate flows from API specifications
- Use APIkit to scaffold flows based on RAML/OAS endpoints

### Required Maven Properties

```xml
<properties>
    <api.layer>System API|Process API|Experience API</api.layer>
    <project.name>${project.artifactId}</project.name>
    <api.version>v1</api.version>
</properties>
```

## Flow Logic

- **Choice routers**: ALL routes inside Choice routers must have at least one processor. An empty `<otherwise>` route is not allowed — if no action is needed, at minimum add a logger or comment explaining why
- **Unreachable logic**: Ensure conditional expressions can logically evaluate to both true and false. Avoid `isEmpty()` checks on values that are guaranteed to be non-empty at that point in the flow
- **httpStatus in main flow**: Never set `httpStatus` to 4xx/5xx in the main flow — use `raise-error` instead and let the error handler set the status code

## Layer-Specific Patterns

Each API layer has its own sub-flow pattern and rules. See [API Layer Patterns](./api-layers.md) for:
- **SAPI**: auth → request → map → respond (backend query params, response status evaluation)
- **PAPI**: orchestrate → aggregate → logic (scatter-gather, foreach+try, async)
- **EAPI**: validate → call PAPIs → shape (date range filtering, pagination, security headers)

For XML examples of these patterns, see [Component Examples](../usage/component.md).

## Reusability — Reuse-First Rule

Before creating any new sub-flow, connector config, or DWL transform, **always scan existing project code first** for reusable components:

- **Sub-flows**: Check if an existing sub-flow already performs similar logic (same target system, similar transformation, shared business logic)
- **Connector configs**: Check `global.xml` for existing connector configurations that connect to the same target system — reuse instead of creating duplicates
- **DWL files**: Check `dwl/` for existing transforms that can be reused or extended (common error responses, shared utility functions, similar field mappings)

**What counts as reusable:**
- An existing `http:request-config` that points to the same target host/basePath
- A sub-flow that calls the same backend endpoint with the same method
- A DWL file that maps the same source/target fields (or a subset)
- Common error response DWL files in `dwl/error/`

**When NOT to reuse:**
- The existing component would need incompatible modifications
- Reusing would create tight coupling between unrelated features
- The user explicitly requests new code

## KB Files Are the Source of Truth

- **NEVER** copy patterns from existing project code (e.g., copying an existing flow to create a new one)
- Existing code may have deviations from current standards
- Always derive implementation strictly from the KB context files (`logging.md`, `api-layers.md`, `dataweave-patterns.md`, etc.)
- If existing code conflicts with KB, **follow the KB**
- Reusing existing components (sub-flows, configs, DWL) is encouraged via the Reuse-First Rule above, but the **implementation patterns** must come from the KB, not from copying existing code

## Common Mistakes

| Mistake | Why It's a Problem | Correct Approach |
|---------|-------------------|------------------|
| Not using template | Inconsistent project structure | Always clone `{org}-template-api` |
| Not using parent POM | Version inconsistencies | Inherit from `{org}-parent-pom` |
| Business logic in interface file | Upgrades overwrite changes | Separate interface and implementation |
| Monolithic flows | Hard to test and maintain | Break into sub-flows and modules |
| Ignoring layer responsibilities | Tight coupling between systems | Follow SAPI/PAPI/EAPI patterns |
| Downloading Exchange specs unnecessarily | Unwanted files may get committed | Check if RAML already exists before downloading |

## Checklist

### Deployment
- [ ] API published to Exchange after completion
- [ ] Code reviewed following team standards

## Related Documentation

- [Project Structure](./project-structure.md) — directory structure, global config rules, config management, health check
- [API Layer Patterns (SAPI/PAPI/EAPI)](./api-layers.md) — layer-specific implementation patterns, connectivity, best practices
- [Error Handling Standards](./error-handling.md) — error response format, global error handler, try-scope exceptions
- [Logging Guidelines](./logging.md) — standard Mule logger, levels, correlation ID propagation
- [DataWeave Patterns](./dataweave-patterns.md) — DWL folder convention, transformation patterns, external file rules
- [Workflow Details](../workflow/workflow.md) — mule-developer skill step procedures
- [Connectors](../reference/connectors.md) — connector capabilities, XML attributes, and approved versions
- [Connector Examples](../usage/connector.md) — real-world connector patterns

## References

- [Step 3. Develop the API - MuleSoft Documentation](https://docs.mulesoft.com/general/api-led-develop)
- [API Development in Studio - MuleSoft Documentation](https://docs.mulesoft.com/studio/latest/api-development-studio)

---
Last updated: 2026-04-09
Owner: Integration Team
