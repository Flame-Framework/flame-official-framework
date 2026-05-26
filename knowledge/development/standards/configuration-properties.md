# Configuration Properties

## Overview

Standard for **all** configuration in MuleSoft projects in this framework: file layout, YAML structure per environment, secure properties (Blowfish + CBC encryption), CloudHub 2.0 internal DNS patterns, externalization rules, and how to add a new property.

For property *naming* details (`dot.notation` keys, etc.) see [naming-conventions.md](./naming-conventions.md).

---

## File Layout

Every project has these files under `src/main/resources/properties/`:

```
properties/
├── config-common.yaml          # Shared across all environments (universal across every API project in the framework)
├── config-local.yaml           # Local development
├── config-dev.yaml             # CloudHub DEV
├── config-qa.yaml              # CloudHub QA
├── config-prod.yaml            # CloudHub PROD
├── secure-config-local.yaml    # Encrypted secrets per env
├── secure-config-dev.yaml
├── secure-config-qa.yaml
└── secure-config-prod.yaml
```

**Allowed envs**: `local`, `dev`, `qa`, `prod`.

## Loading in `mule-config.xml` / `global.xml`

Standard configuration block — required in every project:

```xml
<configuration-properties file="properties/config-common.yaml" />
<configuration-properties file="properties/config-${mule.env}.yaml" />

<secure-properties:config name="Secure_Properties_Config"
        doc:name="Secure Properties Config"
        file="properties/secure-config-${mule.env}.yaml"
        key="${encrypt.key}"
        fileLevelEncryption="false">
    <secure-properties:encrypt algorithm="Blowfish" mode="CBC" />
</secure-properties:config>
```

- `${mule.env}` is set per deployment target (`dev`, `qa`, `prod`)
- `${encrypt.key}` is **never** committed — passed via `-Dencrypt.key={key}` runtime arg or CloudHub secure property

---

## `config-common.yaml` — Shared Properties

Values that NEVER change between environments. Typically:

```yaml
# API metadata (populated by Maven at build time)
api:
  groupId: "${api.groupId}"
  artifactId: "${project.name}"
  version: "${project.version}"
  spec: "resource::${api.groupId}:${api.artifactId}:${api.raml.version}:raml:zip:api.raml"

# JSON Logger
json:
  logger:
    application:
      name: "${project.name}"
    disabled:
      fields: ""
    masked:
      fields: ""

# Project-wide constants (shared lists, static mappings, etc.)
```

---

## `config-{env}.yaml` — Environment-Specific Properties

Properties that DIFFER between environments. **All env files MUST have the same property structure** — only values change.

### Properties that typically differ

| Property Category | Local | Dev | QA | Prod |
|---|---|---|---|---|
| `api.id` | Local API ID | Dev API ID | QA API ID | Prod API ID |
| Microservice hosts | `localhost` | `dev-{org}-{api}-{id}.internal-{dev-space}.irl-e1.cloudhub.io` | `qa-{org}-{api}-{id}.internal-{qa-space}.irl-e1.cloudhub.io` | `{org}-{api}-{id}.internal-{prod-space}.irl-e1.cloudhub.io` |
| Microservice protocol | `HTTP` or `HTTPS` | `HTTPS` | `HTTPS` | `HTTPS` |
| Microservice port | `8081` (varies) | `443` | `443` | `443` |
| MQTT clientId | `mulesoft-local` | `mulesoft-dev` | `mulesoft-qa` | `mulesoft-prod` |
| MQTT topics | `DevAcme/...` | `DevAcme/...` | `DevAcme/...` | `Acme/...` |
| Timeouts | Longer (for debugging) | Standard | Standard | Standard |

### Standard Template

```yaml
# API identification
api:
  id: "{environment-specific-api-id}"

# HTTP Listener
http:
  listener:
    port: "8081"
    protocol: "HTTPS"

# Downstream microservice configuration
# Repeat this block for EACH downstream service
{service-name}:
  host: "{env-specific-host}"
  port: "443"
  basePath: "/api"
  protocol: "HTTPS"
  path:
    {resource}: "/v1/{resource}"
  timeout: 30000
  retries: 3

# MQTT (only for broker-related services)
mqtt:
  broker:
    serverUri: "{env-specific-broker-uri}"
    clientId: "mulesoft-{env}"
  topic:
    {name}: "{EnvPrefix}/{Country}/{Domain}/{Topic}"
```

### Real-World Examples

**`config-dev.yaml` (SAPI — `acme-logging-sapi`):**

```yaml
apiId: "12345678"
http:
  host: "localhost"
  port: ""
  read:
    timeout: "30000"
  idle:
    timeout: "30000"
s3:
  region: "eu-west-1"
  bucket_name: "acme-integration-logs"
```

**`config-dev.yaml` (EAPI — `acme-procurement-eapi`):**

```yaml
api:
  id: "87654321"
http:
  port: "8081"
  protocol: "HTTPS"
acme-purchase-papi:
  host: "dev-acme-purchase-papi-{random}.internal-{dev-space}.irl-e1.cloudhub.io"
  basePath: "/api"
```

---

## What MUST Be Externalized

| Category | Examples |
|---|---|
| Hosts & ports | `http.host`, `http.port`, `backend.host`, `s3.region` |
| Timeouts | `http.read.timeout`, `backend.connection.timeout` |
| URLs / base paths | `acme-purchase-papi.host`, `acme-purchase-papi.basePath` |
| Credentials | `backend.auth.user`, `backend.auth.password` (secure) |
| API IDs (autodiscovery) | `api.id`, `apiId` |
| Retry / batch sizes | `reprocess.max.attempts`, `batch.block.size` |
| Feature flags | `feature.reprocess.enabled` |
| Resource names | `s3.bucket_name`, `objectstore.errors.name` |
| Encryption keys | `encrypt.key` (passed externally, never in YAML) |

## What MUST NOT Be Externalized

| Category | Reason |
|---|---|
| HTTP method names (`GET`, `POST`) | Part of API contract — change = code change |
| RAML-defined enum values (status codes, fixed strings) | Schema-bound |
| DataWeave-internal constants | Part of transformation logic |
| Flow/sub-flow names | Identifiers, not config |
| Hardcoded business rules tied to RAML schema | Schema enforces the rule |

---

## Naming Keys

- **Format**: `dot.notation`, lowercase
- **Pattern**: `{system-or-area}.{component}.{attribute}`
- **Examples**:
  - `http.port`
  - `http.read.timeout`
  - `backend.connection.host`
  - `acme-purchase-papi.host`
  - `s3.region`
  - `reprocess.max.attempts`
- **Keep keys stable across environments** — only the **value** changes per env file. Never have a key in `config-dev.yaml` that doesn't exist in `config-prod.yaml`.

See [naming-conventions.md](./naming-conventions.md) for the broader naming policy.

---

## Secure Properties

### Encryption

Secrets are encrypted with **Blowfish + CBC mode** using `mule-secure-configuration-property-module`.

**To encrypt a value** (Anypoint Studio Secure Properties Tool or CLI):

```bash
java -cp secure-properties-tool.jar \
  com.mulesoft.tools.SecurePropertiesTool \
  string encrypt Blowfish CBC ${encrypt.key} "myPlaintextSecret"
```

The output is wrapped with `![ ... ]` and stored in `secure-config-{env}.yaml`:

```yaml
{your-client-id}:
  client: "![encrypted-client-id]"
  secret: "![encrypted-client-secret]"

# Service-specific credentials
{service-name}:
  username: "![encrypted-username]"
  password: "![encrypted-password]"
```

### Referencing Encrypted Values

In XML / property references:

```xml
<!-- Plain property -->
<http:listener-connection host="${http.host}" port="${http.port}" />

<!-- Secure property -->
<http:request-config name="Backend_HTTP_Config">
    <http:request-connection host="${backend.host}" port="${backend.port}">
        <http:authentication>
            <http:basic-authentication username="${backend.auth.user}" password="${secure::backend.auth.password}" />
        </http:authentication>
    </http:request-connection>
</http:request-config>
```

The `secure::` prefix is **required** for any value defined in a `secure-config-*.yaml`.

### Secure Properties Rules

1. **NEVER store plaintext credentials** — all secrets must use `![...]` format
2. **Local and Dev typically share encryption key** — same encrypted values work for both
3. **QA / Prod use different encryption keys** — encrypted values differ
4. **Encryption key set via runtime property** `-Dencrypt.key={key}`
5. **Do NOT commit encryption keys** — managed via CloudHub secure properties
6. **When adding a new secure property**: ask the user for the encrypted value — NEVER generate encryption yourself

---

## CloudHub 2.0 Internal DNS Pattern

When configuring downstream services on CloudHub 2.0, use the internal DNS pattern (faster + more secure than public URLs):

```
Dev:  {env}-{org}-{api-name}-ref-{random-id}.internal-{space-id}.irl-e1.cloudhub.io
Prod: {org}-{api-name}-ref-{random-id}.internal-{space-id}.irl-e1.cloudhub.io
```

Real example (dev): `dev-acme-backend-sapi-ref-{random}.internal-{dev-space}.irl-e1.cloudhub.io`

```yaml
# config-dev.yaml
backend-sapi:
  host: "dev-acme-backend-sapi-ref-{random}.internal-{dev-space}.irl-e1.cloudhub.io"
  port: "443"
  basePath: "/api"
  protocol: "HTTPS"

# config-prod.yaml
backend-sapi:
  host: "acme-backend-sapi-ref-{random}.internal-{prod-space}.irl-e1.cloudhub.io"
  port: "443"
  basePath: "/api"
  protocol: "HTTPS"
```

### Standard space-id suffixes

| Env | Internal DNS suffix |
|---|---|
| Dev | `internal-{dev-space-id}.irl-e1.cloudhub.io` |
| Prod | `internal-{prod-space-id}.irl-e1.cloudhub.io` |

---

## Defaults Policy

**Do NOT provide property defaults in code.** If a required property is missing, the application MUST fail to start — silent fallbacks hide misconfiguration.

```xml
<!-- ❌ DON'T -->
<http:listener-connection host="${http.host}" port="${http.port:8081}" />

<!-- ✅ DO — fail if http.port is missing -->
<http:listener-connection host="${http.host}" port="${http.port}" />
```

The only exception: optional feature flags can default to `false`.

---

## Avoid Unused Configuration Fields

Every property key in YAML files MUST be referenced somewhere in the project. If a value is hardcoded in the XML (e.g., `method="GET"`), don't duplicate it in config files.

```yaml
# ❌ INCORRECT — method is not used anywhere in the code
endpoint:
  path: "/resource"
  method: "GET"      # remove

# ✅ CORRECT — only properties actually referenced
endpoint:
  path: "/resource"
```

---

## Adding a New Configuration Property

When a new downstream service or feature requires configuration:

1. **Add the property block to ALL env files** (`config-{local,dev,qa,prod}.yaml`) with the same YAML key structure
2. **For sensitive values, add to ALL `secure-config-*.yaml` files** with `![...]` format (ask the user for the encrypted value)
3. **Reference in flows** using `${property.path}` for normal props or `${secure::property.path}` for secure props
4. **Never create env-specific logic in flows** — the config files handle env differences
5. **Verify** the same key set exists across all env files (no env having keys others don't)

---

## Anti-Patterns

| ❌ Anti-Pattern | ✅ Correct |
|---|---|
| Hardcoding URLs / hosts / ports / timeouts in flow XML | Externalize to `config-{env}.yaml` |
| Plaintext credentials in any YAML | Encrypted with Blowfish + CBC, `![...]` format, in `secure-config-{env}.yaml` |
| Different key structures per env (key in `dev` missing in `prod`) | Same key set in every env file |
| `${property:default}` in property references | No defaults — fail fast on missing config |
| Adding env-specific `if` logic in flows | Push the difference into the YAML, keep flows env-agnostic |
| Committing `${encrypt.key}` value | Pass via runtime arg or CloudHub secure property |
| Generating encryption yourself when adding a secret | Ask the user for the pre-encrypted `![...]` value |

---

## Checklist

- [ ] All four env files exist (`config-common.yaml` + `config-{local,dev,qa,prod}.yaml`)
- [ ] All four secure-config files exist (`secure-config-{local,dev,qa,prod}.yaml`)
- [ ] Same key set across all env files (no missing keys)
- [ ] All keys use `dot.notation`, lowercase
- [ ] All hosts, ports, timeouts, URLs, credentials externalized — no hardcoded values in flow XML
- [ ] All secrets use `![...]` format (Blowfish + CBC encrypted)
- [ ] All secret references use `${secure::...}` prefix
- [ ] No defaults in property references (except optional feature flags)
- [ ] CloudHub internal DNS used for downstream services (not public URLs)
- [ ] No unused property keys
- [ ] `${encrypt.key}` NOT committed to repo
- [ ] `mule-config.xml` / `global.xml` loads `config-common.yaml` + `config-${mule.env}.yaml` + `secure-config-${mule.env}.yaml`
- [ ] `<secure-properties:encrypt algorithm="Blowfish" mode="CBC"/>` configured

---

## References

- [Naming Conventions](./naming-conventions.md) — property key format
- [Logging Standards](./logging.md) — log level configuration via `log4j2.xml`
- [Pre-flight Validation](../workflow/pre-flight-validation.md) — automated property checks (P1–P7)

---
Last updated: 2026-04-13
Owner: Integration Team
