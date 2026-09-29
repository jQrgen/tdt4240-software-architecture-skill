# Documenting architecture: Views and Beyond, 4+1, IEEE 1471 / ISO 42010, arc42, C4, ADRs

Scope: how to write down an architecture so that each stakeholder can find the answer to their concern.
Syllabus core: **Views and Beyond** (SAiP), **Kruchten 4+1**, **IEEE Std 1471-2000**. arc42, C4 and ADRs are
**beyond the syllabus** but are what practitioners use; they are included so students can write better project documents.

Chapter pointers (numbering differs between editions, so always say which one you use):

| Topic | SAiP 4th ed. (2021) | SAiP 3rd ed. (2013) | Primary source |
|---|---|---|---|
| Documenting an architecture | ch. 22 | ch. 18 | Clements et al., *Documenting Software Architectures: Views and Beyond*, 2nd ed., Addison-Wesley, 2010 |
| Structures and views (module / C&C / allocation) | ch. 1 | ch. 1 | Bass, Clements & Kazman (structure list); Clements et al. 2010 for view styles |
| Quality-attribute scenarios (six parts) | ch. 3 | ch. 4 | Bass, Clements & Kazman |
| 4+1 view model | not a chapter | not a chapter | Kruchten, IEEE Software 12(6):42-50, 1995, DOI 10.1109/52.469759 |
| Architecture description standard | not a chapter | not a chapter | IEEE Std 1471-2000, superseded by ISO/IEC/IEEE 42010 |

Related files (not repeated here): patterns per view type in [architectural-patterns.md](architectural-patterns.md);
ASRs in [requirements-and-design.md](requirements-and-design.md); the six-part scenario format and per-QA scenarios in
[quality-attributes-classic.md](quality-attributes-classic.md) and
[quality-attributes-4th-edition.md](quality-attributes-4th-edition.md); ATAM and utility trees in
[evaluation.md](evaluation.md); the course document skeleton in
[../templates/architecture-document.md](../templates/architecture-document.md); project advice in
[course-and-project-guide.md](course-and-project-guide.md).

---

## 1. Views and Beyond (SAiP 4th ed. ch. 22, 3rd ed. ch. 18)

### 1.1 Key terms

- **Structure**: a set of elements and their relations, as they exist in the system (or the code).
- **View**: a *representation* of a structure (or a set of coherent elements and relations), written for stakeholders.
  An architecture is too complex to capture in a single, one-dimensional description, so it is documented as
  several views (SAiP 4th ed. ch. 22; 3rd ed. ch. 18; Clements et al. 2010).
- **Principle of Views and Beyond**: *documenting an architecture = document the relevant views, then add
  documentation that applies to more than one view.*
- Which views are "relevant" depends on who will read the documentation and what they need to do with it.
  There is no fixed list.

### 1.2 The three view categories and the patterns that fit each

Adapted from the Wikipendium TDT4240 compendium (CC BY-SA 3.0), extended with the book's structure list.
The pattern grouping in the last column follows the SAiP 3rd ed. ch. 13 catalogue; the 4th ed. has no single pattern
catalogue and presents patterns in the quality-attribute chapters (ch. 4-13).

| Category | Elements | Shows / supports reasoning about | Typical structures | Patterns documented in it |
|---|---|---|---|---|
| **Module views** | Modules (code or data units to build or buy): classes, packages, layers | Responsibilities, code organisation, dependencies; modifiability, reuse, work planning | Decomposition, uses, layers, class/generalisation, data model | Layered |
| **Component-and-connector (C&C) views** | Runtime components (processes, objects, clients, servers, stores) and connectors (calls, pipes, pub-sub bus, protocols) | How the system works at runtime; performance, availability, security, concurrency | Service, concurrency | Broker, MVC, Pipe-and-filter, Client-server, Peer-to-peer, SOA, Publish-subscribe, Shared-data |
| **Allocation views** | Software elements mapped to non-software elements (hardware, file systems, networks, teams) | Deployment, build and file layout, who builds what; performance, availability, cost | Deployment, implementation (install), work assignment | Map-reduce, Multi-tier |

Traps to catch in student work:
- A **component** in SAiP is a *runtime* element (C&C), not "a group of modules". Normalise loose uses.
- **Layers** (module view) are not **tiers** (allocation view). A 3-tier deployment diagram is not a layered view.
- MVC drawn as a class diagram is a module view; the runtime notifications between model and view are C&C.

### 1.3 The view template (what goes into *each* view)

| Part | Content | Common failure |
|---|---|---|
| **Primary presentation** | The elements and relations, usually a diagram (plus a legend/key), sometimes a table | Boxes-and-lines with no key: are arrows "calls", "depends on", "sends data to"? |
| **Element catalogue** | Elements and their properties; relations and their properties; element interfaces; element behaviour | Missing entirely, so the diagram's labels are the only description |
| **Context diagram** | What is in scope for this view and what lies outside it (the environment), and how they interact | External services (e.g. Firebase) left off the picture |
| **Variability guide** | Variation points the architecture allows (e.g. choose backend implementation, number of players, platform) and how to exercise them | Omitted even when the requirements ask for portability/modifiability |
| **Rationale** | Why the design in this view is as it is: which ASRs/QAs drove it, alternatives rejected | "We chose MVC because it is good" with no link to a quality goal |

### 1.4 Documentation beyond views (applies across views)

| Item | Purpose |
|---|---|
| **Documentation roadmap** | How the document is organised; which view to read for which concern; stakeholder-to-section map |
| **How a view is documented** | Explains the view template (so readers know what the five parts are) |
| **System overview** | Short description of the system's purpose and function, users, context and constraints |
| **Mapping between views** | How elements in one view correspond to elements in another (e.g. module to component, component to node); tables work well |
| **Rationale** | Cross-cutting decisions that affect more than one view, with constraints and alternatives |
| **Directory** | Index, glossary, acronym list |

IEEE 1471's "record of known inconsistencies" and the TDT4240 template's "Consistency among views" section belong
here too (see section 3).

### 1.5 Choosing views from stakeholders and concerns

A three-step method from the book:
1. **Build a stakeholder/view table**: rows = stakeholders (developers, testers, maintainers, project manager,
   users, operators, integrators, evaluators; in TDT4240 also the **ATAM evaluation group** and the course staff),
   columns = candidate views; mark the level of detail each stakeholder needs (e.g. detail / some detail / overview).
2. **Combine views** to reduce the count: merge marginal views into a closely related one (see 1.7).
3. **Prioritise and stage**: document the views needed early first (decomposition is often first; it drives work
   assignment); you do not have to finish one view before starting the next.

Rule of thumb for the exam: justify *every* view by naming the stakeholders and concerns it serves.
Teacher feedback on course projects asks exactly this: viewpoint purposes should say **why** each view is included.

### 1.6 Quality views

A quality view extracts the parts of other views that matter for one quality attribute, for one audience:

| Quality view | Shows |
|---|---|
| **Security view** | Components with security roles, trust boundaries, data flows carrying sensitive data, where authentication/authorisation happens |
| **Performance view** | Elements and paths that matter for latency/throughput: queues, threads, caches, network hops, timing budgets |
| **Reliability view** | Redundancy, fault detection (heartbeat, ping/echo), recovery and failover mechanisms |
| **Communications view** | All communication channels and protocols, especially in distributed or multi-team systems |
| **Exception (error-handling) view** | How faults and errors are detected, reported and handled; which element owns which failure |

### 1.7 Combining views

- **Overlay / hybrid view**: elements and relations from two views in one diagram (e.g. C&C components placed on
  deployment nodes). Only works if the association between the two is tight and the result stays readable.
- Good candidates to combine: **deployment + C&C** (processes on nodes), **decomposition + work assignment**
  (module = team), **decomposition + uses/layers**.
- Always keep a key; a combined view without a legend is the most common unreadable diagram.

### 1.8 Documenting behaviour

Structure alone does not say how elements interact over time. Two families:

| Family | What it captures | Notations |
|---|---|---|
| **Traces** | Sequences of activities or interactions for one specific stimulus or scenario | Use-case descriptions, **UML sequence diagrams**, communication diagrams, **activity diagrams**, message sequence charts |
| **Comprehensive models** | The complete behaviour of structural elements, all possible sequences | **State machines** (UML state diagrams), and formal models (e.g. process algebras, Petri nets) |

For a game: a sequence diagram for "player joins match" (trace); a state machine for the game-state manager
(Menu, Lobby, Playing, Paused, GameOver) (comprehensive model).

### 1.9 Notations

| Kind | Examples | Trade-off |
|---|---|---|
| **Informal** | Boxes and lines in a drawing tool, natural language | Cheap and flexible; semantics undefined; no analysis |
| **Semiformal** | **UML** (class, component, deployment, sequence, activity, state diagrams), SysML | Standardised syntax, loose semantics; widely understood; some tool analysis |
| **Formal** | **ADLs** (architecture description languages), e.g. AADL, Wright, Acme | Precise semantics, automated analysis; costly, few readers know them |

TDT4240 expects semiformal UML plus prose. Whatever notation: always a key, and say what boxes and arrows mean.

---

## 2. Kruchten's 4+1 View Model (1995)

Kruchten, P. "The 4+1 View Model of Architecture." *IEEE Software* 12(6):42-50, 1995. DOI 10.1109/52.469759.
Syllabus article. The paper predates UML; its own figures use Booch notation and Kruchten-specific icons.
The UML column below is the conventional modern mapping.

| View | Audience (stakeholders) | Concerns | Typical elements | UML notation (modern) |
|---|---|---|---|---|
| **Logical** | End users (and analysts) | Functionality: what services the system gives users; key abstractions of the problem domain | Classes, objects, class categories, associations, inheritance | Class diagrams, object diagrams (state diagrams for class behaviour) |
| **Process** | System integrators | Performance, scalability, throughput, concurrency, distribution, system integrity, fault tolerance | Processes, tasks, threads, and their communication (messages, RPC, events) | Sequence, communication and activity diagrams |
| **Development** (called *implementation* in some later versions) | Programmers, software (project) managers | Organisation of modules in the development environment; ease of development, reuse, portability, build and configuration, work allocation | Subsystems, packages, libraries, layers, modules | Package and component diagrams |
| **Physical** | System engineers | Mapping software onto hardware; topology, communication, availability, reliability, performance, scalability | Nodes (processors, devices), networks, deployed processes | Deployment diagrams |
| **Scenarios (+1)** | All stakeholders | Show that the four views work together; discover and validate the architecture | Use-case instances, sequences of interactions across elements of the views | Use-case diagrams plus sequence diagrams |

### 2.1 Correspondence between views
The views are not independent; the paper describes mappings:
- **Logical to process**: classes/objects are mapped onto processes and threads (which classes are active, which run in which task).
- **Logical to development**: class categories become modules/subsystems/packages (not always one-to-one).
- **Process to physical**: processes and tasks are mapped onto processing nodes; several mappings may exist (test vs production configurations).
- **Scenarios** tie them together: a scenario is walked through each view to show consistency.

### 2.2 Tailoring
Not every system needs all views. Views that add nothing can be dropped or merged: e.g. the physical view is minor
for a single-processor system, and the process view may be omitted if there is only one process. Similar
logical and development views may be combined for small systems. Scenarios are useful in all cases.

### 2.3 Iterative, scenario-driven process
1. Pick a small number of scenarios based on risk and criticality.
2. Build a strawman architecture; walk the scenarios through it to find the main abstractions, processes, modules, nodes.
3. Document the four views; capture lessons learned.
4. Next iteration: reassess risks, add scenarios, extend or refactor the architecture, test against the scenarios.
5. Iterate until the architecture is stable. Scenarios thus both *drive* and *validate* the design.

### 2.4 Course-specific guidance for 4+1 in TDT4240
Source: the teacher's written feedback published in one 2026 group repo (github.com/Yannic-Neu/battleships-ex).
This is one group's feedback, not an official rubric; treat it as a strong hint of what is checked.
- **Run-time object interactions belong in the process view**, not in the logical view. The logical view is the static
  functional decomposition.
- The **logical view must show external components**, e.g. Firebase (or Supabase) and server-side data, and if an
  ECS is used it must be specified properly (entities, components, systems).
- The **development view should help divide work** among group members: packages/modules (libGDX `core`,
  `android`, `lwjgl3`), who owns what, build dependencies.
- The **physical view should show network types** (e.g. HTTPS/WebSocket over mobile network or Wi-Fi between the
  Android device and the cloud backend), not only boxes for phone and server.
- **Show tactics and patterns in each view**, and write a **consistency among views** section.
- Check the current year's assignment text for the exact required views; the template can change.

---

## 3. IEEE Std 1471-2000 and its successors

*IEEE Recommended Practice for Architectural Description of Software-Intensive Systems* (IEEE Std 1471-2000).
Syllabus article. A **recommended practice**: it prescribes what an architectural description (AD) must contain,
not which views or notations to use.

### 3.1 Conceptual model (terms and relations)

| Concept | Meaning / relation |
|---|---|
| **System** | Fulfils one or more **missions**; inhabits an **environment**; has an **architecture** |
| **Mission** | A use or operation for which the system is intended by stakeholders |
| **Environment** (context) | The setting and circumstances that influence the system (developmental, operational, political, ...) |
| **Architecture** (1471 definition) | "The fundamental organization of a system embodied in its components, their relationships to each other and to the environment, and the principles guiding its design and evolution." (Differs from SAiP's "set of structures" definition; know both.) |
| **Architectural description (AD)** | A collection of products that documents an architecture; describes one architecture |
| **Stakeholder** | Individual, team or organisation with interests in, or concerns relative to, the system |
| **Concern** | An interest pertaining to the system's development, operation or other aspect that is critical or important to stakeholders (e.g. performance, reliability, security, distribution, evolvability) |
| **Viewpoint** | A specification of the conventions for constructing and using a view: its stakeholders, concerns addressed, language/notation, modelling and analysis techniques. A template/pattern for views |
| **View** | A representation of a whole system from the perspective of a related set of concerns; conforms to exactly one viewpoint |
| **Model** | A view consists of one or more models; a model may participate in more than one view |
| **Library viewpoint** | A predefined viewpoint defined outside the AD (e.g. published elsewhere) and reused by reference |
| **Rationale** | Justification for the architectural concepts chosen, and for the viewpoints selected |

Chain to memorise: **stakeholders have concerns → viewpoints are selected to frame concerns → each view conforms to
a viewpoint → views consist of models → rationale explains the choices.** Viewpoint : view ≈ class : instance
(or template : filled-in document).

### 3.2 Required AD contents (clause 5 of the standard)

1. **AD identification, version and overview information** (date, status, issuing organisation, change history, summary, scope, context, glossary, references).
2. **Identification of stakeholders and their architecturally relevant concerns** (at least users, acquirers, developers, maintainers).
3. **Specification of each viewpoint selected**, with the **rationale for selecting it** (name, stakeholders, concerns, language/techniques, source if a library viewpoint).
4. **One or more views** (one per selected viewpoint).
5. **A record of all known inconsistencies** among the AD's required constituents (consistency analysis).
6. **Rationale for the architecture** (alternatives considered, why the chosen concepts were selected).

Mapping to the TDT4240 architecture document: stakeholders and concerns → section "Stakeholders and concerns";
viewpoint specifications → "Architectural viewpoints" table (with purpose = why); views → 4+1 views;
inconsistencies → "Consistency among views"; rationale → "Architectural rationale".
The course has used IEEE 1471 as the basis for the project's architecture document (see Wang, A. I., "Extensive
Evaluation of Using a Game Project in a Software Architecture Course", *ACM Transactions on Computing Education*
11(1), 2011, which evaluates project years from that period). Wang's paper names IEEE 1471
completeness as a project grading criterion in those years (see
[course-and-project-guide.md](course-and-project-guide.md) §6); current criteria may differ, so check the assignment text.

### 3.3 Successors (context only; not TDT4240 syllabus)
- **ISO/IEC/IEEE 42010:2011** superseded IEEE 1471. It kept the core model and added, among other things,
  **architecture frameworks** (e.g. a set of viewpoints for a domain), **architecture description languages**,
  **model kinds** (conventions for one kind of model within a viewpoint), **correspondences and correspondence rules**
  (a formal way to express and check relations between AD elements, generalising 1471's consistency among views;
  the AD must still record known inconsistencies), and explicit
  **architecture decisions and rationale**.
- **ISO/IEC/IEEE 42010:2022** is a further revision with some changed terminology. Cite by year; if you rely on a
  2022 detail, check the standard itself.
- Exam answers should use 1471 terminology unless the question says otherwise.

---

## 4. arc42 and C4 (beyond syllabus, useful in practice)

### 4.1 arc42 (Starke & Hruschka; https://arc42.org/)
A free twelve-section template. The name is not an abbreviation.

| # | Section | Content |
|---|---|---|
| 1 | Introduction and Goals | Requirements overview, top 3-5 quality goals, stakeholders |
| 2 | Architecture Constraints | Technical, organisational, conventions (given, not chosen) |
| 3 | Context and Scope | Business and technical context; external interfaces |
| 4 | Solution Strategy | Fundamental decisions and approaches to reach the quality goals |
| 5 | Building Block View | Static decomposition, hierarchically refined |
| 6 | Runtime View | Important scenarios as behaviour |
| 7 | Deployment View | Infrastructure and mapping of building blocks onto it |
| 8 | Crosscutting Concepts | Persistence, error handling, security, i18n, concurrency, ... |
| 9 | Architecture Decisions | Important decisions (often as ADRs) |
| 10 | Quality Requirements | Quality tree (utility tree) and quality scenarios |
| 11 | Risks and Technical Debt | Known problems, ordered by priority |
| 12 | Glossary | Domain and technical terms |

### 4.2 C4 model (Simon Brown; https://c4model.com/)

| Level | Diagram | Shows | Audience |
|---|---|---|---|
| 1 | **System Context** | The system as one box; people (actors) and external software systems around it | Everyone, including non-technical |
| 2 | **Container** | Separately deployable/runnable units: apps, services, databases, file stores ("container" ≠ Docker) | Technical staff, operations |
| 3 | **Component** | Components inside one container and their responsibilities | Developers, architects |
| 4 | **Code** | Classes etc. for one component; usually left to IDE/UML class diagrams | Developers |

Labelling conventions: every element has a **name**, an **element type** ([Person], [Software System],
[Container: technology], [Component: technology]) and a **one-line responsibility**; every relationship arrow is
**unidirectional and labelled** with its intent and, for containers/components, the protocol or technology
("Reads/writes game state [HTTPS/JSON]"); each diagram has a **title** and a **key/legend**. C4 covers static
structure; dynamic and deployment diagrams are supplementary.

### 4.3 View map: how the frameworks line up

| Doc section (example) | arc42 # | C4 level | 4+1 view | Concern addressed |
|---|---|---|---|---|
| Introduction, quality goals, stakeholders | 1 | - | Scenarios | Why the system exists; what "good" means |
| Constraints | 2 | - | - | What is not negotiable |
| Solution strategy | 4 | - | (all; summary) | Top-level decisions and the tactics/patterns chosen per quality goal |
| Context view | 3 | L1 System Context | Logical (external parts) | System boundary, external interfaces |
| Building block view | 5 | L2 Container, L3 Component | Logical + Development | Static decomposition, work division |
| Runtime view | 6 | Dynamic diagrams | Process | Behaviour, concurrency, performance |
| Deployment view | 7 | Deployment diagrams | Physical | Allocation to hardware and networks |
| Crosscutting concepts | 8 | - | (all) | Persistence, errors, security |
| Decisions (ADRs) | 9 | - | - | Rationale, alternatives, trade-offs |
| Quality requirements | 10 | - | Scenarios | Utility tree, QA scenarios |
| Risks and technical debt | 11 | - | - | Known liabilities |
| Glossary | 12 | - | - | Shared vocabulary |

Use 4+1 as a **completeness check**: if one of the four views maps to no section, something is missing.

---

## 5. Architecture Decision Records (ADRs) (beyond syllabus)

Nygard, M. "Documenting Architecture Decisions", 2011
(https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions). A short text file per
architecturally significant decision, kept with the code.

| Field | Content |
|---|---|
| **Title** | Short phrase, numbered. Nygard uses noun phrases ("ADR 1: Deployment on Ruby on Rails 3.0.10"); many teams use imperative titles such as "ADR-004: Abstract the backend behind an interface". Pick one style and use it throughout |
| **Status** | Proposed / Accepted / Deprecated / Superseded by ADR-NNN |
| **Context** | The forces at play: requirements, QAs at stake, constraints, team, technology. Value-neutral |
| **Decision** | "We will ..." in active voice |
| **Consequences** | Everything that follows, good and bad, including new decisions it forces |

Useful extensions:
- **Alternatives rejected**, each with a one-line reason (this is what makes it rationale, not a diary).
- **Costs accepted**: the trade-off you knowingly take (ties to ATAM tradeoff points).
- **Non-goals**: what deliberately does not change, to bound the decision.
- **Supersession**: ADRs are immutable; a changed mind is a new ADR that supersedes the old, with links both ways.
  Number in order written.
- **Reconstructed ADRs**: for undocumented historical decisions, recover them from code and history and mark them
  "reconstructed" (and, if nobody chose deliberately, say "accepted by default").
- Link each ADR to the **quality goal** it serves and to any **risks** it causes or mitigates.

### 5.1 Filled example (TDT4240-style game project)

> **ADR-003: Abstract the backend behind an interface**
>
> **Status:** Accepted (2026-02-10)
>
> **Context:** The game is a libGDX project with `core`, `android` and `lwjgl3` (desktop) modules. Online matches
> and leaderboards need a cloud backend; the group chose Firebase Realtime Database. The Firebase Android SDK can
> only be used from the `android` module, but game logic lives in `core`, which must stay platform-independent.
> Modifiability is the primary quality attribute (scenario M2: "swap the backend provider in under 2 person-days,
> changing no class in `core`"). Testability matters: game logic must be testable on the desktop without network.
>
> **Decision:** We will define a `BackendService` interface in `core` (create/join lobby, push move, listen for
> opponent moves, submit score). `FirebaseBackend` in `android` implements it; `LocalFakeBackend` in `core` (test
> sources) and a stub in `lwjgl3` implement it for testing and desktop runs. The platform launcher injects the
> implementation into the game at start-up. Callbacks from the backend are delivered on the libGDX render thread
> via `Gdx.app.postRunnable`.
>
> **Consequences:**
> - Benefits: realises the modifiability tactics *encapsulate*, *use an intermediary* and *defer binding* (binding at
>   start-up); `core` has no Firebase import; game logic can be unit-tested with the fake.
> - Costs accepted: one extra interface and adapter per feature; Firebase-specific features (offline persistence
>   tuning, security rules) are not visible through the interface; a small indirection cost on every call (negligible
>   against network latency).
> - Alternatives rejected: (a) calling Firebase directly from `core`: impossible without platform code and would
>   tie all logic to one vendor; (b) a REST server of our own: more work than the course timeline allows; (c) a
>   third-party cross-platform Firebase wrapper: extra dependency with uncertain maintenance.
> - Non-goals: no support for switching backends at runtime; no offline multiplayer.
> - Follow-ups: ADR-005 (data model in the Realtime Database); risk R-2 (callback threading bugs) mitigated by the
>   `postRunnable` rule.

In the architecture document, the ADR list can form (part of) the **rationale** section; tie each ADR to tactics
and patterns and show it in the relevant view (the interface appears in the logical and development views).

---

## 6. Worked real-world example: wallywallet MR 853

GitLab MR https://gitlab.com/wallywallet/wallet/-/merge_requests/853 adds `docs/architecture.md`, an
architecture description of the **Wally** wallet, a Kotlin Multiplatform app whose `:shared` module (about
27 kLOC) builds for five targets. Not course material, but a complete example of applying the course concepts to a real codebase.
Framed as an **ISO/IEC/IEEE 42010** description, structured by **arc42**, drawn at **C4 levels 1-3**, checked against
**4+1**, with six-part QA scenarios, named tactics (SAiP 4th ed.), Views and Beyond conventions (primary
presentation + element catalogue + rationale) and Nygard ADRs.

What it demonstrates, and what to copy:

| Technique | How the MR does it | Lesson |
|---|---|---|
| **Concern-driven stakeholder table** | Columns: stakeholder, concerns, where addressed (quality goal QG-n / section). Rows include end users, merchants/integrators, maintainers, release engineers, app-store gatekeepers, third-party server operators, security auditors | A stakeholder earns a row only by raising a concern some view answers (42010 chain) |
| **AS-IS vs TO-BE labelling** | AS-IS statements describe the baseline commit and name the source file; TO-BE statements appear only in proposed ADRs, risks and roadmap, marked *Proposed*. An appendix pins baseline facts | Keep descriptive and prescriptive architecture apart |
| **View map** | A table mapping each section to arc42 #, C4 level, 4+1 view and concern (as in 4.3 above) | One table shows completeness across frameworks |
| **Intended vs actual layering** | Intended layering vs the real call graph, side by side, naming missing and bypassed layers and **upward dependencies** that form **layer-bridging cycles** (worse than a shortcut) | Layers earn their keep only if enforced; show violations explicitly |
| **Six-part QA goals** | QG-1 Security, QG-2 Reliability under a hostile network, QG-3 Modifiability and testability, QG-4 Portability, QG-5 Startup and interaction responsiveness; each with source, stimulus, artifact, environment, response, numeric response measure; plus a priority order and conflict rule (security wins over responsiveness) | "A goal without a measurable response is an aspiration, not a requirement" |
| **Tactic-argued changes** | Each quality goal lists "tactics in use" (e.g. limit access, limit exposure, retry, defer binding, specialized interfaces) and the code realising them; each proposal names its tactic | Argue design in the book's vocabulary, linked to a QG and a cost |
| **ADRs** | ADR-001-011 reconstruct decisions in the current system (some "accepted by default, not by deliberation"); ADR-012-018 are proposals, each with QG served, costs, alternatives, migration, non-goals | Reconstructed + proposed ADRs make history and direction reviewable one by one |
| **Risk register traced to ADRs** | Risks R-1 to R-11, each traced to the ADR that caused it and the ADR that mitigates it | Mitigations attack causes, not symptoms |
| **Utility tree** | QA → refinement → scenario, rated (business importance, technical difficulty) H/M/L, and stated as usable ATAM input | (H,H) leaves are where architectural effort pays most |
| **Fitness functions** | A proposed ADR makes rules executable as CI checks (module dependency direction, no globals in UI code, data-source types confined to the data module, coverage floor, start-up time budget); after Ford, Parsons & Kua | "A scenario with a CI gate is a requirement, and a scenario without one is a wish" |
| **Runtime and deployment views** | Sequence diagrams per key scenario with numbered architectural observations; deployment covers both runtime allocation and the build/release pipeline | Process and physical views for a mobile app |

**Citation slip to note:** the MR names SAiP **4th ed.** but cites "ch. 4" for quality-attribute scenarios. That is
the **3rd-edition** number; in the 4th edition, scenarios are in **ch. 3** ("Understanding Quality Attributes"; ch. 4
is Availability). Its reference entry also gives the tactic chapters as "ch. 5-13"; in the 4th ed. the QA
chapters with tactics run from ch. 4 (Availability) to ch. 13 (Usability). Lesson for students: pick one edition and use its numbering consistently.

---

## 7. Reviewer checklist for any architecture document

Use when reviewing a TDT4240 architecture document, another group's document before ATAM, or a real AD.

**Framing (IEEE 1471 / 42010)**
- [ ] Identification: title, version, date, status, authors; for TDT4240, front page with game title and chosen COTS.
- [ ] Every stakeholder has concerns; every concern points to a view or section that answers it; the ATAM evaluation group is listed.
- [ ] Each viewpoint is specified (stakeholders, concerns, notation) with a *why* for including it.
- [ ] Known inconsistencies between views recorded ("consistency among views").

**Requirements side**
- [ ] ASRs contain requirements only, no design decisions.
- [ ] Quality goals are six-part scenarios with measurable response measures, prioritised, with a conflict rule.
- [ ] Usability and performance scenarios not mixed up.
- [ ] Constraints (COTS, platform, deadline) kept separate from decisions; COTS effects on control flow and required interfaces described.

**Views**
- [ ] Each view has primary presentation with a key, element catalogue, context, (variability guide where relevant) and rationale.
- [ ] Logical view shows external components (backend, third-party services) and server-side data.
- [ ] Run-time interactions are in the process view (sequence/activity/state diagrams), not the logical view.
- [ ] Development view supports work division; physical view shows nodes and network types.
- [ ] Tactics and patterns visible in the views where they apply; tactics and patterns not confused with each other.
- [ ] Mapping between views given (e.g. module to process to node).
- [ ] Intended vs actual structure compared when documenting an existing system; violations named.

**Rationale and decisions**
- [ ] Rationale links tactics, patterns and structures to the quality goals.
- [ ] Significant decisions recorded (ADR style or equivalent) with alternatives rejected and costs accepted.
- [ ] Risks traced to causes; for evaluations, utility tree with (importance, difficulty) ratings.

**Hygiene**
- [ ] Glossary and acronym list; abbreviations expanded at first use.
- [ ] References with full citations (and one textbook edition used consistently).
- [ ] Diagrams consistent with the code (at the end of the project); changes section kept up to date.
- [ ] For TDT4240, follows the current official course template and deliverable list.

---

### References
- Bass, L., Clements, P., Kazman, R. *Software Architecture in Practice*, 4th ed., Addison-Wesley, 2021 (ch. 22); 3rd ed., 2013 (ch. 18).
- Clements, P., Bachmann, F., Bass, L., Garlan, D., Ivers, J., Little, R., Merson, P., Nord, R., Stafford, J. *Documenting Software Architectures: Views and Beyond*, 2nd ed., Addison-Wesley, 2010.
- Kruchten, P. "The 4+1 View Model of Architecture." *IEEE Software* 12(6):42-50, 1995. DOI 10.1109/52.469759.
- IEEE Std 1471-2000, *IEEE Recommended Practice for Architectural Description of Software-Intensive Systems*.
- ISO/IEC/IEEE 42010:2011 and 42010:2022, *Systems and software engineering: Architecture description*.
- Nygard, M. "Documenting Architecture Decisions", Cognitect blog, 15 Nov 2011, https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions
- Wang, A. I. "Extensive Evaluation of Using a Game Project in a Software Architecture Course." *ACM Transactions on Computing Education* 11(1), 2011.
- arc42: https://arc42.org/ ; C4 model: https://c4model.com/ ; ADR community resources: https://adr.github.io/
- Ford, N., Parsons, R., Kua, P. *Building Evolutionary Architectures*, O'Reilly, 2017.
- wallywallet MR 853: https://gitlab.com/wallywallet/wallet/-/merge_requests/853
- Wikipendium contributors, "TDT4240 Software Architecture", https://www.wikipendium.no/TDT4240_Software_Architecture (CC BY-SA 3.0). Sections 1.2 and parts of 3.1 are adapted from it, with changes, and redistributed under CC BY-SA 4.0 (permitted by BY-SA 3.0's later-version clause). Contributor list: https://www.wikipendium.no/TDT4240_Software_Architecture/history/ ; full attribution in [../CREDITS.md](../CREDITS.md).
