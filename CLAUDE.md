# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DeepCode is an open agentic coding system (Python 3.12+) with a multi-agent LLM runtime. It ships two frontends (interactive CLI/TUI and a Tauri Desktop workbench) over a single shared Agent runtime. The PyPI package (`deepcode-hku`) distributes only the Python runtime; Desktop, tests, and CI assets live in the source repo.

## Common Commands

```bash
# Run the full test suite (pytest, configured in pyproject.toml)
pytest

# Run a single test file or test
pytest tests/test_agent_runner_kernel.py
pytest tests/test_skills.py -k "test_name"

# Lint and format (ruff, line-length 88, target py312)
ruff check .
ruff format .

# Type-check / lint via pre-commit (optional)
pre-commit run --all-files

# Install in editable mode for development
pip install -e .

# Run the CLI entry points
deepcode                  # interactive TUI (default)
deepcode exec "<task>"    # headless one-shot task
deepcode loop "<goal>"    # durable Goal on shared Turn runtime
deepcode --help           # full command list
python deepcode.py        # equivalent, runs from source

# Run the app server (JSON-RPC over stdio, used by Desktop)
python -m app_server

# Desktop development (requires Node + npm)
cd desktop && npm install && npm run dev
```

## Architecture

### Frontend → Application → Domain layers

All product surfaces compose the same `DeepCodeApplication` (`core/application/application.py`), which owns every service and exposes an `ApplicationLease`. Frontends never construct services directly:

- **`cli/`** — console entry points. `deepcode.py` dispatches subcommands (`exec`, `loop`, `schedule`, `automation`, `session`, `skill`, `provider`, `mcp`, `chat/tui`). Each subcommand is a thin adapter over `DeepCodeApplication`.
- **`app_server/`** — single-client stdio JSON-RPC server (`AppServer` + `Dispatcher`) that the Tauri Desktop calls. `protocol/app-server.schema.json` is the authoritative wire schema.
- **`desktop/`** — Tauri + Vite + React frontend (TypeScript). Communicates with the Python runtime through the app server.

### Core runtime (`core/`)

The runtime is organized around a durable **Project → Session → Thread → Turn** lifecycle:

- **`core/agent_runtime/`** — the shared tool-using agent loop (`runner.py`, ~69 KB, ported from nanobot). `injections.py` adapts runtime input to provider messages; `helpers.py` handles message building, token estimation, and tool-result persistence; `tools/` holds the tool registry and per-tool implementations.
- **`core/application/`** — product services composed in `DeepCodeApplication`. Key services: `TurnService`, `ThreadService`, `SessionRuntime`, `GoalExtension`, `AutomationService`, `WorkflowService`, `McpService`, `SkillService`, `FileService`, `GitService`, `WorktreeService`, `ApprovalService`, `ExecutionCoordinator`.
- **`core/domain/`** — Pydantic value objects and enums (`Turn`, `Thread`, `ThreadGoal`, `Approval`, `Automation`, `ExecutionSecurity`, `Workflow`). No I/O, no services.
- **`core/harness/`** — cross-cutting agent infrastructure: `permissions.py` (three-valued allow/ask/deny engine), `sandbox.py` (macOS seatbelt / Linux bubblewrap command wrapping), `approval.py`, `policy.py`, `memory.py`, `skills.py`, `hooks/`, `tools/`, `agents/`.
- **`core/providers/`** — LLM provider abstraction. `base.py` defines `LLMProvider`; `anthropic.py`, `openai_compat.py` (OpenAI / Azure / DeepSeek / Gemini / Groq / OpenRouter), `catalog.py` + `catalog_service.py` resolve models per workflow phase. `reasoning.py` and `profiles.py` handle model-aware thinking/reasoning controls.
- **`core/sessions/`** — durable persistence. `store.py` is the `SessionStore`; `thread_goal_store.py` persists Goals; `models.py` defines the ORM models; `index.py` is the session index; `deletion.py` handles guarded deletion.
- **`core/skills/`** — Agent Skills. `catalog.py` discovers `SKILL.md` files from project + user roots; `runtime.py` resolves skills per-turn with progressive disclosure; `management.py` handles import/create; `roots.py` enumerates skill locations.
- **`core/config.py`** — single source of truth for runtime settings. Layered JSON (`deepcode_config.json`): user-level base at `$DEEPCODE_HOME` or `~/.deepcode` deep-merged with a project-level file walked up from cwd. Accepts both camelCase and snake_case keys; supports `${ENV_VAR}` references.
- **`core/team/`** — multi-agent worktree collaboration (`worktree.py`).
- **`core/loop/`** — `autodream.py`, the autonomous loop-engineering driver.

### Supporting modules

- **`tools/`** — standalone MCP-style tools: `code_implementation_server.py`, `code_indexer.py`, `command_executor.py`, `document_conversion_server.py`, `pdf_converter.py`, `pdf_downloader.py`, `git_command.py`.
- **`workflows/`** — higher-level orchestration: `agent_orchestration_engine.py` (multi-agent workflow engine), `code_implementation_workflow.py`, `codebase_index_workflow.py`, `planning_runtime.py`, `plan_review_runtime.py`, `environment.py`.
- **`prompts/code_prompts.py`** — system prompts and phase prompt templates.
- **`utils/`** — `file_processor.py`, `llm_utils.py`, `loop_detector.py`.
- **`eval/`** — evaluation harness material.
- **`schema/`** — JSON schemas (`mcp-agent.config.schema.json`).

## Key Conventions

- **Python 3.12+**, ruff for lint/format (line-length 88, `target-version = "py312"`). Ruff ignores `E402` (module-level import not at top).
- **Config is layered and cwd-independent**: provider keys live in the user-level base (`~/.deepcode/deepcode_config.json`); project config overrides per-key. Tests isolate via `DEEPCODE_HOME` and `DEEPCODE_SESSIONS_DIR` env vars (see `tests/conftest.py`).
- **Permission model is mechanism, not policy**: `core/harness/permissions.py` decides `ALLOW`/`ASK`/`DENY`; callers surface `ASK` as auto-deny (headless) or a real prompt (interactive). Sensitive-path protection (`.ssh`, `.aws/credentials`, `.env`) is enforced ahead of rules unless Full Access preset is active.
- **Sandbox degrades gracefully**: macOS uses `sandbox-exec` (closed-by-default seatbelt), Linux uses `bwrap`. If no backend exists, `wrap_shell_command` returns the command unchanged — callers must fall back to approval-first.
- **Skills are `SKILL.md` files** with YAML frontmatter, discovered from project roots (`.deepcode/skills/`, `.claude/skills/`) and the user root (`~/.deepcode/skills/`). Max 256 catalog entries, max 8 per turn, max 48k injected chars.
- **Version is canonical in `core/version.py`** (`__version__`); `setup.py` reads it via regex without importing runtime deps.
- **Entry points**: `deepcode` → `deepcode:main`; `deepcode-app-server` → `app_server.__main__:main`.
- **Do not modify `external/`** if such a directory exists; do not submit it.
