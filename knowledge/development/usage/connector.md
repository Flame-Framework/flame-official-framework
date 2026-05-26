# Connectors Usage Examples

## Overview

This document provides real-world examples of how each connector is used in MuleSoft projects in this framework. Use this as a reference for typical configurations, patterns, and common pitfalls.

---

## HTTP Listener — Inbound Configuration

### Standard Listener (global.xml)

```xml
<http:listener-config name="HTTP_Listener_Config">
    <http:listener-connection host="${http.host}" port="${http.port}" protocol="HTTP"/>
</http:listener-config>
```

### HTTPS/TLS Listener

```xml
<http:listener-config name="HTTP_HTTPS_Listener_Config">
    <http:listener-connection host="${http.host}" port="${https.port}" protocol="HTTPS">
        <tls:context>
            <tls:key-store path="${keystore.path}"
                          password="${secure::keystore.password}"
                          keyPassword="${secure::key.password}"/>
        </tls:context>
    </http:listener-connection>
</http:listener-config>
```

### Listener Source with Response/Error-Response

```xml
<http:listener config-ref="{config}" path="/api/*" allowedMethods="GET,POST,PUT,PATCH,DELETE">
    <http:response statusCode="#[vars.httpStatus default 200]">
        <http:headers>#[vars.outboundHeaders default {}]</http:headers>
    </http:response>
    <http:error-response statusCode="#[vars.httpStatus default 500]">
        <http:headers>#[vars.outboundHeaders default {}]</http:headers>
    </http:error-response>
</http:listener>
```

---

## HTTP Request — SAPI to Backend

### Basic Auth to Backend System

```xml
<!-- global.xml -->
<http:request-config name="Backend_Request_Config">
    <http:request-connection host="${backend.host}" port="${backend.port}">
        <http:authentication>
            <http:basic-authentication username="${backend.username}"
                                        password="${secure::backend.password}"
                                        preemptive="true"/>
        </http:authentication>
    </http:request-connection>
</http:request-config>
```

```xml
<!-- implementation sub-flow -->
<sub-flow name="get-orders-impl">
    <logger level="DEBUG"
        message="Calling Backend GET /orders"
        doc:name="Log before backend call"/>

    <http:request method="GET" path="${backend.basePath}/orders" config-ref="Backend_Request_Config">
        <http:headers><![CDATA[#[{
            'x-correlation-id': vars.correlationId default correlationId
        }]]]></http:headers>
        <http:query-params><![CDATA[#[vars.queryParams default {}]]]></http:query-params>
    </http:request>

    <logger level="DEBUG"
        message="Backend GET /orders responded with status #[attributes.statusCode]"
        doc:name="Log after backend call"/>
</sub-flow>
```

**Config properties (`config-local.yaml`):**
```yaml
backend:
  host: "localhost"
  port: "8082"
  basePath: "/api/v1"
  username: "api-user"
```

**Secure properties (`secure-config-local.yaml`):**
```yaml
backend:
  password: "![encryptedValue]"
```

### Client Credentials (OAuth 2.0) Pattern

```xml
<!-- Auth sub-flow — called before backend requests -->
<sub-flow name="get-bearer-token">
    <http:request method="POST" path="${auth.tokenPath}" config-ref="Auth_Request_Config">
        <http:body><![CDATA[#[output application/x-www-form-urlencoded ---
{
    client_id: p('secure::auth.clientId'),
    client_secret: p('secure::auth.clientSecret'),
    grant_type: 'client_credentials'
}]]]></http:body>
    </http:request>
    <set-variable variableName="authToken" value="#[payload.access_token]"/>
</sub-flow>
```

```xml
<!-- Using the token in subsequent requests -->
<http:request method="GET" path="/api/resource" config-ref="Backend_Request_Config">
    <http:headers><![CDATA[#[{
        'Authorization': 'Bearer ' ++ vars.authToken,
        'x-correlation-id': vars.correlationId default correlationId
    }]]]></http:headers>
</http:request>
```

---

## HTTP Request — PAPI to SAPI

### Digest Auth

```xml
<http:request-config name="Backend_Digest_Config">
    <http:request-connection host="${backend.host}" port="${backend.port}">
        <http:authentication>
            <http:digest-authentication username="${backend.username}"
                                         password="${secure::backend.password}"/>
        </http:authentication>
    </http:request-connection>
</http:request-config>
```

### NTLM Auth

```xml
<http:request-config name="Backend_NTLM_Config">
    <http:request-connection host="${backend.host}" port="${backend.port}">
        <http:authentication>
            <http:ntlm-authentication username="${backend.username}"
                                       password="${secure::backend.password}"
                                       domain="${backend.domain}"/>
        </http:authentication>
    </http:request-connection>
</http:request-config>
```

### Response Validator

```xml
<http:request method="GET" path="/api/resource" config-ref="Backend_Config">
    <http:response-validator>
        <http:success-status-code-validator values="200..299"/>
    </http:response-validator>
</http:request>
```

---

## HTTP Request — PAPI to SAPI

### Calling a System API from a Process API

```xml
<!-- global.xml -->
<http:request-config name="Backend_SAPI_Request_Config">
    <http:request-connection host="${backend-sapi.host}" port="8081">
        <http:default-headers>
            <http:default-header key="client_id" value="${backend-sapi.clientId}"/>
            <http:default-header key="client_secret" value="${secure::backend-sapi.clientSecret}"/>
        </http:default-headers>
    </http:request-connection>
</http:request-config>
```

```yaml
# config-local.yaml — CloudHub 2.0 internal URL pattern
backend-sapi:
  host: "http://acme-backend-sapi.private-space.svc.cluster.local:8081"
  basePath: "/api/v1"
  clientId: "dev-client-id"
```

### Scatter-Gather — Parallel Calls to Multiple SAPIs

```xml
<sub-flow name="aggregate-orders-data">
    <scatter-gather>
        <route>
            <http:request config-ref="Backend_SAPI_Config" method="GET"
                path="${backend-sapi.basePath}/orders" target="orders">
                <http:headers><![CDATA[#[{'x-correlation-id': vars.correlationId default correlationId}]]]></http:headers>
                <http:query-params><![CDATA[#[vars.queryParams]]]></http:query-params>
            </http:request>
        </route>
        <route>
            <http:request config-ref="Catalog_SAPI_Config" method="GET"
                path="${catalog-sapi.basePath}/items" target="items">
                <http:headers><![CDATA[#[{'x-correlation-id': vars.correlationId default correlationId}]]]></http:headers>
            </http:request>
        </route>
    </scatter-gather>
    <!-- Aggregate results from parallel calls -->
    <ee:transform>
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    orders: vars.orders,
    items: vars.items
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</sub-flow>
```

---

## ObjectStore — Error Persistence & Reprocess

### Store Failed Record for Retry

```xml
<sub-flow name="persist-error-for-retry">
    <ee:transform>
        <ee:set-variable variableName="errorRecord" resource="dwl/vars/reprocess/objectStoreKeyValue.dwl"/>
    </ee:transform>

    <os:store objectStore="Error_Retry_Store"
        key="#[vars.errorKey]"
        value="#[vars.errorRecord]"
        failIfPresent="false"/>

    <logger level="ERROR"
        message="Error persisted for retry — key=#[vars.errorKey], type=#[error.errorType.identifier]"
        doc:name="Log error persistence"/>
</sub-flow>
```

### Reprocess Failed Records

```xml
<sub-flow name="reprocess-failed-records">
    <os:retrieve-all-keys objectStore="Error_Retry_Store" target="errorKeys"/>

    <foreach collection="#[vars.errorKeys]">
        <os:retrieve objectStore="Error_Retry_Store" key="#[payload]" target="errorRecord"/>
        <try>
            <flow-ref name="retry-processing"/>
            <os:remove objectStore="Error_Retry_Store" key="#[payload]"/>
        <error-handler>
            <on-error-continue>
                <!-- Log retry failure, leave in store for next attempt -->
            </on-error-continue>
        </error-handler>
        </try>
    </foreach>
</sub-flow>
```

### ObjectStore Configuration

```xml
<!-- global.xml -->
<os:object-store name="Error_Retry_Store"
    persistent="true"
    maxEntries="1000"
    entryTtl="7"
    entryTtlUnit="DAYS"
    expirationInterval="1"
    expirationIntervalUnit="HOURS"/>
```

### Contains Check

```xml
<os:contains objectStore="Error_Retry_Store" key="#[vars.errorKey]" target="keyExists"/>
<choice>
    <when expression="#[vars.keyExists]">
        <!-- Key already exists — skip or update -->
    </when>
    <otherwise>
        <os:store objectStore="Error_Retry_Store" key="#[vars.errorKey]" value="#[vars.errorRecord]"/>
    </otherwise>
</choice>
```

---

## Scheduler — Batch Processing

### Cron-Based Job

```xml
<!-- scheduler.xml (dedicated file) -->
<flow name="orders-sync-scheduler">
    <scheduler>
        <scheduling-strategy>
            <cron expression="${scheduler.orders.cron}" timeZone="${scheduler.timezone}"/>
        </scheduling-strategy>
    </scheduler>

    <logger level="INFO" message="Orders sync scheduler triggered" doc:name="Log scheduler trigger"/>

    <ee:transform>
        <ee:variables>
            <ee:set-variable variableName="beginDateTime" resource="dwl/vars/scheduler/beginDateTime.dwl"/>
            <ee:set-variable variableName="endDateTime" resource="dwl/vars/scheduler/endDateTime.dwl"/>
        </ee:variables>
    </ee:transform>

    <flow-ref name="orders-sync-impl"/>
</flow>
```

```yaml
# config-local.yaml
scheduler:
  orders:
    cron: "0 0 6 * * ?"   # Daily at 6am
  timezone: "UTC"
```

---

## VM Connector — Async Processing & Queue Consumption

### Fire-and-Forget Notification

```xml
<!-- global.xml -->
<vm:config name="VM_Config">
    <vm:queues>
        <vm:queue queueName="notification-queue" queueType="TRANSIENT" maxOutstandingMessages="100"/>
    </vm:queues>
</vm:config>
```

```xml
<!-- Publish to queue (non-blocking) -->
<async>
    <vm:publish config-ref="VM_Config" queueName="notification-queue">
        <vm:content><![CDATA[#[{
            correlationId: vars.correlationId,
            event: "order-created",
            payload: payload
        }]]]></vm:content>
    </vm:publish>
</async>
```

```xml
<!-- Listener flow (processes queue messages) -->
<flow name="notification-processor">
    <vm:listener config-ref="VM_Config" queueName="notification-queue"/>
    <flow-ref name="send-notification-impl"/>
</flow>
```

### Consume (Pull from Queue with Timeout)

```xml
<vm:consume config-ref="VM_Config" queueName="notification-queue"
    timeout="5" timeoutUnit="SECONDS"/>
```

---

## Database Connector — Direct Backend Access

### Parameterized Query

```xml
<!-- global.xml -->
<db:config name="Database_Config">
    <db:generic-connection url="${db.url}"
        driverClassName="${db.driver}"
        user="${db.user}"
        password="${secure::db.password}"/>
</db:config>
```

```xml
<sub-flow name="get-orders-from-db">
    <db:select config-ref="Database_Config">
        <db:sql>SELECT order_id, customer_name, status, created_at
                 FROM orders
                 WHERE status = :status AND created_at >= :fromDate</db:sql>
        <db:input-parameters>#[{
            status: vars.orderStatus,
            fromDate: vars.fromDate
        }]</db:input-parameters>
    </db:select>
</sub-flow>
```

### Insert

```xml
<db:insert config-ref="Database_Config">
    <db:sql>INSERT INTO orders (id, name, status) VALUES (:id, :name, :status)</db:sql>
    <db:input-parameters>#[{id: payload.id, name: payload.name, status: 'NEW'}]</db:input-parameters>
</db:insert>
```

### Update

```xml
<db:update config-ref="Database_Config">
    <db:sql>UPDATE orders SET status = :status WHERE id = :id</db:sql>
    <db:input-parameters>#[{status: payload.status, id: payload.id}]</db:input-parameters>
</db:update>
```

### Delete

```xml
<db:delete config-ref="Database_Config">
    <db:sql>DELETE FROM orders WHERE id = :id</db:sql>
    <db:input-parameters>#[{id: vars.orderId}]</db:input-parameters>
</db:delete>
```

### Stored Procedure

```xml
<db:stored-procedure config-ref="Database_Config">
    <db:sql>CALL process_order(:orderId, :result)</db:sql>
    <db:input-parameters>#[{orderId: vars.orderId}]</db:input-parameters>
    <db:output-parameters>
        <db:output-parameter key="result" type="VARCHAR"/>
    </db:output-parameters>
</db:stored-procedure>
```

---

## Caching — EE Cache with ObjectStore

### Reference Data Caching (EAPI)

```xml
<!-- global.xml -->
<os:object-store name="Response_Cache_Store"
    persistent="false"
    maxEntries="100"
    entryTtl="30"
    entryTtlUnit="MINUTES"/>

<ee:object-store-caching-strategy name="Response_Caching_Strategy"
    objectStore="Response_Cache_Store">
    <ee:key-generation-expression>#[attributes.requestPath]</ee:key-generation-expression>
</ee:object-store-caching-strategy>
```

```xml
<!-- Usage in flow -->
<ee:cache cachingStrategy-ref="Response_Caching_Strategy">
    <http:request config-ref="Backend_PAPI" path="/reference-data"/>
</ee:cache>
```

---

## SAP Connector — RFC Calls

### Global Configuration (acme-erp-sapi/global.xml)

```xml
<sap:sap-config name="SAP_Config"
    doc:name="SAP Config" doc:id="GENERATE-UUID">
    <sap:simple-connection-provider-connection
        username="${secure::sap.user}"
        password="${secure::sap.password}"
        systemNumber="00"
        client="${sap.clientId}"
        applicationServerHost="${sap.host}"/>
</sap:sap-config>

<!-- Language-specific variant -->
<sap:sap-config name="SAP_Config_<lang>"
    doc:name="SAP Config <lang>" doc:id="GENERATE-UUID">
    <sap:simple-connection-provider-connection
        username="${secure::sap.user}"
        password="${secure::sap.password}"
        systemNumber="00"
        client="${sap.clientId}"
        applicationServerHost="${sap.host}"
        language="<lang-code>"/>
</sap:sap-config>
```

### Synchronous RFC Call

```xml
<sap:sync-rfc key="Z_BACKEND_SEND_MATERIALS"
    doc:name="Z_BACKEND_SEND_MATERIALS" doc:id="GENERATE-UUID"
    config-ref="SAP_Config"/>
```

### RFC Call with Response Validation

```xml
<sap:sync-rfc key="Z_BACKEND_INVENTORY_ENTRY"
    doc:name="Z_BACKEND_INVENTORY_ENTRY" doc:id="GENERATE-UUID"
    config-ref="SAP_Config"/>

<validation:is-false doc:name="Response Has Errors" doc:id="GENERATE-UUID"
    expression="#[vars.hasErrors]"
    message="#[vars.errorMessage]">
    <error-mapping targetType="BUSINESS:BACKEND_INVENTORY_FAILED"/>
</validation:is-false>
```

### SAP Config Properties

```yaml
# config-dev.yaml
sap:
  system: "{sap-system}"
  clientId: "{sap-client-id}"
  host: "{sap-host}"
  odata:
    host: "{sap-host}"
    port: "8000"
    protocol: "HTTP"
    basePath: "/sap/opu/odata/sap"
    timeout: "90000"
    path:
      materials: "/{custom-srv-1}/materials"
      suppliers: "/{custom-srv-2}/vendors"
      costCenters: "/{custom-srv-3}/cost_centers"
      requisitions: "/{custom-srv-4}/purchase_requisitions"
      purchaseOrders: "/{custom-srv-5}/purchase_orders"
      orderCosts: "/{custom-srv-7}/order_costs"
```

```yaml
# secure-config-dev.yaml
sap:
  user: "![encryptedValue]"
  password: "![encryptedValue]"
  odata:
    token: "![encryptedValue]"
```

---

## MQTT3 Connector — Pub/Sub Messaging

### Global Configuration (acme-broker-sapi/global.xml)

One config per listener flow, each with a unique client ID:

```xml
<mqtt3:config name="MQTT3_Config_Stock_Management"
    doc:name="MQTT3 Config Stock Management" doc:id="GENERATE-UUID">
    <mqtt3:connection
        username="${secure::mqtt.mule.username}"
        password="${secure::mqtt.mule.password}"
        url="${mqtt.mule.url}">
        <reconnection>
            <reconnect-forever/>
        </reconnection>
        <mqtt3:client-id-generator>
            <mqtt3:client-id-custom-expression-generator
                clientId="${mqtt.mule.clientId}-stock-management"/>
        </mqtt3:client-id-generator>
        <mqtt3:connection-options
            keepAliveInterval="${mqtt.mule.keepAlive}"
            cleanSession="false"
            connectionTimeout="${mqtt.mule.connectionTimeout}"/>
    </mqtt3:connection>
</mqtt3:config>
```

### Listener — Subscribe to Topic

```xml
<mqtt3:listener doc:name="Stock Management" doc:id="GENERATE-UUID"
    config-ref="MQTT3_Config_Stock_Management">
    <mqtt3:topics>
        <mqtt3:topic
            topicFilter="${subscribers.topic.stock.management}"
            qos="EXACTLY_ONCE"/>
    </mqtt3:topics>
</mqtt3:listener>
```

### Publish to Topic

```xml
<mqtt3:publish doc:name="Publish" doc:id="GENERATE-UUID"
    config-ref="MQTT3_Config_Publish"
    topic="#[payload.topic]">
    <mqtt3:message><![CDATA[#[vars.req]]]></mqtt3:message>
</mqtt3:publish>
```

### MQTT Config Properties

```yaml
# config-dev.yaml
mqtt:
  mule:
    url: "tcp://{mqtt-broker-host}:{mqtt-broker-port}"
    clientId: "mulesoft-dev"
    keepAlive: "60"
    connectionTimeout: "30"

subscribers:
  topic:
    normalized:
      requisitions: "DevAcme/Region/Normalized/Requisition/+"
      material: "AcmeDev/Region/Normalized/Material/+"
      supplier: "DevAcme/Region/Normalized/Supplier/+"
      order-costs: "DevAcme/Region/Normalized/OrderCosts/+"
      purchase-orders: "DevAcme/Region/Normalized/PurchaseOrder/+"
    region:
      orders: "Acme/Edge/Analytics/Region/OrderEvent"
    stock:
      management: "Acme/Region/Operations/+/{plant-code}/+/Normalised/Transactions/Production"
```

```yaml
# secure-config-dev.yaml
mqtt:
  mule:
    username: "![encryptedValue]"
    password: "![encryptedValue]"
```

---

## Logger — Standard Mule Component

```xml
<logger level="{ERROR|WARN|INFO|DEBUG|TRACE}"
    message="{plain text — may reference variables with #[…]}"
    doc:name="{short description}"/>
```

Typical usage:

```xml
<logger level="INFO"
    message="GET /orders started (correlationId=#[vars.correlationId])"
    doc:name="Log start"/>

<logger level="DEBUG"
    message="Backend GET /orders responded with status #[attributes.statusCode]"
    doc:name="Log after backend call"/>

<logger level="ERROR"
    message="Unhandled error #[error.errorType.identifier]: #[error.description]"
    doc:name="Log unhandled error"/>
```

See [Logging Guidelines](../standards/logging.md) for full rules.

---

## Secure Properties — Configuration & Referencing

### Configuration (global.xml)

```xml
<secure-properties:config name="Secure_Properties_Config"
    file="properties/secure-config-${mule.env}.yaml"
    key="${encrypt.key}">
    <secure-properties:encrypt algorithm="Blowfish" mode="CBC"/>
</secure-properties:config>
```

### Referencing Secure Properties

```xml
<!-- In XML attributes -->
password="${secure::target.password}"

<!-- In DataWeave -->
Mule::p('secure::target.password')
```

---

## APIkit Router — Auto-Generated Routing

### Configuration (global.xml)

```xml
<apikit:config name="{api-name}-config" raml="{api-name}.raml"
    outboundHeadersMapName="outboundHeaders"
    httpStatusVarName="httpStatus"/>
```

### Router (main flow)

```xml
<apikit:router config-ref="{api-name}-config"/>
```

> Do NOT modify the generated flow stubs. Implement logic in separate implementation files.

---

## Common Pitfalls

| Pitfall | Impact | Prevention |
|---------|--------|------------|
| Hardcoding URLs in HTTP request config | Fails across environments | Always use `${property}` references |
| Missing `x-correlation-id` header | Cannot trace requests | Include in ALL outbound HTTP requests |
| ObjectStore without `persistent="true"` | Data lost on restart | Set `persistent="true"` for error retry stores |
| Scheduler in main API XML | Clutters main flow | Always use dedicated `scheduler.xml` |
| Missing `preemptive="true"` on Basic Auth | Extra round-trip per request | Set `preemptive="true"` |
| SQL string concatenation | SQL injection risk | Always use `:paramName` parameterized queries |
| VM persistent queues on CloudHub 2.0 | Not supported, deployment fails | Use TRANSIENT queues on CloudHub 2.0 |
| Missing `target` attribute on HTTP request | Overwrites current payload | Use `target="varName"` when you need to preserve payload |

---

## References

- [Connectors Reference](../reference/connectors.md) — attribute tables and versions
- [Component Examples](./component.md) — core Mule components (choice, foreach, scatter-gather, try/catch, async, raise-error, validation)
- [API Layer Patterns](../standards/api-layers.md)
- [Error Handling Standards](../standards/error-handling.md)
- [Logging Guidelines](../standards/logging.md)

---
Last updated: 2026-04-09
Owner: Integration Team
