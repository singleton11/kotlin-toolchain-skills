---
name: kotlin-toolchain-plugin-authoring
description: Author a Kotlin Toolchain local plugin to extend the declarative build with code generation, build-time processing, custom verification, custom packaging, or any behaviour that `module.yaml` cannot express. TRIGGER when the user asks to write, author, create, design, or extend a Kotlin Toolchain plugin or when during conversion Gradle project to Kotlin Toolchain you realized that there is no native built-in functionality in Kotlin Toolchain that covers part of the Gradle build; when the user references `@TaskAction`, `@Configurable`, `@Input`/`@Output`, `pluginInfo`, `plugin.yaml`, `jvm/amper-plugin`, `${pluginSettings}`, `${module.rootDir}`, `${taskOutputDir}`, `generated.resources`/`generated.sources`, or `commands:` in a Toolchain context; when designing a build-time processing step that is not natively supported in `module.yaml`. SKIP for general Kotlin Toolchain project work (use the `kotlin-toolchain` skill) and for Gradle-plugin ports (use the `gradle-to-kotlin-toolchain-plugin` skill).
---

# Kotlin Toolchain Plugin Authoring

Kotlin Toolchain projects are configured by declarative YAML. **Local plugins** are the official escape hatch: a small `jvm/amper-plugin` module that ships task actions, settings, and (optionally) generated sources/resources alongside your project. This skill is the practical reference for writing one.

See [references/examples.md](references/examples.md) for ready-to-adapt code patterns. The companion `kotlin-toolchain` skill covers project-level usage; the `gradle-to-kotlin-toolchain-plugin` skill covers porting from Gradle.

## When to write a plugin

Write a plugin when:

- A library's standard workflow needs a **build-time step** (code generation, schema compilation, resource transformation, version stamping). Preserve the library's intended workflow rather than hand-writing what the generator would produce.
- You need **custom verification** that runs as part of the build (pre-release checks, schema validation, contract tests).
- You need to **publish a build-time value** into the JAR classpath or downstream tasks (the closest analog to Gradle's `project.version`).
- You need a **named CLI command** for repeated workflows (`./kotlin do release`, `./kotlin do migrate`, etc.).

Do **not** write a plugin when:

- The behaviour is already supported natively in `module.yaml` (dependencies, JDK provisioning, source layouts, basic packaging).
- You're tempted to reuse a Gradle plugin. Toolchain cannot consume Gradle plugins; reimplement the behaviour as a local plugin instead.

## Plugin module layout

A plugin is just another module. Typical multi-module shape:

```
repo-root/
├── kotlin, kotlin.bat            # wrappers (from `kotlin init`)
├── project.yaml                  # registers the plugin
├── plugins/<name>/
│   ├── module.yaml               # product: jvm/amper-plugin
│   ├── plugin.yaml               # tasks: + commands: + generated:
│   └── src/
│       ├── Settings.kt           # @Configurable interface
│       ├── tasks/                # one @TaskAction per file
│       │   ├── Foo.kt
│       │   └── FooSteps.kt       # internal shared helpers (not @TaskAction)
│       └── <domain logic>/
└── <consumer-module>/
    └── module.yaml               # plugins: { <name>: enabled: true, ... }
```

Always pair the plugin with at least one consumer module in the same repo during development — that's the only way to exercise it end-to-end. Local plugins cannot yet be published to Maven, so consumers live alongside the plugin source.

## `project.yaml` — registering the plugin

The project root file must list both the plugin module and the consumers, and it must point at the plugin source under `plugins:`:

```yaml
modules:
  - consumer-app
  - plugins/<name>

plugins:
  - ./plugins/<name>
```

Without the top-level `plugins:` block the plugin id is unresolvable from any consumer's `module.yaml`.

## `module.yaml` — the plugin module

```yaml
product: jvm/amper-plugin

dependencies:
  - <maven-coordinate>:<version>           # default scope
  - <coordinate>:<version>: runtime-only   # required only at runtime
  - <coordinate>:<version>: compile-only   # not in distribution

pluginInfo:
  id: <plugin-id>                                  # what consumers write under `plugins:`
  settingsClass: <fully.qualified.Settings>        # the @Configurable interface

settings:
  jvm:
    jdk:
      version: 21
  kotlin:
    languageVersion: 2.1
```

Key points:
- `product: jvm/amper-plugin` is what marks this module as a plugin.
- `pluginInfo.id` is the consumer-facing identifier (`plugins: { release: enabled }` in the consumer module uses this id).
- `pluginInfo.settingsClass` is the fully-qualified name of the `@Configurable` interface declared in `src/`.
- Watch for libraries with **optional-at-compile-time, required-at-runtime** transitive dependencies (a recurring trap with JGit + Apache MINA SSHD + BouncyCastle). Add the missing runtime artifact as `: runtime-only` once and document why with a comment.

## `@Configurable interface Settings` — the user-facing DSL

Settings are declared as a Kotlin interface annotated with `@Configurable`. Defaults go in interface property getters; nested blocks become nested `@Configurable` interfaces.

```kotlin
@Configurable
interface Settings {
    val someValue: String get() = "default"
    val checks: ChecksSettings
}

@Configurable
interface ChecksSettings {
    val strict: Boolean get() = true
}
```

Consumers configure them in their `module.yaml`:

```yaml
plugins:
  <plugin-id>:
    enabled: true
    someValue: "override"
    checks:
      strict: false
```

Anything the consumer omits uses the interface default.

## `@TaskAction` — the unit of behaviour

Task actions are top-level `fun`s in the plugin module, annotated with `@TaskAction`. Toolchain calls them when the corresponding entry in `plugin.yaml` is executed.

```kotlin
@TaskAction
fun foo(
    @Input moduleRootDir: Path,
    @Output outputDir: Path,
    settings: Settings,
) {
    // body
}
```

Parameter types and their roles:
- `@Input path: Path` — declared input directory/file. Toolchain snapshots its contents for execution avoidance.
- `@Output path: Path` — declared output directory. Toolchain creates the directory and feeds the path in; consumers depend on it by declaring `@Input` on a matching path.
- `settings: Settings` (or any `@Configurable`) — your plugin's typed configuration. Wired separately in `plugin.yaml` (see below).
- Plain `Path` / primitives — passed in literally from `plugin.yaml`.

### Execution avoidance

By default, Toolchain skips re-running a `@TaskAction` when its declared inputs are unchanged. Some tasks have **hidden inputs** (Git history, network state, environment variables) that Toolchain cannot fingerprint. Disable avoidance for those:

```kotlin
@TaskAction(executionAvoidance = ExecutionAvoidance.Disabled)
fun foo(@Output outputDir: Path, settings: Settings) { /* ... */ }
```

Tasks with **no `@Output`** at all are not cacheable and always re-run on demand — that's the right behaviour for purely side-effecting tasks (releases, deployments, pushes).

## `plugin.yaml` — wiring tasks and commands

```yaml
tasks:
  foo:
    action: !<fully.qualified.foo>
      moduleRootDir: ${module.rootDir}
      outputDir: ${taskOutputDir}
      settings: ${pluginSettings}

  bar:
    action: !<fully.qualified.bar>
      input: ${tasks.foo.action.outputDir}/result.txt
      settings: ${pluginSettings}

generated:
  resources:
    - directory: ${tasks.foo.action.outputDir}

commands:
  - foo
```

### Built-in references

| Reference | What it resolves to |
|---|---|
| `${module.rootDir}` | The directory containing the consumer module's `module.yaml`. Pass in as `@Input` when the task needs to inspect the consumer's source tree. |
| `${taskOutputDir}` | A per-task output directory Toolchain manages. Pass in as `@Output` for tasks that produce files. |
| `${pluginSettings}` | The whole `@Configurable` object built from the consumer's `module.yaml`. Pass in once per task that needs settings. **Forgetting this line silently passes `null` and crashes the action.** |
| `${tasks.<task>.action.<param>}` | A cross-reference to another task's parameter. Use it in `generated.resources`/`generated.sources` and to wire `@Input` of one task to `@Output` of another. |

### `generated.resources` and `generated.sources`

Both blocks register a directory (typically a task's `@Output`) as a contribution to the consumer module's build:

- **`generated.resources`** — the directory is added to the JAR classpath. Files end up reachable via `getResourceAsStream("/path/in/jar")`.
- **`generated.sources`** — the directory is added as a Kotlin source root, compiled along with the consumer's own `src/`. Files must be valid `.kt`.

Both are auto-wired by Toolchain: when the consumer builds, the contributing task runs first.

### Tasks vs commands

- **Tasks** are the implementation; the long-form address is `./kotlin task :<consumer-module>:<task>@<plugin-id>`. The Toolchain docs recommend against relying on this internal mangled name.
- **Commands** are the public API; invoked as `./kotlin do <command-name>`. List user-facing tasks under `commands:`.

Rule of thumb:
- A task whose `@Output` is wired into `generated.resources`/`generated.sources` is a **build-graph contributor** — it runs automatically when something downstream needs it. Do **not** list it under `commands:`.
- A task users explicitly invoke (`./kotlin do release`) **must** be in `commands:`.

List commands at any time with `./kotlin show commands`. Scope to specific modules with `-m <module>`.

## File-based task communication

Toolchain has **no shared mutable build-state object** (no Gradle `project.version`, no extension property maps). Tasks communicate exclusively through file paths matched between `@Output` and `@Input`:

1. Producer task takes an `@Output outputDir: Path`, writes files into it.
2. `plugin.yaml` references those files in another task's `@Input` via `${tasks.<producer>.action.outputDir}/<filename>`.
3. Toolchain auto-infers the build-graph dependency from the path match. No `dependsOn`-style API is needed.

The same `@Output` directory can simultaneously be a build-time channel (other tasks read it via `@Input`) and a runtime channel (registered under `generated.resources`, read via `getResourceAsStream`).

## Runtime overrides — environment variables

Toolchain has no `-Pkey=value` CLI properties. For ephemeral overrides (skip-this-check, force-this-value, override-this-detection), read environment variables inside the `@TaskAction`:

```kotlin
val forced = System.getenv("MYPLUGIN_FORCE_VALUE")?.takeIf { it.isNotBlank() }
val skipChecks = System.getenv("MYPLUGIN_SKIP_CHECKS")?.equals("true", ignoreCase = true) == true
```

Document every recognised env var in the plugin's README so users know which overrides exist. Pass the env map through a constructor parameter rather than calling `System.getenv()` deep in the call stack — that keeps logic unit-testable with a controlled map.

## Sharing logic across task actions

Two or more tasks frequently share steps (verify → create → push). Don't try to compose them via Toolchain's build graph for an *atomic* user-facing task — that introduces windows where another process can observe intermediate state. Instead:

1. Extract each step into an `internal fun` in a sibling file (not annotated with `@TaskAction`).
2. The "individual step" tasks (e.g. `createRelease`) call one helper each.
3. The "combo" task (e.g. `release`) calls them in sequence inside a single `@TaskAction`, sharing one resource (a `GitRepo`, an HTTP client, a transaction handle).

This pattern keeps every shape of usage backed by the same shared logic.

## Limitations to design around

- **Plugins are local-only.** Toolchain does not yet support publishing plugins to Maven; ship plugins in-tree alongside consumers. JetBrains has stated this will change.
- **Plugins are module-level.** There is no project-wide plugin concept. Each consumer module that needs the plugin must list it under its `plugins:` block. Cross-module side effects must flow through files.
- **No `${...}` interpolation in `module.yaml`.** As of Toolchain 0.11, consumer settings must be literal strings/booleans/numbers. You cannot reference `${module.rootDir}` or other module properties from a consumer's `module.yaml`.
- **No `Project.afterEvaluate` / no lazy graph.** Settings are static; compute derived values inside the action body.
- **`-h`/`--help` on the global `kotlin` CLI does not list plugin commands.** Use `./kotlin show commands` instead.

## Common pitfalls

- **Build artefacts in the first commit.** `kotlin build` creates `build/` and `.kotlin/`. Write a `.gitignore` covering `build/`, `.kotlin/`, `.idea/`, `*.iml`, `.DS_Store` **before** the first commit; otherwise the dirty-tree check on any future build will see them as tracked changes.
- **Missing `settings: ${pluginSettings}` line.** Every `@TaskAction` that has a `Settings` parameter needs the corresponding wiring line in `plugin.yaml`. Forgetting it crashes the action with an NPE that doesn't point at the YAML.
- **Tasks always re-run.** Expected when a task has no `@Output` (side-effecting tasks). Surprising when a task does have an `@Output` but writes to a different path each invocation — make sure the `@Output Path` you write to is the same one you received as a parameter.
- **SLF4J floods stdout.** Many JVM libraries (JGit, Logback adapters) use SLF4J. Add `org.slf4j:slf4j-nop:2.0.13` (or another binding) to the plugin's `dependencies:` to suppress the "no provider" banner.
- **JGit + MINA SSHD without BouncyCastle.** Apache MINA SSHD declares BouncyCastle compile-optional but reaches for it at runtime to build its random factory. Add `org.bouncycastle:bcprov-jdk18on` and `bcpkix-jdk18on` as `runtime-only` if your plugin uses JGit's SSH transport.
- **Atomic combo via task chain.** Resist the urge to wire `release` as a no-op task that "depends on" `verifyRelease`, `createRelease`, `pushRelease`. Toolchain doesn't provide a `dependsOn` API for that; even if it did, separate task invocations re-open shared resources. Write the combo as a single `@TaskAction` calling internal helpers.
- **Listing internal tasks under `commands:`.** Build-graph contributors (anything whose `@Output` is in `generated.resources`/`generated.sources`) should stay out of `commands:`. They run automatically; exposing them as commands invites users to run them by hand and confuses the public API.

## Validation pattern

A plugin under development without a consumer module to point at is hard to evaluate. Always:

1. Add a small consumer module (`demo-app/`, `sample/`) that enables the plugin with realistic settings.
2. Make the consumer's `main.kt` (or test) read whatever the plugin publishes.
3. Document a "Validation walkthrough" in the plugin's README that someone cloning the repo can paste verbatim and observe the documented outputs.

This serves as both an integration test harness during development and a getting-started for future consumers.

## Additional resources

- For concrete code patterns (Settings interface, `@TaskAction` signatures, `plugin.yaml` wiring, file-based publication, atomic combo task, env-var override), see [references/examples.md](references/examples.md).
- For general Kotlin Toolchain project usage (building, testing, dependencies, multiplatform), use the companion `kotlin-toolchain` skill.
- For porting an existing Gradle plugin to a Toolchain plugin, use the companion `gradle-to-kotlin-toolchain-plugin` skill.
- Toolchain plugin docs: <https://kotlin-toolchain.org/dev/user-guide/plugins/>
