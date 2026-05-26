# Section B-III — Configuration Management

> **ALL-file search required**: Rule B-III.1 requires searching ALL XML files in `src/main/mule/` to ensure no global configurations exist outside `global.xml`, but only report issues if the violation is in a **modified file** or if the **modified file introduced** the violation.

## B-III.1. Global Configuration Location <Blocking>
- Ensure that ALL global configs are inside `global.xml`
- No global configurations should exist in other XML files

### B-III.1.1. Mandatory Search for Global Configurations
**You MUST search ALL XML files in `src/main/mule/` for the following global configuration elements:**

| Element | Description |
|---------|-------------|
| `http:listener-config` | HTTP Listener configuration |
| `http:request-config` | HTTP Request configuration |
| `configuration-properties` | Property file configuration |
| `secure-properties:config` | Secure properties configuration |
| `apikit:config` | APIkit configuration |
| `os:object-store` | Object Store configuration |
| `ee:object-store-caching-strategy` | Caching Strategy configuration |
| `db:config` | Database configuration |
| `vm:config` | VM configuration |
| `jms:config` | JMS configuration |
| `api-gateway:autodiscovery` | API Autodiscovery |
| `tls:context` | TLS Context configuration |
| `oauth:*` | OAuth configurations |
| `<configuration` | Default error handler configuration |

**Search Command**: Search for these patterns in ALL `.xml` files under `src/main/mule/`

**Rule**: If ANY of these elements are found in files OTHER than `global.xml`, it is a **BLOCKING** issue.

> **IMPORTANT**: Do NOT rely only on checking `global.xml`. You MUST actively search ALL other XML files to ensure no global configurations exist outside `global.xml`.

## B-III.2. Environment Configuration Consistency <Attention Point — 3 pts>
- Ensure that data structures across all configured environment property files are identical (e.g., `config-local.yaml`, `config-dev.yaml`, `config-prod.yaml` — environment names vary per organization)
- Values and comments MUST be ignored in this comparison
- Only the structure (keys/properties) must match

## B-III.3. Hardcoded Values <Interactive>
- Validate that NO hardcoded value exists where a property reference should be used
- Values that vary by environment or are configuration-related should be in property files
- **Interactive confirmation required**: For EACH hardcoded value found, you MUST report it as an Interactive finding. Do NOT self-dismiss or decide on behalf of the user.
  1. Present the finding with the exact file path, line number, and the hardcoded value detected
  2. Question to ask: _"This appears to be a hardcoded value. Was this intentional?"_
  3. Options:
     - **YES** — it is hardcoded and it was intentional → classify as **Attention Point (1 pt)**
     - **NO** — it is hardcoded and it was NOT intentional → classify as **Blocking**
     - **It's not hardcoded** — the reviewer misidentified it → dismiss the finding
- Each hardcoded value is classified independently — a single file may produce Attention Points (1 pt each), Blocking issues, or passed findings depending on the user's answers

## B-III.4. Unused Properties <Interactive>
- For modified property files (`.yaml` files under `src/main/resources/properties/`), validate that each property key is actually referenced somewhere in the project
- **How to detect**:
  1. For each property key in the modified `.yaml` file, search for references in:
     - **XML files** (`src/main/mule/**/*.xml`): look for `${property.name}` or `p('property.name')` patterns
     - **DWL files** (`src/main/resources/**/*.dwl`): look for `Mule::p('property.name')` or `p('property.name')` patterns
     - **Other property files** (`src/main/resources/properties/*.yaml`): look for references using `${property.name}`
  2. If a property key has **zero references** across these files, it is potentially unused
- **Why this matters**: Unused properties add noise to configuration files, make environment setup harder, and can cause confusion about which values are actually needed by the application
- **Interactive confirmation required**: For EACH potentially unused property found, you MUST report it as an Interactive finding. Do NOT self-dismiss or decide on behalf of the user.
  1. Present the finding with the exact property file path, the property key name, and its value
  2. Question to ask: _"This property appears to be unused. Is it really unused?"_
  3. Options:
     - **YES** — it is unused and should be removed → classify as **Blocking**
     - **NO** — it is not unused (e.g., used by an external system, CI/CD, or parent POM) → dismiss the finding
- Each property is classified independently

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 4
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (B-III.1, B-III.2, B-III.3, B-III.4), identify the missing ones, go back and validate them, then update your counts before returning.**
