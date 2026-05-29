---
name: gradle-to-kotlin-toolchain-plugin
description: Port a Gradle plugin to a Kotlin Toolchain local plugin. TRIGGER when the user asks to convert, port, migrate, or reimplement a Gradle plugin as a Kotlin Toolchain plugin or when during Gradle to Kotlin Toolchain project conversion AI agent realized that some functionality from Gradle setup isn't natively supported in Kotlin Toolchain; when designing a Kotlin Toolchain local plugin that replicates Gradle plugin behaviour; or when mapping Gradle plugin concepts (Tasks, Extensions, `project.version`, `dependsOn`, `-P` properties, hooks, `Project.afterEvaluate`) to their Kotlin Toolchain analogs (`@TaskAction`, `@Configurable Settings`, file-based `@Output`/`@Input` wiring, env vars, `commands:` registry). SKIP for general Kotlin Toolchain project work (use the `kotlin-toolchain` skill) and for Gradle plugin development that has no Kotlin Toolchain target.
---

# Gradle → Kotlin Toolchain Plugin Conversion

Converting a Gradle plugin to a Kotlin Toolchain local plugin is mostly a *mapping* exercise — the build engine concepts overlap, but several Gradle features have no direct analog and need a redesign rather than a port. This skill captures the conversion workflow and the concept mappings that come up every time.

Read [references/examples.md](references/examples.md) for ready-to-adapt code patterns (Settings interface, `@TaskAction` signatures, `plugin.yaml` shapes, file-based publication).

## Conversion workflow

Follow these phases in order. Skipping investigation routinely produces ports that miss philosophical mismatches.

### Phase 1 — Investigate the source plugin

Before writing a single line of Kotlin, catalogue what the Gradle plugin actually does. Read its docs (`docs/`, `README.md`) and produce a list of:

- **Tasks** — name, purpose, inputs/outputs, what it depends on, whether it is side-effecting.
- **DSL surface** — every option in the `myPlugin { ... }` extension and any nested blocks. Note types, defaults, and which are closures (per-branch / per-context).
- **Pipelines** — multi-phase logic (e.g. axion's 6-phase version inference). Number the phases; each phase is a future helper function.
- **Checks / validations** — pre-action gates and their override flags.
- **Hooks** — `pre`/`post`/`fileUpdate`/`commit`/`push` and the context they receive.
- **CLI overrides** — every `-Pfoo.bar` flag.
- **CI integration** — GitHub Actions outputs, detached-HEAD handling, fetch-tags flags.

Save this as a written plan (markdown file). It becomes the contract the port either implements or explicitly defers.

### Phase 2 — Lock the scope (MVP-first)

Always offer the user three scope tiers — MVP, MVP + key extras, full parity — and lock one before drafting the plan. Several Gradle features have no clean Kotlin Toolchain analog yet; bundling them into MVP triples the work and forces design compromises early. Capture deferred features under a "What's not in this MVP" section in the README so users understand the trade.

Ask the user explicitly:

1. Which scope tier? (MVP / MVP + extras / Full parity.)
2. Repo layout? (Plugin-only / plugin + demo module / plugin self-hosting.) A demo module is strongly recommended for MVPs — see the validation phase below.

### Phase 3 — Scaffold (optional, only if directory is empty)

Use `kotlin init` to grab the wrapper scripts (`kotlin`, `kotlin.bat`) from any template (`multiplatform-cli` works), then copy them into the target repo. Hand-write `project.yaml`, `plugins/<name>/module.yaml`, and the demo module. See the "Standard layout" section below.

### Phase 4 — Implement against the concept mapping

Work through the mapping table in this skill (and [references/examples.md](references/examples.md)). Implement one task at a time, building bottom-up: data classes → Git / IO wrappers → pipeline → checks → task actions → `plugin.yaml` wiring.

### Phase 5 — Validate end-to-end

Run the plugin against the demo module with a real-world scenario derived from the source plugin's docs. For a release plugin: `git init`, `currentVersion`, `createRelease`, verify the tag, commit, re-check the version, exercise the force-version override. Document each step in the README so consumers can reproduce.

## Concept-mapping table

This is the core reference. When a Gradle concept appears in the source plugin, look it up here.

| Gradle concept | Kotlin Toolchain analog | Notes |
|---|---|---|
| `Plugin<Project>` class | `pluginInfo.id` + `settingsClass` in `module.yaml`, `product: jvm/amper-plugin` | One module per plugin; no separate `apply()` method. |
| `Task` subclass with `@TaskAction` method | Top-level `fun foo(...)` annotated `@TaskAction` in plugin module's `src/` | Tasks live one-per-file in `src/tasks/`. Signature is `fun(params): Unit`. |
| `Project.extensions.create("foo", FooExtension::class)` | `@Configurable interface Settings` with property defaults via `get() = ...` | Defaults go in interface getters. Nested blocks → nested `@Configurable` interface property. |
| `task.dependsOn(otherTask)` | `@Input` parameter on one task matching `@Output` of another | Toolchain auto-infers the DAG from path matching — no explicit `dependsOn`. |
| `project.version = scmVersion.version` | A task that writes a file (`version.txt`) marked `@Output`; downstream tasks declare `@Input` on the same path | There is no project-wide shared `version` property in Toolchain. Use the filesystem as the publication channel. |
| `-Prelease.forceVersion=X` (project property) | Environment variable (`RELEASE_FORCE_VERSION=X`) read via `System.getenv()` | Toolchain has no `-P` equivalent yet. Env vars are the documented escape hatch. |
| Application reading the version at runtime | Plugin marks the `@Output` dir as `generated.resources`; app reads it via `getResourceAsStream("/META-INF/.../foo.txt")` | Same file feeds build-time consumers (via `@Input`) and runtime consumers (via classpath). |
| Generated Kotlin source code | `generated.sources` block pointing at a `@TaskAction`'s `@Output` directory | Useful when consumers want compile-time constants. Prefer resources over generated source if the value is only read at runtime. |
| Public task name users invoke | Entry in `commands:` block of `plugin.yaml`; invoked as `./kotlin do <name>` | Tasks are an implementation detail; commands are the public API. Default to wiring every user-facing task as a command and leave build-graph contributors out of `commands:`. |
| `Project.afterEvaluate { ... }` | No analog — settings are static. Compute derived values inside the `@TaskAction`. | Use env vars + settings only. |
| Lazy `Provider<T>` / `Property<T>` | Plain Kotlin values in the action body | Toolchain re-runs the action on demand; no lazy graph. |
| Groovy/Kotlin DSL hooks (`pre { }`, `fileUpdate { }`, `commit { }`) | New `@TaskAction` functions added to the plugin (or to a fork) | Hooks become first-class tasks. There is no closure-based extension point. |
| `dependencies { implementation(...) }` in plugin's `build.gradle` | `dependencies:` in `plugins/<name>/module.yaml` | Same Maven coordinates; supports `: exported`, `: runtime-only`, `: compile-only` suffix scopes. |
| `Configuration` with custom resolution | Not supported; each task pulls what it needs through the plugin module's dependency list | Plugins cannot define new dependency configurations. |
| Custom Gradle task types in `buildSrc` | Standalone `jvm/amper-plugin` module under `plugins/<name>/` | Toolchain plugins are local-only today; no Maven publishing yet. |
| `gradle.taskGraph.whenReady` / mutation API | No analog. | Build graph isn't introspectable from inside an action. |
| `OutputDirectory` / `OutputFile` annotations | `@Output` on a `Path` parameter | Toolchain auto-creates the directory and feeds the path in. |
| `InputDirectory` / `InputFile` / `InputFiles` | `@Input` on a `Path` parameter | Auto-snapshotted for execution avoidance. |
| `Task.outputs.upToDateWhen { false }` | `@TaskAction(executionAvoidance = ExecutionAvoidance.Disabled)` | Use for tasks whose real inputs (Git state, network, env) can't be fingerprinted. |
| `quiet { }` task logging modes | `println(...)` from inside the action; Toolchain captures stdout | No structured logger replacement in MVP. |

## Architectural mismatches that need real design

Three Gradle features have no direct Toolchain analog and need conscious redesign every time:

### 1. No project-wide shared state (no `project.version`)

Gradle plugins routinely set `project.version` and rely on downstream plugins (`maven-publish`, `application`) reading it. Toolchain plugins are module-level and have no shared mutable project object.

**Fix**: turn the value into a file. One `@TaskAction` writes (e.g.) `version.txt` to its `@Output` directory; downstream consumers either declare `@Input` on the same path (build-time) or read it via the classpath after the directory is registered under `generated.resources` (runtime). See [references/examples.md](references/examples.md) for the full `writeVersion` pattern.

### 2. No `-P` properties

Gradle's `-Pfoo.bar=baz` lets users override any DSL value from the CLI. Toolchain has no equivalent.

**Fix**: read overrides from `System.getenv()` inside the `@TaskAction`. Establish a naming convention (`<PLUGINID>_<UPPERCASE>`) and document the mapping in the README:

```
-Prelease.forceVersion=X         → RELEASE_FORCE_VERSION=X
-Prelease.disableChecks          → RELEASE_DISABLE_CHECKS=true
```

Static settings still go through `module.yaml` under the `plugins:` block. Env vars are exclusively for ephemeral overrides.

### 3. No Groovy hooks / closures

Gradle plugins like `axion-release` expose `hooks { pre { ... } }` blocks where users supply arbitrary Groovy code. Toolchain has no equivalent — settings are static YAML.

**Fix**: every named built-in hook (`fileUpdate`, `commit`, `push`) becomes its own `@TaskAction` shipped with the plugin. Arbitrary user closures aren't portable; document them as "out of MVP" and direct users to fork the plugin if they need custom logic.

## Standard plugin layout

This layout is the convention used by the Kotlin Toolchain SDK examples and works for any single-plugin repo:

```
repo-root/
├── kotlin, kotlin.bat            # wrappers (copy from `kotlin init` output)
├── .gitignore                    # MUST exclude build/ and .kotlin/ before any git ops
├── project.yaml                  # registers the plugin
├── README.md                     # mapping table + validation walkthrough
├── plugins/<name>/
│   ├── module.yaml               # product: jvm/amper-plugin + deps + pluginInfo
│   ├── plugin.yaml               # tasks: + commands: + generated:
│   └── src/
│       ├── Settings.kt           # @Configurable interface
│       ├── tasks/                # one @TaskAction per file
│       │   ├── Foo.kt
│       │   └── FooSteps.kt       # internal shared helpers (not @TaskAction)
│       └── <domain logic>/
└── demo-app/                     # exercises the plugin end-to-end
    ├── module.yaml               # plugins: { <name>: enabled: true, ... }
    └── src/main.kt               # consumes the plugin's output
```

Files to create early, before any `@TaskAction`:

- **`project.yaml`** at repo root listing modules and the plugin path:

  ```yaml
  modules:
    - demo-app
    - plugins/<name>
  plugins:
    - ./plugins/<name>
  ```

- **`.gitignore`** with `build/`, `.kotlin/`, `.idea/`, `*.iml`, `.DS_Store`. Do this **before** the first commit — once the build directory is in git history, removing it requires a follow-up commit (and risks dragging the dirty-tree pre-release check into your validation runs).

- **`plugins/<name>/module.yaml`** with `product: jvm/amper-plugin` and `pluginInfo: { id, settingsClass }`.

## Common pitfalls

Drawn from real conversion sessions.

- **Dirty `build/`.** The first build creates `build/`. If you `git init` *after* the first build, the initial commit will sweep `build/` in. Always write `.gitignore` and commit it first, or `git rm -rf --cached build` immediately.
- **Pre-release checks trip on your own edits.** When validating a release plugin (or any plugin that gates on a clean working tree), your in-progress edits will make the check fail. Use the plugin's documented bypass env var (e.g. `RELEASE_DISABLE_UNCOMMITTED_CHECK=true`) instead of committing speculative changes.
- **JGit + Apache MINA SSHD needs BouncyCastle at runtime.** MINA SSHD declares BouncyCastle as compile-optional but builds a random factory from it at runtime. Without BC on the classpath every SSH push dies with `NoClassDefFoundError: org/bouncycastle/crypto/prng/RandomGenerator`. Add `bcprov-jdk18on` and `bcpkix-jdk18on` as `runtime-only` deps, pinned to the version contemporary with the MINA SSHD that JGit pulls in.
- **SLF4J warnings flood stdout.** JGit uses SLF4J. Add `org.slf4j:slf4j-nop:2.0.13` (or your preferred binding) to suppress the "no SLF4J providers were found" banner that otherwise prefixes every task run.
- **`pluginSettings` wiring is per-action.** Every `@TaskAction` that takes a `settings: Settings` parameter needs a corresponding `settings: ${pluginSettings}` line in its `plugin.yaml` action block. Forgetting one line silently passes `null` and crashes the action.
- **`module.yaml` does not support `${...}` interpolation.** This is documented but easy to forget. Plugin settings must be literal strings/booleans/numbers; you cannot reference module rootDir or other module properties from `module.yaml`.
- **Tasks without `@Output` always re-run.** This is the right behaviour for side-effecting tasks (`release`, `createRelease`). It is *wrong* for pure-output tasks; declare an `@Output Path` parameter so Toolchain can cache the result.
- **`@TaskAction(executionAvoidance = ExecutionAvoidance.Disabled)` for hidden-input tasks.** Any task whose real input is Git history, the network, or env vars must disable execution avoidance — Toolchain has no way to fingerprint these.
- **Atomic tasks via single `@TaskAction`, not task chains.** Gradle composes `release` from `verifyRelease → createRelease → pushRelease` via the task graph. In Toolchain, write `release` as one `@TaskAction` that calls internal helpers from the other tasks' files. Sharing one `GitRepo` (or other resource) across all three steps eliminates the window where another process can observe intermediate state.
- **Commands vs tasks.** Tasks are internal (`./kotlin task :module:name@plugin`); commands are public (`./kotlin do name`). List user-facing tasks under `commands:` in `plugin.yaml`. Build-graph contributors (e.g. a `writeVersion` task whose output is wired into `generated.resources`) should *not* be in `commands:` — they run automatically when something downstream needs them.

## Validation pattern

Always end the conversion with an end-to-end smoke test driven from the demo module, using the same scenarios documented in the source plugin. Capture the exact commands in the README under a "Validation walkthrough" section. Sample (from a release plugin):

```sh
git init -b main && git commit --allow-empty -m initial
./kotlin do currentVersion       # => 0.1.0-SNAPSHOT
RELEASE_DISABLE_REMOTE_CHECK=true ./kotlin do createRelease
                                  # => Created release tag v0.1.0
./kotlin do currentVersion       # => 0.1.0
git commit --allow-empty -m next
./kotlin do currentVersion       # => 0.1.1-SNAPSHOT
RELEASE_FORCE_VERSION=2.0.0 ./kotlin do currentVersion
                                  # => 2.0.0
```

This serves three purposes at once: it proves the port works, it documents the mapping from Gradle CLI flags to env vars, and it gives consumers a copy-pasteable getting-started.

## What to put in the README

Every conversion's README should have:

1. One-line description naming the source plugin and stating this is a port.
2. Quick-start with `./kotlin do <command>` examples.
3. `module.yaml` configuration block with all settings and their defaults.
4. **Runtime overrides table** mapping Gradle `-P` flags to env vars.
5. **What's not in MVP** section listing deferred features.
6. **Toolchain limitations worth knowing** — plugins are local-only, module-level, no `${...}` in `module.yaml`. This sets correct expectations.

## Additional resources

- For concrete code patterns lifted from a real conversion (the `axion-release` Gradle plugin ported to a Toolchain local plugin), see [references/examples.md](references/examples.md).
- For general Kotlin Toolchain usage (building, testing, dependencies, multiplatform), use the companion `kotlin-toolchain` skill.
- Toolchain plugin docs: <https://kotlin-toolchain.org/dev/user-guide/plugins/>
