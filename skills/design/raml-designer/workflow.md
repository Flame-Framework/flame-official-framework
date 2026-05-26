# RAML Designer — Workflow

> This file is loaded by the raml-designer skill. Do not execute independently.

## Phase 1: Requirements Gathering
1. Gather requirements from the user (resources, methods, data types)
2. Confirm understanding before proceeding

## Phase 2: Reuse Assessment
1. Search Exchange for existing fragments that could be reused (follow the procedure in `knowledge/architecture/exchange-discovery.md`)
2. Present any reusable fragments to the user

## Phase 3: RAML Design
1. Design the RAML structure following the organization's RAML template
2. Create data types, examples, and resource definitions
3. Use common-data fragments for shared data types

## Phase 4: Self-Validation
1. Self-validate the RAML specification against RAML code-review standards
2. Fix any issues found before delivery

## Phase 5: Delivery
1. Present the complete RAML specification to the user in the conversation for review

## Knowledge Base Files
Load these lazily as needed:
- `knowledge/raml/api-design.md` — Phase 3
- `knowledge/raml/common-data-fragment.md` — Phase 3
- `knowledge/raml/template-spec.md` — Phase 3
- `knowledge/architecture/exchange-discovery.md` — Phase 2
