# AI Agent Skills for Kotlin Toolchain

A collection of AI agent skills for [Kotlin Toolchain](https://kotlin-toolchain.org/) projects (JetBrains' unified Kotlin CLI), following the Agent Skills standard ([agentskills.io](https://agentskills.io)).

## How Skills Work

Skills are self-contained folders packaging instructions and resources for AI agents. Each contains a `SKILL.md` file with YAML frontmatter (name and description) plus guidance the coding agent follows when the skill is active.

## Skills

- `kotlin-toolchain` — build, run, test, package, lint/check, manage dependencies, and configure Kotlin/Java projects with Kotlin Toolchain. Does not apply to Gradle/Maven projects.

## Installation

```
npx skills add singleton11/kotlin-toolchain-skills
```

### Manual Installation
Copy the desired skill folder into your agent's skills directory:
```
cp -r skills/kotlin-toolchain .claude/skills/
```

## Repository Layout
- `skills/` — directory containing all skills
