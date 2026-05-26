# KB Inconsistencies Discovered While Building `scaffold` Skill

> Surfaced on 2026-05-25 during the design and dry-run of `skills/develop/scaffold/`. These are inconsistencies **inside the existing KB docs** — they were not introduced by the scaffold work. Each is intentionally **not fixed** in this pass to keep the scaffold change scoped. Decide separately when/whether to address each one; some have non-trivial blast radius (`code-review` rules and `project-context-builder` patterns reference these docs).

---

## 1. Missing `qa` environment in documented project structure

**File**: `knowledge/development/standards/project-structure.md`
**Lines**: 32-39 (the property file tree)

**Issue**: The documented tree lists property files only for `local`, `dev`, `prod` envs:

```
│       └── properties/
│           ├── config-common.yaml
│           ├── config-local.yaml
│           ├── config-dev.yaml
│           ├── config-prod.yaml
│           ├── secure-config-local.yaml
│           ├── secure-config-dev.yaml
│           └── secure-config-prod.yaml
```

But the real `flame-pokemon-sapi` project has files for **four** envs: `local`, `dev`, `qa`, `prod`. `standards/configuration-properties.md` (the deeper reference for this topic) also discusses 4-env layouts. `scaffold` generates the 4-env layout to match reality.

**Suggested fix**: Add `config-qa.yaml` and `secure-config-qa.yaml` to the tree in `project-structure.md`.

**Impact if changed**: Low. Aligns the doc with what's already happening in practice and what other docs imply.

---

## 2. Connector config naming case inconsistency

**Files**:
- `knowledge/development/standards/project-structure.md` line 98: uses `HTTP_Listener_Config` (capital **C**)
- `knowledge/development/standards/naming-conventions.md` line 74: uses `HTTP_Listener_config` (lowercase **c**)

**Issue**: Two authoritative docs disagree on the casing of the trailing `Config`/`config` segment. Real projects (`flame-pokemon-sapi`) use **lowercase `config`**, matching `naming-conventions.md`.

**Suggested fix**: Pick one (recommend `_config` lowercase to match practice and naming-conventions.md), update the other doc.

**Impact if changed**: Medium.
- `code-review` may have rules referencing one form or the other — would need to be checked.
- `project-context-builder` detects patterns like `HTTP_Listener_*` — case-insensitive recognition probably works but capturing the canonical form matters.
- Any project that follows the "wrong" form (per the eventual canonical choice) would surface as a deviation in code-review.

---

## 3. `.gitignore` list incorrectly includes RAML source folders

**File**: `knowledge/development/standards/project-structure.md`
**Line**: 176

**Issue**: The doc's project-structure checklist line says:

> "Exchange downloaded files are gitignored (exchange_modules/, datatypes/, library/, traits/)"

`exchange_modules/` is genuinely auto-managed by APIkit/Maven and correct to ignore. But `datatypes/`, `library/`, and `traits/` are **user-authored RAML folders** — they are first-class project files and are committed to source control (verified in `flame-pokemon-sapi`).

**Suggested fix**: Remove `datatypes/`, `library/`, `traits/` from that line, keep only `exchange_modules/`.

**Impact if changed**: Low. Aligns the doc with reality. No downstream rules reference this specific gitignore line.

---

## What was done about each item

- For item 1: `scaffold` generates 4 envs anyway (matches reality, not the doc).
- For item 2: `scaffold` uses `HTTP_Listener_config` (lowercase, matches naming-conventions.md and reality).
- For item 3: `scaffold`'s generated `.gitignore` ignores only `exchange_modules/`, not the RAML folders, and includes a comment pointing back to this notes file.

No edits to the KB docs in this pass.
