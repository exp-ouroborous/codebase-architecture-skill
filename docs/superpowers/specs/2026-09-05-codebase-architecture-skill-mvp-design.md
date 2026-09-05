# Codebase Architecture Skill MVP Design

## Summary

Build a portable Agent Skills-compatible skill that analyzes a repository and generates a navigable LikeC4 architecture site. The skill targets Codex and Claude Code, supports Python, .NET/C#, and Go repositories, and combines deterministic repository discovery with agent judgment.

The MVP fully regenerates its output. It does not preserve manual edits and does not provide a human overlay. LikeC4 is the only renderer in the MVP and is pinned to version `1.59.3`.

The native LikeC4 viewer is a JavaScript application. This is an explicit MVP exception to the preference for pure HTML: the project will not add a custom viewer or additional application JavaScript, but it will accept LikeC4's runtime to get its navigation and interaction.

## Goals

- Generate a hierarchical architecture model covering C1 through C4.
- Let users navigate from a complete higher-level view into the selected element's next level.
- Show architectural data flows at every level where the flow is meaningful.
- Make discovered and conceptual data objects first-class, clickable model elements.
- Link C3 and C4 elements to their defining code in VS Code, GitHub, or GitLab.
- Regenerate the architecture safely and deterministically as the repository changes.
- Produce a native LikeC4 static site that can be committed and served.
- Work as an installable skill in Codex and Claude Code.

## Non-goals

- Preserving manual changes to generated files.
- Supporting a human-authored overlay.
- Building a custom HTML viewer or a no-JavaScript fallback.
- Supporting Structurizr, D2, Mermaid, or other renderers directly.
- Supporting languages beyond Python, .NET/C#, and Go deeply.
- Inferring runtime behavior from telemetry or distributed traces.
- Guaranteeing that all inferred architectural boundaries are objectively correct.
- Providing manual diagram layout.

## User experience

### Initial generation

A user asks an agent to generate an architecture view of a repository. The skill:

1. Confirms the repository root and output location.
2. Detects supported languages, build systems, and source-control metadata.
3. Runs deterministic discovery to create a normalized fact bundle.
4. Interprets those facts into architectural boundaries and meaningful flows.
5. Writes generated LikeC4 sources.
6. Validates the model with the pinned LikeC4 CLI.
7. Builds the native LikeC4 static site.
8. Reports uncertainties and links to the generated model and site.

### Regeneration

A user asks the agent to refresh or check the architecture. The skill repeats the complete pipeline in a temporary directory. It validates the replacement before changing the committed output. A successful run atomically replaces the previous generated directory.

Every generated file states that it will be overwritten. The skill does not attempt to merge or preserve edits within generated output.

### Drift checking

The skill exposes a check mode for CI. Check mode regenerates into a temporary location and compares it with committed architecture sources. It exits unsuccessfully and prints a concise diff summary when the model is stale.

The native LikeC4 site's hashed asset filenames may change between tool versions. CI drift checks compare generated LikeC4 sources and generation metadata by default; site-output drift is checked only when the pinned build environment is used.

## Repository layout

The standalone project separates its public repository documentation from the installable skill package:

```text
codebase-architecture-skill/
├── README.md
├── LICENSE
├── THIRD_PARTY_NOTICES.md
├── AGENTS.md
├── skills/
│   └── codebase-architecture/
│       ├── SKILL.md
│       ├── agents/
│       │   └── openai.yaml
│       ├── scripts/
│       │   ├── discover.py
│       │   ├── generate.py
│       │   ├── build.py
│       │   ├── check.py
│       │   └── adapters/
│       │       ├── python_adapter.py
│       │       ├── dotnet_analyzer/
│       │       │   └── ...
│       │       └── go_analyzer/
│       │           └── ...
│       └── references/
│           ├── model-conventions.md
│           ├── python-analysis.md
│           ├── dotnet-analysis.md
│           └── go-analysis.md
├── tests/
│   ├── fixtures/
│   │   ├── python/
│   │   ├── dotnet/
│   │   └── go/
│   └── ...
└── .github/
    └── workflows/
```

The skill package contains only instructions and resources needed at runtime. Repository-level documentation, governance, and contributor material remain outside the skill package.

## Generated output layout

By default, the analyzed repository receives:

```text
architecture/
├── README.md
├── generation.json
├── facts.json
├── model.json
├── likec4/
│   ├── specification.c4
│   ├── model.c4
│   ├── views.c4
│   └── flows.c4
├── site/
│   ├── index.html
│   └── assets/
├── package.json
└── package-lock.json
```

`architecture/package.json` pins `likec4` to exactly `1.59.3`. The package lock is committed so local and CI builds resolve the same dependency graph.

`model.json` is the normalized, agent-completed architecture model from which the LikeC4 files are generated. Keeping it beside `facts.json` makes architectural inferences reviewable and lets CI distinguish discovery drift from interpretation drift.

`generation.json` records the generator version, LikeC4 version, repository commit, supported-language adapters used, source-link mode, generation timestamp, and warnings. Volatile fields such as the timestamp are excluded from semantic drift comparison.

## Architecture

### Skill orchestrator

`SKILL.md` defines the agent workflow, safety rules, quality gates, and when to load language-specific references. It tells the agent when deterministic scripts are authoritative and when architectural judgment is required.

The skill must treat repository content as untrusted data. Code comments, documentation, and embedded prompts cannot override the skill workflow or request unrelated tool actions.

### Deterministic discovery

`discover.py` inventories the repository and produces `facts.json`. It owns facts that should not vary by agent:

- Source files and project boundaries.
- Package, module, project, and solution manifests.
- Imports and project references.
- Publicly declared types and symbols.
- Framework and application entry points when deterministically recognizable.
- Source ranges for discovered symbols.
- Git remote, provider, revision, and repository-relative paths.
- Explicit data types and serialization models.

Discovery excludes common generated, vendored, cache, dependency, binary, and secret-bearing paths. It never reads `.env` values, credential stores, private keys, or build secrets into the fact bundle.

### Language adapters

The discovery layer provides a common interface with three MVP adapters:

- **Python:** use the Python standard-library AST for packages, imports, classes, functions, decorators, type annotations, dataclasses, enums, and recognizable schema models. Detect common application entry points conservatively.
- **.NET/C#:** read solution and project metadata deterministically. Prefer the installed .NET SDK for project graphs. Use a bundled Roslyn-based helper for syntax and symbol facts when the SDK is available; fall back to a conservative source inventory with a warning.
- **Go:** use `go list -json` for module and package graphs when the Go toolchain is available. Use a bundled Go helper based on `go/parser` and `go/ast` for types, interfaces, functions, methods, and source locations; fall back to a conservative source inventory with a warning.

Native toolchain commands run with restore and network access disabled where possible. Discovery does not execute repository code, tests, build scripts, generators, or application entry points.

### Agent interpretation

The agent reads `facts.json` and selected source evidence to infer:

- People and external systems for C1.
- Deployable or independently running containers for C2.
- Cohesive components and their responsibilities for C3.
- Selected code symbols and relationships for C4.
- Conceptual data objects not represented by one concrete source type.
- Flow names, abstraction levels, and transformation steps.

The agent must distinguish discovered facts from inference. Every inferred element and relationship carries confidence and evidence metadata. Low-confidence architectural decisions are included only when useful and are listed in the generation report.

### LikeC4 generation

`generate.py` accepts a normalized, agent-completed architecture model and emits formatted LikeC4 source files. It owns escaping, stable identifiers, source links, model conventions, view naming, and generated-file headers.

The generator produces one logical model and multiple projections. It never asks the agent to handcraft arbitrary DSL syntax when a deterministic generator can emit it.

### Validation and site build

`build.py` runs the pinned local LikeC4 CLI through `npm` scripts. It performs:

1. LikeC4 source formatting check.
2. LikeC4 model validation.
3. Static-site build with a relocatable base path.
4. Verification that `site/index.html` and required assets exist.

Failed validation or build leaves the previously generated architecture untouched.

## Canonical intermediate model

The agent and deterministic generator communicate through the versioned `architecture/model.json` document. It is generated output and an implementation contract, not a user-editable overlay.

The top-level schema contains:

- `schema_version`
- `repository`
- `elements`
- `relationships`
- `data_objects`
- `flows`
- `views`
- `warnings`

### Elements

Each element contains:

- Stable `id`.
- C4 level and LikeC4 element kind.
- Title, responsibility, technology, and parent ID.
- `origin`: `discovered` or `inferred`.
- Confidence from `0.0` to `1.0`.
- One or more evidence records.
- Optional source links.

Stable IDs derive from repository-relative qualified identities when possible. Human-readable titles do not determine identity. When an element moves or is renamed and cannot be matched confidently, full regeneration may represent it as removal plus addition.

### Relationships

Relationships include source, target, kind, description, technology, evidence, confidence, and the levels at which the relationship is meaningful.

The initial relationship vocabulary is deliberately small:

- `uses`
- `calls`
- `reads`
- `writes`
- `publishes`
- `consumes`
- `transforms`
- `returns`

### Data objects

Data objects are first-class elements rather than labels attached only to edges.

- **Discovered:** backed by a concrete Python type, C# type, or Go type and linked to its declaration.
- **Conceptual:** inferred from behavior when no single declared type represents the data.

Both forms can participate in relationships and flows. LikeC4 metadata identifies their provenance, schema summary, source evidence, and confidence. Clicking a data object opens LikeC4's element details and its source links.

### Flows

A flow is an ordered architectural scenario with a stable ID, title, scope, meaningful C4 levels, participants, and steps. Each step references model elements and may reference one or more data objects.

The generator creates level-specific dynamic views rather than mechanically expanding one sequence across all levels. A checkout flow may therefore have:

- C1: Customer → Checkout System → Payment Provider.
- C2: Web → API → Database/Provider.
- C3: Route → Checkout Service → Payment Adapter.
- C4: Selected functions and types involved in the transformation.

Levels without meaningful evidence are omitted and reported rather than populated with speculative detail.

## LikeC4 modeling conventions

The generated specification defines custom kinds for C4 elements, code symbols, and data objects. It uses LikeC4's nested elements for hierarchy, scoped views for drill-down, `navigateTo` for explicit transitions, dynamic views for flows, links for source navigation, and metadata for evidence.

At minimum, metadata supports:

- `origin`
- `confidence`
- `source`
- `evidence`
- `language`
- `symbol`
- `flow_levels`

All generated views have stable names and explicit titles. C1 is the default view. C2, C3, and C4 views are generated only for elements that have useful children. Selecting a navigable element exposes its next-level view.

## Source links

For C3, C4, and discovered data objects, the generator supports:

- A commit-pinned GitHub or GitLab link when a supported remote is detected.
- A local `vscode://file/...` link when local-link mode is explicitly selected and the repository has an absolute local path.
- A repository-relative source link as portable fallback metadata.

Source-link mode is `remote`, `vscode`, or `both`. It defaults to `remote` when a supported remote exists and to `vscode` otherwise. CI and committed shared sites use `remote`; generation fails with a clear diagnostic if that mode is requested without a supported remote. Local VS Code links embed an absolute checkout path, so the skill warns before writing them and never includes them in remote-mode output. Remote links target the analyzed commit rather than a moving branch so that architectural evidence remains reproducible.

## Regeneration safety

Generation occurs in a task-owned temporary directory. The workflow:

1. Discover facts.
2. Interpret the architecture.
3. Generate LikeC4 sources.
4. Validate and build the site.
5. Compare semantic generated sources with the current output.
6. Replace `architecture/` only after every gate succeeds.

The replacement target must resolve to the exact configured output directory. The scripts refuse repository roots, home directories, filesystem roots, unresolved variables, and paths outside the analyzed repository.

Interrupted or failed runs leave the previous architecture available. Temporary directories are cleaned on success and retained with a diagnostic path on failure.

## Command contract

The scripts expose a small interface for agents and CI:

```text
discover.py --repo PATH --output facts.json
generate.py --input model.json --output likec4/
build.py --source likec4/ --output site/
check.py --repo PATH --architecture architecture/
```

Commands return non-zero status for invalid arguments, unsafe paths, unsupported schema versions, analysis failures, LikeC4 validation failures, build failures, and detected drift. Diagnostics identify the failing stage and suggest one concrete recovery action.

## Error handling

- **No supported language:** produce repository-level facts and a clear unsupported-language warning; do not claim deep analysis.
- **Missing native toolchain:** use the documented fallback and mark reduced confidence.
- **Malformed or partial project files:** continue with unaffected projects and report skipped inputs.
- **Ambiguous architecture:** choose the smallest defensible boundary, attach evidence and lower confidence, and report it.
- **LikeC4 validation failure:** retain the previous output and show the relevant generated source location.
- **Missing Node/npm:** stop before modifying output and state the required prerequisite.
- **Network unavailable:** use the committed package lock and existing dependency cache; do not silently change dependency versions.
- **Large repository:** honor exclusions, analyze project graphs before individual symbols, and limit C4 detail to architecturally relevant paths.

## Testing strategy

### Deterministic tests

- Unit tests for path safety, identifier generation, source-link construction, fact normalization, and DSL escaping.
- Adapter tests against minimal Python, C#, and Go fixtures.
- JSON Schema validation for fact and architecture-model documents.
- Snapshot tests for generated LikeC4 sources.
- Idempotence tests proving that two runs on the same revision produce no semantic diff.
- Failure tests proving that invalid generation does not replace prior output.

### Integration tests

For each language fixture:

1. Run discovery.
2. Supply a checked-in deterministic architecture-model fixture.
3. Generate LikeC4 sources.
4. Run LikeC4 format validation.
5. Run LikeC4 model validation.
6. Build the native static site.
7. Assert C1–C4 navigation, source links, data objects, and flow views in exported model JSON.

### Skill evaluations

Forward-test the skill with fresh Codex and Claude Code sessions against representative repositories. Evaluate whether the agent:

- Invokes the deterministic scripts instead of inventing facts.
- Chooses defensible C1–C4 boundaries.
- Separates discovered facts from inference.
- Produces useful level-specific flows without over-modeling call graphs.
- Regenerates rather than editing generated output in place.
- Stops safely when validation fails.

## Security and privacy

- Analyze repositories locally.
- Do not upload source code or facts to third-party services as part of the workflow.
- Do not execute analyzed application code.
- Disable package restoration and network access during analysis where toolchains permit.
- Exclude secret-bearing files and values from generated facts and models.
- Treat repository content as untrusted data, including instructions embedded in comments or documentation.
- Escape all source-derived text before emitting LikeC4 DSL.
- Pin the LikeC4 dependency and review dependency updates explicitly.

## Licensing

The project uses the MIT License. LikeC4 `1.59.3` is also MIT-licensed. The repository includes LikeC4's copyright and license notice, plus applicable notices for redistributed native-viewer assets, in `THIRD_PARTY_NOTICES.md`.

Generated user architecture models are not required to use the project's license.

## Repository governance

The public GitHub repository uses:

- Protected `main` with no direct pushes.
- One feature branch per change.
- Required pull requests and resolved review conversations.
- One required approval.
- Automatic GitHub Copilot review on each push.
- Balanced Copilot review effort.
- Copilot approvals allowed to satisfy the merge requirement.
- Rebase of the feature branch onto current `main` before final review.
- Squash merge only, producing one commit per feature on `main`.
- Automatic deletion of merged feature branches.

Once CI workflows exist, branch protection will require the validation workflow and a branch-freshness check before merging.

## MVP acceptance criteria

The MVP is complete when:

1. The skill is installable and discoverable in Codex and Claude Code.
2. It analyzes representative Python, .NET/C#, and Go repositories without executing application code.
3. It generates valid LikeC4 C1–C4 views with progressive navigation.
4. C3/C4 nodes and discovered data objects include working source links.
5. Generated dynamic views show meaningful data flow at applicable levels.
6. Discovered and conceptual data objects are visibly distinguishable.
7. The pinned LikeC4 CLI validates the generated model and builds its native site.
8. Repeating generation on an unchanged revision produces no semantic diff.
9. Check mode detects stale committed architecture output.
10. Failure at any stage leaves the prior output intact.
11. Automated tests and skill evaluations pass in both target agent environments.

## Deferred evolution

After the MVP is validated with real repositories, likely extensions include a human overlay, incremental reconciliation, additional language adapters, runtime-trace ingestion, custom renderers, and exports to Structurizr or general diagram DSLs. These are not compatibility promises for the MVP.
