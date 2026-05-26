# MUnit Templates — Common Patterns (Usage Examples)

> **Purpose:** Cross-layer MUnit patterns used by SAPI / PAPI / EAPI tests — target-attribute mocking, APIKit router mocking, single-item fixtures for `foreach`, generic error-handling patterns, payload-structure inference example, and anti-pattern reference. For layer-specific templates and complete working examples, see [usage/template-sapi.md](./template-sapi.md), [usage/template-papi.md](./template-papi.md), [usage/template-eapi.md](./template-eapi.md). For rules and quick-reference tables, see [standards/template-by-layer.md](../standards/template-by-layer.md).

## Target-Attribute Mocking Pattern

When mocking a processor that uses `target="..."` in the flow, always set the corresponding variable in the mock return:

```xml
<!-- When mocking a processor with target="result" -->
<munit-tools:mock-when processor="http:request">
    <munit-tools:then-return>
        <munit-tools:variables>
            <munit-tools:variable key="result"
                value='#[{status: "success", data: [...]}]'
                mediaType="application/java"/>
        </munit-tools:variables>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

---

## APIKit Router Mocking (Optional)

```xml
<!-- OPTIONAL: Only mock if specifically testing router behavior -->
<munit-tools:mock-when processor="apikit:router" doc:name="Mock APIKit Router (Optional)">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute whereValue="{router-doc-id}" attributeName="doc:id"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[payload]" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

---

## 4. COMMON PATTERNS LIBRARY

### 4.0 Single-item fixture for `foreach` / `parallel-foreach`

Flow pattern:

```xml
<!-- Flow iterates over quotations array (foreach or parallel-foreach) -->
<foreach collection="#[payload.quotations]">
    <!-- Each iteration receives ONE quotation as payload -->
    <http:request target="requesterId" doc:id="abc-123" .../>
</foreach>
```

Fixtures:

```
src/test/resources/test-data/quotations/
├── <backend-1>-get-quotations.json           # Array: {quotations: [...], totalHits: N}
└── <backend-1>-get-quotations-single.json    # Single element from that array
```

Mock inside the iteration — use the **single-item** fixture as payload to preserve the current item:

```xml
<munit-tools:mock-when processor="http:request">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute whereValue="abc-123" attributeName="doc:id"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload
            value="#[%dw 2.0 output application/json --- readUrl('classpath://test-data/quotations/<backend-1>-get-quotations-single.json', 'application/json')]"
            mediaType="application/json" encoding="UTF-8"/>
        <munit-tools:attributes value="#[{'statusCode': 200}]"/>
        <munit-tools:variables>
            <munit-tools:variable key="requesterId"
                value="#[%dw 2.0 output application/json --- readUrl('classpath://test-data/quotations/<broker>-get-requester.json', 'application/json')]"/>
        </munit-tools:variables>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

> **Mock, Verify-Call, and Assert examples:** see [usage/official-guidelines.md](./official-guidelines.md) — sections `Mock When`, `Verify Call`, `Assert That`, and `Common Matcher Snippets`.

### 4.1 Error Handling Patterns

Generic error-handling patterns used across SAPI / PAPI / EAPI. For SAPI-specific HTTP connectivity and timeout examples, see [usage/template-sapi.md § 1.4](./template-sapi.md#14-error-handling-templates).

**Expect an error (propagated out of the flow):**

```xml
<munit:test name="test-error-handling"
            description="Test error scenario"
            expectedErrorType="HTTP:CONNECTIVITY">
    <munit:behavior>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:then-return>
                <munit-tools:error typeId="HTTP:CONNECTIVITY"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="flow-with-http-request"/>
    </munit:execution>
</munit:test>
```

**Test `on-error-continue` handler logic (error is caught, flow keeps going):**

```xml
<munit:test name="test-on-error-continue">
    <munit:behavior>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:then-return>
                <munit-tools:error typeId="HTTP:BAD_REQUEST"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="flow-with-error-handler"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:verify-call processor="mule:logger" times="1"/>
    </munit:validation>
</munit:test>
```

**Error propagated from a mocked flow-ref (common in PAPI/EAPI orchestrations):**

> **Agent note**: `<your-namespace>:CUSTOM_ERROR` is a placeholder. Before applying this snippet to real test code, **ask the user** for their project's error namespace and the specific error identifier (these are typically declared in `src/main/mule/error-types.xml` or the project's `mule-artifact.json`). Substitute both the namespace and the identifier — `<` / `>` are not valid in actual Mule error type IDs.

```xml
<munit:test name="test-error-propagation"
            description="Error is propagated from mocked flow"
            expectedErrorType="<your-namespace>:CUSTOM_ERROR">
    <munit:behavior>
        <munit-tools:mock-when processor="mule:flow-ref">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="name" whereValue="process-data-flow"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:error typeId="<your-namespace>:CUSTOM_ERROR"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="main-flow"/>
    </munit:execution>
</munit:test>
```

**Global error handler (verifies the global handler logs and returns correct error payload):**

```xml
<munit:test name="global-error-handler-logs-error-test"
            description="Test global error handler logs the error correctly">
    <munit:behavior>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:then-return>
                <munit-tools:error typeId="HTTP:INTERNAL_SERVER_ERROR"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="main-api-flow"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:verify-call processor="mule:logger" times="1">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="doc:name" whereValue="Log Error"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>
        <munit-tools:assert-that
            expression="#[payload.errorCode]"
            is="#[MunitTools::equalTo('500')]"/>
    </munit:validation>
</munit:test>
```

### 4.2 Payload Structure Inference from Final Transform — Example

> **Rules & decision tree:** see [standards/template-by-layer.md](../standards/template-by-layer.md) § Payload Structure Inference. This section shows the rules applied to a concrete transform.

#### Example Application

**Given this final transform in the flow:**
```xml
<ee:transform doc:id="abc-123">
    <ee:message>
        <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
if (isEmpty(payload.d.results))
    {
        httpStatus: 204
    }
else
    payload.d.results map {
        supplierId: $.ID,
        name: $.Name,
        address: {
            street: $.Street,
            city: $.City,
            country: $.Country
        }
    }
]]></ee:set-payload>
    </ee:message>
    <ee:variables>
        <ee:set-variable variableName="httpStatus"><![CDATA[%dw 2.0
output application/java
---
if (isEmpty(payload.d.results)) 204 else 200
]]></ee:set-variable>
    </ee:variables>
</ee:transform>
```

**Agent infers:**
1. **Mock input structure:** `{"d": {"results": [...]}}`
2. **Response type:** ARRAY (due to `map`)
3. **Response fields:** `supplierId`, `name`, `address.street`, `address.city`, `address.country`
4. **Conditional logic:** Creates two scenarios (200 with data, 204 empty)
5. **Variable set:** `vars.httpStatus` (must be asserted)

**Generated mock for 200 scenario:**
```xml
<munit-tools:mock-when processor="http:request">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute whereValue="{backend-doc-id}" attributeName="doc:id"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value='#[{
            "d": {
                "results": [{
                    "ID": "0000000001",
                    "Name": "Test Supplier",
                    "Street": "123 Main St",
                    "City": "Sample City",
                    "Country": "XX"
                }]
            }
        }]' mediaType="application/json" encoding="UTF-8"/>
        <munit-tools:attributes value="#[{statusCode: 200}]"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

**Generated assertions for 200 scenario:**
```xml
<munit-tools:assert-that expression="#[payload]" is="#[MunitTools::notNullValue()]"/>
<munit-tools:assert-that expression="#[sizeOf(payload)]" is="#[MunitTools::greaterThan(0)]"/>
<munit-tools:assert-that expression="#[payload[0].supplierId]" is="#[MunitTools::equalTo('0000000001')]"/>
<munit-tools:assert-that expression="#[payload[0].name]" is="#[MunitTools::equalTo('Test Supplier')]"/>
<munit-tools:assert-that expression="#[payload[0].address.country]" is="#[MunitTools::equalTo('XX')]"/>
<munit-tools:assert-that expression="#[vars.httpStatus]" is="#[MunitTools::equalTo(200)]"/>
```

**Generated mock for 204 scenario:**
```xml
<munit-tools:mock-when processor="http:request">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute whereValue="{backend-doc-id}" attributeName="doc:id"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value='#[{"d": {"results": []}}]' mediaType="application/json" encoding="UTF-8"/>
        <munit-tools:attributes value="#[{statusCode: 200}]"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

**Generated assertions for 204 scenario:**
```xml
<munit-tools:assert-that expression="#[vars.httpStatus]" is="#[MunitTools::equalTo(204)]"/>
```

---

## 5. Anti-Pattern Examples

### 5.1 Compilation Killers

```xml
<!-- ❌ WRONG: HTML entities in DataWeave -->
<munit:payload value='#[{"key": &quot;value&quot;}]'/>

<!-- ✅ CORRECT -->
<munit:payload value='#[{key: "value"}]'/>

<!-- ❌ WRONG: Missing MunitTools:: prefix -->
is="#[notNullValue()]"

<!-- ✅ CORRECT -->
is="#[MunitTools::notNullValue()]"

<!-- ❌ WRONG: Missing with-attributes -->
<munit-tools:mock-when processor="http:request">
    <munit-tools:then-return>...</munit-tools:then-return>
</munit-tools:mock-when>

<!-- ✅ CORRECT -->
<munit-tools:mock-when processor="http:request">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute whereValue="doc-id" attributeName="doc:id"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>...</munit-tools:then-return>
</munit-tools:mock-when>
```

### 5.2 Runtime Failures

```xml
<!-- ❌ WRONG: missing correlationId variable when the flow's logger references it -->
<munit:set-event>
    <munit:payload value="{}"/>
</munit:set-event>

<!-- ✅ CORRECT: set every variable the flow's loggers reference -->
<munit:set-event>
    <munit:payload value="{}"/>
    <munit:variables>
        <munit:variable key="correlationId" value='"test-correlation-id-12345"' mediaType="application/java"/>
    </munit:variables>
</munit:set-event>
```

### 5.3 Logic Errors

```xml
<!-- ❌ WRONG: Doc:id mismatch between mock and verify -->
<munit-tools:mock-when processor="http:request">
    <munit-tools:with-attribute whereValue="abc-123" attributeName="doc:id"/>
</munit-tools:mock-when>

<munit-tools:verify-call processor="http:request">
    <munit-tools:with-attribute whereValue="xyz-789" attributeName="doc:id"/> <!-- NO MATCH! -->
</munit-tools:verify-call>
```
