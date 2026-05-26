# MUnit Templates — EAPI (Usage Examples)

> **Purpose:** Copy-paste MUnit XML templates for EAPI flows (PAPI integration, async operations) plus a complete working EAPI test suite. For rules, quick-reference tables, and EAPI-specific logger handling guidance, see [standards/template-by-layer.md](../standards/template-by-layer.md) § EAPI. For patterns shared across layers (target-attribute mocks, APIKit router, common-patterns library, anti-patterns), see [usage/template-common.md](./template-common.md).

## 3.1 PAPI Integration Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:munit="http://www.mulesoft.org/schema/mule/munit"
      xmlns:munit-tools="http://www.mulesoft.org/schema/mule/munit-tools"
      xsi:schemaLocation="...">

    <!-- EAPI UI Request Test -->
    <munit:test name="post-{entity}-test-200-ok" description="Test {entity} POST for UI consumption">

        <munit:behavior>
            <!-- Mock PAPI Call -->
            <munit-tools:mock-when processor="http:request" doc:name="Mock PAPI">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{papi-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <munit-tools:payload value="#[MunitTools::getResourceAsString('test-data/{entity}/<broker>-get-{entity}.json')]"
                                        mediaType="application/json" encoding="UTF-8"/>
                    <munit-tools:attributes value="#[{
                        statusCode: 200,
                        headers: {
                            'x-correlation-id': 'test-correlation-id'
                        }
                    }]"/>
                </munit-tools:then-return>
            </munit-tools:mock-when>

            <!-- NOTE: Do NOT mock <logger>. Standard Mule loggers are no-ops at runtime
                 in tests. Set any variable referenced by a logger message (e.g., correlationId)
                 in <munit:set-event> below. -->
        </munit:behavior>

        <munit:execution>
            <munit:set-event doc:name="Set UI request event">
                <munit:payload value="#[MunitTools::getResourceAsString('test-data/{entity}/apikit-payload-{entity}.json')]"
                              mediaType="application/json"/>
                <munit:attributes value='#[{
                    "method": "POST",
                    "requestPath": "/api/{entity}",
                    "headers": {
                        "x-correlation-id": "test-correlation-id",
                        "Content-Type": "application/json"
                    }
                }]'/>
                <munit:variables>
                    <munit:variable key="correlationId" value='"test-correlation-id"' mediaType="application/java"/>
                </munit:variables>
            </munit:set-event>
            <flow-ref name="post:\{entity}:application\json:{eapi-config}" doc:name="Invoke EAPI flow"/>
        </munit:execution>

        <munit:validation>
            <!-- Verify transform executed -->
            <munit-tools:verify-call processor="ee:transform" times="1" doc:name="Verify Transform">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{prepare-payload-transform-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
            </munit-tools:verify-call>

            <!-- Verify PAPI called -->
            <munit-tools:verify-call processor="http:request" times="1" doc:name="Verify PAPI called">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{papi-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
            </munit-tools:verify-call>

            <!-- Assert UI-friendly response -->
            <munit-tools:assert-that expression="#[payload.status]"
                                    is="#[MunitTools::equalTo('SUCCESS')]"
                                    doc:name="Assert status"/>

            <munit-tools:assert-that expression="#[payload.correlationId]"
                                    is="#[MunitTools::equalTo('test-correlation-id')]"
                                    doc:name="Assert correlation ID"/>

            <munit-tools:assert-that expression="#[payload.data]"
                                    is="#[MunitTools::notNullValue()]"
                                    doc:name="Assert data present"/>

            <!-- UI specific fields -->
            <munit-tools:assert-that expression="#[payload.displayMessage]"
                                    is="#[MunitTools::equalTo('Operation completed successfully')]"
                                    doc:name="Assert UI message"/>

            <munit-tools:assert-that expression="#[attributes.statusCode]"
                                    is="#[MunitTools::equalTo(200)]"
                                    doc:name="Assert status 200"/>
        </munit:validation>
    </munit:test>

    <!-- EAPI Async Operation Test -->
    <munit:test name="post-{entity}-test-202-accepted" description="Test async operation for UI">
        <munit:behavior>
            <!-- Mock PAPI returns accepted status -->
            <munit-tools:mock-when processor="http:request">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{papi-async-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <munit-tools:payload value='#[{
                        operationId: "op-123456",
                        status: "ACCEPTED",
                        estimatedCompletion: "2025-01-01T10:00:00Z"
                    }]' encoding="UTF-8"/>
                    <munit-tools:attributes value="#[{statusCode: 202}]"/>
                </munit-tools:then-return>
            </munit-tools:mock-when>
        </munit:behavior>

        <munit:execution>
            <!-- Similar setup, including <munit:variables> with correlationId -->
        </munit:execution>

        <munit:validation>
            <!-- Verify async response structure -->
            <munit-tools:assert-that expression="#[attributes.statusCode]"
                                    is="#[MunitTools::equalTo(202)]"/>
            <munit-tools:assert-that expression="#[payload.operationId]"
                                    is="#[MunitTools::equalTo('op-123456')]"/>
            <munit-tools:assert-that expression="#[payload.message]"
                                    is="#[MunitTools::containsString('processing')]"/>
        </munit:validation>
    </munit:test>
</mule>
```

---

## Complete Working EAPI Test Example

```xml
<!-- Complete EAPI Test: Work Orders UI Submission -->
<munit:test name="post-work-orders-test-201-created" description="Test work order submission from UI">

    <munit:behavior>
        <!-- Mock PAPI Work Orders -->
        <munit-tools:mock-when processor="http:request" doc:name="Mock PAPI">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="26716673-1e2f-4d9b-8679-e5f95e4b93e0" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value='#[{
                    "workOrderNumber": "WO-2025-000123",
                    "status": "CREATED",
                    "estimatedCompletion": "2025-02-01T10:00:00Z"
                }]' mediaType="application/json" encoding="UTF-8"/>
                <munit-tools:attributes value="#[{
                    statusCode: 201,
                    headers: {
                        'x-correlation-id': 'ui-corr-456'
                    }
                }]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>

        <!-- NOTE: Do NOT mock <logger>. Standard Mule loggers are no-ops in tests.
             Set any variable referenced by a logger message (e.g., correlationId) in <munit:set-event>. -->
    </munit:behavior>

    <munit:execution>
        <munit:set-event doc:name="Set UI request">
            <munit:payload value='#[{
                "title": "Fix pump malfunction",
                "description": "Pump #123 not working",
                "priority": "HIGH",
                "requestedBy": "user@example.com",
                "equipment": "PUMP-123",
                "location": "Building A"
            }]' mediaType="application/json"/>
            <munit:attributes value='#[{
                "method": "POST",
                "requestPath": "/api/work-orders",
                "headers": {
                    "x-correlation-id": "ui-corr-456",
                    "Content-Type": "application/json",
                    "Authorization": "Bearer test-token"
                }
            }]'/>
            <munit:variables>
                <munit:variable key="correlationId" value='"ui-corr-456"' mediaType="application/java"/>
            </munit:variables>
        </munit:set-event>
        <flow-ref name="post:\work-orders:application\json:<your-eapi-config>"/>
    </munit:execution>

    <munit:validation>
        <!-- Verify transform executed -->
        <munit-tools:verify-call processor="ee:transform" times="1" doc:name="Transform executed">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="85e2ace1-c2f8-4707-8d41-6e07a33a97e8" attributeName="doc:id"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>

        <!-- Verify PAPI called -->
        <munit-tools:verify-call processor="http:request" times="1" doc:name="PAPI called">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="26716673-1e2f-4d9b-8679-e5f95e4b93e0" attributeName="doc:id"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>

        <!-- Assert UI response -->
        <munit-tools:assert-that expression="#[payload.status]"
                                is="#[MunitTools::equalTo('SUCCESS')]"
                                doc:name="Success status"/>

        <munit-tools:assert-that expression="#[payload.workOrderNumber]"
                                is="#[MunitTools::equalTo('WO-2025-000123')]"
                                doc:name="WO number"/>

        <munit-tools:assert-that expression="#[payload.message]"
                                is="#[MunitTools::equalTo('Work order created successfully')]"
                                doc:name="UI message"/>

        <munit-tools:assert-that expression="#[payload.correlationId]"
                                is="#[MunitTools::equalTo('ui-corr-456')]"
                                doc:name="Correlation ID"/>

        <munit-tools:assert-that expression="#[attributes.statusCode]"
                                is="#[MunitTools::equalTo(201)]"
                                doc:name="Status 201"/>
    </munit:validation>
</munit:test>
```
