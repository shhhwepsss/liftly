# Liftly — Android Architecture

**Status:** Draft for implementation
**Stack:** Kotlin + Jetpack Compose + Android SDK
**Last updated:** 2026-05-19 (revised after open-questions review)

## Contents

1. [Architectural style & rationale](#architectural-style--rationale)
2. [Module layout](#module-layout)
3. [Layering](./layering.md)
4. [Local persistence](./persistence.md)
5. [Sync engine (incl. rest timer)](./sync.md)
6. [Trigger rule evaluator placement](./progression.md)
7. [Auth & session](./auth.md)
8. [Notification handling](./notifications.md)
9. [Networking layer](./networking.md)
10. [Dependency injection](./di.md)
11. [Theming, formatting, i18n & navigation](./ux.md)
12. [Testing strategy](./testing.md)
13. [Project tooling](./tooling.md)
14. [Open questions & cross-doc follow-ups](./open-questions.md)

---

## Architectural Style & Rationale

**Recommendation: MVI (Model-View-Intent) with StateFlow.**

MVI wins over plain MVVM-with-StateFlow for Liftly because the workout flow has multiple concurrent, event-driven concerns happening simultaneously: Set logging, Trigger Rule evaluation, Increase/Decrease Modal gating, Rest Timer, and an offline queue that can flush under the UI's feet. In MVVM the ViewModel accumulates side-channel `LiveData`/`SharedFlow` outputs that interact in non-obvious ways; MVI forces a single `UiState` sealed hierarchy per screen, which makes the modal-firing invariant ("at most one Increase Modal and one Decrease Modal per Exercise per Workout") explicit in state rather than hidden in boolean flags. The unidirectional data flow (UI emits `Intent` → ViewModel reduces to new `State` → Compose recomposes) also means the sync engine can deliver `WorkoutSyncResult` events as intents without the ViewModel having to coordinate between multiple mutable flows. Compose-state-only (no ViewModel) is ruled out: the offline queue and flush lifecycle need a component that survives recomposition. Decompose/Voyager add a navigation runtime dependency that buys nothing at MVP single-platform scale.

Each screen has:
- A `UiState` data class (sealed when multiple logical states exist, plain data class otherwise).
- A `ViewModel` that exposes `StateFlow<UiState>` and accepts `Intent` sealed class instances.
- Compose screens observe via `collectAsStateWithLifecycle()`.

---

## Module Layout

**Decision: single-module package layout at MVP, with internal package boundaries that mirror a future multi-module split.**

Multi-module Gradle is valuable when build times hurt, team size creates merge conflicts across features, or dynamic feature delivery is needed. None of these apply to a friends-only MVP with one developer. Premature modularization adds Gradle configuration overhead and forces artificial API surface decisions before the domain is stable. The package boundaries below are drawn to enable extraction with minimal refactoring when the time comes.

```
app/
└── src/main/kotlin/com/liftly/
    ├── core/
    │   ├── db/               # Room setup, Database class, migrations
    │   ├── network/          # OkHttp + Retrofit setup, interceptors
    │   ├── sync/             # SyncEngine, FlushWorker, offline queue logic
    │   ├── auth/             # Token storage (DataStore + Tink), refresh interceptor, AuthRepository
    │   ├── di/               # Hilt modules
    │   └── ui/
    │       ├── theme/        # Material3 theme, dark/light, typography, colors
    │       ├── format/       # WeightFormatter (kg↔Display Unit), TimestampFormatter (UTC→local)
    │       └── components/   # Shared Composable primitives (buttons, sheets, etc.)
    │
    ├── home/
    │   └── ui/               # HomeScreen (Routine list + Streak chip), HomeViewModel
    │
    ├── workout/
    │   ├── data/             # WorkoutRepository, WorkoutDao, SetDao, WorkoutApiService
    │   ├── domain/           # StartWorkoutUseCase, LogSetUseCase, CompleteWorkoutUseCase
    │   ├── evaluator/        # TriggerRuleEvaluator (pure Kotlin, no Android imports)
    │   ├── timer/            # RestTimerController (ViewModel coroutine + AlarmManager backstop)
    │   └── ui/
    │       ├── active/       # ActiveWorkoutViewModel, ActiveWorkoutScreen, SetLoggerSheet, RestTimerOverlay
    │       ├── history/      # HistoryViewModel, HistoryScreen, WorkoutDetailScreen, ExerciseHistoryScreen (with PR badges)
    │       └── summary/      # WorkoutSummaryScreen
    │
    ├── routine/
    │   ├── data/             # RoutineRepository, RoutineDao, RoutineApiService
    │   ├── domain/           # GetRoutinesUseCase, SyncRoutinesUseCase
    │   └── ui/               # RoutineDetailScreen, RoutineEditScreen
    │
    ├── progression/
    │   ├── data/             # ProgressionRepository, ProgressionEventDao
    │   └── ui/               # IncreaseModal, DecreaseModal (Composable bottom sheets)
    │
    ├── bodyweight/
    │   ├── data/             # WeighInRepository, WeighInDao, WeighInApiService
    │   └── ui/               # WeighInScreen, BodyweightPromptDialog
    │
    ├── notification/
    │   ├── data/             # NotificationRepository, NotificationDao, NotificationApiService
    │   ├── fcm/              # LiftlyFirebaseMessagingService
    │   └── ui/               # NotificationCenterScreen, NotificationCenterViewModel
    │
    └── account/
        ├── data/             # AccountRepository, AccountApiService (owns /me bootstrap)
        └── ui/               # SettingsScreen, SettingsViewModel
```

**Package dependency rules (enforced by convention; Detekt rule if needed):**

```
core/sync          → workout/data, routine/data, bodyweight/data, progression/data, account/data
core/auth          → core/network
workout/domain     → workout/data, workout/evaluator
workout/ui         → workout/domain, progression/ui
workout/timer      → workout/data (read-only — exercise rest seconds)
routine/domain     → routine/data
notification/data  → (none outside core)
notification/fcm   → notification/data
account/data       → (none outside core; produces the bootstrap payload consumed by everyone)
home/ui            → account/data (Streak), routine/data, workout/data
```

No package imports `core/di` directly — Hilt wires everything. The `evaluator` subpackage has zero Android dependencies; it can be extracted to a pure JVM module at any time.
