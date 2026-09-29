# Change-time qualities: modifiability, testability, deployability, integrability, other QAs, cross-QA tradeoffs

Use this file when the question is "what does it cost to change, test, ship or plug into this system?" It holds
one card per quality attribute (QA). Runtime qualities (availability, performance, security, safety, usability,
energy efficiency) are in [quality-attributes-runtime.md](quality-attributes-runtime.md). The cross-QA tradeoff
matrix at the end (§7) covers both files.

Theory adapted in part from the Wikipendium TDT4240 compendium (CC BY-SA 3.0).

**Source conventions.** Chapter numbers are *Software Architecture in Practice* (SAiP), Bass, Clements & Kazman:
4th ed. (4e, 2021) first, 3rd ed. (3e, 2013) in parentheses. Chapter titles and numbers are confirmed; section
numbers are not, so none are given. Tactic names and groupings for 4e that were reconstructed rather than checked
against the printed book carry **[verify wording in SAiP 4th ed.]**, the same tag as
[quality-attributes-runtime.md](quality-attributes-runtime.md). Keep the tag when you quote them in an ADR or a
review report; do not silently "correct" them to 3e wording. Cite chapters by title ("the Modifiability chapter");
that is safe in both editions.

Related: scenario elicitation and ASRs in [design-workflow.md](design-workflow.md); package layout and interface
rules in [module-layout-and-interfaces.md](module-layout-and-interfaces.md); which patterns bundle which tactics in
[architectural-patterns.md](architectural-patterns.md); measuring coupling and change coupling from git in
[architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md); turning a QA into an automated check
in [fitness-functions.md](fitness-functions.md); how findings are reported in [review-playbook.md](review-playbook.md).

## Card layout

Every QA card uses the slot letters and names of [quality-attributes-runtime.md](quality-attributes-runtime.md), plus
one extra slot (j). Where two slots share one table they share a heading. Fill the same slots when you add a QA of
your own (§5.2).

| Slot | Use it to |
|---|---|
| (a) Definition | Decide whether the concern really is this QA |
| (b) General scenario + (c) concrete scenario | One table: general values per part, and a filled example. The response measure is a number with a unit and a threshold |
| (d) Tactic tree + (e) code-level realization | One table: the SAiP tactic, and what it looks like in a real codebase |
| (f) Tactic-absent signals | Grep for these, or look for them in a diff |
| (g) Measurement | The metric or fitness function that proves the response measure holds |
| (h) Tradeoffs | What the tactics cost other QAs; record them as tradeoff points |
| (i) Patterns | Patterns that bundle the tactics, and when to pick which |
| (j) Reviewer quick checks | Three to five yes/no questions for a PR or repository review |

---

## 1. Modifiability — SAiP 4e ch. 8 (3e ch. 7)

### 1.1 (a) Definition

Modifiability is the cost and risk of making a change. Plan it with four questions: **what** is likely to change,
**how likely** each change is, **when and by whom** it is made (developer at design time, operator at deploy time,
end user at runtime), and **what it costs** (effort, elapsed time, number of artifacts touched, defects introduced).
Two structural properties drive it: **cohesion** (how strongly the responsibilities inside one module belong
together; aim high) and **coupling** (how likely a change in one module is to ripple into another; aim low).
Do not design for every change. Design for the changes the stakeholders expect, and keep everything else simple.

### 1.2 (b) General scenario and (c) concrete scenario

| Part | General values | Example |
|---|---|---|
| Source | Developer, system administrator, end user | Payments team developer |
| Stimulus | Add, delete or modify a function, a QA, a capacity or a technology | Add a second payment provider |
| Artifact | Code, data, interfaces, components, resources, configuration | `payments` module and its wiring |
| Environment | Design, compile, build, initiation (startup) or runtime | Development time |
| Response | Make the change, test it, deploy it | Provider added as a new adapter behind `PaymentGateway` |
| Response measure | Artifacts touched, effort, elapsed time, money, side effects on other QAs, new defects | ≤ 1 new module, ≤ 2 existing files edited (DI wiring, config), ≤ 3 person-days, no change in `domain/` |

### 1.3 (d) Tactic tree and (e) code-level realization

4e groups [verify wording in SAiP 4th ed.]: **increase cohesion**, **reduce coupling**, **defer binding**. 3e had four groups: *reduce size
of a module* (split module) as a separate group, *increase semantic coherence* under increase cohesion, and
*refactor* under reduce coupling. 4e moves split module into increase cohesion, renames increase semantic coherence
to **redistribute responsibilities**, and no longer lists refactor as a tactic of its own [verify wording in SAiP 4th ed.].

| Group | Tactic | What it looks like in code |
|---|---|---|
| Increase cohesion | Split module | Break a 3 kLOC `AppService`/`utils` into modules that each change for one reason; split a Gradle/npm/Go module whose half the dependents never use |
| | Redistribute responsibilities (3e: increase semantic coherence) | Move formatting out of the domain, persistence out of the ViewModel, validation next to the type it guards |
| Reduce coupling | Encapsulate | An explicit interface with hidden internals: Kotlin `internal`, Java module `exports`, Go `internal/` directories, TS package `exports` map. Python `_private` names and `__all__` are conventions only (`__all__` governs `from m import *`; nothing blocks a direct import), so enforce the boundary with import-linter contracts |
| | Use an intermediary | Break A→B into A→X→B: event bus/publish-subscribe, repository, facade, broker, proxy, message queue. Pick the intermediary by the kind of dependency (data, call, timing, location) |
| | Restrict dependencies | Layer and module rules enforced by the build: Gradle module graph, `internal/`, ArchUnit/Konsist, dependency-cruiser, eslint-plugin-boundaries, import-linter, go-arch-lint (see [fitness-functions.md](fitness-functions.md)) |
| | Abstract common services | One parameterised implementation of similar services: one `HttpClient` wrapper with retry and auth, one `Storage` port for S3/GCS/local disk |
| | (3e) Refactor | Pull duplicated responsibilities out of two modules into one shared place. Treat it as a technique now, not a named 4e tactic |
| Defer binding | Bind later in the life cycle | Turn a code change into a config, flag or plug-in change. See binding-time table below |

A restrict-dependencies rule in ArchUnit (Java/Kotlin on the JVM; needs the `com.tngtech.archunit:archunit-junit5`
test dependency). The rule only runs as an `@ArchTest` field in an `@AnalyzeClasses` class (or when `.check(...)` is
called on it). Full setup, freezing for legacy code and version notes: [fitness-functions.md](fitness-functions.md) §2.1.

```java
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

@AnalyzeClasses(packages = "com.example")
class DomainIsolationTest {
    @ArchTest
    static final ArchRule domainIsolated =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..infrastructure..");
}
```

**Binding-time table (defer binding).** The later the binding, the cheaper the change, but the more machinery,
configurations to test and runtime failure modes you pay for. Choose the earliest binding time that still meets the
change scenario.

| Binding time | SAiP mechanisms | Kotlin / Java / KMP | TypeScript / Node | Python | Go |
|---|---|---|---|---|---|
| Compile / build | Component replacement in build scripts, compile-time parameterisation, aspects | Android `productFlavors` and build types; KMP source sets with `expect`/`actual` | Bundler defines/env replacement (e.g. Vite `import.meta.env`); separate entry points | Optional dependency extras; separate packages | Build tags (`//go:build`); `-ldflags "-X 'example.com/app/build.Version=1.2.3'"` sets a package-level string *variable* (not a `const`; non-string variables are left unchanged) at link time |
| Deployment | Configuration-time binding | Per-environment config files, Helm values; the deploy sets `SPRING_PROFILES_ACTIVE` (Spring then selects `@Profile` beans at startup) | Per-environment config injected by the deploy | Settings chosen by environment | Config file or env selected per deploy |
| Startup / initialisation | Resource files | DI container wiring (Koin/Hilt/Spring) reading config at start, including Spring `@Profile` bean selection; `java.util.ServiceLoader` (`META-INF/services/...` or `provides ... with` in `module-info.java`) | Config read at boot (`process.env`), DI container composition root | `importlib.metadata.entry_points(group="app.plugins")` (selection API is Python 3.10+) plus `[project.entry-points."app.plugins"]` in `pyproject.toml` | Registration in `init()` via blank import, as `database/sql` drivers do (`import _ "..."`) |
| Runtime | Runtime registration, dynamic lookup, interpreting parameters, name servers, plug-ins, publish-subscribe, shared repositories, polymorphism | Feature flags / remote config, strategy objects chosen at runtime, service discovery | Dynamic `import()`, feature flags (e.g. OpenFeature SDKs), service discovery | Feature flags, strategy registry dicts | Feature flags, service discovery; the stdlib `plugin` package works only on some OSes and needs the identical toolchain, so avoid it |

### 1.4 (f) Tactic-absent signals

- **Change coupling**: files in different modules that change in the same commits again and again. It is the most
  reliable modifiability signal because it measures real ripple effects. Mine it from git history as described in
  [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md).
- **Shotgun surgery**: one feature PR touches 5+ modules or every layer for a trivial change.
- **Divergent change / god module**: one file changes for many unrelated reasons (`Utils`, `Manager`, `Helper`,
  `common`, `app.kt` with 1 kLOC+ and high churn).
- **Dependency cycles** between packages or modules (tools in [fitness-functions.md](fitness-functions.md)).
- **Inward dependencies pointing outward**: domain code importing an ORM, HTTP framework, UI toolkit or vendor SDK.
- **Type switches on a closed set** (`when(type)`/`switch` on a provider enum repeated in several files): each new
  variant edits every switch. Replace with polymorphism or a registry.
- **Hard-coded environment facts**: URLs, tenant names, limits and chain/network IDs in source instead of config.
- **Speculative generality**: plug-in frameworks and abstract factories with one implementation and no change
  scenario behind them. That is complexity with no payoff, and it is a finding too.

### 1.5 (g) Measurement

- **Change coupling across module boundaries**: share of co-changing file pairs whose files sit in different modules,
  mined from git as in [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md) §11. Track it per
  quarter; a rising ratio means ripple effects are growing.
- **Files and modules touched per feature PR**: `git diff --stat <base>...<head>` for files,
  `git diff --dirstat=files,0 <base>...<head>` for the spread over directories. Compare with the scenario's measure
  (e.g. ≤ 2 existing files edited).
- **Dependency-rule violations**: count from ArchUnit/Konsist, import-linter, dependency-cruiser or go-arch-lint; a
  frozen baseline that only goes down ([fitness-functions.md](fitness-functions.md) §5).

### 1.6 (h) Tradeoffs

Intermediaries and indirection cost latency and make control flow harder to follow (performance, analysability).
Late binding multiplies configurations to test and adds runtime failure modes (testability, availability).
Over-generalising early raises the cost of the changes that actually arrive.

### 1.7 (i) Patterns

Layers, ports and adapters (hexagonal), plug-in/microkernel, publish-subscribe, client-server, pipe-and-filter.
4e discusses client-server, plug-in, layers and publish-subscribe in the modifiability chapter [verify wording in
SAiP 4th ed.]. Details and tactic bundles: [architectural-patterns.md](architectural-patterns.md).

### 1.8 (j) Reviewer quick checks

- Does the PR's diff stay inside the module the change scenario names, or does it ripple?
- Did any new import cross a layer or module boundary in the forbidden direction?
- Is the new variability bound at the earliest time that meets the scenario (build flag before runtime flag)?
- Does a new abstraction have a change scenario or a second implementation behind it?
- Are the change-coupled file pairs from git history in the same module?

---

## 2. Testability — SAiP 4e ch. 12 (3e ch. 10)

### 2.1 (a) Definition

Testability is how easily software can be made to reveal its faults through (typically execution-based) testing.
Two conditions must hold: you can **control** each component's inputs and internal state, and you can **observe**
its outputs and state. Testability is a property of the architecture, not the same thing as having tests: it is
what makes tests cheap, fast and deterministic.

### 2.2 (b) General scenario and (c) concrete scenario

| Part | General values | Example |
|---|---|---|
| Source | Unit, integration, system or acceptance tester; automated tool; end user | CI pipeline on every PR |
| Stimulus | A test suite runs after an increment is completed or integrated, or the system is delivered | PR changes settlement logic |
| Artifact | The part of the system under test | `settlement` domain module |
| Environment | Design, development, compile, integration, deployment or runtime | CI runner, no network, no real database |
| Response | Execute the suite and capture results; capture the activity that led to a fault; control and monitor state | Suite runs against in-memory fakes and a virtual clock |
| Response measure | Effort to find a fault or reach coverage; test run time; time to prepare the environment; length of the longest dependency chain; reduction in risk exposure | Domain suite < 60 s; 0 tests need network; ≥ 90 % branch coverage of `settlement`; 0 flaky reruns over 50 CI runs |

### 2.3 (d) Tactic tree and (e) code-level realization

4e groups: **control and observe system state** and **limit complexity**; names are largely unchanged from 3e
[verify wording in SAiP 4th ed.].

| Group | Tactic | What it looks like in code |
|---|---|---|
| Control and observe system state | Specialized interfaces | Test-only hooks to set, read, reset or dump state; a verbose/diagnostic mode. Keep them out of production builds (test source set, `internal` + `@VisibleForTesting`, separate debug build type) |
| | Record/playback | Capture traffic crossing an interface and replay it: recorded HTTP fixtures, event logs replayed into a reducer, golden files |
| | Localize state storage | Keep mutable state in one place (a store, a repository, one `StateFlow`), so a test sets it up in one call and inspects it in one place |
| | Abstract data sources | Data behind a port; tests swap an in-memory implementation for the database, network or device |
| | Sandbox | Isolate from the real world: virtual clock, in-memory network, Testcontainers-style disposable databases, a regtest/devnet chain |
| | Executable assertions | `require`/`check`, invariant checks in constructors, assertion helpers at module boundaries that detect a bad state where it arises |
| Limit complexity | Limit structural complexity | Break dependency cycles, isolate environment dependencies, keep inheritance and dependency chains short, high cohesion and low coupling |
| | Limit nondeterminism | Remove or inject every source of unconstrained behaviour: wall-clock time, random seeds, thread scheduling, iteration order of hash maps, network timing |

**Idioms that implement these tactics.**

- **Port with a fake sibling.** For each boundary that tests need to control, define an interface (or abstract base)
  in the inner module with two implementations beside it: the real one and a deterministic fake. The fake is a
  working in-memory implementation, not a mock script. It pays for itself when it also drives UI previews, demo
  mode and local development. The Wally wallet architecture description records this as an abstract ViewModel base
  with an `…Impl` and a deterministic `…Fake` subclass (10 of its 14 ViewModel families follow the idiom).
  Paraphrased, the rule is: one abstraction with a real and a deterministic fake implementation, and no seam that
  serves only a test: the fake should also earn its place in previews, demo mode or local development.
- **Inject time, randomness and concurrency.** Pass a clock, a random source and a dispatcher/executor in through
  the constructor:
  - JVM: `java.time.Clock` (use `Clock.fixed(...)` or a mutable test clock in tests).
  - Kotlin coroutines: inject `CoroutineDispatcher`; test with `runTest` and `StandardTestDispatcher`, which run on
    virtual time (`advanceTimeBy`, `advanceUntilIdle`). The clock type in Kotlin moved between
    `kotlinx.datetime.Clock` and `kotlin.time.Clock` across recent versions; check the project's versions.
  - Go: accept a `func() time.Time` or a small `Clock` interface; pass `*rand.Rand` with a fixed seed.
  - Python: accept a `now: Callable[[], datetime]` parameter; freezing libraries work but hide the dependency.
  - TypeScript: inject a `now()` function; Jest and Vitest fake timers exist, but explicit injection is clearer.
- **Deterministic scheduler.** Run concurrent code under a scheduler the test controls (virtual-time test
  dispatchers, a manual executor that runs queued tasks on demand) instead of `sleep` and retry.
- **Contract tests.** Run one shared test suite against both the fake and the real adapter, so the fake cannot
  drift from reality. Across service boundaries, use consumer-driven contracts (for example Pact).
- **Test pyramid.** Most tests on the domain through ports and fakes; fewer integration tests per adapter against
  real infrastructure in a sandbox; a thin layer of end-to-end tests. An inverted pyramid (mostly UI/E2E) is an
  architectural signal: logic is not reachable without the full stack.

### 2.4 (f) Tactic-absent signals

- **Singletons and global mutable state**: `object` holding state in Kotlin, static fields, module-level mutable
  variables in Python/TS, package-level `var` in Go.
- **Static time and randomness**: `System.currentTimeMillis()`, `Instant.now()`, `Date.now()`, `datetime.now()`,
  `time.Now()`, `Math.random()`, unseeded `Random()` inside domain logic.
- **Infrastructure constructed in the domain**: `new HttpClient()`, `Database.connect(...)`, `boto3.client(...)`,
  `http.DefaultClient` or an SDK client created inside a use case instead of injected.
- **Test-order dependence**: tests pass alone but fail in a suite, or need a fixed order; shared fixtures mutated
  across tests; `@BeforeAll` that seeds global state.
- **Sleeps and retries in tests** (`Thread.sleep`, `delay` in non-virtual time, `time.sleep`, `setTimeout` waits).
- **Mocks of types you do not own** and mocks nested three deep: the port is missing or at the wrong level.
- **Logic only reachable through the UI or a composable body**, so every test needs a rendered screen.

**Detect them.** Static time and randomness outside tests (ripgrep; adjust the globs to the repo's test layout):

```bash
rg -n 'Instant\.now\(|System\.currentTimeMillis\(|Date\.now\(|datetime\.now\(|time\.Now\(|Math\.random\(' \
   --glob '!**/test/**' --glob '!**/tests/**' --glob '!*_test.go' --glob '!*.test.*' --glob '!*.spec.*'
```

Hits in a composition root or an adapter that wraps the clock are fine; hits in domain code are findings.

Test-order dependence: run the suite in random order. These switches are version-dependent; check the project's
versions first.

| Stack | Random order | Notes |
|---|---|---|
| Go | `go test -shuffle=on ./...` | Go 1.17+. Prints the seed; rerun a failure with `-shuffle=<seed>` |
| pytest | Install the pytest-randomly plugin | On by default once installed; `--randomly-seed=<n>` reproduces an order; `-p no:randomly` disables it. Parallel: pytest-xdist `-n auto` |
| JUnit 5 | `junit.jupiter.testmethod.order.default=org.junit.jupiter.api.MethodOrderer$Random` in `src/test/resources/junit-platform.properties` | Class order: `junit.jupiter.testclass.order.default=org.junit.jupiter.api.ClassOrderer$Random` (5.8+). Parallel: `junit.jupiter.execution.parallel.enabled=true` |
| Jest | `jest --randomize` | Jest 29.2+; `--seed=<n>` reproduces an order |
| Vitest | `sequence: { shuffle: true }` under `test` in the Vitest config | Or `--sequence.shuffle` on the CLI |

Wire the random-order run into CI as described in [fitness-functions.md](fitness-functions.md) §6.

### 2.5 (g) Measurement

- **Domain-suite wall time** from the CI job timings; compare with the scenario threshold (e.g. < 60 s).
- **Tests needing network**: run the domain suite with networking off (for example inside
  `docker run --network none ...`) and count failures; the target is zero.
- **Flaky-rerun rate**: tests that failed and then passed on retry in the same pipeline, over the last N runs, from
  CI retry reports or the test-retry plugin's output; the target is zero.
- **Coverage floor on the domain module only** ([fitness-functions.md](fitness-functions.md) §3).

### 2.6 (h) Tradeoffs

Test hooks and assertions left in production are a security back door. Abstracting data sources adds indirection
(small performance and readability cost). Limiting nondeterminism can fight *introduce concurrency* (performance).
Fakes are code to maintain; contract tests are the price of trusting them.

### 2.7 (i) Patterns

Dependency injection, strategy, intercepting filter [verify wording in SAiP 4th ed.], ports and adapters, MVVM/MVI
with the state held in the ViewModel/store. See [design-patterns-in-code.md](design-patterns-in-code.md).

### 2.8 (j) Reviewer quick checks

- Can the changed logic be tested without network, disk, real time or a device?
- Are clock, randomness and dispatchers injected, not read statically?
- Does every new port have a fake, and does a contract test bind the fake to the real adapter?
- Do the new tests pass in random order and in parallel (commands in §2.4)?
- Are test-only interfaces excluded from the production artifact?

---

## 3. Deployability — SAiP 4e ch. 5 (3e: short entry in ch. 12)

### 3.1 (a) Definition

Deployability is how predictably, quickly and cheaply a new version, or a fix, reaches its execution environment
and starts running, including the ability to **roll it back**. It links the architecture to continuous integration,
continuous deployment and the deployment pipeline. Architecture decides whether parts can be deployed
independently and whether old and new versions can run side by side.

### 3.2 (b) General scenario and (c) concrete scenario [verify wording in SAiP 4th ed.]

| Part | General values | Example |
|---|---|---|
| Source | End user, developer, system administrator, operations engineer, component marketplace | Developer merging a fix |
| Stimulus | A new element is available: bug fix, security patch, feature, upgraded component or platform | Patch to the pricing service |
| Artifact | Components or modules, platform, UI, environment, or a system it interoperates with | `pricing` container image and its schema |
| Environment | Full deployment; subset deployment to specified users, VMs, containers or servers | Production, 5 % canary first |
| Response | Incorporate and deploy the new components; monitor; roll back if it misbehaves | Canary promoted automatically on healthy metrics |
| Response measure | Cost (effort, time, money), extent, frequency, failed deployments, time to roll back, impact on other QAs | Merge to 100 % in ≤ 30 min, 0 manual steps, rollback ≤ 5 min, no failed requests during rollout |

### 3.3 (d) Tactic tree and (e) code-level realization [verify wording in SAiP 4th ed.]

| Group | Tactic | What it looks like in code and config |
|---|---|---|
| Manage deployment pipeline | Scale rollouts | Release to a fraction first: canary, staged rollouts (e.g. percentage rollouts in mobile app stores), region by region |
| | Roll back | Immutable, versioned artifacts (image digest, not `latest`); the previous version stays deployable; database changes that the previous version can still run against |
| | Script deployment commands | Every step in the pipeline as code (CI workflow, Helm/Kustomize/Terraform); no wiki page of manual steps |
| Manage deployed system | Manage service interactions | Several versions coexist and requests are routed to the right one: versioned APIs, traffic splitting in a gateway or service mesh |
| | Package dependencies | Ship the component with its dependencies: container images, lockfiles, static binaries, self-contained bundles |
| | Feature toggle | Deploy code dark and switch it on at runtime; the same flag is a kill switch. Remove the flag when the rollout ends |

### 3.4 (f) Tactic-absent signals

- **Non-backward-compatible migrations**: a column renamed or dropped in the same release that stops writing it.
  The fix is **expand/contract** (add new, write both and backfill, switch reads, stop writing old, drop old in a
  later release; pattern in [architectural-patterns.md](architectural-patterns.md)). *Detect it*: in a PR that also
  changes the code using a column, grep the migration files for `DROP COLUMN`, `RENAME COLUMN`,
  `ALTER COLUMN ... TYPE`, and `NOT NULL` added without a default. For Postgres, a migration linter such as squawk
  (`squawk <migration>.sql`) automates this; its rule names change between versions. The runtime check (previous
  release's tests against the new schema) is in [fitness-functions.md](fitness-functions.md) §3.
- **Config baked into images**: environment URLs, credentials or flags in the Dockerfile, `application.yml` inside
  the jar, or constants in source. One artifact should be promotable through all environments with config injected
  at deploy or start time.
- **Manual deploy steps**: README sections like "then SSH in and run", scripts that only work on one laptop.
- **Lockstep releases**: services that must be deployed together because they share a database schema, a library
  version pinned across services, or a synchronous API changed incompatibly. This is a distributed monolith.
- **Missing health endpoints**: no readiness/liveness endpoint (e.g. Kubernetes `readinessProbe`/`livenessProbe`,
  Spring Boot Actuator `/actuator/health`), so the platform cannot tell a bad rollout from a good one.
- **Mobile and desktop clients**: shipped binaries cannot be rolled back quickly. Look for a server-driven
  minimum-supported-version check, remote kill switches, and APIs that tolerate old clients for months.

### 3.5 (g) Measurement

- **The four DORA metrics**, taken from the pipeline and incident tracker, not from surveys: deployment frequency,
  lead time for changes (commit to production), change failure rate, and time to restore service (recent DORA
  reports call it failed deployment recovery time).
- **Manual steps per release**: count them; the target is zero.
- **Rollback time**: measured in a drill, not estimated; compare with the scenario threshold (e.g. ≤ 5 min).

### 3.6 (h) Tradeoffs

Independent deployment raises operational cost (pipelines, observability, versioning discipline). Coexisting
versions raise testing load (every supported version pair). Feature toggles are deferred binding: they add paths to
test and become debt if never removed. Blue/green doubles capacity during the switch.

### 3.7 (i) Patterns

Microservice architecture (independent deployment is its main benefit); **complete replacement**: blue/green (two
full environments, switch traffic at once) and rolling upgrade (replace instances in batches; in Kubernetes a
`RollingUpdate` strategy tuned with `maxSurge`/`maxUnavailable`); **partial replacement**: canary (small share of
real traffic, then widen) and A/B testing (variants for an experiment). Rollback and feature toggle are tactics;
canary and blue/green are patterns that realise them. Choose by this table:

| Pick | When | Requires |
|---|---|---|
| Rolling upgrade | Default for stateless replicated services (e.g. a Kubernetes Deployment) | Old and new versions serve side by side, so N/N-1 API and schema compatibility is mandatory |
| Blue/green | You need an instant switch and instant rollback, and can pay for double capacity for a while | The shared database schema must serve both versions |
| Canary | Per-version health metrics exist and traffic is high enough to show a signal in minutes | Automated promote/abort on those metrics; N/N-1 compatibility |
| A/B testing | Product experiments, not release safety | Sticky user assignment and analytics; N/N-1 compatibility |
| Staged store rollout + server-side minimum-version check + kill switch | Mobile and desktop binaries, which cannot be rolled back | APIs that tolerate old clients for as long as they are supported |
| None of the above is safe | The release includes a migration that is not expand/contract | Split the migration first (§3.4) |

### 3.8 (j) Reviewer quick checks

- Can this change be deployed without deploying anything else at the same time?
- Can the previous version still run against the database and APIs after this change is live?
- Is anything environment-specific baked into the build artifact?
- Is there a health signal and an automated rollback trigger for this component?
- Does every new feature flag have an owner and a removal plan?

---

## 4. Integrability — SAiP 4e ch. 7 (replaces 3e ch. 6 Interoperability)

### 4.1 (a) Definition

Integrability is the cost and risk of making separately developed components work together as intended: adding a
component, integrating a new version of one, or combining existing ones in a new way. It broadens 3e
**interoperability** (the degree to which two or more *systems* can usefully exchange meaningful information through
interfaces, both **syntactically**, where the formats and protocols line up, and **semantically**, where both sides
mean the same thing) in three ways: it covers components inside one system as well as external systems; it is
judged at design, integration and deployment time as well as runtime; and it measures integration *cost*, not only
whether exchange succeeds. 3e's two interoperability concerns remain useful: **discovery** (how the consumer finds
the service's location, identity and interface) and **handling of the response** (reply, broadcast, or forward).

SAiP 4e (the Integrability chapter) frames integration cost as the *distance* between two interfaces along
syntactic, data-semantic, behavioural-semantic, temporal and resource dimensions [verify wording in SAiP 4th ed.:
dimension names and count]. Use it as a checklist: two components
that agree on JSON but disagree on units, ordering guarantees or rate limits are still far apart.

### 4.2 (b) General scenario and (c) concrete scenario [verify wording in SAiP 4th ed.]

| Part | General values | Example |
|---|---|---|
| Source | Mission or system stakeholders, component vendors, component marketplaces | Product decision |
| Stimulus | Add a component; integrate a new version; integrate existing components in a new way | Replace the email vendor |
| Artifact | Whole system, a set of components, component metadata or configuration | `notifications` adapter |
| Environment | Development, integration, deployment or runtime | Development time |
| Response | Changes completed, integrated, tested and deployed | New vendor behind the existing `EmailSender` port |
| Response measure | Components changed, % code changed, effort, money, calendar time, effect on other QAs | Only the adapter module and DI wiring change; ≤ 2 person-days; contract suite green on both adapters |

### 4.3 (d) Tactic tree and (e) code-level realization

4e groups [verify wording in SAiP 4th ed.]: **limit dependencies**, **adapt**, **coordinate**. 3e interoperability had only **locate**
(*discover service*) and **manage interfaces** (*orchestrate*, *tailor interface*); in 4e these survive as
discover, tailor interface and orchestrate.

| Group | Tactic | What it looks like in code |
|---|---|---|
| Limit dependencies | Encapsulate | Vendor types never leave the adapter; the rest of the code sees your own types |
| | Use an intermediary | Message broker, API gateway, anti-corruption layer (ACL) between your model and a foreign one |
| | Restrict communication paths | Only one module talks to the external system; enforce with dependency rules |
| | Adhere to standards | OpenAPI, protobuf/gRPC, AsyncAPI, OAuth 2.0/OpenID Connect, CloudEvents, ISO 8601 times, ISO 4217 currency codes |
| | Abstract common services | One port for a family of providers (storage, payments, notifications) |
| Adapt | Discover (3e: discover service) | Service registry or DNS-based discovery; capability negotiation at runtime |
| | Tailor interface | Adapter/wrapper that adds (translation, buffering, smoothing) or removes (hiding operations from untrusted callers) capabilities |
| | Configure behaviour | Choose protocol version, format or endpoint by configuration instead of code |
| Coordinate | Orchestrate | A workflow or saga coordinator sequences calls so the services need not know each other |
| | Manage resources | Rate limits, quotas, connection pools and back-pressure agreed across the integration |

**Idioms.**

- **Anti-corruption layer** (a Domain-Driven Design term): a translation layer that converts a foreign model into
  yours at the boundary, so vendor concepts and naming do not leak into the domain.
- **Adapters around vendor SDKs**: wrap each SDK behind a port you own, map its exceptions to your error types, and
  keep its DTOs inside the adapter. The contract-test idea from §2.3 applies: one suite, run against each adapter.
- **Versioned contracts**: version the API or schema explicitly; add fields, never change their meaning; read
  tolerantly (ignore unknown fields).
- **Schema registries** for event streams enforce a compatibility mode per subject (Confluent Schema Registry uses
  modes such as `BACKWARD`, `FORWARD`, `FULL`); choose the mode by who upgrades first, producer or consumer.
- **Breaking-change checks in CI**: for protobuf, `buf breaking --against '.git#branch=main'` (buf-specific; the
  rule set is configured in `buf.yaml`, check your buf version's docs). For OpenAPI, a diff tool such as oasdiff
  (`oasdiff breaking base.yaml revision.yaml`; flags and output vary by version). Protobuf rules of thumb: never
  reuse or renumber a field number, mark removed ones `reserved`.

### 4.4 (f) Tactic-absent signals

- Vendor SDK types in domain signatures, entities or database schemas (`Stripe.Charge`, `firebase.*`, `boto3`
  responses passed through use cases).
- The same external system called from several modules, each with its own client and retry logic.
- Unit and meaning mismatches: amounts as floats, satoshis vs coins, milliseconds vs seconds, local time without
  zone. The bytes parse, the meaning is wrong: the classic semantic interoperability failure.
- API or message schema changes with no version bump and no breaking-change check in CI.
- Generated clients checked in without the spec they came from.

### 4.5 (g) Measurement

- **Files and modules touched per vendor swap or version upgrade**: `git diff --stat` on that PR; compare with the
  scenario (e.g. only the adapter module and DI wiring).
- **Modules importing each vendor SDK**: grep the import count per module; the target is exactly one (the adapter).
- **Breaking-change check present in CI** for every published contract (`buf breaking`, oasdiff, API/ABI validators;
  [fitness-functions.md](fitness-functions.md) §3): yes or no.
- **Contract suite green against every adapter** of a port, fake included.

### 4.6 (h) Tradeoffs

Intermediaries and orchestrators add latency and can be single points of failure (performance, availability).
Tailoring interfaces for untrusted callers helps security; translation layers add maintenance. Standards help
semantic agreement but constrain how freely the data model can change (modifiability).

### 4.7 (i) Patterns

Adapter/wrapper, bridge, mediator, service-oriented architecture, dynamic discovery [verify wording in SAiP 4th
ed.]; broker; gateway; ACL; orchestration vs choreography. See [architectural-patterns.md](architectural-patterns.md).

Orchestration vs choreography: **orchestrate** (a central saga or workflow coordinator, e.g. Temporal or AWS Step
Functions) when the flow has compensations or deadlines, or must be visible and changeable in one place.
**Choreograph** (services react to each other's events) when the steps are independent, owned by different teams and
only loosely ordered. Choreography gone wrong: a business flow that can only be reconstructed by reading event
handlers in four or more services.

### 4.8 (j) Reviewer quick checks

- Is every external system reached through exactly one adapter the team owns?
- Do vendor types stop at the adapter?
- Is the contract versioned and checked for breaking changes in CI?
- Are units, time zones, currencies and precision explicit in the contract?
- Does a contract test run against the real integration, or a recorded copy of it?

---

## 5. Other quality attributes — SAiP 4e ch. 14 (3e ch. 12)

4e ch. 14 is titled *Working with Other Quality Attributes*. 3e ch. 12 listed QAs without their own chapter:
variability, portability, development distributability, scalability, deployability, mobility, monitorability and
safety. Deployability and safety have their own 4e chapters (5 and 10); mobility is covered in 4e ch. 18 *Mobile
Systems*. 4e ch. 14 also discusses qualities of the architecture itself (such as conceptual integrity and
buildability), business qualities, and standard quality models [verify wording in SAiP 4th ed.].

### 5.1 ISO/IEC 25010

ISO/IEC 25010:2011 (SQuaRE series) defines a product quality model with eight characteristics: functional
suitability, performance efficiency, compatibility (includes interoperability), usability, reliability (includes
availability), security, maintainability (includes modularity, modifiability and testability) and portability.
It names qualities but gives no scenarios or tactics, so write scenarios anyway. A 2023 revision changes the model,
for example adding safety and renaming some characteristics [verify details before citing]. Use 25010 as a
completeness checklist when eliciting QAs, not as the requirement itself.

### 5.2 How to specify a new QA

3e ch. 12 ("Dealing with 'X-ability'") gives five steps; 4e ch. 14 covers the same ground [verify wording in SAiP
4th ed.]:

1. **Capture scenarios**: write a general scenario from stakeholder concerns, then concrete ones.
2. **Assemble design approaches**: collect the patterns and techniques known to affect the QA.
3. **Model** the QA where you can (queueing models for performance, Markov models for availability).
4. **Assemble tactics** from the approaches and the model's parameters.
5. **Build design checklists** across the seven design-decision categories: allocation of responsibilities,
   coordination model, data model, management of resources, mapping among architectural elements, binding time,
   choice of technology.

In practice, write a new card with slots (a)-(j) from "Card layout" above, and make its (g) Measurement a fitness
function for the response measure.

### 5.3 Short cards

**Portability.** *Concern*: cost of moving software to another platform. A special case of modifiability.
*Tactics*: abstract common services into a **portability layer**; defer binding to build time; encapsulate platform
APIs. *Code*: KMP `expect`/`actual` with a small, enumerable platform surface; Go build tags per OS; a `platform/`
package in TS for Node vs browser. *Signals*: platform checks (`if (Platform.OS == ...)`, `runtime.GOOS`,
`sys.platform`) scattered through shared code; platform types in shared signatures. *Measure*: count of
platform-specific declarations or files; effort to bring up a new target.

**Scalability.** *Concern*: handling more load by adding resources. **Horizontal** (scale out) adds instances;
**vertical** (scale up) adds CPU or memory to one instance; **elasticity** adds and removes instances on demand.
*Tactics*: stateless services, partitioning/sharding, replication and caching, asynchronous work queues.
*Signals*: in-process session state or caches that assume one instance; scheduled jobs that run on every replica;
global locks; database as the only shared bottleneck; auto-increment IDs across shards. *Measure*: response time
and cost per unit of capacity as load grows (e.g. p95 < 300 ms from 1k to 20k requests/s with linear cost). Runtime
performance tactics are in [quality-attributes-runtime.md](quality-attributes-runtime.md).

**Observability.** SAiP 3e calls the related QA *monitorability*: how well operators can observe the system while
it runs. "Observability" is the industry term (logs, metrics, traces). It overlaps with availability's detection
tactics and with testability's *observe state*. *Code*: structured logs with correlation IDs, metrics at every port,
trace context propagated across boundaries (OpenTelemetry is the common standard), health endpoints. *Signals*:
`println`/`console.log` debugging, swallowed exceptions, no request ID across service hops. *Measure*: time to
detect and localise a fault (e.g. from alert to faulty component in ≤ 10 min).

**Variability.** *Concern*: producing a set of variants that differ in known ways (product lines, white-label
apps, editions, tenants). A special case of modifiability. *Tactics*: explicit **variation points** plus deferred
binding (§1.3 table). *Signals*: long-lived forks or branches per customer; `if (customer == "acme")` in code.
*Measure*: effort to add a variant; number of files touched per variant.

**Development distributability.** *Concern*: how well the software supports development by distributed teams.
Coordination cost follows module dependencies (Conway's law: system structure mirrors the communication structure
of the organisation). *Tactics*: align module boundaries with team ownership, low coupling between work units,
stable published interfaces between teams. *Signals*: one module with many owners in `CODEOWNERS`, PRs routinely
needing approval from three or more teams, change coupling across team-owned modules. *Measure*: cross-team PRs as a
share of all PRs; lead time for changes that span teams.

---

## 6. Where the change-time QAs meet

- **Modifiability ↔ integrability**: *limit dependencies* and *reduce coupling* share tactics (encapsulate, use an
  intermediary, restrict dependencies/communication paths, abstract common services). Justify them in an ADR by the
  scenario they serve, since the same code decision can be argued from either QA.
- **Modifiability ↔ deployability**: feature toggles and config are *defer binding*; independent deployment needs
  low coupling at the data and API level, not only in code.
- **Testability ↔ modifiability**: ports that make code testable also make it replaceable; a missing fake usually
  means a missing seam.
- **Deployability ↔ integrability**: versioned contracts are what let versions coexist during a rollout.

## 7. Cross-QA tradeoff matrix (covers runtime and change-time QAs)

Name the tradeoff in every design proposal and every review finding. In ATAM terms, a decision that is a
sensitivity point for more than one QA, typically helping one and hurting another, is a **tradeoff point**
(definition in [evaluation-methods.md](evaluation-methods.md)).

| Decision (QA it serves) | Hurts | Why | Mitigation |
|---|---|---|---|
| Use an intermediary: bus, broker, gateway, facade (modifiability, integrability) | Performance (latency), availability | Extra hop, serialisation, a new component that can fail | Keep intermediaries off the hot path; co-locate or use in-process buses; measure p95 per hop; make the intermediary redundant |
| Redundancy: replicas, spares, multi-region (availability) | Consistency, cost, energy | Replicas diverge; hardware and power multiply; more attack surface | Choose the consistency model explicitly per data type; warm rather than hot spares where recovery time allows; scale spares with demand |
| Encryption and authentication on every hop (security) | Performance, energy, usability | CPU, handshakes, battery; login friction | Terminate TLS at the right boundary; reuse connections and sessions; hardware acceleration; token refresh instead of repeated login |
| Caching and multiple copies of data (performance) | Consistency, modifiability, security | Stale reads; invalidation logic spread through code; cached secrets or per-user data leaking across users | Define TTLs and invalidation owners; cache behind one port; key caches by tenant/user; never cache credentials |
| Microservices, independent deployment (deployability, modifiability) | Testability, operational cost, performance | Distributed failures, harder end-to-end tests, network calls replace function calls, many pipelines | Start as a modular monolith with enforced boundaries; consumer-driven contract tests; shared platform tooling; split only where a deployability or scaling scenario demands it |
| Abstraction layers, ports, generic services (modifiability, portability, testability) | Performance, analysability | Indirection, lost specialised APIs, harder to follow | Abstract only at scenario-backed seams; allow a documented fast path; benchmark the abstraction on the hot path |
| Reduce computational overhead: inline, remove layers (performance) | Modifiability | Removes intermediaries, so coupling rises | Confine it to the measured hot path and fence it with an architecture test |
| Introduce concurrency (performance) | Testability | Nondeterminism | Inject dispatchers; deterministic schedulers in tests; immutable messages between workers |
| Defer binding: flags, plug-ins, runtime config (modifiability, deployability) | Testability, availability, performance | More configurations to test; runtime lookup and late failures | Validate config at startup (fail fast); test the supported flag combinations only; expire flags |
| Specialized test interfaces (testability) | Security | A back door if shipped | Keep them in test source sets or debug builds; a build check that fails if they reach release |
| Undo, history, rich feedback (usability) | Performance, complexity | State history and extra work per action | Bound history length; compute feedback asynchronously |

## 8. Review routine for change-time QAs

1. Find the stated change-time goals (ADRs, README, issue text). If none are stated, write one concrete scenario
   per QA from the change history (what actually changed in the last 6–12 months) and confirm it with the user.
2. For each goal, check which tactics from its card are present in the code and which are missing.
3. Collect signals from the cards with file:line evidence; rank by the scenarios they threaten, not by count.
4. For each fix, name the tactic, the QA it improves and the QA it costs (§7), then propose a fitness function to
   keep it fixed ([fitness-functions.md](fitness-functions.md)).
5. Report in [../templates/architecture-review-report.md](../templates/architecture-review-report.md) following
   [review-playbook.md](review-playbook.md).

---

*Sources: Bass, Clements & Kazman, Software Architecture in Practice, 4th ed. (2021) and 3rd ed. (2013),
Addison-Wesley; ISO/IEC 25010:2011. Parts adapted and paraphrased from the Wikipendium TDT4240 compendium,
https://www.wikipendium.no/TDT4240_Software_Architecture (CC BY-SA 3.0); the port-with-fake idiom paraphrases the
architecture description in https://gitlab.com/wallywallet/wallet/-/merge_requests/853. Contributors and licence
details: [../CREDITS.md](../CREDITS.md).*
