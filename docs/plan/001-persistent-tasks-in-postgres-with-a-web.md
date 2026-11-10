---
plan_id: 001-persistent-tasks-in-postgres-with-a-web
spec: 001-persistent-tasks-in-postgres-with-a-web
feature: Persistent tasks in Postgres with a web task list
gitops_app: ika100-demo/taskboard9-gitops
status: draft
repos:
  - id: taskboard9-gitops
    shape: gitops-app
    summary: Declare the postgres resource, give taskboard9-api its connection and wire taskboard9-web to the API
    acs: [AC-001.1, AC-001.2, AC-001.4, AC-001.5, AC-001.6]
    gitops:
      - {resource: postgres}
      - {uses: postgres, component: taskboard9-api}
      - {env: {API_URL: 'http://taskboard9-api'}, component: taskboard9-web}
    depends_on: []
    done: false
  - id: taskboard9-api
    shape: service-python
    summary: Store tasks in Postgres so they survive restarts and are shared by every instance
    acs: [AC-001.1, AC-001.2, AC-001.3]
    criteria:
      - "Given a task created with POST /tasks {\"title\": \"Buy milk\", \"due_date\": \"2026-12-01\"} (201), when every API process is stopped and started again against the same database, then GET /tasks answers 200 and contains a task with the same id, title \"Buy milk\" and due_date \"2026-12-01\" (system AC-001.1)"
      - "Given an empty database, tasks \"A\", \"B\" and \"C\" created in that order and the API then restarted, when task \"D\" is created and a client sends GET /tasks, then the response is 200 with exactly A, B, C, D in that order, each with id, title and due_date as returned by its POST (system AC-001.2)"
      - "Given two API instances using the same database, when task \"A\" is created through the first and then task \"B\" through the second, then GET /tasks on either instance lists A before B with identical fields (system AC-001.1, AC-001.2)"
      - "Given the database is unreachable (or DATABASE_URL is not set), when a client sends GET /tasks or a valid POST /tasks, then the response arrives within 5 seconds with status 503 and a JSON body {\"detail\": <non-empty string>}, GET /health and GET /ready still answer 200, and once the database is reachable GET /tasks does not contain the task of that POST (system AC-001.3)"
      - "Given the database was unreachable while the API kept running, when it becomes reachable again, then within 30 seconds GET /tasks answers 200 and a valid POST /tasks answers 201, without the API process being restarted, also when the API process was started while the database was unreachable (system AC-001.3)"
      - "Given the database is reachable, when a client sends the requests of this repo's spec 001 (valid and invalid POST /tasks, GET /tasks), then the status codes, fields, 422 bodies and oldest-first order are exactly those of the Contract, and a 422 stores no row (system AC-001.1, AC-001.2)"
    arguments: |
      Supersedes AC-001.7 of this repo's spec 001 ("empty after restart"): withdraw it there with
      `/component:spec --amend 001` and replace its test; do not leave a failing or skipped test behind.
      Config: DATABASE_URL (plus PGHOST, PGPORT, PGDATABASE, PGUSER, PGPASSWORD) comes from the system's
      postgres resource (Secret taskboard9-postgres-app). Suggested stack: psycopg 3 with psycopg_pool (sync,
      the routes are sync), schema applied by the service itself with CREATE TABLE IF NOT EXISTS under
      pg_advisory_lock, never blocking startup. Tests need a real PostgreSQL (a devbox recipe or
      testcontainers); keep an in-process restart test via TestClient(create_app()) on the same database.
    depends_on: []
    done: false
  - id: taskboard9-web
    shape: web-nextjs
    summary: Task board start page listing all tasks with a create form
    acs: [AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8]
    criteria:
      - "Given API_URL points at an API whose GET /tasks answers 200 with [{\"id\": \"1\", \"title\": \"A\", \"due_date\": \"2026-12-01\"}, {\"id\": \"2\", \"title\": \"B\", \"due_date\": null}], when a person opens the start page /, then it shows a level-1 heading \"Tasks\" and a list with \"A\" above \"B\", \"A\" showing \"2026-12-01\" and \"B\" showing \"No due date\" (system AC-001.4)"
      - "Given GET /tasks answers 200 with [], when a person opens /, then it shows the text \"No tasks yet.\" in place of the list, together with the create form (field \"Title\", field \"Due date\", button \"Add task\") (system AC-001.5)"
      - "Given the start page is open, when a person enters the title \"Buy milk\" and the due date 2026-12-01 and presses \"Add task\", then the API receives POST /tasks {\"title\": \"Buy milk\", \"due_date\": \"2026-12-01\"}, and after its 201 the returned task appears as the last list entry with \"2026-12-01\" without a page reload (replacing \"No tasks yet.\" when the list was empty) and both fields are empty; with the due date left empty the body carries \"due_date\": null and the entry shows \"No due date\" (system AC-001.6)"
      - "Given the start page is open, when a person submits a title that is empty or whitespace only, or longer than 200 characters after trimming, then the page shows \"Title is required.\" or \"Title must be at most 200 characters.\" respectively at the title field, keeps the entered title and due date, and sends no POST /tasks to the API (system AC-001.7)"
      - "Given GET /tasks cannot be reached, gives no answer within 8 seconds, or answers anything but 200 with a JSON array (for example 503 or 500), when a person opens /, then the page shows \"Tasks could not be loaded. Please try again later.\" instead of the list and never \"No tasks yet.\" (system AC-001.8)"
      - "Given the start page is open, when a person submits a valid task and POST /tasks cannot be reached, gives no answer within 8 seconds, or answers anything but 201 (for example 503), then the page shows \"The task could not be saved. Please try again.\", keeps the entered title and due date, and the list is unchanged (system AC-001.8)"
    arguments: |
      The start page replaces the template's landing content and must render per request (dynamic, fetch
      with cache: "no-store"), never prerendered at build time. API_URL is server-side only (never
      NEXT_PUBLIC_*), read at request time, default http://localhost:8080; document it in docs/env-vars.md.
      Keep the template's smoke test (a level-1 heading); add a read-only @smoke (heading "Tasks", no load
      error) and a @smoke @writes test creating a smoke-<uuid> task (dev only).
    depends_on: []
    done: false
gitops_pin: []
---

The feature takes three repos: the API moves its store to Postgres, the web UI gets the task board, and this
gitops-app repo declares the database and wires both. The API's HTTP contract does not change (system spec A2),
so the API and the web build in parallel against the contract below; nothing has to wait for anything else.

## taskboard9-gitops — Declare the postgres resource, give taskboard9-api its connection and wire taskboard9-web to the API

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.4, AC-001.5, AC-001.6 · **Depends on:** []

Persistence (AC-001.1, AC-001.2) needs a database that outlives the API pods: one CloudNativePG `postgres` resource
per environment, used only by `taskboard9-api` (system spec A1). `uses` gives the API `DATABASE_URL` and `PG*` from
the Secret `taskboard9-postgres-app`. The web reaches the API inside the namespace through its Service
(`taskboard9-api`, port 80), so `API_URL: http://taskboard9-api`; without it the start page cannot list or create
tasks (AC-001.4 to AC-001.6). Both components already have `expose` and run only in `dev`; no new exposure.
The operations can land first: the API answers 503 until it has a database, and the web ignores `API_URL` until
its new image runs.

## taskboard9-api — Store tasks in Postgres so they survive restarts and are shared by every instance

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.3 · **Depends on:** []

The in-memory store is replaced by a Postgres store; the endpoints, fields, validation and 422 bodies of the API's
spec 001 stay as they are. The API owns all of AC-001.1 to AC-001.3: durable rows, creation order across restarts
and instances (a hidden, database-assigned sequence; the task gains no visible field), and a 503 instead of a
crash or a hang when the database is away, with recovery by reconnecting on the next request. `/ready` must not
depend on the database: if it did, Kubernetes would take the pod out of the Service and callers would get a
gateway error instead of the API's own 503 (AC-001.3). Test concerns: restart is a new process or a new
`create_app()` against the same database; "unreachable" is simulated by stopping the test database or pointing at
a closed port; the 30-second recovery is measured from the moment the database accepts connections again.

## taskboard9-web — Task board start page listing all tasks with a create form

**Shape:** `web-nextjs` · **Criteria:** AC-001.4 to AC-001.8 · **Depends on:** []

The start page `/` becomes the board (A5): heading, list or empty or error text, and the create form on the same
page. The list is fetched server-side from `API_URL` on every request; the form is a client component that posts
to a route handler of the web (`POST /api/tasks`), which validates with the API's rules, forwards to the API and
returns the created task, so the browser never needs the API's address and the new task is appended without a
reload (AC-001.6). Client-side validation (A7) keeps invalid titles from reaching the API (AC-001.7); the route
handler repeats it. Test concerns: Vitest with a mocked `fetch` for every API answer of the Contract, including
timeouts; Playwright e2e against a fake API (or a running taskboard9-api) for the no-reload append and kept values.

## Contract

The provider is `taskboard9-api`; the consumer is `taskboard9-web` (server side only). Base URL: the env var
`API_URL` of the web (`http://taskboard9-api` in the cluster, `http://localhost:8080` locally); no trailing slash,
paths are appended as written. All bodies are `application/json; charset=utf-8`. No authentication (A9).

**Task object** (unchanged from taskboard9-api spec 001)

| Field | Type | Notes |
|---|---|---|
| `id` | string | server-assigned, non-empty, opaque, unique; clients never parse it (UUID text today) |
| `title` | string | stored trimmed, 1 to 200 characters (Unicode code points) |
| `due_date` | string `YYYY-MM-DD` or `null` | calendar date, no time or zone; past dates allowed |

A task has exactly these three fields. Creation order is kept by the API internally and is not exposed.

### `GET /tasks`

- `200`: JSON array of every stored task, oldest first by creation (across restarts and instances), each exactly
  as returned by its `POST`; `[]` when there is none. No query parameters, no pagination.
- `503`: the database is unreachable or not configured; body `{"detail": "Database unavailable"}`.

### `POST /tasks`

Request body: `{"title": "Buy milk", "due_date": "2026-12-01"}`.

- `title` (required): JSON string; trimmed; 1 to 200 code points after trimming.
- `due_date` (optional): `YYYY-MM-DD` string of a real calendar date, or `null`, or absent. Unknown fields,
  including `id`, are ignored.

Responses:

- `201`: the created Task object; the returned title is the trimmed one; `due_date` is `null` when it was absent
  or `null`.
- `422`: validation failure, nothing stored; FastAPI's standard body
  `{"detail": [{"loc": ["body", "title"], "msg": "...", "type": "..."}]}`, one entry per invalid field
  (`loc` `["body", "title"]` or `["body", "due_date"]`).
- `503`: the database is unreachable or not configured; nothing stored; body `{"detail": "Database unavailable"}`.

**What the web sends.** The web trims the title and checks it before sending: empty after trimming gives
"Title is required.", more than 200 code points after trimming (`[...title].length`, not `title.length`) gives
"Title must be at most 200 characters."; then nothing is sent. The due date comes from an `<input type="date">`
whose value is already `YYYY-MM-DD`; an empty field is sent as `"due_date": null`. The body is exactly
`{"title": <trimmed title>, "due_date": <date or null>}`.

**What the web shows** (A6, A8): the heading "Tasks" (level 1); each entry shows its title and its `due_date`
verbatim (`YYYY-MM-DD`), or "No due date" when it is `null`; entries in the order of the array; "No tasks yet."
when the array is empty; "Tasks could not be loaded. Please try again later." when the list cannot be loaded;
"The task could not be saved. Please try again." when a create fails; the field messages above, shown at the
title field. The form fields are labelled "Title" and "Due date", the button is "Add task". After a 201 the
returned task (not the typed values) is appended as the last entry and both fields are cleared. When the list
failed to load, the form stays usable; a successful create then clears the fields and the load error stays until
the page is reloaded.

**Web-internal route** (not called by other repos, stated so both sides agree on the behaviour):
`POST /api/tasks` on the web takes the same body as the API, answers `201` with the API's Task object, `422`
with `{"field": "title", "message": "<one of the two field messages>"}` when its own check fails, and `502` with
`{"message": "The task could not be saved. Please try again."}` for every API failure.

**Database (inside taskboard9-api only).** Table `tasks`: `seq bigint generated always as identity primary key`,
`id uuid not null unique`, `title text not null` (check 1 to 200 characters), `due_date date null`. `GET /tasks`
orders by `seq`. The API creates the table itself, idempotently and safely with several instances starting at
once (`CREATE TABLE IF NOT EXISTS` under an advisory lock), at startup when the database is reachable and
otherwise before the first request that reaches it. The database starts empty (A3).

### Errors

- Error bodies are JSON with a `detail` key: an array of field entries for `422` (FastAPI's shape), a string for
  `503`. The `detail` text is informative; no caller matches on it.
- The web never shows the API's error text to a person; it shows only the texts listed above.
- `GET /tasks`: anything other than `200` with a JSON array (any 4xx or 5xx, a body that is not a JSON array,
  a connection error, a timeout) is a load failure: the page shows "Tasks could not be loaded. Please try
  again later." and never "No tasks yet.". Entries with missing fields are not repaired; the whole load fails.
- `POST /tasks`: anything other than `201` with a Task object (including `422`, which the web's own check
  should make impossible, any other 4xx, any 5xx, a connection error, a timeout) is a save failure: the page
  shows "The task could not be saved. Please try again.", keeps the entered values and leaves the list unchanged.
  The web does not retry a `POST` by itself, so a lost answer can never create the task twice.
- The API answers `500` only for a bug; a database that is down, refusing connections, not configured, or too
  slow (over its timeouts below) is always `503`, with nothing stored.

### Timeouts

- **Web to API** (`GET /tasks`, `POST /tasks`, server side): 8 seconds each (`AbortSignal.timeout(8000)`); on
  expiry the web treats the call as failed (load or save failure above). No retries.
- **Browser to web** (`POST /api/tasks`): the client aborts after 10 seconds and shows the save failure; the
  submit button is disabled while a request is in flight so one press sends one request.
- **API to Postgres**: connect timeout 2 seconds, pool checkout timeout 2 seconds, `statement_timeout` 3 seconds;
  so the API answers `503` within 5 seconds when the database is down or slow, before the web's 8 seconds run out.
- **Recovery**: the API keeps no "database down" state; every request tries the pool, which replaces broken
  connections, so requests succeed again as soon as the database accepts connections, well within the 30 seconds
  of AC-001.3. Liveness (`/health`) and readiness (`/ready`) never touch the database.
- **Startup**: the API starts and becomes ready even when the database is unreachable (task endpoints answer
  `503` until it is reachable); the web starts without the API (the page shows the load failure).

## gitops-app PR

One PR in this repo runs the three operations of the `taskboard9-gitops` entry, in order:
`cplat resource add postgres`, then `cplat component set taskboard9-api` with `uses: [postgres]`, then
`cplat component set taskboard9-web` with `env: {API_URL: http://taskboard9-api}`, followed by `devbox run render`
and `devbox run validate`. It can merge before or after the two component PRs.

No `gitops_pin` entries: both components run only in `dev`, whose pins `/system:ship` lands on `compose/<spec>` (spec 144), and
running the system in staging or prod is a non-goal of this spec (promotion is a separate step). Libraries: none.

After Argo reconciles `taskboard9-dev`, verify:

- the CloudNativePG cluster of the `postgres` resource is healthy and the Secret `taskboard9-postgres-app` exists;
- the `taskboard9-api` pod has `DATABASE_URL` and `PG*` from that Secret, and `GET /tasks` on
  `taskboard9-api.taskboard9-dev.localhost` answers `200`;
- the `taskboard9-web` pod has `API_URL=http://taskboard9-api`, and `taskboard9-web.taskboard9-dev.localhost`
  shows the heading "Tasks" without the load error;
- a task created on the web page is still listed after `kubectl rollout restart deployment/taskboard9-api`;
- both components' dev smoke runs pass.
