# In-process design patterns as tactic carriers

Use this file when a design or review touches code *inside* one deployable: classes, modules, functions, wiring. A design pattern is only worth its indirection if it carries a tactic for a quality attribute (QA) that an architecturally significant requirement (ASR) actually asks for. Name the QA and the tactic, then the pattern. Never the reverse.

Cross-links (do not duplicate here):
- System-level patterns (Layers, Ports and Adapters, Pub-Sub, Broker, Microservices): [architectural-patterns.md](architectural-patterns.md)
- Tactic catalogues per QA: [quality-attributes-change.md](quality-attributes-change.md) (modifiability, testability, deployability) and [quality-attributes-runtime.md](quality-attributes-runtime.md) (performance, availability, security, usability)
- Package layout and interface placement: [module-layout-and-interfaces.md](module-layout-and-interfaces.md)
- Enforcing the resulting structure with tests: [fitness-functions.md](fitness-functions.md)
- Where these findings go in a review: [review-playbook.md](review-playbook.md), [../templates/architecture-review-report.md](../templates/architecture-review-report.md)

> Theory adapted in part from the Wikipendium TDT4240 compendium (CC BY-SA 3.0). See [../CREDITS.md](../CREDITS.md).

**Level rule.** A *design pattern* structures classes and objects inside a module. An *architectural pattern* fixes element types and their interactions for the whole system. Observer is a design pattern; MVC and Publish-Subscribe are architectural patterns that are often *implemented* with Observer. When reporting, state which level you mean.

---

## 1. GoF classification

Gamma, Helm, Johnson and Vlissides, *Design Patterns* (1994), sort the 23 patterns by purpose:

| Purpose | Concern | Patterns |
|---|---|---|
| Creational | Hiding which concrete class gets instantiated, and when | Abstract Factory, Builder, Factory Method, Prototype, Singleton |
| Structural | Composing classes and objects into larger structures | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy |
| Behavioural | Algorithms and the assignment of responsibility and communication | Chain of Responsibility, Command, Interpreter, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor |

- Template Method is **behavioural**, not structural (some secondary sources misfile it).
- Scope: *class* patterns bind through inheritance at compile time (Factory Method, class Adapter, Interpreter, Template Method); *object* patterns bind through composition and can change at run time. Prefer object scope when the ASR asks for run-time or deploy-time variation.
- GoF's four essential elements of a pattern: name, problem, solution, consequences. SAiP describes patterns as context, problem, solution. Use "consequences" as the review lens: every pattern costs something.

---

## 2. Pattern cards

Format: **Tactic / QA** realised. **Use when you see** (code trigger). **Smell** (misuse or overuse). Example.

### Creational

**Factory Method.** Tactic: defer binding, encapsulate (modifiability). Use when you see `if (type == ...) new X() else new Y()` repeated at call sites, or a framework base class that must create a product its subclasses choose. Smell: a "factory" with one product and no variation point; a static simple factory (not GoF) called Factory Method in docs.
```kotlin
abstract class Exporter { abstract fun writer(): Writer
    fun export(r: Report) = writer().use { it.write(r.render()) } }
class CsvExporter : Exporter() { override fun writer() = CsvWriter() }
```

**Abstract Factory.** Tactic: abstract common services, defer binding at startup (modifiability, portability, testability through fake families). Use when you see several related objects that must match (storage client + queue client + clock per environment; per-platform UI widgets). Smell: one family only; a factory that grows into a service locator everything reaches into; adding a product *kind* forces edits to every factory.
```ts
interface Infra { store(): BlobStore; queue(): Queue }
const awsInfra: Infra = { store: () => new S3Store(), queue: () => new SqsQueue() };
const testInfra: Infra = { store: () => new MemStore(), queue: () => new MemQueue() };
```

**Builder.** Tactic: none directly (not a SAiP tactic); it protects modifiability of construction code and enforces invariants at construction, so no half-built object escapes. Use when you see telescoping constructors, many optional parameters, cross-field validation, or objects that must be immutable once built. Smell: builders for types with two fields; in Kotlin/Python/TS, named or default arguments usually replace it; in Go, prefer functional options (`New(opts ...Option)`, as in the Strategy example) unless cross-field validation must run in one `Build()` step.
```java
var spec = RequestSpec.builder("GET", url)
    .header("Accept", "application/json")
    .timeout(Duration.ofSeconds(2))
    .build(); // validates all fields together and returns an immutable value
```

**Singleton.** Tactic: none; it is controlled access to one resource. Treat it as a **review finding by default**: it is a global variable that hides dependencies from signatures, leaks state between tests, and couples callers to a concrete class (testability and modifiability cost). Use when you see a genuinely process-wide resource (metrics registry, logging backend) *and* it is created in the composition root and passed in. Smell: `getInstance()` inside domain code; `object` declarations holding mutable state; lazy init racing across threads. Fix: keep the single instance but inject it (section 3.1). In a review, report it as D5 in [review-playbook.md](review-playbook.md).
```kotlin
// finding: class OrderService { fun place(o: Order) = Db.instance.save(o) }
class OrderService(private val repo: OrderRepository) { fun place(o: Order) = repo.save(o) }
```

### Structural

**Adapter.** Tactic: tailor interface (interoperability); encapsulate, use an intermediary (modifiability). Use when you see vendor SDK types (`stripe.*`, `boto3`, `firebase.*`) imported into domain or use-case code. The app owns the interface; the adapter lives at the edge. Smell: an "adapter" that exposes the vendor types anyway; one adapter per call site.
```python
class PaymentGateway(Protocol):
    def charge(self, cents: int, token: str) -> ChargeId: ...
class StripeGateway:  # adapter, the only module importing stripe
    def charge(self, cents, token):  # illustrative vendor call; check your stripe-python version
        return ChargeId(stripe.PaymentIntent.create(amount=cents, currency="usd", payment_method=token, confirm=True).id)
```

**Facade.** Tactic: encapsulate, use an intermediary, restrict dependencies (modifiability). Use when you see clients orchestrating five objects of one subsystem in the same order, or a module whose internals are imported from all over the codebase. Pair it with an enforced visibility rule (package-private internals, `internal` in Kotlin; in TS an `index.ts` barrel plus package.json `exports` for a package, or a dependency rule such as eslint-plugin-boundaries or dependency-cruiser banning deep imports, see [fitness-functions.md](fitness-functions.md)) or it is decoration. A barrel file alone enforces nothing: `billing/invoicing` can still be deep-imported. Smell: a facade that becomes a god object mirroring every internal method.
```ts
// billing/index.ts is the only import path other modules may use
export { invoiceCustomer } from "./invoicing"; // internals stay unexported
```

**Proxy.** Same interface as the subject, different access policy. Tactics: maintain multiple copies of data / caching (performance), lazy loading (performance: increase efficiency of resource usage), authorize actors / limit access (security), remote proxies (location transparency). Use when you see repeated identical remote calls, expensive objects built but rarely used, or authorization checks copy-pasted in front of a service. Smell: a caching proxy without an invalidation rule or TTL; remote I/O inside a map lock or `compute` block; no stampede protection on a hot key; a security proxy that can be bypassed because the real object is also injectable.
```kotlin
class CachingRates(private val real: Rates, private val ttl: java.time.Duration) : Rates {
    private val cache = ConcurrentHashMap<String, Pair<Instant, BigDecimal>>()
    override fun rate(ccy: String): BigDecimal =
        cache[ccy]?.takeIf { it.first.plus(ttl).isAfter(Instant.now()) }?.second
            ?: real.rate(ccy).also { cache[ccy] = Instant.now() to it } // remote call outside any map lock
}
```
In production prefer a cache library (e.g. Caffeine with `expireAfterWrite`) over a hand-rolled map: it adds size bounds and per-key load coalescing.

**Decorator.** Adds behaviour around an object with the same interface; stackable. Tactics: split module / increase cohesion (keep retries, metrics, logging out of business code); for availability, retry and timeout wrappers. Use when you see cross-cutting code (timing, retry, logging) interleaved with domain logic. Smell: decorator order that matters but is not documented or tested (retry inside vs outside a circuit breaker); deep stacks that make stack traces unreadable.
```go
type Store interface{ Get(ctx context.Context, k string) ([]byte, error) }
type timed struct{ next Store; h Histogram }
func (t timed) Get(ctx context.Context, k string) ([]byte, error) {
    start := time.Now(); defer func() { t.h.Observe(time.Since(start).Seconds()) }()
    return t.next.Get(ctx, k) }
```

**Composite.** Tactic: abstract common services (modifiability): clients treat leaves and groups uniformly. Use when you see recursive part-whole data (UI trees, permission groups, pricing rules, file trees) with `if (x is Group) for (c in x.children) ...` repeated. Smell: over-generality (hard to forbid illegal children); `add()` on leaves that throws; cycles or a child with two parents.
```python
class Rule(Protocol):
    def price(self, cart: Cart) -> Money: ...
class AllOf:
    def __init__(self, *rules: Rule): self.rules = rules
    def price(self, cart): return sum((r.price(cart) for r in self.rules), Money.zero())
```

**Bridge.** Separates an abstraction hierarchy from an implementation hierarchy so both vary independently. Tactics: abstract common services, restrict dependencies (modifiability, portability). Use when you see a class explosion of the form `PdfReportOnS3`, `PdfReportOnDisk`, `CsvReportOnS3` ... Smell: introducing it when only one dimension varies.
```kotlin
interface Sink { fun write(bytes: ByteArray) }                 // implementor
abstract class Report(protected val sink: Sink) { abstract fun publish() }
class PdfReport(sink: Sink) : Report(sink) { override fun publish() = sink.write(renderPdf()) }
```

### Behavioural

**Observer.** In-process publish-subscribe. Tactics: use an intermediary (the observer interface), defer binding through run-time registration (modifiability). Use when you see a model that calls UI, analytics and sync code directly after each change. Smell: see section 5 (re-entrancy, lapsed listeners, unspecified order). If publishers and subscribers must not know each other or cross processes, you need an event bus or broker, which is an architectural decision: see [architectural-patterns.md](architectural-patterns.md).
```ts
type Listener = (e: OrderPlaced) => void;
class Orders { private ls = new Set<Listener>();
  on(l: Listener) { this.ls.add(l); return () => this.ls.delete(l); } // return the unsubscribe
  place(o: Order) { /* ... */ for (const l of [...this.ls]) l({ id: o.id }); } }
```

**Strategy.** Tactic: defer binding (configuration, startup or run time); encapsulate (modifiability, testability). Use when you see a `when`/`switch` on a mode flag in several places selecting an algorithm (pricing, routing, compression, retry policy). In languages with first-class functions a function type is a Strategy. Smell: a strategy interface with one implementation and no configured alternative.
```go
type Backoff func(attempt int) time.Duration
func Exponential(base, max time.Duration) Backoff { // capped: an uncapped base<<n overflows int64
    return func(n int) time.Duration {
        d := base
        for i := 0; i < n && d < max; i++ { d *= 2 }
        return min(d, max) } } // min builtin: Go 1.21+
client := NewClient(WithBackoff(Exponential(100*time.Millisecond, 30*time.Second))) // add jitter in production
```

**Command.** Encapsulates a request as a value. Tactics: undo, cancel (usability, support user initiative); record/playback (testability); queue and schedule work (performance: bound queue sizes, schedule resources); an audit trail (security non-repudiation support). Use when you see requests that must be queued, retried, logged, replayed or undone. Smell: commands that capture live mutable objects instead of data, so they cannot be serialised or replayed.
```kotlin
sealed interface Cmd { fun apply(d: Doc): Doc; fun undo(d: Doc): Doc }
data class Insert(val at: Int, val text: String) : Cmd {
    override fun apply(d: Doc) = d.insert(at, text)
    override fun undo(d: Doc) = d.delete(at, text.length) }
```

**State.** Tactics: increase cohesion, via split module and redistribute responsibilities (3e: increase semantic coherence) (modifiability); explicit lifecycles also serve exception prevention (availability). Use when you see boolean flag soup (`isConnected && !isClosing && retrying`) or the same `switch (status)` in many methods. For protocols and sessions, prefer the explicit transition table of section 3.4 when behaviour per state is small. Smell: states reaching into each other's internals; transitions scattered with no single place to see the lifecycle. State vs Strategy: same structure, but the object switches its own State, while the client chooses the Strategy.
```ts
interface ConnState { open(c: Conn): void; send(c: Conn, m: Uint8Array): void; close(c: Conn): void }
const Closed: ConnState = {
  open: c => { c.socket = connect(); c.state = Open; },       // the state object performs the transition
  send: () => { throw new Error("connection closed"); },
  close: () => {} };
const Open: ConnState = {
  open: () => {},
  send: (c, m) => c.socket!.write(m),
  close: c => { c.socket!.end(); c.socket = undefined; c.state = Closed; } };
class Conn { state: ConnState = Closed; socket?: Socket;
  open() { this.state.open(this); } send(m: Uint8Array) { this.state.send(this, m); } close() { this.state.close(this); } }
```

**Template Method.** Tactic: abstract common services (modifiability via reuse; consistent step order). The base class fixes the algorithm skeleton and calls abstract *primitive* operations and optional *hook* operations. Use when you see several classes copying the same sequence with one or two differing steps. Smell: the skeleton not `final`, so subclasses override it; many hooks with unclear call order; deep inheritance. Prefer Strategy (composition) when the steps vary at run time.
```java
abstract class Importer {
    public final void run(Path p) { var rows = parse(p); validate(rows); store(rows); afterStore(rows); }
    protected abstract List<Row> parse(Path p);
    protected abstract void store(List<Row> rows);
    protected void validate(List<Row> rows) {}  protected void afterStore(List<Row> rows) {} // hooks
}
```

**Mediator.** Tactic: use an intermediary, restrict dependencies (modifiability). Colleagues talk to the mediator, not to each other, turning an N-to-N web into N-to-1. Use when you see UI components or workflow steps holding references to many siblings. Smell: the mediator becomes a god object; a generic "mediator library" that only dispatches requests to single handlers adds indirection without removing coupling.
```python
class CheckoutForm:  # mediator: fields never reference each other
    def changed(self, field: str) -> None:
        if field == "country":
            self.vat.set_rate(rates[self.country.value]); self.total.recompute()
```

**Memento.** Tactic: undo (usability); localize state storage, record/playback (testability). Captures an object's state without exposing its internals so it can be restored. Use when you see undo implemented by making fields public or deep-copying through reflection. Smell: unbounded history (memory); mementos that alias mutable collections.
```python
@dataclass(frozen=True)
class EditorSnapshot: text: str; cursor: int
class Editor:
    def save(self) -> EditorSnapshot: return EditorSnapshot(self._text, self._cursor)
    def restore(self, s: EditorSnapshot) -> None: self._text, self._cursor = s.text, s.cursor
```

**Chain of Responsibility.** Each handler processes or passes the request on. In current code it appears as interceptors and middleware (section 3.5). Tactics: split module, use an intermediary (modifiability); authenticate/authorize actors, validate input at one choke point (security). Use when you see authentication, rate limiting and logging repeated at the top of every handler. Smell: order dependencies that are implicit; a handler that silently swallows requests.
See the Go middleware example in section 3.5.

---

## 3. Non-GoF in-process patterns

### 3.1 Dependency injection and the composition root
Tactics: defer binding (startup), restrict dependencies (modifiability); abstract data sources and sandbox (testability). Constructors declare what a class needs; one **composition root** (`main`, the application module, the DI container config) builds the object graph. Only the root knows concrete classes.
```go
func main() {
    db := postgres.Open(cfg.DSN)
    svc := orders.NewService(postgres.NewOrderRepo(db), clock.System{})
    http.ListenAndServe(cfg.Addr, api.NewRouter(svc))
}
```
Checklist: no `new ConcreteRepo()` or container lookups outside the root; constructors take interfaces owned by the consumer side; a DI framework (Spring, Koin, Hilt, Dagger, NestJS) is optional, manual wiring is fine at small scale. A dependency rule test belongs in [fitness-functions.md](fitness-functions.md).

### 3.2 Repository and Unit of Work (Fowler, *Patterns of Enterprise Application Architecture*, 2002; the per-aggregate rule is from Evans, *Domain-Driven Design*, 2003)
- **Repository:** a collection-like interface for domain objects that hides the persistence mechanism; in DDD, one per aggregate root. Tactics: encapsulate, abstract data sources (modifiability, testability). Define it in the domain/application layer; implement it in infrastructure.
- **Unit of Work:** tracks the objects changed in a business transaction and commits them together. Tactic: transactions (availability, prevent faults): a business operation commits or rolls back as one. Keeps transaction boundaries in the application layer, not scattered across repositories. ORMs often provide it (JPA persistence context, SQLAlchemy `Session`, EF Core `DbContext`).
- Smells: repositories returning ORM query builders or lazy proxies (the leaky abstraction lets persistence types escape); a generic `Repository<T>` with `findAll()` on every table; one repository per table instead of per aggregate; commits inside repository methods.
```kotlin
interface OrderRepository { fun byId(id: OrderId): Order?; fun save(o: Order) }  // domain-owned
```

### 3.3 Result/Either at boundaries
Tactics: exception detection and exception handling (availability), made explicit in a typed interface contract (modifiability: encapsulate); typed errors also serve exception prevention. Return expected failures as values at module and use-case boundaries; reserve exceptions for bugs and truly exceptional infrastructure faults. Map infrastructure errors to domain errors in the adapter.
```ts
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
type PlaceError = "OUT_OF_STOCK" | "PAYMENT_DECLINED";
function place(o: Order): Promise<Result<OrderId, PlaceError>> { /* ... */ }
```
Kotlin: a `sealed interface` hierarchy (the stdlib `Result<T>` fixes the error type to `Throwable`). Go: `(T, error)` with sentinel or typed errors checked via `errors.Is`/`errors.As`. Smell: `catch (e: Exception) { return null }`; raw `SQLException`/`HttpError` crossing into domain code.

### 3.4 Explicit state machines for protocols and sessions
Tactics: redistribute responsibilities (3e: increase semantic coherence) under increase cohesion (modifiability); exception prevention (availability, prevent faults): illegal transitions are rejected before any side effect. For security the table acts like limit access: no action is reachable in the wrong state. Put every legal transition in one table; reject everything else. Test the table exhaustively.
```kotlin
enum class S { IDLE, AWAITING_PAYMENT, PAID, CANCELLED }
enum class E { CHECKOUT, PAYMENT_OK, CANCEL }
val transitions = mapOf(
    (S.IDLE to E.CHECKOUT) to S.AWAITING_PAYMENT,
    (S.AWAITING_PAYMENT to E.PAYMENT_OK) to S.PAID,
    (S.AWAITING_PAYMENT to E.CANCEL) to S.CANCELLED)
fun next(s: S, e: E): S = transitions[s to e] ?: error("illegal $e in $s")
```
Persist the state if the session outlives the process. Use GoF State (section 2) instead when each state carries substantial behaviour.

### 3.5 Interceptors and middleware pipelines
Chain of Responsibility in its modern form: HTTP middleware (Express/Koa, ASP.NET Core, Go `func(http.Handler) http.Handler`), gRPC interceptors, OkHttp interceptors, Ktor plugins. Tactics as in section 2; they give one choke point for authentication, authorization, input validation, rate limiting (security, performance: manage work requests), tracing and metrics (observability for testability and availability).
```go
func Auth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !validToken(r.Header.Get("Authorization")) { http.Error(w, "unauthorized", 401); return }
        next.ServeHTTP(w, r) }) }
```
Review: write down the order; check that routes cannot skip the security middleware (routes registered before it, or on a second router); keep business logic out of middleware.

---

## 4. Pattern to QA map

Name the specific tactic in documents and ADRs, not the tactic category ("use an intermediary", not "reduce coupling"). Tactic names follow SAiP 4e, with the 3e name in parentheses where it differs; the full mapping is in [quality-attributes-change.md](quality-attributes-change.md) and [quality-attributes-runtime.md](quality-attributes-runtime.md).

| Pattern | QA helped (tactic) | QA hurt / cost |
|---|---|---|
| Factory Method | Modifiability (defer binding, encapsulate) | Extra subclass per variant |
| Abstract Factory | Modifiability, portability, testability (abstract common services) | New product kind edits every factory |
| Builder | Modifiability of construction; integrity of objects | Boilerplate |
| Singleton | (none) controlled access | Testability, modifiability (hidden coupling), concurrency hazards |
| Adapter | Interoperability (tailor interface), modifiability (encapsulate) | One indirection; mapping code to maintain |
| Facade | Modifiability (encapsulate, restrict dependencies) | Risk of god object |
| Proxy | Performance (maintain multiple copies of data; lazy loading as increase efficiency of resource usage), security (authorize actors, limit access) | Staleness, consistency, bypass risk |
| Decorator | Modifiability (split module), availability (retry/timeout wrappers) | Order sensitivity, debuggability |
| Composite | Modifiability (abstract common services) | Type safety of the tree |
| Bridge | Modifiability, portability | More types up front |
| Observer | Modifiability (intermediary, run-time registration) | Performance with many listeners, traceability, leaks |
| Strategy | Modifiability, testability (defer binding) | Indirection |
| Command | Usability (undo, cancel), testability (record/playback), performance (queueing) | Class count, serialisation design |
| State / explicit FSM | Modifiability (increase cohesion: redistribute responsibilities; 3e: increase semantic coherence), availability (exception prevention) | More types or a table to maintain |
| Template Method | Modifiability via reuse | Inheritance rigidity |
| Mediator | Modifiability (intermediary, restrict dependencies) | Central complexity |
| Memento | Usability (undo), testability | Memory |
| Chain of Responsibility / middleware | Security (single choke point), modifiability (split module) | Latency per hop, hidden ordering |
| Dependency injection | Testability (abstract data sources), modifiability (defer binding) | Wiring code or framework magic; run-time wiring errors |
| Repository / Unit of Work | Modifiability, testability (abstract data sources), availability (transactions) | Leaky abstraction over the ORM; performance (N+1) if hidden |
| Result/Either | Availability (exception detection, exception handling), modifiability (typed contract) | Verbosity without language support |

---

## 5. Review signals

Report each as a finding with `file:line` evidence, the QA at stake, and a concrete fix (format in [review-playbook.md](review-playbook.md)).

**Pattern-itis (speculative generality).**
This section owns the detection and keep-or-inline rule for speculative generality; other files should link here.
- An interface with exactly one implementation, no test double, no plugin/platform seam, and no ASR asking for variation. Fix: inline it; reintroduce when a second implementation or a boundary appears. Keep interfaces that sit on an architectural boundary (ports) even with one implementation: the seam *is* the need.
- `AbstractFactoryFactory`, `*Manager`/`*Helper` with pattern names but no varying part; Strategy with one strategy; Builder for a two-field type.
- Detect: count production implementations per interface; flag 1:1 outside port packages. Prefer the IDE or language server ("Go to Implementations", LSP `textDocument/implementation`); grep only approximates. Per ecosystem:
  - Java/TS: `grep -rnE 'implements[^{]*\bFoo\b' src/` (TS structural implementers that never write `implements` need tsserver "Go to Implementations").
  - Kotlin: `grep -rnE '(class|object) [A-Za-z0-9_]+[^{=]*:[^{=]*\bFoo\b' src/`, a heuristic; read each hit. Never search for a bare `: Foo`: it matches every property and parameter typed `Foo`.
  - Go: interfaces are satisfied implicitly, so grep cannot find implementers. Use `gopls implementation <file>:<line>:<col>` or the IDE.
  - Python `Protocol`: implementers are structural; grep for explicit subclasses and inspect the types passed at injection sites (composition root, fixtures).
  - Exclude test sources from the production count; count test doubles separately. A fake used in tests is a legitimate second implementation, so the interface stays.

**Service locator and global singletons.** Report as **D5** in [review-playbook.md](review-playbook.md), which owns the signals and fix. Extra in-process signals to grep for: `ServiceLocator.get<T>()`, `container.resolve(...)`, `KoinComponent`/`by inject()` inside domain classes, `ApplicationContext.getBean(...)` outside configuration, module-level singletons imported in Python.

**Observer re-entrancy and leaks.**
- Re-entrancy: a listener that mutates the subject, or subscribes/unsubscribes during `notify`. Signals: iterating the live listener list (`for (l in listeners)` while `add` can run); events that trigger events that trigger the first event. Fix: iterate a snapshot; queue nested events; document whether notification is synchronous.
- Lapsed listener leak: `addListener`/`subscribe`/`on` without a matching remove in the owner's dispose/`onCleared`/`useEffect` cleanup/`close`. Fix: return an unsubscribe handle and tie it to a lifecycle (structured scope, `DisposableEffect`, `AbortController`, `context.Context`).
- Threading: callbacks arriving on a background thread and touching UI or unsynchronised state. Fix: marshal to the owner's thread or executor explicitly.
- Ordering: code relying on listener registration order. Fix: make the dependency explicit (one listener calls the next) or use a Mediator.

**Other quick signals** (use the playbook ID so one issue gets one finding).
- `getInstance()` or `object` with mutable state in domain code: Singleton, report as D5.
- Vendor SDK imports outside one adapter package: missing Adapter, report as I4; propose a port.
- Decorator or middleware stacks built in several places with different orders: centralise in the composition root and test the order.
- Repositories returning `Query`, `QuerySet`, `IQueryable` or ORM entities to controllers: leaky Repository, report as I1 (D6 if ORM entities are the domain model).
- Infrastructure exception types caught in UI code, or swallowed exceptions: missing Result mapping at the boundary, report as C3.
