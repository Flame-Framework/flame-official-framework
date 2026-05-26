# MUnit Templates — PAPI (Usage Examples)

> **Purpose:** Copy-paste MUnit XML templates for PAPI flows (multi-SAPI orchestration, scheduler flow) plus a complete working PAPI test suite. For rules, quick-reference tables, and payload-inference logic, see [standards/template-by-layer.md](../standards/template-by-layer.md) § PAPI. For patterns shared across layers (target-attribute mocks, APIKit router, common-patterns library, anti-patterns), see [usage/template-common.md](./template-common.md).

## 2.1 Multi-SAPI Orchestration Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:munit="http://www.mulesoft.org/schema/mule/munit"
      xmlns:munit-tools="http://www.mulesoft.org/schema/mule/munit-tools"
      xsi:schemaLocation="...">

    <!-- PAPI Orchestration Test - Multiple SAPI Calls -->
    <munit:test name="scheduler-{entity}-test-orchestration" description="Test {entity} orchestration across multiple SAPIs">

        <munit:behavior>
            <!-- Mock SAPI 1: Get Master Data -->
            <munit-tools:mock-when processor="http:request" doc:name="Mock SAPI 1 - Master Data">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{sapi1-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <munit-tools:payload value="#[readUrl('classpath://test-data/{entity}/<broker>-get-{entity}.dwl')]"
                                        mediaType="application/json" encoding="UTF-8"/>
                    <munit-tools:attributes value="#[{statusCode: 200}]"/>
                </munit-tools:then-return>
            </munit-tools:mock-when>

            <!-- Mock SAPI 2: Get Transaction Data -->
            <munit-tools:mock-when processor="http:request" doc:name="Mock SAPI 2 - Transaction Data">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{sapi2-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <munit-tools:payload value="#[readUrl('classpath://test-data/{entity}/<broker>-get-{entity}-secondary.dwl')]" encoding="UTF-8"/>
                    <munit-tools:attributes value="#[{statusCode: 200}]"/>
                </munit-tools:then-return>
            </munit-tools:mock-when>

            <!-- Mock <broker> API for Event Publishing -->
            <munit-tools:mock-when processor="http:request" doc:name="Mock <broker> API">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{broker-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <munit-tools:payload value="#[payload]" encoding="UTF-8"/> <!-- Echo back for verification -->
                    <munit-tools:attributes value="#[{statusCode: 200}]"/>
                </munit-tools:then-return>
            </munit-tools:mock-when>

            <!-- Spy on orchestration logic to verify data passing -->
            <munit-tools:spy processor="try" doc:name="Spy on orchestration">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{orchestration-try-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:before-call>
                    <!-- Verify aggregated data before processing -->
                    <munit-tools:assert-that expression="#[vars.aggregatedData]"
                                            is="#[MunitTools::notNullValue()]"/>
                    <munit-tools:assert-equals actual="#[vars.topic]"
                                              expected='#["<topic-path>/{Entity}/12345"]'/>
                </munit-tools:before-call>
                <munit-tools:after-call>
                    <!-- Verify processing completed -->
                    <munit-tools:assert-that expression="#[vars.processStatus]"
                                            is="#[MunitTools::equalTo('SUCCESS')]"/>
                </munit-tools:after-call>
            </munit-tools:spy>

            <!-- Mock Object Store for error handling -->
            <munit-tools:spy processor="os:store" doc:name="Spy on Object Store">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{os-store-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:before-call>
                    <munit-tools:assert-that expression="#[vars.storeError]"
                                            is="#[MunitTools::nullValue()]"
                                            message="Should not store error in happy path"/>
                </munit-tools:before-call>
            </munit-tools:spy>
        </munit:behavior>

        <munit:execution>
            <munit:set-event doc:name="Set complex orchestration event">
                <munit:payload value='#[{
                    "requestId": "req-123",
                    "entityId": "12345",
                    "operation": "CREATE",
                    "data": {
                        "field1": "value1",
                        "field2": "value2"
                    }
                }]' mediaType="application/json"/>
                <munit:variables>
                    <munit:variable key="correlationId" value="#['test-correlation-id']"/>
                    <munit:variable key="companyId" value="#['company-123']"/>
                    <munit:variable key="scope" value="#['{entity}-orchestration']"/>
                </munit:variables>
            </munit:set-event>
            <flow-ref name="{entity}-orchestration-main-flow" doc:name="Invoke orchestration flow"/>
        </munit:execution>

        <munit:validation>
            <!-- Verify all SAPIs were called -->
            <munit-tools:verify-call processor="http:request" times="1" doc:name="Verify SAPI 1 called">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{sapi1-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
            </munit-tools:verify-call>

            <munit-tools:verify-call processor="http:request" times="1" doc:name="Verify SAPI 2 called">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{sapi2-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
            </munit-tools:verify-call>

            <munit-tools:verify-call processor="http:request" times="1" doc:name="Verify <broker> called">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{broker-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
            </munit-tools:verify-call>

            <!-- Verify Object Store NOT called (happy path) -->
            <munit-tools:verify-call processor="os:store" times="0" doc:name="Verify no error stored">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{os-store-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
            </munit-tools:verify-call>

            <!-- Assert orchestration result -->
            <munit-tools:assert-that expression="#[payload.status]"
                                    is="#[MunitTools::equalTo('SUCCESS')]"/>
            <munit-tools:assert-that expression="#[payload.processedItems]"
                                    is="#[MunitTools::hasSize(2)]"/>
        </munit:validation>
    </munit:test>

    <!-- PAPI Error Handling with Object Store -->
    <munit:test name="scheduler-{entity}-test-orchestration-error" description="Test error handling with Object Store">
        <munit:behavior>
            <!-- Mock SAPI 1 Success -->
            <munit-tools:mock-when processor="http:request">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{sapi1-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <munit-tools:payload value="#[readUrl('classpath://test-data/{entity}/<broker>-get-{entity}.dwl')]" encoding="UTF-8"/>
                    <munit-tools:attributes value="#[{statusCode: 200}]"/>
                </munit-tools:then-return>
            </munit-tools:mock-when>

            <!-- Mock SAPI 2 Failure -->
            <munit-tools:mock-when processor="http:request">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{sapi2-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:then-return>
                    <munit-tools:error typeId="HTTP:CONNECTIVITY"/>
                </munit-tools:then-return>
            </munit-tools:mock-when>

            <!-- Spy on Object Store to verify error storage -->
            <munit-tools:spy processor="os:store">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{os-store-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
                <munit-tools:before-call>
                    <munit-tools:assert-equals actual='#[vars.storeError - "timestamp" - "correlationId"]'
                                              expected='#[{
                                                  "code": 500,
                                                  "type": "HTTP:CONNECTIVITY",
                                                  "message": "Connection failed"
                                              }]'/>
                </munit-tools:before-call>
            </munit-tools:spy>
        </munit:behavior>

        <munit:execution>
            <!-- Same as happy path -->
        </munit:execution>

        <munit:validation>
            <!-- Verify partial processing -->
            <munit-tools:verify-call processor="http:request" times="1" doc:name="Verify SAPI 1 called">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{sapi1-http-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
            </munit-tools:verify-call>

            <!-- Verify error stored -->
            <munit-tools:verify-call processor="os:store" times="1" doc:name="Verify error stored">
                <munit-tools:with-attributes>
                    <munit-tools:with-attribute whereValue="{os-store-doc-id}" attributeName="doc:id"/>
                </munit-tools:with-attributes>
            </munit-tools:verify-call>

            <!-- Assert error response -->
            <munit-tools:assert-equals actual='#[payload.error - "timestamp"]'
                                      expected='#[{
                                          "code": 500,
                                          "message": "Orchestration failed"
                                      }]'/>
        </munit:validation>
    </munit:test>
</mule>
```

## 2.2 Scheduler Flow Template

```xml
<!-- PAPI Scheduler Test -->
<munit:test name="scheduler-{entity}-test-execution" description="Test scheduler flow execution">

    <munit:behavior>
        <!-- Mock multiple backend calls that scheduler orchestrates -->
        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="{sapi-get-pending-doc-id}" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value='#[{
                    items: [
                        {id: "1", status: "PENDING"},
                        {id: "2", status: "PENDING"}
                    ]
                }]' encoding="UTF-8"/>
                <munit-tools:attributes value="#[{statusCode: 200}]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>

        <!-- Mock processing for each item (forEach) -->
        <munit-tools:mock-when processor="http:request">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="{sapi-process-item-doc-id}" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[{processed: true}]" encoding="UTF-8"/>
                <munit-tools:attributes value="#[{statusCode: 200}]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>

    <munit:execution>
        <flow-ref name="scheduler-{entity}-flow" doc:name="Invoke scheduler flow"/>
    </munit:execution>

    <munit:validation>
        <!-- Verify scope variable set -->
        <munit-tools:assert-equals actual="#[vars.scope]"
                                  expected='#["{entity}-scheduler"]'/>

        <!-- Verify forEach executed for each item -->
        <munit-tools:verify-call processor="http:request" times="2" doc:name="Verify process called twice">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="{sapi-process-item-doc-id}" attributeName="doc:id"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>
    </munit:validation>
</munit:test>
```

---

## Complete Working PAPI Test Example

```xml
<!-- Complete PAPI Test: Purchase Orders Orchestration -->
<munit:test name="scheduler-purchase-orders-test-full-orchestration" description="Test complete PO orchestration flow">

    <munit:behavior>
        <!-- Mock <backend-1> SAPI for PO Header -->
        <munit-tools:mock-when processor="http:request" doc:name="Mock <backend-1> PO Header">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="45dbdf2d-10af-4d03-96c1-f5b6c4015d2f" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value='#[{
                    "d": {
                        "results": [{
                            "OrderNumber": "4500000123",
                            "CompanyCode": "1000",
                            "SupplierID": "0000100001",
                            "OrderDate": "2024-01-01T00:00:00Z"
                        }]
                    }
                }]' mediaType="application/json" encoding="UTF-8"/>
                <munit-tools:attributes value="#[{statusCode: 200}]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>

        <!-- Mock <backend-1> SAPI for PO Items -->
        <munit-tools:mock-when processor="http:request" doc:name="Mock <backend-1> PO Items">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="f6ca1b03-71c9-4e67-b988-42f5c36d6aab" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value='#[{
                    "d": {
                        "results": [
                            {"ItemNumber": "00010", "MaterialID": "MAT001", "Quantity": "100"},
                            {"ItemNumber": "00020", "MaterialID": "MAT002", "Quantity": "200"}
                        ]
                    }
                }]' encoding="UTF-8"/>
                <munit-tools:attributes value="#[{statusCode: 200}]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>

        <!-- Mock Supplier SAPI -->
        <munit-tools:mock-when processor="http:request" doc:name="Mock Supplier SAPI">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="3f549153-b888-43b3-944c-40d993e53ff0" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value='#[{
                    "supplierId": "0000100001",
                    "name": "Supplier ABC",
                    "taxId": "XX123456789"
                }]' encoding="UTF-8"/>
                <munit-tools:attributes value="#[{statusCode: 200}]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>

        <!-- Mock Material SAPI (called per item) -->
        <munit-tools:mock-when processor="http:request" doc:name="Mock Material SAPI">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="85e4bc2e-1540-4d80-ac9c-98f2a103234a" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value='#[{
                    "materialId": vars.materialId,
                    "description": "Material Description",
                    "uom": "EA"
                }]' encoding="UTF-8"/>
                <munit-tools:attributes value="#[{statusCode: 200}]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>

        <!-- Mock <broker> API -->
        <munit-tools:mock-when processor="http:request" doc:name="Mock <broker>">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="5cf13767-1ef6-4842-947a-5f77204c881c" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[payload]" encoding="UTF-8"/>
                <munit-tools:attributes value="#[{statusCode: 200}]"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>

        <!-- Spy on <broker> publish -->
        <munit-tools:spy processor="try" doc:name="Spy <broker> publish">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="68edf719-26b0-4930-856b-15c469e4d111" attributeName="doc:id"/>
            </munit-tools:with-attributes>
            <munit-tools:before-call>
                <munit-tools:assert-equals actual="#[vars.topic]"
                                          expected='#["<topic-path>/PurchaseOrder/4500000123"]'/>
                <munit-tools:assert-that expression="#[vars.aggregatedOrder]"
                                        is="#[MunitTools::notNullValue()]"/>
            </munit-tools:before-call>
        </munit-tools:spy>
    </munit:behavior>

    <munit:execution>
        <munit:set-event doc:name="Set orchestration event">
            <munit:variables>
                <munit:variable key="poNumber" value="#['4500000123']"/>
                <munit:variable key="correlationId" value="#['test-corr-123']"/>
                <munit:variable key="scope" value="#['purchase-orders']"/>
                <munit:variable key="companyId" value="#['company-001']"/>
            </munit:variables>
        </munit:set-event>
        <flow-ref name="purchase-orders-orchestration-flow" doc:name="Invoke orchestration"/>
    </munit:execution>

    <munit:validation>
        <!-- Verify all SAPIs called in correct sequence -->
        <munit-tools:verify-call processor="http:request" times="1" doc:name="PO Header called">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="45dbdf2d-10af-4d03-96c1-f5b6c4015d2f" attributeName="doc:id"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>

        <munit-tools:verify-call processor="http:request" times="1" doc:name="PO Items called">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="f6ca1b03-71c9-4e67-b988-42f5c36d6aab" attributeName="doc:id"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>

        <!-- Verify forEach executed for each item -->
        <munit-tools:verify-call processor="http:request" times="2" doc:name="Materials called twice">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="85e4bc2e-1540-4d80-ac9c-98f2a103234a" attributeName="doc:id"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>

        <!-- Verify <broker> called -->
        <munit-tools:verify-call processor="http:request" times="1" doc:name="<broker> called">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute whereValue="5cf13767-1ef6-4842-947a-5f77204c881c" attributeName="doc:id"/>
            </munit-tools:with-attributes>
        </munit-tools:verify-call>

        <!-- Assert orchestration result -->
        <munit-tools:assert-that expression="#[payload.status]"
                                is="#[MunitTools::equalTo('SUCCESS')]"/>
        <munit-tools:assert-that expression="#[payload.purchaseOrderNumber]"
                                is="#[MunitTools::equalTo('4500000123')]"/>
        <munit-tools:assert-that expression="#[payload.itemsProcessed]"
                                is="#[MunitTools::equalTo(2)]"/>
    </munit:validation>
</munit:test>
```
