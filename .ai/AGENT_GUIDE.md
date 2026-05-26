# Agent guide

This file provides guidance to coding agents (Claude Code, Codex, etc.) when working with code in this repository.

## Repository purpose

This repo authors **Claude Code skills** for working with [Kotlin Toolchain](https://kotlin-toolchain.org/dev/) (JetBrains' unified Kotlin CLI; the build engine inside it is Amper). It is not itself a Kotlin project — there is no build system, no tests, and no source code to compile. Each skill is a markdown file that Claude Code loads when its trigger matches.

## Layout

```
skills/
└── <skill-name>/
    └── SKILL.md
```

One skill currently lives here: `skills/kotlin-toolchain/SKILL.md`.

## SKILL.md structure

Each `SKILL.md` is a markdown file with YAML frontmatter:

```yaml
---
name: <skill-name>           # must match the folder name
description: <one paragraph> # see "Description conventions" below
---
```

The body is plain markdown — the prose Claude reads when the skill activates.

### Description conventions

The `description` field is what the harness shows to Claude when deciding whether to invoke the skill. Existing convention here:

- Lead with **what the skill teaches** (e.g. "How to build, run, test, package…").
- Include explicit **TRIGGER** clauses listing file markers, user phrasings, and tools that should fire the skill (e.g. `project.yaml`, `module.yaml`, `./kotlin` wrapper).
- Include explicit **SKIP** clauses for nearby-but-distinct cases the skill should *not* handle (e.g. "SKIP for Gradle/Maven Kotlin projects").

The TRIGGER/SKIP pattern matters: it's what prevents the skill from firing on the wrong projects. Preserve this style when editing or adding skills.

## Editing skills

- Edit `SKILL.md` directly; there is no codegen or build step.
- Keep instructions prescriptive and decision-oriented — the skill body is read in full when activated, so it should answer "what should Claude do next?" rather than catalog every Kotlin Toolchain feature.
- The current `kotlin-toolchain` skill encodes specific opinions (e.g. "do not propose switching to Gradle", "implement build-time codegen as a local plugin, not by hand-writing the output"). Honor and extend that opinionated tone rather than softening it into neutral reference material.

## Reference

Kotlin Toolchain docs and source (used when extending the skill):

- Docs: <https://kotlin-toolchain.org/dev/>
- Source: <https://github.com/JetBrains/kotlin-toolchain>
- Issue tracker: YouTrack project `AMPER`
