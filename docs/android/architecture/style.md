# Architectural Style & Rationale

**Recommendation: MVI (Model-View-Intent) with StateFlow.**

MVI wins over plain MVVM-with-StateFlow for Liftly because the workout flow has multiple concurrent, event-driven concerns happening simultaneously: Set logging, Trigger Rule evaluation, Increase/Decrease Modal gating, Rest Timer, and an offline queue that can flush under the UI's feet. In MVVM the ViewModel accumulates side-channel `LiveData`/`SharedFlow` outputs that interact in non-obvious ways; MVI forces a single `UiState` sealed hierarchy per screen, which makes the modal-firing invariant ("at most one Increase Modal and one Decrease Modal per Exercise per Workout") explicit in state rather than hidden in boolean flags. The unidirectional data flow (UI emits `Intent` → ViewModel reduces to new `State` → Compose recomposes) also means the sync engine can deliver `WorkoutSyncResult` events as intents without the ViewModel having to coordinate between multiple mutable flows. Compose-state-only (no ViewModel) is ruled out: the offline queue and flush lifecycle need a component that survives recomposition. Decompose/Voyager add a navigation runtime dependency that buys nothing at MVP single-platform scale.

Each screen has:
- A `UiState` data class (sealed when multiple logical states exist, plain data class otherwise).
- A `ViewModel` that exposes `StateFlow<UiState>` and accepts `Intent` sealed class instances.
- Compose screens observe via `collectAsStateWithLifecycle()`.
