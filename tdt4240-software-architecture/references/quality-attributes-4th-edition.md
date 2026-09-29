# Quality attributes in SAiP 4th edition: new chapters, renames and changes

> **Caution: read this first.** This file covers what changed between Bass, Clements & Kazman,
> *Software Architecture in Practice*, 3rd ed. (2013) and 4th ed. (2021). The **chapter titles and
> numbers are confirmed** from the publisher's (InformIT) table of contents. The **tactic names,
> tactic groupings and general-scenario wording were NOT fully checked against the printed 4th
> edition**. They are reconstructed from secondary knowledge. Every item below carries one of two tags:
>
> - **[confirmed chapter title]**: the chapter exists with this title and number.
> - **[verify tactic wording in the book]**: plausible and widely cited, but check the exact wording
>   in the book or the lecture slides before relying on it in an exam answer.
>
> When tutoring, pass the flag on to the student. Never state an unverified tactic name as
> certain. The 3rd-edition tactic lists (the baseline the Wikipendium compendium and older exams use) are in
> [quality-attributes-classic.md](quality-attributes-classic.md). This file does not repeat them.

## 1. Where the QA chapters are now [confirmed chapter title]

| 4th ed. ch. | Title | 3rd ed. equivalent | Status |
|---|---|---|---|
| 3 | Understanding Quality Attributes | ch. 4 | Renumbered |
| 4 | Availability | ch. 5 | Renumbered; tactic renames (see §7) |
| 5 | Deployability | (mentioned briefly in ch. 12, Other QAs) | **New chapter** |
| 6 | Energy Efficiency | none | **New chapter** |
| 7 | Integrability | ch. 6 Interoperability | **Replaces and broadens** interoperability |
| 8 | Modifiability | ch. 7 | Renumbered; tactic changes |
| 9 | Performance | ch. 8 | Renumbered; tactic renames |
| 10 | Safety | (mentioned briefly in ch. 12, Other QAs) | **New chapter** |
| 11 | Security | ch. 9 | Renumbered; tactic changes |
| 12 | Testability | ch. 10 | Renumbered |
| 13 | Usability | ch. 11 | Renumbered |
| 14 | Working with Other Quality Attributes | ch. 12 Other QAs | Renumbered/retitled |

Other 4th-ed. chapters that affect QA reasoning [confirmed chapter title]: 15 Software Interfaces,
16 Virtualization, 17 The Cloud and Distributed Computing, 18 Mobile Systems, 23 Managing
Architecture Debt. See [platforms-and-emerging-topics.md](platforms-and-emerging-topics.md).

**No ML or edge chapter.** The 4th edition has **no chapter on machine learning** and **no chapter on
edge or edge-dominant systems**. "Architectures for the Edge" (the Metropolis model) was 3rd-ed. ch. 27
and is gone. If a student cites a "4th-ed. ML chapter" or a "4th-ed. edge chapter", correct them.

The six-part scenario structure (source, stimulus, artifact, environment, response, response
measure) is the same in both editions. Only the chapter number changes (3rd ch. 4 → 4th ch. 3).

## 2. Deployability (4th ed. ch. 5) [confirmed chapter title]

**Definition.** Deployability is about how easily software reaches its execution environment and
starts running: how predictably, quickly and cheaply a new version (or a fix) can be moved into
production, including the ability to **roll it back**. It ties the architecture to continuous
integration and continuous deployment, DevOps practice and the deployment pipeline. In 3rd ed. it
had only a short paragraph in ch. 12, which asked how an executable arrives at its host and is
invoked (adapted from the Wikipendium TDT4240 compendium, CC BY-SA 3.0).

**General scenario** [verify tactic wording in the book]

| Part | Typical values |
|---|---|
| Source | End user, developer, system administrator, operations/DevOps engineer, component marketplace |
| Stimulus | A new element is available to deploy: bug fix, security patch, new feature, upgraded component or platform |
| Artifact | Specific components or modules, the platform, the UI, the environment, or a system it interoperates with |
| Environment | Full deployment; subset deployment to a specified portion of users, VMs, containers or servers |
| Response | Incorporate the new components; deploy them; monitor the new version; roll back if it misbehaves |
| Response measure | Cost (effort, calendar time, money), extent of the deployment, frequency, number of failed deployments, time to roll back, impact on other QAs |

**Tactics** [verify tactic wording in the book]

| Category | Tactics | What they do |
|---|---|---|
| Manage deployment pipeline | Scale rollouts | Release to a fraction of users first (canary releases, gradual rollout), then widen |
| | Roll back | Keep the ability to return to the previous known-good version automatically |
| | Script deployment commands | Automate every deployment step, so deployment is repeatable and needs no manual steps |
| Manage deployed system | Manage service interactions | Let several versions coexist and route requests to the right version |
| | Package dependencies | Ship a component together with its dependencies (containers, bundles) |
| | Feature toggle | Deploy code switched off ("dark") and turn it on at runtime with a flag, which also works as a kill switch |

Patterns the 4th ed. associates with deployability (approximate): microservices, and the
blue/green, rolling-upgrade and canary deployment strategies.

**Game-project example.** Source: the development team. Stimulus: a balance patch for the libGDX
game. Artifact: game-rules config and the backend (e.g. Firebase) rules. Environment: live game with active
matches. Response: roll the patch out to 10% of players behind a remote-config flag; roll back if
the crash rate rises. Measure: rollout in under 15 min, rollback in under 5 min, no match lost.

## 3. Energy efficiency (4th ed. ch. 6) [confirmed chapter title]

**Definition.** Energy efficiency is how well a system minimises the energy it consumes while
still delivering its required function and other QAs. It matters most for battery-powered mobile
and embedded devices, and for large data centres. It trades off mainly against performance
and availability (redundancy costs power).

**General scenario** [verify tactic wording in the book]

| Part | Typical values |
|---|---|
| Source | End user, manager, system administrator, automated agent |
| Stimulus | A request to conserve energy (or a constraint such as low battery) |
| Artifact | Specific devices, servers, VMs, clusters |
| Environment | Runtime, connected or disconnected, battery-powered, low-battery mode, fixed power budget |
| Response | Disable services, deallocate runtime resources, change allocation, run in a lower-power mode, reduce quality of service |
| Response measure | Energy consumed or saved (J, kWh, % of baseline, battery life), while latency, throughput or accuracy stay within a stated bound |

**Tactics** [verify tactic wording in the book]

| Category | Example tactics |
|---|---|
| Monitor resources | Metering (measure consumption directly); static classification (use known power figures per device); dynamic classification (estimate from a model at runtime) |
| Allocate resources | Reduce usage (power down or throttle idle devices, lower CPU frequency); discovery (find the most energy-efficient resource); schedule resources (place work where it costs least energy) |
| Reduce resource demand | Manage event arrival, limit event response, prioritise events, reduce computational overhead, bound execution times, increase efficiency of resource usage (largely the same as the performance demand-side tactics) |

**Game-project example.** Frame-rate cap on the menu screen (30 fps instead of 60), batching of
network writes to the backend, and pausing the game loop when the app goes to the background.
Measure: at most N % battery per 30 min session on a mid-range Android phone.

## 4. Integrability (4th ed. ch. 7) [confirmed chapter title]

**Definition.** Integrability is about the cost and risk of making separately developed components
work together as intended. Examples are adding a new component, integrating a new version of one,
or combining existing components in a new way.

**How it broadens 3rd-ed. interoperability.** 3rd-ed. ch. 6 *Interoperability* was the degree to which
two or more **systems** can usefully exchange meaningful information through interfaces
(syntactic plus semantic), with two concerns: *discovery* and *handling of the response*. Its
tactics were only **Locate** (discover service) and **Manage interfaces** (orchestrate, tailor
interface). Adapted from the Wikipendium TDT4240 compendium (CC BY-SA 3.0). Integrability widens this in three ways:

1. **Scope.** It covers components *inside* one system as well as external systems.
2. **Time.** It is judged at design, integration and deployment time as well as at runtime.
3. **Measure.** It asks what integration costs (the "distance" between interfaces, in
   syntax, data semantics, behaviour, timing and resources), not only whether exchange succeeds.

Interoperability becomes one aspect of integrability. In an exam that uses the 3rd ed., still
answer with the 3rd-ed. interoperability tactics.

**General scenario** [verify tactic wording in the book]

| Part | Typical values |
|---|---|
| Source | Mission or system stakeholders, component marketplaces, component vendors |
| Stimulus | Add a new component; integrate a new version of an existing component; integrate existing components in a new way |
| Artifact | The entire system, a specific set of components, component metadata, component configuration |
| Environment | Development, integration, deployment or runtime |
| Response | Changes are completed, integrated, tested and deployed |
| Response measure | Number of components changed, % of code changed, lines of code, effort, money, calendar time, effects on other QA response measures |

**Tactics** [verify tactic wording in the book]

| Category | Tactics | Note |
|---|---|---|
| Limit dependencies | Encapsulate; use an intermediary; restrict communication paths; adhere to standards; abstract common services | Overlaps heavily with modifiability's "reduce coupling" |
| Adapt | Discover (3rd-ed. "discover service"); tailor interface (kept from 3rd ed.); configure behaviour | Change or look up a component so it fits without editing its source |
| Coordinate | Orchestrate (kept from 3rd ed.); manage resources | Control how components interact at runtime |

Patterns the 4th ed. associates with integrability (approximate): wrappers, bridges, mediators,
service-oriented architecture, dynamic discovery, and adapters.

**Game-project example.** Stimulus: replace Firebase with Supabase. Tactics: *use an intermediary*
(a `BackendService` interface in `core`), *abstract common services* (auth plus storage behind one
facade) and *tailor interface* (an adapter per provider). Measure: only the adapter module and the DI
wiring change, and the swap takes 2 person-days or less.

## 5. Safety (4th ed. ch. 10) [confirmed chapter title]

**Definition.** Safety is the system's ability to avoid entering states that can cause damage,
injury or loss of life to actors in its environment, and to recover from such states or limit the
damage when they occur. It is different from security (no malicious actor is assumed) and from
availability (a stopped system may be safe, and a running one unsafe). In 3rd ed. it was a
one-line entry under ch. 12 Other QAs; the Wikipendium example is a missile system that fires at random
(adapted from the Wikipendium TDT4240 compendium, CC BY-SA 3.0).

**General scenario** [verify tactic wording in the book]

| Part | Typical values |
|---|---|
| Source | A data source (sensor, a component that computes values), a time source, a user or operator |
| Stimulus | Omission (value never arrives), commission (wrong value or action), incorrect data, timing error (too early or too late) |
| Artifact | The portions of the system that affect safety-critical behaviour |
| Environment | Normal operation, degraded operation, manual mode, recovery mode |
| Response | Stay in a safe state; return to a safe state; continue in degraded mode; shut down; switch to manual or backup; notify operators; log the event |
| Response measure | Share of unsafe-state entries avoided or recovered, time to reach a safe state, change in risk exposure, time in degraded mode |

**Tactics** [verify tactic wording in the book]

| Category | Example tactics | Idea |
|---|---|---|
| Unsafe-state avoidance | Substitution (use a simpler, safer mechanism, e.g. a hardware interlock instead of software); predictive model | Stop the hazard from arising |
| Unsafe-state detection | Timeout; timestamp; condition monitoring; sanity checking; comparison (of redundant outputs) | Notice that the system is at or near an unsafe state |
| Containment: redundancy | Replication; functional redundancy; analytic redundancy | Make sure one faulty element cannot decide the outcome alone |
| Containment: limit consequences | Abort; graceful degradation | Reduce the harm while the fault persists |
| Containment: barrier | Firewall; interlock | Physically or logically block the hazard from propagating |
| Recovery | Rollback; repair state; reconfiguration | Get back to a known safe state |

Many safety tactics reuse availability tactics (see [quality-attributes-classic.md](quality-attributes-classic.md)).
The difference is the goal: availability keeps the service running, while safety keeps the system *harmless*, even if that means stopping it.

## 6. Working with other QAs (4th ed. ch. 14) [confirmed chapter title]

This is the successor to 3rd-ed. ch. 12. It covers QAs without their own chapter (e.g. variability,
portability, scalability, development distributability, monitorability) and how to define a new
QA with its own general scenario and tactics. Scalability, portability and similar QAs are still
often explained as special cases of modifiability or performance. Mobility moved into ch. 18
Mobile Systems. The exact list of QAs in ch. 14 is not verified here.

## 7. Tactic renames and additions (3rd → 4th ed.)

**All rows are [verify tactic wording in the book].** Use them as a translation aid, not as authoritative
tactic names.

| QA | 3rd ed. (baseline) | 4th ed. | Change |
|---|---|---|---|
| Availability | Active redundancy (hot spare), Passive redundancy (warm spare), Spare (cold spare) | **Redundant spare** | Three tactics merged into one, with hot/warm/cold as variants |
| Availability | Degradation | **Graceful degradation** | Renamed |
| Performance | Manage sampling rate | **Manage work requests** | Renamed and broadened (also admission control and rate limiting) |
| Performance | Reduce overhead | **Reduce computational overhead** | Renamed |
| Performance | Increase resource efficiency | **Increase efficiency of resource usage** | Renamed |
| Security | Detect message delay | **Detect message delivery anomalies** | Renamed |
| Security | (none) | **Validate input** | Added under resist attacks |
| Security | Lock computer | **Restrict login** | Renamed |
| Security | Change default settings | **Change credential settings** | Renamed |
| Security | Maintain audit trail (recover) | Audit and nonrepudiation under recover | Regrouped |
| Modifiability | Split module, increase semantic coherence | Split module and **Redistribute responsibilities** under increase cohesion | Added |
| Modifiability | Refactor (under reduce coupling) | No longer a separate tactic | Removed or folded in |
| Interoperability → Integrability | Locate / Manage interfaces | Limit dependencies / Adapt / Coordinate | Regrouped and extended (see §4) |

Testability and usability tactics are largely unchanged in name.

## 8. Patterns now live inside the QA chapters

3rd ed. had a separate pattern catalogue in ch. 13 (Architectural Tactics and Patterns: layered,
broker, MVC, pipe-and-filter, client-server, P2P, SOA, publish-subscribe, shared-data, map-reduce,
multi-tier). The **4th ed. has no such chapter** [confirmed chapter title: there is none in the ToC]. Patterns are
presented inside the QA chapter they mainly serve. **The assignments below are approximate:
verify them in the book before quoting a chapter number.**

| QA chapter (4th ed.) | Patterns discussed there (approximate) |
|---|---|
| 4 Availability | Active/passive redundancy, TMR, circuit breaker, process pairs, forward error recovery |
| 5 Deployability | Microservices; blue/green, rolling upgrade, canary |
| 7 Integrability | Adapter/wrapper, bridge, mediator, SOA, dynamic discovery |
| 8 Modifiability | Client-server, plug-in (microkernel), layers, publish-subscribe |
| 9 Performance | Service mesh, load balancer, throttling, map-reduce |
| 10 Safety | Redundant sensors, monitor-actuator, separated safety |
| 11 Security | Intercepting validator, intrusion prevention system |
| 12 Testability | Dependency injection, strategy, intercepting filter |
| 13 Usability | MVC, observer, memento |

Consequence for exam answers: the context/problem/solution triple and the "patterns are built
from tactics" idea still apply. See [architectural-patterns.md](architectural-patterns.md) for the
pattern catalogue and [design-and-game-patterns.md](design-and-game-patterns.md) for GoF patterns.

## 9. Guidance for students (and for tutoring them)

1. **Use the edition the lecturer's slides use.** The public exams (2015, 2016) and the Wikipendium
   compendium use the 3rd ed. Whether TDT4240 has moved to the 4th ed. is **not publicly confirmed**, so
   check the reading list (Leganto) or Blackboard.
2. **If unsure, answer with the 3rd-ed. list and add the 4th-ed. name in parentheses**, e.g.
   "Detect faults, Recover (active redundancy, passive redundancy, spare; 4th ed.: *redundant spare*),
   Prevent faults". This is correct under either edition.
3. **Name chapters by title, not number.** "The Modifiability chapter" is safe; "ch. 7" is correct only in the 3rd ed.
4. **Interoperability questions:** give the 3rd-ed. tactics (discover service, orchestrate, tailor
   interface). Mention integrability only as the 4th-ed. generalisation.
5. **New QAs in the game project:** deployability (backend rules, remote config), energy efficiency
   (battery on Android) and safety (rarely relevant to a game; do not force it) may be chosen as
   secondary QAs. Write them as six-part scenarios with a numeric response measure. Scenario
   technique: [requirements-and-design.md](requirements-and-design.md); templates:
   [../templates/requirements-document.md](../templates/requirements-document.md).
6. **Do not list patterns as tactics.** Rollback and feature toggle are tactics; canary deployment
   and circuit breaker are patterns or strategies that realise tactics.
7. **Never cite a 4th-ed. ML chapter or edge chapter.** They do not exist.

Exam drills: [exam-prep.md](exam-prep.md). Foundations and the QA/tactic/pattern vocabulary:
[foundations.md](foundations.md).

---

*Sources: Bass, Clements & Kazman, Software Architecture in Practice, 4th ed., Addison-Wesley, 2021
(chapter list from the InformIT table of contents,
https://www.informit.com/store/software-architecture-in-practice-9780136885887); 3rd ed., 2013.
Sections marked "Adapted from the Wikipendium TDT4240 compendium" draw on
https://www.wikipendium.no/TDT4240_Software_Architecture (CC BY-SA 3.0, Wikipendium contributors),
paraphrased and changed.*
