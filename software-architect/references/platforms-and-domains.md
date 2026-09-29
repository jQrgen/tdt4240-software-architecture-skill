# Platform and domain contexts: cloud/distributed, containers, mobile, edge/IoT, ML-enabled, quantum, games/real-time

Use this file when the **deployment context or problem domain** changes which quality attributes (QAs)
dominate and which tactics are defaults. Each section gives: architecturally significant
characteristics, ASR questions to ask, default tactics and patterns, code/infra evidence and review
signals, and pitfalls. Section 8 is a one-table summary.

Not here (follow the links): tactic catalogues and scenario templates in
[quality-attributes-runtime.md](quality-attributes-runtime.md) and
[quality-attributes-change.md](quality-attributes-change.md); pattern cards (Load Balancer, Map-Reduce,
Microservices, Publish-Subscribe) in [architectural-patterns.md](architectural-patterns.md); GoF
cards in [design-patterns-in-code.md](design-patterns-in-code.md); module layouts and ports in
[module-layout-and-interfaces.md](module-layout-and-interfaces.md); smell IDs (D1, R1, C6, ...) in
[review-playbook.md](review-playbook.md); automated checks in [fitness-functions.md](fitness-functions.md);
deployment/allocation views in [documentation.md](documentation.md); eliciting ASRs in
[design-workflow.md](design-workflow.md).

> Theory adapted in part from the Wikipendium TDT4240 compendium (CC BY-SA 3.0). See [../CREDITS.md](../CREDITS.md).

**Chapter map.** SAiP = Bass, Clements & Kazman, *Software Architecture in Practice*. 4th ed. (2021):
ch. 15 Software Interfaces, ch. 16 Virtualization, ch. 17 The Cloud and Distributed Computing, ch. 18
Mobile Systems, ch. 26 Quantum Computing. 3rd ed. (2013): ch. 26 Architecture in the Cloud, ch. 27
Architectures for the Edge (edge-dominant systems, **dropped from the 4th ed.**). SAiP has **no
chapter on machine learning or on edge/IoT computing** in either edition; sections 4.2 and 5 are
practitioner material and are labelled as such. Game architecture (section 7) draws on Rollings &
Morris and Nystrom, not SAiP.

**How to use.** Identify every context that applies (a mobile game with a cloud backend hits 1, 3, 7).
Seed the utility tree from section 8, then confirm with the ASR questions or record an assumption.

---

## 1. Cloud and distributed systems (SAiP 4th ed. ch. 17; 3rd ed. ch. 26)

### 1.1 Characteristics that matter architecturally

NIST SP 800-145 definition (the framing SAiP 3rd ed. ch. 26 uses; verify before attributing it to
4th ed. ch. 17): five characteristics (on-demand self-service, broad network access, resource
pooling, rapid elasticity, measured service); service models IaaS / PaaS / SaaS (you manage less as
you move right); deployment models public, private, community, hybrid. (Practitioner framing, not
NIST: FaaS and BaaS are usually placed between PaaS and SaaS.)

What changes for the architect:
- **Failure is normal.** At data-centre scale machines, disks, zones and networks fail routinely. You
  cannot distinguish a slow peer from a dead one; only timeouts make failure detectable.
- **Capacity is elastic but not instant:** autoscaling lags demand (start, image pull, warm-up).
- **Everything is a network call.** Latency has a long tail; fan-out requests are as slow as the
  slowest replica.
- **Cost is a runtime quality.** Measured service turns design choices (chatty calls, egress,
  over-provisioning, idle capacity) into a monthly bill. Treat cost as a QA with a budget and a
  measure, not as an afterthought.
- **Shared tenancy.** Noisy neighbours (performance), cross-tenant leaks (security), one bad deploy
  hitting every tenant (availability).
- **Jurisdiction.** Where data physically lives is a legal constraint (data residency, GDPR
  transfers), which constrains region choice, replication topology and backups.

### 1.2 ASR questions

- Load profile (peak/average, growth, burstiness)? Availability target per user journey, error budget, zone/region outage in scope?
- Which data must be strongly consistent (money, inventory, identity) and which can be eventually
  consistent (feeds, counters, search)? What staleness is acceptable, in seconds?
- Single-tenant or multi-tenant? Isolation level per tenant: shared schema, schema per tenant,
  database per tenant, deployment per tenant?
- Allowed regions for primary, replicas and backups? Cost ceiling and who is alerted? RPO and RTO?

### 1.3 Default tactics and patterns

| Concern | Default | SAiP tactic / pattern |
|---|---|---|
| Scale out | Stateless service instances behind a load balancer; autoscale on a leading signal (queue depth, request rate) rather than CPU alone | Maintain multiple copies of computations; Load Balancer pattern (4e ch. 9) |
| Session state | Keep it **out of the instance**: token-carried state (signed JWT) or a shared store (Redis, database). Sticky sessions only as a deliberate, recorded tradeoff. JWTs: keep them short-lived and small, and pair them with a refresh token or denylist where revocation (logout, compromised account) is an ASR; otherwise use an opaque session ID in a shared store | Enables replacing failed instances (availability) |
| Remote calls | Timeout on every call, bounded retries with exponential backoff and jitter, circuit breaker, bulkhead per dependency | Detect faults (timeout); recover (retry); prevent faults; see R1 |
| Retries | Make writes **idempotent**: client-generated idempotency key stored with the result; dedupe consumers of at-least-once queues | Precondition for safe retry |
| Consistency | Choose per data item; single writer per aggregate; outbox for "update DB and publish event"; sagas with compensations across services | See R4 in the playbook |
| Load smoothing | Queue between request intake and slow work; backpressure; rate limits per tenant | Manage work requests (4e; 3e: manage sampling rate); limit event response; bound queue sizes |
| Tail latency | Timeouts below the caller's budget, hedged requests for idempotent reads, cache hot reads | Bound execution times; maintain multiple copies of data |
| Multi-tenancy | Tenant ID resolved once at the edge, carried in context, enforced in the data layer (row-level security or a mandatory query filter); per-tenant quotas | Limit access; separate entities |
| Cost | Scale-in policies, right-sized requests, storage lifecycle rules, per-tenant cost attribution tags | Measured service as a QA |

**Stateless vs sticky sessions.** Sticky sessions (affinity by cookie or IP) keep in-memory session
state working without a shared store, but: uneven load, sessions lost when the instance dies, harder
scale-in and rolling deploys. Accept them only for short-lived, reconstructible state (e.g. WebSocket
connection affinity) and write an ADR.

**CAP and PACELC (practitioner framing, not SAiP terminology).** CAP (Brewer; formalised by Gilbert &
Lynch, 2002): during a network **p**artition, a replicated store must give up either **c**onsistency
(linearizability) or **a**vailability. PACELC (Abadi, 2012) adds: **e**lse, in normal operation,
trade **l**atency against **c**onsistency. Use them to force an explicit question ("what does this
endpoint return when the replica is cut off?"), not to label whole databases; most stores are tunable
per operation (quorum reads/writes, read-your-writes, bounded staleness).

### 1.4 Evidence and review signals (code and IaC)

Read Kubernetes manifests, Helm charts, Terraform/Pulumi/CDK, `docker-compose.yml`, serverless
configs and the HTTP/gRPC client setup.

| Signal | Where to look | Why it matters |
|---|---|---|
| `replicas: 1` on a user-facing Deployment, or a single instance/ASG size 1 | manifests, Terraform | No redundancy: every deploy or node drain is an outage |
| No `readinessProbe`/`livenessProbe` (and no `startupProbe` for slow starters) | Deployment specs | Traffic routed to unready pods; hung pods never restarted |
| Liveness probe that checks downstream dependencies | probe handler code | A database blip restarts every pod at once (cascading failure) |
| No `resources.requests` | container specs | Noisy neighbour; HPA CPU/memory utilization targets cannot be computed without requests; pods whose containers set neither requests nor limits are BestEffort and evicted first under node pressure. Always a finding |
| No memory `limits` | container specs | No per-container OOM signal; a leak pushes the node into memory pressure and evicts neighbours. A finding |
| CPU `limits` set, or missing | container specs, `container_cpu_cfs_throttled_*` metrics | A deliberate choice, not a default: tight CPU limits cause CFS throttling and p99 latency spikes. Recommend a CPU limit only with a measured reason; where one exists, check the throttling metrics |
| No `PodDisruptionBudget` for replicated services | chart/manifests | Node upgrades can evict all replicas together |
| Secrets as plain `env:` values, in `values.yaml`, `.env` committed, or Terraform variables with defaults | repo grep | C2; note Kubernetes `Secret` objects are only base64-encoded unless encryption at rest or an external secret store is configured |
| HTTP clients built without a timeout (Go `http.Client{}` / `http.Get`, Python `requests.get(url)` with no `timeout=`, JS `fetch` with no `AbortSignal`) | client factories | R1: unbounded waits exhaust threads/connections |
| Retries on non-idempotent POSTs, or retries at several layers (client, mesh, SDK) multiplying | client config, mesh config | Duplicate side effects; retry storms |
| In-memory session maps, local file uploads, in-process caches treated as the source of truth | service code | Breaks under scale-out and restarts |
| One database shared by several services | connection strings | I5; coupling hidden behind "microservices" (R2) |
| No tenant filter at the data layer, tenant ID taken from request body | repositories, SQL | Cross-tenant data leak |
| Region hard-coded, replicas/backups in a region outside the allowed set | IaC | Residency violation |
| No trace-context propagation (W3C `traceparent`, OpenTelemetry instrumentation) or correlation ID across service calls and queue messages; no rate/error/duration (RED) metrics per dependency | HTTP/gRPC client and middleware setup, message headers, metrics registration | C4: failures cannot be localised across services |

Linters that automate part of this (check names are version-dependent; confirm against the installed
version): `kube-linter lint <dir>`, `checkov -d <dir>`, `trivy config <dir>`, `hadolint Dockerfile`.
Turn the must-have rules into CI fitness functions ([fitness-functions.md](fitness-functions.md)).

### 1.5 Pitfalls

- Microservices before a load, team or deployability ASR requires them (R2: a distributed monolith is worse than a modular monolith).
- Autoscaling on CPU for an IO-bound service; scaling the stateless tier while the database is the bottleneck.
- Treating a managed service's SLA as your availability: serial dependencies multiply (availability
  arithmetic in [quality-attributes-runtime.md](quality-attributes-runtime.md)).
- Assuming exactly-once delivery (design for at-least-once plus idempotency); ignoring egress cost in chatty designs.

---

## 2. Virtualization and containers (SAiP 4th ed. ch. 16)

### 2.1 Mechanisms and their tradeoffs

| Unit | Isolation | Start time | Architect's reading |
|---|---|---|---|
| Virtual machine | Own guest kernel on a hypervisor; strong | Tens of seconds to minutes | Strong tenant isolation, OS freedom; heavier, slower to scale |
| Container | Shares the host kernel (namespaces, cgroups); weaker | Sub-second to seconds, plus image pull | Reproducible packaging, dense packing, fast rollout; kernel is a shared attack surface |
| Pod (Kubernetes) | One or more containers sharing network namespace and volumes | as container | Unit of scheduling; home of **sidecars** |
| Serverless / FaaS | Platform-managed (often micro-VMs); you see only the function | Cold start from milliseconds to seconds depending on runtime and package size | Scale to zero, pay per invocation; execution-time limits, statelessness enforced, vendor coupling |

- **Images and layers.** An image is an immutable, versioned artefact of cached, shared layers and the
  deployable unit: promote one digest through environments and inject config at start.
- **Orchestration** (Kubernetes, Nomad, ECS, Cloud Run, ...) takes over placement, restart, scaling and
  rollout. That buys availability tactics (health checks, restart, redundancy) only if the app
  cooperates: fast start, graceful shutdown on SIGTERM, honest health endpoints, no local state.
- **Sidecars** (proxy, log shipper, mesh agent) move mTLS, retries and telemetry out of the app (use an
  intermediary). Cost: latency per hop, another failure point, retry/timeout behaviour split between
  code and mesh config (review both).

### 2.2 ASR questions and choosing the unit

Ask:
- Longest request or job duration, compared with the platform's execution time limit?
- Tolerance for cold-start latency on the first request after idle (p99 budget)?
- Traffic shape: spiky or scale-to-zero, or steady?
- Need for a custom kernel, kernel modules, a GPU, or long-lived connections (WebSockets, streaming gRPC)?
- Tenant isolation requirement: is any tenant's code or input hostile?
- Vendor-portability constraint (multi-cloud, on-premises, exit plan)?

Decide (record the choice and the answers in an ADR):

| Choose | When |
|---|---|
| Serverless / FaaS | Event-driven or spiky load, short stateless handlers well inside the time limit, cold start acceptable, vendor coupling accepted |
| Containers | Long-running services, steady load, WebSockets or other long-lived connections, portability across clouds and on-premises |
| VMs or micro-VMs | Hostile multi-tenant code, kernel or OS requirements, licensed or legacy software that expects a full OS |

### 2.3 QA effects

- **Deployability:** immutable images + declarative manifests enable rolling, blue-green and canary
  deploys and fast rollback (redeploy the previous digest).
- **Performance:** near-native CPU; costs are noisy neighbours without limits, throttling under tight
  CPU limits (latency spikes), image pulls and cold starts.
- **Security:** isolation (separate entities) is weaker than VMs. Defaults: non-root, read-only root
  filesystem, dropped capabilities, minimal base, image scanning, pinned digests.
- **Testability:** the same image runs in CI (sandbox tactic), including Testcontainers-style tests.

### 2.4 Review signals

- `FROM <image>:latest` or unpinned bases; build tools or secrets in the final image (no multi-stage
  build); `USER root` or no `USER`; per-environment images instead of one image plus config.
- SIGTERM ignored or shutdown longer than the grace period (in-flight requests dropped on every
  deploy); durable data written to the container filesystem.
- Serverless: globals assumed to persist, work beyond the platform time limit, synchronous chains
  of functions (latency and cost multiply).

---

## 3. Mobile systems (SAiP 4th ed. ch. 18)

### 3.1 Characteristics

- **Energy** is finite; radio, GPS, screen and CPU wake-ups dominate (energy efficiency QA, 4e ch. 6,
  in [quality-attributes-runtime.md](quality-attributes-runtime.md)).
- **Intermittent connectivity** (offline, handover, variable latency); **resource limits** (memory,
  storage, thermal throttling, wide device range).
- **Lifecycle owned by the OS:** backgrounding, config changes, **process death** while the app looks open.
- **Background execution limits:** Android Doze/App Standby and background-start restrictions,
  iOS background modes and `BGTaskScheduler`; work must be deferrable and constrained.
- **Platform heterogeneity:** Android/iOS/web/desktop, OS versions, screen sizes. Cross-platform
  options (Kotlin Multiplatform sharing logic with native UI or Compose Multiplatform; Flutter;
  React Native) trade shared code against native fidelity and team skills; decide per ADR.
- **Store deployment:** review delays, staged/phased rollout, and **no rollback of installed
  binaries**. Users run old versions for months.
- **Sensors and secure hardware:** noisy, permissioned inputs; Keystore / Keychain / Secure Enclave for keys.

### 3.2 ASR questions

- Which features must work offline? What happens to a write made offline: queued, rejected, merged?
- Conflict policy when two devices edit the same record: last-writer-wins, field-level merge, CRDT, or user resolution?
- Freshness: push (FCM/APNs) or periodic sync? Acceptable staleness per screen?
- Oldest OS version and oldest app version the backend must support? Forced-upgrade policy?
- Energy budget: background sync frequency, location accuracy, allowed wake-ups?
- What secrets live on the device, and what if the device is rooted/jailbroken or lost?
- Which state must survive process death (form input, navigation, in-progress purchase)?

### 3.3 Default tactics and patterns

- **Offline-first:** for the general structure and when it applies, use the "Offline-first sync"
  card (section 3.3) and the selection row in [architectural-patterns.md](architectural-patterns.md);
  for the view-model/repository layout, [module-layout-and-interfaces.md](module-layout-and-interfaces.md).
  Mobile-specific checks: outgoing writes go to a **persistent outbox** (survives process death) with
  idempotency keys and are replayed on reconnect; a written conflict policy **per entity**; server
  revisions or version vectors per record; **tombstones** for deletes so they sync.
- **Energy:** batch network work; use platform schedulers (Android WorkManager with constraints, iOS
  background tasks); push instead of polling; coalesce location updates; reduce frame rate on idle screens.
- **Lifecycle:** persist state that must survive process death (`SavedStateHandle` /
  `onSaveInstanceState`, iOS state restoration); cancel screen-scoped work with its screen.
- **Release safety without rollback:** server-driven flags and remote config (kill switch),
  backward-compatible versioned APIs, a **minimum supported version** check with forced-upgrade
  screen, staged rollout (Play can halt it; App Store phased release can pause). ADR the oldest supported client.
- **Security:** keys in Keystore/Keychain, not `SharedPreferences`/`UserDefaults`; no long-lived backend
  secrets in the binary (extractable); certificate pinning only with a rotation plan.
- **Shared code:** keep platform APIs out of shared modules (C6); expose them through interfaces or
  `expect`/`actual` declarations in KMP.

### 3.4 Review signals

| Signal | Evidence | Smell / QA |
|---|---|---|
| Disk or network IO on the main thread | Android: DB/HTTP calls in `onCreate`/composables without a background dispatcher; StrictMode violations; iOS: synchronous `Data(contentsOf:)` on main | Performance, usability (ANRs, jank) |
| Wakelocks held without timeout, foreground services for work a scheduler could run | `PowerManager.WakeLock.acquire()` without timeout | Energy |
| Polling loops (`while(true) { fetch(); delay(5000) }`) where push or scheduled sync fits | repositories, services | Energy, cost |
| Unbounded image or response caches | custom `HashMap` caches, no size limit on the image loader | R5; OOM on low-end devices |
| State held only in singletons or view objects | global `object AppState`, no saved state | Lost on process death (D5) |
| Backend change that removes a field or endpoint old clients use | API diff, no version gate | I3; broken installed base |
| API keys or tokens in `BuildConfig`, `Info.plist`, JS bundles | grep | C2 |
| Platform SDK types in `commonMain` / shared packages | imports of `android.*`, `UIKit`, Firebase in shared code | C6, I4 |

**Pitfalls:** trusting the emulator's Wi-Fi (test with network conditioning and airplane mode);
"offline support" that is a read cache with no write path or conflict policy; web-style hot-fix
assumptions (a bad binary stays installed; only server-side switches help).

---

## 4. Edge-dominant systems and edge/IoT computing

Two different meanings of "edge"; name which one you mean.

### 4.1 Edge-dominant systems (SAiP 3rd ed. ch. 27; not in the 4th ed.)

In SAiP 3e, "edge" means the **people at the edge of the organisation**: users and outside
contributors who create most of the value (open-source projects, Wikipedia-style peer production,
social platforms, app ecosystems around a platform). It is **not** edge computing.

**Metropolis model** (Kazman & Chen, "The Metropolis Model: A New Logic for Development of
Crowdsourced Systems", *Communications of the ACM*, 2009), three rings:

| Ring | Who | Architectural role |
|---|---|---|
| Core | Small, tightly governed group | Owns the kernel/platform: stable, modular, well-specified interfaces, high quality bar |
| Periphery | Many loosely coordinated contributors | Build plug-ins, apps, extensions, content tools against the core's interfaces |
| Masses | End users who also contribute | Use the system and supply content, data, bug reports, requests |

Principles, paraphrased (check the exact names and list in 3e ch. 27 before quoting): requirements
split into core requirements set centrally and periphery requirements that emerge continuously;
the architecture is likewise split into a stable core and an open periphery; implementation is
fragmented across many independent contributors; testing is distributed to the periphery and the
masses; operations are continuous rather than release-bound; the crowd is managed through
governance, meritocratic promotion and contribution rules rather than a hierarchy.

Apply it to a **platform or plugin ecosystem**: modifiability and integrability at the core boundary
(microkernel/plug-in, versioned extension APIs, sandboxed extensions), plus core scalability. Signals:
extension points exposing internal types (I1/D7), no deprecation policy (I3), plugins with full core
privileges, behaviour depending on plugin load order.

### 4.2 Edge and IoT computing (practitioner material, beyond SAiP)

**Characteristics:** constrained, heterogeneous devices; intermittent or metered links; physical
attacker access; field lifetimes of years; cyber-physical effects, so **safety** may apply
([quality-attributes-runtime.md](quality-attributes-runtime.md)).

**ASR questions:** What must work with no uplink, and for how long? Data loss tolerated offline? What
if an update fails halfway? How is a device enrolled and revoked? Local control-loop deadline? Which
actions must never depend on the cloud?

**Default tactics:**
- **Local autonomy:** control loops and safety interlocks run on the device or gateway; the cloud
  supervises, analyses and configures. A lost uplink degrades features, not safety.
- **Store-and-forward:** durable local buffer with bounded size, explicit drop policy (oldest first,
  downsample), sequence numbers and idempotent ingestion so replays after reconnect are harmless.
- **OTA updates:** signed images verified on device, A/B (dual-slot) partitions with automatic
  rollback if the new image fails its health check, staged rollout by device cohort, update
  framework rather than hand-rolled (TUF and Uptane are established designs; Mender, SWUpdate and
  RAUC are examples of tooling).
- **Device identity:** per-device keys (ideally in a secure element/TPM), mutual TLS or signed
  tokens, a revocation path; never a fleet-wide shared secret.
- **Protocols:** publish-subscribe (MQTT or similar) with QoS matched to data value; versioned payloads (old firmware stays in the field).
- **Safety interplay:** watchdogs, safe states on loss of communication, partitioning of safety-critical from non-critical software.

**Review signals:** device cannot operate when the broker is unreachable; unbounded local queue or
none at all; update path without signature check or rollback; shared credentials baked into
firmware; timestamps from unsynchronised device clocks used for ordering; telemetry schema with no
version field.

---

## 5. ML-enabled systems (practitioner material, beyond SAiP)

SAiP has no ML chapter. The QA vocabulary still applies: write six-part scenarios for accuracy
degradation, latency, cost and rollback the same way as for any other component.

### 5.1 Characteristics

- The **model is a component with its own lifecycle** (data, training, evaluation, release,
  monitoring, retirement) that runs on a different cadence from the code.
- Behaviour is learned from data, so **data is part of the build**: a change in training data is a
  change in the system.
- Correctness is statistical; the model **will** be wrong on some inputs, and silently more wrong as
  the world drifts.
- Many hidden dependencies (feature pipelines, upstream tables, labelling). Standard reference:
  Sculley et al., "Hidden Technical Debt in Machine Learning Systems" (NeurIPS 2015).

### 5.2 ASR questions

- What happens to the user when the model is wrong, slow or unavailable? Is there a safe default?
- Latency and cost budget per prediction; online, batch or on-device inference?
- How fast must a bad model be rolled back, and to what?
- How is quality measured in production (labels arrive when? proxy metrics?), and what drift
  threshold triggers an alert or retraining?
- Regulatory or explainability constraints; which decisions need a human in the loop?

### 5.3 Default tactics

- **Model behind a port:** callers depend on a `Predictor` interface returning a domain result, not
  framework tensors; swapping models or vendors is an adapter change.
- **Version everything together:** model artefact, training data snapshot, feature definitions and
  code commit, recorded in a model registry; serving logs the model version with each prediction.
- **Avoid training-serving skew:** one feature-computation implementation shared by training and
  serving (a feature store or a shared library), plus a test that compares both paths on the same
  input.
- **Safe rollout:** shadow, then canary, automatic rollback on metric regression; keep the previous model warm.
- **Fallbacks:** rules-based default, last good answer, or "no recommendation" on timeout, error or low confidence.
- **Monitoring:** input and prediction drift, outcome metrics when labels arrive, latency, cost; alerts to owners.

### 5.4 LLM integration as an external dependency

Treat a hosted or self-hosted LLM like any unreliable, metered, non-deterministic remote service:
- **Timeouts, retries with backoff, circuit breaker** (R1); streaming does not remove the need for an overall deadline.
- **Cost budgets:** cap tokens per request, user and day; meter and alert; cache reusable results.
- **Non-determinism:** never assume identical output for identical input; store outputs that must be repeatable.
- **Pin versions:** pin model identifiers (not floating aliases) and version prompts as code;
  changing either is a release that goes through evaluation.
- **Validate output at the boundary:** parse into a typed schema, reject or repair invalid output,
  never pass model output unchecked into SQL, shell, HTML or tool calls; treat retrieved documents and
  user input as untrusted (prompt injection is an input-validation problem at an architectural boundary).
- **Eval harness as a fitness function:** a versioned set of cases with graded expectations, run in
  CI on every prompt or model change, with a pass-rate threshold.
- **Keep the provider behind a port:** a second provider or local model is an adapter; tests use a fake.

### 5.5 Review signals

Model loaded from a path with no version; feature code duplicated in a notebook and the service;
no fallback branch around the predict call; vendor SDK calls scattered through handlers (I4);
prompts assembled by string concatenation in many places; no timeout on the LLM client; no
per-tenant token accounting; model output parsed with regex and used directly.

---

## 6. Quantum computing (SAiP 4th ed. ch. 26)

Architecturally, a quantum processor (QPU) is a **remote, specialised coprocessor** in a hybrid
system: a classical host prepares the problem, submits a circuit, and post-processes probabilistic
measurement results over many shots. Design concerns are the familiar ones for a scarce remote
accelerator: interface, queueing latency, error rates, cost. The near-term concern for most systems
is **crypto agility**: Shor's algorithm threatens current public-key cryptography, so keep
algorithms swappable behind an interface and plan the post-quantum migration.

---

## 7. Games and real-time simulation (domain reference)

### 7.1 QA priorities and ASR questions

| QA | Typical measure |
|---|---|
| Performance (frame budget) | 16.7 ms per frame at 60 Hz (8.3 ms at 120 Hz) for input + simulation + render; p99 frame time, not the average |
| Input latency | Input-to-photon or input-to-effect time; network games add RTT |
| Determinism | Same inputs + seed + fixed dt give the same state (required for lockstep, replays, tests) |
| GC / allocation pressure | Zero steady-state allocations per frame on managed runtimes (JVM, .NET, JS) |
| Content modifiability | New level, unit or item added as data, without code changes |
| Cheat resistance | Server authority over outcomes; clients cannot set their own score or position |
| Portability | Same core on desktop, mobile, console, web |

**ASR questions** (each answer points at a subsection):
- Target devices and frame rate (30/60/120 Hz), and the lowest-end device? Sets the frame budget,
  the allocation rules and the fitness functions (7.3, 7.5, 7.10).
- Single-player, co-op or competitive multiplayer, and players per match? Decides authority, sync
  model and tick rate (7.7).
- Are replays, spectating or rollback needed? These force a deterministic simulation with
  tick-stamped input (7.3, 7.7, 7.10).
- Expected entity count per scene? Plain objects with components versus ECS; spatial partition (7.4, 7.5).
- Will designers add content without programmers? Data-driven content, token analysis (7.2, 7.4).
- Target platforms (desktop, mobile, web, console)? Hardware abstraction and the core/platform
  split (7.2, 7.8).
- Managed BaaS or your own server for the backend? The backend port and where authority lives (7.7, 7.8).

### 7.2 Rollings & Morris ch. 17

Source: Rollings & Morris, *Game Architecture and Design: A New Edition* (New Riders, 2003/2004);
confirm the edition and chapter number against the copy you cite (the 1999 Coriolis first edition
may number chapters differently). Verify wording against the text before quoting; the summary below is at concept level.
- **Hardware abstraction:** logic codes against a stable abstraction of graphics, sound, input and OS;
  each platform implements it (encapsulate, abstract common services); cost: harder platform tuning.
- **Token analysis:** list every **token** (an element the player manipulates directly or
  indirectly and that the game supervises: ship, enemy, bullet, score, timer); build a **token
  interaction matrix** (tokens on both axes, each cell the interaction and the event it raises);
  derive the logical structure from it. Tokens become entities/classes; interactions become systems,
  collision handlers or events; a passive token (score) observes events. Each non-empty cell is a
  candidate test scenario.

### 7.3 Game loop

Game loop, update method, component, event queue, object pool, double buffer, spatial partition and
service locator follow Robert Nystrom, *Game Programming Patterns* (2014). The ECS storage notes, the
screen-stack notes, netcode (7.7) and the layout (7.8) are practitioner material from the sources
named in each subsection.

| Variant | Frame-rate effect | Determinism |
|---|---|---|
| Update + render as fast as possible, fixed step | Game speed depends on hardware | Deterministic but wrong speed |
| Fixed step + sleep/vsync cap | Slows down when a frame overruns | Deterministic |
| Variable timestep `update(elapsed)` | Adapts to any frame rate | Not deterministic; large dt causes tunnelling and unstable physics |
| **Fixed update, variable render (accumulator)** | Render rate free; simulation stable | Deterministic given tick-stamped input |
| Subsystems at their own rates / threads | Render not blocked by slow AI or network | Needs a safe state hand-off (double buffer, message queue) |

Canonical fixed-timestep accumulator (after Glenn Fiedler's "Fix Your Timestep!"):

```text
const DT = 1/60                       // simulation step, seconds
const MAX_FRAME = 0.25                // clamp: prevents the spiral of death
acc = 0; prev = now(); tick = 0
loop:
    t = now(); frame = min(t - prev, MAX_FRAME); prev = t
    acc += frame
    inputQueue.record(tick, pollInput())             // stamp input with the next tick to simulate
    while acc >= DT:
        inputs = inputQueue.takeFor(tick)
        previousState = currentState
        currentState = simulate(currentState, inputs, DT)   // pure: returns a NEW state object
        tick++; acc -= DT
    alpha = acc / DT
    render(lerp(previousState, currentState, alpha))
```

`simulate` must return a new state (or write into a second buffer via `next.copyFrom(current)`). If it
mutates `currentState` in place, `previousState = currentState` aliases the two in any
reference-typed language (Kotlin, Java, C#, TS, Python) and `lerp` interpolates an object with
itself. For determinism, record and replay inputs by simulation tick, not by render frame.

Under load the renderer drops frames, not simulation steps; if one `simulate` costs more than DT the
loop falls behind for good (spiral of death), and the clamp trades slow-motion for survival. Engines
that own the loop differ: Unity `FixedUpdate` and Godot `_physics_process` provide a fixed simulation
step; libGDX `render()` is a variable-rate per-frame callback, so implement the accumulator above
inside it (and call Box2D `world.step` with the fixed DT, not the frame delta). Decouple network
send rate (e.g. 10-30 Hz) from render rate. Tactics: bound execution times, manage work requests
(3e: manage sampling rate), introduce concurrency.

### 7.4 Update method, components and ECS

- **Update method:** each object has `update(dt)`; defer adds/removes to frame end; make order explicit.
- **Component pattern:** compose objects from physics/render/input components instead of deep inheritance.
- **Entity-Component-System:** entities are **IDs**; components are plain data; systems run over all
  entities with a given component set. Storage is **data-oriented**: components of one type stored
  contiguously for cache-friendly iteration. Two broad layouts: **archetype/table** storage (entities
  with the same component set share tables; fast iteration, costlier add/remove of components) and
  **sparse-set** storage (per-component dense arrays with an index; cheap add/remove, joins across
  components cost more). Data-oriented examples: Unity DOTS/Entities, Bevy (table/archetype storage
by default, sparse-set storage selectable per component), EnTT, flecs. libGDX Ashley gives the ECS
structure with object-based entities and components; it is not a data-oriented layout, so expect no
cache-locality win.
- **When ECS is overkill:** few entity types, a turn-based or UI-heavy game, or a small team without
  ECS experience; plain objects with components are easier to follow. Choose ECS for many
  similar entities, heavy per-frame iteration, or data-driven content, and record why.
- **Document it:** list components (fields), systems (the component set each reads/writes) and the
  per-frame system order; do not draw entities as classes with behaviour.

### 7.5 Event queue, object pool, double buffer, spatial partition

- **Event queue:** decouples sender and receiver in time; use for audio triggers, cross-system
  events, and applying network messages on the simulation thread. Bound it; expect one-frame
  latency; debugging is harder than direct calls.
- **Object pool:** preallocate bullets, particles, messages; reset on release to avoid GC hitches.
  Pitfalls: missing reset (stale state), references kept after release.
- **Double buffer:** read the current state while writing the next; swap at the end of the step.
  Makes update order irrelevant and gives renderers and network threads a consistent snapshot.
- **Spatial partition:** grid, quadtree/octree or BVH to answer "what is near X" without O(n^2)
  checks. Choose by entity density and movement; rebuild or update cost is the tradeoff.

### 7.6 Service locator and scene/screen state

- **Service locator** (Nystrom): global access to audio, logging, assets with a swappable provider
  (null service for tests). Testability cost: still a hidden dependency (D5), and tests must reset
  global state. Prefer constructor injection into systems; confine the locator to leaf services.
- **Scene/screen state stack:** menu, lobby, play, pause overlay, game over, with enter/exit hooks
  that own asset loading and disposal; overlays push, back pops. Check that leaving a screen releases its assets.

### 7.7 Netcode

| Decision | Options and tradeoffs |
|---|---|
| Authority | **Authoritative server** (clients send inputs, server simulates; cheat-resistant, costs servers) vs peer/host authority (cheap, trusts a client). **Choose** authoritative server for anything competitive, ranked or with an economy; host/peer authority only for co-op or casual play where cheating costs little |
| Latency hiding | **Client-side prediction** of the local player, then **server reconciliation**: server acks the last processed input; the client rewinds to the authoritative state and replays unacknowledged inputs |
| Remote entities | **Entity interpolation**: render others slightly in the past between two received snapshots (smooth, adds latency); extrapolation (fresher, overshoots) |
| Sync model | **Deterministic lockstep** (send only inputs; small bandwidth, many units; needs bit-exact determinism, including floating point across platforms, and waits for the slowest peer) vs **state synchronisation** (send snapshots/deltas; tolerant of non-determinism, more bandwidth). **Choose** lockstep for many units with small per-player input (RTS, simulations), few peers, and a simulation that is bit-exact on every target (fixed-point maths or a single platform); state sync with client prediction and interpolation for fast action games or shooters, mixed platforms, or a non-deterministic physics engine |
| Tick rate | Higher tick = lower latency and better hit accuracy, but more CPU and bandwidth cost per match. **Start** at 20-30 Hz for most action games; 60 Hz or more only for competitive shooters; then measure bandwidth per player (practitioner defaults, not a standard) |
| Lag compensation | Server rewinds hitboxes to the shooter's view time; fairer for the shooter, "shot behind cover" for the target |

Further reading: Valve's "Source Multiplayer Networking" docs; Gabriel Gambetta, "Fast-Paced Multiplayer".

### 7.8 Reference cross-platform layout: hide the backend behind a port

```text
core/            pure simulation, rules, game state, screens' logic; ports (interfaces)
                 NO platform, engine-backend or vendor-SDK imports
  port/MatchBackend.kt   port/Analytics.kt   port/Clock.kt
platform-android/ launcher + adapters (e.g. Firebase-backed MatchBackend)
platform-desktop/ launcher + adapters (local or fake MatchBackend)
platform-web/     launcher + adapters
net/              optional: protocol messages, serialisation, prediction/reconciliation code
```

```kotlin
// core: owned by the consumer, domain types only
interface MatchBackend {
    suspend fun joinLobby(player: PlayerId): Result<LobbyId>
    fun matchEvents(match: MatchId): Flow<MatchEvent>   // delivered off the sim thread
    suspend fun submitMove(match: MatchId, move: Move): Result<Unit>
}
// platform-android: class FirebaseMatchBackend(...) : MatchBackend  -- the only place the SDK appears
// launcher (composition root): MyGame(backend = FirebaseMatchBackend(...))
```

Why: core stays portable and testable (fake backend on desktop and in tests); swapping vendors
(Firebase to Supabase or a custom server) is a new adapter; callbacks are marshalled onto the sim
thread in one place (libGDX: `Gdx.app.postRunnable`). Tactics: encapsulate, use an intermediary,
restrict dependencies, abstract data sources. Enforce "no platform imports in core" (C6) with a
build-level or ArchUnit/Konsist check ([fitness-functions.md](fitness-functions.md)).

### 7.9 Game-specific review signals

| Signal | Evidence | Playbook ID |
|---|---|---|
| God `GameScreen`/`GameManager`: input, rules, rendering, networking and persistence in one class | thousand-line screen class, many unrelated fields | M1, M3 |
| Singleton game state reachable from everywhere | `GameState.instance`, `object World` mutated by UI and network code | D5 |
| Platform or vendor APIs in core | `android.*`, Firebase, Steamworks imports in the shared module | C6, I4 |
| Per-frame allocation | `new Vector2()`, list creation, string formatting, lambdas capturing state inside `update`/`render` | Performance: reduce computational overhead / increase efficiency of resource usage; object pool (7.5) |
| Network or disk IO on the render thread; backend callbacks mutating state from another thread | blocking calls in `render()`; SDK listeners writing game state directly | R3, R1 |
| Variable dt fed to physics or to networked simulation | `world.step(delta, 6, 2)` (libGDX Box2D) or `physics.update(delta)` with the raw frame delta instead of a fixed step | determinism |
| Client decides outcomes | client writes score/result to the backend directly | security (cheat resistance) |
| Content hard-coded | `if (level == 3)` branches, enemy stats in code | content modifiability |
| Screens not disposed | `setScreen(new ...)` without disposing the previous one (libGDX `Game.setScreen` calls `hide()`, not `dispose()`) | R5 |

### 7.10 Fitness functions for games

- **Frame-time benchmark:** run a scripted heavy scene headless or on a reference device; fail CI if
  p95/p99 simulation step time exceeds a budget (e.g. 8 ms of a 16.7 ms frame). Track allocations
  per frame on managed runtimes and fail if steady state is non-zero.
- **Deterministic simulation test:** seed the RNG, feed a recorded input sequence at fixed dt for N
  ticks, and compare a state hash with a golden value (and across platforms if you use lockstep).
- **Rendering kept out of the simulation core:** a dependency rule that core may not import the
  renderer/graphics packages, so simulation runs headless in tests and on a server.

```kotlin
@Test fun simulationIsDeterministic() {
    fun run(): Int { val w = World(seed = 42); repeat(600) { w.step(Inputs.recorded[it], 1f / 60) }; return w.stateHash() }
    assertEquals(run(), run())
    assertEquals(GOLDEN_HASH, run())   // regenerate only with an intentional, reviewed rule change
}
```

### 7.11 Pitfalls

ECS or netcode sophistication before the game needs it (anti-overengineering in
[design-workflow.md](design-workflow.md)); average FPS instead of frame-time percentiles; assuming
floating-point determinism across compilers, CPUs or platforms; letting a managed backend's realtime
listeners become the game loop.

---

## 8. Context summary

| Context | QAs usually on top of the utility tree | Go-to tactics | Top review checks |
|---|---|---|---|
| Cloud / distributed | Availability, performance/scalability, security (tenancy), cost | Stateless replicas + LB, timeouts/retries/idempotency, circuit breaker, queues, outbox, per-tenant limits | Timeouts on all remote calls; replicas > 1 with probes, limits, PDB; secrets management; tenant filter in data layer |
| Containers / serverless | Deployability, security (isolation), performance isolation | Immutable pinned images, config at start, graceful shutdown, sidecars, non-root | Pinned digests; multi-stage builds; SIGTERM handling; no local state; cold-start budget |
| Mobile | Energy, availability under disconnection, usability (responsiveness), deployability (no rollback), security | Offline-first local source of truth, outbox sync, platform schedulers, push, remote flags, min-version gate, Keystore/Keychain | No main-thread IO; no polling or long wakelocks; bounded caches; state survives process death; backend compatible with oldest client |
| Edge-dominant platform (3e ch. 27) | Modifiability and integrability at the core boundary, scalability | Microkernel/plug-in, versioned extension APIs, sandboxed extensions | Extension APIs leak internals; no deprecation policy; plugin privileges |
| Edge / IoT | Availability offline, safety, security (device identity), deployability (OTA) | Local autonomy, store-and-forward, signed A/B OTA with rollback, per-device keys | Works without uplink; bounded buffers; signed updates with rollback; no shared fleet secrets |
| ML-enabled / LLM | Availability (fallbacks), performance and cost budgets, modifiability (model swap), deployability (rollback) | Model/provider behind a port, versioned artefacts, shared feature code, shadow/canary, eval harness | Model and prompt versions pinned; fallback path exists; output validated; drift/cost monitoring |
| Quantum (hybrid) | Security (crypto agility), performance of the classical-quantum interface | Coprocessor behind a service interface; swappable crypto | Hard-coded crypto algorithms |
| Games / real-time | Performance (frame budget, latency), determinism, content modifiability, cheat resistance, portability | Fixed-timestep loop, ECS or components, pools, double buffer, event queue, authoritative server + prediction, platform ports | God screen; singleton state; platform APIs in core; per-frame allocation; IO on render thread |
