# TypeScript Codebase Cleanup: Remove Defensive Over-Engineering and AI Slop

Audit this TypeScript codebase and aggressively simplify AI-generated, defensive, overly abstract, or unnecessarily generic code.

The goal is to make the code look like it was written by an experienced TypeScript engineer:

- simple
- explicit
- strongly typed
- boring
- idiomatic
- easy to trace
- minimal abstraction
- minimal magic

Do not optimize for cleverness.

Do not replace simple code with frameworks, generic helpers, class hierarchies, factories, adapters, builders, or excessive layering.

The primary rule is:

> Validate untrusted data at the boundary. Use concrete types everywhere else.

## 1. Remove unnecessary `any` and loose `unknown`/object bags

Search aggressively for:

```
any
: unknown
Record<string, any>
Record<string, unknown>
{ [key: string]: any }
```

Ask why the value is untyped.

Bad:

```typescript
function getDomain(data: Record<string, any>): string {
  return data.domain ?? "";
}
```

Prefer:

```typescript
interface DomainEvent {
  domain: string;
}
```

Then:

```typescript
event.domain
```

Do not carry `any`/generic objects through the application and repeatedly re-derive shape from them. If the JSON shape is known, parse it directly into a typed structure at the boundary (ideally with runtime validation — see §17–19).

## 2. Stop creating generic conversion helpers

Be suspicious of functions like:

```
toString()
asString()
stringValue()
safeString()
toNumber()
asNumber()
toBoolean()
toObject()
getString()
getOptionalString()
valueOrDefault()
```

Bad:

```typescript
function toString(value: unknown): string {
  if (typeof value === "string") return value;
  if (typeof value === "number") return String(value);
  if (value === null || value === undefined) return "";
  return JSON.stringify(value);
}
```

Ask instead:

> What type is this value actually supposed to be?

If it is a string:

```typescript
function normalizeDomain(domain: string): string {
  return domain.trim().toLowerCase();
}
```

Do not accept `any`/`unknown` just to make a helper "flexible" when every real caller already has one concrete type.

## 3. Remove fallback-to-default programming

Search for code that silently converts invalid states into `""`, `0`, `false`, `null`, `[]`, or `{}`.

Examples:

```typescript
if (value == null) return "";

try {
  doThing();
} catch {
  return null;
}

function getProject(id: string): Project {
  return projects.get(id) ?? ({} as Project);
}
```

Do not hide invalid states. If data is required, throw or return an error result.

Bad:

```typescript
function projectId(project: Project | null): string {
  return project?.id ?? "";
}
```

Prefer fixing the caller so `project` cannot be `null` there.

Or, if absence is genuinely possible:

```typescript
function projectId(project: Project | null): string {
  if (!project) {
    throw new ProjectNotFoundError();
  }
  return project.id;
}
```

Validate/narrow once, then continue with clean code.

## 4. Do not add null/undefined checks everywhere

Optional-chaining and null checks should correspond to actual nullable states.

Bad:

```typescript
function handleProject(project?: Project | null): void {
  if (!project) throw new Error("project is nil");
  if (!project.config) throw new Error("config is nil");
  if (!project.config.domain) throw new Error("domain is nil");

  // actual logic
}
```

If those fields are required by the application model, redesign the types instead.

Prefer:

```typescript
interface Project {
  config: ProjectConfig;
}

interface ProjectConfig {
  domain: string;
}
```

Then:

```typescript
project.config.domain
```

Do not mark fields optional (`field?:`) merely because TypeScript allows it. Optionality should mean the value is genuinely sometimes absent.

## 5. Stop making everything optional

Review type definitions like:

```typescript
interface Config {
  enabled?: boolean;
  port?: number;
  name?: string;
}
```

Do not use `?`/`| undefined` merely to distinguish "missing" from a default unless the distinction actually matters.

If those values are required:

```typescript
interface Config {
  enabled: boolean;
  port: number;
  name: string;
}
```

`?` and `| null` should communicate real optionality — not be a default reflex "in case it's not there later."

## 6. Remove pointless helper extraction

Do not create a helper for every 2–3 lines.

Be suspicious of:

```
getDomainId()
extractProjectId()
resolveName()
buildKey()
parseValue()
stringFrom()
domainFrom()
```

when the helper:

- has one caller
- only accesses a property
- only checks for `null`/`undefined`
- only calls `.trim()`
- only calls `.toString()`
- only performs a type narrow
- only forwards arguments

Bad:

```typescript
function domainIdFrom(record?: DnsRecord | null): string {
  if (!record) return "";
  return record.domainId.toString();
}
```

Prefer:

```typescript
record.domainId.toString()
```

provided `record` is already known to exist.

A helper should represent a real reusable concept, not hide straightforward code.

## 7. Do not introduce interfaces/abstract classes without a real reason

Be suspicious of:

```typescript
interface ProjectService {
  createProject(...): Promise<Project>;
  getProject(...): Promise<Project>;
  updateProject(...): Promise<Project>;
  deleteProject(...): Promise<void>;
  listProjects(...): Promise<Project[]>;
  validateProject(...): Promise<boolean>;
  syncProject(...): Promise<void>;
}
```

especially if there is only one implementation.

Do not create an interface merely because "services should have interfaces."

Prefer concrete classes/functions unless you genuinely need:

- multiple implementations
- substitution (e.g. mocking a true external boundary)
- a narrow consumer-defined contract
- plugin behavior

Bad:

```typescript
interface DomainManager {
  map(...): Promise<void>;
}

class DomainManagerImpl implements DomainManager { ... }
```

with only one implementation. Prefer:

```typescript
class DomainManager {
  map(...): Promise<void> { ... }
}
```

Do not build Java-style `IFoo` / `FooImpl` architecture in TypeScript.

## 8. Keep interfaces small

When an interface is justified, define only what the consumer needs.

Bad:

```typescript
interface Repository {
  create(...): Promise<Project>;
  update(...): Promise<Project>;
  delete(...): Promise<void>;
  get(...): Promise<Project>;
  list(...): Promise<Project[]>;
  count(...): Promise<number>;
  exists(...): Promise<boolean>;
}
```

when a component only needs `get`.

Prefer:

```typescript
interface ProjectGetter {
  get(id: string): Promise<Project>;
}
```

Do not expose giant interfaces just because a concrete repository has many methods.

## 9. Remove unnecessary abstraction layers

Look for chains like:

```
handler
  -> controller
  -> service
  -> manager
  -> processor
  -> repository
  -> store
  -> db client
```

where most layers simply forward arguments.

Bad:

```typescript
class Service {
  getProject(id: string) {
    return this.manager.getProject(id);
  }
}

class Manager {
  getProject(id: string) {
    return this.repository.getProject(id);
  }
}
```

If these layers do not contain meaningful business logic, remove them. One meaningful layer is better than five pass-through layers.

## 10. Remove pass-through wrappers

Search for functions whose entire body is:

```typescript
return dependency.doThing(...);
```

or:

```typescript
return helper(value);
```

or:

```typescript
const result = await dependency.doThing(...);
return result;
```

Simplify:

```typescript
return dependency.doThing(...);
```

Do not add wrappers purely to make the architecture look layered.

## 11. Simplify redundant error handling

Bad:

```typescript
try {
  const result = await repo.get(id);
  return result;
} catch (err) {
  throw err;
}
```

Prefer:

```typescript
return repo.get(id);
```

Bad:

```typescript
catch (err) {
  throw new Error(`error: ${err}`);
}
```

This adds no useful context and destroys the stack/cause chain.

If wrapping, add meaningful context and preserve the cause:

```typescript
catch (err) {
  throw new Error(`load project ${id}`, { cause: err });
}
```

But do not mechanically wrap every error at every layer until the message becomes `failed to process project: failed to get project: failed to retrieve project: ...`. Add context where it materially improves debugging.

## 12. Do not swallow errors

Find:

```typescript
try {
  doSomething();
} catch {
  // nothing
}
```

or:

```typescript
doSomething().catch(() => {});
```

or:

```typescript
void doSomething();
```

Determine whether ignoring the error is intentional. Do not turn bugs into silent success.

If an error can safely be ignored, make the reasoning obvious:

```typescript
try {
  await cache.delete(key);
} catch (err) {
  logger.warn("failed to invalidate cache", { key, err });
}
```

only when cache invalidation failure genuinely should not fail the operation.

## 13. Avoid unnecessary custom error hierarchies

Be suspicious of enormous error class trees:

```typescript
class ValidationError extends Error {}
class RepositoryError extends Error {}
class ServiceError extends Error {}
class DomainError extends Error {}
class InternalError extends Error {}
```

unless callers actually need different behavior based on the error type.

Prefer a small number of meaningful error classes:

```typescript
class ProjectNotFoundError extends Error {}
```

and:

```typescript
if (err instanceof ProjectNotFoundError) { ... }
```

Use structured custom errors only when they carry meaningful information (an error code, a status, a field name).

## 14. Narrow errors idiomatically

Do not manually inspect error strings.

Bad:

```typescript
if (err.message.includes("duplicate")) { ... }
```

Prefer the underlying driver's typed error:

```typescript
if (err instanceof DatabaseError && err.code === "23505") {
  // handle duplicate key
}
```

But do not turn a simple known-driver check into enormous generic error inspection code. Keep it proportional to the actual need.

## 15. Avoid reflection-ish, dynamic-property, and decorator-heavy code unless genuinely necessary

Search for:

```
Reflect.
Object.keys(obj).forEach
eval(
new Function(
Proxy(
```

Bad:

```typescript
function isEmpty(value: unknown): boolean {
  if (typeof value === "object" && value !== null) {
    return Object.keys(value).length === 0;
  }
  return !value;
}
```

Prefer explicit typed logic. Do not reach for dynamic property walking or `Proxy` to avoid writing a few obvious lines.

This kind of dynamism is reasonable in areas such as serializers, ORMs, and framework internals. It should rarely appear in normal handlers/services/domain logic.

## 16. Do not use generics where concrete code is clearer

TypeScript generics are useful, but AI often introduces them for trivial problems.

Be suspicious of:

```typescript
function valueOrDefault<T>(value: T | undefined, fallback: T): T
function convertArray<T, R>(items: T[], fn: (t: T) => R): R[]
function safeCast<T>(value: unknown): T
function getOrDefault<K, V>(map: Map<K, V>, key: K, fallback: V): V
```

Do not create generic utilities just because two call sites look similar.

Prefer concrete domain code where it is easier to understand. A generic abstraction should solve a real recurring problem — and `convertArray` in particular is usually just `.map()`.

## 17. Remove generic object accessor utilities

Bad:

```typescript
function getString(data: Record<string, unknown>, key: string): string
function getNumber(data: Record<string, unknown>, key: string): number
function getBoolean(data: Record<string, unknown>, key: string): boolean
```

This is usually evidence that structured data should have been parsed into a typed shape.

Prefer:

```typescript
interface DeploymentEvent {
  projectId: string;
  port: number;
  enabled: boolean;
}
```

Then use `event.projectId`, `event.port`, `event.enabled`.

## 18. Parse and validate once at the boundary

Bad:

```typescript
const payload: any = JSON.parse(body);
const event = payload.event;
const data = payload.data;
const projectId = data?.projectId;
```

Prefer parsing into a typed, validated shape once — with `zod`, `valibot`, or whatever validation library the project already uses:

```typescript
const QueueEventSchema = z.object({
  event: z.string(),
  data: z.unknown(),
});

const ProjectSyncEventSchema = z.object({
  projectId: z.string(),
});

const payload = QueueEventSchema.parse(JSON.parse(body));
const event = ProjectSyncEventSchema.parse(payload.data);
```

After parsing, business logic should operate on typed values, not `any`/raw JSON.

## 19. Avoid `any`/loose objects for database and API models

If Postgres JSONB, MongoDB, or an external API returns a known document shape, define an interface/type — and validate it if it crosses a real trust boundary.

Bad:

```typescript
const document: any = await collection.findOne({ _id });
const domain = document.domain;
```

Prefer:

```typescript
interface DnsDocument {
  domain: string;
}

const document = await collection.findOne<DnsDocument>({ _id });
```

Fix broad types at the data-access boundary rather than adding extractors everywhere.

## 20. Remove fake defensive type narrowing

Bad:

```typescript
if (typeof data !== "object" || data === null) return null;
const value = data as Record<string, unknown>;
if (typeof value.domain !== "string") return null;
```

if the data came from a contract already controlled by the application.

Either:

- type it correctly upstream, or
- genuinely validate it at the external boundary

Do not repeatedly rediscover types inside trusted application code, and do not use `as` to silence the compiler instead of actually narrowing.

## 21. Avoid excessive DTO/model duplication

Be suspicious when the same data has:

```
ProjectRequest
ProjectDto
ProjectInput
ProjectParams
ProjectData
ProjectModel
ProjectEntity
ProjectResponse
```

with nearly identical fields.

Separate types when the contracts are materially different. Do not duplicate interfaces solely because each layer "needs its own model." If two layers genuinely share the same concept, use the same type (or derive one from the other with `Pick`/`Omit` rather than hand-duplicating fields).

## 22. Remove mapper slop

Look for functions like:

```typescript
function projectToDto(project: Project): ProjectDto {
  return { id: project.id, name: project.name };
}
```

when `ProjectDto` and `Project` are effectively identical and no boundary requires the distinction.

Do not maintain fleets of `toDto` / `fromDto` / `toModel` / `fromModel` without a meaningful difference in representation.

## 23. Avoid constructors/factory functions that do nothing useful

Bad:

```typescript
function createDomain(name: string): Domain {
  return { name };
}
```

when:

```typescript
const domain: Domain = { name };
```

is clearer.

A constructor/factory can be reasonable when it establishes invariants (validation, normalization, computed defaults) or hides meaningful setup. Don't add one just because "objects should have a factory function."

## 24. Remove builder-pattern slop

Do not introduce fluent builders for simple objects.

Bad:

```typescript
const deployment = new DeploymentBuilder()
  .withProjectId(projectId)
  .withRegion(region)
  .withPort(port)
  .withImage(image)
  .build();
```

Prefer:

```typescript
const deployment: Deployment = { projectId, region, port, image };
```

Builders are justified only when construction is genuinely complex (many optional fields, staged validation, or a real multi-step assembly process).

## 25. Do not overuse options-object indirection

Avoid:

```typescript
new Service({
  repository: repo,
  logger: logger,
  metrics: metrics,
});
```

when all three are required and there's no ambiguity from positional args.

Prefer:

```typescript
new Service(repo, logger, metrics);
```

Options objects are useful primarily for optional configuration or APIs with many optional settings, or when the constructor takes several same-typed positional args that would be easy to transpose. Do not introduce them just because it "looks more configurable."

## 26. Avoid unnecessary factories

Bad:

```typescript
class RepositoryFactory {
  static create(kind: string): Repository { ... }
}
```

when the application has one concrete repository.

Prefer constructing the concrete dependency directly. Factories should exist because runtime selection is real, not because "factory pattern" sounds architectural.

## 27. Avoid unnecessary dependency-injection infrastructure

Normal TypeScript dependency injection is usually just:

```typescript
const service = new Service(repo, logger);
```

Do not introduce DI containers, service locators, decorator-based injection frameworks, or reflection-based wiring unless the project genuinely needs them (e.g. a large NestJS app already built around it). Explicit construction is a strength of plain TypeScript.

## 28. Prefer straightforward control flow

Bad:

```typescript
let shouldProcess = false;
if (project) {
  if (project.enabled) {
    if (project.status === "active") {
      shouldProcess = true;
    }
  }
}
```

Prefer early returns:

```typescript
if (!project) return;
if (!project.enabled) return;
if (project.status !== "active") return;

// actual work
```

Or when simple:

```typescript
const shouldProcess =
  project != null && project.enabled && project.status === "active";
```

Choose whichever is easier to read. Do not optimize for clever ternary chains or nested optional-chaining one-liners.

## 29. Prefer early returns over deep nesting

Bad:

```typescript
if (!err) {
  if (project) {
    if (project.enabled) {
      // 80 lines
    }
  }
}
```

Prefer:

```typescript
if (err) throw err;
if (!project) throw new ProjectNotFoundError();
if (!project.enabled) return;

// main logic
```

Keep the happy path visually obvious.

## 30. Remove useless temporary variables

Bad:

```typescript
const rawDomain = event.domain;
const normalizedDomain = rawDomain.trim();
const domain = normalizedDomain.toLowerCase();
```

Prefer:

```typescript
const domain = event.domain.trim().toLowerCase();
```

But do not compress code so aggressively that readability decreases. Intermediate variables should represent meaningful concepts.

## 31. Remove redundant boolean comparisons

Clean up:

```typescript
if (enabled === true)
if (enabled === false)
```

Prefer:

```typescript
if (enabled)
if (!enabled)
```

Unless the value is `boolean | undefined` and the three-way distinction genuinely matters — in which case make that explicit (`enabled === true` really does mean something different from `enabled == null`).

## 32. Remove redundant collection initialization

Question code like:

```typescript
const items: Item[] = [];
```

when a `.map()`/`.filter()` expression or array literal would produce the same result more directly. Don't reach for manual `push` loops where a functional pipeline is clearer, and don't build elaborate functional pipelines where a plain loop is clearer — pick whichever reads more directly for the actual transformation.

## 33. Do not clone/spread defensively without a reason

Be suspicious of reflexive spreading with no real mutation threat.

Bad:

```typescript
function getItems(): Item[] {
  return [...this.items];
}
```

added "just in case" when nothing downstream mutates the result.

Do not add `{ ...obj }` or `[...arr]` copies everywhere as a defensive habit. Copy when ownership/mutation semantics genuinely require it (e.g. returning internal state to a caller that might mutate it).

## 34. Avoid premature performance tricks

Do not introduce:

- manual memoization caches for cheap computations
- `WeakMap`-based object pools
- hand-rolled debouncing/throttling infrastructure
- worker-thread pools
- elaborate caching layers (Redis, in-memory LRU) without evidence of need

without evidence that the code needs them. Simple correct code first. Performance optimizations should solve measured problems.

## 35. Do not scatter async/fire-and-forget calls unnecessarily

Be suspicious of:

```typescript
void doSomethingAsync();
```

or unawaited promises added merely to make something "non-blocking."

Every unawaited async call introduces unhandled-rejection risk, ordering ambiguity, and error-propagation problems. Use fire-and-forget only when the operation genuinely should run independently of the caller (and even then, attach an explicit `.catch()`).

## 36. Avoid event-emitter/queue architecture for simple synchronous work

Bad:

```
handler -> EventEmitter -> listener -> EventEmitter -> processor
```

for logic that could simply be:

```typescript
await processor.process(event);
```

`EventEmitter`s, message queues, and pub/sub are for genuine decoupling across independent consumers, not an architectural requirement. Do not use them to make a normal function call look event-driven.

## 37. Pass only the data a function actually needs

Do not:

- thread an entire `AppContext`/`Config` object into functions that use one field
- accept `any`/a giant options bag "just in case" future callers need more

Typical signature:

```typescript
function getProject(db: Database, id: string): Promise<Project>
```

Pass the caller's actual dependencies through, not a bag of everything.

## 38. Avoid wrapping every operation in ad hoc timeouts

Do not mechanically write:

```typescript
await Promise.race([
  operation(),
  new Promise((_, reject) => setTimeout(() => reject(new Error("timeout")), 5000)),
]);
```

inside every repository/service function. Timeout policy should usually live at meaningful boundaries (the HTTP client, the outer request handler). Repeated nested arbitrary timeouts are difficult to reason about.

## 39. Keep logging simple

Avoid logs that merely narrate every function:

```typescript
logger.info("entering createProject");
logger.info("validating project");
logger.info("calling repository");
logger.info("repository completed");
logger.info("leaving createProject");
```

Log meaningful events: failures, important state transitions, operational decisions, external interactions worth tracing. Do not turn application logs into execution commentary.

## 40. Avoid logging and rethrowing the same error at every layer

Bad:

```typescript
try {
  return await repo.get(id);
} catch (err) {
  logger.error("failed to get project", { err });
  throw err;
}
```

if the caller will also log it.

Prefer logging once at the boundary responsible for handling the failure. Libraries/services should generally throw/reject; request handlers/workers decide when to log.

## 41. Remove obvious comments

Delete comments like:

```typescript
// Check if project exists.
if (!project) {

// Return the result.
return result;

// Convert string to lowercase.
domain = domain.toLowerCase();
```

Keep comments for: business rules, invariants, non-obvious decisions, external system quirks, workarounds, and concurrency/timing reasoning. Comments should explain why, not narrate syntax.

## 42. Avoid over-modularization

Do not create a file/folder for every type/helper.

Bad:

```
src/
  domainParser/
  stringUtils/
  validationHelper/
  projectMapper/
  responseBuilder/
```

Prefer modules organized around actual domains/capabilities. A module should represent a coherent concept, not one function.

## 43. Remove vague utility modules

Audit files/folders named `utils`, `helpers`, `common`, `shared`, `misc`, or `lib` (when used as a dumping ground). These often accumulate unrelated abstractions.

Move useful functions to the module that owns the concept. Delete trivial helpers. Avoid creating another generic utility module during cleanup.

## 44. Avoid unnecessary wrapper types/classes

Bad:

```typescript
class ProjectId {
  constructor(public value: string) {}
}
```

when a plain `string` is sufficient.

A branded type can be appropriate:

```typescript
type ProjectId = string & { readonly __brand: "ProjectId" };
```

if it prevents mixing IDs at the type level. But do not wrap primitives in classes/objects without a concrete benefit — a branded type usually gets you the safety without runtime overhead.

## 45. Use branded/domain types selectively

This can be useful:

```typescript
type ProjectId = string & { readonly __brand: "ProjectId" };
type Region = string & { readonly __brand: "Region" };
```

when it prevents accidentally passing a `Region` where a `ProjectId` is expected.

But do not produce a distinct branded type for every string field on an interface. Use domain types where they materially improve correctness, not everywhere by default.

## 46. Do not duplicate standard library/lodash functionality

Before keeping a helper, check whether native JS/TS or an already-used dependency (e.g. `lodash`, `date-fns`) already does the operation. Prefer `Array.prototype.flat`, `Object.entries`, `structuredClone`, `Array.prototype.at`, optional chaining/nullish coalescing, `Intl` formatters, etc.

Do not maintain custom helpers that poorly reimplement something already available.

## 47. Avoid regex when normal string operations work

Bad:

```typescript
input.replace(/\s+/g, " ");
```

for simple trimming or known delimiters where `.trim()` / `.split()` would do.

Prefer `String` methods where sufficient. Regex should solve regex-shaped problems.

## 48. Simplify string formatting

Avoid:

```typescript
`${value}`
```

when `value` is already a `string` — just use it directly.

Avoid template-literal gymnastics for simple concatenation when a plain `+` or direct value is clearer. But do not replace readable formatting merely for micro-performance.

## 49. Do not re-validate trusted internal objects

Bad:

```typescript
function validateProject(project: Project): void {
  if (!project.id) throw new Error("missing project ID");
  ...
}
```

called in every internal service function.

If `Project` is only ever constructed from already-validated external input (e.g. parsed once with `zod` at the API boundary), don't re-validate the same object throughout the system — let the type system carry that guarantee.

## 50. Keep boundary validation

Do not blindly remove validation. Validation is appropriate for:

- HTTP request bodies/query/path params
- queue messages
- webhooks
- config/env vars
- external API responses
- user input
- parsed untrusted JSON
- data pulled from a schemaless store

The rule is:

> Defend against external uncertainty, not against your own correctly typed code.

## 51. Avoid excessive validation-library ceremony

If normal TypeScript is sufficient:

```typescript
if (!req.domain) {
  throw new Error("domain required");
}
```

do not introduce a large validation-schema stack solely to avoid a few `if` statements. Use an existing validation library (`zod`, `yup`, etc.) when the project already relies on it and the schema complexity actually warrants it — not for every trivial internal function.

## 52. Keep business rules explicit

Do not hide meaningful rules behind generic abstractions.

Bad:

```typescript
if (validator.isValid(project)) { ... }
```

when the real business rule is:

```typescript
if (project.status !== "active") {
  throw new ProjectInactiveError();
}
```

Domain rules should be visible in the code.

## 53. Avoid giant config/context objects passed everywhere

Bad:

```typescript
function process(ctx: AppContext, cfg: Config, options: Options, metadata: Metadata)
```

when the function needs `projectId` and `region`.

Pass the data the function actually needs. Narrow function signatures improve readability and testability.

## 54. Do not introduce unnecessary options objects

Bad:

```typescript
interface GetProjectOptions {
  id: string;
}

function getProject(opts: GetProjectOptions): Promise<Project>
```

Prefer:

```typescript
function getProject(id: string): Promise<Project>
```

Options objects are useful when several meaningful parameters travel together or genuine optional parameters exist.

## 55. Avoid unnecessary return-value wrapper objects

Bad:

```typescript
interface ExistsResult {
  exists: boolean;
}
```

Prefer:

```typescript
function exists(...): Promise<boolean>
```

Use objects when multiple related return values genuinely form a meaningful shape.

## 56. Avoid needless indirection in return types

Bad:

```typescript
function getProject(id: string): { project: Project | null; error: Error | null }
```

hand-rolling a Go-style result tuple in a language with exceptions/promises.

Prefer:

```typescript
function getProject(id: string): Promise<Project>
```

and let it reject on failure — or, if the codebase deliberately uses a `Result<T, E>` pattern, be consistent about it rather than mixing both styles function to function.

## 57. Remove defensive global catch-alls sprinkled through business logic

Search for:

```typescript
process.on("uncaughtException", ...)
```

or `try { ... } catch { /* swallow */ }` wrapped around large blocks of unrelated logic "just in case."

Do not use blanket exception handling to turn programming bugs into normal control flow. Reserve process-level catch-alls for genuine top-level boundaries (the server's request handler, the process entrypoint) where logging and a controlled shutdown are intentional. Do not put them inside normal business functions.

## 58. Avoid throwing for normal, expected outcomes

Do not throw exceptions for:

- validation failures the caller is expected to handle
- "not found" lookups that are a routine outcome
- expected network/timeout conditions in a retry loop

when a `Result`-style return, an `Option`, or a plain `null` check communicates the expected outcome more cheaply than exception-based control flow. Reserve `throw` for genuinely exceptional, unexpected conditions.

## 59. Avoid stateful singleton/global slop

Review:

```typescript
let defaultClient: Client;
let globalConfig: Config;
export const singleton = new Service();
```

Module-level constants are fine for true immutable state (a compiled regex, a loaded config object). Do not use mutable module-level state as a shortcut for dependency wiring. Pass dependencies explicitly through constructors/function parameters.

## 60. Do not abstract simple database operations unnecessarily

Bad:

```
Service -> Repository -> Store -> Dao -> QueryExecutor -> db client
```

Use the minimum layering that matches the application. A repository around SQL/ORM queries is reasonable. A repository wrapped by another class that just forwards everything is not.

## 61. Keep queries obvious

Do not build elaborate query builders for a handful of static queries unless dynamic composition is actually needed. Plain SQL or a straightforward ORM call is often easier to understand:

```typescript
const project = await db.query<Project>(
  `SELECT id, name FROM projects WHERE id = $1`,
  [id],
);
```

Do not hide straightforward queries behind abstraction solely to avoid writing them.

## 62. Avoid transactional abstraction slop

Do not invent elaborate `TransactionManager` / `UnitOfWork` / `TransactionRunner` classes if this suffices:

```typescript
await db.transaction(async (tx) => {
  // ... queries against tx
});
```

Abstract transaction handling only when repeated complexity genuinely justifies it.

## 63. Use constants where they clarify meaning

Do not replace every literal with a named constant.

Good:

```typescript
const DUPLICATE_KEY_CODE = "23505";
```

when the value has domain/driver meaning.

Unnecessary:

```typescript
const EMPTY_STRING = "";
const TRUE_VALUE = true;
```

Constants should communicate meaning, not restate the literal.

## 64. Use union/enum types for closed sets, don't over-stringify

Custom string-literal unions are idiomatic and should be kept:

```typescript
type DeploymentStatus = "pending" | "running" | "failed";
```

Don't "simplify" this back to a raw `string` to reduce type count — that removes real type safety. The anti-pattern is the opposite: hand-rolled string comparisons (`if (status === "pending")`) scattered around where a shared union type already exists or should.

Prefer string-literal unions over `enum` in most cases, unless the codebase already has an established `enum` convention — string unions avoid the extra runtime object and erase cleanly.

## 65. Prefer readable `switch`/object-literal dispatch appropriately

A simple `switch` is often clearer than an indirect handler map:

```typescript
switch (event.type) {
  case "insert":
    return handleInsert(event);
  case "delete":
    return handleDelete(event);
  default:
    return assertUnreachable(event.type);
}
```

An object-literal dispatch table (`{ insert: handleInsert, delete: handleDelete }`) is fine when it's genuinely simpler and the key set is closed — don't build a class-based "handler registry" for what a `switch` or a plain object literal already does clearly.

## 66. Do not force class-hierarchy polymorphism

If there are only two or three slightly different cases, a discriminated union with a `switch` is often clearer than:

```typescript
abstract class Strategy {
  abstract execute(...): void;
}

class InsertStrategy extends Strategy { ... }
class DeleteStrategy extends Strategy { ... }
class ReplaceStrategy extends Strategy { ... }
```

TypeScript does not require every branch to become a class hierarchy. A discriminated union (`{ kind: "insert"; ... } | { kind: "delete"; ... }`) plus a `switch` is usually easier to trace and gives exhaustiveness checking for free.

## 67. Avoid excessive mocking

Do not create an interface purely so every dependency can be mocked with `jest.mock`/`vi.mock`.

Prefer testing real behavior where practical (e.g. against a real or in-memory test database). Use small interfaces around expensive/external boundaries (third-party APIs, payment providers). Do not turn every internal class into an interface because "tests need mocks."

## 68. Keep tests simple too

Apply the cleanup to tests. Remove:

- enormous test builders/fixture factories
- generic fixture systems
- helper pyramids
- mocks for pure logic
- excessive test abstraction layers

Prefer explicit table-driven tests where appropriate:

```typescript
test.each([
  ["EXAMPLE.COM", "example.com"],
  [" example.com ", "example.com"],
])("normalizes %s to %s", (input, want) => {
  expect(normalizeDomain(input)).toBe(want);
});
```

Do not make tests harder to understand than the code they test.

## 69. Do not blindly remove idiomatic TypeScript error/null handling

This cleanup must not mistake normal TypeScript code for slop.

This is idiomatic and should remain:

```typescript
if (!value) {
  throw new Error("value is required");
}
```

Likewise:

```typescript
const item = map.get(key);
if (item === undefined) {
  ...
}
```

and:

```typescript
if (project == null) {
  ...
}
```

can all be correct. The question is whether the failure/absence is genuinely possible and meaningful. Do not remove necessary checks merely to reduce line count.

## 70. Optimize for code reduction, not line golf

The desired code should be shorter because unnecessary concepts disappeared — not because everything was compressed into unreadable chained optional-access/ternary expressions.

Bad cleanup:

```typescript
const x = a?.b?.c ?? (d?.e?.f ?? (g ? h : i));
```

Prefer boring, readable TypeScript.

## 71. Fix the root type instead of adding helpers

Whenever you see `asString(value)`, `asObjectId(value)`, `extractDomain(value)`, `safeValue(value)`, `toRecord(value)`, trace where `value` originated. Ask:

- Why isn't this value already strongly typed?
- Is this data external?
- Can it be parsed/validated into a concrete type at the boundary?
- Can downstream functions accept the concrete type?
- Can this helper then disappear?

Always prefer fixing the source of poor typing.

## 72. Look for "AI architecture"

Be particularly suspicious of combinations like:

```
interfaces.ts
factory.ts
builder.ts
mapper.ts
converter.ts
validator.ts
utils.ts
helpers.ts
manager.ts
processor.ts
service.ts
repository.ts
```

inside one small feature. Do not assume all these layers are necessary. Determine what each one actually does. Collapse layers that merely forward calls or transform identical structures.

## 73. Search patterns to audit

Search the repository for:

```
: any
as any
unknown
Record<string,
Reflect.
eval(
new Function(
Proxy(
try {
catch (
throw new
== null
!= null
interface .* {
Factory
Builder
Manager
Processor
Helper
Utils
Mapper
Converter
Validator
Options
Params
Dto
Entity
Model
ValueOr
Safe
Normalize
Extract
Resolve
Parse
GetString
AsString
ToString
With
process.env
new EventEmitter
setTimeout(
Array<
```

Do not automatically change every match. Use them as places to inspect for unnecessary complexity.

## 74. Apply the "de-slop test" to every change

Before adding or keeping code, ask:

- Does this handle something that can genuinely happen? If no, delete it.
- Is this complexity caused by poor typing upstream? If yes, fix the upstream type.
- Does this interface have more than one meaningful implementation or consumer-driven purpose? If no, use the concrete type.
- Does this helper express a real concept? If no, inline it.
- Does this abstraction reduce total complexity? If no, remove it.
- Would plain TypeScript be easier to understand? If yes, use plain TypeScript.

## 75. Desired style

Prefer code like:

```typescript
async function mapDomain(event: DomainMapEvent): Promise<void> {
  const domain = event.domain.trim().toLowerCase()e

  await this.domains.map(event.projectId, domain);
}
```

over:

```typescript
async function mapDomain(raw: any): Promise<void> {
  const event = convertToDomainMapEvent(raw);
  if (!event) {
    throw new Error("failed converting domain event");
  }

  const projectId = safeString(event.projectId);
  if (!projectId) return;

  const domain = normalizeStringValue(event.domain);
  if (!domain) return;

  await this.domainManager.processDomainMapping(
    new DomainMappingParams(projectId, domain),
  );
}
```

## 76. Refactoring process

Work feature-by-feature.

**Step 1** — Identify where data enters the system: HTTP, a queue (Kafka/SQS/RabbitMQ), a database, a webhook, an external API, environment/config.

**Step 2** — Give the input a concrete type.

**Step 3** — Validate/parse once (with a schema library at real trust boundaries).

**Step 4** — Follow the data through the application. Remove unnecessary `any`, type assertions, generic `Record<string, unknown>` bags, null guards that don't correspond to real absence, converters, extractors, wrapper types, interfaces, pass-through methods, duplicate DTOs, and generic helpers.

**Step 5** — Collapse forwarding layers.

**Step 6** — Delete dead abstractions.

**Step 7** — Run tests and static analysis using the project's normal commands, including where applicable:

```
tsc --noEmit
eslint .
npm test  (or the project's test runner)
prettier --write .
```

Do not change behavior merely to satisfy style preferences.

## 77. Preserve things that are genuinely idiomatic

Do not remove:

- useful interfaces with real multiple implementations
- meaningful error context (`{ cause: err }`, custom error classes with real fields)
- legitimate optional/null handling for genuinely optional data
- boundary validation
- proper resource cleanup (`try/finally`, `using`/`Symbol.dispose` where applicable)
- correct `async`/`await` and promise error propagation
- real event-driven code where decoupling is genuinely needed
- driver-specific error handling
- security-related checks (auth, input sanitization, CSRF/XSS protections)

This is a complexity cleanup, not reckless deletion.

## 78. Final desired result

The codebase should end up with:

- more concrete types/interfaces
- fewer `any`/loose `Record<string, unknown>` values
- fewer generic object bags
- fewer type assertions (`as`)
- fewer conversion helpers
- fewer tiny wrapper functions
- fewer pointless interfaces
- fewer forwarding layers
- fewer factories/builders/managers
- less dynamic/reflective code
- less defensive null handling
- less silent fallback-to-default behavior
- simpler error handling
- fewer redundant DTOs
- more direct function calls
- narrower function signatures
- explicit business logic
- validation concentrated at boundaries
- straightforward idiomatic TypeScript

The important metric is not the number of files changed. The important metric is:

> Can an engineer trace the behavior without jumping through unnecessary abstractions?

## 79. Most important rule

Do not replace one kind of slop with another.

Do not turn:

```typescript
const value = (data as any).domain as string;
```

into:

```typescript
const obj = data as Record<string, unknown>;
if (typeof obj.domain !== "string") {
  throw new Error("invalid domain");
}
const value = obj.domain;
```

and call that a cleanup. The correct solution is usually:

```typescript
interface Event {
  domain: string;
}
```

followed by `event.domain`.

Likewise, do not replace a simple direct call with:

```
interface -> implementation -> manager -> helper -> converter -> validator -> actual function
```

The overriding principle is:

> Make untrusted input safe at the edge. Keep trusted TypeScript code boring everywhere else.
