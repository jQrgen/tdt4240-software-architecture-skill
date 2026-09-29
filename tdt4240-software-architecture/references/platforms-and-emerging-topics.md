# Platforms and emerging topics

Covers cloud, virtualization, distributed computing, mobile, edge-dominant systems, software interfaces, quantum computing and the architect's role, plus a note on machine learning.

**Chapter numbers.** *Software Architecture in Practice* (SAiP) by Bass, Clements & Kazman: the 4th ed. (2021) is current, and the 3rd ed. (2013) is the one used by Wikipendium and by the public 2015/2016 exams. Which edition TDT4240 uses today is **unconfirmed** (see `course-and-project-guide.md`). Give both numbers when you answer.

| Topic | 3rd ed. | 4th ed. | Exam signal |
|---|---|---|---|
| Cloud | ch. 26 "Architecture in the Cloud" | ch. 17 "The Cloud and Distributed Computing" | 2015 exam: cloud deployment models and basic mechanisms |
| Virtualization | (part of ch. 26) | ch. 16 "Virtualization" | none public |
| Software interfaces | (no chapter) | ch. 15 "Software Interfaces" | none public |
| Mobile systems | (touched on under "other QAs": mobility) | ch. 18 "Mobile Systems" | none public; relevant to the Android project |
| Edge-dominant systems | ch. 27 "Architectures for the Edge" | **not in 4th ed.** | 2015 exam: a question on edge-dominant systems |
| Quantum computing | (none) | ch. 26 "A Glimpse of the Future: Quantum Computing" | low |
| Role of architects in projects | parts of ch. 3, 22 | ch. 24 | low to medium |
| Machine learning | none | **none** | not a confirmed syllabus topic |

Related files: QA tactics for availability and performance are in `quality-attributes-classic.md` and `quality-attributes-4th-edition.md`. Allocation patterns (Map-Reduce, Multi-tier) and Client-Server/SOA are in `architectural-patterns.md`. Deployment and allocation views are in `documentation.md`.

---

## 1. Cloud computing (3rd ed. ch. 26; 4th ed. ch. 17)

*Parts adapted from the Wikipendium TDT4240 compendium (CC BY-SA 3.0), with corrections and additions from the textbook.*

### 1.1 What the cloud is (NIST framing, used by SAiP)

**Five essential characteristics.** Learn all five. They are an easy short-answer question.

| Characteristic | Meaning |
|---|---|
| On-demand self-service | A consumer provisions compute or storage alone, without a human at the provider |
| Broad network access | Capabilities are reached over the network with standard mechanisms from varied clients (phones, laptops, thin clients) |
| Resource pooling | Provider resources serve many consumers (multi-tenant) and are assigned dynamically; the consumer usually does not know or control the exact location |
| Rapid elasticity | Capacity scales out and in quickly, sometimes automatically, and appears unlimited to the consumer |
| Measured service | Usage is metered, so consumers pay per use and resource use can be monitored and reported |

**Service models.** What the consumer manages shrinks as you go from IaaS to SaaS.

| Model | Consumer gets | Consumer manages | Examples (illustrative) |
|---|---|---|---|
| IaaS | Virtual machines, storage and networks | OS, middleware, runtime, app, data | Raw VMs from a cloud provider |
| PaaS | A platform to develop, deploy and run apps | App and data only | Managed app platforms; for the course, Firebase is BaaS/PaaS-like |
| SaaS | A finished application used over the network | Only user settings | Web email, office suites |

**Deployment models.** The 2015 exam asked for these.

| Model | Who uses the infrastructure | Typical reason |
|---|---|---|
| Public | Open to the general public; owned by a cloud provider | Cost, elasticity, no hardware to own |
| Private | One organisation only; hosted on or off site | Control, security, regulation |
| Community | Several organisations with shared concerns (mission, policy, compliance) | Shared cost with a trusted group |
| Hybrid | Two or more of the above, joined so data and apps can move between them (e.g. "cloud bursting" into public at peak load) | Keep sensitive parts private and scale the rest publicly |

### 1.2 Base mechanisms

- **Hypervisor.** Software that runs VMs on one physical host and gives each one virtualised CPU, memory and I/O. It isolates the VMs and schedules them onto the shared hardware.
- **Virtual machine (VM).** A software computer with its own guest OS. It can be created, stopped, snapshotted or moved (migrated) between hosts.
- **Virtual file system / storage.** Storage is presented to VMs as disks or files but lives on pooled, often replicated storage in the data centre.
- **Virtual network.** Each VM gets network addresses and connectivity that are independent of the physical network. The provider maps virtual addresses to physical hosts.
- **Data centre scale.** There are huge numbers of commodity machines, so failure of individual parts is **normal**, not exceptional.

### 1.3 Multi-tenancy

One running instance of an application (or a shared infrastructure) serves several **tenants** (customer organisations), and each tenant sees only its own data and configuration.
- Benefits: lower cost per tenant, one version to upgrade, and better resource utilisation.
- Architectural concerns:
  - **Security**: isolate the tenants' data.
  - **Performance**: the "noisy neighbour" problem.
  - **Modifiability**: per-tenant customisation without forking the code.
  - **Availability**: one fault can hit every tenant.

### 1.4 Quality-attribute implications (corrected)

Wikipendium files auto-scaling under *Availability* and "failure is common" under *Performance*. **Those two headings are swapped.** Use this table:

| QA | Cloud-specific point | Tactics / mechanisms |
|---|---|---|
| **Performance / scalability** | Load varies. **Auto-scaling** adds or removes instances to follow demand (horizontal scaling = elasticity). Shared resources and the network add latency | Load balancer, maintain multiple copies of computations (server pool), caching, scale out |
| **Availability** | Failure is **frequent** at data-centre scale. The platform itself must be available, and applications must **detect** failures and **recover** from them | Heartbeat or health checks, redundancy across hosts or zones, retry, timeouts, state kept outside the instances so a failed one can be replaced |
| **Security** | Multi-tenancy and hosting on third-party infrastructure increase risk (shared hardware, provider access, data location or jurisdiction) | Separate entities (VM isolation), encrypt data, authenticate and authorize actors, audit trail |
| **Cost / measured service** | Pay per use, so architecture choices become running costs | Scale in when idle, choose the right service model |

### 1.5 Example technologies (3rd ed.)

- **HDFS** (Hadoop Distributed File System). Stores very large files split into blocks that are replicated across nodes. It is built to tolerate node failure: availability through replication.
- **NoSQL databases.** Key-value, document, column-family and graph stores. They give up some relational guarantees (joins, strict consistency) in exchange for horizontal scalability and a flexible schema. They connect to the CAP / consistency trade-off; see the 4th-ed. state discussion below.
- **MapReduce.** A programming model and framework: *map* over partitioned data in parallel, then *reduce* to combine the results. The framework handles scheduling, distribution and re-running failed tasks. It is the **Map-Reduce allocation pattern** in `architectural-patterns.md`.

### 1.6 4th-edition additions (ch. 17, distributed computing)

Know these concepts. The exact subsection names are **unverified**, so check the book.

| Concept | Gist | QA link |
|---|---|---|
| **Load balancer** | An intermediary that spreads requests over a pool of identical instances and stops sending to unhealthy ones (health checks) | Performance, availability |
| **Autoscaling** | Monitors load (CPU, queue length, request rate) and creates or destroys instances automatically. New instances need time to start, so scaling lags behind demand | Performance, cost |
| **Timeouts** | In a distributed system you cannot tell a slow service from a dead one. A timeout turns "no answer" into a detectable failure that you can retry or fail over | Availability |
| **Long-tail latency** | With many servers, a small fraction of requests are much slower than the median. A request that fans out to many servers is only as fast as its slowest reply. Mitigations: send hedged or duplicate requests and take the first reply, set timeouts, avoid overloaded instances | Performance |
| **State management** | Keep instances **stateless** where possible and put session or state in a shared store (database or cache). Then any instance can serve any request, and instances can be killed or added freely. Stateful services need replication and a consistency choice | Availability, scalability, modifiability |

**Course link.** A libGDX game with a Firebase backend is a client of a PaaS/BaaS. Good exam or report points:
- Network calls to the backend need timeouts and a plan for offline use.
- Keep game state authoritative in one place.
- Put the backend behind an interface so it could be swapped (modifiability, testability).

---

## 2. Virtualization (4th ed. ch. 16)

Keep this conceptual. The chapter's exact terminology and figures are **unverified**.

| Mechanism | What is isolated or shared | Start-up | Typical use |
|---|---|---|---|
| **Virtual machine** | Each VM has a full guest OS on a hypervisor. Isolation is strong | Slow (boots an OS) | IaaS; running different OSes on one host |
| **Container** | Processes share the host OS kernel. The image packages the app plus its libraries. Isolation is lighter | Fast | Consistent packaging "build once, run anywhere"; microservices |
| **Pod** | A group of one or more containers scheduled together on one host, sharing network and storage (the Kubernetes concept) | n/a | Co-locate tightly coupled containers, e.g. an app plus a sidecar |
| **Serverless / FaaS** | You deploy functions. The platform starts instances per request or event and scales them to zero | Can be slow on the first call ("cold start") | Event handlers, glue code (e.g. Firebase Cloud Functions in some projects) |

Architectural points:
- **Images** are immutable artefacts that are versioned and deployed. They support **deployability** (4th ed. ch. 5) and reproducible environments.
- The more virtualised the unit, the more its lifecycle is owned by the platform. **Design for disposability**: stateless instances and state kept outside them (§1.6).
- Virtualization is also a **testability** tactic: a *sandbox* isolates the system under test.
- Isolation is also a **security** tactic: *separate entities*.
- Trade-offs:
  - VMs have stronger isolation than containers, but more overhead.
  - Serverless means no server management, but cold starts, execution-time limits and vendor lock-in.

---

## 3. Software interfaces (4th ed. ch. 15)

**[verify against the book]** The chapter's structure is summarised from general knowledge. Answer at this level and do not quote specifics.

- **Interface.** The boundary through which elements interact. It is what an element *provides* (and *requires*) and what others may assume about it. Everything behind it is hidden (encapsulation / information hiding, Parnas).
- **Resources and operations.** An interface exposes resources (data, services) and the operations on them, with syntax (signatures, message formats) and semantics (pre- and post-conditions, effects, QA properties such as latency).
- **Interaction styles.** Examples are call-return / RPC and REST-style resource operations versus asynchronous messaging and events. Choose based on coupling, latency and failure behaviour.
- **Data representation.** Exchange formats (e.g. JSON, XML, binary schemas) and their effect on size, performance and evolvability.
- **Error handling.** Make errors part of the contract: error codes or exceptions, what the caller must do (retry? idempotent?), and behaviour on timeout. This connects to availability tactics.
- **Versioning / evolution.** Interfaces outlive implementations. Options: keep backward-compatible changes only (add, never remove), run several versions side by side, deprecate with a schedule. This connects to modifiability and integrability.
- **Documenting an interface.** In the Views & Beyond style: identity, resources provided (syntax and semantics), data types, error handling, variability, QA characteristics, rationale, usage guide. Cross-link `documentation.md`.

Course link: the backend-abstraction interface (e.g. a `DatabaseService` or `NetworkAPI` interface in `core`, implemented per platform or backend) is a design decision about interfaces that you can argue for with modifiability and testability.

---

## 4. Mobile systems (4th ed. ch. 18)

Most relevant to TDT4240, because the project is an **Android game in libGDX**. The concerns and course tie-ins below are standard. The chapter's own subsection names are **unverified**.

| Concern | Why it is special on mobile | Architectural responses | libGDX / project example |
|---|---|---|---|
| **Energy** | The battery is finite. CPU, GPU, radio, GPS and screen all drain it | Energy-efficiency tactics (4th ed. ch. 6): monitor and allocate resources, reduce resource demand (batch network calls, lower sampling or frame rate when idle) | Cap FPS in menus; don't poll Firebase in a tight loop, use listeners |
| **Intermittent connectivity** | The network appears, disappears and changes (Wi-Fi to cellular). Bandwidth and latency vary | Offline-first design, local cache and sync, queue outgoing updates, timeouts and retry, idempotent operations | Multiplayer game: handle a disconnect mid-match; the Firebase offline cache |
| **Sensors** | Touch, accelerometer, GPS and camera give noisy, platform-specific input | Abstract sensors behind interfaces; filter or smooth the data; manage the sampling rate | libGDX `Input` abstraction across the desktop (lwjgl3) and android modules |
| **Resources** | Limited memory, CPU, storage and thermal budget, and devices vary widely | Manage resources (pooling, asset lifecycle), degrade gracefully on weak devices | `AssetManager`, `dispose()` textures, object pools |
| **Lifecycle** | The OS can pause, background or kill the app at any time | Save state on pause, restore on resume, and never assume you keep running | libGDX `ApplicationListener.pause()/resume()/dispose()`; the game state manager |
| **Deployment and updates** | Distribution goes through app stores (review delays); users may not update; many OS versions and screen sizes | Backward-compatible backend APIs, feature flags or server-side config, version checks | Keep the backend schema compatible with older clients |

Exam or report angle: mobility was listed as an "other QA" in 3rd ed. ch. 12 (battery, intermittent connectivity). In the 4th ed. it has its own chapter, and energy efficiency has its own QA chapter (ch. 6). See `quality-attributes-4th-edition.md`.

---

## 5. Edge-dominant systems (3rd ed. ch. 27, "Architectures for the Edge")

> **Not edge computing.** In SAiP 3rd ed., "the edge" means the **people at the edge of an organisation**: users and outside contributors who create much of the value. It does **not** mean computing near IoT devices or CDNs. If a student mixes these up, correct them. The chapter is **absent from the 4th ed.**, but it **was examined in 2015**.
>
> Wikipendium has only the heading for this chapter. Everything below comes from the textbook, with details kept deliberately at the level we are confident of.

**Edge-dominant systems** are systems whose value is mostly created by users and peers at the edge rather than by a central organisation. Examples: open-source projects (Linux, Apache), crowdsourced or peer-produced content (Wikipedia), social networks and platforms (YouTube, Facebook-style), and app ecosystems built on a core platform.

### 5.1 The Metropolis model (Kazman & Chen)

Three concentric rings:

| Ring | Who | Role |
|---|---|---|
| **Core** | A small, tightly controlled group of developers (committers / architects) | Build and guard the **kernel/platform**, which is stable, high-quality and modular, with well-defined interfaces |
| **Periphery** | Many loosely coordinated developers and contributors | Build on the core: plug-ins, apps, extensions, patches, content-producing tools. They are often volunteers and self-selected |
| **Masses** | End users (prosumers) | Use the system **and** contribute content, bug reports, data and feature requests |

The Metropolis model is set against the traditional, centrally controlled **waterfall/V-style** view of development. Kazman & Chen also state a set of Metropolis principles. **Do not list them from memory**; point students to the book if an exam asks for them.

### 5.2 How edge-dominant development differs

| Activity | Traditional system | Edge-dominant system |
|---|---|---|
| **Requirements** | Elicited up front from known stakeholders and fixed in a specification | **Emergent**. They come continuously from the periphery and the masses (feature requests, forks, usage) and are never "complete" |
| **Development** | A known team under central management | The core is small and tightly governed. The periphery is large, distributed and self-organising, and its work is only partly under control |
| **Architecture** | Designed for one product | The **core architecture is the key asset**: modular, stable interfaces and extension points so the periphery can innovate without breaking the core |
| **Testing / QA** | A dedicated test phase by the organisation | **Distributed**. The periphery and masses test by using the system ("many eyes"). The core concentrates on tests of the kernel |
| **Releases** | Planned, infrequent, big-bang | **Continuous or ongoing**. Frequent small releases of the core. The periphery releases on its own schedule |
| **Governance** | Management hierarchy | Meritocratic promotion from periphery to core; licensing and contribution rules |

Architectural implications to state in an answer:
- Invest heavily in **modifiability and integrability** at the core boundary: a plug-in/microkernel style, stable APIs, and versioning (see §3).
- Also invest in **scalability and availability**, because many unknown users and contributors arrive.
- **Architect the core, enable the periphery**: you cannot specify everything, so specify the platform and its rules.

---

## 6. Quantum computing (4th ed. ch. 26), lower exam relevance

It is the last chapter of the 4th ed. and titled "A Glimpse of the Future". Expect a definition-level question at most.

- **Qubit.** The quantum analogue of a bit. It can be in a **superposition** of |0> and |1>, described by complex amplitudes.
- **Measurement.** Reading a qubit collapses it to 0 or 1, with probabilities given by the amplitudes. Results are therefore **probabilistic**, and algorithms are run repeatedly or designed so that the wanted answer has high probability. Measurement destroys the superposition, so intermediate state cannot be inspected the way it can in classical debugging.
- **Entanglement.** Qubits whose states are correlated, so they cannot be described independently.
- **Quantum computer as a coprocessor.** In practice a classical host prepares the problem, sends a quantum program (a circuit) to the QPU, and post-processes the measured results. Architecturally, this is a **specialised remote accelerator**, similar to a GPU or a cloud service: think about the interface, latency, queuing, error rates and cost.
- **Algorithms relevant to architects:**
  - **Shor's algorithm** (factoring) threatens today's public-key cryptography, which motivates planning for post-quantum crypto (a **security / modifiability** concern: can you swap crypto algorithms?).
  - **Grover's algorithm** gives a quadratic speed-up for unstructured search.
  - Other candidate uses are simulation and optimisation.
- **Constraints:** noise and decoherence (error correction is needed), limited qubit counts, and no cloning of quantum state.

---

## 7. Machine learning: NOT a confirmed syllabus topic

**The facts:** SAiP 4th ed. has **no chapter on machine learning**, and neither does the 3rd ed. No public TDT4240 course description, exam paper or compendium found lists ML as a topic. Do not tell students that ML is examinable.

> **Beyond syllabus: architecting ML-enabled systems.** This is general engineering practice and not course material. No citations are given here on purpose.
>
> - **The model is a component, not the system.** Most of the architecture sits around it: data ingestion, validation, feature computation, training, serving, monitoring.
> - **Data pipelines.** Version the data as well as the code. Validate schemas and distributions before training. Keep training-time and serving-time feature computation consistent, to avoid "training/serving skew".
> - **Model serving.** Serve the model behind an interface (an intermediary or service) so models can be replaced without touching clients. This is ordinary modifiability. Choose batch or online inference; latency budgets and fallbacks are performance and availability tactics.
> - **Monitoring for drift.** Input data and the real world change over time, so accuracy decays silently. Monitor input distributions and outcome metrics, alert on drift, and plan retraining and rollback (deployability).
> - **The course QA vocabulary still applies.** Write six-part scenarios for these concerns just as for any other system (e.g. "when input drift exceeds threshold X, alert within 1 h").

---

## 8. The role of the architect in projects (4th ed. ch. 24)

**Summary.** The architect works alongside the **project manager**. Roughly, the architect owns the technical decisions and the manager owns the schedule, budget and people, and both need each other: the architecture drives the work breakdown and estimates, and project constraints shape the architecture. The architect should stay involved through the **whole lifecycle**:
- elicit ASRs;
- design, and document "just enough" for the stakeholders;
- evaluate;
- guide the implementation and check that the code conforms to the architecture;
- manage architecture debt.

In **agile** projects this means incremental, just-in-time architecture: enough up-front design to address the high-risk ASRs, then refinement per iteration. In **distributed development** the architecture determines how work can be split across teams, so module boundaries become coordination boundaries (Conway's law).

TDT4240 link: the group's development view should support dividing work among members (teacher feedback, see `course-and-project-guide.md`), and someone should own consistency between the code and the architecture document. The chapter's exact subsection structure is **unverified**. Related 3rd-ed. material is in ch. 22 (management and governance), ch. 15 (agile) and ch. 24 (competence), which is now 4th-ed. ch. 25.

---

## Quick exam checklist

- [ ] Name the 5 NIST characteristics, the 3 service models and the 4 deployment models (a 2015 exam topic).
- [ ] Explain the hypervisor, VMs, virtual storage and networks, and multi-tenancy.
- [ ] Put the QAs in the right place: auto-scaling is **performance/scalability**; frequent failure with detection and recovery is **availability**.
- [ ] Contrast VM, container and serverless in one sentence each.
- [ ] Define **edge-dominant**, which is not edge computing; draw the core, periphery and masses rings; contrast requirements, development, testing and releases.
- [ ] Mobile: energy, connectivity, sensors, resources, lifecycle, updates, each tied to the libGDX project.
- [ ] Quantum: qubit, superposition, measurement, coprocessor, and Shor's threat to cryptography.
- [ ] ML: say plainly that it is outside the SAiP syllabus.
