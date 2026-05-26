# MuleSoft MUnit — Allowed Components (Usage Examples)

> **Purpose:** Code examples (XML and DWL) for the approved components catalogued in [reference/allowed-components-list.md](../reference/allowed-components-list.md). Use this file as a copy-paste source for writing tests, flows, configs, and error handlers. For component priorities, tables, and what-is-allowed decisions, see the reference file.

---

## Required MUnit Namespace Declarations

Every MUnit test file must include these namespace declarations:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"
      xmlns:munit="http://www.mulesoft.org/schema/mule/munit"
      xmlns:munit-tools="http://www.mulesoft.org/schema/mule/munit-tools"
      xsi:schemaLocation="
        http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd
        http://www.mulesoft.org/schema/mule/munit http://www.mulesoft.org/schema/mule/munit/current/mule-munit.xsd
        http://www.mulesoft.org/schema/mule/munit-tools http://www.mulesoft.org/schema/mule/munit-tools/current/mule-munit-tools.xsd">
```

Add these namespaces only when mocking their respective connectors:

```xml
xmlns:ee="http://www.mulesoft.org/schema/mule/ee/core"              <!-- mocking/spying DataWeave transforms -->
xmlns:os="http://www.mulesoft.org/schema/mule/os"                   <!-- mocking Object Store operations -->
```

---

## Core Mule Components

### HTTP

```xml
<!-- HTTP Listener - REST API endpoint -->
<http:listener doc:name="Listener"
               config-ref="HTTP_Listener_config"
               path="${http.path}"/>

<!-- HTTP Request - outbound call -->
<http:request doc:name="Request"
              config-ref="HTTP_Request_configuration"
              method="POST"
              path="${endpoint.path}"/>
```

### APIkit

```xml
<apikit:router doc:name="APIkit Router" config-ref="api-config"/>
<apikit:console doc:name="APIkit Console" config-ref="api-config"/>
```

### Logger (standard Mule)

```xml
<logger level="INFO" message="#[vars.logMessage]" doc:name="Log message"/>
```

### SAP

```xml
<sap:sync-rfc doc:name="Execute BAPI"
              config-ref="SAP_Config"
              key="#[vars.functionModule]"/>
```

### Object Store

```xml
<os:store doc:name="Store"
          objectStore="Object_store"
          key="#[vars.key]"/>

<os:retrieve doc:name="Retrieve"
             objectStore="Object_store"
             key="#[vars.key]"/>

<os:contains doc:name="Contains"
             objectStore="Object_store"
             key="#[vars.key]"/>
```

### Validation

```xml
<validation:is-null doc:name="Is null"
                    config-ref="Validation_Config"
                    value="#[payload]"
                    message="Value cannot be null"/>

<validation:is-false doc:name="Is false"
                     config-ref="Validation_Config"
                     expression="#[vars.condition]"
                     message="Validation failed"/>
```

---

## Mandatory Test Structure

`<munit:set-event>` must be the **first element inside `<munit:execution>`** (not inside `<munit:behavior>`). It sets up the initial event before the flow under test runs.

```xml
<munit:test name="test-name-test"
            doc:id="unique-id"
            description="Test description">

    <!-- Behavior: Setup mocks and test data -->
    <munit:behavior>
        <!-- Mocks go here -->
    </munit:behavior>

    <!-- Execution: set-event FIRST, then flow-ref -->
    <munit:execution>
        <munit:set-event doc:name="Set Event">
            <munit:payload value="#[readUrl('classpath://test-data/sample-request.json')]" mediaType="application/json"/>
            <munit:attributes value="#[{queryParams: {id: '123'}}]"/>
            <munit:variables>
                <munit:variable key="correlationId" value='"test-correlation-id-12345"' mediaType="application/java"/>
            </munit:variables>
        </munit:set-event>
        <flow-ref doc:name="Flow-ref to flow-under-test"
                  name="flow-under-test"/>
    </munit:execution>

    <!-- Validation: Assert results -->
    <munit:validation>
        <!-- Assertions go here -->
    </munit:validation>

</munit:test>
```

### Tests with expected errors — omit `<munit:validation>`

```xml
<!-- WRONG: Empty validation with only comments -->
<munit:test name="test-error" expectedErrorType="HTTP:CONNECTIVITY">
    <munit:behavior>...</munit:behavior>
    <munit:execution>...</munit:execution>
    <munit:validation>
        <!-- Error is expected, validation handled by expectedErrorType -->
    </munit:validation>
</munit:test>

<!-- CORRECT: Omit validation section for error tests -->
<munit:test name="test-error" expectedErrorType="HTTP:CONNECTIVITY">
    <munit:behavior>...</munit:behavior>
    <munit:execution>...</munit:execution>
    <!-- No validation section - expectedErrorType handles it -->
</munit:test>
```

---

## Mocking Patterns

### Pattern 1: Mock APIkit Router (most common)

```xml
<munit-tools:mock-when processor="apikit:router" doc:id="mock-router-id" doc:name="Mock APIKit Router">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute whereValue="router-doc-id" attributeName="doc:id"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[output application/json --- { status: 'success' }]" mediaType="application/json" encoding="UTF-8"/>
        <munit-tools:attributes value="#[{statusCode: 200}]"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Pattern 2: Mock HTTP Request

```xml
<munit-tools:mock-when doc:name="Mock http:request"
                       processor="http:request">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id"
                                    whereValue="http-request-doc-id"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[MunitTools::getResourceAsString('classpath://test-data/response.json')]"
                             mediaType="application/json" encoding="UTF-8"/>
        <munit-tools:attributes value="#[{statusCode: 200}]"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Pattern 3: Mock Object Store Retrieve All

```xml
<munit-tools:mock-when processor="os:retrieve-all" doc:id="mock-os-id" doc:name="Mock OS Retrieve All">
    <munit-tools:then-return>
        <munit-tools:payload value="#[output application/json --- [{ key: 'value' }]]" mediaType="application/json" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Pattern 4: Mock Object Store Retrieve Single

```xml
<munit-tools:mock-when processor="os:retrieve" doc:id="mock-os-id" doc:name="Mock OS Retrieve">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute whereValue="os-retrieve-doc-id" attributeName="doc:id"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[{ data: 'value' }]" mediaType="application/json" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Pattern 5: Mock flow-ref

```xml
<munit-tools:mock-when doc:name="Mock flow-ref"
                       processor="flow-ref">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="name"
                                    whereValue="subflow-name"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[{}]" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Pattern 6: Mock SAP sync-rfc

```xml
<munit-tools:mock-when doc:name="Mock sap:sync-rfc"
                       processor="sap:sync-rfc">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:name"
                                    whereValue="Execute BAPI"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[MunitTools::getResourceAsString('sap-response.xml')]"
                             mediaType="application/xml" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Error Mocking

```xml
<munit-tools:mock-when doc:name="Mock with error"
                       processor="http:request">
    <munit-tools:then-return>
        <munit-tools:error typeId="HTTP:CONNECTIVITY"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

---

## Event Setup

### Compact form (most common)

```xml
<munit:set-event doc:name="Set Event">
    <munit:payload value="#[MunitTools::getResourceAsString('request.json')]"
                   mediaType="application/json"/>
    <munit:attributes value="#[{
        headers: {
            'correlation-id': 'test-correlation-id',
            'content-type': 'application/json'
        },
        queryParams: {
            'param1': 'value1'
        },
        uriParams: {
            'id': '123'
        }
    }]"/>
    <munit:variables>
        <munit:variable key="httpStatus" value="200"/>
        <munit:variable key="correlationId" value="test-correlation-id"/>
    </munit:variables>
</munit:set-event>
```

### Full HTTP attributes template (adapt per test case)

The complete set of HTTP attributes Mule exposes at runtime. Use this as a base and trim to only the keys the flow actually reads. Replace `{METHOD}`, `{resource}`, `{id}`, query/uri params, and headers for the specific endpoint and scenario.

```xml
<munit:attributes value="#[{
    headers: {
        'client_id': 'test-client-id',
        'client_secret': 'test-client-secret',
        'accept': '*/*',
        'content-type': 'application/json'
    },
    clientCertificate: null,
    method: '{METHOD}',
    scheme: 'http',
    queryParams: { },
    requestUri: '/api/{resource}/{id}',
    queryString: '',
    version: 'HTTP/1.1',
    listenerPath: '/api/*',
    localAddress: '/127.0.0.1:8091',
    relativePath: '/api/{resource}/{id}',
    uriParams: { 'id': '{id}' },
    rawRequestPath: '/api/{resource}/{id}',
    rawRequestUri: '/api/{resource}/{id}',
    remoteAddress: '/127.0.0.1:51671',
    requestPath: '/api/{resource}/{id}'
}]"/>
```

**Adaptation notes:**
- `method`: `GET`, `POST`, `PUT`, `PATCH`, or `DELETE` as needed.
- `queryParams`: Populate for list/search endpoints (e.g., `{ 'status': 'active', 'page': '1' }`). Leave `{}` for endpoints without query parameters. When populated, also update `requestUri`, `queryString`, and `rawRequestUri` to include the query string.
- `uriParams`: Populate for endpoints with path parameters (e.g., `{ 'materialId': 'MAT-12345' }`). Leave `{}` for collection endpoints without path params.
- `headers`: Include only headers the flow reads (e.g., `client_id`, `client_secret`, custom headers).
- Paths (`requestUri`, `relativePath`, `rawRequestPath`, `rawRequestUri`, `requestPath`): Update all to match the actual endpoint path.
- If the flow does not reference advanced attributes like `scheme`, `listenerPath`, `localAddress`, etc., omit them and use only the keys the flow needs.

---

## Assertion Patterns

### Pattern 1: Assert Not Null (most common)

```xml
<munit-tools:assert-that doc:name="Assert payload not null"
                         expression="#[payload]"
                         is="#[MunitTools::notNullValue()]"
                         message="Payload should not be null"/>
```

### Pattern 2: Assert Equals Value

```xml
<munit-tools:assert-that doc:name="Assert status equals success"
                         expression="#[payload.status]"
                         is="#[MunitTools::equalTo('success')]"/>
```

### Pattern 3: Assert HTTP Status Code

```xml
<munit-tools:assert-that doc:name="Assert HTTP 200"
                         expression="#[attributes.statusCode]"
                         is="#[MunitTools::equalTo(200)]"/>
```

### Pattern 4: Simple Assert Equals

```xml
<munit-tools:assert-equals doc:name="Assert HTTP status"
                           actual="#[vars.httpStatus]"
                           expected="200"/>
```

> `vars.httpStatus` is an Integer — use `equalTo(400)` not `equalTo('400')`. See the reference file for full type-matching rules.

### Pattern 5: Assert Null Value

```xml
<munit-tools:assert-that doc:name="Assert no error"
                         expression="#[payload.errorMessage]"
                         is="#[MunitTools::nullValue()]"/>
```

### Pattern 6: Assert Contains String

```xml
<munit-tools:assert-that doc:name="Assert contains expected text"
                         expression="#[payload.message]"
                         is="#[MunitTools::containsString('expected')]"/>
```

### Pattern 7: Assert Any Of (multiple valid values)

```xml
<munit-tools:assert-that doc:name="Assert valid status"
                         expression="#[attributes.statusCode]"
                         is="#[MunitTools::anyOf([MunitTools::equalTo(200), MunitTools::equalTo('200')])]"/>
```

### Pattern 8: Custom DataWeave Assertion

```xml
<munit-tools:assert doc:id="assert-id" doc:name="Assert custom condition">
    <munit-tools:that><![CDATA[#[
        import * from dw::test::Asserts
        ---
        vars.logInfo must eachItem(haveKey("PERNR"))
    ]]]></munit-tools:that>
</munit-tools:assert>
```

---

## Verification Patterns

### Pattern 1: Verify Called Exactly N Times

```xml
<munit-tools:verify-call doc:name="Verify router called once"
                         processor="apikit:router"
                         times="1">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id"
                                    whereValue="router-doc-id"/>
    </munit-tools:with-attributes>
</munit-tools:verify-call>
```

### Pattern 2: Verify Called At Least N Times

```xml
<munit-tools:verify-call doc:name="Verify http request called"
                         processor="http:request"
                         atLeast="1">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id"
                                    whereValue="http-request-doc-id"/>
    </munit-tools:with-attributes>
</munit-tools:verify-call>
```

### Pattern 3: Verify NOT Called (`times="0"`)

```xml
<munit-tools:verify-call doc:name="Verify OS remove not called"
                         processor="os:remove"
                         times="0">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id"
                                    whereValue="os-remove-doc-id"/>
    </munit-tools:with-attributes>
</munit-tools:verify-call>
```

---

## Spy Patterns

### Spy on Transform to Capture Data

```xml
<munit-tools:spy processor="ee:transform" doc:id="spy-id" doc:name="Spy on transform">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute whereValue="transform-doc-id" attributeName="doc:id"/>
    </munit-tools:with-attributes>
    <munit-tools:before-call>
        <set-variable variableName="beforeTransform" value="#[payload]"/>
    </munit-tools:before-call>
    <munit-tools:after-call>
        <set-variable variableName="afterTransform" value="#[payload]"/>
    </munit-tools:after-call>
</munit-tools:spy>
```

---

## MunitTools Utility Usage

### Load test resource file

```xml
#[MunitTools::getResourceAsString('classpath://test-data/response.json')]
#[MunitTools::getResourceAsString('classpath://clients/odata-response.dwl')]
```

### DataWeave test module

```dataweave
import * from dw::test::Asserts
---
vars.logInfo must eachItem(haveKey("PERNR"))
```

---

## Global Configuration Patterns

### HTTP Listener

```xml
<http:listener-config name="HTTP_Listener_config" doc:name="HTTP Listener config">
    <http:listener-connection host="0.0.0.0" port="${http.port}"/>
</http:listener-config>
```

### HTTP Request (Basic Auth)

```xml
<http:request-config name="HTTP_Request_configuration" doc:name="HTTP Request configuration">
    <http:request-connection host="${service.host}"
                             port="${service.port}"
                             protocol="HTTPS">
        <http:authentication>
            <http:basic-authentication username="${service.username}"
                                       password="${service.password}"/>
        </http:authentication>
    </http:request-connection>
</http:request-config>
```

### HTTP Request (OAuth)

```xml
<http:request-config name="OAuth_HTTP_Request_configuration" doc:name="OAuth HTTP Request">
    <http:request-connection host="${service.host}" port="${service.port}" protocol="HTTPS">
        <http:authentication>
            <oauth:client-credentials-grant-type
                clientId="${oauth.client.id}"
                clientSecret="${oauth.client.secret}"
                tokenUrl="${oauth.token.url}">
                <oauth:token-request-parameters>
                    <oauth:token-request-parameter key="grant_type" value="client_credentials"/>
                </oauth:token-request-parameters>
            </oauth:client-credentials-grant-type>
        </http:authentication>
    </http:request-connection>
</http:request-config>
```

### SAP

```xml
<sap:sap-config name="SAP_Config" doc:name="SAP Config">
    <sap:simple-connection-provider-connection
        applicationServerHost="${sap.host}"
        username="${sap.username}"
        password="${sap.password}"
        systemNumber="${sap.systemNumber}"
        client="${sap.client}"
        language="EN"/>
</sap:sap-config>
```

### Object Store

```xml
<os:object-store name="Error_Object_Store"
                 doc:name="Object store"
                 maxEntries="10000"
                 entryTtl="15"
                 entryTtlUnit="DAYS"
                 expirationInterval="1"
                 expirationIntervalUnit="HOURS"/>
```

### Secure Properties

```xml
<secure-properties:config name="Secure_Configuration_Properties"
                          doc:name="Secure Configuration Properties"
                          file="secure-config-${mule.env}.yaml"
                          key="${encrypt.key}">
    <secure-properties:encrypt algorithm="<encryption-algorithm>"/>
</secure-properties:config>
```

### API Autodiscovery

```xml
<api-gateway:autodiscovery apiId="${api.id}"
                           flowRef="main-api-flow"
                           doc:name="API Autodiscovery"/>
```

---

## Error Handling

### Global Error Handler

```xml
<error-handler name="global-error-handler">
    <!-- Handle known errors - continue processing -->
    <on-error-continue
        enableNotifications="true"
        logException="true"
        doc:name="On Error Continue"
        type="APIKIT:BAD_REQUEST">
        <ee:transform doc:name="Transform error response">
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    "code": 400,
    "message": "Bad Request",
    "description": error.description
}]]></ee:set-payload>
            </ee:message>
        </ee:transform>
        <set-variable variableName="httpStatus" value="400"/>
    </on-error-continue>

    <!-- Handle unexpected errors - propagate -->
    <on-error-propagate
        enableNotifications="true"
        logException="true"
        doc:name="On Error Propagate"
        type="ANY">
        <ee:transform doc:name="Transform error response">
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{
    "code": 500,
    "message": "Internal Server Error",
    "description": error.description,
    "correlationId": vars.correlationId
}]]></ee:set-payload>
            </ee:message>
        </ee:transform>
        <set-variable variableName="httpStatus" value="500"/>
    </on-error-propagate>
</error-handler>
```

### APIkit Error Handler

```xml
<error-handler name="apikit-error-handler">
    <on-error-propagate type="APIKIT:BAD_REQUEST">
        <set-variable variableName="httpStatus" value="400"/>
        <ee:transform>
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{message: "Bad request"}]]></ee:set-payload>
            </ee:message>
        </ee:transform>
    </on-error-propagate>

    <on-error-propagate type="APIKIT:NOT_FOUND">
        <set-variable variableName="httpStatus" value="404"/>
        <ee:transform>
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{message: "Resource not found"}]]></ee:set-payload>
            </ee:message>
        </ee:transform>
    </on-error-propagate>

    <on-error-propagate type="APIKIT:METHOD_NOT_ALLOWED">
        <set-variable variableName="httpStatus" value="405"/>
        <ee:transform>
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{message: "Method not allowed"}]]></ee:set-payload>
            </ee:message>
        </ee:transform>
    </on-error-propagate>
</error-handler>
```

---

## DataWeave Transformation Patterns

### Standard Response Transformation

```dataweave
%dw 2.0
output application/json
---
{
    "data": payload,
    "metadata": {
        "correlationId": vars.correlationId,
        "timestamp": now() as String {format: "yyyy-MM-dd'T'HH:mm:ss.SSSZ"}
    }
}
```

### Error Response Transformation

```dataweave
%dw 2.0
output application/json
---
{
    "code": vars.httpStatus default 500,
    "message": error.errorType.identifier default "UNKNOWN_ERROR",
    "description": error.description default "An unexpected error occurred",
    "correlationId": vars.correlationId
}
```

### SAP Response Transformation

```dataweave
%dw 2.0
output application/json
---
payload.BAPI_RESPONSE.*ITEM map {
    "id": $.MATERIAL,
    "description": $.MATL_DESC,
    "type": $.MATL_TYPE
}
```

### Null-Safe Access

```dataweave
%dw 2.0
output application/json
---
{
    "value": payload.field default "",
    "nested": payload.level1.level2.value default null,
    "array": payload.items default []
}
```

---

