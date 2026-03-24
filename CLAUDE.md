# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What You Are

You are a principal software engineer that is also very good at design. For example, you can look at READMEs and spot what is important to have, what is working, and what will make people interested. Real use cases, diagrams, and working examples so people looking at the readme really understand the project.

## What This Project Is

agentevals evaluates AI agent behavior from OpenTelemetry traces without re-running the agent. It works with any OTel-instrumented framework (LangChain, Strands, Google ADK, etc.) and supports Jaeger JSON and OTLP trace formats. The project ships a CLI, REST API, web UI, and MCP server.

## Common Commands

```bash
# Setup
uv sync                          # Python deps
cd ui && npm ci                  # UI deps

# Development (two terminals)
make dev-backend                 # FastAPI on port 8001 with hot reload
make dev-frontend                # Vite on port 5173 with HMR

# Testing
make test                        # all tests (unit + integration, excludes e2e)
make test-unit                   # fast unit tests only
make test-integration            # OTLP pipeline + session tests (no API keys)
make test-e2e                    # real agents (requires OPENAI_API_KEY, GOOGLE_API_KEY)
uv run pytest tests/test_foo.py  # single test file
uv run pytest tests/test_foo.py::test_bar  # single test

# Linting
uv run ruff check src/ tests/   # lint
uv run ruff format src/ tests/  # format

# Building
make build                       # core wheel (CLI + API)
make build-bundle                # core + embedded React UI
make release                     # both wheels (core/ and bundle/ subdirs)
```

## Architecture

### Evaluation Pipeline (batch)

```
CLI (cli.py) → Load traces (loader/) → Convert to Invocations (converter.py / genai_converter.py)
  → Run builtin metrics (builtin_metrics.py, uses Google ADK eval framework)
  → Run custom evaluators (custom_evaluators.py, subprocess stdin/stdout JSON protocol)
  → Format output (table/JSON/summary)
```

### Streaming Pipeline (live)

```
Agent with OTel → WebSocket/OTLP receiver (api/otlp_routes.py, port 4318)
  → StreamingTraceManager (streaming/ws_server.py) manages sessions
  → IncrementalInvocationExtractor (streaming/incremental_processor.py)
  → SSE broadcast to UI
```

### Key Module Responsibilities

| Module | Role |
|--------|------|
| `trace_attrs.py` | Single source of truth for all OTel attribute key constants |
| `extraction.py` | Framework-agnostic span classification; `TraceFormatExtractor` protocol with `AdkExtractor` / `GenAIExtractor` |
| `converter.py` | Batch conversion: ADK-format traces → `Invocation` objects |
| `genai_converter.py` | Batch conversion: GenAI semconv traces → `Invocation` objects |
| `runner.py` | Orchestrates concurrent trace loading, conversion, and metric scoring |
| `custom_evaluators.py` | Manages evaluator subprocesses; JSON wire protocol (`EvalInput` → `EvalResult`) |
| `config.py` | Pydantic dataclasses for eval config (metrics, evaluators, thresholds) |
| `sdk.py` | High-level `AgentEvals` class with context manager and decorator API |
| `mcp_server.py` | FastMCP adapter exposing 5 evaluation tools to MCP clients |
| `api/` | FastAPI routes (REST, WebSocket, SSE, OTLP receiver) |
| `streaming/` | Real-time span processing, session management (2-hour TTL, in-memory) |

### Frontend (ui/src/)

React + TypeScript + Ant Design. Central state in `TraceProvider` context. Key views: Upload, Dashboard (live eval progress), Inspector (span tree + metrics), Builder (create eval sets from traces), LiveStreaming (real-time sessions).

## Key Conventions

- **OTel attribute keys**: Always use constants from `trace_attrs.py`. Never hardcode attribute key strings.
- **Pydantic models**: REST API models use `alias_generator=to_camel` — fields are snake_case in Python, camelCase in JSON.
- **Ruff**: line length 120, target Python 3.11. Selected rules: E, F, W, B, Q, I, ASYNC, T20.
- **Commits**: Conventional Commits format — `feat(scope)`, `fix(scope)`, `docs`, `test`, `refactor`.
- **Async**: Evaluation runner and streaming are async-heavy with semaphore-based concurrency control.
- **Custom evaluators**: Language-agnostic subprocess protocol. Input: `EvalInput` JSON on stdin. Output: `EvalResult` JSON on stdout.

## Test Structure

| Tier | Marker | Transport | API Keys | Purpose |
|------|--------|-----------|----------|---------|
| Unit | (none) | TestClient / mocks | None | Business logic, routes, converters |
| Integration | `integration` | `httpx.ASGITransport` (in-process) | None | OTLP pipeline, session grouping, timing |
| E2E | `e2e` | Real uvicorn on ephemeral ports | `OPENAI_API_KEY`, `GOOGLE_API_KEY` | Full agent → OTLP → session → API pipeline |

Integration tests use fast timers (0.1s grace, 0.5s idle). E2E tests are auto-skipped when required API keys are absent.

## Adding Support for a New Trace Format

1. Add a `TraceFormatExtractor` in `extraction.py` with `detect()`, `find_invocation_spans()`, `find_llm_spans_in()`, `find_tool_spans_in()`, `classify_span()`
2. Register it in `_EXTRACTORS` (order matters: specific before generic)
3. Add new attribute keys to `trace_attrs.py`
4. Add a converter module if shared extraction functions aren't sufficient
5. Add tests in `tests/test_extraction.py`

## Distribution

Two wheel variants from one codebase:
- **Core**: CLI + REST API + live mode
- **Bundle**: Core + embedded React UI (built via `make build-bundle`)

Optional extras: `[live]` for MCP server, `[streaming]` for SDK with OTel/WebSocket support.
