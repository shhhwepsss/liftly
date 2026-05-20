# Theming, Formatting, i18n & Navigation

## Material3 theme (`core/ui/theme/`)
- `LiftlyTheme.kt` wraps `MaterialTheme` with custom `ColorScheme` for both dark and light palettes.
- Theme selection reads from `SettingsRepository` (DataStore). `LiftlyTheme` accepts a `darkTheme: Boolean` parameter driven by `isSystemInDarkTheme()` or the user's explicit preference from settings.
- **Dynamic color (Material You):** disabled at MVP. Dynamic color produces unpredictable contrast on gym-bright screens and requires API 31+. A static color palette is the correct call for a fitness app where readability under bright light matters. Can be opt-in post-MVP.
- Typography and shape tokens are defined once in `LiftlyTheme` and never overridden at the composable level — no hardcoded `fontSize` or `Color` anywhere in UI code.

## Formatting layer (`core/ui/format/`)

All persistence is canonical: weights in kg, timestamps as UTC `Instant`. Conversion to the user's preferred display happens in one place.

- **`WeightFormatter`** — reads Display Unit from `SettingsRepository`. Exposes `formatKg(weightKg: Double): String` returning e.g. `"60 kg"` or `"132 lb"`. Includes `formatRange(lowerKg, upperKg)` and `formatIncrement(deltaKg)`.
- **`TimestampFormatter`** — exposes `formatTime(instant: Instant): String`, `formatRelativeDay(instant: Instant): String` ("Today", "Yesterday", "Mon 12 May"), `formatDuration(seconds: Long): String` for the Rest Timer.
- Composables call the formatters via injected references (Hilt-provided through `FormatModule`). ViewModels never format — `UiState` holds raw kg / `Instant` values.
- A custom Detekt rule (`NoRawWeightFormatting`) flags any `"$weight kg"` string template or direct `Instant.toString()` in `*/ui/**` files at lint time.

## i18n
- All strings in `res/values/strings.xml` and `res/values-{locale}/strings.xml`. Zero hardcoded text in Compose code — Detekt rule `ForbiddenComment` configured to flag string literals in UI files.
- `stringResource()` is the only permitted string access in Composables.
- Locale override (user picks a language different from device locale) is handled by wrapping the `Activity` context with `Locale`-overridden `Configuration`. The setting is stored in DataStore; `MainActivity.onCreate` applies it before `setContent {}`.
- Locale switching restarts `MainActivity` (via `recreate()`) — the simplest correct solution; the overhead is acceptable since it is a deliberate user action.

---

## Navigation

**Decision: Compose Navigation (`androidx.navigation.compose` 2.8+) with native type-safe routes via `@Serializable` route classes.**

Since 2.8 (released 2024), Jetpack Navigation supports type-safe routes using kotlinx.serialization data classes — no codegen, no third-party dependency, and arguments flow end-to-end with full type safety. This replaces the earlier proposal of hand-rolled sealed-class route wrappers. Compose Destinations, Voyager, and Decompose are no longer needed.

Route structure (each route is a `@Serializable` data class or object):

```
AppNavGraph
├── AuthGraph
│   └── SignInRoute
├── MainGraph (bottom nav: Home / History / Notifications / Settings)
│   ├── HomeRoute                       — Routine list + Streak chip (read from /me bootstrap cache)
│   ├── RoutineDetailRoute(routineId)
│   ├── RoutineEditRoute(routineId?)
│   ├── WorkoutGraph
│   │   ├── ActiveWorkoutRoute(workoutId)   — auto-resumed on launch when an in-progress WorkoutEntity exists
│   │   └── WorkoutSummaryRoute(workoutId)
│   ├── HistoryRoute
│   │   └── WorkoutDetailRoute(workoutId)
│   │       └── ExerciseHistoryRoute(exerciseId)   ← deep link target; PR badges rendered next to qualifying Sets
│   ├── NotificationCenterRoute
│   └── SettingsRoute
└── BodyweightGraph
    └── WeighInRoute
```

Deep links from FCM use the `liftly://` URI scheme routed to `ExerciseHistoryRoute` and `NotificationCenterRoute`. The `NavController` is hoisted to `MainActivity`; Composables receive `NavController` references only through the closest `NavHost` lambda scope, never as constructor parameters.

**Tablet support is out of scope at MVP** — no `WindowSizeClass` scaffolding. If a two-pane tablet layout is added later, the navigation graph will need restructuring; this is an accepted future cost in exchange for MVP simplicity.
