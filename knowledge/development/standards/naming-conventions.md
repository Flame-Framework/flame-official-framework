# Naming Conventions

## Overview

This document is the **single source of truth** for naming everything in a MuleSoft project in this framework: flows, sub-flows, variables, connector configs, DWL files, property keys, and documentation attributes. Other standards files reference this one — do not redefine naming rules elsewhere.

## Quick Reference

| Element | Convention | Example |
|---|---|---|
| APIKit-generated flow | `{verb}:\{path}:{api-config-name}` | `get:\products:acme-erp-sapi-config` |
| Main flow | `{api-name}-main` | `acme-erp-sapi-main` |
| Implementation flow | `{purpose}-impl-flow` or `implementation-{resource}-{api-name}Flow` | `implementation-products-acme-erp-sapiFlow` |
| Sub-flow | `{purpose}-sub-flow` (kebab-case) | `orders-sub-flow`, `reprocess-errors-sub-flow` |
| Variable | camelCase | `correlationId`, `httpStatus`, `pageNumber` |
| Connector config | `{System}_{Type}_Config` (PascalCase + underscores) | `HTTP_Listener_Config`, `HTTP_Request_Config` |
| API config (APIKit) | `{api-name}-config` (kebab-case) | `acme-orders-papi-config` |
| Property file | `{config-type}-{env}.yaml` | `config-dev.yaml`, `secure-config-prod.yaml` |
| Property key | `dot.notation` (lowercase) | `http.port`, `backend.connection.timeout` |
| DWL filename | camelCase, descriptive | `errorMessageObjectStore.dwl`, `befores3.dwl` |
| Mule XML file | `{api-name}.xml` (main) or `{purpose}.xml` (others) | `acme-erp-sapi.xml`, `error-handler.xml` |

## Flows

### APIKit-Generated Flows

APIKit generates flows from RAML using its own pattern — **do not rename these**:

```
{verb}:\{path}:{content-type}:{api-config-name}
```

Real-world examples:
- `get:\products:acme-erp-sapi-config`
- `post:\orders\(orderId)\items:application\json:acme-orders-papi-config`
- `post:\purchaseOrders\approval:application\json:acme-procurement-eapi-config`

### Main Flow

The HTTP listener entry-point flow:

- **Pattern**: `{api-name}-main`
- **Examples**: `acme-logging-sapi-main`, `acme-erp-sapi-main`, `acme-orders-papi-main`
- **One per project** — handles inbound listener, sets `vars.correlationId`, logs the request, routes to APIkit

### Implementation Flows

The flow that holds the actual business logic for an endpoint:

- **Pattern A** (preferred): `implementation-{resource}-{api-name}Flow`
- **Pattern B**: `{resource}-{verb}-impl-flow`
- **Examples**: `implementation-products-acme-erp-sapiFlow`, `post-po-approvers-impl-flow`

## Sub-flows

- **Pattern**: `{purpose}-sub-flow` or `{purpose}-subflow` (both seen in current code — prefer **`-sub-flow`** for new code)
- **kebab-case**, descriptive of what the sub-flow does
- **Examples**: `orders-sub-flow`, `backend-purchaseOrder-response-validator-subflow`, `reprocess-all-errors-from-os-subflow`
- **NEVER**: `subFlow1`, `helper`, `process`

## Variables

- **camelCase**, no prefix
- **Standard variables** (always named exactly this way):
  - `correlationId` — request tracing ID
  - `httpStatus` — outbound HTTP response status
- **Project-specific variables**: still camelCase, descriptive
  - Examples: `lastExecution`, `pageNumber`, `totalAmount`, `integrationId`, `idLog`, `yearMonth`
- **NEVER**: `v1`, `temp`, `data`, `result`, `x`

## Connector Configurations

- **Pattern**: `{System}_{ConnectorType}_Config` — PascalCase words joined by underscores
- **Examples**: `HTTP_Listener_config`, `Validation_Config`, `Amazon_S3_Configuration`, `HTTP_Request_Config`, `Backend_HTTP_Request_Config`
- **Inconsistency to fix going forward**: some legacy code uses kebab-case (`acme-orders-papi-httpListenerConfig`). For **new code**, use the underscore pattern above.

## API (APIKit) Configurations

- **Pattern**: `{api-name}-config` — kebab-case
- **Examples**: `acme-erp-sapi-config`, `acme-orders-papi-config`, `acme-procurement-eapi-config`
- **One per project**, defined in the main XML

## Property Files

- **Filename pattern**: `{config-type}-{env}.yaml`
  - `config-{env}.yaml` — non-secret properties
  - `secure-config-{env}.yaml` — encrypted secrets
  - `config-common.yaml` — values shared across all envs
- **Allowed envs**: `local`, `dev`, `qa`, `prod`
- **Location**: `src/main/resources/properties/`
- **Examples**: `config-dev.yaml`, `secure-config-prod.yaml`, `config-common.yaml`

## Property Keys

See [configuration-properties.md](./configuration-properties.md) for full rules. Naming summary:

- **Format**: `dot.notation`, lowercase
- **Pattern**: `{system}.{component}.{attribute}`
- **Examples**: `http.port`, `backend.connection.timeout`, `s3.bucket_name`, `acme-purchase-papi.host`
- **Note**: legacy code mixes underscores within segments (`bucket_name`) — keep dot-notation for the **separators** between segments. Within a segment, prefer no separator (e.g., `bucketname`) for new keys.

## DWL Files

See [dataweave-patterns.md](./dataweave-patterns.md) for folder structure. Naming summary:

- **Filename**: camelCase, descriptive of the output it produces
- **Examples**: `errorMessageObjectStore.dwl`, `errorMessageToReprocess.dwl`, `keyError.dwl`, `befores3.dwl`
- **NEVER**: `transform.dwl`, `dwl1.dwl`, `mapping.dwl`

## Mule XML Files

- **Main file**: `{api-name}.xml` — e.g., `acme-erp-sapi.xml`
- **Common files** (under `src/main/mule/common/`): purpose-named — `error-handler.xml`, `reprocess.xml`
- **Global config file** (top-level under `src/main/mule/`): `global.xml`
- **Implementation files** (under `src/main/mule/implementation/`): named after the resource — `orders.xml`, `requisitions-watcher.xml`, `requisitions/requisitions-management.xml`

## Documentation Attributes (`doc:name`, `doc:id`)

- **`doc:id`**: auto-generated UUID by Studio — do not author manually, do not remove
- **`doc:name`**: required on every processor, human-readable description of what the processor does
  - Good: `"Set Logger Context"`, `"Call Backend Products API"`, `"Validate Order Payload"`
  - Bad: `"Transform"`, `"Logger"`, `"HTTP Request"` (the type is already obvious from the element)

## Forbidden Patterns

- Spaces in any identifier (`my flow` → `my-flow`)
- Special characters except `-` and `_` per the conventions above
- Generic names: `flow1`, `subFlow1`, `var1`, `temp`, `data`, `process`
- Ambiguous abbreviations: `proc`, `mgr`, `hdlr` (use `processor`, `manager`, `handler`)
- Mixed conventions in one file (don't mix `mySubFlow` and `my-sub-flow`)

## Naming Checklist

- [ ] Flow name follows APIKit pattern (if RAML-generated) or `{api-name}-main` / `implementation-*` pattern
- [ ] Sub-flow name uses `-sub-flow` suffix and is kebab-case
- [ ] All variables are camelCase; standard variables use exact reserved names
- [ ] Connector configs use `{System}_{Type}_Config` pattern
- [ ] APIKit config uses `{api-name}-config` pattern
- [ ] Property files follow `{config-type}-{env}.yaml` pattern
- [ ] Property keys use `dot.notation`
- [ ] DWL filenames are camelCase and describe what they output
- [ ] Every processor has a meaningful `doc:name`
- [ ] No spaces, generic names, or mixed conventions

## References

- [Configuration Properties](./configuration-properties.md)
- [Logging Standards](./logging.md)
- [DataWeave Patterns](./dataweave-patterns.md)
- [Project Structure](./project-structure.md)

---
Last updated: 2026-04-13
Owner: Integration Team
