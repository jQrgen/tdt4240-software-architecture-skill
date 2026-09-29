# Architectural pattern catalogue: code signatures, erosion signals, selection, anti-patterns

Use this file when you choose a pattern for a new system or feature, when you need to tell which pattern a repository actually follows, and when you check whether a claimed pattern still holds in the code. Every card says what the pattern looks like in a repo, how it erodes, and how to enforce it.

> Theory adapted in part from the Wikipendium TDT4240 compendium (CC BY-SA 3.0). See [../CREDITS.md](../CREDITS.md).

Related files (do not duplicate them here):
- Tactic definitions and QA scenarios: [quality-attributes-runtime.md](quality-attributes-runtime.md) (availability, performance, security, safety, energy efficiency, usability) and [quality-attributes-change.md](quality-attributes-change.md) (modifiability, testability, deployability, integrability)
- GoF and UI-level patterns as code (Observer, Strategy, Adapter, Facade, Repository class): [design-patterns-in-code.md](design-patterns-in-code.md)
- Package/module layouts and interface design: [module-layout-and-interfaces.md](module-layout-and-interfaces.md)
- Recovering the as-is pattern from code: [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md)
- Enforcement rules and tool configs: [fitness-functions.md](fitness-functions.md)
- Smell IDs (D1, R2, ...) used below: [review-playbook.md](review-playbook.md)
- Cloud, edge, mobile and game specifics: [platforms-and-domains.md](platforms-and-domains.md)

---

## 1. What a pattern is, and how to use one

**SAiP definition.** An architectural pattern is a package of design decisions that recurs in practice, has known properties that make it reusable, and describes a class of architectures. Patterns are found in working systems, not invented. SAiP describes each pattern as a triple:

| Part | Meaning | What to write in an ADR |
|---|---|---|
| Context | A recurring situation that gives rise to a problem | The system situation today |
| Problem | The problem stated generally, including the QAs that must be met | The driving ASRs / QA scenarios |
| Solution | Element types, interaction mechanisms (connectors), topological layout, semantic constraints; plus the QAs the configuration provides. (SAiP lists weaknesses separately in each pattern's description.) | The chosen structure plus the accepted weaknesses |

**Patterns bundle tactics.** A tactic is one design decision aimed at one QA response. A pattern bundles several tactics and usually trades QAs against each other. Applying a pattern has side effects; you *augment* it with further tactics to repair them (e.g. a broker adds latency and a single point of failure, so add redundancy to the broker). When you write ADRs or reviews, name the pattern **and** the tactics it brings; never list a pattern as if it were a tactic.

**Categories follow the view types** (see [documentation.md](documentation.md)):
- **Module** patterns structure code units: Layered. Most practitioner "architectures" (hexagonal, clean, modular monolith) are also module patterns.
- **Component-and-connector (C&C)** patterns structure runtime elements and their interactions: Broker, MVC, Pipe-and-Filter, Client-Server, Peer-to-Peer, SOA, Publish-Subscribe, Shared-Data.
- **Allocation** patterns map software to hardware, files, networks or teams: Map-Reduce, Multi-tier.

**Edition note.** SAiP 3rd ed. (2013), ch. 13 "Architectural Tactics and Patterns", holds the catalogue of the eleven patterns in §2. The 4th ed. (2021) has no catalogue chapter; each QA chapter ends with a Patterns section instead (for example redundancy patterns and circuit breaker under availability, microservices under deployability, layers/plug-in/pub-sub/client-server under modifiability, load balancer/throttling/map-reduce/service mesh under performance). When citing a 4th-ed. placement, say "the Patterns section of the <QA> chapter" and give no page or section numbers. Anything else in this file is marked **beyond the SAiP catalogue** and attributed to its actual source.

**Coplien (1998), "Software Design Patterns: Common Questions and Answers".** A pattern names a recurring problem in a context together with its solution, and captures the *forces* that the solution balances. Coplien stresses *generative* patterns: they tell you how to build something and the structure emerges from applying them, rather than just describing a structure after the fact. A pattern is not a reusable component or a code library; you re-implement it each time to fit the forces. Patterns become most useful in a *pattern language*, where each pattern's resulting context is the next one's starting context. Practical consequence: when you apply a pattern, write down the forces and the resulting context (what is still unsolved) in the ADR.

**Procedure when proposing a pattern:**
1. State the top 2-3 QA scenarios it must satisfy (from [design-workflow.md](design-workflow.md)).
2. Pick a default from §4, and name the runner-up and why it lost.
3. List the weaknesses from the card and the tactics you add to repair them.
4. Write the code signature you expect (folders, modules, build targets) and the fitness function that will keep it true.
5. Record it as an ADR ([../templates/adr.md](../templates/adr.md)).

---

## 2. Classic catalogue (SAiP)

### 2.1 Layered (module)
- **Intent:** Split software into layers with a one-way *allowed-to-use* relation so layers can be built, replaced and ported independently.
- **Tactics bundled:** encapsulate, restrict dependencies, use an intermediary, abstract common services.
- **QAs:** + modifiability, portability, reuse, testability (stub lower layers), parallel team work. − performance (indirection), up-front cost.
- **Constraints:** Every unit belongs to exactly one layer; at least two layers; allowed-to-use points downward only. **Strict** layering allows use of the layer directly below only. **Relaxed** layering allows any lower layer. Using a layer further down than the next one is *layer bridging*: allowed in the relaxed form, but document each bridge. Upward use is forbidden; notify upward with callbacks or events.
- **Code signature:** folders or build modules like `ui/ → application/ → domain/ → infrastructure/` (Java/Kotlin packages `..web..`, `..service..`, `..repository..`; Python `app/api`, `app/services`, `app/db`; Go `internal/handler`, `internal/service`, `internal/store`). In Gradle/Maven, separate modules whose `dependencies` only point down.
- **Variant check:** traditional layering lets domain call infrastructure; most modern codebases invert this edge (domain owns repository interfaces, infrastructure implements them), which is the hexagonal/clean variant in §3.1. Check which one the repo's ADRs claim before flagging domain→infrastructure imports.
- **Erosion signals:** imports from a lower layer to a higher one (smell D1); views or handlers importing the DB/ORM directly (D4); a `common`/`utils` package that every layer imports and that imports back (M1, D2); bridges that no document mentions.
- **Enforce:** ArchUnit `layeredArchitecture()`, import-linter `layers` contract, dependency-cruiser `forbidden` rules, go-arch-lint; see [fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem). Two minimal examples (since ArchUnit 1.0, `layeredArchitecture()` must be followed by a `considering...Dependencies()` call; late 0.x versions offered it optionally and earlier ones lacked it; verify against the installed version):

```java
@ArchTest
static final ArchRule layers = layeredArchitecture().consideringAllDependencies()
    .layer("Web").definedBy("..web..")
    .layer("Service").definedBy("..service..")
    .layer("Persistence").definedBy("..persistence..")
    .whereLayer("Web").mayNotBeAccessedByAnyLayer()
    .whereLayer("Service").mayOnlyBeAccessedByLayers("Web")
    .whereLayer("Persistence").mayOnlyBeAccessedByLayers("Service");
```

```ini
# .importlinter  (higher layers listed first; each may import only those below it)
[importlinter]
root_package = myapp

[importlinter:contract:layers]
name = Layered architecture
type = layers
layers =
    myapp.api
    myapp.services
    myapp.domain
```
- **Don't use when:** the system is tiny, or a hot path cannot afford the indirection, or nobody will enforce the rules (an unenforced layer diagram misleads reviewers).

### 2.2 Broker (C&C)
- **Intent:** Let clients call services without knowing where they live or how to reach them; binding can change at runtime.
- **Tactics bundled:** use an intermediary, discover service, encapsulate, (often) maintain multiple copies of computations behind the broker.
- **QAs:** + modifiability (move/replace servers), interoperability, availability (reroute around failed servers). − latency (extra hop), the broker is a bottleneck and single point of failure, security target, harder to test end to end.
- **Constraints:** Clients attach only to the broker (optionally via a client-side proxy); servers attach only to the broker. In SAiP's description the reply goes back through the broker; POSA (Buschmann et al., 1996) also describes a direct-communication variant where the broker only locates the server.
- **Code signature:** generated client stubs/proxies (gRPC, CORBA IDL, Java RMI), a registry/discovery client (Consul, Eureka, Kubernetes Service DNS), message brokers used for request/reply (RabbitMQ RPC, NATS request). Callers never hold concrete host names.
- **Erosion signals:** hard-coded service URLs next to broker lookups; services calling each other directly "for speed"; one broker instance with no redundancy.
- **Enforce:** a rule that only the generated proxy package may open network clients; config lint for literal hosts ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- **Don't use when:** there are one or two known servers (plain client-server is simpler), or for latency-critical per-request paths where the extra hop breaks the budget.

### 2.3 Model-View-Controller (C&C)
- **Intent:** Keep UI code separate from application state and logic so UIs can change without touching the model, and several views stay consistent.
- **Tactics bundled:** increase semantic coherence (3rd ed.; "redistribute responsibilities" is the nearest 4th-ed. modifiability tactic), encapsulate, defer binding (listeners registered at runtime).
- **QAs:** + UI modifiability, several synchronized views, model testable without UI. − complexity for simple UIs; update storms if the model notifies too often; some toolkits fit poorly.
- **Constraints:** At least one model, view and controller. The model does not depend on concrete views or controllers; views learn about changes by notification, usually implemented with GoF Observer.
- **Code signature:** `controllers/`, `views/`/`templates/`, `models/` (Rails, Django's MTV variant, Spring MVC `@Controller` + templates, ASP.NET MVC). On clients, `Model` classes exposing listeners or observable state.
- **Erosion signals:** business rules inside controllers or view templates (M3); models importing view or HTTP types (D1); "fat controller" files far larger than their models.
- **Enforce:** a rule that `models/`/domain packages may not import web or UI frameworks ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- **Don't use when:** the UI is trivial or throwaway. For modern reactive clients prefer MVVM or MVI/UDF (§3.5).

### 2.4 Pipe-and-Filter (C&C)
- **Intent:** Process a stream of data through independent transformation steps that can be recombined and run in parallel.
- **Tactics bundled:** encapsulate, introduce concurrency, split module, (with buffered pipes) bound queue sizes.
- **QAs:** + reuse and modifiability (recombine filters), throughput through concurrency. − poor fit for interactive request/response; per-filter overhead and format conversion; hard to share state; error handling across stages needs design.
- **Constraints:** Pipes connect output ports to input ports; connected filters agree on the data type; filters are independent and keep no shared state.
- **Code signature:** Unix pipelines; Java `Stream`/Kotlin `Flow` chains; Go goroutines connected by channels; Node streams `.pipe()`; Apache Beam / Flink / Kafka Streams topologies; ETL DAGs (Airflow tasks); middleware chains; image/audio processing stages.
- **Erosion signals:** filters that read or write a shared global or DB table to talk to each other; a stage that knows the next stage's concrete type; unbounded channels or queues between stages (R5).
- **Enforce:** each filter module depends only on the message-type module (import rule); each filter is unit-tested in isolation as input → output with no shared state; queues between stages have explicit bounds (test or config lint) ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- **Don't use when:** the core is interactive, or steps need rich shared mutable state.

### 2.5 Client-Server (C&C)
- **Intent:** Centralize shared resources and services behind servers that many clients call with request/reply.
- **Tactics bundled:** increase resources and maintain multiple copies of computations (server replicas), authenticate and authorize actors, limit access.
- **QAs:** + centralized control, consistency and security, reuse of common services, scalability by replicating servers. − the server is a bottleneck and single point of failure unless replicated; moving functionality between client and server later is expensive.
- **Constraints:** Clients connect to servers; servers may be clients of other servers; the number of tiers may be restricted.
- **Code signature:** a `client/` or app module with an API client (Retrofit, Ktor client, `fetch`/axios, `requests`/httpx, `net/http`), a `server/` module with routes/handlers, a shared contract (OpenAPI spec, `.proto`, shared DTO module).
- **Erosion signals:** authorization done only in the client (C1); clients that assume server internals (DB IDs, table names); no versioning on the shared contract (I3).
- **Enforce:** contract tests (Pact or schema diff in CI); server-side authz checks tested at the boundary ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- **Don't use when:** peers are equal and there is no trusted party (P2P), or the server's cost and SPOF are unacceptable.

### 2.6 Peer-to-Peer (C&C)
- **Intent:** Equal peers, each both client and server, share services over a common protocol with no central server.
- **Tactics bundled:** active redundancy, discover service, maintain multiple copies of data.
- **QAs:** + availability (no single point of failure), scalability (each peer adds capacity), low infrastructure cost. − security and trust, data consistency, availability of *a specific* datum when peers leave, backup/recovery; small networks may never reach their goals.
- **Constraints:** Rules may limit connections per peer and define special roles (supernodes, trackers).
- **Code signature:** DHT or gossip libraries (libp2p, Kademlia implementations), node software that both listens and dials, blockchain full nodes, WebRTC data channels between clients, local-network sync (mDNS discovery).
- **Erosion signals:** a "bootstrap" or "coordinator" node that everything silently depends on; trust decisions taken on unauthenticated peer data.
- **Enforce:** tests that run the network with the bootstrap node removed; signature verification at every peer boundary ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- **Don't use when:** you need an authoritative source of truth (payments, rankings, anti-cheat) and cannot afford a consensus protocol (proof-of-work/stake, BFT) with its latency and cost; use client-server with a trusted server instead. Also not when the peer count stays tiny.

### 2.7 Service-Oriented Architecture (C&C)
- **Intent:** Compose services from different providers, languages and platforms through published contracts.
- **Tactics bundled:** discover service, orchestrate, tailor interface, use an intermediary, adhere to standards.
- **QAs:** + interoperability, modifiability (swap a service behind its contract), reuse. − complexity, middleware overhead and latency, no control over external services' evolution, weak performance guarantees.
- **Constraints:** Consumers use services only through published contracts, possibly via intermediaries: an ESB (routing, transformation), a registry, an orchestration server.
- **Code signature:** WSDL/SOAP or OpenAPI contracts per service, an ESB or integration platform (MuleSoft, Apache Camel routes), BPMN workflow engines, many vendor SDKs (payments, identity, analytics) composed by a backend.
- **Erosion signals:** vendor SDK types spread across the domain (I4); business logic accumulating inside ESB routes or transformation scripts; consumers depending on undocumented fields.
- **Enforce:** only adapter packages may import vendor SDKs; contract tests per external service ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- **Don't use when:** one team owns a small system and no third-party integration is involved.

### 2.8 Publish-Subscribe (C&C)
- **Intent:** Producers announce events without knowing who or how many consumers exist.
- **Tactics bundled:** use an intermediary, defer binding (subscription at runtime).
- **QAs:** + modifiability and extensibility (add subscribers without touching publishers), loose coupling. − latency, scalability (SAiP rates it negatively: broker throughput and fan-out cost), predictability of delivery time, ordering and delivery guarantees, harder to test and to trace control flow.
- **Constraints:** Components connect to the event bus, not to each other. Unlike GoF Observer, where the subject holds references to observers, the bus decouples publisher and subscriber completely.
- **Code signature:** in process: Guava `EventBus`, Spring `ApplicationEventPublisher`, Node `EventEmitter`, Kotlin `SharedFlow`; across processes: Kafka topics, RabbitMQ exchanges, NATS subjects, Google Pub/Sub, SNS/SQS, Redis pub/sub.
- **Erosion signals:** a subscriber whose result the publisher waits for (hidden request/reply); event payloads that mirror a table schema; ordering assumptions nobody documented; no dead-letter handling; events with no owner or schema registry.
- **Enforce:** schema registry compatibility checks; a test that publishing with zero subscribers succeeds; trace propagation checks (C4) ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- **Don't use when:** there is one known receiver, or the interaction needs a synchronous reply or strict ordering and timing.

### 2.9 Shared-Data / Repository (C&C)
- **Intent:** Many independent accessors share and manipulate a large persistent data set that no single one owns.
- **Tactics bundled:** maintain multiple copies of data (replicas), transactions, limit access. The *blackboard* variant adds notification.
- **QAs:** + data consistency, accessor independence, centralized data management. − the store is a bottleneck and SPOF; every accessor couples to the schema, so schema changes ripple.
- **Constraints:** Accessors interact only through the store, not directly. Variants: *repository* (accessors initiate) and *blackboard* (the store notifies accessors).
- **Code signature:** several apps or jobs with the same DB connection string; shared `schema.sql`/migrations used by several deployables; a shared Firestore/Mongo collection read by client, admin tool and cloud functions.
- **Erosion signals:** in a service architecture, this pattern appearing by accident is smell I5 (shared DB between services); accessors coordinating via "status" columns; migrations that need several teams to sign off.
- **Enforce:** DB users/roles per accessor with least privilege; a check that only one deployable owns migrations for each schema ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- **Don't use when:** data is private to one component, or write contention would make the store the bottleneck.

### 2.10 Map-Reduce (allocation)
- **Intent:** Process a very large, partitionable data set in parallel on a cluster of commodity machines, tolerating node failure.
- **Tactics bundled:** introduce concurrency, increase resources, retry (re-run failed tasks), schedule resources.
- **QAs:** + throughput on big data, horizontal scalability, availability through task re-execution. − overhead not justified on small data; parallelism collapses if data cannot be split evenly (skew); multi-step jobs are complex to orchestrate.
- **Constraints:** Input exists as files or partitions; map functions are stateless and do not talk to each other; map and reduce communicate only via emitted `<key, value>` pairs that the framework shuffles and sorts.
- **Code signature:** Hadoop `Mapper`/`Reducer` classes, Spark `map`/`reduceByKey`/`groupBy` jobs, Beam pipelines, BigQuery/Presto batch SQL, cloud dataflow job specs.
- **Erosion signals:** mappers calling external services or a shared DB; one key receiving most records; jobs run on data small enough for one machine.
- **Enforce:** unit-test map and reduce functions as pure functions; monitor partition skew ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- **Don't use when:** the data is small, the processing is interactive, or the data cannot be partitioned.

### 2.11 Multi-tier (allocation)
- **Intent:** Group components into tiers (e.g. presentation, logic, data) deployed on separate execution environments for operational, security or scaling reasons.
- **Tactics bundled:** separate entities, limit exposure, increase resources per tier, maintain multiple copies of computations (load balancer per tier).
- **QAs:** + security (data tier behind a firewall), per-tier scaling and availability, modifiability. − cost and complexity, latency from network hops.
- **Constraints:** Each component belongs to exactly one tier. SAiP notes it can be read as C&C or allocation depending on how tiers are defined, and catalogues it under allocation.
- **Tier vs layer:** a layer is a code-time module grouping with allowed-to-use; a tier is a runtime grouping mapped to execution nodes. Several layers can live in one tier. Do not confuse them in reviews or docs.
- **Code signature:** separate deployables per tier (SPA bundle, API service, DB), Terraform/Helm/Compose files that place them in different networks or subnets, security groups that only let the API reach the DB.
- **Erosion signals:** the presentation tier holding DB credentials (C2); direct client-to-DB connections that bypass the logic tier; tiers that always scale together (a sign the split buys nothing).
- **Enforce:** network policies / security groups as code, checked in CI; secret scanning in client bundles ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- **Don't use when:** everything runs on one device or in one small process.

---

## 3. Practitioner patterns (mostly beyond the SAiP catalogue)

Format: intent · QAs · code signature · erosion · enforce · don't use when. Tactic names refer to the QA reference files; enforcement tools and configs are in [fitness-functions.md](fitness-functions.md).

### 3.1 Module structure
**Hexagonal / ports and adapters** (beyond the SAiP catalogue; Alistair Cockburn, 2005).
- Intent: the application core defines *ports* (interfaces); adapters implement them for HTTP, DB, queues, vendor SDKs. The core imports nothing from infrastructure.
- QAs: + testability, modifiability, integrability; − more interfaces and mapping code.
- Signature: `domain/` or `core/` with `port`/`spi` interfaces; `adapters/in/web`, `adapters/out/persistence`; in Go, interfaces declared in the consuming package; in KMP, interfaces in `commonMain` with `expect`/`actual` or DI-supplied implementations.
- Erosion: ports returning ORM entities or vendor types (I1); core importing `javax.persistence`/`jakarta.persistence`, `sqlalchemy`, `android.*` (D1, D6).
- Enforce: ArchUnit/Konsist rule "core may not depend on adapters or frameworks"; import-linter `forbidden` contract; dependency-cruiser rule from `src/core` to `src/adapters` ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- Don't use when: the app is a thin CRUD layer over one DB with no integrations.

**Clean / onion architecture** (beyond the SAiP catalogue; onion: Jeffrey Palermo, 2008; clean: Robert C. Martin, 2012 post and 2017 book).
- Intent: concentric rings (entities, use cases, interface adapters, frameworks); the dependency rule says source dependencies point inward only. A layered pattern with dependency inversion at the infrastructure boundary.
- QAs: as hexagonal; − ceremony (use-case classes, mappers) that can dwarf small features.
- Signature: `entities/`, `usecases/`/`interactors/`, `interfaceadapters/`/`presentation/`, `frameworks/`/`data/`; Android "domain/data/presentation" modules.
- Erosion: use cases that just forward to repositories (ceremony without rules); inner rings importing framework annotations (D6).
- Enforce: one build module per ring so the compiler rejects outward imports (Gradle `domain` module with no Android plugin; Go `internal/domain` with an import rule) ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- Don't use when: the team cannot name any business rule that lives in the inner ring.

**Modular monolith** (beyond the SAiP catalogue; no single originator).
- Intent: one deployable split into modules by business capability, each with a public API and private internals, often owning its own schema.
- QAs: + modifiability, simple operations, cheap refactoring across modules, a path to extract services later; − one deploy cadence and one failure domain.
- Signature: `modules/billing`, `modules/catalog` with `api`/`internal` subpackages; Gradle/Maven modules per capability; Java `module-info.java`; Spring Modulith; Go `internal/` per module; TS workspaces with `exports` fields.
- Erosion: modules reaching into each other's `internal` (E5); shared tables across modules; cycles between modules (D2).
- Enforce: Spring Modulith `ApplicationModules.of(App.class).verify()`; Gradle modules with `implementation` (not `api`) dependencies plus Kotlin `internal` / JPMS `exports` to hide internals; eslint-plugin-boundaries or dependency-cruiser across workspace packages; one schema owner per module ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- Don't use when: teams truly need independent deploys or different runtime scaling per capability.

**Microkernel / plug-in** (POSA 1996; SAiP 4th ed. lists plug-in under modifiability).
- Intent: a minimal core plus plug-ins bound at load or run time through an extension API.
- QAs: + extensibility, third-party contribution, modifiability; − API stability burden, plug-in isolation and security, harder testing of combinations.
- Signature: `ServiceLoader`/`META-INF/services`, OSGi bundles, Python entry points, VS Code/Eclipse/IntelliJ extension manifests, Webpack/Babel/ESLint plugin APIs, Go plugin interfaces registered in `init()`.
- Erosion: core importing concrete plug-ins; plug-ins using core internals; no versioned extension API (I3).
- Enforce: plug-ins compile only against a separate `api`/`spi` artifact; a test that boots the core with zero plug-ins ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- Don't use when: all "plug-ins" are written by the core team and ship together.

### 3.2 UI state
**MVVM, MVI and unidirectional data flow** (beyond the SAiP catalogue; MVVM: John Gossman, Microsoft, 2005; MVI popularized on Android from Cycle.js; UDF in Redux/Elm/Flux style).
- Intent: MVVM exposes observable view state from a ViewModel; MVI/UDF models the screen as `state = reduce(state, intent)` with a single immutable state and one-way flow of events up and state down.
- QAs: + testability (pure reducers), predictability, UI modifiability; − boilerplate, state-object churn and recomposition cost if state is too coarse.
- Signature: Android `ViewModel` exposing `StateFlow<UiState>`, sealed `Intent`/`Action` classes, Redux/Zustand stores, Elm-style `update`, SwiftUI `ObservableObject`, Compose `collectAsState`.
- Erosion: views mutating shared state directly; ViewModels calling HTTP clients (skip use cases, D4); several sources of truth for the same screen; `GlobalScope` work from UI (R3).
- Enforce: Konsist/ArchUnit "ViewModels do not depend on data-layer classes"; reducer unit tests with no Android or DOM dependencies ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- Don't use when: a static or form-only screen; a plain template suffices.

### 3.3 Distribution and integration
**Microservices** (beyond the 3rd-ed. catalogue; Lewis and Fowler, 2014; SAiP 4th ed. discusses it under deployability).
- Intent: independently deployable services, each owning its data, organized around business capabilities.
- QAs: + deployability, team autonomy, per-service scaling and fault isolation; − performance (network), consistency, operational cost, harder testing.
- Signature: one repo or folder per service with its own Dockerfile, pipeline and DB migrations; service-to-service clients; per-service Helm charts.
- Erosion: distributed monolith (§5, R2); shared DB (I5); a shared "common" library every service must upgrade in lockstep (D3, R2).
- Enforce: per-service DB credentials; contract tests in each pipeline; a check that each service deploys on its own (no multi-service release jobs) ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- Don't use when: one team, a new product, unclear domain boundaries. Extract from a modular monolith once seams are proven.

**Event-driven: three different things** (beyond the SAiP catalogue; Fowler's 2017 distinction).
- *Event notification:* thin event ("order 42 changed"); consumers call back for data. Low coupling, but the call-back creates runtime coupling.
- *Event-carried state transfer:* the event carries the data consumers need; consumers keep local copies. + availability and latency; − eventual consistency and duplicated data.
- *Event sourcing:* see below; it is a storage pattern, not an integration style.
- Review rule: ask which one the code actually uses; many systems mix them undeclared.
- Signature: event classes named in past tense (`OrderPlaced`), topic definitions, consumers with handler methods; payload size tells notification (IDs only) from state transfer (full snapshots).
- Erosion: "events" that are commands in disguise (`SendEmail` published to one consumer); consumers that call the producer synchronously on every event; no idempotency on consumers.
- Choose:
  - *Notification* when consumers only need to know something happened, can tolerate a call-back, and the producer's data is large or sensitive.
  - *Event-carried state transfer* when consumers must keep working while the producer is down, or call-back load would be too high, and eventual consistency plus duplicated data is acceptable.
  - *Event sourcing* only when audit or temporal queries are a requirement.
- Enforce: schema-registry compatibility checks on every event type; a consumer test that delivers the same message twice and out of order and asserts one correct outcome ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- Don't use when: one known consumer needs a synchronous answer; call it directly behind an interface.

**CQRS** (beyond the SAiP catalogue; Greg Young, building on Meyer's command-query separation).
- Intent: separate write model (commands, invariants) from read models (queries, projections), possibly in separate stores.
- QAs: + read performance and scalability, simpler write model; − eventual consistency between sides, more moving parts.
- Signature: `commands/` + `handlers/`, `queries/` + `projections/`, read-model tables or search indexes fed by events.
- Erosion: queries reading the write tables anyway; commands returning read models; projections nobody can rebuild from scratch.
- Enforce: `queries/` may not import command handlers or write repositories; a job that rebuilds each projection in CI or staging ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- Don't use when: reads and writes have the same shape and similar load.

**Event sourcing** (beyond the SAiP catalogue; described by Fowler and Greg Young).
- Intent: persist the append-only sequence of domain events; derive current state by replaying (plus snapshots).
- QAs: + auditability, temporal queries, replay for debugging; − schema evolution of events, replay cost, complexity.
- Signature: `EventStore`, `append(streamId, events, expectedVersion)`, aggregate `apply(event)` methods, EventStoreDB (now KurrentDB), Axon, Marten.
- Erosion: events edited in place; events named after CRUD operations; no upcasting strategy; other services reading the event store directly as an integration API.
- Enforce: event classes are append-only (a test that fails if a released event type's schema changes incompatibly); replay test from an empty store ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- Don't use when: nobody needs history or audit; a status column suffices.

**Saga** (beyond the SAiP catalogue; Garcia-Molina and Salem, 1987; popularized for microservices).
- Intent: a long-running business transaction across services as a sequence of local transactions with compensating actions. *Orchestration*: a coordinator tells each participant what to do. *Choreography*: participants react to each other's events.
- QAs: + availability without distributed locks; − complexity, no isolation (intermediate states are visible).
- Signature: saga/process-manager classes with state machines, Temporal/Camunda workflows (orchestration); chains of event handlers (choreography).
- Erosion: missing compensations; choreography chains nobody can draw; 2PC attempted across HTTP; steps that are not idempotent although delivery is at-least-once.
- Enforce: a test per step that runs its compensation; a generated diagram of the event chain kept in the docs ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- Don't use when: one database can hold the whole transaction.

**Transactional outbox** (beyond the SAiP catalogue; documented at microservices.io by Chris Richardson).
- Intent: write the business change and an `outbox` row in one local transaction; a relay (poller or CDC such as Debezium) publishes the row.
- QAs: + reliable messaging without dual writes; − latency, at-least-once delivery so consumers must be idempotent.
- Signature: `outbox` table, relay job, `idempotency_key`/dedup table on consumers.
- Erosion: `repo.save(); broker.publish()` in the same method without an outbox (R4); relay that deletes rows before the broker acknowledges.
- Enforce: an import rule that only the relay package may import the broker producer client (so no method both saves via a repository and publishes); a consumer idempotency test that replays the same message twice ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- Don't use when: the message is best-effort (metrics, cache hints) and loss is acceptable.

**Backend for Frontend (BFF)** (beyond the SAiP catalogue; described by Sam Newman, from practice at SoundCloud).
- Intent: one backend per client type (web, mobile, partner) that aggregates downstream calls and shapes responses for that client.
- QAs: + client-specific performance (fewer round trips), frontend team autonomy; − duplicated logic across BFFs, one more deployable per client.
- Signature: `bff-web/`, `bff-mobile/` services or GraphQL servers owned by the client teams; resolvers that fan out to domain services.
- Erosion: domain rules or pricing living in the BFF (M3, M5); one BFF serving every client (it has become a gateway).
- Enforce: BFF modules may not import domain-rule or pricing packages; they call domain services only through their published clients ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- Don't use when: a single client type exists, or the API already fits the client.

**API gateway** (beyond the SAiP catalogue).
- Intent: a single entry point that routes, authenticates, rate-limits and terminates TLS (Kong, Envoy, AWS API Gateway, NGINX, Traefik).
- QAs: + security and cross-cutting policy in one place, hides internal topology; − extra hop, SPOF and bottleneck unless replicated.
- Signature: gateway route files, auth plugins, rate-limit policies checked into an infra repo.
- Erosion: business logic or orchestration in gateway plugins or scripts; services that trust any request because "the gateway checked" while also being reachable directly (C1).
- Enforce: network policy (Kubernetes `NetworkPolicy`, security groups) so services accept traffic only from the gateway or mesh, plus a test that calls a service directly and expects refusal; a config lint that gateway route files contain no scripts or custom code ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- Don't use when: one or two services and no external clients; a reverse proxy or the service itself covers TLS and auth.

**Service mesh / sidecar** (SAiP 4th ed. lists service mesh under performance).
- Intent: sidecar proxies (Envoy under Istio, or Linkerd's proxy) handle mTLS, retries, timeouts, routing and telemetry outside application code.
- QAs: + uniform resilience, security and observability without code changes; − operational cost, latency per hop, a new control plane to run.
- Signature: sidecar injection labels in Kubernetes manifests, `VirtualService`/`DestinationRule` (Istio) or `ServiceProfile` (Linkerd) resources. Resource kinds are version-dependent: newer Linkerd releases configure retries and timeouts on Gateway API `HTTPRoute` rather than `ServiceProfile`, and Istio ambient mode runs without sidecar injection. Check the installed version.
- Erosion: retries configured in both app and mesh, multiplying load during incidents (R1); services still doing their own TLS inconsistently.
- Enforce: a CI check that no route has retries configured both in app code (retry library config) and in mesh resources; a policy that every namespace has mesh mTLS in strict mode ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- Don't use when: a handful of services; a library or gateway covers the needs.

**Serverless / FaaS** (beyond the SAiP catalogue).
- Intent: functions triggered by events or HTTP, scaled and billed per invocation by the platform (AWS Lambda, Google Cloud Functions, Azure Functions).
- QAs: + scale to zero, low ops for spiky load, deployability per function; − cold starts, execution time limits, vendor lock-in, hard local testing and tracing.
- Signature: `serverless.yml`, SAM/CDK/Terraform function definitions, `handler(event, context)` entry points, one folder per function.
- Erosion: functions calling functions synchronously in chains (distributed monolith, R2); state in function globals assumed to persist; domain logic duplicated per handler (M2, M3).
- Enforce: an IaC or code lint that no function invokes another function synchronously (e.g. SDK `invoke` calls with a request/response invocation type); handlers import shared domain modules instead of redefining rules ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- Don't use when: steady high load (cost), long-running work, or tight tail-latency budgets.

**Offline-first sync** (beyond the SAiP catalogue).
- Intent: the local store (SQLite/Room, IndexedDB, Realm, Core Data) is the source of truth for the UI; a sync engine reconciles with the server using last-writer-wins, version vectors or CRDTs.
- QAs: + availability and latency on bad networks; − conflict resolution, schema migration on devices, larger client footprint.
- Signature: a repository that reads from the local DB and exposes a stream; a sync worker (WorkManager, background fetch, service worker) with a pending-changes queue.
- Erosion: UI reading the network directly; no documented conflict policy; sync that silently drops rejected writes. See [platforms-and-domains.md](platforms-and-domains.md).
- Enforce: UI and ViewModel modules may not import the HTTP/network client; a sync test that creates conflicting edits on two clients and asserts the documented resolution ([fitness-functions.md](fitness-functions.md#2-structural-rules-per-ecosystem)).
- Don't use when: the app is online-only by nature (live bidding, real-time multiplayer) or data must never be stale on screen.

**Cloud-edge split** (beyond the SAiP catalogue).
- Intent: place latency-, bandwidth- or privacy-sensitive work on device or edge nodes, and heavy, shared or authoritative work in the cloud.
- Decide per function with explicit latency, bandwidth, energy and privacy scenarios; record the placement in the deployment view.
- Erosion: authoritative decisions (payments, entitlements) taken only on the device; edge nodes that need the cloud to be up for every request. Details in [platforms-and-domains.md](platforms-and-domains.md).
- Enforce: a test that runs the edge component with the cloud endpoint unreachable and asserts its degraded-mode scenario; a rule that entitlement/payment decisions are only made in server modules ([fitness-functions.md](fitness-functions.md#3-non-structural-fitness-functions)).
- Don't use when: all work fits the latency budget from the cloud and no data has privacy or bandwidth constraints.

**Durable job queue / competing consumers** (beyond the SAiP catalogue; Hohpe and Woolf's *Competing Consumers*).
- Intent: a deferred side effect (email, webhook, payment, export) is recorded as a job in durable storage and executed by one or more workers that claim jobs, so it survives dependency outages and restarts. Unlike the transactional outbox, the side effect *is* the job; there may be no event or broker at all.
- QAs: + availability of the request path (it only enqueues), no lost work, retry without blocking users, horizontal scaling of workers; − latency until the effect happens, at-least-once execution (handlers must be idempotent), a worker process to deploy and monitor.
- Signature: a jobs/outbox table with `status` (`pending -> sending -> sent | retry | dead`), `attempts`, `run_after`, `locked_until`/`lease_expires_at`, `last_error`; a claim query with Postgres `SELECT ... FOR UPDATE SKIP LOCKED` (9.5+); or a library: River (Go, Postgres), Oban (Elixir, Postgres), Solid Queue and good_job (Rails), Hangfire (.NET), Celery, RQ, Dramatiq, ARQ (Python, broker/Redis), BullMQ (Node, Redis), Sidekiq (Ruby, Redis), SQS with a visibility timeout.
- Design rules: enqueue in the caller's transaction (the job exists if and only if the business change committed); a lease or visibility timeout longer than the worst-case batch (batch size x per-call timeout + margin); a dead state after a delivery deadline, with an alert and a replay path; idempotent handlers or an idempotency key sent to the receiver; payloads that hold references, not secrets or unbounded PII; a purge job for finished rows. Retry rules in [quality-attributes-runtime.md](quality-attributes-runtime.md) §1 (e) Retry.
- Erosion: must-not-lose work sent through in-memory mechanisms (FastAPI `BackgroundTasks`, `asyncio.create_task`, an executor); a lease shorter than the batch duration (duplicates); no dead state (poison jobs retry forever); workers started inside every web replica with no concurrency limit; handlers that are not idempotent.
- Enforce: a test that kills the worker mid-batch and asserts each job runs to completion exactly once in effect; a test that a second worker cannot claim a leased row; an architecture rule that the mail/webhook client is imported only by job handlers ([fitness-functions.md](fitness-functions.md)).
- Don't use when: loss is acceptable and stated (best-effort analytics), or the caller needs the result synchronously.

### 3.4 Resilience set
Tactic details live in [quality-attributes-runtime.md](quality-attributes-runtime.md). Signature and erosion here:

| Pattern | Source | Code signature | Erosion / smell |
|---|---|---|---|
| Timeout | Detection mechanism behind SAiP's fault-detection tactics | explicit client timeouts (`OkHttp` `callTimeout`, `context.WithTimeout`, `AbortSignal.timeout`, `httpx.Timeout`) | defaults or none; Go `http.DefaultClient` has no timeout (R1) |
| Retry + backoff | SAiP 4th ed. has a *retry* tactic; backoff with jitter from practice | retry policies (Resilience4j, Polly, `tenacity`, `p-retry`) | retries on non-idempotent calls; retries at several layers multiplying load (R1) |
| Circuit breaker | SAiP 4th ed. availability pattern; popularized by Nygard, *Release It!* | Resilience4j `CircuitBreaker`, Polly, `opossum`, `gobreaker` | breaker with no fallback; one breaker shared by unrelated dependencies |
| Bulkhead | Beyond the SAiP catalogue; Nygard | separate thread pools, connection pools or semaphores per dependency | one shared pool that a slow dependency exhausts (R5) |
| Rate limiter / throttling | SAiP 4th ed. throttling pattern; manage work requests | token bucket in gateway or middleware, `golang.org/x/time/rate` | limit enforced only client-side (C1) |
| Load balancer | SAiP 4th ed. performance pattern | LB config, Kubernetes Service, client-side LB | sticky sessions hiding state in instances |
| Redundant spare | SAiP availability tactic/pattern (active, passive, cold spare) | replicas, standby DB, failover config | failover never tested |
| Triple modular redundancy | SAiP availability pattern: three components plus a voter | three independent computations and a voting step | the three share a common-mode failure (same code, same input) |

### 3.5 Migration set
- **Strangler fig** (beyond the SAiP catalogue; Fowler, 2004): route traffic through a facade or proxy; move capabilities one by one to the new system; retire the old one. Signature: routing rules per path, feature flags. Erosion: the new system calls back into the old one for everything; no retirement date (E2).
- **Branch by abstraction** (beyond the SAiP catalogue; Paul Hammant): introduce an interface over the old implementation, move callers to it, build the new implementation behind it, switch, delete the old one, all on trunk. Erosion: both implementations kept indefinitely (E2, E3).
- **Expand/contract (parallel change)** (beyond the SAiP catalogue; described by Danilo Sato on martinfowler.com): add the new schema or API alongside the old one, migrate readers and writers, then remove the old one. Required for zero-downtime DB migrations. Erosion: a migration that renames or drops a column in the same release that stops using it.
- **Anti-corruption layer** (Evans, *Domain-Driven Design*, 2003): a translation layer that keeps a legacy or external model from leaking into your domain. Signature: `acl/` or `integration/<vendor>/` with translators and your own types on the inside. Erosion: vendor types in domain signatures (I4, M5).

**Migration checklist** (apply to any of the four):
1. Write the target state and the retirement condition for the old path in an ADR before starting.
2. Keep both paths behind one seam (routing rule, interface, or flag) and measure traffic on each.
3. Make every step independently deployable and reversible; never combine expand and contract in one release.
4. Add a fitness function that fails when new code uses the old path.
5. Track the old path's remaining call sites; the migration is done only when they reach zero and the code is deleted.

---

## 4. Comparison and selection

### 4.1 Comparison table
`+` promotes, `-` inhibits, `0` neutral or depends on implementation. For **Ops cost**, `+` means cheaper to operate and `-` means more expensive. Use it as a starting point for tradeoff discussion, not as a verdict; your scenarios decide.

| Pattern | Modif. | Perf. | Avail. | Security | Testab. | Deploy. | Scal. | Ops cost |
|---|---|---|---|---|---|---|---|---|
| Layered | + | - | 0 | + | + | 0 | 0 | 0 |
| Broker | + | - | + / - (broker SPOF) | - | - | + | + | - |
| MVC | + | 0 | 0 | 0 | + | 0 | 0 | 0 |
| Pipe-and-Filter | + | + (throughput) / - (latency) | 0 | 0 | + | 0 | + | 0 |
| Client-Server | 0 | - (server bottleneck) | - unless replicated | + | 0 | + (server-side) | + | 0 |
| Peer-to-Peer | 0 | + | + | - | - | - | + | + |
| SOA | + | - | 0 | 0 | - | + | + | - |
| Publish-Subscribe | + | - | + | 0 | - | + | - (SAiP); partitioned brokers such as Kafka can make it + | - |
| Shared-Data | - (schema coupling) | - | - (store SPOF) | + | 0 | - | - | + |
| Map-Reduce | 0 | + (batch) | + | 0 | + | 0 | + | - |
| Multi-tier | + | - | + | + | 0 | + | + | - |
| Hexagonal / clean | + | 0 | 0 | 0 | + | 0 | 0 | 0 |
| Modular monolith | + | + | 0 | 0 | + | - | 0 | + |
| Microkernel | + | 0 | 0 | - | - | + | 0 | 0 |
| Microservices | + | - | + | 0 | - | + | + | - |
| CQRS | 0 | + | 0 | 0 | 0 | 0 | + | - |
| Event sourcing | 0 | 0 | 0 | + (audit) | + | 0 | 0 | - |
| Serverless | 0 | - (cold start) | + | 0 | - | + | + | + (spiky) / - (steady) |

### 4.2 Selection guide
Find the dominant scenario or constraint, start from the default, and switch only when the condition holds.

| Dominant scenario / constraint | Default | Runner-up | Runner-up wins when |
|---|---|---|---|
| Team under ~10 people, new product, domain still moving | Modular monolith | Microservices | Two or more teams need independent deploy cadence and boundaries are stable |
| Independent deploy cadence per team | Microservices (or services per team) | Modular monolith with separate release trains | Teams share one release process anyway, or ops capacity is thin |
| High read volume, reads have the same shape as writes | Read replicas + cache (+ indexes) | CQRS | Read queries need joins or denormalization across aggregates that the write schema cannot serve within the latency budget |
| Read shape differs from write shape (search, dashboards, cross-aggregate views) | CQRS with projections | Materialized views in the same DB | One DB can refresh the views within the freshness requirement |
| Stream or batch transformations | Pipe-and-filter | Map-Reduce | Data exceeds one machine and partitions cleanly |
| Many third-party integrations | Hexagonal + anti-corruption layer per vendor | SOA with ESB | Many heterogeneous organizations and an existing integration platform |
| Complex UI state, many async events | MVI / UDF | MVVM | Screens are mostly independent forms with little shared state |
| Third parties extend the product | Microkernel / plug-in | Pub-sub extension points | Extensions only react to events and never add behavior to the core flow |
| Portability across platforms or backends | Layered / hexagonal with platform adapters | Multiplatform shared core (KMP, shared TS) | Most logic is identical across platforms |
| Location transparency over changing servers | Broker / service discovery | Client-server + load balancer | Server set is small and static |
| Unknown, changing receivers of events | Publish-subscribe | Direct calls behind an interface | There is one receiver and it must reply |
| Long cross-service business process | Saga (orchestration) | Choreography | Few steps (about three or fewer) and teams own each step independently |
| Reliable events from a DB change | Transactional outbox | CDC directly on business tables | You accept exposing the table schema as the event contract |
| A deferred side effect (email, webhook, payment) must survive a dependency outage and restarts | Outbox/job table + worker on the existing DB (`FOR UPDATE SKIP LOCKED`, or a DB-backed job library) | Broker + task library (Celery, BullMQ, Sidekiq) | Sustained load above roughly 50 jobs/s, or a broker is already operated |
| Must work offline / on poor networks | Offline-first sync | Cache with request queue | Data is read-mostly and conflicts are impossible |
| Replace a legacy system gradually | Strangler fig | Branch by abstraction | The legacy part lives inside the same codebase |
| Security zones between UI, logic and data | Multi-tier | Single tier with strict authz | Everything runs on one trusted host |

### 4.3 Recognizing the as-is pattern from a repo
Before reviewing against a pattern, confirm which one is really there. Use the style-detection table and procedure in [architecture-recovery-and-metrics.md §9](architecture-recovery-and-metrics.md#9-style-detection), then check the result against the matching card's constraints here.

### 4.4 Worked choices
1. *A document service converts uploads: virus scan, OCR, thumbnail, index. Operators add steps each quarter.* **Pipe-and-filter** with a queue between stages: steps are independent and recombinable (modifiability) and run in parallel (throughput). Weakness accepted: per-stage latency, fine for async processing. Runner-up Map-Reduce lost: documents arrive one by one, not as a large partitioned batch. Add: bounded queues and a dead-letter stage.
2. *A four-developer startup builds a B2B invoicing product; the domain changes weekly.* **Modular monolith** with modules per capability (customers, invoices, payments). Runner-up microservices lost: one team, unstable boundaries, no ops capacity. Add: a module-boundary fitness function now so extraction stays possible later.
3. *A retail app integrates three payment providers, two shipping carriers and a tax API; providers change yearly.* **Hexagonal + anti-corruption layer per vendor**: the domain owns `PaymentGateway`, `ShippingQuote` ports; each vendor gets an adapter translating its model. Runner-up SOA with an ESB lost: one organization, no integration platform to justify.
4. *A Kotlin Multiplatform app shows a live order-tracking screen fed by pushes, retries and local cache.* **MVI/UDF** in the shared ViewModel: one immutable `UiState`, events reduced in one place, testable in `commonTest`. Runner-up MVVM with several mutable flows lost: state from three async sources must stay consistent.

**How to argue a choice:** name the pattern, quote the scenario that drives it, name the QA it serves, name the weakness you accept and the tactic that repairs it, and say why the runner-up lost.

### 4.5 Patterns combine; document each in its own view
Real systems use several patterns at once, each answering a different structural question. Do not argue "layered vs client-server"; they are orthogonal.
- **Module view:** layered or hexagonal inside each deployable; modular monolith boundaries.
- **C&C view:** client-server between app and backend; pub-sub between services; pipe-and-filter inside a processing worker; MVI inside the UI.
- **Allocation view:** multi-tier or cloud-edge placement; map-reduce for offline analytics.
When a review finds two patterns in conflict (e.g. an event-driven integration style and a shared database), name the conflict as a tradeoff point and record which one wins in an ADR. See [documentation.md](documentation.md) for view selection.

### 4.6 Pattern conformance checklist (for reviews)
Run this against every pattern the docs or ADRs claim:
1. Is the pattern named anywhere (README, ADR, architecture description)? If not, recover it first (§4.3).
2. List the pattern's constraints from its card; turn each into a checkable rule (import direction, allowed connectors, data ownership).
3. Check each rule against the code or dependency graph; cite `file:line` for every violation.
4. Check that the weaknesses on the card have their repairing tactics (e.g. broker redundancy, pub-sub dead-letter handling, outbox for events).
5. Check that a fitness function enforces the constraints; if not, propose one from [fitness-functions.md](fitness-functions.md).
6. Classify each violation: intentional and documented (fine), intentional and undocumented (E1), or accidental erosion (the matching smell ID).
7. Report in [../templates/architecture-review-report.md](../templates/architecture-review-report.md) with severity per [review-playbook.md](review-playbook.md).

---

## 5. Architectural anti-patterns

Each entry: what it is, how to detect it, and the smell IDs in [review-playbook.md](review-playbook.md).

**Big ball of mud** (Foote and Yoder, 1997). No discernible structure; everything can call everything.
- Detect: dependency graph is one large strongly connected component; no module has a stable public API; many files change together at random ([architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md)).
- Smells: D2, M1, M2. Fix: pick seams by change coupling, carve out a modular monolith incrementally, freeze new edges with a fitness function.

**Distributed monolith.** Services that must change, deploy and fail together.
- Detect: several services with the same DB credentials or migrations (shared DB); release notes or pipelines that deploy services in lockstep; synchronous call chains three or more hops deep per user request (chatty sync calls); commits or PRs that routinely touch several services at once (cross-service change coupling); shared "common model" libraries that force simultaneous upgrades.
- Smells: R2, I5, I2, M2, D3. Fix: merge services that always change together, or split data ownership and switch to asynchronous events with an outbox.

**God service / god module.** One service or module holds most rules and data; everything depends on it.
- Detect: highest fan-in and fan-out, largest churn, most owners; names like `core`, `common`, `manager`, `main-service`.
- Smells: M1, M4, D3. Fix: split by business capability using change-coupling clusters.

**Cyclic modules.** Modules that depend on each other directly or transitively.
- Detect: `madge --circular`, dependency-cruiser `no-circular`, `jdeps -verbose:package` or `--dot-output` fed to a cycle check (jdeps does not report cycles itself), Gradle project cycles failing the build, import-linter `independence` contracts (forbid any imports between sibling modules, which rules out cycles among them); Go refuses import cycles at compile time, so look for cycles hidden through interface registration or globals.
- Smells: D2, D5. Fix: extract the shared concept into a lower module, or invert with an interface owned by the depending side.

**Leaky abstraction.** An interface exposes implementation types or failure modes.
- Detect: port or repository signatures returning ORM entities, `ResultSet`, HTTP response objects, vendor SDK types; callers catching vendor exceptions.
- Smells: I1, D6, I4. Fix: map to domain types at the adapter; translate errors to domain errors.

**Chatty interfaces.** Many fine-grained calls across a process or network boundary per use case.
- Detect: loops that call a remote client or DB per item (N+1); traces with dozens of spans per request; UI making many calls to render one screen.
- Smells: I2, R2. Fix: coarse-grained or batch endpoints, BFF aggregation, data-loader batching, event-carried state.

**Anemic domain model where it hurts.** Entities are data bags and the rules live scattered in services, controllers and UI.
- Detect: the same validation or pricing rule duplicated across handlers; entities with only getters/setters while services contain long conditional logic about their state. Harm is real only when rules are rich and duplicated; a CRUD app with anemic models is fine.
- Smells: M3, M2. Fix: move invariants into the entity or a domain service; make illegal states unrepresentable.

**Architecture sinkhole** (named by Mark Richards, *Software Architecture Patterns*, O'Reilly, 2015). In a layered system most requests pass through layers that add nothing.
- Detect: service methods that only forward to a repository with identical signatures; mappers that copy fields one-to-one at every boundary. A minority of pass-throughs is normal; a majority means the layering is ceremony.
- Smells: M4. Fix: relax layering for pure reads (document the bridge), or collapse the empty layer.

**Golden hammer** (Brown et al., *AntiPatterns*, 1998). One familiar pattern or technology applied everywhere regardless of scenario (every integration is a Kafka topic, every feature a microservice).
- Detect: ADRs with no runner-up considered; the same pattern chosen for contradictory QA priorities.
- Smells: E1. Fix: re-run §4.2 for the affected decisions and record the runner-up.

**Premature distribution.** Services or serverless functions introduced before the domain boundaries are known.
- Detect: services with fewer than a handful of endpoints that are always called together; most commits touching several services; a platform team larger than the product team.
- Smells: R2, M4, M2. Fix: merge back into a modular monolith and re-split later along observed change coupling.

**Pattern claimed but not present.** A README or ADR says "hexagonal" or "layered" but the code does not follow it.
- Detect: compare the documented rules with the recovered dependency graph; any violated rule is a finding.
- Smells: E1, E4. Fix: either enforce the rule with a fitness function or change the documentation to describe the as-is architecture.
