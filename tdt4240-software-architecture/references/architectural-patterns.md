# Architectural patterns: module, C&C and allocation

This file covers the 11 architectural patterns in the course's classic list (SAiP 3rd ed. ch. 13). For each one it gives the quality attributes (QAs) it promotes and inhibits, a TDT4240 game example, and when not to use it. After that come a comparison table, a guide for picking a pattern from a scenario with practice questions, some newer patterns, and Coplien (1998).

> Parts of this file are adapted from the Wikipendium TDT4240 compendium (CC BY-SA 3.0), https://www.wikipendium.no/TDT4240_Software_Architecture, by its contributors (listed in ../CREDITS.md). The text has been paraphrased, restructured, corrected and extended with material from SAiP. Corrections are marked **[correction]**.

Related files:
- GoF design patterns (Observer, State and others) and game patterns: `design-and-game-patterns.md`
- The tactics named in the tables: `quality-attributes-classic.md` and `quality-attributes-4th-edition.md`
- Pattern-to-view mapping: `documentation.md`
- Old exam questions: `exam-prep.md`

**Edition note.** In SAiP 3rd ed. (2013), ch. 13 "Architectural Tactics and Patterns" has a catalogue grouped into module, C&C and allocation patterns. The 4th ed. (2021) has no separate catalogue chapter; instead each QA chapter ends with a Patterns section. Approximately (unverified; see [quality-attributes-4th-edition.md](quality-attributes-4th-edition.md) §8): ch. 4 Availability has the redundancy patterns and circuit breaker; ch. 5 Deployability has microservice architecture; ch. 7 Integrability has adapter, service-oriented architecture and dynamic discovery; ch. 8 Modifiability has client-server, plug-in (microkernel), layers and publish-subscribe; ch. 9 Performance has service mesh, load balancer, throttling and map-reduce; ch. 13 Usability has MVC, observer and memento. Both the 2015 and 2016 exams cite SAiP 3rd ed.; the 2015 exam's pattern-choice question used this list (see §6).

---

## 1. What a pattern is (SAiP)

An **architectural pattern** is a package of design decisions that:
- keeps coming up in practice,
- has known properties that make it reusable, and
- describes a *class* of architectures.

Patterns are **discovered, not invented**.

**Three-part description.** The 2016 exam asked for this. A pattern is a triple:

| Part | Meaning |
|---|---|
| **Context** | A recurring, common situation that gives rise to a problem. |
| **Problem** | The problem stated in general terms, usually including the QAs that must be met. |
| **Solution** | A successful architectural answer to the problem, described abstractly. |

The **solution** is itself described by:
- **element types**, e.g. layer, filter, broker, server;
- **interaction mechanisms / connectors**, e.g. calls, pipes, events;
- **topological layout**, i.e. how the elements may be arranged;
- **semantic constraints** on topology, element behaviour and interaction mechanisms;
- **weaknesses**, which SAiP includes in each catalogue entry.

**Tactics vs patterns.** A tactic is a single design decision aimed at one QA response. A pattern *bundles* tactics and often trades QAs against each other. Applying a pattern usually causes side effects, and further tactics are added to repair them (SAiP calls this "augmenting" a pattern with tactics). In the course documents, **do not list patterns as tactics**; teacher feedback flags this repeatedly.

**Architectural vs design pattern:** see `design-and-game-patterns.md` (e.g. MVC is architectural; Observer is a design pattern often used to implement it).

**Categories** follow the three view types (see `documentation.md`):
- **Module** patterns structure code and data units. The course list has one: Layered.
- **Component-and-connector (C&C)** patterns structure runtime elements and their interactions.
- **Allocation** patterns map software onto non-software things: hardware, file systems, networks, teams.

---

## 2. Module pattern

### 2.1 Layered
- **Context:** A complex system whose parts must be developed and evolved separately, with a clear, documented separation of concerns.
- **Problem:** Split the software so modules can be built and changed independently, with little interaction between the parts. The aims are portability, modifiability and reuse.
- **Solution:**
  - Elements: **layers**, each a cohesive set of modules with a public interface.
  - Relation: **allowed-to-use**, which is a specialisation of *uses*.
  - Topology: a stack, drawn as boxes stacked on top of each other.
- **Constraints:**
  - Every piece of software belongs to exactly one layer.
  - There are at least two layers.
  - The allowed-to-use relation is **unidirectional and downward**. In the strict form, a layer uses only the layer directly beneath it.
  - **Layer bridging** means a layer uses a layer further down than the one directly beneath it. It is an *exception* that must be **documented**, not the norm.
  - **Upward use is forbidden.** A layer must not use the layers above it. Upward notification is done with callbacks or events, which do not create an upward dependency.
  - **[correction]** Wikipendium says every layer may use the public interfaces of all other layers and calls that "layer bridging". That is wrong.
- **Promotes:** modifiability, portability, reuse, testability (a layer can be tested with lower layers stubbed), and separate development by different teams.
- **Inhibits:** performance, because each call passes through several layers. It also adds up-front cost and complexity. If bridging is uncontrolled, the modifiability benefit disappears.
- **Game example:** A libGDX project split into:
  - *presentation*: screens, rendering and input;
  - *game logic*: rules and ECS systems;
  - *services*: a backend interface for leaderboard and lobby;
  - *platform/data*: a Firebase adapter, with Android and desktop launchers.

  Game logic depends only on a `BackendService` interface, so moving from Firebase to Supabase touches only the bottom layer. This is often the development view.
- **Don't use when:** the system is tiny, or tight real-time budgets cannot absorb the indirection, for example a hot rendering path that has to call straight through to the GPU layer. Also avoid it if the team cannot enforce the boundaries: a layer diagram the code ignores is worse than no layers at all.

---

## 3. Component-and-connector patterns

### 3.1 Broker
- **Context:** Many services are spread across several servers, and clients should not need to know where a service lives or how to reach it.
- **Problem:** Structure distributed software so that users need not know the nature and location of service providers, and so that the binding between users and providers can change at runtime.
- **Solution:**
  - Elements: **client**, **server**, **broker**, and optional client-side and server-side **proxies** that handle marshalling.
  - Servers register with the broker. The broker locates a suitable server and forwards the request.
  - In SAiP's description the broker mediates both request and response, so **the reply goes back through the broker**. POSA (Buschmann et al. 1996) also has a *direct-communication* variant in which the broker only locates the server and the two then talk directly. Wikipendium's "servers reply directly to the client" describes that variant, not the default.
- **Constraints:** A client attaches only to a broker (possibly through a proxy). A server attaches only to a broker.
- **Promotes:** modifiability (servers replaced, moved or added transparently), availability (a failed server can be substituted), interoperability, and performance through spreading load over servers.
- **Inhibits:** **latency** (extra hop both ways), **single point of failure** and bottleneck, **security target**, hard to test.
- **Game example:** A matchmaking service that routes "find game" requests to regional game servers. Clients know only the matchmaker endpoint.
- **Don't use when:** there are only one or two known servers, since a direct client-server link is simpler. Also avoid it for latency-critical per-frame traffic.

### 3.2 Model-View-Controller (MVC)
- **Context:** The user interface changes more often than the domain logic. Users want several views of the same data, and the views must stay consistent.
- **Problem:** Keep UI functionality separate from application functionality while still responding to user input and to changes in the data.
- **Solution:**
  - **Model**: application data and state, and it announces changes.
  - **View**: renders the model.
  - **Controller**: interprets user input and turns it into model updates (and view selection).
  - Relation: *notifies*. MVC is **usually implemented with the Observer pattern**: views, and sometimes controllers, subscribe to the model.
- **Constraints:** There is at least one instance of each. The model does not depend on concrete views or controllers.
- **Promotes:** modifiability of the UI, several synchronised views, and testability of the model without a UI.
- **Inhibits:** it adds complexity that may not pay off for simple UIs. The abstractions may not fit some UI toolkits, and a UI can flood the model with update events.
- **Game example:** `GameModel` holds the board, players and turn; `GameScreen` (the view) renders it; `InputController` maps taps to moves; the model notifies the view through listeners. Very common in TDT4240 projects, often with a State-based screen manager.
- **Don't use when:** the UI is trivial or throwaway. Also consider whether a pure ECS game loop already separates data (components) from behaviour (systems). In that case, say how the two fit together rather than forcing MVC on top.

### 3.3 Pipe-and-Filter
- **Context:** Many systems transform streams of discrete data items from input to output, and the same kinds of transformation recur.
- **Problem:** Split the system into reusable, loosely coupled parts with simple, generic interaction, so they can be flexibly combined and possibly run in parallel.
- **Solution:**
  - Elements: **filters**, which transform data read on input ports and write to output ports.
  - Connectors: **pipes**, which carry data from one filter's output to another's input, preserving order and possibly buffering.
- **Constraints:**
  - Pipes connect filter output ports to filter input ports.
  - Connected filters must agree on the data type.
  - Filters are independent and keep no shared state.
- **Promotes:** modifiability and reuse (filters can be recombined), and performance through concurrency, because filters can run in parallel.
- **Inhibits:** it is poorly suited to interactive systems. Many small filters add computational overhead, including format conversion between filters. It is also a weak fit for long-running computations that need shared state.
- **Game example:** An asset build pipeline: load sprite, trim, pack into a texture atlas, compress. Another example is an audio or post-processing effect chain.
- **Don't use when:** the core is interactive request/response game logic, or when steps need rich shared state.

### 3.4 Client-Server
- **Context:** Many distributed clients want to access shared resources and services, and access and quality of service must be controlled.
- **Problem:** Improve scalability and availability by centralising the control of those resources, while the resources themselves may be distributed across several servers.
- **Solution:**
  - Elements: **clients**, which request services, and **servers**, which provide them.
  - Connector: **request/reply**, often over a network.
  - A component may act as both client and server.
- **Constraints:** Clients connect to servers. Servers may be clients of other servers, but the number of tiers may be restricted.
- **Promotes:** modifiability and reuse (common services in one place), security and consistency (centralised control), scalability (servers can be replicated).
- **Inhibits:** the server can be a **performance bottleneck** and a **single point of failure**; where functionality lives (client or server) is costly to change later.
- **Game example:** An authoritative game server, or a Backend-as-a-Service (BaaS) such as Firebase used as the server, that validates moves and stores lobbies and high scores. Android clients send requests and receive state.
- **Don't use when:** all participants are equal and there is no trusted central party (use P2P). Also avoid it where the server's cost and single point of failure are unacceptable.

### 3.5 Peer-to-Peer (P2P)
- **Context:** Distributed computational entities that are all equally important collaborate, and none of them is a natural central server.
- **Problem:** Connect the set of distributed entities through a common protocol so they can organise and share their services with high availability and scalability.
- **Solution:**
  - Elements: **peers**, each of which acts as both client and server.
  - Connectors: request/reply interaction, where a search or request is routed through intermediate peers.
  - Peers join and leave dynamically. There may be special peers such as supernodes.
- **Constraints:** Rules may limit how many connections a peer has, and may define peer roles.
- **Promotes:** availability (no single point of failure), scalability (every peer adds capacity), lower cost (no dedicated server).
- **Inhibits:** security, data consistency, availability of *particular* data (peers leave), backup and recovery; small systems may not reach their quality goals.
- **Game example:** Two phones playing a local-network game where each device runs the simulation and exchanges moves directly, as in lockstep for a turn-based game. The classic outside example is BitTorrent file sharing.
- **Don't use when:** you need an authoritative source for anti-cheat, scores, or consistency. Also avoid it when the number of peers is tiny and the benefits never appear.

### 3.6 Service-Oriented Architecture (SOA)
- **Context:** Many services from different providers, possibly written in different languages and running on different platforms, must be combined, and consumers should not depend on how the services are implemented.
- **Problem:** Support interoperability between distributed components that run on different platforms and are written in different languages, provided by different organisations, and distributed across the Internet.
- **Solution:**
  - Elements: **service providers** and **service consumers**.
  - Optional infrastructure:
    - an **ESB** (Enterprise Service Bus), which routes and transforms messages;
    - a **service registry**, where services are published and discovered at runtime;
    - an **orchestration server**, which runs business workflows that invoke several services.
  - Connectors: SOAP, REST and asynchronous messaging.
- **Constraints:** Consumers use services only through their published interfaces or contracts. Service consumers connect to providers, but may use intermediaries such as the ESB.
- **Promotes:** interoperability, modifiability (a service can be swapped behind its contract), and reuse.
- **Inhibits:** complex to build; no control over how external services evolve; middleware overhead; services can become bottlenecks; usually no performance guarantees.
- **Game example:** The game uses separate external services: authentication (an identity provider), payments, analytics and a push-notification service, each through its published API, composed by the app backend.
- **Don't use when:** a single team owns a small monolith. The infrastructure is overkill for a student game unless the point is to integrate third-party services.

### 3.7 Publish-Subscribe
- **Context:** Independent producers and consumers of data must interact, and their number and identity are not known in advance or keep changing.
- **Problem:** Let producers send information to consumers without knowing who or how many the consumers are.
- **Solution:**
  - Elements: components with **publish** and/or **subscribe** ports.
  - Connector: an **event bus** (a publish-subscribe connector) that delivers each announced event to all current subscribers of that event type.
  - A component may both publish and subscribe.
- **Constraints:** All components connect to the event bus, not to each other.
- **Looser than GoF Observer.** In Observer the *subject* holds direct references to its observers and calls them itself. In pub-sub the bus sits in between, so publishers and subscribers do not know each other.
- **Promotes:** modifiability (add or remove subscribers without touching publishers), extensibility, and loose coupling.
- **Inhibits:** **latency**, scalability, **predictability** of delivery time, control over ordering, guaranteed delivery; **harder to test and reason about** because control flow is implicit.
- **Game example:**
  - An in-game event bus: `EnemyDestroyed` is published, and the score system, sound system, achievements and analytics each subscribe.
  - Across the network: a Firebase Realtime Database listener pushes state changes to every client in a lobby.
- **Don't use when:** there is a single known receiver, or the interaction needs a synchronous reply or strict ordering and timing guarantees.

### 3.8 Shared-Data (Repository)
- **Context:** Several computational components need to share and manipulate large amounts of data, and the data does not belong to any one of them.
- **Problem:** Store and manipulate persistent data that many independent components access.
- **Solution:**
  - Elements: one or more **shared-data stores** and several **data accessors**.
  - Connector: *data reading and writing*, which may include queries and triggers.
  - Variants:
    - *repository*, where accessors initiate the interaction;
    - *blackboard*, where the store notifies accessors of changes.
- **Constraints:** Data accessors interact only with the data store(s), not directly with each other.
- **Promotes:** data consistency, modifiability of the accessors (they are independent of each other), and scalability of data management.
- **Inhibits:**
  - the store can be a **performance bottleneck** and a **single point of failure**;
  - producers and consumers are tightly **coupled to the shared data model**, so schema changes ripple.
- **Game example:** Firestore holds user profiles, match history and the leaderboard. The game client, an admin tool and a Cloud Function all read and write it.
- **Don't use when:** the data is private to one component, or write contention would make the store the bottleneck of a real-time loop.

---

## 4. Allocation patterns

### 4.1 Map-Reduce
- **Context:** Businesses need to analyse very large volumes of data quickly, for example logs or telemetry.
- **Problem:** Efficiently process a large data set by distributing it across many nodes, in a way that can be parallelised and is robust to node failure.
- **Solution:**
  - Elements: a **map** function (filters and transforms records in parallel on many nodes), a **reduce** function (combines intermediate results), and an infrastructure (e.g. Hadoop) that allocates software to hardware nodes and schedules and monitors jobs. Frameworks such as Hadoop shuffle/sort intermediate results between map and reduce.
  - The pattern runs on a **distributed infrastructure, typically a commodity cluster**.
  - **[correction]** Wikipendium says it "needs specialised hardware". That is imprecise: the point is ordinary machines coordinated by a framework.
- **Constraints:**
  - The data to analyse exists as a set of files or partitions.
  - Map functions are stateless and do not communicate with each other.
  - Map and reduce instances communicate only through the emitted `<key, value>` pairs, which the infrastructure shuffles and sorts.
- **Promotes:** performance on big data through massive parallelism, scalability (add nodes), and availability, because failed tasks are re-run.
- **Inhibits / limits:**
  - the overhead is not justified without a large data set;
  - parallelism is lost if the data cannot be split into similar-sized subsets;
  - operations that need several reduce steps are complex to orchestrate.
- **Game example:** Offline analysis of millions of match logs to balance weapons or compute global rankings. It is never part of the live game.
- **Don't use when:** the data is small, the processing is interactive or real-time, or the data cannot be partitioned.

### 4.2 Multi-tier
- **Context:** In a distributed deployment, the infrastructure must be divided into distinct subsets.
- **Problem:** Split the system into computationally independent execution structures (groups of hardware and software), connected by some communication medium, for operational or business reasons.
- **Solution:**
  - Elements: **tiers**, which are logical groupings of components, for example presentation, business logic and data.
  - Tiers are typically deployed on separate execution environments.
  - Relations: *is-part-of* (a component belongs to a tier), *communicates-with* (between tiers), and *allocated-to* (a tier runs on execution nodes).
  - It is a specialisation of the generic **deployment** (software-to-hardware) structure.
- **Constraints:** A component belongs to exactly one tier.
- **Category.** SAiP notes that multi-tier can be read as a C&C or an allocation pattern, depending on the criteria that define the tiers (component type/function vs computing infrastructure); the book catalogues it under allocation.
- **Promotes:** security (the data tier can sit behind a firewall), performance and availability through per-tier scaling and replication, and modifiability.
- **Inhibits:** substantial up-front cost and complexity. More network hops add latency.
- **Tier vs layer.** A **layer** is a *module* (code-time) grouping with an allowed-to-use relation; a **tier** is a runtime/deployment grouping of components onto execution environments. Several layers can run in one tier, and a tier boundary is typically a process or network boundary. Mixing them up is a classic exam and report error.
- **Game example:** Android client (presentation, local logic), Cloud Functions (move validation, matchmaking), Firestore (data). Show it in the physical/deployment view, with network types.
- **Don't use when:** everything runs on one device, such as a single-player offline game.

---

## 5. Comparison table

| Pattern | Category | Promotes | Inhibits | Typical tactic bundle |
|---|---|---|---|---|
| Layered | Module | Modifiability, portability, reuse | Performance; up-front cost | Encapsulate, restrict dependencies, use an intermediary, abstract common services |
| Broker | C&C | Modifiability, interoperability, availability | Latency, single point of failure, security, testability | Use an intermediary, discover service, encapsulate |
| MVC | C&C | UI modifiability, testability of the model | Complexity for simple UIs | Increase semantic coherence\*, encapsulate, defer binding (listeners) |
| Pipe-and-Filter | C&C | Reuse, modifiability, concurrency | Interactivity; overhead | Encapsulate, introduce concurrency, split module |
| Client-Server | C&C | Centralised control, reuse, scalability | Server as bottleneck and single point of failure | Increase resources, maintain multiple copies of computations, authorize actors |
| Peer-to-Peer | C&C | Availability, scalability | Security, consistency | Active redundancy, discover service, maintain multiple copies of data |
| SOA | C&C | Interoperability, modifiability | Performance, complexity, control of evolution | Discover service, orchestrate, tailor interface, use an intermediary |
| Publish-Subscribe | C&C | Modifiability, extensibility | Latency, predictability, testability | Use an intermediary, defer binding (runtime registration) |
| Shared-Data | C&C | Data consistency, accessor independence | Store bottleneck and single point of failure; coupling to the schema | Maintain multiple copies of data, transactions, limit access |
| Map-Reduce | Allocation | Performance on big data, scalability | Overhead on small data; needs partitionable data | Introduce concurrency, increase resources, retry |
| Multi-tier | Allocation | Security, per-tier scalability | Cost, latency | Separate entities, increase resources, schedule resources, maintain multiple copies of computations (load balancer) |

The tactic bundles are typical pairings for your own reasoning, not a list from the book. \*Tactic names follow the 3rd ed.; the 4th ed. (ch. 8) replaces "increase semantic coherence" with "redistribute responsibilities".

---

## 6. Choosing a pattern for a scenario

The 2015 exam gave 5 scenarios and asked which of 10 patterns fits best. The 10 were Layered, Broker, MVC, Pipe-and-Filter, Client-Server, P2P, SOA, Pub-Sub, Map-Reduce and Multi-tier; Shared-Data was not among the options. Look for the **cue words**:

| Cue in the scenario | Pattern |
|---|---|
| Stream or sequence of transformations; batch; stages; conversion | Pipe-and-Filter |
| Huge data set, analysis, parallel on a cluster | Map-Reduce |
| Several views of the same data; the UI changes often | MVC |
| Portability; replace the platform or OS; separation of abstraction levels | Layered |
| Clients must not know where services live; location transparency | Broker |
| Central shared resource; many clients; controlled access | Client-Server |
| Equal nodes, no central server, nodes join and leave | Peer-to-Peer |
| Integrate heterogeneous services from different organisations or languages | SOA |
| Unknown or changing receivers; events; notifications | Publish-Subscribe |
| Deploy on separate hardware (web, application and database servers); firewall zones | Multi-tier |
| Many independent accessors around one persistent store | Shared-Data |

**How to answer:**
1. Name the pattern.
2. Quote the cue from the scenario.
3. Name the QA it serves.
4. Name one weakness and why it is acceptable here.
5. If two patterns fit, say why you rejected the runner-up.

### Practice scenarios
These are my own practice cases, not the original exam text.

1. *A photo app applies resize, then a colour filter, then watermarking, then compression to each uploaded image. Users can choose which steps to run.*
   **Pipe-and-Filter.** It is a sequential transformation of a data stream, and the steps are independent and recombinable (modifiability and reuse). Its weakness, poor interactivity, does not matter for batch processing.
2. *A game studio wants to compute weekly statistics from 5 TB of match logs.*
   **Map-Reduce.** The data set is huge and partitionable, and records can be mapped in parallel (performance and scalability). The overhead is justified at this data volume.
3. *A strategy game shows the same battlefield as a 3D view, a minimap and a unit list. The UI is redesigned every season.*
   **MVC.** There are several synchronised views of one model, and the UI changes often. Observer-based change notification keeps the views consistent.
4. *A mobile game must later run on iOS and desktop, and the team wants to replace the Firebase backend next year without touching gameplay code.*
   **Layered.** It serves portability and modifiability, with a downward-only dependency on a backend abstraction. Client-Server describes the runtime interaction but does not answer the *code-structure* question.
5. *In-game achievements, sound effects and analytics must all react to game events, and new reactors are added by different developers.*
   **Publish-Subscribe.** Receivers are unknown and changing, and publishers must stay unchanged (modifiability). MVC was rejected because there is no UI/model separation question here; the receivers are independent reactors. The latency cost is negligible within a single process. (Side note: pub-sub beats a plain GoF Observer here because the bus decouples even the subject from its listeners.)

---

## 7. Also seen in the 4th ed. and in practice (beyond the classic course list)

These are not in the 3rd-ed. ch. 13 catalogue. Use them for context, or to justify a design, but label them as beyond the classic list in exam answers.

| Pattern | One-line gist | Main QA |
|---|---|---|
| Microservices | Small, independently deployable services, each owning its data, communicating over the network | Deployability, modifiability; costs performance and operational complexity |
| Microkernel / plug-in | A minimal core plus plug-ins bound at runtime or load time (POSA Microkernel) | Modifiability, extensibility |
| Circuit breaker | Stops calling a failing remote service after repeated failures and fails fast until it recovers | Availability |
| Load balancer | Distributes requests over a pool of replicas | Performance, availability |
| Service mesh | Sidecar proxies handle inter-service communication: routing, retries, TLS, telemetry | Performance, availability, security, observability |
| Event sourcing | Stores state as an append-only log of events and derives the current state by replaying it | Auditability, testability (replay); costs complexity |

In the 4th ed. several of these appear in the QA chapters' Patterns sections (see the edition note at the top for the chapter mapping).

---

## 8. Coplien (1998): "Software Design Patterns: Common Questions and Answers"

**Citation:** Coplien, J. O. (1998). Software design patterns: Common questions and answers. In L. Rising (Ed.), *The Patterns Handbook: Techniques, Strategies, and Applications*. Cambridge University Press, pp. 311-320. It is a syllabus article; see `foundations.md` §8 for the list.

The article is a Q&A introduction to what patterns are and where the software patterns movement comes from, namely the architect Christopher Alexander.

**What a pattern is.** A pattern names a recurring problem in a context together with its solution. Following Alexander, it is both *the thing* (the structure) and *the directions for making the thing*. A pattern captures proven, non-obvious design knowledge used by experienced designers, and explains *why* the solution works, not just *what* it is.

**Generative vs non-generative.**

| | Non-generative | Generative |
|---|---|---|
| Nature | Descriptive, passive | Prescriptive, active |
| Role | Records structures observed in existing systems | Guides how to build: applying it helps generate the system and its properties |
| Coplien's interest | | The article stresses generative patterns (check the text for his exact position) |

**[correction]** Wikipendium equates non-generative patterns with "Gamma patterns", meaning the GoF. There is no confirmation that the article says this, so do not repeat it.

**Alexandrian form**, the classic way to write a pattern:

1. **Name**: a short, memorable handle that becomes shared vocabulary.
2. **Problem**: the recurring problem the pattern addresses.
3. **Context**: the situation in which the problem occurs and the pattern applies.
4. **Forces**: the competing concerns and trade-offs the solution must balance.
5. **Solution**: the structure and how to build it.
6. **Examples**: known uses showing the pattern applied.
7. **Resulting context**: the state after applying the pattern, including what is still unresolved, which often leads to other patterns.
8. **Rationale**: why it works and where it came from.

**Pattern language.** A pattern language is a structured collection of patterns that build on each other. Applied in sequence, each pattern's resulting context is the next one's context, so applying them in order moves a design from requirements to a complete architecture. That is more than a *catalogue*, which is an unordered list.

**Idioms vs patterns.** An **idiom** is a low-level pattern tied to one programming language (e.g. a C++ technique). Coplien prefers language-independent design patterns, so an idiom can be recast as a more general pattern. The article illustrates the idea with a C++ example written in Alexandrian form.

**Exam-ready contrast:**
- The SAiP triple is **context / problem / solution**.
- The Alexandrian form adds **name, forces, examples, resulting context and rationale**.
- GoF uses its own template: intent, motivation, applicability, structure, participants, consequences, and so on.

If asked for "the 3 parts of a pattern description", answer **context, problem and solution**, and define each one.
