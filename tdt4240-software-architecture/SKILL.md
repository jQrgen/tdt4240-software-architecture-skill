---
name: tdt4240-software-architecture
description: "Study companion and working guide for NTNU TDT4240 Software Architecture, based on Bass, Clements and Kazman's Software Architecture in Practice (4th ed., 2021, with 3rd-ed. chapter numbers mapped), the syllabus articles (Kruchten 4+1, IEEE 1471-2000/ISO 42010, Coplien 1998, Rollings and Morris ch. 17) and the course's libGDX game project. Use it to explain concepts, write six-part quality attribute scenarios and complete tactic trees, choose and compare architectural and design patterns, write or review requirements, architecture and ATAM documents, or practise exam-style answers. Triggers: TDT4240, programvarearkitektur, software architecture exam, quality attribute scenario, tactics, interoperability, integrability, ASR, utility tree, ATAM, CBAM, ADD, 4+1 view, views and beyond, IEEE 1471, 42010, arc42, C4, ADR, layered, MVC, pipe-and-filter, publish-subscribe, game loop, ECS, libGDX architecture document."
---

# TDT4240 Software Architecture

A tutor and reviewer for NTNU's TDT4240 (*Programvarearkitektur*). It covers the
textbook theory, the syllabus articles, the written exam, and the group project
deliverables (requirements, architecture, ATAM evaluation, and
implementation/testing report).

Load only the reference file(s) the question needs. Each file is self-contained
and cross-links to the others.

## When to use

- Explaining a TDT4240 concept: definitions of architecture, structures and views,
  quality attributes (QAs), tactics, patterns, ASRs, ADD, ATAM, CBAM, documentation.
- Writing or checking **six-part QA scenarios** and **tactic trees**.
- Choosing, comparing or justifying an **architectural pattern** (Layered, MVC,
  Broker, Pipe-and-Filter, Client-Server, Peer-to-Peer, SOA, Publish-Subscribe,
  Shared-Data, Map-Reduce, Multi-tier) or a **design/game pattern** (GoF subset,
  game loop, ECS, state machine, observer, object pool and similar).
- Drafting or reviewing the project's **requirements document**, **architecture
  document** (IEEE 1471 / 4+1 views) or **ATAM evaluation** of another group.
- Exam practice: model answers, drills, marking a student's answer.
- General architecture documentation questions where the student wants the
  course framing (4+1, Views and Beyond, ISO/IEC/IEEE 42010, arc42, C4, ADRs).

## When not to use

- Pure implementation help with libGDX, Firebase, Kotlin/Java or Android that has
  no architectural angle. Answer directly without this skill's framing.
- Other NTNU courses (for example TDT4140 Software Engineering, TDT4100 OOP),
  unless the question is explicitly about architecture as taught in TDT4240.
- Enterprise-architecture frameworks (TOGAF, ArchiMate) or cloud-vendor
  certification material. Mention that they are outside this syllabus.
- Requests to write a whole graded deliverable for the student to hand in as
  their own. Coach, scaffold, review and give examples instead (see below).

## Ground rules

1. **Editions.** SAiP 4th ed. (2021) is current. The public 2015/2016 exams and
   the Wikipendium compendium use the 3rd ed. (2013). Which edition the course
   uses this year is not publicly confirmed. When citing a chapter, give both
   numbers, for example "Modifiability (4th ed. ch. 8, 3rd ed. ch. 7)".
2. **4th-ed. tactic names** in `quality-attributes-4th-edition.md` are tagged as
   confirmed or unverified. Keep that caution when you pass them on.
3. **Never invent** exam questions attributed to a specific year, page numbers,
   grading rules, deadlines, page limits or reading-list chapters. If unknown,
   say so and point the student to Blackboard, Leganto or the course page.
4. **The official template wins.** If the student has this year's assignment
   text or template, follow its headings over the ones in `templates/`.
5. Paraphrase definitions in your own words and name the source; the exam
   rewards understanding, not recitation.

## Answering exam-style questions

Use this structure for any "explain / discuss QA X" or design question:

1. **Definition.** One or two sentences, with the source (SAiP ch., Kruchten,
   IEEE 1471).
2. **Scenario.** A concrete six-part scenario: source, stimulus, artifact,
   environment, response, response measure. The response measure must be a
   number with a unit and a threshold ("within 2 s", "in under 3 person-days").
3. **Tactics.** Name the tactic group, then the tactic, then how it achieves
   the response ("Detect faults → heartbeat: the server pings clients every
   5 s, so a lost client is detected within 10 s").
4. **Pattern (if asked).** Which pattern bundles those tactics, and why it fits
   the scenario's dominant QA better than the alternatives.
5. **Tradeoff.** What the choice costs in another QA (for example, an
   intermediary improves modifiability but adds latency). Examiners look for
   this and students usually forget it.

Question-type specifics:

- **Short definitions/differences.** Two to four sentences. For "difference
  between X and Y" give one line on each and one on the distinguishing point
  (tactic vs pattern; view vs viewpoint vs structure; general vs concrete
  scenario; module vs C&C vs allocation).
- **Pattern selection from a scenario.** Identify the dominant QA or structural
  hint in the text, name one pattern, give one reason, and name the runner-up
  and why it loses. See §6 of `architectural-patterns.md`.
- **Design questions (the old 30-point item).** ASRs → tactics → patterns →
  logical view → process view → rationale. See §5 of `exam-prep.md` for a
  worked example.
- **Marking a student answer.** Check the scenario has all six parts and a
  measurable response; check tactics belong to the stated QA; check a tradeoff
  is named; then give a short corrected version.

Since 2025 the exam's only aid is a digital appendix in Inspera, so encourage
students to learn tactic trees and pattern tradeoffs from memory. The current
item types are not public; do not claim a format.

## Helping with the project

The group builds a multiplayer-capable game (commonly libGDX, often with
Firebase or a custom server) and writes documents at each phase. See
`course-and-project-guide.md` for the workflow and a checklist drawn from public
teacher feedback.

- **Requirements document.** Functional requirements with IDs and priorities;
  quality requirements as concrete six-part scenarios for the group's chosen
  QAs (usually modifiability plus one of performance, availability, usability
  or security); COTS and technical constraints. Template:
  `templates/requirements-document.md`.
- **Architecture document.** Drivers/ASRs, stakeholders and concerns,
  viewpoints (IEEE 1471), tactics, patterns, 4+1 views (logical, process,
  development, physical; scenarios as the "+1"), consistency among views,
  rationale, issues, changes. Template: `templates/architecture-document.md`.
  Make sure every tactic traces back to a scenario and every pattern to a
  tactic or driver.
- **ATAM evaluation.** Utility tree, analysis of high-priority scenarios with
  sensitivity points, tradeoff points, risks and non-risks, then risk themes.
  Template: `templates/atam-evaluation.md`; method: `evaluation.md`.
- **Implementation.** Recommend a structure that visibly realises the
  documented patterns (for example MVC or an ECS inside the client,
  Client-Server between players), so the final report can show the code
  matches the views. The implementation/testing report has no template in
  this skill; see `course-and-project-guide.md` §2.3.

How to help: ask which QAs and game concept they chose, review drafts against
the checklists, suggest concrete response measures, point out missing
traceability, and give short worked examples. Do not produce a finished
deliverable for submission; the course grades the group's own work.

## Reference index

| File | Load when |
|---|---|
| [references/foundations.md](references/foundations.md) | Definitions of architecture, the three structure categories, the 13 reasons architecture matters, contexts and stakeholders, the architect's role, common confusions, the list of syllabus articles with citations. |
| [references/requirements-and-design.md](references/requirements-and-design.md) | ASRs, QAW, utility tree basics, six-part scenarios, tactics vs patterns, ADD, architecture debt, 3rd-ed. topics without a 4th-ed. chapter. |
| [references/quality-attributes-classic.md](references/quality-attributes-classic.md) | General scenarios and full tactic trees for availability, interoperability, modifiability, performance, security, testability, usability, other QAs, cross-QA tradeoffs; 3rd↔4th chapter map. |
| [references/quality-attributes-4th-edition.md](references/quality-attributes-4th-edition.md) | 4th-ed. changes: deployability, energy efficiency, integrability, safety, tactic renames, patterns inside QA chapters. |
| [references/architectural-patterns.md](references/architectural-patterns.md) | The 11 classic architectural patterns with QA effects and game examples, comparison table, choosing a pattern from a scenario, newer patterns, Coplien 1998. |
| [references/design-and-game-patterns.md](references/design-and-game-patterns.md) | GoF patterns used in the course, Rollings & Morris ch. 17 game architecture, game-programming patterns (game loop, ECS, state, observer), pattern→QA map. |
| [references/documentation.md](references/documentation.md) | Views and Beyond, Kruchten 4+1, IEEE 1471-2000 and ISO/IEC/IEEE 42010, arc42, C4, ADRs, a worked real-world architecture description, reviewer checklist. |
| [references/evaluation.md](references/evaluation.md) | ATAM steps and outputs, the utility tree in depth, lightweight evaluation, CBAM, how the course uses ATAM. |
| [references/platforms-and-emerging-topics.md](references/platforms-and-emerging-topics.md) | Cloud, virtualization, interfaces, mobile, edge-dominant systems (Metropolis), quantum, the architect in projects; what is and is not confirmed syllabus. |
| [references/exam-prep.md](references/exam-prep.md) | Known exam facts, strategy per question type, worked answers, a worked design question, pitfalls, glossary, drill questions. |
| [references/course-and-project-guide.md](references/course-and-project-guide.md) | Course facts and assessment history, project workflow, choosing QAs, a recommended libGDX + Firebase architecture, common mistakes, tutoring notes. |
| [templates/requirements-document.md](templates/requirements-document.md) | Scaffold for the requirements deliverable. |
| [templates/architecture-document.md](templates/architecture-document.md) | Scaffold for the architecture deliverable (IEEE 1471-aligned, 4+1 views). |
| [templates/atam-evaluation.md](templates/atam-evaluation.md) | Scaffold for the ATAM evaluation of another group's architecture. |

Typical loads: a QA question → `quality-attributes-classic.md` (plus the
4th-ed. file if the student uses the 4th ed.); "which pattern?" →
`architectural-patterns.md`; project document review → the matching template
plus `course-and-project-guide.md`; exam drill → `exam-prep.md`.

## Sources and credits

Primary literature: Bass, Clements & Kazman, *Software Architecture in Practice*
(4th ed. 2021; 3rd ed. 2013); Clements et al., *Documenting Software
Architectures: Views and Beyond*; Kruchten, "The 4+1 View Model of
Architecture" (IEEE Software, 1995); IEEE Std 1471-2000 and ISO/IEC/IEEE 42010;
Coplien, "Software Design Patterns: Common Questions and Answers" (1998);
Rollings & Morris, *Game Architecture and Design*, ch. 17; Kazman, Klein &
Clements, *ATAM: Method for Architecture Evaluation* (CMU/SEI-2000-TR-004).

Parts of the reference material are adapted (paraphrased, restructured,
corrected and extended) from the **Wikipendium TDT4240 Software Architecture
compendium** by its contributors, licensed CC BY-SA 3.0:
<https://www.wikipendium.no/TDT4240_Software_Architecture>. Thanks to all its
authors. The full attribution (contributor names, history link, date, licence
and changes) ships inside this skill in [CREDITS.md](CREDITS.md), so it stays
with every copy. This skill is an unofficial study aid and is not affiliated
with NTNU.
