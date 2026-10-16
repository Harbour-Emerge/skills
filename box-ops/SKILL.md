---
name: box-ops
description: Operate the Emerge server editone-prod (Hostinger VPS, the demo environment) safely — SSH/scp access, what runs where, the port map, the co-tenant you must not touch, disk and resource facts, and the rules for containers, images and the deploy lock. Use before running any command on the server, debugging a container, or claiming a port.
---

# Box operations

**Verified 2026-10-16** against the live server.

## One server, and it is the demo environment

There is exactly one server: **`editone-prod`**, a Hostinger VPS.

| Alias | Address | User | Key | Hostname |
|---|---|---|---|---|
| `editone-prod` | `200.141.15.77` | `root` | `~/.ssh/editone-prod` | `srv1888767.hstgr.cloud` |

```bash
ssh editone-prod
scp <file> editone-prod:<path>
```

- **Access is allowed.** The owner has authorised SSH and scp to this box for deploys,
  debugging, QA and config changes. You do not need to ask before connecting.
- **It is the demo environment, not production.** The name `editone-prod` is
  historical. Treat it with care because people demo from it, but it is not a
  customer-production box.
- **The GCP box `lms` (34.131.254.234) is decommissioned.** It is unreachable, and
  so is its Cloud SQL `demo-db`. Anything that still targets it is dead: the
  `lms-tutor` deploy workflow, `CAS-Admin`, `cleverclass.ai/deploy/`, the old
  `lms-paper-orchestrator/deploy/`, `emerge-frontend/enable-tls.sh`,
  `/opt/editone/deploy/offsite.sh` and `replicate-to-test.sh`. Its 2026-09-22 dumps
  are in `/opt/emerge/migration`.
- Do not read, print or copy `~/.ssh/editone-prod-credentials.txt`. Never use any
  `.exposed.bak` key in `~/.ssh`; treat those as compromised.

## The co-tenant: never touch `ajeet`

The VPS is shared with tenant **`ajeet`** (uid 1000). Their services include Aira
Studio, FlyBrain, redroid, guacamole, omniroute, zetta2, `ava-*`, `gpu-chat-search-*`,
`shriramleela_web` and the `gta-v`, `makkhi-world`, `aira-studio` and `flybrain`
nginx vhosts. They live in:
- `/home/ajeet` and their systemd units;
- `/opt/{zetta2,zetta2v2,redroid,hopper,jadx,guacamole,aira-certbot}`.

Do not restart, rebuild, reconfigure or delete anything of theirs. If a change of
yours needs one of their services touched, say so and let a human do it.

## What is ours

| Path | What |
|---|---|
| `/opt/emerge/<name>` | One Emerge stack per directory: `src/` (the deployed tree), `.env` (mode 600), `DEPLOYED_COMMIT`, `.deploy-history` |
| `/opt/emerge/marketplace` | Marketplace API, worker and processor relay |
| `/opt/emerge/marketplace-engines` | Plugin engines: slides, mindmap, timeline, flashcards, molecule-bench, emerge-llm |
| `/opt/emerge/meet`, `/opt/emerge/meet-livekit` | Meet API, media stack, LiveKit |
| `/opt/emerge/devops` | Jenkins controller; `/opt/emerge/track` is Loki/Alloy/Grafana |
| `/opt/emerge/db` | `emerge-db`, Postgres 18 shared by CAS, core, survey, marketplace and mail |
| `/opt/retrieval` | Retrieval data; the stack is `/opt/emerge/retrieval` |
| `/opt/papersetter` | Paper setter (lms-paper-setter-school), outside `/opt/emerge` |
| `/opt/emerge-demo` | Older demo stack (debate, gapanalyzer, lessonplanner, omr, papersetter) |
| `/opt/editone` | CleverClass, a Docker Swarm stack `editone` (8050, 18000) |
| `/opt/exam-studio` | Exam Studio. Retired: nginx, orchestrator and questionbank are stopped |

`exam-studio-pgvector` holds retrieval's vector index. **It must stay running**
even though Exam Studio is retired.

## Hardware and disk

- 8 cores and 31 GB RAM. `free -h` before starting anything large.
- One 387 GB disk; Docker's root is `/var/lib/docker` on it, so `df -h /` is the real
  number here.

## Networking

- **Host nginx owns 80 and 443** and every `*.tryemerge.in` vhost
  (`/etc/nginx/sites-enabled/`). Certificates come from certbot HTTP-01 through
  nginx; `/etc/letsencrypt/renewal-hooks/deploy/nginx-reload.sh` reloads it.
  Run `nginx -t` before every reload.
- **Docker-published ports bypass UFW.** Publish on `127.0.0.1:` only. A `0.0.0.0`
  publish is on the internet whatever UFW says. Services that nginx reaches
  through the bridge also publish on `172.17.0.1:`.
- The only intended public container ports are LiveKit's `7881/tcp` and `7882/udp`.

## Port map (loopback unless noted)

| Port | Owner |
|---|---|
| 5432, 8072 | `emerge-db` |
| 5433 | `exam-studio-pgvector` (retrieval vectors) |
| 7880 | LiveKit API; 7881/7882 public media |
| 8061 | retrieval |
| 8065, 8067, 8068, 8070 | emerge-forms (Caddy health, backend, frontend, Caddy) |
| 8071 | CAS |
| 8073 | Internal Admin Console |
| 8076 | emerge-labs |
| 8077 | emerge-core |
| 8078 | papersetter |
| 8079 | emerge-mail |
| 8082 | Jenkins |
| 8090, 8091 | Marketplace API, processor |
| 8092, 8093, 8094 | Grafana, Loki, Alloy |
| 8100 | checker |
| 8200, 8400, 8450 | mindmap, slides, timeline engines |
| 8310 | marketplace-labs |
| 8460 | board |
| 8470 | Meet API |
| 8480, 8481 | poll, Meet object store (`meet-minio`) |
| 8490 | companion |
| 8530 | lesson planner |
| 8620 | GRASP |
| 8700 | debate |
| 9000, 9001 | `emerge-object-store` |
| 8050, 18000 | CleverClass (Swarm, public) |
| 5555, 8060, 8088, 8780, 20128 | `ajeet` — not ours |

Before claiming a port: `ss -tlnp | sort -t: -k2 -n`.

## Images and the nightly prune

`/etc/cron.d/docker-image-prune` runs
`docker image prune -af --filter until=24h --filter label!=keep=true` every night.
Any unused image older than 24 hours is deleted unless it carries `keep=true`. So:
- a `:rollback` tag must be relabelled `keep=true`, or it is gone the next morning;
- a long-lived base or vendor image needs the label too.

## The deploy lock

Every deploy, manual or Jenkins, serialises on one lock:

```bash
mkdir /root/deploy.lock && echo "<who> <what> $(date -u +%FT%TZ)" > /root/deploy.lock/owner
...
rm -rf /root/deploy.lock
```

`mkdir` is the atomic step; if it fails, read `owner` and wait. Jenkins takes the same
lock (its owner names the build) and waits up to 20 minutes for it.

## Secrets

- `.env` files are mode 600. Back up any file you edit to
  `/root/<name>.bak-<timestamp>` with `chmod 600` first.
- Never print secrets, run `printenv`, or show `docker inspect` env values. Grep for
  key names only (`grep -c '^KEY=' .env`).

## Jenkins and logs

- Jenkins: https://devops.tryemerge.in. The controller runs in a container on
  `127.0.0.1:8082`, and a host agent (`jenkins-agent.service`) runs deploy scripts as
  root. API: `curl -u admin:$(cat /root/.jenkins-api-token) http://127.0.0.1:8082/job/emerge/job/<repo>/job/<branch>/lastBuild/api/json`.
- Logs: https://track.tryemerge.in (Loki, 14-day retention).
- Both sign in through CAS. Config lives in `Harbour-Emerge/emerge-devops`.

## Safe triage sequence

Read state before changing it:

```bash
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
docker logs <container> --tail=100
free -h; df -h /
ss -tlnp | sort -t: -k2 -n
ls -d /root/deploy.lock && cat /root/deploy.lock/owner
```

## Always pass `-f` and a pinned project name

Every compose file pins `name:`. Run compose with an explicit `-f <path>` (and `-p`
where a stack is split), and prefer `stop <service>` over `down`. A bare
`docker compose down` from the wrong directory can reach someone else's stack.

## Never

- `docker system prune` in any form, or `docker volume rm`.
- Restart the Docker daemon, change UFW rules, or reboot the VPS without the owner.
- Touch anything belonging to `ajeet`.
- Destructive migrations (`migrate:fresh`, forced seeds) against `emerge-db`.
- Publish a container port on `0.0.0.0`.
- Work in `/tmp` (other sessions share it); use `mktemp -d`.

## When you need something you must not do yourself

Say what needs to happen, on which service, why, and what breaks if it is not done.
Then stop and let a human run it.
