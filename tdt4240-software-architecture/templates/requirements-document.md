# Template: TDT4240 requirements document

> **Follow the official template on Blackboard if it differs; section names below mirror recent public student documents.**
> Replace every `<...>` placeholder. Delete the `<!-- guidance -->` comments before you hand in.
> Background theory: [requirements-and-design.md](../references/requirements-and-design.md) sections 1, 2 and 6 (requirement kinds, ASRs, six-part scenarios); [quality-attributes-classic.md](../references/quality-attributes-classic.md) and [quality-attributes-4th-edition.md](../references/quality-attributes-4th-edition.md) (general scenarios, tactics); [course-and-project-guide.md](../references/course-and-project-guide.md) sections 3 (choosing quality attributes) and 5 (common mistakes: checklist from public teacher feedback). The next deliverable is the [architecture-document.md](architecture-document.md) template.

---

## Front page

| Field | Value |
|---|---|
| Game title | `<Name of the game>` |
| Group number | `<Group NN>` |
| Members | `<Full name, NTNU username>` (separate members with `<br>`) |
| Chosen COTS | `<e.g. libGDX (Java/Kotlin), Android SDK, Firebase Realtime Database / Firestore>` |
| Primary quality attribute | `<e.g. Modifiability>` |
| Secondary quality attributes | `<e.g. Usability, Performance>` |
| Document | Requirements document, version `<x.y>` |
| Date | `<YYYY-MM-DD>` |

<!-- Public teacher feedback in a 2026 student repo says the front page should name the game and the COTS (see course-and-project-guide.md section 5); confirm in the assignment text. Recent examples pick modifiability as primary, but whether that is mandatory for every group is not confirmed; check the assignment text. State in one sentence in the project summary why the primary QA matters for this game; see ../references/course-and-project-guide.md section 3 (Choosing quality attributes). -->

## 1. Project summary

`<3-5 sentences: what the game is, the platform, the chosen quality attributes, and what this document contains.>`

<!-- Written last. A reader should know the game and the QA priorities after this paragraph alone. -->

## 2. Introduction and game concept

`<Game idea, genre, number of players, single device vs online multiplayer, a typical round from start to finish. One mock-up or screen-flow sketch helps.>`

<!-- Keep it to about half a page. Describe the game, not the architecture: no classes, patterns or modules here. -->

## 3. Functional requirements

| ID | Description | Priority |
|---|---|---|
| FR1 | The user shall be able to create a game lobby and receive a join code. | H |
| FR2 | The user shall be able to join a lobby by entering a join code. | H |
| FR3 | `<...>` | M |
| FR4 | `<e.g. The user shall be able to change the sound volume in settings.>` | L |

Priority: **H** = the game is not playable without it; **M** = expected in the delivered game; **L** = nice to have, implemented if time allows.

<!-- Guidance:
- One testable behaviour per row, phrased "The user/system shall ...". Avoid "and" joining two features.
- Low-priority requirements are fine and expected; they show scope and give the ATAM group and the teacher something to trade off.
- No quality words ("fast", "easy") here; those belong in section 4 as scenarios.
- No design decisions ("uses Firebase listeners"); a mandated technology is a constraint (section 5). -->

## 4. Quality requirements

<!-- Each requirement is a concrete six-part scenario (SAiP 4th ed. ch. 3; 3rd ed. ch. 4). A general scenario is system-independent; yours must be concrete, about THIS game, with a measurable response measure. A requirement without a number is a wish, not a requirement. One response measure per scenario; split compound measures into separate IDs.
Six-part scenario structure from SAiP (4th ed. ch. 3; 3rd ed. ch. 4); wording partly adapted from the Wikipendium TDT4240 compendium (CC BY-SA 3.0, credit to its authors; full credit: ../CREDITS.md). -->

Scenario IDs: **M** = modifiability, **U** = usability, **P** = performance (if you pick others, e.g. **A** = availability, **S** = security, **T** = testability, **I** = interoperability/integrability, **D** = deployability; security and safety both start with S, so pick unambiguous prefixes and define them here). Aim for about three per chosen attribute, ordered by priority.

### 4.1 Modifiability (primary)

**M1: Add a new game mode** (priority H) — filled example

| Part | Value |
|---|---|
| Source of stimulus | Developer on the team |
| Stimulus | Wishes to add a new game mode (e.g. a timed mode) |
| Artifacts | Game logic and screen/state code in the `core` module |
| Environment | Design / development time |
| Response | The mode is added, tested and deployed without changing existing game modes or the backend interface |
| Response measure | At most `<3>` existing classes modified; done within `<8>` person-hours |

**M2: `<e.g. Replace the backend (Firebase -> another service)>`**

| Part | Value |
|---|---|
| Source of stimulus | `<...>` |
| Stimulus | `<...>` |
| Artifacts | `<...>` |
| Environment | `<...>` |
| Response | `<...>` |
| Response measure | `<number + unit>` |

**M3:** `<copy the table>`

### 4.2 Usability

**U1: First-time player starts a game** (priority M) — filled example

| Part | Value |
|---|---|
| Source of stimulus | New end user who has never played the game |
| Stimulus | Wants to learn how to start and play a round |
| Artifacts | Main menu, tutorial screen, in-game UI |
| Environment | Runtime, first launch |
| Response | The game shows a short tutorial and clear buttons; the player starts a round without outside help |
| Response measure | 9 of 10 test users start a round within `<60>` s without asking for help |

**U2:** `<e.g. undo / cancel a move>` **U3:** `<...>` (copy the table)

### 4.3 Performance

**P1: Opponent's move appears** (priority M) — filled example

| Part | Value |
|---|---|
| Source of stimulus | Remote opponent (another device) |
| Stimulus | Submits a move (sporadic event) |
| Artifacts | Backend sync (e.g. Firebase listener) and the local game state and renderer |
| Environment | Normal operation, both devices on 4G or Wi-Fi |
| Response | The move is received, the local state updated and the board redrawn |
| Response measure | Latency under `<1>` s for 95% of moves |

**P2:** `<e.g. frame rate stays at or above <30> FPS with <N> sprites on screen on a mid-range test phone>` **P3:** `<...>` (copy the table)

<!-- Do NOT mix usability and performance (a recurring teacher comment):
- Usability = the user's ability to learn, operate, recover from errors, feel in control. Measures: task time for a user, error count, success rate, satisfaction.
- Performance = the system's timing under events. Measures: latency, throughput, jitter, frame rate, miss rate.
- "The menu opens within 200 ms" is PERFORMANCE even though a user taps it. "The user finds the settings within 10 s" is USABILITY.
- Source of a usability scenario is an end user; the stimulus is an attempt to use the system. Source of a performance scenario is an event arriving (user, network, timer).
Other checks: the response measure is a number with a unit and a threshold; the environment is the operating condition when the stimulus arrives (e.g. design time, runtime, normal operation, overload, degraded network), not a physical location; the artifact is part of your system, not "the app" every time. -->

## 5. COTS components and technical constraints

<!-- A constraint is a decision you no longer have freedom over: it was fixed by the course (e.g. an Android game) or it follows from a COTS you have committed to. List what each COTS forces on you. Do not present your own later design choices (e.g. MVC, ECS) as constraints; those belong in the architecture document. For each COTS, the teacher asks for three things: the technical/architectural constraints it imposes, the interfaces you must implement, and how it affects control flow. -->

| COTS | Constraints it imposes | Interfaces you must implement / extend | Effect on control flow |
|---|---|---|---|
| libGDX | Java/Kotlin; `core` module shared by the `android` and `lwjgl3` launchers (gdx-liftoff layout); rendering through libGDX classes (`SpriteBatch`, `Stage`, `Screen`) | `ApplicationListener` / `ApplicationAdapter` or `Game`; `Screen` for each screen; `InputProcessor` for input | Inversion of control: libGDX owns the main loop and calls `create()`, `resize()`, `render()` (once per frame), `pause()`, `resume()` and `dispose()` on the `ApplicationListener`. With `Game`/`Screen`, `Game` forwards these to the active screen, whose `render(float delta)` receives the frame time; with a plain `ApplicationAdapter`, read it with `Gdx.graphics.getDeltaTime()`. Game logic runs inside the per-frame render call and must not block it |
| Android | Must run on the target API level `<minSdk ...>`; app can be paused or killed by the OS; touch input; varying screen sizes; network may drop | Android launcher (`AndroidApplication`) provided by libGDX; any platform service (e.g. auth) via an interface in `core` implemented in `android` | Lifecycle events arrive via `pause()` / `resume()`; state must be saved on pause. Platform code is reached through interfaces because `core` cannot depend on Android APIs |
| Firebase `<Realtime Database / Firestore, Auth>` | Needs network; JSON/document data model; security rules; Android SDK only (desktop launcher needs a stub or the REST API); free-tier quotas | Wrapper interface in `core` (e.g. `BackendAPI`) with an Android implementation; listener callbacks | Asynchronous: results arrive in callbacks on another thread, not in the render loop. Post results back to the render thread (e.g. `Gdx.app.postRunnable`) |
| `<other: Supabase, Box2D, Ashley ECS...>` | `<...>` | `<...>` | `<...>` |

Other technical constraints: `<e.g. must be playable on the course's test devices; source in the group's GitLab/GitHub repo; delivery deadline>`.

<!-- Only list constraints you actually have. Business constraints (deadline, group size) are fine here or in the architecture document's business drivers. Deeper platform material: [platforms-and-emerging-topics.md](../references/platforms-and-emerging-topics.md). -->

## 6. Issues

| # | Issue | Status / decision |
|---|---|---|
| I1 | `<Open question, e.g. "Is online multiplayer feasible with our Firebase quota?">` | `<Open / resolved: ...>` |

<!-- Record unresolved questions and known conflicts between requirements (e.g. a performance goal that pulls against modifiability). -->

## 7. Changes

| Version | Date | Change | Author(s) |
|---|---|---|---|
| 1.0 | `<YYYY-MM-DD>` | First delivery | `<...>` |
| 1.1 | `<YYYY-MM-DD>` | `<e.g. Revised P1 after teacher feedback; added FR9>` | `<...>` |

<!-- Log every change after the first delivery, including changes made after ATAM or feedback. -->

## 8. Individual contributions

| Member | Contribution to this document |
|---|---|
| `<Name>` | `<e.g. Functional requirements, U1-U3>` |

<!-- Be specific. Per ../references/course-and-project-guide.md section 1.1, group members normally get the same grade unless someone's contribution is inadequate; check the assignment text for how individual contributions affect grading. -->

## References

`<Full citations for anything you used, e.g. Bass, Clements & Kazman, Software Architecture in Practice, 4th ed., Addison-Wesley, 2021; libGDX and Firebase documentation URLs.>`

---

## Self-check before hand-in

- [ ] Front page has game title, group number, all members, chosen COTS, primary and secondary QAs, date.
- [ ] Project summary can be read alone and matches the final content.
- [ ] Game concept describes the game, with no design decisions.
- [ ] Every functional requirement has an ID, one testable behaviour and a H/M/L priority; low-priority items included.
- [ ] No quality words or technology choices hidden in functional requirements.
- [ ] Every quality requirement is a concrete six-part scenario with an ID (M1..., U1..., P1...).
- [ ] Every response measure has a number, unit and threshold.
- [ ] Usability scenarios measure users; performance scenarios measure timing. None are mixed.
- [ ] The primary QA has at least as many and as detailed scenarios as the others.
- [ ] Each COTS lists constraints, interfaces to implement, and effect on control flow (libGDX loop, Android lifecycle, Firebase async callbacks).
- [ ] Issues, changes and individual contributions sections are filled in.
- [ ] References included; placeholders and guidance comments removed.
- [ ] Scenario IDs are ready to be reused as ASRs in the architecture document and in the ATAM utility tree ([atam-evaluation.md](atam-evaluation.md)).
