# Decree 53/2022/NĐ-CP §17 — Data Localization Middleware

Implementation of Article 17 data localization requirements for the UOIS `hk2026` Docker stack.
No original UOIS files were modified — all changes are additive.

---

## What was built

### 1. Data classification & routing (`server/data_router.py`)

Every GraphQL request is classified by scanning the query string for keywords:

| Category | Keywords | Storage |
|---|---|---|
| `user_profile` | user, group, role, membership | MySQL Local_DB (Vietnam) |
| `app_metadata` | event, project, facility, plan | PostgreSQL Global_DB |

For every request, an `ip_logs` row is written to MySQL.
For `user_profile` mutations, a `user_profile_audit` row is also written.

Classification is per-request — a single request cannot be split across databases.

---

### 2. Proxy middleware (`server/gqlproxy_localized.py`)

Copy of `gqlproxy.py` with three additions:

- Fixed a `None`-header bug that caused HTTP 500 on every unauthenticated request (`aiohttp` raises `TypeError` if a header value is `None`)
- Calls `DataRouter.process()` after each upstream response
- Attaches `X-Data-Category` and `X-Decree53-Routed` headers to the response for observability

---

### 3. FastAPI entrypoint (`server/main_localized.py`)

Copy of `main.py` with three modifications:

1. Imports `gqlproxy_localized` instead of `gqlproxy`
2. Adds MySQL connection pool to the FastAPI lifespan
3. Registers `/decree53/health` endpoint

A `sys.modules` stub prevents duplicate Prometheus metric registration on reload.

---

### 4. Local database schema (`server/schema_local_db.sql`)

MySQL schema for `local_db`:

- `ip_logs` — logs every request (IP, endpoint, GQL operation, category, status)
- `user_profile_audit` — snapshots every user_profile mutation payload
- 6 mirror tables (schema only): `users_mirror`, `groups_mirror`, `memberships_mirror`, `roles_mirror`, `roletypes_mirror`, `grouptypes_mirror`

---

### 5. Docker Compose override (`docker-compose.hk2026.decree53.yml`)

Run with:

```bash
docker compose \
  -f docker-compose.hk2026.yml \
  -f docker-compose.hk2026.decree53.yml \
  up --build
```

Adds two services:

- `local_mysql` — MySQL 8.0, exposed on port `3307`, database `local_db`
- `mysql_init` — one-shot container that applies `schema_local_db.sql` on first start

Sets `GQL_PROXY=http://gql_ug:8000/gql` to bypass Apollo Federation (v1/v2 incompatibility).

---

### 6. Integration tests (`scripts/test_decree53.py`)

8 tests, all passing:

| # | Test | Expected |
|---|---|---|
| 1 | `/decree53/health` | MySQL connected |
| 2 | `users` query | `X-Decree53-Routed: local` |
| 3 | `groups` query | `X-Decree53-Routed: local` |
| 4 | `events` query | `X-Decree53-Routed: global` |
| 5 | `projects` query | `X-Decree53-Routed: global` |
| 6 | Mixed query (users + events) | `user_profile` wins → `local` |
| 7 | Authenticated user mutation | audit row written |
| 8 | MySQL `ip_logs` row count | > 0, both categories present |

Run:

```bash
python scripts/test_decree53.py
```

Requires the stack to be running. Optional env overrides: `API_URL`, `LOCAL_DB_HOST`, `LOCAL_DB_PORT`.

---

### 7. Diagrams & report

| File | Description |
|---|---|
| `scripts/er_diagram.html` | ER diagram — Local_DB vs Global_DB tables, ACTIVE/SCHEMA badges |
| `scripts/pipeline_diagram.png` | Request routing pipeline diagram |
| `Decree53_Report.pdf` | Full technical report |
