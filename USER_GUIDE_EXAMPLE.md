# AgentForge User Guide

AgentForge is a locally hosted workbench for building and operating AI automation. You configure your own model providers, create reusable agents, combine them into sequential crews and stateful flows, then run or schedule the result from a browser.

Your provider credentials, application database, run history, and artifacts stay on the machine where you deploy it.

## Contents

- [Build automation](#build-automation)
- [Run and inspect automation](#run-and-inspect-automation)
- [Schedule and operate](#schedule-and-operate)
- [Blueprints and optimization](#blueprints-and-optimization)
- [Private remote access](#private-remote-access)
- [Troubleshooting](#troubleshooting)

## Start here

## Build automation

### 1. Add a model provider

Open **Providers** and select **Add Provider**.

Supported provider styles are OpenAI, Anthropic, Ollama, and Custom/OpenAI-compatible endpoints. Provide a name, provider type, credentials when needed, base URL when needed, a default model, and optionally an available-model list.

Save, then use **Test Connection**. The provider card records its last test result. Model lists can be loaded remotely for supported providers after the provider is saved.

Provider credentials are encrypted at rest and only shown as redacted values.

### 2. Add reusable API connections

Open **Connections** and select **Add Connection**. Connections are reusable profiles for the HTTP API Caller tool.

Set:

- name and base URL;
- authentication type: None, API Key, Bearer Token, or Basic Auth;
- credentials and optional default headers;
- an optional description.

Use **Test** to send a GET request to the base URL with configured authentication. You cannot delete a connection while an active agent tool assignment depends on it.

### 3. Understand tools

Tool choices are configured per agent. The UI uses server-provided schemas, so available configuration fields depend on the selected tool.

| Tool | Use | Operator responsibility |
| --- | --- | --- |
| Tavily Web Search | Web research | Set `TAVILY_API_KEY`; choose result count/depth. |
| HTTP API Caller | Call a configured external API | Create an API connection; constrain endpoint/method/body templates. |
| File Reader | Read text, PDF, DOCX, or spreadsheet content | Explicitly list allowed directories. |
| File Writer | Save content to disk | Choose a safe output directory. |
| PDF Writer | Create a PDF artifact | Choose a safe output directory. |
| Email (Mailgun) | Send messages and one local attachment | Configure Mailgun and permitted sender/recipients. |

File paths are constrained to configured directories. Do not expect an agent to read or write arbitrary files.

### 4. Create an agent

Open **Agents** and select **Create Agent**.

An agent is a reusable CrewAI worker definition. It combines:

- **Name** — unique label;
- **Role** — specific job title or operating lens;
- **Goal** — measurable responsibility;
- **Backstory** — optional domain context, standards, and constraints;
- **Provider and model** — model execution choice;
- **LLM configuration** — supported controls such as temperature, token limit, top-p, penalties, and stop sequences;
- **Tools** — approved capabilities plus their configuration;
- **Memory** — optional rolling summary of successful runs;
- **Tags** — searchable labels.

A useful agent definition is narrow enough to be dependable. Put task-specific instructions in the run input; put enduring responsibilities, quality standards, and limitations in the agent definition.

Example:

```text
Role: Market Research Analyst

Goal: Gather source-aware public information and organize it into an actionable brief, clearly separating confirmed facts from hypotheses.

Backstory: You prefer primary sources, identify uncertainty, and never state an unsupported inference as fact.
```

Use **Clone** to create a variation without changing the original. Use the Design Assistant when you want a provider to draft the role, goal, backstory, and starter task from a plain-language description; always review and edit its result before saving.

### 5. Create a sequential crew

A crew is an ordered set of tasks assigned to agents. AgentForge executes only sequential crews: each task runs after the prior task and receives prior output as context through CrewAI.

Open **Crews** and select **Create Crew**:

1. Give it a name and optional description.
2. Keep the process as `sequential`.
3. Add tasks in intended order.
4. For each task, select an active agent, describe the task, and specify expected output.
5. Save.

Use a crew when roles should be separated, such as research followed by writing, extraction followed by validation, or analysis followed by report production.

### 6. Create a flow

A flow is a stateful multi-step pipeline. Use it when you need branching, shared state, direct LLM calls, review gates, or reusable orchestration beyond a sequential crew.

Open **Flows** and select **Create Flow**. A run starts with:

```json
{
  "input": "the submitted run input",
  "user_input": "the submitted run input"
}
```

Step templates reference state with double braces, for example `{{input}}` or `{{research}}`.

| Step | Use |
| --- | --- |
| Agent | Run one saved agent and place its output in a state key. |
| Crew | Run one saved sequential crew and place its output in state. |
| LLM | Make a direct provider/model call for classification, extraction, evaluation, or a small focused task. |
| Router | Compare a state value and choose the first matching branch or default branch. |
| Code | Perform configured transformation operations over flow state. Keep input/output JSON-compatible. |
| Human review | Pause the flow for an operator to inspect or edit state, continue, skip, or abort. |

Give every output-producing step a clear, unique state key. Before saving, use draft validation and correct duplicate IDs, duplicate output keys, missing references, or invalid branches.

A simple dependable flow is:

1. Research agent writes `research`.
2. LLM evaluates `{{research}}` and writes `quality`.
3. Router sends weak results to a refiner and strong results to a writer.
4. Human review checks the final draft before an email or file-writing step.

## Run and inspect automation

### Start a run

Open an agent, crew, or flow, enter the input, and select **Run**. Start requests return promptly; execution proceeds in the background.

A run transitions through:

```text
pending → running → success | failed | timeout
```

Flows may also become `paused` for human review or `aborted` by an operator.

The run page shows final output, formatted/raw views, status, duration, trigger source, errors, artifacts, and a live event stream. Use **Re-run** to load a previous input back into the form; review it before starting the next run.

### Read traces

Traces are the audit trail. Common events include:

- `reasoning` — setup, model/runtime progress, and memory activity;
- `tool_call` / `tool_result` — approved tool activity and safe result/error information;
- `output` — generated output or failure information;
- `crew_started`, `task_started`, `task_completed`, `context_passed`, `crew_completed`;
- `flow_step_started`, `flow_state_updated`, `flow_route_selected`, `flow_paused`, `flow_resumed`, `flow_aborted`, `flow_completed`.

Use traces to locate a failed provider call, incorrect tool configuration, blocked file path, unexpected state value, or routing decision. Trace content and inline output are size-bounded; inspect linked artifacts for full large output.

### Human review

A flow review gate pauses execution with saved state and resume context. From the paused-run page, review the next step and state preview, then:

- approve and continue without changes;
- edit permitted state values and continue;
- skip the next eligible step where offered;
- abort the flow.

Reviewing does not rerun earlier completed steps. Resume/abort actions are recorded in the trace.

## Schedule and operate

### Schedules

Create schedules from the target detail page or **Schedules**. A schedule targets one active agent, crew, or flow.

Supported schedule types:

- **Interval** — repeat every configured number of seconds.
- **Cron** — run at times defined by a standard five-field cron expression.
- **Event** — store an event configuration for an external triggering integration; confirm your deployment's event integration before relying on it.

Each scheduled run uses the saved input payload and is recorded with trigger source `schedule`. Toggle a schedule off rather than deleting it when you may want to restore it later.

Avoid intervals shorter than the normal execution time. The scheduler uses a concurrency guard to prevent overlapping runs for the same schedule.

### Notifications and dashboard

The dashboard summarizes active agents, crews, flows, runs today, schedules, unread notifications, paused flows, and recent executions.

Notifications highlight scheduled-run outcomes and operational events. Mark individual notifications or all notifications as read after review.

### Optimization insights

Completed runs receive a descriptive optimization record. It summarizes performance and reliability signals and suggests improvements such as clearer task input, narrower agent scope, or timeout/tool changes.

Insights are advisory. AgentForge does not silently modify agents, crews, flows, schedules, or prompts.

## Blueprints and optimization

Blueprints are a separate draft library for reusable agent, crew, and flow designs. Use them to capture natural-language design requests, generated analysis, editable JSON, tags, notes, validation previews, and application history before or alongside creating live definitions.

A blueprint is not itself a runnable asset. Review the preview, resolve missing provider/tool/agent/crew references, then create or update the corresponding live item through the main builders. Archive drafts you want to retain but remove from active work.

## Private remote access

For access beyond the same machine, place AgentForge behind a private reverse proxy such as nginx on a private network or tailnet. Use TLS, authenticate the proxy or application API, and add the exact browser origin to `CORS_ORIGINS`.

Do not expose the service directly to the public internet without a deliberate authentication, TLS, firewall, backup, and update plan.

## Troubleshooting

### The browser cannot open the app

- Run `./start.sh` and wait for its health-check confirmation.
- Open `http://localhost:<APP_PORT>`.
- Check `agentforge.log`.
- Confirm the port is not already owned by another process.

### Provider test or run fails

- Recheck API key, base URL, provider type, and model name.
- For Ollama, confirm the server is reachable from the application host.
- Confirm the account can access the model.
- Check the first error or `reasoning` event in the run trace.

### A tool fails

- Tavily: set `TAVILY_API_KEY`.
- HTTP: test the connection, verify endpoint template variables, method, body, and extraction path.
- File tools: make sure the requested path resolves inside the configured directory.
- Mailgun: set API key/domain and verify sender/recipient requirements.

### A flow routes or pauses unexpectedly

- Inspect the state snapshot immediately before `flow_route_selected`.
- Compare router state key, operator, and expected value.
- Inspect step run conditions and whether a prior output key is blank or malformed.
- For paused flows, open the run detail page; review actions must be taken there.

### A schedule did not run

- Confirm it is active and targets an enabled definition.
- Validate interval/cron configuration and the saved input.
- Check the dashboard, notifications, and server log.
- Avoid overlapping long-running schedules for the same target.

### Secrets no longer work after restart

The encryption key changed. Restore the original `ENCRYPTION_KEY` (or Docker-generated durable key) or re-enter provider and connection credentials.

### Where to find logs and data

- Server log: `agentforge.log`
- Local database by default: `data/agentforge.db`
- Docker persistent data: `./data`
- PID file for local startup: `.pid`
- Run artifacts: the configured `ARTIFACT_BASE_DIR`

Back up `.env` securely, the database, and artifact directory together. Restoring only the database without the matching encryption key prevents decryption of saved credentials.


