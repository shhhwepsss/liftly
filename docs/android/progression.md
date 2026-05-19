# Trigger Rule Evaluator Placement

**Location:** `workout/evaluator/TriggerRuleEvaluator.kt`

**API surface:**

```kotlin
// Pure function — zero Android or Coroutine imports
object TriggerRuleEvaluator {
    /**
     * @param sets the Sets logged so far in the current Workout for ONE Exercise, in order
     * @param triggerRule / deloadRule the rule config for THAT Exercise
     * @param modalsFiredThisWorkout modals already shown for THAT Exercise in THIS Workout —
     *        enforces the "at most one Increase and one Decrease per Exercise per Workout" invariant
     *        without the evaluator owning any state
     */
    fun evaluate(
        sets: List<SetSnapshot>,
        triggerRule: TriggerRuleConfig,
        deloadRule: DeloadRuleConfig,
        modalsFiredThisWorkout: Set<ModalKind>  // INCREASE, DECREASE
    ): ModalTrigger?  // null = no modal
}

data class SetSnapshot(val reps: Int, val weightKg: Double, val position: Int)
data class TriggerRuleConfig(val variant: String, val setPosition: Int, val threshold: Int)
data class DeloadRuleConfig(val variant: String, val setPosition: Int, val threshold: Int)
sealed class ModalTrigger { object Increase : ModalTrigger(); object Decrease : ModalTrigger() }
```

The evaluator mirrors the server's `rule-evaluator.ts` logic exactly: check the configured `setPosition` (0 = first, -1 = last) against the threshold, return the appropriate trigger.

**Unknown variant strings** fall through to a no-op (no modal fires). The server re-evaluates on flush and the canonical outcome is whatever the server decides. The architectural prevention for client/server divergence is the post-MVP force-update strategy that keeps all installed clients within a supported version window — see [[project-liftly-force-update]].

**Testing:** parameterized JVM unit tests in `workout/evaluator/TriggerRuleEvaluatorTest.kt`. No `@RunWith(AndroidJUnit4)`, no instrumentation. Test cases cover: trigger fires on first Set at upper bound, does not fire below upper bound, deload fires on last Set below lower bound, modal already fired (no double-fire), edge case of single Set in set list. These run in under 1 second on any JVM.
