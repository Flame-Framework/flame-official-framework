---
name: <skill-name>
description: "<one-liner>"
type: simple | workflow
phase: design | develop | test | review | core
user-invocable: true | false
argument-hint: "<usage>"
mcp-dependencies: []                # MCPs the skill MUST have available to run
mcp-dependencies-optional: []       # MCPs the skill CAN use when present, but works without
knowledge-dependencies: []
allowed-tools: Read, Grep, Glob, Write, Edit, Bash
---

# <Skill Name>

## Role
<The functional role of this skill>

## Communication Style
<How the skill interacts with the user>

## Bootstrap
Before doing anything, read `config/framework.yaml` and `config/organization-standards.md` to load the framework configuration.

## Core Principles
<Inviolable rules>

## Workflow
<!--
- If `type: workflow`: keep this section as a single line referencing the external workflow file:
  `Read `workflow.md` FIRST before proceeding.`
- If `type: simple`: replace this section with a `## Procedure` block containing the numbered steps inline (no separate workflow.md).
-->
