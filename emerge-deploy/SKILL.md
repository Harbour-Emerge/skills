---
name: emerge-deploy
description: Ship changes to the Emerge estate on the editone-prod VPS — Jenkins auto-deploys on merge, the manual deploys that remain (Marketplace API, Meet, poll, companion, debate, timeline, Vercel), the deploy lock, SHA-tagged images with keep=true rollbacks, health polling and rollback. Use before deploying, writing a Dockerfile/compose/Jenkinsfile, or debugging a failed deploy.
---

# Deploying the Emerge estate

**Verified 2026-10-16.** Read `box-ops` before running anything on the server and
`emerge-context` if you are unsure which repo deploys where.

Everything deploys to one server, `editone-prod` (the demo environment). The GCP
demo box and its GitHub Actions deploys are gone; do not copy patterns from them.

## Two paths

### 1. Jenkins: merge and it deploys

Jenkins at https://devops.tryemerge.in deploys a repo when its deploy branch moves.
The branch is `main`, except `master` for lms-grasp, lms-lesson-planner and
lms-paper-setter-school. PRs get tests only.

On Jenkins: CAS, Survey-Forms, emerge-core, Internal-Admin-Console, emerge-mail,
emerge-devops, lms-retrieval-engine, marketplace-processor, lms-board, lms-mind-map,
lms-flashcard-engine, lms-slide-builder, lms-lab-engine, lms-grasp,
lms-lesson-planner, lms-paper-setter-school, AnswerSheet-Checker,
lms-paper-orchestrator. Check the live list rather than trusting this one:

```bash
ssh editone-prod 'curl -s -u admin:$(cat /root/.jenkins-api-token) "http://127.0.0.1:8082/job/emerge/api/json?tree=jobs\[name\]"'
```

What a build does, from the repo's `Jenkinsfile` (one `emergeDeploy(...)` call into the
shared library in `Harbour-Emerge/emerge-devops`):
1. runs the repo's tests;
2. takes `/root/deploy.lock` (waits up to 20 minutes);
3. syncs the commit into `/opt/emerge/<name>/src` (git reset for CAS, retrieval,
   forms and the console; tar overlay elsewhere) and writes `DEPLOYED_COMMIT`;
4. runs the repo's own `deploy/deploy.sh` as root;
5. polls the health URL, relabels `:rollback` with `keep=true`, appends to
   `.deploy-history` and releases the lock.

After merging, watch the build before you call it deployed:

```bash
ssh editone-prod 'curl -s -u admin:$(cat /root/.jenkins-api-token) "http://127.0.0.1:8082/job/emerge/job/<repo>/job/<branch>/lastBuild/api/json?tree=number,result,building"'
```

Known behaviours:
- The whole pipeline has a time limit. A build that queued behind other deploys
  can time out and show `ABORTED` with green tests; rerun it with a POST to
  `.../job/<branch>/build`.
- Timing-sensitive tests can fail when several builds run at once. Read the
  failure before rerunning; rerun only when the change cannot have caused it.
- A failed build leaves the previous container serving.

Adding a repo: a `Jenkinsfile` in the repo, an entry in `casc/repos.yaml` of
emerge-devops, a controller restart, and the GitHub App `emerge-devops-ci` installed
on the repo.

### 2. Manual deploys

These are not on Jenkins and are deployed by hand on the box:

| Thing | Where | How |
|---|---|---|
| Marketplace API (`Harbour-Emerge/Marketplace`) | `/opt/emerge/marketplace` | tar overlay of the commit into `src/`, move deleted paths into `stale-<sha>/` first, `pg_dump` backup, then `src/backend/deploy/deploy.sh` |
| Meet | `/opt/emerge/meet` | `src/deploy/deploy.sh`; refuses while a class is live unless `MEET_DEPLOY_DURING_CLASS=yes` |
| Meet media stack | `/opt/emerge/meet` | `MEET_SRC=/opt/emerge/meet/src docker compose -p meet-media -f src/deploy/media/docker-compose.yml --env-file /opt/emerge/meet/.env up -d --build` |
| Poll, Companion, Debate | `/opt/emerge/{poll,companion,marketplace-debate}` | copy the merged tree into `src/` (keep `src/vendor/` wheels), write `src/DEPLOYED_COMMIT`, run `src/deploy/deploy.sh` |
| Timeline engine | `/opt/emerge/marketplace-engines` | built from the lms-visual-playground checkout via that directory's `deploy.sh` |
| Storefronts | Vercel | the owner deploys; report the SHA |

Manual deploy routine:
1. `mkdir /root/deploy.lock` and write an owner file.
2. Back up anything you will overwrite, and the database if the deploy migrates.
3. Copy the tree. **A tar overlay keeps files deleted upstream**; move them aside
   into `stale-<sha>/` or copy into a fresh directory and swap it in.
4. Run the deploy script, then poll health with a deadline.
5. Release the lock and check the public URL through nginx, not only loopback.

Wheels for emerge-llm, emerge-retrieval and emerge-subjects come from the GitHub
releases into each stack's `src/vendor/`. Never move a published tag; release a new
version.

## Patterns every stack follows

- **Pin the compose project `name:`** and call compose with `-f`.
- **Tag images with the commit SHA** and keep a `:current` / `:rollback` pair. Re-point
  `:rollback` only after health passes, and label it `keep=true` so the nightly
  image prune does not delete it.
- **Publish on `127.0.0.1` only.** Docker publishes bypass UFW.
- **Poll health with a deadline; never sample once.** A request right after
  `up -d` returns `000` from a healthy service.
- **Separate liveness from correctness.** Container `HEALTHCHECK` on plain
  `/health`; the deploy script checks the deeper signal (providers, callbacks, CSP).
  Use a GET with `-D -` for header checks; not every app answers HEAD.
- **Migrate as a separate step** before rolling, never as an entrypoint.
- **Keep secrets on the box** in `.env` (mode 600). CI never carries them.
- **Multi-stage builds, non-root runtime user**, `CGO_ENABLED=0` for Go.
- **Preflight what fails silently**: required env keys, the effective env from
  `docker compose config` (inline `environment:` overrides `env_file:`), port
  ownership, required vendor files. Fail before stopping anything.
- **`.gitattributes` forcing LF** in every repo with a shell script. These repos are
  edited on Windows; a CRLF `deploy.sh` dies with `bash\r: No such file or
  directory`.

## nginx vhosts

Public hostnames are host-nginx vhosts in `/etc/nginx/sites-enabled/`. Repos that
ship a vhost keep it in `deploy/nginx-<name>.conf`. This nginx needs
`listen 443 ssl http2;` rather than a separate `http2 on;`. Back up, edit, run
`nginx -t`, then `systemctl reload nginx`. Get new certificates with certbot through
nginx once the CNAME resolves.

## Rollback

```bash
docker tag <image>:rollback <image>:current
docker compose -f <compose> up -d <service>
```

Or redeploy the previous commit. The `.deploy-history` file lists past commits.

## After a failed deploy

1. `docker ps` and `docker logs <container> --tail=100`.
2. For Jenkins, read the console: `.../job/<branch>/<n>/consoleText`.
3. `ls -d /root/deploy.lock && cat /root/deploy.lock/owner`. A lock left behind by a
   dead deploy blocks every later one.
4. Roll back first, diagnose after.

Volumes that must not be lost: database volumes, `meet-minio` recordings, retrieval
data in `/opt/retrieval/data`, and any `caddy_data` or certificate store.

## A green deploy is not a reviewed product

Deploying does not mean a product passed review. Report what was verified (health,
the public URL, a QA flow) and what was not.
