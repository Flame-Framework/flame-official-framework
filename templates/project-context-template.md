---
schema-version: 1
generated-by: flame/project-context-builder@<framework-version>
generated-on: <YYYY-MM-DD>
last-confirmed-on: <YYYY-MM-DD>
status: complete
project: <artifact-id>
artifact-id: <artifact-id>
group-id: <group-id>
app-version: <version>
raml-artifact: <raml-artifact-or-empty>
raml-version: <raml-version-or-empty>
layer: <SAPI|PAPI|EAPI|Broker|Unknown>
layer-code: <sys|prc|exp|broker|unknown>
mule-runtime: <e.g. 4.6.8+>
java: "<e.g. 17>"
parent-pom: <artifact-id>:<version>
repo: <git-remote-url>
---

# Project Context — <project>

> Authoritative project-specific facts, conventions, and approved deviations. Generated and maintained by the FLAME `project-context-builder` skill. Do **not** edit manually — manual edits invalidate deviation signatures and may be ignored or flagged by code-review.

## Business Context

<!-- Include this section ONLY if the project has business rules not visible in code (multi-tenant bindings, special domain logic, etc.). Omit entirely if not applicable. -->

## External Constraints

- <fact imposed from outside the project>
- <...>

## Integrations

| System | Direction | Protocol | Connector | Config Name | Auth | Timeout (ms) |
|---|---|---|---|---|---|---|
| <name> | inbound\|outbound | HTTPS\|HTTP\|JMS\|SFTP\|... | <e.g. http:request-config, db:config> | <Mule config name as declared in global.xml> | <method> | <ms> |

## Related APIs

- <name and short purpose>
- <...>

## Domain Entities

- <entity>
- <...>

## Implementation Footprint

A feature-by-feature snapshot of what infrastructure and patterns the project uses. Captured by `project-context-builder` from the code; confirmed with the user.

| Feature | Used | Notes |
|---|---|---|
| Standard error handler | <yes\|no> | <e.g. apikit-error-handler reference> |
| Reprocess via ObjectStore | <yes\|no> | <pattern / object-store name if yes> |
| Scheduler error handling | <yes\|no> | |
| Broker email notification | <yes\|no> | |
| Custom error types | <yes\|no> | <comma-separated list of types if yes> |
| Scatter-Gather | <yes\|no> | |
| Parallel-Foreach | <yes\|no> | |
| Response Caching | <yes\|no> | |
| Scheduler | <yes\|no> | <cron expression if yes> |
| Rate Limiting | <yes\|no> | |

## Project Conventions

_None._

<!--
When conventions exist, replace `_None._` with one entry per convention:

### <short label>

```yaml
category: <DataWeave | Flow | Config | Naming | Connector | Logging | Domain>
description: <what the convention is>
evidence: <file:line refs or summary>
enforcement: advisory
```

Project Conventions are read by `mule-developer` when generating new code so it follows them.
`code-review` does NOT enforce them — they emit no findings.
-->

## Documented Deviations

_None._

<!--
When deviations exist, replace `_None._` with one entry per deviation:

### DEV-001 — <short title>

```yaml
rule-violated: <track>/<file>#<anchor>
scope:
  files: [<glob>]
  flows: [<flow-name>]
  config-keys: []
justification: |
  Two-or-more-sentence explanation of WHY this deviation is acceptable.
  Must reference the business or technical reason.
proof-of-finding:
  file: <relative-path>
  lines: "<start>-<end>"
  snippet-hash: <sha256 hex>
approved-by: "<Name> (<TICKET-REF>)"
approved-on: <YYYY-MM-DD>
expires: <YYYY-MM-DD>
severity-cap: acknowledged
builder-signature: <hex token>
review-status: active
```
-->

## Open Issues

_None._

<!--
When open issues exist, one entry per:

### OI-001 — <short title>

```yaml
rule-violated: <track>/<file>#<anchor>
scope: <files / flows>
observed-on: <YYYY-MM-DD>
notes: |
  What the agent saw and why it could not be promoted to a deviation.
decision-required: code-fix
```
-->

## Change Log

| Date | Origin | Summary |
|---|---|---|
| <YYYY-MM-DD> | project-context-builder | Initial generation |
