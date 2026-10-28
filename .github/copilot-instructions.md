# Copilot instructions

This repo is a permission-aware MCP server demo. The full rules for AI agents live in **`CLAUDE.md`** at the repo root. They apply to Copilot equally, so read it before suggesting or generating code.

- **What to build:** `docs/DESIGN.md` (the source of truth)
- **What order to build it in:** `docs/TASKS.md`. Do one task at a time, then stop for review.

## Non-negotiables (summary of CLAUDE.md)

1. **The MCP server holds no credentials of its own.** It uses only `USER_TOKEN` from its environment, with no admin, service, or fallback token. It fails to start if the token is missing.
2. **The MCP server contains no permission logic.** It never filters fields or rows or checks roles. The API enforces everything.
3. **Field policy comes only from `data_dictionary.yaml`.** Never hard-code field lists or role-to-field maps.
4. **Default deny.** Columns not in the dictionary (e.g. `home_address`) are never returned.
5. **Row scope is applied in SQL** `WHERE` clauses, not in Python after the query.
6. **Search only matches fields the caller can see.**
7. **Audit logs record field names only**, never values or tokens.
8. **Read-only.** No write endpoints or tools.

If a request would break one of these, say so instead of doing it.

## Stack

Python 3.11+ with `uv`, FastAPI, sqlite3 (no ORM), PyYAML, the official MCP Python SDK (`FastMCP`, stdio), httpx, and pytest. Don't add other dependencies without asking.

## When using agent mode

- Run `uv run pytest` before reporting a task done.
- `grep -rn "tok_" mcp_server/` must return nothing.
- Report: files changed, test results, any deviation from DESIGN.md, and open questions.
