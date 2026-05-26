# Code Review

The `/code-review` skill is FLAME's automated review of MuleSoft code. It dispatches a set of sub-agents (one per rule section) in parallel, collects their findings, and produces a single PASS/FAIL verdict with a structured report.

There are two tracks:
- `--dev` — reviews MuleSoft XML, DataWeave, MUnit, and configuration
- `--raml` — reviews RAML API specifications

```bash
/code-review <project-name> --dev|--raml
```

---

## Acceptance Criteria

A review **PASSES** only when *every* applicable threshold is satisfied. **One blocking finding fails the review outright** — no override, no exceptions (Code Review Integrity rule, see `tracks/{dev,raml}/context.md`).

### DEV track

| Severity | Behavior |
|---|---|
| **Blocking** | Any single Blocking finding (count ≥ 1) → **FAIL** |
| **Attention Point (1, 2, or 3 pts)** | Sum points across findings; if total > the configured maximum → **FAIL** |
| **Interactive** | Sub-agent reports; orchestrator presents to the user one at a time; user's answer reclassifies the finding as Blocking, Attention Point (1 pt), or dismissed |
| **Claude Action** | Suggestions only — never fail a review |

**DEV PASS rule**: `0 Blocking AND total Attention Points ≤ max-attention-points` → PASS.

### RAML track

| Severity | Behavior |
|---|---|
| **Critical** | Any single Critical finding (count ≥ 1) → **FAIL** |
| **Medium** | Compute `(violated Medium rules / total Medium rules) × 100`. If percentage > threshold → **FAIL** |
| **Low** | Suggestions only — never fail a review |

**RAML PASS rule**: `0 Critical AND Medium Issues Percentage ≤ medium-threshold-percent` → PASS.

---

## Why These Criteria

The thresholds were chosen to balance **safety** (block production-risk issues) with **velocity** (don't gate every minor warning).

- **Blocking / Critical = auto-fail** — These cover violations that ship real risk: hidden errors, secrets in plain text, missing mocks that hit production systems, broken API contracts. Letting any of them through defeats the purpose of having a gate.
- **Attention Points (DEV) — point-weighted total** — Issues like "missing logger in a non-critical flow" shouldn't fail a review on their own, but five of them stacking up should. The point system (1/2/3 by impact) lets a single high-impact issue carry more weight than three trivial ones.
- **Medium percentage (RAML) — proportional rather than absolute** — RAML files vary wildly in size; a fixed count doesn't scale. Using the percentage of *violated rules* (out of all Medium rules) keeps the bar consistent across small and large APIs.
- **Interactive (DEV)** — Some checks (hardcoded values, unused properties) genuinely depend on intent. Rather than guess, the reviewer asks the user once and lets them classify; the answer is then locked in.
- **Claude Action / Low** — Improvement suggestions that *should* be raised but never warrant blocking a merge.

The Code Review Integrity rule (results cannot be altered by user request) ensures the gate stays meaningful.

---

## Configuring the Thresholds

All thresholds are configurable in `config/framework.yaml`. Defaults are applied when a field is omitted, *except* `review.dev.base-branch` which is required.

```yaml
review:
  dev:
    max-attention-points: 6       # default 6 — DEV PASS if total ≤ this
    base-branch: "main"           # REQUIRED — branch to diff against during pre-flight (e.g. "main", "dev", "develop")
  raml:
    medium-threshold-percent: 50  # default 50 — RAML PASS if Medium % ≤ this

organization:
  project-prefix: ""              # e.g. "myorg" — used for project-naming validation rules (A.1, B.1.1, B.6.2)

raml:
  common-data:                    # the org's shared RAML asset (resourceTypes, securitySchemes, errorResponse, etc.)
    group-id: ""                  # Anypoint Exchange organization ID
    asset-id: "common-data"
    version: ""                   # e.g. "<x.y.z>"

test:
  coverage:
    application-percent: 80       # default 80
    flow-percent: 90              # default 90
    processor-percent: 80         # default 80
```

**Field-by-field reference:**

| Field | Default | Purpose |
|---|---|---|
| `review.dev.max-attention-points` | `6` | DEV track FAIL threshold for total Attention Points. |
| `review.dev.base-branch` | **required** | Branch the pre-flight uses for `git diff` to identify modified files. No fallback — if missing, the skill halts and prompts. |
| `review.raml.medium-threshold-percent` | `50` | RAML track FAIL threshold (% of Medium rules violated). |
| `organization.project-prefix` | `""` | Project-name prefix used in RAML naming rules (Section A.1, B.1.1, B.6.2). Empty = no prefix required. |
| `raml.common-data.{group-id, asset-id, version}` | — | Coordinates of the org's `common-data` Exchange asset, used to validate `exchange.json` dependencies and `!include` paths. |
| `test.coverage.*-percent` | `80 / 90 / 80` | MUnit coverage thresholds enforced by pre-flight C4. Also referenced by the munit-generator workflow. |

---

## Files & Folders

```
skills/review/code-review/
├── README.md                                # this file
├── SKILL.md                                 # entry point (persona + bootstrap)
├── workflow.md                              # orchestrator dispatch logic
└── tracks/
    ├── dev/
    │   ├── context.md                       # severity levels, PASS/FAIL, pre-flight, output format
    │   └── sections/                        # 8 sub-agent rule files (one per section)
    │       ├── project-structure-folders.md (A-I)
    │       ├── properties-pom.md            (A-II)
    │       ├── secure-config-code-quality.md (A-III)
    │       ├── error-handling-flow-logic.md (B-I)
    │       ├── logging-correlationid.md     (B-II)
    │       ├── configuration-management.md  (B-III)
    │       ├── dataweave-quality.md         (B-IV, B-VII)
    │       └── security-testing.md          (B-V, B-VI)
    └── raml/
        ├── context.md                       # severity levels, PASS/FAIL, pre-flight, output format
        └── sections/                        # 5 sub-agent rule files
            ├── project-structure.md         (A)
            ├── root-file-library.md         (B.1 + B.2)
            ├── resources-parameters.md      (B.3 + B.4)
            ├── datatypes.md                 (B.5)
            └── naming-common-data-practices.md (B.6 + B.7 + B.8 + B.9)
```

---

## Workflow at a Glance

The orchestrator (`workflow.md`) sequences six steps. Detailed behavioral rules and output format live in each track's `context.md`.

1. **Argument parsing & track selection** — parse `--dev` / `--raml`; resolve project path.
2. **Load track context** — read `tracks/{dev,raml}/context.md`.
3. **Pre-flight** — DEV: detect modified files via `git diff <base-branch>`; identify project layer + name. RAML: read `exchange.json`, identify root RAML file.
4. **Dispatch sub-agents** — DEV: 8 in parallel; RAML: 5 in parallel.
5. **Resolve Interactive findings** (DEV only) — present each to the user one at a time.
6. **Compile output + verdict** — apply consistency check, determine PASS/FAIL per the track's rules.

---

## Adding or Tuning Rules

- **A new rule**: add it to the appropriate `sections/*.md` file with a severity tag (`<Blocking>`, `<Attention Point — N pts>`, `<Critical>`, `<Medium>`, etc.) and update `RULES_EXPECTED` at the bottom of that file.
- **Adjust an Attention Point's weight**: edit the rule's tag (e.g., `<Attention Point — 2 pts>` → `<Attention Point — 3 pts>`) AND update the point-values table in `tracks/dev/context.md` § Severity Levels.
- **Adjust a track's threshold**: edit the corresponding `review.*` field in `config/framework.yaml`. No code changes needed.
- **Toggle Medium → Critical (RAML)**: change the severity tag in the section file AND update the `Total Medium Rules` list in `tracks/raml/context.md` (it counts how many Medium rules exist).

The Code Review Integrity rule applies to additions too: rules must be objective and applicable from the section files, not from prompt overrides.
