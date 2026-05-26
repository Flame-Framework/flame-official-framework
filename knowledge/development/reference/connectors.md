# Connectors Reference

Quick reference for connector capabilities, attributes, and approved versions. For XML examples, see [Connector Examples](../usage/connector.md).

---

## Approved Versions

**Always use these exact versions** when adding dependencies to `pom.xml`. Do not upgrade or downgrade without tech lead approval.

### Parent POM Dependencies (inherited by all projects)

These come from `{org}-parent-pom`. **Do NOT add these to child pom.xml** — they are inherited automatically.

| Connector | groupId | artifactId | Version |
|-----------|---------|------------|---------|
| HTTP Connector | org.mule.connectors | mule-http-connector | 1.10.4 |
| APIkit Module | org.mule.modules | mule-apikit-module | 1.11.1 |
| Secure Properties | com.mulesoft.modules | mule-secure-configuration-property-module | 1.2.7 |
| MUnit Runner | com.mulesoft.munit | munit-runner | 3.5.0 |
| MUnit Tools | com.mulesoft.munit | munit-tools | 3.5.0 |

### Project-Level Dependencies (add to child pom.xml when needed)

Only add these when the project actually uses the connector.

| Connector | groupId | artifactId | Version | Used In |
|-----------|---------|------------|---------|---------|
| ObjectStore | org.mule.connectors | mule-objectstore-connector | 1.2.3 | acme-orders-papi, acme-erp-sapi |
| VM Connector | org.mule.connectors | mule-vm-connector | 2.0.1 | acme-orders-papi |
| Validation Module | org.mule.modules | mule-validation-module | 2.0.7 | acme-orders-papi, acme-erp-sapi |
| SAP Connector | com.mulesoft.connectors | mule-sap-connector | 5.9.13 | acme-erp-sapi |
| MQTT3 Connector | com.mulesoft.connectors | mule4-mqtt3-connector | 1.0.6 | acme-broker-sapi |
| Paho MQTT Client | org.eclipse.paho | org.eclipse.paho.client.mqttv3 | 1.2.5 | acme-broker-sapi |

### pom.xml Dependency Template

```xml
<dependency>
    <groupId>{groupId}</groupId>
    <artifactId>{artifactId}</artifactId>
    <version>{version}</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

> `<classifier>mule-plugin</classifier>` is required for all MuleSoft connectors/modules. Omit it only for plain Java libraries like Paho MQTT Client.

### Version Notes

- **ObjectStore**: v1.2.2 and v1.2.3 both exist in the project. Prefer **1.2.3** for new projects
- **Validation Module**: v2.0.6 and v2.0.7 both exist. Prefer **2.0.7** for new projects
- **SAP Connector**: Enterprise connector — requires MuleSoft EE license
- **MQTT3**: Only for broker integrations — do not use for HTTP-based APIs

---

## HTTP Connector

**Namespace prefix:** `http:`

### Listener Attributes

| Attribute | Description | Example |
|-----------|-------------|---------|
| `host` | Server address | `${http.host}` or `0.0.0.0` |
| `port` | Port number | `${http.port}` or `8081` |
| `protocol` | HTTP or HTTPS | `HTTP` |
| `path` | Listener path | `/api/*` |
| `allowedMethods` | Accepted HTTP verbs | `GET,POST,PUT,PATCH,DELETE` |

### Request Attributes

| Attribute | Description | Values |
|-----------|-------------|--------|
| `method` | HTTP verb | GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD |
| `path` | Target endpoint path | Supports `{placeholder}` for URI params |
| `config-ref` | Reference to global config | Required |
| `target` | Store response in variable instead of payload | Variable name (e.g., `backendResponse`) |
| `targetValue` | What to store in target variable | `#[payload]` or `#[message]` |
| `requestStreamingMode` | Streaming behavior | `AUTO` (default), `ALWAYS`, `NEVER` |
| `sendCorrelationId` | Propagate correlation ID | `ALWAYS`, `AUTO`, `NEVER` |

### Authentication Types

| Type | Notes |
|------|-------|
| **Basic Auth** | Use `preemptive="true"` to avoid extra round-trip |
| **Digest Auth** | Challenge-response based |
| **NTLM Auth** | Requires `domain` attribute |
| **Bearer Token** | Set via headers: `'Authorization': 'Bearer ' ++ vars.authToken` |
| **Client Credentials** | Custom sub-flow that POSTs to token endpoint |

### Response Attributes (available after request)

| Expression | Description |
|------------|-------------|
| `attributes.statusCode` | HTTP status code (200, 404, etc.) |
| `attributes.headers` | Response headers map |
| `attributes.reasonPhrase` | Status reason (e.g., "OK", "Not Found") |

### Response Validator

Use `<http:success-status-code-validator values="200..299"/>` to auto-raise errors on non-2xx responses.

---

## ObjectStore Connector

**Namespace prefix:** `os:`

Provides persistent key-value storage for error persistence, retry patterns, caching, and idempotency checks.

### Configuration Attributes

| Attribute | Description | Default |
|-----------|-------------|---------|
| `persistent` | Survives restart | `false` |
| `maxEntries` | Max entries before eviction | `unlimited` |
| `entryTtl` | Time-to-live for each entry | `unlimited` |
| `entryTtlUnit` | TTL unit | `MILLISECONDS` |
| `expirationInterval` | How often to check expired entries | `1` |
| `expirationIntervalUnit` | Expiration check unit | `MINUTES` |

### Operations

| Operation | Key Attributes |
|-----------|---------------|
| `os:store` | `key`, `value`, `failIfPresent` (default true), `failOnNullValue` (default true) |
| `os:retrieve` | `key`, `target` (store result in variable) |
| `os:retrieve-all-keys` | `target` (returns list of all keys) |
| `os:remove` | `key` |
| `os:contains` | `key`, `target` (boolean result) |

---

## VM Connector

**Namespace prefix:** `vm:`

In-memory messaging for asynchronous processing, flow decoupling, and load distribution.

### Queue Types

| Queue Type | Description | Use When |
|-----------|-------------|----------|
| `TRANSIENT` | In-memory, faster, lost on restart | Non-critical async processing |
| `PERSISTENT` | Disk-backed, survives restart | Critical messages (**not available on CloudHub 2.0**) |

### Operations

| Operation | Description |
|-----------|-------------|
| `vm:publish` | Fire-and-forget message to queue |
| `vm:consume` | Pull message from queue (with timeout) |
| `vm:listener` | Trigger flow on message arrival |

---

## Database Connector

**Namespace prefix:** `db:`

**Supported databases:** MySQL, Oracle, Microsoft SQL Server, Derby, Generic JDBC

### Operations

| Operation | Description |
|-----------|-------------|
| `db:select` | Query with parameterized SQL |
| `db:insert` | Insert with parameterized SQL |
| `db:update` | Update with parameterized SQL |
| `db:delete` | Delete with parameterized SQL |
| `db:stored-procedure` | Call stored procedure with input/output params |

> **Always use parameterized queries** (`:paramName`) — never concatenate values into SQL strings.

---

## Logger Component

**Element:** `<logger>` (core Mule namespace, no prefix needed)

Standard Mule logger used for all logging. See [Logging Guidelines](../standards/logging.md) for level and message standards.

```xml
<logger level="INFO" message="Order retrieval started" doc:name="Log start"/>
```

| Attribute | When |
|-----------|------|
| `level="INFO"` | Flow start/end, key business milestones |
| `level="DEBUG"` | Troubleshooting detail (request/response context, variable state) |
| `level="WARN"` | Degraded operation, retry persistence |
| `level="ERROR"` | Inside `<on-error-*>` scopes |

---

## Scheduler

Triggers flows on a time-based schedule. Used for batch processing and periodic jobs.

| Attribute | Description | Example |
|-----------|-------------|---------|
| `expression` | Cron expression | `0 0 6 * * ?` (daily at 6am) |
| `timeZone` | Timezone | `UTC` |
| `frequency` | Fixed frequency interval | `30` |
| `timeUnit` | Frequency unit | `MINUTES` |

> **Important:** Scheduler flows must go in a dedicated `scheduler.xml` file, not in the main API XML.

---

## Secure Properties Module

Reads encrypted property values from YAML files. Uses Blowfish algorithm.

### Referencing Secure Properties

| Context | Syntax |
|---------|--------|
| XML attributes | `${secure::target.password}` |
| DataWeave | `Mule::p('secure::target.password')` |

> All credential values in secure YAML files must be encrypted: `![encryptedValue]`

---

## SAP Connector

**Namespace prefix:** `sap:`

Enterprise connector for SAP RFC (Remote Function Call) integration. Requires MuleSoft EE license.

### Connection Attributes

| Attribute | Description | Example |
|-----------|-------------|---------|
| `username` | SAP username | `${secure::sap.user}` |
| `password` | SAP password | `${secure::sap.password}` |
| `systemNumber` | SAP system number | `00` |
| `client` | SAP client ID | `${sap.clientId}` |
| `applicationServerHost` | SAP server hostname | `${sap.host}` |
| `language` | SAP language (optional) | `EN` |

### Operations

| Operation | Description | Key Attributes |
|-----------|-------------|----------------|
| `sap:sync-rfc` | Execute synchronous RFC call | `key` (RFC function name), `config-ref` |
| `sap:async-rfc` | Execute asynchronous RFC call | `key`, `config-ref` |

### Rules

- SAP credentials MUST be in secure config (`${secure::sap.user}`, `${secure::sap.password}`)
- Multiple SAP configs can exist (e.g., `SAP_Config` for default, `SAP_Config_<lang>` for language-specific configs)
- Validate SAP response for errors after every RFC call using `validation:is-false`
- Use `<error-mapping targetType="BUSINESS:SAP_*">` for SAP-specific error types

---

## MQTT3 Connector

**Namespace prefix:** `mqtt3:`

Message broker connector for MQTT protocol. Used exclusively in broker-related services for event-driven integration.

### Connection Attributes

| Attribute | Description | Example |
|-----------|-------------|---------|
| `username` | MQTT broker username | `${secure::mqtt.mule.username}` |
| `password` | MQTT broker password | `${secure::mqtt.mule.password}` |
| `url` | Broker URL | `${mqtt.mule.url}` |

### Connection Options

| Attribute | Description | Example |
|-----------|-------------|---------|
| `keepAliveInterval` | Keep-alive interval (seconds) | `${mqtt.mule.keepAlive}` |
| `cleanSession` | Start with clean session | `false` |
| `connectionTimeout` | Connection timeout (seconds) | `${mqtt.mule.connectionTimeout}` |

### Client ID Generator

Each MQTT config must have a unique client ID per flow:

| Attribute | Description | Example |
|-----------|-------------|---------|
| `clientId` | Unique client identifier | `${mqtt.mule.clientId}-stock-management` |

### Operations

| Operation | Description | Key Attributes |
|-----------|-------------|----------------|
| `mqtt3:listener` | Subscribe to topic(s), triggers flow on message | `topicFilter`, `qos` |
| `mqtt3:publish` | Publish message to a topic | `topic`, `<mqtt3:message>` |

### QoS Levels

| Level | Name | Description |
|-------|------|-------------|
| `AT_MOST_ONCE` | QoS 0 | Fire-and-forget, no delivery guarantee |
| `AT_LEAST_ONCE` | QoS 1 | Guaranteed delivery, possible duplicates |
| `EXACTLY_ONCE` | QoS 2 | Guaranteed exactly-once delivery |

### Rules

- One MQTT config per listener flow (e.g., `MQTT3_Config_Stock_Management`, `MQTT3_Config_Orders`)
- Client ID format: `${mqtt.mule.clientId}-{flow-name}` — must be unique per config
- Always use `reconnect-forever` for reconnection strategy
- Always use `cleanSession="false"` to preserve subscriptions across reconnects
- Use `EXACTLY_ONCE` QoS for all topic subscriptions
- Topic prefixes: `DevAcme/` for dev, `Acme/` for prod

---

## APIkit Router

Auto-generates REST API routing from RAML specifications.

| Attribute | Description |
|-----------|-------------|
| `raml` | RAML file name |
| `outboundHeadersMapName` | Variable for response headers (use `outboundHeaders`) |
| `httpStatusVarName` | Variable for HTTP status code (use `httpStatus`) |

> The APIkit router is auto-generated — do NOT modify the generated flow stubs. Implement logic in separate implementation files.

---

## References

- [Connector Examples](../usage/connector.md) — real-world patterns with full XML
- [Mule Components](./mule-components.md) — core components (choice, foreach, try/catch, validation, etc.)
- [Logging Guidelines](../standards/logging.md)
- [Error Handling Standards](../standards/error-handling.md)
- [HTTP Connector Docs](https://docs.mulesoft.com/http-connector/latest/)
- [Database Connector Docs](https://docs.mulesoft.com/db-connector/latest/)
- [VM Connector Docs](https://docs.mulesoft.com/vm-connector/latest/)
- [ObjectStore Connector Docs](https://docs.mulesoft.com/os-connector/latest/)

---
Last updated: 2026-04-10
Owner: Integration Team
