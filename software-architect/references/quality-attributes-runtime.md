# Runtime qualities: availability, performance, security, safety, energy efficiency, usability

Use this file when a design or review turns on how the system behaves **while it runs**. For each quality attribute (QA) it gives
the theory from Bass, Clements & Kazman, *Software Architecture in Practice* (SAiP), the code idioms that realise each tactic, and
the signals that show a tactic is missing.

Not here (follow the links): change-time qualities such as modifiability, testability, deployability and integrability in
[quality-attributes-change.md](quality-attributes-change.md); eliciting ASRs, writing concrete scenarios and utility trees in
[design-workflow.md](design-workflow.md); pattern catalogue in [architectural-patterns.md](architectural-patterns.md); smell
catalogue and report format in [review-playbook.md](review-playbook.md); turning measures into automated checks in
[fitness-functions.md](fitness-functions.md); sensitivity and tradeoff points in [evaluation-methods.md](evaluation-methods.md);
mobile, cloud and edge specifics in [platforms-and-domains.md](platforms-and-domains.md).

> Theory adapted in part from the Wikipendium TDT4240 compendium (CC BY-SA 3.0). Paraphrased, corrected and extended; see
> [../CREDITS.md](../CREDITS.md).

## 0. Chapter map and how to read a card

### Where each QA lives in SAiP
| QA | 4th ed. (2021) chapter | 3rd ed. (2013) equivalent |
|---|---|---|
| Understanding QAs (scenarios, tactics) | 3 | 4 |
| Availability | 4 | 5 |
| Energy Efficiency | 6 | none (new in 4th ed.) |
| Performance | 9 | 8 |
| Safety | 10 | brief entry in ch. 12 Other QAs |
| Security | 11 | 9 |
| Usability | 13 | 11 |

**Caution tags.** 4th-ed. chapter titles and numbers are confirmed from the publisher's table of contents; tactic names and
groupings are not fully checked against the printed book. **[verify wording in SAiP 4th ed.]** marks plausible, widely cited but
unconfirmed wording: use it as written, do not "correct" it back to 3rd-ed. wording, and pass the flag on in deliverables. Renamed
tactics give the 3rd-ed. name in parentheses. Cite chapters by title ("the Availability chapter"); that is safe in both editions.

### Vocabulary you must keep straight
- **Six-part scenario:** source, stimulus, artifact, environment, response, response measure. The response measure is a number
  with a unit and a threshold, or it is not a scenario.
- **General scenario:** system-independent; lists allowed values per part. Use it as a checklist to generate concrete scenarios.
- **Concrete scenario:** one value per part, for one system. This is what goes in an ADR or a review finding.
- **Tactic:** one design decision that affects the response of one QA. **Pattern:** a reusable bundle of tactics that usually
  affects several QAs. Circuit breaker, TMR and load balancer are patterns; retry, heartbeat and bound queue sizes are tactics.
  Never list a pattern as a tactic.

### How to read a card
Each QA has the same sections; use them in this order. Libraries in (e) are examples, not endorsements; APIs are stable unless
flagged "version-dependent", so check the project's lockfile before writing code against them.

| Section | Use it to |
|---|---|
| (a) Definition | Decide whether the concern really is this QA (performance vs usability, security vs safety) |
| (b) General scenario | Generate candidate concrete scenarios with stakeholders |
| (c) Concrete scenario | Copy the shape; replace values with the system's own numbers |
| (d) Tactic tree | Pick candidate tactics; in a review, ask which branches are unrepresented |
| (e) Code-level realization | Map a chosen tactic to the idiom in the codebase's ecosystem |
| (f) Tactic-absent signals | Grep/read for these; each maps to a smell ID (D/M/I/R/C/E codes) in [review-playbook.md](review-playbook.md), or says "no playbook smell; report directly" |
| (g) Measurement | The fitness function or SLI that proves the response measure holds |
| (h) Tradeoffs | Record as tradeoff points in the ADR or evaluation |
| (i) Patterns | Patterns SAiP discusses in the chapter; they bundle the tactics above |
| (j) Reviewer quick checks | Change-time cards in [quality-attributes-change.md](quality-attributes-change.md) only: yes/no questions for a PR or repository review. Change-time cards also merge (b)+(c) and (d)+(e) into one table each |

The Availability, Performance and Security cards also carry a **Choose** table between (d) and (e): use it to pick one tactic
for a given response measure instead of listing the whole tree.

### Using a card in a review
Do not walk the whole tactic tree. Check only the branches the scenario needs.
1. Get the concrete scenario and its response measure from the ADRs, SLOs or [design-workflow.md](design-workflow.md). If
   none exists, write one from (b) and mark it **assumed** in the report.
2. Mark the tactic branches in (d) that this scenario needs (use the Choose table where there is one).
3. Run the (f) signals as searches over the modules on the scenario's code path, not the whole repository.
4. For each tactic that is present, check that it is configured: timeouts and bounds set to values that fit the response
   measure, fallback exercised by a test, failover actually tested.
5. Record each missing or misconfigured tactic as a finding with the playbook smell ID and `file:line` evidence.
6. Record the (h) tradeoffs the fix introduces as tradeoff points ([evaluation-methods.md](evaluation-methods.md)).
7. Propose the (g) measurement as the fitness function that keeps the fix in place.

### Using a card in a design
1. Write the concrete scenario with a numeric response measure.
2. Pick tactics with the Choose table (or (d) where there is none); name the rejected alternatives.
3. In the ADR, name the (e) idiom for the project's stack, the (h) tradeoffs accepted, and the (g) fitness function.

## 1. Availability (4th ed. ch. 4; 3rd ed. ch. 5)

**(a) Definition.** Availability is the property of being there and ready to do the job when needed. It includes reliability (not
failing) and adds recovery (coming back after a failure). Keep the chain apart: a **fault** is the cause, an **error** is the
resulting incorrect internal state, a **failure** is a deviation from specified behaviour observable from outside. Tactics stop
faults from becoming failures, or bound and repair the damage.

### (b) General scenario
| Part | Allowed values |
|---|---|
| Source | Internal or external: people, hardware, software, physical infrastructure, physical environment |
| Stimulus | Fault: omission, crash, incorrect timing (early/late), incorrect response |
| Artifact | Processors, communication channels, storage, processes, affected artifacts in the environment |
| Environment | Normal operation, startup, shutdown, repair mode, degraded operation, overloaded operation |
| Response | Prevent the fault becoming a failure; detect it (log, notify); recover (disable the source, be temporarily unavailable, mask/repair, run degraded) |
| Response measure | Availability %, time to detect, time to repair, time in degraded mode, rate of faults prevented or handled |

### (c) Concrete scenario: payments API
| Part | Value |
|---|---|
| Source | Third-party card processor |
| Stimulus | Omission: processor stops answering authorization calls |
| Artifact | `payments-service` outbound processor client |
| Environment | Peak load, 400 req/s |
| Response | Calls time out; circuit opens; new payments are accepted as `PENDING` and queued; status page and on-call notified |
| Response measure | Detection within 10 s; zero payment requests lost; p99 API latency stays under 800 ms; queued payments settle within 15 min of processor recovery |

### (d) Tactic tree
- **Detect faults**
  - *Monitor*: a separate component watches the health of others.
  - *Ping/echo*: the monitor sends a request and expects a reply within a deadline (pull).
  - *Heartbeat*: the monitored component emits periodic liveness messages (push).
  - *Timestamp*: tag events with a clock or sequence number to detect ordering errors and loss.
  - *Condition monitoring*: check conditions or design assumptions (e.g. checksums).
  - *Sanity checking*: check that an output or operation is plausible for the domain.
  - *Voting*: compare results of redundant components (TMR is the classic form). Variants: replication (identical copies; catches
    hardware faults), functional redundancy (diverse implementations; catches design faults), analytic redundancy (diverse inputs
    and algorithms).
  - *Exception detection*: detect conditions that alter normal flow (system exceptions, parameter fences, parameter typing,
    timeouts).
  - *Self-test*: a component runs its own test procedures.
- **Recover from faults: preparation and repair**
  - *Redundant spare* (3rd ed.: active redundancy / passive redundancy / spare) [verify wording in SAiP 4th ed.]: hot spare
    processes the same inputs; warm spare receives periodic state updates; cold spare is out of service until needed. Hot is
    fastest to fail over and costs most.
  - *Rollback*: return to a saved good state (checkpoint) and continue.
  - *Exception handling*: once detected, mask, repair or report the exception.
  - *Software upgrade*: in-service upgrade without interrupting service.
  - *Retry*: repeat an operation that failed from a transient fault; bounded, with backoff.
  - *Ignore faulty behaviour*: discard messages from a source known to be spurious.
  - *Graceful degradation* (3rd ed.: degradation) [verify wording in SAiP 4th ed.]: keep critical functions, shed less critical
    ones.
  - *Reconfiguration*: reassign responsibilities to the resources still working.
- **Recover from faults: reintroduction**
  - *Shadow*: run the repaired component alongside, observe it, then promote it.
  - *State resynchronization*: bring a recovering component's state up to date.
  - *Escalating restart*: restart at the smallest granularity that works (thread, process, node), escalating only as needed.
  - *Non-stop forwarding*: split control plane from data plane; the data plane keeps forwarding on last known good state while the
    control plane recovers.
- **Prevent faults**
  - *Removal from service*: take a component down pre-emptively (e.g. scheduled restart for leaks).
  - *Transactions*: bundle state updates atomically so partial updates cannot corrupt state.
  - *Predictive model*: watch health indicators and act before a fault (queue approaching limit).
  - *Exception prevention*: make exceptions impossible (types, smart pointers, wrappers).
  - *Increase competence set*: handle more cases as normal operation, so fewer count as faults.

### Choose
| If the scenario says | Choose | Not |
|---|---|---|
| RTO seconds, no data loss | Hot spare / active-active replicas with synchronous or quorum replication | Restore from backup |
| RTO minutes, RPO seconds | Warm replica with automated failover | Manual failover runbook only |
| RTO hours, some data loss acceptable | Cold restore from backup, with a scheduled restore test | Paying for hot standby |
| A dependency may hang or fail | Timeout on every call first; add retry only for transient errors on idempotent operations | Retry without a timeout |
| Dependency is non-critical and a fallback exists | Circuit breaker + fallback (cached value, default, hidden feature) | Failing the whole request |
| Dependency is critical, no fallback | Timeout + fail fast, or accept as `PENDING` and queue | Blocking threads until it recovers |
| One slow dependency must not starve others | Bulkhead (separate bounded pool per dependency) | One shared pool |
| A write may be retried by client or middleware | Idempotency key enforced by a unique constraint | Read-then-write duplicate check |
| A state change must also emit an event | Transactional outbox | DB write then broker publish (R4) |

### (e) Code-level realization
| Tactic | Idiom (examples across ecosystems) |
|---|---|
| Monitor, heartbeat, ping/echo | Health endpoint split into **liveness** (process alive; no dependency checks) and **readiness** (can serve; checks DB/queues). Wire to Kubernetes `livenessProbe` / `readinessProbe`. Spring Boot Actuator exposes liveness/readiness health groups; in Go/Node/Python write a small handler. Never put downstream checks in liveness: a DB blip then restarts every pod. |
| Exception detection (timeout) | A deadline on **every** remote call. Go: `context.WithTimeout` and `http.Client{Timeout: ...}`. Kotlin: `withTimeout` works for suspending calls only (cancellation is cooperative, so a blocking JDBC call or OkHttp `execute()` inside it is not interrupted); also set client-level timeouts (Ktor `HttpTimeout` plugin, OkHttp `callTimeout`, JDBC `setQueryTimeout`). `withTimeout` throws `TimeoutCancellationException`, a subclass of `CancellationException`, so generic cancellation handling or `runCatching` can hide it: prefer `withTimeoutOrNull` or catch `TimeoutCancellationException` explicitly. JS: `AbortSignal.timeout(ms)` (version-dependent: Node 16.14+/17.3+ and current browsers); `fetch` has no overall timeout by default, and axios defaults to `timeout: 0` (none). Python: `httpx` timeouts (requests has no default timeout). Java: `HttpClient` request timeout, JDBC query timeout. |
| Retry | Bounded attempts, exponential backoff **plus jitter**, only for idempotent or idempotency-keyed operations, only on transient errors. Examples: resilience4j `Retry` (JVM), Polly (.NET; v7 policies vs v8 resilience pipelines, version-dependent), tenacity (Python: `stop_after_attempt`, `wait_random_exponential`), p-retry (JS), a hand-rolled loop in Go. |
| Ignore faulty behaviour + graceful degradation | **Circuit breaker** pattern (resilience4j `CircuitBreaker`, Polly, `sony/gobreaker` in Go, opossum in Node) with a fallback: cached value, default, `PENDING` state, feature hidden. |
| Graceful degradation (isolation) | **Bulkhead**: separate bounded pools per dependency. JVM: resilience4j `Bulkhead` or a dedicated bounded `ThreadPoolExecutor`. Kotlin: `Dispatchers.IO.limitedParallelism(n)` (version-dependent: experimental when introduced in kotlinx.coroutines 1.6). Go: buffered-channel semaphore or `golang.org/x/sync/semaphore`. Kotlin coroutines: a `SupervisorJob` (or `supervisorScope`) so one child's failure does not cancel its siblings; this isolates faults but restarts nothing. |
| Transactions + retry safety | **Idempotency keys**: client sends a unique key per logical operation (e.g. an `Idempotency-Key` header); server stores key and result and replays the result on duplicates. Enforce with a unique constraint, not a read-then-write. |
| Transactions across a DB and a broker | **Transactional outbox**: write the event to an outbox table in the same DB transaction as the state change; a relay (poller or CDC such as Debezium) publishes it. Consumers must dedupe. |
| Redundant spare | Multiple replicas behind a load balancer (hot); DB primary with streaming replica (warm); restore-from-backup runbook (cold). Test failover, not just configure it. |
| Rollback | Deployment rollback, DB point-in-time recovery, saga compensation for multi-service workflows. |
| Escalating restart | Restart the smallest unit first; escalate after N failures in T. Supervisor trees (Erlang/OTP, Akka) with restart intensity limits; process-manager restart policies (systemd `Restart=on-failure` with start limits, Docker `restart: on-failure`); Kubernetes container restart, then pod rescheduling to another node. |
| Predictive model | Alert on saturation trends (queue depth, disk, connection pool usage), not only on errors. |

### (f) Tactic-absent signals
| Signal in code | Missing tactic | Playbook smell |
|---|---|---|
| HTTP/gRPC/DB client created without a timeout or deadline | Exception detection (timeout) | R1 in [review-playbook.md](review-playbook.md) |
| `catch (e: Exception) {}`, `except: pass`, `_ = err`, empty `.catch(() => {})` | Exception detection / handling | C3 |
| Retry loop without cap, backoff or jitter; retry on non-idempotent POST | Retry done wrong | R1 |
| Synchronous chain of 3 or more remote hops on one request path | Degradation, reconfiguration; availability multiplies along the chain (§7) | R2 |
| One shared thread/connection pool for all downstreams | Bulkhead (graceful degradation) | R1 (its fix names the bulkhead) |
| DB write then broker publish in the same handler, no outbox | Transactions | R4 |
| Liveness probe that checks the database | Monitor misapplied | C4 (closest; say "liveness probe checks dependencies" in the finding) |
| Single instance, single AZ, no documented restore | Redundant spare | R7 |

- **(g) Measurement.** SLI: fraction of successful requests (or successful minutes) over a window; MTTR from incident data;
  failover time from a game-day or chaos test. Fitness function: a fault-injection test (Toxiproxy, a chaos tool, or a test double
  that hangs) asserting that the caller returns within its deadline and degrades as specified. See
  [fitness-functions.md](fitness-functions.md).
- **(h) Typical tradeoffs.** Redundancy costs money, energy and replica synchronization latency, and widens the attack surface.
  Shorter heartbeat periods detect faster but add load. Retries amplify outages (retry storms) unless capped and jittered.
  Transactions and checkpoints cost latency and storage. Strong consistency across replicas costs availability during partitions.
- **(i) Patterns in the chapter (approximate; verify in SAiP 4th ed.).** Redundant spares (active/passive redundancy), triple
  modular redundancy (TMR), circuit breaker, process pairs, forward error recovery.

## 2. Performance (4th ed. ch. 9; 3rd ed. ch. 8)

**(a) Definition.** Performance is about time: the system's ability to meet timing requirements as events arrive. Response time =
**processing time** (resources consumed) + **blocked time** (contention, waiting for a resource, waiting for another computation).
Tactics either reduce the work or make resources serve it better.

### (b) General scenario
| Part | Allowed values |
|---|---|
| Source | Internal or external |
| Stimulus | Arrival of a periodic, sporadic (unknown times, known minimum gap) or stochastic event |
| Artifact | The system or one or more components |
| Environment | Normal, peak load, overload, emergency |
| Response | Process events; possibly change the level of service |
| Response measure | Latency, deadline, throughput, jitter, miss rate, data loss |

### (c) Concrete scenario: mobile wallet
| Part | Value |
|---|---|
| Source | User |
| Stimulus | Opens the wallet home screen (stochastic) |
| Artifact | Balance and recent-transactions screen, local DB, sync client |
| Environment | Cold start, mid-range Android phone, 4G |
| Response | Render cached balance immediately, refresh in background, update in place |
| Response measure | First meaningful content within 1.0 s p90; fresh balance within 3 s p90; no frame over 32 ms during refresh |

### (d) Tactic tree
- **Control resource demand**
  - *Manage work requests* (3rd ed.: manage sampling rate; some summaries also say manage event arrival) [verify wording in SAiP
    4th ed.]: reduce or shape the arriving work, e.g. sample less often, rate-limit clients, admission control.
  - *Limit event response*: process events only up to a maximum rate; queue or drop the rest.
  - *Prioritize events*: serve important events first; ignore or delay low-priority ones.
  - *Reduce computational overhead* (3rd ed.: reduce overhead) [verify wording in SAiP 4th ed.]: remove intermediaries, co-locate
    communicating parts, cut serialization and hops.
  - *Bound execution times*: cap time per event (iteration limits, anytime algorithms).
  - *Increase efficiency of resource usage* (3rd ed.: increase resource efficiency) [verify wording in SAiP 4th ed.]: better
    algorithms and data structures on the critical path.
- **Manage resources**
  - *Increase resources*: faster or more CPUs, memory, network.
  - *Introduce concurrency*: parallelism and pipelining to cut blocked time.
  - *Maintain multiple copies of computations*: replicated servers or processes, usually behind a load balancer. This is not
    caching.
  - *Maintain multiple copies of data*: caching and data replication; the new problems are consistency and what to cache.
    Do not merge the two: copies of computations address processing capacity; copies of data address data-access latency
    and bring consistency problems.
  - *Bound queue sizes*: cap queued arrivals; requires an overflow policy (reject, drop, shed).
  - *Schedule resources*: a policy for contended resources: FIFO; fixed priority (semantic importance, deadline monotonic, rate
    monotonic); dynamic priority (round-robin, earliest deadline first, least slack); static scheduling (cyclic executive).

### Choose
Profile first: find whether the time goes to processing or to blocking (§2(a)) before picking a tactic.

| If the measurement shows | Choose | Not |
|---|---|---|
| Read-heavy, staleness of seconds or more tolerated | Cache with TTL, size bound and a written invalidation rule | Read replica (more cost, same staleness problem) |
| Read-heavy, data must be fresh | Read replica, or indexes and better queries on the primary | Cache without invalidation |
| CPU-bound and stateless | More copies of computation behind a load balancer | Bigger cache |
| Latency dominated by hops, serialization or per-item calls | Reduce overhead: batch endpoints, fix N+1, collapse the chain | More replicas |
| Load bursts beyond capacity | Rate limit at the edge plus bounded queue with explicit reject (429/503) | Unbounded queue |
| Some requests matter more than others under load | Prioritize events: separate queues or pools per class | One FIFO for all |

### (e) Code-level realization
| Tactic | Idiom |
|---|---|
| Manage work requests | Rate limiter at the edge (token bucket: Bucket4j, resilience4j `RateLimiter`, `golang.org/x/time/rate`, API gateway limits); debounce/throttle UI input; lower sensor sampling rates. |
| Limit event response + bound queue sizes | Bounded queues and channels (`ArrayBlockingQueue`, `Channel(capacity)`, buffered Go channel, `asyncio.Queue(maxsize=...)`) with explicit reject/drop; HTTP 429 or 503 with `Retry-After`; load shedding. Backpressure in streams (Reactive Streams, Kotlin `Flow` `buffer`/`conflate`). |
| Prioritize events | Separate queues or topics per priority; priority executors; critical traffic on its own pool. |
| Reduce computational overhead | Remove chatty call chains; batch endpoints; avoid per-row remote calls; binary protocols where measured to matter. |
| Increase efficiency of resource usage | Fix N+1 queries (join/`IN` query, DataLoader batching, JPA fetch joins, Django `select_related`/`prefetch_related`); indexes; streaming instead of loading whole payloads. |
| Bound execution times | Query timeouts, statement timeouts, per-request deadlines propagated (`context.Context`, gRPC deadlines). |
| Introduce concurrency | Async IO and structured concurrency (coroutines, `async`/`await`, goroutines with `errgroup`); parallel fan-out with a concurrency cap. Keep IO off the UI/main thread. |
| Multiple copies of computations | Stateless service replicas behind a load balancer; horizontal autoscaling. |
| Multiple copies of data | Cache with TTL and bounded size (Caffeine, Redis with eviction policy, `functools.lru_cache(maxsize=...)`); read replicas; CDN. Write down the invalidation rule next to the cache. |
| Schedule resources | Separate pools per workload class; job schedulers with priorities; Kubernetes requests/limits and priority classes. |

### (f) Tactic-absent signals
| Signal in code | Missing tactic | Playbook smell |
|---|---|---|
| Loop that issues one query or remote call per item | Increase efficiency / reduce overhead | I2 |
| `LinkedBlockingQueue()` with no capacity, `Channel(UNLIMITED)`, unbounded in-memory map used as cache | Bound queue sizes; bounded copies of data | R5 |
| Synchronous chain of 3 or more remote hops on one request path | Reduce computational overhead | R2 (I2 if the cause is chattiness) |
| Network, disk or DB access on the Android main thread, iOS main queue, or a JS event-loop hot path with sync IO | Introduce concurrency | R3 |
| No rate limit on public endpoints | Manage work requests | R6 |
| Cache with no TTL, eviction or invalidation rule | Multiple copies of data (misapplied) | R5 |
| Remote call without deadline | Bound execution times | R1 |

- **(g) Measurement.** Measure percentiles (p50, p95, p99, p99.9), never averages alone. Fitness functions: load test in CI or
  nightly (k6, Gatling, Locust, JMeter) asserting a percentile threshold at a stated arrival rate; query-count assertions in
  integration tests to catch N+1; Android Macrobenchmark or iOS XCTest metrics for startup and frame timing. See
  [fitness-functions.md](fitness-functions.md).
- **(h) Typical tradeoffs.** Reduce computational overhead vs modifiability (fewer intermediaries means tighter coupling). Caches
  and replicas vs consistency and memory. Concurrency vs testability (nondeterminism) and correctness (races). Increase resources
  vs cost and energy. Limiting events and bounding queues drops work, which may violate availability or usability scenarios;
  choose the rejection behaviour deliberately.
- **(i) Patterns in the chapter (approximate; verify in SAiP 4th ed.).** Service mesh, load balancer, throttling, map-reduce.

## 3. Security (4th ed. ch. 11; 3rd ed. ch. 9)

**(a) Definition.** Security is the ability to protect data and services from unauthorized access while still serving authorized
parties. Core properties: **confidentiality**, **integrity**, **availability** (CIA). Supporting properties: **authentication**,
**authorization**, **nonrepudiation**. The tactic tree follows the physical-security analogy: detect, resist, react, recover.

### (b) General scenario
| Part | Allowed values |
|---|---|
| Source | Human or system; identified (correctly or not) or unknown; internal or external |
| Stimulus | Unauthorized attempt to read data, change or delete data, access services, change behaviour, or reduce availability |
| Artifact | System services, data at rest, data in transit, components, resources |
| Environment | Online/offline, connected/disconnected, behind a firewall or open, fully/partly/not operational |
| Response | Data and services protected; parties identified with assurance; actions nonrepudiable; resources available to legitimate users; activity tracked; attacks detected and reported |
| Response measure | Extent of compromise, time to detect, attacks resisted, time to recover, data exposed |

### (c) Concrete scenario: IoT gateway
| Part | Value |
|---|---|
| Source | Attacker on the site LAN with a cloned device identity |
| Stimulus | Sends forged telemetry and a firmware-update command to the gateway |
| Artifact | Gateway ingest endpoint and device-command handler |
| Environment | Online, normal operation |
| Response | mTLS rejects the unregistered certificate; commands require a signed payload checked against the fleet key; rejection is audited and raised as an alert |
| Response measure | 100% of forged messages rejected in the security test suite; alert raised within 60 s; no command executes without a valid signature |

### (d) Tactic tree
- **Detect attacks**
  - *Detect intrusion*: compare traffic or request patterns to known malicious signatures.
  - *Detect service denial*: compare traffic to known denial-of-service profiles.
  - *Verify message integrity*: checksums, hashes, MACs, signatures.
  - *Detect message delivery anomalies* (3rd ed.: detect message delay) [verify wording in SAiP 4th ed.]: abnormal delivery timing
    or patterns can indicate man-in-the-middle or replay.
- **Resist attacks**
  - *Identify actors*: identify the source of every external input (user ID, key, IP, port).
  - *Authenticate actors*: verify the claimed identity (passwords, certificates, MFA).
  - *Authorize actors*: verify rights to the specific data or service.
  - *Limit access*: control what can reach which resources (firewalls, DMZ, single entry point).
  - *Limit exposure*: reduce the attack surface: fewer services and entry points per host.
  - *Encrypt data*: at rest and in transit.
  - *Separate entities*: physical or virtual separation of sensitive data and processes.
  - *Validate input* (new in 4th ed.) [verify wording in SAiP 4th ed.]: check all input for syntax and semantic validity before
    use.
  - *Change credential settings* (3rd ed.: change default settings) [verify wording in SAiP 4th ed.]: force replacement of default
    credentials and settings.
- **React to attacks**
  - *Revoke access*: restrict access, even for normally legitimate users, while under attack.
  - *Restrict login* (3rd ed.: lock computer) [verify wording in SAiP 4th ed.]: limit attempts, lock out or slow down after
    repeated failures.
  - *Inform actors*: notify operators and cooperating systems.
- **Recover from attacks**
  - *Audit* (3rd ed.: maintain audit trail) [verify wording in SAiP 4th ed.]: record actions and effects to trace attackers and
    support recovery.
  - *Nonrepudiation* (new in 4th ed.) [verify wording in SAiP 4th ed.]: ensure a sender cannot deny sending and a recipient cannot
    deny receiving (signatures, trusted receipts).
  - Restore state using the availability recovery tactics (rollback, redundant spare).

### Choose
| If the threat or scenario is | Choose first | Then |
|---|---|---|
| Users reading or changing other users' data | Authorize actors per object in the service layer | Audit on sensitive actions |
| Forged callers or tokens | Authenticate actors: verified tokens in middleware; mTLS between services | Revoke access (short token lifetimes, revocation list) |
| Injection or malformed input | Validate input at the boundary into typed values; parameterized queries | Detect intrusion (WAF) as a second layer, not the first |
| Credential stuffing or brute force | Restrict login: per-account and per-IP limits, progressive delay | MFA; inform actors |
| Interception or tampering in transit | Encrypt data (TLS with verification on); verify message integrity (signatures, HMAC with timestamp) | Nonrepudiation where disputes matter |
| Large blast radius if one part is breached | Limit exposure and separate entities (separate admin surface, per-tenant isolation, least-privilege IAM) | Audit |

### (e) Code-level realization
| Tactic | Idiom |
|---|---|
| Authenticate actors | Verify tokens server-side in middleware (JWT signature, issuer, audience, expiry); OIDC library rather than hand-rolled parsing; mTLS between services. |
| Authorize actors | Server-side authorization at the boundary **and** per resource: middleware plus policy objects or a policy engine (Spring Security method security, ASP.NET Core policies, OPA, Casbin, a `Policy.canEdit(user, doc)` function). Object-level checks on every lookup by ID. |
| Validate input | Schema validation at the boundary, parse into typed domain values: zod (TS), pydantic (Python), Bean Validation / Jakarta Validation `@Valid` (JVM), `go-playground/validator` (Go). Parameterized queries always. |
| Limit exposure / limit access | Minimal public routes; admin and debug endpoints on a separate port or network; deny-by-default network policies; least-privilege IAM roles. |
| Encrypt data | TLS everywhere; field-level encryption for sensitive columns; platform keystores for mobile keys (Android Keystore, iOS Keychain). |
| Change credential settings / secrets | Secrets out of code and images: environment injection from a secret manager (Vault, cloud secret managers, Kubernetes Secrets with encryption at rest); secret scanning in CI (gitleaks, trufflehog). No default passwords in shipped config. |
| Restrict login / detect service denial | Rate-limit auth endpoints per account and per IP; progressive delays; CAPTCHA only as a last resort. |
| Verify message integrity / nonrepudiation | Signed webhooks (HMAC with timestamp to block replay); signed commands; append-only audit log with actor, action, target, time, request ID. |
| Audit | Structured security events emitted from the domain layer, not only from the UI; retention and access control on the log itself. |
| Separate entities | Separate databases or schemas per trust level; tenant isolation enforced in the data-access layer, not in callers. |

### (f) Tactic-absent signals
| Signal in code | Missing tactic | Playbook smell |
|---|---|---|
| Permission checks only in the frontend (hidden buttons, route guards) with no server check | Authorize actors | C1 |
| Handler loads an entity by ID from the request without checking ownership | Authorize actors (object level) | C1 |
| String-concatenated SQL/shell/HTML; request bodies used without a schema | Validate input | C7 |
| API keys, passwords or private keys in source, config committed to git, or mobile binaries | Change credential settings; limit exposure | C2 |
| No rate limit on login, OTP or password-reset | Restrict login | R6 |
| Admin, actuator or debug endpoints on the public listener | Limit exposure | C8 |
| `verify=False` (requests/httpx), `InsecureSkipVerify: true` (Go `tls.Config`), trust-all `X509TrustManager` or `HostnameVerifier` returning `true` (JVM/Android), `rejectUnauthorized: false` or `NODE_TLS_REJECT_UNAUTHORIZED=0` (Node) | Encrypt data | C9 |
| JWT decoded without signature or audience check: PyJWT `options={"verify_signature": False}`, `jwt.decode` in jsonwebtoken where `jwt.verify` is needed, `alg: none` accepted | Authenticate actors | C9 |
| Security-relevant actions not logged, or logs contain tokens and PII | Audit | C4 (closest; name it an audit gap in the finding) |

- **(g) Measurement.** Fitness functions: authz tests per endpoint (every route has a test asserting 401/403 for the wrong actor);
  dependency and secret scanning in CI; DAST against a staging deployment; an architecture test asserting that all controllers go
  through the auth middleware or carry an authorization annotation. Measures: time to detect, time to revoke a credential, share
  of endpoints with authz tests.
- **(h) Typical tradeoffs.** Encryption, authentication and auditing cost latency, CPU and battery. Lockouts and extra login steps
  cost usability; revoking access costs availability for legitimate users. Separation and limited exposure raise cost and
  operational complexity. Audit trails raise privacy (GDPR) questions about what is logged and for how long.
- **(i) Patterns in the chapter (approximate; verify in SAiP 4th ed.).** Intercepting validator, intrusion prevention system.

### Practitioner cross-references (outside SAiP)
- **STRIDE** (Microsoft threat-modelling mnemonic: spoofing, tampering, repudiation, information disclosure, denial of service,
  elevation of privilege): map each threat to a tactic group, e.g. spoofing to authenticate actors, repudiation to nonrepudiation.
- **OWASP Top 10 / ASVS**: seed concrete scenarios from them; categories change between editions, so cite the edition year.

## 4. Safety (4th ed. ch. 10; 3rd ed. brief entry in ch. 12)

**(a) Definition.** Safety is the ability to avoid entering states that cause or lead to damage, injury or loss of life to actors
in the environment, and to recover from or limit the damage of such states. It differs from security (no malicious actor is
assumed) and from availability (a stopped system can be safe; a running one can be unsafe). It applies wherever software actuates
the physical world: industrial IoT, medical, automotive. Practitioner extension (not SAiP): the same containment tactics
(limits, barriers, fail-safe defaults) apply to irreversible financial operations; do not cite SAiP for that use.

### (b) General scenario [verify wording in SAiP 4th ed.]
| Part | Allowed values |
|---|---|
| Source | Data source (sensor, computing component), time source, user or operator |
| Stimulus | Omission (value never arrives), commission (wrong value or action), incorrect data, timing error (too early or late) |
| Artifact | The parts of the system that affect safety-critical behaviour |
| Environment | Normal, degraded, manual, recovery mode |
| Response | Stay in or return to a safe state; continue degraded; shut down; switch to manual or backup; notify; log |
| Response measure | Share of unsafe-state entries avoided or recovered, time to reach a safe state, change in risk exposure, time in degraded mode |

### (c) Concrete scenario: IoT gateway controlling a heater
| Part | Value |
|---|---|
| Source | Temperature sensor |
| Stimulus | Commission: sensor reports 20 °C while the redundant sensor reports 85 °C |
| Artifact | Gateway control loop |
| Environment | Normal operation, heater on |
| Response | Comparison detects disagreement; control loop cuts heater output; alarm raised; hardware thermal cutoff remains as a barrier |
| Response measure | Heater off within 2 s of disagreement exceeding 5 °C; 100% of injected disagreement faults handled in hardware-in-the-loop tests |

### (d) Tactic tree [verify wording in SAiP 4th ed.]
- **Unsafe state avoidance**: substitution (use a simpler or safer mechanism, e.g. a hardware interlock instead of software);
  predictive model.
- **Unsafe state detection**: timeout; timestamp; condition monitoring; sanity checking; comparison (of redundant outputs).
- **Containment**
  - *Redundancy*: replication; functional redundancy; analytic redundancy.
  - *Limit consequences*: abort; degradation (the chapter's exact name for this is unverified).
  - *Barrier*: firewall; interlock.
- **Recovery**: rollback; repair state; reconfiguration.

Many safety tactics reuse availability tactics. The goal differs: availability keeps the service running; safety keeps the system
harmless, even if that means stopping it.

### (e) Code-level realization
| Tactic | Idiom |
|---|---|
| Sanity checking / condition monitoring | Range and rate-of-change checks on every sensor reading, as typed value objects that cannot hold out-of-range values. |
| Timeout / timestamp | Reject stale readings by timestamp; watchdog timer that forces the safe state if the control loop stops ticking. |
| Comparison / redundancy | Read two or three independent sensors; vote or compare before actuating. |
| Barrier (interlock) | Actuator commands go through one guard module that enforces invariants; nothing else may call the driver (enforce with an architecture test). Keep a hardware or firmware limit as the final barrier. |
| Limit consequences | Fail-safe defaults: on unknown state, actuators go to off/closed. Practitioner extension for money flows (not SAiP): per-transaction and per-day limits, two-person approval above a threshold. |
| Recovery | Explicit state machine with a named safe state and tested transitions into it. |

- **(f) Tactic-absent signals.** Actuator or irreversible-operation calls reachable from many modules, i.e. no single guard
  (C10); sensor values used without range or staleness checks (C7); exceptions in the control loop that are logged and ignored
  (C3); no defined safe state in the state machine (C10).
- **(g) Measurement.** Fault-injection tests that drive each hazard's stimulus and assert time-to-safe-state; architecture test
  that only the guard module depends on actuator drivers. For regulated domains, defer to the applicable standard (e.g. IEC 61508,
  ISO 26262, IEC 62304) and say that the skill does not replace its process.
- **(h) Typical tradeoffs.** Safety vs availability (shutting down is safe but unavailable); redundancy vs cost, weight and
  energy; conservative limits vs usability and throughput.
- **(i) Patterns in the chapter (approximate; verify in SAiP 4th ed.).** Monitor-actuator, separated safety, redundant sensors,
  design assurance levels.

## 5. Energy efficiency (4th ed. ch. 6; not in 3rd ed.)

**(a) Definition.** Energy efficiency is how well the system minimizes energy consumption while still delivering its function and
meeting its other QA requirements. It dominates on battery-powered mobile and embedded devices and matters at data-centre scale.
The main tensions are with performance and availability (redundancy and polling cost power).

### (b) General scenario [verify wording in SAiP 4th ed.]
| Part | Allowed values |
|---|---|
| Source | End user, manager, system administrator, automated agent |
| Stimulus | Request or constraint to conserve energy (e.g. low battery, power budget) |
| Artifact | Devices, servers, VMs, clusters |
| Environment | Runtime; connected or disconnected; battery-powered; low-power mode; fixed budget |
| Response | Disable services, deallocate resources, change allocation, lower-power mode, reduce quality of service |
| Response measure | Energy consumed or saved (J, kWh, % battery, battery life), with latency, throughput or accuracy kept within a bound |

### (c) Concrete scenario: mobile wallet background sync
| Part | Value |
|---|---|
| Source | OS (device enters battery saver) |
| Stimulus | Low-power mode while the wallet has pending sync work |
| Artifact | Background sync and notification components |
| Environment | Android and iOS, screen off, metered network |
| Response | Defer non-urgent sync to charging and unmetered network; keep push-driven payment notifications |
| Response measure | Wallet under 1% of battery per 24 h idle in platform battery stats; incoming payment notified within 60 s |

**(d) Tactic tree [verify wording in SAiP 4th ed.].** Group names as reported in some summaries: resource monitoring, resource
allocation, resource demand reduction; others phrase them as monitor resources, allocate resources, reduce resource demand.

- **Resource monitoring**: metering (measure consumption directly); static classification (use known power figures per device);
  dynamic classification (estimate from a runtime model).
- **Resource allocation**: reduce usage (power down or throttle idle devices, lower CPU frequency); discovery (find the most
  energy-efficient resource); schedule resources (place work where it costs least energy).
- **Resource demand reduction**: largely mirrors performance's control resource demand: manage work requests, limit event
  response, prioritize events, reduce computational overhead, bound execution times, increase efficiency of resource usage. Exact
  list unverified.

### (e) Code-level realization
| Tactic | Idiom |
|---|---|
| Schedule resources / reduce usage | Deferrable work through the OS scheduler: Android WorkManager with constraints (charging, unmetered network, battery not low); iOS `BGTaskScheduler` (`BGAppRefreshTaskRequest`, `BGProcessingTaskRequest` with `requiresExternalPower`). Server side: batch jobs in off-peak or low-carbon windows. |
| Manage work requests | Replace polling with push (FCM/APNs, WebSocket, server-sent events); exponential backoff on idle. |
| Limit event response / batching | Batch sensor reads and network calls; coalesce writes; request location at the coarsest accuracy that meets the need. |
| Reduce computational overhead | Compress payloads; avoid redundant parsing; cache results that are expensive to recompute. |
| Resource monitoring | Android: Android Studio Power Profiler (version-dependent: it replaced the Energy Profiler in recent Android Studio releases and needs a supported device), `adb shell dumpsys batterystats`; Battery Historian is legacy and unmaintained. iOS: Xcode Energy gauge, MetricKit. Cloud: cost and utilization per service as a proxy. |

- **(f) Tactic-absent signals.** All map to R8 unless noted. Search:
  - polling: `rg -n 'while \(true\)' -A5 | rg 'delay\('` in repositories and services; `Timer.scheduledTimer` or
    `Handler.postDelayed` loops in background code;
  - unreleased wakeups: `rg -n 'newWakeLock|\.acquire\('` with no matching `release(` and no timeout argument;
    `requestLocationUpdates` with no `removeLocationUpdates`;
  - raw scheduling: `AlarmManager.setRepeating` or custom timers for deferrable work instead of WorkManager/`BGTaskScheduler`;
  - one network call per sensor sample instead of batches.
- **(g) Measurement.** Battery drain per hour in a scripted scenario on a reference device; wakeups per hour; bytes transferred
  per session; for servers, energy or CPU-seconds per request.
- **(h) Typical tradeoffs.** Deferred sync vs freshness (performance, usability); fewer replicas vs availability; coarser sampling
  vs accuracy and safety.
- **(i) Patterns in the chapter (approximate; verify in SAiP 4th ed.).** Sensor fusion, kill abnormal tasks, power monitor.
  These are patterns, not tactics; batching and throttling in (e) are idioms for the demand-reduction tactics.

## 6. Usability (4th ed. ch. 13; 3rd ed. ch. 11)

**(a) Definition.** Usability is how easy it is for users to accomplish a task and what support the system gives them. It covers
learning, using efficiently, minimizing the impact of errors, adapting the system, and increasing confidence and satisfaction.
Most usability is UI design; the architectural part is **enabling** cancel, undo, progress and adaptation, which needs the right
separation and state handling underneath.

### (b) General scenario
| Part | Allowed values |
|---|---|
| Source | End user, possibly in a specialized role |
| Stimulus | User tries to use the system efficiently, learn it, minimize error impact, adapt or configure it |
| Artifact | The system or the part the user interacts with |
| Environment | Runtime or configuration time |
| Response | Provide the needed features or anticipate the user's needs |
| Response measure | Task time, error count, tasks completed, satisfaction, knowledge gained, success ratio, time or data lost on error |

### (c) Concrete scenario: mobile wallet send flow
| Part | Value |
|---|---|
| Source | Wallet user |
| Stimulus | Starts a payment to the wrong contact and wants to stop it |
| Artifact | Send flow and transaction submission |
| Environment | Runtime, payment submitted but not yet broadcast |
| Response | Cancel is visible until broadcast; cancellation stops signing and network work end to end; draft restored |
| Response measure | Cancel honoured within 300 ms in 100% of tests before broadcast; no orphan transaction; draft data not lost |

### (d) Tactic tree
- **Support user initiative**
  - *Cancel*: listen for the request, stop the activity, free resources, notify collaborators.
  - *Undo*: keep state or reversible operations to return to an earlier state.
  - *Pause/resume*: suspend a long operation, free resources, continue later.
  - *Aggregate*: apply one operation to a group of objects.
- **Support system initiative**
  - *Maintain task model*: know what the user is doing, to give context-sensitive help.
  - *Maintain user model*: know the user's expertise and preferences, to adapt help and defaults.
  - *Maintain system model*: know the system's own expected behaviour, to show progress and time remaining.

### (e) Code-level realization and architectural implications
| Tactic | What the architecture must provide | Idiom |
|---|---|---|
| Undo | Operations as objects with inverses, or snapshots of state | **Command** pattern with `execute`/`undo` on a history stack; **Memento** for snapshots; event sourcing for domain-level undo; soft delete with a restore window. |
| Cancel | Cancellation propagated from UI to the deepest IO call, not just a hidden spinner | Kotlin structured concurrency (cancel the scope's `Job`; suspend functions cooperate); JS `AbortController` passed to `fetch` and downstream; Go `context.Context` threaded through every call; .NET `CancellationToken`; Python asyncio task cancellation (`TaskGroup` is version-dependent: 3.11+); Swift `Task` cancellation. Server must also treat abandoned requests as cancelled. |
| Pause/resume | Resumable, checkpointed operations | Resumable uploads (chunked with offsets), WorkManager/`BGTaskScheduler` for continuation, persisted job state. |
| Aggregate | Batch APIs in the domain and backend | Bulk endpoints with per-item results, not N calls from the client. |
| Maintain system model | Progress events from long operations | Progress callbacks or `Flow`/observable streams; server-sent progress; honest estimates. |
| Separation enabling all of the above | UI isolated from domain state | MVC/MVVM/MVI; **Observer** (reactive state) so the UI reflects model changes without polling. |

- **(f) Tactic-absent signals.**
  - A cancel button that only hides UI while the coroutine, promise or request continues: `fetch(` with no `signal`,
    `GlobalScope.launch`, or a `Job` that the cancel handler never cancels (R3).
  - Network or disk IO on the main thread causing jank (R3).
  - Destructive actions (`delete`, `remove`, hard `DELETE` endpoints) with no undo, confirmation or restore window: no
    playbook smell; report directly.
  - Long operations (uploads, sync, exports) that expose no progress callback or stream: no playbook smell; report directly.
- **(g) Measurement.** Task time and error rate from usability tests; instrumentation of abandonment and retry rates; automated
  tests that cancel mid-operation and assert no side effects.
- **(h) Typical tradeoffs.** Undo and pause/resume need history (memory, complexity, performance). User models raise privacy and
  security concerns. MVC-style separation also helps modifiability.
- **(i) Patterns in the chapter (approximate; verify in SAiP 4th ed.).** Model-view-controller, observer, memento.

## 7. Availability arithmetic and SLOs

Steady-state availability: **A = MTBF / (MTBF + MTTR)**, with MTBF the mean time between failures and MTTR the mean time to
repair. Two levers: raise MTBF (prevent and mask faults) or cut MTTR (detect and recover faster). MTBF 1000 h with MTTR 1 h gives
about 99.9%; halving MTTR gives about 99.95%, the same as doubling MTBF and usually far cheaper. Say in the scenario whether
scheduled downtime counts. Serial dependencies multiply: a request that needs five services each at 99.9% succeeds at most about
99.5% of the time. This is why long synchronous chains are an availability smell.

| Availability | Downtime per year (365 d) | Per 30 days |
|---|---|---|
| 99% | 3 d 15 h 36 min | 7 h 12 min |
| 99.9% | 8 h 45 min 36 s | 43 min 12 s |
| 99.95% | 4 h 22 min 48 s | 21 min 36 s |
| 99.99% | 52 min 33.6 s | 4 min 19.2 s |
| 99.999% | 5 min 15.4 s | 25.9 s |

### Translating a scenario into an SLO and error budget
1. **SLI**: pick a measurable indicator matching the response measure, e.g. share of payment requests answered with non-5xx within
   800 ms.
2. **SLO**: target over a rolling window, e.g. 99.9% over 30 days.
3. **Error budget**: 1 minus SLO. At 99.9% over 30 days: 0.1% of requests, or 43 min 12 s of full outage equivalent.
4. **Policy**: what happens when the budget burns fast (freeze risky releases, prioritize reliability work). Alert on burn rate,
   not on single errors.
5. Record the SLO in the ADR that chose the tactics, and link the dashboard or alert rule as the fitness function.

## 8. Latency vs throughput vs jitter

| Measure | Meaning | Report as | Typical tactic lever |
|---|---|---|---|
| Latency | Time from stimulus to response for one event | Percentiles (p50/p95/p99) at a stated load | Reduce overhead, concurrency, caching, bound execution times |
| Throughput | Events completed per unit time | Sustained rate at which latency SLO still holds | Increase resources, multiple copies of computations, batching |
| Jitter | Variation in latency between events | Spread (e.g. p99 minus p50) or standard deviation | Scheduling, bounded queues, avoiding GC and contention spikes |
| Miss rate / data loss | Share of deadlines missed or events dropped | Percent at a stated load | Prioritize events, bound queue sizes with an explicit policy |

Checks before accepting a performance claim:
- [ ] Latency is quoted with a percentile **and** the arrival rate it was measured at.
- [ ] Throughput is quoted as the rate at which the latency target still holds, not the maximum before collapse.
- [ ] Averages are not used alone (a good mean hides a bad p99), and batching did not break a latency or jitter target.
- [ ] The measure lives in a repeatable test or production SLI, not a one-off benchmark.

*Sources: Bass, Clements & Kazman, Software Architecture in Practice, 4th ed., Addison-Wesley, 2021 (chapter titles from the
publisher's table of contents); 3rd ed., 2013. Wikipendium TDT4240 compendium,
https://www.wikipendium.no/TDT4240_Software_Architecture (CC BY-SA 3.0), paraphrased and corrected. Full attribution in
[../CREDITS.md](../CREDITS.md).*
