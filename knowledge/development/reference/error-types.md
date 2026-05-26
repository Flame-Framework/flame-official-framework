# Error Types Reference

Catalog of every error type used (or allowed to be raised) across MuleSoft APIs in this framework. The agent must use error types from this catalog — **never invent new namespaces or types without first registering them here**.

For implementation rules see [error-handling.md](../standards/error-handling.md). For real `<error-handler>` XML examples see [error-handling.md](../usage/error-handling.md).

---

## Error Type Format

All error types follow the pattern:

```
NAMESPACE:ERROR_CODE
```

- **NAMESPACE**: UPPER_SNAKE_CASE, identifies the originating module/domain
- **ERROR_CODE**: UPPER_SNAKE_CASE, identifies the specific error
- Example: `APIKIT:BAD_REQUEST`, `APP:VALIDATION`, `MRP:INVALID_TYPE`

---

## Mule Built-in Errors (do not redeclare)

These come from the Mule runtime / connectors. The agent must handle them but never `raise-error` them.

### APIKit (`APIKIT:*`)

Raised by APIKit when the request fails the RAML contract.

| Type | When Raised | Default HTTP |
|---|---|---|
| `APIKIT:BAD_REQUEST` | Invalid request body, schema violation | 400 |
| `APIKIT:NOT_FOUND` | Path doesn't match any RAML resource | 404 |
| `APIKIT:METHOD_NOT_ALLOWED` | Method not defined for path | 405 |
| `APIKIT:NOT_ACCEPTABLE` | `Accept` header not supported | 406 |
| `APIKIT:UNSUPPORTED_MEDIA_TYPE` | `Content-Type` not supported | 415 |

### HTTP Connector (`HTTP:*`)

Raised by `<http:request>` when the outbound call fails.

| Type | When Raised | Default HTTP Mapping |
|---|---|---|
| `HTTP:CONNECTIVITY` | Cannot reach backend (DNS, refused, etc.) | 502 |
| `HTTP:TIMEOUT` | Read or connection timeout | 504 |
| `HTTP:UNAUTHORIZED` | Backend returned 401 | 401 (propagate) |
| `HTTP:FORBIDDEN` | Backend returned 403 | 403 (propagate) |
| `HTTP:NOT_FOUND` | Backend returned 404 | 404 (propagate) |
| `HTTP:BAD_REQUEST` | Backend returned 400 | 400 (propagate) |
| `HTTP:INTERNAL_SERVER_ERROR` | Backend returned 5xx | 502 |
| `HTTP:SECURITY` | TLS / certificate failure | 500 |
| `HTTP:RETRY_EXHAUSTED` | All retry attempts failed | 502 |

### Validation Module (`VALIDATION:*`)

Raised by `<validation:*>` processors.

| Type | When Raised | Default HTTP |
|---|---|---|
| `VALIDATION:INVALID_BOOLEAN` | Failed boolean check | 400 |
| `VALIDATION:NULL` | Null value rejected | 400 |
| `VALIDATION:NOT_MATCHES_REGEX` | String regex check failed | 400 |
| `VALIDATION:INVALID_NUMBER` | Number validation failed | 400 |
| `VALIDATION:INVALID_SIZE` | Size constraint failed | 400 |
| `VALIDATION:UNPROCESSABLE_CONTENT` | **Custom-raised in reprocess flow** to flag retry-eligible | 422 |

### Object Store (`OS:*`)

| Type | When Raised | Default HTTP |
|---|---|---|
| `OS:KEY_NOT_FOUND` | `os:retrieve` couldn't find key | (handle locally, not propagated) |
| `OS:STORE_NOT_AVAILABLE` | Object store unreachable | 500 |

### Mule Core (`MULE:*`)

| Type | When Raised | Default HTTP |
|---|---|---|
| `MULE:EXPRESSION` | DataWeave/expression evaluation failed | 500 |
| `MULE:UNKNOWN` | Unmapped error | 500 |
| `MULE:RETRY_EXHAUSTED` | Until-successful exhausted | 502 |
| `MULE:CRITICAL` | Unrecoverable | 500 |

---

## Custom Error Types

These are raised explicitly via `<raise-error>` in framework flows. **The agent may only raise types listed here.** To add a new type, register it in this table first.

### `APP:*` — Application-level errors

| Type | When to Raise | HTTP Status | Used In |
|---|---|---|---|
| `APP:VALIDATION` | Business validation failed (beyond schema) | 400 | (recommended pattern, used across PAPIs) |
| `APP:BUSINESS_RULE` | Business rule violation (e.g., insufficient stock) | 422 | (recommended pattern) |
| `APP:REPROCESS` | Operation failed but is retry-eligible — triggers Object Store store | 503 | acme-orders-papi, acme-erp-sapi reprocess flows |

### `ERROR:*` — Domain-specific not-found / conflict

| Type | When to Raise | HTTP Status | Real Usage |
|---|---|---|---|
| `ERROR:ORDER_NOT_FOUND` | No order matches the backend query | 404 | `acme-orders-papi/implementation/orders.xml` (description: `"No Order Found"`) |

### `MRP:*` — Material Requirements Planning

| Type | When to Raise | HTTP Status | Real Usage |
|---|---|---|---|
| `MRP:INVALID_TYPE` | MRP requisition type not supported | 400 | `acme-purchase-papi/implementation/mrp/requisitions-management.xml:41` |

### `CUSTOM:*` — Backend-system-specific (avoid for new code)

These exist in legacy code; **prefer `APP:*` for new code**.

| Type | When to Raise | HTTP Status | Real Usage |
|---|---|---|---|
| `CUSTOM:LEGACY_ERROR` | Legacy backend-specific business error | varies | `acme-{system}-sapi/implementation/{resource}.xml` (e.g., description: `"Validation failed for this resource"`) |
| `CUSTOM:CUSTOM_ERROR` | Generic catch-all (deprecated — too vague) | 500 | `acme-{system}-sapi/implementation/{resource}.xml` |

### `VALIDATION:UNPROCESSABLE_CONTENT` — Reprocess Trigger

Raised explicitly inside reprocess flows to mark a request as retry-eligible. The error handler catches it, stores the original payload to the Object Store, and returns 422 to the caller.

```xml
<raise-error type="VALIDATION:UNPROCESSABLE_CONTENT"
    description="Failed to process — stored for retry" />
```

Real usage: `acme-orders-papi/src/main/mule/common/reprocess.xml:168`

---

## Raising an Error — Required Format

```xml
<raise-error type="APP:VALIDATION"
    description="Customer ID must be numeric"
    doc:name="Raise Validation Error" />
```

- **`type`**: REQUIRED — must be from the catalog above
- **`description`**: REQUIRED — human-readable, included in error response
- **`doc:name`**: REQUIRED — see [naming-conventions.md](../../DEV/standards/naming-conventions.md)

---

## Mapping to HTTP Status

Status code mapping is centralized in DWL files under `src/main/resources/dwl/error/statusCodes/`. The error handler reads the file based on `error.errorType`.

Standard mapping table (encoded in `dwl/error/statusCodes/default.dwl`):

| Error Type Pattern | HTTP Status |
|---|---|
| `APIKIT:BAD_REQUEST`, `VALIDATION:*`, `APP:VALIDATION`, `MRP:INVALID_TYPE` | 400 |
| `HTTP:UNAUTHORIZED` | 401 |
| `HTTP:FORBIDDEN` | 403 |
| `APIKIT:NOT_FOUND`, `HTTP:NOT_FOUND`, `ERROR:*_NOT_FOUND` | 404 |
| `APIKIT:METHOD_NOT_ALLOWED` | 405 |
| `APIKIT:NOT_ACCEPTABLE` | 406 |
| `APIKIT:UNSUPPORTED_MEDIA_TYPE` | 415 |
| `APP:BUSINESS_RULE`, `VALIDATION:UNPROCESSABLE_CONTENT` | 422 |
| `APP:REPROCESS` | 503 |
| `HTTP:CONNECTIVITY`, `HTTP:RETRY_EXHAUSTED`, `MULE:RETRY_EXHAUSTED`, `HTTP:INTERNAL_SERVER_ERROR` | 502 |
| `HTTP:TIMEOUT` | 504 |
| Everything else (`MULE:*`, `OS:*`, `CUSTOM:*`, unmapped) | 500 |

---

## Mapping to Response Message

Response message bodies live in `src/main/resources/dwl/error/message/{errorCategory}.dwl`. Standard schema:

```json
{
    "error": {
        "code": "400",
        "message": "Bad request"
    }
}
```

See [error-handling.md](../usage/error-handling.md) for the full DWL templates.

---

## Reprocess-Eligible Errors

When the layer is **PAPI** and the operation is data-sync (write to backend), these errors trigger the Object Store reprocess pattern:

- `HTTP:CONNECTIVITY`
- `HTTP:TIMEOUT`
- `HTTP:RETRY_EXHAUSTED`
- `MULE:RETRY_EXHAUSTED`
- `HTTP:INTERNAL_SERVER_ERROR`
- `APP:REPROCESS` (explicitly raised)

**Not reprocess-eligible** (client error — retrying won't help):
- All `APIKIT:*`
- All `VALIDATION:*` (except `UNPROCESSABLE_CONTENT` which IS the reprocess trigger)
- `HTTP:BAD_REQUEST`, `HTTP:UNAUTHORIZED`, `HTTP:FORBIDDEN`, `HTTP:NOT_FOUND`
- `APP:VALIDATION`, `APP:BUSINESS_RULE`

SAPI and EAPI layers do **not** include the reprocess pattern — they propagate errors back to the caller.

---

## Forbidden Practices

- ❌ Inventing new namespaces (`MY_API:*`, `ORG:*`) — use `APP:*` or register here first
- ❌ Raising without a `description` attribute
- ❌ Catching `MULE:ANY` to swallow errors silently — always log and re-raise/transform
- ❌ Using `CUSTOM:CUSTOM_ERROR` in new code (too vague)
- ❌ Returning HTTP 200 for an error response — use the status code mapping
- ❌ Including stack traces in the response body — log them, don't expose them

---

## Adding a New Error Type

1. Confirm an existing type doesn't already cover the case
2. Add a row to the appropriate custom table above with: type, when-to-raise, HTTP status, location
3. Update `dwl/error/statusCodes/default.dwl` with the new mapping
4. Update `dwl/error/message/*.dwl` if a non-standard message is needed
5. Document in the project's `README.md`

---

## References

- [Error Handling Standards](../standards/error-handling.md)
- [Error Handling Usage Examples](../usage/error-handling.md)
- [Logging Standards](../standards/logging.md) — error-handler logger requirements

---
Last updated: 2026-04-13
Owner: Integration Team
