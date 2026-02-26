# Spring Boot Development Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for building Spring Boot applications with version-appropriate best practices.

## What It Does

- Detects Spring Boot and Java versions from your build file
- Applies version-specific patterns (Boot 3.5.x vs 4.0.x)
- Enables virtual threads by default on Java 21+
- Configures JSpecify/NullAway null safety
- Sets up observability (OpenAPI, Micrometer, distributed tracing)
- Writes JavaDocs and JUnit tests for every change
- Runs the build and checks SonarQube for violations
- Verifies APIs against live documentation for the detected version

## Supported Versions

| Spring Boot | Spring Framework | Java | Jackson | JUnit |
|-------------|-----------------|------|---------|-------|
| 3.5.x | 6.x | 17, 21 | 2.x | 5 (Jupiter) |
| 4.0.x | 7.x | 17, 21, 25 | 3.x | 6 |

## Usage

Install as a Claude Code skill, then from any Spring Boot project:

```
Add a REST endpoint for user registration
```

```
Set up observability with OpenTelemetry
```

```
Enable NullAway with JSpecify annotations
```

## Installation

Add this skill to your Claude Code configuration:

```bash
claude mcp add-skill spring-boot https://github.com/adityamparikh/spring-boot-skill
```

Or clone into your skills directory:

```bash
git clone https://github.com/adityamparikh/spring-boot-skill.git ~/.claude/skills/spring-boot
```

## License

MIT
