---
name: gradle-to-kotlin-toolchain-project
description: Migrate an entire Gradle Kotlin/JVM project to Kotlin Toolchain — module.yaml + project.yaml, dependency catalog reuse, CI rewrite, and the set of local plugins that replace Gradle plugins with no native Toolchain equivalent (axion-release, jib, detekt, ktlint, the `application` plugin, custom `processResources` tasks). TRIGGER when the user asks to convert, migrate, port, or move a Gradle project (build.gradle / build.gradle.kts at the repo root) to Kotlin Toolchain or Amper; when the repo currently has Gradle wrapper + build script and the user wants to switch the whole build, not just port one plugin; when the conversation references replacing Gradle's `application` plugin, axion-release, jib, detekt, ktlint with Toolchain equivalents at the project level; when adapting an existing CI pipeline (`./gradlew build`, `./gradlew check`, `./gradlew release`) to `./kotlin` commands. SKIP for porting a single Gradle plugin in isolation (use the `gradle-to-kotlin-toolchain-plugin` skill), for authoring a new Toolchain plugin from scratch (use `kotlin-toolchain-plugin-authoring`), and for general Kotlin Toolchain project work where Gradle is already gone (use `kotlin-toolchain`).
---

# Gradle → Kotlin Toolchain Project Migration

Migrating a whole Gradle Kotlin/JVM project to Kotlin Toolchain is two distinct exercises stapled together: a mostly-mechanical **translation** of dependencies and configuration, plus a non-trivial set of **plugin replacements** for everything Gradle did via plugins that Toolchain has no native answer for. This skill is the project-level playbook; it leans on the companion skills for the plugin pieces.

Read [references/examples.md](references/examples.md) for ready-to-adapt code: a Ktor-app `module.yaml`, multi-plugin `project.yaml`, the smallest hand-written plugin (a `package` plugin producing a stable `build/libs/` artifact), and CI workflow templates.

The companion skills handle the plugin-shaped subproblems:

- **`kotlin-toolchain`** — Toolchain syntax reference (module.yaml shape, layout, catalogs). Cite from here rather than re-explaining.
- **`gradle-to-kotlin-toolchain-plugin`** — the per-plugin port workflow (concept-mapping table, MVP scoping, validation pattern). You will end up running this workflow several times (once per Gradle plugin without a native).
- **`kotlin-toolchain-plugin-authoring`** — for plugins you write from scratch (no Gradle source to port), like a `package` plugin to stabilise CI artifact paths.

## Migration philosophy

1. **Preserve source code wherever possible.** If the migration is forcing edits to business logic, that's a strong signal a local plugin can fill the gap instead. Application code that reads `release.properties` from the classpath, for instance, should keep working — write a plugin that publishes that exact file, don't rewrite the consumer to read a different format.
2. **Treat every Gradle plugin as either "drop", "use Toolchain native", or "reimplement as a local plugin"** — never "use Gradle plugin". Toolchain cannot consume Gradle plugins; do not try.
3. **The version catalog is the migration's pivot.** `gradle/libs.versions.toml` is byte-identical before and after; Toolchain reads it as the built-in `$libs.*` catalog. Every Maven coordinate in any plugin YAML or module YAML must end up as a `$libs.*` reference — that's the single source of truth for Dependabot post-migration.
4. **Land it as a single commit if possible.** A whole-project migration churns enough files that splitting commits hurts review (and the in-between commits don't build). Aim for one well-titled commit (`build: migrate to Kotlin Toolchain`) with a comprehensive PR description.

## Migration workflow

Five phases in order. Skipping investigation routinely produces migrations that miss subtle behavioural mismatches (notably around linting and resource generation).

### Phase 1 — Inventory the Gradle build

Before any YAML is written, produce a written inventory of the existing build:

- **All plugins** in `build.gradle(.kts)`'s `plugins { }` block. Categorise each as:
  - **Drop** — Gradle-only behaviour not needed under Toolchain (e.g. the wrapper-generation plugin).
  - **Native** — covered by `module.yaml` directly (`org.jetbrains.kotlin.jvm`, `org.jetbrains.kotlin.plugin.serialization`, the `application` plugin's `mainClass`, JDK toolchains, BOM imports, scope qualifiers).
  - **Local plugin** — needs reimplementation (typical: `axion-release`, `jib`, `detekt`, `ktlint`, anything that wires custom tasks into `check`).
- **All custom tasks** registered in the build script (`tasks.register("...")`, `tasks.named("...")`). Note their inputs, outputs, and what wires them into the build graph (`processResources.dependsOn(...)`, `check.dependsOn(...)`). Each one is a future `@TaskAction` in some plugin.
- **All source-code dependencies on build-generated artefacts.** Grep `src/` for resource names that are produced by custom tasks. (Canonical example: `MetaReleaseService` reading `release.properties` from the classpath, where the file is written by a custom `generateReleaseProperties` Gradle task.) Each of these is a constraint the migration must honour without source changes.
- **`gradle/libs.versions.toml`** — note which `[versions]` and `[plugins]` entries are *only* used by Gradle plugins (those become dead weight to remove later).
- **CI workflows** — every `./gradlew <task>` invocation must map to a `./kotlin` command or be replaced. Note artefact upload paths (typically `build/libs/`), version-extraction shell pipelines, and any `-P` property flags.

Save this inventory as a markdown plan (e.g. `MIGRATION_PLAN.md`); it becomes the checklist the PR description verifies.

### Phase 2 — Decide layout, scope, and the plugin set

These three decisions shape everything that follows:

#### Layout: `maven-like`

Gradle projects use Maven layout (`src/main/kotlin`, `src/main/resources`, `src/test/kotlin`, `src/test/resources`). Toolchain's default layout is `src`/`test`/`resources`/`testResources`. To avoid moving every source file, set `layout: maven-like` in `module.yaml`. This is fully supported for `jvm/app` and `jvm/lib` products and is the migration-friendly choice.

#### Plugin set

For a typical Gradle Kotlin/JVM service, the plugin set you'll end up with is:

| Gradle plugin / feature | Replacement | Notes |
|---|---|---|
| `org.jetbrains.kotlin.jvm`, `kotlin.plugin.serialization` | Native (`settings.jvm.jdk`, `settings.kotlin.serialization: json`) | Use `$libs.*` for Kotlin libs if you want pin control; otherwise the toolchain default applies. |
| `application` plugin (mainClass + `build/libs/<name>.jar`) | `settings.jvm.mainClass` covers the entry point. The JAR's *location* needs a small `package` local plugin so CI has a stable upload path — see [references/examples.md](references/examples.md). |
| `pl.allegro.tech.build.axion-release` (or similar git-tag versioning) | A `release` local plugin (JGit-based). Vendor one if available; otherwise see the `gradle-to-kotlin-toolchain-plugin` skill. Publishes the version as a file (e.g. `META-INF/release/version.txt`) under `generated.resources`. |
| `com.google.cloud.tools.jib` | A `jib` local plugin wrapping `jib-core` directly. The community sample at the spring-petclinic Amper repo is a good starting point; you will likely need to extend `ContainerSettings` with `ports`, `environment`, `user` (the sample omits these). |
| `io.gitlab.arturbosch.detekt` | A `detekt` local plugin that subprocess-launches `detekt-cli`. Vendor from `amper/build-sources/detekt/` and **change the type-resolution default** — see "Mismatches to watch" below. |
| `org.jlleitschuh.gradle.ktlint` | A `ktlint` local plugin, hand-written along the same shape as detekt (subprocess-launching `ktlint-cli`). |
| Custom `generateXyz.properties` tasks | An additional `@TaskAction` on the relevant plugin (usually the release plugin). Output dir wired into `generated.resources`. |

Each plugin is its own `jvm/amper-plugin` module under `plugins/<name>/`. Wire them all from `project.yaml`:

```yaml
modules:
  - plugins/<name1>
  - plugins/<name2>
  ...
plugins:
  - ./plugins/<name1>
  - ./plugins/<name2>
  ...
```

#### Scope tiers

For each candidate local plugin, decide between **vendor** (copy a known-working source from a public repo), **author** (write from scratch following `kotlin-toolchain-plugin-authoring`), or **extend** (vendor and add fields).

Default scope for a single migration PR:
- Vendor the release, jib, detekt plugins from known-good sources.
- Author the ktlint and `package` plugins.
- Defer anything else (e.g. publishing to Maven Central) to a follow-up.

### Phase 3 — Implement

Recommended order, top-down so each step is testable:

1. **Bring in the wrapper.** `cp` `kotlin` and `kotlin.bat` from any reference Toolchain project (or run `kotlin init` in a scratch dir). Pin the version in `.sdkmanrc` (`kotlintoolchain=<version>`) so it matches the wrapper.
2. **Write `project.yaml`** registering all the plugin module paths (alphabetical order is conventional).
3. **Vendor / author each plugin** under `plugins/<name>/`. Run `./kotlin show modules` after adding each to verify the project model still loads.
4. **Write the root `module.yaml`.** `product: jvm/app`, `layout: maven-like`, dependencies as `$libs.*` references, `plugins:` block enabling each local plugin with the minimum non-default settings. Repeat `./kotlin show modules`.
5. **Validate each plugin in isolation** via `./kotlin task :<module>:<task>@<plugin>` or `./kotlin do <command>`. Fix model errors as they surface.
6. **Rewrite CI workflows** (see Phase 4).
7. **Delete Gradle files** — `build.gradle.kts`, `gradlew`, `gradlew.bat`, `gradle/wrapper/` — but **keep `gradle/libs.versions.toml`**; it's still the catalog.
8. **Sweep `gradle/libs.versions.toml`** for now-unused entries — every former `[plugins]` block entry and any `[versions]` keys that only fed those plugins.
9. **Sweep plugin YAMLs for hardcoded coordinates.** Every literal Maven coordinate in `module.yaml` or `plugins/*/plugin.yaml` should be a `$libs.<key>` reference to the catalog. This is the rule the migration must end on.

### Phase 4 — Rewrite CI

The migration mostly *simplifies* CI workflows:

- **Drop `actions/setup-java`.** The `./kotlin` wrapper auto-provisions its own JDK on first run. Set `KOTLIN_CLI_NO_WELCOME_BANNER: "1"` at the job level to keep logs clean.
- **`./gradlew build && ./gradlew check`** → `./kotlin build && ./kotlin check`. `kotlin check` runs every plugin's `checks:` registrations (detekt + ktlint + tests).
- **`./gradlew release`** → `./kotlin do release` (the release plugin exposes it as a command).
- **`./gradlew currentVersion -q | …`** → `./kotlin do currentVersion | tail -n1` (or similar — depends on how the local release plugin formats output).
- **`./gradlew jib -Djib.to.tags=…`** → `./kotlin do jib`. Dynamic tags can be passed via env vars consumed inside the plugin (Toolchain has no `-P`/`-D` equivalent — see the mismatches section).
- **Artefact upload path** must change. Gradle's `build/libs/<name>.jar` is gone; the Toolchain's `jarJvm` task writes to a Toolchain-internal path. **Author a `package` local plugin** that stages the JAR at `${module.rootDir}/build/libs/${module.name}.jar` so CI's `actions/upload-artifact` line stays simple. See [references/examples.md](references/examples.md) for the 30-line plugin.

### Phase 5 — Validate end-to-end

Run each user-facing command at least once locally. Capture the outputs in the PR description's Test plan:

```sh
./kotlin show modules            # project model loads, all expected modules listed
./kotlin clean && ./kotlin build # main build path is clean
./kotlin test                    # if local environment supports it (see mongo:3.2 caveat below)
./kotlin check                   # detekt + ktlint + tests; expect zero violations after Phase 3
./kotlin do currentVersion       # release plugin reachable
./kotlin do jib                  # or jibBuildTar to avoid pushing during local validation
./kotlin do package              # if a package plugin exists; verify build/libs/<name>.jar exists with the right Main-Class manifest
./kotlin do ktlintFormat         # round-trips cleanly
```

Document each command's expected output in the PR description; reviewers will use it to compare against the CI run.

## Mismatches to watch

These trip every project-level migration. Address each explicitly rather than waiting for surprise behaviour after merge.

### `module.yaml` does not support `${...}` interpolation

A documented Toolchain limitation that comes up immediately in consumer config. Setting `configFile: ${module.rootDir}/detekt.yml` in `module.yaml` *does not interpolate* — the string is taken literally. Use a path relative to the module root instead (`configFile: detekt.yml`). `${...}` interpolation only works inside `plugin.yaml` (where it's resolved by the plugin SDK, not the YAML parser).

### Module name comes from the directory name

There is no `name:` field in `module.yaml`. The module name is the basename of the directory containing the file. In CI this affects task output paths: `actions/checkout` clones the repo at a directory whose basename is the GitHub repo name, so `${module.name}` resolves to e.g. `sdkman-broker-2` in upstream CI but `17ef` in a worktree. Any path you encode in plugin task outputs should use `${module.name}` so it follows the directory.

### Plugin settings need `enabled: true` *or* the shorthand `<plugin>: enabled`

Two valid forms for enabling a plugin in `module.yaml`:

```yaml
plugins:
  release: enabled         # shorthand — only valid when no other settings are set
  jib:                     # long form — required when any setting is configured
    enabled: true
    container:
      mainClass: com.example.App
```

Providing settings *without* `enabled: true` silently does **not** enable the plugin — Toolchain warns ("Plugin X is not enabled, but has some explicit configuration") and skips it. Always enable explicitly when any setting is present.

### Detekt type resolution defaults differ from Gradle

The vendored upstream detekt plugin (`amper/build-sources/detekt/`) passes `--classpath` to detekt-cli unconditionally, enabling **type resolution**. Gradle's default `detekt` task does **not** enable type resolution. The result: a migration that uses the vendored plugin as-is will surface 10–100 "new" detekt violations that the Gradle build silently ignored (notably from the `UnreachableCode` rule on elvis-with-return patterns).

**Fix**: when vendoring the detekt plugin, add a `useTypeResolution: Boolean` setting to its `@Configurable Settings` interface with a `get() = false` default. Gate the `--classpath` flag on this setting. This matches Gradle parity by default; consumers who want the stronger checks opt in. See [references/examples.md](references/examples.md) for the full patch.

### No dependency exclusions

`Amper has no equivalent of Gradle's `exclude(group, module)` for transitive dependency exclusions. If the source build excluded e.g. `kotlinx-coroutines-core` from an Exposed dependency, that exclusion silently disappears under Toolchain — the library lands on the runtime classpath. Document the trade-off in the PR (usually a few hundred KB of unused classes; rarely a behavioural issue) and move on.

### No plugin-to-plugin dependencies

Plugins are isolated; one local plugin cannot declare a dependency on another local plugin in the same project. If two plugins logically belong together (e.g. a release plugin that writes the version, and the same plugin needs to publish that version in two formats), put both task actions in the same plugin module rather than splitting across two plugins.

### No `-P` / `-D` CLI overrides

Toolchain has no Gradle-equivalent `-Pkey=value` or `-Dkey=value` for overriding plugin settings from the CLI. For ephemeral overrides (force-version, skip-checks), read environment variables inside the `@TaskAction`. See the `gradle-to-kotlin-toolchain-plugin` skill's "Architectural mismatches" section for the pattern.

### Test resource conflicts with `generated.resources`

When a local plugin emits a file via `generated.resources` *and* the project has a test fixture at the same classpath path (e.g. plugin writes `release.properties` to main; `src/test/resources/release.properties` overrides with a known-test-version value), the JVM's classpath ordering puts test resources first — the fixture wins during tests. This usually works correctly but is brittle. If a test asserts on a specific value, prefer injecting a stub service over relying on classpath precedence.

### Dependabot needs a stub `build.gradle.kts`

Dependabot's gradle-ecosystem file fetcher requires a `build.gradle` or `build.gradle.kts` in the configured directory before it will scan `gradle/libs.versions.toml` for updates. Without it, `package-ecosystem: "gradle"` silently no-ops. Keep an **empty** `build.gradle.kts` at the project root with a comment explaining why:

```kotlin
// Intentionally empty. This project is built with the Kotlin Toolchain;
// `gradle/libs.versions.toml` is the dependency catalog (consumed natively).
// This stub exists so GitHub Dependabot's `package-ecosystem: gradle` detector
// discovers the project. See `.github/dependabot.yml`.
```

The Toolchain ignores this file entirely.

### mongo:3.2 and other amd64-only images won't run on Apple Silicon

Older images pinned by testcontainers (`mongo:3.2` is the canonical example) are amd64-only and crash immediately under Rosetta/QEMU on M-series Macs (`runtime: failed to create new OS thread (have 2 already; errno=22)`). This is **unchanged by the migration** — the same image fails identically under `./gradlew check`. Flag it in the PR's Test plan as a pre-existing host incompatibility, not a regression. CI on `ubuntu-latest` (amd64) is unaffected.

## What goes in the PR

A whole-project migration PR is large; a clear description shortens review by hours. Sections to include:

1. **Summary** — one paragraph stating "replaces Gradle build with Kotlin Toolchain", followed by the plugin layout (n local plugins, what each replaces).
2. **What's preserved** — the catalog, `detekt.yml`, business-logic source files. Reviewers want to confirm the migration isn't a covert rewrite.
3. **What replaces what** — a 1:1 mapping table of `./gradlew <task>` → `./kotlin <command>`. Also include the build-script-to-yaml summary.
4. **Test plan** — every command you ran in Phase 5, with expected output. Mark items that only CI can exercise (testcontainers, full check pipeline) so reviewers know where local validation ends.
5. **Notes / follow-ups** — items deliberately out of this PR (often: bumping deprecated container images, grouping more Dependabot entries, deferring publish-to-Maven of plugins). Keep this PR focused; chase follow-ups separately.

## Common pitfalls

- **Pushing without the `package` plugin.** A whole-project migration that forgets a stable artifact path leaves CI uploading `build/artifacts/CompiledJvmArtifact/` (Toolchain's internal class-file tree) instead of a JAR. Always add a `package` plugin before swapping the artifact upload path.
- **`./kotlin check` hangs locally on testcontainers.** `kotlin check` includes `kotlin test`, which spins up testcontainers, which may not work on the developer's host. For local validation, prefer running individual plugin tasks (`./kotlin task :<module>:check@ktlint`, `./kotlin task :<module>:analyze@detekt`) plus `./kotlin build` rather than the full `check`.
- **Forgetting `enabled: true`.** If a plugin shows up under `./kotlin show modules` but its task never runs, check that the consumer's `module.yaml` has `enabled: true` (not just plugin settings — settings without `enabled` are silently ignored).
- **Committing `build/`.** `kotlin build` creates `build/`. Add it to `.gitignore` early (drop the old `.gradle/` entry, add `build/` if not already there).
- **Cycle detected on `writeVersion`.** If the release plugin is enabled on the *root* module (where `${module.rootDir}` is the project root that contains `build/`), Toolchain's dependency inference may treat all build outputs as inputs of any task that takes `@Input moduleRootDir: Path`. Patch the release plugin's `moduleRootDir` parameters with `@Input(inferTaskDependency = false)`.
- **Hardcoded `1.23.8` everywhere.** Detekt-cli, ktlint-cli, jib-core, and JGit versions get hardcoded into plugin YAMLs during vendoring. Sweep them into `gradle/libs.versions.toml` and use `$libs.*` references; otherwise Dependabot can't track them.
- **String interpolation into Maven coordinates.** `plugin.yaml` does not allow `${pluginSettings.version}` *inside* a Maven coordinate string (`com.example:foo:${...}` fails with "Value of type 'ShadowDependency' doesn't support string interpolation"). Either hardcode the version in `plugin.yaml` or use a catalog reference (`$libs.foo`).

## Additional resources

- For concrete code patterns (root `module.yaml`, multi-plugin `project.yaml`, a minimal `package` plugin, CI workflow templates, the `useTypeResolution` patch), see [references/examples.md](references/examples.md).
- For per-plugin port specifics (the concept-mapping table, MVP scoping, file-based publication), use the companion `gradle-to-kotlin-toolchain-plugin` skill.
- For writing a plugin from scratch (the `package` plugin or any other), use `kotlin-toolchain-plugin-authoring`.
- For Toolchain syntax reference, use `kotlin-toolchain`.
- Toolchain docs: <https://kotlin-toolchain.org/dev/>
