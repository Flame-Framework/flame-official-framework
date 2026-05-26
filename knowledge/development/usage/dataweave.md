# DataWeave Cookbook

Code patterns and examples for DataWeave 2.0 transformations. See [DataWeave Patterns](../standards/dataweave-patterns.md) for rules and folder conventions.

---

### 1. Standard Response Mapping

**Best Practices:** Extract response data to a variable. Add `default []` for null safety.

```dataweave
%dw 2.0
output application/json

var items = payload.value default []
---
{
    items: items map (item) -> {
        id: item.item_id,
        name: item.item_name,
        email: item.contact_email,
        isActive: item.active_flag
    },
    totalHits: sizeOf(items)
}
```

### 2. Conditional Field Inclusion

```dataweave
%dw 2.0
output application/json
---
{
    id: payload.id,
    name: payload.name,
    (email: payload.email) if !isEmpty(payload.email),
    (phone: payload.phone) if payload.phone?,
    (adminInfo: payload.admin) if vars.isAdmin == true
}
```

### EAPI Response Shaping (Consumer-Specific)

```dataweave
%dw 2.0
output application/json
---
{
    customerId: payload.customer_id,
    fullName: payload.first_name ++ " " ++ payload.last_name,
    totalOrders: sizeOf(payload.orders),
    lastOrderDate: max(payload.orders.order_date),
    activeSubscriptions: payload.subscriptions filter $.status == "active"
}
```

### 3. Nested Object Mapping

```dataweave
%dw 2.0
output application/json
---
{
    order: {
        orderId: payload.order_id,
        customer: {
            id: payload.customer.customer_id,
            name: payload.customer.full_name,
            email: payload.customer.email_address
        },
        items: payload.line_items map (item) -> {
            productId: item.product_id,
            quantity: item.qty,
            price: item.unit_price
        }
    }
}
```

### 4. Filtering Collections

```dataweave
%dw 2.0
output application/json
---
{
    activeOrders: payload filter ($.status == "active"),
    highValue: payload filter ($.total > 1000),
    recentOrders: payload filter ($.createdAt > (now() - |P30D|))
}
```

### 5. Grouping Data

```dataweave
%dw 2.0
output application/json
---
payload groupBy $.category map {
    category: $.category[0],
    count: sizeOf($),
    items: $ map {
        id: $.id,
        name: $.name
    }
}
```

### 6. Error Response Format

```dataweave
%dw 2.0
output application/json

// Standard error response format
fun errorResponse(code: String, message: String) = {
    error: {
        code: code,
        message: message
    }
}

// For SAPI: Include error type in message
fun sapiErrorResponse(code: String, errorType: String, backendError: String) = {
    error: {
        code: code,
        message: errorType ++ " - " ++ backendError
    }
}
---
errorResponse(vars.errorCode, vars.errorMessage)
```

> **Standard Format**: All APIs must return errors as `{error: {code, message}}`

### 7. Pagination Response

```dataweave
%dw 2.0
output application/json
---
{
    data: payload.items,
    pagination: {
        offset: vars.offset as Number,
        limit: vars.limit as Number,
        total: payload.totalCount,
        hasMore: (vars.offset + vars.limit) < payload.totalCount,
        (nextOffset: vars.offset + vars.limit) if (vars.offset + vars.limit) < payload.totalCount
    }
}
```

### 8. Date Formatting

```dataweave
%dw 2.0
output application/json
import * from dw::core::Dates
---
{
    isoDate: payload.date as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"},
    displayDate: payload.date as String {format: "dd/MM/yyyy"},
    dueDate: payload.date + |P30D|,
    daysSince: daysBetween(payload.date, now()),
    parsedDate: payload.dateString as Date {format: "yyyy-MM-dd"}
}
```

### 9. Null Safety

```dataweave
%dw 2.0
output application/json
---
{
    name: payload.name default "Unknown",
    email: payload.email ?? payload.alternateEmail ?? "no-email@example.com",
    city: payload.address.city?,
    (manager: payload.manager.name) if payload.manager?
}
```

### 10. Flattening Nested Arrays

```dataweave
%dw 2.0
output application/json
---
payload.departments flatMap (dept) ->
    dept.employees map (emp) -> {
        department: dept.name,
        employeeId: emp.id,
        employeeName: emp.name
    }
```

### 11. Distinct, Aggregations & Merging

```dataweave
// Distinct values
payload distinctBy $.customerId

// Aggregations
{
    totalAmount: sum(payload.amount),
    averageAmount: avg(payload.amount),
    minAmount: min(payload.amount),
    maxAmount: max(payload.amount),
    orderCount: sizeOf(payload)
}

// Merge objects
vars.baseConfig ++ vars.envConfig ++ { timestamp: now() }

// Dynamic key generation
payload reduce ((item, acc = {}) -> acc ++ {(item.key): item.value})
```

### 12. XML to JSON Mapping

```dataweave
%dw 2.0
output application/json
---
{
    orders: payload.Orders.*Order map (order) -> {
        id: order.@id,
        status: order.Status,
        items: order.Items.*Item map (item) -> {
            sku: item.SKU,
            qty: item.Quantity as Number
        }
    }
}
```

### 13. JSON to XML Mapping

```dataweave
%dw 2.0
output application/xml
---
{
    Orders: {
        (payload map (order) -> {
            Order @(id: order.id): {
                Status: order.status,
                Items: {
                    (order.items map (item) -> {
                        Item: {
                            SKU: item.sku,
                            Quantity: item.qty
                        }
                    })
                }
            }
        })
    }
}
```

### 14. Request Validation

```dataweave
%dw 2.0
output application/json

fun validateRequired(value, fieldName) =
    if (isEmpty(value))
        [{field: fieldName, message: "$(fieldName) is required"}]
    else []

fun validateEmail(email, fieldName) =
    if (email matches /^[\w._%+-]+@[\w.-]+\.\w{2,}$/)
        []
    else [{field: fieldName, message: "Invalid email format"}]
---
{
    isValid: isEmpty(vars.errors),
    error: vars.errors
}
```

### 15. Lookup/Join Pattern

```dataweave
%dw 2.0
output application/json
---
vars.orders map (order) -> {
    orderId: order.id,
    customerName: (vars.customers filter ($.id == order.customerId))[0].name default "Unknown",
    productNames: order.productIds map (pid) ->
        (vars.products filter ($.id == pid))[0].name default "Unknown"
}
```

## Performance

```dataweave
// Single pass with reduce (preferred over multiple passes)
payload reduce ((item, acc = {total: 0, count: 0}) -> {
    total: acc.total + item.amount,
    count: acc.count + 1
})

// Chunk large datasets
import * from dw::core::Arrays
payload divideBy 100 map (chunk) -> chunk map (item) -> transform(item)
```

---

## References

- [DataWeave Patterns](../standards/dataweave-patterns.md) — rules and folder conventions
- [DataWeave 2.0 Documentation](https://docs.mulesoft.com/dataweave/2.4/)

---
Last updated: 2026-04-09
Owner: Integration Team
