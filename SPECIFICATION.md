# Self-Hosted AI Automation Workbench Specification

> This is a specification for a new, independent application. AgentForge is mentioned only as the project that inspired the general idea. Do not reuse its branding, source, data, endpoint names, schemas, or wording.

## Purpose

Build a privacy-minded, self-hosted browser application for designing, running, inspecting, and scheduling LLM automations. A technical operator should run it locally from a CLI or Docker, supply their own model-provider credentials, build reusable workers and workflows, and retain control of application data.

It closes the gap between one-off prompting/scripts and a custom automation platform: models, prompts, tools, inputs, outputs, side effects, and run history must all be visible and inspectable.

## Scope

The v1 UI must provide:

| Area | Behavior |
| --- | --- |
| Dashboard | Active definitions, recent run status, pending reviews, schedules, notifications. |
| Providers | Configure, test, edit, disable, and remove LLM providers; never return full secrets. |
| Connections | Reusable encrypted HTTP connection profiles and dependency-aware deletion. |
| Tools | Server-owned schema-driven registry; clients render forms from metadata. |
| Workers | Reusable role/goal/context/model/tool definitions, tags, optional bounded memory, clone and run actions. |
| Teams | Ordered worker tasks with task prompts and expected outputs. |
| Workflows | Ordered stateful worker, team, direct LLM, router, constrained code, and human-review steps. |
| Runs | Start, observe live traces, inspect output/artifacts, abort, and resume paused reviews. |
| Schedules | Interval and cron scheduling for any execution target. |
| Blueprints | Portable, versioned worker/team/workflow drafts with resolution preview and application tracking. |
| Insights | Descriptive post-run recommendations only; never automatically change a definition. |

Out of scope: multi-tenant SaaS, billing, public sharing, arbitrary shell access, unbounded filesystem/network access, visual graph editing, and parallel/distributed/hierarchical execution.

## Required technology choices

Use these defaults unless a documented compatibility issue requires a replacement:

- Python 3.11+ backend with FastAPI.
- SQLAlchemy and Alembic; SQLite is the default database and database URL is configurable.
- **CrewAI as the execution runtime** for workers, tasks, and sequential teams. Pin a compatible CrewAI version. Wrap it behind an application service so database models, API contracts, validation, tracing, and UI do not depend on CrewAI internals.
- LiteLLM (or CrewAI's supported provider integration) to normalize provider/model invocation where useful.
- APScheduler with a durable job store or schedule reconstruction at startup.
- A static SPA, served by the backend in the default deployment.
- SSE for live run updates, with polling fallback.
- Docker Compose with one application image and a persisted data bind mount.

CrewAI is responsible for the worker/task/crew execution layer, including task context propagation and tools adapted to CrewAI's tool interface. The application remains responsible for secret handling, provider/connection storage, allowlists, workflow control flow, human review, run snapshots, artifact storage, scheduling, and audit traces.

## Deployment and operations

Include `start.sh`, `stop.sh`, `.env.example`, `Dockerfile`, `compose.yaml`, and a container entrypoint.

The CLI startup flow must validate Python, create an isolated environment, install pinned dependencies, create missing configuration without overwriting existing values, generate a durable encryption key when absent/placeholder, create data/artifact directories, apply migrations, refuse an unsafe port collision, write PID/log files, and wait for `/api/health`.

`docker compose up --build` must work from a fresh checkout. The compose file maps a configurable host port, mounts `./data` for database/artifacts/durable key fallback, applies migrations before serving, defines a health check, and does not bake secrets into the image.

Document: host/port, database URL, encryption key, CORS origins, log level, API token, artifact directory/inline threshold, provider credentials, tool-service credentials, and run timeout. Default to loopback-only exposure. Remote access requires explicit origins, an API token, and documented TLS/reverse-proxy guidance.

## Data model

Use UUIDs and UTC timestamps. Archive/disable definitions with history instead of deleting them.

### Provider and connection

`ModelProvider`: unique name, kind (OpenAI-compatible, Anthropic, Ollama, generic compatible endpoint), endpoint, encrypted token, default/manual model list, connection-test metadata, enabled state, timestamps.

`ApiConnection`: unique name, base URL, auth mode (`none`, API-key header, bearer, basic), encrypted credential payload, default headers, description, test metadata, enabled state, timestamps.

Adapters validate endpoints, run low-cost tests, and redact all credentials in responses, errors, logs, exports, model prompts, and traces.

### Tools

The server owns a tool registry. Each definition has an ID, display name, description, required environment variables, and JSON form schema. A worker stores an ID plus validated configuration. Reject unknown tool IDs and fields.

Implement web search, configured HTTP calls, file reading, file writing, PDF creation, and optional email delivery. Convert each approved tool into a traceable CrewAI-compatible tool. Each tool must have a timeout, output-size limit, structured/redacted trace, and clear user-facing error.

Never add arbitrary shell execution in v1. File reads require explicit allowed directories. File/PDF writes require an output directory and strict resolved-path containment. HTTP calls use saved connections and URL policy.

### Workers and teams

`Worker`: unique name, role, goal, optional context, provider/model reference, validated generation config, memory flag, tags, tool assignments, enabled state, timestamps.

`WorkerMemory`: one bounded, structured rolling-memory record per worker. Only successful eligible runs update it. Operators can clear it.

`Team`: name, optional description, enabled state, process type. V1 permits only `sequential`.

`TeamTask`: position, worker reference, task description, expected output. Use CrewAI Task and Crew objects to run tasks in order. Pass prior task output through CrewAI task context and explicitly record each task's lifecycle in the application's run trace.

### Workflows

> **Design note — CrewAI Flows.** Earlier work on the AgentForge platform (this spec's inspiration) found that CrewAI Flows, not Crews, were the natural fit whenever branching logic or human checkpoints were required: Crews suit parallel/collaborative agent work, while Flows suit conditional routing and structured pipelines. This specification deliberately does **not** adopt CrewAI Flows. `Team` is restricted to `sequential` CrewAI Crews only, and all branching, routing, and human-review logic instead lives in the app-level `Workflow` resource below (`router`, `human_review`, etc.), outside of CrewAI entirely. The trade-off is intentional: it keeps control flow, pause/resume state, and audit trace format independent of CrewAI's Flow implementation, at the cost of re-implementing logic CrewAI Flows already provide natively. A builder who wants Flow-native branching instead should treat this as the one place in the spec to consciously reconsider, not silently follow.

`Workflow` has name, description, enabled state, timestamps, and ordered JSON steps. Support:

| Kind | Behavior |
| --- | --- |
| `worker` | Run saved worker with templated input and save output under a state key. |
| `team` | Run saved CrewAI sequential team with templated input and save output. |
| `llm` | Direct configured provider/model request with templated prompt. |
| `router` | Evaluate ordered state conditions and select a named branch/default. |
| `code` | Run a small deterministic JSON transformation DSL; never arbitrary Python/shell. |
| `human_review` | Pause, persist state/resume context, show editable preview, then resume or abort after operator action. |

Use `{{state_key}}` templates and JSON-compatible state. Validate pre-save: unique step IDs and outputs, valid types/references, valid provider references, reachable branches, no dangling targets/invalid start, and no cycles until a later explicit loop policy exists.

### Runs and related records

`Run`: exactly one target (worker/team/workflow), status, input, output preview, truncation/full-artifact reference, duration, safe error, trigger source, immutable configuration snapshot, structured trace, state for workflow runs, timestamps.

Statuses: `pending`, `running`, `success`, `failed`, `timeout`, `paused`, `aborted`.

Large outputs are run-scoped artifacts. Artifact paths are server-generated, reads are restricted to the requested run, and API responses contain bounded previews.

Include Schedule, Notification, Blueprint, and Insight records. A blueprint stores a versioned portable JSON draft, source prompt/analysis, preview, tags, status, and application metadata. Insights are advisory summaries, metrics, score, and prioritized recommendations.

## Execution lifecycle

1. Validate enabled target, references, tool configuration, and required credentials.
2. Create a durable `pending` run and immutable target snapshot; transition to `running`.
3. Return promptly from the start endpoint and execute in a background task.
4. Invoke Worker/Team operations via CrewAI. Create CrewAI agents/tasks/crews from the snapshot—not mutable live records.
5. Emit capped, JSON-safe, time-stamped trace events for lifecycle, CrewAI task events, direct LLM calls, and tool calls. Redact secrets.
6. Enforce overall run timeout and smaller provider/tool timeouts.
7. At human review, persist state and resume stack, then set `paused`. Resume only after validating edited state. Abort must be idempotent and checked between steps.
8. On terminal completion, persist duration/output/error; update eligible memory, create insight, and notify on scheduled execution or failure.

## API

Provide JSON REST under `/api`, standard errors, list pagination, and unauthenticated `/api/health`.

| Resource | Minimum operations |
| --- | --- |
| `/providers`, `/connections` | CRUD and test; providers also list models. |
| `/tools` | List metadata/get one definition. |
| `/workers` | CRUD, clone, memory get/clear, run start, optional design assist/export. |
| `/teams`, `/workflows` | CRUD, clone, start; workflows validate draft. |
| `/runs` | Get/list, formatted output, artifact list/read, SSE, pause state, resume, abort. |
| `/schedules` | CRUD and enable/disable. |
| `/blueprints`, `/insights`, `/notifications`, `/dashboard` | CRUD/tracking or read actions needed by the UI. |

Use 400 invalid request, 401 invalid/absent API token, 404 missing resource, and 409 dependency conflict. Secret fields return only a status or redacted hint.

## Security and reliability

- Encrypt persisted credentials with an authenticated symmetric key configured by environment. Document key rotation consequences.
- Never expose plaintext secrets in API/UI/logs/errors/exports/traces/prompts/telemetry.
- Mitigate SSRF: validate URL schemes; block metadata, link-local, reserved, and unapproved private targets; document any development allowlist.
- Perform resolved-path containment checks and reject traversal/symlink escapes for file tools and artifacts.
- Require an API token for non-loopback access; document TLS/reverse proxy use.
- Cap request, trace, output preview, artifact-read, tool-output, and memory sizes.
- Validate schedule target exclusivity and cron syntax. Reconstruct schedules exactly once after restart.
- Use migrations for all persistent changes; do not rely solely on ORM auto-create in production.
- Return safe client errors while preserving redacted diagnostic logs.

## Blueprint/design assist

Blueprints use published versioned schemas and human-readable references rather than local database IDs where possible. Preview/import reports resolved and unresolved requirements before live materialization.

An optional design assistant must use a selected configured provider, request strict schema-valid JSON, pass generated JSON through normal validators, save prompt/version metadata, and never create live definitions or trigger side-effect tools without confirmation.

## Acceptance tests

Automated tests and a documented manual smoke test must cover:

- fresh CLI and Docker health checks;
- clean and upgrade migrations;
- encrypted/redacted provider and connection secrets;
- safe provider failure handling;
- tool-schema validation;
- file/artifact containment;
- CrewAI-backed worker snapshots, traces, timeout/error, and large artifacts;
- CrewAI sequential team context passing;
- workflow validation, routing, deterministic code transform, pause/resume/abort;
- schedule reconstruction without duplicates;
- dashboard/notification/insight updates;
- CORS/API-token behavior;
- portable blueprint preview for both resolved and unresolved references.

## Build milestones

1. App shell, configuration, migrations, static UI, CLI/Docker.
2. Secure provider and connection management.
3. Tool registry, CrewAI tool adapters, containment, redacted tracing.
4. Worker CRUD and reliable CrewAI worker execution.
5. Run history/SSE/artifacts/output formatting/memory.
6. CrewAI sequential teams.
7. Workflows, validation, routers, DSL code steps, human review/resume/abort.
8. Scheduler, notifications, dashboard, insights.
9. Blueprints/design assistant/import-export preview.
10. Security review, fresh-install verification, test/documentation completion.