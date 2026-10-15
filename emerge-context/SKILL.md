---
name: emerge-context
description: Orientation for the Emerge / Exam Studio / CleverClass estate on D:\ — which repo owns what, the verified stack of each, what is actually deployed and where, and which widely-repeated claims are now stale. Load this first on any task touching these products before deciding which repo to open.
---

# Emerge estate — orientation

## 2026-10-15 update (read first)

The GCP demo box is gone; everything runs on the Hostinger VPS `editone-prod` (see
`box-ops`). Deployment now goes through Jenkins at https://devops.tryemerge.in: a push to
the deploy branch (main, or master for lms-grasp, lms-lesson-planner and
lms-paper-setter-school) runs the repo's `Jenkinsfile` (one `emergeDeploy(...)` call from
the shared library in `Harbour-Emerge/emerge-devops`), gates on tests, then runs the
repo's own `deploy.sh` on the box. PRs get tests only. Repos on Jenkins: emerge-devops,
CAS, Survey-Forms, lms-retrieval-engine, marketplace-processor, lms-grasp,
lms-lesson-planner, lms-paper-setter-school, lms-lab-engine, lms-board, lms-mind-map,
lms-flashcard-engine, lms-slide-builder, AnswerSheet-Checker, emerge-core,
lms-paper-orchestrator (Exam Studio). Pending: Internal-Admin-Console (PR open) and
emerge-mail (repo just created). Not on Jenkins by choice: emerge-frontend and
landing-page-emerge (Vercel via GitHub Actions), Marketplace (outside the org),
cleverclass.ai. Retired: lms-tutor, CAS-Admin. The deployment table below is historical.

Which repo to open, what it actually is, and what you have been told about it
that is no longer true.

**Verified 2026-08-30.** Claims carry evidence or are marked *unverified*.
This supersedes `D:\EMERGE_SETUP.md` (written 2026-08-26) where they disagree —
the disagreements are listed in the last section, and they are the point of this
document.

Infrastructure, SSH, ports and the shared-host rules live in **`box-ops`**.
Shipping procedure lives in **`emerge-deploy`**. This file is the map only.

## The products

Three unrelated products share this drive and, partly, one demo box.

**Emerge** — an AI-native academic operating system: student portal, faculty
portal, and an AI companion called Jerry. A joint venture of Harbour Tech and
EditOne International. Public site <https://www.tryemerge.in>.

**Exam Studio** (internally "the LMS repos") — question-paper generation for
schools. Three independently deployable services plus a tutor. A separate
product that merely shares the box.

**CleverClass.AI** — a storefront for Adwani Publishing House, Nagpur, selling
study guides for Maharashtra State Board and CBSE. Despite its README, it is a
full commerce platform, not a chatbot.

## Repo map

Every repo below has its own `CLAUDE.md` — **42 files across the 8 repos**.
Read the repo's own file before editing it. This table only routes you.

### Emerge

| Path | What it is | Stack |
|---|---|---|
| `D:\Emerge-Design` | The Emerge design system **and** the landing page. A generator — see below. | Plain ES modules, `node build.mjs`; landing built by `python landing/v2/build.py`. No `package.json`. No git remote. |
| `D:\Survey-Forms` | `emerge-forms` — the anonymous survey platform. Deploys to `survey.tryemerge.in`. | Go modular monolith + PostgreSQL + Next.js 16 |

### The platform — added 2026-09-08, previously missing from this file

These four are the core of Emerge proper and this document omitted them
entirely. **Do not read that earlier silence as absence.**

| Path | What it is | Stack |
|---|---|---|
| `D:\CAS` | Identity + authorization for every Mentis service. A **sidecar, never a gateway**: a service posts its token plus the user's to `POST /v1/policies` and gets entities *plus boolean expressions* to apply to its own rows. Domain-agnostic and must stay so. | Go, chi, pgx (no ORM), RS256+JWKS, gRPC, argon2id. No CI. |
| `D:\emerge-core` | The Emerge backend: schools, people, students, staff, teachers, guardians. **One database per school** + a control plane; `X-School-Id` selects it. | Go, Ent ORM, chi, PostgreSQL |
| `D:\Internal-Admin-Console` | `mentis-console` — one admin for every Mentis service. Runs at `admin.auth.tryemerge.in` and `admin.auth-demo.tryemerge.in`. | Next.js App Router, TS |
| `D:\CAS-Admin` | **Archived.** Superseded by Internal-Admin-Console, which took over the same two URLs and ports. | Next.js 14, MUI v6 |

CAS's `docs/*.md` is the reliable half of that repo — unusually for this estate.
**Discard any CAS `CLAUDE.md` dated before 2026-09-03**: the earlier set
described a `/v1/authorize` endpoint, `X-CAS-*` headers, ForwardAuth and a DNF
solver, none of which exist. Full detail on all four, with `file:line`, is in
`ultra-plan/references/estate-truth.md`.

### Exam Studio

| Path | What it is | Host (loopback) | In container |
|---|---|---|---|
| `D:\lms-paper-orchestrator` | Django + DRF. Blueprints, drafts, slots, CP-SAT auto-fill. **Entry point.** | `8063` | `8300` |
| `D:\lms-questionbank-engine` | FastAPI. Question generation, dedup, grounding. | `8062` | `8200` |
| `D:\lms-retrieval-engine` | FastAPI. BM25 + BGE dense retrieval, cross-encoder rerank, figure store. | `8060` | `8000` |
| `D:\lms-tutor` | Grounded chatbot, citations only from course materials. Part of the **older** LMS stack, not the Exam Studio compose stack. | — | `8100` |

The Exam Studio stack is defined by **one compose file in the orchestrator** —
`lms-paper-orchestrator/deploy/docker-compose.yml` — which runs all three
services plus `exam-studio-nginx` (`8055`). The host ports are defaults, not
constants: `EXAM_PORT_RETRIEVAL`, `EXAM_PORT_QUESTIONBANK`,
`EXAM_PORT_ORCHESTRATOR` (`docker-compose.yml:87,154,224`). Inside the network,
services address each other by service name on the **container** port — e.g.
`http://retrieval:8000`, via `QB_RETRIEVAL_BASE_URL` and
`PO_RETRIEVAL_BASE_URL`. A local `uvicorn` run listens on the container port,
not the 806x number.

Supporting data and scratch, not services: `D:\lms-data`, `D:\lms-data-enriched`
(corpora), `D:\lms-cache` (HuggingFace weights), `D:\lms-ui-scratch`,
`D:\qb-scratch`, `D:\qb-dangling`. Treat the scratch dirs as disposable.

### Other education repos

| Path | What it is | Stack |
|---|---|---|
| `D:\cleverclass.ai` | CleverClass.AI storefront + full admin | Next.js 16, React 19, Tailwind 4, Zustand, Zod, react-hook-form / FastAPI 0.115 + **SQLite (WAL), hand-written SQL, no ORM** + ChromaDB |
| `D:\phd-monitoring` | PhD student management, multi-role approval chains | **Laravel 11 + PHP 8.2 + Sanctum + MySQL** / React CRA 5 in `client-new/` (Tailwind 3 prefixed `tw-`, MUI 7) |

### Explicitly out of scope

`D:\Backend` (SportJacks microservices), `D:\runnable`, `D:\scheduler` (empty),
`D:\paper-setter`, `D:\ImageFilter`, `D:\Portfolio`, and the `sat-*` /
`saturnalia*` repos (a college fest).

`D:\commerce-platform` is the ambiguous one: a pnpm/Turbo monorepo explicitly
**"ported from cleverclass.ai"** per its README, currently Phase 0 of 11 —
scaffold only, no app code. Not runnable. Treat as out of scope unless the task
names it.

## Git state — read this before believing any repo is clean

Checked 2026-08-30. **Every active repo has an uncommitted tree, and most are on
a feature branch.** An assumption that you are looking at `main` is usually wrong.

| Repo | Branch | Dirty files | Remote |
|---|---|---|---|
| `Survey-Forms` | `deploy/dockerize-cicd` | 7 | `github.com/Harbour-Emerge/Survey-Forms` |
| `Emerge-Design` | `master` | 38 | **none configured** |
| `cleverclass.ai` | `main` | 7 | `github.com/KanishkJagya1/cleverclass.ai` |
| `phd-monitoring` | `main` | 4 | `github.com/akarsh911/phd-monitoring` |
| `lms-paper-orchestrator` | `feat/composer-loop` | 6 | `github.com/Harbour-Emerge/lms-paper-orchestrator` |
| `lms-questionbank-engine` | `feat/exam-studio-wavea` | 5 | `github.com/Harbour-Emerge/lms-questionbank-engine` |
| `lms-retrieval-engine` | `feat/exam-studio-scoping` | 11 | `github.com/Harbour-Emerge/lms-retrieval-engine` |
| `lms-tutor` | `main` | 4 | `github.com/Harbour-Emerge/lms-tutor` |

**The five `mentisAP` repos moved to the `Harbour-Emerge` org** — corrected
2026-08-30. GitHub still redirects the old URLs, so a stale remote keeps working
and gives no sign it is stale until someone creates a repo at the freed-up old
name, at which point pushes go somewhere else entirely. All six local remotes
(the five above plus `paper-setter`) were rewritten to `Harbour-Emerge` on that
date. Verified by comparing `git ls-remote ... HEAD` across both orgs: identical
SHAs, so these are redirects to one repo, not forks.

**`lms-questionbank-engine` is ARCHIVED and read-only.** `git push` to it fails
with `remote: This repository was archived so it is read-only.` and a 403 — which
surfaces only on push, never on fetch or `ls-remote`. Plan around that before
starting work that has to land there.

Re-run before acting; these numbers move:

```bash
for r in Survey-Forms Emerge-Design cleverclass.ai phd-monitoring \
         lms-paper-orchestrator lms-questionbank-engine lms-retrieval-engine lms-tutor; do
  printf "%-26s %s %s\n" "$r" "$(git -C /d/$r rev-parse --abbrev-ref HEAD)" \
    "$(git -C /d/$r status --porcelain | wc -l)"
done
```

## Emerge-Design is a generator, not a page set

The **167 `.dc.html` artboards at the repo root are build output.** Editing one
is wasted work — the next build overwrites it.

```
src/registry.mjs   the authoritative screen list
src/data.mjs       the single shared dataset every screen renders from
src/screens/       30 modules holding the actual screen definitions
src/ui.mjs         FONTS, BASE_CSS, the screen() primitive, canvas W/H
build.mjs          the generator -> writes *.dc.html at the root
```

Edit `src/`, then rebuild with `node build.mjs`. There is no `package.json` and
no npm install step.

The landing page is separate, under `landing/v2/`, built by
`python landing/v2/build.py`. That build **asserts** the page stays vector-only
(no `<img>`), that tags balance, that CSS braces balance, and that no warm/cream
colours survive. If the build fails, it is telling you something real — do not
route around it.

## Where things are deployed

| Thing | Where | How |
|---|---|---|
| `www.tryemerge.in` | Vercel project `emerge-landing`, team `saturnalia` | built from `Emerge-Design/landing/v2` |
| `emerge-student-portal.vercel.app` | Vercel project `emerge-student-portal` | |
| `survey.tryemerge.in` | demo box `lms` — 3 containers behind Caddy, plus an opt-in local Postgres | `Survey-Forms/deploy/` |
| Exam Studio | demo box, `exam-studio-*` containers | |
| CleverClass | demo box, `cc-web` / `cc-api` / `cc-proxy`, ports 8050 (storefront) and 8051 (admin) | `cleverclass.ai/deploy/`, from `/opt/cleverclass` |
| `phd-monitoring` | `phdportal.thapar.edu` via a **Jenkins** job triggered by `.github/workflows/main.yml` | not on the demo box |

`phd-monitoring` deploys nowhere near the other products. Do not reason about it
from the demo box.

There is also an `editone-prod` host in `~/.ssh/config` (Hostinger VPS,
`srv1888767.hstgr.cloud`) described as CleverClass/EditOne production — but
**no repo on `D:\` references it at all**. Its role is genuinely unknown; ask
before deploying anything to it. See `box-ops`.

## Landmines worth knowing before you start

These are the estate-wide ones. Per-repo traps live in each repo's `CLAUDE.md`.

- **`emerge-forms` is not a finished product.** `docs/build/state.md` records
  phase `IMPLEMENTATION / IN_PROGRESS` with testing, security, performance, UX,
  red-team and release gates all `PENDING`, and architecture and planning
  `SKIPPED` by user direction. A green deploy is not evidence of a reviewed
  product.
- **CleverClass's own docs lie.** `docs/deployment.md` describes Vercel and
  Render; neither is used. `README.md` describes a chatbot; it is a commerce
  platform. Source deploy facts from `deploy/`, never from `docs/`.
- **CleverClass customer auth is a stub.** The `/admin` cookie gate in
  `middleware.ts` is a UX check, not the security boundary; admin auth is real
  (scrypt, HttpOnly session, CSRF double-submit, deny-by-default **permission**
  strings like `books:write` — not roles).
- **`phd-monitoring`'s permission flags are decorative.** The 28 `can_*` columns
  are `enum('true','false')` **strings**, so the string `'false'` is truthy and
  guards that negate them never fire. Real policy is ~62 ad-hoc string compares
  on `current_role->role`. Never "simplify" a compare against `'false'` into a
  truthiness check — and note `can_add_student` (singular, checked) does not
  match the column `can_add_students` (plural, actual).
- **`phd-monitoring` has unauthenticated write endpoints** and a global request
  logger that writes `Authorization: Bearer` headers into
  `storage/logs/laravel.log`, later servable to any admin. Do not add to that
  pattern.
- **`X-School-Id` in Exam Studio is scoping, not authentication.** Presence is
  required by retrieval and the orchestrator (400 without it) and data is
  isolated by whatever value is sent — but the header is unsigned, so any caller
  can claim any tenant. `exam-studio-nginx` adds nothing: anyone who can reach
  8055 can read and write every tenant's data. Cross-tenant reads return 404,
  never 403, deliberately — preserve that. See `backend`.
- **The retrieval engine degrades silently.** A wrong HuggingFace cache layout
  makes it fall back to a hashing embedder and serve plausible nonsense at
  `200 OK`. Check `/health/providers?strict=true`.
- **Go tests skip silently without a database.** `testsupport.DB` skips when
  `TEST_DATABASE_URL` is unset, so CI can be green while proving nothing.
  `ci.yml` asserts the skip message is absent — keep that check.
- **CRLF breaks the deploy.** These repos are edited on Windows with
  `core.autocrlf=true` and run on Linux; without `.gitattributes` forcing LF,
  `deploy.sh` fails with a `bash\r` "No such file or directory" error naming
  neither the file nor the cause.
- **`phd-monitoring` has a stale root `migrations/` directory.** The real ones
  are the 66 files in `server/database/migrations/`.

## Corrections to `D:\EMERGE_SETUP.md`

That document is a good orientation and mostly still accurate, but it is four
days old and wrong in these specific places. Prefer this file:

1. **The landing page notify form is no longer fake.** `EMERGE_SETUP.md` §6 says
   the `#notify` handler "only hides the form", captures nothing and sends no
   request. That was true of
   `landing/v2/index.template.html.bak-20260826:547`, which is the preserved
   pre-change copy. The **current** `landing/v2/index.template.html:610` does a
   real `fetch('/api/connect-emerge', {method:'POST', ...})` with validation,
   429 handling and server-message surfacing. The form is wired in source.
   *Whether the built page has been redeployed to Vercel is unverified* — check
   before telling anyone emails are being collected in production.
2. **CleverClass's stack is not what the setup doc implies.** It is FastAPI +
   **SQLite with hand-written SQL** (no ORM, no Postgres) behind a Next.js 16 /
   React 19 / Tailwind 4 frontend.
3. **`phd-monitoring` is Laravel + MySQL**, not part of the Python/Go estate,
   and deploys via Jenkins to `phdportal.thapar.edu`.
4. **43 `CLAUDE.md` files is now 42.**
5. **The LMS repos are on feature branches**, not `main` — see the git table.
6. `EMERGE_SETUP.md` §5 lists the remaining deploy steps for `survey.tryemerge.in`
   and states nothing in the bundle had ever been built or run. **Re-verify that
   status before relying on it** — it was written during an outage and the box
   has been back since.
7. **443 is not open at the firewall.** `EMERGE_SETUP.md` §3.3 says it is;
   `Survey-Forms/deploy/Caddyfile:24-26` records the opposite, verified by
   probing from outside. The stack currently serves **plain HTTP on 8070**, and
   `survey.tryemerge.in` is the target state, not the current one. See
   `emerge-deploy` for the four things a switch to HTTPS requires.
8. **Exam Studio has no CI deploy path.** Its bundle lives in
   `lms-paper-orchestrator/deploy/` and is run by hand. The `deploy.yml` in
   `lms-retrieval-engine` and `lms-tutor` targets the *older* LMS stack via
   `/opt/lms/lms-deploy/deploy.sh`, a script tracked by no repo.

### Repo docs that still need fixing

`Emerge-Design/CLAUDE.md:179-196` and `Emerge-Design/landing/CLAUDE.md` both
still describe the `#notify` form as a no-op stub with "no fetch, no XHR anywhere
in the file". That is now false (see item 1). This is precisely the kind of stale
doc that leads someone to tell a stakeholder "no emails are being collected" when
the endpoint is real, tested, and persisting to a database. Fix those two files.

When you find another of these, fix it here rather than in `EMERGE_SETUP.md`,
and date the correction.

## Conventions

- **Clone into `D:\<RepoName>`.** Never into the session scratchpad — it is
  session-scoped and gets cleaned, and these repos are worked on outside Claude.
- Read the target repo's `CLAUDE.md` before editing it.
- Generated files are not source. `.dc.html` artboards are build output.
- **Never commit a `.env`.** `Survey-Forms/deploy/.env` holds the database
  password, IP encryption key, admin bootstrap password and SMTP app password.
  CI deliberately does not inject them; the file living on the box is the point.
- Test emails go to `akarshagri121@` and `masterreyan@` only.
- The demo box has four tenants. Do not touch what you did not create — `box-ops`.
