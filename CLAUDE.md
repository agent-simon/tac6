# CLAUDE.md — AI Assistant Guide for tac6

This file provides essential context for AI assistants (Claude Code and similar) working in this repository.

---

## Project Overview

**Natural Language SQL Interface** — a full-stack web application that converts natural language queries into SQL using OpenAI or Anthropic LLMs, executes them against uploaded CSV/JSON/JSONL data stored in SQLite, and displays results in a browser table.

Additionally, the repository contains **ADW (AI Developer Workflow)**, an autonomous GitHub issue processing system that uses Claude Code CLI to classify, plan, implement, test, review, and document code changes end-to-end.

---

## Repository Structure

```
tac6/
├── app/
│   ├── client/          # Vite + TypeScript frontend (port 5173)
│   │   ├── src/
│   │   │   ├── main.ts         # All UI logic (single entrypoint)
│   │   │   ├── api/client.ts   # Typed API client (fetch-based)
│   │   │   └── types.d.ts      # Global TypeScript types
│   │   ├── public/sample-data/ # Sample CSV/JSON/JSONL files
│   │   ├── package.json        # Vite + TypeScript devDeps only
│   │   └── vite.config.ts      # Proxy /api → localhost:8000
│   └── server/          # FastAPI backend (port 8000)
│       ├── server.py           # FastAPI app, all route handlers
│       ├── main.py             # (alternative entrypoint, see server.py)
│       ├── pyproject.toml      # uv-managed dependencies
│       ├── core/
│       │   ├── constants.py        # NESTED_DELIMITER, LIST_INDEX_DELIMITER
│       │   ├── data_models.py      # Pydantic request/response models
│       │   ├── file_processor.py   # CSV/JSON/JSONL → SQLite conversion
│       │   ├── insights.py         # Column statistical insights
│       │   ├── llm_processor.py    # OpenAI/Anthropic SQL generation
│       │   ├── sql_processor.py    # SQL execution + schema retrieval
│       │   └── sql_security.py     # SQL injection protection layer
│       ├── db/                 # SQLite database (git-ignored, auto-created)
│       │   └── database.db
│       └── tests/
│           ├── core/
│           │   ├── test_file_processor.py
│           │   ├── test_llm_processor.py
│           │   └── test_sql_processor.py
│           └── test_sql_injection.py
├── adws/                # AI Developer Workflow system
│   ├── adw_plan.py              # Phase: create implementation plan
│   ├── adw_build.py             # Phase: implement the plan
│   ├── adw_test.py              # Phase: run tests + auto-fix failures
│   ├── adw_review.py            # Phase: review vs spec + screenshots
│   ├── adw_document.py          # Phase: generate documentation
│   ├── adw_patch.py             # Quick patch workflow
│   ├── adw_plan_build.py        # Orchestrator: plan + build
│   ├── adw_plan_build_test.py   # Orchestrator: plan + build + test
│   ├── adw_plan_build_review.py # Orchestrator: plan + build + review
│   ├── adw_plan_build_test_review.py
│   ├── adw_plan_build_document.py
│   ├── adw_sdlc.py              # Full SDLC: all phases
│   ├── adw_modules/
│   │   ├── agent.py        # Claude Code CLI integration
│   │   ├── data_types.py   # Pydantic models (ADWState, etc.)
│   │   ├── state.py        # State management for workflow chaining
│   │   ├── workflow_ops.py # Core business logic
│   │   ├── r2_uploader.py  # Cloudflare R2 screenshot uploads
│   │   └── utils.py        # Utility functions
│   ├── adw_triggers/
│   │   ├── trigger_cron.py    # Polls GitHub every 20s
│   │   └── trigger_webhook.py # Webhook server (port 8001)
│   └── adw_tests/             # ADW system tests
├── .claude/
│   ├── settings.json       # Claude Code permissions + hook registration
│   ├── commands/           # Slash command definitions (.md files)
│   └── hooks/              # Lifecycle hooks (Python scripts via uv)
│       ├── pre_tool_use.py     # Blocks rm -rf and .env access
│       ├── post_tool_use.py    # Logs tool usage
│       ├── notification.py     # Desktop notifications
│       ├── stop.py             # Session end handler
│       ├── subagent_stop.py    # Subagent stop handler
│       ├── pre_compact.py      # Pre-compaction handler
│       └── user_prompt_submit.py  # Logs user prompts
├── scripts/
│   ├── start.sh            # Start both backend + frontend
│   ├── stop_apps.sh        # Stop all services
│   ├── reset_db.sh         # Reset SQLite database
│   ├── copy_dot_env.sh     # Copy .env.sample → .env
│   ├── expose_webhook.sh   # Expose webhook via cloudflared
│   ├── kill_trigger_webhook.sh
│   ├── clear_issue_comments.sh
│   └── delete_pr.sh
├── specs/                  # Feature specification files
├── logs/                   # Structured Claude Code session logs
├── agents/                 # ADW workflow output directories
├── .env.sample             # Environment variable template (root-level)
├── .mcp.json               # MCP server config (Playwright for E2E)
└── playwright-mcp-config.json
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vite 6, TypeScript 5.8, vanilla DOM (no framework) |
| Backend | FastAPI 0.115, Uvicorn, Python 3.10+ |
| Database | SQLite (file-based, per-session, in `app/server/db/`) |
| LLM Providers | OpenAI (`gpt-4.1-mini`) and/or Anthropic (`claude-3-haiku-20240307`) |
| Package mgr (Python) | `uv` (not pip/poetry) |
| Package mgr (Node) | `bun` (scripts use `npm` as fallback) |
| Testing (Python) | pytest |
| Linting (Python) | ruff |
| E2E Testing | Playwright via MCP server |
| ADW Automation | Claude Code CLI |

---

## Development Commands

### Start Everything

```bash
./scripts/start.sh          # Starts backend (8000) + frontend (5173)
```

The script checks for `app/server/.env`, kills existing processes on those ports, starts both services, and handles graceful shutdown on `Ctrl+C`.

### Backend (FastAPI)

```bash
cd app/server
cp ../.env.sample .env      # First time setup (note: .env.sample is at root)
uv sync --all-extras        # Install all dependencies including dev
uv run python server.py     # Start with hot reload (uvicorn reload=True)
uv run pytest               # Run all tests
uv run pytest tests/test_sql_injection.py -v   # Run specific test file
uv run ruff check .         # Lint
uv run ruff format .        # Format
uv add <package>            # Add dependency
uv remove <package>         # Remove dependency
```

### Frontend (Vite)

```bash
cd app/client
bun install                 # Install dependencies
bun run dev                 # Dev server on http://localhost:5173
bun run build               # Production build (tsc + vite build)
bun run preview             # Preview production build
```

### Reset Database

```bash
./scripts/reset_db.sh       # Deletes db/database.db
```

---

## Environment Variables

Copy `.env.sample` to `app/server/.env`. The backend loads it automatically via `python-dotenv`.

| Variable | Required | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | Yes (for ADW) | Required for Claude Code CLI and Anthropic LLM |
| `OPENAI_API_KEY` | Optional | For OpenAI LLM (takes priority over Anthropic) |
| `GITHUB_PAT` | Optional | GitHub personal access token for ADW |
| `CLAUDE_CODE_PATH` | Optional | Path to `claude` binary (default: `claude`) |
| `E2B_API_KEY` | Optional | For cloud sandbox agent execution |
| `CLOUDFLARED_TUNNEL_TOKEN` | Optional | For exposing webhook via cloudflared |
| `CLOUDFLARE_ACCOUNT_ID` | Optional | For R2 screenshot uploads |
| `CLOUDFLARE_R2_ACCESS_KEY_ID` | Optional | R2 credentials |
| `CLOUDFLARE_R2_SECRET_ACCESS_KEY` | Optional | R2 credentials |
| `CLOUDFLARE_R2_BUCKET_NAME` | Optional | R2 bucket for screenshots |
| `CLOUDFLARE_R2_PUBLIC_DOMAIN` | Optional | Public URL for R2 bucket |
| `GITHUB_REPO_URL` | ADW only | Target GitHub repository URL |
| `GITHUB_WEBHOOK_SECRET` | ADW webhook | For validating GitHub webhook signatures |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | Optional | Returns to root after each command |
| `ADW_DEBUG` | Optional | Enables verbose ADW output |

**IMPORTANT**: Never read, modify, or commit `.env` files. The `pre_tool_use.py` hook will block any attempt to access `.env` files (except `.env.sample`).

---

## API Endpoints (Backend)

All endpoints are under `/api/`:

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/upload` | Upload CSV/JSON/JSONL → SQLite table |
| `POST` | `/api/query` | Natural language query → SQL → results |
| `GET` | `/api/schema` | Get all tables and column schemas |
| `POST` | `/api/insights` | Statistical insights for table columns |
| `GET` | `/api/generate-random-query` | LLM-generated sample query |
| `GET` | `/api/health` | Health check with DB status |
| `DELETE` | `/api/table/{table_name}` | Delete a table |
| `GET` | `/docs` | Auto-generated FastAPI Swagger UI |

### LLM Provider Selection

`generate_sql()` and `generate_random_query()` in `core/llm_processor.py` route based on API key availability:
1. If `OPENAI_API_KEY` is set → use OpenAI (`gpt-4.1-mini`)
2. Else if `ANTHROPIC_API_KEY` is set → use Anthropic (`claude-3-haiku-20240307`)

---

## Security Architecture

SQL injection protection is critical to this project. All SQL operations must go through `core/sql_security.py`.

### Key Security Functions

```python
from core.sql_security import (
    validate_identifier,      # Validates table/column names
    escape_identifier,        # Escapes identifiers with SQLite [] notation
    execute_query_safely,     # Executes queries with safe identifier injection
    validate_sql_query,       # Checks for dangerous patterns
    check_table_exists,       # Safe table existence check
    SQLSecurityError,         # Exception for security violations
)
```

### Rules for New SQL Code

1. **Always** use `execute_query_safely()` for database operations — never raw `conn.execute(f"... {table_name} ...")`
2. Use `identifier_params=` for table/column names, `params=` for values
3. Use `allow_ddl=True` only for legitimate DDL operations (e.g., DROP TABLE on user request)
4. Validate identifiers with `validate_identifier()` before use
5. Never concatenate user input directly into SQL strings
6. SQL comments (`--`, `/* */`) are blocked in user queries

### What's Blocked

- DDL operations without `allow_ddl=True`: DROP, CREATE, ALTER, TRUNCATE
- DML write operations: DELETE, UPDATE, INSERT
- Multiple statement execution (`;` followed by SQL)
- SQL comment injection (`--`, `/*`, `*/`)
- Classic injection patterns (`' OR 1=1`, UNION SELECT, etc.)
- SQL keywords as identifiers (SELECT, DROP, DELETE, etc.)

---

## Frontend Architecture

The frontend is intentionally minimal — no framework, pure TypeScript compiled by Vite.

- `src/main.ts` — single file containing all UI logic, initialized on `DOMContentLoaded`
- `src/api/client.ts` — typed API client wrapping `fetch()`, proxied to backend via Vite
- `src/types.d.ts` — global TypeScript type declarations matching server Pydantic models
- No state management library; uses local variables within function scopes
- Input debouncing: 400ms delay before query execution; UI is disabled during queries
- Keyboard shortcut: `Cmd+Enter` / `Ctrl+Enter` to submit queries

### Vite Proxy

In development, `/api/*` requests are proxied to `http://localhost:8000` (configured in `vite.config.ts`). No CORS issues in dev mode.

---

## Data Models (Pydantic — `core/data_models.py`)

```python
QueryRequest(query: str, llm_provider: "openai"|"anthropic", table_name: Optional[str])
QueryResponse(sql: str, results: List[Dict], columns: List[str], row_count: int, execution_time_ms: float, error: Optional[str])
FileUploadResponse(table_name: str, table_schema: Dict[str, str], row_count: int, sample_data: List[Dict], error: Optional[str])
DatabaseSchemaResponse(tables: List[TableSchema], total_tables: int, error: Optional[str])
InsightsRequest(table_name: str, column_names: Optional[List[str]])
InsightsResponse(table_name: str, insights: List[ColumnInsight], generated_at: datetime, error: Optional[str])
RandomQueryResponse(query: str, error: Optional[str])
HealthCheckResponse(status: "ok"|"error", database_connected: bool, tables_count: int, uptime_seconds: float)
```

All error responses use `error: Optional[str] = None` pattern — endpoints return 200 with error field rather than HTTP error codes in most cases.

---

## File Processing

Supported upload formats (`core/file_processor.py`):

- **CSV** — parsed via pandas, columns lowercased and spaces→underscores
- **JSON** — must be a top-level array of objects; converted to DataFrame
- **JSONL** — newline-delimited JSON; nested objects are flattened using `NESTED_DELIMITER` (`__`) and list items use `LIST_INDEX_DELIMITER` (`_idx_`)
  - Two-pass: first pass discovers all field names, second pass creates consistent records with `None` for missing fields

Table names are sanitized: non-alphanumeric characters replaced with `_`, must start with letter or underscore, validated against SQL keywords.

---

## Testing

### Backend Tests

```bash
cd app/server
uv run pytest                              # All tests
uv run pytest tests/test_sql_injection.py  # Security tests
uv run pytest tests/core/ -v               # Unit tests with verbose output
```

Tests use:
- `pytest` fixtures for in-memory SQLite databases
- `unittest.mock.patch` to mock `sqlite3.connect`
- No external API calls in unit tests (LLM calls are mocked or skipped)

Test files:
- `tests/core/test_sql_processor.py` — SQL execution, schema retrieval, dangerous query blocking
- `tests/core/test_file_processor.py` — CSV/JSON/JSONL conversion
- `tests/core/test_llm_processor.py` — LLM prompt formatting
- `tests/test_sql_injection.py` — Comprehensive SQL injection protection tests

### E2E Tests (Playwright via MCP)

E2E tests are defined as Claude Code slash commands in `.claude/commands/e2e/`:
- `test_basic_query.md` — basic query execution flow
- `test_complex_query.md` — complex queries with filtering
- `test_random_query_generator.md` — random query generation
- `test_sql_injection.md` — SQL injection protection via UI
- `test_disable_input_debounce.md` — input debouncing behavior

Run E2E via Claude Code: `/test_e2e` or individual `/e2e:test_basic_query` etc.

---

## Claude Code Configuration

### Permissions (`.claude/settings.json`)

**Allowed by default:**
- `mkdir`, `uv`, `find`, `mv`, `grep`, `npm`, `ls`, `cp`, `chmod`, `touch`
- `Write` tool
- `./scripts/copy_dot_env.sh`

**Denied:**
- `git push --force` / `git push -f`
- `rm -rf` (also blocked at hook level)

### Hooks

All hooks run as uv scripts. They **never block** on errors (exit 0 on failure unless intentionally blocking):

| Hook | Trigger | Behavior |
|---|---|---|
| `pre_tool_use.py` | Before every tool call | Blocks `rm -rf` and `.env` file access (exits 2 to block) |
| `post_tool_use.py` | After every tool call | Logs tool usage to session log |
| `notification.py` | On notification events | Sends desktop notifications (with `--notify` flag) |
| `stop.py` | Session end | Chat-mode session cleanup (with `--chat` flag) |
| `subagent_stop.py` | Subagent session end | Subagent cleanup |
| `pre_compact.py` | Before context compaction | Pre-compaction handler |
| `user_prompt_submit.py` | User message submitted | Logs prompts (with `--log-only` flag) |

Session logs are written to `logs/{session_id}/`.

### Slash Commands

Available in `.claude/commands/`:
- `/start` — Start the application
- `/test` — Run test suite
- `/test_e2e` — Run E2E tests
- `/commit` — Generate git commit
- `/pull_request` — Create PR
- `/feature`, `/bug`, `/chore` — Plan issue types
- `/implement` — Implement a plan
- `/review` — Review implementation
- `/document` — Generate documentation
- `/patch` — Quick patch workflow
- `/prime` — Prime context with codebase
- `/install` — Install and setup
- `/prepare_app` — Prepare application
- `/classify_issue` — Classify GitHub issue
- `/classify_adw` — ADW workflow extraction
- `/conditional_docs` — Conditional documentation
- `/generate_branch_name` — Generate git branch name
- `/resolve_failed_test` — Fix failing tests
- `/resolve_failed_e2e_test` — Fix failing E2E tests

---

## ADW System

ADW automates the full GitHub issue development lifecycle using Claude Code CLI as the execution engine.

### Architecture

- **State**: Each workflow run has a unique 8-char `adw_id`; state persisted in `agents/{adw_id}/adw_state.json`
- **Composition**: Scripts are chainable via pipes (stdout JSON → stdin)
- **Agents**: Claude Code CLI sessions run in `agents/{adw_id}/{phase}/` with JSONL output

### ADW State (`ADWState`)

```python
adw_id: str          # Unique 8-char workflow ID
issue_number: int    # GitHub issue number
branch_name: str     # Git branch for changes
plan_file: str       # Path to plan spec file
issue_class: str     # "/chore", "/bug", or "/feature"
```

### Quick Start

```bash
cd adws/
export GITHUB_REPO_URL="https://github.com/owner/repo"
export ANTHROPIC_API_KEY="sk-ant-..."

uv run adw_plan_build.py 123          # Process issue #123 (plan + build)
uv run adw_sdlc.py 123                # Full SDLC (plan + build + test + review + document)
uv run adw_triggers/trigger_cron.py  # Monitor GitHub continuously
```

### Model Selection

Edit `adw_modules/agent.py` line ~129:
- `model="sonnet"` — default, faster and cheaper
- `model="opus"` — for complex tasks

### Branch Naming Convention

```
{type}-{issue_number}-{adw_id}-{slug}
# Example: feat-456-e5f6g7h8-add-user-authentication
```

---

## Key Conventions

### Python

- Use `logging` module (not `print`) for all output; logger per module via `logging.getLogger(__name__)`
- Log format: `[SUCCESS] message` or `[ERROR] message`
- Pydantic models for all request/response types
- Type hints on all function signatures
- `uv` for package management — use `uv add`/`uv remove`, not `pip install`
- Run scripts with `uv run python script.py` (not `python script.py` directly)

### TypeScript/Frontend

- No framework — vanilla DOM manipulation
- All types defined in `src/types.d.ts` as global `interface` declarations
- API calls go through `api` object in `src/api/client.ts`
- Disable UI (button + input) during async operations; re-enable in `finally` block
- Display errors via `displayError()` helper function

### Git

- Do not force push (`git push --force` is denied in settings)
- Do not delete files with `rm -rf` (blocked by hook)
- Commit messages should be descriptive and reference the issue/feature
- ADW uses semantic branch naming: `feat-`, `fix-`, `chore-`

### SQL Safety

- **Never** bypass `sql_security.py`; all database operations must use `execute_query_safely()`
- Use `identifier_params` for table/column names (not f-strings)
- Use `params` tuple for value parameters (not string formatting)

---

## Common Tasks

### Add a New API Endpoint

1. Add Pydantic models to `app/server/core/data_models.py`
2. Add business logic to appropriate `core/` module
3. Add route handler in `app/server/server.py`
4. Add corresponding TypeScript types to `app/client/src/types.d.ts`
5. Add API method to `app/client/src/api/client.ts`
6. Write tests in `app/server/tests/`

### Add a New File Format

1. Add converter function in `app/server/core/file_processor.py`
2. Add dispatch in `app/server/server.py` `upload_file()` handler
3. Update file extension check (currently `.csv`, `.json`, `.jsonl`)

### Run the Full Stack Locally

```bash
# 1. Setup (first time only)
cp .env.sample app/server/.env
# Edit app/server/.env with API keys

# 2. Install dependencies
cd app/server && uv sync --all-extras && cd ../..
cd app/client && bun install && cd ../..

# 3. Start
./scripts/start.sh
# Backend: http://localhost:8000
# Frontend: http://localhost:5173
# API Docs: http://localhost:8000/docs
```

### Troubleshooting

- **Backend won't start**: Check `.env` exists in `app/server/`, verify Python 3.10+
- **CORS errors**: Ensure backend is on port 8000, check `vite.config.ts` proxy
- **No query results**: Verify at least one `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` is set
- **SQLite errors**: Run `./scripts/reset_db.sh` to clear the database
- **Hook blocking tool**: Check `.claude/hooks/pre_tool_use.py` — it blocks `.env` access and `rm -rf`
