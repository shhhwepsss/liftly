# Networking Layer

**Decision: Retrofit + OkHttp + kotlinx.serialization.**

Ktor client is modern and multiplatform, but at Android-only MVP it adds a build dependency that provides nothing Retrofit doesn't. Moshi vs kotlinx.serialization: kotlinx.serialization is the Kotlin-idiomatic choice, integrates with `@Serializable` data classes, and avoids a reflection-based dependency. Retrofit's `kotlinx-serialization-converter` is stable.

**Interceptors:**

| Interceptor | Purpose |
|---|---|
| `AuthInterceptor` | Attaches `Authorization: Bearer <accessToken>` header to all requests except auth endpoints |
| `IdempotencyKeyInterceptor` | Injects `X-Idempotency-Key` header for POST /workouts, POST /workouts/:id/flush, POST /workouts/:id/complete; key is pre-generated and stored with the queued entity |
| `NetworkFailureInterceptor` | On `IOException` for queued endpoints: catches the exception and routes to the queue rather than propagating; returns a synthetic success to the call site so the ViewModel doesn't error-state the UI |
| `LoggingInterceptor` | OkHttp `HttpLoggingInterceptor`, `BASIC` level in release, `BODY` in debug |

**Error surfacing:** API errors are mapped in a `ResultWrapper<T>` sealed class (`Success`, `NetworkError`, `ApiError(code, message)`) at the Repository layer. ViewModels translate `ApiError` to human-readable `UiError` strings via string resources — no raw HTTP status codes leak to the UI. The `NetworkFailureInterceptor` handles the specific case of offline hot-path writes so the ViewModel's error path is never hit during a workout.
