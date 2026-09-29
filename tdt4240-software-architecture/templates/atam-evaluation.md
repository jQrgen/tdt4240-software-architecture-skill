# Template: ATAM evaluation of another group's architecture

> **The official template takes precedence.** If the TDT4240 course staff publish an ATAM
> template or assignment text (Blackboard or the course page), follow its headings, order and
> page limits. Use this file only as a fill-in aid and a completeness check. Method: [../references/evaluation.md](../references/evaluation.md). Scenarios and tactics: [../references/quality-attributes-classic.md](../references/quality-attributes-classic.md), [../references/quality-attributes-4th-edition.md](../references/quality-attributes-4th-edition.md). Patterns: [../references/architectural-patterns.md](../references/architectural-patterns.md).
> Evaluated documents usually follow [requirements-document.md](requirements-document.md) and [architecture-document.md](architecture-document.md).

How to use: replace every `<...>`, delete guidance quotes (`>`) and *(example)* rows (a fictional
Firebase-backed libGDX game), and keep scenario IDs identical to the evaluated group's (e.g. M1, P2).

---

## 1. Introduction

| Field | Value |
|---|---|
| Evaluating group | Group <n>: <member names> |
| Evaluated group | Group <m>, game "<title>" |
| Document(s) evaluated | Architecture document v<x.y>, dated <date>; requirements document v<x.y> |
| Date(s) of evaluation | <date(s)> |
| Game concept (1-2 sentences) | <what the game is, platforms, single/multiplayer> |
| COTS | <e.g. libGDX, Android, Firebase Realtime Database / Firestore> |

> State that the evaluation reflects the document version above; later changes are out of scope.

## 2. Evaluation method

ATAM (Architecture Tradeoff Analysis Method; Kazman, Klein & Clements, CMU/SEI-2000-TR-004;
SAiP 4th ed. ch. 21; 3rd ed. also ch. 21) analyses an architecture against prioritised quality-attribute
scenarios. Its outputs include the prioritised scenarios (utility tree), a mapping of architectural
approaches to QAs, risks and non-risks, sensitivity and tradeoff points, and risk themes. The nine steps:

| # | Step | How we did it |
|---|---|---|
| 1 | Present the ATAM | <e.g. short briefing to the evaluated group> |
| 2 | Present business drivers | <read from their docs / meeting> |
| 3 | Present the architecture | <read from their docs / presentation by group m> |
| 4 | Identify architectural approaches | <see section 4> |
| 5 | Generate quality-attribute utility tree | <see section 5> |
| 6 | Analyse architectural approaches | <see section 6> |
| 7 | Brainstorm and prioritise scenarios | <see section 7> |
| 8 | Analyse architectural approaches (again) | <new high-priority scenarios from step 7> |
| 9 | Present results | <report + any meeting> |

**Deviations from full ATAM:** <e.g. no separate stakeholder phase (only the two groups took part); steps 2-3 done from their documents; time-boxed to <n> hours>.

## 3. Business drivers and summary of the evaluated architecture

**Business drivers / goals** (from the evaluated group's documents):
- <e.g. deliver a playable multiplayer game by the course deadline>
- <e.g. easy to add new game modes after delivery>
- Primary QA: <e.g. modifiability>. Secondary QAs: <e.g. performance, usability>.
- Constraints: <e.g. Android, libGDX, Firebase, group of 6, one semester>

**Architecture summary** (our understanding; keep to what the document says):
- Main architectural pattern(s): <e.g. client-server with MVC in the client>
- Key views provided: <logical / process / development / physical>
- Notable gaps or ambiguities in the document: <list; say where information was missing>

## 4. Architectural approaches identified

| # | Approach (tactic or pattern) | Kind | Where in the document | QA(s) it targets |
|---|---|---|---|---|
| A1 | <e.g. MVC> | Architectural pattern | <section / view> | <modifiability> |
| A2 | <e.g. State pattern for screens> | Design pattern | <section> | <modifiability> |
| A3 | <e.g. Encapsulate backend behind an interface> | Tactic (reduce coupling) | <section> | <modifiability, testability> |
| A4 *(example)* | Direct calls from game controllers to the Firebase SDK | (absence of a tactic) | Logical view | Modifiability (claimed) |

> Keep tactics and patterns apart: a tactic targets one QA response; a pattern bundles tactics.

## 5. Utility tree

Rating: (business importance, technical difficulty), each H/M/L. Scenarios rated highest, typically
(H,H), then (H,M)/(M,H), are analysed in section 6 as time permits; state the cutoff we used.

| Quality attribute | Refinement | Scenario (ID + short text) | (Importance, Difficulty) |
|---|---|---|---|
| <QA> | <refinement> | <ID: source, stimulus ... response measure> | (<H/M/L>, <H/M/L>) |
| Modifiability *(example)* | Replace backend | M1: a developer replaces Firebase with another backend at design time; done in under <n> person-days, touching only the backend module | (H, H) |
| Performance *(example)* | Game-state sync latency | P1: opponent's move is shown on the other device within <n> ms under normal network conditions | (H, M) |

## 6. Analysis of high-priority scenarios

> Copy this table once per high-priority scenario.

| Field | Content |
|---|---|
| Scenario ID | <e.g. M1> |
| Scenario (six parts) | Source: <> / Stimulus: <> / Artifact: <> / Environment: <> / Response: <> / Response measure: <> |
| Attribute(s) | <QA> |
| Architectural approaches | <A-numbers from section 4> |
| Sensitivity points | <S-n: property of one or more components critical to this response> |
| Tradeoff points | <T-n: a property that is a sensitivity point for more than one QA (typically improving one while degrading another)> |
| Risks | <R-n: decision that may cause an undesired QA response> |
| Non-risks | <N-n: decision judged sound for this scenario, with the assumption it relies on> |
| Reasoning | <why; refer to views, diagrams, tactic names> |

*(example)*

| Field | Content |
|---|---|
| Scenario ID | M1 |
| Scenario (six parts) | Source: developer / Stimulus: replace Firebase with another backend / Artifact: backend access code / Environment: design time / Response: change made and tested / Response measure: <= <n> person-days, only backend module changed |
| Attribute(s) | Modifiability |
| Architectural approaches | A4 |
| Sensitivity points | S1: number of classes that import Firebase SDK types |
| Tradeoff points | T1: calling the Firebase SDK directly (no adapter layer) removes one level of indirection, slightly better for performance, but spreads backend coupling across controllers, worse for modifiability and testability |
| Risks | R1: a single Firebase dependency without an abstraction threatens the modifiability goal M1; replacing the backend would touch every controller |
| Non-risks | N1: using Firebase realtime listeners for sync is adequate for turn-based play (assumes fewer than <n> updates/s) |
| Reasoning | The logical view shows controllers calling the Firebase SDK directly; no backend interface or intermediary (reduce-coupling tactics) is present |

## 7. Brainstormed scenarios and prioritisation

| ID | Type (use case / growth / exploratory) and scenario | QA | Votes | In utility tree already? | Analysed in step 8? |
|---|---|---|---|---|---|
| B<n> | <type>: <> | <> | <n> | <yes/no (ID)> | <yes/no> |
| B1 *(example)* | Growth: add a 4-player mode after delivery | Modifiability | 5 | yes (M1) | no |

> Voting rule used: <e.g. about 30% of the scenario count per participant, rounded up, as SAiP suggests; verify>. Compare with the utility tree: new drivers? missed QAs?

## 8. Consolidated findings

**Risks**

| ID | Risk | Scenario(s) | Affected QA / business driver |
|---|---|---|---|
| R1 *(example)* | Firebase used directly from controllers, no backend abstraction | M1 | Modifiability; "add modes after delivery" |
| R<n> | <> | <> | <> |

**Non-risks**

| ID | Non-risk | Scenario(s) |
|---|---|---|
| N<n> | <> | <> |

**Sensitivity points**

| ID | Sensitivity point | QA affected |
|---|---|---|
| S<n> | <> | <> |

**Tradeoff points**

| ID | Tradeoff point | QAs in tension |
|---|---|---|
| T<n> | <> | <e.g. modifiability vs performance> |

> A tradeoff point is a decision that is a sensitivity point for two or more QAs in opposite directions;
> name the QA and the response for every S and T entry (see [../references/evaluation.md](../references/evaluation.md), section 2.5).

## 9. Risk themes and impact on business drivers

| Theme | Risks grouped | Business driver threatened | Impact |
|---|---|---|---|
| <e.g. Tight coupling to external services> | R1, R<n> | <e.g. extend after delivery> | <H/M/L + one sentence> |

## 10. Recommendations

| # | Recommendation | Addresses | Suggested tactic / pattern | Effort (H/M/L) |
|---|---|---|---|---|
| 1 *(example)* | Introduce a backend interface (e.g. `GameBackend`) with a Firebase implementation, created by a factory | R1, M1 | Encapsulate; use an intermediary (adapter/interface); Factory Method or simple factory (Abstract Factory only if several related backend services are swapped together) | M |
| <n> | <> | <> | <> | <> |

> Recommend; do not redesign their system. Point to document gaps separately from design risks.

## 11. Evaluation process reflection

- What worked well: <e.g. utility tree made priorities explicit>
- What was hard: <e.g. process view too thin to judge performance scenarios>
- What we would do differently: <>
- What we learned for our own architecture: <>

## 12. References

- Evaluated group's requirements and architecture documents: <group, title, version, date>.
- Kazman, R., Klein, M., Clements, P. *ATAM: Method for Architecture Evaluation*. CMU/SEI-2000-TR-004, 2000.
  https://insights.sei.cmu.edu/library/atam-method-for-architecture-evaluation/
- Bass, L., Clements, P., Kazman, R. *Software Architecture in Practice*, 4th ed., Addison-Wesley, 2021
  (4th ed. ch. 21, "Evaluating an Architecture"; 3rd ed. 2013, ch. 21, "Architecture Evaluation"). Add any other sources you used.
