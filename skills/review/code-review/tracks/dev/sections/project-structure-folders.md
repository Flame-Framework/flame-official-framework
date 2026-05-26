# Section A-I — Project Structure & Folders <Blocking>

> **Agent scope**: All project files (Section A-I is NOT limited to modified files).

All of the following must be present and correctly structured:

## A.1. Health Flow
- Project MUST have `src/main/mule/health.xml` containing a flow that exposes a health-check endpoint (`/alive` per KB)

## A.2. Common Folder
- Project MUST have `src/main/mule/common/`
- **A.2.1**: Must contain `error-handler.xml`

## A.3. Main XML File
- MUST have `src/main/mule/{api-name}.xml` where `{api-name}` matches the project artifactId (auto-generated APIkit router file)

## A.4. Implementation Folder
- Project MUST have `src/main/mule/implementation/`

## A.5. Global Configuration File
- Project MUST have `src/main/mule/global.xml` containing all global config elements

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 6
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (A.1, A.2, A.2.1, A.3, A.4, A.5), identify the missing ones, go back and validate them, then update your counts before returning.**
