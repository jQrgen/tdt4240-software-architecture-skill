# Recovering the as-is architecture: graphs, reflexion, metrics, git history

Use this file before you design into an existing codebase (SKILL.md Build step 0) and at the start of every
architecture review ([review-playbook.md](review-playbook.md)). The goal is a short, evidence-backed picture of the
architecture the code *actually has*, kept separate from the architecture people *say* it has. Everything
downstream (findings, ADRs, fitness functions) is only as good as this picture.

Keep two classes of statement apart: **as-is** (a claim about the code at a named commit, always with evidence) and
**intended / to-be** (what docs, the team, or your proposal say the structure should be; mark it as such). The gap
between them is the architectural debt; never blur it by describing intent in the present tense.

## 1. Outputs, timebox, evidence

### What to produce

Sketch one view per structure category (SAiP: module, component-and-connector, allocation). Each answers a different
question; do not merge them into one "box diagram".

| View | Question | Recover these elements |
|---|---|---|
| **Module** (static) | What are the implementation units, and what depends on what? | Build units (Gradle/Maven modules, packages, workspaces, crates), package tree, uses/dependency edges, layer order, cycles |
| **C&C** (runtime) | What runs, and how do the running parts talk? | Entry points (`main`, servlets, handlers, lambdas, activities), processes/services, connectors (HTTP, gRPC, queues, DB drivers, sockets, in-process event buses, flows/channels), threads/coroutines/goroutines, external systems, data stores |
| **Allocation** | Where does it run, how is it built and shipped, who owns it? | Deployment units (images, APKs, jars, functions), environments, config and secret sources, CI stages, module-to-artifact mapping, code ownership |

Also name explicitly: the **composition root(s)** (where objects are wired together), every **external system**, and
every **data store** with its owning module.

### Timebox

The overall review depth (quick / pass / deep), its budget and the report sections it must fill are defined once in
[../SKILL.md](../SKILL.md) ("Review a repository"). This table is the recovery share of each depth.

| Depth | Recovery budget | Covers | Stop when |
|---|---|---|---|
| **Quick** | ~15-20 min | §2 steps 1-4 and 10; a module list and a guessed layer order | You can name the build units, entry points and composition root |
| **Pass** | ~45 min | All of §2, a tool-generated module graph (§4-5), one scenario trace (§6), churn top 20 (§11) | You can draw the three views and list 3-5 suspected divergences |
| **Deep dive** | half day+ | Reflexion model (§8), metrics (§10), change coupling (§11), 2-3 traced scenarios | Every finding in the report has level-A or level-B evidence |

State the depth you ran in the output block (§13). Never present a quick pass as a full recovery.

### Evidence levels and citation

Use the A-D scale defined in [../SKILL.md](../SKILL.md) ("Evidence rules"): A measured (command + output + SHA),
B observed (`path/File.kt:120-134`), C inferred (say from what; flag as unverified), D assumed or reported (never the
sole basis for a Blocker or Major).

Record `git rev-parse --short HEAD` once, at the start. Line numbers without a SHA rot.

## 2. Quick-pass checklist (10 steps)

Run in order. Each step names what to look for and the signal that matters.

0. **Snapshot and file universe.**
   ```bash
   git rev-parse --short HEAD; git status --short | wc -l          # SHA; uncommitted files (review HEAD, note N)
   git rev-parse --is-shallow-repository; git rev-list --count HEAD  # "true" or a count of 1 = no usable history
   git ls-files | rg -v '^(\.claude/worktrees|build|dist|node_modules|kotlin-js-store|\.venv)/|/(build|generated)/' > files.txt
   ```
   Use `files.txt` (tracked files only) as the universe for every count, grep and hotspot; plain `find` also picks up
   stale worktree copies (`.claude/worktrees/`), build output, `kotlin-js-store/`, `.db` files and logs, and can
   double file counts. With `rg`, pass the same exclusions as `-g '!.claude/worktrees/**'` etc., or pipe `files.txt`
   through `xargs rg`. On a shallow clone, ask before `git fetch --unshallow` (network; changes the repo); otherwise
   mark churn, hotspots and co-change "not checked: shallow clone" and use file size only as a reading pointer.
   List the exclusions in the report's Scope and method.
1. **Manifests and build graph.** Find every build file (§3). List build units and their declared inter-unit edges.
   This is the most reliable module view you will get; start here, not in `src/`.
2. **Entry points.** `main` functions, `Application` subclasses, Android manifest activities/services, `server.ts` /
   `index.ts` / `app.listen`, `if __name__ == "__main__"`, ASGI/WSGI `app` objects, `cmd/*/main.go`, `Program.cs`,
   lambda/function handlers, CLI `bin` entries in `package.json` or `[project.scripts]`.
3. **Composition root / DI.** Where are implementations bound to interfaces? Koin `module { single { } }` /
   `startKoin`, Hilt `@Module` + `@InstallIn`, Dagger `@Component`, Spring `@Configuration` / `@Bean` / component scan,
   NestJS `@Module({ providers })`, FastAPI `Depends(...)`, Go/Rust/manual wiring in `main`. Global singleton signals:
   Kotlin `object` holding state, `lateinit var` at top level, `companion object { lateinit var instance }`,
   `getInstance()`, module-level mutable globals in Python/TS, Go package-level `var db *sql.DB`, `!!` on a global app
   reference. Many singletons = the composition root is implicit and every consumer is coupled to it.
4. **Package tree.** `tree -d -L 3 src` (or `find src -type d | head -100`). Note naming schemes (by layer:
   `controllers/services/repositories`; by feature: `orders/`, `billing/`; by port/adapter). Folder names are
   hypotheses, not evidence (§14).
5. **External systems via import grep.** Grep for client libraries: HTTP (`ktor.client`, `okhttp`, `retrofit`,
   `axios`, `fetch(`, `requests`, `httpx`, `net/http`), brokers (`kafka`, `amqp`, `nats`, `sqs`, `pubsub`), SDKs
   (`aws-sdk`, `firebase`, `stripe`). Record which modules import them: that set is the de-facto adapter layer.
   Directories that talk to external systems, with file counts (extend the pattern to the stack):

   ```bash
   rg -l -e 'okhttp|retrofit|ktor\.client|axios|requests|httpx|net/http|kafka|amqp|nats|aws-sdk|firebase|stripe|jdbc|exposed|sqlalchemy|prisma|gorm' \
      -g '!**/test/**' -g '!**/tests/**' -g '!**/*_test.go' \
   | sed -E 's#/[^/]+$##' | sort | uniq -c | sort -rn
   ```
6. **Persistence ownership.** Find DB drivers/ORMs (`jdbc`, `exposed`, `room`, `sqldelight`, `hibernate`, `prisma`,
   `typeorm`, `sqlalchemy`, `django.db`, `database/sql`, `gorm`, `EntityFramework`) and migrations. Which modules touch
   which tables? More than one writer per table, or several services on one schema, is a key finding.
7. **Concurrency model.** Coroutine scopes and dispatchers, `Thread`/`ExecutorService`, `async`/`await`, worker
   threads, `asyncio`, Celery/RQ workers, `go func`, channels, actors, schedulers/cron. Note shared mutable state
   reachable from more than one of them.
8. **Deployment artifacts.** `Dockerfile*`, `docker-compose*.yml`, Helm charts, k8s manifests, Terraform/Pulumi/CDK,
   `serverless.yml`, SAM/CloudFormation, Procfile, `fly.toml`, app store build config, CI files
   (`.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`).
9. **Test layout.** Unit vs integration vs e2e directories, fakes/mocks, test fixtures that start real infra
   (Testcontainers, docker-compose in CI). Which modules have no tests? Existing architecture tests (ArchUnit,
   Konsist, dependency-cruiser rules, import-linter contracts) are an explicit statement of intent: read them.
10. **Existing intent.** `docs/adr/`, `doc/architecture*`, arc42/C4/Structurizr files, `README*`, `CONTRIBUTING*`,
    `CLAUDE.md`, `AGENTS.md`, `CODEOWNERS`. This is your hypothesized model for §8, not a description of the code.

## 3. Files to read per ecosystem

| Ecosystem | Read | Module edges come from |
|---|---|---|
| Gradle (JVM/Android) | `settings.gradle(.kts)`, each `build.gradle(.kts)`, `gradle/libs.versions.toml`, `buildSrc/` or `build-logic/` convention plugins | `include(":a", ":b:c")` gives the units; `implementation(project(":x"))` / `api(project(":x"))` / `projects.x` give edges (`projects.x` type-safe accessors exist only if `settings.gradle(.kts)` has `enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")`; still a feature preview, so version-dependent). `api` leaks the dependency to consumers |
| Kotlin Multiplatform | as Gradle, plus `kotlin { sourceSets { ... } }` | Source sets (`commonMain`, `jvmMain`, `androidMain`, `iosMain`, `wasmJsMain`) and their `dependencies { }`; `expect`/`actual` declarations mark the platform seam |
| Maven | root `pom.xml` `<modules>`, each module `pom.xml`, `<dependencyManagement>` | `<dependency>` on sibling `groupId:artifactId` |
| npm / pnpm / yarn / Bun | root `package.json` `workspaces` (also Bun, with `bun.lock` or older `bun.lockb`; `bun run --filter` runs per workspace), `pnpm-workspace.yaml`, each package `package.json` | `dependencies` on `workspace:*` or sibling names; `tsconfig.json` `paths` and `references`; `nx.json` + `project.json` tags; `turbo.json` pipelines |
| Python | `pyproject.toml` (`[project]`, `[tool.poetry]`, `[tool.importlinter]`), `setup.cfg`, `src/` layout, `requirements*.txt`; uv workspaces: root `[tool.uv.workspace] members` + `uv.lock` | Imports; uv workspace members depend on each other via `[tool.uv.sources] x = { workspace = true }`; `[tool.importlinter]` contracts if present |
| Go | `go.mod`, `go.work`, `cmd/`, `internal/`, `pkg/` | Imports; `internal/` is compiler-enforced visibility; `go.work` lists local modules |
| .NET | `*.sln`, each `*.csproj`, `Directory.Build.props`, `Directory.Packages.props` | `<ProjectReference Include="..\X\X.csproj" />` |
| Rust | root `Cargo.toml` `[workspace] members`, each crate `Cargo.toml` | `[dependencies] x = { path = "../x" }` or `x.workspace = true` |

## 4. Dependency graph commands

Prefer the build tool's own graph for module edges and a language-level tool for package edges. These are commands
whose behaviour is stable across recent versions; flags do drift, so **verify against the installed version**
(`--help`) before quoting output in a report.

**Side effects.** Commands marked *(writes)* create or update build directories and caches in the repo (`build/`,
`.gradle/`, `.kotlin/`, `target/`, `node_modules/`, `.venv/`), may download dependencies and may start daemons. Skip
them when the repo must stay untouched and use the static alternatives below; `./gradlew --offline
--project-cache-dir /tmp/<x>` reduces the writes but does not remove them. The Python and TS extractors and the grep
fallback read files only.

**Static module graph (no build tool run)**

```bash
rg -n 'include\s*\(|^\s*include\s+["\x27]' settings.gradle*                  # Gradle units
rg -n 'project\(\s*["\x27]:|projects\.[a-zA-Z]' -g '*.gradle' -g '*.gradle.kts'   # Gradle unit edges
rg -n '<module>' -g 'pom.xml'                                                  # Maven units; edges: sibling <artifactId>
rg -n '"workspace:|"file:\.\./' -g 'package.json'                                # JS workspace edges
```

Read `settings.gradle(.kts)` by hand too: `include(":a", ":b")` spans lines, and `projectDir` remapping changes paths.

**JVM**

```bash
./gradlew projects                                              # (writes) module tree
./gradlew :app:dependencies --configuration runtimeClasspath    # (writes) KMP: e.g. jvmRuntimeClasspath
./gradlew :app:dependencyInsight --dependency okhttp --configuration runtimeClasspath   # (writes)
mvn dependency:tree                                             # (writes ~/.m2, may download) add -pl <module>
jdeps -summary -recursive -cp 'libs/*' app.jar                  # jar-to-jar
jdeps -verbose:package -cp 'libs/*' app.jar                     # package-to-package
jdeps --dot-output out/ -verbose:package app.jar                # .dot files for Graphviz
```

`jdeps` reads compiled bytecode, so it sees every real type reference. It misses reflection, DI-container wiring and
`ServiceLoader` (§14), compile-time constants that javac/kotlinc inline (`static final` primitives and Strings,
`const val`), and source-retention annotations. If it aborts on missing classes, add `--ignore-missing-deps` (recent
JDKs); for multi-release jars add `--multi-release <jdk-version>`. Check `jdeps --help` on the installed JDK.

**TypeScript / JavaScript**

Zero-install default (stdlib Python; resolves `./`, `../`, ESM `.js` specifiers and `compilerOptions.paths` aliases
such as `@/*`; does not follow `extends`, so merge an inherited `paths` by hand; bare package imports are dropped):

```python
# usage: python3 tsedges.py <tsconfig.json> <src-dir> | sort | uniq -c | sort -rn > edges.txt
import json, os, re, sys
cfg_path, src = sys.argv[1], sys.argv[2]
raw = open(cfg_path, encoding="utf-8").read()
raw = re.sub(r'("(?:\\.|[^"\\])*")|/\*.*?\*/|//[^\n]*', lambda m: m.group(1) or "", raw, flags=re.S)
raw = re.sub(r",\s*([}\]])", r"\1", raw)               # tsconfig allows comments and trailing commas
opts = json.loads(raw).get("compilerOptions", {})      # "extends" is NOT followed: merge by hand if used
base = os.path.normpath(os.path.join(os.path.dirname(cfg_path), opts.get("baseUrl", ".")))
aliases = [(k.rstrip("*"), [os.path.join(base, v.rstrip("*")) for v in vs])
           for k, vs in opts.get("paths", {}).items()]
EXTS = ["", ".ts", ".tsx", ".d.ts", ".js", ".jsx", ".mts", "/index.ts", "/index.tsx", "/index.js"]
SPEC = re.compile(r"""(?:from\s+|import\s*\(\s*|require\s*\(\s*|^\s*import\s+)['"]([^'"]+)['"]""", re.M)
def to_file(p):
    for e in EXTS:
        c = p + e
        if os.path.isfile(c): return os.path.relpath(c)
    if p.endswith(".js"): return to_file(p[:-3])       # ESM style "./x.js" pointing at x.ts
    return None
for d, _, files in os.walk(src):
    if "node_modules" in d: continue
    for name in files:
        if not re.search(r"\.(m?[jt]sx?)$", name): continue
        f = os.path.join(d, name)
        for spec in SPEC.findall(open(f, encoding="utf-8", errors="replace").read()):
            if spec.startswith("."):
                cands = [os.path.normpath(os.path.join(d, spec))]
            else:
                cands = [os.path.normpath(t + spec[len(k):]) for k, ts in aliases if spec.startswith(k) for t in ts]
            hit = next((r for r in map(to_file, cands) if r), None)
            if hit: print(f"{os.path.relpath(f)} -> {hit}")   # bare package imports (react, zod) are dropped
```

Tool-based (`npx` downloads into the npm cache; run from outside the repo or only when a devDependency exists):

```bash
npx madge --circular --extensions ts,tsx src/                   # cycles; add --ts-config tsconfig.json for paths
npx madge --extensions ts,tsx --image graph.svg src/            # needs Graphviz installed
npx -p dependency-cruiser depcruise src --include-only "^src" --output-type dot | dot -T svg > deps.svg
npx nx graph                                                    # Nx workspaces; project-level graph
```

The npm package is `dependency-cruiser`; `depcruise` is only its binary, so use `npx -p dependency-cruiser depcruise`
unless it is already a devDependency. It needs a config for rules; recent versions create one with
`npx -p dependency-cruiser depcruise --init`. It also has
folder-level reporters (`--output-type ddot`, `archi`, or `--collapse "^src/[^/]+"`) that collapse file edges for you;
check `--help` of the installed version.

**Python**

Zero-install default (stdlib `ast`; resolves relative imports and expands `from pkg import name` to `pkg.name` when
`name` is a module, which the grep pattern cannot; also counts imports under `if TYPE_CHECKING:`, so check those):

```python
# usage: python3 pyedges.py <src-root> <top-package> | sort | uniq -c | sort -rn > edges.txt
import ast, pathlib, sys
root, top = pathlib.Path(sys.argv[1]), sys.argv[2]
mods = {}                                              # dotted module name -> (file, is_package)
for f in (root / top).rglob("*.py"):
    parts = list(f.relative_to(root).with_suffix("").parts)
    is_pkg = parts[-1] == "__init__"
    mods[".".join(parts[:-1] if is_pkg else parts)] = (f, is_pkg)
def resolve(name):                                     # longest prefix that is a known internal module
    while name and name not in mods:
        name = name.rpartition(".")[0]
    return name
for mod, (f, is_pkg) in sorted(mods.items()):
    pkg = mod if is_pkg else mod.rpartition(".")[0]
    try:
        tree = ast.parse(f.read_text(encoding="utf-8"), str(f))
    except (SyntaxError, UnicodeDecodeError):
        print(f"skip {f}", file=sys.stderr); continue
    for n in ast.walk(tree):
        if isinstance(n, ast.Import):
            targets = [a.name for a in n.names]
        elif isinstance(n, ast.ImportFrom):
            if n.level:                                # relative: level 1 = this package, 2 = parent ...
                p = pkg.split(".")
                if n.level - 1 >= len(p): continue
                base = ".".join(p[: len(p) - (n.level - 1)] + ([n.module] if n.module else []))
            else:
                base = n.module or ""
            # "from app import crud" -> app.crud when crud is a module, else app
            targets = [base if a.name == "*" else f"{base}.{a.name}" for a in n.names]
        else:
            continue
        for t in targets:
            to = resolve(t)
            if to and to != mod:
                print(f"{mod} -> {to}")
```

Tool-based (pydeps needs Graphviz for images):

```bash
pydeps mypkg --max-bacon 2 --cluster          # needs Graphviz; --noshow to only write the file
pydeps mypkg --show-cycles
lint-imports                                  # import-linter, if [tool.importlinter] contracts exist
```

**Go, .NET, Rust**

```bash
go list -f '{{.ImportPath}}: {{join .Imports " "}}' ./...
go mod graph                                   # module-level (third-party) requirements
dotnet list package --include-transitive       # NuGet deps; `dotnet list X.csproj reference` for project refs
cargo tree --workspace -e no-dev               # add -d to find duplicate versions
```

Go forbids import cycles between packages, so look instead at layer direction (does `internal/domain` import
`internal/postgres`?), at `internal/` boundaries, and at service-level cycles over the network.

### Grep fallback (no tool installed, or mixed-language repo)

Produce `from -> to` pairs at package granularity, then aggregate. Kotlin/Java example. Take `from` from each file's
`package` directive, never from its path: Kotlin packages need not match folders, and the Kotlin coding conventions
recommend omitting the common root package from the directory tree (`src/main/kotlin/ui/X.kt` holds
`package com.acme.ui`). Set `K` (package segments kept) and `NS` (internal namespace) for the repo:

```bash
# 1. file -> package, from the package directive
rg --no-heading --no-line-number --with-filename -o -r '$1' \
   '^package\s+([A-Za-z0-9_.]+)' -g '*.kt' -g '*.java' . > pkgmap.txt
# 2. imports, joined to the importing file's package
rg --no-heading --no-line-number --with-filename -o -r '$1' \
   '^import\s+(?:static\s+)?([A-Za-z0-9_.*]+)' -g '*.kt' -g '*.java' . \
| awk -F: -v K=4 -v NS='^com\\.acme\\.' '
    function trunc(s,   n, i, p, out) {             # first K non-empty segments
      n = split(s, p, "."); out = ""
      for (i = 1; i <= K && i <= n; i++) if (p[i] != "") out = out (out == "" ? "" : ".") p[i]
      return out
    }
    NR == FNR { pkg[$1] = $2; next }
    ($1 in pkg) {
      n = split($2, t, "."); to = ""                # drop class, static member, or *
      for (i = 1; i <= n; i++) { if (t[i] ~ /^[A-Z*]/) break; to = to (to == "" ? "" : ".") t[i] }
      from = trunc(pkg[$1]); to = trunc(to)
      if (to != "" && from != to && to ~ NS) print from " -> " to   # internal cross-package edges only
    }' pkgmap.txt - | sort | uniq -c | sort -rn > edges.txt
```

Limits: the class is detected by its leading uppercase letter, so Kotlin imports of top-level functions or properties
(`import com.acme.util.formatDate`) keep the member name as a segment; truncation to `K` hides this only when the package
already has `K` or more segments. Files in the default package (no `package` line) are skipped. Same-package references need no
import and never appear, which is what you want at this granularity.

Other languages, same pipeline with a different pattern:

| Language | Pattern (`-r '$1'`) | Note |
|---|---|---|
| Python | `^\s*(?:from\|import)\s+([\w.]+)` | Coarse: loses submodule edges (`from app import crud` yields `app`) and relative imports; use the `ast` extractor above |
| TS/JS | `from\s+['"]([^'"]+)['"]` and `require\(['"]([^'"]+)['"]\)` | Use the TS extractor above for `./`, `../` and `paths` aliases |
| Go | prefer `go list` above | |
| C# | `^using\s+([\w.]+);` | Namespaces need not match folders |

### Single-package or flat modules

The package-level graph is empty, silently, when most files share one package: same-package references need no
import. Common in KMP starters and small services (e.g. 73 of 84 Kotlin files in one package). Detect it first:

```bash
git ls-files '*.kt' '*.java' | xargs rg -I --no-line-number -o '^package\s+[\w.]+' | sort | uniq -c | sort -rn
# Python/TS: count files per top-level directory instead
```

If one package (or one build unit) holds most of the code:

1. Collapse by Gradle module and source set (`composeApp/src/commonMain`, `server/src/main`) from the paths; that is
   the module view.
2. Recover intra-module edges at file level from declarations. For Kotlin, collect top-level non-private
   declarations per file and search other files for them as whole words:

```python
# usage: git ls-files '*.kt' | python3 ktdecl.py | sort | uniq -c | sort -rn > file_edges.txt
# Edges "fileA -> fileB" when fileA mentions a top-level, non-private declaration of fileB.
import re, sys, collections
files = [l.strip() for l in sys.stdin if l.strip()]
DECL = re.compile(r"^(?:@\w+(?:\([^)]*\))?\s+)*(?:(?:public|internal|expect|actual|data|sealed|enum|abstract|open|"
                  r"inline|value|annotation|suspend|operator|infix|const|lateinit|fun)\s+)*"
                  r"(?:class|interface|object|fun|val|var|typealias)\s+(?:<[^>]*>\s*)?(?:[\w.<>?, ]+\.)?(\w+)", re.M)
STRIP = re.compile(r'"""[\s\S]*?"""|"(?:\\.|[^"\\\n])*"|/\*[\s\S]*?\*/|//[^\n]*')
src = {f: STRIP.sub(" ", open(f, encoding="utf-8", errors="replace").read()) for f in files}
owners = collections.defaultdict(set)                  # name -> files declaring it at top level (column 0)
for f, text in src.items():
    for name in DECL.findall(text):
        owners[name].add(f)
collide = {n: fs for n, fs in owners.items() if len(fs) > 1}
for n, fs in sorted(collide.items()):
    print(f"collision {n}: {' '.join(sorted(fs))}", file=sys.stderr)
for f, text in src.items():
    for tok in set(re.findall(r"\b[A-Za-z_]\w*\b", text)):
        for g in owners.get(tok, ()):
            if g != f and tok not in collide:
                print(f"{f} -> {g}")
```

   Each line is one referenced name, so `uniq -c` counts how many of B's declarations A uses. Comments and strings
   are stripped roughly. Names declared in several files (`expect`/`actual` pairs, same-named helpers in client and
   server) are reported on stderr and skipped: resolve them by hand. Confirm every edge you cite with
   `rg -nw '<name>' <file>` (level B). Private and member declarations are ignored by design.
3. Group files into elements by name or responsibility (routes, persistence, wallet, UI screens) with the §5 path
   regexes and aggregate.
4. Report the signal: the whole module is one package, so layering cannot be enforced by package rules (E4 in
   [review-playbook.md](review-playbook.md)); enforcement needs subpackages, Gradle modules, or Konsist rules on
   file names.

## 5. Collapse the file graph into an architecture graph

File-level graphs are unreadable past ~50 nodes. Map each node to an architectural element with an ordered regex
table (first match wins), aggregate, and count edges. Write the regexes against the node format your §4 tool emitted:
file paths (madge, dependency-cruiser, pydeps) or dotted package names (grep fallback, `jdeps -verbose:package`,
`go list`). Regexes of the wrong form silently map every edge to `unmapped` and give an empty graph.

| Regex on path nodes | Regex on package nodes | Element |
|---|---|---|
| `^app/src/.*/ui/` | `\.ui(\.\|$)` | `ui` |
| `^app/src/.*/(viewmodel\|presentation)/` | `\.(viewmodel\|presentation)(\.\|$)` | `presentation` |
| `^core/domain/` | `\.domain(\.\|$)` | `domain` |
| `^core/data/\|/repository/` | `\.(data\|repository)(\.\|$)` | `data` |
| `^infra/\|/(db\|http\|kafka)/` | `\.(infra\|db\|http\|kafka)(\.\|$)` | `adapters` |
| `.*` | `.*` | `unmapped` (inspect; should shrink to near zero) |

```bash
# edges.txt: "<count> <from> -> <to>" (paths or package names, as produced in §4)
# map.tsv:   "<regex>\t<element>", regexes written against those node names (unescaped | in awk)
awk 'NR==FNR { re[++n] = $1; el[n] = $2; next }
     function m(p,  i) { for (i = 1; i <= n; i++) if (p ~ re[i]) return el[i]; return "unmapped" }
     { a = m($2); b = m($4); if (a != b) w[a " -> " b] += $1 }
     END { for (e in w) print w[e], e }' FS='\t' map.tsv FS=' ' edges.txt | sort -rn
```

Reading the collapsed graph:

- **Layer order**: sort elements by fan-out minus fan-in. Top layers have high fan-out, low fan-in (UI, entry
  points); bottom layers the reverse (domain model, utilities, platform). Any edge from a lower element to a higher one
  is a candidate layering violation; its edge count tells you whether it is one stray import or a design.
- **Thin vs thick edges**: 1-3 imports across a boundary are usually fixable in an afternoon; hundreds mean the
  boundary does not exist.
- **Hubs**: one element with both high fan-in and high fan-out (`common`, `utils`, `core`, `AppState`) is a change
  amplifier; see §10.
- Draw it as Mermaid with edge weights as labels; see [documentation.md](documentation.md) for notation.

## 6. Component-and-connector evidence (runtime)

The module graph says what *compiles against* what. Runtime structure can differ sharply (one module may run as three
processes; two modules may only talk over a queue).

1. **Entry points → components.** Each deployable process, function, or app with its own `main` is a component.
2. **Connectors.** Grep and record type, direction, sync/async, and payload format:

   | Connector | Grep signals |
   |---|---|
   | HTTP/RPC out | `HttpClient`, `WebClient`, `RestTemplate`, `Retrofit`, `axios`, `fetch(`, `requests.`, `httpx.`, `http.NewRequest`, `grpc.Dial` / `grpc.NewClient` |
   | HTTP in | `@RestController`, `routing {`, `app.get(`, `router.`, `@app.get`, `APIRouter`, `http.HandleFunc`, `MapGet` |
   | Messaging | `KafkaProducer`, `@KafkaListener`, `amqp`, `channel.basicPublish`, `SQS`, `SNS`, `PubSub`, `nats.`, `publish(`, `subscribe(`, `emit(`, `on(` |
   | DB | driver/ORM names from §2 step 6; connection strings in config |
   | Streaming/sockets | `WebSocket`, `socket.io`, `SSE`, `EventSource`, Ktor `webSocket {` |
   | In-process async | `Flow`, `StateFlow`, `SharedFlow`, `Channel`, `EventBus`, `LiveData`, RxJava/RxJS `Subject`, Node `EventEmitter`, Go channels |

3. **Concurrency.** For each component: thread pools, dispatchers, event loops, worker counts. Mark shared mutable
   state touched from more than one execution context.
4. **Trace 1-3 key scenarios end to end.** Pick the scenarios tied to the top quality-attribute concerns (the
   checkout, the sync, the login). Start at the entry point and follow calls until the response leaves the system.
   At every boundary crossed (module, process, network, thread, transaction) note: file:line, connector, sync/async,
   timeout/retry present?, error handling. A trace table is itself strong level-B evidence for performance and
   availability findings (see [quality-attributes-runtime.md](quality-attributes-runtime.md)).
5. **Use runtime data when it exists.** Distributed traces or service maps (OpenTelemetry, Jaeger, Zipkin, APM
   dashboards), API gateway or ingress route tables, service mesh configs. These show real call graphs, including
   edges the code hides behind configuration.
6. **Data ownership.** Read migrations (`db/migration/`, `alembic/versions/`, `prisma/migrations/`, `migrations/`)
   and schema files. Map each table to the module or service that writes it. Tables written by two services are a
   shared-database integration, whatever the diagrams say.

## 7. Allocation view

| Recover | Where |
|---|---|
| Deployment units | Dockerfiles (`COPY`/`CMD` show which module ships), Helm/k8s `Deployment`s, serverless functions, mobile/desktop artifacts |
| Module → artifact | Build files (`application { mainClass }`, `jar`/`shadowJar`, `bin`, `[project.scripts]`, `cmd/*`), Dockerfile build stages. A multi-stage `COPY --from=<stage>` that embeds one workspace's output in another's package (an SPA built into `backend/app/frontend`) is a build/deploy edge: record it as module -> artifact even though no import exists |
| Environments | Compose profiles, Helm values per env, Terraform workspaces, `.env.*`, CI deploy jobs |
| Config and secrets | Env vars read in code (`System.getenv`, `process.env`, `os.environ`, `os.Getenv`), config files, secret managers (Vault, AWS/GCP secret managers, k8s `Secret`), hard-coded values (a finding) |
| CI stages | Build → test → scan → package → deploy; which checks gate merge; whether architecture tests run |
| Ownership | `CODEOWNERS`, `git shortlog` per directory (§11) |

Note modules that ship in several artifacts and artifacts assembled from several teams' modules (deployability).

## 8. Reflexion model

Murphy, Notkin and Sullivan (1995) compare a hypothesized high-level model with the source via an explicit mapping:

1. **Hypothesized model**: the intended elements and allowed edges (from docs, ADRs, architecture tests, or the team).
2. **Mapping**: path regexes → elements (the §5 table).
3. **Compute**: collapse extracted edges through the mapping and compare.

| Result | Meaning | Action |
|---|---|---|
| **Convergence** | Edge in model and in code | Record; it is an allowed edge in the rule set the fitness function enforces |
| **Divergence** | Edge in code, not in model | Finding; cite edge count and 1-3 example file:line |
| **Absence** | Edge in model, not in code | Either the model is wrong or a feature bypasses the intended path; investigate |

Iterate: when a divergence is a modelling mistake, refine the model or mapping, and say which you changed.

```mermaid
graph TD
    UI[ui] --> VM[presentation]
    DOM[domain] --> DATA[data]
    DATA --> AD[adapters]
    UI -. "divergence: 42 imports" .-> DATA
    DOM -. "divergence: 7 imports" .-> AD
    VM -. "absence" .-> DOM
    linkStyle 3,4 stroke:#c0392b,stroke-width:2px
    linkStyle 5 stroke:#999,stroke-dasharray:4
```

Present intended and actual side by side (two subgraphs) when the audience needs to see the whole debt at once.
Encode the hypothesized model's allowed-dependency rules as an architecture test, with today's divergences recorded
as a frozen baseline (ArchUnit `FreezingArchRule`, dependency-cruiser known violations, import-linter
`ignore_imports`; see [fitness-functions.md](fitness-functions.md)), so the divergence count can only go down. A test
that only asserts existing convergences catches no new divergence.

## 9. Style detection

Signals suggest a style; confirm against the edges ([architectural-patterns.md](architectural-patterns.md)).

| Code signal | Probable style | Confirm by |
|---|---|---|
| `ports/`, `adapters/`, `application/`, `domain/`; interfaces in domain, implementations in adapters | Ports and adapters (hexagonal) / clean | Domain imports no framework or adapter package |
| `controllers/`, `services/`, `repositories/` | Layered (often "by-layer" packaging) | No upward edges; layer-skipping edges (controller → repository) are documented bridges if relaxed layering was chosen, otherwise violations |
| `features/<name>/` each with its own ui/data | Feature-sliced / vertical slices | Cross-feature imports are rare and go through public APIs |
| `EventBus`, `publish/subscribe`, broker clients, `@EventListener` | Event-driven / publish-subscribe | Publishers do not know subscribers; check for request/reply hidden in events |
| `ServiceLoader`, plugin dirs, `entry_points`, dynamic `import()` / `require` of paths from config | Microkernel / plug-in | A stable core API; plug-ins depend on core only |
| Folders per business capability, each with `api` + `internal` subpackages or modules | Modular monolith | Cross-module imports only hit `api` |
| Several deployables, each with own DB schema, talking over HTTP/queues | Microservices / service-based | Independent deploy pipelines; no shared tables |
| Several services, one schema/DB | Distributed monolith (shared database) | Migration ownership; cross-service joins |
| `Pipeline`, `Stage`, stream operators, shell-pipe-style chains, Beam/Flink jobs | Pipe-and-filter | Filters share no state |
| One `Store`, reducers, actions, unidirectional flow | Redux/MVI-style | Views never mutate state directly |
| Terraform/Helm/Compose placing deployables in separate networks or subnets | Multi-tier | Which tier can reach which (security groups, network policies) |
| MVC/MVP/MVVM naming in UI code | UI patterns | ViewModel has no reference to views; see [design-patterns-in-code.md](design-patterns-in-code.md) |

Mixed signals are normal. Report the dominant style per subsystem, not one style for the repo.

## 10. Metrics: where to look, not verdicts

Metrics point your reading. Never report a number as a finding on its own; pair it with the code it points to and
the quality attribute it threatens.

### Martin's package metrics

| Metric | Definition | Read as |
|---|---|---|
| **Ca** (afferent) | Elements outside that depend on this one | Responsibility: many dependants make change costly |
| **Ce** (efferent) | Elements outside this one depends on | Dependence: many dependencies make it fragile |
| **I** (instability) | Ce / (Ca + Ce), 0 = stable, 1 = unstable | Stable Dependencies Principle: depend in the direction of decreasing I |
| **A** (abstractness) | Abstract types / total types in the element | Stable Abstractions Principle: stable elements should be abstract |
| **D** (distance) | \|A + I − 1\| | Distance from the main sequence |

**Zone of pain** (A≈0, I≈0): concrete and heavily depended on, so painful to change (a concrete `utils` everyone
imports, a shared DB schema). **Zone of uselessness** (A≈1, I≈1): abstractions nobody depends on. Martin counts
classes; at module granularity count modules and say so.

```python
# usage: python3 martin.py < edges.txt   (lines: "[count] from -> to")
import sys
from collections import defaultdict
ca, ce, mods = defaultdict(set), defaultdict(set), set()
for line in sys.stdin:
    p = line.split()
    if "->" not in p: continue
    k = p.index("->"); a, b = p[k - 1], p[k + 1]
    mods |= {a, b}
    if a != b: ce[a].add(b); ca[b].add(a)
rows = []
for m in mods:
    n_a, n_e = len(ca[m]), len(ce[m])
    rows.append((n_e / (n_a + n_e) if n_a + n_e else 0.0, m, n_a, n_e))
for i, m, n_a, n_e in sorted(rows):
    print(f"{m:40} Ca={n_a:3} Ce={n_e:3} I={i:.2f}")
```

Abstractness caveats: Go interfaces are satisfied implicitly and usually declared by the consumer, so A per package
is misleading; in TypeScript count interfaces and abstract classes as abstract, but decide whether data-shape
interfaces and `type` aliases (DTOs) count, since they inflate A, and note that structural typing lets
implementations skip importing the interface, so Ca of interface-only modules is undercounted; Kotlin `sealed` hierarchies and Rust traits need a decision on how to count.
State your counting rule or skip A and D.

### Cycles

Compute strongly connected components on the collapsed graph (`networkx.strongly_connected_components` over the §5
output). madge `--circular` and pydeps `--show-cycles` report file/module-level cycles; map their members through §5
before reporting. Report each SCC larger than one element with its members and
the thinnest edge to cut. Cycles between modules defeat independent build, test and release.

### Size and complexity

| Tool | Gives | Example |
|---|---|---|
| `scc`, `tokei`, `cloc` | LOC per language/dir; `scc` also a complexity estimate | `scc --by-file -s complexity src` |
| `lizard` | Cyclomatic complexity (CCN), function length; many languages | `lizard -w -C 15 src/` (warnings above CCN 15) |
| `radon` | Python CC and maintainability index | `radon cc -s -a mypkg`, `radon mi mypkg` |
| `gocyclo` | Go CC | `gocyclo -over 15 .` |
| `detekt` | Kotlin complexity and style rules | Gradle plugin; read its report's complexity section |

Thresholds are heuristics: McCabe suggested 10 per function; many teams flag 15-20; files over ~500-1000 lines in a
hotspot deserve a look. Complexity only matters where code changes (§11).

### Cohesion signals

- **Internal/external edge ratio** per element: imports that stay inside ÷ imports that leave. A low ratio suggests
  the boundary cuts through a concept.
- **LCOM** variants disagree with each other and punish data classes and DTOs; use only as a rough pointer to classes
  that are really two classes.
- **Co-change** (§11) is the strongest cohesion signal: files that always change together belong together.

### Connascence (Page-Jones)

A vocabulary for coupling strength. Static forms, weaker to stronger: name, type, meaning (magic values), position,
algorithm. Dynamic forms: execution order, timing, value, identity. Rules: prefer weaker forms, keep strong forms
local (inside a module), and reduce degree (how many elements share it). Use it to explain *why* a cross-module edge
is dangerous ("both services must agree on the meaning of status code 7").

## 11. Git history (behavioural analysis)

Tornhill's point: the code that matters is the code that changes. Combine history with structure.

**Churn** (commits touching each file, last 12 months):

```bash
git log --since=12.month --format=format: --name-only | grep -v '^$' | sort | uniq -c | sort -rn | head -50
```

12 months is the default window (the report template assumes it). Shorten to 3-6 months for fast-moving repos, and
state the window in the output block.

**Hotspots** = churn × size or complexity. Join the churn list with `scc --by-file` or `lizard` output; the top 10 by
product are where modifiability findings pay off first. A complex file nobody touches is low priority.

**Change coupling** with code-maat (JAR name and version vary by release; build or download per its README):

```bash
git log --all --numstat --date=short --pretty=format:'--%h--%ad--%aN' --no-renames --after='12 months ago' > git.log
java -jar code-maat-<version>-standalone.jar -l git.log -c git2 -a coupling          # degree, avg revs
java -jar code-maat-<version>-standalone.jar -l git.log -c git2 -a revisions
java -jar code-maat-<version>-standalone.jar -l git.log -c git2 -a entity-ownership
```

code-maat can aggregate files into architectural elements with `-g groups.txt`, one `<path-prefix or ^regex$> =>
<element>` per line (regex support depends on the version; check its README). Convert the §5 path-form mapping into
that format; `map.tsv` cannot be passed as-is.

**Pure git/awk co-change fallback** (pairs of files committed together; skips huge commits):

```bash
git log --since=12.month --no-merges --format='@@%h' --name-only \
| awk '
  function flush(   i, j, k) {
    if (n > 1 && n <= 30)
      for (i = 1; i <= n; i++) for (j = i + 1; j <= n; j++) {
        if (f[i] < f[j]) k = f[i] " <-> " f[j]; else k = f[j] " <-> " f[i]
        pair[k]++
      }
    n = 0
  }
  /^@@/ { flush(); next }
  NF    { f[++n] = $0 }
  END   { flush(); for (k in pair) print pair[k], k }' \
| sort -rn | head -30
```

Change coupling across module boundaries (or across services/repos) with no static edge between them is hidden
coupling: shared assumptions, duplicated logic, or a protocol in two places. It is often the best finding in a review.

**Ownership**: `git shortlog -sn HEAD -- path/to/module` (pass `HEAD`; without a revision, shortlog reads stdin when
not on a terminal). Many small contributors on a hotspot = coordination cost; one contributor = bus-factor risk.

Caveats:
- Squash merges collapse many changes into one commit: coupling looks stronger, churn lower.
- Bulk reformat, license-header, and mass-rename commits inflate churn; exclude them (`--invert-grep --grep=...` or
  by SHA) or cap commit size as the awk script does.
- Renames split a file's history unless you follow them; code-maat recommends `--no-renames`, so moved files appear as
  new ones. Check `git log --follow` for key files.
- Exclude generated and vendored files (`build/`, `dist/`, `*.lock`, `*_pb.*`, `generated/`) before ranking.
- **Shallow clone** (`git rev-parse --is-shallow-repository` prints `true`): history is truncated. Ask before
  `git fetch --unshallow`; otherwise write "not checked: shallow clone" for churn, hotspots, co-change and ownership,
  and use file size only as a pointer for where to read.
- **Young (< ~3 months) or single-author repo**: commit counts carry little signal. Use the whole history, weight by
  lines changed rather than commits, and read commit-message themes as fragility signals:

  ```bash
  git log --no-merges --numstat --format= | awk 'NF==3 && $1!="-" {c[$3]+=$1+$2} END {for (f in c) print c[f], f}' \
  | sort -rn | head -30                                              # lines changed per file, whole history
  git log --no-merges --format=%s | rg -i -o 'race|retry|fallback|timeout|flaky|revert|workaround|hotfix|deadlock' \
  | tr A-Z a-z | sort | uniq -c | sort -rn                           # recurring themes
  git log --no-merges --format='%h %s' -i -E --grep='race|retry|fallback|revert' --name-only   # which files
  ```

  Files that recur under "fix race", "retry" or "fallback" commits are fragile areas; read them first. State the
  reduced confidence in the report, and skip ownership analysis for a single author (bus factor is 1 by definition).

## 12. From metrics to quality-attribute claims

| Observation | Tentative claim | QA (see) | Confirm before reporting |
|---|---|---|---|
| Upward edges / skip-layer edges in collapsed graph | Layering not enforced; changes ripple across layers | Modifiability, testability ([quality-attributes-change.md](quality-attributes-change.md)) | Open 2-3 edges; check they are production code |
| SCC spanning modules | Modules cannot be built, tested or released independently | Modifiability, deployability | Find the thinnest edge to cut |
| High Ca, low A (zone of pain) | Every change to it is expensive | Modifiability | Check its churn: pain only if it changes |
| Hotspot: high churn × high complexity | Defect-prone, slow to change | Modifiability, reliability | Read the file; check bug-fix commit share |
| Cross-module co-change without static edge | Hidden coupling (shared meaning/format) | Modifiability, integrability | Identify the shared assumption |
| Domain imports framework/DB/HTTP types | Domain not portable or unit-testable | Testability, portability | Confirm in domain source set, not tests |
| Two writers on one table / one schema, many services | Shared-DB integration | Modifiability, availability, deployability | Migration ownership |
| Synchronous chain of 3 or more remote hops on one request path in a key scenario (R2) | Latency and availability multiply along the chain | Performance, availability ([quality-attributes-runtime.md](quality-attributes-runtime.md)) | Timeouts, retries, circuit breakers present? |
| Global mutable singletons accessed from many threads | Races, hidden coupling, untestable | Reliability, testability | Access paths and synchronisation |
| Secrets in code or config committed to git | Credential exposure | Security | `git log -S` for the value |
| One-author hotspot | Knowledge concentration | (Organisational risk) | Ownership share over time |

## 13. Output block

Paste into [../templates/architecture-review-report.md](../templates/architecture-review-report.md) or design notes; write "not checked", never blank.

```markdown
### As-is architecture (recovered)
- Commit: `<sha>` · Working tree: clean | N uncommitted files (reviewed at HEAD) · History: full | shallow | N commits
- Depth: quick | pass | deep · Tools: <static settings.gradle parse, pyedges.py, madge 8.x, ...> · Read-only: yes | no
- Excluded from the file universe: <.claude/worktrees/, build/, kotlin-js-store/, generated/, ...>
- Build units: <list> · Entry points: <file:line, ...> · Composition root: <file:line or "implicit: N singletons">
- External systems: <name → connector → module> · Data stores: <store → owning module(s)>
- Deployment units: <artifact ← modules> · Environments: <list> · Config/secrets: <sources>
- Dominant style(s): <style per subsystem, with confirming evidence>
- Module view: <Mermaid collapsed graph, edge counts as labels>
- Layer order (by fan-in/fan-out): <top → bottom>
- Reflexion vs <intended model source>: convergences N · divergences N (top 3 with counts + file:line) · absences N
- Cycles: <SCCs with members>
- Hotspots (churn × complexity, window: <12 months>): <top 5 files> or "not checked: shallow clone"
- Hidden coupling (co-change without static edge): <top pairs>
- Unverified (level C/D) claims: <list>
```

## 14. Pitfalls

- **Judging by folder names.** `domain/` that imports Spring, `repositories/` that render HTML. Names are level-C
  evidence; the edges decide.
- **Edges the static graph misses.** Reflection, DI containers (Spring component scan, Hilt, Koin `get()`),
  `ServiceLoader`, annotation processing/KSP/kapt output, Python `importlib`, JS dynamic `import()` and `require` of
  computed paths, Go `plugin`, string-keyed event buses, HTTP calls to your own services. Grep for these and add the
  edges by hand, marked as such.
- **Edges the static graph over-counts.** Test-only dependencies (`testImplementation`, `devDependencies`,
  `*_test.go`, `tests/`), type-only imports (`import type` in TS), and doc/sample code. Exclude tests from the
  production graph; analyse them separately.
- **Generated code.** Protobuf/OpenAPI clients, Room/SQLDelight/Prisma output, Compose resources. Include as a node
  if it is a real boundary; exclude from size and churn rankings.
- **Build variants and targets.** Android flavors/build types, KMP targets, Go build tags, TS `paths` per project,
  feature flags. An edge may exist only in one variant; name the variant you analysed.
- **Monorepo scope.** Tools run from a sub-package see only that package; run from the root or per package and
  merge.
- **Treating metrics as verdicts.** High instability is correct for UI and entry points; zone of pain is fine for a
  stable standard library. Always tie a number to a quality-attribute scenario before it becomes a finding.
- **Describing intent as fact.** If the only source is a README or diagram, the claim is level C, however confident it
  sounds.

Structure vocabulary (§1) follows SAiP's module / C&C / allocation categories. Theory adapted in part from the
Wikipendium TDT4240 compendium (CC BY-SA 3.0); see [../CREDITS.md](../CREDITS.md).
