---
name: kotlin-toolchain
description: How to build, run, test, package, lint/check/verify, manage dependencies, and configure Kotlin/Java projects with Kotlin Toolchain — JetBrains' unified Kotlin CLI (formerly Amper, now the engine inside the Toolchain). TRIGGER when the repo contains `project.yaml`, `module.yaml`, or a `./kotlin` wrapper; when the user asks to build/run/test/package a Kotlin project, add or remove dependencies, run lint/check/verification, or otherwise configure a Kotlin project that doesn't use Gradle/Maven; or when the user references Kotlin Toolchain or Amper. SKIP for Gradle/Maven Kotlin projects.
---

# Kotlin Toolchain

JetBrains' unified CLI entry point for Kotlin (JVM, Android, iOS, multiplatform) and Java projects, announced at KotlinConf'26 (May 2026, currently in Alpha). Uses declarative YAML configuration instead of Gradle build scripts. Amper, previously a standalone build tool, is now the build engine inside Kotlin Toolchain — the YAML schema (`project.yaml`, `module.yaml`) is unchanged.

## Installation

Install the CLI once; the `kotlin` command is then on `PATH` and auto-provisions the JDK on first use.

```sh
# SDKMAN (macOS / Linux / WSL)
sdk install kotlintoolchain

# macOS / Linux
curl -fsSL https://kotl.in/install.sh | sh

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm 'https://kotl.in/install.ps1' | iex"
```

When the global `kotlin` command runs inside a project that ships local wrapper scripts (`kotlin` / `kotlin.bat` in the project root), it detects them and proxies the call into the wrapper. This pins the project to the wrapper's Kotlin Toolchain version regardless of which version is installed on the developer's machine. **Always invoke `kotlin` from the project root** so the wrapper is picked up; never bypass it by calling a globally-installed binary directly when a wrapper exists.

## CLI Commands

```sh
kotlin init                  # Create a new project from templates
kotlin build                 # Compile and link all code
kotlin run                   # Run the application
kotlin test                  # Run all tests
kotlin check                 # Run tests + all registered checks (lint, API verification, etc.)
kotlin clean                 # Remove build output and caches
kotlin show modules          # List project modules
kotlin show dependencies     # Show dependency tree
kotlin show checks           # List registered checks
kotlin update                # Update Kotlin Toolchain to the latest version
kotlin generate-completion   # Create shell completion scripts
```

All commands support `-h`/`--help` for detailed options.


## Project Structure

```
project-root/
├── kotlin, kotlin.bat     # Optional local wrappers (global `kotlin` proxies into them when present)
├── project.yaml           # Project-level config (toolchain versions, etc.)
├── libs.versions.toml     # Version catalog (Gradle-compatible)
├── module-name/
│   ├── module.yaml        # Module configuration
│   ├── src/               # Production sources (Kotlin + Java mixed)
│   ├── resources/         # Resources (copied into JAR)
│   ├── test/              # Test sources
│   └── testResources/     # Test-only resources
└── another-module/
    ├── module.yaml
    └── ...
```

For single-module projects, `module.yaml`, `src/`, and `test/` live at the project root.

## Configuration Files

### module.yaml

Central config per module. Key sections:

```yaml
product: jvm/app    # Product type: jvm/app, jvm/lib, android/app, lib (multiplatform), etc.

dependencies:
  - org.example:artifact:1.0.0           # Maven coordinates
  - ./other-module                        # Module dependency (relative path)
  - $libs.ktor.client                     # From version catalog
  - bom: io.ktor:ktor-bom:2.2.0          # BOM import
  - org.example:foo:1.0.0: exported      # Exposed to dependents (like Gradle api())
  - org.example:bar:1.0.0: compile-only  # Compile-only scope
  - org.example:baz:1.0.0: runtime-only  # Runtime-only scope

test-dependencies:
  - io.mockk:mockk:1.13.0

settings:
  jvm:
    mainClass: org.example.MainKt   # Entry point (default: main() in main.kt)
    jdk:
      version: 21
  kotlin:
    languageVersion: 2.0
  compose:
    enabled: true

test-settings:
  kotlin:
    languageVersion: 2.0

repositories:
  - https://maven.pkg.jetbrains.space/public/p/compose/dev
```

### project.yaml

Project-level settings at the root (toolchain versions, shared settings across modules).

### libs.versions.toml

Standard Gradle version catalog format. Can be at root or in `gradle/` directory. Referenced in module.yaml as `$libs.<key>`. Built-in toolchain catalogs: `$kotlin.*`, `$compose.*` — versions derived from settings.

## Testing

- Default framework: [kotlin.test](https://kotlinlang.org/api/latest/kotlin.test/) (preconfigured — no extra dependency needed)
- Test sources live in `test/`
- Test dependencies go in `test-dependencies:`
- Test-specific settings go in `test-settings:`

## Checks and Linters

`kotlin check` runs all tests plus every registered check. Filter with named checks (`kotlin check detekt apiCheck`), skip with `--skip <name>` (e.g. `--skip tests`), or restrict to modules with `-m <module>` (repeatable). Use `kotlin show checks` to list what's registered. A check fails when its underlying task throws.

Kotlin Toolchain ships **no bundled linters** — `tests` is the only built-in check. Tools like detekt, ktlint, or API-compatibility verification are not preinstalled: register them as local-plugin tasks under `checks:` in `plugin.yaml`. Do not invoke linter binaries directly or wire in a Gradle plugin — both bypass the toolchain's check pipeline.

## Multiplatform

Platform-specific code uses `@platform` directory suffixes: `src@jvm/`, `src@ios/`, `src@android/`, etc. Common code in `src/` is visible to all platform-specific directories, but not vice versa.

Platform-specific dependencies and settings use the `@platform` qualifier:

```yaml
dependencies@android:
  - androidx.core:core-ktx:1.12.0
```

## iOS apps

For an `ios/app` module, Toolchain generates and manages the Xcode project. On first build, if no Xcode project exists, it creates `module.xcodeproj` (target `app`) beside the module and writes a **complete** default `Info.plist`. It then points the `INFOPLIST_FILE` build setting at that plist and uses it **verbatim** — it does **not** enable Xcode's `GENERATE_INFOPLIST_FILE`, so the `Info.plist` must be self-contained.

Two consequences worth knowing:

- **A pre-existing `Info.plist` is used as-is and never completed.** Toolchain writes its default plist *only* when no `Info.plist` exists. If you supply your own, it must itself contain the required `CFBundle*` keys (`CFBundleIdentifier`, `CFBundleExecutable`, `CFBundleName`, …). A partial plist produces an `.app` with no bundle id, and the simulator refuses to install it:

  ```
  Simulator device failed to install the application. Missing bundle ID.
  ```

  A fresh `kotlin init` iOS app never hits this (the generated default plist is complete). It typically bites when migrating a project that already had an `Info.plist` — see the `gradle-to-kotlin-toolchain-project` skill for the Gradle/KMP case.

- **The generated project is named `module.xcodeproj` (target `app`) and is only created when absent** — it is not regenerated when `module.yaml` settings change. Delete it to force a clean regeneration.

## Plugins

If a behavior is not natively supported by the declarative YAML, use Kotlin Toolchain's **local plugin** system to extend the build — this is the escape hatch for custom build logic.

Do **not** try to reuse or adapt a Gradle plugin inside a Kotlin Toolchain project. If an equivalent Gradle plugin exists for what you need, reimplement the behavior as a Kotlin Toolchain local plugin instead.

When a library's standard workflow includes a build-time processing step (code generation, schema compilation, resource transformation, etc.), implement that step as a local plugin. Do not bypass or skip processing by hand-writing the would-be-generated code, and do not use the library in a degraded/runtime-only mode. Preserve the library's intended workflow.

## Build Tool Policy

Treat Kotlin Toolchain as a fixed project requirement. Do not propose switching to Gradle or re-open the Kotlin Toolchain/Gradle tradeoff because a library is more commonly used with Gradle. When build-time processing is needed, implement it within the Kotlin Toolchain workflow. Keep discussion focused on the chosen approach rather than alternative build systems, unless the user explicitly asks.

## Key Conventions

- Kotlin and Java sources can be mixed freely in the same `src/` folder
- Default entry point: `main()` in `main.kt` (compiles to `MainKt` class)
- Top-level functions in `myFile.kt` compile to class `MyFileKt`
- `exported` dependencies expose types to downstream modules; only mark `exported` if your public API uses those types

## Common Pitfalls

- **Don't run `gradle ...`** in a Kotlin Toolchain project — there is no `build.gradle(.kts)`. Use `kotlin ...`.
- **Don't hand-write code that a build-time generator should produce.** Add a local plugin instead.
- **Don't pin the JDK manually** outside `settings.jvm.jdk.version` — the toolchain provisions it.
- **Don't add `compose:` settings** to a module that doesn't actually use Compose; only enable it where needed.
- **The CLI is `kotlin`, not `kotlin-toolchain` or `amper`.** When wrapper scripts exist locally, `./kotlin` and the global `kotlin` from the project root behave identically (the latter proxies to the former).

## References

- Docs: <https://kotlin-toolchain.org/dev/>
- Source: <https://github.com/JetBrains/kotlin-toolchain> (the old `JetBrains/amper` repo is archived)
- Issue tracker: YouTrack project `AMPER` (still named after the build engine)
