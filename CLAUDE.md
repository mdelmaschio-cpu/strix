# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this
repository. See `AGENTS.md` for the shorter, complementary guide aimed at agents that just want to
*run* Strix (as a scanning tool against some other target) rather than modify this codebase.

## What this is

Strix (`strix-agent`) is an open-source autonomous AI penetration-testing tool. Multi-agent
orchestration drives a real toolkit (proxy, browser, terminal, sandbox) against a target to find,
validate (working PoCs), and report vulnerabilities. Python 3.12+, managed with `uv`. A small Go
(Bubble Tea) TUI sidecar renders the interactive terminal UI; a prebuilt React SPA (`strix view`)
serves a local results dashboard.

## Commands

```bash
make setup-dev        # uv sync + pre-commit install
uv run strix --target ./app-directory     # run from source
uv run pytest                              # full test suite
uv run pytest tests/test_execution.py                       # single test file
uv run pytest tests/test_execution.py::test_name -v          # single test

make format            # ruff format .
make lint              # ruff check . --fix
make type-check        # mypy strix/ && pyright strix/
make security          # bandit -r strix/
make check-all         # format + lint + type-check + security — run before opening a PR
make pre-commit        # uv run pre-commit run --all-files
```

Go TUI (only needed when touching `strix/interface/tui/`):
```bash
make tui-build   # cd strix/interface/tui && go build ./cmd/strix-tui
make tui-test    # go test -race ./...
make tui-lint    # gofmt -l . && go vet ./...
```

Local viewer SPA (only needed when touching `strix/interface/viewer/frontend/`): editable/source
installs never build it — the compiled output is committed and shipped. If you change frontend
source, rebuild and commit the result:
```bash
make viewer   # cd strix/interface/viewer/frontend && npm ci && npm run build
```
Commit both the source change and the regenerated `strix/interface/viewer/static/`.

`make wheel` builds a platform wheel; it requires Go 1.24.x+ to compile and bundle the TUI sidecar
(`scripts/tui_sidecar_hook.py`) and fails rather than shipping a wheel without it.

## Configuration for running scans

```bash
export STRIX_LLM="openai/gpt-5.4"        # any LiteLLM model id
export LLM_API_KEY="your-api-key"
uv run strix --target ./app-directory
```
Config persists to `~/.strix/cli-config.json`. Requires Docker running (pulls/uses the sandbox
image on first run). Settings are defined via pydantic-settings in `strix/config/settings.py` /
`strix/config/models.py` (env-var aliases like `STRIX_LLM`, `LLM_API_KEY`, `LLM_API_BASE`,
`STRIX_REASONING_EFFORT`).

## Architecture

### Agent graph, not a single loop

A scan is a tree of `SandboxAgent`s coordinated at runtime, not one big prompt loop:

- **`strix/core/runner.py`** (`run_strix_scan`) — top-level entry point. Builds the root agent,
  brings up the sandbox session, wires an `AgentCoordinator`, and drives the root agent loop.
  Handles fresh runs and resume (`agents.json` + `agents.db` under the run's state dir).
- **`strix/core/agents.py`** (`AgentCoordinator`) — single owner of graph state: agent
  statuses (`running`/`waiting`/`completed`/`stopped`/`crashed`/`failed`/`budget_paused`),
  parent/child links, mailboxes, and SDK runtime handles. All cross-agent messaging and
  spawning goes through it.
- **`strix/core/execution.py`** — spawns/respawns child agents and runs the per-agent turn loop.
- **`strix/agents/factory.py`** (`build_strix_agent`) — assembles a `SandboxAgent`: renders the
  system prompt, attaches the base tool set plus any extra tools, and wraps every tool with
  argument coercion, output bounding/spilling, and error-as-result handling so a bad tool call
  never crashes the run. Root agents get `finish_scan`; child agents get `agent_finish`.
- **`strix/agents/prompt.py`** — renders the system prompt (scan mode, whitebox/blackbox,
  loaded skills, scope context).
- Agents talk to each other and spawn subagents via tools in
  **`strix/tools/agents_graph/`** (`create_agent`, `send_message_to_agent`, `wait_for_agents`,
  `stop_agent`, `view_agent_graph`) — this is how the "graph of agents" in the README is actually
  implemented.

### Tools (`strix/tools/`)

Each capability is its own subpackage exposing SDK `Tool` objects, registered as a base set in
`strix/agents/factory.py::_BASE_TOOLS` (plus filesystem/shell capabilities from the sandbox SDK):
`agent_browser` (Playwright), `proxy` (Caido HTTP interception), `shell`, `apply_patch`,
`reporting` (writes vulnerability/dependency reports), `notes`, `todo`, `finish`/`respond`
(lifecycle/interactive control), `load_skill`, `thinking`, `web_search`, `view_image`,
`agents_graph`. Third-party integrations register extra tools via
`register_agent_tools()` (e.g. `strix/runtime/backends.py`) rather than editing the factory.
Tool outputs are size-bounded and overflow spills to the sandbox workspace
(`strix/tools/output_store.py`) so oversized results don't blow the context.

### Runtime / sandbox (`strix/runtime/`)

Scans execute inside a Docker sandbox (image built from `containers/Dockerfile`) reached through
`strix/runtime/docker_client.py` / `session_manager.py`. `strix/runtime/caido_bootstrap.py` wires
up the interception proxy inside the sandbox. `backends.py` supports pluggable execution backends.

### Skills — two unrelated meanings, don't conflate them

- **`strix/skills/`** — internal knowledge packs the *pentest agents* dynamically load mid-scan
  (up to 5 at a time) for deep, task-specific expertise: `/vulnerabilities`, `/frameworks`,
  `/technologies`, `/protocols`, `/tooling`, `/cloud`, `/reconnaissance`, `/custom`. Loaded via the
  `load_skill` tool and injected into that agent's system prompt. See `strix/skills/README.md`.
  Two more categories, `/scan_modes` and `/coordination`, aren't agent-selectable — `prompt.py`'s
  `_resolve_skills` auto-injects `scan_modes/<mode>` for every agent and `coordination/root_agent`
  (or `coordination/source_aware_whitebox` for whitebox scans) for the root agent.
- **`skills/`** (repo root) — consumer-facing Agent Skills (SKILL.md) installed into *coding
  agents* like Claude Code via `npx skills add usestrix/strix`, teaching them to drive Strix itself
  (`penetration-testing-with-strix`, `managed-pentesting-with-strix`,
  `fix-security-vulnerabilities-with-strix`, `ci-security-scanning-with-strix`).

### Reporting (`strix/report/`)

Findings flow through `strix/report/state.py` (in-memory report state for the running scan) into
`writer.py` (Markdown reports), `dedupe.py` (finding dedup), `sarif.py` (SARIF 2.1.0 export), and
`usage.py` (LLM cost/budget tracking, enforced via `strix/core/hooks.py`). A run's artifacts land
in `strix_runs/<run-name>/`: `penetration_test_report.md`, `vulnerabilities/*.md`,
`vulnerabilities.json`, `findings.sarif`, `run.json`.

### Interface (`strix/interface/`)

`main.py`/`cli.py`/`cli_args.py` are the CLI entry point (`strix = "strix.interface.main:main"`).
`interactive.py` + `tui/` run the interactive terminal UI, backed by the Go Bubble Tea sidecar
(`tui/sidecar.py` launches the compiled binary; `tui/runtime.py` and
`tui/backend/controller.py` bridge it to the Python backend). `viewer/` is the `strix view` local
web dashboard (auth-tokened localhost server in `server.py`, prebuilt SPA in `static/`, source in
`frontend/`).

### LLM layer (`strix/llm/`, `strix/config/models.py`)

Model access goes through the OpenAI Agents SDK with a LiteLLM-backed provider
(`StrixProvider` / `configure_sdk_model_defaults`), so any LiteLLM-supported model id works via
`STRIX_LLM`. `compaction.py` and `context_budget.py` manage conversation/context-window pressure
across long-running scans.

## Code conventions

- Ruff-enforced: 100-char lines, double quotes, extensive rule set (see `pyproject.toml`
  `[tool.ruff.lint]`); several targeted `per-file-ignores` exist for lazy imports used to avoid
  circular deps or heavy optional dependencies — check that section before adding a new one.
- mypy runs in `strict = true` mode against `strix/`; pyright runs in `strict` mode too. Type
  hints are required on all functions.
- bandit security scanning runs over `strix/` as part of `make check-all` and pre-commit.
- Import convention: `known-first-party = ["strix"]`, `lines-after-imports = 2`.
- PEP 8, 100-char line limit, docstrings on public methods (per `CONTRIBUTING.md`).

## PR workflow

Open an issue first, branch from `main`, keep changes small/focused, run `make check-all`, update
docs/tests, then PR. Pre-commit hooks (ruff, mypy, bandit, pyupgrade, basic hygiene checks) run via
`make pre-commit` / `uv run pre-commit install`.
