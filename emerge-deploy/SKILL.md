---
name: emerge-deploy
description: Ship changes to the Emerge estate — build, migrate, roll, smoke-test and roll back. Encodes the deploy patterns these repos actually use (SHA-tagged images, on-box .env, TLS-ALPN on a box where port 80 is taken, preflight checks for the failures that are silent). Use before deploying, writing a Dockerfile/compose/workflow, or debugging a failed deploy.
---

# Deploying the Emerge estate

## 2026-10-15 update (read first)

Deploys are triggered by Jenkins (https://devops.tryemerge.in), not GitHub Actions; the
"three generations" and the Actions `deploy.yml` descriptions below are historical. The
mechanics on the box did not change: each repo still ships `deploy/deploy.sh`, Jenkins
syncs the commit into `/opt/emerge/<name>/src` (git reset for CAS, retrieval, forms and
the console; tar overlay with `stale-<sha>/` handling elsewhere), writes
`DEPLOYED_COMMIT`, runs the script as root under `/root/deploy.lock`, polls the health
URL, and relabels `:rollback` with `keep=true` so the nightly prune keeps it. Manual
ref/preview deploys for CAS and the console live under Emerge → Manual deploys in
Jenkins. Exam Studio's bundle was rewritten for the VPS (no retrieval service, no sudo).
Adding a repo: Jenkinsfile in the repo, an entry in `casc/repos.yaml` of emerge-devops,
restart the controller; the GitHub App `emerge-devops-ci` delivers the webhooks.

`Survey-Forms/deploy/` is the reference implementation. Anything new should look
like it. Read `box-ops` before running a command on a remote host, and
`emerge-context` if you are unsure which repo deploys where.

**Verified 2026-08-30** by reading the files named below.

## Deploy targets

| Product | Target | Mechanism |
|---|---|---|
| `emerge-forms` | demo box `lms`, 3 containers | push to `main` → CI gate → SSH → `./deploy/deploy.sh` |
| Exam Studio | demo box, `exam-studio-*` | **manual only** — `./deploy.sh deploy` by hand on the box |
| CleverClass | demo box, `cc-web`/`cc-api`/`cc-proxy` from `/opt/cleverclass` | **manual** — `deploy/deploy.sh` by hand |
| older LMS stack | demo box, `lms-deploy-*` | push to `main` in `lms-tutor` or `lms-retrieval-engine` → SSH → `/opt/lms/lms-deploy/deploy.sh` (a script in **no repo**) |
| `emerge-landing`, `emerge-student-portal` | Vercel, team `saturnalia` | Vercel CLI, already authenticated |
| `phd-monitoring` | `phdportal.thapar.edu` | Jenkins job triggered by `.github/workflows/main.yml` |

### Three generations — know which one you are looking at

Do not copy a pattern from the wrong one:check the generation of the code you are working with.

1. **Oldest, thinnest** — `lms-tutor` and `lms-retrieval-engine`
   `.github/workflows/deploy.yml` both SSH in and run
   `/opt/lms/lms-deploy/deploy.sh`, a script that **lives only on the box and is
   tracked by no repo**. This targets the *older LMS stack*, not Exam Studio. A
   push to `main` in either repo redeploys that whole stack.
2. **Middle** — `cleverclass.ai/deploy/`: own compose project, staged build,
   `EXIT`-trap post-deploy verification, and a regression check that the sibling
   LMS is still up afterwards.
3. **Newest and most complete** — `Survey-Forms/deploy/` and
   `lms-paper-orchestrator/deploy/` (Exam Studio): named subcommands, SHA image
   tagging, poll-don't-sample health gates, backup scripts.

**Exam Studio has no CI deploy path at all.** Its bundle is the most mature one
in the estate, and it is run by hand from a checkout on the box. Do not go
looking for a workflow that triggers it — there isn't one.

## The golden path for emerge-forms

Run **from the repository root, on the box** — not from `deploy/`, not from your
workstation:

```bash
./deploy/deploy.sh
```

It does, in order (`deploy/deploy.sh:7-10`):

1. **Preflight** the things that fail silently (see below).
2. **Build** both images tagged with the commit SHA —
   `IMAGE_TAG="$(git rev-parse --short HEAD || echo manual)"` (`deploy.sh:121`).
3. **Migrate** as a separate step, before rolling (`deploy.sh:139`).
4. **Roll** the services.
5. **Smoke-test** the real origin: `curl -fsS http://127.0.0.1:8067/healthz`,
   and `die` if it does not answer (`deploy.sh:163-165`).

Any failure leaves the previous containers running and prints the rollback
command. Rollback is by re-tagging, because images carry the SHA:

```bash
IMAGE_TAG=<previous-sha> docker compose -f deploy/docker-compose.yml up -d
```

`deploy.sh` prints the exact command with the previous tag filled in
(`deploy.sh:186`).

## The stack shape

```
        :${PUBLIC_PORT:-8070}  (the only public port)
                |
           [ caddy ]  ── single origin ──┐
                |                        │
   /api /admin /media              everything else
                |                        │
        [ backend :8099 ]        [ frontend :3000 ]
           Go                        Next.js 16
                |
         Cloud SQL demo-db  (managed, named in .env)
```

### It is serving plain HTTP today, not HTTPS

The Caddyfile has **two modes selected entirely by `SITE_ADDRESS`**, and nothing
else changes between them (`Caddyfile:13-21`):

| `SITE_ADDRESS` | Behaviour |
|---|---|
| `:8070` | plain HTTP, ACME skipped entirely — **CURRENT** |
| `survey.tryemerge.in` | Caddy obtains a certificate and serves HTTPS on 443 |

A bare `:port` gives Caddy no hostname to get a certificate for, so it skips ACME.
Note `deploy/.env.example` ships `SITE_ADDRESS=survey.tryemerge.in` — the
*target*, not the current state. The live `deploy/.env` on the box is what
decides, and the Caddyfile's own comment marks HTTP as current.

**Flipping to HTTPS needs four things together, and three are outside this file**
(`Caddyfile:23-34`):

1. **TCP 443 open on the GCP firewall for this VM — it is NOT**, verified by
   probing from outside on 2026-08-26. Do not repeat the claim that 443 is open.
2. An A record `survey.tryemerge.in` → `34.131.254.234`, set in the Hostinger
   panel (no API credential exists here — a human must do it).
3. `SITE_ADDRESS` and `PUBLIC_ORIGIN` both changed in `deploy/.env`, and
   `APP_ENV` set back to `production`.
4. The `http_port 8064` line uncommented (`Caddyfile:62`), because port 80 on
   this box belongs to another tenant's nginx. Moving Caddy's HTTP port off 80
   leaves the ACME HTTP-01 challenge with nowhere to answer, so it falls through
   to **TLS-ALPN-01 over 443** — the only way a certificate can be issued here.

Doing any one of the four alone leaves the stack broken or certificate-less.

| Service | Container | Published |
|---|---|---|
| backend | `emerge-forms-backend` | `127.0.0.1:8067` → `8099` |
| frontend | `emerge-forms-frontend` | `127.0.0.1:8068` → `3000` |
| caddy | `emerge-forms-caddy` | `${PUBLIC_PORT:-8070}` public, `127.0.0.1:8065` health |
| postgres | `emerge-forms-postgres` | **opt-in only** — see below |

**The default stack is three services, not four.** The `postgres` service is
gated behind `profiles: ["localdb"]` (`docker-compose.yml:215`) and does **not**
start unless you ask for it:

```bash
docker compose --profile localdb up -d    # only for a throwaway local run
```

The intended database is the managed Cloud SQL instance named in `.env`. The
local container exists so the stack can be exercised end to end while Cloud SQL
network access is pending. It is deliberately **not** a silent fallback: without
the profile it does not start, so a wrong `DATABASE_URL` fails loudly instead of
quietly writing responses into a container nobody backs up. Do not remove the
profile to "make it work" — that is the failure the design is preventing.

The image is pinned to `postgres:18-alpine` to match the managed instance
version. A dump from a newer server cannot be read by an older
`pg_dump`/`pg_restore`, and the mismatch surfaces as a bare "server version
mismatch" on every backup and restore. Keep them equal.

Caddy routes `@api path /api/* /admin/* /media/*` to `backend:8099` and
everything else to `frontend:3000` (`deploy/Caddyfile:82-114`).

**One origin is a product decision, not a deployment convenience.** It is what
lets the anonymous session cookie stay `HttpOnly` + `SameSite=Lax` with no CORS
and no CSRF token on the public path. Splitting the API onto its own hostname
silently breaks that: the cookie stops being sent and every respondent restarts
their session on every request. Do not "tidy" this into two hostnames.

## Preflight checks — and why each exists

`deploy.sh` checks these *before* stopping anything, because every one of them
fails silently or is expensive to diagnose afterwards (`deploy.sh:24-101`):

| Check | The silent failure it catches |
|---|---|
| `PUBLIC_ORIGIN` and `APP_ENV` agree | the server enforces the https rule under production; a mismatch boot-loops the container instead of printing a reason |
| `SITE_ADDRESS` and `PUBLIC_PORT` agree | Caddy listens on a port Docker never published: healthy container, connection timeout, nothing in any log |
| no raw `#` in the database password | a `#` truncates the URL at the fragment; the symptom is an auth failure, not a parse error |
| the public port is ours or free | the box runs four other tenants; binding their port breaks their service, not yours |
| `output: 'standalone'` still in `next.config.ts` | the frontend image copies `.next/standalone`, which Next only emits with that line. Removing it breaks nothing locally — `next dev` and `next build` behave identically — and surfaces minutes into a build as an opaque `COPY` error naming a path, not a setting. **This has already happened once.** |
| the **effective** env via `docker compose config` | an inline `environment:` entry silently overrides `env_file:`. This exact trap already cost one deploy: `APP_ENV` was pinned to `production` in compose while `.env` said `development`, so every check above passed and the container still exited at boot |

When you add a setting whose failure mode is silent, add a preflight check for
it. That is the pattern this file exists to continue.

## Traps that have already cost time

**Port 80 is not available.** It belongs to another tenant's nginx. ACME uses
**TLS-ALPN-01 over 443**, and Caddy's HTTP listener is moved to loopback `8065`
(`Caddyfile:30`, `62`, `68`). **Never add a `80:80` mapping** — it fails to bind
and takes the stack down with it.

**The compose subnet is pinned at `172.29.0.0/16`, and every service must also
list `networks:` explicitly** (`docker-compose.yml:47-59`). This is not a style
point — it already bit: without the per-service declaration Docker attached
services to a different network than the file documents. `TRUSTED_PROXIES` and
the subnet are two halves of one setting, and drift between them produces **no
error** — just every request attributed to the proxy, one country in analytics
and one bucket in the rate limiter. Two hops sit in front of the backend and
both must be trusted (`docker-compose.yml:98`).

**CRLF kills the deploy.** These repos are edited on Windows with
`core.autocrlf=true` and run on Linux. Without `.gitattributes` forcing LF,
`deploy/deploy.sh` checks out with CRLF and bash fails on the first line with
`/usr/bin/env: 'bash\r': No such file or directory` — naming neither the file nor
the cause. `.gitattributes` covers `.sh`, `.yml`, `.yaml`, `.sql`, `.go`,
`Dockerfile`, `Caddyfile`, `Makefile`, `.env.example`. **Do not remove it**, and
extend it when you add a file type a Linux tool parses.

**`df /` lies on the box.** Docker's data root is a different, much larger disk.
See `box-ops`.

**The Makefile hardcodes `export PATH := /opt/homebrew/bin:$(PATH)`** — macOS
only. `make` does not work as written on Windows or Linux.

## Secrets

**Never commit a `.env`.** `deploy/.env` holds the database password, the IP
encryption key, the admin bootstrap password and the SMTP app password.

**CI deliberately does not inject them.** The file living on the box at
`/opt/lms/emerge-forms/src/deploy/.env`, mode `600`, owner `founder_mentis`, is
the point of the design — the database password and IP encryption key never pass
through GitHub.

`deploy/CREDENTIALS.md` **is tracked in git**, and that is intentional: it is a
record of *where* each secret lives and how to rotate it, containing no values.
Verified 2026-08-30 — zero high-entropy strings. **Keep it that way.** If you
ever add a real value there, you have leaked it to the remote.

The variables (names only) in `deploy/.env.example`:

```
APP_ENV  PUBLIC_ORIGIN  SITE_ADDRESS  ACME_EMAIL  IMAGE_TAG
DATABASE_URL  LOCAL_DB_NAME  LOCAL_DB_USER  LOCAL_DB_PASSWORD
IP_HASH_KEY  IP_ENCRYPTION_KEY  IP_RETENTION
SESSION_TTL  ADMIN_SESSION_TTL  ADMIN_BOOTSTRAP_USERNAME  ADMIN_BOOTSTRAP_PASSWORD
SMTP_HOST  SMTP_PORT  SMTP_TLS  SMTP_USERNAME  SMTP_PASSWORD  SMTP_FROM  SMTP_FROM_NAME  SMTP_TIMEOUT
GEO_PROVIDER  GEO_MMDB_PATH  MAX_UPLOAD_SIZE  NEXT_PUBLIC_LOG_LEVEL
```

Note `PUBLIC_PORT` is **not** in `.env.example` although compose and the
preflight both use it — it defaults to `8070`. If you are deploying to 443, set
it explicitly and add it to the example file.

The application **must not run as `postgres`**. Create the app role with
`deploy/create-db-user.sql`, which is idempotent and takes the password at run
time rather than storing it.

## CI/CD

`.github/workflows/ci.yml` — runs on `push` and `pull_request`. Three jobs: Go
tests against a real Postgres service container, the frontend build, and a
buildx build of both images with `push: false` (build-as-a-test).

**The assertion that makes CI meaningful** (`ci.yml:72-76`): `testsupport.DB`
skips every database-backed test when `TEST_DATABASE_URL` is unset, so if the
database ever stops being reachable the suite goes **green by skipping** instead
of red by failing. CI therefore greps the test log and fails the build on the
skip message:

```yaml
if grep -q "TEST_DATABASE_URL not set" /tmp/gotest.log; then
```

**Keep this check.** A green run without it proves close to nothing. Copy the
pattern into any suite that skips on a missing dependency.

`.github/workflows/deploy.yml` — triggers on push to `main`, calls `ci.yml` as a
gate, then deploys over SSH with `appleboy/ssh-action@v1.2.0` using secrets
`DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`.

## Conventions to follow in anything new

Observed across these repos, in rough order of how load-bearing they are:

- **Pin the compose project `name:`.** Universal here — `emerge-forms`,
  `exam-studio`, `cleverclass`. It is what stops a `docker compose down` run
  from the wrong directory reaching another tenant's containers.
- **Tag images with the commit SHA.** It is what makes rollback a one-liner.
  Exam Studio goes further with a `:current` / `:rollback` two-tag scheme
  re-pointed **only after health passes** (`lms-paper-orchestrator/deploy/deploy.sh:400-426`,
  `repoint_rollback()`). That is the better pattern — prefer it for anything new.
- **Poll for health with a deadline; never sample once.** The single most
  consistent convention across every stack here. A `curl` immediately after
  `up -d` returns `000` from a perfectly healthy service. Survey-Forms polls for
  180s; Exam Studio's `healthcheck.sh` polls to a configurable 300-420s.
- **Separate liveness from correctness.** Exam Studio's container `HEALTHCHECK`
  probes plain `/health` (liveness), while `healthcheck.sh` separately calls
  `/health/providers?strict=true` (correctness). Copy this split wherever a
  service can degrade while still answering.
- **Multi-stage builds**, non-root runtime user, `CGO_ENABLED=0` for Go.
- **Build both images from the repository root** so one `.dockerignore` governs
  both contexts — that is why it exists at the root rather than per-image.
- **Migrate as a separate step** before rolling, never as a container entrypoint.
- **Healthcheck every service**, and gate dependents on
  `condition: service_healthy`.
- **Smoke-test the public origin** after rolling, and fail loudly if it is silent.
- **Keep secrets on the box**, not in CI.
- `.gitattributes` forcing LF, in every repo that ships a shell script.

## Known gaps — do not perpetuate these

Real inconsistencies found across the estate. If you are touching one of these
files anyway, fix it; do not copy the gap into something new.

- **Every Python image runs as root.** `retrieval`, `questionbank`, `tutor`,
  `cleverclass` backend and the Django orchestrator all lack a non-root `USER`,
  while every Go and Node image has one (`Survey-Forms/backend/Dockerfile:51`,
  `frontend/Dockerfile:52`, `cleverclass.ai/frontend/Dockerfile:30`). New Python
  images should add one.
- **None of the four `lms-*` repos has a `.gitattributes`.** That is exactly the
  CRLF risk profile Survey-Forms documents — `lms-paper-orchestrator/deploy/deploy.sh`
  is bash, run on Linux, from a repo edited on Windows.
- **`cleverclass.ai` has no image tagging scheme** — plain rebuild, so there is
  no tag to roll back to.
- **Survey-Forms `deploy/dockerize-cicd` is 22 commits ahead of `main`, 0
  behind.** `deploy.yml` only triggers on `main`, so none of the Docker/CI work
  reaches the box until that branch is merged.
- **`backend/adminctl`, a ~15.5 MB compiled binary, is committed to git.**
  `.gitignore` covers `/bin/` but not that path. It looks like an accidental
  build-artifact commit — worth removing, but confirm with the owner first since
  rewriting history on a shared branch is not a drive-by fix.

## Before you deploy — checklist

- [ ] You are on the box, at the repository root
- [ ] The branch is what you think it is (**every repo here is usually on a
      feature branch with a dirty tree** — see `emerge-context`)
- [ ] `deploy/.env` exists on the box, mode `600`
- [ ] `docker compose config` shows the effective env you expect
- [ ] The public port is yours or free — four tenants share this box
- [ ] You know the current image tag, so you can roll back
- [ ] You are not about to restart, prune, or rebuild another tenant's anything

## After a failed deploy

1. `docker compose -f deploy/docker-compose.yml ps` — state and health
2. `docker logs emerge-forms-caddy --tail=50` — **certificate errors appear
   here**, not in the app logs
3. `curl -fsS http://127.0.0.1:8065/_up` — Caddy itself, no certificate needed
4. `curl -fsS http://127.0.0.1:8067/healthz` — backend, pings the database pool
5. Roll back with the command `deploy.sh` printed; diagnose afterwards

Volumes that must not be lost: **`caddy_data`** (ACME account key and
certificates — deleting it burns Let's Encrypt rate limit) and **`media`**
(respondent-visible uploads).

## A green deploy is not a reviewed product

`emerge-forms` records phase `IMPLEMENTATION / IN_PROGRESS` in
`docs/build/state.md`, with testing, security, performance, UX, red-team and
release gates all `PENDING`. Deploying it does not change that. Say so when
reporting a successful deploy rather than implying readiness.
