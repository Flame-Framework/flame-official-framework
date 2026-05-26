# Exchange Discovery — Reuse Assessment Procedure

Before creating any new API, fragment, data type, or connector configuration, search Anypoint Exchange for existing assets that can be reused or extended.

## When to Run

- **Solution Designer**: before proposing a new API in the architecture
- **RAML Designer**: before creating a new data type or fragment
- **Mule Developer**: before creating a new connector configuration or integration pattern
- **Data Mapping**: before defining a new canonical model

## Step 1: Search

Use `anypoint_search_assets` with:
- `search`: the entity name, domain, or system (e.g., "customer", "SAP", "materials")
- `types`: filter by asset type relevant to the task:
  - `["raml-fragment"]` — when looking for reusable data types or traits
  - `["rest-api"]` — when looking for existing APIs
  - `["connector"]` — when looking for connectors
  - `["template"]` — when looking for implementation templates
  - Omit `types` for a broad search across all asset types

## Step 2: Evaluate Results

For each asset found, use `anypoint_get_asset` to retrieve details (description, version, dependencies).

Assess each result against these criteria:

| Criteria | Reuse as-is | Extend | Build new |
|----------|-------------|--------|-----------|
| **Functional match** | Covers 100% of the requirement | Covers 70%+ with minor additions | Covers less than 70% |
| **Version** | Latest version, actively maintained | Older version but stable | Deprecated or abandoned |
| **Compatibility** | Same API layer and patterns | Different layer but adaptable | Incompatible patterns |
| **Dependencies** | No conflicting dependencies | Minor version alignment needed | Major dependency conflicts |

## Step 3: Inspect Specification (if reuse is viable)

- For APIs: use `anypoint_get_api_endpoints` to get a compact view of available resources and methods
- For fragments: use `anypoint_get_api_specification` to download and inspect the data types
- For connectors: use `anypoint_get_asset` to check supported operations and configuration

## Step 4: Present to User

Present findings in this format:

```
### Exchange Discovery Results

**Search term**: [what was searched]
**Assets found**: [count]

| Asset | Type | Version | Recommendation | Reason |
|-------|------|---------|----------------|--------|
| [name] | [type] | [version] | Reuse / Extend / Skip | [brief reason] |

**Recommendation**: [reuse asset X / no reusable assets found, proceed with new creation]
```

## When No MCP is Available

If the Anypoint MCP server is not configured (`mcp.servers.anypoint` not in `framework.yaml`), skip this procedure and inform the user:

> "Exchange discovery skipped — Anypoint MCP server is not configured. To enable reuse assessment, configure the MCP server in `config/framework.yaml`."
