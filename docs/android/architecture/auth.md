# Auth & Session

## Sign-in

**Primary path: Credential Manager with `GetGoogleIdOption`.**

1. Launch `GetGoogleIdOption` (Credential Manager API, replacing the deprecated `GoogleSignInClient`).
2. Extract ID token from `GoogleIdTokenCredential`.
3. POST to `POST /auth/google` with `{ idToken }` (no client secret needed — the backend verifies against Google's public keys).
4. Receive `{ accessToken, refreshToken, user }`.
5. Store tokens (see below).
6. Fetch FCM token via `FirebaseMessaging.getToken()` and POST to `/me/fcm-token` asynchronously (non-blocking; sign-in navigates immediately).
7. Trigger the [`/me` bootstrap](./sync.md#bootstrap-on-sign-in--app-open) call and navigate to home.

**Fallback path: legacy `GoogleSignInClient`.** Credential Manager + `GetGoogleIdOption` requires Google Play Services. For de-Googled Android variants (GrapheneOS, LineageOS without microG), the app catches `GetCredentialException` indicating Play Services unavailability and falls back to the legacy `GoogleSignInClient` flow. Both paths converge on the same `POST /auth/google` exchange. This is defensive: at any time a tester in the friend group may be on such a device.

## Token storage

**Decision: Proto DataStore with Tink-encrypted serializer.**

Google deprecated `androidx.security:security-crypto` (which provided `EncryptedSharedPreferences`) in 2024 with no maintained replacement. The recommended substitute is to keep encryption explicit and use a supported substrate. Implementation:

- A Proto-DataStore `auth_proto.pb` stores the access token, refresh token, and access-token expiry.
- The serializer wraps Tink AES-GCM (`com.google.crypto.tink:tink-android`).
- The Tink AEAD key is generated once per install and wrapped by an Android Keystore-backed AES key (alias `liftly_auth_v1`). The wrapping key never leaves secure hardware on devices with a TEE.
- Non-sensitive preferences (theme, locale, etc.) live in a separate, unencrypted Proto DataStore. Mixing sensitive data into the preferences store is the anti-pattern to avoid.

## Refresh-on-401 interceptor

An OkHttp `Authenticator` (not an `Interceptor`) handles 401 responses. It synchronously calls `POST /auth/refresh { refreshToken }` on the calling thread, stores the new access token, and retries the original request with the new token. The `Authenticator` is re-entrant-safe: it tracks in-flight refresh requests so concurrent 401s collapse onto a single refresh call.

## Refresh failure

If `POST /auth/refresh` returns 401 (expired or revoked refresh token), the `Authenticator` clears both tokens from the Proto DataStore, emits a `SessionExpiredEvent` to a singleton `AuthEventBus` (a `SharedFlow`), and the root `NavHost` observes this flow and navigates to the Sign-In screen. Any in-progress network requests fail with a cancelled exception; the offline queue is preserved and will re-flush after the user signs back in.

## FCM token rotation

`LiftlyFirebaseMessagingService.onNewToken()` POSTs the new token to `/me/fcm-token` immediately. If the device is offline at rotation time, the rotation is recorded in a small `pending_fcm_token` row in DataStore and `AuthRepository` retries on next foreground / next successful network call. This is the minimal queue needed for the FCM rotation path — separate from the workout flush queue because lifecycle and retry semantics differ.
