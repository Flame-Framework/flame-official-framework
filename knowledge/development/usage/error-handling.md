# Error Handling Usage Examples

Real `<error-handler>` blocks from real-world projects. Use these as scaffolds when adding error handling to new flows. For rules see [error-handling.md](../standards/error-handling.md). For the catalog of error types see [error-types.md](../reference/error-types.md).

---

## File Layout

Common error handlers live in:

```
src/main/mule/common/error-handler.xml   ← shared, referenced from APIKit + endpoints
src/main/mule/common/reprocess.xml       ← PAPI only — Object Store retry sub-flows
```

Resource-specific handlers (when an endpoint needs custom error mapping) live alongside the implementation:

```
src/main/mule/implementation/{resource}/error-handler-{resource}.xml
```

---

## Example 1 — Standard APIKit Error Handler

Source: `acme-logging-sapi/src/main/mule/common/error-handler.xml`

This is the **canonical APIKit error handler**. Every project in this framework has a variant of this in `common/error-handler.xml`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:ee="http://www.mulesoft.org/schema/mule/ee/core">

    <error-handler name="apikit-error-handler"
        doc:name="APIKit Error Handler">

        <on-error-propagate enableNotifications="true" logException="true"
            type="APIKIT:BAD_REQUEST"
            doc:name="On APIKit Bad Request">
            <logger level="ERROR"
                message="Bad request #[error.errorType.identifier]: #[error.description] (correlationId=#[vars.correlationId])"
                doc:name="Log bad request"/>

            <ee:transform>
                <ee:message>
                    <ee:set-payload resource="dwl/error/message/badRequest.dwl" />
                </ee:message>
                <ee:variables>
                    <ee:set-variable variableName="httpStatus"><![CDATA[400]]></ee:set-variable>
                </ee:variables>
            </ee:transform>
        </on-error-propagate>

        <on-error-propagate type="APIKIT:NOT_FOUND" doc:name="On APIKit Not Found">
            <!-- same shape — sets httpStatus=404, payload=notFound.dwl -->
        </on-error-propagate>

        <on-error-propagate type="APIKIT:METHOD_NOT_ALLOWED" doc:name="On APIKit Method Not Allowed">
            <!-- same shape — sets httpStatus=405 -->
        </on-error-propagate>

        <on-error-propagate type="APIKIT:NOT_ACCEPTABLE" doc:name="On APIKit Not Acceptable">
            <!-- same shape — sets httpStatus=406 -->
        </on-error-propagate>

        <on-error-propagate type="APIKIT:UNSUPPORTED_MEDIA_TYPE" doc:name="On APIKit Unsupported Media Type">
            <!-- same shape — sets httpStatus=415 -->
        </on-error-propagate>

        <on-error-propagate type="ANY" doc:name="On Any Other Error">
            <logger level="ERROR"
                message="Unhandled error #[error.errorType.identifier]: #[error.description] (correlationId=#[vars.correlationId])"
                doc:name="Log unhandled error"/>
            <ee:transform>
                <ee:message>
                    <ee:set-payload resource="dwl/error/message/genericError.dwl" />
                </ee:message>
                <ee:variables>
                    <ee:set-variable variableName="httpStatus"><![CDATA[
                        readUrl('classpath://dwl/error/statusCodes/default.dwl', 'application/dw')
                    ]]></ee:set-variable>
                </ee:variables>
            </ee:transform>
        </on-error-propagate>
    </error-handler>
</mule>
```

**Notes**:
- Every `<on-error-propagate>` has a `doc:name` (validation requirement)
- Every handler logs at `level="ERROR"` with the error type and (where useful) `vars.correlationId` inline
- Payloads come from `dwl/error/message/*.dwl` — never inline
- `httpStatus` is set explicitly per handler

---

## Example 2 — Standard Error Response Payload

Source: `acme-logging-sapi/src/main/resources/dwl/error/message/badRequest.dwl` (representative)

```dwl
%dw 2.0
output application/json
---
{
    "error": {
        "code": "400",
        "message": "Bad request"
    }
}
```

For dynamic mapping (status code from error type), the file becomes:

```dwl
%dw 2.0
output application/json
---
{
    "error": {
        "code": (vars.httpStatus default 500) as String,
        "message": error.description default "Internal server error"
    }
}
```

---

## Example 3 — Status Code Mapping File

Source pattern: `src/main/resources/dwl/error/statusCodes/default.dwl`

```dwl
%dw 2.0
output application/dw
---
error.errorType.asString match {
    case t if t startsWith "APIKIT:BAD_REQUEST"      -> 400
    case t if t startsWith "APIKIT:NOT_FOUND"        -> 404
    case t if t startsWith "APIKIT:METHOD_NOT"       -> 405
    case t if t startsWith "APIKIT:NOT_ACCEPTABLE"   -> 406
    case t if t startsWith "APIKIT:UNSUPPORTED"      -> 415
    case t if t startsWith "VALIDATION:UNPROCESS"    -> 422
    case t if t startsWith "VALIDATION:"             -> 400
    case t if t startsWith "APP:VALIDATION"          -> 400
    case t if t startsWith "APP:BUSINESS_RULE"       -> 422
    case t if t startsWith "APP:REPROCESS"           -> 503
    case t if t startsWith "HTTP:UNAUTHORIZED"       -> 401
    case t if t startsWith "HTTP:FORBIDDEN"          -> 403
    case t if t startsWith "HTTP:NOT_FOUND"          -> 404
    case t if t startsWith "HTTP:TIMEOUT"            -> 504
    case t if t startsWith "HTTP:CONNECTIVITY"       -> 502
    case t if t startsWith "HTTP:RETRY_EXHAUSTED"    -> 502
    case t if t startsWith "MRP:INVALID_TYPE"        -> 400
    case t if t startsWith "ERROR:" and (t endsWith "_NOT_FOUND") -> 404
    else -> 500
}
```

---

## Example 4 — Resource-Specific Error Handler (PAPI)

Source pattern: `acme-orders-papi/src/main/mule/common/error-handler.xml`

When an endpoint needs a custom error response (different schema, extra fields), define a dedicated handler:

```xml
<error-handler name="error-handler-mrp-status"
    doc:name="MRP Status Error Handler">

    <on-error-propagate enableNotifications="true" logException="true"
        doc:name="On Any MRP Error">
        <logger level="ERROR"
            message="MRP error #[error.errorType.identifier]: #[error.description] (correlationId=#[vars.correlationId])"
            doc:name="Log MRP error"/>

        <ee:transform>
            <ee:message>
                <ee:set-payload resource="dwl/errors/mrpRequisitions/mrpStatus.dwl" />
            </ee:message>
            <ee:variables>
                <ee:set-variable variableName="httpStatus"><![CDATA[
                    error.errorMessage.code default 500
                ]]></ee:set-variable>
            </ee:variables>
        </ee:transform>
    </on-error-propagate>
</error-handler>
```

**Reference from a flow:**

```xml
<flow name="post:\mrpRequisitions\status:application\json:acme-orders-papi-config">
    <!-- ... -->
    <error-handler ref="error-handler-mrp-status" />
</flow>
```

---

## Example 5 — Reprocess Error Handler (PAPI Only)

Source: `acme-orders-papi/src/main/mule/common/reprocess.xml`

PAPIs that perform retry-eligible operations include the reprocess pattern. The error handler stores the failed payload to the Object Store; a scheduled flow retries them later.

### 5a — Triggering Reprocess (in the endpoint flow)

```xml
<error-handler name="error-handler-with-reprocess"
    doc:name="PAPI Error Handler With Reprocess">

    <!-- Retry-eligible: store and return 503 -->
    <on-error-propagate
        type="HTTP:CONNECTIVITY,HTTP:TIMEOUT,HTTP:RETRY_EXHAUSTED,MULE:RETRY_EXHAUSTED,APP:REPROCESS"
        doc:name="On Retry-Eligible Error">

        <logger level="ERROR"
            message="Storing payload for reprocess — #[error.errorType.identifier]: #[error.description] (correlationId=#[vars.correlationId])"
            doc:name="Log reprocess store"/>

        <ee:transform>
            <ee:variables>
                <ee:set-variable variableName="objectStoreKey"
                    resource="dwl/error/reprocess/keyError.dwl" />
                <ee:set-variable variableName="objectStoreValue"
                    resource="dwl/error/reprocess/storeError.dwl" />
            </ee:variables>
        </ee:transform>

        <os:store key="#[vars.objectStoreKey]" objectStore="object-store-errors">
            <os:value>#[vars.objectStoreValue]</os:value>
        </os:store>

        <ee:transform>
            <ee:message>
                <ee:set-payload resource="dwl/error/reprocess/responseStoredError.dwl" />
            </ee:message>
            <ee:variables>
                <ee:set-variable variableName="httpStatus"><![CDATA[503]]></ee:set-variable>
            </ee:variables>
        </ee:transform>
    </on-error-propagate>

    <!-- Non-retryable: standard error response -->
    <on-error-propagate type="ANY" doc:name="On Non-Retryable Error">
        <ee:transform>
            <ee:message>
                <ee:set-payload resource="dwl/error/message/genericError.dwl" />
            </ee:message>
            <ee:variables>
                <ee:set-variable variableName="httpStatus"
                    resource="dwl/error/statusCodes/default.dwl" />
            </ee:variables>
        </ee:transform>
    </on-error-propagate>
</error-handler>
```

### 5b — Reprocessing Stored Errors (scheduled sub-flow)

```xml
<sub-flow name="reprocess-all-errors-from-os-subflow">
    <os:retrieve-all objectStore="object-store-errors" />
    <set-variable value="#[[]]" variableName="processedErrorsArray" />

    <foreach collection="#[dw::Core::entriesOf(payload)]">
        <try>
            <ee:transform>
                <ee:set-payload resource="dwl/error/reprocess/errorMessageToReprocess.dwl" />
                <ee:set-variable variableName="objectStoreKeyValue"
                    resource="dwl/vars/reprocess/objectStoreKeyValue.dwl" />
            </ee:transform>

            <os:remove key="#[vars.objectStoreKeyValue]" objectStore="object-store-errors" />

            <flow-ref name="original-implementation-sub-flow" />

            <error-handler>
                <on-error-continue type="ANY" doc:name="On Reprocess Failure - Re-store">
                    <!-- If retry fails again, raise UNPROCESSABLE_CONTENT
                         which gets caught by the outer reprocess handler
                         to put it back in the Object Store. -->
                    <raise-error type="VALIDATION:UNPROCESSABLE_CONTENT"
                        description="Reprocess attempt failed - re-stored" />
                </on-error-continue>
            </error-handler>
        </try>
    </foreach>
</sub-flow>
```

### 5c — Reprocess DWL Files

Located in `src/main/resources/dwl/error/reprocess/`:

| File | Purpose |
|---|---|
| `keyError.dwl` | Generates the Object Store key (typically `correlationId` + timestamp) |
| `storeError.dwl` | Formats the value to store (original payload + error metadata + retry count) |
| `errorMessageObjectStore.dwl` | Builds the object stored in OS during initial failure |
| `errorMessageToReprocess.dwl` | Transforms the stored entry back into the original payload for retry |
| `responseStoredError.dwl` | The 503 response body returned to the caller |

---

## Anti-Patterns (DON'T)

```xml
<!-- ❌ DON'T — error handler without doc:name on the on-error block -->
<error-handler name="my-handler">
    <on-error-propagate type="APIKIT:BAD_REQUEST">
        <ee:transform> ... </ee:transform>
    </on-error-propagate>
</error-handler>

<!-- ❌ DON'T — inline payload instead of DWL file -->
<on-error-propagate type="APIKIT:BAD_REQUEST" doc:name="On Bad Request">
    <ee:transform>
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0 output application/json --- { error: "bad" }]]></ee:set-payload>
        </ee:message>
    </ee:transform>
</on-error-propagate>

<!-- ❌ DON'T — error handler scope with no logger at all -->
<on-error-propagate type="ANY" doc:name="On Any">
    <!-- no <logger level="ERROR"> here means the error is silently consumed -->
    <ee:transform> ... </ee:transform>
</on-error-propagate>

<!-- ❌ DON'T — swallow error with on-error-continue at top level (loses traceability) -->
<on-error-continue type="ANY" doc:name="Swallow All">
    <set-payload value='#[{}]' />
</on-error-continue>

<!-- ❌ DON'T — reprocess in a SAPI or EAPI -->
<!-- The reprocess pattern is PAPI-only — SAPI/EAPI propagate errors to the caller -->
```

---

## Layer Summary

| Layer | Standard Handler | Reprocess? | Resource-Specific Handlers? |
|---|---|---|---|
| **SAPI** | `apikit-error-handler` | ❌ Never | Only when backend has unique error semantics |
| **EAPI** | `apikit-error-handler` | ❌ Never | Common — different consumers want different shapes |
| **PAPI** | `apikit-error-handler` + reprocess sub-handlers | ✅ For data-sync writes | Common per resource |

---

## References

- [Error Handling Standards](../standards/error-handling.md)
- [Error Types Reference](../reference/error-types.md)
- [Logging Standards](../standards/logging.md) and [Logging Usage Examples](./logging.md)
- [Reference Flows](./reference-flows.md)

---
Last updated: 2026-04-13
Owner: Integration Team
