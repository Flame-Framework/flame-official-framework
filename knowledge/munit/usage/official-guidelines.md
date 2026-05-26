# MUnit Official Guidelines (Usage Examples)

> **Purpose:** Code examples for the official MUnit guidelines — processor identification, mocking, assertions, routing tests, error handling, async/VM/batch, lifecycle hooks, environment config, coverage POM, integration tests. For methodology, strategy, and best practices, see [reference/official-guidelines.md](../reference/official-guidelines.md).

---

## Processor Identification for Mocks

### Scenario 1 — Single processor of its type (use `doc:name`)

```xml
<munit-tools:mock-when processor="http:request">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:name" whereValue="Get Customer Details"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[{}]" mediaType="application/json" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Scenario 2 — Multiple processors, different configs (`doc:name` + `config-ref`)

```xml
<munit-tools:mock-when processor="db:select">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:name" whereValue="Get Orders"/>
        <munit-tools:with-attribute attributeName="config-ref" whereValue="Oracle_Database_Config"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[[]]" mediaType="application/java" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Scenario 3 — Multiple HTTP requests, different methods or paths

```xml
<munit-tools:mock-when processor="http:request">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:name" whereValue="Call External API"/>
        <munit-tools:with-attribute attributeName="method" whereValue="POST"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[{}]" mediaType="application/json" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Scenario 4 — Flow references (use `name`)

```xml
<munit-tools:mock-when processor="mule:flow-ref">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="name" whereValue="process-order-subflow"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[{status: 'processed'}]" mediaType="application/json" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

---

## Set Event

```xml
<munit:set-event>
    <munit:payload value="#[{ 'name': 'John', 'age': 30 }]" mediaType="application/json"/>
    <munit:attributes value="#[{ 'queryParams': { 'id': '123' } }]"/>
    <munit:variables>
        <munit:variable key="myVar" value="#['someValue']"/>
    </munit:variables>
</munit:set-event>
```

---

## Mock When

```xml
<munit-tools:mock-when processor="http:request">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:name" whereValue="Call External API"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[MunitTools::getResourceAsString('mock-responses/api-response.json')]"
                             mediaType="application/json" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Mocking an HTTP request with `target` attribute

```xml
<!-- Flow: <http:request target="result" targetValue="#[output application/java --- payload]"> -->
<munit-tools:mock-when processor="http:request">
    <munit-tools:then-return>
        <munit-tools:variables>
            <munit-tools:variable key="result"
                value='#[{id: "123", name: "John"}]'
                mediaType="application/java"/>
        </munit-tools:variables>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

---

## Assert That

### Basic assertions

```xml
<munit-tools:assert-that
    expression="#[payload]"
    is="#[MunitTools::equalTo('expected value')]"/>

<munit-tools:assert-that
    expression="#[payload]"
    is="#[MunitTools::notNullValue()]"/>

<munit-tools:assert-that
    expression="#[payload.name]"
    is="#[MunitTools::equalTo('John')]"/>
```

### Assert full payload — compare as object (preferred)

```xml
<!-- GOOD: compare as object -->
<munit-tools:assert-that
    expression="#[output application/java --- payload]"
    is="#[MunitTools::equalTo({id: '123', name: 'John'})]"/>

<!-- AVOID: comparing JSON strings -->
<munit-tools:assert-that
    expression="#[payload]"
    is="#[MunitTools::equalTo('{&quot;id&quot;:&quot;123&quot;,&quot;name&quot;:&quot;John&quot;}')]"/>
```

### Assert media type

```xml
<munit-tools:assert-that
    expression="#[payload]"
    is="#[MunitTools::withMediaType('application/json')]"/>
```

### Assert flow variables

```xml
<munit-tools:assert-that
    expression="#[vars.userId]"
    is="#[MunitTools::equalTo('123')]"/>

<munit-tools:assert-that
    expression="#[vars.isValid]"
    is="#[MunitTools::equalTo(true)]"/>
```

### Payloads with timestamps — assert fields individually

```xml
<munit-tools:assert-that
    expression="#[payload.status]"
    is="#[MunitTools::equalTo('success')]"/>

<munit-tools:assert-that
    expression="#[payload.id]"
    is="#[MunitTools::equalTo('123')]"/>

<munit-tools:assert-that
    expression="#[payload.timestamp]"
    is="#[MunitTools::notNullValue()]"/>
```

---

## Verify Call

```xml
<munit-tools:verify-call processor="http:request" times="1"/>
<munit-tools:verify-call processor="mule:logger" atLeast="1"/>
```

### Targeted verification

```xml
<munit-tools:verify-call processor="http:request" times="1">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:name" whereValue="Request External API"/>
    </munit-tools:with-attributes>
</munit-tools:verify-call>
```

### Cached (`ee:cache`) — use `atLeast="0"`

When the HTTP request is wrapped in `<ee:cache>`, the processor may not execute on every run (cache hit). `times="1"` fails with `Expected 1 but got 0 calls`.

```xml
<munit-tools:verify-call processor="http:request" atLeast="0">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute whereValue="abc123" attributeName="doc:id"/>
    </munit-tools:with-attributes>
</munit-tools:verify-call>
```

---

## Spy

```xml
<munit-tools:spy processor="http:request">
    <munit-tools:before-call>
        <munit-tools:assert-that expression="#[payload.id]" is="#[MunitTools::notNullValue()]"/>
    </munit-tools:before-call>
    <munit-tools:after-call>
        <munit-tools:assert-that expression="#[payload.status]" is="#[MunitTools::equalTo('success')]"/>
    </munit-tools:after-call>
</munit-tools:spy>
```

---

## Common Mocking Scenarios

### Mock HTTP Request

```xml
<munit-tools:mock-when processor="http:request">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:name" whereValue="GET Users"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[MunitTools::getResourceAsString('test-data/users-response.json')]"
                             mediaType="application/json" encoding="UTF-8"/>
        <munit-tools:attributes value="#[{ 'statusCode': 200 }]"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Mock Database Select

```xml
<munit-tools:mock-when processor="db:select">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="config-ref" whereValue="Database_Config"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[MunitTools::getResourceAsString('test-data/db-response.json')]"
                             mediaType="application/json" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Mock Flow Reference

```xml
<munit-tools:mock-when processor="mule:flow-ref">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="name" whereValue="subflow-name"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#['mocked response']" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Mock with error

```xml
<munit-tools:mock-when processor="http:request">
    <munit-tools:then-return>
        <munit-tools:error typeId="HTTP:CONNECTIVITY"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Mock connectivity error targeted by method

```xml
<munit-tools:mock-when processor="http:request">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="method" whereValue="GET"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:error typeId="HTTP:CONNECTIVITY"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

---

## Testing Choice Router Branches

### Example flow

```xml
<flow name="process-customer-flow">
    <choice>
        <when expression="#[payload.customerType == 'premium']">
            <flow-ref name="premium-customer-subflow"/>
        </when>
        <when expression="#[payload.customerType == 'standard']">
            <flow-ref name="standard-customer-subflow"/>
        </when>
        <otherwise>
            <flow-ref name="unknown-customer-subflow"/>
        </otherwise>
    </choice>
</flow>
```

### Test — premium branch

```xml
<munit:test name="process-customer-flow-premium-branch-test"
            description="Test premium customer routing">
    <munit:behavior>
        <munit:set-event>
            <munit:payload value="#[{ 'customerType': 'premium', 'customerId': '123' }]"
                          mediaType="application/json"/>
        </munit:set-event>
        <munit-tools:mock-when processor="mule:flow-ref">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="name" whereValue="premium-customer-subflow"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[{discount: 20}]" mediaType="application/json" encoding="UTF-8"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="process-customer-flow"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:verify-call processor="mule:flow-ref" times="1">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="name" whereValue="premium-customer-subflow"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>
        <munit-tools:verify-call processor="mule:flow-ref" times="0">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="name" whereValue="standard-customer-subflow"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>
    </munit:validation>
</munit:test>
```

### Test — otherwise (default) branch

```xml
<munit:test name="process-customer-flow-otherwise-branch-test"
            description="Test unknown customer routing">
    <munit:behavior>
        <munit:set-event>
            <munit:payload value="#[{ 'customerType': 'vip', 'customerId': '789' }]"
                          mediaType="application/json"/>
        </munit:set-event>
        <munit-tools:mock-when processor="mule:flow-ref">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="name" whereValue="unknown-customer-subflow"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[{discount: 0}]" mediaType="application/json" encoding="UTF-8"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="process-customer-flow"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:verify-call processor="mule:flow-ref" times="1">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="name" whereValue="unknown-customer-subflow"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>
    </munit:validation>
</munit:test>
```

---

## Testing Scatter-Gather

### Flow

```xml
<flow name="aggregate-data-flow">
    <scatter-gather>
        <route>
            <http:request config-ref="HTTP_Config" method="GET" path="/users" doc:name="Get Users"/>
        </route>
        <route>
            <http:request config-ref="HTTP_Config" method="GET" path="/orders" doc:name="Get Orders"/>
        </route>
    </scatter-gather>
    <ee:transform>
        <!-- Combine results -->
    </ee:transform>
</flow>
```

### Test — all routes succeed

```xml
<munit:test name="aggregate-data-flow-all-routes-success-test"
            description="Test scatter-gather with all routes successful">
    <munit:behavior>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="doc:name" whereValue="Get Users"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[[{id: '1', name: 'John'}]]" mediaType="application/json" encoding="UTF-8"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="doc:name" whereValue="Get Orders"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[[{orderId: '100', amount: 50}]]" mediaType="application/json" encoding="UTF-8"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="aggregate-data-flow"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:verify-call processor="http:request" times="2"/>
        <munit-tools:assert-that expression="#[payload]" is="#[MunitTools::notNullValue()]"/>
    </munit:validation>
</munit:test>
```

---

## Testing First Successful Router

```xml
<munit:test name="first-successful-primary-route-test"
            description="Primary route returns successfully">
    <munit:behavior>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="doc:name" whereValue="Primary API"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[{source: 'primary'}]" mediaType="application/json" encoding="UTF-8"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="first-successful-flow"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:assert-that expression="#[payload.source]" is="#[MunitTools::equalTo('primary')]"/>
        <munit-tools:verify-call processor="http:request" times="0">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="doc:name" whereValue="Fallback API"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>
    </munit:validation>
</munit:test>
```

---

## Handling Complex Payloads

### XML from file

```xml
<munit:set-event>
    <munit:payload value="#[MunitTools::getResourceAsString('test-data/{entity}/apikit-payload-{entity}.xml')]"
                  mediaType="application/xml"/>
</munit:set-event>
```

### Inline XML

```xml
<munit:set-event>
    <munit:payload mediaType="application/xml"><![CDATA[
        <order>
            <orderId>123</orderId>
            <customer>
                <name>John Doe</name>
            </customer>
        </order>
    ]]></munit:payload>
</munit:set-event>
```

### XML with namespaces

```xml
<munit:set-event>
    <munit:payload mediaType="application/xml"><![CDATA[
        <ns0:Order xmlns:ns0="http://example.com/orders">
            <ns0:OrderId>123</ns0:OrderId>
            <ns0:Amount>100.00</ns0:Amount>
        </ns0:Order>
    ]]></munit:payload>
</munit:set-event>
```

### Asserting XML content

```xml
<munit-tools:assert-that
    expression="#[payload.order.orderId]"
    is="#[MunitTools::equalTo('123')]"/>

<munit-tools:assert-that
    expression="#[payload.ns0#Order.ns0#OrderId]"
    is="#[MunitTools::equalTo('123')]"/>
```

### Streaming / binary payloads

```xml
<!-- Streaming -->
<munit-tools:mock-when processor="file:read">
    <munit-tools:then-return>
        <munit-tools:payload value="#[MunitTools::getResourceAsStream('test-data/large-file.csv')]"
                            mediaType="text/csv" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>

<!-- Binary (PDF) -->
<munit-tools:mock-when processor="http:request">
    <munit-tools:then-return>
        <munit-tools:payload value="#[MunitTools::getResourceAsStream('test-data/document.pdf')]"
                            mediaType="application/pdf" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### Java objects as payload

```xml
<munit:set-event>
    <munit:payload value="#[{
        id: '123',
        items: [
            {sku: 'ABC', qty: 2},
            {sku: 'DEF', qty: 1}
        ],
        total: 150.00
    }]" mediaType="application/java"/>
</munit:set-event>
```

---

## Testing Error Handling

> **Moved.** All error-handling examples live in the layer-specific template files:
> - SAPI-specific HTTP connectivity / timeout — [usage/template-sapi.md § 1.4](./template-sapi.md#14-error-handling-templates)
> - Generic patterns (expect error, on-error-continue, mocked-flow propagation, global handler) — [usage/template-common.md § 4.1](./template-common.md#41-error-handling-patterns)

---

## Testing Async, Batch, VM

### Async — mock the inner processor and verify with `timeout`

```xml
<munit:test name="flow-with-async-test" description="Test flow containing async scope">
    <munit:behavior>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="doc:name" whereValue="Async Notification"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[{status: 'sent'}]" mediaType="application/json" encoding="UTF-8"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="process-with-async-flow"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:assert-that expression="#[payload.processed]" is="#[MunitTools::equalTo(true)]"/>
        <munit-tools:verify-call processor="http:request" times="1" timeout="5000">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="doc:name" whereValue="Async Notification"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>
    </munit:validation>
</munit:test>
```

### Async — test the sub-flow directly

```xml
<munit:test name="async-notification-subflow-test">
    <munit:behavior>
        <munit:set-event>
            <munit:payload value="#[{eventId: '123'}]" mediaType="application/json"/>
        </munit:set-event>
        <munit-tools:mock-when processor="http:request">
            <munit-tools:then-return>
                <munit-tools:payload value="#[{sent: true}]" mediaType="application/json" encoding="UTF-8"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="send-notification-subflow"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:verify-call processor="http:request" times="1"/>
    </munit:validation>
</munit:test>
```

### VM — mock publish and consume

```xml
<!-- VM publish -->
<munit-tools:mock-when processor="vm:publish">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:name" whereValue="Queue Order"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return/>
</munit-tools:mock-when>

<!-- VM consume -->
<munit-tools:mock-when processor="vm:consume">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="queueName" whereValue="orderQueue"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[{orderId: '123'}]" mediaType="application/json" encoding="UTF-8"/>
    </munit-tools:then-return>
</munit-tools:mock-when>
```

### VM listener tested via direct flow-ref

```xml
<munit:test name="vm-listener-flow-processes-message-test">
    <munit:behavior>
        <munit:set-event>
            <munit:payload value="#[{orderId: '123', items: [{sku: 'ABC'}]}]"
                          mediaType="application/json"/>
        </munit:set-event>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="vm-order-processor-flow"/>
    </munit:execution>
    <munit:validation>
        <!-- assertions -->
    </munit:validation>
</munit:test>
```

### Batch — test input phase

```xml
<munit:test name="batch-job-input-phase-test">
    <munit:behavior>
        <munit:set-event>
            <munit:payload value="#[MunitTools::getResourceAsString('test-data/batch-input.json')]"
                          mediaType="application/json"/>
        </munit:set-event>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="batch-job-flow"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:assert-that expression="#[payload]" is="#[MunitTools::notNullValue()]"/>
    </munit:validation>
</munit:test>
```

### Batch — extract step into a sub-flow

```xml
<munit:test name="batch-step-process-record-test">
    <munit:behavior>
        <munit:set-event>
            <munit:payload value="#[{id: '1', name: 'Test'}]" mediaType="application/json"/>
        </munit:set-event>
    </munit:behavior>
    <munit:execution>
        <flow-ref name="process-single-record-subflow"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:assert-that expression="#[payload.processed]" is="#[MunitTools::equalTo(true)]"/>
    </munit:validation>
</munit:test>
```

### Enable batch job source

```xml
<munit:config name="batch-test-config">
    <munit:enable-flow-sources>
        <munit:enable-flow-source value="batch-job-flow"/>
    </munit:enable-flow-sources>
</munit:config>
```

---

## Before / After Hooks

### Before test (runs before each test)

```xml
<munit:before-test name="before-each-test">
    <munit:set-event>
        <munit:payload value="#[{}]"/>
    </munit:set-event>
</munit:before-test>
```

### Before test — shared mocks

```xml
<munit:before-test name="setup-mocks">
    <munit-tools:mock-when processor="http:request">
        <munit-tools:with-attributes>
            <munit-tools:with-attribute attributeName="method" whereValue="GET"/>
        </munit-tools:with-attributes>
        <munit-tools:then-return>
            <munit-tools:payload value="#[{status: 'success'}]" mediaType="application/json" encoding="UTF-8"/>
        </munit-tools:then-return>
    </munit-tools:mock-when>
</munit:before-test>
```

### After test, before suite, after suite

```xml
<munit:after-test name="after-each-test">
    <mule:logger message="Test completed"/>
</munit:after-test>

<munit:before-suite name="before-all-tests">
    <!-- Initialize test data -->
</munit:before-suite>

<munit:after-suite name="after-all-tests">
    <!-- Cleanup -->
</munit:after-suite>
```

---

## Loading Test Resources

```xml
<!-- As String -->
<munit-tools:payload value="#[MunitTools::getResourceAsString('test-data/api-response.json')]" encoding="UTF-8"/>

<!-- As Stream -->
<munit-tools:payload value="#[MunitTools::getResourceAsStream('test-data/api-response.json')]" encoding="UTF-8"/>
```

---

## Common Matcher Snippets

```xml
<!-- Equals -->
#[MunitTools::equalTo('expected')]

<!-- Not Equal -->
#[MunitTools::not(MunitTools::equalTo('value'))]

<!-- Null / Not Null -->
#[MunitTools::nullValue()]
#[MunitTools::notNullValue()]

<!-- Contains String -->
#[MunitTools::containsString('substring')]

<!-- Greater / Less Than -->
#[MunitTools::greaterThan(10)]
#[MunitTools::lessThan(100)]

<!-- Collection Size -->
#[MunitTools::hasSize(5)]

<!-- Empty Collection -->
#[MunitTools::empty()]
```

---

## Testing with Environment Properties

### Test properties file (`src/test/resources/application.localtest.properties`)

```properties
api.host=localhost
api.port=8081
db.host=test-db
wildcard.card=AS
```

### Global property override in a test suite

```xml
<munit:config name="MUnit_Config"/>

<!-- Override mule.env for tests -->
<global-property name="mule.env" value="localtest"/>
```

### Config-properties reference in main app

```xml
<configuration-properties
    file="application.${mule.env}.properties"/>
```

### Parameterizations alternative

```xml
<munit:config name="MUnit_Config">
    <munit:parameterizations>
        <munit:parameterization name="test-config">
            <munit:parameters>
                <munit:parameter propertyName="api.host" value="localhost"/>
            </munit:parameters>
        </munit:parameterization>
    </munit:parameterizations>
</munit:config>
```

---

## Maven Plugin Coverage Configuration

```xml
<plugin>
    <groupId>com.mulesoft.munit.tools</groupId>
    <artifactId>munit-maven-plugin</artifactId>
    <version>${munit.version}</version>
    <executions>
        <execution>
            <id>test</id>
            <phase>test</phase>
            <goals>
                <goal>test</goal>
                <goal>coverage-report</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <coverage>
            <runCoverage>true</runCoverage>
            <formats>
                <format>html</format>
                <format>json</format>
            </formats>
            <ignoreFiles>
                <ignoreFile>global-error-handler.xml</ignoreFile>
                <ignoreFile>common-logging.xml</ignoreFile>
            </ignoreFiles>
        </coverage>
    </configuration>
</plugin>
```

### Example coverage summary output

```
[INFO] ===============================================================================
[INFO] MUnit Coverage Summary
[INFO] ===============================================================================
[INFO]  * Resources: 1 - Flows: 6 - Processors: 21
[INFO]  * Application Coverage: 85.71%
```

### Fail-build enforcement (recommended)

Enforce coverage thresholds by failing the build if they're not met. Coverage thresholds are configurable in `config/framework.yaml` → `test.coverage.*-percent` (defaults shown below: 80% application, 90% flow):

```xml
<plugin>
    <groupId>com.mulesoft.munit.tools</groupId>
    <artifactId>munit-maven-plugin</artifactId>
    <configuration>
        <coverage>
            <runCoverage>true</runCoverage>
            <failBuild>true</failBuild>
            <requiredApplicationCoverage>80</requiredApplicationCoverage>
            <requiredFlowCoverage>90</requiredFlowCoverage>
            <ignoreFiles>
                <ignoreFile>global.xml</ignoreFile>
            </ignoreFiles>
        </coverage>
    </configuration>
</plugin>
```

---

## MUnit Maven Dependencies

Every project must declare these in `pom.xml`:

```xml
<dependency>
    <groupId>com.mulesoft.munit</groupId>
    <artifactId>munit-runner</artifactId>
    <version>`<your-version>`</version>
    <classifier>mule-plugin</classifier>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>com.mulesoft.munit</groupId>
    <artifactId>munit-tools</artifactId>
    <version>`<your-version>`</version>
    <classifier>mule-plugin</classifier>
    <scope>test</scope>
</dependency>
```

---

## Integration Testing — Enable Flow Sources

```xml
<munit:config name="Integration_Test_Config">
    <munit:enable-flow-sources>
        <munit:enable-flow-source value="api-main-flow"/>
        <munit:enable-flow-source value="api-listener-config"/>
    </munit:enable-flow-sources>
</munit:config>
```
