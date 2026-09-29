# Exam preparation: question types, worked answers, pitfalls, glossary

Use this file to coach students for the TDT4240 written exam and to check exam-style answers. The theory lives in the other reference files. This file holds exam strategy, model answers, common mistakes and drill material.

Cross-links: [foundations](foundations.md) · [requirements-and-design](requirements-and-design.md) · [quality-attributes-classic](quality-attributes-classic.md) · [quality-attributes-4th-edition](quality-attributes-4th-edition.md) · [architectural-patterns](architectural-patterns.md) · [design-and-game-patterns](design-and-game-patterns.md) · [documentation](documentation.md) · [evaluation](evaluation.md) · [platforms-and-emerging-topics](platforms-and-emerging-topics.md) · [course-and-project-guide](course-and-project-guide.md)

> Parts of the glossary, the tactic lists and the pattern material are adapted from the Wikipendium TDT4240 compendium (https://www.wikipendium.no/TDT4240_Software_Architecture, CC BY-SA 3.0, by its contributors). The text has been paraphrased, restructured, corrected and extended. SAiP means Bass, Clements & Kazman, *Software Architecture in Practice*. The 4th edition (2021) is current. The public exam papers cite the 3rd edition (2013).

---

## 1. What is known about the exam

| Fact | Status |
|---|---|
| 4-hour written school exam in Inspera | Confirmed on the NTNU course pages for 2025 to 2027 |
| The only aid is a digital appendix in Inspera that summarises the syllabus | Confirmed (current) |
| 50% of the grade (the group project is the other 50%), and both parts must be completed | Confirmed from Spring 2025. It was 40/60 exam/project for about 2020 to 2024, and about 70/30 around 2008 to 2011 |
| A re-sit may be oral | Stated on the course pages |
| Item types in the current Inspera exam (multiple choice or written, point split) | **Unknown.** No public paper after 2016 was found |

**Public old papers (2015, 2016).** Both were 4 hours long and followed the same structure:

1. **About 20 short questions.** Definitions, lists, "explain the difference" and small diagrams.
2. **Pattern selection.** Choose the most suitable architectural pattern for each of 5 scenarios. The candidates were Layered, Broker, MVC, Pipe-and-Filter, Client-Server, Peer-to-Peer, SOA, Publish-Subscribe, Map-Reduce and Multi-tier.
3. **Essay topics.** In 2015 these were edge-dominant systems (3rd ed. ch. 27, Metropolis model) and cloud (deployment models public, private, community and hybrid, plus base mechanisms).
4. **QA scenarios.** Write quality attribute scenarios for 3 QAs. The 2015 paper asked for usability, modifiability and security.
5. **Architecture design, 30 points.** Given a system description, give the ASRs, tactics and patterns, a process view, a logical view and the rationale.

Tell students plainly: *the format after 2016 is not public.* Practise all five item types, but do not promise that the current exam uses them. Aids have also changed. In 2015 and 2016 printed copies of IEEE 1471 and Kruchten were allowed, in 2022 to 2024 all printed material was allowed, and now only the digital appendix is allowed. So students must know the definitions by heart.

---

## 2. Strategy per question type

Timings below assume the 2015/16 structure scaled to 240 minutes. Scale them to the point weights on the actual paper. A good rule is about 1 minute per point, which leaves roughly 10 minutes to read and 10 to check.

| Type | Time (guide) | Answer skeleton |
|---|---|---|
| Short question | 2-4 min each (about 60 min for 20) | Definition in one sentence, then the list or parts (named exactly), then a one-line example or consequence. Stop there. |
| Pattern selection | 3 min each (about 15 min) | Pattern name, then the scenario cue that points to it, then the QA it serves, then (optionally) the runner-up and why it was rejected. |
| Essay (cloud, edge) | 20-25 min | Define the concept, then its key concepts or models (list), then the architectural implications per QA, then an example, then a short conclusion. |
| QA scenarios | 5-7 min each (about 20 min) | A six-part table with **one concrete** scenario per QA. Every cell must be specific, and the response measure needs a number and a unit. |
| Design task (30 pts) | 60-75 min | (a) ASRs (functional, quality, constraints), (b) a mini utility tree, (c) tactics per QA, (d) patterns with justification, (e) logical view, (f) process view, (g) rationale linking (c)-(f) back to (a). |

**Pattern-selection cue table**

| Scenario cue | Pick |
|---|---|
| Data transformed in stages, streaming, batch conversion | Pipe-and-Filter |
| Clients unaware of server location, many services, need for a middleman | Broker |
| Several UIs over the same data that must stay in sync | MVC |
| Central shared resource, many requesters | Client-Server |
| No central authority, equal nodes, resilience through decentralisation (file sharing) | Peer-to-Peer |
| Heterogeneous systems from different organisations, published service contracts | SOA |
| Producers and consumers that must not know each other, events | Publish-Subscribe |
| Huge data set, parallel analysis on a cluster | Map-Reduce |
| Deploy presentation, business and data parts on separate machines | Multi-tier |
| Separate portability or abstraction levels in code, strict downward use | Layered |

**Design-task checklist.** Name every tactic with its book name. Name every pattern and say which tactic(s) it realises. Each view needs a legend or a list of elements. Every QA you claim must appear in the rationale. State the assumptions you made about gaps in the problem text.

---

## 3. Worked short answers

These are model answers, 3 to 6 lines each. The details are in the linked reference files.

**SAiP definition of software architecture.** The software architecture of a system is the set of structures needed to reason about the system. Those structures are made up of software elements, the relations among them, and the properties of both. Consequences: every system has an architecture, not every structure is architectural, and the architecture is an abstraction that leaves out details that do not matter for reasoning. (The IEEE 1471 definition is different: "fundamental organisation ... embodied in its components, their relationships ... and the principles guiding its design and evolution".)

**Purpose of module, C&C and allocation views.** *Module views* show implementation units (code or data) and relations such as is-part-of, uses and is-a. They support work assignment, modifiability, reuse and impact analysis. *Component-and-connector views* show runtime elements and their interactions. They support reasoning about runtime QAs such as performance, availability and security. *Allocation views* map software elements to the environment (hardware, file system, teams). They support reasoning about deployment, resource use and work assignment.

**4+1 development view (Kruchten).** It shows how the software is organised in the development environment: modules, subsystems, layers and libraries. Its audience is programmers and software managers. It supports reuse, portability, build and configuration management, and dividing work among teams. It is typically shown as a layered subsystem diagram with import and export rules.

**Architecture influences (3rd ed. influence cycle).** An architecture is shaped by its stakeholders' requirements and business goals, by the developing organisation (structure, skills, schedule, existing assets), by the technical environment (available technology and standards), and by the architect's own experience and education. The cycle closes because the finished system in turn influences the organisation, the business goals, customer expectations, the technical environment and the architect's future experience.

**Reasons architecture matters (pick 4-6 and explain each).** It enables or inhibits the driving QAs. It supports reasoning about and managing change. It allows early prediction of qualities. It is a vehicle for stakeholder communication. It carries the earliest and hardest-to-change decisions. It constrains the implementation. It shapes (and is shaped by) the organisation. It supports cost and schedule estimation. It is a reusable core for product lines. It channels developer creativity. It is a basis for training.

**Three groups of availability tactics.** *Detect faults* (ping/echo, heartbeat, monitor, timestamp, sanity checking, condition monitoring, voting, exception detection, self-test). *Recover from faults*, split into preparation and repair (active, passive and cold redundancy, exception handling, rollback, software upgrade, retry, ignore faulty behaviour, degradation, reconfiguration) and reintroduction (shadow, state resynchronization, escalating restart, non-stop forwarding). *Prevent faults* (removal from service, transactions, predictive model, exception prevention, increase competence set).

**Template Method class diagram (in text).**
```
AbstractClass (abstract)
  + templateMethod()          // final; calls step1(), step2(), hook() in fixed order
  # step1()  {abstract}
  # step2()  {abstract}
  # hook()   {default: empty}
        ▲ (generalisation)
ConcreteClassA: overrides step1(), step2()
ConcreteClassB: overrides step1(), step2(), hook()
```
It is a *behavioral* GoF pattern: the invariant algorithm skeleton sits in the superclass and subclasses supply the variable steps ("Hollywood principle": don't call us, we'll call you).

**ASR.** An architecturally significant requirement is a requirement that has a profound effect on the architecture: the architecture would probably be quite different without it. ASRs are usually QA requirements, sometimes functional requirements or constraints. They are found in requirements documents, by interviewing stakeholders (for example in a Quality Attribute Workshop), from business goals, and through a utility tree.

**Performance-model parameters** (3rd ed. ch. 14, QA modelling; *verify the exact list against the book*). For a queuing model: the arrival rate of requests, the queuing discipline, the scheduling algorithm, the service time per request, the topology (how the queues and servers are connected), network bandwidth, and the routing algorithm. The model predicts latency and throughput, and shows where a bottleneck forms.

**ADD (Attribute-Driven Design).** A method that designs an architecture iteratively, driven by ASRs. 3rd-ed. loop: choose an element to design, identify the ASRs for it, generate a design solution (patterns and tactics), verify and refine the requirements into constraints on the child elements, then repeat. 4th-ed. ADD 3.0 steps: review inputs; set the iteration goal by choosing drivers; choose elements to refine; choose design concepts; instantiate elements, allocate responsibilities and define interfaces; sketch views and record decisions; analyse and review against the iteration goal.

**Architectural erosion.** The implemented (descriptive, as-built) architecture drifts away from the intended (prescriptive, as-designed) architecture as changes are made without respecting its rules. Perry & Wolf separate *drift* (insensitivity to the architecture) from *erosion* (outright violations). Symptoms include layer bypasses, upward or cyclic dependencies, and a documentation that no longer matches the code. Remedies: conformance checking, reconstruction, and refactoring.

**Keeping code and architecture consistent** (3rd ed. ch. 19). (1) *Embed the design in the code*: name packages, modules and classes after the architectural elements, and annotate with the element each one implements. (2) *Frameworks* that enforce the structure. (3) *Code templates* (skeletons) for element types, so that for example every component gets the same fault-handling scaffold. (4) *Conformance checking*: static analysis and dependency rules, plus architecture reconstruction. (5) Beyond the book: *fitness functions* in CI, meaning automated checks that fail the build on a forbidden dependency.

**ATAM, abbreviation and outputs.** Architecture Tradeoff Analysis Method (SEI). Outputs: a concise presentation of the architecture; the business goals; prioritised QA requirements expressed as scenarios; a utility tree; a mapping of architectural decisions to QA requirements; the sets of risks and non-risks; sensitivity points and tradeoff points; and risk themes that are tied back to the business goals. There are also intangible benefits, such as stakeholder communication.

**Architecture reconstruction** (3rd ed. ch. 20). Recovering the as-built architecture from an existing system. Phases: *raw view extraction* (parse code, traces and builds), *database construction* (store the extracted facts), *view fusion and manipulation* (combine and abstract the facts into architectural views), and *architecture analysis* (conformance to the intended design). It is used for documentation, conformance checking and migration.

**Software product lines** (3rd ed. ch. 25). A set of software-intensive systems that share a common, managed set of features and are developed from a common set of core assets in a prescribed way. The architecture is the central core asset. It contains explicit *variation points* (with mechanisms such as inheritance, configuration, plug-ins and build-time selection). Benefits: lower cost per product and faster time to market. The main cost is the up-front investment in variability.

**Four groups of modifiability tactics.** *Reduce the size of a module* (split module). *Increase cohesion* (increase semantic coherence). *Reduce coupling* (encapsulate, use an intermediary, restrict dependencies, refactor, abstract common services). *Defer binding* (bind values later: at compile, build, deploy, startup or run time, for example through configuration files, plug-ins or publish-subscribe).

**Sensitivity point vs tradeoff point.** A *sensitivity point* is a property of one or more components or relationships that is critical for achieving one particular QA response. For example, the level of encryption is a sensitivity point for security. A *tradeoff point* is a sensitivity point for **more than one** QA, where the effects pull in different directions. For example, the encryption level improves security but hurts performance.

**ATAM vs CBAM, and the CBAM process.** ATAM finds risks, sensitivity points and tradeoffs, but ignores cost. CBAM (Cost Benefit Analysis Method) builds on ATAM's output to put economic value on strategies. Steps: (1) collate scenarios and keep the top third; (2) refine them into worst, current, desired and best response levels; (3) prioritise, each stakeholder spreading 100 votes, and drop the lower half; (4) assign utility to the response levels; (5) develop architectural strategies and their expected responses; (6) find expected utility by interpolation; (7) compute benefit B_i = Σ_j b_ij × W_j; (8) choose strategies by VFC = B_i / C_i within the budget; (9) confirm with intuition.

**Game loop vs frame rate.** The game loop repeats *process input, update the simulation, render*. If update and render are coupled one-to-one, the frame rate is limited by the slowest part (physics, AI, rendering), and game speed changes with hardware. Decoupling helps: use a fixed-timestep update with variable rendering (interpolation), run AI or network work at a lower tick rate or on its own thread, and cap the frame rate. This trades simplicity for stable performance. Relate it to the process view and to the performance tactics (limit event response, introduce concurrency, bound execution times).

**Three parts of a pattern description.** Context (the recurring situation that gives rise to the problem), problem (the problem and the forces or QAs at stake), and solution (element types, interaction mechanisms or connectors, topology and constraints). Coplien's fuller Alexandrian form adds name, forces, resulting context, rationale and examples.

**GoF categories.** *Creational* covers object creation (Singleton, Factory Method, Abstract Factory, Builder, Prototype). *Structural* covers the composition of classes and objects (Adapter, Composite, Decorator, Facade, Proxy, Bridge, Flyweight). *Behavioral* covers interaction and the distribution of responsibility (Observer, State, Strategy, Template Method, Command, Iterator, and others).

**Availability models.** Steady-state availability = MTBF / (MTBF + MTTR), where MTBF is mean time between failures and MTTR is mean time to repair. Five nines (99.999%) allows about 5 minutes of downtime per year. *Markov models* represent the system as states (for example both-up, one-up, both-down) with failure and repair rates as transition probabilities, and solve for the probability of being in an up state. They are useful for comparing redundancy tactics.

**Utility tree.** A top-down prioritisation used in ATAM (and in design). The root is "Utility", then the QAs, then refinements, and the leaves are concrete scenarios. Each leaf is rated (business importance, architectural impact or difficulty) as H/M/L. The (H,H) leaves drive the analysis and the design.

**When to use Composite.** When clients must treat individual objects and compositions of objects uniformly in a part-whole *tree*: scene graphs, GUI containers, file systems, grouped game units. Component declares the common operations, Leaf implements them, and Composite holds children and forwards the operations to them. It is a *structural* GoF pattern.

---

## 4. QA scenario answers (six-part)

Rules: one stimulus per scenario; a *concrete* artifact and environment; a measurable response measure with a number and unit.

### Usability
| Part | Generic (web shop) | Game (mobile quiz) |
|---|---|---|
| Source | End user | First-time player |
| Stimulus | Wants to undo adding an item to the cart | Wants to learn how to join a match |
| Artifact | Cart UI | Lobby screen and tutorial |
| Environment | Runtime, normal operation | First launch, no account yet |
| Response | Offers undo; the cart returns to its previous state | An interactive tutorial shows how to join |
| Response measure | Undo completes within 1 s, in at most 1 click | 90% of test players join a match within 2 min without outside help |

### Modifiability
| Part | Generic | Game |
|---|---|---|
| Source | Developer | Developer on the team |
| Stimulus | Add a new payment provider | Add a new question category or game mode |
| Artifact | Payment module | Game-mode module and question repository |
| Environment | Design/development time | Development time |
| Response | Change made, tested and deployed with no side effects | Mode added without changing the networking or UI core |
| Response measure | At most 2 person-days, and at most 3 classes changed outside the new adapter | At most 1 new class plus 1 registration line, done within 4 hours |

### Security
| Part | Generic | Game |
|---|---|---|
| Source | Unauthenticated external attacker | Malicious player with a modified client |
| Stimulus | Tries to read other users' orders | Sends a forged "correct answer" or score update |
| Artifact | Order API and database | Game server and score store |
| Environment | Online, normal operation | Ranked match in progress |
| Response | Request denied and logged; admin notified | Server rejects the unvalidated score, logs it and flags the account |
| Response measure | 100% of unauthorised requests denied; detection within 1 min | 100% of forged scores rejected; flagged within 5 s |

### Availability
| Part | Generic | Game |
|---|---|---|
| Source | Internal hardware | Game server process |
| Stimulus | Database node crashes | Match server crashes |
| Artifact | Data tier | Match session state |
| Environment | Normal operation | Match in progress, peak evening load |
| Response | Fail over to a warm standby, log and notify | Clients reconnect to a new instance; match resumed from the last checkpoint |
| Response measure | Downtime under 30 s; no committed data lost | Players back in the match within 10 s; at most 1 question lost |

### Performance
| Part | Generic | Game |
|---|---|---|
| Source | Users (external) | 8 players in a match |
| Stimulus | 1,000 search requests/s (stochastic) | All answer the same question within 1 s |
| Artifact | Search service | Answer-handling server logic |
| Environment | Peak load | Normal operation over mobile 4G |
| Response | Requests processed | Answers ranked and the result shown to all |
| Response measure | 95th-percentile latency under 300 ms | Results on every screen within 500 ms of the last answer |

### Testability
| Part | Generic | Game |
|---|---|---|
| Source | Unit tester (CI) | Developer |
| Stimulus | Completes a code increment | Finishes the scoring component |
| Artifact | Pricing component | Scoring logic |
| Environment | Development, CI pipeline | Development time |
| Response | Tests run automatically; state observable | Scoring tested headless with a fake backend |
| Response measure | 85% branch coverage; the suite runs in under 5 min | 100% of scoring rules covered; runs in under 30 s without a device |

Interoperability (3rd ed.) or Integrability (4th ed.) follows the same shape. For example: the game integrates a new leaderboard backend; the measure is at most 1 adapter class and 1 day.

---

## 5. Worked design question: a multiplayer mobile quiz game

**Problem (invented practice task).** Up to 8 players join real-time quiz matches on Android phones. Questions come from a server. There is a global leaderboard. New question packs and game modes will be added often. Constraints: Android, libGDX, a cloud backend (for example Firebase), and a 4-person team with 10 weeks.

**(a) ASRs**
- Functional: create or join a match via a code; synchronised question rounds; scoring; leaderboard; login.
- Quality: modifiability (new modes and packs), performance (synchronised rounds), availability (surviving dropped connections), security (anti-cheat on scores), usability (join in a few taps).
- Constraints: Android and libGDX, a managed cloud backend, team size and deadline.

**(b) Utility tree (excerpt)**

| QA | Refinement | Scenario | (BI, AI) |
|---|---|---|---|
| Modifiability | New game mode | Add a mode in at most 1 day, with no changes to network code | (H,H) |
| Performance | Round sync | Question shown on all devices within 300 ms of each other | (H,H) |
| Availability | Reconnect | A player dropped for 10 s or less rejoins with the score kept | (H,M) |
| Security | Score integrity | 100% of client-sent scores are recomputed server-side | (M,M) |
| Usability | Join | Join a match in at most 3 taps and 20 s | (M,L) |

**(c) Tactics per QA**
- Modifiability: encapsulate (a `Backend` interface hides Firebase); use an intermediary (an event bus between game logic and UI); abstract common services (a networking service); defer binding (game modes registered through a factory or configuration; question packs loaded as data at runtime).
- Performance: reduce overhead (send small deltas, not full state); introduce concurrency (network I/O off the render thread); bound execution times / manage sampling rate (fixed-timestep update); maintain multiple copies of data (cache the question pack locally before the match).
- Availability: heartbeat (presence detection); retry with backoff; state resynchronisation on reconnect; passive redundancy is provided by the managed backend.
- Security: authenticate actors; authorise actors (database rules); validate answers server-side (the authoritative server computes the score); limit exposure (clients write only to their own answer node).
- Usability: maintain a task model (a guided join flow); support user initiative (cancel matchmaking).

**(d) Patterns and justification**
- *Client-Server* (with the managed backend as the server). Gives one authoritative state for fairness and security, and is simpler than P2P.
- *MVC* on the client. Separates the libGDX rendering (View) from the game state (Model) and input handling (Controller), which supports modifiability and testability.
- *Publish-Subscribe* between backend listeners and the Model. Keeps the model loosely coupled from network events, which supports modifiability.
- *Layered* client code (presentation, game logic, services/backend adapter). Allows swapping Firebase, which supports modifiability and portability.
- Design patterns: *State* (screens and match states: Lobby, Question, Reveal, Results), *Factory Method* (game modes), *Observer* (model to view), *Singleton* only for the asset manager, with the testability cost noted.

**(e) Logical view (in text).** Main abstractions: `Match` (id, players, current round, state), `Player`, `Question`/`QuestionPack`, `GameMode` (interface: `nextQuestion()`, `score(answer)`) with implementations `ClassicMode` and `SpeedMode`, `ScoreService`, `Leaderboard`. `MatchController` handles input and calls `Match`. `MatchView` observes `Match`. `BackendService` is an interface implemented by `FirebaseBackend`, an external component shown at the boundary. `GameStateManager` holds a stack of `Screen` states. Show the tactics on the diagram: the interface (encapsulate), the factory (defer binding), and the observer arrows (intermediary).

**(f) Process view (in text).** Processes and threads: the libGDX render thread (input, then update, then render at 60 fps); a network thread (backend listeners and callbacks); the server-side function (validates answers and computes scores). Round sequence: the host's round timer expires; the server publishes the question id and deadline; clients' listeners receive it and post an event to the render thread; players answer; clients write their answers; the server function scores them and publishes the results; clients update the Model and the View re-renders. On a dropped connection: heartbeat loss, then the retry loop, then resync with the current round snapshot.

**(g) Rationale.** Modifiability is the top business driver (frequent new content), so every variation point sits behind an interface or factory, and content is data, not code (defer binding). The authoritative server trades a little latency for security and consistency; the 300 ms target stays reachable because only small messages are sent and packs are preloaded (a tradeoff point between security and performance). MVC and Observer keep libGDX rendering separate from networking, which also lets the logic be tested without a device. Alternatives rejected: P2P (hard to prevent cheating, NAT problems on mobile) and a self-hosted server (conflicts with the team size and deadline constraints).

---

## 6. Pitfalls (common lost points)

| Pitfall | Correct view |
|---|---|
| Listing patterns as tactics | A *tactic* is a single design decision that affects one QA response (for example heartbeat). A *pattern* is a packaged, recurring solution that bundles several tactics (for example Broker) and often trades QAs off. Name both, and say which tactics a pattern realises. |
| Writing a general scenario when a concrete one is asked for | General scenario: system-independent ("a developer wishes to change the UI"). Concrete scenario: specific source, artifact, environment and a measurable number. Exam and project scenarios should be concrete. |
| Unmeasurable response measure ("fast", "easy", "secure") | Give a number and unit: time, percentage, count, effort, or probability. |
| Mixing up layer and tier | A *layer* is a module (code) grouping with a strict downward allowed-to-use relation (module view). A *tier* is a runtime/deployment grouping on separate hardware (allocation or C&C view). One tier can contain several layers. Layer bridging is a documented exception, not "any layer may use any other". |
| Observer vs Publish-Subscribe | Observer (GoF, design level): the subject holds direct references to its observers and calls them. Publish-Subscribe (architectural): publishers and subscribers are anonymous to each other, decoupled by an event bus or broker, possibly distributed and asynchronous. |
| Calling caching "maintain multiple copies of computations" | Caching is *maintain multiple copies of data*. *Copies of computations* means replicated servers or server pools, for example behind a load balancer. |
| "Limit exposure = security by obscurity" | Limit exposure means reducing the attack surface: fewer open ports, services and access points. |
| Template Method filed as structural | It is **behavioral** (GoF). Composite, Adapter, Facade and Decorator are structural. |
| Using 3rd and 4th edition names interchangeably without saying so | 3rd ed.: Interoperability (ch. 6), tactics and patterns chapter 13, CBAM ch. 23, edge ch. 27. 4th ed.: Integrability (ch. 7), new chapters for Deployability, Energy Efficiency and Safety, no separate CBAM or edge chapter. Some tactic names changed (for example *graceful degradation*). Say which edition you use; see [quality-attributes-4th-edition](quality-attributes-4th-edition.md). |
| Mixing usability and performance scenarios | "The screen loads within 1 s" is performance. "The user completes the task in 3 taps" is usability. |
| Putting design decisions in the ASR list | ASRs are requirements. "Use Firebase" is a constraint only if it was imposed; if the team chose it, it is a decision. |
| Runtime interactions in the logical view | Objects interacting at run time belong in the process view. The logical view shows abstractions and static relations. |
| Rationale that does not mention the QAs | Every pattern and tactic must be traced back to a named ASR or scenario. |

---

## 7. Glossary (alphabetical)

| Term | Definition |
|---|---|
| Abstract Factory | Creational pattern: an interface for creating families of related objects without naming concrete classes. |
| ADD | Attribute-Driven Design: an iterative method that designs by choosing patterns and tactics to satisfy ASRs. |
| ADL | Architecture description language: a formal notation for architecture (for example AADL). |
| ADR | Architecture decision record: a short note giving the context, decision, status and consequences (Nygard). Beyond the syllabus. |
| Allocation view | A view that maps software elements to environment elements (hardware, files, teams). |
| Architectural drift | The implementation diverges from the architecture through insensitivity to it. |
| Architectural erosion | The implementation violates the architecture's rules over time. |
| Architectural pattern | A context-problem-solution package of element types and interactions that recurs at system level. |
| Architectural description (AD) | IEEE 1471: a collection of products that document an architecture. |
| Architecture | The set of structures needed to reason about a system: elements, relations and properties (SAiP). |
| ASR | Architecturally significant requirement: a requirement with a profound effect on the architecture. |
| ATAM | Architecture Tradeoff Analysis Method: scenario-based evaluation that finds risks, sensitivity points and tradeoff points. |
| Availability | Whether the system is there and ready to carry out its task when needed, including masking or repairing faults. |
| Broker | Pattern where an intermediary locates servers for clients and forwards requests and replies. |
| Business goal | An organisational objective that motivates architectural requirements. |
| CBAM | Cost Benefit Analysis Method: adds costs and benefits to architectural strategies (builds on ATAM). |
| C&C view | Component-and-connector view: runtime components and their interaction paths. |
| Client-Server | Pattern where clients request services from servers that share resources. |
| Cloud deployment models | Public, private, community, hybrid. |
| Cohesion | How strongly the responsibilities in a module relate; aim high. |
| Component | A runtime element in a C&C structure (SAiP sense). |
| Composite | Structural pattern that treats leaves and part-whole trees uniformly. |
| Concern | A stakeholder interest in the system (IEEE 1471 / 42010). |
| Concrete scenario | A QA scenario made specific to one system, with a measurable response. |
| Connector | A runtime interaction mechanism between components (call, event bus, pipe). |
| Constraint | A design decision with zero degrees of freedom, imposed from outside. |
| Conway's law | System structure mirrors the organisation's communication structure. |
| Coupling | The degree of dependency between modules; aim low. |
| Defer binding | Modifiability tactic: fix values or choices as late in the life cycle as practical. |
| Degradation | Availability tactic: keep critical functions and shed less important ones. |
| Deployability | 4th ed. QA: how easily software is put into operation (predictably, quickly, reversibly). |
| Development view | 4+1 view of static software organisation for programmers and managers. |
| Edge-dominant system | 3rd ed. ch. 27: a system whose value is created at the edge by crowds (open source, wikis); Metropolis model. |
| Element catalogue | The part of a view listing its elements, their properties and interfaces. |
| Encapsulate | Modifiability tactic: explicit interface that hides internals. |
| Factory Method | Creational pattern: subclasses decide which concrete class to create. |
| Fitness function | An automated check that an architectural property still holds (beyond the syllabus). |
| Four contexts | Technical, project life cycle, business, professional (3rd ed. ch. 3). |
| Game loop | The repeated cycle of input, update and render that drives a game. |
| General scenario | A system-independent QA scenario template. |
| GoF | Gang of Four: Gamma, Helm, Johnson and Vlissides, *Design Patterns* (1994). |
| Heartbeat | Availability tactic: a component periodically sends a signal; a missing signal indicates a fault. |
| IaaS / PaaS / SaaS | Cloud service models: infrastructure, platform, software as a service. |
| IEEE 1471-2000 | Recommended practice for architectural description; superseded by ISO/IEC/IEEE 42010. |
| Integrability | 4th ed. QA (successor to interoperability): cost and risk of making separately developed components work together. |
| Interoperability | 3rd ed. QA: the ability of systems to usefully exchange meaningful information. |
| Kruchten 4+1 | Logical, process, development and physical views plus scenarios. |
| Latency | The time between a stimulus's arrival and the system's response. |
| Layered | Module pattern: layers with a strict, one-way allowed-to-use relation. |
| Limit exposure | Security tactic: reduce the attack surface. |
| Logical view | 4+1 view of key functional abstractions for end users. |
| Map-Reduce | Allocation pattern: parallel map and combining reduce over large data sets on a cluster. |
| Markov model | An availability model with states and transition rates used to compute up-state probability. |
| Metropolis model | Kazman and Chen's model of edge-dominant systems: core, periphery, masses. |
| Modifiability | The cost and risk of making changes. |
| Module | An implementation unit of code or data that provides a coherent set of responsibilities. |
| Module view | A view of implementation units and their static relations. |
| MTBF / MTTR | Mean time between failures / mean time to repair. |
| Multi-tier | Allocation pattern: components grouped into tiers deployed on separate infrastructure. |
| MVC | Model-View-Controller: separates UI (view) and input handling (controller) from data and state (model). |
| Non-risk | A decision judged sound for a QA scenario, recorded with the assumption it relies on; if the assumption breaks it becomes a risk (ATAM). |
| Observer | Behavioral pattern: a subject notifies registered observers of state changes. |
| Peer-to-Peer | Pattern of equal peers acting as both clients and servers. |
| Performance | The ability to meet timing requirements. |
| Physical view | 4+1 view mapping software to hardware nodes and networks. |
| Pipe-and-Filter | Pattern of successive data transformations (filters) connected by pipes. |
| Process view | 4+1 view of concurrency, distribution and runtime interaction. |
| Product line | A family of systems built from shared core assets in a prescribed way. |
| Publish-Subscribe | Pattern where anonymous publishers and subscribers communicate through an event bus. |
| QAW | Quality Attribute Workshop: a stakeholder workshop to elicit and prioritise QA scenarios. |
| Quality attribute | A measurable or testable property that indicates how well the system satisfies stakeholder needs. |
| Rationale | The documented reasons for architectural decisions and rejected alternatives. |
| Reconstruction | Recovering the as-built architecture from an implementation. |
| Response measure | The measurable part of a QA scenario that makes it testable. |
| Risk | An architectural decision that may cause a QA response not to be met (ATAM). |
| Risk theme | A group of related risks tied back to business goals (ATAM). |
| Sensitivity point | A property that is critical for achieving one QA response. |
| Shared-Data | Pattern where data accessors communicate through a shared repository. |
| Singleton | Creational pattern: one instance with a global access point; hurts testability. |
| SOA | Service-oriented architecture: consumers use services through published contracts. |
| Stakeholder | Anyone with an interest in the system. |
| State | Behavioral pattern: behaviour delegated to state objects that change as the context's state changes. |
| Tactic | A design decision that influences the response of one QA. |
| Template Method | Behavioral pattern: fixed algorithm skeleton in a superclass, steps in subclasses. |
| Testability | The ease of making the software reveal its faults through testing. |
| Throughput | The number of events processed per unit of time. |
| Token (Rollings & Morris) | A game element the player manipulates, used for token analysis of game architecture. |
| Tradeoff point | A property that is a sensitivity point for more than one QA, with conflicting effects. |
| Usability | How easily users accomplish tasks and the support the system gives them. |
| Utility tree | A hierarchy from utility to QAs to refinements to prioritised scenarios. |
| View | A representation of a set of elements and relations, from the perspective of related concerns. |
| Viewpoint | The conventions for constructing and using a view (purpose, audience, notation). |

---

## 8. Spaced-repetition question bank

Use these as flashcards. Ask the question, wait for the student's answer, then compare it with the model answer.

1. Q: SAiP definition of architecture? A: The set of structures needed to reason about the system: software elements, the relations among them, and the properties of both.
2. Q: The three kinds of structures or views? A: Module, component-and-connector, allocation.
3. Q: The six parts of a QA scenario? A: Source, stimulus, artifact, environment, response, response measure.
4. Q: General vs concrete scenario? A: A general one is system-independent; a concrete one is specific to a system and measurable.
5. Q: The three groups of availability tactics? A: Detect faults, recover from faults, prevent faults.
6. Q: Heartbeat vs ping/echo? A: With heartbeat the monitored component sends signals itself; with ping/echo a monitor asks and waits for a reply.
7. Q: Active vs passive redundancy? A: Active: all replicas process the input (hot spare). Passive: only the primary processes it and sends state updates to the spares (warm spare).
8. Q: The four groups of modifiability tactics? A: Reduce module size, increase cohesion, reduce coupling, defer binding.
9. Q: The two groups of performance tactics? A: Control resource demand, manage resources.
10. Q: Caching belongs to which tactic? A: Maintain multiple copies of data.
11. Q: The four groups of security tactics? A: Detect, resist, react to and recover from attacks.
12. Q: CIA? A: Confidentiality, integrity, availability.
13. Q: The two groups of testability tactics? A: Control and observe system state; limit complexity.
14. Q: The two groups of usability tactics? A: Support user initiative; support system initiative.
15. Q: 3rd-ed. interoperability tactic groups? A: Locate (discover service); manage interfaces (orchestrate, tailor interface).
16. Q: Tactic vs pattern? A: A tactic is one decision for one QA; a pattern bundles several tactics into a recurring solution with tradeoffs.
17. Q: The three parts of a pattern description? A: Context, problem, solution.
18. Q: The GoF categories? A: Creational, structural, behavioral.
19. Q: The category of Template Method? A: Behavioral.
20. Q: When to use Composite? A: When part-whole trees must be treated uniformly by clients.
21. Q: Observer vs publish-subscribe? A: Observer keeps direct references to its observers; pub-sub decouples anonymous parties through a bus.
22. Q: Layer vs tier? A: A layer is a code grouping with downward use; a tier is a runtime or deployment grouping on separate hardware.
23. Q: The 4+1 views and their audiences? A: Logical (end users), process (integrators), development (programmers and managers), physical (system engineers), scenarios (all).
24. Q: The purpose of the +1? A: Scenarios tie the views together and validate the architecture.
25. Q: IEEE 1471: view vs viewpoint? A: A view represents the system for a set of concerns; a viewpoint is the template and conventions for building it.
26. Q: IEEE 1471's successor? A: ISO/IEC/IEEE 42010 (2011, revised 2022).
27. Q: What is an ASR? A: A requirement with a profound effect on the architecture.
28. Q: Sources of ASRs? A: Requirements documents, stakeholder interviews and the QAW, business goals, the utility tree.
29. Q: How is a utility tree leaf rated? A: H/M/L for business importance and architectural impact (or difficulty).
30. Q: What does ATAM stand for? A: Architecture Tradeoff Analysis Method.
31. Q: Name five ATAM outputs. A: Utility tree, risks and non-risks, sensitivity points, tradeoff points, risk themes (also the approaches and the prioritised scenarios).
32. Q: Sensitivity point vs tradeoff point? A: A sensitivity point is critical for one QA; a tradeoff point is critical for several QAs, in conflict.
33. Q: What does CBAM add to ATAM? A: Costs, utility curves and a value-for-cost ranking of strategies.
34. Q: The CBAM value-for-cost formula? A: VFC = B_i / C_i, where B_i = Σ (utility gain × scenario weight).
35. Q: What does ADD stand for, and what drives it? A: Attribute-Driven Design, driven by ASRs, applied iteratively.
36. Q: Erosion vs drift? A: Erosion violates the architecture; drift is insensitivity that lets the implementation wander.
37. Q: Four ways to keep code and architecture consistent? A: Embed the design in the code, frameworks, code templates, conformance checking.
38. Q: The phases of architecture reconstruction? A: Raw view extraction, database construction, view fusion, analysis.
39. Q: What is a product line? A: A family of systems built from shared core assets with managed variation points.
40. Q: The availability formula? A: MTBF / (MTBF + MTTR).
41. Q: Roughly how much downtime per year does five nines allow? A: About 5 minutes.
42. Q: Cloud service models? A: IaaS, PaaS, SaaS.
43. Q: Cloud deployment models? A: Public, private, community, hybrid.
44. Q: The rings of the Metropolis model? A: Core, periphery, masses.
45. Q: Which pattern fits streaming data transformation? A: Pipe-and-Filter.
46. Q: Which pattern fits several synchronised UIs over the same data? A: MVC.
47. Q: Which pattern fits parallel analysis of a huge data set? A: Map-Reduce.
48. Q: Why is Singleton risky? A: Hidden global state, which hurts testability and makes initialisation order implicit.
49. Q: How does a coupled game loop affect frame rate? A: The frame rate is capped by the slowest step; a fixed-timestep update decouples simulation speed from rendering.
50. Q: What makes a response measure acceptable? A: It is observable and quantified, with a number and unit.
51. Q: What is the difference between a constraint and a decision? A: A constraint is imposed with no freedom; a decision is chosen from alternatives.
52. Q: What does Rollings & Morris token analysis do? A: Identify tokens, analyse their interactions, then derive a logical view.
