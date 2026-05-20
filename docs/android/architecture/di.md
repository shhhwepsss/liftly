# Dependency Injection

**Decision: Hilt.**

Koin is simpler to configure but its runtime container means DI errors surface at runtime rather than compile time. Manual DI is feasible for a small project but does not scale gracefully and lacks the lifecycle-aware `ViewModelComponent` Hilt provides out of the box. Hilt's `@HiltViewModel` annotation integrates with Compose navigation's `hiltViewModel()` with zero boilerplate. The team is one person — Hilt's annotation processing overhead is a one-time setup cost, not a daily burden.

Hilt modules live in `core/di/`:
- `NetworkModule` — OkHttp, Retrofit, interceptors.
- `DatabaseModule` — Room database, all DAOs.
- `RepositoryModule` — binds Repository interfaces to implementations.
- `SyncModule` — SyncEngine, FlushWorker configuration.
- `AuthModule` — Proto DataStore + Tink wiring, TokenRepository.
- `FormatModule` — WeightFormatter, TimestampFormatter (read SettingsRepository).
