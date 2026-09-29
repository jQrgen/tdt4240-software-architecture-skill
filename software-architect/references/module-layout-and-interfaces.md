# From architecture to code: module layouts, interfaces, skeletons per ecosystem

Use this file when a design decision has to become directories, build modules, interfaces and compiling code.
Upstream: the quality-attribute scenarios and chosen patterns from [design-workflow.md](design-workflow.md) and
[architectural-patterns.md](architectural-patterns.md). Downstream: the rules you lay out here get enforced by
[fitness-functions.md](fitness-functions.md) and recorded in [../templates/adr.md](../templates/adr.md). Work
through the sections in order; for existing code, use §8 instead of §7.

## 1. Layout principles

### 1.1 Package by feature or by layer

| Situation | Choose | Why |
|---|---|---|
| Most changes touch one business capability end to end | **By feature** at the top level, layers inside each feature | Keeps a change inside one directory. Cohesion follows the change (SAiP *increase cohesion*) |
| Small service (one bounded context, < ~10k LOC) | **By layer** (`domain/`, `application/`, `adapters/`) | Only one feature exists, so a feature split adds nothing |
| Several teams or features that must not know each other | **By feature as build modules** plus shared `core` modules | The build plus a module-graph check can forbid feature-to-feature edges (§3) |
| Library or SDK | **By public API surface** (`api` vs `impl`) | Consumers see a stable, small surface |

Default: **feature outside, layer inside**. A multi-feature app whose top level is only `controllers/ services/
repositories/ models/`, where every feature change touches all four, has low cohesion: report it.

### 1.2 The dependency rule

- Source dependencies point **toward policy** (domain, use cases) and **away from detail** (frameworks, I/O, UI).
- The domain imports nothing from frameworks, drivers, HTTP, SQL, UI or DI containers. It may use the standard
  library and deliberately chosen value-type libraries such as a money or time library.
- Only the data or adapter layer touches a data source: a database handle, socket, file system, SDK client or wallet
  object. A type that holds one of these outside that layer is a violation.
- Nothing points back up: a view-model does not know views exist; a repository never navigates or shows errors. An
  upward call is a **layer-bridging cycle**, worse than a layer skip. Invert it with a result, event or callback port.

### 1.3 Package-level principles (R. C. Martin, *Clean Architecture*; not SAiP terminology)

- **Acyclic Dependencies (ADP):** the module graph is a DAG. Break cycles by inverting the dependency (move the
  interface to the consumer) or by extracting the shared part into a new module.
- **Stable Dependencies (SDP):** depend toward stability; a module many depend on should itself depend on little.
- **Stable Abstractions (SAP):** stable modules should be abstract (interfaces, value types). Metrics for instability
  and abstractness: [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md).
- In SAiP terms these implement the *reduce coupling* modifiability tactics (*encapsulate*, *use an intermediary*,
  *restrict dependencies*, *abstract common services*); see [quality-attributes-change.md](quality-attributes-change.md).

### 1.4 Structural rules to apply by default

1. **One composition root per executable** (`main`, `Application`, `App.kt`, `Program.cs`): the only place that
   constructs adapters and wires use cases. Service locators and `getInstance()` singletons elsewhere are findings.
2. **Single source of truth per data slice.** One repository/store owns each slice's caching, refresh and write-back
   policy. Two handles reaching the same data is a finding. Code-level signals: the same DAO/table/client type
   constructed or injected in more than one non-data module; two `MutableStateFlow`s or store caches holding the same
   entity; SharedPreferences/DataStore/`localStorage` read outside the owning repository.
3. **Tiny shared kernel.** `core`/`shared`/`common` holds value types and small pure utilities. Once it grows business
   logic or becomes everyone's dependency for unrelated reasons, it is a coupling hub: split it or push code back.
4. **Ports are owned by the consumer.** `PaymentGateway` lives beside the use case that needs it, not beside the
   Stripe adapter; the adapter depends on the port. That is what makes dependencies point inward.
5. **Cross-cutting concerns live at the edges:** logging, tracing, metrics, auth, transactions and retries go in
   adapters, middleware, decorators or interceptors wired at the root. If the domain must log, inject a port.
6. **No feature depends on another feature.** Cross-feature navigation and data go through a shared contract
   module (routes, events, or a port in `core`). Keep fakes for every port in one `testing`/`testFixtures` module.

## 2. The allowed-dependency table

Write this table before creating directories. Put it in the ADR and in the architecture description. **Read as: the
row may depend on the column.** Anything not marked is forbidden.

| row → may use ↓ col | domain | application | adapters/in | adapters/out | platform | shared-kernel | composition root |
|---|---|---|---|---|---|---|---|
| **domain** | – | | | | | ✓ | |
| **application** | ✓ | – | | | | ✓ | |
| **adapters/in** (HTTP, CLI, UI) | ✓ (types only) | ✓ | – | | ✓ | ✓ | |
| **adapters/out** (DB, HTTP clients) | ✓ | ✓ (ports only) | | – | ✓ | ✓ | |
| **platform** (expect/actual, OS APIs) | | | | | – | ✓ | |
| **shared-kernel** | | | | | | – | |
| **composition root** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | – |

State under the table: `adapters/in` never imports `adapters/out` (a controller opening a DB connection is a
violation); feature `X` never depends on feature `Y` (list exceptions with reason and expiry); only `adapters/out` may
import `java.sql`, `jakarta.persistence`, `pg`, `sqlalchemy`, `database/sql`, `System.Data`, HTTP client libraries and
cloud SDKs; HTTP server frameworks (Spring Web, express, FastAPI, `net/http` handlers) appear only in `adapters/in` and
the composition root.
**Name concrete packages**: a rule that says "infrastructure" cannot be checked.

Each ✓ and each named package becomes one assertion in [fitness-functions.md](fitness-functions.md).

**Variant: one feature package inside a package-by-layer app.** Most brownfield work adds a layered feature
(`app/notifications/`) to a host app organised as `api/`, `core/`, `models/`, `crud/`. Do not restructure the host;
give the feature its own layers and state how it may touch the host's columns:

| row → may use ↓ col | feature/domain | feature/service | feature/adapters | feature/wiring | host `models` | host `core` (config, db) | host `api` |
|---|---|---|---|---|---|---|---|
| **feature/domain** | – | | | | | | |
| **feature/service** (use cases, ports) | ✓ | – | | | | | |
| **feature/adapters** (DB rows, SMTP, HTTP) | ✓ | ✓ (ports) | – | | ✓ | ✓ | |
| **feature/wiring** (providers, lifespan hooks) | ✓ | ✓ | ✓ | – | | ✓ | |
| **host `api`** (routes, deps) | | ✓ (entry functions only) | | ✓ | ✓ | ✓ | – |
| **host `models` / `core`** | | | | | – | – | |

The host never imports feature internals except the service entry points and the wiring; the feature never imports
host routes. Enforce with import-linter `protected` (feature adapters importable only by feature wiring) and
`forbidden` (host `models`/`core` never import the feature).

## 3. Enforcement by language and build system

Prefer the **compiler** over a **linter**, and a linter over **convention**. Record the mechanism beside each rule.

| Ecosystem | Mechanism | Enforced by | Notes |
|---|---|---|---|
| Gradle (JVM/Android/KMP) | Separate modules; `implementation` vs `api` configurations | **Compiler** (classpath) | `implementation` deps are not on consumers' compile classpath. Use `api` only for types that appear in your public signatures. `api` exists only with `java-library` (or `com.android.library` / `kotlin("multiplatform")`), not with plain `java`/`application` |
| Kotlin | `internal` visibility | **Compiler** | Visible within one Kotlin module (compilation unit) only. Make adapters `internal` and expose a factory |
| KMP | Source sets `commonMain`/`androidMain`/`iosMain`/`jvmMain`; `expect`/`actual` | **Compiler** | `commonMain` cannot see platform APIs. Keep the platform surface small, in a `:core:platform` module. Prefer `expect fun` or a `commonMain` interface with platform implementations wired in `:app`; `expect class` is Beta and emits a compiler warning |
| Java 9+ | JPMS `module-info.java` `exports` / `requires` | **Compiler + runtime** | Non-exported packages are inaccessible. Many apps run on the classpath where this does not apply, so check first |
| Java | package-private (no modifier) | **Compiler** | Cheap, effective boundary. Make the adapter class package-private and expose the port through a public static factory (or a public `@Configuration` in the same package) that returns the port type, since the composition root lives in another package |
| JVM (any) | ArchUnit / Konsist (Kotlin) rules in tests | **Test** (CI) | For rules the module graph cannot express, such as naming or annotations. See [fitness-functions.md](fitness-functions.md) |
| TypeScript | Workspaces (npm/pnpm/yarn) + `package.json` `"exports"` | **Resolver** (Node, and TS with `moduleResolution` `node16`/`nodenext`/`bundler`) | Deep imports into non-exported paths fail to resolve. Undeclared cross-package imports are rejected only with pnpm's isolated `node_modules` or Yarn PnP; with npm or Yarn classic, hoisting lets them resolve, so add dependency-cruiser or eslint `import/no-extraneous-dependencies` |
| TypeScript | Project references (`composite: true`, `references`) | **Compiler** (partial) | Gives build order and forces declared edges. It is not a full layering checker |
| TypeScript | dependency-cruiser, eslint-plugin-boundaries | **Linter** (CI) | For layer rules inside one package |
| Nx monorepo | `@nx/enforce-module-boundaries` with project tags (Nx 16+; older: `@nrwl/nx/enforce-module-boundaries`) | **Linter** | Use only if the repo already uses Nx. Do not introduce Nx just for this |
| Python | Leading underscore `_module`, `__all__` | **Convention** | Nothing stops `from pkg._impl import x` |
| Python | import-linter contracts (`layers`, `forbidden`, `independence`, `protected`, `acyclic_siblings`), run with `lint-imports` | **Linter** (CI) | The practical enforcement for Python. See §4.4 |
| Go | `internal/` directories | **Compiler** (go tool) | Importable only from within the tree rooted at the parent of `internal/` |
| Go | `cmd/<binary>/main.go`; no `pkg/` by default | **Convention** | One composition root per binary. Use `pkg/` only if the repo already does or to mark a deliberate public library API |
| Go | go-arch-lint, or a `go list -deps` script in CI | **Linter** | For layer rules inside `internal/` |
| Rust | Crates in a workspace; `pub(crate)`, `pub(super)` | **Compiler** | A crate boundary is the layer boundary. The dependency graph is in `Cargo.toml` |
| .NET | Separate projects (`ProjectReference`) + `internal`; `InternalsVisibleTo` | **Compiler** | `internal` = same assembly. Grant `InternalsVisibleTo` to test projects only, never to bypass a production boundary |
| .NET | NetArchTest / ArchUnitNET | **Test** | For rules inside one project |

Module systems enforce only the edges you declare: nothing stops a PR from adding
`implementation(project(":feature:orders"))` to another feature, a new `ProjectReference`, or a new Cargo dependency.
Also check the declared graph in CI (a small Gradle task or test over project dependencies, a test over `.csproj`
`ProjectReference`s, `cargo metadata` for Rust); see [fitness-functions.md](fitness-functions.md).

Tool config syntax changes between major versions: check the repo's tool version first and note it in the ADR.

## 4. Blueprints

Pick one blueprint and adapt it to the repo's conventions. Never add a second style beside an existing one without an ADR.

### 4.1 Kotlin Multiplatform app (multi-module, view → view-model → repository, UDF)

The standard Google/JetBrains app layering. A review of a KMP wallet app found that its single `:shared` module let
composables read global singletons and application code call navigation, because Kotlin `internal` enforces nothing
inside one module. The review proposed this split so that a view reaching for a database would no longer compile.

```
:core:model        commonMain/  value types only
:core:platform     commonMain/  interfaces or `expect fun`s (clipboard, prefs, clock)
                   androidMain/ iosMain/ jvmMain/   platform implementations
:core:data         commonMain/  repository interfaces + impls; ONLY module with DB/HTTP/SDK handles
:core:domain       commonMain/  use cases over repository interfaces (optional tier)
:core:ui           commonMain/  theme, shared components (no feature logic)
:core:testing      commonMain/  fakes for every repository/port
:feature:checkout  commonMain/  CheckoutScreen.kt, CheckoutViewModel.kt, CheckoutUiState.kt
:feature:orders    commonMain/  ...                (no feature -> feature edges)
:app               commonMain/  composition root, navigation graph; androidMain/ iosMain/ entry points
```
Rules: `feature → (domain →) data → model`; a feature may depend on `:core:data` directly when no use-case tier
exists. `:core:ui` is for feature modules and `:app` only; `:core:platform` is for `:core:data` and `:app` (domain
reaches it only through a port such as `Clock`). One `StateFlow<XUiState>` per screen (immutable data class or sealed
Loading/Ready/Error); intents flow up; one-shot events per §7.5 (state field or channel, one choice per codebase).
Add the use-case tier only when two view-models share logic. Here repository interfaces live in `:core:data`; the
hexagonal variant (§4.2, §7) puts ports beside the use case and reverses the edge (adapters depend on domain).

### 4.2 JVM / Spring Boot hexagonal (ports and adapters)

```
ordering/                          (Gradle or Maven multi-module, or packages + ArchUnit)
  domain/        Order, OrderLine, Money            no Spring, no JPA annotations
  application/   PlaceOrder, port/in/PlaceOrderUseCase, port/out/OrderRepository, PaymentGateway
  adapters/in/web/        OrderController (@RestController) -> PlaceOrderUseCase
  adapters/out/persistence/ JpaOrderRepository, OrderEntity (JPA lives HERE only)
  adapters/out/payment/     HttpPaymentGateway
  bootstrap/     Application.kt/java, @Configuration beans = composition root
```
Keep `@Entity` out of `domain` and map at the adapter. That costs one mapper per aggregate: pay it when the domain has
invariants worth protecting. For CRUD without invariants, a plain layered Spring app plus an ADR is fine.

### 4.3 TypeScript service or monorepo

```
packages/ordering/            (or a single-package src/ with dependency-cruiser rules)
  src/domain/                 pure TS, no imports from node:*, express, pg, etc.
  src/application/            use cases + ports (interfaces)
  src/adapters/http/          express/fastify/hono routes -> use cases
  src/adapters/postgres/      OrderRepository impl
  src/adapters/payments/      PaymentGateway impl
  src/main.ts                 composition root: construct adapters, inject, start server
  package.json                "exports": { ".": "./dist/index.js" }  (public surface only)
```
In a monorepo, shared contracts (DTO schemas, generated clients) go in `packages/contracts`; business logic never does.

### 4.4 Python (src layout + import-linter)

```
src/shop/
  domain/        order.py (dataclasses, no ORM)
  application/   ports.py (typing.Protocol), place_order.py
  adapters/inbound/   api.py (FastAPI/Flask routes)
  adapters/outbound/  sql_orders.py, http_payments.py
  main.py        composition root
tests/           fakes.py, test_place_order.py
```
import-linter in `pyproject.toml` (layers listed highest first; higher may import lower):
```toml
[tool.importlinter]
root_package = "shop"
include_external_packages = true   # required for forbidding third-party packages
[[tool.importlinter.contracts]]
name = "Hexagonal layers"
type = "layers"
layers = ["shop.main", "shop.adapters", "shop.application", "shop.domain"]
[[tool.importlinter.contracts]]
name = "Domain is framework-free"
type = "forbidden"
source_modules = ["shop.domain", "shop.application"]
forbidden_modules = ["sqlalchemy", "fastapi", "requests"]
[[tool.importlinter.contracts]]
name = "Inbound adapters never reach outbound adapters"
type = "forbidden"
source_modules = ["shop.adapters.inbound"]
forbidden_modules = ["shop.adapters.outbound"]
```
Run `lint-imports`. Fuller contracts: [fitness-functions.md](fitness-functions.md) §2.4.

### 4.4b FastAPI + SQLModel/SQLAlchemy + Alembic (brownfield)

Adding a feature package to an existing app (e.g. the FastAPI full-stack template layout `app/api`, `app/core`,
`app/models.py`, `app/crud.py`, `app/alembic`):

- **Composition root per request:** FastAPI `Depends` providers (typically `app/api/deps.py`) are where ports get their
  adapters. Add `get_notifier(session: SessionDep) -> Notifier` there or in the feature's `wiring.py`; routes depend on
  the provider, never construct adapters. Tests replace providers with `app.dependency_overrides[get_notifier] = ...`.
- **One `wiring.py` per process:** the web app and a worker process (`python -m app.worker`) each build their graph in
  one place; the worker does not import FastAPI.
- **In-process background resources** (pollers, schedulers, client pools) start and stop in the `lifespan` handler
  (`FastAPI(lifespan=...)`), not at import time and not in deprecated `@app.on_event`. Whether a separate worker
  process is possible depends on the host (recon: deploy target and process model).
- **Alembic sees only imported models.** A new SQLModel/SQLAlchemy table must be imported where `env.py` builds
  `target_metadata` (often via `from app.models import SQLModel`), or `alembic revision --autogenerate` silently
  misses it. Put new models in a module that module imports, then read the generated migration.
- **Who owns the unit of work.** An outbox or job insert must join the caller's session and commit with the business
  change. Existing CRUD helpers that call `session.commit()` internally break this: either add non-committing
  variants (`flush()` only) and commit once in the route/service, or enqueue after the helper's commit and accept the
  gap (state it in the ADR).
- **Tests and the root `conftest.py`:** templates often have an autouse fixture that opens a real DB session for every
  test. Put pure unit tests for the feature where they do not inherit it (a separate test root), or run them with
  `pytest --noconftest`; do not make domain tests need Postgres.

Django equivalent: a feature is an app (`INSTALLED_APPS`); wire signal handlers and start nothing heavy in
`AppConfig.ready()`; enqueue side effects with `transaction.on_commit(lambda: ...)` so they run only after commit
(or write an outbox row inside the transaction when the effect must not be lost).

### 4.5 Go

```
cmd/orderd/main.go                 composition root (flags/env -> config -> wire -> run); go.mod at repo root
internal/ordering/domain/          order.go (types + invariants, stdlib only)
internal/ordering/app/             place_order.go + ports (interfaces declared HERE, by the consumer)
internal/ordering/adapters/postgres/  repo.go (database/sql)
internal/ordering/adapters/httpapi/   handler.go (net/http)
internal/platform/                 logging, config helpers (tiny)
```
Go idiom: accept interfaces, return structs; declare interfaces in the consuming package; `context.Context` first.

### 4.6 .NET clean architecture

```
src/Ordering.Domain/          entities, value objects; references nothing
src/Ordering.Application/     use cases, ports (IOrderRepository, IPaymentGateway); refs Domain
src/Ordering.Infrastructure/  EF Core DbContext, HTTP clients; refs Application (implements ports)
src/Ordering.Api/             ASP.NET Core endpoints + Program.cs composition root; refs Application, Infrastructure
tests/Ordering.Application.Tests/   fakes; InternalsVisibleTo only for tests
```
The `ProjectReference` graph enforces it. Make infrastructure classes `internal` and expose one
`AddInfrastructure(this IServiceCollection services)` extension method for `Program.cs` to call.

## 5. Designing the interfaces (SAiP 4e ch. 15, *Software Interfaces*)

An interface is what an element provides and requires, and what others may assume about it; everything behind it is
hidden (information hiding, Parnas). For each boundary interface, decide and write down:

| Decision | Options and guidance |
|---|---|
| **Scope** | Who may call it: in-process within a module, across modules, across processes, public. Expose the smallest set of operations. Give each client role its own interface rather than one fat one |
| **Interaction style** | Sync call or RPC (gRPC, in-process calls): simple, but couples caller availability to callee availability. REST: resource-oriented, cache-friendly. Messaging or events: decouples in time, but adds eventual consistency and ordering questions. Streaming: continuous data or back-pressure. Choose from the QA scenario: availability under callee failure (temporal decoupling), absorbing load spikes and caller-perceived response time favour async; tight end-to-end request/response latency, read-your-writes and simplicity favour sync |
| **Data representation** | JSON (readable, loose), Protocol Buffers/Avro (schema-evolved, compact), domain types in-process. Never leak persistence entities or framework types across the boundary |
| **Error semantics** | Enumerate the failure cases in the contract: domain rejection vs transient failure vs caller bug. State which are retryable. Specify timeout behaviour. Use HTTP status plus a problem body (RFC 9457), gRPC status codes, or typed results |
| **Idempotency** | Any operation a client may retry (all network calls) takes an idempotency key or is naturally idempotent (PUT by id). The key must come from the caller (e.g. an `Idempotency-Key` header) and be reused on retry; a key generated inside the server per call only protects retries inside the adapter. The §7 skeletons pass the caller's `requestId` through to the payment port |
| **Versioning and evolution** | Prefer *additive* changes: new optional fields and new operations. Readers ignore unknown fields (*tolerant reader*). Postel's "be liberal in what you accept" has a caveat: over-liberal acceptance entrenches peers' bugs (see RFC 9413), so reject malformed input and tolerate only unknown-but-well-formed input. For breaking changes, use a new version (path, header, package `v2`, protobuf package), run both, and deprecate on a published schedule. Never reuse protobuf field numbers; reserve them |
| **Contract-first** | For cross-team or cross-process APIs, write the contract (OpenAPI for HTTP, `.proto` for gRPC, AsyncAPI for message channels) before the code. Generate stubs and run a compatibility check in CI (for protobuf, e.g. `buf breaking`) |
| **QA properties** | State latency, throughput limits and rate limits in the contract. Callers design timeouts from them |

For the Views and Beyond interface documentation template, see [documentation.md](documentation.md).

## 6. Cross-cutting decisions checklist

Decide each once per system, record it as an ADR, and wire it in the composition root.

- [ ] **Error model.** Choose exceptions internally or `Result`/sealed types/`error` values. At boundaries, translate:
  adapters convert driver exceptions into domain or port errors, and inbound adapters convert them into
  HTTP/gRPC/UI states. Expected outcomes (declined, out of stock) are values. No `SQLException` in the domain.
- [ ] **Concurrency model.** Structured concurrency: Kotlin coroutine scopes, Python `asyncio.TaskGroup` (3.11+), Go
  `context.Context` + `errgroup` (x/sync), Java `StructuredTaskScope` only where the JDK has it (preview status varies).
  **Inject** dispatchers/executors; hard-coded `Dispatchers.IO` or `time.Sleep` in logic is a finding. Cancellation
  propagates, and every blocking call has a timeout.
- [ ] **Configuration.** Read once at the root into a typed, validated object; fail fast. No `process.env`/`os.environ` below it.
- [ ] **Observability hooks.** Put structured logging, tracing (OpenTelemetry API) and metrics in adapters, middleware
  or decorators. Propagate a correlation or trace id through the context. Decide log levels and PII rules.
- [ ] **Transaction and consistency boundaries.** One transaction per use case, opened by the application layer
  through a unit-of-work port or by the adapter, never in the domain. If a use case writes to a DB *and* publishes
  a message, decide explicitly: transactional outbox, idempotent consumer, or accepted risk.
- [ ] **Time and randomness as ports.** Inject `Clock` (`java.time.Clock`; `kotlin.time.Clock` on Kotlin 2.1.20+ with
  `@OptIn(ExperimentalTime::class)`, or `kotlinx.datetime.Clock` on older setups, deprecated from kotlinx-datetime
  0.7; a Go `func() time.Time`; a Python callable) and id/random generators.
- [ ] **Security boundary.** Authenticate in inbound adapters, authorize in the application layer, validate at trust boundaries.
- [ ] **Serialization.** DTO ↔ domain mapping lives in adapters; one serialization library per boundary.

## 7. Skeleton recipes: one feature in four languages

Feature *place order*: `Order` with an invariant, ports `OrderRepository` and `PaymentGateway`, the `PlaceOrder` use
case, an adapter stub, a fake and the composition root. Replace `*Placeholder` types with the repo's real clients.

Two decisions every recipe below makes explicitly; keep them when you copy it:
- **Idempotency key from the caller.** `requestId` is generated once per user submission by the client (or read
  from an `Idempotency-Key` header by the inbound adapter) and reused on retry. It becomes both `Order.id` and the
  payment key, so a retried request cannot charge twice. Never mint the key inside the use case.
- **Charge-then-save is an accepted risk.** A crash between the two leaves a charge with no order until the client
  retries with the same `requestId` (the gateway replays the original result; `save` must be an idempotent upsert).
  If clients may not retry, save the order as PENDING first, then charge, then mark it PAID/REJECTED, or use an
  outbox plus reconciliation (§6).

### 7.1 Kotlin (JVM or KMP `commonMain`; coroutines assumed)

Hexagonal module names: `:ordering:adapters` depends on `:ordering:domain` (ports live with the use case). Do not
combine this with §4.1's `:core:domain → :core:data` edge; pick one direction per repo and record it in the §2 table.
```kotlin
// :ordering:domain — Order.kt
data class OrderLine(val sku: String, val qty: Int, val unitPriceCents: Long)
data class Order(val id: String, val customerId: String, val lines: List<OrderLine>) {
    init { require(lines.isNotEmpty()) { "order needs at least one line" } }
    val totalCents: Long get() = lines.sumOf { it.qty * it.unitPriceCents }
}
// :ordering:domain — ports owned by the use case
interface OrderRepository { suspend fun save(order: Order) }  // idempotent upsert by order.id
sealed interface Payment { data class Approved(val ref: String) : Payment; data class Declined(val reason: String) : Payment }
interface PaymentGateway { suspend fun charge(customerId: String, cents: Long, idempotencyKey: String): Payment }
sealed interface PlaceOrderResult { data class Placed(val order: Order) : PlaceOrderResult; data class Rejected(val reason: String) : PlaceOrderResult }
class PlaceOrder(private val orders: OrderRepository, private val payments: PaymentGateway) {
    // requestId comes from the caller so a retried request reuses it
    suspend operator fun invoke(requestId: String, customerId: String, lines: List<OrderLine>): PlaceOrderResult {
        if (lines.isEmpty()) return PlaceOrderResult.Rejected("empty order")
        val order = Order(requestId, customerId, lines)
        // charge-then-save: accepted risk, recovered by client retry with the same requestId (see above, §6)
        return when (val p = payments.charge(customerId, order.totalCents, idempotencyKey = requestId)) {
            is Payment.Declined -> PlaceOrderResult.Rejected(p.reason)
            is Payment.Approved -> { orders.save(order); PlaceOrderResult.Placed(order) }
        }
    }
}
// :ordering:adapters — adapter stub, internal; only the factory is public
internal class SqlOrderRepository(private val db: DbHandlePlaceholder) : OrderRepository {
    override suspend fun save(order: Order) = TODO("upsert Order rows via the project's DB client")
}
fun sqlOrderRepository(db: DbHandlePlaceholder): OrderRepository = SqlOrderRepository(db)
// :ordering:testing — fake
class FakePaymentGateway(var next: Payment = Payment.Approved("test")) : PaymentGateway {
    val keys = mutableListOf<String>()
    override suspend fun charge(customerId: String, cents: Long, idempotencyKey: String) = next.also { keys += idempotencyKey }
}
// :app — composition root
class AppGraph(db: DbHandlePlaceholder, payments: PaymentGateway) {
    val placeOrder = PlaceOrder(sqlOrderRepository(db), payments)
}
```
KMP note: `java.util.UUID` is not available in `commonMain`. Generate `requestId` in the client (the view-model, §7.5)
from a platform implementation or the multiplatform UUID source the repo already uses (`kotlin.uuid.Uuid` needs Kotlin
2.0.20+ and `@OptIn(ExperimentalUuidApi::class)`).

### 7.2 TypeScript

```ts
// src/domain/order.ts
export type OrderLine = Readonly<{ sku: string; qty: number; unitPriceCents: number }>;
export type Order = Readonly<{ id: string; customerId: string; lines: readonly OrderLine[]; totalCents: number }>;
export function createOrder(id: string, customerId: string, lines: readonly OrderLine[]): Order {
  if (lines.length === 0) throw new Error("order needs at least one line"); // programming error, caller checks first
  return { id, customerId, lines, totalCents: lines.reduce((s, l) => s + l.qty * l.unitPriceCents, 0) };
}
// src/application/ports.ts
export interface OrderRepository { save(order: Order): Promise<void> } // idempotent upsert by order.id
export type Payment = { kind: "approved"; ref: string } | { kind: "declined"; reason: string };
export interface PaymentGateway { charge(customerId: string, cents: number, idempotencyKey: string): Promise<Payment> }
// src/application/place-order.ts
export type PlaceOrderResult = { kind: "placed"; order: Order } | { kind: "rejected"; reason: string };
export class PlaceOrder {
  constructor(private readonly orders: OrderRepository, private readonly payments: PaymentGateway) {}
  // requestId comes from the caller so a retried request reuses it
  async execute(requestId: string, customerId: string, lines: readonly OrderLine[]): Promise<PlaceOrderResult> {
    if (lines.length === 0) return { kind: "rejected", reason: "empty order" };
    const order = createOrder(requestId, customerId, lines);
    const p = await this.payments.charge(customerId, order.totalCents, requestId); // charge-then-save: see above, §6
    if (p.kind === "declined") return { kind: "rejected", reason: p.reason };
    await this.orders.save(order);
    return { kind: "placed", order };
  }
}
// src/adapters/postgres/order-repository.ts
export class PgOrderRepository implements OrderRepository {
  constructor(private readonly db: DbClientPlaceholder) {}
  async save(_order: Order): Promise<void> { throw new Error("TODO: upsert via the project's DB client"); }
}
// test/fakes.ts
export class FakePaymentGateway implements PaymentGateway {
  keys: string[] = []; next: Payment = { kind: "approved", ref: "test" };
  async charge(_c: string, _cents: number, key: string) { this.keys.push(key); return this.next; }
}
// src/adapters/http: const requestId = req.get("Idempotency-Key"); if (!requestId) respond 400; else placeOrder.execute(...)
// src/main.ts — composition root
const placeOrder = new PlaceOrder(new PgOrderRepository(db), payments); // db, payments built above from config
```
Constructor parameter properties (`constructor(private readonly x: T)`) fail under TS 5.8 `erasableSyntaxOnly` and
Node's built-in type stripping. If the repo uses either, declare fields explicitly and assign them in the constructor.

### 7.3 Python (3.10+)

```python
# src/shop/domain/order.py
@dataclass(frozen=True)
class OrderLine:
    sku: str; qty: int; unit_price_cents: int

@dataclass(frozen=True)
class Order:
    id: str; customer_id: str; lines: tuple[OrderLine, ...]
    def __post_init__(self) -> None:
        if not self.lines: raise ValueError("order needs at least one line")  # invariant; use case checks first
    @property
    def total_cents(self) -> int: return sum(l.qty * l.unit_price_cents for l in self.lines)

# src/shop/application/ports.py
class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...  # idempotent upsert by order.id
class EmptyOrder(Exception): ...
class PaymentDeclined(Exception): ...
class PaymentGateway(Protocol):  # returns a ref, or raises PaymentDeclined
    def charge(self, customer_id: str, cents: int, idempotency_key: str) -> str: ...

# src/shop/application/place_order.py
class PlaceOrder:
    def __init__(self, orders: OrderRepository, payments: PaymentGateway) -> None:
        self._orders, self._payments = orders, payments
    def __call__(self, request_id: str, customer_id: str, lines: tuple[OrderLine, ...]) -> Order:
        if not lines: raise EmptyOrder()
        order = Order(request_id, customer_id, lines)  # request_id from the caller, reused on retry
        self._payments.charge(customer_id, order.total_cents, idempotency_key=request_id)  # charge-then-save: see above
        self._orders.save(order)
        return order

# src/shop/adapters/outbound/sql_orders.py (stub)
class SqlOrderRepository:
    def __init__(self, conn: "DbConnectionPlaceholder") -> None: self._conn = conn
    def save(self, order: Order) -> None: raise NotImplementedError("upsert via the project's DB client")

# tests/fakes.py
class FakePaymentGateway:
    def __init__(self) -> None: self.keys: list[str] = []; self.decline: str | None = None
    def charge(self, customer_id: str, cents: int, idempotency_key: str) -> str:
        if self.decline: raise PaymentDeclined(self.decline)
        self.keys.append(idempotency_key); return "test-ref"

# src/shop/main.py (composition root)
place_order = PlaceOrder(SqlOrderRepository(conn), payments)
```
Imports (`dataclasses.dataclass`, `typing.Protocol`, cross-module types) are omitted. This variant uses exceptions for
the expected outcomes (`EmptyOrder`, `PaymentDeclined`, both declared in `ports.py`) to show the other error model.
Pick one model per codebase (§6).

### 7.4 Go

```go
// internal/ordering/domain/order.go
package domain

type OrderLine struct { SKU string; Qty int; UnitPriceCents int64 }
type Order struct { ID, CustomerID string; Lines []OrderLine }
var ErrEmptyOrder = errors.New("order needs at least one line")

func NewOrder(id, customerID string, lines []OrderLine) (Order, error) {
	if len(lines) == 0 { return Order{}, ErrEmptyOrder }
	return Order{ID: id, CustomerID: customerID, Lines: lines}, nil
}
func (o Order) TotalCents() (t int64) { for _, l := range o.Lines { t += int64(l.Qty) * l.UnitPriceCents }; return }

// internal/ordering/app/place_order.go — ports declared by the consumer
package app
type OrderRepository interface{ Save(ctx context.Context, o domain.Order) error } // idempotent upsert by ID
type PaymentGateway interface {
	Charge(ctx context.Context, customerID string, cents int64, idempotencyKey string) (ref string, err error) // ErrDeclined on decline
}
var ErrDeclined = errors.New("payment declined")
type PlaceOrder struct { Orders OrderRepository; Payments PaymentGateway }
// requestID comes from the caller (Idempotency-Key header) so a retried request reuses it.
func (uc PlaceOrder) Execute(ctx context.Context, requestID, customerID string, lines []domain.OrderLine) (domain.Order, error) {
	o, err := domain.NewOrder(requestID, customerID, lines)
	if err != nil { return domain.Order{}, err }
	if _, err := uc.Payments.Charge(ctx, customerID, o.TotalCents(), requestID); err != nil { // charge-then-save: see above
		return domain.Order{}, fmt.Errorf("charge: %w", err)
	}
	if err := uc.Orders.Save(ctx, o); err != nil { return domain.Order{}, fmt.Errorf("save: %w", err) }
	return o, nil
}
// adapters/postgres/repo.go: type Repo struct{ DB *sql.DB }  (Save = upsert, stub); app/fakes_test.go: type fakePayments struct{ keys []string; err error }
// cmd/orderd/main.go:  uc := app.PlaceOrder{Orders: postgres.Repo{DB: db}, Payments: pay}
```
Imports (`context`, `errors`, `fmt`, the domain package) are omitted. The stdlib has no UUID generator; ids come from
the client, or use the repo's existing generator or `crypto/rand` for server-assigned ids not used for dedup.

### 7.5 UI variant: view → view-model → repository, immutable UiState, UDF

Kotlin / Compose (Multiplatform `ViewModel` needs a recent `androidx.lifecycle`; check the repo's version):
```kotlin
data class CheckoutUiState(val lines: List<OrderLine> = emptyList(), val placing: Boolean = false, val error: String? = null,
                           val placedOrderId: String? = null)  // option (a): one-shot event as state
sealed interface CheckoutIntent { data object Submit : CheckoutIntent; data object Navigated : CheckoutIntent }  // data object: Kotlin 1.9+
class CheckoutViewModel(private val placeOrder: PlaceOrder, private val customerId: String, private val newId: () -> String) : ViewModel() {
    private val _state = MutableStateFlow(CheckoutUiState()); val state: StateFlow<CheckoutUiState> = _state.asStateFlow()
    private var requestId = newId()                              // one per submission, reused if the user retries
    fun onIntent(i: CheckoutIntent) { when (i) {
        CheckoutIntent.Submit -> submit()
        CheckoutIntent.Navigated -> _state.update { it.copy(placedOrderId = null) }  // view acknowledges the event
    } }
    private fun submit() { viewModelScope.launch {
        _state.update { it.copy(placing = true, error = null) }
        when (val r = placeOrder(requestId, customerId, _state.value.lines)) {
            is PlaceOrderResult.Placed -> { requestId = newId(); _state.update { it.copy(placing = false, placedOrderId = r.order.id) } }
            is PlaceOrderResult.Rejected -> _state.update { it.copy(placing = false, error = r.reason) }
        }
    } }
}
// Option (b) instead of placedOrderId: private val _effects = Channel<String>(Channel.BUFFERED)
//   val effects: Flow<String> = _effects.receiveAsFlow()   // collect lifecycle-aware on Dispatchers.Main.immediate
@Composable fun CheckoutScreen(state: CheckoutUiState, onIntent: (CheckoutIntent) -> Unit) { /* render only */ }
```
One-shot events (navigate, snackbar) have two accepted designs: (a) Google's Android architecture guidance: the event
is a `UiState` field that the view consumes and then acknowledges with an intent that clears it; (b) a private
`Channel` exposed as `receiveAsFlow()` and collected on `Dispatchers.Main.immediate` with lifecycle awareness, which can
still lose an event if no collector is active. Accept either; flag only a mix of both in one codebase, or a public
`Channel`/`MutableStateFlow` the view can write to.

TS / React (React 18+ `useSyncExternalStore`; the view-model is a plain class, testable without React):
```ts
type CheckoutState = Readonly<{ placing: boolean; error?: string }>;
export class CheckoutViewModel {
  private state: CheckoutState = { placing: false }; private listeners = new Set<() => void>();
  private requestId: string;                                     // one per submission, reused on retry
  constructor(private readonly placeOrder: PlaceOrder, private readonly newId: () => string) { this.requestId = newId(); }
  subscribe = (l: () => void) => { this.listeners.add(l); return () => { this.listeners.delete(l); }; };
  getSnapshot = () => this.state;
  private set(s: CheckoutState) { this.state = s; this.listeners.forEach((l) => l()); }
  async submit(customerId: string, lines: readonly OrderLine[]) {
    this.set({ placing: true });
    const r = await this.placeOrder.execute(this.requestId, customerId, lines);
    if (r.kind === "placed") this.requestId = this.newId();
    this.set(r.kind === "placed" ? { placing: false } : { placing: false, error: r.reason });
  }
}
// const s = useSyncExternalStore(vm.subscribe, vm.getSnapshot);  <CheckoutView state={s} onSubmit={...} />
```
Review signals: a composable/component reading a global, repository or SDK directly (bypasses the view-model); several
independent state flows per screen (renders combinations that never co-existed); a mutable state holder or channel
exposed publicly; one-shot events handled two different ways in one codebase; a new idempotency key minted on every
retry of the same submission.

## 8. Migration recipe: introduce a boundary into existing code

Use **branch by abstraction**: the in-process counterpart of the strangler fig pattern, which intercepts calls at the
system edge (proxy/routing) instead. One step per PR; main stays green and releasable.

1. **Baseline.** Run the dependency check you intend to enforce ([fitness-functions.md](fitness-functions.md)) and
   record today's violations in a baseline/allow-list file. CI fails only on *new* violations.
2. **Introduce the port,** owned by the consumer, with the operations callers use today. Implement it with an adapter
   that delegates to the existing code. Change nothing else.
3. **Route callers through the port,** one call site or feature per PR, wiring the adapter at the composition root
   (create one now if missing; callers that fetched a global move first).
4. **Move the implementation behind the port** into the adapter/data module and make it `internal`/package-private,
   so the compiler stops new direct uses. Do not start this until every caller goes through the port.
5. **Invert upward edges:** lower-layer calls into UI or navigation become returned results, events or a callback port.
6. **Tighten:** delete fixed entries from the baseline; when it is empty, delete it. Promote linter rules to module
   boundaries where the build allows. Record the ADR and update the allowed-dependency table.

A pass-through repository that never gains policy of its own gets merged into its neighbour, not kept for symmetry.

## 9. Hand-off checklist

- [ ] Allowed-dependency table (§2) with concrete package names is in the ADR or architecture description.
- [ ] Every rule has a §3 mechanism (compiler, linter or test) that runs in CI; migrations have a baseline and burn-down.
- [ ] One composition root per executable; no service locator or global mutable singleton below it.
- [ ] Ports are consumer-owned and each has a fake; the domain compiles with no framework, driver, UI or SDK dependency.
- [ ] Each data slice has exactly one owning repository or store.
- [ ] Boundary interfaces state error semantics, idempotency, versioning policy and QA properties (§5).
- [ ] Cross-cutting decisions (§6) are made and recorded ([../templates/adr.md](../templates/adr.md)).
- [ ] A test runs the use case end to end with fakes: no network, real clock or randomness.
- [ ] The ADR links the QA scenarios that drove the layout ([design-workflow.md](design-workflow.md)) and any review
      findings it resolves ([review-playbook.md](review-playbook.md)).
