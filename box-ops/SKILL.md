---
name: box-ops
description: Operate the shared mentis-demo VM and the other production hosts safely — SSH access, the four-tenant container map, port allocation, disk and resource facts, and the rules about what must never be touched. Use before running any command on a remote host, debugging a container, or claiming a port.
---

# Box operations

Rules for touching remote hosts in this ecosystem. Read this **before** the first
SSH command, not after something breaks.

## The one rule

**The demo box is shared by four unrelated tenants. Anything you did not create
is someone else's production.**

Do not restart, rebuild, prune, or reconfigure a container, volume, network, or
port you did not create. If a change genuinely requires another tenant's
container to be restarted, **say so and let a human do it**. This is not a
formality — a stray `docker system prune -a` here takes down three products that
have nothing to do with your task.

## Hosts

| Alias | Address | User | Key | What runs there |
|---|---|---|---|---|
| `lms` | `34.131.254.234` | `founder_mentis` | `~/.ssh/lms_deploy_key` | The shared demo box — Exam Studio, older LMS stack, CleverClass, paper-checker |
| `editone-prod` | `200.141.15.77` | `root` | `~/.ssh/editone-prod` | Hostinger VPS (`srv1888767.hstgr.cloud`), CleverClass/EditOne production |

```bash
ssh lms            # passwordless sudo available
ssh editone-prod   # root; treat every command as production
```

**`editone-prod` is undocumented in every repo.** No repo on `D:\` references
that host, IP, or hostname — it exists only in `~/.ssh/config`. Before deploying
anything there, confirm with the owner what it actually serves; do not infer it
from `cleverclass.ai/deploy/`, which targets the demo box instead.

`~/.ssh/editone-prod-credentials.txt` exists on this workstation. Do not read,
print, or copy it. There are also `.exposed.bak` key files in `~/.ssh` — treat
those as compromised and never use them.

GCP project for the demo box: `project-63fe9c6a-3165-455d-a7b`. Network tags
`allow-8080`, `http-server`, `https-server`.

**Those tags do not mean 443 is reachable.** `EMERGE_SETUP.md` says 443 is open
at the firewall; `Survey-Forms/deploy/Caddyfile:24-26` records the opposite,
**verified by probing from outside on 2026-08-26**. Trust the probe, and re-probe
before planning anything that needs 443 rather than trusting either document.

## The demo box tenant map

| Containers | Tenant | Yours to touch? |
|---|---|---|
| `exam-studio-{retrieval,questionbank,orchestrator,nginx}` | Exam Studio | yes, if the task is Exam Studio |
| `lms-deploy-{engine,tutor,ops,nginx,dozzle}-1` | older LMS stack | no |
| `cc-{web,api,proxy}` | CleverClass | yes, if the task is CleverClass |
| `paper-checker-paper-checker-1` | paper-checker | no |
| `emerge-forms-{backend,frontend,caddy,...}` | emerge-forms | yes |

Confirm before assuming: `docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'`

## Port allocation

Public: `80`, `8050`, `8051`, `8055`, `8070`, `8080`, `9000`
Loopback only: `8060`, `8062`, `8063`, `8065`, `8067`, `8068`, `8069`, `18000`

| Port | Owner |
|---|---|
| `80` | another tenant's nginx — **cannot be claimed** |
| `443` | **not currently reachable** — see the firewall note above. Reserved for emerge-forms if and when HTTPS is turned on |
| `8070` | emerge-forms Caddy, the current public port (`PUBLIC_PORT`) |
| `8067` / `8068` / `8069` | emerge-forms backend / frontend / local Postgres, loopback |
| `8050` / `8051` | CleverClass storefront / admin (`cc-proxy`) |
| `8055` | `exam-studio-nginx` |
| `8060` | lms-retrieval-engine (container `8000`, override `EXAM_PORT_RETRIEVAL`) |
| `8062` | lms-questionbank-engine (container `8200`, override `EXAM_PORT_QUESTIONBANK`) |
| `8063` | lms-paper-orchestrator, Exam Studio entry point (container `8300`, override `EXAM_PORT_ORCHESTRATOR`) |
| `8065` | emerge-forms Caddy HTTP listener, loopback |
| `8067` | emerge-forms backend healthz, loopback |
| `18000` | CleverClass API build-only loopback |

Before claiming a port: `sudo ss -tlnp | sort -t: -k2 -n`

## Facts that will mislead you

**`df /` lies about disk space.** `/` shows ~3 GB free, but Docker's data root is
`/opt/lms/docker` on a **99 GB disk with ~51 GB free**. Check the right one:

```bash
df -h /opt/lms/docker    # the number that matters
docker system df         # what Docker thinks it is using
```

Never act on a `df /` reading here. A "disk full" conclusion from it is wrong.

**Port 80 is not available for ACME.** It belongs to another tenant's nginx.
Anything needing a certificate must use **TLS-ALPN-01 over 443**. Do not add a
`80:80` mapping to a compose service — it fails to bind and takes the stack down
with it.

**The VM service account has narrow scopes.** The SQL Admin API refuses with
`ACCESS_TOKEN_SCOPE_INSUFFICIENT`. Widening scopes requires stopping a VM that
serves four tenants — so it does not happen casually. Work around it or ask.

**RAM is 7 GB total, ~5 GB available.** Model-loading services (the retrieval
engine's BGE embedder, cross-encoder rerank) are the memory risk. Check before
starting anything large: `free -h`.

## Database

```
instance   project-63fe9c6a-3165-455d-a7b:us-central1:demo-db
public IP  34.132.193.251:5432
database   survey-db
role       survey_app
```

The application **must not run as `postgres`**. The `survey_app` role is created
by `Survey-Forms/deploy/create-db-user.sql`, with the password passed in at run
time and never stored in the file.

Cloud SQL access is IP-allowlisted. If the box cannot reach the database, the
first thing to check is whether `34.131.254.234/32` is in the `demo-db`
authorized networks — that has been the cause before.

## DNS and mail

`tryemerge.in` runs on **Hostinger** nameservers (`aurora.dns-parking.com`,
`nebula.dns-parking.com`). Apex and `www` point at Vercel.

**DNS records are changed in the Hostinger panel** — not in Vercel, not from a
repo. There is no Hostinger API credential on this machine, so a Claude
**cannot** make DNS changes. Ask; do not attempt.

Mail goes through Hostinger SMTP as `admin@tryemerge.in`, via
`smtp.hostinger.com:465` implicit TLS, app password used **with** its dashes.
The relay was verified accepting mail on 2026-08-26, but **relay acceptance is
not inbox delivery** — SPF/DKIM for `tryemerge.in` have not been checked, so a
strict receiver may still file it as spam. Confirm before a first real send to a
customer.

Test sends go to `akarshagri121@` and `masterreyan@`, never to real recipients.

## Vercel — team `saturnalia`

| Project | Production URL |
|---|---|
| `emerge-landing` | https://www.tryemerge.in |
| `emerge-student-portal` | `emerge-student-portal.vercel.app` |

The CLI on this machine is authenticated and scoped to `saturnalia`. No domains
are registered at team level — they attach per project.

## Safe triage sequence

Read state before changing it. In order:

```bash
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
docker logs <container> --tail=100          # yours only
free -h; df -h /opt/lms/docker
sudo ss -tlnp | sort -t: -k2 -n
```

Certificate problems appear in the **Caddy** logs, not the app logs:

```bash
docker logs emerge-forms-caddy --tail=50
```

## Every stack pins its compose project name — do not remove it

```yaml
name: emerge-forms   # Survey-Forms/deploy/docker-compose.yml:25
name: exam-studio    # lms-paper-orchestrator/deploy/docker-compose.yml:23
name: cleverclass    # cleverclass.ai/deploy/docker-compose.yml:8
```

All three carry near-identical comments giving the reason: **a `docker compose
down` run from the wrong directory can otherwise reach containers belonging to
another tenant.** Without an explicit `name:`, Compose derives the project name
from the directory, and two checkouts named `deploy/` collide.

Consequences for you:

- **Always pass `-f <path>` explicitly** and check the project name before any
  `down`, `stop`, or `rm`: `docker compose -f deploy/docker-compose.yml config --format json | head`
- Never add a new stack to this box without a pinned `name:`.
- `docker compose down` with no `-f` from a random cwd is how you take down
  someone else's product. Prefer `stop` over `down`, and prefer naming the
  service.

## Never, on any shared host

- `docker system prune` in any form
- `docker volume rm` — `caddy_data` holds the ACME account key and certificates;
  deleting it burns Let's Encrypt rate limit. `media` holds respondent-visible
  uploads.
- restarting the Docker daemon
- editing another tenant's compose file, nginx config, or `/opt/lms/*`
- `migrate:fresh`, `db:seed --force`, or any destructive migration against a
  shared database
- changing firewall rules or network tags
- stopping the VM

`cleverclass.ai/deploy/deploy.sh` already refuses `docker system prune -a` and
never touches `/opt/lms`. Match that posture in anything new you write.

## When you need something you must not do yourself

Say exactly this, with specifics: what needs to happen, on which host, to which
container, why, and what breaks if it is not done. Then stop and let a human run
it. Do not "just try it and see".
