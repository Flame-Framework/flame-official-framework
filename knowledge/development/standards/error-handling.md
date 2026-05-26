# Error Handling Standards

## Overview

This document defines the error handling patterns and standards for all MuleSoft APIs. Consistent error handling ensures predictable API behavior and simplifies debugging. It establishes patterns for the API Led architecture, standardizing error logging (with the standard Mule `<logger>`) and the error response format.

## API Led Architecture - Error Handling by Layer

Each layer must handle errors independently and propagate relevant information up.

| Layer | Naming Convention | Layer Code | Responsibility |
|-------|-------------------|------------|----------------|
| **Experience API** | `{org}-*-eapi` | `exp` | Presents error messages directly to consumers |
| **Process API** | `{org}-*-papi` | `prc` | Orchestrates errors from multiple system APIs and applies retry logic |
| **System API** | `{org}-*-sapi` | `sys` | Handles errors from integrations with backend systems |

### Layer-Specific Guidelines

**Experience API (EAPI)**
- Return user-friendly messages to consumers
- Never expose internal system details
- Use the standard error response format

**Process API (PAPI)**
- Orchestrate errors from multiple SAPIs
- Apply retry logic where appropriate
- Aggregate error information from downstream services

**System API (SAPI)**
- Check if the backend returns relevant error messages
- Customize the `errors.message` with: `error.type ++ " - " ++ backend error`
- Map backend-specific errors to standard error codes

## Error Response Structure

All APIs must return errors in this standard format:

```json
{
    "error": {
        "code": "500",
        "message": "Internal Server Error"
    }
}
```

### Field Descriptions

| Field | Required | Description |
|-------|----------|-------------|
| `error.code` | Yes | HTTP status code as string (e.g., "400", "500") |
| `error.message` | Yes | Human-readable error message |

### Error Message Format Pattern

For System APIs, construct error messages using this pattern:
```
error.type ++ " - " ++ useful information from the backend
```

Example: `"HTTP:CONNECTIVITY - Backend system unavailable: connection timeout"`

## Error Logging

Use the standard Mule `<logger>` at `level="ERROR"` inside every error handler scope. Keep messages plain text; include the error type and (where useful) the correlation ID inline. Do not embed multi-line DataWeave inside the message.

```xml
<logger level="ERROR"
    message="Error #[error.errorType.identifier]: #[error.description] (correlationId=#[vars.correlationId])"
    doc:name="Log error"/>
```

See [Logging Guidelines](./logging.md) for the full logging rules.

## Global Error Handler Configuration

### File Location
`src/main/mule/common/error-handler.xml`

### Standard Implementation

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:ee="http://www.mulesoft.org/schema/mule/ee/core">

    <!-- Default global error handler (set via <configuration defaultErrorHandler-ref="common-error-handler"/>) -->
    <error-handler name="common-error-handler">
        <on-error-continue type="ANY">
            <logger level="ERROR"
                message="Unhandled error #[error.errorType.identifier]: #[error.description] (correlationId=#[vars.correlationId])"
                doc:name="Log unhandled error"/>
            <ee:transform>
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: {
        code: "500",
        message: error.description default "An unexpected error occurred"
    }
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>
            <set-variable variableName="httpStatus" value="500"/>
        </on-error-continue>
    </error-handler>

    <!-- APIkit error handler (referenced in main flow via <error-handler ref="apikit-error-handler">) -->
    <error-handler name="apikit-error-handler">

        <!-- Validation Errors (400) -->
        <on-error-propagate type="APIKIT:BAD_REQUEST, VALIDATION:INVALID_PAYLOAD">
            <logger level="ERROR" message="Bad request: #[error.description]" doc:name="Log bad request"/>
            <ee:transform>
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: {
        code: "400",
        message: error.description default "Invalid request"
    }
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>
            <set-variable variableName="httpStatus" value="400"/>
        </on-error-propagate>

        <!-- Not Found (404) -->
        <on-error-propagate type="APIKIT:NOT_FOUND">
            <logger level="ERROR" message="Resource not found: #[error.description]" doc:name="Log not found"/>
            <ee:transform>
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: {
        code: "404",
        message: "Resource not found"
    }
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>
            <set-variable variableName="httpStatus" value="404"/>
        </on-error-propagate>

        <!-- Method Not Allowed (405) -->
        <on-error-propagate type="APIKIT:METHOD_NOT_ALLOWED">
            <logger level="ERROR" message="Method not allowed: #[error.description]" doc:name="Log method not allowed"/>
            <ee:transform>
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: {
        code: "405",
        message: "HTTP method not supported"
    }
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>
            <set-variable variableName="httpStatus" value="405"/>
        </on-error-propagate>

        <!-- Unsupported Media Type (415) -->
        <on-error-propagate type="APIKIT:UNSUPPORTED_MEDIA_TYPE">
            <logger level="ERROR" message="Unsupported media type: #[error.description]" doc:name="Log unsupported media type"/>
            <ee:transform>
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: {
        code: "415",
        message: "Content-Type not supported"
    }
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>
            <set-variable variableName="httpStatus" value="415"/>
        </on-error-propagate>

        <!-- Connectivity Errors (503) -->
        <on-error-propagate type="HTTP:CONNECTIVITY">
            <logger level="ERROR" message="Downstream connectivity error: #[error.description]" doc:name="Log connectivity error"/>
            <ee:transform>
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: {
        code: "503",
        message: "Downstream service unavailable"
    }
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>
            <set-variable variableName="httpStatus" value="503"/>
        </on-error-propagate>

        <!-- Timeout Errors (504) -->
        <on-error-propagate type="HTTP:TIMEOUT">
            <logger level="ERROR" message="Downstream timeout: #[error.description]" doc:name="Log timeout"/>
            <ee:transform>
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: {
        code: "504",
        message: "Downstream service timeout"
    }
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>
            <set-variable variableName="httpStatus" value="504"/>
        </on-error-propagate>

        <!-- Catch-All (500) -->
        <on-error-propagate type="ANY">
            <logger level="ERROR"
                message="Unhandled error #[error.errorType.identifier]: #[error.description]"
                doc:name="Log catch-all error"/>
            <ee:transform>
                <ee:message>
                    <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: {
        code: "500",
        message: "An unexpected error occurred"
    }
}]]></ee:set-payload>
                </ee:message>
            </ee:transform>
            <set-variable variableName="httpStatus" value="500"/>
        </on-error-propagate>

    </error-handler>

</mule>
```

## HTTP Status Code Mapping

| Error Type | HTTP Status | Description |
|------------|-------------|-------------|
| `APIKIT:BAD_REQUEST` | 400 | Invalid request payload or parameters |
| `VALIDATION:*` | 400 | Validation failures |
| `APIKIT:UNAUTHORIZED` | 401 | Authentication required |
| `HTTP:FORBIDDEN` | 403 | Insufficient permissions |
| `APIKIT:NOT_FOUND` | 404 | Resource not found |
| `APIKIT:METHOD_NOT_ALLOWED` | 405 | HTTP method not supported |
| `HTTP:CONFLICT` | 409 | Resource conflict |
| `APIKIT:UNSUPPORTED_MEDIA_TYPE` | 415 | Invalid content type |
| `HTTP:TIMEOUT` | 504 | Gateway timeout |
| `HTTP:CONNECTIVITY` | 503 | Service unavailable |
| `ANY` | 500 | Internal server error |

## Error Handler Reference in Flows

When referencing the global error handler in flows, always include the `description` attribute:

```xml
<!-- CORRECT -->
<error-handler ref="apikit-error-handler" description="APIkit error handler"/>

<!-- INCORRECT — missing description -->
<error-handler ref="apikit-error-handler"/>
```

> **Note:** The template defines two error handlers: `apikit-error-handler` (referenced in main API flow, handles APIkit errors) and `common-error-handler` (set as default via `<configuration defaultErrorHandler-ref="common-error-handler"/>`, catches unhandled errors).

## httpStatus Variable Placement

The `httpStatus` variable set to 4xx or 5xx MUST only exist **inside error handlers**, never in the main flow. If the main flow needs to signal an error, use `raise-error` instead:

```xml
<!-- CORRECT — raise error, let error handler set httpStatus -->
<raise-error type="APP:VALIDATION_ERROR" description="Invalid input data"/>

<!-- INCORRECT — setting error httpStatus in main flow -->
<set-variable variableName="httpStatus" value="400"/>
```

## Flow-Level Error Handling

> **Note:** Try-scope error handlers inside flow files are an **allowed exception** to the centralization rule. The global error handler in `error-handler.xml` handles all standard errors, but flow-level try-scope handlers are permitted for specific error interception (e.g., catching HTTP:NOT_FOUND to return empty, or raising custom errors from backend failures).

### Using Try Scope

```xml
<try>
    <http:request config-ref="Backend_Config" path="/resource"/>
<error-handler>
    <on-error-continue type="HTTP:NOT_FOUND">
        <logger level="INFO" message="Resource not found, returning empty" doc:name="Log not-found fallback"/>
        <set-payload value="#[null]"/>
    </on-error-continue>
    <on-error-propagate type="HTTP:CONNECTIVITY">
        <raise-error type="APP:BACKEND_UNAVAILABLE"
            description="Backend service is not available"/>
    </on-error-propagate>
</error-handler>
</try>
```

### Custom Error Types

Define custom error types for business-specific scenarios:

```xml
<!-- Raise custom error -->
<raise-error type="ORDER:INVALID_STATUS"
    description="Order status transition not allowed"/>

<!-- Handle custom error -->
<on-error-propagate type="ORDER:INVALID_STATUS">
    <logger level="ERROR" message="Invalid order status transition: #[error.description]" doc:name="Log invalid status"/>
    <ee:transform>
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    error: {
        code: "422",
        message: error.description
    }
}]]></ee:set-payload>
        </ee:message>
    </ee:transform>
    <set-variable variableName="httpStatus" value="422"/>
</on-error-propagate>
```

## Error Persistence with ObjectStore

For critical operations that require retry capability:

```xml
<sub-flow name="persist-error-for-retry">
    <ee:transform>
        <ee:set-variable variableName="errorRecord"><![CDATA[%dw 2.0
output application/json
---
{
    originalPayload: vars.originalPayload,
    errorType: error.errorType.identifier,
    errorMessage: error.description,
    timestamp: now(),
    retryCount: 0,
    correlationId: vars.correlationId
}]]></ee:set-variable>
    </ee:transform>

    <os:store
        objectStore="Error_Retry_Store"
        key="#[vars.correlationId]"
        value="#[vars.errorRecord]"/>

    <logger level="WARN" message="Error persisted for retry: #[vars.correlationId]" doc:name="Log retry persistence"/>
</sub-flow>
```

### ObjectStore Configuration

```xml
<os:object-store name="Error_Retry_Store"
    persistent="true"
    maxEntries="1000"
    entryTtl="7"
    entryTtlUnit="DAYS"
    expirationInterval="1"
    expirationIntervalUnit="HOURS"/>
```

## Error Handling Best Practices

### Do's

- Always include `correlationId` (inline in the message) in error log lines
- Log errors with sufficient context before returning
- Use specific error types before generic ones
- Map backend errors to appropriate HTTP status codes
- Return user-friendly messages in EAPI responses
- Use ObjectStore for critical operations that need retry

### Don'ts

- Don't expose stack traces in production responses
- Don't log sensitive data in error messages
- Don't swallow errors silently (always log)
- Don't return internal error codes to external consumers
- Don't use `on-error-continue` without proper logging

## Special Error Handling Scenarios

### Scheduler Processing Error

Uses common error handler with manual reprocessing and additional error handling for the initial flow. Apply the error handling defined in this document for the logic flows.

### Broker Processing Error

An email is sent to the operations team for all scenarios covered in the error handler from the subscription from the broker service.

### Custom Error

Errors that are not covered by the current common error handler should have a custom error type defined.

## Error Handling Checklist

### General Configuration
- [ ] Global error handler configured in `common/error-handler.xml`
- [ ] All APIKIT error types handled
- [ ] HTTP connectivity errors handled (timeout, unavailable)
- [ ] Custom business error types defined
- [ ] Error responses follow standard format `{error: {code, message}}`

### Logging & Tracing
- [ ] Every `<on-error-*>` scope contains a `<logger level="ERROR">` with a plain-text message
- [ ] Error messages include the error type and (where useful) `vars.correlationId`
- [ ] Sensitive data excluded from error logs

### Layer-Specific
- [ ] In SAPI, check if backend returns relevant errors and customize `errors.message` with `error.type ++ " - " ++ backend error`
- [ ] ObjectStore configured for retry scenarios (if applicable)

## References

- [Main Development Guide](./api-development-best-practices.md)
- [Logging Guidelines](./logging.md)

---
Last updated: 2026-05-16
Owner: Integration Team
