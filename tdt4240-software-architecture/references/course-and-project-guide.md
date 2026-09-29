# TDT4240 course facts and group project guide

Use this file to answer questions about how TDT4240 is run and to coach groups
through the game project. For theory, see the other references:
[foundations.md](foundations.md), [requirements-and-design.md](requirements-and-design.md),
[quality-attributes-classic.md](quality-attributes-classic.md),
[quality-attributes-4th-edition.md](quality-attributes-4th-edition.md),
[architectural-patterns.md](architectural-patterns.md),
[design-and-game-patterns.md](design-and-game-patterns.md),
[documentation.md](documentation.md), [evaluation.md](evaluation.md),
[platforms-and-emerging-topics.md](platforms-and-emerging-topics.md) and
[exam-prep.md](exam-prep.md). The document templates are in
[../templates/](../templates/).

> **Last checked: September 2026.** Course facts change from year to year. Before
> stating any of them as fact, tell the student to confirm them on
> <https://www.ntnu.edu/studies/courses/TDT4240> (choose the right year) and on
> Blackboard. Deadlines, phase weights and the document templates come **only**
> from Blackboard and the current assignment text.

---

## 1. Course facts

| Item | Value (Spring 2026/2027 course pages) |
|---|---|
| Code and name | TDT4240 Software Architecture (Norwegian: *Programvarearkitektur*) |
| Credits | 7.5 ECTS |
| Term and place | Spring, Trondheim |
| Language | English |
| Coordinator and lecturer | Alf Inge Wang, Department of Computer Science |
| Prerequisites | TDT4100 Object-Oriented Programming and TDT4140 Software Engineering, or equivalent |
| Teaching | Lectures and exercises, plus a group development project |
| Course materials | "To be announced at the start of the term". The reading list is in Leganto, which is not public |

**Content, paraphrased from the course page:** central architecture concepts;
using and describing design and architectural patterns; architecture design
methods; achieving quality attributes; documenting and evaluating architectures
(ATAM); patterns for specific domains; and "software architecture and games",
practised through the project.

### 1.1 Assessment (from Spring 2025)

| Part | Weight | Details |
|---|---|---|
| Written school exam | 50% | 4 hours, in Inspera. The **only** aid is a digital appendix inside Inspera that summarises the most relevant parts of the syllabus |
| Coursework (group project) | 50% | Group members normally get the same grade, unless someone's contribution is inadequate |

- Both parts must be completed to pass.
- A re-sit may be **oral** instead of written.
- **Current exam item types are unknown.** No public exam paper after 2016 was
  found. Do not tell students the exam is multiple choice, or that it is not.
  Point them to past papers and the course's own information. See
  [exam-prep.md](exam-prep.md) for the known 2015/2016 structure.

### 1.2 Assessment history

| Period | Exam / project | Exam form and aids |
|---|---|---|
| About 2008-2011 | about 70 / 30 | Written exam (Wang's ACM TOCE paper) |
| 2015-2016 | (papers public) | 4 h written. Aids: printed IEEE 1471-2000 and Kruchten 1995, dictionary, simple calculator |
| 2020 | 40 / 60 | See the 2020 course page |
| 2021 | 40 / 60 | 4 h **home** exam |
| 2022-2024 | 40 / 60 | 4 h school exam in Inspera, aid code A (all printed and handwritten material) |
| 2025 onward | 50 / 50 | 4 h school exam in Inspera, digital appendix only |

Why it matters: older compendia and student notes assume open-book exams.
Since 2025 students cannot bring notes, so they must know definitions, tactic
lists and pattern trade-offs from memory.

### 1.3 Textbook

The edition in use now is not publicly confirmed. Exams from 2015-2016 cite Bass,
Clements & Kazman, *Software Architecture in Practice*, 3rd ed. (2013). The 4th ed.
(2021) is the current edition. Students must check Leganto or Blackboard. See
[foundations.md](foundations.md) for both chapter numberings. Never claim a list
of required chapters.

---

## 2. Project workflow

### 2.1 Introductory exercises (individual or pairs, early in the term)

| Exercise | Typical content | Purpose |
|---|---|---|
| libGDX warm-up | Small games such as **Helicopter** (sprites bouncing, animation, collision) and **Pong** | Learn the COTS framework: `ApplicationAdapter`/`Game`, `render()` loop, `SpriteBatch`, input |
| Pattern exercise | Apply **Singleton** and other patterns, e.g. a **State**-based `GameStateManager` (stack of states/screens) | Practise design patterns in the same code base |

Exact exercise texts vary by year. Treat the list as typical, not mandatory.

### 2.2 The group project

- **Product:** an Android game, usually multiplayer, written in **Java or
  Kotlin** with **libGDX**. Projects are usually generated with **gdx-liftoff**,
  giving a `core` module (shared game code), an `lwjgl3` module (desktop
  launcher, good for fast testing) and an `android` module.
- **Backend:** usually **Firebase** (Realtime Database or Firestore, sometimes
  with Cloud Functions or Auth). Some groups use **Supabase**.
- **Group size:** about 5-7 students in recent years (6 in Battleships-EX 2026, 7
  in Fat Piggies).

### 2.3 Deliverables seen in recent years

| Deliverable | Content | Skill resource |
|---|---|---|
| Requirements document | Game concept, functional requirements, QA scenarios, COTS and technical constraints, issues, changes, individual contributions | [../templates/requirements-document.md](../templates/requirements-document.md) |
| Architecture document | Drivers/ASRs, stakeholders and concerns, viewpoints, tactics, patterns, 4+1 views, consistency, rationale, issues, changes, contributions | [../templates/architecture-document.md](../templates/architecture-document.md) |
| ATAM evaluation | Evaluate **another** group's architecture (the 2022 Ronaslice repo mentions peer review by two groups) | [../templates/atam-evaluation.md](../templates/atam-evaluation.md), [evaluation.md](evaluation.md) |
| Implementation document / testing | Test the functional and quality requirements; evaluate how well the code matches the architecture | [documentation.md](documentation.md) |
| Post-mortem (historically) | Post-mortem analysis (Wang & Stålhane 2005); not confirmed as a current deliverable | - |

**Current deadlines, order and weights come from Blackboard.** Do not guess them.

Historical phase order (Wang's TOCE paper, about 2008-2010): COTS learning ->
design patterns -> requirements and architecture -> ATAM of another group ->
implementation and testing -> post-mortem.

---

## 3. Choosing quality attributes

- Many groups choose **modifiability as the primary QA**, with **usability** or
  **performance** as secondary (Battleships-EX 2026: modifiability, usability,
  performance). This is seen in examples, **not confirmed as mandatory**. Check
  the assignment text.
- Choose QAs you can **design for and test**. For each one, write concrete
  six-part scenarios with a numeric response measure (see
  [quality-attributes-classic.md](quality-attributes-classic.md)).
- Good game-project scenarios:
  - *Modifiability:* a developer adds a new game mode or power-up at design time;
    it is done by changing at most N classes, in under X hours, with no change to
    the backend interface.
  - *Usability:* a first-time player starts a match without instructions; they
    reach gameplay in under X s / Y taps.
  - *Performance:* during normal play on a mid-range phone, the opponent's move
    appears within X ms; the game keeps at least 30 fps (pick a measurable value).
- Keep each scenario to one QA. A "fast UI" scenario is performance; a
  "learnable UI" scenario is usability.

---

## 4. Recommended architecture for a typical libGDX + Firebase game

A defensible default. Present it as one option; the group must justify its own
choices against its own QA goals.

| Decision | Option | QA served (tactic) |
|---|---|---|
| Client structure | **MVC** (model = game state, view = libGDX screens/rendering, controller = input handling), or **ECS** (entities, components as plain data, systems as logic) for games with many interacting objects | Modifiability (increase cohesion, reduce coupling), testability |
| Screen flow | **State** pattern: a `GameStateManager` / screen manager with menu, lobby, play and game-over states | Modifiability (split module, encapsulate) |
| Backend access | Backend behind an **interface** defined in `core` (e.g. `BackendApi`), implemented in the `android` module with the Firebase SDK, injected at startup by the Android launcher; a fake or stub for `lwjgl3` and tests | Modifiability (use an intermediary, encapsulate, defer binding), testability (abstract data sources), portability |
| UI updates | **Observer** (or listener callbacks from the backend) so views react to model or database changes | Modifiability (reduce coupling), performance (no polling) |
| Object creation | **Factory** for entities, power-ups or screens | Modifiability |
| Shared services | Asset manager or sound manager, possibly a **Singleton** (note its testability cost) | Performance (reuse loaded assets) |
| Distribution | **Client-server**: phones as clients, Firebase as the managed server/shared data store | Availability and scalability handled by the managed service |

**Why the interface is necessary, not just nice:** `core` is plain Java/Kotlin
shared by the desktop and Android launchers, so it cannot depend on Android
Firebase SDKs directly. The standard solution is an interface in `core`, an
implementation in `android`, and injection through the `Game` constructor in
`AndroidLauncher`. This is also a textbook modifiability and testability tactic,
so say so in the rationale.

### 4.1 Tactics and patterns per view (4+1)

| View | What to show | Tactics/patterns to point out |
|---|---|---|
| Logical | Model, view, controller (or ECS entities, components, systems); screens; the `BackendApi` interface; **external components (Firebase) and server-side data** (collections/nodes, e.g. `lobbies`, `games`, `players`) | MVC/ECS, State, Factory, Observer; encapsulate, use an intermediary |
| Process | Runtime interactions: game loop (`render()` = update + draw), input -> controller -> model -> view, async backend callbacks, what happens when the opponent moves (sequence diagram) | Observer/pub-sub, client-server; introduce concurrency (async calls), limit event response |
| Development | Gradle modules (`core`, `android`, `lwjgl3`), packages (`model`, `view`, `controller`, `backend`), **who owns what** | Layering, restrict dependencies; supports dividing work |
| Physical | Android device, (desktop for dev), Firebase cloud; **network types** (Wi-Fi/4G/5G, HTTPS/WebSocket to Firebase) | Client-server, multi-tier |
| Scenarios (+1) | Walk one or two QA scenarios through all views | Ties views to the QA goals |

---

## 5. Common mistakes: checklist from public teacher feedback

Paraphrased from teacher feedback published in a 2026 student repo. Run this
checklist on any draft a student shares.

- [ ] Front page gives the **game title** and the chosen **COTS** components.
- [ ] Usability and performance scenarios are **not mixed** (one QA per scenario).
- [ ] COTS section covers **technical and architectural constraints**, the
      **interfaces you must implement** (e.g. `ApplicationListener`,
      `Screen`) and how the framework **affects control flow** (libGDX owns the
      game loop and calls you).
- [ ] ASRs contain **requirements only**, not design decisions.
- [ ] Stakeholders include the **ATAM evaluation group**.
- [ ] Each viewpoint states its **purpose: why** the view is included and for whom.
- [ ] **Runtime object interactions go in the process view**, not the logical view.
- [ ] **Patterns are not listed as tactics.** Tactics are single design decisions
      for one QA; patterns bundle tactics.
- [ ] The patterns section states the **problem** each pattern solves and its
      **high-level use**; detailed design belongs in the views.
- [ ] The logical view shows **external components** (e.g. Firebase) and
      **server-side data**, and specifies **ECS properly** (entities, components,
      systems) if ECS is used.
- [ ] **Tactics and patterns are visible in each view**, not only listed in a section.
- [ ] The development view helps **divide work** among developers.
- [ ] The physical view shows **network types**.
- [ ] The rationale **ties tactics, patterns and structures to the QA goals**.
- [ ] **References** are included.
- [ ] (Also noted) Low-priority functional requirements are allowed.

---

## 6. Grading criteria named historically

From Wang's ACM TOCE paper on the game project (about 2008-2010). Current
criteria may differ; check the assignment text.

| Criterion | What it means in practice |
|---|---|
| IEEE 1471 completeness | Stakeholders, concerns, viewpoints, views, inconsistencies, rationale all present |
| Working implementation | The game runs and implements the stated functional requirements |
| Code-architecture consistency | The code structure matches the documented views; deviations are explained |
| Readability | Clear writing and diagrams with legends |
| Testable requirements | QA scenarios have measurable response measures; tests refer to them |
| Rationale | Decisions are justified against QA goals and alternatives |
| Template use | The provided document templates are followed |
| COTS description | Framework and backend constraints are described |

---

## 7. Public example repositories

These are **examples of what past groups handed in, not model answers.** Grades
are not always public and practice changes year to year. Use them to see the
scope and format, not to copy content.

| Repo | Year | Notes |
|---|---|---|
| <https://github.com/Yannic-Neu/battleships-ex> | 2026 | Group 10; requirements and architecture documents with the teacher's written feedback (source of section 5) |
| <https://github.com/Hallvaeb/programvarearkitektur-ronaslice-game> | 2022 | Lists the deliverables: requirements, architecture, ATAM evaluations, implementation document, UML |

Related (unofficial): the libGDX + Firebase interface pattern is demonstrated in
<https://github.com/AndreasWintherMoen/libgdx-firebase-tutorial>. The course
paper: A. I. Wang, game project in software architecture course, available at
<https://folk.idi.ntnu.no/alfw/publications/game-project-in-swa-evaluation_final.pdf>.

---

## 8. Tutoring notes

- Say where uncertainty exists: textbook edition, current exam item types,
  deadlines, whether modifiability must be primary.
- When asked "what is due when?", answer "check Blackboard", then help with the content.
- Don't write a group's graded document for them. Review drafts against section 5,
  suggest scenarios and structure, and explain the trade-offs.
- The course framing of a team building a simple multiplayer mobile game also
  appears in the Wikipendium compendium. *Adapted from the Wikipendium TDT4240
  compendium (CC BY-SA 3.0), <https://www.wikipendium.no/TDT4240_Software_Architecture>.*
