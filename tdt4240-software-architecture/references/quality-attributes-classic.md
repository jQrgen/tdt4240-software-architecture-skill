# Quality attributes: general scenarios and full tactic trees (3rd-ed. baseline)

This file is the detailed reference for the seven classic quality attributes (QAs) in Bass, Clements & Kazman, *Software Architecture in Practice* (SAiP). The last section covers the "other QAs" chapter. The tactic lists follow the **3rd edition (2013)**, which is the version that past TDT4240 exams (2015, 2016) and the Wikipendium compendium cite. The **4th edition (2021)** is the current one. Where it renames or regroups a tactic, this file marks it inline as *(4th ed.: X; verify)*. Those 4th-ed. names come from recall rather than a checked copy of the book, so tell students to check them against their own copy.

Related files, not repeated here:
- `foundations.md`: definitions of architecture, structures and views, and the tactic vs pattern distinction.
- `requirements-and-design.md`: ASRs, QAW, utility tree, ADD, and how to write concrete scenarios for the project.
- `quality-attributes-4th-edition.md`: the new 4th-ed. QA chapters (deployability, energy efficiency, integrability, safety) and the renumbering.
- `architectural-patterns.md`: which patterns bundle which tactics.
- `evaluation.md`: ATAM, sensitivity and tradeoff points.
- `exam-prep.md`: drill questions.

> Parts of this file are adapted from the Wikipendium TDT4240 compendium (CC BY-SA 3.0), https://www.wikipendium.no/TDT4240_Software_Architecture, by its contributors. The text has been paraphrased, restructured, corrected and extended with material from SAiP.

## Table of contents

0. [How to use this file, and chapter map](#0-how-to-use-this-file-and-chapter-map)
1. [Availability](#1-availability)
2. [Interoperability (4th ed.: Integrability)](#2-interoperability-4th-ed-integrability)
3. [Modifiability](#3-modifiability)
4. [Performance](#4-performance)
5. [Security](#5-security)
6. [Testability](#6-testability)
7. [Usability](#7-usability)
8. [Other quality attributes](#8-other-quality-attributes)
9. [Cross-QA tradeoff matrix and exam checklist](#9-cross-qa-tradeoff-matrix-and-exam-checklist)

---

## 0. How to use this file, and chapter map

### Chapter map (3rd vs 4th edition)

| QA | 3rd ed. (2013) chapter | 4th ed. (2021) chapter |
|---|---|---|
| Understanding QAs (scenarios, tactics in general) | 4 | 3 |
| Availability | 5 | 4 |
| Interoperability | 6 | replaced by **Integrability**, ch. 7 |
| Modifiability | 7 | 8 |
| Performance | 8 | 9 |
| Security | 9 | 11 |
| Testability | 10 | 12 |
| Usability | 11 | 13 |
| Other QAs | 12 | 14 ("Working with Other Quality Attributes") |
| Deployability / Energy efficiency / Safety | (ch. 12, briefly, or absent) | 5 / 6 / 10 |

Nobody has publicly confirmed which edition TDT4240 uses today. The reading list is in Leganto, which is not publicly visible. Give both chapter numbers when you cite one.

### Six-part scenario vocabulary (same in both editions)

| Part | Question it answers |
|---|---|
| **Source (of stimulus)** | Who or what generates the stimulus? |
| **Stimulus** | What event or condition arrives? |
| **Artifact** | Which part of the system is stimulated? |
| **Environment** | What state or mode is the system in when the stimulus arrives? |
| **Response** | What does the system do? |
| **Response measure** | How is the response measured? It must be a number with a unit and a threshold. |

- A **general scenario** is system-independent. It lists the *allowed values* for each part.
- A **concrete scenario** picks one value per part for one specific system. It is what goes in the requirements document.
- A **tactic** is a design decision that influences the response of *one* QA. A **pattern** bundles several tactics and usually affects several QAs.

### Answer template for "explain QA X" questions

Give: (1) the definition, (2) the six-part general scenario, (3) one concrete scenario, (4) the top-level tactic groups and then the tactics, (5) the main tradeoffs.

---

## 1. Availability

3rd ed. ch. 5 / 4th ed. ch. 4. Adapted in part from the Wikipendium compendium (CC BY-SA 3.0).

### Definition
Availability is the property that the software is **there and ready to carry out its task when you need it**. It includes reliability (not failing) and adds **recovery** (coming back after failing). The related umbrella term is **dependability** (Avizienis et al.): the ability to avoid failures that are more frequent or more severe than is acceptable.

Key vocabulary:
- A **fault** is the cause.
- An **error** is the intermediate incorrect state.
- A **failure** is a deviation from specified behaviour that someone outside the system can observe.

Availability tactics aim to stop faults from becoming failures, or to bound the damage and repair it.

### The formula and the nines

Steady-state availability:

```
A = MTBF / (MTBF + MTTR)
```

- MTBF is mean time between failures.
- MTTR is mean time to repair.
- The formula shows two levers. You can increase MTBF (prevent and mask faults) or decrease MTTR (detect and recover faster). Most tactics work on MTTR.
- Scheduled downtime may or may not count, so the scenario must say which.

| Availability | Downtime per year (365 d) | Downtime per 90 days |
|---|---|---|
| 99.0% | 3 d 15 h 36 min (87.6 h) | 21 h 36 min |
| 99.9% ("three nines") | 8 h 45 min 36 s | 2 h 9 min 36 s |
| 99.99% | 52 min 34 s | 12 min 58 s |
| **99.999% ("five nines")** | **5 min 15 s** | **1 min 18 s** |
| 99.9999% | 31.5 s | 7.8 s |

Worked example: MTBF = 1000 h and MTTR = 1 h gives A = 1000/1001, about 99.9%. Halving MTTR to 0.5 h gives about 99.95%. That is the same gain as doubling MTBF, and usually much cheaper.

### General scenario (3rd ed.)

| Part | Allowed values |
|---|---|
| Source | Internal or external: people, hardware, software, physical infrastructure, physical environment |
| Stimulus | A fault: omission (no response), crash, incorrect timing (early or late), incorrect response |
| Artifact | Processors, communication channels, persistent storage, processes |
| Environment | Normal operation, startup, shutdown, repair mode, degraded operation, overloaded operation |
| Response | Prevent the fault from becoming a failure. Detect it (log it, notify operators or other systems). Recover: disable the source of the fault events, be temporarily unavailable while repairing, fix or mask the fault, or run in degraded mode while repairing. |
| Response measure | Time or interval the system must be available; availability %; time to detect the fault; time to repair; time or interval in degraded mode; proportion or rate of faults prevented or handled |

**Wikipendium note.** The compendium's "general scenario" for availability ("a server disk fails in normal operation; a backup server takes over; normal operation within 30 s") is really a **concrete** scenario, because it fixes a single value for each part. Use it to explain the difference between general and concrete scenarios.

### Concrete game-project scenario

| Part | Value |
|---|---|
| Source | Firebase Realtime Database (external service) |
| Stimulus | Omission: the backend stops responding in the middle of a multiplayer match |
| Artifact | Client-side network/sync module of the Android libGDX game |
| Environment | Normal operation, 2 players in an active match |
| Response | Detect the loss within the timeout, show a "reconnecting" overlay, keep the local game state, retry with backoff, and resync when the connection returns |
| Response measure | Loss detected within 5 s; match resumes without state loss if connectivity returns within 60 s; the app does not crash in 100% of test runs |

### Tactic tree (3rd ed.)

- **Detect faults**
  - *Ping/echo*: send an asynchronous request and expect a reply within a deadline. It checks reachability and round-trip time.
  - *Monitor*: a separate component watches the health of other parts (processors, processes, I/O, memory).
  - *Heartbeat*: the monitored component itself sends periodic "I'm alive" messages. If they stop, the monitor assumes a fault. The message can also carry data.
  - *Timestamp*: tag events or messages with a local clock or sequence number, to detect wrong ordering or lost messages.
  - *Sanity checking*: check that an operation or output is plausible, based on knowledge of the internal design or the domain.
  - *Condition monitoring*: check conditions in a process or device, or validate design assumptions, for example with checksums.
  - *Voting*: several components compute the same thing and a voter compares the results. The common form is triple modular redundancy (TMR). It has three variants:
    - *Replication*: identical copies. This catches hardware faults but not design bugs.
    - *Functional redundancy*: different implementations of the same function (design diversity). This catches common-mode design faults.
    - *Analytic redundancy*: different inputs or sensors and different algorithms, whose results are checked against each other.
  - *Exception detection*: detect a condition that changes normal flow. Examples are system exceptions, parameter fences (a known pattern next to data, to detect overruns), parameter typing, and timeouts.
  - *Self-test*: a component runs its own test procedures.
- **Recover from faults: preparation and repair**
  - *Active redundancy (hot spare)*: all replicas process the same input in parallel, so a spare is already in sync and fail-over takes milliseconds. *(4th ed.: merged into "Redundant spare"; verify)*
  - *Passive redundancy (warm spare)*: only the active node processes input and sends periodic state updates to the spares. Fail-over requires catching up. *(4th ed.: "Redundant spare"; verify)*
  - *Spare (cold spare)*: a spare that is out of service until it is needed. It must be powered on, reset and loaded with state, so recovery is slow. *(4th ed.: "Redundant spare"; verify)*
  - *Exception handling*: once an exception is detected, handle it by returning an error code, masking the fault, or repairing it.
  - *Rollback*: return to a previously saved good state (checkpoint, "rollback line") and continue from there.
  - *Software upgrade*: install new code while the system is running, without affecting service (in-service upgrade, patching, hot swap).
  - *Retry*: repeat an operation that failed because of a *transient* fault. It needs a limit on the number of attempts and usually backoff.
  - *Ignore faulty behaviour*: discard messages or events from a source known to be spurious or faulty.
  - *Degradation*: when components fail, keep the most critical functions and drop less critical ones. *(4th ed.: "Graceful degradation"; verify)*
  - *Reconfiguration*: reassign responsibilities to the resources that still work.
- **Recover from faults: reintroduction**
  - *Shadow*: run a repaired or upgraded component in "shadow mode" and watch its behaviour before it becomes active.
  - *State resynchronization*: bring a recovering component's state up to date. It partners with active and passive redundancy.
  - *Escalating restart*: restart at the smallest effective granularity first (a thread, then a process, then a node), and escalate only if needed. This keeps the impact small.
  - *Non-stop forwarding*: split the supervisory (control) plane from the data plane, so the data plane keeps working on its last known good state while the supervisor recovers. It comes from router design.
- **Prevent faults**
  - *Removal from service*: temporarily take a component out of service to head off expected failures, for example by rebooting to clear memory leaks.
  - *Transactions*: group state updates with ACID properties, so partial updates cannot corrupt state.
  - *Predictive model*: monitor health indicators and predict faults before they happen (for example, a queue growing towards its limit).
  - *Exception prevention*: stop exceptions from happening at all, with smart pointers, abstract data types and wrappers.
  - *Increase competence set*: design the component to handle more cases as normal operation, so fewer conditions count as faults.

### Typical tradeoffs
- Redundancy costs money, energy and complexity, and synchronizing replicas costs **performance**.
- Heartbeats and ping/echo add network load. A shorter period means faster detection but more overhead.
- Retry can increase load during an outage (retry storms). Use backoff and a cap.
- Rollback and transactions cost latency and storage.
- Redundant nodes widen the **security** attack surface.

### Exam traps
- "Name the three groups of availability tactics" (asked in 2015): **detect faults, recover from faults, prevent faults**. Recovery then splits into *preparation and repair* and *reintroduction*.
- Heartbeat is **pushed** by the monitored component; ping/echo is **pulled** by the monitor. Do not mix them up.
- Hot, warm and cold spares differ in **how in-sync the spare is**, which means how fast fail-over is. Hot is fastest and most expensive.
- Replication is a variant of **voting**, not a separate detect tactic, even though Wikipendium lists it that way.
- Five nines is about **5 minutes per year**, not per month.
- Retry only helps against **transient** faults. A deterministic bug fails again.

---

## 2. Interoperability (4th ed.: Integrability)

3rd ed. ch. 6. The 4th ed. replaces it with **Integrability** in ch. 7; see `quality-attributes-4th-edition.md`. Adapted in part from the Wikipendium compendium (CC BY-SA 3.0).

### Definition
Interoperability is the degree to which two or more systems can **usefully exchange meaningful information through interfaces** in a particular context.
- **Syntactic interoperability** is the ability to exchange data: formats and protocols line up.
- **Semantic interoperability** is the ability to *interpret* the exchanged data correctly: both sides mean the same thing by it.

A system is never interoperable in isolation, only relative to other systems. The two motivations are:
- providing a service to other systems, possibly unknown ones;
- building capabilities by composing existing systems (a system of systems).

The two key concerns are:
- **Discovery**: the consumer must learn the location, identity and interface of the service, either at runtime or before it.
- **Handling of the response**: the service can report back to the requester, broadcast the result, or send it on to another system.

### General scenario (3rd ed.)

| Part | Allowed values |
|---|---|
| Source | A system that initiates a request to interoperate with another system |
| Stimulus | A request to exchange information among systems |
| Artifact | The systems that wish to interoperate |
| Environment | The systems that wish to interoperate are discovered at runtime, or are known before runtime |
| Response | The request is (appropriately) rejected and the relevant entities are notified, *or* it is accepted and the information exchanged successfully. Either way it may be logged by one or more of the systems. |
| Response measure | Percentage of information exchanges correctly processed; percentage correctly rejected |

### Concrete game-project scenario

| Part | Value |
|---|---|
| Source | The game client (libGDX, Android) |
| Stimulus | Submits a finished match result to the leaderboard service |
| Artifact | The client and the backend (Firebase Firestore, or a Cloud Function) |
| Environment | Both known at build time; normal operation |
| Response | The result is written in the agreed schema, or rejected with an error code if it fails validation. The client displays the updated rank. |
| Response measure | 100% of well-formed results stored and correctly interpreted (right player and score field); 100% of malformed results rejected |

### Tactic tree (3rd ed.; complete)

- **Locate**
  - *Discover service*: find a service by searching a known directory service, at runtime or earlier. The search can be by type, name, location or other attributes.
- **Manage interfaces**
  - *Orchestrate*: a control mechanism (an orchestrator or workflow engine) coordinates, manages and sequences calls to services, so that the services themselves need not know about each other.
  - *Tailor interface*: add or remove capabilities of an interface. Adding covers translation, buffering and data smoothing. Removing covers hiding functions from untrusted users. The Adapter pattern is the classic way to do it.

**4th-ed. note (verify):** Integrability has broader groups: *limit dependencies* (encapsulate, use an intermediary, restrict communication paths, adhere to standards, abstract common services), *adapt* (discover, tailor interface, configure behaviour) and *coordinate* (orchestrate, manage resources).

### Typical tradeoffs
- Orchestrators and directories are single points of failure, and each hop adds latency (**performance**).
- Tailoring for untrusted consumers helps **security**. Translation layers add maintenance cost.
- Standard formats help semantic interoperability, but can constrain **modifiability** of the data model.

### Exam traps
- Syntactic interoperability is not enough. The classic failure is a unit mismatch (metres vs feet): the bytes parse but the meaning is wrong.
- Interoperability has only **two** tactic groups and **three** tactics in the 3rd ed. Do not pad the answer with modifiability tactics.
- Interoperability is **not** a chapter in the 4th ed. If the course uses the 4th ed., answer with integrability.

---

## 3. Modifiability

3rd ed. ch. 7 / 4th ed. ch. 8. Adapted in part from the Wikipendium compendium (CC BY-SA 3.0).

### Definition
Modifiability is about the **cost and risk of making changes**. Plan for change by asking four questions: **what** can change, how **likely** the change is, **when and by whom** it is made, and what it **costs**. Two design measures drive it:
- **Cohesion**: how strongly the responsibilities inside a module are related. Aim high.
- **Coupling**: how likely a change to one module is to spread to another. Aim low. A change that spreads this way is called a *ripple effect*.

### General scenario (3rd ed.)

| Part | Allowed values |
|---|---|
| Source | End user, developer, system administrator |
| Stimulus | A directive to add, delete or modify functionality, or to change a quality attribute, capacity or technology |
| Artifact | Code, data, interfaces, components, resources, configurations |
| Environment | Runtime, compile time, build time, initiation time, design time |
| Response | One or more of: make the change, test it, deploy it |
| Response measure | Cost in number, size or complexity of affected artifacts; effort; elapsed time; money; how far the change affects other functions or QAs; new defects introduced |

### Concrete game-project scenario

| Part | Value |
|---|---|
| Source | A developer in the group |
| Stimulus | Wants to add a new power-up type |
| Artifact | Game model / ECS systems and the asset loader |
| Environment | Design/development time |
| Response | The power-up is added as a new component and system (or strategy class) plus an asset entry, and tested |
| Response measure | At most 3 new classes and 0 changed existing classes outside the factory registration; done within 4 person-hours; no regressions in the existing test suite |

### Tactic tree (3rd ed.)

- **Reduce the size of a module**
  - *Split module*: break a large module into smaller ones, so each change touches less code. *(4th ed.: moved under "Increase cohesion"; verify)*
- **Increase cohesion**
  - *Increase semantic coherence*: move responsibilities that do not serve the module's purpose to another module, new or existing. *(4th ed.: also "Redistribute responsibilities"; verify)*
- **Reduce coupling**
  - *Encapsulate*: put an explicit interface around a module and hide its internals, which is information hiding (Parnas).
  - *Use an intermediary*: break a direct dependency A → B by inserting X, giving A → X → B. X can be a publish-subscribe bus, a broker, a repository or a proxy. The type of intermediary depends on the type of dependency.
  - *Restrict dependencies*: limit which modules a module can see or talk to. Examples are layer rules, visibility modifiers and build-enforced module boundaries.
  - *Refactor*: pull duplicated responsibilities out of two modules into one shared place. *(4th ed.: not listed separately; verify)*
  - *Abstract common services*: implement similar services once, in a more general, parameterised form.
- **Defer binding**
  - Bind values or decisions **later in the life cycle** than where they are defined, so a change becomes a change of parameter, configuration or plug-in instead of a code change. The later the binding, the cheaper the change, but the more supporting machinery it needs.

### Binding-time table (defer binding)

| Binding time | Mechanisms (3rd-ed. examples) | Game-project example |
|---|---|---|
| Compile / build time | Component replacement (in a build script or makefile); compile-time parameterisation; aspects | Gradle build flavour or `expect/actual` platform code |
| Deployment time | Configuration-time binding | Different Firebase project per build or deploy |
| Startup / initialisation time | Resource files | Level definitions or tuning values in JSON loaded at start |
| Runtime | Runtime registration; dynamic lookup (e.g. of services); interpreting parameters; startup-time binding; name servers; plug-ins; publish-subscribe; shared repositories; polymorphism | Strategy/State objects chosen at runtime (polymorphism); an event bus between systems (publish-subscribe); Remote Config |

### Typical tradeoffs
- Intermediaries and indirection cost **performance** (the classic tension with *reduce overhead*).
- Late binding makes **testability** harder, because there are more configurations to test, and it adds runtime failure modes (**availability**).
- Over-generalising (abstracting common services too early) adds complexity without payoff.

### Exam traps
- "Four groups of modifiability tactics" (asked in 2016): **reduce module size, increase cohesion, reduce coupling, defer binding**. In the 4th ed. there are three groups, because reduce size is folded into cohesion *(verify)*.
- Do not list **patterns** (MVC, layers) as tactics. The course feedback explicitly flags this as a mistake. Say instead: "MVC realises *encapsulate* and *use an intermediary*".
- A response measure like "easy to change" is not measurable. Count classes, hours or files.
- Modifiability happens mostly at **design and development time**. A runtime environment is valid only for things like end-user configuration.

---

## 4. Performance

3rd ed. ch. 8 / 4th ed. ch. 9. Adapted in part from the Wikipendium compendium (CC BY-SA 3.0).

### Definition
Performance is about **time**: the system's ability to meet timing requirements when events arrive. Response time has two components:
- **processing time**, when resources are consumed;
- **blocked time**, which comes from contention for resources, waiting for a resource to become available, and waiting for other computations.

### General scenario (3rd ed.)

| Part | Allowed values |
|---|---|
| Source | Internal or external to the system |
| Stimulus | Arrival of a **periodic, sporadic or stochastic event** |
| Artifact | The system, or one or more of its components |
| Environment | Operational mode: normal, emergency, peak load, overload |
| Response | Process the events; possibly change the level of service |
| Response measure | Latency, deadline, throughput, jitter, miss rate (some summaries add data loss) |

Arrival patterns:
- **Periodic** events arrive at fixed intervals, such as a 60 Hz frame tick.
- **Stochastic** events arrive according to a probability distribution, such as player actions.
- **Sporadic** events arrive at an unknown time, but with a known minimum gap between them.

### Concrete game-project scenario

| Part | Value |
|---|---|
| Source | The player (external) |
| Stimulus | Taps to fire, stochastic, during a match with 50 on-screen entities |
| Artifact | Input handling, the game loop and the render pipeline |
| Environment | Normal operation on a mid-range Android phone |
| Response | The shot is processed and rendered |
| Response measure | Visual feedback within 50 ms; a stable 60 fps with no more than 1% of frames above 33 ms |

### Tactic tree (3rd ed.)

- **Control resource demand** (reduce the work that arrives)
  - *Manage sampling rate*: sample input streams at a lower rate, accepting lower fidelity. *(4th ed.: "Manage work requests"; verify)*
  - *Limit event response*: queue events, or process them only up to a maximum rate. Events that exceed it are queued or dropped.
  - *Prioritize events*: give events priorities, so low-priority ones can be ignored or delayed under load.
  - *Reduce overhead*: remove intermediaries and co-locate communicating components. This trades directly against modifiability. *(4th ed.: "Reduce computational overhead"; verify)*
  - *Bound execution times*: put a cap on the time spent on an event, for example a fixed number of iterations or an anytime algorithm.
  - *Increase resource efficiency*: use better algorithms and data structures on the critical path. *(4th ed.: "Increase efficiency of resource usage"; verify)*
- **Manage resources** (make the existing resources work better)
  - *Increase resources*: faster or more processors, memory or network. It costs money.
  - *Introduce concurrency*: process requests in parallel (threads, processes, pipelining), which cuts blocked time.
  - *Maintain multiple copies of computations*: **replicas or server pools** (usually behind a load balancer), so that no single server is a bottleneck.
  - *Maintain multiple copies of data*: **caching** and **data replication**, which keep copies close to where they are used. The new problems are keeping the copies **consistent** and deciding **what to cache**.
  - *Bound queue sizes*: cap the number of queued arrivals. It needs an overflow policy (drop or reject) and is often paired with limit event response.
  - *Schedule resources*: when resources are contended, a scheduling policy decides who gets them:
    - *FIFO*: all requests are equal and are served in arrival order.
    - *Fixed-priority*: each source has a static priority. Variants include *semantic importance*, *deadline monotonic* (shorter deadline gets higher priority) and *rate monotonic* (shorter period gets higher priority, for periodic tasks).
    - *Dynamic priority*: priorities change at runtime. Examples are *round-robin* and *earliest deadline first (EDF)*; least-slack-first is another.
    - *Static scheduling*: a cyclic executive, with the schedule computed offline.

**Correction to Wikipendium:** the compendium equates *maintain multiple copies of computations* with caching. That is wrong. Copies of computations means **replicated servers or processes** (for example a server pool behind a load balancer). Caching belongs to the separate tactic **maintain multiple copies of data**, which the compendium leaves out.

### Typical tradeoffs
- *Reduce overhead* against **modifiability**: fewer intermediaries means tighter coupling.
- Caches and replicas against **consistency** and memory. Concurrency against **testability** (nondeterminism) and correctness (races).
- *Increase resources* against **cost** and energy.
- *Limit event response* and *bound queue sizes* can lose events, which may break a **usability** or **availability** requirement.

### Exam traps
- In a game, frame time is a **deadline** or **latency** measure. "Runs smoothly" is not a response measure.
- Do not confuse performance with **usability**. The teacher's feedback explicitly flags mixing the two. "Menu is intuitive" is usability; "menu opens in < 200 ms" is performance.
- Scalability is related to performance but is not the same thing (see §8).
- Know the two top-level groups: **control resource demand** and **manage resources**.

---

## 5. Security

3rd ed. ch. 9 / 4th ed. ch. 11. Adapted in part from the Wikipendium compendium (CC BY-SA 3.0).

### Definition
Security is the system's ability to **protect data and information from unauthorised access while still giving access to people and systems that are authorised**. An attack is an attempt to breach this. The core properties are **CIA**:
- **Confidentiality**: data and services are protected from unauthorised access.
- **Integrity**: data and services are not manipulated without authorisation.
- **Availability**: the system is available for legitimate use, so resisting denial of service is part of security.

The supporting properties are:
- **Authentication**: the parties are who they claim to be.
- **Nonrepudiation**: a sender cannot deny sending, and a recipient cannot deny receiving.
- **Authorization**: a party gets only the privileges it is entitled to.

### General scenario (3rd ed.)

| Part | Allowed values |
|---|---|
| Source | A human or another system, which may have been identified before (correctly or not) or may be unknown; internal or external |
| Stimulus | An unauthorised attempt to display data, change or delete data, access system services, change the system's behaviour, or reduce availability |
| Artifact | System services, data within the system, a component or resource of the system, data produced or consumed by the system |
| Environment | Online or offline; connected or disconnected; behind a firewall or open; fully, partly or not operational |
| Response | Data and services protected from unauthorised access or manipulation; parties identified with assurance; parties unable to repudiate; resources available for legitimate use; activity tracked; attacks detected and relevant parties notified |
| Response measure | How much of the system is compromised; whether and how fast an attack is detected and identified; how many attacks are resisted; time to recover; how much data is exposed or vulnerable |

### Concrete game-project scenario

| Part | Value |
|---|---|
| Source | An external, unauthenticated user with a modified APK |
| Stimulus | Tries to write a fake high score directly to the backend |
| Artifact | The leaderboard collection in Firebase |
| Environment | Online, normal operation |
| Response | The write is rejected by the security rules or by server-side validation, and the attempt is logged |
| Response measure | 100% of unauthenticated writes rejected; authenticated users can modify only their own documents; an attempt is visible in the logs within 1 min |

### Tactic tree (3rd ed.)

The book's analogy is physical security: detect, resist, react, recover.

- **Detect attacks**
  - *Detect intrusion*: compare network traffic or service request patterns with known malicious signatures or profiles.
  - *Detect service denial*: compare incoming traffic with known denial-of-service (DoS) attack profiles.
  - *Verify message integrity*: use checksums or hash values to detect messages or files that have been altered.
  - *Detect message delay*: look for abnormal message delivery times, which can reveal a man-in-the-middle attack. *(4th ed.: "Detect message delivery anomalies"; verify)*
- **Resist attacks**
  - *Identify actors*: identify the **source of any external input**, for example by user ID, access code, IP address, protocol or port. This is not about identifying the attacker as such.
  - *Authenticate actors*: make sure an actor is who it claims to be (passwords, certificates, biometrics, multi-factor).
  - *Authorize actors*: make sure an authenticated actor has the rights to access or change data or services (access control lists, roles).
  - *Limit access*: control what can reach which resources (firewalls, DMZ, single access points, restricted memory or processes).
  - *Limit exposure*: **reduce the attack surface**. Put fewer services and access points on each host, so one successful attack exposes as little as possible. This is not "security through obscurity".
  - *Encrypt data*: protect data at rest and in transit (symmetric or asymmetric encryption, TLS).
  - *Separate entities*: separate sensitive data or processes physically (different servers, air gaps) or virtually (VMs, network segments).
  - *Change default settings*: force users to change default credentials and settings, so attackers cannot rely on well-known defaults. *(4th ed.: "Change credential settings"; verify)*
  - *(4th ed. adds "Validate input"; verify)*
- **React to attacks**
  - *Revoke access*: restrict access to sensitive resources, even for normally legitimate users, while an attack is suspected.
  - *Lock computer*: lock out after repeated failed login attempts. *(4th ed.: "Restrict login"; verify)*
  - *Inform actors*: notify operators, other personnel or cooperating systems about a suspected attack.
- **Recover from attacks**
  - *Maintain audit trail*: record user and system actions and their effects, to trace attackers, support **nonrepudiation** and aid recovery. *(4th ed.: audit and nonrepudiation listed under recover; verify)*
  - *Restore*: bring the system back to a correct state, reusing the availability recovery tactics (rollback, redundancy and so on).

**Corrections to Wikipendium glosses:**
- *Limit exposure* means reducing the attack surface. It does not mean hiding how the system works.
- *Identify actors* means identifying the source of input. It is a prerequisite for authentication and does not mean "finding the attacker".
- *Change default settings* means forcing users to replace defaults, not having no defaults.

### Typical tradeoffs
- Encryption, authentication and auditing cost **performance** (latency, CPU, battery).
- Lockouts and more login steps hurt **usability**. Revoking access can hurt **availability** for legitimate users.
- Separate entities and limit exposure increase **cost** and deployment complexity.
- Audit trails raise **privacy** (GDPR) concerns about what is logged.

### Exam traps
- CIA is the core. Authentication, nonrepudiation and authorization are *supporting* properties. Know all six.
- There are four groups: **detect, resist, react, recover**. Students often forget *react*.
- In a game project, "we use Firebase Auth" is a design choice, not a scenario. The scenario needs a stimulus (an attack) and a measure.
- Client-side checks alone never satisfy a security scenario, because the client is under the attacker's control.

---

## 6. Testability

3rd ed. ch. 10 / 4th ed. ch. 12. Adapted in part from the Wikipendium compendium (CC BY-SA 3.0).

### Definition
Testability is the **ease with which software can be made to reveal its faults through (typically execution-based) testing**. For a system to be testable you must be able to (1) **control** each component's inputs and internal state, and (2) **observe** its outputs and state. Plan testing early, build infrastructure for injecting faults and recording state, and automate tests for every increment.

### General scenario (3rd ed.)

| Part | Allowed values |
|---|---|
| Source | Unit, integration, system or acceptance testers, or end users, running tests manually or with automated tools |
| Stimulus | A set of tests is run because a coding increment was completed (a class, layer or service), a subsystem was integrated, the whole system was implemented, or the system was delivered |
| Artifact | The part of the system being tested |
| Environment | Design, development, compile, integration, deployment or runtime |
| Response | Execute the test suite and capture the results; capture the activity that led to the fault; control and monitor the system's state |
| Response measure | Effort to find a fault or class of faults; effort to reach a given state-space coverage; probability that the next test reveals a fault; time to run the tests; length of the longest dependency chain; time to prepare the test environment; reduction in risk exposure |

### Concrete game-project scenario

| Part | Value |
|---|---|
| Source | A developer running automated unit tests (JUnit) |
| Stimulus | Completes the scoring and collision logic for a sprint |
| Artifact | Game-model classes in the `core` module |
| Environment | Development time, on CI without an Android device or a real Firebase |
| Response | Tests run against a fake backend and a headless game model; results are reported |
| Response measure | At least 80% line coverage of the `core` game logic; the suite runs in under 60 s; no test needs network access |

### Tactic tree (3rd ed.; complete)

- **Control and observe system state**
  - *Specialized interfaces*: test-only interfaces for controlling or capturing variable values, such as set/get, report, reset, and a verbose output mode.
  - *Record/playback*: capture the state or data crossing an interface so it can be replayed later, for example recorded input sequences that reproduce a bug.
  - *Localize state storage*: keep state in one place (one object or store), so it is easy to set up and inspect.
  - *Abstract data sources*: put data sources behind interfaces so a test database or fake can replace them. In a game, abstract the Firebase backend behind an interface.
  - *Sandbox*: isolate the system from the real world so experiments cannot do damage (virtualisation, simulated clocks, mock networks).
  - *Executable assertions*: hand-coded assertions placed at chosen points, to detect when the program is in a faulty state.
- **Limit complexity**
  - *Limit structural complexity*: avoid or remove cyclic dependencies, isolate and encapsulate dependencies on the external environment, and aim for high cohesion and low coupling. A shallow inheritance and dependency depth also helps.
  - *Limit nondeterminism*: find and remove sources of unconstrained behaviour, such as unconstrained parallelism, random seeds and wall-clock dependence. For example, inject a fixed random seed and a fixed timestep.

### Typical tradeoffs
- Specialized interfaces and assertions add code, and can create a **security** back door if they are left in production.
- Abstracting data sources adds indirection, which helps **modifiability** but costs a little **performance**.
- Limiting nondeterminism can conflict with *introduce concurrency* (**performance**).

### Exam traps
- Singletons and global state hurt testability. Name *localize state storage* and *abstract data sources* as the fix.
- A testability response measure is about **effort, coverage or time**, not "the code is tested".
- Testability is not the same as having tests. It is a *property of the architecture* that makes testing cheap.

---

## 7. Usability

3rd ed. ch. 11 / 4th ed. ch. 13. Adapted in part from the Wikipendium compendium (CC BY-SA 3.0).

### Definition
Usability is **how easy it is for users to accomplish a task, and what kind of support the system gives them**. It covers five areas:
- **learning** the system's features;
- **using the system efficiently**;
- **minimising the impact of user errors**;
- **adapting** the system to the user's needs;
- **increasing confidence and satisfaction**.

### General scenario (3rd ed.)

| Part | Allowed values |
|---|---|
| Source | The end user, possibly in a specialised role |
| Stimulus | The user tries to use the system efficiently, learn it, minimise the impact of errors, adapt it, or configure it |
| Environment | Runtime or configuration time |
| Artifact | The system, or the specific part the user interacts with |
| Response | Provide the features the user needs, or anticipate the user's needs |
| Response measure | Task time; number of errors; number of tasks accomplished; user satisfaction; gain in user knowledge; ratio of successful to total operations; time or data lost when an error occurs |

### Concrete game-project scenario

| Part | Value |
|---|---|
| Source | A first-time player |
| Stimulus | Wants to learn how to start a multiplayer match |
| Environment | Runtime, first launch |
| Artifact | Main menu and lobby UI |
| Response | Guided flow: the tutorial prompt, then "Play online", then a clear waiting indicator; cancel is always available |
| Response measure | 9 out of 10 test users start a match within 60 s without help; 0 dead-end screens |

### Tactic tree (3rd ed.; complete)

- **Support user initiative** (the user acts, and the system must respond with feedback)
  - *Cancel*: the system must listen for a cancel request, stop the activity, free its resources and tell the collaborating components.
  - *Undo*: keep enough state (or reversible operations) to go back to an earlier state. The Command and Memento patterns are common ways to do it.
  - *Pause/resume*: temporarily stop a long-running operation and continue it later, freeing resources in between.
  - *Aggregate*: let the user apply one operation to a group of objects (multi-select, batch operations).
- **Support system initiative** (the system anticipates what the user needs)
  - *Maintain task model*: keep information about the task the user is doing, to give context-sensitive help. The classic example is automatic capitalisation at the start of a sentence.
  - *Maintain user model*: keep the user's knowledge, behaviour and preferences, to adapt the level of help and customisation (for example, hiding hints from experienced players).
  - *Maintain system model*: keep a model of the system's own expected behaviour, to give feedback such as progress bars and estimated time remaining.

### Typical tradeoffs
- Undo and pause/resume need state history, which costs memory and **performance** and adds complexity.
- User models raise **privacy** and **security** concerns.
- Usability patterns such as MVC also help **modifiability**, because UI changes stay separate.

### Exam traps
- Usability is the QA where **architecture** matters less than UI design. The architectural part is *enabling* cancel, undo and models, which requires separating the UI from the model.
- Measures must be observable, for example task time, error count or satisfaction score. "User-friendly" is not a measure.
- Keep usability scenarios separate from performance ones.

---

## 8. Other quality attributes

3rd ed. ch. 12 / 4th ed. ch. 14, plus separate 4th-ed. chapters. Adapted in part from the Wikipendium compendium (CC BY-SA 3.0).

| QA | Meaning | Notes |
|---|---|---|
| **Variability** | The ability of a system and its artifacts to support producing a set of variants that differ in known ways | A special case of modifiability, and central to software **product lines**. Mechanisms: variation points, defer binding. |
| **Portability** | The ease with which software built to run on one platform can be changed to run on another | A special case of modifiability. Key tactic: a **portability layer**, i.e. abstract platform services (for example libGDX backends, or Kotlin `expect/actual`). |
| **Development distributability** | How well the software can be developed by distributed teams | Coordination cost follows module dependencies (Conway's law). Low coupling between work units helps. |
| **Scalability** | The ability to handle more load by adding resources | **Horizontal** (scale out) adds logical units, such as another server in a cluster; in the cloud this is **elasticity**, adding and removing instances on demand. **Vertical** (scale up) adds resources to one physical unit, such as more memory or CPU. Measures: how the response changes as load grows, and cost per unit of capacity. |
| **Deployability** | How an executable reaches its host platform and is then invoked; the time, cost and predictability of deployment | Its own chapter in the 4th ed. (ch. 5), covering CI/CD, rollback and canary releases. |
| **Mobility** | Problems of mobile platforms: moving between networks, battery, intermittent connectivity, small screens | Its own chapter (Mobile Systems, ch. 18) in the 4th ed. Very relevant to Android game projects. |
| **Monitorability** | How well operations staff can observe the system while it runs | Built with metrics, logging and health endpoints. It overlaps with availability's *monitor* tactic. |
| **Safety** | The system's ability to avoid entering states that cause or lead to damage, injury or loss of life | Its own chapter (ch. 10) in the 4th ed. Not the same as security: safety is about *harm*, whatever the intent. |

### ISO/IEC 25010 (the standard quality model)
ISO/IEC 25010:2011 (the SQuaRE series) defines a **product quality model** with eight characteristics:
- functional suitability
- performance efficiency
- compatibility (which includes interoperability)
- usability
- reliability (which includes availability)
- security
- maintainability (which includes modifiability and testability)
- portability

Each characteristic has sub-characteristics. The 3rd ed. discusses it as a standard list and notes that it does not give scenarios or tactics. A 2023 revision of 25010 changes the model; for example it adds safety and renames some characteristics (verify the details before citing it). Its taxonomy differs from SAiP's: modifiability and testability are sub-characteristics of maintainability, and availability is a sub-characteristic of reliability.

### How to specify a new QA
The book's recipe in the other-QAs chapter runs roughly as follows (verify the exact steps in your edition):
1. **Capture scenarios** for the new QA. Build a general scenario from stakeholder concerns.
2. **Assemble design approaches**: find the tactics and patterns that affect the QA.
3. **Model** the QA where possible, for example with queueing models for performance or Markov models for availability.
4. **Build a design checklist** covering the seven categories of design decision (see below).

### The seven design-decision categories (checklist used in every QA chapter)
Allocation of responsibilities; coordination model; data model; management of resources; mapping among architectural elements; binding-time decisions; choice of technology. In every 3rd-ed. QA chapter, the book gives a checklist for each category. For example, for availability under allocation of responsibilities it asks: which responsibilities must be highly available, and how are faults detected for each of them?

---

## 9. Cross-QA tradeoff matrix and exam checklist

### Common tactic conflicts

| Tactic (QA it serves) | Hurts | Why |
|---|---|---|
| Use an intermediary / encapsulate (modifiability) | Performance | Extra indirection and hops |
| Reduce overhead (performance) | Modifiability | Removes intermediaries, so coupling increases |
| Active redundancy (availability) | Cost, performance, security | Synchronisation, hardware, more attack surface |
| Encrypt data / authenticate (security) | Performance, usability | CPU and latency; friction at login |
| Multiple copies of data / caching (performance) | Consistency, modifiability | Stale data, invalidation logic |
| Introduce concurrency (performance) | Testability | Nondeterminism |
| Defer binding (modifiability) | Testability, performance | More configurations to test; runtime lookup |
| Undo / pause-resume (usability) | Performance, complexity | State history |
| Specialized test interfaces (testability) | Security | A back door if shipped |

### Top-level tactic groups (memorise)

| QA | Groups (3rd ed.) |
|---|---|
| Availability | Detect faults · Recover (preparation and repair; reintroduction) · Prevent faults |
| Interoperability | Locate · Manage interfaces |
| Modifiability | Reduce module size · Increase cohesion · Reduce coupling · Defer binding |
| Performance | Control resource demand · Manage resources |
| Security | Detect · Resist · React · Recover |
| Testability | Control and observe system state · Limit complexity |
| Usability | Support user initiative · Support system initiative |

### Checklist before answering a QA exam question
- [ ] All six parts are present, and the response measure is a **number with a unit**.
- [ ] The scenario is labelled as general or concrete, and the one asked for is given.
- [ ] Only tactics are named as tactics. Patterns are named separately, with the tactics they realise.
- [ ] Performance scenarios are kept apart from usability ones, and security from safety.
- [ ] At least one tradeoff is named for each tactic chosen.
- [ ] Chapter numbers are cited for both editions, with 4th-ed. renames flagged as needing verification.
