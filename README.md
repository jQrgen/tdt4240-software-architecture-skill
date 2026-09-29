# TDT4240 Software Architecture skill

An [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
that turns Claude into a study companion and reviewer for NTNU's **TDT4240 Software
Architecture** (*Programvarearkitektur*). It explains the syllabus, writes and checks
six-part quality attribute scenarios and tactic trees, helps choose architectural and
design patterns, coaches exam answers, and reviews the group project's requirements,
architecture and ATAM documents.

The skill lives in [`tdt4240-software-architecture/`](tdt4240-software-architecture/):
a short [`SKILL.md`](tdt4240-software-architecture/SKILL.md) plus reference files and
document templates that Claude loads only when a question needs them.

> Unofficial study aid. Not affiliated with or endorsed by NTNU or the course staff.
> Always check this year's reading list (Leganto), Blackboard and the course page;
> official templates and instructions take precedence.

## Syllabus coverage

Chapter numbers refer to Bass, Clements & Kazman, *Software Architecture in Practice*
(SAiP). The 4th edition (2021) is current; the public 2015/2016 exams and the Wikipendium
compendium use the 3rd edition (2013). The skill gives both numbers where they differ.

| Topic | SAiP 4th ed. | SAiP 3rd ed. | Reference file |
|---|---|---|---|
| What architecture is, structures, why it matters, contexts | ch. 1-2 | ch. 1-3 | `foundations.md` |
| Quality attributes, scenarios and tactics in general | ch. 3 | ch. 4 | `quality-attributes-classic.md` |
| Availability, modifiability, performance, security, testability, usability | ch. 4, 8, 9, 11, 12, 13 | ch. 5, 7, 8, 9, 10, 11 | `quality-attributes-classic.md` |
| Interoperability → integrability | ch. 7 (integrability) | ch. 6 (interoperability) | both QA files |
| Deployability, energy efficiency, safety, other QAs | ch. 5, 6, 10, 14 | ch. 12 (other QAs) | `quality-attributes-4th-edition.md` |
| ASRs, QAW, utility tree, ADD, architecture debt | ch. 19, 20, 23 | ch. 16, 17 | `requirements-and-design.md` |
| Architectural patterns (Layered, MVC, Broker, Pipe-and-Filter, Client-Server, P2P, SOA, Pub-Sub, Shared-Data, Map-Reduce, Multi-tier) | inside QA chapters | ch. 13 | `architectural-patterns.md` |
| GoF design patterns, game architecture (Rollings & Morris ch. 17), game loop, ECS | article / book | article / book | `design-and-game-patterns.md` |
| Documentation: Views and Beyond, Kruchten 4+1, IEEE 1471 / ISO 42010 (+ arc42, C4, ADRs) | ch. 22 | ch. 18 | `documentation.md` |
| Evaluation: ATAM, utility tree, lightweight evaluation, CBAM | ch. 21 | ch. 21, 23 | `evaluation.md` |
| Cloud, virtualization, interfaces, mobile, edge-dominant systems, quantum | ch. 15-18, 26 | ch. 26-27 | `platforms-and-emerging-topics.md` |
| Coplien (1998) on design patterns | article | article | `architectural-patterns.md` |
| Exam strategy, worked answers, glossary, drills | | | `exam-prep.md` |
| Course facts, project workflow, common feedback | | | `course-and-project-guide.md` |
| Requirements, architecture and ATAM document scaffolds | | | `templates/` |

Which edition and which chapters are on this year's list is not publicly confirmed; the
skill says so rather than guessing.

## Install

**Claude Code (personal, all projects)**

```sh
git clone https://github.com/jQrgen/tdt4240-software-architecture-skill.git
mkdir -p ~/.claude/skills
cp -R tdt4240-software-architecture-skill/tdt4240-software-architecture ~/.claude/skills/
# or keep it updatable with a symlink:
# ln -s "$PWD/tdt4240-software-architecture-skill/tdt4240-software-architecture" ~/.claude/skills/
```

**Claude Code (one project, shared with your group)**

Copy or symlink the `tdt4240-software-architecture` folder into the project's
`.claude/skills/` directory and commit it.

**claude.ai / Claude desktop**

Zip the `tdt4240-software-architecture` folder (the zip must contain the folder with
`SKILL.md` at its top level) and upload it under *Settings → Capabilities → Skills*.

```sh
cd tdt4240-software-architecture-skill
zip -r tdt4240-software-architecture.zip tdt4240-software-architecture
```

Claude picks the skill up automatically when a question matches its description; you
can also ask for it by name.

## Example prompts

- "Write a six-part availability scenario for our libGDX multiplayer game and list the
  tactics that achieve it."
- "Give me the full modifiability tactic tree and one tradeoff for each group."
- "Which pattern fits best: a system where sensor readings are transformed in a fixed
  sequence of steps? Why not Pub-Sub?"
- "Explain the difference between a view, a viewpoint and a structure (IEEE 1471 vs SAiP)."
- "Review the process view in our architecture document. Is it consistent with the
  logical view?"
- "We are evaluating another group with ATAM. Help us build the utility tree and find
  sensitivity and tradeoff points."
- "Quiz me on TDT4240 exam-style short questions and mark my answers."
- "What changed in the QA chapters between the 3rd and 4th edition of SAiP?"

Test prompts with expected behaviour are in [`evals/evals.json`](evals/evals.json).

## Credits

Based in part on the **Wikipendium TDT4240 Software Architecture compendium**
(<https://www.wikipendium.no/TDT4240_Software_Architecture>; page history
<https://www.wikipendium.no/TDT4240_Software_Architecture/history/>; version last
modified 11 January 2022), licensed CC BY-SA 3.0, written by: matsbyr, agavaa, forbord, hoanghn, Gustav Dyngeseth, sindrsb, Myau,
bujordet, thormartin91, iverjo, finninde, Esso, torbjoks, nina, hoyby, henloef,
larsekje, mariufa, haakonmt, simenkj, sklirg, Andreas Melzer, loremipsum,
balazsorban, mathierl. The compendium text has been paraphrased, restructured,
corrected against the textbook and extended; it is not reproduced verbatim.
The same attribution ships inside the skill folder as
[`tdt4240-software-architecture/CREDITS.md`](tdt4240-software-architecture/CREDITS.md),
so installed copies keep it.

The approach to writing an architecture description (stakeholders and concerns,
viewpoints, views and rationale in the IEEE 1471 / ISO 42010 style) draws on the
wallywallet architecture documentation merge request
<https://gitlab.com/wallywallet/wallet/-/merge_requests/853>, used as a worked
real-world example in `references/documentation.md`.

Primary literature:

- Len Bass, Paul Clements, Rick Kazman. *Software Architecture in Practice*, 4th ed.,
  Addison-Wesley, 2021 (3rd ed. 2013 mapped where chapter numbers differ).
- Paul Clements et al. *Documenting Software Architectures: Views and Beyond*,
  2nd ed., Addison-Wesley, 2010.
- Philippe Kruchten. "The 4+1 View Model of Architecture." *IEEE Software* 12(6), 1995.
- IEEE Std 1471-2000, *Recommended Practice for Architectural Description of
  Software-Intensive Systems*, and its successor ISO/IEC/IEEE 42010.
- James O. Coplien. "Software Design Patterns: Common Questions and Answers." In
  *The Patterns Handbook*, Cambridge University Press, 1998.
- Andrew Rollings, Dave Morris. *Game Architecture and Design: A New Edition*,
  New Riders, 2004, ch. 17.
- Rick Kazman, Mark Klein, Paul Clements. *ATAM: Method for Architecture Evaluation*,
  CMU/SEI-2000-TR-004, 2000.
- Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides. *Design Patterns*,
  Addison-Wesley, 1994.

## License

This repository is licensed under
[Creative Commons Attribution-ShareAlike 4.0 International](LICENSE) (CC BY-SA 4.0).
It is an adaptation of CC BY-SA 3.0 material from Wikipendium; licensing the adaptation
under a later version of the same license is permitted by CC BY-SA 3.0 §4(b). If you
share or adapt it, credit the Wikipendium authors and this repository and keep the
same license.

Textbook and article titles are cited for reference only; no copyrighted text from them
is included.

## Author

Jørgen S. Notland ([github.com/jQrgen](https://github.com/jQrgen))
