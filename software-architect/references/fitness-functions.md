# Fitness functions and architecture tests per ecosystem, with CI wiring

Use this file when a design decision, ADR or review finding needs a guard that keeps holding after the author has
moved on. It covers what to enforce, which tool to reach for per ecosystem, how to roll a check out without a
permanently red build, and how to wire it into CI. Scenario writing lives in
[quality-attributes-runtime.md](quality-attributes-runtime.md) and
[quality-attributes-change.md](quality-attributes-change.md); module boundaries in
[module-layout-and-interfaces.md](module-layout-and-interfaces.md); smells that call for a guard in
[review-playbook.md](review-playbook.md); recovering the as-is graph first in
[architecture-recovery-and-metrics.md](architecture-recovery-and-metrics.md); ADR format in
[../templates/adr.md](../templates/adr.md).

**Accuracy rule.** Tool syntax below is limited to forms that are stable and documented. Every tool section ends with
a *Verify against the installed version* line naming what has changed across majors. Before writing config into a
repository, read the version pinned in its lockfile/build file and check the official docs linked there. If the repo
pins an older major, adapt to it; do not upgrade a tool just to use the syntax shown here. Never invent flags.

---

## 0. The idea in one paragraph

An **architectural fitness function** (Ford, Parsons & Kua, *Building Evolutionary Architectures*) is any objective,
automatable check that tells you how close the system is to an architectural characteristic. Documents do not enforce
architecture; builds do. A rule that exists only in a wiki or ADR decays with every contributor who has not read it;
a rule the build checks survives turnover. Classify each fitness function on three axes before you build it:

| Axis | Values | What it means for you |
|---|---|---|
| Scope | **atomic** / **holistic** | One characteristic in isolation (layering) vs several interacting (latency under failover). Holistic ones need an environment. |
| Cadence | **triggered** / **continuous** | Runs on an event (PR, nightly, release) vs monitors production constantly (SLO alerts, synthetic probes). |
| Result | **static** / **dynamic** | Fixed pass/fail threshold vs a threshold that depends on context (e.g. latency budget scaled by load). |

Structural rules are atomic + triggered + static: PR pipeline. Holistic and continuous ones go to nightly jobs,
staging or production monitoring; say which in the ADR.

## 1. The enforcement ladder

Enforce every rule at the **lowest rung that can express it**. Lower rungs are cheaper, faster and harder to bypass.

| Rung | Mechanism | Examples | Strength |
|---|---|---|---|
| 1 | **Compiler / module system** | Gradle/Maven modules, Go `internal/`, Rust crates + `pub(crate)`, Kotlin `internal`, .NET project references, TS project references, `package.json` `exports`, JPMS `module-info.java` | Violation does not compile. Cannot be suppressed by a comment. |
| 2 | **Dependency-rule tool** | dependency-cruiser, import-linter, go-arch-lint, depguard, eslint-plugin-boundaries, Nx module boundaries | Declarative rules over the import graph; baselines supported. |
| 3 | **Architecture unit test** | ArchUnit, Konsist, NetArchTest, ArchUnitNET, a Go test over `go list` | Arbitrary predicates (naming, annotations, inheritance) in the test suite. |
| 4 | **Custom script / grep** | Section 4 one-liners, budget scripts | Last resort: fast to write, easy to fool (aliases, re-exports, dynamic imports). |

If a rule has no home yet: first try to express it in the build graph (split a module, move to `internal/`); then in a
linter the repo already runs (ESLint, detekt, golangci-lint, Ruff banned-api); then add one dependency-rule tool; use
an architecture unit test for non-import predicates (naming, annotations); grep only as a ticketed stopgap.

**Tie every rule to a quality-attribute scenario and the tactic it protects** (tactic names as in *Software
Architecture in Practice*, 4th ed.). A check without a scenario is a style preference. Record the link in the test
name or rule comment:

| QA scenario (response measure) | Tactic the rule protects | Rule | Rung / cadence |
|---|---|---|---|
| Modifiability: swap the persistence engine in < 2 dev-weeks, touching no domain code | Reduce coupling: Restrict dependencies, Use an intermediary (repository interface) | `domain` must not depend on `infrastructure`, ORM or HTTP packages | 1 or 2 (PR) |
| Testability: 100 % of domain logic unit-testable without a device/DB | Testability: Control and observe system state (Abstract data sources); Reduce coupling: Restrict dependencies | no framework types (Android, Spring, Django) in `domain` | 2 or 3 (PR) |
| Modifiability: a new feature ships without editing other features | Increase cohesion: Split module; Reduce coupling: Restrict dependencies | features independent of each other; `core` never depends on `feature` | 1 (build) |
| Deployability: any module builds and releases alone | Reduce coupling: Restrict dependencies | no package cycles | 2 (PR) |
| Performance: p95 checkout < 300 ms at 200 rps | Performance: Control resource demand, Manage resources | load-test threshold | n/a, nightly load test (section 3) |
| Interoperability: old clients keep working for 2 releases | Interoperability: Manage interfaces; Modifiability: Encapsulate | API/ABI compatibility check | PR gate (API-compat tool, section 3) |

Name each test/rule after its ADR (`adr012_core_does_not_depend_on_feature`) so a failure points to the rationale.

## 2. Structural rules per ecosystem

Per ecosystem: dependency, "domain must not depend on infrastructure", no cycles, CI command, gotchas. Names like
`com.acme` and `src/domain` are placeholders for the recovered layout.

### 2.1 JVM (Java, Kotlin/JVM, Scala): ArchUnit

Dependency (Gradle Kotlin DSL, latest 1.x the repo can take):
`testImplementation("com.tngtech.archunit:archunit-junit5:<version>")`.
```java
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;
import static com.tngtech.archunit.library.Architectures.layeredArchitecture;
import static com.tngtech.archunit.library.dependencies.SlicesRuleDefinition.slices;

@AnalyzeClasses(packages = "com.acme")
class ArchitectureTest {
    @ArchTest
    static final ArchRule adr007_domain_is_framework_free =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..infrastructure..", "..web..", "org.springframework..", "jakarta.persistence..");

    @ArchTest
    static final ArchRule adr007_layers =
        layeredArchitecture().consideringAllDependencies()
            .layer("Web").definedBy("..web..")
            .layer("Application").definedBy("..application..")
            .layer("Domain").definedBy("..domain..")
            .layer("Infrastructure").definedBy("..infrastructure..")
            .layer("Config").definedBy("..config..")   // composition root / DI wiring
            .whereLayer("Web").mayNotBeAccessedByAnyLayer()
            .whereLayer("Application").mayOnlyBeAccessedByLayers("Web", "Infrastructure", "Config")
            .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Infrastructure", "Web", "Config")
            .whereLayer("Infrastructure").mayOnlyBeAccessedByLayers("Config");

    @ArchTest
    static final ArchRule adr009_no_cycles_between_features =
        slices().matching("com.acme.(*)..").should().beFreeOfCycles();
}
```

- **Considering variant.** Since 1.0 `layeredArchitecture()` must be followed by `consideringAllDependencies()`,
  `consideringOnlyDependenciesInLayers()` or `consideringOnlyDependenciesInAnyPackage(...)`. With
  `consideringAllDependencies()`, `mayOnlyBeAccessedByLayers`/`mayNotBeAccessedByAnyLayer` also count accesses from
  classes outside every defined layer (the root `@SpringBootApplication` class, `..config..` wiring, DI modules).
  Either define those classes as a layer (the `Config` layer above) or switch to
  `consideringOnlyDependenciesInLayers()`. JDK/library targets only matter for `mayOnlyAccessLayers`.
- **Infrastructure is constrained too.** Without the `whereLayer("Infrastructure")` line, Domain -> Infrastructure and
  Application -> Infrastructure both pass the layered rule; the dependency inversion the ADR intends is then enforced
  only by the separate `noClasses` rule, and only for domain.
- **Baseline for legacy code.** Wrap a rule: `FreezingArchRule.freeze(rule)`. Existing violations are recorded in a
  violation store (default: a directory in the project) and only new ones fail. Store creation must be allowed in
  `archunit.properties` (`freeze.store.default.allowStoreCreation=true`) on the first run; commit the store.
- **Scope.** ArchUnit analyses compiled JVM bytecode. It sees nothing in Kotlin/Native, Kotlin/JS, Wasm or iOS source
  sets of a KMP project; use Konsist or the Gradle module graph there. Compile-time constants inlined by
  `javac`/`kotlinc` leave no bytecode dependency.
- CI: runs as part of `./gradlew test` / `mvn test`; no separate step needed. Exclude test classes with
  `@AnalyzeClasses(packages = "com.acme", importOptions = ImportOption.DoNotIncludeTests.class)`. `archunit-junit5`
  needs `tasks.test { useJUnitPlatform() }`, or no test runs.
- **Gradle filter gotcha.** In a multi-project build, `./gradlew test --tests '*Architecture*'` applies the filter to
  every `test` task, and each subproject without a matching class fails with "No tests found for given includes".
  Target the module that holds the arch tests (`./gradlew :architecture-tests:test`), or set
  `tasks.test { filter { isFailOnNoMatchingTests = false } }`.

**jdeps as a coarse gate** (ships with the JDK): `jdeps -verbose:package build/classes/java/main` prints package edges;
`grep` for a forbidden edge in a pre-ArchUnit repo. `jdeps --jdk-internals` flags JDK-internal API use before upgrades.

Verify against the installed version: 0.x → 1.0 made the `considering...` call mandatory on layered architectures and
removed deprecated APIs; freeze-store property names are documented at https://www.archunit.org/userguide/html/000_Index.html.

### 2.2 Kotlin, KMP and Android: Gradle modules first, then Konsist

**Rung 1: the Gradle module graph.** A boundary between Gradle modules is the strongest one available: `:core:domain`
simply cannot see `:core:data` unless declared. Use `implementation` (not `api`) so transitive types do not leak, and
`internal` for module-private types. When a layer lives inside one module, the first fix is to split the module
(see [module-layout-and-interfaces.md](module-layout-and-interfaces.md)).

Sketch of a check forbidding `:core -> :feature` edges (root `build.gradle.kts`; adapt to your Gradle version):
```kotlin
// SKETCH (ADR-012): fail configuration if any :core* project depends on a :feature* project.
subprojects {
    afterEvaluate {
        if (path.startsWith(":core")) {
            configurations.forEach { conf ->
                conf.dependencies.withType(ProjectDependency::class.java).forEach { dep ->
                    // Gradle >= 8.11: dep.path; older: dep.dependencyProject.path
                    if (dep.path.startsWith(":feature")) {
                        throw GradleException("$path -> ${dep.path} violates ADR-012 (core must not depend on feature)")
                    }
                }
            }
        }
    }
}
```

This runs at configuration time and fails every build, which is what you want. Check interaction with configuration
cache and isolated projects on recent Gradle; a convention plugin applied to each module is the cleaner long-term home.
Also available: the **Dependency Analysis Gradle Plugin** (`com.autonomousapps.dependency-analysis`, task `buildHealth`)
for unused/misdeclared `api` vs `implementation`, and **detekt**'s `ForbiddenImport` rule for import bans inside a
module (configure from the detekt docs for the pinned version).

**Rung 3: Konsist** (source-based, so it covers every KMP source set, unlike ArchUnit). Dependency:
`testImplementation("com.lemonappdev:konsist:<version>")`.
```kotlin
import com.lemonappdev.konsist.api.Konsist
import com.lemonappdev.konsist.api.architecture.KoArchitectureCreator.assertArchitecture
import com.lemonappdev.konsist.api.architecture.Layer
import com.lemonappdev.konsist.api.ext.list.withNameEndingWith
import com.lemonappdev.konsist.api.verify.assertTrue
import org.junit.jupiter.api.Test

class ArchitectureKonsistTest {
    @Test
    fun adr007_clean_layers() {
        Konsist.scopeFromProject().assertArchitecture {
            val domain = Layer("Domain", "com.acme.domain..")
            val data = Layer("Data", "com.acme.data..")
            val presentation = Layer("Presentation", "com.acme.presentation..")
            domain.dependsOnNothing()
            data.dependsOn(domain)
            presentation.dependsOn(domain)
        }
    }
    @Test
    fun adr011_viewmodels_live_in_presentation() = Konsist.scopeFromProject().classes()
        .withNameEndingWith("ViewModel").assertTrue { it.resideInPackage("..presentation..") }
}
```

Gotchas: Konsist parses source, so it misses dependencies introduced only via generated code; put the tests in a
JVM-only test module or source set; `scopeFromProject()` includes test sources, so narrow with
`scopeFromProduction()` or `scopeFromModule(...)` when needed. In CI, run that module's task
(`./gradlew :konsist-test:test`), not a root-level `test --tests` filter (see the Gradle filter gotcha in 2.1).

Verify against the installed version: Konsist is pre-1.0 and its API has moved (`assert {}` was renamed to
`assertTrue {}`; the architecture DSL was added during the 0.x series; extension import paths changed). Docs:
https://docs.konsist.lemonappdev.com/. Gradle `ProjectDependency` API changed in 8.11 (see comment above).

### 2.3 TypeScript / JavaScript

**Rung 1.** TS project references (`"references"` + `"composite": true`) make one package's sources unavailable to
another unless referenced. In a monorepo, a `package.json` `"exports"` map hides internal files from other packages,
but only when the resolver honours it: TypeScript enforces `exports` only with `moduleResolution` `node16`,
`nodenext` or `bundler`, and ignores it under `node`/`node10`. Check `tsconfig.json` before relying on it as a boundary.

**dependency-cruiser** (general-purpose, rules on the resolved import graph). Generate a starter with
`npx depcruise --init`, then keep the rules you need:
```js
// .dependency-cruiser.js (newer versions may generate .cjs/.mjs)
module.exports = {
  forbidden: [
    { name: "adr-007-domain-not-to-infra", severity: "error",
      from: { path: "^src/domain" }, to: { path: "^src/(infrastructure|web)" } },
    { name: "no-circular", severity: "error", from: {}, to: { circular: true } },
  ],
  options: { doNotFollow: { path: "node_modules" }, tsConfig: { fileName: "tsconfig.json" } },
};
```

CI: `npx depcruise src --config .dependency-cruiser.js --output-type err` (non-zero exit on `error` violations).
Baseline: record known violations to `.dependency-cruiser-known-violations.json` and run with `--ignore-known`; the
recording command was a separate `depcruise-baseline` binary in older majors and `--output-type baseline` in newer
ones. Verify against the installed version (config extension, baseline command, flag names):
https://github.com/sverweij/dependency-cruiser.

**eslint-plugin-boundaries** (element-type rules inside ESLint, good IDE feedback):
```js
// eslint.config.js (flat config) excerpt; this object also needs plugins: { boundaries }
settings: { "boundaries/elements": [
  { type: "domain", pattern: "src/domain/*" },
  { type: "application", pattern: "src/application/*" },
  { type: "infrastructure", pattern: "src/infrastructure/*" },
] },
rules: { "boundaries/element-types": [2, { default: "disallow", rules: [
  { from: "domain", allow: ["domain"] },
  { from: "application", allow: ["application", "domain"] },
  { from: "infrastructure", allow: ["infrastructure", "application", "domain"] },
] }] },
```

A pattern ending in `/*` makes **each subfolder a separate element** (`src/domain/order` and `src/domain/customer` are
two `domain` elements). With `default: "disallow"`, imports between sibling elements fail unless a same-type allow
exists, which is why every layer above allows its own type; drop a same-type allow only when the ADR says siblings
must be independent. Files placed directly in `src/domain/` match no element and are not checked, so move them into a
subfolder or add an element for them. Needs a working import resolver for TS paths (e.g.
`eslint-import-resolver-typescript`). Verify against the installed
version: rule names and the selector syntax have changed across majors (newer majors introduce a unified dependencies
rule that supersedes `element-types`); read the README of the installed version at
https://github.com/javierbrea/eslint-plugin-boundaries.

**Nx workspaces**: tag projects (`"tags": ["type:domain"]` in `project.json`) and set
`@nx/enforce-module-boundaries` with `depConstraints: [{ sourceTag: "type:domain", onlyDependOnLibsWithTags:
["type:domain"] }]`. Verify: the rule was `@nrwl/nx/enforce-module-boundaries` before the `@nx` rename; see
https://nx.dev (enforce-module-boundaries docs) for the installed version.

**Zero-dependency option**: ESLint core `no-restricted-imports` with `patterns: [{ group: ["**/infrastructure/**"],
message: "ADR-007: domain must not import infrastructure" }]`, scoped to `src/domain/**` via a `files` entry. It
matches import strings, not resolved paths, so aliases can bypass it. Use `@typescript-eslint/no-restricted-imports`
if type-only imports need distinct handling.

**madge** as a quick cycle gate: `npx madge --circular --extensions ts,tsx src` (add `--ts-config tsconfig.json` for
path aliases). Verify exit-code behaviour on the pinned version before relying on it as a gate:
https://github.com/pahen/madge.

### 2.4 Python: import-linter

```toml
# pyproject.toml (also supported: setup.cfg [importlinter] sections or a .importlinter file)
[tool.importlinter]
root_package = "acme"
include_external_packages = true   # needed to forbid third-party packages such as sqlalchemy

[[tool.importlinter.contracts]]
name = "ADR-007 layers"
type = "layers"
layers = ["acme.api", "acme.application", "acme.domain"]

[[tool.importlinter.contracts]]
name = "ADR-007 domain is framework-free"
type = "forbidden"
source_modules = ["acme.domain"]
forbidden_modules = ["acme.infrastructure", "sqlalchemy", "django"]

[[tool.importlinter.contracts]]
name = "ADR-009 features are independent"
type = "independence"
modules = ["acme.billing", "acme.shipping", "acme.catalog"]
```

- Layers are listed **highest first**; a lower layer importing a higher one fails, which also rules out cycles
  between layers. `independence` forbids imports among the listed siblings in both directions.
- CI: `lint-imports` (non-zero exit on a broken contract). It statically follows imports, including those inside
  functions, but not `importlib` or string-based dynamic imports.
- Baseline: `ignore_imports = ["acme.domain.legacy -> acme.infrastructure.db"]` per contract, each with a comment
  naming the owner and removal ticket.
- **pydeps** is for visualisation (`pydeps acme --show-cycles`), not enforcement.
- Startup budget: `python -X importtime -c "import acme.api" 2> importtime.log` writes per-module import cost to
  stderr. Cumulative times are nested (a parent's figure includes its children), so do not sum that column: read the
  cumulative value on the line for the top-level module (`acme.api`), or sum the `self` column, and fail above a budget.

Verify against the installed version: options such as `root_packages` (plural), `containers` for layers and newer
contract types depend on the release; see https://import-linter.readthedocs.io/.

### 2.5 Go

**Rung 1.** A package under `internal/` can only be imported by code rooted at the parent of `internal`. Import
cycles are compile errors, so no cycle rule is needed.

**go-arch-lint** (component graph):
```yaml
# .go-arch-lint.yml
version: 3
workdir: internal
allow:
  depOnAnyVendor: true   # first rollout: third-party imports allowed; tighten later with vendors: + canUse:
components:
  domain: { in: domain/** }
  app:    { in: app/** }
  infra:  { in: infrastructure/** }
deps:
  app:   { mayDependOn: [domain] }
  infra: { mayDependOn: [domain, app] }
```

Run `go-arch-lint check`. Without `depOnAnyVendor`, every third-party import (uuid in domain, pgx in infrastructure)
must be declared under `vendors` and granted via `canUse`, or it is reported as a violation.

**golangci-lint depguard** (import deny-lists; config shape as in golangci-lint v1 with depguard v2). depguard is not
in the default linter set, so the settings do nothing unless the linter is enabled:
```yaml
linters:
  enable: [depguard]
linters-settings:
  depguard:
    rules:
      domain:
        files: ["**/internal/domain/**"]
        deny:
          - pkg: "github.com/acme/app/internal/infrastructure"
            desc: "ADR-007: domain must not depend on infrastructure"
```

**Plain `go test`, no extra tool** (checks transitive dependencies via `go list`):
```go
package archtest

import (
	"os/exec"
	"strings"
	"testing"
)

const module = "github.com/acme/app"

func TestADR007DomainDoesNotDependOnInfrastructure(t *testing.T) {
	out, err := exec.Command("go", "list", "-f", `{{.ImportPath}} {{join .Deps " "}}`,
		module+"/internal/domain/...").CombinedOutput()
	if err != nil {
		t.Fatalf("go list: %v\n%s", err, out)
	}
	forbidden := module + "/internal/infrastructure"
	for _, line := range strings.Split(strings.TrimSpace(string(out)), "\n") {
		fields := strings.Fields(line) // package path followed by its transitive deps
		for i := 1; i < len(fields); i++ {
			dep := fields[i]
			if dep == forbidden || strings.HasPrefix(dep, forbidden+"/") {
				t.Errorf("%s depends on %s", fields[0], dep)
			}
		}
	}
}
```

Use `.Imports` instead of `.Deps` to check direct imports only. Place it in e.g. `internal/archtest/`.

Verify against the installed version: go-arch-lint's schema `version` and vendor options (`depOnAnyVendor`,
`vendors`, `canUse`) differ between releases (https://github.com/fe3dback/go-arch-lint); depguard v1 used a flat
`list-type`/`packages` config, v2 uses `rules`; golangci-lint v2 requires `version: "2"`, keeps `linters: enable:`
and moves settings to `linters: settings: depguard:` (https://golangci-lint.run/).

### 2.6 .NET

**Rung 1.** Separate projects (`Acme.Domain.csproj` with no reference to `Acme.Infrastructure`) plus `internal` and
`InternalsVisibleTo` for tests. **Rung 3**, NetArchTest (`dotnet add package NetArchTest.Rules`):
```csharp
[Fact]
public void Adr007_DomainDoesNotDependOnInfrastructure()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .That().ResideInNamespace("Acme.Domain")
        .ShouldNot().HaveDependencyOn("Acme.Infrastructure")
        .GetResult();
    Assert.True(result.IsSuccessful,
        "ADR-007 violated by: " + string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>()));
}
```

ArchUnitNET (`TngTech.ArchUnitNET`) ports the ArchUnit API and adds cycle/slice rules. Verify against the installed
version: NetArchTest's original package is lightly maintained and a community fork exists
(`NetArchTest.eNhancedEdition`) with extended APIs; failing-type reporting differs between them. Docs:
https://github.com/BenMorris/NetArchTest and https://archunitnet.readthedocs.io.

### 2.7 Rust

Workspace crates are the boundary (`domain` crate has no dependency on `infra` in its `Cargo.toml`); inside a crate
use `pub(crate)` / `pub(super)` to keep internals private. Cargo rejects cycles between crates (dev-dependencies aside). **cargo-deny**
(`cargo deny init`, `cargo deny check`) enforces dependency policy (bans, licences, advisories, sources), not layering.
Verify config keys against https://embarkstudios.github.io/cargo-deny/ for the pinned version.

## 3. Non-structural fitness functions

Map each to its QA scenario and tactic. Fast, deterministic ones on PRs; noisy ones nightly with trend tracking.

| QA / tactic | Fitness function | Tooling and form |
|---|---|---|
| Performance: Control resource demand, Manage resources (latency budget) | Load test with thresholds | **k6**: `export const options = { thresholds: { http_req_duration: ["p(95)<300"], http_req_failed: ["rate<0.01"] } };` (k6 exits non-zero when a threshold fails). **Gatling**: `assertions(global().responseTime().percentile(95.0).lt(300))` in the Java DSL; Scala DSL differs. **Locust**: no built-in thresholds; in an `events.quitting` listener set `environment.process_exit_code = 1` when `environment.stats.total.get_response_time_percentile(0.95)` exceeds the budget. |
| Performance: Control resource demand, e.g. Reduce computational overhead (hot path) | Microbenchmark vs baseline | **JMH** has no pass/fail; export `-rf json` and compare with a script. **pytest-benchmark**: `--benchmark-autosave` on main, `--benchmark-compare --benchmark-compare-fail=mean:10%` on PRs. **Go**: `go test -bench=. -count=10` on base and head, compare with `benchstat`; benchstat reports but does not fail, so wrap it. CI runners are ephemeral: `--benchmark-compare` reads the local `.benchmarks/` directory and benchstat needs the base run's output, so persist the main-branch baseline as a CI cache/artifact keyed by commit, or run base and head in the same job (checkout base, bench, checkout head, bench, compare). Run on dedicated or pinned runners; shared CI runners are noisy. |
| Performance: Control resource demand (web payload) | Bundle size budget | `size-limit` (`[{ "path": "dist/index.js", "limit": "50 kB" }]`, `npx size-limit`), or the bundler's own budget (Angular `budgets`; webpack `performance: { hints: "error", maxAssetSize: ... }`, since without `hints: "error"` webpack only warns). |
| Performance: Control resource demand (startup) | Startup / first-frame budget | Android **Macrobenchmark** `StartupTimingMetric`; iOS `XCTApplicationLaunchMetric` in `measure(metrics:)`; Python `-X importtime`; JVM: time to readiness probe. Assert against a budget with tolerance. |
| Interoperability: Manage interfaces; Modifiability: Encapsulate (published API) | API/ABI compatibility | Kotlin **binary-compatibility-validator** (`apiDump`, `apiCheck`; KGP 2.2+ ships an experimental built-in ABI validation that is superseding it). Java **japicmp** (Maven plugin or CLI). TS **API Extractor** (`api-extractor run`; `--local` updates the report file). Protobuf **`buf breaking --against '.git#branch=main'`**. OpenAPI: a diff tool such as oasdiff in breaking-change mode, pinned. |
| Interoperability: Manage interfaces (service contracts) | Consumer-driven contract tests | **Pact**: consumers publish pacts, providers verify in CI, `can-i-deploy` gates release via a Pact Broker. |
| Security: supply chain (no dedicated SAiP tactic; supports Resist attacks) | Vulnerability and licence scans | `osv-scanner` (v2 syntax `osv-scanner scan source -r .`; v1 was `osv-scanner -r .`), Dependabot or Renovate for update PRs, `cargo deny check`, GitLab dependency scanning. Fail on new high/critical only. |
| Testability: measures the payoff of Control and observe system state, Limit complexity | Coverage floor on domain modules only | Kover or JaCoCo `jacocoTestCoverageVerification` scoped to `:core:domain`; `pytest --cov=acme.domain --cov-fail-under=90`; Jest `coverageThreshold` with a per-path key; Go: `go test -coverprofile` + a script. A global floor rewards testing getters; a floor on the money logic does not. |
| Deployability: Manage deployed system; Availability: Prevent faults (zero-downtime deploys) | Migration backward-compatibility | Run the **previous** release's test suite (or smoke tests) against the **new** schema; enforce expand/contract (no drop/rename in the same release as the code change). |
| Availability: Detect faults, Recover from faults (e.g. Degradation, Retry) | Chaos / fault injection | Toxiproxy for latency/drop between services in integration tests; Chaos Mesh, Litmus or cloud fault-injection services in staging. Assert the scenario's response measure (e.g. degraded mode within 2 s), not just "didn't crash". |
| Energy efficiency: Monitor resources (Metering); checks that Reduce resource demand paid off (mobile) | Energy/CPU profiling hooks | Android Macrobenchmark `PowerMetric` (experimental, needs devices with power rails), Battery Historian; iOS `XCTCPUMetric`/`XCTMemoryMetric` in tests, MetricKit and Xcode Organizer energy reports in the field. Usually nightly or pre-release, not per PR. |

Verify against the installed version for every row: these tools evolve fast (k6 extensions, Gatling DSL, osv-scanner
v1→v2 CLI, Kover 0.7→0.8+ config DSL, BCV moving into the Kotlin Gradle plugin). Link to official docs in the ADR.

## 4. Grep fallback one-liners

For a first look during review, or a ticketed stopgap CI gate. They miss aliases, relative paths, re-exports, star
imports via facades, and dynamic imports.

`rg` exits 0 on a match, 1 on no match and 2 on an error (missing or renamed directory, bad pattern); with an error it
can exit 2 even when it also printed matches. So `! rg ...` is for **interactive review only**: it turns the error
exit into success, and a `!`-inverted command never trips `set -e`, so in a multi-line CI script only the last
`! rg` line can fail the job. In CI use a form that passes only on exit 1:

```sh
# Passes only on "no match" (exit 1); fails on a match (0) and on an error (2). Safe under `set -e`.
deny() { s=0; rg -n "$@" || s=$?; [ "$s" -eq 1 ]; }

deny '^import .*\.infrastructure\.' src/main/kotlin/com/acme/domain                             # Kotlin
deny '^import .*\.infrastructure\.' src/main/java/com/acme/domain                               # Java
deny "(from|require\()\s*['\"][^'\"]*/infrastructure(/|['\"])" src/domain                       # TS/JS
deny '^\s*(from|import)\s+acme\.infrastructure' src/acme/domain                                 # Python
deny '"github.com/acme/app/internal/infrastructure' internal/domain                             # Go
deny '^using\s+Acme\.Infrastructure' src/Acme.Domain                                            # C#
deny 'use\s+(crate::infrastructure|infra)\b' crates/domain/src                                  # Rust
deny '^import android\.' core/domain/src                                                        # Android leakage
```

Keep only the lines whose directories exist in the repo; with `deny`, a missing directory fails the job loudly
instead of disabling the gate. For a quick look in a terminal, plain `rg -n PATTERN DIRS` is enough.

In a review, a hit is an AS-IS finding with file:line evidence; see [review-playbook.md](review-playbook.md).

## 5. Ratchet and rollout

Never introduce a structural check that fails on day one for code nobody is changing. A permanently red build trains
people to ignore it; a fitness function that encodes a structure you have not reached yet is a permanently red build.

1. **Measure.** Run the rule in report-only mode; count violations; attach the number to the ADR.
2. **Baseline.** Record existing violations (ArchUnit freeze store, dependency-cruiser known violations,
   import-linter `ignore_imports`, detekt baseline, ESLint bulk suppressions via `--suppress-all` /
   `eslint-suppressions.json` in ESLint >= 9.24; older ESLint needs a third-party baseline tool). Commit the baseline.
3. **Warn.** Run on PRs as non-blocking for one or two iterations; fix false positives in the rule, not with
   suppressions.
4. **Fail on new violations only.** Make the job required. The baseline may only shrink: review any PR that grows it
   as an architecture change.
5. **Ratchet down.** Burn down the baseline with the refactoring the ADR proposes; delete the baseline entry when fixed
   so the baseline cannot silently re-absorb the violation. ArchUnit's freeze store drops violations that no longer
   occur on its own; commit the updated store. dependency-cruiser known violations: regenerate the baseline after a
   fix and commit the smaller file. import-linter `ignore_imports`: prune entries by hand, and check whether the
   installed version can report unmatched ignored imports (e.g. an `unmatched_ignore_imports_alerting` option) and set
   it to error.
6. **Harden.** When the baseline is empty, move the rule down the ladder if possible (e.g. into a module boundary).

Suppressions carry a justification, an owner and a removal ticket in the same line or block, and are reviewed like
code (list them in the periodic architecture review). Each check costs maintenance and must earn it: a check nobody
trusts gets suppressed wholesale and is worse than none, so fix or remove a flaky check within days. Test and rule
names carry the ADR ID; the ADR's enforcement line names the test or rule.

## 6. CI wiring

GitHub Actions (on every PR; adapt the setup step to the ecosystem):
```yaml
# .github/workflows/architecture.yml
# Match action majors and runtime versions to the repo's other workflows.
name: architecture
on: { pull_request: {}, push: { branches: [main] } }
jobs:
  arch-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-java@v5
        with: { distribution: temurin, java-version: "21" }
      - run: ./gradlew :architecture-tests:test         # module holding the ArchUnit / Konsist tests
      - uses: actions/setup-node@v5
        with: { node-version-file: .nvmrc }             # or a current LTS, e.g. "22" or "24"
      - run: npm ci
      - run: npx depcruise src --config .dependency-cruiser.js --output-type err
```

Gotcha: do not replace the Gradle step with a root-level `./gradlew test --tests '*Architecture*'` in a multi-module
build; every module without a matching test fails with "No tests found for given includes" (see 2.1).

GitLab CI (on every MR and on the default branch):
```yaml
# .gitlab-ci.yml excerpt
architecture:
  stage: test
  image: python:3.12
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  script:
    - pip install -e . import-linter
    - lint-imports
```

Emit JUnit XML where the tool supports it (`artifacts: reports: junit:` in GitLab, a test-reporter action in GitHub)
so violations show inline in the PR/MR. Mark the job as a required check once past the warn phase.

**CI checklist**
- [ ] Each check names the ADR/scenario it enforces, in the job/test/rule name.
- [ ] Structural checks run on every PR/MR and on the default branch; they finish in under ~2 minutes.
- [ ] Holistic/noisy checks (load, chaos, energy, benchmarks) run nightly or pre-release with stored trends.
- [ ] Tool versions are pinned (lockfile, Gradle catalog, `go install ...@vX.Y.Z`, `pip` constraints).
- [ ] Baseline committed; job fails only on new violations; failure output names rule, offending edge and ADR.
- [ ] Job is required for merge after the warn phase; no `allow_failure: true` / `continue-on-error: true` left behind.
- [ ] Local run command documented in CONTRIBUTING or the README, identical to CI.

## 7. Rule type to recommended tool

| Rule type | JVM | Kotlin / KMP / Android | TS / JS | Python | Go | .NET | Rust |
|---|---|---|---|---|---|---|---|
| Layer direction | Modules; ArchUnit `layeredArchitecture` | Gradle modules; Konsist `assertArchitecture` | dependency-cruiser; eslint-plugin-boundaries; Nx tags | import-linter `layers` | `internal/`; go-arch-lint | Project refs; NetArchTest | Workspace crates |
| Forbidden dependency (framework in domain) | ArchUnit `noClasses()...` | Konsist; detekt `ForbiddenImport` | dependency-cruiser `forbidden`; `no-restricted-imports` | import-linter `forbidden` | depguard; `go list` test | NetArchTest `HaveDependencyOn` | Crate deps; cargo-deny `bans` for third-party |
| No cycles | ArchUnit `slices().beFreeOfCycles()` | Gradle modules (cycles rejected); Konsist | dependency-cruiser `circular`; madge | import-linter `layers`/`independence` (listed modules only); pydeps `--show-cycles` to detect; check whether the installed import-linter offers a dedicated cycle contract | Compiler | ArchUnitNET slices | Cargo |
| Feature independence | ArchUnit slices | Gradle `:feature` modules + edge check | Nx tags; boundaries | import-linter `independence` | go-arch-lint | NetArchTest / project refs | Crates |
| Naming / placement | ArchUnit `classes()...` | Konsist declaration checks | eslint-plugin-boundaries / custom lint | Custom test | Custom `go/packages` test | NetArchTest | Clippy / custom |
| Public API stability | japicmp | binary-compatibility-validator / KGP ABI validation | API Extractor | Custom snapshot of `__all__` | `gorelease` / apidiff-style tooling, verify | Package validation / API compat tools, verify | `cargo-semver-checks` |
| Performance budget | JMH + script; Gatling | Macrobenchmark | size-limit; k6 | pytest-benchmark; Locust | `go test -bench` + benchstat | BenchmarkDotNet + script | criterion + script |
| Dependency policy | osv-scanner; Renovate | osv-scanner; Renovate | osv-scanner; npm audit; Renovate | osv-scanner; pip-audit | osv-scanner; govulncheck | osv-scanner; `dotnet list package --vulnerable` | cargo-deny; cargo-audit |

Cells marked "verify": confirm the invocation in the repo's toolchain first. When unsure, propose rule and tool and
leave exact config for the implementer to confirm.
