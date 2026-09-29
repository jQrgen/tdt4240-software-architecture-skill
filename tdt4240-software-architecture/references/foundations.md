# Foundations: definitions, structures, why architecture matters, contexts

Use this file for "what is architecture?"-type questions, the 13 reasons, structures and views,
contexts and stakeholders, and terminology disputes. Quality attributes and tactics are in
`quality-attributes-classic.md` / `quality-attributes-4th-edition.md`. Patterns are in `architectural-patterns.md`
and `design-and-game-patterns.md`. Views, viewpoints and IEEE 1471 / 4+1 documentation are in `documentation.md`.
ASRs, constraints and design methods (ADD) are in `requirements-and-design.md`; ATAM and evaluation are in `evaluation.md`.

> Parts of this file are adapted from the Wikipendium TDT4240 compendium
> (https://www.wikipendium.no/TDT4240_Software_Architecture, CC BY-SA 3.0), restructured and corrected.
> Wikipendium contributors are credited by name in `../CREDITS.md`; page history:
> https://www.wikipendium.no/TDT4240_Software_Architecture/history/

---

## 1. Edition note (read first)

- **Current textbook edition:** Bass, Clements, Kazman, *Software Architecture in Practice* (SAiP), **4th ed.**,
  Addison-Wesley (SEI Series), 2021. It has 26 chapters.
- **Wikipendium and the public 2015/2016 exams use 3rd-edition (2013) numbering.** A student's old notes will
  say "ch. 13 tactics and patterns" or "ch. 18 documentation", and those numbers are 3rd-edition numbers.
- **No public source confirms which edition or which chapters TDT4240 requires today.** The reading list is in
  Leganto, and course announcements are on Blackboard. Tell students to check there. Never claim a required-chapter list.
- When citing a chapter, give both numbers where they differ, e.g. "Modifiability (4th ed. ch. 8; 3rd ed. ch. 7)".

### 3rd to 4th edition chapter mapping (by title)

| 3rd ed. (2013) | 4th ed. (2021) |
|---|---|
| 1 What Is Software Architecture? | 1 What Is Software Architecture? |
| 2 Why Is Software Architecture Important? | 2 Why Is Software Architecture Important? |
| 4 Understanding Quality Attributes | 3 Understanding Quality Attributes |
| 5 Availability | 4 Availability |
| 6 Interoperability | 7 Integrability (renamed and broadened) |
| 7 Modifiability | 8 Modifiability |
| 8 Performance | 9 Performance |
| 9 Security | 11 Security |
| 10 Testability | 12 Testability |
| 11 Usability | 13 Usability |
| 12 Other Quality Attributes | 14 Working with Other Quality Attributes |
| 16 Architecture and Requirements | 19 Architecturally Significant Requirements |
| 17 Designing an Architecture | 20 Designing an Architecture |
| 21 Architecture Evaluation | 21 Evaluating an Architecture |
| 18 Documenting Software Architectures | 22 Documenting an Architecture |
| 24 Architecture Competence | 25 Architecture Competence |
| 26 Architecture in the Cloud | 17 The Cloud and Distributed Computing |

**New in the 4th ed.:** 5 Deployability, 6 Energy Efficiency, 10 Safety, 15 Software Interfaces,
16 Virtualization, 18 Mobile Systems, 23 Managing Architecture Debt, 24 The Role of Architects in Projects,
26 A Glimpse of the Future: Quantum Computing.

**3rd-ed. chapters with no same-titled 4th-ed. chapter (check the book):** 3 The Many Contexts of Software
Architecture, 13 Architectural Tactics and Patterns, 14 Quality Attribute Modeling and Analysis,
15 Architecture in Agile Projects, 19 Architecture, Implementation, and Testing, 20 Architecture Reconstruction
and Conformance, 22 Management and Governance, 23 Economic Analysis of Architectures (CBAM),
25 Architecture and Software Product Lines, 27 Architectures for the Edge. It is not verified whether this material
was dropped or merged into other 4th-ed. chapters. Say "check the book" rather than guessing a location.
(Section 5 below covers the ch. 3 contexts material; where it sits in the 4th ed. is likewise unverified.)

---

## 2. Definitions (paraphrased)

| Term | Definition to use | Source / note |
|---|---|---|
| **Software architecture (SAiP)** | The set of structures you need in order to reason about a system. Each structure is made of software elements, the relations among them, and the properties of both. | SAiP 3rd and 4th ed., ch. 1. **This is the expected exam answer** when asked for the textbook (Bass, Clements, Kazman) definition, a frequently asked question. |
| **Architecture (IEEE 1471-2000)** | The fundamental organisation of a system, embodied in its components, their relationships to each other and to the environment, and the principles that guide its design and evolution. | IEEE 1471 (superseded by ISO/IEC/IEEE 42010:2011, revised 2022). Differs from SAiP: it adds *principles* and *evolution*, and it uses "components" loosely. 42010:2011 rewords it as the fundamental concepts or properties of a system in its environment, embodied in its elements, relationships and principles of design and evolution. Quote the 1471 wording only when citing 1471. |
| **{elements, form, rationale}** | Perry and Wolf: architecture = elements (processing, data, connecting) + form (weighted properties and relationships that constrain them) + rationale (why the choices were made). | Perry and Wolf 1992. **Used in earlier years of the course.** Mention it for historical context only. |
| **System** | A collection of components organised to accomplish a specific function or set of functions. | IEEE 1471. |
| **Architectural description (AD)** | The collection of products (documents, models, diagrams) that document an architecture. | IEEE 1471. The architecture exists whether or not an AD does. |
| **Module** | An implementation unit (code or data) that provides a coherent set of responsibilities: a package, class or layer. Exists at design/build time. | SAiP. |
| **Component** | In SAiP, a **runtime** element in a component-and-connector (C&C) structure: a process, service, object instance, client or server. Components interact through **connectors** (calls, pipes, pub-sub bus, protocols). | SAiP. See the note below. |
| **Connector** | The runtime interaction mechanism between components: procedure call, pipe, event bus, protocol. | SAiP. |
| **Element** | Generic word for the parts of any structure (modules, components, hardware nodes, teams...). | SAiP. |
| **View** | A representation of a whole system from the perspective of a related set of concerns. In SAiP terms: a representation of a structure (a coherent set of elements and relations), written for stakeholders. | IEEE 1471; SAiP. |
| **Stakeholder** | A person, group or organisation with an interest in (a stake in) the system. | IEEE 1471; SAiP. |
| **Concern** | An interest in the system that is relevant to one or more stakeholders (e.g. performance, cost, modifiability). | IEEE 1471. |
| **Viewpoint** | The conventions for constructing and using a view: its purpose, audience/concerns, notations, and techniques for creating and analysing it. A template; a view conforms to a viewpoint. | IEEE 1471. |
| **Pattern** | A named, reusable solution: a {context, problem, solution} triple that is found in practice (discovered, not invented). Coplien stresses that a pattern describes both the thing and how to build it. | SAiP 3rd ed. ch. 13; Coplien 1998. |
| **Architectural pattern** | A pattern at system level: element types, their interaction mechanisms, their topology and constraints (e.g. Layered, Broker, MVC, Client-Server). | SAiP. |
| **Design pattern** | A pattern for a problem inside a subsystem or module, at class/object level (e.g. GoF Observer, State, Factory). | GoF 1994; course lectures. |
| **Tactic** | A design decision that influences the response of **one** quality attribute (e.g. heartbeat, encapsulate, introduce concurrency). The building blocks of design. Patterns bundle several tactics. | SAiP 3rd ed. ch. 4 (intro), 5-11 (per-QA tactics), 13 (tactics and patterns); 4th ed. ch. 3 (intro), 4-13 (per-QA tactics). |
| **Functional requirement** | What the system must do, and how it must react to stimuli at runtime. | SAiP. |
| **Quality attribute requirement** | How well the system must do it: a measurable or testable property that qualifies a function (latency, availability %, time to change). Written as six-part scenarios. | SAiP 3rd ed. ch. 4 / 4th ed. ch. 3. |
| **Constraint** | A design decision with zero degrees of freedom that is already made for you (mandated language, framework, platform, protocol, reuse of an existing component, e.g. "Android + libGDX"). Constraints are given, not chosen. Business/project constraints (deadline, budget) also limit the design space but are not design decisions themselves. | SAiP. |
| **ASR (architecturally significant requirement)** | A requirement that has a profound effect on the architecture: the architecture would likely be very different without it. | SAiP 3rd ed. ch. 16 / 4th ed. ch. 19; see `requirements-and-design.md`. |

**Architecture vs design.** All architecture is design, but not all design is architecture. Architecture is the subset of
design decisions that affect reasoning about the whole system and its qualities; the rest is detailed (non-architectural) design.

**Normalise "component".** Wikipendium's glossary says a component is "two or more modules providing
services". Do not teach that. In SAiP a component is a **runtime** element in C&C structures, while a module is a
**static** implementation unit. One module can become many components at runtime (e.g. many client
instances), and one component can use code from many modules.

**Architectural vs design pattern:** the difference is scope and what they constrain. Architectural patterns shape
the whole system's structure and its quality attributes (Layered, Client-Server, Pub-Sub). Design patterns solve local
object-design problems (Observer, State). The same idea can appear at both levels, e.g. MVC (architectural) is
usually built with Observer (design). Pub-sub is looser than Observer because an event bus decouples
publishers from subscribers.

**Implications of the SAiP definition (useful short answers):**
- Every software system has an architecture, whether or not it is documented or was designed deliberately.
- Architecture is an abstraction. It omits private implementation details and keeps what affects reasoning.
- Not every structure is architectural. A structure counts only if it supports reasoning about the system and its qualities.
- A system has several structures. No single structure is "the" architecture.
- Behaviour (how elements interact) is part of the architecture as far as it affects reasoning.
- "Good" or "bad" only makes sense relative to a purpose (the quality attributes and business goals).

---

## 3. The three structure categories

Each category answers a different question. The structure is the thing itself; the view is its documentation.

| Category | Question it answers | Structures (element / relation) | QAs it helps reason about |
|---|---|---|---|
| **Module** (static, design/build time) | How is the system decomposed into implementation units? What is each unit responsible for, and what does it depend on? | **Decomposition** (module / is-a-submodule-of); **Uses** (module / uses: needs the other to be correct); **Layers** (layer / allowed-to-use, strictly downward); **Class / generalisation** (class / is-a, instance-of); **Data model** (data entity / relationships, cardinality) | Modifiability, portability, testability, reuse, ability to build incrementally or build subsets, division of work |
| **Component-and-connector (C&C)** (runtime) | What are the runtime elements and how do they interact? What data flows where, and what runs in parallel? | **Service** (services / protocols, connectors, e.g. SOA or backend services); **Concurrency** (logical threads / shared resources and synchronisation; later mapped to processes) | Performance, availability, security, reliability, scalability, interoperability/integrability |
| **Allocation** (software mapped to non-software) | Where does the software run, where is it stored, and who builds it? | **Deployment** (components / allocated-to hardware nodes and networks, e.g. Android phone, Firebase cloud); **Implementation** (modules / mapped to files, directories, build modules, e.g. libGDX `core` / `android`); **Work assignment** (modules / assigned-to teams or developers) | Performance and availability (via deployment), security (network zones), buildability and configuration management (implementation), project management and Conway's law (work assignment) |

**Worked example (TDT4240-style multiplayer libGDX game with Firebase):**
- Module: `model`, `view`, `controller`, `network` packages; layers UI -> game logic -> backend interface.
- C&C: the game client component talks to the Firebase realtime database over a pub-sub-style listener connector; the
  game loop and the network callback thread form the concurrency structure.
- Allocation: the client runs on Android phones and the database in the Google cloud (deployment); a Gradle `core` module plus an
  `android` launcher module (implementation); person A owns networking and person B owns rendering (work assignment).

A frequently asked exam question is the purpose of module, C&C and allocation views. Answer with the "question it
answers" column plus one QA each. For how these relate to Kruchten's 4+1 (logical, process, development,
physical) see `documentation.md`.

---

## 4. Why architecture matters: 13 reasons (SAiP ch. 2, both editions)

Paraphrased. Learn them as a list, because short exam questions ask for "reasons architecture is important".

1. **QA enabler/inhibitor.** The architecture largely decides whether the driving quality attributes can be met.
2. **Managing change.** Its decisions tell you which changes are local, non-local or architectural, so change can be reasoned about.
3. **Early prediction.** Analysing the architecture predicts system qualities before the system is built.
4. **Communication.** A documented architecture is a shared vocabulary for stakeholders with different concerns.
5. **Earliest decisions.** It holds the first design decisions, which are the most fundamental and the most expensive to change later.
6. **Implementation constraints.** Implementers must work within the elements, interfaces and rules it sets.
7. **Organisational structure (Conway's law).** The architecture shapes team and work-breakdown structure, and the
   organisation's communication structure in turn shapes the architecture.
8. **Evolutionary prototyping** (4th ed.: *enabling incremental development*). A skeleton system can be built early and grown incrementally.
9. **Cost and schedule.** It is the main artefact the architect and project manager use to estimate effort and schedule.
10. **Product lines.** It can become a reusable, transferable model at the heart of a family of products.
11. **Assembly over creation** (4th ed.: *incorporation of independently developed elements*). Development focuses on integrating components (including COTS, e.g. libGDX and Firebase) rather than writing everything.
12. **Channelled creativity.** Restricting design alternatives reduces complexity and makes the system easier to understand.
13. **Training.** It is the natural starting point for onboarding new team members.

*Adapted from the Wikipendium TDT4240 compendium (CC BY-SA 3.0).*

---

## 5. Contexts, influence cycle and stakeholders (3rd ed. ch. 3)

**Edition note:** 3rd ed. ch. 3 "The Many Contexts of Software Architecture" has no 4th-ed. counterpart. The same
ideas are spread across 4th ed. ch. 1-2 (what and why), ch. 24 (role of architects in projects) and ch. 25
(competence). Exact placement is unverified; check the book.

| Context | Guiding question | Key content |
|---|---|---|
| **Technical** | What technical role does the architecture play in the system? | Achieving QA requirements, and the current technology environment (cloud, mobile, frameworks). |
| **Project life cycle** | How does architecture relate to the other development phases? | It fits into waterfall, iterative, agile and model-driven processes. Core activities are: make the business case, understand ASRs, create/select the architecture, document and communicate it, analyse/evaluate it, implement and test against it, ensure conformance. |
| **Business** | How does the architecture affect, and how is it affected by, the organisation's business? | It must satisfy business goals from many stakeholders with differing expectations. It shapes organisational structure (Conway) and supports product lines. |
| **Professional** | What is the architect's role? | Skills, knowledge and duties beyond technology (communication, negotiation, leadership), shaped by education and experience. |

*Adapted from the Wikipendium TDT4240 compendium (CC BY-SA 3.0); the life-cycle activities and the cycle below are added from SAiP.*

**Architecture influence cycle.** Architectures are influenced by the business goals and needs of stakeholders, the
technical environment, and the architect's experience. Once built, the system and its architecture feed back:
they change the organisation's business goals (new markets, product lines), its structure, the technical
environment, and the architect's own experience. Those changes then shape future architectures. Exam point:
the influence runs **both ways**, and the architecture is not just a product of requirements.

**Stakeholders.** Anyone with a stake in the system. Each stakeholder has concerns, and concerns often conflict (e.g.
the customer wants low cost, the user wants performance, the maintainer wants modifiability). The architect must
elicit, prioritise and balance them. Typical stakeholders listed in SAiP include users, customers,
the project manager, developers/implementers, testers, maintainers, integrators, deployers and system administrators,
evaluators, business managers, product-line managers, and representatives of external systems. **In the course project**,
add the course staff and the **ATAM evaluation group** (course feedback explicitly asks for the ATAM group; see
`course-and-project-guide.md`).

---

## 6. The architect's role and competence (4th ed. ch. 24-25; 3rd ed. ch. 24)

Keep this brief unless asked; details beyond this summary: check the book.
- **Role in projects (4th ed. ch. 24):** the architect works closely with the project manager (the architecture
  informs work breakdown, estimates and risk), works in incremental and agile settings (enough up-front architecture
  to address ASRs, then evolve it), and works with distributed teams (interfaces follow team boundaries, Conway).
- **Competence (4th ed. ch. 25; 3rd ed. ch. 24):** competence is about carrying out **duties** (e.g. eliciting
  ASRs, designing, documenting, evaluating, guiding implementation, managing stakeholders), with the right **skills**
  (communication, negotiation, leadership, abstraction, dealing with ambiguity) and **knowledge** (patterns, tactics,
  technologies, the domain). SAiP also covers the competence of an *organisation* at architecture, not only of an individual.
- Core message: architecture work is as much social and organisational as it is technical.

---

## 7. Common confusions

| Confusion | Correct distinction |
|---|---|
| **Layer vs tier** | A **layer** is a *module* grouping with a strictly downward *allowed-to-use* relation (static, design-time). A **tier** is an *allocation*/runtime grouping of components deployed on separate hardware or processes (Multi-tier pattern). A 3-layer app can run in one tier; a 3-tier system is about deployment. Layer bridging (skipping a layer) is a documented exception. Upward *uses* violate the pattern; upward notification via callbacks/events registered by the upper layer is the accepted way to communicate upward. |
| **Module vs component** | Module = static implementation unit (module structures). Component = runtime element (C&C structures). Do not define a component as "two or more modules". |
| **View vs structure** | A **structure** is the set of elements and relations as they exist in the system (or its code/deployment). A **view** is a *representation* (documentation) of one or more structures for stakeholders. You document views; the system has structures. |
| **View vs viewpoint** | A **viewpoint** is the template/specification (purpose, concerns, stakeholders, notation). A **view** is an instance of it for one particular system. Example: "Process viewpoint: shows runtime concurrency for performance reasoning" vs "the process view of our game". IEEE 1471 requires the chosen viewpoints to be specified and justified. |
| **Tactic vs pattern** | A tactic targets one QA and is a single design decision; a pattern is a bundle of decisions (and tactics) with trade-offs. Course feedback: do not list patterns as tactics. |
| **Constraint vs design decision** | Constraints are given (e.g. "must use libGDX"); decisions are chosen and need rationale. Do not defend a choice as if it were a constraint. |
| **QA requirement vs functional requirement** | "Player can join a match" is functional. "Joining completes within 2 s under normal load for 95% of attempts" is a QA (performance) requirement. |
| **Architectural vs design pattern** | System-wide structure (MVC, Broker) vs local object design (Observer, Template Method). See section 2. |

---

## 8. Syllabus articles

Besides the textbook, the course has used these articles. The list is Wikipendium's 2015 list plus items from older course years. The current reading list is in Leganto, so check there before assuming any item is on this year's list.

| Article | Where it is covered |
|---|---|
| Kruchten, P. "The 4+1 View Model of Architecture." *IEEE Software* 12(6):42-50, 1995. | `documentation.md` §2 |
| IEEE Std 1471-2000, *IEEE Recommended Practice for Architectural Description of Software-Intensive Systems* (successor: ISO/IEC/IEEE 42010). | `documentation.md` §3 |
| Coplien, J. O. "Software Design Patterns: Common Questions and Answers." In *The Patterns Handbook: Techniques, Strategies, and Applications*, Cambridge University Press, 1998, pp. 311-320. | `architectural-patterns.md` §8 |
| Rollings, A. and Morris, D. *Game Architecture and Design: A New Edition*, New Riders, 2004, ch. 17, pp. 462-500 (the 2015 page range). | `design-and-game-patterns.md` Part B |

**Historical (older course years):**

- Perry, D. E. and Wolf, A. L. "Foundations for the Study of Software Architecture." *ACM SIGSOFT Software Engineering Notes* 17(4):40-52, 1992. The {elements, form, rationale} definition, see §2 above.
- Wang, A. I. and Stålhane, T. (2005), on using post-mortem analysis to evaluate software architecture student projects. Linked to the project's former post-mortem phase, see `course-and-project-guide.md` §2.3. Check the exact title and venue before citing it.

---

**Sources:** Bass, Clements, Kazman, *Software Architecture in Practice*, 3rd ed. (Pearson/Addison-Wesley, 2013)
and 4th ed. (Addison-Wesley, 2021); IEEE Std 1471-2000, *Recommended Practice for
Architectural Description of Software-Intensive Systems*; Perry, D.E. and Wolf, A.L., "Foundations for the Study of
Software Architecture", ACM SIGSOFT Software Engineering Notes 17(4):40-52, 1992; Coplien, J.O., "Software Design Patterns: Common
Questions and Answers", in *The Patterns Handbook*, Cambridge University Press, 1998, pp. 311-320; Gamma, E., Helm, R.,
Johnson, R. and Vlissides, J., *Design Patterns: Elements of Reusable Object-Oriented Software*, Addison-Wesley, 1994;
Conway, M.E., "How Do Committees Invent?", *Datamation*, April 1968, pp. 28-31; Wikipendium contributors,
"TDT4240: Software Architecture", https://www.wikipendium.no/TDT4240_Software_Architecture (CC BY-SA 3.0, adapted).
