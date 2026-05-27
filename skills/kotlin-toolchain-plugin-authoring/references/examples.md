# Plugin authoring patterns — concrete code

Ready-to-adapt snippets for writing a Kotlin Toolchain local plugin. Adapt names, packages, imports, and types to your domain. Every pattern below appears in `kotlin-toolchain-release-plugin`, which can be read as a complete worked example.

## 1. `project.yaml` — registering the plugin

```yaml
modules:
  - <consumer-module>
  - plugins/<name>

plugins:
  - ./plugins/<name>
```

Without the top-level `plugins:` block the plugin id is unresolvable from any consumer module.

## 2. `plugins/<name>/module.yaml` — the plugin module

```yaml
product: jvm/amper-plugin

dependencies:
  - <coordinate>:<version>
  - <coordinate>:<version>: runtime-only

pluginInfo:
  id: <plugin-id>
  settingsClass: com.example.<name>.Settings

settings:
  jvm:
    jdk:
      version: 21
  kotlin:
    languageVersion: 2.1
```

## 3. `@Configurable interface Settings`

Defaults via interface getters. Nested blocks become nested `@Configurable` interfaces.

```kotlin
package com.example.release

import org.jetbrains.amper.plugins.Configurable

@Configurable
interface Settings {
    val tagPrefix: String get() = "v"
    val initialVersion: String get() = "0.1.0"
    val ignoreUncommittedChanges: Boolean get() = false
    val releaseBranchPattern: String get() = "main|master"

    val checks: ChecksSettings
}

@Configurable
interface ChecksSettings {
    val uncommittedChanges: Boolean get() = true
    val aheadOfRemote: Boolean get() = true
}
```

Consumer `module.yaml`:

```yaml
plugins:
  release:
    enabled: true
    tagPrefix: "v"
    initialVersion: "0.1.0"
    releaseBranchPattern: "main|master"
    checks:
      uncommittedChanges: true
      aheadOfRemote: true
```

Anything not specified uses the interface default.

## 4. `@TaskAction` — one per file under `src/tasks/`

```kotlin
package com.example.release.tasks

import com.example.release.Settings
import com.example.release.git.GitRepo
import com.example.release.version.VersionPipeline
import org.jetbrains.amper.plugins.Input
import org.jetbrains.amper.plugins.TaskAction
import java.nio.file.Path

@TaskAction
fun currentVersion(
    @Input moduleRootDir: Path,
    settings: Settings,
) {
    val pipeline = VersionPipeline(settings)
    GitRepo.open(moduleRootDir, settings.repoDir).use { repo ->
        println(pipeline.infer(repo).version)
    }
}
```

Key points:
- Top-level `fun`, not a method on a class.
- `@Input` / `@Output` on `Path` parameters declare build-graph inputs/outputs.
- The `Settings` parameter is wired separately in `plugin.yaml` (see section 6).
- Print to `println(...)` for user-visible output; Toolchain captures stdout.

## 5. `@TaskAction(executionAvoidance = ExecutionAvoidance.Disabled)` — hidden inputs

Use when the task's true input is Git state, network, environment variables, or anything else Toolchain cannot fingerprint via `@Input`.

```kotlin
package com.example.release.tasks

import com.example.release.Settings
import com.example.release.git.GitRepo
import com.example.release.version.VersionPipeline
import org.jetbrains.amper.plugins.ExecutionAvoidance
import org.jetbrains.amper.plugins.Input
import org.jetbrains.amper.plugins.Output
import org.jetbrains.amper.plugins.TaskAction
import java.nio.file.Path
import kotlin.io.path.ExperimentalPathApi
import kotlin.io.path.createParentDirectories
import kotlin.io.path.deleteRecursively
import kotlin.io.path.div
import kotlin.io.path.writeText

@OptIn(ExperimentalPathApi::class)
@TaskAction(executionAvoidance = ExecutionAvoidance.Disabled)
fun writeVersion(
    @Input moduleRootDir: Path,
    @Output outputDir: Path,
    settings: Settings,
) {
    val pipeline = VersionPipeline(settings)
    val inferred = GitRepo.open(moduleRootDir, settings.repoDir).use { pipeline.infer(it) }

    outputDir.deleteRecursively()
    val versionFile = outputDir / "META-INF" / "release" / "version.txt"
    versionFile.createParentDirectories()
    versionFile.writeText(inferred.version + "\n")
}
```

The `outputDir` parameter must be the *same* `Path` you write to — otherwise downstream consumers depending on `${tasks.writeVersion.action.outputDir}` won't find the result.

## 6. `plugin.yaml` — tasks, commands, generated

```yaml
tasks:
  currentVersion:
    action: !com.example.release.tasks.currentVersion
      moduleRootDir: ${module.rootDir}
      settings: ${pluginSettings}

  writeVersion:
    action: !com.example.release.tasks.writeVersion
      moduleRootDir: ${module.rootDir}
      outputDir: ${taskOutputDir}
      settings: ${pluginSettings}

  release:
    action: !com.example.release.tasks.release
      moduleRootDir: ${module.rootDir}
      settings: ${pluginSettings}

generated:
  resources:
    - directory: ${tasks.writeVersion.action.outputDir}

# Public entry points invoked via `./kotlin do <name>`.
# `writeVersion` stays out — it's a build-graph contributor (its @Output is
# wired into generated.resources above) and runs automatically when something
# downstream needs the version file.
commands:
  - currentVersion
  - release
```

Key points:
- `!com.example.release.tasks.currentVersion` is the YAML tag form, addressing the `@TaskAction` function by fully-qualified name.
- Every `Settings` parameter needs its own `settings: ${pluginSettings}` line. Missing this is the single most common wiring mistake.
- `${tasks.<task>.action.<param>}` cross-references another task's parameter — use it to compose outputs into `generated.*` blocks or into other tasks' `@Input` parameters.

## 7. File-based publication — the `project.version` analog

Toolchain has no shared mutable build state. To publish a value from one task to other tasks (or to runtime code), write a file in your `@Output` directory:

```kotlin
@OptIn(ExperimentalPathApi::class)
@TaskAction(executionAvoidance = ExecutionAvoidance.Disabled)
fun writeVersion(
    @Input moduleRootDir: Path,
    @Output outputDir: Path,
    settings: Settings,
) {
    // ... compute inferred ...
    val versionFile = outputDir / "META-INF" / "release" / "version.txt"
    versionFile.createParentDirectories()
    versionFile.writeText(inferred.version + "\n")
}
```

Two consumption paths feed off the same `@Output`:

### Build-time (another `@TaskAction`)

Declare `@Input` on a matching path; Toolchain auto-infers the dependency:

```kotlin
@TaskAction
fun packageWithVersion(
    @Input versionFile: Path,
    // ...
) {
    val version = versionFile.readText().trim()
    // ...
}
```

```yaml
tasks:
  packageWithVersion:
    action: !com.example.packageWithVersion
      versionFile: ${tasks.writeVersion.action.outputDir}/META-INF/release/version.txt
```

### Runtime (application classpath)

Register the directory under `generated.resources` in `plugin.yaml` (see section 6). The file lands on the JAR classpath:

```kotlin
fun version(): String? =
    object {}.javaClass.getResourceAsStream("/META-INF/release/version.txt")
        ?.bufferedReader()?.use { it.readText().trim() }
```

## 8. Environment-variable overrides

Toolchain has no `-P` properties. For ephemeral CLI overrides, read env vars inside the action — passing the env map through a constructor parameter keeps logic unit-testable.

```kotlin
class VersionPipeline(
    private val settings: Settings,
    private val env: Map<String, String?> = System.getenv(),
) {
    fun infer(repo: GitRepo): InferredVersion {
        val forced = env["RELEASE_FORCE_VERSION"]?.takeIf { it.isNotBlank() }
        val forceSnapshot = env["RELEASE_FORCE_SNAPSHOT"].asBoolean()
        // ...
    }
}

private fun String?.asBoolean(): Boolean =
    this != null && this.equals("true", ignoreCase = true)
```

Document every recognised env var in the plugin's README so consumers know which overrides exist.

## 9. Sharing logic across task actions

Two or more `@TaskAction`s frequently share steps. Extract each step into an `internal fun` in a sibling file (not annotated with `@TaskAction`). The individual-step task and the combo task both call the same helpers.

```kotlin
package com.example.release.tasks

import com.example.release.Settings
import com.example.release.checks.ReleaseChecks
import com.example.release.git.GitRepo
import com.example.release.version.VersionPipeline

internal fun verifyOrFail(repo: GitRepo, settings: Settings, pipeline: VersionPipeline) {
    val failures = ReleaseChecks(settings, pipeline).run(repo)
    if (failures.isNotEmpty()) {
        error(failures.joinToString(prefix = "Pre-release checks failed:\n  - ", separator = "\n  - "))
    }
}

internal fun createReleaseTag(
    repo: GitRepo,
    settings: Settings,
    pipeline: VersionPipeline,
): String { /* ... */ }

internal fun pushReleaseTag(repo: GitRepo, tagName: String) { /* ... */ }
```

The combo task is then one `@TaskAction` sharing a single resource (`GitRepo`) across all three steps:

```kotlin
@TaskAction
fun release(
    @Input moduleRootDir: Path,
    settings: Settings,
) {
    val pipeline = VersionPipeline(settings)
    GitRepo.open(moduleRootDir, settings.repoDir).use { repo ->
        verifyOrFail(repo, settings, pipeline)
        val tagName = createReleaseTag(repo, settings, pipeline)
        pushReleaseTag(repo, tagName)
    }
}
```

Each individual step also gets exposed as its own `@TaskAction` for the "stage now, finish later" workflow:

```kotlin
@TaskAction
fun createRelease(@Input moduleRootDir: Path, settings: Settings) {
    val pipeline = VersionPipeline(settings)
    GitRepo.open(moduleRootDir, settings.repoDir).use { repo ->
        verifyOrFail(repo, settings, pipeline)
        createReleaseTag(repo, settings, pipeline)
    }
}
```

Both paths back into the same shared helpers, so implementations cannot drift.

## 10. Pre-action verification pattern

For plugins with pre-action checks (axion's `verifyRelease`, schema validation, contract tests), extract them into a dedicated class taking settings + env. Run inside the `@TaskAction`; turn a non-empty failure list into a single `error(...)`.

```kotlin
class ReleaseChecks(
    private val settings: Settings,
    private val pipeline: VersionPipeline,
    private val env: Map<String, String?> = System.getenv(),
) {
    fun run(repo: GitRepo): List<String> {
        if (env["RELEASE_DISABLE_CHECKS"].asBoolean()) return emptyList()

        val failures = mutableListOf<String>()

        if (settings.checks.uncommittedChanges && !settings.ignoreUncommittedChanges) {
            if (repo.isDirty()) {
                failures += "Working tree has uncommitted changes. " +
                    "Commit or stash them, set ignoreUncommittedChanges=true in module.yaml, " +
                    "or run with RELEASE_DISABLE_UNCOMMITTED_CHECK=true."
            }
        }
        // ... more checks
        return failures
    }
}
```

Every failure message should name (1) the setting that toggles it off and (2) the env var that bypasses it for an ad-hoc run. Diagnostic ergonomics are the difference between a debuggable plugin and a frustrating one.

## 11. Consumer / validation module

A small consumer in the same repo doubles as an integration-test harness and a getting-started example.

```yaml
# consumer-app/module.yaml
product: jvm/app

plugins:
  release:
    enabled: true
    tagPrefix: "v"
    initialVersion: "0.1.0"
    releaseBranchPattern: "main|master"

settings:
  jvm:
    mainClass: com.example.demo.MainKt
    jdk:
      version: 21
  kotlin:
    languageVersion: 2.1
```

```kotlin
// consumer-app/src/main.kt — exercises whatever the plugin publishes
package com.example.demo

private const val VERSION_RESOURCE = "/META-INF/release/version.txt"

fun main() {
    val version = readVersionFromClasspath() ?: "(version unavailable)"
    println("demo-app version: $version")
}

private fun readVersionFromClasspath(): String? =
    object {}.javaClass.getResourceAsStream(VERSION_RESOURCE)
        ?.bufferedReader()
        ?.use { it.readText().trim() }
        ?.takeIf { it.isNotEmpty() }
```

Smoke test:

```sh
./kotlin run :consumer-app
./kotlin do <command-name>          # invokes the plugin's public commands
./kotlin show commands              # lists everything in plugin.yaml's commands: block
```

## 12. `.gitignore` — write this first

Add **before** the first commit, otherwise build output ends up in your initial commit and every dirty-tree check will spuriously fail thereafter.

```
build/
.kotlin/
.idea/
*.iml
.DS_Store
```
