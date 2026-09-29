# Design workflow: ASRs, scenarios, utility trees, ADD, tradeoff reasoning, vocabulary, debt

Use when designing or extending a system: turn vague goals into measurable QA scenarios, prioritise them, run
Attribute-Driven Design (ADD) iterations in a real repository, choose options with explicit tradeoffs, and
decide what *not* to build.

Sources: Bass, Clements & Kazman, *Software Architecture in Practice* (SAiP), 4th ed., 2021 (chapter numbers are
4th-ed.); ADD 3.0 also follows Cervantes & Kazman, *Designing Software Architectures: A Practical Approach*
(1st ed. 2016; 2nd ed. 2024).

Related files (do not duplicate their content here):
- Per-QA general scenarios and tactic catalogues: [quality-attributes-runtime.md](quality-attributes-runtime.md),
  [quality-attributes-change.md](quality-attributes-change.md)
- Pattern catalogue and selection: [architectural-patterns.md](architectural-patterns.md),
  [design-patterns-in-code.md](design-patterns-in-code.md)
- Turning a design into packages and interfaces: [module-layout-and-interfaces.md](module-layout-and-interfaces.md)
- ATAM, sensitivity/tradeoff/risk definitions in depth: [evaluation-methods.md](evaluation-methods.md)
- Enforcing decisions in CI: [fitness-functions.md](fitness-functions.md)
- Recovering the as-is structure, hotspots, co-change: [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md)
- Views, arc42, C4, 42010: [documentation.md](documentation.md); ADR form: [../templates/adr.md](../templates/adr.md)

Theory adapted in part from the Wikipendium TDT4240 compendium (CC BY-SA 3.0); see [../CREDITS.md](../CREDITS.md).

---

## 1. Core vocabulary (SAiP ch. 1-2)

Use these terms precisely in briefs, ADRs and review findings.

**Software architecture (SAiP).** The set of structures needed to reason about a system, comprising software
elements, relations among them, and properties of both. Consequences to apply:
- Every system has an architecture, documented or not. In a review, recover it; do not assume the README is it.
- It is an abstraction with several structures; a clean package tree says nothing about runtime failure behaviour.
- "Good" is only meaningful relative to stated quality goals. Never judge a design without them.
- All architecture is design; not all design is architecture. Architectural decisions are the ones whose
  consequences reach across elements or affect system-wide qualities.

**Element, relation, property.** Element = a part in a structure (module, component, node, team); relation =
how two connect (uses, calls, allocated-to); property = an attribute that matters (latency budget, owner, trust zone).

**Module vs component.** A *module* is a static implementation unit (package, library, Gradle/npm/Go module).
A *component* is a runtime element (process, service, actor, thread pool) interacting via *connectors*
(call, queue, pipe, pub-sub bus, protocol). One module can yield many runtime components and vice versa.

**Structure vs view vs viewpoint.** A *structure* exists in the code or deployment. A *view* is a documented
representation of one or more structures for a set of stakeholder concerns. A *viewpoint* (ISO/IEC/IEEE 42010)
is the convention for building a kind of view: concerns, stakeholders, notation, analysis techniques.
Structures are recovered from code; views are written; viewpoints are chosen.

### Useful structures and the question each answers

| Category | Structure | Elements / relation | Question it answers | Recover it from |
|---|---|---|---|---|
| Module | Decomposition | module / is-submodule-of | Where does responsibility X live? What can one team own? | directory tree, build modules |
| Module | Uses | module / uses (correctness depends on) | What breaks or must be retested if Y changes? Can we ship a subset? | import graph, build deps |
| Module | Layers | layer / allowed-to-use (downward) | Which dependencies are legal? What is portable? | package rules, lint config |
| Module | Class (generalisation) | class / is-a, instance-of | Where are the extension points? Is inheritance healthy? | type hierarchy |
| Module | Data model | entity / relationship, cardinality | What is the source of truth? Where are consistency boundaries? | schema, migrations, ORM models |
| C&C | Service | service / connector (protocol) | What talks to what at runtime, synchronously or not? What fails together? | API specs, clients, traces |
| C&C | Concurrency | logical thread / shared resource, sync | Where can races, contention or deadlock occur? | executors, coroutines, locks, goroutines |
| Allocation | Deployment | component / allocated-to node, network | What runs where? Blast radius? Latency between parts? Trust zones? | IaC, manifests, Dockerfiles |
| Allocation | Implementation | module / mapped-to file, build artefact | How is it built and released? What does a change rebuild? | build scripts, CI config |
| Allocation | Work assignment | module / assigned-to team | Who must coordinate for a change? Does ownership match boundaries? | CODEOWNERS, git blame |

**Tactic vs pattern.** A *tactic* is a single design decision that influences the response of one quality
attribute (heartbeat, encapsulate, introduce concurrency, authenticate actors). A *pattern* is a packaged
{context, problem, solution} that bundles several tactics and trades several QAs at once (layered, broker,
publish-subscribe, CQRS). Name both in designs: "Publish-subscribe, realising *use an intermediary* and
*defer binding*; costs latency and end-to-end traceability."

**ASR (architecturally significant requirement).** A requirement that profoundly affects the architecture:
without it, the architecture would likely be different (SAiP ch. 19). Section 3.

**Sensitivity point, tradeoff point, risk, non-risk.** A sensitivity point is a decision/parameter critical to
one QA response; a tradeoff point is a sensitivity point for two or more QAs; a risk is a potentially
problematic decision (or a missing one) given the QA goals; a non-risk is a sound decision resting on a stated
assumption. Full definitions and examples: [evaluation-methods.md](evaluation-methods.md).

### Common confusions (correct them in reviews and ADRs)

| Confusion | Correct distinction |
|---|---|
| Layer vs tier | Layer = module grouping with a downward allowed-to-use relation (static). Tier = runtime/deployment grouping. Three layers can run in one process; three tiers are three deployables. |
| Constraint vs decision | A constraint is given (regulation, mandated platform, existing contract). A decision is chosen and needs rationale. Never defend a choice by calling it a constraint. |
| Requirement vs design in ASR lists | "Use Kafka" is a decision. "Order events reach billing within 5 s at 2k events/s" is the requirement it answers. |
| Tactic vs pattern | "MVVM" is a pattern; "encapsulate", "restrict dependencies" are tactics it realises. |
| Observer vs publish-subscribe | Observer: subject holds references to observers. Pub-sub: publishers and subscribers are anonymous to each other via a bus/broker. |
| Caching as "copies of computations" | Caching is *maintain multiple copies of data*; replicated servers behind a load balancer are *maintain multiple copies of computations*. |
| Performance vs usability | "Screen renders in 1 s" is performance; "user completes checkout in 3 steps without help" is usability. |

---

## 2. Why architecture matters (SAiP ch. 2)

Use these one-liners to justify architectural work (ADR, refactor, fitness function); cite the relevant one only.

1. It enables or inhibits the driving quality attributes; features can be bolted on, qualities mostly cannot.
2. It lets you reason about change: which changes are local, non-local, or architectural.
3. It lets you predict qualities (latency, availability) before building and load-testing everything.
4. It gives stakeholders a shared vocabulary and a place to negotiate conflicting concerns.
5. It holds the earliest and most expensive-to-reverse decisions.
6. It constrains implementation, so many developers can work without re-deciding the fundamentals.
7. It shapes (and is shaped by) team structure; see Conway's law in section 14.
8. It enables incremental development: a walking skeleton can be grown feature by feature.
9. It is the basis for cost and schedule estimates.
10. It can be a reusable model across a product family (shared core assets, variation points).
11. It lets you integrate independently developed elements (libraries, SaaS, frameworks) instead of building.
12. It restricts the vocabulary of design alternatives, which reduces accidental complexity.
13. It is the natural starting point for onboarding.

### Contexts and the architecture influence cycle (SAiP 3rd ed. ch. 3; spread over 4th ed. ch. 1-2, 24-25)

Architecture lives in four contexts. Use each one as a question during the ASR hunt (section 3), before
reading only the code:

| Context | Question it answers for the ASR hunt | Where to look |
|---|---|---|
| Technical | Which quality attributes must the structure deliver, on which platform and technology constraints? | Runtime/infra config, SLOs, framework and platform choices, integration points |
| Project life cycle | How is the system built, delivered and evolved (iterative, release cadence, who deploys)? | CI/CD pipelines, branching model, release notes, deployment manifests |
| Business | Which business goals, market, cost or time-to-market pressures and org structure drive it? | README, product docs, team/ownership files (CODEOWNERS), the user |
| Professional | What skills, experience and preferences does the architect/team bring, and what do they lack? | Team size and stack history, existing conventions, the user |

**Architecture influence cycle.** Business goals, the technical environment and the architect's (and team's)
experience shape the architecture; the built system then feeds back into all three: it changes what the
business can offer next, becomes part of the technical environment for later systems, and changes what the
architect and organisation know and prefer. Use this to justify asking about organisation and business
drivers in step 1 of the design procedure: a structure that fits the code but not the business goals or
the team that must run it is a wrong structure, and the chosen structure will constrain that business and
team afterwards.

---

## 3. Architecturally significant requirements (SAiP ch. 19)

### Three kinds of requirement

| Kind | Says | Example (B2B invoicing SaaS) | Architectural freedom |
|---|---|---|---|
| Functional | What the system does | "Customers can export invoices as PDF" | High; many structures deliver it |
| Quality attribute | How well, under what conditions | "Export of 10k invoices completes within 60 s at p95" | Drives tactic and pattern choice |
| Constraint | A decision already made | "EU data residency", "must run on the customer's Kubernetes", "Postgres is the company standard" | None; record the consequences |

Functionality rarely determines structure; QAs and constraints do. Record every constraint with its
consequence (`# | Constraint | Consequence`), e.g. "Must deploy as a single container -> no sidecars; in-process
circuit breaker instead of a service mesh."

### Where ASRs hide

Requirements documents rarely state QAs well. Mine these sources, in roughly this order of yield:

| Source | What to look for |
|---|---|
| SLOs, SLAs, on-call runbooks | Availability and latency targets, error budgets, RPO/RTO |
| Postmortems, incident tickets | Recurring failure modes = availability, performance or deployability ASRs |
| Backlog, roadmap, sales commitments | The most likely change in the next 12 months = modifiability/integrability ASR |
| Compliance (GDPR, PCI DSS, HIPAA, SOC 2, ISO 26262/IEC 62304 in safety domains) | Mostly constraints plus security/safety ASRs |
| Team topology | Number of teams, time zones, ownership; drives decomposition and interface stability |
| Deploy target | Mobile store review cycle, embedded OTA, serverless limits, on-prem customers |
| Cost ceilings | Cloud budget per tenant/request; energy or battery budgets |
| The code itself | See signals table below |

**Code signals of latent ASRs** (the code shows what hurt, even when nobody wrote it down):

| Signal in code | Probable ASR / pain |
|---|---|
| Hand-rolled retry loops, backoff, circuit breakers, `catch { /* ignore */ }` near I/O | Availability against an unreliable dependency |
| Caches, memoisation, denormalised read tables, `@Cacheable`, Redis wrappers | Performance (latency or load) |
| Feature flags, config toggles, canary/blue-green scripts | Deployability, release-risk control |
| Many adapters/mappers for external systems, anti-corruption layers | Integrability with changing third parties |
| Plugin registries, SPI/`ServiceLoader`, strategy maps keyed by tenant | Modifiability/variability per customer |
| Fakes, test doubles at every port, contract tests | Testability is valued; keep ports stable |
| Audit logs, idempotency keys, outbox tables | Security/accountability, exactly-once-ish business semantics |
| Offline queues, sync conflict resolution, local DB on device | Availability under intermittent connectivity |
| Wake-lock handling, batching of network calls, WorkManager/BGTaskScheduler constraints | Energy efficiency on mobile |

Rule: **ASRs are requirements, not design decisions.** If a draft ASR names a pattern, framework or class,
rewrite it as the measurable outcome it was meant to achieve, unless it is a genuine constraint.

---

## 4. Question bank and assumption protocol

0. **Answer from the repository first.** For each candidate question below, look in the section 3 sources and
   the code-signals table (SLO files, Helm/Terraform values, runbooks, postmortems, config, load-test scripts).
   Record each answer found as `CONFIRMED (source: <file path>)`.
1. Ask the user **at most 3-5 questions**: the highest-impact ones the repo could not answer, i.e. those whose
   answers would change the design. Everything else becomes a flagged assumption.
2. If no user is available (non-interactive or subagent run), do not ask; apply the assumption protocol below,
   and still output the 3-5 questions so a human can answer them later:

   | ID | Question | Why it changes the structure | Default assumed |
   |---|---|---|---|
   | Q1 | Can the host run a non-HTTP always-on process? | Decides worker process vs in-app thread vs cron | ASSUMED: no; in-app worker started in lifespan |

   Reference the question IDs from the scenarios and ADRs that depend on them ("QS-A1 (depends on Q1)"), so a
   changed answer shows exactly what to revisit.

| QA | Questions that change the design |
|---|---|
| Performance / scalability | Expected load now and in 12 months (req/s, events/s, data volume)? Latency budget at p95/p99 for the critical path? Burst shape? |
| Availability | Acceptable downtime per month? Which operations must keep working when dependency X is down? RPO (data loss) and RTO (recovery time)? |
| Modifiability | What is the most likely change in the next 12 months? Which parts change weekly vs yearly? |
| Integrability | Who integrates with us, and who do we integrate with? Do we control their release cadence? Versioning expectations? |
| Security | Who are the threat actors (anonymous internet, malicious tenant, insider, compromised device)? What data is sensitive? Compliance regime? |
| Deployability | Deploy cadence? Zero-downtime required? Rollback time? Mobile store or OTA constraints? Can the host run a non-HTTP always-on process (worker), cron, or does it scale to zero? (Read `deployment*.md`, `Procfile`, `fly.toml`, `app.yaml`, `vercel.json` first.) |
| Energy / mobile / IoT | Target devices, battery or power budget, connectivity profile (offline, metered, flaky)? |
| Safety | Can an output harm people, property or money irreversibly? What is the safe state? |
| Testability | Can production-like dependencies run in CI? What must be deterministic? |
| Usability | Who are the users, and what task must be fast or error-proof? |

**Assumption protocol** when answers are unavailable:
1. Write the scenario anyway with plausible numbers and mark it `ASSUMED` (e.g. `ASSUMED: 200 req/s peak`);
   change it to `CONFIRMED` once a stakeholder agrees the number or a repo artifact states it. These are the only
   two status values.
2. List every `ASSUMED` item in the design brief and in the ADR's context section.
3. Prefer designs that stay correct if the assumption is wrong by 10x in either direction, or that are cheap to
   reverse (section 11).
4. Name the observable that would falsify the assumption (a metric, a load test, a product decision) so it can be
   revisited.

---

## 5. Six-part quality-attribute scenarios

| Part | Question | Common mistake |
|---|---|---|
| Source | Who or what generates the stimulus? | Omitted, or "the system" |
| Stimulus | What event or condition arrives? | Two stimuli in one scenario |
| Artifact | Which part of the system is stimulated? | "The app" instead of a named element |
| Environment | Mode or state when it arrives (normal, peak, degraded, startup, offline)? | Left empty |
| Response | What the system does | Writes the tactic ("uses a cache") instead of observable behaviour |
| Response measure | How the response is judged | Unmeasurable ("fast", "robust") |

A **general scenario** is system-independent and lists allowed values per part (catalogued per QA in
[quality-attributes-runtime.md](quality-attributes-runtime.md) and [quality-attributes-change.md](quality-attributes-change.md)).
A **concrete scenario** picks one value per part for this system. Design and review work only with concrete ones.

**ID convention (use it in briefs, utility trees, ADRs, matrices and test names):**
- Scenarios: `QS-<QA letter><n>`, e.g. `QS-P1`. Letters: A availability, D deployability, E energy, I
  integrability, M modifiability, O operability/observability, P performance, S security, F safety,
  T testability, U usability.
- Ranked documentation quality goals (section 7): `QG-<n>`, where n is the priority rank.
- Status: `CONFIRMED` or `ASSUMED` (section 4).
- Counts: a design brief carries up to 7 scenarios; an architecture description ranks the top 3-5 as QGs.

### Bad-to-good rewrites

| Bad | Good (compressed six-part) |
|---|---|
| "The API must be fast." | Clients (source) send search requests (stimulus) to the catalogue API (artifact) at 500 req/s peak (environment); results returned (response) with p95 < 200 ms, p99 < 500 ms (measure). |
| "The system must be highly available." | Infrastructure fault (source): the primary Postgres node of the orders database (artifact) crashes (stimulus) during business-hours peak (environment); the orders service fails over to the synchronous replica and resumes writes (response); writes resume within 60 s, 0 committed orders lost (measure). |
| "The code must be easy to change." | A developer (source) adds a new payment provider (stimulus) to the payments module (artifact) at design time (environment); change is confined to one new adapter plus config (response); <= 3 person-days, 0 changes outside `payments/adapters` (measure). |
| "Use Redis to be scalable." | Tactic, not a scenario. Ask what load and latency Redis was meant to achieve and write that. |

### Response measures: good vs bad

| QA | Bad | Good (unit + threshold + percentile/probability) |
|---|---|---|
| Performance | "quick" | p95 < 150 ms at 1,000 req/s; throughput >= 5k msg/s |
| Availability | "always up" | 99.9 % monthly (~43 min downtime); RTO 5 min; RPO 0 |
| Modifiability | "easy" | <= 2 modules touched; <= 1 person-day; 0 public API changes |
| Deployability | "CI/CD" | lead time commit->prod < 1 h; rollback < 5 min; 0 downtime |
| Security | "secure" | 100 % of cross-tenant reads denied and audited; detection < 15 min |
| Energy | "battery-friendly" | background sync <= 2 % battery/day on reference device |
| Testability | "testable" | domain module unit suite < 60 s with no network; 90 % branch coverage of pricing rules |
| Usability | "intuitive" | 90 % of first-time users complete onboarding in < 3 min without help |
| Safety | "safe" | 0 actuator commands outside limits; safe state entered within 100 ms of sensor loss |
| Integrability | "pluggable" | new webhook consumer integrated in < 2 days using only the public schema |

### One concrete scenario per QA (across system types)

| ID | QA | System | Source / stimulus | Artifact / environment | Response / measure |
|---|---|---|---|---|---|
| `QS-A1` | Availability | Web API (checkout) | Payment provider times out | Checkout service, peak sale | Order accepted as `PENDING`, retried async; 99.95 % of checkouts accepted, 0 double charges |
| `QS-D1` | Deployability | Web API | Developer merges a fix | CI/CD pipeline, business hours | Canary to 5 %, auto-promote or roll back; commit to 100 % prod < 45 min; rollback < 3 min |
| `QS-E1` | Energy efficiency | Mobile app | OS schedules background sync | Sync worker, battery < 20 % | Batch and defer non-urgent sync; <= 1 % battery per day in background |
| `QS-I1` | Integrability | Library (SDK) | Partner adopts a new major API version | SDK transport layer, design time | Only the version adapter changes; <= 1 new file, 0 changes to public SDK types |
| `QS-M1` | Modifiability | Data pipeline | Analyst requests a new derived metric | Transformation stage, design time | New transform registered as a stage; <= 1 day, no change to ingestion or storage stages |
| `QS-P1` | Performance | IoT gateway | 5,000 sensors report every 10 s | Ingestion service, normal operation | All readings persisted; end-to-end lag p99 < 2 s, 0 dropped |
| `QS-F1` | Safety | IoT (industrial) | Temperature sensor stops reporting | Heater controller, running | Controller enters safe state (heater off) within 500 ms; alarm raised |
| `QS-S1` | Security | Web API (multi-tenant) | Authenticated user of tenant A requests tenant B's invoice | Invoice API, normal | Request denied with 404, audited; 100 % of cross-tenant attempts blocked (verified by test suite) |
| `QS-T1` | Testability | Mobile app | Developer runs domain tests | Offline-sync domain module, CI | Runs on the JVM/host with no emulator; suite < 30 s; deterministic clock injected |
| `QS-U1` | Usability | Mobile app | New user makes a mistaken transfer | Payment confirmation screen, normal | Undo offered for 10 s; 95 % of mistaken transfers cancelled successfully in usability tests |

### Scenarios become tests

Each concrete scenario should map to something executable:
- Performance -> load test with the stated rate and percentile assertion (k6, Gatling, Locust, JMH for hot paths).
- Availability -> fault-injection or chaos test with the stated stimulus; failover drill measuring RTO/RPO.
- Modifiability, testability -> architecture fitness function (dependency rules, change-scope checks); see
  [fitness-functions.md](fitness-functions.md).
- Security -> negative authorisation tests per tenant boundary; dependency/secret scanning in CI.
- Deployability -> pipeline metrics (lead time, rollback drill).

For every QA, put the scenario ID (e.g. `QS-P1`) in the test or fitness-function name so a failure traces back
to the scenario.

---

## 6. Utility tree

A top-down prioritisation (also ATAM step 5; see [evaluation-methods.md](evaluation-methods.md)).

```
Utility
 └─ Quality attribute            (e.g. Availability)
     └─ Attribute refinement     (e.g. Provider outage)
         └─ Concrete scenario    (Importance, Difficulty)  each rated H / M / L
```

- **Importance**: business value to stakeholders. **Difficulty**: architectural impact/technical risk to achieve.
  State the rating order explicitly; sources vary.
- **(H,H)**: the design drivers. Address them in the first ADD iterations and in ADRs.
- **(H,M)/(H,L)**: must be met, usually with standard tactics; verify, do not over-design.
- **(M/L, H)**: challenge them (costly, low value; prime "deliberately not doing" candidates). **(L,L)**: note, move on.

### Worked tree: payments API (card and bank payments for merchants)

| QA | Refinement | Concrete scenario (compressed) | (I, D) |
|---|---|---|---|
| Security | Tenant isolation | Merchant A cannot read or refund merchant B's payments; 100 % denied and audited | (H, M) |
| Security | Secret handling | Card data never persisted or logged outside the tokenisation boundary (PCI DSS scope) | (H, H) |
| Availability | Provider outage | Acquirer unavailable 10 min: payments queued or routed to secondary; 0 lost, 0 duplicated | (H, H) |
| Performance | Authorisation latency | p99 authorisation < 800 ms end-to-end at 300 TPS peak | (H, M) |
| Integrability | New acquirer | Add an acquirer in <= 10 person-days touching only a new adapter | (M, M) |
| Modifiability | Fee rules | Finance changes a fee rule without a deploy (config, validated) within 1 h | (M, L) |
| Deployability | Zero-downtime | Schema and code changes deploy with 0 failed requests | (H, M) |
| Usability | Dashboard | Merchant finds a payment by reference in < 10 s | (L, L) |

Reading: the (H,H) leaves (card-data boundary, provider outage with exactly-once semantics) drive iteration 1:
tokenisation boundary as a separate deployable with minimal surface, idempotency keys, an outbox, and a
retry/route policy per acquirer.

---

## 7. Quality goals for documentation

For architecture descriptions (arc42 §1.2, the "Introduction and Goals" section of
[../templates/architecture-description.md](../templates/architecture-description.md)):
- List only the **top 3-5 quality goals**, each as a six-part scenario with IDs (`QG-1` ...), in **priority order**.
- State a **conflict rule**: when two goals conflict, the higher-ranked one wins unless an ADR records an
  explicit exception. This turns design arguments into lookups.
- Tie each goal to the tactics that realise it and to the code where they live (file paths), and to a fitness
  function or test where one exists.
- Keep the full utility tree in the quality-requirements section (goals are its top leaves).
- Separate **AS-IS** statements (true of the current code, citing files) from **TO-BE** proposals (*Proposed*).

---

## 8. QAW-lite and PALM

The SEI Quality Attribute Workshop (QAW) elicits QA scenarios before or independent of a design. When a user
wants requirements help, run a compressed, 30-minute version in conversation:

1. **Context (5 min):** business goals, users, key functions, known constraints.
2. **Current plan (5 min):** existing architecture or intended approach, external systems.
3. **Drivers (5 min):** draft a list of candidate ASRs; confirm with the user.
4. **Brainstorm (5 min):** propose 8-15 raw scenarios covering each driver, including growth and failure cases.
5. **Consolidate and prioritise (5 min):** merge near-duplicates; ask the user to pick the top 3-5 (or rate I/D).
6. **Refine (5 min):** expand the top scenarios to six-part form; mark `ASSUMED` values. Output feeds the utility tree and ADD.

**PALM** (Pedigreed Attribute eLicitation Method, SEI) elicits business goals, records their source and value
("pedigree"), and derives the QA requirements they imply; use its idea (trace each QA to a business goal)
even without the workshop.

---

## 9. Attribute-Driven Design (ADD 3.0; SAiP ch. 20, Cervantes & Kazman)

ADD is iterative. Inputs: design purpose, primary functional requirements, prioritised QA scenarios,
constraints, and architectural concerns (cross-cutting needs such as logging, error handling, configuration).

| Step | ADD 3.0 | What to do in a repository |
|---|---|---|
| 1 | Review inputs | Read README, ADRs, build files, CI, SLOs; gather/confirm scenarios (sections 3-6); list constraints |
| 2 | Establish iteration goal by selecting drivers | Pick 1-3 drivers, normally the (H,H) leaves; state the goal in one sentence |
| 3 | Choose elements to refine | Iteration 1: the whole system. Later: the module/service the drivers touch, found via the import graph |
| 4 | Choose design concepts that satisfy the drivers | Candidate patterns/tactics/components; compare at least two (section 11) |
| 5 | Instantiate elements, allocate responsibilities, define interfaces | Create/adjust packages and interface signatures; see [module-layout-and-interfaces.md](module-layout-and-interfaces.md) |
| 6 | Sketch views and record design decisions | Minimal C4/module sketch; one ADR per significant decision ([../templates/adr.md](../templates/adr.md)) |
| 7 | Analyse the design; review iteration goal and design purpose | Walk each driver scenario through the design; add fitness functions; list new risks as next drivers |

Repeat from step 2 until the drivers are addressed or the budget is spent.

### Iteration planning

| Situation | Iteration 1 | Following iterations |
|---|---|---|
| Greenfield | Establish overall structure: reference architecture, deployment units, main layers or bounded contexts, walking skeleton through the (H,H) path | Refine the elements behind each remaining driver; then cross-cutting concerns (observability, config, security plumbing) |
| Brownfield | Recover the as-is structure first ([architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md)); treat it as a constraint | Refine only the elements the new drivers touch; prefer strangler-style increments over rewrites |

### Sources of design concepts

| Concept | Examples | Where |
|---|---|---|
| Reference architectures | Web app with SPA + API, mobile app with offline sync, event-driven microservices, lambda/kappa pipelines | [platforms-and-domains.md](platforms-and-domains.md) |
| Architectural patterns | Layered, hexagonal, pipe-and-filter, broker, pub-sub, CQRS, microkernel | [architectural-patterns.md](architectural-patterns.md) |
| Design patterns | Strategy, Adapter, Observer, Repository | [design-patterns-in-code.md](design-patterns-in-code.md) |
| Tactics | Per-QA catalogues | [quality-attributes-runtime.md](quality-attributes-runtime.md), [quality-attributes-change.md](quality-attributes-change.md) |
| Deployment patterns | Blue-green, canary, sidecar, multi-region active-passive | [platforms-and-domains.md](platforms-and-domains.md) |
| Externally developed components | Frameworks, libraries, managed services, platforms | Checklist below |

### Library / framework / service selection checklist

- [ ] Which driver does it serve? If none, why add it?
- [ ] Lock-in: how much of our code will import it directly? Can it sit behind a port we own?
- [ ] Exit cost: what does replacing it take (data export, API differences)?
- [ ] Ecosystem health: release cadence, maintainers, open issues, security advisory history.
- [ ] Licence compatibility with our distribution model (copyleft, SSPL/BSL-style source-available, patents).
- [ ] Team skill and operability: can the team debug it at 3 a.m.? Is it already in the stack?
- [ ] QA fit: measured, not assumed, performance/footprint on our target (mobile binary size, cold start).
- [ ] Constraints it imposes: threading model, control inversion (framework owns the loop), transitive deps.
Record the outcome as an ADR when the answer to lock-in or exit cost is "high".

---

## 10. Seven categories of design decisions (SAiP 3rd ed. §4.6; check 4e placement) [verify]

Use as a checklist so no category is decided by accident. For each driver, ask which categories it touches.

| Category | Decide | Code-level evidence |
|---|---|---|
| Allocation of responsibilities | Which element owns which responsibility | Package ownership, service boundaries |
| Coordination model | How elements interact: sync/async, protocols, guarantees (ordering, delivery) | HTTP/gRPC clients, queues, event schemas |
| Data model | Entities, operations, ownership, consistency, lifecycle | Schemas, migrations, which service writes which table |
| Management of resources | Threads, memory, connections, quotas, battery; who arbitrates | Pool sizes, dispatchers, rate limiters, timeouts |
| Mapping among architectural elements | Module-to-runtime, runtime-to-hardware, data-to-store | Build modules -> deployables, manifests |
| Binding time | When a variation is fixed: compile, build, deploy, startup, runtime | Flags, config, DI wiring, plugin loading |
| Choice of technology | Languages, frameworks, stores, platforms | Build files, lockfiles, IaC |

---

## 11. Tradeoff reasoning

For every significant decision:
1. Generate **at least two** credible options (plus "do nothing / keep current" in brownfield).
2. Score them against the **ranked** scenarios, not against generic virtues.
3. Name the sensitivity and tradeoff points each option creates.
4. Check reversibility and timing.
5. Record the decision, rejected options and why, in an ADR.

### Worked mini decision matrix

Two scoring forms; pick by stakes:
- **Quick default** (the form in [../templates/architecture-description.md](../templates/architecture-description.md)
  §5): score each option per scenario as `+` satisfies, `0` neutral, `-` hurts, `?` unknown (becomes a risk or
  spike), plus a Cost/effort column. Use it for most briefs.
- **Weighted numeric matrix** (below): use it for contested or one-way-door decisions, where reviewers need to
  dispute individual cells.

Decision: how the orders service notifies billing and shipping. Drivers in priority order:
QS-A1 availability (orders accepted when billing is down), QS-M1 modifiability (new consumer without changing
orders), QS-P1 performance (order confirm p95 < 300 ms), QS-O1 operability (on-call traces one order end to end
in < 5 min). Score 1-3; weight by rank (4, 3, 2, 1).

| Option | QS-A1 (x4) | QS-M1 (x3) | QS-P1 (x2) | QS-O1 (x1) | Total | Notes |
|---|---|---|---|---|---|---|
| Synchronous HTTP calls from orders | 1 | 1 | 2 | 3 | 14 | Billing outage fails checkout; each new consumer edits orders; one synchronous trace |
| Transactional outbox + message broker | 3 | 3 | 3 | 1 | 28 | Adds broker operations, eventual consistency, duplicate delivery -> consumers must be idempotent; async hops break naive tracing |
| Shared database table polled by consumers | 3 | 2 | 3 | 2 | 26 | Orders only writes locally; couples schemas across teams (hidden contract); rows are queryable when debugging |

Choose outbox + broker. It wins on QS-M1, the second-ranked driver, not by sweeping every column: it scores
worst on QS-O1. Tradeoff point: asynchrony improves availability and modifiability but moves consistency to
"eventual" and harms end-to-end traceability (mitigate with correlation IDs propagated through message headers
and distributed tracing; add that as a follow-up task tied to QS-O1). The scores are judgements; write the
reasoning in the ADR so reviewers can dispute a cell, not the whole choice.

Matrix rules:
- If one option scores best in every column, you have missed a driver or left out its cost. Add the missing
  column (operability, cost, consistency, debuggability) before deciding.
- If swapping two adjacent weights flips the winner, the decision is close; say so in the ADR and name the
  scenario whose confirmation would settle it.
- Check each Total by hand (score x weight, summed); a wrong sum undermines the claim that the matrix is auditable.

### Reversibility and timing

- **One-way vs two-way doors** (practitioner heuristic popularised by Amazon, not SAiP): a two-way door is cheap
  to reverse (a library behind our own interface, a feature flag); decide fast. A one-way door is costly to
  reverse (public API shape, data model of a shared store, choice of primary database, cross-service protocol);
  slow down, prototype, write an ADR.
- **Last responsible moment** (practitioner heuristic from lean software development): defer a decision until
  the cost of not deciding exceeds the cost of deciding, but make it before options close by default. Deferring
  is itself a design act: keep the option open with an interface or binding-time choice.
- Increase reversibility deliberately: put volatile choices behind ports, bind late, version interfaces,
  keep data migrations expand-then-contract.

### When an ADR is warranted

Use the single trigger list in [documentation.md](documentation.md) §7 ("Is it ADR-worthy?"): one-way doors,
structure changes, QA tradeoffs, new external systems, cross-cutting conventions, multi-team impact, (H,H)
scenarios and accepted debt. Do not write ADRs for reversible, local choices (naming, a private helper library).

---

## 12. Anti-overengineering

- **Every boundary maps to an ASR.** Each module split, service, queue, interface or abstraction layer must cite
  the scenario (or constraint) that justifies it. An interface with one implementation, no test double, no
  plugin or platform seam, and not on a layer or hexagonal boundary (port) is a candidate for removal. Keep ports
  even with one implementation.
- Detect overengineering in review with the signals already listed in
  [quality-attributes-change.md](quality-attributes-change.md) (speculative generality) and
  [design-patterns-in-code.md](design-patterns-in-code.md) (section 5, pattern-itis): pass-through layers that
  only forward calls, identical DTO-to-entity mappers, and "separate" services that are always deployed together
  or share one database.
- Start with the simplest structure that meets the ranked scenarios: a modular monolith before microservices,
  in-process events before a broker, one database with clear ownership before polyglot persistence, unless a
  driver says otherwise.
- Do not design for load, tenants or integrations nobody has asked for; design so they can be added
  (seams, ports), then stop.
- Produce a **"deliberately not doing"** list in every design brief and ADR set, e.g.:
  - "No separate read model (CQRS): read load is < 50 req/s; revisit if QS-P2 is breached."
  - "No multi-region: 99.9 % target is met single-region with multi-AZ."
  - "No plugin system: only one customer variant exists; Strategy + config is enough."
  Each item names the trigger that would reopen it.

---

## 13. Architecture debt (SAiP ch. 23)

Architecture debt is structural technical debt: flaws in how files and modules relate that make every
subsequent change cost more (the "interest"). Identify it from history plus structure, not from opinion.

- **Hotspots:** groups of architecturally connected files that attract a disproportionate share of bug fixes and
  changes. Find with churn x complexity and bug-fix commit counts.
- **Co-change:** files that repeatedly change together without a structural dependency explaining it
  (hidden coupling), mined from version control.
- **Structural anti-patterns** described in the architecture-debt literature SAiP draws on (Kazman, Cai, Mo et
  al.); report findings under these names:
  - *Unstable interface*: a widely depended-on element that changes often, dragging its dependents along.
  - *Modularity violation*: files that co-change repeatedly without a structural dependency between them.
  - *Unhealthy (improper) inheritance*: a base class depends on its subclasses, or a client depends on both
    base and subclass.
  - *Clique*: a dependency cycle among classes/files.
  - *Package cycle*: a dependency cycle among packages/modules.
  - *Crossing*: an element with high fan-in and high fan-out that changes often with both sides.
  Check exact names against the chapter. [verify]
- **Design rule spaces (DRSpaces):** the chapter's analysis models architecture as design rules (key interfaces)
  and the files that depend on them, typically shown as a design structure matrix (DSM). The research tools
  (Titan, later the commercial DV8) compute DRSpaces and these anti-patterns; without them, approximate with a
  DSM built from the import graph plus co-change mining (see
  [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md)). [verify]

Concrete commands and metrics: [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md).

**Remediation recipe:**
1. Quantify interest: changes and bug fixes in the hotspot over the last N months vs elsewhere.
2. Write an ADR for the target structure (break the cycle, introduce a stable interface, split or merge modules).
3. Add a fitness function that fails on regression (e.g. no cycle between `billing` and `orders`), initially
   with a baseline/allow-list of existing violations; see [fitness-functions.md](fitness-functions.md).
4. Refactor incrementally behind the check; shrink the allow-list each step.
5. Re-measure churn/defects in the area to confirm the interest went down.

---

## 14. The architect's role as operating rules (SAiP ch. 24-25)

SAiP frames competence as duties, skills and knowledge; the role in projects covers working with project
management, incremental/agile development and distributed teams. Operate by these rules:

- Make the drivers explicit before proposing structure; restate them at the top of any design.
- Design enough up front to address the (H,H) scenarios and one-way doors; evolve the rest iteratively.
- Deliver decisions as code the team can build on: package skeletons, interfaces, a walking skeleton, fitness
  functions. A diagram without an enforced rule decays.
- Explain tradeoffs in stakeholder terms (cost, risk, time to market), not pattern names.
- Keep decisions traceable: scenario -> tactic/pattern -> ADR -> code location -> test.
- Review conformance continuously; flag drift with evidence (file:line), not taste.
- Distinguish facts about the code (AS-IS) from proposals (TO-BE) in every output.

**Conway's law** (Conway, "How Do Committees Invent?", *Datamation*, 1968): a system's design tends to mirror
the communication structure of the organisation that builds it. Apply it both ways: check that module and
service boundaries align with team ownership (CODEOWNERS, commit authorship), and when proposing a boundary,
say which team owns each side. A boundary split across teams needs a stable, versioned interface; a boundary
inside one team can stay cheap and internal.
