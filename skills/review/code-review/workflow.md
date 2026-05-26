# Code Review — Orchestrator Workflow

> Dispatch logic for the code-review skill. Detailed behavioral rules, severity definitions, pre-flight steps, output format, and PASS/FAIL logic live in the per-track `tracks/{dev,raml}/context.md` files. This workflow only sequences the orchestration.

---

## Step 0 — Argument Parsing & Track Selection

Parse `$ARGUMENTS`:
1. **Project name** (required) — everything except the flag.
2. **Track flag** — `--dev` or `--raml`.

If no flag → ask the user via `AskUserQuestion` which track to run; do not proceed until answered.
If no project name → respond with usage (`/code-review <project-name> --dev|--raml`) and stop.

Resolve the project path; verify the directory exists. If not, suggest the closest match under the configured workspace.

---

## Step 1 — Load Track Context

Read the context file for the chosen track:
- DEV → `tracks/dev/context.md`
- RAML → `tracks/raml/context.md`

The context file provides: severity levels, PASS/FAIL rules, pre-flight steps, sub-agent behavioral rules, sub-agent output format, and the final orchestrator output format. **Do not duplicate any of that content here.**

---

## Step 2 — Pre-flight

Run the pre-flight steps defined in the chosen track's context.md. Save the results — they will be passed to every sub-agent.

- **DEV** pre-flight produces: modified-files list, project layer (SAPI / PAPI / EAPI), project name.
- **RAML** pre-flight produces: root RAML file, `exchange.json` metadata.

---

## Step 3 — Dispatch Sub-Agents

### Step 3a — Resolve the authoritative rule list per section

For each section about to be dispatched, resolve the authoritative list of rule IDs the sub-agent must validate:

1. **If `review.<track>.sections.<section-id>.rules` is configured in `config/framework.yaml`** → use that list. This is the customization path for orgs that want to disable / extend rules.
2. **Otherwise (fallback)** → scan the section file headings for rule IDs (pattern: `## A.N`, `**A.N.M**:`, `### B-I.N`, etc.) and use the result.

**Integrity check (yaml-driven mode only):** every rule ID listed in framework.yaml MUST appear in the corresponding section file's content. If any are missing, halt with a Claude Action: "framework.yaml lists rule `<ID>` for section `<section>`, but `<section-file>` does not define it. Fix the yaml or the section file before retrying."

### Step 3b — Dispatch

#### DEV Track — 8 parallel sub-agents

Dispatch all 8 in parallel using the Agent tool:

| Sub-agent | Section File | framework.yaml key |
|-----------|-------------|-------------------|
| Project Structure & Folders | `tracks/dev/sections/project-structure-folders.md` | `review.dev.sections.A-I` |
| Properties & POM | `tracks/dev/sections/properties-pom.md` | `review.dev.sections.A-II` |
| Secure Config & Code Quality | `tracks/dev/sections/secure-config-code-quality.md` | `review.dev.sections.A-III` |
| Error Handling & Flow Logic | `tracks/dev/sections/error-handling-flow-logic.md` | `review.dev.sections.B-I` |
| Logging & Correlation ID | `tracks/dev/sections/logging-correlationid.md` | `review.dev.sections.B-II` |
| Configuration Management | `tracks/dev/sections/configuration-management.md` | `review.dev.sections.B-III` |
| DataWeave Quality | `tracks/dev/sections/dataweave-quality.md` | `review.dev.sections.B-IV` |
| Security & Testing | `tracks/dev/sections/security-testing.md` | `review.dev.sections.B-V` |

Each sub-agent receives:
- Its section file (rule definitions to validate against).
- **The authoritative rule ID list resolved in Step 3a — the sub-agent MUST validate every ID in this list.**
- Pre-flight results (modified files, project layer, project name).
- **The relevant sections of `<PROJECT_ROOT>/project-context.md` (if present)** — facts, conventions, documented deviations, open issues — for project-specific context.
- The behavioral rules + sub-agent output format from `tracks/dev/context.md`.

#### RAML Track — 5 parallel sub-agents

Dispatch all 5 in parallel using the Agent tool:

| Sub-agent | Section File | framework.yaml key |
|-----------|-------------|-------------------|
| Project Structure | `tracks/raml/sections/project-structure.md` | `review.raml.sections.A` |
| Root File & Library | `tracks/raml/sections/root-file-library.md` | `review.raml.sections.B.1+B.2` |
| Resources & Parameters | `tracks/raml/sections/resources-parameters.md` | `review.raml.sections.B.3+B.4` |
| DataTypes | `tracks/raml/sections/datatypes.md` | `review.raml.sections.B.5` |
| Naming, Common-Data & Practices | `tracks/raml/sections/naming-common-data-practices.md` | `review.raml.sections.B.6+B.7+B.8+B.9` |

Each sub-agent receives its section file, the authoritative rule ID list (from Step 3a), plus the behavioral rules + sub-agent output format from `tracks/raml/context.md`.

### Step 3c — Verify Coverage and Re-dispatch

After all sub-agents return, for each sub-agent:

1. Compare its returned `VALIDATED_LIST` against the authoritative list it was given.
2. If `VALIDATED_LIST` is missing any rule IDs → dispatch ONE follow-up Agent call to the same sub-agent with:
   - The list of missed rule IDs only
   - Instruction: "You did not validate the following rules in your previous response: `<list>`. Re-check your section file for each one and return findings (or `No issues found.` per rule) in the standard format."
3. Repeat up to **2 retries per sub-agent**.
4. If after 2 retries any rule IDs are still missing → emit a Claude Action: "Sub-agent for `<section>` could not validate rules `<list>` after 2 retries. Manual investigation required (likely a stale rule ID in framework.yaml or a section file inconsistency)." Continue compilation with the partial coverage; mark the affected rules in the final report.

## Result Compilation
1. Collect all findings from sub-agents
2. **DEV track only:** cross-reference findings against active deviations using `tracks/dev/deviation-loader.md`. Findings that match an active, valid deviation are rewritten to severity "Acknowledged Deviation" (still listed; do NOT consume the Attention Points budget). Findings against non-waivable rules are never rewritten regardless of deviations.
3. Classify by severity (Blocking, Attention Point, Acknowledged Deviation, Claude Action, Interactive)
4. Present interactive findings one at a time
5. Generate final verdict: PASS / FAIL using the rules in `tracks/dev/context.md`. Acknowledged Deviations do not contribute to FAIL.

---

## Step 4 — Resolve Interactive Findings (DEV track only)

For each finding returned with severity `Interactive`, present it to the user using the format defined in `tracks/dev/context.md` § Interactive Questions — Visual Emphasis. **One finding at a time.** Wait for the user's response, then classify the finding per the rule's classification logic.

The RAML track has no Interactive findings — proceed directly to compilation.

---

## Step 5 — Compile Final Output

Merge all sub-agent findings using the **Review Output Format** in the chosen track's context.md:
- DEV → `tracks/dev/context.md` § Review Output Format
- RAML → `tracks/raml/context.md` § Review Output Format

Apply the **Output Consistency Check** from context.md before finalizing — the counts in the summary must EXACTLY match the items listed.

---

## Step 6 — Determine Verdict

PASS / FAIL is determined per the track's rules in context.md:

- **DEV** — any Blocking finding (count ≥ 1) → **FAIL**. Otherwise, if total Attention Points > `review.dev.max-attention-points` (default `6`, configurable in `config/framework.yaml`) → **FAIL**. Otherwise → **PASS**.
- **RAML** — any Critical finding (count ≥ 1) → **FAIL**. Otherwise, if percentage of violated Medium rules > `review.raml.medium-threshold-percent` (default `50`%, configurable) → **FAIL**. Otherwise → **PASS**.

The verdict is **non-negotiable** — see the Code Review Integrity rule in context.md.
