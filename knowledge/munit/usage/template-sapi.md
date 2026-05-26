# MUnit Templates — SAPI (Usage Examples)

> **Purpose:** Copy-paste MUnit XML templates for SAPI flows (GET with OData query, POST, PATCH, error handling) plus a complete working SAPI test suite. For rules, quick-reference tables, and payload-inference logic, see [standards/template-by-layer.md](../standards/template-by-layer.md) § SAPI. For patterns shared across layers (target-attribute mocks, APIKit router, common-patterns library, anti-patterns), see [usage/template-common.md](./template-common.md).

## 1.0 Bearer Token Flow-Ref Mock (SAPI OAuth2)

When the SAPI uses OAuth2, mock the bearer-token `flow-ref` using its `doc:id` from the flow:

```xml
<!-- In your implementation flow, you have: -->
<flow-ref name="bearer-token" doc:id="abc-123-def"/>

<!-- In your test, mock using the processor's doc:id: -->
<munit-tools:mock-when doc:name="Mock bearer token" processor="flow-ref">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute whereValue="abc-123-def" attributeName="doc:id"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[payload]" encoding="UTF-8"/>
        <munit-tools:variables>
            <munit-tools:variable key="bearerToken" value='#[{"Authorization": "Bearer test-token-123"}]'/>
        </munit-tools:variables>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

## 1.1 GET Endpoint Template with OData Query

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:munit="http://www.mulesoft.org/schema/mule/munit"
      xmlns:munit-tools="http://www.mulesoft.org/schema/mule/munit-tools"
      xsi:schemaLocation="http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd
                          http://www.mulesoft.org/schema/mule/munit http://www.mulesoft.org/schema/mule/munit/current/mule-munit.xsd
                          http://www.mulesoft.org/schema/mule/munit-tools http://www.mulesoft.org/schema/mule/munit-tools/current/mule-munit-tools.xsd">

    <!-- GET with OData Query Parameters - Happy Path -->
    <munit:test name="get-{entity}-test-200-ok" description="Test GET {entity} returns 200 with data">

        <munit:behavior>
            <!-- Optional: Mock Bearer Token Flow (if API uses OAuth2) -->
            <!-- Uncomment if your API requires bearer token authentication
            <munit-tools:mock-when processor="flow-reference" doc:name="Mock Bearer Token Flow">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="bearer-token-flow" attributeName="name"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <munit-tools:variables>
                        <munit-tools:variable key="bearerToken" value="#['test-bearer-token']"/>
                    </munit-tools:variables>
                </munit-tools:then-return>
            </munit-tools:mock-when>
            -->

            <!-- Mock HTTP Request to Backend (OData / REST / SOAP) -->
            <munit-tools:mock-when processor="http:request" doc:name="Mock HTTP Request">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{backend-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <munit-tools:payload value="#[MunitTools::getResourceAsString('test-data/{entity}/<backend-1>-get-{entity}.json')]"
                                        mediaType="application/json"
                                        encoding="UTF-8"/>
                    <munit-tools:attributes value="#[{
                        statusCode: 200,
                        reasonPhrase: 'OK',
                        headers: {
                            'x-correlation-id': 'test-correlation-id'
                        }
                    }]"/>
                </munit-tools:then-return>
            </munit-tools:mock-when>
        </munit:behavior>

        <munit:execution>
            <munit:set-event doc:name="Set Event with Query Parameters">
                <munit:attributes value='#[{
                    "method": "GET",
                    "requestPath": "/api/{entity}",
                    "queryParams": {
                        "id": "12345",
                        "$filter": "status eq '\''active'\''",
                        "$top": "10"
                    },
                    "headers": {
                        "x-correlation-id": "test-correlation-id"
                    }
                }]' mediaType="application/json"/>
                <munit:variables>
                    <!-- For OData queries, set the filter as a variable -->
                    <munit:variable key="id" value="#[&quot;id eq '12345'&quot;]"/>
                    <munit:variable key="correlationId" value='"test-correlation-id-12345"' mediaType="application/java"/>
                </munit:variables>
            </munit:set-event>
            <!-- Always call the main APIKit-routed flow -->
            <flow-ref name="get:{entity}:{api-config-name}" doc:name="Invoke main flow"/>
        </munit:execution>

        <munit:validation>
            <!-- Verify HTTP request was called exactly once -->
            <munit-tools:verify-call processor="http:request" times="1" doc:name="Verify HTTP called once">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{backend-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
            </munit-tools:verify-call>

            <!-- Assert payload structure -->
            <munit-tools:assert-that expression="#[payload]"
                                    is="#[MunitTools::notNullValue()]"
                                    doc:name="Assert payload not null"/>

            <!-- Assert specific fields for SAPI transformation validation -->
            <munit-tools:assert-that expression="#[payload[0].id]"
                                    is="#[MunitTools::equalTo('12345')]"
                                    doc:name="Assert ID field"/>

            <munit-tools:assert-that expression="#[payload[0].name]"
                                    is="#[MunitTools::notNullValue()]"
                                    doc:name="Assert name exists"/>

            <!-- Assert status code -->
            <munit-tools:assert-that expression="#[attributes.statusCode]"
                                    is="#[MunitTools::anyOf([MunitTools::equalTo(200), MunitTools::equalTo('200')])]"
                                    doc:name="Assert status 200"/>
        </munit:validation>
    </munit:test>

    <!-- GET Empty Scenario - 204 No Content -->
    <munit:test name="get-{entity}-test-204-no-content" description="Test GET {entity} with empty backend returns 204">
        <munit:behavior>
            <munit-tools:mock-when processor="http:request">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{backend-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <!-- Backend returns empty result -->
                    <munit-tools:payload value="#[{value: []}]" mediaType="application/json" encoding="UTF-8"/>
                    <munit-tools:attributes value="#[{statusCode: 200}]"/>
                </munit-tools:then-return>
            </munit-tools:mock-when>
        </munit:behavior>

        <munit:execution>
            <!-- Same as happy path -->
        </munit:execution>

        <munit:validation>
            <!-- Empty result should return 204 No Content -->
            <munit-tools:assert-that expression="#[vars.httpStatus]"
                                    is="#[MunitTools::equalTo(204)]"
                                    doc:name="Assert status 204"/>
            <munit-tools:assert-equals actual="#[payload]"
                                      expected="#[{}]"
                                      doc:name="Assert empty payload"/>
        </munit:validation>
    </munit:test>
</mule>
```

## 1.2 POST Endpoint Template

```xml
<!-- POST with Request Body - Happy Path -->
<munit:test name="post-{entity}-test-201-created" description="Test POST {entity} creates resource">

    <munit:behavior>
        <!-- Optional: Mock Bearer Token Flow (if API uses OAuth2) -->
        <!-- Uncomment if your API requires bearer token authentication
        <munit-tools:mock-when processor="flow-reference" doc:name="Mock Bearer Token Flow">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="bearer-token-flow" attributeName="name"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:variables>
                    <munit-tools:variable key="bearerToken" value="#['test-bearer-token']"/>
                </munit-tools:variables>
            </munit-tools:then-return>
        </munit-tools:mock-when>
        -->

        <!-- Mock HTTP POST to Backend -->
        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="{backend-post-doc-id}" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[MunitTools::getResourceAsString('test-data/{entity}/<backend-1>-post-{entity}.json')]" encoding="UTF-8"/>
                <munit-tools:attributes value="#[{
                    statusCode: 201,
                    headers: {
                        'Location': '/api/{entity}/new-id-123'
                    }
                }]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>

    <munit:execution>
        <munit:set-event>
            <munit:payload value="#[MunitTools::getResourceAsString('test-data/{entity}/apikit-payload-{entity}.json')]"
                          mediaType="application/json"/>
            <munit:attributes value='#[{
                "method": "POST",
                "requestPath": "/api/{entity}"
            }]'/>
            <munit:variables>
                <munit:variable key="correlationId" value='"test-correlation-id-12345"' mediaType="application/java"/>
            </munit:variables>
        </munit:set-event>
        <flow-ref name="post-{entity}-request"/>
    </munit:execution>

    <munit:validation>
        <munit-tools:assert-that expression="#[attributes.statusCode]"
                                is="#[MunitTools::equalTo(201)]"/>
        <munit-tools:assert-that expression="#[payload.id]"
                                is="#[MunitTools::equalTo('new-id-123')]"/>
    </munit:validation>
</munit:test>

<!-- POST Validation Error -->
<munit:test name="post-{entity}-test-400-bad-request" description="Test POST {entity} validation error">
    <munit:behavior>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="{backend-post-doc-id}" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:error typeId="HTTP:BAD_REQUEST">
                    <munit-tools:error-message>
                        <munit-tools:payload value='#[{
                            "error": {
                                "code": "VALIDATION_ERROR",
                                "message": "Required field missing: name"
                            }
                        }]' mediaType="application/json" encoding="UTF-8"/>
                    </munit-tools:error-message>
                </munit-tools:error>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <!-- Rest of test -->
</munit:test>
```

## 1.3 PATCH Endpoint Template

```xml
<!-- PATCH with URI Parameters -->
<munit:test name="patch-{entity}-test-204-no-content" description="Test PATCH {entity} updates resource">

    <munit:behavior>
        <!-- Optional: Mock Bearer Token Flow (if API uses OAuth2) -->
        <!-- Uncomment if your API requires bearer token authentication
        <munit-tools:mock-when processor="flow-reference" doc:name="Mock Bearer Token Flow">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="bearer-token-flow" attributeName="name"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:variables>
                    <munit-tools:variable key="bearerToken" value="#['test-bearer-token']"/>
                </munit-tools:variables>
            </munit-tools:then-return>
        </munit-tools:mock-when>
        -->

        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="{backend-patch-doc-id}" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[null]" encoding="UTF-8"/>
                <munit-tools:attributes value="#[{statusCode: 204}]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>

    <munit:execution>
        <munit:set-event>
            <munit:payload value="#[MunitTools::getResourceAsString('test-data/{entity}/apikit-payload-{entity}.json')]"/>
            <munit:attributes value='#[{
                "method": "PATCH",
                "requestPath": "/api/{entity}/12345",
                "uriParams": {
                    "id": "12345"
                }
            }]'/>
            <munit:variables>
                <munit:variable key="id" value="#['12345']"/>
                <munit:variable key="correlationId" value='"test-correlation-id-12345"' mediaType="application/java"/>
            </munit:variables>
        </munit:set-event>
        <flow-ref name="patch-{entity}-request"/>
    </munit:execution>

    <munit:validation>
        <munit-tools:assert-that expression="#[attributes.statusCode]"
                                is="#[MunitTools::equalTo(204)]"/>
    </munit:validation>
</munit:test>
```

## 1.4 Error Handling Templates

SAPI-specific HTTP error templates. For generic error-handling patterns that apply across all layers (expect-an-error, on-error-continue, mocked-flow propagation, global error handler), see [usage/template-common.md § 4.1 Error Handling Patterns](./template-common.md#41-error-handling-patterns).

```xml
<!-- HTTP Connectivity Error -->
<munit:test name="get-{entity}-test-503-connectivity" description="Test connectivity error handling">
    <munit:behavior>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="{backend-http-doc-id}" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:error typeId="HTTP:CONNECTIVITY"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>

    <munit:execution>
        <!-- Same setup as happy path -->
    </munit:execution>

    <munit:validation>
        <munit-tools:assert-that expression="#[payload.errors[0].code]"
                                is="#[MunitTools::equalTo('503')]"/>
        <munit-tools:assert-that expression="#[payload.errors[0].message]"
                                is="#[MunitTools::containsString('Service Unavailable')]"/>
    </munit:validation>
</munit:test>

<!-- HTTP Timeout Error -->
<munit:test name="get-{entity}-test-504-timeout" description="Test timeout error handling">
    <munit:behavior>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="{backend-http-doc-id}" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:error typeId="HTTP:TIMEOUT"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <!-- Rest of test -->
</munit:test>
```

---

## Complete Working SAPI Test Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:munit="http://www.mulesoft.org/schema/mule/munit"
      xmlns:munit-tools="http://www.mulesoft.org/schema/mule/munit-tools"
      xsi:schemaLocation="http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd
                          http://www.mulesoft.org/schema/mule/munit http://www.mulesoft.org/schema/mule/munit/current/mule-munit.xsd
                          http://www.mulesoft.org/schema/mule/munit-tools http://www.mulesoft.org/schema/mule/munit-tools/current/mule-munit-tools.xsd">

    <munit:config name="suppliers-test-suite.xml"/>

    <!-- Complete SAPI Test: GET Suppliers from <backend-1> -->
    <munit:test name="get-suppliers-test-200-ok" description="Test GET suppliers returns 200 with transformed data">

        <munit:behavior>
            <!-- Mock <backend-1> HTTP request -->
            <munit-tools:mock-when processor="http:request" doc:name="Mock <backend-1> HTTP">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="81f8ed56-ab43-4e52-b75e-857313b2ff96" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <munit-tools:payload value='#[{
                        "d": {
                            "results": [{
                                "ID": "0000000001",
                                "Name": "Supplier ABC Ltd",
                                "Street": "123 Main Street",
                                "City": "Sample City",
                                "Country": "XX",
                                "TaxID": "XX123456789"
                            }]
                        }
                    }]' mediaType="application/json" encoding="UTF-8"/>
                    <munit-tools:attributes value="#[{
                        statusCode: 200,
                        reasonPhrase: 'OK',
                        headers: {
                            'x-correlation-id': 'test-corr-id-123'
                        }
                    }]"/>
                </munit-tools:then-return>
            </munit-tools:mock-when>
        </munit:behavior>

        <munit:execution>
            <munit:set-event doc:name="Set GET suppliers event">
                <munit:attributes value='#[{
                    "method": "GET",
                    "requestPath": "/api/v2/suppliers",
                    "queryParams": {
                        "$filter": "Country eq '\''XX'\''",
                        "$top": "100"
                    },
                    "headers": {
                        "x-correlation-id": "test-corr-id-123"
                    }
                }]' mediaType="application/json"/>
                <munit:variables>
                    <munit:variable key="filter" value="#[&quot;Country eq 'XX'&quot;]"/>
                    <munit:variable key="correlationId" value='"test-corr-id-123"' mediaType="application/java"/>
                </munit:variables>
            </munit:set-event>
            <flow-ref name="get-suppliers-subflow" doc:name="Invoke suppliers subflow"/>
        </munit:execution>

        <munit:validation>
            <!-- Verify backend called once -->
            <munit-tools:verify-call processor="http:request" times="1" doc:name="Verify <backend-1> called">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="81f8ed56-ab43-4e52-b75e-857313b2ff96" attributeName="doc:id"/>
                </munit-tools:with-attributes>
            </munit-tools:verify-call>

            <!-- Assert transformed payload structure -->
            <munit-tools:assert-that expression="#[payload]"
                                    is="#[MunitTools::notNullValue()]"
                                    doc:name="Payload not null"/>

            <munit-tools:assert-that expression="#[sizeOf(payload)]"
                                    is="#[MunitTools::equalTo(1)]"
                                    doc:name="One supplier returned"/>

            <!-- Assert field-level transformations -->
            <munit-tools:assert-that expression="#[payload[0].supplierId]"
                                    is="#[MunitTools::equalTo('0000000001')]"
                                    doc:name="Supplier ID mapped"/>

            <munit-tools:assert-that expression="#[payload[0].name]"
                                    is="#[MunitTools::equalTo('Supplier ABC Ltd')]"
                                    doc:name="Name mapped"/>

            <munit-tools:assert-that expression="#[payload[0].country]"
                                    is="#[MunitTools::equalTo('XX')]"
                                    doc:name="Country mapped"/>

            <!-- Assert HTTP status -->
            <munit-tools:assert-that expression="#[attributes.statusCode]"
                                    is="#[MunitTools::equalTo(200)]"
                                    doc:name="Status 200"/>
        </munit:validation>
    </munit:test>
</mule>
```
