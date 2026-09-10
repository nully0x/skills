---
name: rust-style
description: Style guide for writing Rust code. Use when writing new Rust code, reviewing Rust PRs, or when the user wants the agent to follow strict Rust conventions. Covers type design, error handling, abstraction discipline, and avoiding AI-generated over-engineering.
---

# Rust Writing Style Guide

Write Rust like an experienced engineer: simple, explicit, strongly typed, boring, idiomatic, easy to trace, minimal abstraction, minimal magic.

Do not optimize for cleverness.

Do not replace simple code with frameworks, generic helpers, trait objects, factories, builders, adapters, or excessive layering.

The primary rule is:

> Validate untrusted data at the boundary. Use concrete types everywhere else.

## Type Design

### Deserialize JSON once at the boundary

If the JSON shape is known, deserialize directly into a struct. Do not carry `serde_json::Value` through the application.

Bad:

```rust
fn get_domain(data: &serde_json::Value) -> String {
    data.get("domain")
        .and_then(|v| v.as_str())
        .unwrap_or("")
        .to_string()
}
```

Prefer:

```rust
#[derive(Deserialize)]
struct DomainEvent {
    domain: String,
}
```

Then `event.domain`.

### Do not represent required values as Option

If a field is required by the application model, use a concrete type.

Bad:

```rust
struct Config {
    enabled: Option<bool>,
    port: Option<u16>,
    name: Option<String>,
}
```

Prefer:

```rust
struct Config {
    enabled: bool,
    port: u16,
    name: String,
}
```

`Option` should communicate real optionality, not a default reflex.

### Avoid excessive DTO/model duplication

Do not create `ProjectRequest`, `ProjectDto`, `ProjectInput`, `ProjectParams`, `ProjectData`, `ProjectModel`, `ProjectEntity`, `ProjectResponse` with nearly identical fields. Use the same type when the concept is the same.

### Use newtypes selectively

A newtype is appropriate when it prevents mixing IDs or adds domain behavior:

```rust
struct ProjectId(String);
struct Region(String);
```

Do not produce a distinct newtype for every string field on a struct.

### Avoid unnecessary wrapper structs

Bad:

```rust
struct ProjectId {
    value: String,
}
```

when a plain `String` is sufficient.

### Avoid constructors that do nothing useful

Bad:

```rust
impl Domain {
    fn new(name: String) -> Self {
        Self { name }
    }
}
```

when `Domain { name }` is clearer and all fields are public within the crate.

## Error Handling

### Return Result, do not swallow errors

Bad:

```rust
if let Err(_) = do_something() {
    return Ok(());
}
```

or:

```rust
let _ = do_something();
```

If an error can safely be ignored, make the reasoning obvious:

```rust
if let Err(err) = cache.delete(key) {
    warn!(?err, key, "failed to invalidate cache");
}
```

### Simplify redundant error handling

Bad:

```rust
let result = repo.get(id)?;
Ok(result)
```

Prefer:

```rust
repo.get(id)
```

If wrapping, add meaningful context:

```rust
repo.get(id).with_context(|| format!("load project {id}"))?
```

Do not mechanically wrap every error at every layer.

### Avoid unnecessary custom error hierarchies

Prefer a single focused error type per module:

```rust
#[derive(thiserror::Error, Debug)]
enum ProjectError {
    #[error("project not found")]
    NotFound,
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),
}
```

Use structured custom errors only when they carry meaningful information.

### Match on error variants idiomatically

Do not manually inspect error strings:

```rust
if err.to_string().contains("duplicate") { ... }
```

Prefer the underlying driver's typed error:

```rust
if let Some(db_err) = err.as_database_error() {
    if db_err.code().as_deref() == Some("23505") {
        // handle duplicate key
    }
}
```

### Avoid panic for normal errors

Do not use `panic!`, `.unwrap()`, or `.expect()` for validation failures, missing DB rows, network errors, or malformed user input. Return `Result`.

Panics are appropriate for invariant violations, programmer errors, or truly unreachable branches.

## Abstraction

### Do not introduce traits without a real reason

Rust traits should usually be defined by the consumer and kept small. Do not create a trait merely because "services should have traits."

Prefer concrete types unless you genuinely need multiple implementations, substitution at runtime, testing at that boundary, plugin behavior, or a narrow consumer-defined contract.

Bad:

```rust
trait DomainManager {
    fn map(&self, ...);
}

struct DomainManagerImpl;
```

with only one impl. Prefer:

```rust
struct DomainManager;
```

### Keep traits small

When a trait is justified, define only what the consumer needs.

Bad:

```rust
trait Repository {
    fn create(&self, ...);
    fn update(&self, ...);
    fn delete(&self, ...);
    fn get(&self, ...);
    fn list(&self, ...);
    fn count(&self, ...);
    fn exists(&self, ...);
}
```

when a component only needs `get`.

Prefer:

```rust
trait ProjectGetter {
    fn get(&self, id: &str) -> Result<Project, Error>;
}
```

### Remove unnecessary abstraction layers

If layers simply forward arguments, remove them. One meaningful layer is better than five pass-through layers.

Bad:

```rust
impl Service {
    fn get_project(&self, id: &str) -> Result<Project, Error> {
        self.manager.get_project(id)
    }
}

impl Manager {
    fn get_project(&self, id: &str) -> Result<Project, Error> {
        self.repository.get_project(id)
    }
}
```

### Remove pass-through wrappers

If a function's entire body is `dependency.do_thing(...)`, inline it.

### Avoid unnecessary factories

Bad:

```rust
struct RepositoryFactory;

impl RepositoryFactory {
    fn create(kind: &str) -> Box<dyn Repository> { ... }
}
```

when the application has one concrete repository.

### Avoid unnecessary dependency-injection infrastructure

Normal Rust dependency injection is usually just:

```rust
let service = Service::new(repo, logger);
```

Do not introduce DI containers, service locators, or registries unless the project genuinely needs them.

### Do not use generics where concrete code is clearer

Bad:

```rust
fn value_or_default<T: Default>(...) -> T
fn convert_vec<T, R>(...) -> Vec<R>
```

Prefer concrete domain code. A generic abstraction should solve a real recurring problem, not just DRY out two call sites.

### Do not force trait-object polymorphism

If there are only two or three slightly different cases, a `match` on an enum may be clearer than `Vec<Box<dyn Strategy>>`.

## Control Flow

### Prefer early returns

Bad:

```rust
if err.is_none() {
    if let Some(project) = project {
        if project.enabled {
            // 80 lines
        }
    }
}
```

Prefer:

```rust
let project = project.ok_or(Error::NotFound)?;
if !project.enabled {
    return Ok(());
}
// main logic
```

Keep the happy path visually obvious.

### Prefer straightforward control flow

Bad:

```rust
let mut should_process = false;
if let Some(project) = project {
    if project.enabled {
        if project.status == Status::Active {
            should_process = true;
        }
    }
}
```

Prefer early returns or a simple boolean expression.

### Remove redundant boolean comparisons

```rust
if enabled == true   // -> if enabled
if enabled == false  // -> if !enabled
```

### Remove useless temporary variables

Bad:

```rust
let raw_domain = event.domain;
let normalized_domain = raw_domain.trim();
let domain = normalized_domain.to_lowercase();
```

Prefer:

```rust
let domain = event.domain.trim().to_lowercase();
```

## Data and Conversions

### Do not create generic conversion helpers

Be suspicious of `to_string_value()`, `as_string()`, `safe_string()`, `to_int()`, `get_string()`, `get_optional_string()`, `value_or_default()`.

Ask: what type is this value actually supposed to be? If it is a string, accept `&str`.

### Remove fallback-to-default-value programming

Do not silently convert invalid or missing states into `String::new()`, `0`, `false`, `None` turned into a default, `Vec::new()`.

If data is required, return an error.

Bad:

```rust
fn project_id(project: Option<&Project>) -> String {
    project.map(|p| p.id.clone()).unwrap_or_default()
}
```

Prefer fixing the caller so `project` doesn't need to be `Option` there. Or if absence is genuinely possible:

```rust
fn project_id(project: Option<&Project>) -> Result<&str, Error> {
    project.map(|p| p.id.as_str()).ok_or(Error::ProjectNotFound)
}
```

### Remove generic map accessor utilities

Bad:

```rust
fn get_string(data: &HashMap<String, serde_json::Value>, key: &str) -> String
```

Deserialize into a struct instead.

### Remove mapper slop

Do not maintain fleets of `to_dto` / `from_dto` / `to_model` / `from_model` conversions without a meaningful difference in representation. Where a real conversion is needed, prefer `From`/`TryFrom` over bespoke free functions.

### Remove pointless helper extraction

Do not create a helper for every 2–3 lines that has one caller, only accesses a field, only checks `Option`/`Result`, only calls `.trim()` or `.to_string()`, or only forwards arguments.

## Concurrency

### Do not spawn tasks unnecessarily

Every spawned task introduces lifecycle, cancellation, and panic-propagation concerns. Use concurrency when the operation actually benefits from running independently.

### Avoid channel-based architecture for simple synchronous work

Bad:

```
handler -> mpsc channel -> worker task -> mpsc channel -> processor
```

for logic that could simply be `processor.process(event).await?`.

### Do not wrap every operation in ad hoc timeouts

Timeout policy should usually live at meaningful boundaries, not repeated inside every repository/service function.

## Logging

### Keep logging simple

Do not narrate every function:

```rust
info!("entering create_project");
info!("validating project");
```

Log meaningful events: failures, important state transitions, external interactions.

### Avoid logging and returning the same error at every layer

Libraries/services should return `Result`. Handlers/workers decide when to log.

## Comments and Naming

### Remove obvious comments

Delete comments that narrate syntax:

```rust
// Check if project exists.
if project.is_none() {

// Return the result.
return result;
```

Keep comments for business rules, invariants, non-obvious decisions, external system quirks, workarounds, and `unsafe` reasoning.

### Use constants where they clarify meaning

```rust
const DUPLICATE_KEY_CODE: &str = "23505";  // good: domain meaning
const EMPTY_STRING: &str = "";              // bad: restates the literal
```

## Modules

### Avoid over-modularization

Do not create a module for every type/helper. Modules should represent coherent concepts, not one function.

### Remove vague utility modules

Audit modules named `utils`, `helpers`, `common`, `shared`, `misc`, or `core`. Move useful functions to the module that owns the concept.

## Validation

### Keep boundary validation

Validation is appropriate for HTTP request bodies, queue messages, webhooks, config/env vars, external APIs, user input, deserialized untrusted JSON.

### Do not re-validate trusted internal structs

If `Project` is only ever constructed from already-validated external input, validate once at construction. Do not repeatedly re-validate throughout the system.

### Keep business rules explicit

Bad:

```rust
if validator.is_valid(&project) { ... }
```

Prefer:

```rust
if project.status != Status::Active {
    return Err(Error::ProjectInactive);
}
```

## Tests

### Keep tests simple

Apply the same principles. Remove enormous test builders, generic fixture systems, helper pyramids, mocks for pure logic.

Prefer explicit table-driven tests:

```rust
#[test]
fn normalizes_domain() {
    let cases = [
        ("EXAMPLE.COM", "example.com"),
        (" example.com ", "example.com"),
    ];

    for (input, want) in in cases {
        assert_eq!(normalize_domain(input), want);
    }
}
```

## Things to Preserve

Do not remove:

- useful traits with real multiple implementations
- meaningful error context (`anyhow::Context`, `#[from]` conversions)
- legitimate `Option` handling for genuinely optional data
- boundary validation
- proper resource cleanup (`Drop` impls, RAII guards)
- ownership/borrowing that the borrow checker actually requires
- `Arc<Mutex<T>>` / `RwLock<T>` protecting real shared mutable state across threads
- channel synchronization that is actually needed for concurrent handoff
- driver-specific error handling
- `unsafe` blocks that are correctly justified and documented
- security-related checks

## The Overriding Principle

> Make untrusted input safe at the edge. Keep trusted Rust code boring everywhere else.

Do not replace one kind of slop with another. The correct solution to `data["domain"].as_str().unwrap().to_string()` is usually:

```rust
#[derive(Deserialize)]
struct Event {
    domain: String,
}
```

followed by `event.domain`.
