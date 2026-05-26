# Scaffold — Workflow

> Read this file FIRST. It orchestrates the generation of a new MuleSoft 4 project skeleton. Invoked only by `mule-developer`. On success, emits `SCAFFOLD_OK PROJECT_ROOT=<absolute-path>` and hands control back without user prompt. On failure, emits `SCAFFOLD_ABORTED reason=<...>` and aborts cleanly.

---

## Step 1 — Resolve Inputs (auto-derive everything possible)

Read silently. No user prompts in this step.

### 1a. From the argument
- `PROJECT_NAME` = the `<project-name>` argument passed by `mule-developer`.
- `LAYER` = derive from suffix: `-sapi` → SAPI, `-papi` → PAPI, `-eapi` → EAPI. If suffix is absent or non-standard, mark `LAYER = UNKNOWN`.

### 1b. From `config/framework.yaml`
- `WORKSPACES` = list under `workspaces:`. Expand `~` to the user's home.
- `PROJECT_PREFIX` = `organization.project-prefix` (may be empty).
- `PARENT_POM` = `parent-pom:` block (may be absent → mark `PARENT_POM = NONE`).
- `MCP_ENABLED` = `mcp.servers.anypoint.enabled` (default false if missing).

### 1c. From MCP (if `MCP_ENABLED`)
Run in parallel:
- `mcp__anypoint__anypoint_get_organization` → `HOME_ORG_ID` (used as Maven `<groupId>` for the Mule app and for Exchange asset lookup unless framework.yaml overrides).
- `mcp__anypoint__anypoint_search_assets` with `search=PROJECT_NAME`, `types=["raml"]` → `RAML_HITS`.

### 1d. From Mulesoft Nexus (always)
Fetch the current `<release>` version for each artifact via:
```
https://repository.mulesoft.org/nexus/content/repositories/public/<groupId-path>/<artifactId>/maven-metadata.xml
```
- `mule-apikit-module`
- `mule-http-connector`
- `mule-secure-configuration-property-module`
- `mule-maven-plugin`
- Mule runtime (artifact for `<app.runtime>`)

Store as `VERSIONS = { apikit, http, secure, maven_plugin, runtime }`. If any fetch fails, **STOP and ask the user** (do not guess).

---

## Step 2 — Identify Gaps and Ask the User (single batched question)

Collect all the genuine ambiguities. If any are present, ask them in ONE `AskUserQuestion` call (multi-question form). Skip the question entirely if no ambiguity exists.

| Field | Ambiguity exists when… | Question to ask |
|---|---|---|
| `WORKSPACE` | `len(WORKSPACES) > 1` | "Which workspace should host the project?" — options = each workspace path |
| `LAYER` | `LAYER == UNKNOWN` | "Which API layer is this project?" — options: SAPI / PAPI / EAPI |
| `RAML_SOURCE` | `MCP_ENABLED` is true AND `len(RAML_HITS) > 1` | "Multiple Exchange assets match — which one to use?" — options = each hit's `groupId/assetId/version` |
| `RAML_SOURCE` | `MCP_ENABLED` is true AND `len(RAML_HITS) == 0` | "No RAML found in Exchange for `<PROJECT_NAME>`. Provide a local RAML path, or paste a different Exchange asset (`groupId:assetId:version`)?" — text input |
| `RAML_SOURCE` | `MCP_ENABLED` is false | "MCP is disabled. Provide a local RAML path or Exchange coordinates (`groupId:assetId:version`)?" — text input |

When `len(WORKSPACES) == 1` → use it without asking. When `len(RAML_HITS) == 1` → use it without asking. When `LAYER` is derivable → use it without asking.

After this step, all of the following are set:
- `PROJECT_NAME`, `LAYER`, `WORKSPACE`, `RAML_SOURCE`, `PARENT_POM`, `MCP_ENABLED`, `VERSIONS`, `HOME_ORG_ID`.

Compute `PROJECT_ROOT = <WORKSPACE>/<PROJECT_NAME>`.

---

## Step 3 — Pre-flight Refusals

Stop with `SCAFFOLD_ABORTED reason=...` if any:
- `<WORKSPACE>` directory does not exist → reason: `"workspace path '<WORKSPACE>' does not exist; create it first"`.
- `<PROJECT_ROOT>` already exists (any contents) → reason: `"target folder '<PROJECT_ROOT>' already exists; refusing to overwrite"`.
- Any required `VERSIONS` lookup failed and the user did not supply a value → reason: `"could not look up version for <artifact>"`.

---

## Step 4 — Fetch the RAML

Branch on `RAML_SOURCE`:

### 4a. Exchange asset
- `mcp__anypoint__anypoint_download_asset_project` with `groupId/assetId/version` → downloads ZIP, unpacks to a temp dir.
- Move the RAML files into `<PROJECT_ROOT>/src/main/resources/api/` (created later in Step 6).
- Read the bundled `exchange.json` → extract `dependencies[]` → store as `RAML_TRANSITIVE_DEPS`. (These are Exchange fragments the RAML depends on, e.g., `common-data` — they must be declared in the child pom as `<scope>provided</scope>` deps.)
- `RAML_MAIN_FILE` = the `main` field from the downloaded `exchange.json`.

### 4b. Local file path
- Validate the path exists and is a `.raml` file.
- Copy it (and any sibling files in the same directory tree, excluding `exchange_modules/`, `node_modules/`, `.git/`) into `<PROJECT_ROOT>/src/main/resources/api/`.
- `RAML_TRANSITIVE_DEPS = []` (no Exchange deps unless the user explicitly provides them).
- `RAML_MAIN_FILE` = the basename of the supplied path.

### 4c. Endpoint enumeration
- If `MCP_ENABLED`: call `mcp__anypoint__anypoint_get_api_endpoints` to enumerate `(method, path)` pairs from the RAML.
- Else: parse the local RAML to enumerate them.
- Store as `ENDPOINTS = [{method, path}, ...]`. This drives the stub flow generation in Step 6.

---

## Step 5 — Inspect Parent POM (Option B — Maven-driven inheritance check)

Skip this step if `PARENT_POM = NONE` (no parent configured) — then inline all base deps in Step 6.

When `PARENT_POM` is set:

1. **Create a temp stub pom** at `/tmp/scaffold-stub-<random>/pom.xml`:
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0">
     <modelVersion>4.0.0</modelVersion>
     <parent>
       <groupId>{PARENT_POM.group-id}</groupId>
       <artifactId>{PARENT_POM.artifact-id}</artifactId>
       <version>{PARENT_POM.version}</version>
     </parent>
     <groupId>{HOME_ORG_ID}</groupId>
     <artifactId>{PROJECT_NAME}</artifactId>
     <version>1.0.0</version>
     <packaging>mule-application</packaging>
   </project>
   ```
2. **Run** `mvn -q -f /tmp/scaffold-stub-.../pom.xml help:effective-pom -Doutput=/tmp/scaffold-stub-.../effective.xml`.
3. **On success**: parse `effective.xml`. For each base artifact in `VERSIONS`, check whether it appears under `<dependencies>`, `<dependencyManagement>`, or `<build>/<plugins>`/`<pluginManagement>`. Build the set `INHERITED = { apikit?, http?, secure?, maven_plugin?, runtime? }`.
4. **On failure** (Maven not on PATH / parent resolution failed / network):
   - Log a warning: `"Could not resolve parent POM via Maven — falling back to convention (assuming parent provides all base deps)"`.
   - Set `INHERITED = { all true }` (i.e., child pom won't redeclare anything).
5. **Cleanup**: remove `/tmp/scaffold-stub-...`.

---

## Step 6 — Generate the File Tree

Create all directories and files atomically. Order matters — base structure first, then files that depend on it. If any write fails, **roll back** by removing `<PROJECT_ROOT>` entirely before emitting `SCAFFOLD_ABORTED`.

### 6a. Directory skeleton (per `standards/project-structure.md`)
```
<PROJECT_ROOT>/
├── pom.xml
├── mule-artifact.json
├── .gitignore
├── src/
│   ├── main/
│   │   ├── mule/
│   │   │   ├── common/
│   │   │   │   └── error-handler.xml
│   │   │   ├── implementation/    (empty — ND-x fills it)
│   │   │   ├── global.xml
│   │   │   ├── health.xml
│   │   │   └── <PROJECT_NAME>.xml
│   │   └── resources/
│   │       ├── api/               (RAML files placed in Step 4)
│   │       ├── dwl/
│   │       │   ├── payloads/      (empty — ND-x fills it)
│   │       │   ├── vars/          (empty — ND-x fills it)
│   │       │   └── error/         (empty — ND-x fills it)
│   │       └── properties/
│   │           ├── config-common.yaml
│   │           ├── config-local.yaml
│   │           ├── config-dev.yaml
│   │           ├── config-qa.yaml
│   │           ├── config-prod.yaml
│   │           ├── secure-config-local.yaml
│   │           ├── secure-config-dev.yaml
│   │           ├── secure-config-qa.yaml
│   │           └── secure-config-prod.yaml
│   └── test/munit/                (placeholder, empty)
```

### 6b. `pom.xml`
- Declare `<parent>` block if `PARENT_POM` is set.
- Declare base deps only when `INHERITED` says they aren't inherited (i.e., for each of `apikit`, `http`, `secure`, `maven_plugin`, `runtime`: declare if `!INHERITED.X`, using the looked-up version from `VERSIONS`).
- Declare the RAML asset Exchange dep (the project's own RAML) with `<scope>provided</scope>` if `RAML_SOURCE` was Exchange.
- Declare each entry from `RAML_TRANSITIVE_DEPS` as a `<scope>provided</scope>` dep.
- Set `<groupId>` = `HOME_ORG_ID`, `<artifactId>` = `PROJECT_NAME`, `<version>` = `1.0.0`, `<packaging>mule-application`.
- **Required `<properties>` block** (per `standards/api-development-best-practices.md`):
  - `<api.layer>` set to the full layer name — map `LAYER` → `"System API"` / `"Process API"` / `"Experience API"`
  - `<project.name>${project.artifactId}</project.name>` (verbatim — uses Maven variable expansion at build time)
  - `<api.version>v1</api.version>` (default; can be overridden by user later)
  - `<mule.maven.plugin.version>` and `<app.runtime>` with values from `VERSIONS` (skip if inherited)
- Default `<maven.compiler.source>` and `<maven.compiler.target>` to `17` unless inherited.
- Include the Anypoint Exchange Maven repo + Mulesoft public repo declarations only if `PARENT_POM = NONE`.

### 6c. `mule-artifact.json`
```json
{
  "name": "<PROJECT_NAME>",
  "minMuleVersion": "<VERSIONS.runtime>",
  "requiredProduct": "MULE",
  "classLoaderModelLoaderDescriptor": { "id": "mule", "attributes": { "exportedResources": [] } },
  "secureProperties": [],
  "bundleDescriptorLoader": { "id": "mule", "attributes": {} }
}
```

### 6d. `src/main/mule/<PROJECT_NAME>.xml` (main API XML)
- `<apikit:config>` with `name="<PROJECT_NAME>-config"`, `api="<RAML_MAIN_FILE>"`, `outboundHeadersMapName="outboundHeaders"`, `httpStatusVarName="httpStatus"`.
- Main flow `<PROJECT_NAME>-main` per `usage/reference-flows.md` for `LAYER`:
  - `<http:listener>` referencing `HTTP_Listener_config` from `global.xml`, with `<http:response>` + `<http:error-response>` blocks.
  - `<set-variable variableName="correlationId" value="#[attributes.headers.'x-correlation-id' default correlationId]"/>`.
  - `<apikit:router>` with `config-ref="<PROJECT_NAME>-config"`.
  - `<error-handler ref="apiKit-error-handler"/>`.
- One **stub flow per (method, path)** from `ENDPOINTS`:
  - APIkit-generated flow name pattern: `<method>:\<path>:<PROJECT_NAME>-config`.
  - Body: log start (INFO with `correlationId`) → placeholder comment `<!-- ND-x: implementation pending -->` → log end (INFO with `correlationId`).
  - No backend HTTP requests, no transforms — those come in ND-x.

### 6e. `src/main/mule/global.xml`
- `<configuration-properties>` chain: common file + env-specific file with `${mule.env:local}` default.
- `<secure-properties:config>` with Blowfish CBC, file = `properties/secure-config-${mule.env:local}.yaml`, `key="${secure.key}"`.
- `<http:listener-config name="HTTP_Listener_config">` with port = `${http.port}`.
- **No downstream connector configs.** ND-x adds those when needed.

### 6f. `src/main/mule/health.xml` (mandatory `/alive` endpoint)

Per `standards/project-structure.md` (line 141-158), every API must expose `/alive`. Generate verbatim:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:ee="http://www.mulesoft.org/schema/mule/ee/core"
      xsi:schemaLocation="
http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd
http://www.mulesoft.org/schema/mule/ee/core http://www.mulesoft.org/schema/mule/ee/core/current/mule-ee.xsd">

    <flow name="get:\alive:<PROJECT_NAME>-config">
        <ee:transform doc:name="Build /alive Response">
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

</mule>
```

The RAML **must** declare `/alive` as an endpoint for APIkit to route to this flow. If the downloaded/local RAML does not declare it, log a warning in Step 8 and instruct the user to add `/alive` to the RAML before publishing. (Adding it to the RAML is the user's call — the scaffold does not modify the RAML.)

### 6g. `src/main/mule/common/error-handler.xml`
Standard apiKit error handler per `standards/error-handling.md` + `usage/error-handling.md`:
- `<on-error-propagate type="APIKIT:BAD_REQUEST">` → 400
- `<on-error-propagate type="APIKIT:NOT_FOUND">` → 404
- `<on-error-propagate type="APIKIT:METHOD_NOT_ALLOWED">` → 405
- `<on-error-propagate type="APIKIT:UNSUPPORTED_MEDIA_TYPE">` → 415
- `<on-error-propagate type="APIKIT:NOT_ACCEPTABLE">` → 406
- `<on-error-propagate type="ANY">` → 500
- Each block uses external DWL from `dwl/error/message/` for the response payload, sets `vars.httpStatus`, and logs at ERROR level.

### 6h. `src/main/resources/properties/config-common.yaml`
```yaml
http:
  port: "8081"

api:
  name: "<PROJECT_NAME>"
  version: "v1"
```
Plus a sensible default for `secure.key` (left as a placeholder commented note instructing the user to pass it via `-M-Dsecure.key=...` at runtime).

`api.name` and `api.version` are referenced by the `/alive` endpoint in `health.xml`. They are placed in `config-common.yaml` because they are genuinely the same across all environments (per `standards/project-structure.md` line 129 — "only properties truly the same across all environments belong in the common file").

Env files (`config-{local,dev,qa,prod}.yaml`) start empty with just a top-line comment.

Secure env files (`secure-config-{local,dev,qa,prod}.yaml`) start empty with a top-line comment instructing that values must be `![encrypted]` Blowfish CBC format.

### 6i. `.gitignore`
```
target/
.idea/
.vscode/
.DS_Store
*.iml
.classpath
.project
.settings/
exchange_modules/
```

**Note**: only `exchange_modules/` (auto-managed by APIkit / Maven) is ignored from `src/main/resources/api/`. The RAML files themselves — `flame-*-sapi.raml`, `library/`, `dataTypes/`, `traits/` — are **NOT** gitignored; they are first-class project files committed to source control. (The `standards/project-structure.md` line 176 mentions ignoring `datatypes/, library/, traits/` — that appears to be an error in the doc; the actual practice in `flame-pokemon-sapi` keeps them tracked. Logged in `notes/kb-inconsistencies-found-2026-05.md` for the maintainer.)

---

## Step 7 — Pre-flight Validation

Run from `<PROJECT_ROOT>`. All checks must pass.

### 7a. XML well-formedness
For every `.xml` file under `<PROJECT_ROOT>/src/main/mule/` and `<PROJECT_ROOT>/src/main/resources/api/`:
```bash
xmllint --noout <file>
```
On any failure → roll back and `SCAFFOLD_ABORTED reason="generated XML is not well-formed: <file>"`.

### 7b. YAML well-formedness
For every `.yaml` file under `<PROJECT_ROOT>/src/main/resources/properties/`:
```bash
python3 -c "import sys, yaml; yaml.safe_load(open(sys.argv[1]))" <file>
```
On any failure → roll back and `SCAFFOLD_ABORTED reason="generated YAML is invalid: <file>"`.

### 7c. Maven compile
```bash
mvn -q -f <PROJECT_ROOT>/pom.xml compile
```
- On success → continue.
- On failure → capture the Maven error log, **do not roll back** (leave the project for the user to inspect), and `SCAFFOLD_ABORTED reason="mvn compile failed; see log"`. The user can fix the gap (often a missing dep that should have been inherited) and the next mule-developer run will pick up.

### 7d. Mule artifact validation
```bash
python3 -c "import sys, json; json.loads(open(sys.argv[1]).read())" <PROJECT_ROOT>/mule-artifact.json
```
On failure → roll back and `SCAFFOLD_ABORTED reason="mule-artifact.json invalid"`.

---

## Step 8 — Report and Hand Off

On success:

```
SCAFFOLD_OK PROJECT_ROOT=<absolute-path>

Generated files:
  <tree of created files>

Plugin/connector versions (looked up from Mulesoft Nexus):
  mule-apikit-module: <version>
  mule-http-connector: <version>
  mule-secure-configuration-property-module: <version>
  mule-maven-plugin: <version>
  mule runtime: <version>

Parent POM: <coords or 'none'>
Inherited from parent: <list or 'n/a'>
RAML source: <Exchange asset coords or local path>
RAML transitive deps: <list or 'none'>
Endpoints stubbed: <count>

Pre-flight validation:
  [x] XML well-formed
  [x] YAML well-formed
  [x] mvn compile
  [x] mule-artifact.json valid
```

Then return control to `mule-developer` immediately. **No "should I continue?" prompt.** mule-developer parses `PROJECT_ROOT` from the marker line and continues with its Checkpoint Resume Protocol → Step 0 (project-context-builder) → ND-x.

---

## Rollback Semantics

A "rollback" means: `rm -rf <PROJECT_ROOT>` after a write succeeded but a later step failed. The single exception is Step 7c (`mvn compile` failure) — there the project is **kept** so the user can diagnose, but the skill still emits `SCAFFOLD_ABORTED` so mule-developer doesn't proceed.

---

## Knowledge Base Files

Loaded lazily as needed:
- `knowledge/development/standards/project-structure.md` — Step 6 directory skeleton
- `knowledge/development/standards/naming-conventions.md` — Step 6 file/element names
- `knowledge/development/standards/api-layers.md` — Step 6d main flow pattern
- `knowledge/development/usage/reference-flows.md` — Step 6d main flow XML
- `knowledge/development/standards/error-handling.md` + `knowledge/development/usage/error-handling.md` — Step 6f error handler XML
- `knowledge/development/standards/configuration-properties.md` — Step 6e + 6g properties layout
- `knowledge/development/standards/logging.md` — Step 6d logger placement in stubs
- `knowledge/development/standards/api-development-best-practices.md` — Step 6b pom conventions (parent inheritance, repos, etc.)
