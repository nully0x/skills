# Kotlin Android Style Guide

Style and architecture guide for Kotlin Android apps. Use it when writing new Kotlin or Android code, reviewing Android PRs, structuring an Android project, wiring UI state, or calling HTTP APIs from Android. It enforces a functional-core style with pure domain logic, sealed state, and reducer UI shells.

Write Kotlin for Android like an experienced engineer: pure functions over immutable data at the core, a thin imperative shell at the edges, concrete types, and no framework ceremony beyond what the platform requires.

The primary rule is:

> Keep `domain/` free of Android imports. Code that imports `android.*` is shell code, not logic.

## Architecture

### Functional core, imperative shell

- Model one screen's state as a single immutable `data class`.
- Model user intents as a `sealed interface` of actions.
- Own every transition in one pure function: `fun reduce(state: State, action: Action, rng: Random): State`.
- Inject randomness, clocks, and I/O as parameters. Pure logic stays deterministic and testable.
- Activities, Fragments, and Services only render state and dispatch actions. They hold no rules and no mutable shared caches.
- Use `copy()` for state changes. Never mutate a state object in place.

### Package layout

Package by layer inside one module until size forces more:

```text
dev/example/app/
  ui/        Activities, Composables, adapters: render state, dispatch actions
  domain/    pure models, actions, reducers. No android.* imports. JVM-testable.
  data/      API clients, repositories, DTO-to-domain mapping
```

Split into multi-module builds only when teams or build times demand it.

### State modeling

- Use `sealed interface` plus exhaustive `when` for phases and results. Never boolean flags like `isLoading` plus `error` plus `data`; model mutually exclusive states as variants.
- Nullable fields must communicate real optionality. `pickedAnswer: Int?` is null only before the player answers.
- Prefer `List`/`Map` read-only interfaces in state. Return unmodifiable or freshly built collections.

## Errors as data

- Model fallible outcomes as `sealed interface Result<out T> { data class Success<T>(val value: T); data class Failure(val cause: Failure) }` or a domain-specific equivalent.
- Catch `IOException` and parsing errors at the data boundary. Never let exceptions cross a layer.
- Do not use exceptions for control flow. Do not return null to mean "failed"; use the sealed type.

## Coroutines and Flow

- All I/O runs in suspend functions on `Dispatchers.IO`. Never block the main thread. Never `runBlocking()` in app code.
- Expose state with `StateFlow`, streams with `Flow`. Collect in the UI with lifecycle awareness.
- No `GlobalScope`. Rely on structured concurrency for cancellation.

## Calling HTTP APIs

- Default stack: Retrofit for declarative endpoints, OkHttp for transport and timeouts, kotlinx.serialization for JSON. Parse JSON once at the boundary into DTOs, then map to domain models.
- All endpoint functions are `suspend`. HTTPS only; cleartext is blocked by default since API 28.
- Declare `<uses-permission android:name="android.permission.INTERNET" />` in the manifest.
- A plain `HttpURLConnection` with manual JSON parsing is acceptable only for zero-dependency demo apps. It is not a best practice for real data.

## UI

- Prefer Jetpack Compose when its dependency cost is acceptable; it renders state declaratively and pairs naturally with reducers.
- Classic Views are acceptable for small, dependency-free apps. Then keep exactly one `render(state)` function per screen, and make every click listener a one-line `dispatch(action)`.
- Design for the audience first. A children's game needs large touch targets, bright solid colors, and instant visual feedback. Do not reuse an adult or tool-style theme for it.
- Keep user-facing strings, colors, and dimensions in `res/`. No magic values in code.

## Testing

- Domain logic tests run on the JVM with plain JUnit. No emulator, no Robolectric.
- Seed `Random` for deterministic tests. Drive full rounds through `reduce` and assert final state.
- One behavior per test. Name tests by behavior. Use fakes over mocks.
- Keep `render()` a thin mapping. Extract visual decisions (which background, which text color per state) into pure helpers, and unit test them like domain code. Cover the Activity itself with instrumented tests (Espresso) when the app grows.

## Build

- Verify with `assembleDebug` and `testDebugUnitTest` before considering work done.
- Kotlin sources live fine under `src/main/java`; keep one source root.
- Introduce a version catalog (`libs.versions.toml`) once more than three libraries exist.
- Release builds need R8 and signing config; do not ship debug builds.
