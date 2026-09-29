# Architecture review playbook: smell catalogue, PR checklist, finding rules, severity

Use this file whenever you review a diff, a pull/merge request, a repository or a design question through an
architecture lens. It tells you what to look for (smell catalogue), how to phrase what you find (finding rules),
and how to rank it (severity rubric). Recovery techniques live in
[architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md); the ATAM-style procedure lives in
[evaluation-methods.md](evaluation-methods.md); the executable guards referenced by each smell live in
[fitness-functions.md](fitness-functions.md); the report skeleton is
[../templates/architecture-review-report.md](../templates/architecture-review-report.md).

Theory adapted in part from the Wikipendium TDT4240 compendium (CC BY-SA 3.0); see [../CREDITS.md](../CREDITS.md).

**Two classes of statement, never mixed.** Every sentence in a review is either **AS-IS** (a claim about the code at a
named commit, backed by a file:line, a command output or a metric) or **TO-BE** (a proposal, marked as such, with its
cost). A finding states AS-IS evidence first, then the TO-BE fix. Do not describe intended architecture as if it were
the current one, and do not present a proposal as a fact about the code.

---

## 1. Review scopes and depth

Pick the row first; it fixes how much to load and what to produce. Escalate a row only when evidence demands it
(e.g. a small PR that adds a new cross-module edge becomes row 2).

| Scope | Load | Steps | Output |
|---|---|---|---|
| **Small PR** (< ~200 changed lines, one module) | This file (section 4 checklist) | 1. Read the diff and the touched files' imports. 2. Run the checklist (use its commands for new edges and migrations). 3. Check only smells the diff could introduce (D1, D4, D5, M3, R1, C2, C3, C5). | 0-3 inline findings; "no architectural concerns" is a valid, short answer. |
| **PR adding a module, dependency or integration** | This file + [module-layout-and-interfaces.md](module-layout-and-interfaces.md) + [fitness-functions.md](fitness-functions.md) | 1. Draw the before/after dependency edges. 2. Check direction and stability (D1-D3, D7). 3. Check the new interface (I1-I4) and runtime behaviour (R1, R3, R5). 4. Apply the ADR worthiness test. | Findings + proposed fitness function + "needs ADR: yes/no" with a draft title. |
| **Repository review** | This file + [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md) + [evaluation-methods.md](evaluation-methods.md) | 1. Recover the as-is module view and dependency graph. 2. Recover stated quality goals (README, ADRs, issues) or elicit them. 3. Sweep the full catalogue group by group. 4. Group findings into risk themes. | Full report from [../templates/architecture-review-report.md](../templates/architecture-review-report.md). |
| **"Is this ready for scale / production?"** | This file + [quality-attributes-runtime.md](quality-attributes-runtime.md) + [evaluation-methods.md](evaluation-methods.md) | 1. Write 3-6 quality-attribute scenarios with response measures (load, failure, deploy, security). 2. Walk each scenario through the code path. 3. Focus on R, C and I5. 4. Mark each scenario as met / at risk / unknown. | Lightweight ATAM output: scenarios, risks, non-risks, sensitivity and tradeoff points, prioritized fixes. |
| **Design question** ("should we split X?", "where does Y go?") | [design-workflow.md](design-workflow.md) + relevant pattern file | 1. Restate as a driver (scenario). 2. Find the as-is evidence for the smell the question implies. 3. Give 2-3 options with tradeoffs. | Recommendation + ADR draft ([../templates/adr.md](../templates/adr.md)). |

---

## 2. Smell catalogue

Every card lists the same fields in the same order: *What*, *Signals*, *QA / tactic*, *Why* (the scenario that breaks),
*Fix*, *Default severity*, *Fitness function*, *False positives*. **Default severity** is a starting point: raise it one
level when the smell hurts a quality attribute rated (H, H) in the utility tree, lower it one level when the attribute
is not a stated driver. Tactic names follow SAiP 4th ed. (Bass, Clements, Kazman, 2021) where one exists, with the 3rd-ed.
name in parentheses when it changed (as in [quality-attributes-change.md](quality-attributes-change.md)). Patterns in
`rg` are starting points; confirm each hit by reading the file.

### D. Dependency structure

**D1 Upward / inverted dependency**
- *What:* an inner layer (domain, use cases) imports an outer one (UI, persistence, transport, framework).
- *Signals:* in domain packages, `rg -n '^import (android\.|androidx\.compose|io\.ktor|org\.springframework)'`,
  `rg -n '^(from|import) (sqlalchemy|django|flask|fastapi|requests)' domain/`, `rg -n '"(net/http|database/sql)"' internal/domain`,
  `rg -n "from '(express|@prisma/client|axios|react)'" src/domain`.
- *QA / tactic:* modifiability, testability, portability; violates *restrict dependencies*, *encapsulate*.
- *Why:* "Swap Postgres for DynamoDB" or "add a CLI front end" now touches domain code; domain tests need the framework.
- *Fix:* define a port in the domain, implement it in an adapter, wire it in the composition root.
  ```kotlin
  // before (domain/Order.kt)
  import androidx.room.Entity
  @Entity data class Order(@PrimaryKey val id: String, val total: Long)
  // after
  data class Order(val id: String, val total: Long)                               // domain
  interface OrderRepository { fun save(o: Order) }                                // domain
  class RoomOrderRepository(private val dao: OrderDao) : OrderRepository {        // data
      override fun save(o: Order) = dao.insert(o.toEntity())
  }
  ```
- *Default severity:* Major. *Fitness function:* layer rule (ArchUnit/Konsist/import-linter/dependency-cruiser).
- *False positives:* pure value libraries (kotlinx.datetime, `decimal`, `time`) are acceptable in domain if the team decided so.

**D2 Dependency cycles**
- *What:* packages or modules that depend on each other directly or transitively.
- *Signals:* `npx madge --circular --extensions ts src/`, dependency-cruiser `no-circular` rule, `pydeps pkg --show-cycles`,
  ArchUnit `slices().matching("com.acme.(*)..").should().beFreeOfCycles()` (the direct JVM check). `jdeps -verbose:package
  --dot-output out/ app.jar` only prints edges; jdeps has no cycle report, so look for A->B / B->A pairs in the DOT
  output or feed it to a graph tool. Go refuses package import cycles at compile time, so in Go look for cycles at the
  service or repository level instead.
- *QA / tactic:* modifiability, buildability; violates *restrict dependencies* (reduce-coupling group).
- *Why:* nothing in the cycle can be changed, tested, released or extracted alone.
- *Fix:* find the weakest edge; move the shared type down, or invert the edge with an interface owned by the lower module.
- *Default severity:* Major (Minor inside one small package). *Fitness function:* cycle-free slices test in CI.
- *False positives:* intra-class or intra-file mutual references; Kotlin sealed hierarchies in one file.

**D3 Unstable dependency (Stable Dependencies Principle)**
- *What:* a module many others depend on (high Ca) depends on a volatile one (high instability I = Ce/(Ca+Ce), high churn).
- *Signals:* instability per module from [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md);
  churn from `git log --since=12.months --name-only --format=` counted per directory.
- *QA / tactic:* modifiability; violates *restrict dependencies*, *abstract common services*.
- *Why:* every change in the volatile module ripples to all dependants of the stable one.
- *Fix:* depend on an abstraction owned by the stable side; or move the volatile part out of the stable module.
- *Default severity:* Minor (Major if the churn is measured and ripple changes show up in history).
- *Fitness function:* instability threshold check on named modules. *False positives:* stable-by-design libraries with low churn.

**D4 Layer skipping / bypass**
- *What:* a view, controller or composable reaches the repository, DAO, DB client or HTTP client directly, skipping the
  application/service layer that owns the rule.
- *Signals:* `rg -n 'Repository|Dao|db\.|prisma\.' ui/ screens/ components/ controllers/`; in Compose, calls into
  data sources inside `@Composable` bodies; in Go, handlers using `*sql.DB`.
- *QA / tactic:* modifiability, testability, security (rules enforced in one path only); *restrict dependencies*.
- *Why:* the next rule added to the service is silently skipped by the bypass path.
- *Fix:* route through the use case / view model; if the service is a pass-through, it may be removable (decide via ADR).
- *Default severity:* Major if a rule or authz check is skipped, else Minor. *Fitness function:* layered-architecture test.
- *False positives:* relaxed layering documented in an ADR (e.g. read-side queries straight to a projection).

**D5 Hidden dependencies (global singletons, service locator, static mutable state)**
- *What:* components reach collaborators through globals instead of receiving them.
- *Signals:* Kotlin top-level mutable state `rg -n '^(private |internal |public )?(lateinit )?var \w+' --type kotlin`;
  mutable state inside objects `rg -n -A20 '^(private |internal )?object \w+' --type kotlin | rg '\bvar\b'`;
  `ServiceLocator\.get|getInstance\(\)|Injector\.`; module-level mutable dicts in Python; package-level `var` in Go;
  exported `let` singletons in TS. Confirm every hit by reading; a zero-hit result is not proof of absence.
- *QA / tactic:* testability (*localize state storage*, *abstract data sources*), modifiability; concurrency safety.
- *Why:* tests cannot substitute collaborators; initialization order and nullable globals cause startup crashes;
  unsynchronized writes from background threads are data races.
- *Fix:* constructor injection from one composition root; keep the singleton *lifetime* but not the global *access*.
  ```kotlin
  // before: AppState.instance!!.cart.items   // after: class CheckoutViewModel(private val cart: CartRepository)
  ```
- *Default severity:* Major when mutable and shared across threads, else Minor. *Fitness function:* forbid new top-level `var`
  in named packages (Konsist / custom lint). *False positives:* immutable constants, loggers, DI container itself.

**D6 Framework bleed (ORM entities / annotations as the domain model)**
- *What:* `@Entity`, `@Table`, Pydantic/SQLAlchemy models, Prisma types or `json:"..."` DTO structs used as domain objects.
- *Signals:* `rg -n '@(Entity|Table|Column|Serializable|JsonProperty)' domain/`, `class \w+\(Base\)` in domain,
  domain functions typed with `Prisma.*`.
- *QA / tactic:* modifiability, testability; violates *encapsulate*.
- *Why:* a column rename or serializer change becomes a domain change; invariants live in ORM hooks.
- *Fix:* separate persistence model and mapper at the adapter boundary; keep this only where the model is CRUD-trivial.
- *Default severity:* Minor (Major in a rich domain). *Fitness function:* forbid persistence annotations in domain packages.
- *False positives:* deliberate "transaction script" or CRUD apps where the ORM model *is* the model (say so, move on).

**D7 Transitive leak through public API**
- *What:* a module re-exports dependencies it should hide, so consumers compile against its internals.
- *Signals:* Gradle `api(...)` for implementation libraries (`api` exists with `java-library`, `com.android.library` and
  Kotlin Multiplatform, not with plain `java`/`application`; check with `./gradlew :mod:dependencies --configuration
  compileClasspath`); TS barrels `export * from './internal'`; Python packages without `__all__` whose `__init__` imports everything.
- *QA / tactic:* modifiability, build time; violates *encapsulate*.
- *Why:* a library upgrade inside the module breaks or recompiles every consumer that accidentally used it.
- *Fix:* `implementation(...)` by default, `api(...)` only for types in public signatures; explicit named exports;
  package.json `exports` map; Go `internal/`.
- *Default severity:* Minor. *Fitness function:* dependency-analysis plugin or a "no deep imports" lint rule.
- *False positives:* types that genuinely appear in the module's public signatures.

### M. Modularity

**M1 God class / module / enum; `utils`/`common`/`shared` dumping grounds**
- *What:* one element with many unrelated responsibilities, or an enum whose `when`/`switch` branches carry behaviour
  for every feature (every new screen or type edits it).
- *Signals:* files > ~1000 lines, high fan-in *and* fan-out; for a suspected god enum `<Enum>` (e.g. `ScreenId`):
  fan-in `rg -l '\b<Enum>\.' | wc -l`, branch sites `rg -n 'when \((\w+\.)?<enumProperty>' --type kotlin`, edit
  frequency `git log --since=12.months --format= --name-only -- path/<Enum>.kt | wc -l`; directories named
  `utils|common|shared|helpers|misc` with the highest churn.
- *QA / tactic:* modifiability; violates *increase cohesion* (tactics: *split module*, *redistribute responsibilities*;
  3rd ed.: increase semantic coherence).
- *Why:* "add a screen" edits the central enum plus N `when` expressions; merge conflicts concentrate there.
- *Fix:* move behaviour to the variants (sealed class / polymorphism / registry), split utils by the domain concept they serve.
- *Default severity:* Minor, Major if churn data shows it is a merge hotspot. *Fitness function:* max file size or
  fan-out budget on named files. *False positives:* generated code, data-only enums, flat DTO files.

**M2 Feature scattered (shotgun surgery, change coupling)**
- *What:* one conceptual change requires edits in many modules.
- *Signals:* co-change analysis from git (files that change together > ~50% of the time across module boundaries);
  a feature name appearing in many top-level directories.
- *QA / tactic:* modifiability; *redistribute responsibilities*.
- *Why:* every change to the feature needs coordinated edits and reviews in several modules, and one site gets missed.
- *Fix:* organise by feature/bounded context rather than technical layer at the top level; co-locate what co-changes.
- *Default severity:* Minor. *Fitness function:* co-change report in CI or a periodic job, alerting when cross-module
  coupling for a named feature rises.
- *False positives:* intentional cross-cutting changes (logging migration).

**M3 Logic in the wrong place (feature envy; business rules in UI or controllers)**
- *What:* pricing, validation, state transitions or permission rules computed in a view, controller or handler.
- *Signals:* arithmetic and branching on domain fields in `*Screen.kt`, `*.tsx`, `views.py`, HTTP handlers;
  a method using another object's fields more than its own.
- *QA / tactic:* modifiability, testability, consistency across clients; *redistribute responsibilities*.
- *Why:* the web client re-implements the discount rule and drifts; users get different totals per client.
- *Fix:* move the rule into a domain function with a unit test; the UI calls it.
- *Default severity:* Major when the same rule exists or will exist in a second client (web + mobile), else Minor.
- *Fitness function:* forbid imports of domain calculation helpers in `ui/` and arithmetic on `Money` outside domain
  (Konsist/ArchUnit rule). *False positives:* purely presentational logic (formatting, layout choices).

**M4 Mis-sized modules**
- *What:* one module per class (nano-modules, lots of wiring) or one module for everything (no enforceable boundary).
- *Signals:* module count vs. team size; modules with 1-2 files; a single Gradle/npm/Go module containing all layers.
- *QA / tactic:* modifiability, build time, team autonomy; *split module* (or merge).
- *Why:* nano-modules make every change a multi-module wiring edit; a single module lets any layer import any other.
- *Fix:* size modules by reason to change and ownership; merge nano-modules; split a monolith along the recovered layers.
- *Default severity:* Minor. *Fitness function:* module-graph rule (Gradle/workspace) matching the recorded layer ADR.
- *False positives:* small projects where one module plus package rules is enough.

**M5 Duplicated domain models without an anti-corruption layer**
- *What:* two parts of the system each define `User`/`Order` and convert ad hoc, or share one model across contexts
  where the meanings differ.
- *Signals:* several classes of the same name in different modules; mapping code scattered at call sites.
- *QA / tactic:* modifiability, integrability (*tailor interface*).
- *Why:* a rule fixed in one copy stays wrong in the other; users see two answers to the same question.
- *Fix:* decide per context: shared kernel (one model, jointly owned) or separate models with one translator (ACL).
- *Default severity:* Minor, Major if the copies already disagree on a rule.
- *Fitness function:* forbid imports of the other context's model package outside the translator.
- *False positives:* deliberate per-context models.

### I. Interfaces

**I1 Leaky abstraction**
- *What:* a port returns ORM entities, vendor SDK types, HTTP responses or SQL rows.
- *Signals:* interface signatures containing `ResultSet`, `Response`, `Prisma.*`, `firebase.*`, `*sql.Rows`.
- *QA / tactic:* modifiability, testability; violates *encapsulate*.
- *Why:* the adapter's vendor cannot be replaced without editing every caller of the port.
- *Fix:* return domain types or DTOs owned by the port's module. *Default severity:* Minor (Major on a published API).
- *Fitness function:* rule that `ports`/`domain` packages may not reference vendor or ORM packages.
- *False positives:* adapters' internal helper interfaces.

**I2 Chatty interfaces / N+1 across boundaries**
- *What:* per-item remote or DB calls in a loop across a process or module boundary.
- *Signals:* `await`/`client.get`/`repo.find` inside `for`/`map`; ORM lazy loading in templates; query logs with repeated shapes.
- *QA / tactic:* performance (*reduce computational overhead*, *increase efficiency of resource usage*).
- *Why:* latency grows linearly with collection size; fine in tests with 3 rows, fails at 3,000.
- *Fix:* batch endpoint or query, `IN (...)`, dataloader, eager fetch; coarse-grained remote interfaces.
- *Default severity:* Major on hot paths with measured latency, else Minor.
- *Fitness function:* query-count assertion in an integration test. *False positives:* bounded, tiny loops (< ~5 items) off the hot path.

**I3 Breaking contract changes without versioning**
- *What:* removed/renamed fields, changed semantics or new required fields in an API, event schema or DB contract used by others.
- *Signals:* diff touches OpenAPI/proto/GraphQL/Avro/event classes; `buf breaking` output for protobuf; no version bump.
- *QA / tactic:* integrability, deployability (*adhere to standards*, independent deploys).
- *Why:* a consumer deployed on its own schedule fails to parse the new payload in production.
- *Fix:* expand/contract: add new, dual-support, migrate consumers, remove old; version the contract.
- *Default severity:* Blocker if consumers are deployed independently.
- *Fitness function:* schema compatibility check in CI (e.g. `buf breaking`, a schema-registry compatibility mode, OpenAPI diff).
- *False positives:* internal types with a single consumer compiled together.

**I4 Vendor SDK everywhere**
- *What:* a third-party SDK (payments, analytics, cloud storage, LLM) imported in many modules.
- *Signals:* `rg -l "from ['\"]stripe['\"]|require\(['\"]stripe['\"]\)|import com\.google\.firebase|^\s*(import|from) (boto3|openai)"`,
  then count distinct top-level modules in the result; > 2 is the smell.
- *QA / tactic:* modifiability, testability; *use an intermediary*, *tailor interface*.
- *Why:* a vendor change or outage workaround touches every module that imported the SDK.
- *Fix:* one adapter behind a narrow port; ban the import elsewhere with a lint rule.
- *Default severity:* Minor, Major if a vendor change is a stated scenario.
- *Fitness function:* ESLint `no-restricted-imports`, import-linter `forbidden` contract, or ArchUnit rule limiting the SDK to one package.
- *False positives:* an SDK that is itself the platform (Android framework in an Android app layer).

**I5 Shared database between services**
- *What:* two deployable units read/write the same tables.
- *Signals:* same connection string or schema name in several services' config; migrations in one repo touching tables another owns.
- *QA / tactic:* deployability, modifiability, availability; violates data ownership.
- *Why:* a column change needs coordinated deploys; one service's heavy query degrades the other.
- *Fix:* one owner per table; others use its API or consume its events; read replicas/views as a temporary step.
- *Default severity:* Major.
- *Fitness function:* per-service DB credentials limited to owned schemas (enforced by the database, not by convention).
- *False positives:* one service with several processes (web + worker) sharing its own DB.

### R. Runtime

**R1 Remote calls without timeout, retry/backoff, circuit breaker or idempotency**
- *What:* outbound HTTP/RPC/DB calls on a request path with no explicit time bound, or retried without idempotency.
- *Signals:* bare `fetch(` (browsers: no timeout; Node's undici-based fetch: ~300 s header/body defaults, version-dependent,
  far above any user-facing budget; pass `AbortSignal.timeout(ms)`, available in Node 17.3+ and current browsers), `axios`
  without `timeout` (default 0 = none), Python `requests.get(` without `timeout=` (none by default),
  Go `http.Get(` / `http.DefaultClient` (no timeout), OkHttp without `callTimeout` (connect/read/write default to 10 s,
  but no overall call timeout), Ktor client without the `HttpTimeout` plugin (engine-dependent defaults).
- *QA / tactic:* availability (*retry*, *exception detection*, circuit breaker pattern), performance (*bound execution times*).
- *Why:* one slow dependency exhausts threads/connections and takes the caller down.
- *Fix:* explicit timeout per call, bounded retries with exponential backoff and jitter for idempotent operations only,
  idempotency keys for writes, breaker/bulkhead on critical dependencies.
  ```ts
  // before: const r = await fetch(url)
  // after:  const r = await fetch(url, { signal: AbortSignal.timeout(2000) })  // + retry w/ jitter only if idempotent
  ```
- *Default severity:* Major on user-facing paths.
- *Fitness function:* lint rule banning the raw client outside one wrapper that sets timeouts (ESLint
  `no-restricted-syntax`/`no-restricted-imports`, detekt/Konsist rule); test that the wrapper rejects after its budget.
- *False positives:* one-shot CLI tools, calls wrapped by a shared client that sets these.

**R2 Distributed monolith**
- *What:* separately deployed services that must be released together, call each other synchronously in chains, or share models/libraries with domain logic.
- *Signals:* lockstep version bumps; a synchronous chain of 3 or more remote hops on one request path (A calls B calls C
  calls D); shared "common-domain" library.
- *QA / tactic:* deployability, availability (availability multiplies along the chain).
- *Why:* you pay the network, ops and consistency costs of distribution without independent deployability.
- *Fix:* merge into a modular monolith, or decouple with async events and own data. *Default severity:* Major.
- *Fitness function:* contract tests per service; alert on lockstep releases.
- *False positives:* deliberate co-release during an early extraction phase recorded in an ADR.

**R3 Mixed or unstructured concurrency**
- *What:* several concurrency models side by side (threads + coroutines + in-house reactive + callbacks), fire-and-forget
  work with no owner, or blocking I/O on the UI thread.
- *Signals:* `GlobalScope.launch` (marked `@DelicateCoroutinesApi`), `runBlocking` in production code, `go func()` without
  `context`/`errgroup`/`WaitGroup`, `Thread {` / `thread {`, un-awaited promises (`@typescript-eslint/no-floating-promises`),
  `asyncio.create_task` without a kept reference, DB/network calls in `@Composable` or on `Dispatchers.Main`.
- *QA / tactic:* performance (*introduce concurrency*, *schedule resources*), availability, testability (*limit nondeterminism*).
- *Why:* orphaned work outlives its screen or request, leaks, races on shared state, and cannot be cancelled or tested.
- *Fix:* one structured model per platform with scopes tied to lifecycles; bridge legacy threads at one boundary; record the choice in an ADR.
- *Default severity:* Major. *Fitness function:* detekt `GlobalCoroutineUsage`, `no-floating-promises`, or a Konsist rule
  forbidding `GlobalScope`/`Thread` in named packages. *False positives:* a documented single app-lifetime scope.

**R4 Dual writes without an outbox**
- *What:* commit to the DB and publish to a broker/HTTP in the same handler, without atomicity.
- *Signals:* `repo.save(...)` followed by `producer.send(...)`/`publish(...)` in one function.
- *QA / tactic:* availability/integrity (*transactions*).
- *Why:* a crash or broker outage between the two writes leaves systems permanently inconsistent.
- *Fix:* transactional outbox table + relay, or change-data-capture; consumers idempotent.
- *Default severity:* Major where the event drives money, stock or state elsewhere.
- *Fitness function:* architecture test that broker clients are only called from the outbox relay.
- *False positives:* best-effort notifications where loss is acceptable and stated.

**R5 Unbounded queues, caches and resources**
- *What:* a queue, cache, thread or task pool whose size grows with input instead of being capped.
- *Signals:* `Channel(UNLIMITED)`, `LinkedBlockingQueue()` without capacity, `Executors.newCachedThreadPool()`,
  `Executors.newFixedThreadPool(n)` (threads capped, but its queue is an unbounded `LinkedBlockingQueue`; use
  `ThreadPoolExecutor` with an `ArrayBlockingQueue` and a rejection policy); Go `for ... { go handle(x) }` or a
  goroutine per request with no semaphore, worker pool or `errgroup.Group.SetLimit(n)` (goroutine count grows with
  input); `new Map()` caches with no eviction; `functools.cache` / `@lru_cache(maxsize=None)`; one thread per session.
- *QA / tactic:* performance (*bound queue sizes*, *limit event response*), availability.
- *Why:* under load spikes memory or thread count grows until the process is killed, which converts overload into an outage.
- *Fix:* bounds, eviction policy, backpressure, rejection with a clear error. *Default severity:* Major on server paths, Minor on clients.
- *Fitness function:* load test with a memory ceiling; lint rule on unbounded constructors. *False positives:* queues bounded upstream by design.

**R6 No admission control or rate limit on exposed endpoints**
- *What:* public, login, OTP, password-reset or expensive endpoints accept unlimited requests per client.
- *Signals:* no rate-limit middleware, gateway policy or limiter (Bucket4j, resilience4j `RateLimiter`,
  `golang.org/x/time/rate`, `express-rate-limit`) on those routes; no HTTP 429 anywhere in the codebase.
- *QA / tactic:* performance (*manage work requests*), security (*restrict login*, *detect service denial*), availability.
- *Why:* one client or a credential-stuffing run exhausts capacity for everyone, or brute-forces an account.
- *Fix:* limits per account and per IP at the edge; 429 with `Retry-After`; progressive delay on auth failures.
- *Default severity:* Major on auth endpoints, Minor elsewhere.
- *Fitness function:* integration test that the request after the limit in one window returns 429.
- *False positives:* limits enforced in the gateway, CDN or WAF (verify the config before dropping the finding).

**R7 Single point of failure**
- *What:* a component on a critical path with one instance, one zone, or no tested restore.
- *Signals:* `replicas: 1` for a user-facing service; a single DB instance with no replica or backup job; one broker
  node; no restore runbook; a leader-only scheduler with no failover.
- *QA / tactic:* availability (*redundant spare*, *rollback*).
- *Why:* one crash or zone outage takes the scenario down for the full repair time.
- *Fix:* redundancy sized to the scenario's RTO/RPO; automated failover; a restore that has been run, not just written.
- *Default severity:* Major when an availability scenario names that path, else Minor.
- *Fitness function:* scheduled restore test; game day that kills the instance and measures recovery time.
- *False positives:* internal tools and batch jobs whose scenario tolerates hours of downtime.

**R8 Mobile polling and unreleased wakeups**
- *What:* a mobile app that wakes the radio, CPU or GPS on its own timer instead of using push or the OS scheduler.
- *Signals:* `while (true) { ...; delay(` in repositories or services; `Timer.scheduledTimer` or `Handler.postDelayed`
  loops in background code; `AlarmManager.setRepeating`; `newWakeLock(` / `.acquire()` with no matching `release()` and
  no timeout; `requestLocationUpdates` with no `removeLocationUpdates`; one network call per sensor sample.
- *QA / tactic:* energy efficiency (*reduce usage*, *schedule resources*, *manage work requests*).
- *Why:* the app drains battery while idle, and the OS may restrict or kill it.
- *Fix:* push (FCM/APNs) for server events; WorkManager or `BGTaskScheduler` with constraints for deferrable work; batch
  requests and release locks. See [platforms-and-domains.md](platforms-and-domains.md).
- *Default severity:* Major on mobile, Minor on desktop.
- *Fitness function:* battery drain or wakeup count on a reference device in a scripted idle scenario; a lint rule
  banning `AlarmManager.setRepeating` and raw wake locks outside one module.
- *False positives:* foreground refresh while the screen is on and the user is waiting.

### C. Cross-cutting

**C1 Scattered authorization / auth only in the UI**
- *What:* authorization decided in the client or duplicated per handler instead of at one policy point.
- *Signals:* permission checks in components or route guards with none in handlers/services; checks copied per endpoint.
- *QA / tactic:* security (*authorize actors*, *limit access*).
- *Why:* a direct API call skips the UI check; a new endpoint copied without the check leaks data.
- *Fix:* enforce in the service/policy layer; UI checks are cosmetic. *Default severity:* Blocker when a server endpoint lacks the check.
- *Fitness function:* test enumerating all routes and asserting each has a policy annotation or middleware.
- *False positives:* gateway-enforced policy (verify the gateway config before dropping the finding).

**C2 Secrets and environment config in code**
- *What:* credentials, tokens or environment-specific endpoints committed to source.
- *Signals:* `rg -n -i '(api[_-]?key|secret|password|token)\s*[:=]\s*["\x27][^"\x27]{8,}'`, hard-coded hostnames and
  environment-specific IDs/URLs (tenant, region, network), gitleaks or trufflehog output.
- *QA / tactic:* security (*limit exposure*), deployability (*defer binding*).
- *Why:* anyone with repo access holds production credentials; changing an environment needs a rebuild.
- *Fix:* env/secret manager, config schema validated at startup; rotate anything that was committed.
- *Default severity:* Blocker for live secrets (also rotate), Minor for non-secret config.
- *Fitness function:* secret scanner in pre-commit and CI. *False positives:* test fixtures and documented public keys.

**C3 Inconsistent error handling across boundaries**
- *What:* errors swallowed, or each module inventing its own error type with no translation at boundaries.
- *Signals:* `catch (e) {}`, `except Exception: pass`, `_ = err`, `runCatching {}` whose failure is dropped; one error type per module.
- *QA / tactic:* availability (*exception detection*, *exception handling*), usability.
- *Why:* a failed payment or write is reported as success; operators see no signal until data is wrong.
- *Fix:* error model per boundary; translate at adapters; never swallow. *Default severity:* Major where money/data is involved, else Minor.
- *Fitness function:* Ruff `S110` (try-except-pass) and `BLE001` (blind-except); golangci-lint errcheck with
  `check-blank: true` to flag `_ = err`; ESLint `no-empty`; detekt `EmptyCatchBlock`.
- *False positives:* deliberate best-effort cleanup (close/delete) with a comment saying so.

**C4 Observability gaps**
- *What:* new paths cannot be traced, correlated or measured in production.
- *Signals:* no correlation/trace ID propagated across calls, logs without request context, no metrics tied to SLOs, `println` as logging.
- *QA / tactic:* availability (*monitor*), testability (*record/playback*).
- *Why:* an incident on the new path takes hours to localize because requests cannot be followed across services.
- *Fix:* structured logging, trace propagation (e.g. OpenTelemetry), SLO metrics on new paths.
- *Default severity:* Minor, Major for "ready for production" scope.
- *Fitness function:* integration test asserting the `traceparent` header is propagated on outbound calls.
- *False positives:* batch tools where a run log is sufficient observability.

**C5 Test-hostile seams**
- *What:* logic that reads the clock, randomness, network or files directly instead of through an injected seam.
- *Signals:* `System.currentTimeMillis()`, `Date.now()`, `datetime.now()`, `time.Now()` inside logic; `new HttpClient()`/`OkHttpClient()` inside services; file or network I/O in constructors.
- *QA / tactic:* testability (*abstract data sources*, *specialized interfaces*, *limit nondeterminism*).
- *Why:* expiry and scheduling rules cannot be tested deterministically, so they ship untested.
- *Fix:* inject `Clock`/`TimeSource`, clients and randomness. *Default severity:* Minor.
- *Fitness function:* forbid direct clock calls in domain packages. *False positives:* timestamps in adapters and logging.

**C6 Platform code leaking into shared code (KMP `commonMain`, shared web/mobile packages)**
- *What:* shared source sets or packages that depend on one platform's APIs.
- *Signals:* `rg -n '^import (java\.|android\.)' src/commonMain` (compiles only while every target is JVM-based);
  growing `expect`/`actual` surface (`rg -c '^\s*(expect|actual) '`); `window`/`document` in shared TS packages used by Node.
- *QA / tactic:* portability; *abstract common services*.
- *Why:* adding an iOS, JS or Wasm target fails to compile, or the "shared" code silently forks per platform.
- *Fix:* narrow `expect` interfaces or injected platform services. See [platforms-and-domains.md](platforms-and-domains.md).
- *Default severity:* Major if adding a target is a stated scenario.
- *Fitness function:* build every KMP target in CI (`./gradlew build` with all targets enabled) plus a Konsist rule
  forbidding `java.`/`android.` imports in `commonMain`. *False positives:* JVM-only projects that use KMP layout for convenience.

**C7 Unvalidated input at the boundary**
- *What:* external input (requests, messages, files, sensor readings) used before it is parsed into typed, checked values.
- *Signals:* string-concatenated SQL, shell or HTML (`"SELECT ... " + id`, f-strings passed to `execute(`, `os.system(`
  with request data, `innerHTML =`); request bodies read as raw maps with no schema (zod, pydantic, Bean Validation
  `@Valid`, `go-playground/validator`); sensor values used without range or staleness checks.
- *QA / tactic:* security (*validate input*), safety (*sanity checking*, *condition monitoring*).
- *Why:* injection, or invalid data that reaches the domain and corrupts state or drives an actuator.
- *Fix:* parse at the adapter into domain types that cannot hold invalid values; parameterized queries always.
- *Default severity:* Blocker for injection, Major otherwise.
- *Fitness function:* architecture test that controllers accept only validated DTO types; a SAST rule for query concatenation.
- *False positives:* input already validated by a gateway schema or generated server stub (check that it covers the field).

**C8 Exposed admin or debug surface**
- *What:* management, debug or internal endpoints reachable on the public listener.
- *Signals:* Spring Boot `management.endpoints.web.exposure.include=*` with no separate `management.server.port`;
  Go `net/http/pprof` imported into the public server (it registers `/debug/pprof/` on `http.DefaultServeMux`); `/admin`
  routes on the main router without authentication; Swagger UI or GraphQL introspection enabled in production config.
- *QA / tactic:* security (*limit exposure*, *limit access*).
- *Why:* attackers read config, heap dumps or metrics, or trigger admin actions outside the normal authorization path.
- *Fix:* separate port or network for management; deny by default; authenticate admin routes.
- *Default severity:* Blocker when unauthenticated and public, Major otherwise.
- *Fitness function:* smoke test against the public URL asserting admin paths return 404 or 403.
- *False positives:* health endpoints that expose only status.

**C9 Transport or token verification disabled**
- *What:* TLS certificate checks or token signature checks turned off.
- *Signals:* Python `requests`/`httpx` `verify=False`; Go `tls.Config{InsecureSkipVerify: true}`; a trust-all
  `X509TrustManager` or a `HostnameVerifier` that returns `true` (JVM/Android); Node `rejectUnauthorized: false` or
  `NODE_TLS_REJECT_UNAUTHORIZED=0`; PyJWT `options={"verify_signature": False}`; `jsonwebtoken` `jwt.decode` used where
  `jwt.verify` is needed; tokens accepted with `alg: none` or without audience and expiry checks.
- *QA / tactic:* security (*encrypt data*, *authenticate actors*, *verify message integrity*).
- *Why:* anyone on the network path can read or alter traffic, and anyone can forge a token.
- *Fix:* remove the override; trust a private CA explicitly if one is needed; verify signature, issuer, audience and expiry in middleware.
- *Default severity:* Blocker outside test code.
- *Fitness function:* CI search for the patterns above that fails outside test source sets.
- *False positives:* test code against a local self-signed server.

**C10 No single safety guard or safe state**
- *What:* actuator or irreversible-operation calls reachable from many modules, or a control state machine with no named safe state.
- *Signals:* the actuator driver or payout/transfer client imported by more than one module; control-loop exceptions
  logged and ignored; no `SAFE`/`OFF` state, or no transition into it on fault.
- *QA / tactic:* safety (*barrier*, *limit consequences*, *recovery*).
- *Why:* one caller that skips the invariant check can cause physical harm or irreversible loss.
- *Fix:* one guard module owns the driver and enforces invariants; fail-safe defaults; tested transitions into the safe state.
- *Default severity:* Blocker where harm is physical or irreversible.
- *Fitness function:* architecture test that only the guard module depends on the driver; fault-injection test asserting time-to-safe-state.
- *False positives:* simulators and test harnesses.

### E. Evolution

**E1 Decision without ADR, or ADR contradicted by code**
- *What:* a significant decision with no recorded rationale, or a recorded one the code no longer follows.
- *Signals:* a new framework, datastore, protocol or concurrency model with no ADR; an accepted ADR whose rule the code breaks.
- *QA / tactic:* all (lost rationale).
- *Why:* the next team reverses the decision without knowing its reason, or copies the violation as precedent.
- *Fix:* write or supersede the ADR ([../templates/adr.md](../templates/adr.md)). *Default severity:* Minor, Major when
  the contradiction is a security or data rule. *Fitness function:* the rule the ADR states, as an architecture test.
- *False positives:* reversible, local choices that fail the ADR worthiness test (section 4).

**E2 Parallel abstractions and half-finished migrations**
- *What:* two or more mechanisms for the same job, both still in use.
- *Signals:* two HTTP clients, two DI styles, two state/observer mechanisms (e.g. an in-house observer beside `StateFlow`), `legacy`/`v2` twins with both still called.
- *QA / tactic:* modifiability.
- *Why:* every new contributor must learn all variants, and bugs get fixed in only one of them.
- *Fix:* pick the target, write the ADR, migrate call sites. *Default severity:* Minor, Major with three or more mechanisms.
- *Fitness function:* ratcheting test: the count of legacy-client call sites must not increase.
- *False positives:* a migration with an owner, a tracked count and an end date.

**E3 Stale feature flags**
- *What:* flags that no longer vary but still branch the code.
- *Signals:* flags fully rolled out for months; branches never taken; flag checks in domain logic.
- *QA / tactic:* modifiability, testability (combinatorial states).
- *Why:* dead branches are still maintained and tested, and a flipped stale flag resurrects old behaviour.
- *Fix:* expiry date per flag, remove dead branch. *Default severity:* Nit/Minor.
- *Fitness function:* CI check that fails when a flag is past its expiry date. *False positives:* kill switches and ops toggles kept on purpose.

**E4 Build structure not reflecting intended layers**
- *What:* intended layers exist only as packages in one build module; nothing prevents an illegal import.
- *Signals:* one Gradle/npm/Go module holding all layers and no architecture test in CI.
- *QA / tactic:* modifiability; *restrict dependencies*.
- *Why:* the layer rule erodes one convenient import at a time.
- *Fix:* separate Gradle/npm workspace/Go modules or enforce with a rule. *Default severity:* Minor.
- *Fitness function:* the layer rule itself, or the split into build modules. *False positives:* see M4.

**E5 Accidental public API surface**
- *What:* internals that other packages can import and therefore depend on.
- *Signals:* everything `public` in a library (enable Kotlin `explicitApi()`), TS deep imports into `dist/internal`,
  Go packages that should be under `internal/`, Python modules without leading underscore imported by other packages.
- *QA / tactic:* modifiability (every exposed symbol becomes a contract); *encapsulate*.
- *Why:* refactoring an internal class breaks a consumer you did not know existed.
- *Fix:* narrow visibility. *Default severity:* Minor.
- *Fitness function:* published API check (e.g. binary-compatibility-validator, api-extractor).
- *False positives:* applications (not libraries) with no external consumers.

---

## 3. Smell -> quality attribute -> tactic index

| Smell | Primary QA hurt | Tactic / pattern to restore |
|---|---|---|
| D1, D4, D6 | Modifiability, testability | Restrict dependencies; encapsulate; ports and adapters |
| D2, D3, D7, E5 | Modifiability | Restrict dependencies; use an intermediary; abstract common services; encapsulate |
| D5, C5 | Testability | Localize state storage; abstract data sources; dependency injection |
| M1, M2, M4 | Modifiability | Split module; redistribute responsibilities (increase cohesion) |
| M3, M5 | Modifiability, consistency | Redistribute responsibilities; anti-corruption layer |
| I1, I4 | Modifiability, integrability | Use an intermediary; tailor interface |
| I2 | Performance | Reduce computational overhead; increase efficiency of resource usage |
| I3, I5, R2 | Deployability, integrability | Adhere to standards; versioned contracts; own your data |
| R1 | Availability, performance | Retry, exception detection, bound execution times; circuit breaker |
| R3, R5 | Performance, availability | Schedule resources; bound queue sizes; limit event response |
| R4 | Integrity/availability | Transactions (outbox) |
| R6 | Performance, security, availability | Manage work requests; restrict login; detect service denial |
| R7 | Availability | Redundant spare; rollback (tested restore) |
| R8 | Energy efficiency | Reduce usage; schedule resources; manage work requests (push) |
| C1, C2, C8 | Security | Authorize actors; limit access; limit exposure |
| C7 | Security, safety | Validate input; sanity checking |
| C9 | Security | Encrypt data; authenticate actors; verify message integrity |
| C10 | Safety | Barrier (interlock); limit consequences; recovery to a safe state |
| C3, C4 | Availability | Exception handling; monitor |
| C6 | Portability | Abstract common services; defer binding |
| E1-E4 | Modifiability | Architecture decisions recorded and enforced |

---

## 4. PR review checklist (architecture lens only)

Leave style, naming and correctness to other reviewers unless they cross a boundary.

- [ ] **New edges and direction.** List imports added across module/package boundaries:
  `git diff -U0 origin/main...HEAD | rg '^\+\s*(import |from \S+ import|.*require\()'`, then map each new import
  to its module. Does any point outward/upward (D1) or close a cycle (D2)?
- [ ] **New public API.** New exported symbols, `api(...)` deps, barrel exports, endpoints, event types (D7, E5, I3)?
- [ ] **New module, or its absence.** Does the change deserve its own module (new bounded context, new deployable), or does it create a nano-module (M4)?
- [ ] **Bypassed ports.** UI/handlers calling DB/HTTP/SDK directly (D4, I4)?
- [ ] **New cross-cutting concern.** Auth, caching, retries, logging added locally instead of in the shared mechanism (C1, C3, C4)?
- [ ] **Data ownership.** New tables, new writers to someone else's tables, new events (I5, R4)?
- [ ] **Migration safety.** Schema and contract changes follow expand/contract; old and new versions can run side by side
  during rollout. Find risky statements with `git diff origin/main...HEAD -- ':(glob)**/migrations/**' '*.sql' | rg -i '^\+.*(drop column|rename column|drop table|not null)'`;
  any hit needs the expand/contract steps.
- [ ] **Config and flags.** New env vars validated at startup; secrets not in code (C2); flags have an owner and removal date (E3).
- [ ] **Observability for new paths.** New remote calls or jobs emit logs with correlation ID and a metric (C4); timeouts set (R1).
- [ ] **Tests at boundaries.** Contract or adapter tests for new integrations; domain logic unit-tested without infrastructure (C5).
- [ ] **Manifest dependency additions.** Every new entry in `build.gradle.kts`/`libs.versions.toml`, `package.json`, `pyproject.toml`, `go.mod`: needed, maintained, license OK, in the right module, `implementation` not `api`.
- [ ] **ADR worthiness.** Needs an ADR if *any* holds: costly to reverse; crosses module or team boundaries; sets a precedent others will copy. If yes, request one and propose a title.

---

## 5. Finding quality rules

1. **Evidence is mandatory**, with its level on the scale shared with the report template and
   [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md): **A** tool-verified (command + output
   excerpt), **B** read in code (file:line), **C** inferred (pattern seen in N places, not all read; names, layout, docs),
   **D** reported (by a person or ticket; never the sole basis for Blocker or Major). No evidence, no finding; ask a
   question instead. Evidence level and confidence (section 6) are separate fields: level says where the claim comes
   from, confidence says how sure you are of the conclusion drawn from it.
2. **Tie to a scenario.** State which quality-attribute scenario the smell endangers ("adding an iOS target", "p95 < 300 ms
   at 50 rps", "payment provider outage"). If you cannot name one, the finding is taste; drop it.
3. **Concrete fix.** Give the diff, the config line or the rule to add, not "consider decoupling".
4. **Effort** S (< 1 day), M (< 1 week), L (larger; needs a plan and an ADR).
5. **Alternative and its tradeoff.** Name at least one other fix or "accept the risk" and what each costs; that is where
   sensitivity and tradeoff points surface.
6. **No taste-based findings.** Preferences on style, pattern names or frameworks without a QA impact are not findings.
7. **Fewer, stronger findings.** Ten well-evidenced findings beat forty weak ones. Merge duplicates; cite the count ("and 11 similar sites").
8. **Acknowledge non-risks briefly.** One line each for decisions that are sound; it calibrates trust in the rest.
9. **Group into risk themes** for repo reviews (e.g. "Composition by globals", "Unbounded remote calls"); ATAM calls these risk themes.
10. **Accepted risk goes into an ADR**, with the reason and a revisit trigger, not into silence.
11. **Phrase for the author.** Impact first, then evidence, then fix. Name the code, not the person. No generic advice
    ("follow SOLID"), no lecture on theory the fix does not need.

---

## 6. Severity rubric, confidence, sorting

| Severity | Meaning | Examples |
|---|---|---|
| **Blocker** | Must not merge/ship: breaks a stated quality goal, loses data, exposes data or breaks consumers. | Endpoint without server-side authz (C1); live secret committed (C2); breaking event schema with independent consumers (I3). |
| **Major** | Should be fixed before or soon after merge; materially raises the cost or risk of a stated scenario. | Domain imports Room/SQLAlchemy (D1); remote call without timeout on checkout path (R1); dual write for payments (R4). |
| **Minor** | Worth fixing when the area is next touched; local cost. | `utils` package growing (M1); `api(...)` where `implementation(...)` suffices (D7); missing `Clock` seam (C5). |
| **Nit** | Optional polish with an architectural flavour. | Stale flag for an internal tool (E3); ADR title inconsistent with the decision. |

**Confidence:** *Confirmed* (evidence read or tool output), *Likely* (strong pattern, not fully traced), *Question*
(you need an answer from the author to judge). Questions are phrased as questions and never carry Blocker severity.

**Sort** by severity (Blocker first), then confidence (Confirmed, Likely, Question), then fix cost (cheapest first), so
the top of the list is the most important, most certain and cheapest to act on.

---

## 7. Worked example

Fictional TypeScript order service (`orders-api`) and Kotlin Android app (`shop-app`), commit `abc1234`.
Stated drivers: *A1* checkout p95 < 400 ms and degrade gracefully when the payment provider is slow;
*M1* add a web client reusing the domain; *S1* only order owners may read orders. Compressed; real findings use the
full block from the report template.

```markdown
### Risk theme T1: remote dependencies can stall checkout (A1)

**ARCH-01 [Major | Confirmed | Effort S] Payment call has no timeout or retry policy (R1)**
AS-IS (evidence B): `src/payments/stripeClient.ts:42` `await fetch(PAY_URL, {method: "POST", body})`; no signal, no retry.
Scenario: provider latency spikes to 30 s; each checkout holds a socket for up to undici's ~300 s default; A1 breaches.
TO-BE: `signal: AbortSignal.timeout(1500)`, `Idempotency-Key` header, 2 jittered retries; on failure return 202, finish async.
Alternative: queue all payments; simpler failure handling, but the UI loses instant confirmation.
Guard: lint rule banning `fetch(` outside the `src/http/` wrapper.

**ARCH-02 [Major | Confirmed | Effort M] Order saved and event published as a dual write (R4)**
AS-IS (evidence B): `src/orders/createOrder.ts:77-81` `await repo.save(order); await bus.publish("OrderCreated", ...)`.
Scenario: broker down after commit -> stock never reserved; no reconciliation exists.
TO-BE: outbox table written in the same transaction; relay publishes; consumers dedupe by event ID.
Alternative: CDC (e.g. Debezium) on the orders table; no relay code, but adds an ops component.
Guard: architecture test that `bus.publish` is only called from `src/outbox/relay.ts`.

### Risk theme T2: the domain cannot be reused by a second client (M1)

**ARCH-03 [Major | Confirmed | Effort M] Domain imports persistence and UI frameworks (D1, D6)**
AS-IS (evidence A: `rg -n '^import androidx' domain/` -> 2 hits): `domain/.../cart/Cart.kt:5` `import androidx.room.Entity`;
`.../cart/PriceRules.kt:3` `import androidx.compose.runtime.mutableStateOf`.
Scenario: M1 web client needs `PriceRules`; it drags Room and Compose into a Kotlin/JS or Wasm build.
TO-BE: plain data classes in `domain`; `CartEntity` + mapper in `data`; `StateFlow` exposed by the view model.
Alternative: keep Room annotations and accept that the web client will not reuse domain; record in an ADR.
Guard: Konsist/ArchUnit rule "domain depends on nothing but kotlin.* and kotlinx.coroutines/datetime".

**ARCH-04 [Minor | Likely | Effort S] Views read repositories directly in 6 screens (D4)**
AS-IS (evidence C: 6 `rg` hits, 1 read): `ui/orders/OrdersScreen.kt:58` `ordersRepository.observe()` + 5 similar sites.
TO-BE: route through `OrdersViewModel`; add a layered-architecture test.
Alternative: document relaxed layering for read-only screens in an ADR; logic duplicates once M1 arrives.

**ARCH-05 [Minor | Question | Effort S] Is `AppGraph.instance` mutated after startup? (D5)**
AS-IS (evidence B): `app/AppGraph.kt:12` `lateinit var instance`, written in `ShopApp.onCreate` and `DebugMenu.kt:140`.
Scenario: release build reaches `DebugMenu` -> background readers race on a partially built graph.

### Non-risks
- Server-side ownership check in `src/orders/policy.ts:20` applied in every order route: S1 is met.
- Coroutines only, scopes tied to `viewModelScope`; no `GlobalScope` (evidence A: `rg -n GlobalScope` returned 0 hits).
```

Report the full set with [../templates/architecture-review-report.md](../templates/architecture-review-report.md);
turn accepted risks and chosen fixes with effort L into ADRs.
