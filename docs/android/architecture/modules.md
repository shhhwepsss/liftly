# Module Layout

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
