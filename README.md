# Self-Hosted AI Automation Workbench (FKA AgentForge)- LLM Coding Assistant Specification

This repository contains a specification and build prompt for an **independent, self-hosted AI automation workbench** originally known as AgentForge: a privacy-minded web app for designing, running, observing, and scheduling LLM-powered workers, teams, and workflows, built on **FastAPI and CrewAI**.

It does not contain source code. Instead, it contains everything a coding assistant (or a human engineering team) needs to build the product from scratch.

> *AgentForge* is cited in the specification only as inspiration for the general product category. Nothing here reuses its branding, source code, database schema, endpoint names, or implementation details, and building from this spec should not either.

## What's in this repo

| File | Purpose |
| --- | --- |
| [`SPECIFICATION.md`](./SPECIFICATION.md) | The full product/technical specification: UX requirements, required tech stack, domain model, API contract, security requirements, testing/acceptance criteria, and a 10-milestone build order. This is the source of truth. |
| [`BUILD_PROMPT.md`](./BUILD_PROMPT.md) | A ready-to-use prompt for feeding this spec to an LLM coding assistant, plus focused follow-up prompts for a CrewAI integration audit and a security review once a working baseline exists. |

## Required stack

The specification pins a specific backend stack rather than leaving it open-ended:

- **Python 3.11+ / FastAPI** for the application server.
- **CrewAI** as the execution runtime for workers, tasks, and sequential teams (agents/tasks/crews are built from immutable run snapshots, never live records, and wrapped so the rest of the app doesn't depend on CrewAI internals).
- **SQLAlchemy + Alembic**, SQLite by default.
- **APScheduler** for interval/cron scheduling, reconstructed from saved definitions at startup.
- A static SPA frontend served by the backend, with **SSE** for live run updates.

## How to use this

1. **Start a new, empty repository** for the actual application (do not build inside this spec repo).
2. Copy `SPECIFICATION.md` into that new repository.
3. Open your coding assistant of choice (e.g. Claude Code, Cursor, Copilot Workspace) with that repository loaded.
4. Copy the main build prompt from [`BUILD_PROMPT.md`](./BUILD_PROMPT.md) into the assistant, exactly as written.
5. Let the assistant work through the milestones in `SPECIFICATION.md` ("Build milestones") in order - each milestone should be runnable and tested before the next one starts.
6. Once a working baseline exists, use the **follow-up prompts** in `BUILD_PROMPT.md` (CrewAI integration audit, security review) to harden and validate the build.

## Why a spec instead of code?

This kit is meant to be reusable across coding assistants or teams while still guaranteeing a consistent outcome: the required stack (FastAPI + CrewAI + SQLAlchemy/Alembic + APScheduler) is fixed, but implementation details within that stack are left to the builder. The specification is intentionally prescriptive about *behavior, security, runtime, and milestones*, while leaving room for idiomatic patterns underneath.

## Swapping the execution runtime or other stack elements

The specification prescribes CrewAI as the execution runtime, and the build prompt treats it as required. If you want to build against a different runtime (a custom orchestrator, LangGraph, AutoGen, etc.), the spec's own architecture makes this a contained change rather than a rewrite. You'll need to edit `SPECIFICATION.md` and `BUILD_PROMPT.md` yourself before handing them to a coding assistant. 

In particular:

- The spec already requires CrewAI to be **wrapped behind an application execution service** so the database models, API contracts, validation, tracing, and UI never depend on CrewAI internals, and requires agents/tasks/crews to be built fresh from an immutable run snapshot each run rather than persisted as live CrewAI objects. That isolation boundary is what makes substitution feasible - preserve it regardless of which runtime you choose. This means you can swap in other stack elements as you'd like.
- Replace the "CrewAI as the execution runtime" line in `SPECIFICATION.md`'s "Required technology choices" with your chosen runtime, and carry that substitution through every other place CrewAI is named: tool adapters (`Tools`), `TeamTask` execution, the "Execution lifecycle" section, "Acceptance tests," and "Build milestones" (items 3, 4, and 6).
- Preserve the underlying contract regardless of runtime: construct agents/tasks from the run snapshot (not live records), propagate prior task output as context to later tasks, wrap tools with timeout/allowlist/size-limit/redaction controls, and emit lifecycle trace events in the application's trace format.
- In `BUILD_PROMPT.md`, update the "required runtime dependency" line and rename/rewrite the "CrewAI integration audit" follow-up prompt to check your chosen runtime's integration instead (snapshot-based construction, tool wrapping, trace emission, credential isolation).
- The rest of the stack (FastAPI, SQLAlchemy/Alembic, APScheduler, SSE) is runtime-independent and does not need to change, unless you have a preference to do so. If so, instruct your coding assistant to replace stack components as needed, and be sure to be consistent in those declarations.

## Scope notes

- First release is **single-user, self-hosted, local-only by default** - not a multi-tenant SaaS product, **NOR** is this something to be considered production ready.
- No unbounded shell/filesystem/internet access; tool use is constrained, adapted into traceable CrewAI-compatible tools, and auditable by design.
- Execution is sequential in v1 - no parallel or distributed workflow execution.

See `SPECIFICATION.md` ("Purpose" and "Scope") for the full scope discussion.

## Things that made it into production but aren't part of this spec, based on real-world use

AgentForge itself went through an extensive multi-phase build and was validated against real world pipelines, both personal and professional. I've tested on RFP question extraction, helping me find HVAC contractors, market research, and even as a content creation and editing team. A few capabilities proved necessary along the way that this spec intentionally leaves out, generalizes, or hands to the builder's discretion:

- **CrewAI Flows for branching and human checkpoints.** In production, CrewAI Flows (not Crews) turned out to be the natural fit whenever a pipeline needed conditional routing or a human checkpoint — Crews suit parallel/collaborative agent work, Flows suit structured, branching pipelines. This spec deliberately does not use CrewAI Flows at all; `Team` is sequential-only, and routing/human-review logic is reimplemented at the app level via the `Workflow` resource instead (see the design note in `SPECIFICATION.md`, under "Workflows"). That's a conscious architectural trade, not an omission, but it means the Flow-based approach already proven in production isn't what a builder following this spec will end up with.
- **Chunked extraction with post-processing.** Single-call LLM extraction broke down on large or complex source documents (PDF/DOCX/XLSX). The fix that actually worked in production was chunking the input plus a set of post-processing code operations — merge JSON arrays back together, reassign sequential IDs across chunks, and run a coverage check to catch structural extraction gaps. This spec's `code` workflow step only describes a generic "deterministic JSON transformation DSL" and doesn't call out this specific chunk/merge/reindex/verify pattern, even though it's what made large-document extraction reliable. The `code` step pattern can be usefully extended to other processing that may be more efficient in deterministic functions versus probabilistic token-heavy LLM calls.
- **A formatted run-results viewer, user guide, and a Role/Goal/Backstory design guide.** Production added a dedicated formatted view for run output and reference documentation to help operators write effective worker definitions. The spec covers "formatted output" at the API level but doesn't require authoring guidance or a user-facing guide.
- **A Blueprint concept with LLM-assist** Manual configuration of multiple, individual agents for Flows was time-consuming. Production added a "Blueprint" capability with LLM assist to build reusable JSON-based agent Blueprints that could be re-used across agents. LLM-assist allows users to stipulate an end objective and then a recommendation is made on the flow design.
- **UI refinements from real usage**: tagging, a card/table view toggle, and collapsible trace steps. The spec requires tags and live traces functionally but leaves these specific UX patterns to the builder.
- **Mailgun as the concrete email tool**, and manual document parsing libraries (pdfplumber, python-docx, openpyxl) preferred over the OpenAI Assistants API specifically because they preserve source-location tracking for RFP/presales traceability. The spec only requires "optional email delivery" and generic file tools, provider-agnostic by design.
- **A working default of GPT-4o at temperature 0.2** for structured extraction tasks. The spec leaves model/temperature choice to the operator's configured providers rather than prescribing a default.

## License

Licensed under [CC BY 4.0](./LICENSE) - free to share and adapt, including commercially, with attribution.