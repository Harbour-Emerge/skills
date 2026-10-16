---
name: repo-audit
description: Deeply understand an unfamiliar repo using parallel read-only agents, verify every claim against code, then write or refresh its CLAUDE.md files. Use when onboarding onto a repo, when existing docs are suspected stale, or when asked to "audit", "map", "understand in depth", or "generate CLAUDE.md".
---

# Repo audit

Produce a verified mental model of a repository, then persist it as `CLAUDE.md`
files. The output is only worth having if every load-bearing claim was checked
against code — this skill exists because documentation in these repos has
repeatedly been wrong in ways that cost hours.

## The rule that makes this skill work

**Never restate a document. Verify it or drop it.**

Existing `README.md`, `docs/`, and even prior `CLAUDE.md` files are *leads*, not
sources. Several in this ecosystem are actively misleading:

- `cleverclass.ai/docs/deployment.md` describes Vercel/Render. Neither exists;
  it deploys as Docker containers on a shared VM.
- `cleverclass.ai/README.md` describes a chatbot. It is a full commerce platform.
- `lms-paper-orchestrator/deploy/README.md` says the stack is "UNBUILT and UNRUN".
  It was deployed and healthy on 2026-08-26.
- `phd-monitoring` has a root `migrations/` directory that is stale junk; the real
  ones are in `server/database/migrations/`.

Every claim in your output carries either `file:line` evidence or the word
*unverified*. No third category.

**Calibrate on the repo, not on this estate.** The four `lms-*` repos carry
unusually rigorous self-documentation — dated, commit-pinned, `file:line` cited,
with explicit "this other doc is stale" callouts, and they hold up under
checking. Most repos are nothing like that. Do not let a good `CLAUDE.md` here
train you to trust one elsewhere, and note that even in this estate the rigorous
docs coexist with flatly false ones (`cleverclass.ai/docs/`,
`Emerge-Design/CLAUDE.md` on the notify form). Rigor of presentation is not
evidence of accuracy.

## Phase 1 — cheap solo recon (do this yourself, do not delegate)

```bash
R=<repo path>
git -C $R remote -v; git -C $R rev-parse --abbrev-ref HEAD
git -C $R status --short | head -20; git -C $R log --oneline -10
ls $R
find $R -name CLAUDE.md -not -path "*/node_modules/*"
cat $R/package.json $R/go.mod $R/requirements.txt $R/pyproject.toml $R/composer.json 2>/dev/null
```

You need this before writing agent prompts, or you will brief the agents wrong.
Note the branch and dirty count — **these repos are usually on a feature branch
with an uncommitted tree**, and an audit that assumes `main` describes fiction.

## Phase 2 — fan out read-only analysts

Spawn `codebase-analyst` agents **in one message** so they run concurrently.
Partition by *domain*, never by "go read the repo" — overlapping agents produce
overlapping prose and no extra coverage.

Standard partition for a full-stack repo (4-6 agents):

| Agent | Owns |
|---|---|
| structure | dir layout, source vs build output, entry points, build/test/lint commands quoted verbatim |
| backend | frameworks, HTTP layer, persistence, migrations, config loading, error handling |
| frontend | framework/router/styling, component conventions, data fetching, forms, client auth |
| data | schema, migrations, core entities, relationships, seed/fixture state |
| security+auth | authn/authz, session/cookie flags, CSRF, rate limiting, tenancy, secret handling |
| deploy+CI | Dockerfiles, compose, workflows, deploy scripts, env var inventory (names only) |

Scale down for a single-service repo; drop `frontend` for an API-only repo.

### Prompt template for each analyst

Every prompt must contain these five things or the report comes back useless:

1. **The exact paths** it owns, and an instruction not to wander outside them.
2. **`file:line` evidence required for every claim.**
3. **Named suspicions to verify or refute** — feed it the specific things you
   doubt. This is where most of the value comes from.
4. **"Explicitly flag anything that contradicts what I stated."** Agents default
   to agreeable; you must ask for the contradiction.
5. **Read-only. Do not edit any file.** And: *never print secret values, variable
   names only; if you open a `.env`, report only that it exists and stop.*

Ask for tables over prose, and require the literal word `NONE` where something is
absent — "no tests found" and silence are indistinguishable otherwise.

## Phase 3 — reconcile, do not concatenate

Agent reports disagree with each other and with the docs. That is the point.
For each conflict, go look yourself; the tiebreaker is code, not the more
confident agent. Subagents are sometimes wrong or over-hedged — spot-check any
claim you are about to write down as load-bearing.

Specifically re-check yourself:

- anything about **what is deployed and where** (most often stale)
- anything about **auth actually being enforced** (frequently decorative)
- any **command you are about to publish** — read it out of `package.json` or
  the `Makefile`; do not reconstruct it from memory

## Phase 4 — write the CLAUDE.md files

One at the repo root, plus one in each significant subdirectory (`backend/`,
`frontend/`, `deploy/`, `server/routes/`, test dirs with unusual runners). The
repos here carry 4-7 each; that granularity works because a Claude editing
`backend/` should not have to read frontend prose.

### Root CLAUDE.md structure

```markdown
# <repo> — orientation
**Verified <YYYY-MM-DD>.** Claims below were checked against code unless marked *unverified*.

## What this is
Two sentences. What it does, who it serves. Not marketing.

## Stack
Table. Exact versions read from the manifest, never approximated.

## Layout
Table of the 5-10 dirs that matter. Mark BUILD OUTPUT explicitly.

## Commands
install / dev / build / test / lint — quoted verbatim from scripts.
State plainly if there are NO tests.

## Architecture
The 3-5 things that are non-obvious from the file tree.
Contracts that cross module boundaries.

## Traps
The section that earns the file. See below.

## Conventions
What a Claude must match. What it must not touch.
```

### The Traps section

This is the highest-value part of the document. A trap qualifies if it
**fails silently or misleads** — not merely "is complicated". Real examples from
this ecosystem, as a calibration bar:

- Docker-published `0.0.0.0` ports on `editone-prod` bypass UFW, so a service
  looks firewalled in `ufw status` while it is open to the internet.
- Go tests **skip** silently when `TEST_DATABASE_URL` is unset, so CI goes green
  proving nothing. `ci.yml` asserts the skip message is absent — keep that check.
- The retrieval engine falls back to a hashing embedder on a wrong HuggingFace
  cache layout and serves plausible nonsense at `200 OK`. Check
  `/health/providers?strict=true`.
- `phd-monitoring` role flags are `enum('true','false')` **strings**, so negating
  one is never true and the guard never fires. Do not "simplify" a comparison
  against the string `'false'` into a truthiness check.
- Windows `core.autocrlf=true` plus a Linux runtime: without `.gitattributes`
  forcing LF, `deploy.sh` checks out with CRLF and dies with a `bash\r` "No such
  file or directory" error naming neither the file nor the cause.

Write each trap as: the symptom you would actually observe, then the cause, then
the check or fix.

### Rules for the prose

- Date it, and say what was verified versus assumed.
- Generated files are not source — name the generator and the rebuild command.
  (`Emerge-Design` `.dc.html` artboards are build output of `src/`.)
- Prefer "do X" over "X is recommended".
- Never write a command you have not seen in a file.
- No secret values. Variable names and file locations only.

## Phase 5 — independent check

Run `final-reviewer` against the written `CLAUDE.md` files with the instruction:
*try to prove these documents would mislead a Claude who trusted them.* Fix what
it finds. Unsupported claims get deleted, not softened.

## Definition of done

- every significant subdir has a `CLAUDE.md`
- every command in them was read out of a real file
- every trap has a symptom, a cause, and a check
- conflicts between agents were resolved by reading code, not by averaging
- no secret values anywhere in the output
- the root file states its verification date
