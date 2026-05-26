# Section B-II — CorrelationId in Connectors

> **Before validating this section**, you MUST know the project layer (SAPI, PAPI, EAPI) — provided by the orchestrator from the pre-flight step.

## B-II.2. CorrelationId Implementation

### B-II.2.1. CorrelationId in Connectors <Blocking>
- correlationId MUST always be sent in HTTP requests and any other connectors that support this field

### B-II.2.2. CorrelationId Priority
- Priority is sending it as `vars.correlationId`

### B-II.2.3. Alternative CorrelationId Format <Attention Point — 1 pt>
- Sending only `correlationId` is acceptable but treated as an attention point
- Should suggest using `vars.correlationId` instead

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 3
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (B-II.2.1, B-II.2.2, B-II.2.3), identify the missing ones, go back and validate them, then update your counts before returning.**
