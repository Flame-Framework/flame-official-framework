# Common-Data RAML Fragment Reference

## Overview

**IMPORTANT**: This fragment MUST be used as a dependency for ALL new RAML API specifications. It provides standardized components that ensure consistency across all APIs in the organization.

## Fragment Details

| Property | Value |
|----------|-------|
| **Asset ID** | `common-data` |
| **Group ID** | `<ANYPOINT_ORG_ID>` |
| **Current Version** | `<COMMON_DATA_VERSION>` |
| **Type** | RAML 1.0 Library Fragment |
| **Main File** | `common-data.raml` |
| **Classifier** | `raml-fragment` |

## Exchange Dependency Declaration

Add this to your API specification's `exchange.json` file:

```json
{
  "dependencies": [
    {
      "groupId": "<ANYPOINT_ORG_ID>",
      "assetId": "common-data",
      "version": "<COMMON_DATA_VERSION>"
    }
  ]
}
```

## RAML Usage

Import the library in your RAML specification:

```raml
#%RAML 1.0
title: Your API Title
version: v1
baseUri: your-api-name/api/

uses:
  common: exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/common-data.raml
```

---

## Available Components

### 1. Resource Types

| Resource Type | Description | File |
|--------------|-------------|------|
| `default-collection-resource-type` | Standard collection resource type with GET, POST, PUT, PATCH, DELETE | `collections/default-collection.raml` |

#### ResourceType Parameters

| Parameter | HTTP Method | Purpose |
|-----------|-------------|---------|
| `errorResponse` | ALL | Error response type for 4xx/5xx responses |
| `objectResponse` | GET | Response type for retrieving data (200) |
| `createObject` | POST | Request body type for creating resources |
| `createResponse` | POST | Response type after creation (201) |
| `putRequestObject` | PUT | Request body type for full update |
| `patchRequestObject` | PATCH | Request body type for partial update |

#### Supported HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Successfully retrieved resource |
| 201 | Created new resource(s) |
| 204 | No Content (PUT/PATCH/DELETE success) |
| 400 | Bad Request - Invalid request |
| 401 | Unauthorized - Not authorized |
| 403 | Forbidden - Access not allowed |
| 404 | Not Found - Resource doesn't exist |
| 429 | Too Many Requests - Rate limit exceeded |
| 500 | Internal Server Error |
| 502 | Bad Gateway - Downstream error |
| 504 | Gateway Timeout - Downstream timeout |

#### Usage Example

```raml
/resources:
  type:
    lib.collectionType:
      errorResponse: lib.errorResponse
      objectResponse: lib.get-resource-response[]
      createObject: lib.resource-request
      createResponse: lib.post-resource-response

  /{resourceId}:
    type:
      lib.collectionType:
        errorResponse: lib.errorResponse
        objectResponse: lib.get-resource-response
        patchRequestObject: lib.resource-request
```

---

### 2. Security Schemes

| Security Scheme | Type | Description | File |
|-----------------|------|-------------|------|
| `client-credentials` | x-custom | Client ID Enforcement policy | `securitySchemas/client-credentials.raml` |
| `basic-authentication` | Basic Authentication | HTTP Basic Auth with client ID/secret | `securitySchemas/basic-authentication.raml` |
| `jwt-token` | OAuth 2.0 | JWT Bearer Token authentication | `securitySchemas/oauth.raml` |

#### client-credentials (RECOMMENDED)

```raml
#%RAML 1.0 SecurityScheme
description: The Client ID Enforcement policy restricts access to a protected resource by allowing requests only from registered client applications.
type: x-custom
displayName: authentication by Client ID Enforcement
describedBy:
  headers:
    client_id:
      description: client Id provided by API Manager
      type: string
    client_secret:
      description: The Client secret key provided by the API Manager
      type: string
  responses:
    401:
      description: Unauthorized or invalid client application credentials
    500:
      description: Bad response from authorization server, or WSDL SOAP Fault error
```

#### Usage

```raml
# In library.raml
securitySchemes:
  client_credentials: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/securitySchemas/client-credentials.raml

# In root API file
securedBy: lib.client_credentials
```

---

### 3. Traits

#### Header Traits

| Trait | Purpose | Headers | File |
|-------|---------|---------|------|
| `correlation-id-trait` | End-to-end transaction tracing | `x-correlation-id` (optional) | `traits/correlation-id-trait.raml` |
| `transaction-id-trait` | Transaction ID across calls | `x-transaction-id` (optional) | `traits/transaction-id-trait.raml` |
| `trackable-trait` | Operation tracking | `x-correlation-id` (optional) | `traits/trackable-trait.raml` |
| `version-trait` | API version header | `version` (e.g., "v1") | `traits/version-trait.raml` |
| `x-source-system-trait` | Source system identification | `x-source-system` (optional) | `traits/x-source-system-trait.raml` |
| `accept-required-trait` | Accept header requirement | `Accept` (json/xml) | `traits/accept-required-trait.raml` |
| `content-type-required-trait` | Content-Type requirement | `Content-Type` (application/json) | `traits/content-type-required-trait.raml` |
| `client-id-trait` | Client credentials headers | `client_id`, `client_secret` | `traits/client-id-trait.raml` |
| `client-authorization-required-trait` | Bearer token requirement | `Authorization` (Bearer token) | `traits/client-authorization-required-trait.raml` |
| `jwt-trait` | JWT authentication | `authorization` (Bearer JWT) | `traits/jwt-trait.raml` |

#### Query Parameter Traits

| Trait | Purpose | Parameters | File |
|-------|---------|------------|------|
| `pageable-trait` | Standard pagination | `limit`, `offset` | `traits/pageable-trait.raml` |
| `pageable-odata-trait` | OData-style pagination | `next` (URL to next page) | `traits/pageable-odata-trait.raml` |
| `sortable-trait` | Result sorting | `orderBy`, `order` (asc/desc) | `traits/sortable-trait.raml` |
| `orderable-trait` | Result ordering | `orderBy`, `orderType` (asc/desc) | `traits/orderable-trait.raml` |
| `queryable-trait` | Filtering and field selection | `select`, `filter` | `traits/queryable-trait.raml` |

#### Trait Details

**pageable-trait:**
```raml
queryParameters:
  limit:
    description: maximum number of records in the request [use 0 for all]
    type: integer
    default: 1
    minimum: 0
    maximum: 99999
    example: 1
  offset:
    description: OFFSET says to skip that many rows before beginning to return rows.
    type: integer
    default: 0
    minimum: 0
    maximum: 99999
    example: 0
```

**sortable-trait:**
```raml
queryParameters:
  orderBy:
    description: "Order by fields: <<fields>>"
    type: string
    required: false
  order:
    description: Order of Sorting
    enum: [desc, asc]
    required: true
```

**queryable-trait:**
```raml
queryParameters:
  select:
    type: string
    required: false
    description: List of fields to be fetched for given entity
    example: "orderId, OrderType, OrderAmount, OrderStatus"
  filter:
    type: string
    required: false
    description: Filter criteria to be used while querying given entity
    example: "orderType eq 'open' and orderStatus eq 'active'"
```

#### Usage

```raml
/resources:
  get:
    is: [lib.pageable-trait, lib.sortable-trait, lib.correlation-id-trait]
    description: Retrieve all resources with pagination and sorting

  post:
    is: [lib.content-type-required-trait, lib.correlation-id-trait]
    description: Create a new resource
```

---

### 4. Data Types

| Type | Description | File |
|------|-------------|------|
| `error-response` | Standardized error response format | `dataTypes/error-response-data-type.raml` |

#### error-response Structure

```raml
#%RAML 1.0 DataType
properties:
  errors:
    type: array
    items:
      properties:
        code: string
        message: string

example:
  errors:
    -
      code: "123456"
      message: "example error"
```

#### Usage

```raml
# In library.raml
types:
  errorResponse: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/dataTypes/error-response-data-type.raml
```

---

### 5. Date/Time Format Patterns (MANDATORY)

**CRITICAL**: All date and time fields in dataTypes MUST use the standardized format patterns from common-data. Do NOT use plain `type: string` for date/time fields.

**File Location**: `dataTypes/commons/patterns/format-pattern.raml`

#### Available Format Types

| Format Type | Pattern | Example | Use Case |
|-------------|---------|---------|----------|
| `dateFormat_aaaa-MM-dd` | yyyy-MM-dd | `"2023-12-31"` | **Standard date (RECOMMENDED)** |
| `dateFormat_ddmmaaaa` | dd/MM/yyyy | `"29/02/2024"` | European date format |
| `dateFormat_mmaaaa` | MM/yyyy | `"12/2023"` | Month/Year only |
| `timeFormat_hhmm` | HH:mm | `"23:59"` | Time without seconds |
| `timeFormat_hhmmss` | HH:mm:ss | `"23:59:59"` | Time with seconds |
| `dateTimeFormat_aaaa-MM-ddThhmmss` | ISO 8601 | `"2023-12-31T23:59:59"` | DateTime without timezone |
| `dateTimeFormat_aaaa-MM-ddThhmmssZ` | ISO 8601 UTC | `"2023-12-31T23:59:59Z"` | **DateTime with UTC (RECOMMENDED)** |
| `dateTimeFormat_ddmmaaaa_hh_mm` | dd/MM/yyyy HH:mm | `"31/12/2023 23:59"` | European datetime |
| `dateTimeFormat_ddmmaaaa_hh_mm_ss` | dd/MM/yyyy HH:mm:ss | `"31/12/2023 23:59:59"` | European datetime with seconds |
| `dateTimeFormat_aaaa-MM-dd_hhmmss` | yyyy-MM-dd HH:mm:ss | `"2023-12-31 23:59:59"` | ISO date with space separator |
| `dateTimeFormat_aaaa/MM/dd_hhmmss` | yyyy/MM/dd HH:mm:ss | `"2023/12/31 23:59:59"` | Alternative datetime |

#### Usage in DataTypes (MANDATORY PATTERN)

**Step 1**: Import the format patterns in your dataType file:
```raml
#%RAML 1.0 DataType

uses:
  format: ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/dataTypes/commons/patterns/format-pattern.raml

type: object
properties:
  # ... your properties
```

**Step 2**: Use the format types for date/time fields:
```raml
properties:
  creationDate:
    type: format.dateFormat_aaaa-MM-dd
    description: Record creation date
    required: true
    example: "2025-01-15"

  lastModifiedDate:
    type: format.dateFormat_aaaa-MM-dd
    description: Last modification date
    required: false
    example: "2025-03-02"

  createdAt:
    type: format.dateTimeFormat_aaaa-MM-ddThhmmssZ
    description: Record creation timestamp
    required: false
    example: "2025-01-15T10:30:00Z"
```

#### WRONG vs CORRECT Examples

**WRONG - Using plain string for dates:**
```raml
# dataTypes/response/quotations.raml
properties:
  validFrom:
    type: string                    # WRONG!
    description: Valid from date
    example: "15-01-2021"           # Non-standard format
```

**CORRECT - Using format patterns:**
```raml
#%RAML 1.0 DataType

uses:
  format: ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/dataTypes/commons/patterns/format-pattern.raml

type: object
properties:
  validFrom:
    type: format.dateFormat_aaaa-MM-dd    # CORRECT!
    description: Valid from date
    required: true
    example: "2021-01-15"                  # Standard format
```

#### Why This Matters

1. **Validation**: Format patterns include regex validation to ensure data integrity
2. **Consistency**: All APIs use the same date/time formats across the organization
3. **Documentation**: Consumers immediately understand the expected format
4. **Interoperability**: Standardized formats reduce integration issues between systems

---

## Complete Example

### library.raml

```raml
#%RAML 1.0 Library

securitySchemes:
  client_credentials: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/securitySchemas/client-credentials.raml

types:
  errorResponse: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/dataTypes/error-response-data-type.raml
  resource-request: !include ../dataTypes/requests/resource-request.raml
  get-resource-response: !include ../dataTypes/responses/get-resource-response.raml
  post-resource-response: !include ../dataTypes/responses/post-resource-response.raml

resourceTypes:
  collectionType: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/collections/default-collection.raml

traits:
  pageable: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/traits/pageable-trait.raml
  sortable: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/traits/sortable-trait.raml
  correlation-id: !include ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/traits/correlation-id-trait.raml
```

### DataType with Date Fields

```raml
#%RAML 1.0 DataType

uses:
  format: ../exchange_modules/<ANYPOINT_ORG_ID>/common-data/<COMMON_DATA_VERSION>/dataTypes/commons/patterns/format-pattern.raml

type: object
displayName: Order Response
description: Represents an order entity
properties:
  orderId:
    type: string
    description: Unique order identifier
    required: true
    example: "ORD-001"
  orderDate:
    type: format.dateFormat_aaaa-MM-dd
    description: Date when order was placed
    required: true
    example: "2025-01-15"
  createdAt:
    type: format.dateTimeFormat_aaaa-MM-ddThhmmssZ
    description: Record creation timestamp
    required: false
    example: "2025-01-15T10:30:00Z"
  status:
    type: string
    description: Order status
    required: true
    enum: [PENDING, CONFIRMED, SHIPPED, DELIVERED]
    example: "PENDING"
```

---

## Required Standard Headers (via Traits)

When designing APIs, apply these traits as appropriate:

### For All Endpoints
- `correlation-id-trait` - Enables request tracing across services

### For GET Collection Endpoints
- `pageable-trait` or `pageable-odata-trait` - Enable pagination
- `sortable-trait` - Enable sorting
- `queryable-trait` - Enable filtering

### For POST/PUT/PATCH Endpoints
- `content-type-required-trait` - Ensure Content-Type header is provided

### For Secured Endpoints
- `client-id-trait` - When using client ID enforcement
- `jwt-trait` - When using JWT authentication

---

## Fragment File Structure

```
common-data/
├── common-data.raml                    # Main library file
├── exchange.json                       # Exchange metadata
├── collections/
│   └── default-collection.raml         # ResourceType definition
├── dataTypes/
│   ├── error-response-data-type.raml   # Error response type
│   └── commons/
│       └── patterns/
│           └── format-pattern.raml     # Date/Time format patterns
├── securitySchemas/
│   ├── basic-authentication.raml       # Basic Auth scheme
│   ├── client-credentials.raml         # Client ID Enforcement
│   └── oauth.raml                      # JWT/OAuth scheme
└── traits/
    ├── accept-required-trait.raml
    ├── client-authorization-required-trait.raml
    ├── client-id-trait.raml
    ├── content-type-required-trait.raml
    ├── correlation-id-trait.raml
    ├── jwt-trait.raml
    ├── orderable-trait.raml
    ├── pageable-odata-trait.raml
    ├── pageable-trait.raml
    ├── queryable-trait.raml
    ├── sortable-trait.raml
    ├── trackable-trait.raml
    ├── transaction-id-trait.raml
    ├── version-trait.raml
    └── x-source-system-trait.raml
```

---

## Best Practices

1. **Always import this fragment** as the first step when creating a new API specification
2. **Use the standardized error-response type** for all error responses (4xx, 5xx)
3. **Apply correlation-id-trait** to all endpoints for observability
4. **Use pageable-trait** for any endpoint returning collections
5. **Apply security schemes globally** and override only when necessary
6. **Use format patterns for ALL date/time fields** - never use plain `type: string`
7. **Check for fragment updates** regularly to benefit from improvements

---

## Updating the Fragment Version

When a new version of common-data is released:

1. Check for updates in Anypoint Exchange
2. Update the version in your `exchange.json`
3. Test your API specification for compatibility
4. Update this reference document with the new version number

---

**Fragment Version**: <COMMON_DATA_VERSION>
**Document Last Updated**: 2026-01-09
**Source**: Anypoint Exchange (<ANYPOINT_ORG_ID>/common-data)
