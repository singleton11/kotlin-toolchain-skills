# AI Agent Skills for Kotlin Toolchain

A collection of AI agent skills for [Kotlin Toolchain](https://kotlin-toolchain.org/) projects (JetBrains' unified Kotlin CLI), following the Agent Skills standard ([agentskills.io](https://agentskills.io)).

## How Skills Work

Skills are self-contained folders packaging instructions and resources for AI agents. Each contains a `SKILL.md` file with YAML frontmatter (name and description) plus guidance the coding agent follows when the skill is active.

## Skills

- `kotlin-toolchain` — build, run, test, package, lint/check, manage dependencies, and configure Kotlin/Java projects with Kotlin Toolchain. Does not apply to Gradle/Maven projects.
- `kotlin-toolchain-plugin-authoring` — author a Kotlin Toolchain local plugin from scratch. Covers `@Configurable Settings`, `@TaskAction`, `@Input`/`@Output`, `plugin.yaml` tasks/commands/generated wiring, file-based task communication, env-var overrides, and the limitations to design around.
- `gradle-to-kotlin-toolchain-plugin` — port an existing Gradle plugin to a Kotlin Toolchain local plugin. Covers the investigation workflow, the Gradle → Toolchain concept-mapping table, architectural mismatches that need redesign (no `project.version`, no `-P` properties, no Groovy hooks), and common pitfalls drawn from real conversions.
- `gradle-to-kotlin-toolchain-project` — migrate an entire Gradle Kotlin/JVM project to Kotlin Toolchain. Covers the inventory + plugin-set decision + CI rewrite workflow, the categories of local plugin that replace Gradle plugins with no native equivalent (git-tag versioning, container-image building, linters, and the `application` plugin's stable `build/libs/<name>.jar`), the Dependabot stub-buildfile trick, and project-level mismatches to watch (no `module.yaml` interpolation, vendored-linter default drift, root-module task-loop traps).

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
