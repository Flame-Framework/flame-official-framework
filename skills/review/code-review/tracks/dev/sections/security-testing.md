# Section B-V — Security & Testing

## Mandatory Sensitive Data Grep Patterns

You MUST search for ALL of these patterns across the project files before concluding B-V validation. Do NOT rely on ad-hoc scanning — run these searches systematically.

**Credential patterns**:
`password`, `passwd`, `pwd`, `secret`, `api_key`, `apikey`, `api-key`, `token`, `access_token`, `refresh_token`, `bearer`, `credential`, `auth`, `client_secret`, `client_id`, `private_key`, `encryption_key`, `signing_key`, `connectionString`, `connection_string`, `jdbc`

**PII patterns**:
`userId`, `user_id`, `email`, `tenant`, `tenantId`, `tenant_id`, `ssn`, `social_security`, `credit_card`, `card_number`, `phone_number`, `address`

**Search in these file types**: `*.xml`, `*.yaml`, `*.yml`, `*.properties`, `*.json`, `*.dwl`

**Exclude from search**: `pom.xml`, `log4j2.xml`, `exchange_modules/`, `.git/`

For each match found, evaluate whether the value is actual sensitive data or just a field name/reference. Report all potentially sensitive values as Interactive findings.

---

## B-V.1. No Unencrypted Sensitive Data <Interactive>
- Validate that NO sensitive data is unencrypted
- Sensitive data includes: credentials, passwords, API keys, tokens, access keys, secrets, etc.
- All sensitive data MUST be in secure-config files and properly encrypted
- **Interactive confirmation required**: For EACH value flagged as potentially sensitive, you MUST report it as an Interactive finding. Do NOT self-dismiss or decide on behalf of the user.
  1. Present the finding with the exact file path, line number, and the suspected sensitive value
  2. Question to ask: _"This appears to be unencrypted sensitive data. Is this actually sensitive?"_
  3. Options:
     - **YES** — it is sensitive data and must be encrypted → classify as **Blocking**
     - **It's not sensitive data** — the reviewer misidentified it (e.g., pagination token, dictionary key) → dismiss the finding
- Each finding is classified independently

## B-V.2. No Sensitive Data in MUnit Tests <Interactive>
- Validate that MUnit test files do NOT contain any sensitive data, even if they appear to be "test" or "dummy" values
- This includes `client_id`, `client_secret`, API keys, tokens, passwords, or any credential-like values in:
  - `munit:attributes` (headers, queryParams, etc.)
  - `munit:variables`
  - `munit-tools:payload`
  - Any other test configuration
- **Why this matters**: Test files are committed to version control and can be mistaken for real credentials, used as templates leading to real credential exposure, or flagged in security audits
- **Required Action**: Use clearly fake placeholder values like `"test-client-id"`, `"test-client-secret"`, or remove them entirely if not needed for the test
- **Interactive confirmation required**: For EACH value flagged as potentially sensitive in MUnit files, you MUST report it as an Interactive finding. Do NOT self-dismiss or decide on behalf of the user.
  1. Present the finding with the exact MUnit file path, line number, and the suspected sensitive value
  2. Question to ask: _"This appears to be sensitive data in a MUnit test. Is this actually sensitive?"_
  3. Options:
     - **YES** — it is sensitive data and must be replaced with fake placeholders → classify as **Blocking**
     - **It's not sensitive data** — the reviewer misidentified it → dismiss the finding
- Each finding is classified independently

---

## B-VI.1. MUnit Tests Exist <Blocking>
- Validate that there is a MUnit test covering the development performed

## B-VI.2. MUnit Suite for New Flows <Blocking>
- For NEW flows, validate that a suite exists referencing the developed flow
- This applies only to new flows, not modifications to existing flows

## B-VI Remediation Suggestion
If **B-VI.1 or B-VI.2 is violated**, include this suggestion in the finding's ACTION field:
> "Would you like me to run `/munit-generator` to generate the missing MUnit tests for the affected flows?"

---

## Mandatory Verification

After completing ALL validations, append this to your output:

```
RULES_EXPECTED: 4
RULES_VALIDATED: [count of distinct rules you checked]
VALIDATED_LIST: [comma-separated list of rule IDs you checked]
```

**If RULES_VALIDATED < RULES_EXPECTED, compare your VALIDATED_LIST against the expected rules (B-V.1, B-V.2, B-VI.1, B-VI.2), identify the missing ones, go back and validate them, then update your counts before returning.**
