# FlowForge

Inspectable workflow automation for repeatable local and team tasks.

> Planning repository — documentation and Git initialization only. No application, package, solution, database, tests, CI workflow, or runnable demo has been implemented yet. Everything below describes the proposed implementation, unless explicitly marked as verified.

## Goals

Give developers and operations teams a small, understandable way to compose multi-step jobs, run them, and explain failures. The distinguishing feature is a versioned workflow definition with an inspectable execution history, rather than a collection of opaque scripts.

## Target users

Developers automating repository chores, small teams moving data between services, and operators who need repeatable scheduled tasks.

## MVP scope

- Versioned JSON workflow definitions with named steps, typed inputs, dependencies, and validation before execution.
- Directed acyclic graph execution with bounded parallelism, cancellation, timeouts, and explicit failure propagation.
- Built-in HTTP request, JSON transform, and file-output steps; restrict destinations and filesystem roots.
- Manual execution and a basic schedule trigger, run history, per-step logs, and retry controls.
- Local secrets referenced by key; redact configured values from logs and exported runs.

## Stretch goals

- Visual graph editor and reusable subworkflows.
- Approval steps, authenticated webhooks, and import/export bundles.
- Plugin isolation and multi-worker execution after a single-worker engine is reliable.

## Proposed tech stack

C# / .NET 10, ASP.NET Core API, Blazor web interface, EF Core with SQLite for the first local deployment, xUnit. Consider PostgreSQL only when concurrent team use justifies it.

Select compatible stable dependency versions during milestone 1, record them in project files, and pin the SDK/toolchain after the compatibility spike. No packages or development workloads are installed by this documentation task.

## Architecture

Domain owns graph validation and state transitions. Application orchestrates runs through IStepHandler implementations. Infrastructure persists definitions, runs, and artifacts. A hosted worker claims runnable steps; the web host queues work rather than executing steps inside requests. Workflow versions are immutable once used. Dependencies flow inward to Domain, and handlers receive bounded execution contexts.

## Proposed folder structure

Only README.md and .gitignore exist initially, alongside Git metadata. The following tree is a plan; create directories when their implementation begins.

```text
FlowForge/
  README.md
  .gitignore
  FlowForge.slnx
  src/FlowForge.Domain/
  src/FlowForge.Application/
  src/FlowForge.Infrastructure/
  src/FlowForge.Api/
  src/FlowForge.Web/
  tests/FlowForge.UnitTests/
  tests/FlowForge.IntegrationTests/
  samples/workflows/
  docs/decisions/
```

## Data models and invariants

WorkflowDefinition(Id, Version, Name, Inputs, Steps); StepDefinition(Id, Handler, DependsOn, RetryPolicy, Timeout); WorkflowRun(Id, DefinitionVersion, State, StartedAtUtc); StepRun(RunId, StepId, Attempt, State, ErrorCode). State transitions: Pending -> Running -> Succeeded | Failed | Cancelled. Retries create new attempts without mutating earlier logs.

## Proposed API and contracts

POST /api/workflows validates and creates a version; POST /api/workflows/{id}/runs accepts input and an idempotency key; GET /api/runs/{id} returns run and step status; POST /api/runs/{id}/cancel requests cooperative cancellation. Return validation paths for malformed graphs and 409 for incompatible state changes.

Version persisted/exported formats from their first implementation. Validate at boundaries and return structured errors rather than leaking internal exceptions. The contracts above remain proposals until backed by code and contract tests.

## Implementation milestones

1. Define JSON schema and graph invariants; reject cycles, missing dependencies, and unknown handlers.
2. Build pure execution engine with fake clock/handlers; demonstrate deterministic failure and cancellation.
3. Persist runs and recover interrupted work; explicitly classify indeterminate external side effects.
4. Add API and history UI; ship three runnable workflow examples and a demo walkthrough.

Each milestone should end with a small demonstrable result, a written acceptance check, and tests for its important failure modes. Work through a vertical slice before expanding the stretch scope.

## Testing and CI plan

Test diamond graphs, cycles, retry exhaustion, cancellation races, crash recovery, secret redaction, and concurrent claims. Integration tests use temporary databases and a local HTTP fake, never production endpoints. CI should restore, format-check, build, run unit/integration tests on Windows and Linux, and publish test results.

CI is planned, not configured. Once a solution exists, add .github/workflows/ci.yml with read-only default permissions, explicit toolchain setup, dependency restore, build, and the checks above. Keep ordinary CI offline from real services and hardware. Store test reports as artifacts; do not publish packages, create releases, or deploy from the first workflow. Record hardware/manual results separately from automated passes.

## Setup and development

### Current checkout

Open this README in a Markdown viewer. Git is initialized locally; no build or run step exists yet. Check the working tree with `git status --short --branch`. Documentation files start untracked; no initial commit or remote is created.

### Prerequisites and first implementation

After scaffolding, create FlowForge.slnx and the projects in the tree, wire references inward, add EF Core migrations, and provide a sample settings file with workspace and artifact roots. Use development-only local secret storage. Run the API/worker and web interface with a sample HTTP fake before enabling external endpoints.

Install only the dependencies needed for the selected milestone. Add a checked-in sample configuration with harmless defaults; keep local databases, recordings, generated datasets, credentials, and device identifiers outside version control. Pick a license before public distribution; no license is implied by this plan.

### Future commands

These commands are examples for after the referenced projects/files have been created; they will not build this documentation-only repository.

```powershell
dotnet restore FlowForge.slnx
dotnet test FlowForge.slnx
dotnet run --project src/FlowForge.Api
```

## Supporting libraries and boundaries

ConfigLens for validated settings; DataSynth for workflow fixture data. Keep step abstractions local until a stable extension contract exists.

Consume stable library versions through local package artifacts initially. Avoid absolute ProjectReference paths into sibling repositories and avoid Git submodules for the MVP. Keep a consumer's first vertical slice runnable using a small local interface or fixture while a supporting package is under development.

## Realistic limitations

MVP supports DAGs, not loops or arbitrary code execution. A database transaction cannot make external HTTP effects exactly once: handlers must document idempotency and uncertain outcomes. Exposing user-controlled URLs or paths requires access controls and destination restrictions before multi-user hosting.

Scope and performance targets are hypotheses until measured. Document reproducible dataset sizes, hardware, dependency versions, and failure cases alongside any benchmark claim. Do not present planned capabilities as shipped features.

## Documentation provenance

Prepared 2026-10-23. The previous chat supplied the project names and the AutoDiag platform change, but its detailed assistant responses were unavailable content references. The user confirmed the seven proposed application themes in this session; individual designs here are newly proposed plans, not a recovered transcript.

