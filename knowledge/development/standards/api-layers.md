# API Layer Patterns (SAPI/PAPI/EAPI)

## Overview

This document describes the rules and responsibilities specific to each API layer in our API-led connectivity architecture. For implementation examples, see [Best Practices](./api-development-best-practices.md) and [Connector Examples](../usage/connector.md).

## System API (SAPI)

### Purpose
System APIs provide a means of accessing underlying systems of record and exposing their data. They abstract the complexity of the backend systems.

### Reference Implementation
`acme-erp-sapi` - Integrates with an ERP backend system

### Key Characteristics

| Aspect | Pattern |
|--------|---------|
| Responsibility | Direct backend connectivity |
| Data scope | Raw or minimally transformed data |
| Consumer | Process APIs only |
| Naming | `{org}-{backend-system}-sapi` |
| **Layer Code** | `sys` (for logging context) |

### Flow Structure: Payload Mapping Location

Payload transformations belong in the implementation sub-flow, **not** in the API router flow. The main flow should only handle: logging start, variable setup, flow-ref to sub-flow, and error handling.

### Backend Query Parameters

Even if the API endpoint doesn't accept query parameters from the client, you may still need to send **fixed query parameters** to the backend. Backend filtering requirements are independent of API contract parameters.

### SAPI Sub-Flow Pattern

```
auth → request → map → respond
```

1. Main flow: logger START → flow-ref to sub-flow → logger END
2. Sub-flow: transform request → call backend → transform response
3. Transformations belong in the sub-flow, NOT in the main/router flow

### SAPI Rules

- **Minimal transformation**: Transform only what's necessary (field mapping, format conversion)
- **No business logic**: Business rules belong in Process APIs
- **Consistent error mapping**: Map backend errors to standard HTTP status codes with `error.type ++ " - " ++ backend error`
- **Backend query params**: Even if the API has no query params, the sub-flow may need to send fixed query params to the backend — extract via DWL in `dwl/vars/`
- **Response status evaluation**: After backend calls, evaluate status code with a Choice router (200 → success, 404 → not found, otherwise → error)
- **Idempotent operations**: Support retry-safe operations
- **Connection pooling**: Configure pools for backend connections
- **Timeouts**: Set appropriate timeouts for backend calls
- **Retry mechanisms**: Implement retries for transient failures
- **Circuit breakers**: Use for unstable backend dependencies

---

## Process API (PAPI)

### Purpose
Process APIs orchestrate data from multiple System APIs and apply business logic. They provide a layer of abstraction between experience and system layers.

### Reference Implementations
- `acme-orders-papi` - Orders orchestration with error persistence
- `acme-customers-papi` - Customer data processing with scheduled jobs

### Key Characteristics

| Aspect | Pattern |
|--------|---------|
| Responsibility | Business logic orchestration |
| Data scope | Aggregated and transformed data |
| Consumer | Experience APIs |
| Naming | `{org}-{domain}-papi` |
| **Layer Code** | `prc` (for logging context) |

### PAPI Sub-Flow Pattern

```
orchestrate → aggregate → logic
```

1. Main flow: logger START → store original request → flow-ref to orchestration sub-flow → logger END
2. Orchestration sub-flow: call SAPI-A (store in `target` var) → call SAPI-B (store in `target` var) → aggregate results
3. For collections: use Foreach with Try/Catch per item — log errors per item with `vars.counter`

### PAPI Rules

- **Orchestrate, don't implement**: Call SAPIs, don't duplicate their logic
- **Use `target` attribute**: Store SAPI responses in variables (`target="sapiAResponse"`) to preserve payload between calls
- **Error persistence**: Use ObjectStore for critical operations that need retry
- **Parallel processing**: Use Scatter-Gather for independent SAPI calls, Parallel-Foreach for batch item processing (set `maxConcurrency="5"`)
- **Foreach error handling**: Wrap each item in Try with On-Error-Continue — one failed item should not stop the loop
- **Transaction management**: Implement saga patterns for distributed transactions
- **Idempotency keys**: Generate and track idempotency keys for retry scenarios
- **Scheduled jobs**: Use CRON expressions for batch processing in dedicated `scheduler.xml`
- **Async notifications**: Use `<async>` scope for fire-and-forget downstream notifications

---

## Experience API (EAPI)

### Purpose
Experience APIs expose data and functionality tailored to specific consumer needs (web, mobile, partners). They shape the response for the consumer.

### Reference Implementation
`acme-analytics-eapi` - Analytics platform consumer API

### Key Characteristics

| Aspect | Pattern |
|--------|---------|
| Responsibility | Consumer-facing interface |
| Data scope | Consumer-specific view |
| Consumer | End users, applications |
| Naming | `{org}-{consumer/channel}-eapi` |
| **Layer Code** | `exp` (for logging context) |

### EAPI Sub-Flow Pattern

```
validate → call PAPIs → shape response
```

1. Main flow: logger START → store original request → flow-ref to sub-flow → (optional) async post-processing → logger END
2. Sub-flow: validate input → transform for PAPI → call PAPI → shape consumer response
3. For PATCH: extract URI params → call PAPI with resource ID → shape response

### EAPI Rules

- **Consumer-centric design**: Shape responses for consumer needs — never expose internal field names
- **Hide complexity**: Don't expose internal error details or system structure
- **Input validation**: Use `validation:is-true` and `raise-error` for input validation before calling PAPIs
- **Date range filtering**: Enforce date range limits to prevent resource exhaustion (e.g., max 90 days)
- **Pagination**: Extract offset/limit from query params, pass to PAPI, shape response with pagination metadata (`data`, `pagination: {offset, limit, total, hasMore}`)
- **Security headers**: Set `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security` via `outboundHeaders`
- **Async post-processing**: Use `<async>` scope for fire-and-forget notifications after the main response
- **Rate limiting**: Implement constraints to prevent abuse
- **Caching**: Cache reference data and slow-changing responses
- **Versioning**: Support API versioning for backwards compatibility

---

## Layer Communication Rules

| From | To | Allowed | Notes |
|------|-----|---------|-------|
| EAPI | PAPI | Yes | Primary pattern |
| EAPI | SAPI | Avoid | Only for simple lookups |
| PAPI | SAPI | Yes | Required pattern |
| PAPI | PAPI | Avoid | Can cause tight coupling |
| SAPI | SAPI | No | Never |
| SAPI | PAPI | No | Inverts hierarchy |

## References

- [Best Practices](./api-development-best-practices.md) — flow patterns, payload location, pagination, security headers
- [Main Development Guide](./api-development-best-practices.md)
- [Error Handling Standards](./error-handling.md)

---
Last updated: 2026-04-09
Owner: Integration Team
