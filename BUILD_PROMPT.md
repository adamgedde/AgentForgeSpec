# LLM Coding Assistant Prompt

Copy this prompt into a coding assistant with `SPECIFICATION.md`.

```text
Build the new independent self-hosted AI automation workbench described in SPECIFICATION.md. Read it completely before coding; it is the source of truth.

CrewAI is a required runtime dependency for worker, task, and sequential-team execution. Pin a compatible version and wrap it behind an application execution service. Do not make the database, API, UI, workflow engine, security controls, or trace format dependent on CrewAI internals.

AgentForge is mentioned only as the inspiration for the broad idea. Do not reuse its branding, source, schema, endpoints, text, or implementation. Do not mention unrelated products or organizations.

Work in the required milestone order. Before each milestone, state the smallest safe plan. For every milestone, deliver a vertical slice: migration/model, server validation, service/runtime, UI, tests, and docs. Run formatting, checks, and focused tests before continuing.

Treat credentials, URLs, filesystem access, SSRF, tool calls, prompts, traces, artifacts, and remote exposure as security-sensitive. Never return, log, trace, export, prompt, or commit plaintext secrets. Do not introduce shell execution; implement the constrained JSON transform DSL.

Use snapshots for every run. Build CrewAI agents/tasks/crews from the run snapshot, not mutable live definitions. Translate approved tools into traceable CrewAI-compatible tools with timeout, allowlist, size limit, and redaction controls. Use native application logic for workflows, routing, scheduling, human review, storage, and observability.

Start at milestone 1. Report assumptions, files changed, commands run, test results, and remaining limitations at the end of each milestone. Complete only when every acceptance test in SPECIFICATION.md passes and both fresh CLI and Docker deployments are documented and verified.
```

## Focused follow-ups

### CrewAI integration audit

```text
Audit CrewAI integration against SPECIFICATION.md. Confirm agents, tasks, and sequential teams are built from immutable run snapshots; approved tools are adapted through a traceable wrapper; task lifecycle/context is represented in application traces; and provider/model credentials remain outside CrewAI-visible traces. Fix defects and add regression tests.
```

### Security review

```text
Audit every secret, URL, file path, tool configuration, workflow template, artifact, and trace path against SPECIFICATION.md. Fix confirmed vulnerabilities, add regression tests, and document residual risks. Preserve local-only defaults.
```

