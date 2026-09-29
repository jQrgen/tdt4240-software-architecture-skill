# Software Architect skill for Claude

An [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
that makes Claude work like a senior/staff software architect inside your codebase. It
designs and builds features and systems, reviews pull requests and whole repositories for
architecture, and writes decision records and architecture documentation. It is language
and framework agnostic, with concrete examples for Kotlin/Java/KMP, TypeScript, Python, Go,
Rust and .NET.

The skill lives in [`software-architect/`](software-architect/): a procedural
[`SKILL.md`](software-architect/SKILL.md) plus reference files and templates that Claude
loads only when a task needs them.

## What it does

**Design and build.** Recovers the current structure, elicits architecturally significant
requirements as six-part quality-attribute scenarios with numeric response measures, picks
tactics and patterns with explicit tradeoffs and a named runner-up, lays out modules,
ports and the composition root, writes ADRs, produces a compilable skeleton with one
vertical slice, and adds architecture tests (fitness functions) to your existing CI.

- "Design a notification service. It must deliver 99.9 % of messages within 5 s and survive
  a provider outage."
- "How should I structure offline sync in this KMP app?"
- "Split the `core` module of this Gradle project; it has become a dumping ground."

**Review a PR.** Classifies the change, maps every new import edge onto the intended
dependency rules, checks for bypassed ports, missing timeouts/idempotency, contract breaks
and ADR drift, and returns a verdict with at most ~7 ranked findings, each with file:line
evidence, a concrete fix and a fitness function to prevent recurrence.

- "Review this PR for layering violations." / "Architecture review of MR !412."

**Review a repository.** Recovers the as-is module, component-and-connector and allocation
views, compares them with the intended architecture (reflexion model), finds cycles,
hotspots and change coupling, runs a lightweight ATAM-style evaluation on the top
scenarios, and reports risk themes and a now/next/later roadmap.

- "Recover the architecture of this repo and tell me why it is so hard to change."
- "Is this service ready to handle 10x traffic?"

**Decide, document, enforce.**

- "Event-driven or request/response between orders and billing?"
- "Write an ADR for moving from SQLite to Postgres."
- "Document this system with arc42 and C4 diagrams."
- "Make 'the domain must not depend on Android' an enforced rule."

## Install

**Claude Code (personal, all projects)**

```sh
git clone https://github.com/jQrgen/tdt4240-software-architecture-skill.git
mkdir -p ~/.claude/skills
ln -s "$PWD/tdt4240-software-architecture-skill/software-architect" ~/.claude/skills/software-architect
# or copy it: cp -R tdt4240-software-architecture-skill/software-architect ~/.claude/skills/
```

**Claude Code (one project, shared with the team)**

Copy or symlink `software-architect/` into the project's `.claude/skills/` directory and
commit it.

**claude.ai / Claude desktop**

Zip the folder so that the zip contains `software-architect/SKILL.md` at its top level, then
upload it under *Settings → Capabilities → Skills*.

```sh
cd tdt4240-software-architecture-skill
zip -r software-architect.zip software-architect
```

Claude picks the skill up when a request matches its description; you can also ask for it
by name.

## File map

Paths under `references/` and `templates/` are inside `software-architect/`.

| Path | Contents |
|---|---|
| `software-architect/SKILL.md` | Stance, mode selection, evidence rules, build/review/document/enforce procedures, finding format, terminology, reference index |
| `references/design-workflow.md` | ASRs, question bank, six-part scenarios, utility tree, ADD 3.0, tradeoff reasoning, anti-overengineering, architecture debt |
| `references/quality-attributes-runtime.md` | Availability, performance, security, safety, energy efficiency, usability: scenarios, tactics, code signals |
| `references/quality-attributes-change.md` | Modifiability, testability, deployability, integrability, cross-QA tradeoff matrix |
| `references/architectural-patterns.md` | Classic and practitioner patterns with code signatures, erosion signals, selection guide, anti-patterns |
| `references/design-patterns-in-code.md` | GoF and related in-process patterns as tactic carriers |
| `references/module-layout-and-interfaces.md` | Package layouts, allowed-dependency tables, interface design, skeletons in several languages, migration recipe |
| `references/fitness-functions.md` | Enforcement ladder, architecture tests per ecosystem, ratchets, CI wiring |
| `references/architecture-recovery-and-metrics.md` | Dependency graphs, reflexion models, Martin metrics, git-history hotspots |
| `references/review-playbook.md` | Smell catalogue, PR checklist, finding rules, severity rubric, worked example |
| `references/evaluation-methods.md` | Mini-ATAM, ATAM, lightweight evaluation, CBAM |
| `references/documentation.md` | ISO/IEC/IEEE 42010, Views and Beyond, 4+1, arc42, C4, ADRs, docs-as-code |
| `references/platforms-and-domains.md` | Cloud, containers, mobile, edge/IoT, ML-enabled, quantum, games and real-time |
| `templates/adr.md` | ADR with a mandatory "Enforced by" field, plus a filled example |
| `templates/architecture-description.md` | Design brief and lean arc42/C4/42010 architecture description |
| `templates/architecture-review-report.md` | PR and repository review reports |
| `software-architect/CREDITS.md` | Sources, attribution and licence notes |
| `evals/evals.json` | Practitioner prompts with expected behaviour, for testing the skill |

## Grounded in TDT4240

The theoretical backbone is the syllabus of NTNU's course TDT4240 Software Architecture:
Bass, Clements and Kazman, *Software Architecture in Practice* (4th ed., 2021, with 3rd-ed.
chapter numbers where topics moved), with quality attributes, six-part scenarios and
tactics, architectural and design patterns, Views and Beyond, Kruchten's 4+1 view model,
IEEE 1471 / ISO/IEC/IEEE 42010, ATAM and CBAM, and the chapters on cloud, mobile and
edge systems. The skill turns that material into working procedures for real codebases,
adding practitioner tools the syllabus does not cover: ADRs, C4, arc42, fitness functions
and behavioural code analysis. This repository started as a study companion for the course
and was rewritten as a practitioner skill. It is not affiliated with or endorsed by NTNU.

## Credits

Parts of the theory are adapted and paraphrased from the Wikipendium compendium
[TDT4240: Software Architecture](https://www.wikipendium.no/TDT4240_Software_Architecture)
(CC BY-SA 3.0). Its contributors, as listed on the page: matsbyr, agavaa, forbord,
hoanghn, Gustav Dyngeseth, sindrsb, Myau, bujordet, thormartin91, iverjo, finninde, Esso,
torbjoks, nina, hoyby, henloef, larsekje, mariufa, haakonmt, simenkj, sklirg, Andreas
Melzer, loremipsum, balazsorban, mathierl.

The architecture-description approach (AS-IS/TO-BE discipline, view map, ranked quality
goals with a conflict rule, intended vs actual layering, embodied vs proposed ADRs,
executable architecture rules) draws on the Wally wallet architecture description in
[wallywallet/wallet MR !853](https://gitlab.com/wallywallet/wallet/-/merge_requests/853).

Primary literature:

- Len Bass, Paul Clements, Rick Kazman. *Software Architecture in Practice*, 4th ed.,
  Addison-Wesley, 2021 (3rd ed. 2013).
- Paul Clements et al. *Documenting Software Architectures: Views and Beyond*, 2nd ed.,
  Addison-Wesley, 2010.
- Philippe Kruchten. "The 4+1 View Model of Architecture." *IEEE Software* 12(6), 1995.
- ISO/IEC/IEEE 42010 and its predecessor IEEE Std 1471-2000.
- Rick Kazman, Mark Klein, Paul Clements. *ATAM: Method for Architecture Evaluation*,
  CMU/SEI-2000-TR-004, 2000.
- Humberto Cervantes, Rick Kazman. *Designing Software Architectures: A Practical
  Approach*, Addison-Wesley, 2016.
- Neal Ford, Rebecca Parsons, Patrick Kua. *Building Evolutionary Architectures*,
  O'Reilly, 2017.
- Michael Nygard. "Documenting Architecture Decisions", 2011; Simon Brown, the C4 model;
  Gernot Starke and Peter Hruschka, arc42.
- David L. Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules", 1972;
  Robert C. Martin's package metrics; Adam Tornhill, *Your Code as a Crime Scene*; Murphy,
  Notkin and Sullivan, software reflexion models, 1995.
- James O. Coplien, "Software Design Patterns: Common Questions and Answers", 1998;
  Rollings and Morris, *Game Architecture and Design*; Robert Nystrom, *Game Programming
  Patterns*.

The full list with details is in [`software-architect/CREDITS.md`](software-architect/CREDITS.md).
Textbook and article titles are cited for reference only; no copyrighted text from them is
included.

Author: Jørgen S. Notland ([github.com/jQrgen](https://github.com/jQrgen)).

## License

[Creative Commons Attribution-ShareAlike 4.0 International](LICENSE) (CC BY-SA 4.0). This
repository is an adaptation of CC BY-SA 3.0 material from Wikipendium; licensing the
adaptation under a later version of the same license is permitted by CC BY-SA 3.0. If you
share or adapt it, credit the Wikipendium authors and this repository and keep the same
license.
