# Layering

## Layer responsibilities

| Layer | Location | Responsibility |
|---|---|---|
| UI | `*/ui/**` | Render `UiState`, emit `Intent`, no business logic |
| ViewModel | `*/ui/**/*ViewModel.kt` | Reduce intents to state, coordinate use cases, hold `StateFlow` |
| Use Case | `*/domain/*UseCase.kt` | Single-operation orchestration; optional but used where ViewModel logic would otherwise be fat |
| Repository | `*/data/*Repository.kt` | Abstract the cache/network boundary; returns domain models |
| DAO / API Service | `*/data/` | Room DAOs and Retrofit interfaces; never referenced above the Repository |

Use cases are adopted selectively, not universally. `LogSetUseCase` earns its existence: it writes to the local queue, runs the Trigger Rule evaluator, and returns a `ModalTrigger?` — three responsibilities that the ViewModel should not inline. `GetRoutinesUseCase` is probably just `routineRepository.getRoutines()`; skip the use case wrapper there.

## Read path

```
ActiveWorkoutScreen
  → collectAsStateWithLifecycle(viewModel.uiState)
  ← ActiveWorkoutViewModel holds StateFlow<ActiveWorkoutUiState>
    ← WorkoutRepository.observeActiveWorkout(): Flow<WorkoutWithSets>
      ← Room DAO emits updates reactively
```

The ViewModel never calls the network directly. The Repository decides whether to serve from Room cache or fetch from the network. During an active Workout the ViewModel reads exclusively from Room — the network path is irrelevant until flush.

## Write path (Set log)

```
User taps "Log Set"
  → ActiveWorkoutScreen emits LogSetIntent(reps, weight)
  → ActiveWorkoutViewModel calls LogSetUseCase(set)
    → LogSetUseCase:
        1. Insert SetQueueEntity into Room (queued_sets table)
        2. Call TriggerRuleEvaluator.evaluate(setsSoFar, ruleConfig)
        3. If ModalTrigger returned AND not already fired this Workout: emit ModalTriggerEvent
  → ViewModel updates UiState (set list grows, modal flag flipped if needed)
  → Compose recomposes; if modal flag set, IncreaseModal / DecreaseModal renders as bottom sheet
```

The Set is persisted locally before the ViewModel returns. The UI never waits for a network call. This is the ≤ 5-second tap target path.

## Compose recomposition strategy

- Every screen consumes exactly one `StateFlow<UiState>` via `collectAsStateWithLifecycle`.
- One-shot events (navigation, snackbars, modal triggers) are delivered via a separate `Channel<UiEvent>` consumed with `LaunchedEffect` — not baked into the state snapshot. This avoids the "re-show modal on recomposition" bug.
- `UiState` is a `data class` annotated `@Immutable`; all list fields use `ImmutableList` (kotlinx.collections.immutable). `@Immutable` is a contract that the Compose compiler relies on for skipping recomposition when input references are unchanged — `data class` alone does not imply it.
