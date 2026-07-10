---
name: gradle-to-kotlin-toolchain-project
description: Migrate an entire Gradle Kotlin/JVM project to Kotlin Toolchain — module.yaml + project.yaml, dependency-catalog reuse, CI rewrite, and the local plugins that replace Gradle plugins with no native Toolchain equivalent. TRIGGER when the user asks to convert, migrate, port, or move a Gradle project (build.gradle / build.gradle.kts at the repo root) to Kotlin Toolchain or Amper; when the repo currently has a Gradle wrapper + build script and the user wants to switch the whole build, not just port one plugin; when the conversation references replacing Gradle plugins (the `application` plugin, git-tag versioning such as axion-release, container-image building such as jib, linters such as detekt/ktlint, custom `processResources`/codegen tasks) with Toolchain equivalents at the project level; when adapting an existing CI pipeline (`./gradlew build`, `./gradlew check`, `./gradlew release`) to `./kotlin` commands. SKIP for porting a single Gradle plugin in isolation (use the `gradle-to-kotlin-toolchain-plugin` skill), for authoring a new Toolchain plugin from scratch (use `kotlin-toolchain-plugin-authoring`), and for general Kotlin Toolchain project work where Gradle is already gone (use `kotlin-toolchain`).
---

# Gradle → Kotlin Toolchain Project Migration

Migrating a whole Gradle Kotlin/JVM project to Kotlin Toolchain is two distinct exercises stapled together: a mostly-mechanical **translation** of dependencies and configuration, plus a non-trivial set of **plugin replacements** for everything Gradle did via plugins that Toolchain has no native answer for. This skill is the project-level playbook; it leans on the companion skills for the plugin pieces.

Read [references/examples.md](references/examples.md) for ready-to-adapt code: a JVM-app `module.yaml`, a multi-plugin `project.yaml`, the smallest hand-written plugin (a `package` plugin producing a stable `build/libs/` artifact), and CI workflow templates. That reference code is drawn from one real migration — treat it as a template to adapt, not a fixed recipe.

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
- **All source-code dependencies on build-generated artefacts.** Grep `src/` for resource names that are produced by custom tasks (e.g. a service that reads a `release.properties` / `version.txt` off the classpath, where the file is written by a custom `generate…` Gradle task). Each of these is a constraint the migration must honour without source changes.
- **`gradle/libs.versions.toml`** — note which `[versions]` and `[plugins]` entries are *only* used by Gradle plugins (those become dead weight to remove later).
- **CI workflows** — every `./gradlew <task>` invocation must map to a `./kotlin` command or be replaced. Note artefact upload paths (typically `build/libs/`), version-extraction shell pipelines, and any `-P` property flags.

Save this inventory as a markdown plan (e.g. `MIGRATION_PLAN.md`); it becomes the checklist the PR description verifies.

### Phase 2 — Decide layout, scope, and the plugin set

These three decisions shape everything that follows:

#### Layout: `maven-like`

Gradle projects use Maven layout (`src/main/kotlin`, `src/main/resources`, `src/test/kotlin`, `src/test/resources`). Toolchain's default layout is `src`/`test`/`resources`/`testResources`. To avoid moving every source file, set `layout: maven-like` in `module.yaml`. This is fully supported for `jvm/app` and `jvm/lib` products and is the migration-friendly choice.

#### Plugin set

Sort every Gradle plugin from the Phase 1 inventory into one of three buckets — **drop**, **use Toolchain native**, or **reimplement as a local plugin** — never "keep using the Gradle plugin" (Toolchain cannot consume Gradle plugins). The table below lists the categories that recur in JVM projects, with representative plugins as examples; your project will hit some subset, not all of them.

| Gradle plugin / feature (examples) | Replacement | Notes |
|---|---|---|
| Kotlin/JVM + serialization (`org.jetbrains.kotlin.jvm`, `kotlin.plugin.serialization`) | **Native** (`settings.jvm.jdk`, `settings.kotlin.serialization: json`) | Use `$libs.*` for Kotlin libs if you want pin control; otherwise the toolchain default applies. |
| The `application` plugin (mainClass + `build/libs/<name>.jar`) | **Native** for the entry point (`settings.jvm.mainClass`); a small **`package` local plugin** for the JAR's *location* so CI has a stable upload path — see [references/examples.md](references/examples.md). |
| Git-tag versioning (e.g. `axion-release`) | A **`release` local plugin** (typically JGit-based). Vendor one if it exists; otherwise port it via the `gradle-to-kotlin-toolchain-plugin` skill. Publishes the version as a file (e.g. `META-INF/<group>/version.txt`) under `generated.resources`. |
| Container-image building (e.g. `jib`) | A **local plugin** wrapping the tool's own library (e.g. `jib-core`) directly. When vendoring a community sample, expect to extend its settings (ports, environment, user are commonly omitted) **and verify it applies every configured tag** — a bare push often emits only the implicit `latest`; read CI tag overrides from an env var (Toolchain has no `-P`/`-D`). |
| Linters (e.g. `detekt`, `ktlint`) | A **local plugin** that subprocess-launches the tool's CLI. Vendor a known-good one where it exists; otherwise hand-write along the same shape. **Re-check the vendored plugin's defaults against the Gradle plugin** — see "Mismatches to watch". |
| Custom `generateXyz` / `processResources` tasks | An additional **`@TaskAction`** on the relevant plugin (often the release plugin). Output dir wired into `generated.resources`. |

Each local plugin is its own `jvm/amper-plugin` module under `plugins/<name>/`. Wire them all from `project.yaml`:

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

A reasonable default for a single migration PR: vendor the plugins that already have a known-good Toolchain source, author the small or bespoke ones (a `package` plugin, a thin linter wrapper), and defer anything non-essential (e.g. publishing to Maven Central) to a follow-up.

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
- **`./gradlew build && ./gradlew check`** → `./kotlin build && ./kotlin check`. `kotlin check` runs every plugin's `checks:` registrations (your linters + tests).
- **`./gradlew release`** → `./kotlin do release` (the release plugin exposes it as a command).
- **`./gradlew currentVersion -q | …`** → resolve the version *without* `| tail -n1` (see "Capturing a command's output value" under Mismatches): read the tag the release just made (`git describe --tags --exact-match HEAD`) or a file the plugin writes. `tail -n1` returns the toolchain's `<task> successful` banner, not the version.
- **`./gradlew jib -Djib.to.tags=…`** → `./kotlin do jib` with the tags supplied through an env var the plugin reads (no `-P`/`-D`/`--setting`). Confirm the plugin actually *applies* the tag list — see the container-image row in the plugin-set table.
- **Artefact upload path** must change. Gradle's `build/libs/<name>.jar` is gone; the Toolchain's `jarJvm` task writes to a Toolchain-internal path. **Author a `package` local plugin** that stages the JAR at `${module.rootDir}/build/libs/${module.name}.jar` so CI's `actions/upload-artifact` line stays simple. See [references/examples.md](references/examples.md) for the 30-line plugin.

### Phase 5 — Validate end-to-end

Run each user-facing command at least once locally. Capture the outputs in the PR description's Test plan:

```sh
./kotlin show modules            # project model loads, all expected modules listed
./kotlin clean && ./kotlin build # main build path is clean
./kotlin test                    # if local environment supports it (see host-incompatible containers below)
./kotlin check                   # linters + tests; expect zero violations after Phase 3
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

There is no `name:` field in `module.yaml`. The module name is the basename of the directory containing the file. In CI this affects task output paths: `actions/checkout` clones the repo into a directory whose basename is the GitHub repo name, while a local worktree or a clone under a different folder name resolves `${module.name}` to something else entirely. Never hardcode the module name into a path you encode in plugin task outputs — use `${module.name}` so it follows the directory.

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

### A vendored linter plugin can carry different defaults than the Gradle plugin

A subprocess-launching linter plugin you vendor may enable stricter analysis than the Gradle plugin did, surfacing a wave of "new" violations the Gradle build silently ignored. Diff the flags the vendored plugin passes against what the Gradle plugin's default task passed, and add an opt-in setting for anything stricter so the default matches the old behaviour.

The canonical instance: the upstream detekt plugin (`amper/build-sources/detekt/`) passes `--classpath` to `detekt-cli` unconditionally, enabling **type resolution**, while Gradle's default `detekt` task does not. Used as-is it surfaces violations Gradle never reported (notably `UnreachableCode` on elvis-with-return patterns).

**Fix**: add a `useTypeResolution: Boolean` setting (`get() = false`) to the plugin's `@Configurable Settings` and gate the `--classpath` flag on it. That matches Gradle parity by default; consumers who want the stronger checks opt in. See [references/examples.md](references/examples.md) for the full patch.

### No dependency exclusions

Toolchain has no equivalent of Gradle's `exclude(group, module)` for transitive dependency exclusions. If the source build excluded e.g. `kotlinx-coroutines-core` from one of its dependencies, that exclusion silently disappears under Toolchain — the library lands on the runtime classpath. Document the trade-off in the PR (usually a few hundred KB of unused classes; rarely a behavioural issue) and move on.

### No plugin-to-plugin dependencies

Plugins are isolated; one local plugin cannot declare a dependency on another local plugin in the same project. If two plugins logically belong together (e.g. a release plugin that writes the version, and the same plugin needs to publish that version in two formats), put both task actions in the same plugin module rather than splitting across two plugins.

### No `-P` / `-D` CLI overrides

Toolchain has no Gradle-equivalent `-Pkey=value` or `-Dkey=value` for overriding plugin settings from the CLI, and you can't count on a `--setting` flag either (the pinned CLI may reject it outright). For ephemeral overrides (force-version, skip-checks, dynamic image tags), read environment variables inside the `@TaskAction`. See the `gradle-to-kotlin-toolchain-plugin` skill's "Architectural mismatches" section for the pattern.

### Capturing a command's output value (`./kotlin do …` stdout is not a clean channel)

A `@TaskAction`'s `println` is **not** a machine-readable channel. The toolchain renders task stdout inside a decorated log line — `<ts> INFO :<module>:<task>@<plugin> <value>` — and prints a trailing `<task> successful` banner, so `./kotlin do currentVersion | tail -n1` yields `currentVersion successful`, not the version (and lowering `--log-level` to `error`/`off` drops the value line entirely, since it is emitted at INFO). Capture a value robustly by either:

- **Reading a side channel the plugin writes** — have the task write the value to a file named by an env var (e.g. `RELEASE_VERSION_FILE`) and `cat` it. Most decoupled: the captured value matches what the plugin stamps into artifacts.
- **Reading the git state the command produced** — for a release tag, `version=$(git describe --tags --exact-match HEAD)` then strip the prefix; no plugin-output parsing at all.

If you must parse stdout, match the task-coordinate line instead of taking the last one: `awk '/<task>@<plugin>/ { v = $NF } END { print v }'`.

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

### amd64-only container images won't run on Apple Silicon

Older images pinned by testcontainers (e.g. `mongo:3.2`) are amd64-only and crash immediately under Rosetta/QEMU on M-series Macs (`runtime: failed to create new OS thread (have 2 already; errno=22)`). This is **unchanged by the migration** — the same image fails identically under `./gradlew check`. Flag it in the PR's Test plan as a pre-existing host incompatibility, not a regression. CI on `ubuntu-latest` (amd64) is unaffected.

### iOS (KMP): a migrated `Info.plist` loses its `CFBundle*` keys → app won't install

A Gradle/KMP iOS app keeps its bundle metadata in the hand-maintained `iosApp.xcodeproj`, **not** in `Info.plist`. The KMP wizard sets `GENERATE_INFOPLIST_FILE = YES` and `PRODUCT_BUNDLE_IDENTIFIER = …` in the pbxproj, so Xcode *synthesizes* the `CFBundle*` keys at build time (`CFBundleIdentifier` from `PRODUCT_BUNDLE_IDENTIFIER`, `CFBundleExecutable` from `EXECUTABLE_NAME`, …) and merges them with the checked-in `Info.plist`. That plist is therefore intentionally **partial** — it carries only extras (usage descriptions, launch storyboard) and often has no `CFBundleIdentifier` at all.

Toolchain does **not** consume the existing `.xcodeproj`. It generates its own `module.xcodeproj` and uses your `Info.plist` verbatim (via `INFOPLIST_FILE`, **without** `GENERATE_INFOPLIST_FILE`), writing its own complete default plist *only* when none exists. So the migrated partial plist survives, nothing synthesizes the `CFBundle*` keys, and the built `.app` has no bundle id:

```
Simulator device failed to install the application. Missing bundle ID.
```

A fresh `kotlin init` iOS app doesn't hit this (Toolchain writes a complete default plist), which is why it surfaces only on migration.

**Fix**: make the module's `Info.plist` self-contained — add the standard `CFBundle*` keys (the values reference build settings Toolchain already sets), keeping your app-specific keys alongside them:

```xml
<key>CFBundleDevelopmentRegion</key><string>$(DEVELOPMENT_LANGUAGE)</string>
<key>CFBundleExecutable</key><string>$(EXECUTABLE_NAME)</string>
<key>CFBundleIdentifier</key><string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
<key>CFBundleInfoDictionaryVersion</key><string>6.0</string>
<key>CFBundleName</key><string>$(PRODUCT_NAME)</string>
<key>CFBundlePackageType</key><string>APPL</string>
<key>CFBundleShortVersionString</key><string>1.0</string>
<key>CFBundleVersion</key><string>1</string>
```

(`PRODUCT_BUNDLE_IDENTIFIER` is set on the generated target, so `$(PRODUCT_BUNDLE_IDENTIFIER)` resolves.) See the `kotlin-toolchain` skill's "iOS apps" section for the underlying generate-project behavior.

### KMP: carry over *every* source set's dependencies — don't drop "belongs-elsewhere-looking" ones

A Gradle KMP module declares dependencies per source set (`commonMain`, `jvmMain`, `androidMain`, `iosMain`, `commonTest`, `jvmTest`, …). **Each block must be carried over to the matching Amper section** — `dependencies` (common), `dependencies@jvm` / `@android` / `@ios` (platform), `test-dependencies`, `test-dependencies@<platform>`. Translate the source sets exhaustively; do not cherry-pick the "obvious library" deps. Two facts make silent drops costly:

- **A `*Main` dependency is also on that target's *test* classpath.** In KMP, `jvmTest` extends `jvmMain`, so an `implementation(...)` in `jvmMain` is visible to JVM tests. Dropping a `*Main` dep can remove something the tests silently relied on — with no compile error, only a runtime failure.
- **A dependency can look like it belongs to another module but be load-bearing here.** Canonical case: `jvmMain { implementation(compose.desktop.currentOs) }` in a *shared library*. It reads like a desktop-app dependency (easy to drop when there's a separate desktop-app module), but it supplies the **Skiko native runtime** (`skiko-awt-runtime-<os>`, which carries `libskiko-<os>.dylib` + its `.sha256`) that the module's own **JVM Compose UI tests** (`compose.uiTest` / `runComposeUiTest`) load at runtime.

Dropping `compose.desktop.currentOs` compiles fine, then JVM Compose UI tests crash at runtime with a cryptic:

```
org.jetbrains.skiko.LibraryLoadException: Cannot find libskiko-macos-arm64.dylib.sha256, proper native dependency missing.
```

This is **not** a Toolchain bug — a Gradle build fails identically without it (`compose.ui:ui-test` pulls only Skiko's *classes*, never the native runtime; the natives come from the desktop artifact). It was simply not carried over. Restore it under `dependencies@jvm`, matching where Gradle had it:

```yaml
dependencies@jvm:
  - $compose.desktop.currentOs
```

Or, cleaner for a library (keeps Skiko natives off consumers' classpaths), scope it to tests:

```yaml
test-dependencies@jvm:
  - $compose.desktop.currentOs
```

**Guard against drops:** after writing `module.yaml`, diff each Gradle source set's dependency list against its Amper counterpart (names + count), and run `./kotlin show dependencies -m <module>` to confirm the resolved graph matches the Gradle build's per-configuration dependencies. Anything present in a Gradle `*Main`/`*Test` block but absent from the corresponding Amper section is a bug until proven otherwise.

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
- **Hardcoded tool versions in plugin YAMLs.** Vendoring drops literal versions (the linter CLIs, the image-builder library, JGit, etc.) into `module.yaml`/`plugin.yaml`. Sweep every one into `gradle/libs.versions.toml` and use `$libs.*` references; otherwise Dependabot can't track them.
- **String interpolation into Maven coordinates.** `plugin.yaml` does not allow `${pluginSettings.version}` *inside* a Maven coordinate string (`com.example:foo:${...}` fails with "Value of type 'ShadowDependency' doesn't support string interpolation"). Either hardcode the version in `plugin.yaml` or use a catalog reference (`$libs.foo`).

## Additional resources

- For concrete code patterns (root `module.yaml`, multi-plugin `project.yaml`, a minimal `package` plugin, CI workflow templates, the `useTypeResolution` patch), see [references/examples.md](references/examples.md).
- For per-plugin port specifics (the concept-mapping table, MVP scoping, file-based publication), use the companion `gradle-to-kotlin-toolchain-plugin` skill.
- For writing a plugin from scratch (the `package` plugin or any other), use `kotlin-toolchain-plugin-authoring`.
- For Toolchain syntax reference, use `kotlin-toolchain`.
- Toolchain docs: <https://kotlin-toolchain.org/dev/>
