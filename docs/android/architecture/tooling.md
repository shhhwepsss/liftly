# Project Tooling

- **Build:** Gradle KTS (`build.gradle.kts` throughout). Version catalog at `gradle/libs.versions.toml` for all dependency versions.
- **Annotation processing:** KSP for both Room (`room-compiler`) and Hilt (`hilt-android-compiler`).
- **Compose compiler:** Kotlin Compose compiler plugin version pinned to match the Kotlin version in `libs.versions.toml`. Compose compiler metrics enabled in CI to catch stability regressions on the workout screen.
- **Static analysis:** Detekt with the default rule set plus custom rules: no hardcoded strings in UI files, no direct DAO access from ViewModel, `ForbiddenImport` blocking `android.*` in the `evaluator` package, `NoRawWeightFormatting` blocking direct kg/lb string interpolation in UI.
- **Formatting:** ktlint via the Gradle plugin (`jlleitschuh/ktlint-gradle`), enforced in CI pre-merge.
- **Minimum SDK:** API 26 (Android 8.0). Rationale: Credential Manager `GetGoogleIdOption` requires Play Services and works reliably from API 23+, WorkManager from API 14, and modern `androidx.security` patterns assume API 23+. API 26 is the lowest practical floor for ~98% device coverage while keeping Java 8 APIs available without desugaring.
- **Target SDK:** API 35 (Android 15, latest stable).
- **Compile SDK:** API 35.
