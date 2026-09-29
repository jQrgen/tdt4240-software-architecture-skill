# Template: architecture decision record with enforcement

Nygard core (Title, Status, Context, Decision, Consequences) with MADR-style extras (considered options,
deciders). The headings are not MADR's. If the repo already uses MADR, keep its headings and map: Options
considered -> Considered Options, Decision -> Decision Outcome, Sensitivity/Enforced by/Revisit when -> More
Information; Status/Date/Deciders go in MADR's YAML front matter (MADR 4 calls deciders `decision-makers`). Store as `docs/adr/NNNN-kebab-title.md`
(e.g. `docs/adr/0007-hide-payment-provider-behind-port.md`). Numbers are sequential and never
reused or renumbered, including for rejected or superseded records.

**Guidance.** One decision per ADR. Once Accepted, the record is immutable: to change course,
write a new ADR and set the old one to `Superseded by ADR-NNNN` (only the Status line may be edited).
Write one when the decision meets an ADR-worthy signal in [documentation.md section 7](../references/documentation.md)
(the single trigger list);
skip choices reversible in a single PR. Keep it to 1-2 pages. Every ADR must name how it is enforced; an
unenforced structural decision decays into folklore (see [fitness functions](../references/fitness-functions.md)).
If the code already violates the decision, enforce with a baseline (ArchUnit freeze store, dependency-cruiser
known violations; see fitness-functions.md section 5) and state the burn-down plan under Consequences.
For decisions already embodied in code, write a Reconstructed ADR whose Evidence cites the `file:line` and the
commit that introduced it; use "accepted by default" when nobody chose deliberately.
Use scenario IDs in the `QS-<letter><n>` form (QS-M1, QS-A1) from the [design workflow](../references/design-workflow.md); sensitivity and
tradeoff points use the ATAM meaning from [evaluation methods](../references/evaluation-methods.md).

```markdown
# ADR-NNNN: <Imperative decision phrase, e.g. "Isolate X behind Y">

- **Status:** Proposed | Accepted | Rejected | Deprecated | Superseded by ADR-NNNN
- **Origin:** Deliberate | Reconstructed from code/git (accepted by default)
- **Date:** YYYY-MM-DD
- **Deciders:** <names or roles>

## Context
- Forces: <what pushes the decision now; business goal, pain, incident>
- Driving QA scenarios: <scenario IDs, e.g. QS-M1, QS-A1>
- Constraints: <fixed tech, regulation, team skills, deadlines>
- Evidence: <path/File.kt:123, dependency-graph output, latency p99 measurement>
- Assumptions: <ASSUMED items and open question IDs, e.g. "200 req/s peak (Q2)"> ; falsified by: <metric, load test or
  product decision that would show the assumption wrong>

## Options considered
| Option | QAs + | QAs - | Cost | Reversibility |
|---|---|---|---|---|
| A: ... | | | | easy / moderate / hard |
| B: ... | | | | |

## Decision
We will <decision, stated so a reviewer can check code against it>.

## Consequences
- Benefits: ...
- Costs accepted: ...
- Non-goals: <what this ADR deliberately does not decide>
- Risks: <what could go wrong, and the mitigation>

## Sensitivity and tradeoff points
- Sensitivity: <parameter/element whose change strongly moves one QA>
- Tradeoff: <element that improves one QA while degrading another>

## Enforced by
<test path | lint rule | CI job>  -- or --
Not enforceable: review checklist item <X>, because <reason no tool can check it>.

## Revisit when
- <trigger: load > N rps, second provider added, team split, library EOL>

## Links
- Supersedes / related: ADR-NNNN; issue/PR: <url>; scenario: <QS-ID>
```

## Filled example

```markdown
# ADR-0007: Hide the payment provider behind a port in the domain module

- **Status:** Accepted
- **Origin:** Deliberate
- **Date:** 2026-03-14
- **Deciders:** Tech lead (checkout), payments owner, platform architect

## Context
- Forces: provider SDK types leak into order logic; switching or adding a provider touches ~40 files.
- Driving QA scenarios: QS-M1 (add a second provider in <= 5 dev-days, no domain changes),
  QS-T1 (domain tests run without network or provider sandbox).
- Constraints: must keep the current provider through Q3; PCI scope must not grow.
- Evidence: `checkout/domain/OrderService.kt:88` imports `com.stripe.model.PaymentIntent`;
  jdeps shows 11 domain classes depending on `com.stripe`.
- Assumptions: ASSUMED a second provider is needed within 12 months (Q1); falsified by: the product roadmap drops
  multi-provider support, in which case option B's cost buys only QS-T1.

## Options considered
| Option | QAs + | QAs - | Cost | Reversibility |
|---|---|---|---|---|
| A: Keep calling SDK directly | none; only saves the up-front work | modifiability, testability | none now | hard (spreads further) |
| B: Port in domain, adapter in infra | modifiability, testability | mapping code; provider-specific features need port extensions | ~6 dev-days | moderate |
| C: External payment orchestration service | modifiability across teams | availability (new hop), ops cost | high | hard |

## Decision
We will define `PaymentGateway` (port) and provider-neutral value types in `checkout.domain`,
implement `StripePaymentGateway` in `checkout.infra.payments`, and wire it at the composition root.

## Consequences
- Benefits: provider swap confined to one adapter; domain tests use a fake gateway.
- Costs accepted: DTO mapping code; provider-specific features need an explicit port extension.
- Non-goals: multi-provider routing and retries policy (future ADR).
- Migration: 11 existing violations baselined in the ArchUnit freeze store; burned down in follow-up PRs;
  delete the store entries when empty.
- Risks: port shaped around one provider's model; mitigated by reviewing it against a second provider's API.

## Sensitivity and tradeoff points
- Sensitivity: the number of provider-specific operations exposed on the port is critical to QS-M1 (effort
  to add a provider); every extra operation adds adapter work per provider.
- Tradeoff: the port improves modifiability/testability but hides provider-specific features (e.g. 3DS flows,
  idempotency keys) behind a lowest-common-denominator API; exposing them needs port extensions.

## Enforced by
`checkout/src/test/kotlin/arch/PaymentBoundaryTest.kt` (ArchUnit, runs in CI job `test`), frozen because
11 violations exist; the freeze store is committed and only new violations fail:
    FreezingArchRule.freeze(
        noClasses().that().resideInAPackage("..checkout.domain..")
            .should().dependOnClassesThat().resideInAPackage("com.stripe.."))
(TypeScript equivalent: an entry in the `forbidden` array of `.dependency-cruiser.js`/`.cjs` (name depends
 on version), with known violations recorded and CI run with `--ignore-known`:
 `{ name: "domain-no-payment-sdk", severity: "error", from: { path: "^src/domain/" }, to: { path: "^node_modules/(stripe|@stripe)/" } }`)

## Revisit when
- A second provider is contracted, or a third provider-specific feature needs a port extension.

## Links
- Scenarios QS-M1, QS-T1; related: ADR-0003 (hexagonal layout for checkout)
```
