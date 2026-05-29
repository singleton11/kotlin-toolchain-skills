# Migration patterns — concrete code

Patterns lifted from a real migration (a Gradle + Ktor + Exposed + MongoDB + Postgres service to Kotlin Toolchain). Adapt names, scopes, and versions to your project.

## 1. `project.yaml` — register all local plugins

Module list is alphabetical. The `plugins:` block points at each plugin's source directory; without it, the consumer module's `plugins:` block cannot resolve the plugin id.

```yaml
modules:
  - plugins/detekt
  - plugins/jib
  - plugins/ktlint
  - plugins/package
  - plugins/release

plugins:
  - ./plugins/detekt
  - ./plugins/jib
  - ./plugins/ktlint
  - ./plugins/package
  - ./plugins/release
```

The root module (`.`) is included by default and does not need to be listed.

## 2. Root `module.yaml` — a JVM/Ktor application

`product: jvm/app` plus `layout: maven-like` to keep `src/main/kotlin`/`src/main/resources`/`src/test/kotlin`/`src/test/resources` exactly where Gradle had them. Every dependency is a `$libs.*` reference; the catalog is the single source of truth.

```yaml
product: jvm/app

layout: maven-like

repositories:
  - id: jitpack
    url: https://jitpack.io

plugins:
  release: enabled
  jib:
    enabled: true
    container:
      mainClass: io.example.App
      jvmArgs: ["-Xms256m", "-Xmx512m"]
      ports: ["8080"]
      user: "1000"
      environment:
        SERVICE_PROFILE: "prod"
    baseImage:
      fullName: eclipse-temurin:21-jre-alpine
    targetImage:
      name: registry.example.com/team/service
      tags: ["latest"]
  detekt:
    enabled: true
    configFile: detekt.yml          # relative to module root; ${...} interpolation NOT supported here
    buildUponDefaultConfig: true
    rulesetClasspath:
      - $libs.detekt.rules
  ktlint: enabled
  package: enabled

settings:
  jvm:
    mainClass: io.example.App
    jdk:
      version: 21
  kotlin:
    serialization: json
    freeCompilerArgs: ["-Xskip-metadata-version-check"]

dependencies:
  - $libs.arrow.core
  - $libs.ktor.server.core
  - $libs.ktor.server.netty
  - $libs.ktor.server.content.negotiation
  - $libs.ktor.serialization.kotlinx.json
  # ... database / config / logging libs ...

test-dependencies:
  - $libs.kotest.runner.junit5
  - $libs.kotest.assertions.core
  - $libs.ktor.server.test.host
  - $libs.testcontainers.mongodb
  - $libs.testcontainers.postgresql
  - $libs.mockk
```

### Notes

- `release: enabled` is the shorthand form and is only valid when no plugin settings are set.
- `jib:` and `detekt:` use the long form (`enabled: true` plus settings) because they pass configuration. **Forgetting `enabled: true` when settings are present silently disables the plugin** — Toolchain only warns.
- `configFile: detekt.yml` is a relative path because `${module.rootDir}` interpolation does not work in `module.yaml`.

## 3. Stub `build.gradle.kts` for Dependabot

GitHub Dependabot's `package-ecosystem: gradle` file fetcher requires a `build.gradle(.kts)` in the configured directory before it scans `gradle/libs.versions.toml`. Without this stub, the Dependabot weekly run silently does nothing for Gradle-ecosystem dependencies.

```kotlin
// Intentionally empty.
//
// This project is built with the Kotlin Toolchain (Amper engine), not Gradle.
// All dependency versions live in `gradle/libs.versions.toml`, which the
// Toolchain consumes natively as its `$libs.*` catalog.
//
// This file exists so that GitHub Dependabot's `package-ecosystem: gradle`
// integration discovers the project: Dependabot's detector requires either a
// `build.gradle(.kts)` or `settings.gradle(.kts)` at the configured directory
// before it will scan `gradle/libs.versions.toml` for updatable dependencies.
// See `.github/dependabot.yml`.
```

Source confirmation: <https://github.com/dependabot/dependabot-core/blob/main/gradle/lib/dependabot/gradle/file_fetcher.rb> — `required_files_in?` gates on `SUPPORTED_BUILD_FILE_NAMES = %w(build.gradle build.gradle.kts)`.

## 4. `.sdkmanrc` — pin the Toolchain version

Add a `kotlintoolchain` entry alongside the existing `java` line. The value must match the version pinned in your `kotlin` wrapper script (`kotlin_cli_version=…` at the top of the wrapper).

```
# Enable auto-env through the sdkman_auto_env config
java=21.0.7-tem
kotlintoolchain=0.11.0
```

## 5. `gradle/libs.versions.toml` — catalog cleanup

Two cleanup passes after the initial migration:

1. **Remove the `[plugins]` block entirely.** Toolchain has no use for Gradle plugin coordinates; every former entry is dead.
2. **Remove `[versions]` keys that only fed those plugins** (e.g. `kotlin`, `axion-release-plugin`, `jib-plugin`, `ktlint-plugin`, `detekt-plugin` — anything whose `version.ref` was used only by `[plugins]` entries).

Then **add catalog entries for every coordinate the local plugins use** so Dependabot can track them centrally. Group by purpose with a comment header:

```toml
# Linters
detekt = "1.23.8"
detekt-rules = "1.0.1"
ktlint = "1.5.0"

# Local plugin runtime
jib-core = "0.28.1"
jgit = "7.0.0.202409031743-r"
# Pinned to the version contemporary with MINA SSHD 2.13.x (what jgit 7.0.0 ships against);
# without bouncycastle on the runtime classpath, SSH push fails with NoClassDefFoundError.
bouncycastle = "1.78.1"
slf4j = "2.0.13"

[libraries]
detekt-cli = { module = "io.gitlab.arturbosch.detekt:detekt-cli", version.ref = "detekt" }
detekt-rules = { module = "com.github.marc0der:detekt-rules", version.ref = "detekt-rules" }
ktlint-cli = { module = "com.pinterest.ktlint:ktlint-cli", version.ref = "ktlint" }
jib-core = { module = "com.google.cloud.tools:jib-core", version.ref = "jib-core" }
jgit = { module = "org.eclipse.jgit:org.eclipse.jgit", version.ref = "jgit" }
jgit-ssh-apache = { module = "org.eclipse.jgit:org.eclipse.jgit.ssh.apache", version.ref = "jgit" }
bouncycastle-bcprov = { module = "org.bouncycastle:bcprov-jdk18on", version.ref = "bouncycastle" }
bouncycastle-bcpkix = { module = "org.bouncycastle:bcpkix-jdk18on", version.ref = "bouncycastle" }
slf4j-nop = { module = "org.slf4j:slf4j-nop", version.ref = "slf4j" }
```

Every plugin's `module.yaml` and `plugin.yaml` then refers to these via `$libs.detekt.cli` etc. — never a hardcoded coordinate.

## 6. The smallest plugin: `package`

Stages the application JAR at `${module.rootDir}/build/libs/${module.name}.jar`, giving CI a stable, build-tool-agnostic path equivalent to what Gradle's `application` plugin published. Roughly 30 lines of Kotlin + two short YAMLs.

### `plugins/package/module.yaml`

```yaml
product: jvm/amper-plugin

description: Stages the application JAR at build/libs/${module.name}.jar for CI artifact uploads.

pluginInfo:
  settingsClass: io.example.amper.plugins.pkg.Settings

settings:
  jvm:
    jdk:
      version: 21
  kotlin:
    languageVersion: 2.1
```

### `plugins/package/src/Settings.kt`

```kotlin
package io.example.amper.plugins.pkg

import org.jetbrains.amper.plugins.Configurable

@Configurable
interface Settings {
    /**
     * Optional override for the file name of the staged JAR (no extension).
     * Defaults to the module name.
     */
    val artifactName: String?
}
```

### `plugins/package/src/PackageApplication.kt`

```kotlin
package io.example.amper.plugins.pkg

import org.jetbrains.amper.plugins.CompilationArtifact
import org.jetbrains.amper.plugins.Input
import org.jetbrains.amper.plugins.Output
import org.jetbrains.amper.plugins.TaskAction
import java.nio.file.Path
import kotlin.io.path.copyTo
import kotlin.io.path.createParentDirectories

@TaskAction
fun packageApplication(
    @Input jar: CompilationArtifact,
    @Output outputJar: Path,
) {
    jar.artifact.copyTo(outputJar.createParentDirectories(), overwrite = true)
}
```

### `plugins/package/plugin.yaml`

```yaml
tasks:
  package:
    action: !io.example.amper.plugins.pkg.packageApplication
      jar: ${module.jar}                                                      # CompilationArtifact-typed; auto-wires the jarJvm dependency
      outputJar: ${module.rootDir}/build/libs/${module.name}.jar              # project-relative @Output, same pattern as BCV plugin's apiDump

commands:
  - name: package
    performedBy: package
```

Why a plugin and not a shell step in CI:
- `${module.jar}` automatically wires the dependency on the module's `:jarJvm` task — running `./kotlin do package` builds the JAR first as a side-effect.
- The path contract (`${module.rootDir}/build/libs/${module.name}.jar`) is encoded in the plugin, so CI doesn't need to know Toolchain internals (`build/tasks/_<module>_jarJvm/<module>-jvm.jar`) and continues to work if those internals change.
- Consumers in different repos just enable the plugin (`package: enabled`) and `./kotlin do package` produces the same shape everywhere.

## 7. Detekt `useTypeResolution` patch

The vendored upstream detekt plugin (`amper/build-sources/detekt/`) passes `--classpath` unconditionally, enabling detekt's type-resolution mode. Gradle's default `detekt` task does **not** enable type resolution, which means a fresh migration that uses the vendored plugin as-is surfaces violations Gradle never reported (notably `UnreachableCode` on `?: return@get` elvis patterns).

Add an opt-in setting to the plugin to match Gradle parity by default.

### In `plugins/detekt/src/Settings.kt`

```kotlin
@Configurable
interface Settings {
    val configFile: Path?
    val buildUponDefaultConfig: Boolean get() = false

    /**
     * Run detekt with type resolution enabled (passes the module's compile classpath via `--classpath`).
     *
     * Type resolution activates additional rules that depend on type information, but some of those rules
     * (notably `UnreachableCode`) are known to produce false positives. Defaults to `false` to match the
     * behaviour of the Gradle detekt plugin's default `detekt` task; set to `true` to opt in.
     */
    val useTypeResolution: Boolean get() = false

    /**
     * Extra rule-set jars to load into Detekt via `--plugins`.
     */
    val rulesetClasspath: Classpath
}
```

### In `plugins/detekt/src/runDetekt.kt` (inside the args-building block)

```kotlin
// Provide classpath for type resolution if opted in (see Settings.useTypeResolution).
if (commonParameters.settings.useTypeResolution) {
    val cp = commonParameters.moduleClasspath.resolvedFiles
    if (cp.isNotEmpty()) {
        add("--classpath")
        add(cp.joinToString(File.pathSeparator))
    }
}
```

Consumers who want type-resolved analysis opt in via `module.yaml`:

```yaml
plugins:
  detekt:
    enabled: true
    useTypeResolution: true
    configFile: detekt.yml
    ...
```

## 8. Release plugin — also publish legacy `release.properties`

If the application source already reads its version from a classpath resource like `release.properties` (key=value format), preserve that contract by adding a sibling `@TaskAction` to the release plugin that writes the same format. Source code stays untouched.

### `plugins/release/src/tasks/WriteReleaseProperties.kt`

```kotlin
package io.example.release.tasks

import io.example.release.Settings
import io.example.release.git.GitRepo
import io.example.release.version.VersionPipeline
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

/**
 * Companion to [writeVersion] that publishes the same version in the legacy
 * `release=<version>` properties shape, for apps that already read this file
 * via `java.util.Properties.load(getResourceAsStream("release.properties"))`.
 */
@OptIn(ExperimentalPathApi::class)
@TaskAction(executionAvoidance = ExecutionAvoidance.Disabled)
fun writeReleaseProperties(
    @Input(inferTaskDependency = false) moduleRootDir: Path,
    @Output outputDir: Path,
    settings: Settings,
) {
    val pipeline = VersionPipeline(settings)
    val inferred = GitRepo.open(moduleRootDir, settings.repoDir).use { pipeline.infer(it) }

    outputDir.deleteRecursively()
    val propertiesFile = outputDir / "release.properties"
    propertiesFile.createParentDirectories()
    propertiesFile.writeText("release=${inferred.version}\n")
}
```

### Register in `plugins/release/plugin.yaml`

```yaml
tasks:
  writeVersion:
    action: !io.example.release.tasks.writeVersion
      moduleRootDir: ${module.rootDir}
      outputDir: ${taskOutputDir}
      settings: ${pluginSettings}

  writeReleaseProperties:
    action: !io.example.release.tasks.writeReleaseProperties
      moduleRootDir: ${module.rootDir}
      outputDir: ${taskOutputDir}
      settings: ${pluginSettings}

generated:
  resources:
    - directory: ${tasks.writeVersion.action.outputDir}
    - directory: ${tasks.writeReleaseProperties.action.outputDir}
```

Neither task is in `commands:` — they're build-graph contributors that run automatically when something downstream needs the resources.

## 9. The release plugin and the root module

When the release plugin's tasks take an `@Input moduleRootDir: Path`, and the consumer module is the **project root** (so `${module.rootDir}` is the same directory that contains `build/`), Toolchain's dependency inference may treat every build output as an input of those tasks. The result is a "Task dependency loop detected" error during `./kotlin show modules` or `./kotlin build`.

Patch every release-plugin task that takes `moduleRootDir`:

```kotlin
@TaskAction
fun currentVersion(
    @Input(inferTaskDependency = false) moduleRootDir: Path,   // ← inferTaskDependency = false
    settings: Settings,
) { /* ... */ }
```

The `inferTaskDependency = false` opt-out tells Toolchain "this Path is a configuration input, not a directory whose contents drive task dependencies". Apply it to every task in the plugin that takes a directory path that overlaps with the build tree.

## 10. CI workflow — `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches-ignore: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    env:
      KOTLIN_CLI_NO_WELCOME_BANNER: "1"

    steps:
    - uses: actions/checkout@v6
      with:
        fetch-depth: 0  # Needed for the release plugin to determine the version from git tags

    - name: Build with Kotlin Toolchain
      run: |
        ./kotlin build
        ./kotlin check
        ./kotlin do package

    - name: Upload build artifacts
      uses: actions/upload-artifact@v7
      with:
        name: build-artifacts
        path: build/libs/
```

`setup-java` is gone (the `./kotlin` wrapper auto-provisions its JDK). The artifact path is the stable `build/libs/` that the `package` plugin produces.

## 11. CI workflow — `.github/workflows/release.yml`

```yaml
---
name: Release

on:
  push:
    branches:
      - main

concurrency:
  group: release-main
  cancel-in-progress: true

permissions:
  contents: write  # Needed to push Git tags

jobs:
  build:
    name: "Release"
    runs-on: ubuntu-latest
    env:
      KOTLIN_CLI_NO_WELCOME_BANNER: "1"
    steps:
      # Log in to whatever container registry the image plugin pushes to.
      # (Swap this step for your provider's login action / CLI.)
      - name: Log in to container registry
        run: echo "$REGISTRY_TOKEN" | docker login "$REGISTRY_HOST" -u "$REGISTRY_USER" --password-stdin
        env:
          REGISTRY_HOST: ${{ secrets.REGISTRY_HOST }}
          REGISTRY_USER: ${{ secrets.REGISTRY_USER }}
          REGISTRY_TOKEN: ${{ secrets.REGISTRY_TOKEN }}

      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - name: Verify
        run: |
          ./kotlin clean
          ./kotlin check
          ./kotlin test

      - name: Tag release
        run: ./kotlin do release

      - name: Resolve current release version
        id: tag_version
        run: |
          # Read the tag the release just created — do NOT parse `./kotlin do currentVersion`
          # stdout (it is a decorated log line and ends with a `<task> successful` banner).
          current_version=$(git describe --tags --exact-match HEAD | sed 's/^v//')
          echo "Version set to: $current_version"
          echo "version=$current_version" >> $GITHUB_OUTPUT

      - name: Build and publish Docker image
        run: ./kotlin do jib
```

Notes:

- Resolve the released version from a side channel, not `./kotlin do <cmd>` stdout — the toolchain wraps task output in a log line and appends a `<task> successful` banner, so `| tail -n1` captures the banner, not the value. Read the git tag (above) or have the plugin write the version to a file named by an env var and `cat` it.
- Dynamic tags for the image are pushed into the image plugin via env vars consumed inside the task action (Toolchain has no `-Pkey=value` for CLI overrides). Document the env var in the plugin's README.

## 12. `.github/dependabot.yml` — group new local-plugin deps

The migration adds new runtime libraries (jib-core, jgit, bouncycastle, slf4j-nop, detekt-cli, ktlint-cli) that didn't exist as catalog entries before. Group them so Dependabot consolidates PRs:

```yaml
version: 2
updates:
  - package-ecosystem: "gradle"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 5
    groups:
      ktor:
        patterns: ["io.ktor*"]
      arrow:
        patterns: ["io.arrow-kt*"]
      testing:
        patterns: ["io.kotest*", "io.mockk*"]
      exposed:
        patterns: ["org.jetbrains.exposed*"]
      detekt:
        patterns: ["io.gitlab.arturbosch.detekt*", "com.github.marc0der*"]
      ktlint:
        patterns: ["com.pinterest.ktlint*"]
      jib:
        patterns: ["com.google.cloud.tools*"]
      release-plugin:
        patterns: ["org.eclipse.jgit*", "org.bouncycastle*", "org.slf4j*"]

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 5
```

Drop any pre-existing groups whose patterns no longer match the catalog (e.g. a `kotlin: org.jetbrains.kotlin*` group becomes dead if the migration removed Kotlin coordinates from the catalog in favour of toolchain-managed versions).

## 13. `.gitignore`

Replace any Gradle-specific lines with the Toolchain build directory:

```
# Kotlin Toolchain
build/
```

The Toolchain also writes to `~/Library/Caches/JetBrains/Kotlin/` (macOS) / `~/.cache/JetBrains/Kotlin/` (Linux) — those are outside the repo and don't need ignoring.

## 14. PR description scaffold

```markdown
## Summary

Replaces the Gradle build with the Kotlin Toolchain (Amper engine), driven by
`./kotlin build / test / check / do <command>` via the bundled wrapper.

- Single root config — module.yaml + project.yaml at the repo root, layout maven-like
  so no source files move. The existing gradle/libs.versions.toml is reused verbatim
  as the Toolchain's `$libs.*` catalog.
- N local plugins (plugins/{release,jib,detekt,ktlint,package}/) reimplementing the
  Gradle plugins they replace, per the kotlin-toolchain guidance "reimplement instead
  of adapt".
  - release/ — replaces axion-release. JGit-based; derives the version from git tags.
  - jib/ — replaces Gradle jib plugin; wraps jib-core directly.
  - detekt/ — runs detekt-cli in a subprocess; useTypeResolution defaults to false to
    match Gradle's default detekt task behaviour.
  - ktlint/ — runs ktlint-cli in a subprocess.
  - package/ — stages the application JAR at build/libs/<name>.jar for CI uploads.
- CI workflows rewritten to call ./kotlin directly. The setup-java step is gone; the
  ./kotlin wrapper auto-provisions its own JDK.
- build.gradle.kts retained as an empty stub so Dependabot continues to find the
  catalog (see file comment for details).

## What's preserved

- gradle/libs.versions.toml — single source of truth for all versions, reused as-is.
- detekt.yml, .editorconfig, docker-compose.yml, all source under src/ — unchanged.

## What replaces what

| Old | New |
|---|---|
| ./gradlew build | ./kotlin build |
| ./gradlew test | ./kotlin test |
| ./gradlew check | ./kotlin check |
| ./gradlew run | ./kotlin run |
| ./gradlew currentVersion | ./kotlin do currentVersion |
| ./gradlew release | ./kotlin do release |
| ./gradlew jib | ./kotlin do jib |
| ./gradlew ktlintFormat | ./kotlin do ktlintFormat |

## Test plan

- [x] ./kotlin show modules — clean, all expected modules listed
- [x] ./kotlin clean && ./kotlin build — succeeds
- [x] ./kotlin do currentVersion — prints git-derived version
- [x] ./kotlin do ktlintFormat — succeeds
- [x] ./kotlin task :<module>:analyze@detekt — clean checkstyle report
- [x] ./kotlin do jibBuildTar — produces OCI image tar
- [ ] ./kotlin test (testcontainers) — runs on CI ubuntu-latest only

## Notes / follow-ups

- mongo:3.2 is amd64-only and crashes on Apple Silicon under Rosetta. Pre-existing host
  incompatibility unchanged by this PR; CI on ubuntu-latest is unaffected.
- ...
```
