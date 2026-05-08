# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This repository is a **Claude Code skill** — not a Spring Boot application. It provides instructions that Claude Code follows when working on Spring Boot projects. There is no build system, application code, or tests to run here.

## Repository Structure

- **SKILL.md** — The skill definition file. Contains the frontmatter (name, description, allowed-tools, argument-hint) and the full set of guidelines Claude Code applies to Spring Boot projects: version detection, virtual threads, JSpecify null safety, observability, and the build-then-verify workflow.
- **references/** — Supplementary reference docs that SKILL.md points to:
  - `nullaway-setup.md` — Error Prone + NullAway build configuration for Maven and Gradle
  - `observability.md` — OpenAPI (springdoc), Micrometer metrics, and distributed tracing setup
- **README.md** — Installation instructions and supported version matrix

## How the Skill Works

When Claude Code enters a Spring Boot project and this skill is active:

1. SKILL.md is loaded into context via the skill frontmatter
2. Claude reads the target project's build file to detect Spring Boot and Java versions
3. The version-specific sections in SKILL.md determine which patterns apply (3.5.x vs 4.0.x)
4. References are read on demand when the task involves NullAway or observability
5. Claude verifies APIs against live documentation via Context7 (see "Verify Library and Framework Usage" in SKILL.md)
6. After the build passes, SonarQube analysis runs to check for violations (workflow steps 8–9 in SKILL.md)

## Editing Guidelines

- Keep SKILL.md self-contained for the core workflow. Only factor out content into `references/` when it's lengthy build configuration or setup recipes that are read conditionally.
- The frontmatter `description` field in SKILL.md controls when Claude Code activates this skill — update it if adding new trigger scenarios.
- Version-specific differences live in the "Version-Specific Differences" section of SKILL.md. When Spring Boot releases new versions, add a new entry there and update the compatibility table.
- The `allowed-tools` frontmatter field controls which tools the skill can use. Add new tool patterns there if the skill needs access to additional MCP servers, and ensure corresponding workflow steps in SKILL.md reference those tools.
