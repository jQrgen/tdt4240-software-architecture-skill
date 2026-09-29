# ASRs, eliciting quality requirements, and designing an architecture

Covers SAiP 4th ed. (2021) ch. 19 (ASRs), ch. 20 (Designing an Architecture) and ch. 23 (Architecture Debt). Also covers the 3rd-ed. (2013) equivalents ch. 16 (→ 4th ch. 19) and ch. 17 (→ 4th ch. 20), plus 3rd-ed. material with no same-titled 4th-ed. chapter: ch. 13, 14, 15, 19, 20, 22 and 25 (CBAM, ch. 23, is in `evaluation.md`). Chapter numbers come from the two tables of contents. No page numbers are given on purpose. NTNU has not published which edition or chapters are on this year's list, so tell students to check Leganto or Blackboard.

Related files: per-QA general scenarios and tactics are in `quality-attributes-classic.md` and `quality-attributes-4th-edition.md`. The pattern catalogue is in `architectural-patterns.md`. Views and rationale are in `documentation.md`. ATAM and CBAM are in `evaluation.md`. The project templates are `templates/requirements-document.md` and `templates/architecture-document.md`.

Parts adapted from the Wikipendium TDT4240 compendium (CC BY-SA 3.0), <https://www.wikipendium.no/TDT4240_Software_Architecture>. Its ASR section is a single paragraph, so the QAW, utility tree, PALM and ADD material here comes from the textbook.

---

## 1. Three kinds of requirement

| Kind | What it says | Example (multiplayer mobile game) | Architect's freedom |
|---|---|---|---|
| **Functional** | What the system must do and how it reacts to input | "Two players can join the same match via a lobby code" | Many structures can deliver it. Functionality alone rarely decides the architecture |
| **Quality attribute (QA)** | How well it must do it; a qualification of a functional requirement or of the whole product | "A move reaches the opponent's screen within 500 ms at the 95th percentile" | Drives the architecture: picks tactics and patterns |
| **Constraint** | A design decision already made, with zero degrees of freedom | "Must use libGDX", "Android only", "Firebase as backend", "Java or Kotlin" | None. You accept it and record its consequences |

**Constraints are given, not chosen.** They come from outside the design (the course, the customer, the organisation, law, or an existing platform). Rules for students:
- List constraints separately from decisions. If the team *chose* Firebase, that is a decision and needs rationale. If the course *mandates* it, it is a constraint. Never defend a decision by calling it a constraint.
- A constraint still has architectural consequences, so record them (for example, "the COTS framework owns the game loop, so our code runs inside its `render()` callback"). Course feedback asks for the COTS section to describe technical and architectural constraints, interfaces you must implement, and the effect on control flow.
- Tip from the MR 853 model document: use a table with columns `# | Constraint | Consequence`.

## 2. Architecturally significant requirements (ASRs)

**Where:** 4th ed. ch. 19 "Architecturally Significant Requirements"; 3rd ed. ch. 16 "Architecture and Requirements".

**Definition.** An ASR is a requirement that has a profound effect on the architecture: without it, the architecture would probably be quite different. ASRs are mostly QA requirements, plus a few functional requirements and constraints that shape structure. Typical signs: high business value *and* high technical difficulty, or a wide effect across many elements.

**Sources of ASRs:**

| Source | How ASRs are obtained | Caveat |
|---|---|---|
| Requirements documents | Mine them for QA-relevant statements | Usually vague or missing on QAs ("the system shall be fast"). Many ASRs are not written down at all, and much of the document is not architecturally significant |
| Stakeholder interviews and workshops | QAW (section 3); interviews with key stakeholders | Stakeholders need prompting with scenarios to state QAs concretely |
| Business goals | Derive QAs from why the system is being built; PALM (section 5) | Goals must be turned into measurable scenarios |
| Utility tree | Architect-led top-down refinement (section 4) | Useful when stakeholders are unavailable; also a step in ATAM |

**Rule: ASRs are requirements, not design decisions.** This comes up again and again in TDT4240 project feedback.

| Wrong (a design decision in the ASR list) | Right (a requirement) |
|---|---|
| "Use MVC to separate the UI" | "A new game screen can be added by one developer in under 4 hours without changing the game logic" |
| "Use Firebase Realtime Database" (unless mandated, when it is a constraint) | "Game state is synchronised between two players within 500 ms" |
| "Implement the State pattern for game states" | "Adding a new game mode changes at most 3 classes" |
| "Use an interface to abstract the backend" | "The backend provider can be replaced within 2 person-weeks" |

Patterns and tactics are the *answer* to ASRs and belong in the tactics, patterns and rationale sections. A related feedback point: **do not list patterns as tactics** (section 7).

**Checklist for the "Architectural drivers / ASRs" section of the project architecture document:**
- [ ] Split into functional, quality and business drivers (plus constraints, if the template asks for them).
- [ ] Every quality ASR points to a concrete six-part scenario in the requirements document (for example "see M1, P2").
- [ ] No pattern, framework or class name appears unless it is a mandated constraint.
- [ ] The primary QA (often modifiability in recent example projects; follow your own assignment) comes first.
- [ ] Each ASR is later traced to at least one tactic or pattern in the rationale.

## 3. Quality Attribute Workshop (QAW)

A facilitated, stakeholder-centred method from the SEI. It elicits QA requirements *before* the architecture exists, using scenarios. It is system-centric and does not evaluate a design. Steps in order:

| # | Step | What happens |
|---|---|---|
| 1 | **QAW presentation and introductions** | Facilitators explain the method's purpose and steps; everyone introduces themselves |
| 2 | **Business/mission presentation** | A stakeholder representing the business presents the system's business context, goals, and high-level functional requirements, constraints and QAs |
| 3 | **Architectural plan presentation** | The architect presents the architectural plans as they currently stand: known technical constraints, other systems to interact with, and planned approaches |
| 4 | **Identification of architectural drivers** | Facilitators distil a list of key drivers (ASRs, business drivers) from steps 2 and 3 and check it with stakeholders |
| 5 | **Scenario brainstorming** | Each stakeholder proposes scenarios; facilitators make sure each driver has at least one |
| 6 | **Scenario consolidation** | Similar scenarios are merged, so votes are not split between near-duplicates |
| 7 | **Scenario prioritisation** | Each stakeholder gets a vote budget (a common rule is about 30% of the scenario count, rounded up) and votes, possibly in rounds |
| 8 | **Scenario refinement** | The top scenarios are refined into the six-part form, and business goals, relevant QAs and open questions are recorded |

**Outputs:** a prioritised list of refined QA scenarios, plus a list of drivers and raw scenarios that feed the utility tree, ADD, and later ATAM. Treat the voting fraction as typical, not a fixed rule. The exact wording of the step titles differs slightly between sources.

## 4. The utility tree

A top-down way to organise and prioritise QA requirements. It is also ATAM step 5 (see `evaluation.md`).

```
Utility                               (root: overall "goodness" of the system)
 └─ Quality attribute                 (e.g. Performance)
     └─ Attribute refinement          (e.g. Latency of game moves)
         └─ Concrete scenario (X,Y)   (a six-part-ish scenario, rated)
```

Each leaf gets two ratings, each H/M/L:
- **Business value / importance to stakeholders** (first letter);
- **Architectural impact / technical difficulty or risk** (second letter).

(H,H) leaves are the ASRs that need the most architectural attention. (H,L) leaves are important but easy. (L,*) leaves can wait. Many ATAM write-ups use the order (importance, difficulty), but state which order you use, because sources differ.

### Worked example: two-player online mobile game (libGDX + Firebase-style backend)

| QA | Refinement | Concrete scenario | (Value, Difficulty) |
|---|---|---|---|
| Performance | Move latency | When a player makes a move during a normal match, the opponent's client shows it within 500 ms in 95% of moves | (H, H) |
| Performance | Frame rate | During gameplay with 50 animated sprites on a mid-range Android phone, the game renders at 60 FPS or more in 95% of frames | (H, M) |
| Modifiability | New game mode | A developer adds a new game mode at design time, changing at most 3 existing classes, within 2 person-days, with no new defects in existing modes | (H, M) |
| Modifiability | Backend portability | The team replaces the backend service with another provider in at most 2 person-weeks, without changing game-logic or UI code | (M, H) |
| Availability | Lost connection | When a player's network drops for 10 s during a match, the client reconnects and resumes the same match state with no lost moves | (H, H) |
| Usability | Learning | A first-time player completes the tutorial and starts a match within 3 minutes, without external help, in 90% of test sessions | (M, L) |
| Security | Cheating | When a modified client sends an illegal move, it is rejected server-side and logged in 100% of cases | (M, M) |

Reading it: design effort goes first to move latency and reconnection (H,H), then backend portability. That ordering should show up in the tactics and rationale sections of the architecture document.

## 5. PALM (a short mention)

**PALM (Pedigreed Attribute eLicitation Method)** is the SEI method in 3rd ed. ch. 16 for eliciting business goals from stakeholders (typically senior management), in a workshop of roughly a day. Business goals are captured in a structured *business-goal scenario*: who holds the goal, what it is about, how it is measured, and its "pedigree" (where it came from and how much it is worth). Each goal is then linked to the QA requirements it implies. Stakeholders can also be prompted with a standard catalogue of business-goal categories. The purpose is to make QA requirements traceable to real business reasons rather than stated in a vacuum. (One source used during research expanded PALM as "Pragmatic Architecture Lifecycle Method"; the SEI expansion is the one above. Check the exact number of scenario fields in the edition you use.)

## 6. Six-part QA scenarios

| Part | Question | Game example (reconnection) |
|---|---|---|
| **Source (of stimulus)** | Who or what generates it? | The mobile network |
| **Stimulus** | What event or condition arrives? | The connection drops for 10 s mid-match |
| **Artifact** | What part is stimulated? | Client networking module and match-state sync |
| **Environment** | In what mode or state? | Normal operation, match in progress |
| **Response** | What does the system do? | Pauses input, shows "reconnecting", retries, resynchronises match state |
| **Response measure** | How is the response judged, in numbers? | Match resumes within 5 s of the network returning; 0 moves lost |

- **General scenario:** system-independent. It lists the possible values of each part for a QA (for example, availability stimulus: omission, crash, incorrect timing, incorrect response). The general scenarios are catalogued in the QA chapters; see `quality-attributes-classic.md` (3rd ed.) and `quality-attributes-4th-edition.md` (4th ed.).
- **Concrete scenario:** a general scenario instantiated for one system, with specific values. The project requirements document needs concrete ones.
- **The response measure must be measurable** (a number with a unit and a threshold): time, percentage, count of changed modules, person-hours, FPS. "Fast", "easy", "robust" and "user-friendly" are not measures. As the MR 853 document puts it, a goal without a measurable response is an aspiration, not a requirement.
- Common course mistakes: mixing up usability and performance scenarios (perceived speed of a UI task can be either; decide whether the concern is *timing* (performance) or *user effort, learning or error recovery* (usability)); writing the tactic into the response ("uses a cache"); and leaving the environment empty.

## 7. Tactics vs patterns

| | Tactic | Pattern |
|---|---|---|
| Scope | One design decision that affects **one QA response** | A package of design decisions that **bundles several tactics** |
| Granularity | Primitive building block | Composite; a known solution with a known context |
| Example | Heartbeat, Encapsulate, Introduce concurrency, Authenticate actors | Client-server, Layered, Publish-subscribe, MVC, Broker |
| Trade-offs | Usually focused on one QA; may hurt others | Explicitly trades several QAs at once |

**A pattern is a {context, problem, solution} triple** (3rd ed. ch. 13; adapted from the Wikipendium compendium, CC BY-SA 3.0):
- **Context:** a recurring situation in the world that gives rise to a problem.
- **Problem:** the problem that arises in that context, often including the QAs to be met.
- **Solution:** element types, interaction mechanisms or connectors, their topological layout, and semantic constraints on topology, behaviour and interaction.

Patterns are discovered in practice, not invented, and real systems use several at once.

**Augmenting a pattern with tactics.** Applying a pattern has side effects on other QAs. Further tactics repair these, and the result is the pattern *augmented* by tactics. Example: Broker gives modifiability and interoperability, but it creates a single point of failure (availability) and adds a hop (performance). The fixes are *active or passive redundancy* for the broker, plus *heartbeat* to detect its failure, and possibly *maintain multiple copies of computations* (a broker pool) with load balancing. Each added tactic can in turn bring new side effects, so you keep going until the side effects are acceptable. In ch. 13 the 3rd ed. works an example like this. The 4th ed. no longer has a separate tactics-and-patterns chapter; patterns appear inside each QA chapter.

**Do not list patterns as tactics** in the project document. "MVC" goes under patterns; "Encapsulate", "Use an intermediary" and "Restrict dependencies" go under modifiability tactics. The rationale connects the two.

## 8. Attribute-Driven Design (ADD)

**Where:** 4th ed. ch. 20 "Designing an Architecture" (ADD 3.0); 3rd ed. ch. 17 (an earlier ADD formulation whose step wording differs). *Verify the step wording against the edition you use.*

ADD is iterative. Each iteration takes a few drivers, picks the elements to refine, and applies design concepts (patterns, tactics, reference architectures, externally developed components) to them.

**Inputs:** design purpose, primary functional requirements, QA scenarios (prioritised, for example by a utility tree), constraints, and architectural concerns. **Output:** a sketch of the architecture (views, decisions and rationale) that is refined over iterations.

| Step | 4th ed. (ADD 3.0), paraphrased |
|---|---|
| 1 | **Review inputs** (purpose, requirements, QAs, constraints, concerns) |
| 2 | **Establish the iteration goal by selecting drivers** |
| 3 | **Choose one or more elements of the system to refine** (the whole system in the first iteration) |
| 4 | **Choose one or more design concepts that satisfy the selected drivers** |
| 5 | **Instantiate architectural elements, allocate responsibilities, and define interfaces** |
| 6 | **Sketch views and record design decisions** (with rationale) |
| 7 | **Perform analysis of the current design and review the iteration goal and achievement of the design purpose** |
| - | Iterate from step 2 until the design purpose is met or budget runs out |

Exam tips: be able to list the steps, and explain why ADD is driven by QAs rather than functionality (functionality can be delivered by almost any structure; QAs cannot).

**Worked iteration (the game from section 4):**

| Step | Iteration 1 |
|---|---|
| Drivers | Move latency (H,H), reconnection (H,H); constraints: libGDX, Android, hosted backend |
| Element to refine | The whole system |
| Design concepts | Client-server with an authoritative server-side match state; *Introduce concurrency* (networking off the render thread); *Retry* and *State resynchronization* for reconnection; *Use an intermediary* (a backend interface) for portability |
| Instantiate | Client: `GameScreen`, `MatchController`, `NetworkService` interface, `FirebaseNetworkService` implementation; server: match document or record plus validation rules/functions |
| Views and decisions | Process view (render thread vs network callbacks); physical view (phone, backend, network type); decision records with rejected alternatives (for example peer-to-peer rejected for cheating and NAT reasons) |
| Analysis | Walk the latency and reconnection scenarios through the sketch. Open issue: the cost of a round-trip per move; that becomes the next iteration's driver |

## 9. Related 3rd-edition topics (no 4th-ed. chapter of the same title)

Each item below is 3rd-ed. material. **Check whether it is on this year's list.** The 2015 and 2016 exams asked short questions on several of them (ADD, performance-model parameters, erosion, keeping code and architecture consistent, reconstruction, product lines, availability models).

**Architecture in agile projects (3rd ed. ch. 15).** The question is how much up-front architecture a project needs. The chapter uses Boehm and Turner's analysis: the right amount of up-front architecture work grows with system size, complexity and volatility. Small, stable projects need little, large ones much more. Agile and architecture are compatible. Useful practices: an initial architectural skeleton, spikes to explore risky decisions, and letting the architecture evolve with each iteration while ASRs are known early. Avoid both extremes: "big design up front" and "no design".

**QA modelling and analysis (3rd ed. ch. 14).**
- **Performance, queuing model:** requests arrive, queue, get scheduled and are serviced. The parameters needed are the arrival rate of events, the queuing discipline, the scheduling algorithm, the service time for events, the network topology, the network bandwidth, and the routing algorithm. Results are latency and throughput estimates.
- **Availability, Markov model:** states (for example "both up", "one failed", "both failed") with transition rates (failure rate λ, repair rate μ). Solving it gives steady-state availability. Basic formula: availability = MTBF / (MTBF + MTTR).
- The chapter also covers the choice of analysis technique across the life cycle: thought experiments, back-of-the-envelope analysis, checklists, analytic models, simulation, prototypes, and measurement of the running system. Cost and confidence rise in that order.

**Architecture, implementation and testing (3rd ed. ch. 19).** The concern is keeping code and architecture consistent. Techniques include embedding the design in the code (package and module structure that mirrors architectural elements, annotations or naming), frameworks and code templates that force architectural conventions, and synchronising documentation and code at defined points (for example, at the end of an iteration or release). If you do not synchronise, record known deviations. Testing: the architecture defines the units and the integration order, and gives testers the interfaces to test against. This links to the course project's grading on consistency between code and architecture. A modern equivalent (outside the syllabus) is automated fitness functions in CI, as in the MR 853 document.

**Architecture reconstruction and conformance (3rd ed. ch. 20).** Reconstruction recovers the as-built architecture from an existing system. Activities: raw view extraction (static parsing, dynamic tracing, build files), database construction, view fusion (combining static and dynamic information), and analysis. **Conformance checking** compares the reconstructed (as-built) architecture with the intended (as-designed) one. **Erosion** (also called drift or decay) is the growing gap between the two, caused by changes that ignore the architecture's rules. An example is an upward layer dependency.

**Software product lines (3rd ed. ch. 25).** A product line is a set of systems built from a shared set of core assets, with the architecture as the most important one. **Variation points** are the places where products differ. They are realised by **variation mechanisms**, for example inclusion or omission of elements, build-time selection, parameterisation or configuration, inheritance or specialisation, component substitution, and plug-ins. Variability is a special case of modifiability.

**Management and governance (3rd ed. ch. 22) [verify against the book].** Check whether it is on this year's list. The chapter looks at architecture from the project-management side. The architect owns the technical decisions and the project manager owns budget, schedule and staffing, and the two must work closely together (for example, the architecture's module structure feeds the work breakdown and the estimates). The material is organised around planning, organising, implementing and measuring a project. Organising includes global or distributed development, where module boundaries become team boundaries and interfaces must be stable and well documented because coordination is expensive. Measuring covers tracking progress and architecture-related metrics. **Governance** covers who may make and change architecture decisions and how conformance is checked, for example an architecture review board, design reviews and conformance checks of code against the architecture. The 4th-ed. counterpart is closest to ch. 24 (the role of the architect in projects), see `platforms-and-emerging-topics.md` §8.

## 10. Architecture debt (4th ed. ch. 23)

- **What it is:** a kind of technical debt at the level of structure rather than individual lines of code. Design flaws in how files and modules relate (for example, cyclic dependencies, modularity violations where supposedly independent parts always change together, unstable interfaces) make every later change cost more. That extra cost is the "interest".
- **How it is found:**
  - **Co-change analysis:** mine the version-control history for files that repeatedly change together in the same commits, especially without a structural dependency that explains it.
  - **Hotspots:** groups of architecturally connected files that attract a disproportionate share of bug fixes and change effort.
  - The chapter models dependencies with matrix-style representations and recognises recurring flaw patterns. Check the book for its names and tooling rather than relying on memory.
- **How it is paid down:** quantify the interest (extra effort spent in the hotspot, from the history) against the cost of refactoring. Refactor the flawed structure, for example by breaking cycles, introducing proper interfaces, or splitting and merging modules. Make the business case with those numbers, then measure again to confirm the gain.
- **Link to the project:** "Issues" and "Changes" sections in the project documents are where known debt and deviations should be recorded honestly.
