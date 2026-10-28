# TASKS.md

The build plan, derived from `docs/DESIGN.md`. Work **one task at a time**, in order. Each task ends with its acceptance checks passing and a stop for human review (see `CLAUDE.md`).

**Owner key:** 🤖 = agent builds and the human reviews · 👤 = the human does it

| # | Task | Owner | Plan hour | Status |
|---|---|---|---|---|
| T0 | Project scaffold | 🤖 | 2 | ☐ |
| T1 | Data dictionary | 👤 + 🤖 | 2 | ☐ |
| T2 | Database schema and seed | 🤖 | 2 | ☐ |
| T3 | API: auth and policy module | 🤖 | 3 | ☐ |
| T4 | API: endpoints | 🤖 | 3 | ☐ |
| T5 | MCP server: tools and resource | 🤖 | 4 | ☐ |
| T6 | Audit logging | 🤖 | 5 | ☐ |
| T7 | Denied-requests summary script | 🤖 | 5 | ☐ |
| T8 | Client configs (VS Code and Claude Desktop) | 🤖 + 👤 | 6 | ☐ |
| T9 | End-to-end demo and evidence | 👤 | 7 | ☐ |
| T10 | README | 🤖 + 👤 | 7 | ☐ |

**How to start each task in Claude Code:**

> Read CLAUDE.md and docs/DESIGN.md. Do task T_n from docs/TASKS.md only. Stop when its acceptance checks pass and report back.

After each task, record what you corrected in `docs/AGENT_LOG.md`.

---

## T0: Project scaffold

**Goal:** an empty, runnable project skeleton.

- `pyproject.toml` with the dependencies listed in CLAUDE.md (managed with uv)
- Folders matching the CLAUDE.md layout, each with an `__init__.py` where needed
- A `.gitignore` covering `.venv/`, `__pycache__/`, `data/*.db`, `audit/audit.jsonl`, and `.env`

**Acceptance**
- [ ] `uv sync` succeeds
- [ ] `uv run pytest` runs (0 tests is fine)
- [ ] Layout matches CLAUDE.md

---

## T1: Data dictionary

**Goal:** `data_dictionary.yaml`, matching DESIGN.md §6 exactly.

👤 You own the content: check every field's description, owner, sensitivity, and allowed roles. 🤖 The agent may draft it from §6.

- 10 fields: `id` through `performance_rating`
- `home_address` deliberately **absent**
- A short header comment explaining the default-deny rule

**Acceptance**
- [ ] It parses with `yaml.safe_load`
- [ ] Every field has all 6 keys: `name`, `description`, `type`, `owner`, `sensitivity`, `allowed_roles`
- [ ] Sensitivity values are only `public`, `internal`, or `confidential`
- [ ] Roles are only `HR`, `Engineer`, or `Manager`
- [ ] A test (`tests/test_dictionary.py`) asserts the checks above

---

## T2: Database schema and seed

**Goal:** `api/db.py` (connection helper) and `api/seed.py` (creates `data/company.db` from scratch on every run).

**Tables**
- `employees`: the 10 dictionary fields, plus `home_address`
- `roles`: `name`, `row_scope` (`all` | `own_department`)
- `users`: `username`, `role`, `employee_id`
- `tokens`: `token`, `username`

**Seed data**
- About 12 employees across Engineering, Sales, and HR
- Must include:

  | Person | Department | Notes |
  |---|---|---|
  | Alice | HR | |
  | Bob | Engineering | |
  | Dave | Engineering | Manager; `manager_id` of the other engineers |
  | Carol Diaz | Engineering | |
  | Frank Osei | Sales | |

- Realistic but fake values for salary, rating, and address
- Users: `alice` (HR), `bob` (Engineer), `dave` (Manager)
- Tokens: `tok_alice_hr`, `tok_bob_eng`, `tok_dave_mgr`

**Acceptance**
- [ ] Running the seed twice gives the same database (it is idempotent)
- [ ] Tests confirm Carol and Frank exist, Dave is an Engineering manager, and every user maps to an employee

---

## T3: API auth and policy module

**Goal:** `api/auth.py` and `api/policy.py`, unit-tested on their own before any endpoints exist.

**`auth.resolve(token)`**
- Returns `CallerContext(username, role, department, row_scope)`
- Raises `InvalidToken` on an unknown token

**`policy` (loads the dictionary once)**
- `allowed_fields(role)`: the fields this role may see
- `redact(row, role)`: returns the visible fields plus the list of `redacted_fields`
- `row_filter_sql(ctx)`: returns a SQL `WHERE` fragment and its parameters (`all` gives no filter)
- `searchable_columns(role)`: the subset of `name`, `department`, `title` the role may see

**Acceptance**
- [ ] No field names are hard-coded in `policy.py`; everything comes from the YAML
- [ ] `home_address` never appears in `allowed_fields` for any role
- [ ] Tests assert the exact allowed-field set for each of the 3 roles, matching DESIGN §5
- [ ] Dave's row filter restricts to Engineering; Alice's and Bob's apply no filter

---

## T4: API endpoints

**Goal:** `api/main.py` with `GET /health`, `GET /employees?q=`, and `GET /employees/{id}`, following DESIGN §7.

- Bearer-token dependency using `auth.resolve`
- Row scope applied in SQL
- Search runs only over `searchable_columns(role)`
- `rows_excluded` is computed with a second, count-only query
- Errors: 401 `invalid_token`, 403 `out_of_scope`, 404 `not_found`

**Acceptance** (in `tests/test_api.py`, using `TestClient`)
- [ ] `/employees/{carol}` returns: salary for Alice; no salary but a rating for Dave; neither for Bob
- [ ] `/employees/{frank}` as Dave returns 403 `out_of_scope`; as Alice and Bob it returns 200
- [ ] `?q=Sales` as Dave returns 0 rows with `rows_excluded` > 0
- [ ] No response for any user contains `home_address`
- [ ] A missing or bad token returns 401
- [ ] `meta` contains the correct user, role, and row_scope

---

## T5: MCP server tools and resource

**Goal:** `mcp_server/server.py` and `mcp_server/client.py`, following DESIGN §8.

- Tools: `search_employees(query)`, `get_employee(id)`, `describe_field(field_name)`
- Resource: `datadict://employee`
- Reads `USER_TOKEN` at startup and exits with a clear message if it is missing
- Maps API errors to readable tool errors (§8)
- Tool descriptions explain `redacted_fields` and `rows_excluded` to the model

**Acceptance**
- [ ] Works in MCP Inspector with each of the 3 tokens
- [ ] Tests (with the API mocked, or run in-process) cover: token forwarded in the header, missing token fails, 401 and 403 mapped correctly
- [ ] `grep -rn "tok_" mcp_server/` finds nothing
- [ ] `grep -rniE "allowed_roles|role ==|salary" mcp_server/` finds nothing outside tool descriptions

---

## T6: Audit logging

**Goal:** `mcp_server/audit.py`, which appends one JSON line per tool call (DESIGN §9).

- Covers success and failure, and the `describe_field` tool too
- Fields: `ts`, `user`, `role`, `tool`, `args`, `status`, `fields_returned`, `fields_denied`, `row_scope`, `rows_excluded`
- On a 401 there is no `meta`, so `user` is logged as `"unknown"`

**Acceptance**
- [ ] Each tool call produces exactly one line
- [ ] A test asserts that no token string and no field value (for example, a known salary number) ever appears in the log

---

## T7: Denied-requests summary script

**Goal:** `audit/summarize_denied.py`, kept to about 20–40 lines and using the standard library only.

**Output**
1. Top denied fields, by count
2. For each denied field, the users who requested it, with counts
3. Out-of-scope requests per user, plus total `rows_excluded`

**Acceptance**
- [ ] Runs against a sample log fixture in `tests/fixtures/` and prints the expected summary
- [ ] Handles an empty or missing log gracefully

---

## T8: Client configs

**Goal:** working MCP configs for all 3 users in both clients.

- 🤖 Generate `configs/vscode.mcp.json` and `configs/claude_desktop_config.json`, with one server entry per user: `employees-alice`, `employees-bob`, `employees-dave`
- Each entry passes a different `USER_TOKEN` in `env`
- 🤖 **Check the current config format** for each client in its official documentation before writing. VS Code and Claude Desktop use different top-level keys.
- 👤 Copy the configs into the real locations, fix absolute paths (Claude Desktop needs the full path to `uv`), and restart the clients

**Acceptance**
- [ ] All 3 servers show as connected in both clients
- [ ] The 3 tools are listed in each

---

## T9: End-to-end demo 👤

**Goal:** prove the acceptance criteria in DESIGN §11, using the scenarios in `docs/TESTING.md`.

- Run every scenario in both clients
- Take screenshots or a short recording, saved to `docs/demo/`
- Run the summary script and include its output

**Acceptance**
- [ ] Every DESIGN §11 criterion is ticked off, with evidence

---

## T10: README

**Goal:** `README.md`, the public face of the repo.

🤖 The agent drafts; 👤 you edit the pitch and the requirement-mapping table.

**Sections**
1. Pitch
2. Architecture diagram
3. Quickstart
4. Client setup
5. Demo, with screenshots
6. How it maps to the requirements
7. Links to the design doc and agent log

**Acceptance**
- [ ] A fresh clone plus the quickstart steps reaches a working demo
