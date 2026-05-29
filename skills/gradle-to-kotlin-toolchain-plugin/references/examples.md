# Conversion patterns — concrete code

Patterns lifted from a real conversion (the Gradle plugin `allegro/axion-release-plugin` ported to a Kotlin Toolchain local release plugin). Adapt names, types, and imports to your plugin.

## 1. `plugins/<name>/module.yaml`

The plugin module itself is a `jvm/amper-plugin`. Declare the Maven coordinates of every library the plugin uses, including transitive optionals that fire only at runtime.

```yaml
product: jvm/amper-plugin

dependencies:
  - org.eclipse.jgit:org.eclipse.jgit:7.0.0.202409031743-r
  - org.eclipse.jgit:org.eclipse.jgit.ssh.apache:7.0.0.202409031743-r
  # MINA SSHD (pulled in by jgit.ssh.apache) declares BouncyCastle as optional
  # at compile time but builds its random factory from it at runtime. Pin to
  # the version contemporary with MINA SSHD 2.13.x.
  - org.bouncycastle:bcprov-jdk18on:1.78.1: runtime-only
  - org.bouncycastle:bcpkix-jdk18on:1.78.1: runtime-only
  - org.slf4j:slf4j-nop:2.0.13

pluginInfo:
  id: release
  settingsClass: com.example.release.Settings

settings:
  jvm:
    jdk:
      version: 21
  kotlin:
    languageVersion: 2.1
```

Key points:
- `pluginInfo.id` is what users address tasks by (`...@release`) and what the `plugins:` block in consumer `module.yaml` uses as the key.
- `pluginInfo.settingsClass` is the fully-qualified name of the `@Configurable` interface (next section).
- Use suffix scopes (`: runtime-only`, `: compile-only`, `: exported`) just like in a normal Toolchain module.

## 2. `@Configurable interface Settings` — the Gradle extension analog

Defaults via interface property getters. Nested blocks become nested `@Configurable` interfaces.

```kotlin
package com.example.release

import org.jetbrains.amper.plugins.Configurable

@Configurable
interface Settings {
    val repoDir: String get() = ""
    val tagPrefix: String get() = "v"
    val versionSeparator: String get() = ""
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

Consumer side, in the demo module's `module.yaml`:

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

Anything not specified uses the interface's default.

## 3. `@TaskAction` signature — Gradle Task analog

One top-level `fun` per file under `src/tasks/`. Parameters are typed; `Path` parameters carry `@Input` or `@Output`; the settings object is wired separately in `plugin.yaml` (next section).

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

## 4. `plugin.yaml` — task + command registry

Each task's `action:` block wires the `@TaskAction` function's parameters. `${module.rootDir}`, `${taskOutputDir}`, `${pluginSettings}` are the documented references. The `commands:` block names the public entry points; everything else is build-graph-only.

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
# `writeVersion` is intentionally NOT exposed — it's a build-graph contributor
# (its @Output is wired into generated.resources above) that runs automatically
# when something downstream needs the version file.
commands:
  - currentVersion
  - release
```

Key points:
- `!com.example.release.tasks.currentVersion` is the YAML tag form addressing the `@TaskAction` function by fully-qualified name.
- Every `Settings` parameter needs its own `settings: ${pluginSettings}` line — forgetting it silently passes `null`.
- `${tasks.<task>.action.<param>}` cross-references another task's parameter, letting you reuse an `@Output` directory in `generated.resources` / `generated.sources`.

## 5. File-based publication — the `project.version` analog

The Gradle idiom `version = scmVersion.version` shares a value across the whole build. Toolchain has no such shared property. Replace it with a `@TaskAction` that writes a file:

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

`ExecutionAvoidance.Disabled` is required because the true input (Git history, branch name, working-tree state) cannot be fingerprinted by Toolchain.

That one `@Output` directory feeds two consumption paths:

### Build-time (other `@TaskAction`s)

Any other task — in this plugin, another plugin, or user code — declares `@Input` on the same path. Toolchain auto-infers the dependency from path matching:

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

Register the directory under `generated.resources` (already shown in section 4) so it ends up on the JAR classpath:

```kotlin
fun version(): String? =
    object {}.javaClass.getResourceAsStream("/META-INF/release/version.txt")
        ?.bufferedReader()?.use { it.readText().trim() }
```

This is the cleanest analog to `project.version`: one publication point, two consumption channels, no Toolchain-side shared state.

## 6. Env-var overrides — the `-P` analog

Settings hold static config; env vars hold ephemeral CLI overrides. Read them inside the action body.

```kotlin
class VersionPipeline(
    private val settings: Settings,
    private val env: Map<String, String?> = System.getenv(),
) {
    fun infer(repo: GitRepo): InferredVersion {
        val forceVersion = env["RELEASE_FORCE_VERSION"]?.takeIf { it.isNotBlank() }
        val forceSnapshot = env["RELEASE_FORCE_SNAPSHOT"].asBoolean()
        // ...
    }
}

private fun String?.asBoolean(): Boolean =
    this != null && this.equals("true", ignoreCase = true)
```

Pass the env map through the constructor (rather than calling `System.getenv()` directly inside methods) so tests can inject a controlled map.

Document the mapping prominently in the README:

```
-Prelease.forceVersion=X              → RELEASE_FORCE_VERSION=X
-Prelease.forceSnapshot               → RELEASE_FORCE_SNAPSHOT=true
-Prelease.disableChecks               → RELEASE_DISABLE_CHECKS=true
-Prelease.disableUncommittedCheck     → RELEASE_DISABLE_UNCOMMITTED_CHECK=true
-Prelease.disableRemoteCheck          → RELEASE_DISABLE_REMOTE_CHECK=true
-Prelease.overriddenBranchName=X      → RELEASE_OVERRIDDEN_BRANCH_NAME=X
```

## 7. Atomic combo task — one `@TaskAction` calling internal helpers

Gradle builds atomic chains via `dependsOn` / `finalizedBy`. In Toolchain, write a single `@TaskAction` that calls internal `fun`s extracted from each step. Sharing one resource (here, `GitRepo`) across all steps eliminates the window where another process can observe intermediate state.

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
): String { /* ... returns the tag short name ... */ }

internal fun pushReleaseTag(repo: GitRepo, tagName: String) { /* ... */ }
```

Then `release` is just:

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

Each individual step (`createRelease`, `pushRelease`) is also exposed as its own `@TaskAction` for the "tag locally now, push later" workflow. The shared internal helpers prevent the implementations from drifting.

## 8. Demo module — the validation harness

The demo module proves the plugin works end-to-end. Its `module.yaml` enables the plugin and supplies a configuration; its `src/main.kt` consumes whatever the plugin publishes.

```yaml
# demo-app/module.yaml
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
// demo-app/src/main.kt
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
./kotlin run :demo-app
# => demo-app version: 0.1.0-SNAPSHOT
```

## 9. Pre-release / validation pattern

For plugins with pre-action checks (axion's `verifyRelease`), extract them into a dedicated class taking settings + env. Run inside the `@TaskAction`; turn a non-empty failure list into a single `error(...)`.

```kotlin
class ReleaseChecks(
    private val settings: Settings,
    private val pipeline: VersionPipeline,
    private val env: Map<String, String?> = System.getenv(),
) {
    fun run(repo: GitRepo): List<String> {
        if (env["RELEASE_DISABLE_CHECKS"].asBoolean()) return emptyList()

        val failures = mutableListOf<String>()

        if (settings.checks.uncommittedChanges && /* ... */) {
            if (repo.isDirty()) failures += "Working tree has uncommitted changes. ..."
        }
        // ... more checks
        return failures
    }
}
```

Make every failure message tell the user exactly which setting toggles it off and which env var bypasses it:

> "Working tree has uncommitted changes. Commit or stash them, set `ignoreUncommittedChanges: true` in `module.yaml`, or run with `RELEASE_DISABLE_UNCOMMITTED_CHECK=true`."

## 10. `project.yaml`

The root file. Lists every module (including plugins) and points at the plugin source directory under `plugins:`:

```yaml
modules:
  - demo-app
  - plugins/release

plugins:
  - ./plugins/release
```

The `plugins:` block at root level is what makes the plugin discoverable; without it, consumer `module.yaml` files that reference `plugins: { release: enabled }` cannot resolve the id.

## 11. `.gitignore` — write this first

Add **before** the first commit, otherwise the build output will be in your initial commit and every subsequent dirty-tree check (your own or the plugin's) will spuriously fail.

```
build/
.kotlin/
.idea/
*.iml
.DS_Store
```
