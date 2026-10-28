# AGENT_LOG.md

A record of how AI coding agents were directed on this project: what was asked, what came back, and what the human corrected and why.

**Who writes this:** the human only (see `CLAUDE.md`). Agents report deviations in their task summary; the human decides and records them here.

**Why it exists:** it is evidence that the design came first, that agents wrote the code, and that a person reviewed and steered every step.

---

## Summary

Fill this in at the end.

| Metric | Value |
|---|---|
| Tasks completed | _/11 |
| Agent(s) used | Claude Code / Copilot agent mode |
| Tasks accepted on first pass | _ |
| Tasks needing corrections | _ |
| Hard-rule violations caught (CLAUDE.md) | _ |
| Design changes triggered by the build | _ (see DESIGN.md §13) |

**Biggest lessons:** 2–3 bullets on what agents got wrong most often, and what instructions fixed it.

---

## How to log an entry

Copy the template below for each task. Keep each entry short: a few lines per heading is enough. Only log real corrections and decisions; skip trivial ones like renaming a variable.

**Correction categories** (makes patterns easy to spot later):

| Tag | Meaning |
|---|---|
| `RULE` | Broke a CLAUDE.md hard rule (e.g. a fallback token, filtering in the MCP server) |
| `DESIGN` | Deviated from DESIGN.md (wrong response shape, wrong error code) |
| `BUG` | Logic or test error |
| `SCOPE` | Did more than the task asked (extra features, extra dependencies) |
| `MISSING` | Skipped something the task required |
| `DOC` | Design was unclear, so I updated DESIGN.md or CLAUDE.md |

---

## Template

```markdown
### T_n: <task name>
**Date:** YYYY-MM-DD · **Agent:** Claude Code | Copilot · **Result:** Accepted | Accepted after fixes | Redone

**Prompt:**
> <the prompt you gave, or "standard task prompt">

**Agent output (summary):**
- <files created or changed, what it built>

**Corrections:**
- `TAG` <what was wrong> → <what I changed or asked for> (why)

**Decisions:**
- <any judgment call you made, and whether DESIGN.md was updated>

**Tests:** <pass/fail counts after fixes>
```

---

## Example entry

This is illustrative only. Delete it once real entries exist.

### T5: MCP server tools and resource
**Date:** 2026-10-28 · **Agent:** Claude Code · **Result:** Accepted after fixes

**Prompt:**
> Standard task prompt (T5).

**Agent output (summary):**
- Created `mcp_server/server.py`, `client.py`, and 3 tools plus a resource; added 6 tests.

**Corrections:**
- `RULE` Server used a default token (`tok_alice_hr`) when `USER_TOKEN` was missing, so it would "still work". → Removed it; the server now exits with an error. This would have silently given everyone HR access. (CLAUDE.md rule 1)
- `RULE` `get_employee` removed `salary` from results "for safety". → Removed; filtering belongs in the API only. Double filtering hides API bugs. (CLAUDE.md rule 2)
- `DESIGN` The tool description didn't mention `rows_excluded`. → Added, so the model tells Dave that matches exist outside his scope.

**Decisions:**
- Kept `describe_field` reading the YAML locally rather than via the API; metadata is public (DESIGN §12).

**Tests:** 6/6 passing; `grep tok_ mcp_server/` is clean.

---

## Entries

<!-- Add one entry per task, newest at the bottom. -->
