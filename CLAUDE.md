# CLAUDE.md

Instructions for AI coding agents working in this repo.

## Source of truth

- **`docs/DESIGN.md` defines what to build.** Read it fully before writing any code. If something here conflicts with it, DESIGN.md wins.
- **`docs/TASKS.md` defines the order of work.** Do one task at a time, meet its acceptance criteria, then stop and summarize for review. Do not start the next task until the human approves.
- If the design is ambiguous or seems wrong, **ask. Do not improvise.** Propose the change and wait.

## Hard rules (never break these)

1. **No privileged credentials in the MCP server.** No admin token, service account, fallback token, or default token. It is only ever given the user's `USER_TOKEN` from its environment, and it fails to start if that is missing.
2. **No permission logic in the MCP server.** It does not filter fields or rows, check roles, or read `allowed_roles` to decide anything. All enforcement happens in the API.
3. **The data dictionary is the only field-policy source.** The API loads `data_dictionary.yaml` at startup. Never hard-code field lists or role-to-field mappings in Python.
4. **Default deny.** Any column missing from the dictionary is never returned. That includes `home_address`, which must never appear in any response, log, or test fixture output.
5. **Row scope is applied in SQL**, in the `WHERE` clause, never by filtering in Python after the query.
6. **Search only matches visible fields.** Build the searchable-columns list per request from the caller's allowed fields.
7. **The audit log contains field names, never field values or tokens.**
8. **Read-only.** No write endpoints and no write tools.

If a task seems to require breaking a rule, stop and explain why instead of doing it.

## Stack

- Python 3.11+, managed with `uv`
- **API:** FastAPI, uvicorn, sqlite3 from the standard library (no ORM), PyYAML
- **MCP server:** the official MCP Python SDK (`from mcp.server.fastmcp import FastMCP`), stdio transport, `httpx` for API calls
- **Tests:** pytest, plus FastAPI `TestClient`

Do not add other dependencies without asking.

## Repo layout

```
api/            main.py (FastAPI app), auth.py (token → user/role/dept),
                policy.py (dictionary loading + field/row filtering), db.py, seed.py
mcp_server/     server.py (tools + resource), client.py (httpx wrapper), audit.py
audit/          audit.jsonl (gitignored), summarize_denied.py
configs/        example client configs for alice / bob / dave
tests/          test_api.py, test_mcp_server.py
data_dictionary.yaml
```

## Commands

```bash
uv sync                                          # install dependencies
uv run python -m api.seed                        # (re)create data/company.db
uv run uvicorn api.main:app --reload --port 8000 # run the mock API
USER_TOKEN=tok_bob_eng uv run python -m mcp_server.server   # run the MCP server (stdio)
uv run pytest                                    # run tests
uv run python audit/summarize_denied.py          # audit summary
npx @modelcontextprotocol/inspector uv run python -m mcp_server.server  # manual testing (set USER_TOKEN)
```

## Conventions

- Type hints everywhere. Keep functions small. Docstrings on public functions.
- Tool descriptions are written for the AI model. They must explain `redacted_fields` and `rows_excluded`, and tell the model to say when data is restricted rather than guess.
- API responses always use the envelope from DESIGN.md §7: `data`, `redacted_fields`, `meta`.
- Configuration comes from environment variables: `API_BASE_URL` (default `http://localhost:8000`), `USER_TOKEN`, and `AUDIT_LOG_PATH` (default `audit/audit.jsonl`).
- Mock tokens appear only in `api/seed.py`, `configs/`, and tests. Nowhere in `mcp_server/`.

## Testing expectations

- Every task adds or updates tests.
- Permission tests run the **same request as Alice, Bob, and Dave** and assert on the differences. These tests are the core of the project.
- Before reporting a task done: run `uv run pytest` and paste the result, and run `grep -rn "tok_" mcp_server/`, which must return nothing.

## When you finish a task

Reply with:
1. what you built (files changed)
2. the test results
3. any deviation from DESIGN.md and why
4. anything you were unsure about

The human records corrections in `docs/AGENT_LOG.md`. Do not edit that file yourself.
