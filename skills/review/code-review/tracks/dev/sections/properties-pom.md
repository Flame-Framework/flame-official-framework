# Section A-II — Properties & POM <Blocking>

> **Agent scope**: All project files (Section A-II is NOT limited to modified files).

All of the following must be present and correctly structured:

## A.5. Properties Folder Structure
- Must have a properties folder inside `src/main/resources`
- **A.5.1**: Must include `config.yaml`, `common-config.yaml`, or `config-common.yaml`
- **A.5.2**: Must include a local-environment config file (e.g., `config-local.yaml`)
- **A.5.3**: Must include a development-environment config file (e.g., `config-dev.yaml`)
- **A.5.4**: Must include a production-environment config file (e.g., `config-prod.yaml`)

## A.6. Parent POM
- Must have a `<parent>` element declared in `pom.xml`
- If `parent-pom` is configured in `config/framework.yaml`, the project's `<parent>` MUST match the configured `group-id`, `artifact-id`, and `version`

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 6
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (A.5, A.5.1, A.5.2, A.5.3, A.5.4, A.6), identify the missing ones, go back and validate them, then update your counts before returning.**
