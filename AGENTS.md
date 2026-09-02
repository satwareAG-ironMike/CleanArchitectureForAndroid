# AGENTS.md

Clean Architecture for Android sample app ("WhoAmI"): Kotlin 2.3.0, AGP 9.3.1, Gradle 9.7.1, JDK 17, Compose, minSdk 23 / targetSdk 36.
This fork (`satwareAG-ironMike`) is satware AG's modern Android development base; upstream is EranBoudjnah/CleanArchitectureForAndroid (book sample). The fork intentionally diverges - do not sync blindly.

## Workflow

- `master` is protected: PRs only, 0 required approvals, no force-push, no deletion, linear history (merge via squash or rebase). Work in `feat/<scope>` branches.
- CI on PRs: "Android CI" (jar, unit tests, connected tests on emulator api-35) and "Kotlin-Linter" (ktlint on changed `.kt` files). All required runs must conclude green.
- A physical Android device is attached to this workstation for integration testing. Verify instrumentation claims on it (`adb devices` first) before reporting done; the emulator config in CI is the fallback.

## Commands

| Task | Command |
|------|---------|
| Build | `./gradlew jar` |
| All unit tests | `./gradlew test` |
| One module | `./gradlew :home:domain:test` |
| One class | `./gradlew :home:domain:test --tests 'com.mitteloupe.whoami.home.domain.SomeTest'` |
| Style lint | `ktlint . '!**/build/**'` |
| Detekt | `./gradlew detekt` |
| Instrumentation APKs | `./gradlew assembleEspresso assembleEspressoAndroidTest` |

Verification order for code changes: `ktlint` -> unit tests -> device instrumentation.

## Gotchas

- **Instrumentation tests use the `espresso` build type, not debug.** `app/build.gradle.kts` sets `testBuildType = "espresso"`, so `connectedDebugAndroidTest` does not run them. Full device flow (install both APKs, run `SmokeTests` via `HiltTestRunner`, pull failure screenshots): `.github/workflows/script/run_tests.sh`.
- **The pre-commit hook runs standalone `ktlint` from PATH**, not the Gradle plugin. It is installed by the Gradle `installGitHook` task wired to `:app:preBuild` - run one Gradle build after a fresh clone, and keep a `ktlint` binary on PATH.
- **Konsist architecture tests are unit tests** in `app/src/test/java/com/mitteloupe/whoami/architecture/`. Layer violations fail `./gradlew test`, not just review.
- **Kotlin 2.3.0 + AGP 9.3.1 is above Google's listed compatibility range** (Kotlin 2.3 lists AGP 8.2.2-8.13). It works here, but verify both sides together before bumping either - see developer.android.com/build/kotlin-support.
- `org.gradle.parallel=false` in `gradle.properties` is deliberate. Do not flip it silently.
- JDK 17 everywhere (Kotlin `jvmTarget = 17`); the foojay resolver in `settings.gradle.kts` can provision it.

## Architecture

Layers: UI -> Presentation -> Domain -> Data -> Data Source. Dependencies point inward; Konsist enforces this.

- Features are vertical slices: `home/` and `history/`, each split into `ui`, `presentation`, `domain`, `data` modules.
- `architecture:{ui,presentation,domain}` hold layer contracts and base classes; `architecture:presentation-test` holds presentation test utilities.
- `architecture:instrumentation-test` holds shared device-test rules: `WebServerRule` (MockWebServer), `HiltInjectorRule`, `ScreenshotFailureRule`, `LocalStoreRule`, `SdkAwareGrantPermissionRule`.
- `datasource:{architecture,source,implementation}` - data source contracts and implementations.
- Typesafe project accessors are enabled: use `projects.home.domain`, never string paths like `"//home:domain"`.
- DI: Hilt via KSP. Navigation: androidx.navigation3. State: Kotlin Flow, no LiveData.

## Deliberate choices - do not "modernize" away

The README "Choices" section is authoritative.

- Pure ViewModels + Flows. No Google Architecture Components / LiveData - they leak Android into Presentation and push state persistence onto the ViewModel.
- Mappers are injected classes, not extension functions (Mockito cannot stub statics; injection keeps testability).
- Both Mockito-Kotlin and MockK are present deliberately to demonstrate each. Prefer Mockito-Kotlin for new tests.
- Fakes were evaluated and rejected as too verbose and expensive to maintain.

## Style

- `.editorconfig` is authoritative: ktlint `android_studio` style, 100-char lines, 4-space indent, no trailing commas.
- detekt is configured from the shared root `detekt.yml`.
- All dependency/plugin versions live in `gradle/libs.versions.toml` - never hardcode versions in module build files.

## Modernization direction (2026)

Ranked from current-state research (Sep 2026). Validate each on the device before claiming a win.

1. Baseline Profiles + Macrobenchmark for cold start and navigation paths.
2. Convention plugins via an included `build-logic` composite build (today `buildSrc` holds a single `project-java-library.gradle.kts`).
3. Version bumps (Kotlin/AGP/Compose BOM) in lockstep, checked against the Google Kotlin-AGP table every time.
4. Configuration cache + Android lint fatal checks wired into CI.
