# Section B-IV — DataWeave Quality & Best Practices

## B-IV.1. Shared DWL File Modification <Blocking>
- Validate that NO change was made to a DWL file that is shared by multiple Transform Messages
- If a DWL file is used in multiple places, it CANNOT be modified without ensuring all usages are compatible

## B-IV.2. Hardcoded or Missing Field Mappings in DWL Files <Interactive>
- Validate that DWL transformation files do NOT have hardcoded empty strings or static values where a field mapping should exist
- **How to detect**: Analyze the DWL transformation structure and look for any field that is assigned a hardcoded value (e.g., `""`, `null`, `0`, or any static string/number) instead of mapping from the source payload, variables, or properties
- **Example of violation**:
  ```dataweave
  {
    validFrom: parseDate($.Audat) default "",      // Correct - maps from source payload
    quotationType: "",                              // VIOLATION - hardcoded empty string
    quotationCreator: $.Ernam default "",          // Correct - maps from source payload
    environment: p('mule.env'),                    // Correct - maps from property
    correlationId: vars.correlationId,             // Correct - maps from variable
  }
  ```
- **Why this matters**: Hardcoded values in transformations cause data loss - the actual data from the source system is ignored and replaced with static values
- Additionally, you can compare with similar DWL files to identify inconsistencies, but the primary detection method is analyzing the transformation structure itself
- **Interactive confirmation required**: For EACH hardcoded or missing field mapping found, you MUST report it as an Interactive finding. Do NOT self-dismiss or decide on behalf of the user.
  1. Present the finding with the exact DWL file path, line number, and the hardcoded/missing mapping detected
  2. Question to ask: _"This appears to be a hardcoded value in a DWL mapping. Was this intentional?"_
  3. Options:
     - **YES** — it is hardcoded and it was intentional → classify as **Attention Point (1 pt)**
     - **NO** — it is hardcoded and it was NOT intentional → classify as **Blocking**
     - **It's not hardcoded** — the reviewer misidentified it → dismiss the finding
- Each hardcoded mapping is classified independently — a single DWL file may produce Attention Points (1 pt each), Blocking issues, or passed findings depending on the user's answers

## B-IV.3. DWL Folder Naming <Claude Action>
- Give suggestions regarding the naming of folders used to store DWL files ONLY WHEN NECESSARY
- Look for inconsistencies such as:
  - `dw` vs `dwl`
  - `dwl.payload` vs `dwl.payloads`
  - Other similar inconsistencies that should be unified

## B-IV.4. Inline DataWeave in Transform Messages <Attention Point — 2 pts>
- Validate that Transform Message components use **external `.dwl` files** (`resource="dwl/..."`) instead of inline CDATA blocks
- **How to detect**: Look for `<ee:transform>` components where the DataWeave code is written inline inside `<![CDATA[...]]>` rather than referenced via the `resource` attribute
- **Example of violation**:
  ```xml
  <!-- VIOLATION - inline DataWeave -->
  <ee:transform doc:name="Transform Message">
    <ee:message>
      <ee:set-payload><![CDATA[%dw 2.0
  output application/json
  ---
  { status: "ok" }]]></ee:set-payload>
    </ee:message>
  </ee:transform>
  ```
- **Correct usage**:
  ```xml
  <!-- CORRECT - external DWL file -->
  <ee:transform doc:name="Transform Message">
    <ee:message>
      <ee:set-payload resource="dwl/transform-response.dwl" />
    </ee:message>
  </ee:transform>
  ```
- **Exception**: Simple single-value assignments (e.g., `<ee:set-variable variableName="httpStatus"><![CDATA[200]]></ee:set-variable>`) that contain only a literal value or a trivial one-line expression are exempt from this rule
- **Why this matters**: External DWL files improve code readability, enable reuse across multiple transforms, simplify unit testing of transformations in isolation, and make diffs cleaner during code reviews

## B-VII.1. Best Practices Suggestions <Claude Action>
- Suggest development improvements based on market best practices
- **Reference**: Read and use `knowledge/development/anypoint-studio-best-practices.md` (workspace-relative)
- Suggestions are Claude Action severity — only suggest WHEN NECESSARY

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 5
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (B-IV.1, B-IV.2, B-IV.3, B-IV.4, B-VII.1), identify the missing ones, go back and validate them, then update your counts before returning.**
