# FLAME — Full Lifecycle Applications MuleSoft Engine

AI-powered toolkit for MuleSoft development. Installed once, operates on any project.

## First-Time Setup

### 1. Install skills (automatic — Claude performs this step)

The user does **not** run any command manually. When Claude reads this `SETUP.md`, it performs the install itself. The user can have cloned FLAME to any location — `~/Desktop/FLAME`, `~/projects/flame`, `~/work/flame`, anywhere — so paths must be derived dynamically.

**Instructions for Claude:**

1. **Detect the FLAME root.** You read `SETUP.md` from an absolute path. The directory containing it is the FLAME root — call it `<FLAME_ROOT>`. Do not hardcode any guess; use the actual path you read.

2. **Ensure the target directory exists.** Run `mkdir -p ~/.claude/skills`.

3. **Symlink every skill into `~/.claude/skills/`.** Each skill lives at `<FLAME_ROOT>/skills/<phase>/<skill-name>/` (e.g. `skills/develop/mule-developer/`). Claude Code expects each skill as a **direct child** of `~/.claude/skills/`, so the symlink must point one level deeper than the phase folder.

   Run this loop, substituting the real `<FLAME_ROOT>` absolute path:
   ```bash
   for skill in <FLAME_ROOT>/skills/*/*/; do
     name=$(basename "$skill")
     ln -sfn "$skill" ~/.claude/skills/"$name"
   done
   ```
   `-s` makes it a symbolic link, `-f` overwrites a stale link of the same name, `-n` prevents following an existing symlink-to-directory.

4. **Verify.** Run `ls -l ~/.claude/skills/` and confirm each FLAME skill appears as a symlink pointing back into `<FLAME_ROOT>`. Report the list of installed skills to the user.

5. **Conflict check.** If a non-FLAME skill with the same name already exists in `~/.claude/skills/` (a real folder, not a symlink into FLAME), do **not** overwrite it. Stop, list the conflicts, and ask the user how to proceed.

### 2. Configure the framework

If `config/framework.yaml` is empty (only comments), you must run the setup below. Read `config/framework.yaml` — if it only contains comments, ask the user the following questions to generate the configuration. Write the answers directly into `config/framework.yaml`.

**Mandatory questions — ask these always:**

1. "What is your organization name?" → `organization.name`
2. "What prefix do your MuleSoft projects use? (e.g., `acme-`, `myorg-`)" → `organization.project-prefix`
3. "What is the full path to the workspace folder where your MuleSoft projects live? (e.g., `~/AnypointStudio/studio-workspace`)" → `workspaces` (accept multiple paths, one per line)

**Optional questions — ask these and accept "no" or "skip" as valid answers:**

4. "Does your team use a custom parent POM for MuleSoft projects? If yes, provide the groupId, artifactId, and version. If not, just say no."
   - If yes → fill `parent-pom.group-id`, `parent-pom.artifact-id`, `parent-pom.version`
   - If no → omit the `parent-pom` section entirely

5. "Do you want to install FLAME's MCP to connect with Anypoint Platform?
   1. Yes, install FLAME's MCP
   2. No, skip MCP"

   **If option 1 — install FLAME's MCP:**

   The MCP server's own `README.md` is the **source of truth** for installation steps. SETUP.md must not duplicate those steps, because they may change as the MCP evolves. Claude's job is to clone the repo, read the README, walk the user through it, and verify at the end that the MCP actually works.

   1. **Ask where to clone.** Default location: `~/.flame/mcp/anypoint-mcp-server`. Accept any path the user prefers. Call the chosen path `<MCP_PATH>`.

   2. **Clone the repo:**
      ```bash
      git clone https://github.com/Flame-Framework/anypoint-mcp-server.git <MCP_PATH>
      ```

   3. **Read the cloned README.** Use the Read tool on `<MCP_PATH>/README.md`. Treat its install procedure as authoritative — do not invent steps that are not in the README.

   4. **Walk the user through the README's steps, one at a time.** For each step:
      - Explain in plain language what it does and why.
      - If the step can be automated (e.g. running `npm install`, fetching the User ID from `/accounts/api/me`, writing a config file), run it yourself.
      - If the step needs the user (e.g. creating a Connected App in Anypoint Platform → Access Management → Connected Apps, providing Client ID/Secret), guide them through it and wait for their response.
      - Do not skip ahead. Confirm each step succeeded before moving on.

   5. **Update `framework.yaml`.** Add `mcp.servers.anypoint: { enabled: true }` so FLAME skills know the MCP is available.

   6. **Restart prompt.** Tell the user to restart Claude Code so the new MCP server loads. The current session ends here — the test in step 7 happens after they return.

   7. **Test the MCP (after restart).** When the user comes back in a new session, before proceeding with anything else, verify the install:
      - Call a read-only Anypoint MCP tool such as `mcp__anypoint__anypoint_get_organization`.
      - If it returns data → the MCP is live. Report success and continue.
      - If it errors or the tool is not available → diagnose: did the JSON config get written correctly? Are credentials valid? Did Claude Code actually reload? Walk the user through fixing whatever broke, then re-test.

   8. **Capture the target Business Group.** Some Anypoint customers have multiple business groups under a single Connected App. Determine where FLAME should publish Exchange assets and create Design Center projects:
      - Use the response from `mcp__anypoint__anypoint_get_organization` (already called in step 7) to retrieve the user's home organization.
      - If the response includes `subOrganizations`, present the list to the user via `AskUserQuestion`:
        > "Which Anypoint Business Group should FLAME publish to?"
        > 1. `<Home Org Name>` (root)
        > 2. `<Sub-org 1 name>`
        > 3. `<Sub-org 2 name>`
        > ...
      - If there are no sub-orgs (single business group), confirm with the user: "FLAME will publish to `<Home Org Name>`. OK?"
      - Record the chosen Business Group's `groupId` (organization ID) and human-readable name in `framework.yaml` under `anypoint.business-group`.

   **If option 2 — skip:**
   - Omit the `mcp` section entirely from `framework.yaml`. FLAME skills that have MCP-dependent enhancements will degrade gracefully (per each skill's `mcp-dependencies-optional` list).

6. "Does your organization publish shared RAML fragments (libraries, common-data, security schemes, etc.) to Anypoint Exchange that projects should import from?"

   **If Q5 was option 1 (FLAME MCP installed):**
   - Use the MCP to query Exchange for RAML fragments/libraries published by the user's organization (`mcp__anypoint__anypoint_search_assets` filtered by RAML library/fragment classifiers).
   - **If fragments found** → present the list to the user.
     - For each fragment, ask: "Is this **mandatory** (all RAML projects must use it) or **recommended** (suggested but not required)?"
     - Record each chosen fragment under `raml.shared-fragments` as a list entry with `group-id`, `asset-id`, `version`, and `enforcement: mandatory|recommended`.
     - If user picks none → omit the `raml.shared-fragments` section.
   - **If no fragments found** → ask: "I couldn't find any shared RAML fragments in your Exchange. Is that correct?"
     - If user confirms → omit the section.
     - If user says they DO have fragments → fall back to the manual entry flow below.

   **If Q5 was option 2 (no MCP):**
   - Ask directly: "Do you have shared RAML fragments published to Anypoint Exchange that projects should import from?"
   - If no → omit the section.
   - If yes → for each fragment, collect: `group-id` (organization/business ID), `asset-id`, `version`, and ask whether it's mandatory or recommended. Repeat until the user has none left to add.
   - Record under `raml.shared-fragments` as a list.

7. "For code reviews, what is the maximum number of attention points before a review fails? (default: 6)"
   - If user provides a number → `review.dev.max-attention-points`
   - If user says "default" or skips → use 6

8. "For RAML reviews, what percentage of medium rules failing causes a review to fail? (default: 50)"
   - If user provides a number → `review.raml.medium-threshold-percent`
   - If user says "default" or skips → use 50

**After all answers, generate `config/framework.yaml` with this structure:**

```yaml
framework:
  name: "FLAME"
  version: "1.0.0"

organization:
  name: "<answer 1>"
  project-prefix: "<answer 2>"
  api-naming: "<answer 2>{domain}-{layer}"
  layers:
    sapi: { code: "sys", full: "System API" }
    papi: { code: "prc", full: "Process API" }
    eapi: { code: "exp", full: "Experience API" }

workspaces:
  - "<answer 3>"

# Optional sections below — only include if the user provided values

parent-pom:
  group-id: "<answer 4>"
  artifact-id: "<answer 4>"
  version: "<answer 4>"

mcp:
  servers:
    <server-name>: { enabled: true }

anypoint:
  business-group:
    id: "<answer 5 step 8 — chosen org/sub-org id>"
    name: "<answer 5 step 8 — chosen org/sub-org name>"

raml:
  shared-fragments:
    - group-id: "<answer 6, fragment 1>"
      asset-id: "<answer 6, fragment 1>"
      version: "<answer 6, fragment 1>"
      enforcement: "<mandatory or recommended>"
    # Add one entry per fragment. Omit the entire `raml.shared-fragments` block if the user has none.

review:
  dev:
    max-attention-points: <answer 7 or 6>
  raml:
    medium-threshold-percent: <answer 8 or 50>
```

## Your first project

FLAME is now installed framework-wide. It does **not** scaffold new MuleSoft projects — Anypoint Studio does that. For each new project:

1. In Anypoint Studio: **File → New → Mule Project**.
2. Choose **Generate from Exchange** (recommended) or **Generate from local file** and point at your RAML.
3. Let Studio create the project under one of your configured workspaces (the paths from `workspaces:` in `framework.yaml`). Studio produces `pom.xml` (with `mule-apikit-module` and connectors), `mule-artifact.json`, and the APIkit-routed flow skeletons.
4. Run `/mule-developer <project-name>` from any working directory. The skill will locate the Studio-generated folder by searching your workspaces, then auto-invoke `project-context-builder` on first run.

If you run `/mule-developer <project-name>` for a name FLAME can't find in any workspace, the skill pauses and asks whether it's a new project (it'll point you to the Studio steps above), an existing project at a different location, or a typo.

## Available Skills

| Command | Phase | Description |
|---------|-------|-------------|
| `/helper` | Core | Senior Developer assistant |
| `/raml-designer` | Design | RAML 1.0 specification design |
| `/mule-developer <project>` | Develop | MuleSoft implementation (lazy-loading, checkpoint/resume) |
| `/munit-generator <project>` | Test | MUnit test generation (Mode A) and fixing (Mode B) |
| `/code-review <project> --dev\|--raml` | Review | Multi-sub-agent code review |
| `project-context-builder` _(invoked by `/mule-developer`)_ | Discover | Auto-discovers project facts, conventions, and approved deviations into `project-context.md` |

## Structure

```
FLAME/
├── config/          # Organization-wide settings
├── skills/          # All skills organized by lifecycle phase
│   ├── core/        # Cross-cutting utilities
│   ├── design/      # Phase 1: API design and architecture
│   ├── develop/     # Phase 2: MuleSoft implementation
│   ├── test/        # Phase 3: Unit and integration testing
│   └── review/      # Phase 4: Code review
├── knowledge/       # Domain knowledge bases (lazy-loadable)
├── templates/       # Output templates
├── examples/        # Reference examples
├── mcp/             # MCP server submodules
└── scripts/         # Install, update, validate
```

## Updating

From the directory where you cloned FLAME:

```bash
git pull
```

If an `./scripts/update.sh` exists in the repo, run that after pulling — it handles any post-update tasks (re-linking new skills, etc.).
