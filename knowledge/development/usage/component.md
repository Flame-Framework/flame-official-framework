# Mule Core Component Examples

## Overview

Real-world examples of Mule 4 core components (routers, scopes, validators, error handling) as used in real-world projects. For attribute reference, see [Mule Components](../reference/mule-components.md). For connector examples, see [Connector Examples](./connector.md).

---

## Choice Router

### Response Status Evaluation After Backend Call

```xml
<choice>
    <when expression="#[attributes.statusCode == 200]">
        <flow-ref name="process-success-response"/>
    </when>
    <when expression="#[attributes.statusCode == 404]">
        <flow-ref name="handle-not-found"/>
    </when>
    <otherwise>
        <flow-ref name="handle-error-response"/>
    </otherwise>
</choice>
```

### Choice-Based Upsert Pattern (PAPI)

```xml
<choice>
    <when expression="#[vars.existingRecord != null]">
        <http:request method="PUT" path="/records/{id}" config-ref="Backend_SAPI">
            <http:body>#[payload]</http:body>
            <http:uri-params>#[{id: vars.existingRecord.id}]</http:uri-params>
        </http:request>
    </when>
    <otherwise>
        <http:request method="POST" path="/records" config-ref="Backend_SAPI">
            <http:body>#[payload]</http:body>
        </http:request>
    </otherwise>
</choice>
```

---

## Foreach

### Foreach with Try/Catch — Process Items Individually (PAPI)

From `acme-orders-papi` items flow — each item is processed independently, errors logged per item:

```xml
<foreach doc:name="For Each Resource" doc:id="GENERATE-UUID"
         collection="#[vars.originalRequest.resources]">

    <try doc:name="Try Process Single Resource" doc:id="GENERATE-UUID">
        <flow-ref name="process-single-resource-sub-flow" doc:id="GENERATE-UUID"
                  doc:name="Process Single Resource"/>

        <error-handler>
            <on-error-continue type="ANY" doc:id="GENERATE-UUID"
                               doc:name="Continue on Single Item Error">
                <logger level="ERROR"
                    message="Error processing item #[vars.counter] — #[error.errorType.identifier]: #[error.description]"
                    doc:name="Log Item Error"/>
            </on-error-continue>
        </error-handler>
    </try>
</foreach>
```

### Foreach with ObjectStore — Reprocess Failed Records

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

---

## Parallel Foreach

### Batch Processing with Error Persistence (PAPI)

```xml
<parallel-foreach collection="#[payload.items]" maxConcurrency="5">
    <try>
        <http:request config-ref="Backend_SAPI" method="POST" path="/process">
            <http:body>#[payload]</http:body>
        </http:request>
    <error-handler>
        <on-error-continue>
            <set-variable variableName="failedItem" value="#[payload]"/>
            <flow-ref name="persist-error-to-objectstore"/>
        </on-error-continue>
    </error-handler>
    </try>
</parallel-foreach>
```

---

## Scatter-Gather

### Parallel Calls to Multiple SAPIs (PAPI)

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

## Try Scope

### Try with Error Mapping — Backend Validation (acme-erp-sapi)

```xml
<try doc:name="Try Backend Call" doc:id="GENERATE-UUID">
    <http:request method="POST" path="/inventory/entries"
                  doc:name="Post Inventory Entry" doc:id="GENERATE-UUID"
                  config-ref="Backend_HTTP_Config"/>

    <validation:is-false doc:name="Response Has Errors" doc:id="GENERATE-UUID"
                         expression="#[vars.hasErrors]"
                         message="#[vars.errorMessage]">
        <error-mapping targetType="BUSINESS:BACKEND_INVENTORY_FAILED"/>
    </validation:is-false>

    <error-handler>
        <on-error-propagate type="BUSINESS:BACKEND_INVENTORY_FAILED" doc:id="GENERATE-UUID">
            <logger level="ERROR"
                message="Backend inventory operation failed: #[error.description]"
                doc:name="Log Backend Error"/>
            <ee:transform doc:name="Set Error Response" doc:id="GENERATE-UUID">
                <ee:message>
                    <ee:set-payload resource="dwl/error/message/backend-error-response.dwl"/>
                </ee:message>
            </ee:transform>
        </on-error-propagate>
    </error-handler>
</try>
```

### Try with On-Error-Continue — Skip Individual Item

```xml
<try doc:name="Try Process Item" doc:id="GENERATE-UUID">
    <http:request method="POST" path="/process" config-ref="Backend_SAPI">
        <http:body>#[payload]</http:body>
    </http:request>

    <error-handler>
        <on-error-continue type="HTTP:CONNECTIVITY, HTTP:TIMEOUT" doc:id="GENERATE-UUID">
            <logger level="ERROR"
                message="Backend unavailable, skipping item — #[error.errorType.identifier]"
                doc:name="Log Connectivity Error"/>
        </on-error-continue>
    </error-handler>
</try>
```

---

## Async Scope

### Fire-and-Forget Notification (EAPI)

```xml
<async doc:name="Async Post-Processing" doc:id="GENERATE-UUID">
    <logger level="DEBUG"
        message="Sending async notification"
        doc:name="Before Async Notification"/>

    <ee:transform doc:name="Transform Notification Payload" doc:id="GENERATE-UUID">
        <ee:message>
            <ee:set-payload resource="dwl/payloads/notification-request.dwl"/>
        </ee:message>
    </ee:transform>

    <http:request method="POST" doc:name="Post Notification" doc:id="GENERATE-UUID"
                  config-ref="HTTP-Request_Notification-Config"
                  path="${notification.path}"
                  sendCorrelationId="ALWAYS"
                  correlationId="#[vars.correlationId default correlationId]">
        <http:headers><![CDATA[#[output application/java
---
{"Content-type" : "application/json"}]]]></http:headers>
    </http:request>

    <logger level="DEBUG"
        message="Async notification sent"
        doc:name="After Async Notification"/>
</async>
```

---

## Raise Error

### Date Range Validation (EAPI)

```xml
<set-variable variableName="startDate" value="#[attributes.queryParams.startDate]"/>
<set-variable variableName="endDate" value="#[attributes.queryParams.endDate]"/>

<choice>
    <when expression="#[(vars.endDate as Date) - (vars.startDate as Date) > |P90D|]">
        <raise-error type="VALIDATION:DATE_RANGE"
            description="Date range cannot exceed 90 days"/>
    </when>
</choice>
```

---

## Validation Module

### Validate URI Param Matches Body (acme-orders-papi)

```xml
<validation:is-true
    doc:name="uriParams.orderId == payload.orderId"
    doc:id="fef7c370-0d85-4a0a-9367-fdc4f8297cbf"
    config-ref="Validation_Config"
    expression="#[attributes.uriParams.orderId == payload.order.id]"
    message="#[&quot;Order ID in the body doesn't match with the Order ID in URI Params&quot;]"/>
```

### Validate Record Exists (acme-orders-papi)

```xml
<validation:is-true
    doc:name="Asset ID doesn't exist."
    doc:id="267a56cd-d11e-40b5-b342-7939cf633422"
    config-ref="Validation_Config"
    expression="#[vars.asset != []]"
    message="#[&quot;Asset ID doesn't exist.&quot;]"/>
```

### Validate with Error Mapping — Backend Response (acme-erp-sapi)

```xml
<validation:is-false
    doc:name="Response Has Errors"
    doc:id="464edced-31f3-4abf-ac07-5824858ea0e2"
    expression="#[vars.hasErrors]"
    message="#[vars.errorMessage]">
    <error-mapping targetType="BUSINESS:BACKEND_INVENTORY_FAILED"/>
</validation:is-false>
```

---

## Dynamic Flow Reference

```xml
<set-variable variableName="scope" value="#[attributes.queryParams.scope default 'default']"/>
<flow-ref name="#[vars.scope ++ '-main-flow']"/>
```

---

## Payload Mapping Location (Correct vs Incorrect)

Transformations belong in the implementation sub-flow, **not** in the API router flow:

```xml
<!-- INCORRECT — transform in router flow -->
<flow name="post:\endpoint:api-config">
    <logger level="INFO" message="..." doc:name="Log start"/>
    <ee:transform>  <!-- WRONG -->
        <ee:set-payload resource="dwl/payloads/mapping.dwl"/>
    </ee:transform>
    <flow-ref name="impl-subflow"/>
</flow>

<!-- CORRECT — transform in implementation sub-flow -->
<flow name="post:\endpoint:api-config">
    <logger level="INFO" message="..." doc:name="Log start"/>
    <flow-ref name="impl-subflow"/>
</flow>

<sub-flow name="impl-subflow">
    <ee:transform>  <!-- CORRECT -->
        <ee:set-payload resource="dwl/payloads/mapping.dwl"/>
    </ee:transform>
    <http:request ... />
</sub-flow>
```

---

## References

- [Mule Components Reference](../reference/mule-components.md) — attribute tables
- [Connector Examples](./connector.md) — connector-specific patterns
- [Error Handling Standards](../standards/error-handling.md)
- [API Layer Patterns](../standards/api-layers.md)

---
Last updated: 2026-04-10
Owner: Integration Team
