# Architecture evaluation: mini-ATAM in a repo, ATAM, LAE, CBAM

Use this file when you must judge whether an architecture (designed or as-built) can meet its quality goals, rank remediation options, or run a formal evaluation on request. The repo-review procedure in [../SKILL.md](../SKILL.md) and [review-playbook.md](review-playbook.md) calls section 3 (mini-ATAM) of this file.

Sources: Bass, Clements & Kazman, *Software Architecture in Practice* (SAiP), 4th ed. (2021), ch. 21 "Evaluating an Architecture"; Kazman, Klein & Clements, *ATAM: Method for Architecture Evaluation*, CMU/SEI-2000-TR-004 (2000); CBAM from SAiP 3rd ed. (2013), ch. 23 "Economic Analysis of Architectures".

Cross-links (do not duplicate here):
- Six-part QA scenarios, ASRs, utility trees while designing: [design-workflow.md](design-workflow.md)
- Tactic trees (used as questionnaires in section 8): [quality-attributes-runtime.md](quality-attributes-runtime.md), [quality-attributes-change.md](quality-attributes-change.md)
- Recovering the as-is structure before tracing scenarios: [architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md)
- Turning findings into continuous checks: [fitness-functions.md](fitness-functions.md)
- Report layout: [../templates/architecture-review-report.md](../templates/architecture-review-report.md); decisions that follow: [../templates/adr.md](../templates/adr.md)

> Theory adapted in part from the Wikipendium TDT4240 compendium (CC BY-SA 3.0). See [../CREDITS.md](../CREDITS.md).

---

## 1. When to evaluate, and what it can tell you

Evaluate when a decision is about to become expensive to reverse:

| Trigger | Why now | Suggested form |
|---|---|---|
| Before a costly commitment (datastore, service split, sync protocol, vendor SDK, public API) | Paper is cheap; migrations are not | Mini-ATAM on the proposal, or CBAM if choosing among options |
| After a major change (new deployment topology, new tenant model, 10x load) | Old non-risks rest on assumptions that may now be false | Mini-ATAM focused on changed drivers |
| Inherited or acquired system | Nobody can state which decisions carry which qualities | Recovery first, then mini-ATAM on the as-built code |
| Large PR that touches a boundary or a shared mechanism | A local change can move a sensitivity point | Scenario walkthrough of the PR (section 8) |
| Recurring incidents of one kind | Probably a risk theme, not a bug | Mini-ATAM scoped to that attribute |

Precondition: something to evaluate (code, a design, or at least the main decisions) and a statement of goals. No goals means no evaluation, only opinions; elicit drivers first (section 3, step 1).

**An evaluation can tell you:** which decisions support or threaten each prioritised scenario; where the design is sensitive; which decisions trade one quality against another; which risks cluster into systemic themes; what the stakeholders actually care about (a side effect of writing scenarios).

**It cannot tell you:** that the system *will* meet a numeric target (reasoning shows plausibility; measurement shows fact: use load tests, chaos experiments, spikes); anything about scenarios nobody wrote down; whether the code is free of defects (evaluation targets decisions, not bugs); a grade or pass/fail. ATAM explicitly produces neither.

---

## 2. Precise definitions (SAiP 4e ch. 21; CMU/SEI-2000-TR-004)

Always name the attribute *and* the response when you state any of these.

**Architectural approach.** A pattern or tactic (or a combination) that the system uses to achieve a quality: Ports and Adapters, retry with backoff, a read cache, an outbox. In code it is a structure you can point at.

```ts
// src/payments/stripe-adapter.ts : approach = "use an intermediary" (modifiability)
//   + timeout, a form of exception detection under Detect Faults (availability)
import type { Agent } from "node:https";
export class StripeAdapter implements PaymentPort {
  constructor(private agent: Agent, private timeoutMs = 2000) {}
}
```

**Sensitivity point.** A property of one or more components and/or component relationships that is critical for achieving a particular QA response. Changing it noticeably moves that response.

```ts
// src/payments/stripe-adapter.ts:14
import { Agent } from "node:https";
const agent = new Agent({ keepAlive: true, maxSockets: 20 }); // SP: p99 checkout latency at 200 req/s depends on this;
                                                              // beyond 20 in-flight charges, requests queue in the agent.
```

**Tradeoff point.** A property that affects more than one attribute and is a sensitivity point for more than one attribute, typically improving one while degrading another. Every tradeoff point is a sensitivity point; not every sensitivity point is a tradeoff point.

```python
# app/wallet/balance_cache.py:8
BALANCE_TTL_S = 30        # TP: higher TTL -> lower read latency and DB load (performance)
                          #     but longer window of stale balances (consistency, user trust)
```
```kotlin
// shared/src/commonMain/kotlin/crypto/Vault.kt:21
// Argon2id(...) is an illustrative wrapper, not a specific KMP library (no standard Kotlin Argon2 API exists).
val kdf = Argon2id(memoryKiB = 65_536, iterations = 3)  // TP: higher memory/iterations raise offline-attack cost (security)
                                                       //     but lengthen unlock and risk OOM kills on low-RAM devices
                                                       //     (performance, availability). Measure on the slowest supported device.
```

**Risk vs non-risk.** Both are judgements about *decisions*, not defects. A **risk** is an architecturally important decision (or a missing one) that is potentially problematic given the QA requirements. A **non-risk** is a decision judged sound, always recorded *with the assumption it relies on*: if the assumption breaks, it becomes a risk.

```go
// internal/orders/handler.go:57 : RISK: the charge call runs inside the DB transaction.
// A slow provider holds row locks and a pooled connection for its full latency (availability, performance).
tx, _ := db.BeginTx(ctx, nil)
res, err := payments.Charge(ctx, req) // network call inside tx
```
A non-risk is written the same way, with its assumption next to the evidence:

```conf
# infra/redis.conf:12 : NON-RISK: queued jobs survive a Redis restart (availability of the job queue)
# ASSUMING losing up to ~1 s of acknowledged enqueues on a Redis crash is acceptable (fsync once per second).
appendonly yes
appendfsync everysec
```

A null-pointer bug is not a risk in this sense; "no idempotency key on charge retries" is, because it is a decision (or absence of a tactic) that threatens a scenario.

**Risk theme.** A cluster of risks with a common underlying cause, stated as a systemic weakness and tied to the business goal it threatens. Example: "The payment provider is treated as a local, reliable call -> threatens goal 'never lose or double-charge a paying order'". The evidence is several call sites that share one root cause:

```ts
// src/payments/stripe-adapter.ts:31  R2: no timeout or breaker on the outbound call
const res = await this.http.post(CHARGE_URL, body);  // agent set, no timeout, no breaker
// src/payments/stripe-adapter.ts:33  R3: retried on 5xx without an Idempotency-Key header -> double charge possible
// src/orders/service.ts:88           R1: charge awaited inside db.transaction(...) -> DB connection held for provider latency
await db.transaction(async (tx) => { await tx.orders.insert(o); await payments.charge(o); });
```
Three risks, one theme (RT1): fix the assumption, not the three lines one by one.

---

## 3. Mini-ATAM in one sitting (default for repo and design reviews)

A compressed, single-evaluator version of ATAM steps 2-6 plus 9. It is not an ATAM (no stakeholder phase); say so in the report.

1. **Drivers.** Read README, ADRs, docs/, issue labels, SLOs, runbooks, and ask the user. Write 3-6 business goals and constraints. Anything you infer rather than read is marked **ASSUMED**.
2. **Approaches.** From recovery (package graph, entry points, config, infra files), list the approaches in use with file evidence. Absence counts: "no timeout on outbound HTTP" is an approach finding.
3. **Utility tree.** 5-8 leaves, each a scenario with a response measure, rated (importance, difficulty) as H/M/L. Importance comes from the user or docs; if you rated it, mark ASSUMED. Difficulty is yours.
4. **Trace the top 3** (H,H) then (H,M)/(M,H) scenarios through the code: entry point -> modules -> external calls -> storage. At each hop note which approach acts, which parameter the response depends on, and what else that parameter affects.
5. **Fill the analysis table** (section 4), one row per (scenario, decision). Every risk row needs `file:line` evidence.
6. **Cluster themes.** Group risks by root cause (2-4 themes). Link each theme to a driver from step 1. A theme with no driver is either a missing driver (ask) or not important.
7. **Findings.** For each theme: fixes (tactic or pattern, with a sketch), and a fitness function that would stop regression ([fitness-functions.md](fitness-functions.md)).

**Reading the ratings** (importance to success, difficulty/architectural impact):

| Rating | Action |
|---|---|
| (H,H) | Analyse first; this is where the architecture must prove itself |
| (H,M), (M,H) | Next, as time allows |
| (H,L) | Important but easy; a one-line check that the obvious approach is there |
| (L,H) | Hard and unimportant; question whether its cost is justified |
| (L,L) | Drop from analysis |

A quality attribute with no leaves is a gap; a leaf without a response measure is not yet a scenario. If the user's priorities contradict your tree, that disagreement is itself a finding.

**Tracing checklist** (step 4), per hop:
- [ ] Entry point found (route, handler, CLI command, screen/ViewModel, job consumer).
- [ ] Synchronous vs asynchronous boundary identified; what blocks what.
- [ ] Every outbound call: timeout, retry policy, idempotency, fallback. Note absences.
- [ ] Shared resources on the path: pools, locks, caches, queues, their sizes and config keys.
- [ ] Where state lives and who may mutate it; transaction scope.
- [ ] Module boundaries crossed and whether dependencies point the intended way.
- [ ] Config or environment that changes behaviour (feature flags, per-env pool sizes).

**Timebox.** For a medium repo (tens of thousands of lines) spend roughly: drivers 10%, approaches/recovery 20%, utility tree 10%, tracing 40%, table and themes 20%. If time runs short, analyse fewer scenarios deeply rather than all scenarios shallowly, and list the unanalysed leaves as open.

### 3.1 Worked example: TypeScript checkout API + worker

System (illustrative; a different fictional system from the `orders-api` example in [review-playbook.md](review-playbook.md) §7, and IDs are local to each example): `api/` (Fastify HTTP), `worker/` (BullMQ jobs on Redis), Postgres, a payment provider behind `src/payments/`. Drivers read from README and user: G1 "never lose or double-charge a paying order"; G2 "checkout feels instant at Black Friday load"; G3 "add a second payment provider next quarter". Constraint: two-person team, single region.

Recovered approaches: AP1 Layers (`routes -> services -> repos`); AP2 `PaymentPort` interface with one adapter; AP3 async fulfilment via queue; AP4 Redis read cache for product prices; AP5 no timeouts or breaker on the provider client (absence).

```
Utility
├── Performance
│   └── Checkout latency
│       └── P1 (H,H) 200 checkout req/s at peak, normal ops -> p99 < 800 ms
├── Availability
│   ├── Provider outage
│   │   └── A1 (H,H) payment provider returns 5xx for 5 min -> orders accepted, charged later, zero double charges
│   └── Worker crash
│       └── A2 (H,M) worker killed mid-job -> job resumes, fulfilment within 10 min
├── Modifiability
│   └── New provider
│       └── M1 (H,M) add Adyen alongside Stripe -> <= 5 person-days, only src/payments/ changes   [importance ASSUMED]
├── Security
│   └── Tampered price
│       └── S1 (M,H) client posts altered price -> server recomputes, 0 accepted
└── Testability
    └── T1 (M,L) run checkout service tests without network -> < 30 s, no provider sandbox
```

Analysed P1 and A1 (the (H,H) leaves); M1 next if time allows. Two rows of the analysis table:

| ID | Attr | Env | Stimulus | Response | Approaches | Sensitivity | Tradeoff | Risk | Non-risk | Reasoning | Evidence |
|---|---|---|---|---|---|---|---|---|---|---|---|
| P1 | Perf | Peak | 200 checkout/s | p99 < 800 ms | AP1, AP2, AP4 | SP1 pool size 20 for provider calls; SP2 price-cache TTL | TP1 price TTL 300 s: latency vs price consistency | R1 charge call inside DB tx: pool connections held for provider latency, pool (10) saturates | N1 AP4 cache fine *assuming* prices change < hourly | Provider p99 ~600 ms (ASSUMED, from provider status page / APM; confirm) held inside the DB tx: 10 connections cap throughput near 16 tx/s; confirm with a load test | `src/orders/service.ts:88`, `src/db.ts:5`, `src/payments/stripe-adapter.ts:14` |
| A1 | Avail | Provider 5xx 5 min | Charge fails | Accept, charge later, no double charge | AP2, AP3, AP5 | SP3 retry count/backoff | TP2 retries: availability vs provider load and latency | R2 no timeout or breaker (AP5); R3 retries without idempotency key: double-charge possible | N2 AP3 queue durable *assuming* Redis AOF on | Synchronous charge in request path; checkout fails for the outage; retries re-POST charge | `src/payments/stripe-adapter.ts:31`, `api/routes/checkout.ts:40`, `infra/redis.conf` (ASSUMED) |

**Risk theme RT1 (R1, R2, R3): "The payment provider is treated as a local, reliable call."** No timeout, no breaker, no idempotency, and it runs inside a DB transaction. Threatens G1 and G2. Fixes: move the charge out of the transaction (outbox row, worker charges), add timeout + circuit breaker in the adapter, send an idempotency key derived from the order ID. Fitness functions (an import rule alone cannot see transaction scopes, so split the check):
- **Structural:** after the outbox change only the worker's charge processor may call the provider. dependency-cruiser rule in the `forbidden` array: `{ name: "payments-only-from-charge-worker", severity: "error", from: { path: "^(api|src/orders)/" }, to: { path: "^src/payments/" } }` (or the equivalent eslint-plugin-boundaries element rule).
- **Transaction scope, runtime guard (preferred):** open transactions only through a `withTransaction()` helper that sets an `AsyncLocalStorage` flag; `PaymentPort` adapters throw in test/dev when the flag is set. Any integration test that charges inside a transaction then fails, including indirect calls.
- **Transaction scope, lint (cheaper, weaker):** an ESLint `no-restricted-syntax` selector such as `CallExpression[callee.property.name='transaction'] CallExpression[callee.property.name='charge']`. It catches only the lexical case, and the exact selector depends on the DB client's API.
- Keep: a contract test asserting every adapter call sends an idempotency key; a nightly P1 load test.

---

## 4. Analysis table template

One row per (scenario, decision). Keep IDs stable (`P1`, `AP2`, `SP2`, `TP1`, `R3`, `N2`, `RT1`) so themes and fixes can reference them.

| Scenario ID | Attribute | Environment | Stimulus | Response (measure) | Approaches / decisions | Sensitivity | Tradeoff | Risk | Non-risk (+ assumption) | Reasoning | Evidence file:line |
|---|---|---|---|---|---|---|---|---|---|---|---|
| | | | | | | | | | | | |

Probe questions per row: Which approaches realise this response? Which parameter is it most sensitive to? What else does that parameter affect? Which assumption makes this safe? What is missing or dubious? Where in the code is it?

---

## 5. Full ATAM (only when requested)

Participants: **evaluation team** (usually 3-5 people external to the project; SAiP names roles such as team leader, evaluation leader, scenario scribe, proceedings scribe, questioner; one person may hold several), **project decision makers** (project manager, architect, customer), **architecture stakeholders** (developers, testers, integrators, operators, users, and others who articulate QA needs).

| Phase | Who | Content |
|---|---|---|
| 0 Partnership and preparation | Team leaders + key decision makers | Scope, logistics, stakeholder list, docs handed over |
| 1 Evaluation | Team + decision makers | Steps 1-6 |
| 2 Evaluation (continued) | Team + decision makers + stakeholders | Recap, then steps 7-9 |
| 3 Follow-up | Team + evaluation client | Final written report, team self-assessment |

Phases 1 and 2 are usually separated by a gap of weeks for follow-up questions; check SAiP ch. 21 before quoting durations.

| # | Step (paraphrased) | Claude's equivalent |
|---|---|---|
| 1 | Present the ATAM: method, outputs, expectations | State that you will run ATAM steps, what you will produce, and which steps need humans |
| 2 | Present business drivers: goals, functions, constraints, stakeholders | Elicit from the user and docs; write drivers; mark ASSUMED items |
| 3 | Present the architecture: views, how it meets drivers | Recover views from code ([architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md)) or read the architecture description; summarise in one page |
| 4 | Identify architectural approaches (not yet analysed) | List patterns and tactics with file evidence, including absences |
| 5 | Generate the QA utility tree; decision makers prioritise | Draft the tree; ask the user to confirm importance ratings |
| 6 | Analyse approaches against the top scenarios | Trace scenarios through code; fill the section 4 table |
| 7 | Stakeholders brainstorm and vote on scenarios (SAiP suggests votes per person of about 30% of the scenario count; verify rounding in the book) | Ask the user to name scenarios from other roles (ops, security, product) or propose them from issues and incident history; flag where they disagree with the tree |
| 8 | Analyse approaches again for new high-priority scenarios | Repeat step 6 for the new leaves |
| 9 | Present results | Write the report (section 9, [../templates/architecture-review-report.md](../templates/architecture-review-report.md)) |

Outputs: a concise architecture presentation; articulated business goals; prioritised QA scenarios in a utility tree; risks and non-risks; sensitivity and tradeoff points; risk themes linked to business goals; a mapping of approaches to the qualities they help or hurt. Intangibles: stakeholder communication, clarified requirements, better documentation.

---

## 6. Lightweight Architecture Evaluation (SAiP ch. 21)

A cut-down ATAM for organisations that evaluate regularly and whose participants already know the method and the business context.

- Run by internal peers, not an external team; no Phase 0 partnership or Phase 3 formal report.
- Follows the ATAM step order but compresses steps whose content everyone knows (method, business drivers) to a recap; fits in hours to about a day. SAiP gives a step table with time allocations; take figures only from the book.
- Step 1 (present the ATAM) is skipped because participants know the method. Steps 7 and 8 are normally omitted or cut short: the internal stakeholders contribute their scenarios directly to the utility tree in step 5, so do not run a stakeholder brainstorm. Most of the time goes to steps 5-6. Total is on the order of a working day; take exact per-step hours from the SAiP table.
- Still yields a utility tree, risks, non-risks, sensitivity and tradeoff points, with less depth and less stakeholder coverage.
- Use when: a team with a known architecture wants a check every release or before a big epic. Prefer full ATAM for high-stakes, cross-organisation, or acquisition decisions. The mini-ATAM in section 3 is lighter still (single evaluator, no stakeholder session).

---

## 7. CBAM (Cost Benefit Analysis Method)

ATAM tells you where the risks and tradeoffs are; CBAM tells you which architectural strategies are worth their cost. Use it when several remediation options compete for a limited budget. It builds on ATAM outputs and models each scenario's utility as a function of its response (a utility-response curve). Described in SAiP 3rd ed. ch. 23; the 4th ed. does not devote a chapter to it.

Steps (condensed from the book's nine):
1. **Collate and refine scenarios.** Keep about the top third by business priority; give each four response levels: worst, current, desired, best.
2. **Prioritise.** Stakeholders split 100 votes; keep the top half (a separate, later cut). Weight: top scenario = 1.0, others = votes / top votes.
3. **Assign utility** (0-100) to each response level: this is the utility-response curve.
4. **Propose strategies** and estimate each one's expected response on every scenario it touches (effects may be negative). Derive each estimate from the scenario trace (which hop is the bottleneck, and does the strategy touch it?), not from the strategy's reputation.
5. **Expected utility** by interpolating on the curve; benefit b_ij = expected utility - current utility.
6. **Total benefit** B_i = sum_j (b_ij x W_j). **Value for cost** VFC_i = B_i / C_i (the benefit-to-cost ratio, i.e. the ROI used to rank strategies). Pick by VFC within budget.
7. **Sanity check** against business goals and intuition; if they disagree, revisit estimates or missing scenarios.

### 7.1 Worked example: choosing remediations for the checkout system

Votes (100 split across six scenarios from section 3.1): P1 45, A1 27, M1 18, S1 6, T1 4, A2 0. The top half survives:

| Scenario | Votes | W | Worst | Current | Desired | Best |
|---|---|---|---|---|---|---|
| P1 checkout p99 | 45 | 1.0 | 2000 ms -> 0 | 1200 ms (ASSUMED; measure) -> 30 | 600 ms -> 80 | 300 ms -> 100 |
| A1 orders accepted in provider outage | 27 | 0.6 | 0% -> 0 | 0% -> 0 | 95% -> 85 | 99.9% -> 100 |
| M1 days to add a provider | 18 | 0.4 | 30 d -> 0 | 20 d -> 40 | 5 d -> 90 | 2 d -> 100 |

Strategies (costs in person-weeks):

| Strategy | Expected responses | Expected utility | b_ij | B_i | C_i | VFC |
|---|---|---|---|---|---|---|
| S-A Extend caching to catalogue and stock-availability reads (prices already cached by AP4) | P1: 1100 ms (reads are not the traced bottleneck; R1 still caps throughput) | P1: 30 + 100 x (50/600) = 38.3 | P1 +8.3 | 8.3 | 3 | 2.8 |
| S-B Split payments module behind `PaymentPort` with per-provider adapters | M1: 6 d | M1: 40 + 14 x (50/15) = 86.7 | M1 +46.7 | 46.7 x 0.4 = 18.7 | 4 | 4.7 |
| S-C Move the charge out of the DB tx: outbox row, worker charges async, circuit breaker | A1: 95%; P1: 500 ms (the ~600 ms provider call leaves the request path; R1 removed) | A1: 85; P1: 80 + 100 x (20/300) = 86.7 | A1 +85; P1 +56.7 | 85 x 0.6 + 56.7 = 107.7 | 5 | 21.5 |

Ranking by VFC: S-C, S-B, S-A. With 9 person-weeks: S-C (5) + S-B (4); S-A waits. Sanity check: the intuitive cheap fix (a cache) does not touch the traced bottleneck, so it ranks last; the fix for R1 wins on both latency and money (G1, G2). Always derive expected responses from the scenario trace, not from the strategy's reputation. Also note hidden effects the scenarios do not capture: S-C makes "order accepted" and "order charged" separate states (add a scenario for charge-failure handling), and S-C is cheaper once S-B exists, so estimate combinations when strategies interact.

> CBAM notes adapted in part from the Wikipendium TDT4240 compendium (CC BY-SA 3.0), corrected against SAiP 3rd ed.

---

## 8. Other techniques and when to pick which

| Technique | Answers | Pick when | Cost | Output |
|---|---|---|---|---|
| Review checklist ([review-playbook.md](review-playbook.md)) | Are common structural problems present? | Every PR or repo scan; no stated goals yet | Low | Findings list |
| Tactics-based questionnaire (SAiP 4e gives one per QA chapter; use the tactic trees in [quality-attributes-runtime.md](quality-attributes-runtime.md) and [quality-attributes-change.md](quality-attributes-change.md) as questions: "Is there a timeout on each outbound call? Where?") | Which tactics for attribute X are present or missing? | One attribute dominates (e.g. availability after incidents) | Low-medium | Present/absent table with evidence |
| Scenario walkthrough of a PR | Does this change move a sensitivity point or break a scenario? | PR touches boundaries, shared infra, config of pools/TTLs/retries | Low | PR comments tied to scenario IDs |
| Mini-ATAM (section 3) | Can this architecture meet its top scenarios? | Repo review, design review, inherited system | Medium | Table, themes, fixes |
| Fitness functions ([fitness-functions.md](fitness-functions.md)) | Is a known constraint still holding? | After a finding is fixed; to keep an ADR true | Low per run | CI pass/fail |
| Spike or prototype | Does option X actually behave as claimed? | Reasoning is inconclusive; a tradeoff point's shape is unknown | Medium | Measured numbers, a decision in an ADR |
| Load/soak/chaos test | What is the actual response measure? | Performance or availability scenario is (H,H) | Medium-high | Measured p99, error rates, recovery time |
| CBAM (section 7) | Which fix is worth it? | Several options, limited budget | Medium | Ranked strategies |
| Full ATAM or LAE (sections 5-6) | Organisation-level assurance | Explicitly requested; many stakeholders | High | Formal report |

**PR walkthrough triggers.** Run a scenario walkthrough when the diff matches `timeout|retry|backoff|ttl|maxSockets|poolSize|max_connections|concurrency|prefetch|@Transactional|BEGIN|transaction(`, changes files under `infra/`, `helm/`, `*.tf`, `docker-compose*`, or changes a module's public interface. In the PR comment, name the scenario ID each hit affects and say whether it moves a known sensitivity point.

Combine: reasoning finds the risk, measurement confirms it, fitness functions keep it fixed.

---

## 9. Presenting results

Use [../templates/architecture-review-report.md](../templates/architecture-review-report.md) (repository variant) and its section order; do not invent another. Within it:
- [ ] State the method and its deviations in "Scope and method" ("mini-ATAM, single evaluator, no stakeholder phase"); mark ASSUMED drivers.
- [ ] Put the top risk themes in the executive summary ("Top 3 risks"), each linked to the business goal it threatens and its risk IDs.
- [ ] Group the Findings section under their risk theme; rank findings with the severity / confidence / cheapest-fix rubric in [review-playbook.md](review-playbook.md) §6. Each finding has `file:line` evidence, the scenario ID, a concrete fix (tactic/pattern plus sketch) and a fitness function.

Add what the template does not prompt for:
- Sensitivity and tradeoff points worth an ADR (list under "Suggested ADRs"; the team should decide consciously).
- Non-risks with their assumptions (they tell the team what to monitor).
- Unanalysed utility-tree leaves (under "Assumptions and open questions").

Common mistakes to avoid:
- **Scenarios without measures** ("the system should be fast"). Not a scenario; no evaluation is possible.
- **Confusing sensitivity with tradeoff.** A tradeoff point must be a sensitivity point for two or more attributes. "Weak design" is neither.
- **Sensitivity points without attribute and response** named.
- **Risks without evidence or driver link.** No `file:line` means unverified; no driver means unprioritised.
- **Non-risks without the assumption** they depend on.
- **Reporting bugs as architectural risks**, or grading the architecture instead of listing risks.
- **Presenting reasoning as measurement.** Say "likely saturates" and recommend the load test; do not claim numbers you did not measure.
