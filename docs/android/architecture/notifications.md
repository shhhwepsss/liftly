# Notification Handling

**FCM service (`notification/fcm/LiftlyFirebaseMessagingService.kt`):**

Every FCM message carries a `notificationId` in its data payload (matching the server's `notifications.id`). On receive:

1. Upsert `MirroredNotificationEntity` into Room by `notificationId` (idempotent; the same Notification might arrive via both FCM and the `/notifications` poll).
2. The badge unread count is exposed by `NotificationBadgeRepository` as a `Flow<Int>` derived from a Room `@Query` on `notifications WHERE read_at_utc IS NULL`. Consumers convert to `StateFlow` at their subscription scope (e.g., `viewModelScope` via `.stateIn(scope, WhileSubscribed(5_000), 0)`).
3. If app is **foregrounded**: the `NotificationCenterViewModel` observes the Room flow and recomposes automatically. No system notification posted (app is already open).
4. If app is **backgrounded or killed**: post a system `NotificationCompat` notification with a deep link `PendingIntent`. FCM's `onMessageReceived` is always called for data-only messages (no `notification` block in the FCM payload — the client controls the display entirely, which is the correct pattern for reliable foreground/background parity).

**Notification Center read path:** `GET /notifications` is called on `NotificationCenterScreen` entry (not on every app open) and on pull-to-refresh.

- Server response is upserted into `MirroredNotificationEntity` by id.
- Read-state reconciliation runs at the same time: any local row with `readAtUtc IS NOT NULL` where the server still reports `read_at_utc = null` triggers a fire-and-forget `PATCH /notifications/:id/read`. This is the **catch-up sync for offline mark-as-read actions**; there is no dedicated pending-reads queue.

**Mark as read (single):** `PATCH /notifications/:id/read` is called optimistically — `readAtUtc` is written to Room before the HTTP call returns. On network failure, the local row stays marked read and the next `GET /notifications` reconciliation catches up.

**Mark all read:** `POST /notifications/read-all` is exposed as an action in the Notification Center overflow menu. The action sets `readAtUtc` on all local rows in a single Room transaction, then fires the POST. Same offline-tolerance pattern as single-read.

**Deep links from push notifications:** the `PendingIntent` uses a deep link URI (`liftly://exercise/{exerciseId}/history`) routed through the Compose NavHost. All deep link URIs are registered in `AndroidManifest.xml` with `<intent-filter>` for `liftly://` scheme.

If the target of a deep link no longer exists (Custom Exercise that the user has since deleted, etc.), navigation lands on the History root and a snackbar surfaces "That exercise no longer exists." This keeps the user on a useful screen and explains why they didn't reach the expected destination.
