---
name: backend
description: Server-side conventions across this estate — the Go modular monolith (Survey-Forms), Django/DRF and FastAPI (Exam Studio), FastAPI+SQLite (CleverClass), and Laravel (phd-monitoring). Use when writing or changing API handlers, business logic, persistence, migrations, background jobs, or CLIs in any of these repos.
---

# Backend work

Match the repo you are in. These stacks have little in common with each other, so
the first job is knowing which one you are touching — see `emerge-context`.

**Verified 2026-08-30** against the files cited.

| Repo | Stack | Data |
|---|---|---|
| `Survey-Forms/backend` | **Go 1.27**, stdlib `net/http` | PostgreSQL via `pgx/v5`, goose migrations |
| `lms-paper-orchestrator` | Django + DRF | PostgreSQL |
| `lms-questionbank-engine` | FastAPI | |
| `lms-retrieval-engine` | FastAPI | BM25 + BGE dense + cross-encoder rerank |
| `cleverclass.ai/backend` | FastAPI 0.115 | **SQLite (WAL), hand-written SQL, no ORM** + ChromaDB |
| `phd-monitoring/server` | **Laravel 11 / PHP 8.2** + Sanctum | MySQL, 52 Eloquent models |

Always read the repo's own `CLAUDE.md` first — there are 42 across the estate.

## Go — `Survey-Forms/backend` (the flagship)

A modular monolith. `cmd/` holds **four binaries**, all built static into one
image (`CGO_ENABLED=0`):

| Binary | Purpose |
|---|---|
| `cmd/server` | the HTTP service |
| `cmd/migrate` | migrations, run as a separate deploy step |
| `cmd/seed` | seed data |
| `cmd/adminctl` | admin operations |

`internal/` is the domain boundary. Do not add a package outside it without a
reason:

```
admin  analytics  api  config  connect  domain  geo  httpx  mailer
media  privacy  ratelimit  response  session  store  testsupport
```

**Routing is stdlib `http.NewServeMux`** (Go 1.22+ pattern routing) — three of
them: `cmd/server/main.go:172`, `internal/api/public/api.go:163`,
`internal/api/admin/router.go:93`. There is no chi, no gin. **Do not introduce a
router dependency**; the split between the public and admin muxes is the
structure that matters.

Notable deps, and nothing beyond them without a reason: `jackc/pgx/v5`,
`pressly/goose/v3`, `google/uuid`, `oschwald/maxminddb-golang` +
`maxmind/mmdbwriter` (geo), `HugoSmits86/nativewebp` + `golang.org/x/image`
(media), `golang.org/x/time` (rate limiting), `golang.org/x/crypto`.

### Migrations

Goose, numbered SQL in `internal/store/migrations/` — `00001_init.sql`,
`00002_question_draft_order.sql`, and so on. Keep the zero-padded sequence.
Run them as a **separate step**, never from a container entrypoint:

```bash
docker compose -f deploy/docker-compose.yml run --rm --entrypoint migrate backend up
```

### Session cookies — the rule not to break

Two sessions with **deliberately different** flags:

- **Public respondent session** (`internal/api/public/cookie.go:51-65`) — name
  `ef_s_<slug>`, **per-form rather than shared**; `Path=/api`, narrower than `/`;
  `HttpOnly`; `SameSite=Lax` *not* Strict, because a referral from a messaging
  app arrives as a cross-site GET and Strict would drop the session; `Secure`
  taken from config, **never sniffed from the request**; `MaxAge` derived from
  the session expiry rather than a fixed constant.
- **Admin session** (`internal/admin/middleware.go:38-49`) — `__Host-ef_admin`
  when secure (the browser refuses that prefix unless Secure + `Path=/` + no
  Domain), plain `ef_admin` in dev, and `SameSite=Strict` because an admin
  session has no legitimate cross-site entry point.

**There is no CSRF token, deliberately.** The single-origin architecture is the
substitute: Next proxies same-origin so the public path never triggers a CORS
preflight. Explicit single-allowed-origin CORS exists only as a backstop for the
admin/dev path. If you split the origin, you must add CSRF — the two decisions
are one decision.

JavaScript must never be able to read a session token. There is a test asserting
the public cookie is `HttpOnly` (`internal/api/public/api_test.go:199`) — keep
assertions like that.

### The testing trap

`internal/testsupport/db.go:41`:

```go
t.Skip("TEST_DATABASE_URL not set; skipping database-backed tests")
```

Every database-backed test **skips silently** without that variable, so a broken
CI database yields a **green run proving almost nothing**. `ci.yml:76` therefore
fails the build on the skip message:

```yaml
if grep -q "TEST_DATABASE_URL not set" /tmp/gotest.log; then
```

**Never remove that assertion**, and when you write a suite that skips on a
missing dependency, add the equivalent. A skip is not a pass.

```bash
TEST_DATABASE_URL='postgres://...' go test ./...   # actually runs them
go test ./...                                      # green, and meaningless
```

### Client IP, proxies and rate limiting

**Parse forwarding headers right-to-left, discarding trusted hops from the
right.** `internal/httpx/clientip.go`: a forwarding header is trusted only if the
direct TCP peer is itself in `TRUSTED_PROXIES`, and an unparseable hop **stops**
the walk rather than being skipped — skipping would let an attacker hide the
trust boundary.

This is easy to get backwards, and it **has been gotten backwards in this very
estate**: `lms-tutor`'s `client_ip()` returns the *first* `X-Forwarded-For` hop
unconditionally — a live, documented, unfixed bug, and trivially spoofable.
`cleverclass.ai/backend`'s `ratelimit.py:94-122` enforces the correct
last-entry rule. When you write or review IP derivation anywhere, check the
direction first.

`TRUSTED_PROXIES` and the compose network subnet (`172.29.0.0/16`) are **two
halves of one setting**. Two hops sit in front of the backend and both must be
trusted. Drift between them produces **no error** — just every request
attributed to the proxy: one country in analytics, one bucket in the rate
limiter. If you touch either, check the other. See `emerge-deploy`.

Rate limiting uses in-process bounded token buckets (`golang.org/x/time/rate`)
behind a `Limiter` interface, with **two independently keyed limiters**:
session-create keyed by **IP**, response-writes keyed by **session** — because
respondents sit behind carrier-grade NAT and IP-keying writes would throttle a
whole town. Login has its own three-layer limiter (per-username, per-IP, plus a
global semaphore capping concurrent argon2 verifications to bound memory).
Keep the keying choices; they are reasoned, not arbitrary.

Respondent IPs are hashed and encrypted (`IP_HASH_KEY`, `IP_ENCRYPTION_KEY`,
`IP_RETENTION`) — `internal/privacy`. Treat that as a product requirement, not
an optional feature, and never log a raw address.

## Python — Exam Studio

Three services, each independently deployable, talking over HTTP on loopback:

| Service | Host | Container | Role |
|---|---|---|---|
| `lms-paper-orchestrator` | `8063` | `8300` | Django + DRF. Blueprints, drafts, slots, CP-SAT auto-fill. **Entry point.** |
| `lms-questionbank-engine` | `8062` | `8200` | FastAPI. Generation, dedup, grounding. |
| `lms-retrieval-engine` | `8060` | `8000` | FastAPI. Retrieval and rerank. |

The whole stack is one compose file in the orchestrator,
`lms-paper-orchestrator/deploy/docker-compose.yml`, plus `exam-studio-nginx` on
`8055`. **Services address each other by service name on the container port** —
`http://retrieval:8000` — through `QB_RETRIEVAL_BASE_URL` (`:142`) and
`PO_RETRIEVAL_BASE_URL` (`:205`). The image default is
`http://lms-retrieval-engine:8000` and is deliberately overridden to match the
compose service name; if you rename a service, that override is what breaks.
Host ports are defaults overridable via `EXAM_PORT_*`, so never hardcode `8060`
in code — only in a local curl.

These are on **feature branches** (`feat/composer-loop`,
`feat/exam-studio-wavea`, `feat/exam-studio-scoping`) with dirty trees. Confirm
what you are looking at before changing it.

### Conventions shared by the two FastAPI services

Match these; they are stated as house rules in the repos' own docs.

- **Flat `api.py`, no `APIRouter` split.** The `app` and the engine singleton are
  built **at import time**, so a load failure is a startup failure rather than a
  first-request failure.
- **Routes are `def`, not `async def`** — deliberately, so CPU-bound work runs on
  the threadpool instead of blocking the event loop.
- **No `Depends()` DI graph.** The only per-request injection is
  `Header(default=None)` for the tenant header.
- **`pydantic_settings.BaseSettings` with an `env_prefix`** (`QB_`, `RE_`,
  `TUTOR_`, `PO_` for Django) and one process-wide `settings` singleton.
- **Pinned to `--workers 1`** — every one of these services holds in-process
  state (a loaded embedding model, in-memory indexes, SQLite). Do not assume you
  can scale one horizontally by raising the worker count.
- Background jobs are module-level dicts plus a `threading.Lock`. No queue, no
  persistence — a restart loses them.
- **`questionbank` says "print, do not log"** — every diagnostic is
  `print(f"[qb] ...")`. The Go service uses structured `slog`. Pick the
  convention of the service you are in; do not universalize either.

### Where the CP-SAT solver lives

`lms-paper-orchestrator` is **two packages with a one-way dependency**, enforced
by a CI import smoke-check: `paper_orchestrator/` is **stdlib-only and never
imports Django**, and that is where the solver lives —
`paper_orchestrator/solver.py`, whose sole caller is `composer/autofill.py`.
Keep Django out of it.

`composer/repository.py:74` `append()` is the **only write path** — it performs
the mutation plus an append-only event journal entry in one transaction, under a
compare-and-swap on `state_version`. Do not write to those models directly.

The orchestrator has **no `/health` route**. Its container `HEALTHCHECK` probes
`/admin/login/` instead (`deploy/Dockerfile.orchestrator:64-70`), which is exempt
from the tenant middleware — and this requires `127.0.0.1` in `PO_ALLOWED_HOSTS`
or every probe returns 400 and the container never becomes healthy.

Also note `/nginx-health` on `exam-studio-nginx` is answered **by nginx itself**
and stays 200 even when every upstream is down. It is not a stack health check.

### Two things to know before touching them

**The retrieval engine degrades silently.** An empty or wrong model cache makes
`build_embedder()` fall back to `HashingEmbedder` (`embeddings.py:70` — it
prints a line and carries on), and the service then serves hashed bag-of-words
at **`200 OK`**. It does not crash. This has already happened once in
production; `/health/providers` exists specifically to catch it
(`api.py:34-49`).

```bash
curl -fsS 'http://127.0.0.1:8061/health/providers?strict=true'
```

`strict=true` answers **503 instead of 200** when degraded (`api.py:437-471`),
so it is usable as a real gate — put it in a deploy smoke test rather than
eyeballing `degraded == false`. Note the source's own warning: the parameter is
`strict: bool = False`, and calling the function directly rather than over HTTP
would default it to a truthy default-object — call it over HTTP.

Any change touching embedding or the model cache must be validated against that
endpoint, not against "the request succeeded".

**`X-School-Id` is scoping, not authentication.** That exact phrase appears in
all three services' own code comments, and the precise version matters:

| Service | Header behaviour |
|---|---|
| retrieval | **required** — 400 without it, and rejects a leading `_` (`api.py:280-296`) |
| orchestrator | **required** — `TenantMiddleware` returns `tenant_required` 400 (`composer/middleware.py:32-64`) |
| questionbank | **optional** — `Header(default=None)`; an absent header is its own "untenanted" bucket, no 400 (`api.py:66-74`). The weakest of the three. |

So *presence* is enforced in two of three, and data is correctly isolated by
whatever value is presented. What is **not** enforced anywhere is *identity*:
the header is unsigned, so any caller can claim any tenant. nginx adds nothing —
`deploy/nginx/exam-studio.conf:4-9` says plainly that anyone who can reach 8055
can read and write every tenant's data.

The compensating control, applied consistently: a cross-tenant read returns
**404, never 403**, so existence cannot be probed across tenants. Preserve that
when you touch these handlers — a "more helpful" 403 leaks the thing the 404 is
hiding.

Treat this as a known-temporary pattern requiring real auth in front, not a
template to copy. Say so explicitly rather than describing tenancy as enforced.

## Python — CleverClass

FastAPI with **SQLite in WAL mode and hand-written SQL** — no ORM anywhere.
Connections come from `backend/app/db/conn.py`; schema is
`backend/app/db/schema.sql` plus ~30 numbered files in `app/db/migrations/`,
applied by `app/db/migrate.py`. `app/services/` holds ~40 modules;
`app/api/routes.py` plus ~10 domain routers.

```bash
uvicorn main:app --reload --port 8000
cd backend && pytest
```

The test suite is **44 standalone subprocess suites plus one pytest-style** —
do not try to collect them in-process. See `backend/tests/CLAUDE.md`.

Admin auth is real and worth matching: scrypt (stdlib) hashing, HttpOnly session
cookie, CSRF double-submit, and **permission-based, deny-by-default**
authorization — routes check named permissions like `books:write` against
`ROLE_PERMISSIONS` in `app/services/permissions.py`. Only `super_admin` manages
admins. Customer-facing `/account` auth is still a stub, and the `/admin` cookie
check in `middleware.ts` is a UX gate, not the boundary.

Do not source deploy facts from `docs/` — it is stale. See `emerge-context`.

## PHP — phd-monitoring

Laravel 11, PHP 8.2, Sanctum tokens (ability `server:access`, 10-day expiry),
MySQL. 36 controllers (~10k lines), 52 models, **66 migrations in
`server/database/migrations/`**. The root-level `migrations/` directory is stale
junk — ignore it.

```bash
composer install && php artisan migrate && php artisan serve
```

There are **no tests**: `phpunit.xml` declares `tests/Unit` and `tests/Feature`
but `server/tests/` does not exist on disk, so `phpunit` errors out.

### The authorization situation — read before touching any guard

The 28 `can_*` columns on `roles` are `enum('true','false')` — **strings**. The
string `'false'` is truthy in PHP, so a guard that negates one **never fires**.
Two controllers are already broken this way (`FacultyController.php:28,115,261`
reads the base `role` not `current_role`; `StudentController.php:28,109` checks
`can_add_student`, singular, while the column is `can_add_students`, plural).

Real policy is ~62 ad-hoc string comparisons on `current_role->role`, e.g.
`FormLevelController.php:25`. A user's `current_role` — switchable via
`POST /api/switch-role` — is what gets checked, not their base role.

**Never "simplify" a comparison against the string `'false'` into a truthiness
check.** That turns a hole that is already open into one that looks deliberate,
without fixing anything. If authorization here genuinely needs fixing, the fix
is comparing role names, not resurrecting the flag columns — and it is a
scoped piece of work, not a drive-by cleanup.

Also present and not to be extended: unauthenticated write endpoints in
`routes/api.php` (`POST /register` hardcoding `role_id=1`, `POST /create-role`,
`GET /api/roles` dumping every capability row) and a global `LogRequestResponse`
middleware writing `Authorization: Bearer` headers into
`storage/logs/laravel.log`, later servable to any admin.

## Cross-cutting rules

- **Read the repo's `CLAUDE.md` first.** Conventions differ sharply per repo.
- **Migrations are forward-only and run as their own step.** Never at boot.
- **Never commit a `.env`.** Secrets live on the box; CI does not inject them.
- **Never log secrets, tokens, or raw IPs.**
- A test that skips is not a test that passes — assert the skip cannot happen.
- Errors: return them; do not swallow. When a failure mode is silent, add a
  check that makes it loud — that is the house style, most visible in
  `deploy.sh`'s preflight and `ci.yml`'s skip assertion.
- Do not add a dependency that duplicates something already in the module graph
  (particularly: no HTTP router in the Go service).
- When you change something whose failure is invisible, say so explicitly in
  your report rather than reporting a clean build.
