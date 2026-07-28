# Agent guide

## Repository purpose

Authors agent skills for [Kotlin Toolchain](https://kotlin-toolchain.org/dev/) (JetBrains' unified Kotlin
CLI; Amper is the build engine inside it). There is no build system, no tests, and no source to compile —
only markdown.

## Layout

```
skills/
└── <skill-name>/
    ├── SKILL.md              # frontmatter + body
    └── references/           # optional; loaded on demand
        └── examples.md
```

Four skills: `kotlin-toolchain`, `kotlin-toolchain-plugin-authoring`, `gradle-to-kotlin-toolchain-plugin`,
`gradle-to-kotlin-toolchain-project`.

## Frontmatter

```yaml
---
name: <skill-name>    # must match the folder name
description: <one paragraph>
---
```

The `description` is what the harness sees when deciding whether to activate the skill. Convention:

- Lead with what the skill teaches ("How to build, run, test, package…").
- **TRIGGER** clauses: file markers, user phrasings, identifiers that should fire it.
- **SKIP** clauses: nearby cases another skill owns.

Keep TRIGGER/SKIP coverage even when trimming prose — it's what stops the skill firing on the wrong project.

## Editing skills

- Edit the markdown directly; there is no codegen step.
- Keep instructions prescriptive and decision-oriented: "what should the agent do next?", not a catalog of
  Toolchain features.
- Honor the existing opinions ("do not propose switching to Gradle", "implement build-time codegen as a
  local plugin, not by hand-writing the output"). Don't soften them into neutral reference material.
- One fact, one home. If something already lives in another skill, link to that skill by name in a clause,
  don't restate it. `kotlin-toolchain` owns project syntax; `kotlin-toolchain-plugin-authoring` owns plugin
  mechanics; the two `gradle-to-*` skills own only their conversion deltas.
- Prose rules: no sentences about the document itself, no sentences explaining why the previous sentence is
  useful, imperative mood, ≤1 bold span per bullet, no restating a code block in a "Key points" list.
- Never paraphrase error strings, flags, env-var names, Maven coordinates, or YAML keys.

## Reference

- Docs: <https://kotlin-toolchain.org/dev/>
- Source: <https://github.com/JetBrains/kotlin-toolchain>
- Issue tracker: YouTrack project `KTC`
