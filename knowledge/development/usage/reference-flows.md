# Reference Flows

Three canonical end-to-end Mule flow examples — one per API layer — drawn from real-world projects. **Use these as scaffolds when creating new endpoints.** Each example shows the full happy-path: listener → loggers → processing → response, plus references to the supporting DWL files and error handlers.

For layer-specific rules see [api-layers.md](../standards/api-layers.md). For naming see [naming-conventions.md](../standards/naming-conventions.md).

---

## Anatomy Common to All Layers

Every endpoint flow has the same skeleton:

```
1. START logger        (INFO, plain text)
2. Optional input validation
3. Inbound transform   (DWL — request payload to backend shape)
4. <http:request>      (or other connector — propagate x-correlation-id)
5. Outbound transform  (DWL — backend response to API contract)
6. END logger          (INFO, plain text)
7. <error-handler ref="apikit-error-handler" />
```

PAPI orchestrations repeat steps 3–5 once per backend call. Optional DEBUG loggers around outbound calls and transforms are encouraged when troubleshooting.

---

## Reference Flow 1 — SAPI

**Source**: `acme-erp-sapi/src/main/mule/acme-erp-sapi.xml`
**Endpoint**: `GET /products`
**Backend**: Backend system

### Main Flow (entry point — one per project)

```xml
<flow name="acme-erp-sapi-main">
    <http:listener config-ref="HTTP_Listener_config" path="/api/*"
        doc:name="HTTP Listener">
        <http:response statusCode="#[vars.httpStatus default 200]" />
        <http:error-response statusCode="#[vars.httpStatus default 500]">
            <http:body>#[payload]</http:body>
        </http:error-response>
    </http:listener>

    <set-variable variableName="correlationId"
        value="#[attributes.headers.'x-correlation-id' default correlationId]"
        doc:name="Set correlationId" />

    <apikit:router config-ref="acme-erp-sapi-config" doc:name="APIKit Router" />

    <error-handler ref="apikit-error-handler" />
</flow>
```

### Endpoint Flow (RAML-generated)

```xml
<flow name="get:\products:acme-erp-sapi-config">
    <logger level="INFO"
        message="GET /products started (correlationId=#[vars.correlationId])"
        doc:name="Log start"/>

    <flow-ref name="implementation-products-acme-erp-sapiFlow"
        doc:name="Call Implementation" />

    <logger level="INFO"
        message="GET /products completed (correlationId=#[vars.correlationId])"
        doc:name="Log end"/>
</flow>
```

### Implementation Sub-flow (the actual work)

```xml
<sub-flow name="implementation-products-acme-erp-sapiFlow">
    <logger level="DEBUG"
        message="Calling backend GET /products"
        doc:name="Log before backend call"/>

    <http:request config-ref="Backend_HTTP_Request_Config"
        method="GET" path="/products" doc:name="GET Backend Products">
        <http:headers><![CDATA[#[{
            'x-correlation-id': vars.correlationId default correlationId
        }]]]></http:headers>
        <http:query-params>#[attributes.queryParams]</http:query-params>
    </http:request>

    <logger level="DEBUG"
        message="Backend GET /products responded with status #[attributes.statusCode]"
        doc:name="Log after backend call"/>

    <ee:transform doc:name="Map Backend Response to API Contract">
        <ee:message>
            <ee:set-payload resource="dwl/payloads/products/response.dwl" />
        </ee:message>
    </ee:transform>
</sub-flow>
```

### Supporting Files

| File | Purpose |
|---|---|
| `src/main/resources/dwl/payloads/products/response.dwl` | Maps Backend response to RAML schema |
| `src/main/mule/common/error-handler.xml` | `apikit-error-handler` |
| `src/main/mule/global.xml` | `HTTP_Listener_config`, `Backend_HTTP_Request_Config`, `Secure_Properties_Config` |

---

## Reference Flow 2 — PAPI

**Source**: `acme-orders-papi/src/main/mule/acme-orders-papi.xml` + `implementation/orders.xml`
**Endpoint**: `POST /orders/{orderId}/items`
**Orchestration**: Calls `acme-broker-sapi` (lookup customer), then `acme-erp-sapi` (post order item)

### Endpoint Flow

```xml
<flow name="post:\orders\(orderId)\items:application\json:acme-orders-papi-config">
    <logger level="INFO"
        message="POST /orders/#[attributes.uriParams.orderId]/items started (correlationId=#[vars.correlationId])"
        doc:name="Log start"/>

    <flow-ref name="orders-sub-flow" doc:name="Process Order" />

    <logger level="INFO"
        message="POST /orders/#[attributes.uriParams.orderId]/items completed (correlationId=#[vars.correlationId])"
        doc:name="Log end"/>

    <error-handler ref="error-handler-with-reprocess" />
</flow>
```

### Implementation Sub-flow — Orchestration

```xml
<sub-flow name="orders-sub-flow">
    <!-- Save inbound payload for later reprocess if needed -->
    <set-variable variableName="originalPayload" value="#[payload]"
        doc:name="Save Original Payload" />

    <!-- === Call 1: Broker SAPI to resolve customer === -->
    <logger level="DEBUG"
        message="Calling acme-broker-sapi GET /customers/#[attributes.uriParams.orderId]"
        doc:name="Log before broker call"/>

    <http:request config-ref="Broker_SAPI_Request_Config"
        method="GET" path="/customers/#[attributes.uriParams.orderId]"
        doc:name="GET Customer from Broker">
        <http:headers><![CDATA[#[{
            'x-correlation-id': vars.correlationId default correlationId
        }]]]></http:headers>
    </http:request>

    <logger level="DEBUG"
        message="acme-broker-sapi responded with status #[attributes.statusCode]"
        doc:name="Log after broker call"/>

    <set-variable variableName="customer" value="#[payload]"
        doc:name="Save Customer Lookup" />
    <set-payload value="#[vars.originalPayload]"
        doc:name="Restore Original Payload" />

    <!-- === Transform original request + customer data into Backend shape === -->
    <ee:transform doc:name="Build Backend Order Request">
        <ee:message>
            <ee:set-payload resource="dwl/payloads/orders/backendRequest.dwl" />
        </ee:message>
    </ee:transform>

    <!-- === Call 2: Backend SAPI to post order item === -->
    <logger level="DEBUG"
        message="Calling acme-erp-sapi POST /items"
        doc:name="Log before backend call"/>

    <http:request config-ref="Backend_SAPI_Request_Config"
        method="POST" path="/items" doc:name="POST Item to Backend">
        <http:headers><![CDATA[#[{
            'x-correlation-id': vars.correlationId default correlationId
        }]]]></http:headers>
    </http:request>

    <logger level="DEBUG"
        message="acme-erp-sapi responded with status #[attributes.statusCode]"
        doc:name="Log after backend call"/>

    <!-- === Build API response === -->
    <ee:transform doc:name="Build API Response">
        <ee:message>
            <ee:set-payload resource="dwl/payloads/orders/response.dwl" />
        </ee:message>
        <ee:variables>
            <ee:set-variable variableName="httpStatus"><![CDATA[201]]></ee:set-variable>
        </ee:variables>
    </ee:transform>
</sub-flow>
```

### PAPI-Specific Notes

- **`originalPayload` is saved** before the first backend call so it can be re-used after the lookup
- **Reprocess error handler** is referenced — failures in `acme-erp-sapi` are retry-eligible
- **`x-correlation-id` propagated** on every outbound call
- **HTTP 201** explicitly set for the create operation

### Supporting Files

| File | Purpose |
|---|---|
| `src/main/resources/dwl/payloads/orders/backendRequest.dwl` | Builds Backend request from API + customer lookup |
| `src/main/resources/dwl/payloads/orders/response.dwl` | Builds API response |
| `src/main/resources/dwl/error/reprocess/keyError.dwl` | Reprocess key (used by reprocess handler) |
| `src/main/mule/common/reprocess.xml` | Reprocess sub-flows |
| `src/main/mule/common/error-handler.xml` | Includes `error-handler-with-reprocess` |

---

## Reference Flow 3 — EAPI

**Source**: `acme-procurement-eapi/src/main/mule/acme-procurement-eapi.xml`
**Endpoint**: `POST /purchaseOrders/approval`
**Backend**: Single PAPI call (`acme-purchase-papi`)

### Endpoint Flow

```xml
<flow name="post:\purchaseOrders\approval:application\json:acme-procurement-eapi-config">
    <logger level="INFO"
        message="POST /purchaseOrders/approval started (correlationId=#[vars.correlationId])"
        doc:name="Log start"/>

    <flow-ref name="post-po-approvers-subflow" doc:name="Process PO Approval" />

    <logger level="INFO"
        message="POST /purchaseOrders/approval completed (correlationId=#[vars.correlationId])"
        doc:name="Log end"/>
</flow>
```

### Implementation Sub-flow

```xml
<sub-flow name="post-po-approvers-subflow">
    <!-- 1. Validate inbound payload (EAPI = consumer-facing, validate strictly) -->
    <ee:transform doc:name="Validate Inbound">
        <ee:variables>
            <ee:set-variable variableName="validationErrors"
                resource="dwl/vars/purchaseOrders/validateApprovalRequest.dwl" />
        </ee:variables>
    </ee:transform>

    <choice doc:name="Has Validation Errors?">
        <when expression="#[sizeOf(vars.validationErrors) > 0]">
            <raise-error type="APP:VALIDATION"
                description="#[vars.validationErrors joinBy '; ']"
                doc:name="Raise Validation Error" />
        </when>
        <otherwise>
            <logger level="DEBUG" message="No validation errors" doc:name="Log no validation errors"/>
        </otherwise>
    </choice>

    <!-- 2. Transform to PAPI shape -->
    <ee:transform doc:name="Build PAPI Request">
        <ee:message>
            <ee:set-payload resource="dwl/payloads/purchaseOrders/papiApprovalRequest.dwl" />
        </ee:message>
    </ee:transform>

    <!-- 3. Call PAPI -->
    <logger level="DEBUG"
        message="Calling acme-purchase-papi POST /purchaseOrders/approval"
        doc:name="Log before PAPI call"/>

    <http:request config-ref="Acme_Purchase_PAPI_Request_Config"
        method="POST" path="/purchaseOrders/approval"
        doc:name="POST PO Approval to PAPI">
        <http:headers><![CDATA[#[{
            'x-correlation-id': vars.correlationId default correlationId
        }]]]></http:headers>
    </http:request>

    <logger level="DEBUG"
        message="acme-purchase-papi responded with status #[attributes.statusCode]"
        doc:name="Log after PAPI call"/>

    <!-- 4. Shape response for EAPI consumer -->
    <ee:transform doc:name="Shape Consumer Response">
        <ee:message>
            <ee:set-payload resource="dwl/payloads/purchaseOrders/consumerResponse.dwl" />
        </ee:message>
    </ee:transform>
</sub-flow>
```

### EAPI-Specific Notes

- **Strict input validation** before any backend call (consumer-facing)
- **No reprocess** — EAPIs propagate errors back to the consumer immediately
- **Response-shaping transform** at the end — EAPI is responsible for the consumer-facing contract
- `APP:VALIDATION` raised on validation failure with the joined error messages

---

## Quick Decision Table

| You're building... | Start from |
|---|---|
| GET endpoint hitting one backend | Reference Flow 1 (SAPI) |
| POST/PUT endpoint hitting one backend | Reference Flow 1 (SAPI), add a DEBUG logger before the request if needed |
| Endpoint orchestrating 2+ backends | Reference Flow 2 (PAPI) |
| Data-sync write that needs retry resilience | Reference Flow 2 (PAPI) + reprocess handler |
| Consumer-facing endpoint with strict validation | Reference Flow 3 (EAPI) |
| Endpoint that just proxies to one PAPI | Reference Flow 3 (EAPI) without the validation block |

---

## References

- [API Layers](../standards/api-layers.md)
- [Logging Standards](../standards/logging.md) and [Logging Usage Examples](./logging.md)
- [Error Handling Usage](./error-handling.md)
- [Connector Usage](./connector.md)
- [Naming Conventions](../standards/naming-conventions.md)

---
Last updated: 2026-05-16
Owner: Integration Team
