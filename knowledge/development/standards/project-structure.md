# MuleSoft Project Structure

## Overview

This document defines the standard project structure for all MuleSoft APIs in this framework. Every new project must follow this structure, based on the `{org}-template-api` template.

## Directory Structure

```
src/
├── main/
│   ├── mule/
│   │   ├── common/
│   │   │   └── error-handler.xml      # Global error handlers
│   │   ├── implementation/
│   │   │   └── {feature}.xml          # Business logic by feature
│   │   ├── global.xml                 # Global configurations (ALL configs here)
│   │   ├── health.xml                 # Health check endpoint (/alive)
│   │   └── {api-name}.xml             # APIkit router (auto-generated)
│   └── resources/
│       ├── api/
│       │   └── {api-name}.raml        # API specification
│       ├── dwl/
│       │   ├── payloads/              # DWL for ee:set-payload
│       │   │   └── {entity}/          # Grouped by entity
│       │   ├── vars/                  # DWL for ee:set-variable
│       │   │   └── {entity}/          # Grouped by entity/context
│       │   └── error/                 # Error handling DWL
│       │       ├── message/           # Error response payloads
│       │       ├── statusCodes/       # HTTP status code mappings
│       │       └── reprocess/         # ObjectStore error handling
│       └── properties/
│           ├── config-common.yaml      # Shared across environments
│           ├── config-local.yaml      # Local development
│           ├── config-dev.yaml        # Development environment
│           ├── config-prod.yaml       # Production environment
│           ├── secure-config-local.yaml
│           ├── secure-config-dev.yaml
│           └── secure-config-prod.yaml
```

## Mule XML File Organization

### `global.xml` (top-level)

All global configurations live at `src/main/mule/global.xml`, **not** inside `common/`. Every config element must live in this single file — nowhere else.

### `common/` — Shared Infrastructure

| File | Purpose | Rule |
|------|---------|------|
| `error-handler.xml` | Global error handlers | All standard error handling centralized here |

### `implementation/` — Business Logic

- One XML file per feature/entity (e.g., `orders.xml`, `materials.xml`)
- Contains sub-flows with the actual business logic
- Never put global configs or error handlers here

### Separation of Concerns

The auto-generated interface file defines routes; implementation files contain business logic:

```xml
<!-- {api-name}.xml - Auto-generated, do not modify -->
<flow name="api-main">
    <http:listener config-ref="HTTP_Listener_Config" path="/api/*"/>
    <apikit:router config-ref="api-config"/>
</flow>

<flow name="get:\orders:api-config">
    <flow-ref name="get-orders-implementation"/>
</flow>
```

```xml
<!-- implementation/orders.xml - Business logic here -->
<sub-flow name="get-orders-implementation">
    <!-- Transformations, connector calls, business logic -->
</sub-flow>
```

### Root Files

| File | Purpose |
|------|---------|
| `{api-name}.xml` | APIkit router — auto-generated, do not modify |
| `health.xml` | Health check endpoint (`/alive`) |

### Scheduler Flows

- Scheduler flows go in a **dedicated `scheduler.xml`** file, not in the main API XML

## Global Configuration Location

ALL global configurations MUST be inside `global.xml`. The following config element types must only exist in `global.xml`, never in other XML files.

**Naming convention:** Use PascalCase with underscores for config names (e.g., `HTTP_Listener_Config`, `Secure_Properties_Config`). Each word starts with a capital letter.

- `http:listener-config`, `http:request-config`
- `db:config`, `db:generic-connection`
- `os:object-store`
- `vm:config`
- `jms:config`
- `email:smtp-config`
- `secure-properties:config`
- `configuration-properties`
- `apikit:config`
- `tls:context`
- `api-gateway:autodiscovery`
- Any custom connector `*:config`

## Configuration Management

### Property Files

- Use property files for environment-specific configurations
- Never hard-code credentials or URLs
- Use secure properties for sensitive data (Blowfish encryption)
- **Only add properties that are actually used** — keep configuration files clean and minimal
- **Unused properties**: Every property key in `.yaml` files must be referenced somewhere in the project. Remove any property that is not actually used

### Property Loading Order

1. `config-common.yaml` — Base configuration (shared values only)
2. `config-${env}.yaml` — Environment overrides (local, dev, prod)
3. `secure-config-${env}.yaml` — Encrypted credentials

**Do NOT put everything in `config-common.yaml`** — only properties that are truly the same across all environments belong in the common file. See [Configuration Properties](./configuration-properties.md) for what belongs in each file.

### Environment Configuration Consistency

- Data structures for Local, Dev, and Prod environments MUST be identical — only values may differ
- Only add properties that are actually used — remove any property not referenced in the project
- If a value is hardcoded in the XML (e.g., `method="GET"`), don't duplicate it in config files

For detailed environment conventions (common vs env-specific split, secure property format, CloudHub DNS patterns, adding new properties), see [Configuration Properties](./configuration-properties.md).

## Health Check Endpoint

Every API must implement the `/alive` endpoint:

```xml
<flow name="get:\alive:api-config">
    <ee:transform>
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    status: "alive",
    apiName: p('api.name'),
    apiVersion: p('api.version'),
    timestamp: now()
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>
```

## Project Structure Checklist

- [ ] Cloned from `{org}-template-api`
- [ ] Parent POM configured (`{org}-parent-pom`)
- [ ] API specification imported from Exchange
- [ ] Interface and implementation in separate XML files
- [ ] Health check endpoint (/alive) implemented
- [ ] All global configs in global.xml only
- [ ] All error handling in error-handler.xml only
- [ ] Scheduler flows in dedicated scheduler.xml
- [ ] DWL files organized in `dwl/payloads/`, `dwl/vars/`, `dwl/error/` subfolders
- [ ] Property files configured for all environments (local, dev, prod)
- [ ] Environment config structure identical across all envs
- [ ] No hard-coded credentials or URLs
- [ ] Secure properties configured for sensitive data (Blowfish encryption)
- [ ] Exchange downloaded files are gitignored (exchange_modules/, datatypes/, library/, traits/)
- [ ] Config files only contain properties that are actually used

## References

- [API Development Best Practices](./api-development-best-practices.md)
- [DataWeave Patterns (DWL folder convention)](./dataweave-patterns.md)
- [Configuration Properties](./configuration-properties.md)
- [Error Handling Standards](./error-handling.md)

---
Last updated: 2026-04-09
Owner: Integration Team
