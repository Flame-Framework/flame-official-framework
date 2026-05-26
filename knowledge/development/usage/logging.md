# Logging Usage Examples

Real XML implementations of the logging standards. For the rules and conventions, see [standards/logging.md](../standards/logging.md).

---

## Standard Mule Logger

Use Mule's built-in `<logger>` component everywhere. No custom config block is needed.

```xml
<logger level="INFO" message="Order retrieval started" doc:name="Log start"/>
```

Attributes:
- **`level`** — `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`
- **`message`** — plain text; may include inline variable references with `#[…]`
- **`doc:name`** — short description shown in Anypoint Studio

---

## START Logger (entry of an endpoint flow)

Use INFO level. Keep the message short and descriptive. If the correlation ID or a key parameter is useful for tracing, include it inline.

### Endpoint with query parameters (e.g., `GET /orders?status=active`)

```xml
<flow name="get:\orders:api-config">
    <logger level="INFO"
        message="GET /orders started (status=#[attributes.queryParams.status default 'all'], correlationId=#[vars.correlationId])"
        doc:name="Log start"/>

    <flow-ref name="get-orders-implementation"/>
    <!-- ... END logger ... -->
</flow>
```

### Endpoint with URI parameters (e.g., `GET /orders/{orderId}`)

```xml
<flow name="get:\orders\(orderId):api-config">
    <logger level="INFO"
        message="GET /orders/#[attributes.uriParams.orderId] started (correlationId=#[vars.correlationId])"
        doc:name="Log start"/>
    <!-- ... -->
</flow>
```

### Endpoint without query/URI parameters (e.g., `POST /orders`)

```xml
<flow name="post:\orders:api-config">
    <logger level="INFO"
        message="POST /orders started (correlationId=#[vars.correlationId])"
        doc:name="Log start"/>
    <!-- ... -->
</flow>
```

---

## END Logger (exit of an endpoint flow)

Use INFO level. No payload.

```xml
<logger level="INFO"
    message="GET /orders completed (correlationId=#[vars.correlationId])"
    doc:name="Log end"/>
```

---

## BEFORE / AFTER External HTTP Calls

Use DEBUG level. Logging around outbound calls is recommended for troubleshooting but not strictly required at every call site. When logged, keep the message plain and let DEBUG-level controls (per environment) decide whether it appears.

```xml
<logger level="DEBUG"
    message="Calling backend POST /orders"
    doc:name="Log before backend call"/>

<http:request config-ref="Backend_HTTP_Request_Config" method="POST" path="/api/v1/orders">
    <http:headers><![CDATA[#[{
        'x-correlation-id': vars.correlationId
    }]]]></http:headers>
</http:request>

<logger level="DEBUG"
    message="Backend responded with status #[attributes.statusCode]"
    doc:name="Log after backend call"/>
```

### GET requests

Same pattern — no payload in either logger:

```xml
<logger level="DEBUG"
    message="Calling backend GET /products"
    doc:name="Log before backend call"/>

<http:request config-ref="Backend_HTTP_Request_Config" method="GET" path="/products">
    <http:headers><![CDATA[#[{
        'x-correlation-id': vars.correlationId
    }]]]></http:headers>
</http:request>

<logger level="DEBUG"
    message="Backend GET /products responded with status #[attributes.statusCode]"
    doc:name="Log after backend call"/>
```

---

## BEFORE / AFTER Transform (optional)

Use DEBUG when troubleshooting transformation issues.

```xml
<logger level="DEBUG" message="Before transform" doc:name="Log before transform"/>

<ee:transform>
    <ee:message>
        <ee:set-payload resource="dwl/payloads/orders/response.dwl"/>
    </ee:message>
</ee:transform>

<logger level="DEBUG" message="After transform" doc:name="Log after transform"/>
```

---

## Error Handler Logger

Always log inside `<on-error-*>` scopes at `level="ERROR"`. See [error-handling.md](./error-handling.md) for the full error-handler XML.

```xml
<on-error-continue type="ANY">
    <logger level="ERROR"
        message="Unhandled error #[error.errorType.identifier]: #[error.description] (correlationId=#[vars.correlationId])"
        doc:name="Log unhandled error"/>
    <!-- transform + httpStatus -->
</on-error-continue>
```

---

## Inbound Correlation ID (in main flow)

Capture the inbound correlation ID into `vars.correlationId` so it can be forwarded on every outbound call. This is the only mandatory traceability variable.

```xml
<set-variable variableName="correlationId"
    value="#[attributes.headers.'x-correlation-id' default correlationId]"
    doc:name="Set correlationId"/>
```

---

## Outbound Correlation ID Propagation

Every outbound HTTP request MUST send the correlation ID in the `x-correlation-id` header:

```xml
<http:request config-ref="Backend_HTTP_Request_Config" method="GET" path="/resource">
    <http:headers><![CDATA[#[{
        'x-correlation-id': vars.correlationId default correlationId
    }]]]></http:headers>
</http:request>
```

This rule is checked by code-review (B-II.2.1).

---

## Anti-Pattern Examples (DON'T)

```xml
<!-- ❌ DON'T: use json-logger:logger -->
<json-logger:logger config-ref="JSON_Logger_Config" message="..."/>
<!-- ✅ DO: use the standard <logger> -->
<logger level="INFO" message="..." doc:name="..."/>

<!-- ❌ DON'T: embed multi-line DataWeave inside the message attribute -->
<logger level="INFO" message="#[output application/json ---
{
    correlationId: vars.correlationId,
    queryParams: attributes.queryParams
}]" doc:name="Bad logger"/>
<!-- ✅ DO: keep the message plain; inline #[vars.x] is fine -->
<logger level="INFO"
    message="GET /orders started (correlationId=#[vars.correlationId])"
    doc:name="Log start"/>

<!-- ❌ DON'T: omit the correlation ID header on outbound calls -->
<http:request config-ref="Backend_HTTP_Request_Config" path="/x"/>
<!-- ✅ DO: send x-correlation-id from vars.correlationId -->

<!-- ❌ DON'T: log secrets or full unredacted payloads at INFO -->
<logger level="INFO" message="#[payload]" doc:name="Bad logger"/>
```

---

## References

- [Logging Standards](../standards/logging.md) — rules and conventions
- [Reference Flows](./reference-flows.md) — full end-to-end flows showing loggers in context
- [Error Handling Usage](./error-handling.md) — error-path loggers in error-handler blocks

---
Last updated: 2026-05-16
Owner: Integration Team
