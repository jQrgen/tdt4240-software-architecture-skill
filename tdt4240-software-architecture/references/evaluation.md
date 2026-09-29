# Architecture evaluation: ATAM, utility tree, CBAM, lightweight evaluation

Sources: Bass, Clements & Kazman, *Software Architecture in Practice* (SAiP), 4th ed. (2021) ch. 21 "Evaluating an Architecture" (3rd ed., 2013: also ch. 21 "Architecture Evaluation"; CBAM is 3rd ed. ch. 23 "Economic Analysis of Architectures"); Kazman, Klein & Clements, *ATAM: Method for Architecture Evaluation*, CMU/SEI-2000-TR-004. The CBAM section is partly adapted from the Wikipendium TDT4240 compendium (CC BY-SA 3.0, https://www.wikipendium.no/TDT4240_Software_Architecture), completed and corrected from the book.

Related files: scenarios and the six-part format are in `quality-attributes-classic.md` / `quality-attributes-4th-edition.md`; ASRs, QAW and ADD are in `requirements-and-design.md`; the course procedure is in `course-and-project-guide.md`; the fill-in form is `../templates/atam-evaluation.md`.

---

## 1. Why, when and by whom

**Why evaluate.** Architecture decisions are the earliest and hardest to change, and they enable or inhibit the quality attributes (QAs). Finding a bad decision on paper costs a meeting; finding it after implementation costs a rewrite. Evaluation also has side benefits: it forces the business goals and QA requirements to be written down and prioritised, it gets stakeholders talking to each other, and it improves the architecture documentation (the architect has to present it).

**When.**
- Early (after the main decisions, before heavy coding): the most leverage. This is where TDT4240 puts it.
- Any time a major decision is on the table ("evaluate as you design").
- Before buying/acquiring a system, or when an existing system must evolve (evaluate the as-built architecture).
- Precondition: there must be *something* to evaluate (at least the main views and decisions) and people who can explain it.

**Cost/benefit.** An evaluation costs participant time (evaluation team, architect, decision makers, stakeholders). It pays off when the system is large or risky, the QAs are demanding, or decisions are expensive to reverse. For a small, low-risk decision a lightweight or self-evaluation is enough. (SAiP gives effort figures; quote them only from the book.)

**Who evaluates (4th ed. ch. 21 framing).**

| Form | Who | Typical use | Strength / weakness |
|---|---|---|---|
| Evaluation by the designer | The architect, continuously while designing | Every design decision (ADD's "analyse" step) | Cheap, constant; blind to own assumptions |
| Peer review | Colleagues from the same organisation / other teams | Milestones, lightweight evaluation | Fairly cheap, some independence; shared blind spots |
| Analysis by outsiders | External team (e.g. SEI-trained ATAM team) | High-stakes systems, acquisition | Objective, experienced; expensive, needs ramp-up |

ATAM is the canonical outsider method; the Lightweight Architecture Evaluation is its peer-review variant.

---

## 2. ATAM (Architecture Tradeoff Analysis Method)

**Abbreviation:** ATAM = **A**rchitecture **T**radeoff **A**nalysis **M**ethod (SEI, Kazman, Klein & Clements). An exam may ask exactly this.

**Purpose.** Assess the consequences of architectural decisions *in light of the business goals and QA requirements*, expressed as prioritised scenarios. ATAM does not produce a grade or a pass/fail and does not need detailed code; it finds risks, sensitivity points and tradeoffs.

### 2.1 Participants

| Group | Who | Role |
|---|---|---|
| Evaluation team | Usually 3-5 people external to the project. SAiP lists roles: team leader, evaluation leader, scenario scribe, proceedings scribe, questioner (one person can hold several roles) | Runs the method, asks questions, records results, writes the report |
| Project decision makers | Project manager, architect, customer/client representative | Present business drivers and the architecture; have authority over the project |
| Architecture stakeholders | Developers, testers, integrators, maintainers, users, operators, ... | Articulate QA needs, brainstorm and vote on scenarios (Phase 2) |

### 2.2 Phases

| Phase | Name | Participants | Content |
|---|---|---|---|
| 0 | Partnership and preparation | Evaluation team leadership + key decision makers | Agree logistics, scope, stakeholders, which architecture docs to hand over; team studies the docs |
| 1 | Evaluation (part 1) | Evaluation team + decision makers | Steps 1-6 |
| 2 | Evaluation (part 2) | Evaluation team + decision makers + stakeholders | Steps 7-9 (steps 1-6 are briefly recapped for the new participants) |
| 3 | Follow-up | Evaluation team + evaluation client | Write and deliver the final report; team self-assessment |

Phase 1 and Phase 2 are usually separated by a pause (weeks) in which the team follows up on open questions. Exact durations: check SAiP ch. 21.

### 2.3 The nine steps

**Phase 1 (steps 1-6): evaluation team + decision makers**

| # | Step | What happens | Output |
|---|---|---|---|
| 1 | Present the ATAM | Evaluation leader explains the method, what outputs to expect | Shared expectations |
| 2 | Present business drivers | Project manager/customer: business goals, main functions, constraints (technical, economic, managerial), stakeholders, architectural drivers | Business goals and drivers |
| 3 | Present the architecture | Architect presents views, how the architecture meets the drivers, technical constraints, other systems it interacts with | Concise architecture presentation |
| 4 | Identify architectural approaches | Team catalogues the patterns and tactics used (not analysed yet) | List of approaches |
| 5 | Generate the QA utility tree | Decision makers refine the QAs into concrete, prioritised scenarios (section 3) | Utility tree |
| 6 | Analyse architectural approaches | For the highest-ranked scenarios, the architect explains how the approaches achieve them; the team probes | Risks, non-risks, sensitivity points, tradeoff points, (initial) mapping approaches -> QAs |

**Phase 2 (steps 7-9): stakeholders join**

| # | Step | What happens | Output |
|---|---|---|---|
| 7 | Brainstorm and prioritise scenarios | Larger stakeholder group proposes use-case, growth and exploratory scenarios; they vote (SAiP: each stakeholder gets votes equal to about 30% of the number of scenarios, rounded up; verify). Result is compared with the utility tree; new high-priority scenarios are added as leaves | Prioritised scenario list, validated/extended utility tree |
| 8 | Analyse architectural approaches (again) | Repeat step 6 with the new high-priority scenarios from step 7 | More risks, non-risks, sensitivity and tradeoff points |
| 9 | Present results | Team presents findings back to stakeholders, including risk themes | Final briefing, later a written report (Phase 3) |

Mnemonic for the two groups: **1-6 = present, present, present, identify, tree, analyse**; **7-9 = brainstorm, analyse, present**.

### 2.4 ATAM outputs (exam list)

1. A concise **presentation of the architecture** (forced by step 3; often better than the existing documentation).
2. Articulated **business goals**.
3. Prioritised **QA requirements expressed as scenarios**.
4. The **utility tree**.
5. A set of **risks** and **non-risks**.
6. A set of **sensitivity points** and **tradeoff points**.
7. **Risk themes**: risks grouped by common underlying cause, each tied back to the business goals it threatens.
8. A **mapping of architectural approaches (decisions) to QAs**, showing how each approach helps or hurts.

Intangible outputs: stakeholder communication, a sense of community around the architecture, clarified QA requirements, better documentation.

### 2.5 Precise definitions (2016 exam: sensitivity point vs tradeoff point)

| Term | Definition | Test |
|---|---|---|
| **Sensitivity point** | A property of one or more components and/or component relationships (i.e. an architectural decision/parameter) that is critical for achieving a particular QA response. Changing it noticeably changes that response. | Affects **one** QA significantly |
| **Tradeoff point** | A property that affects **more than one** QA and is a sensitivity point for more than one, improving one while degrading another. | Sensitivity point for **two or more** QAs, in opposite directions |
| **Risk** | An architecturally important decision that is potentially problematic given the QA requirements (or an important decision not yet made). | "This may cause us to miss scenario X" |
| **Non-risk** | A good decision, judged safe, that often rests on an explicit or implicit assumption. Record the assumption: if it breaks, the non-risk becomes a risk. | "Fine, *as long as* ..." |
| **Risk theme** | A cluster of related risks pointing at a systemic weakness | "No consistent strategy for handling network loss" |

Key relationship: **every tradeoff point is a sensitivity point, but not every sensitivity point is a tradeoff point.** Sensitivity/tradeoff points are neutral facts about the design; risks/non-risks are judgements.

**Game-project examples (libGDX + Firebase multiplayer game):**

| Category | Example |
|---|---|
| Sensitivity point | The rate at which the client pushes player positions to Firebase determines how "live" the opponent's moves appear (performance/latency of shared state). |
| Tradeoff point | Server-side move validation in Cloud Functions: raises security (cheating prevented) but adds a network round trip per move (performance). Also: an interface layer over Firebase (use an intermediary) raises modifiability/portability but adds indirection and a little overhead. |
| Risk | All game state lives in a global `GameManager` singleton touched by every screen: modifiability and testability scenarios (e.g. "add a new game mode in < 2 days") are threatened. No reconnect strategy for a dropped Firebase connection: availability scenario at risk. |
| Non-risk | Using the State pattern (one `Screen` per game state) is fine for modifiability, *assuming* the game keeps a small number of screens. |
| Risk theme | "Global mutable state" (from the singleton and static listeners) threatens both modifiability and testability, hence the business goal of iterating quickly on gameplay. |

### 2.6 Analysis table template (steps 6 and 8)

One row per (scenario, decision) pair. Use IDs so risks can be clustered into themes later.

| Scenario (ID, text, priority) | Approach / decision | Sensitivity | Tradeoff | Risk | Non-risk | Reasoning |
|---|---|---|---|---|---|---|
| M1 (H,H): add a new power-up type in < 4 h, touching <= 3 classes | ECS: power-up = new component + system | S1: granularity of components | - | - | N1: new systems plug in without changing others (assumes system execution order is irrelevant) | ECS isolates behaviour in systems; order dependence would break the assumption |
| P1 (H,M): opponent's move visible within 500 ms on 4G | Realtime Database listener + client-side prediction | S2: push frequency | T1: push frequency vs battery/data use | R1: no throttling; burst writes may hit quota | - | Each move is one write; no batching |
| S1 (M,H): a modified client cannot report an illegal move | Client-authoritative moves, no server validation | - | T2: validation vs latency (if added) | R2: cheating possible | - | Security rules check only authentication, not game rules |

For each scenario in step 6/8 the evaluators ask: Which approaches realise this scenario? What are their parameters (sensitivity)? What else do those parameters affect (tradeoff)? What assumptions does this rely on (non-risk)? What is unaddressed or dubious (risk)?

---

## 3. The utility tree in depth

**Structure:** root **Utility** (the overall "goodness" of the system) -> **quality attribute** -> **attribute refinement** -> **concrete scenario** (leaf, ideally in six-part form with a response measure).

**Prioritisation:** each leaf gets two ratings on H/M/L:
1. **Importance to the success of the system** (business value), rated by the decision makers/customer.
2. **Difficulty / architectural impact / risk** of achieving it, rated by the architect.

**Reading the ratings:**
- **(H,H)**: important *and* hard. Analyse these first in step 6; this is where the architecture must prove itself and where risks hide.
- **(H,M), (M,H)**: next in line.
- **(H,L)**: important but easy, so little analysis is needed.
- **(L,H)**: hard but unimportant; question whether it is worth the cost.
- **(L,L)**: usually dropped from the analysis.
- A QA with no leaves is a gap; a leaf without a response measure is not yet a scenario.

**Example (multiplayer mobile game):**

```
Utility
├── Modifiability
│   ├── New gameplay content
│   │   └── (H,H) A developer adds a new game mode in <= 2 person-days, changing no networking code
│   └── Backend replacement
│       └── (M,H) Replace Firebase with Supabase in <= 1 week; only the backend adapter module changes
├── Performance
│   ├── Frame rate
│   │   └── (H,M) 60 FPS with 50 on-screen entities on a mid-range Android phone
│   └── Network latency
│       └── (H,H) Opponent's move rendered within 500 ms on 4G under normal load
├── Usability
│   └── Learnability
│       └── (M,L) A first-time player completes the tutorial in < 3 minutes without help
└── Availability
    └── Connection loss
        └── (H,M) After a 10 s network drop the match resumes with no lost moves
```

Step 6 would start with the two (H,H) leaves. The utility tree is built top-down by decision makers in Phase 1; step 7's stakeholder brainstorm is the bottom-up cross-check. If stakeholders rank highly a scenario the tree missed, that is itself a finding (the decision makers and stakeholders disagree on priorities).

Utility trees are also used outside ATAM to capture ASRs (see `requirements-and-design.md`).

---

## 4. Lightweight Architecture Evaluation

A shortened, in-house variant of ATAM for organisations or teams that evaluate regularly, where participants already know the method and the business context. At a high level:

- It is run by internal peers (peer review), not an external team, so no Phase 0 partnership or Phase 3 formal report is needed.
- It follows the ATAM step sequence, but steps whose content participants already know (explaining the method, business drivers) are cut to a brief recap, and it fits into a few hours to about a day.
- It still produces a utility tree, prioritised scenarios, and risks/non-risks/sensitivity/tradeoff points, but with less depth and less stakeholder coverage.
- Tradeoff: much cheaper and repeatable (e.g. every iteration), but less objective and less thorough than a full ATAM.

The book gives a specific step-by-step table with time allocations; verify the exact step list and durations against SAiP ch. 21 (the section exists in the 4th ed. and probably also in the 3rd ed.; unverified) before quoting it. The TDT4240 peer ATAM is effectively a lightweight evaluation.

---

## 5. CBAM (Cost Benefit Analysis Method)

**Status in the course:** SAiP 3rd ed. ch. 23. The Wikipendium compendium marks CBAM as *not* part of the Spring 2019 syllabus. The 4th ed. has no dedicated CBAM chapter (it may be mentioned in passing; verify). CBAM did appear in the 2016 exam (ATAM vs CBAM, the CBAM process), so know it if the current reading list includes the 3rd ed. chapter.

**Purpose.** ATAM tells you *what* the risks and tradeoffs are; CBAM tells you *which architectural strategies are worth their cost*. It makes economic questions explicit, e.g. "Would stakeholders pay more for a faster system?", "How much availability can the budget buy?", "Is a two-month delay acceptable for extra security?". It builds on ATAM output (scenarios, utility tree, approaches) and models the utility of each scenario as a function of its response measure (a utility-response curve).

### 5.1 The nine steps

| # | Step | Detail |
|---|---|---|
| 1 | Collate scenarios | Gather ATAM scenarios; stakeholders may add more; prioritise by business goals; **keep the top third** |
| 2 | Refine scenarios | For each, state response levels: **worst** (minimum acceptable threshold), **current**, **desired**, **best** (beyond which no further utility) |
| 3 | Prioritise scenarios | Each stakeholder distributes **100 votes**; weights come from the votes; **drop the lower half** |
| 4 | Assign utility | Stakeholders assign utility (0-100) to each response level (worst, current, desired, best) of each remaining scenario, giving a utility-response curve |
| 5 | Develop architectural strategies and their expected response levels | Architects propose strategies (AS_i) addressing the scenarios and estimate the response each one would produce for every scenario it affects |
| 6 | Determine expected utility | Read the utility of each expected response level off the curve by **interpolation** |
| 7 | Calculate total benefit | **B_i = sum_j (b_ij x W_j)**, where b_ij = expected utility of AS_i on scenario j minus current utility of scenario j, and W_j = scenario j's weight (normalised votes) |
| 8 | Choose strategies by value for cost | **VFC_i = B_i / C_i** (C_i = cost of AS_i); rank by VFC and pick from the top **within the budget** |
| 9 | Confirm with intuition | Do the chosen strategies fit the business goals? If not, revisit assumptions (missing scenarios, bad cost or utility estimates) |

Note: a strategy can affect several scenarios, sometimes negatively (a tradeoff), so b_ij may be negative.

### 5.2 Tiny worked example

Two scenarios survive step 3, weights W normalised from votes:

| Scenario | W | Utility curve (response -> utility) |
|---|---|---|
| S1 performance: match load time | 0.6 | 5 s -> 0 (worst), 2 s -> 40 (current), 1 s -> 80 (desired), 0.5 s -> 100 (best) |
| S2 availability: match survives server fault | 0.4 | 99% -> 50 (current), 99.9% -> 90 (desired) |

Strategies (step 5), costs in person-weeks:

| Strategy | Expected response | Expected utility (interpolated) | b_ij | B_i | C_i | VFC |
|---|---|---|---|---|---|---|
| A: cache match assets | S1: 1.5 s (S2 unchanged) | S1: halfway 40-80 = 60 | S1: +20 | 20 x 0.6 = **12** | 4 | **3.0** |
| B: passive redundancy | S2: 99.9%; S1 slows to 2.2 s | S2: 90; S1: 40 - (0.2/3) x 40 = 37.3 | S2: +40; S1: -2.7 | 40 x 0.4 - 2.7 x 0.6 = **14.4** | 8 | **1.8** |

B has the higher total benefit, but A has the better value for cost. With a budget of 10 person-weeks, choose A (4) first; B (8) no longer fits, so the result is A alone. With 12 or more, choose both. Step 9: check that "faster loading first, availability later" matches the business goals.

### 5.3 ATAM vs CBAM (2016 exam topic)

| Aspect | ATAM | CBAM |
|---|---|---|
| Full name | Architecture Tradeoff Analysis Method | Cost Benefit Analysis Method |
| Question answered | Does the architecture satisfy the QA goals? Where are the risks and tradeoffs? | Which architectural strategies give the most benefit for their cost? |
| Focus | Technical consequences of decisions | Economic consequences of decisions |
| Input | Business drivers, architecture, stakeholders | ATAM output (scenarios, approaches, risks) plus cost estimates |
| Treatment of scenarios | Each prioritised scenario is analysed against the architecture (H/M/L ranking) | Scenarios get response-level ranges, numeric weights (100 votes) and utility curves |
| Quantification | Qualitative (H/M/L, risks, points) | Quantitative (utility, B_i, C_i, VFC) |
| Output | Utility tree, risks/non-risks, sensitivity/tradeoff points, risk themes, approach-QA mapping | Ranked, costed set of strategies chosen within budget |
| Relationship | Comes first | Extends ATAM; typically run afterwards |

(The Wikipendium page contrasts them as "a set of scenarios vs one scenario at a time"; that phrasing is garbled. The real difference is qualitative risk analysis vs quantitative cost-benefit ranking.)

---

## 6. How TDT4240 uses ATAM

- In the project, each group evaluates **another group's architecture** with ATAM after the architecture document is delivered (historically, sometimes by two groups; the current setup is in the assignment text).
- The evaluating group is a **stakeholder** of the evaluated architecture, and the evaluated group's architecture document should list the ATAM evaluation group in its stakeholders table (explicit teacher feedback).
- In practice this is a compressed, peer-run ATAM: the evaluated group's document stands in for steps 2-3, the evaluators identify approaches (step 4), build a utility tree from the stated quality requirements (step 5), analyse scenarios (step 6/8), and report risks, non-risks, sensitivity and tradeoff points and risk themes (step 9). Follow the current course template and deadlines, which are not confirmed here.
- Useful checks when evaluating: Are the QA scenarios measurable? Does every high-priority scenario map to a tactic or pattern? Are tactics shown in the views? Does the rationale connect decisions to QAs? Are external components (e.g. Firebase) visible in the views?
- Use `../templates/atam-evaluation.md` for the report structure and the analysis table.

### Common student mistakes

- Calling any design weakness a "tradeoff point". It is a tradeoff point only if the *same* decision affects two QAs in opposite directions.
- Listing a sensitivity point without saying which QA and which response it is sensitive to.
- Non-risks without the assumption they depend on.
- Utility-tree leaves that are not scenarios ("the game should be fast").
- Forgetting risk themes, or listing them without linking them to business goals.
- Treating ATAM as a grade of the other group's work instead of a structured search for risks.
