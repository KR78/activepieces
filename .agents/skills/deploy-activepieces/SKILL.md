---
name: deploy-activepieces
description: |
  Build + deploy custom Activepieces changes to ap.vec.run. Builds the Docker image
  from the KR78/activepieces fork's lab branch, restarts the compose stack, and
  verifies the live site. Use when modifying Activepieces source (frontend, backend,
  or pieces), after merging upstream, or when ap.vec.run needs to reflect local
  changes. Also covers the three-branch git model + how to keep customizations
  during upstream sync.
version: "1.0"
alwaysApply: false
category: deployment
tags:
  - deployment
  - activepieces
  - docker
  - nginx
  - fork
  - rs4k
---

# Deploy Activepieces Custom Changes

Builds + deploys the custom Activepieces instance at **`https://ap.vec.run`** (Tailscale-only).

**You are on the rs4k server (`159.195.196.111` / Tailscale `100.119.5.91`).**
The repo is at `/root/projects/activepieces`. You should be on the `lab` branch.

## What you're deploying

A custom Docker image built from source (the KR78/activepieces fork) — NOT the
upstream ghcr image. This lets us add custom features.

| Component | Value |
|---|---|
| Container (app) | `activepieces-app` |
| Container (workers) | `activepieces-worker-1`, `activepieces-worker-2` (2 replicas) |
| Image | `activepieces:custom` (built locally) |
| Port | `127.0.0.1:8085` → container `:80` |
| Domain | `ap.vec.run` (Tailscale-only) |
| nginx | `/etc/nginx/sites-enabled/ap.vec.run` → `proxy_pass :8085` |
| Compose file | `/root/infrastructure/server-provision/configs/activepieces/docker-compose.custom.yml` |
| Compose project name | `activepieces` |
| Env file | `/root/projects/activepieces/.env` (secrets — gitignored) |
| Dependencies | PostgreSQL 14 (pgvector), Redis 7 |

## The three-branch git model

```
upstream (activepieces/activepieces)  ← read-only mirror
   │
   ▼
main (pure upstream mirror)           ← never commit customizations here
   │
   ▼ (merge)
lab  (main + customizations)           ← this is what ap.vec.run builds from
   │
   ▼ (feature branches branch off here, PR-ready)
feat/<name>
```

| Branch | Purpose | Pushes to |
|---|---|---|
| `main` | Pure upstream mirror (fast-forwards from `upstream/main`) | `origin/main` |
| `lab` | `main` + our customizations — **the build source** | `origin/lab` |
| `feat/<name>` | PR-ready feature branches | (stay local until PR) |

**Current state:** `lab` is the active branch + what the container builds from.

## Deploy: the fast path

After editing source files on the `lab` branch:

```bash
cd /root/projects/activepieces

# 1. Commit your changes (clean source for the image build)
git add -A
git commit -m "feat: <description>"

# 2. Deploy (builds image + restarts stack + verifies)
/root/infrastructure/server-provision/scripts/deploy-activepieces.sh

# 3. (optional) push the lab branch to the fork
git push origin lab
```

The deploy script does:
1. Checks out `lab`
2. Runs `docker build -f Dockerfile -t activepieces:custom .`
3. `docker compose down` + `up -d` (with `--env-file` for secret interpolation)
4. Waits 25s for boot, verifies HTTP 200 + container health

### Deploy flags

```bash
deploy-activepieces.sh                # build + restart (uses Docker cache)
deploy-activepieces.sh --no-cache     # clean rebuild (no Docker cache)
deploy-activepieces.sh --check        # just verify, don't rebuild
```

## Deploy: manual (if the script isn't available)

```bash
cd /root/projects/activepieces

# Build the image
docker build -f Dockerfile -t activepieces:custom .

# Restart the stack (MUST use --env-file for postgres password interpolation)
cd /root/infrastructure/server-provision/configs/activepieces
docker compose -p activepieces --env-file /root/projects/activepieces/.env \
  -f docker-compose.custom.yml down
docker compose -p activepieces --env-file /root/projects/activepieces/.env \
  -f docker-compose.custom.yml up -d

# Verify (wait ~25s for boot)
sleep 25
curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:8085/
docker inspect --format='{{.State.Health.Status}}' activepieces-app
```

## The build: what happens

The Dockerfile is multi-stage (`/root/projects/activepieces/Dockerfile`):

1. **base** — `node:24.14.0-bullseye-slim`, installs bun 1.3.1, python3, g++, git, poppler
2. **build** — `bun install` + `turbo run build` (builds web, api, engine, worker)
3. **run** — final image, serves on `:80`

**Build time:** ~5-8 minutes (cold), ~2-3 minutes (warm cache).
**Image size:** ~1.8 GB.
**Memory:** the build needs 3GB+ free (Node/Bun can be memory-hungry).

## After the build: verify

```bash
# Container health
docker inspect --format='{{.State.Health.Status}}' activepieces-app
# → "healthy"

# HTTP (internal)
curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:8085/
# → HTTP 200

# HTTP (through nginx + Tailscale)
curl -s -o /dev/null -w 'HTTP %{http_code}\n' \
  --resolve ap.vec.run:443:100.119.5.91 https://ap.vec.run/
# → HTTP 200

# Logs (check for errors)
docker logs activepieces-app --tail 30
```

## Syncing with upstream (upgrading Activepieces)

When a new Activepieces version is released upstream:

```bash
cd /root/projects/activepieces

# 1. Fetch upstream
git fetch upstream

# 2. Update main (pure mirror)
git checkout main
git merge --no-edit upstream/main
git push origin main

# 3. Merge main into lab (preserves customizations)
git checkout lab
git merge --no-edit main
# If conflicts: resolve them, then git add + git commit
git push origin lab

# 4. Rebuild + restart
/root/infrastructure/server-provision/scripts/deploy-activepieces.sh --no-cache
```

### If the merge has conflicts

```bash
git status                          # see conflicted files
# edit each conflicted file, resolve the conflict markers
git add <resolved-files>
git commit                          # completes the merge
git push origin lab

# then rebuild
/root/infrastructure/server-provision/scripts/deploy-activepieces.sh --no-cache
```

**Conflict resolution principle:** prefer upstream's changes for core functionality; keep our customizations for custom features. If unsure, preserve the `lab` (our) side.

## Where to make customizations

| Type | Where | Example |
|---|---|---|
| New piece (integration) | `packages/pieces/` | A custom API connector |
| Modify frontend UI | `packages/web/` | Custom branding |
| Modify backend API | `packages/server/api/` | Custom endpoint |
| Modify engine | `packages/engine/` | Execution logic |
| Docker/build changes | `Dockerfile` or `docker-compose.custom.yml` | Add an env var |
| Worker URL routing | `packages/server/api/src/app/workers/machine/machine-service.ts` | The `getInternalUrl` patch (see below) |

## Customizations on the lab branch

### getInternalUrl patch (machine-service.ts)

The app sends `PUBLIC_URL` to workers via Socket.IO settings. Workers use this
URL for piece bundle fetches. The upstream code uses `getPublicUrl` which returns
`https://ap.vec.run` — but the app container only listens on port 80 (HTTP).
nginx terminates TLS, not the app. So the worker tried HTTPS on port 443 →
`ECONNREFUSED`.

**The patch:** `machine-service.ts` uses `getInternalUrl` instead of `getPublicUrl`.
`getInternalUrl` checks `AP_INTERNAL_URL` first (set to `http://ap.vec.run` in `.env`),
so the worker fetches over HTTP on port 80. The network alias `ap.vec.run` → app
container makes Docker DNS resolve it correctly.

**Files changed:**
- `packages/server/api/src/app/workers/machine/machine-service.ts` — `getPublicUrl` → `getInternalUrl`
- `.env` — `AP_INTERNAL_URL=http://ap.vec.run`
- `docker-compose.custom.yml` — network alias `ap.vec.run` on the app service

**During upstream sync:** this patch may conflict if upstream changes `machine-service.ts`.
Resolve by keeping the `getInternalUrl` version (our customization).

## The .env file (secrets — gitignored)

Located at `/root/projects/activepieces/.env`. Key vars:

| Var | Purpose |
|---|---|
| `AP_ENCRYPTION_KEY` | 32-char hex — encrypts stored credentials (MUST be exactly 32 hex chars) |
| `AP_JWT_SECRET` | 64-char hex — signs JWTs |
| `AP_POSTGRES_PASSWORD` | Postgres superuser password |
| `AP_FRONTEND_URL` | `https://ap.vec.run` (the public URL) |
| `AP_EXECUTION_MODE` | `UNSANDBOXED` (no isolate sandbox — simpler, works on any host) |
| `AP_TELEMETRY_ENABLED` | `false` (telemetry off) |
| `AP_EXECUTION_DATA_RETENTION_DAYS` | `30` — **REQUIRED**. Without this, `generateEngineToken` throws + piece bundle fetches fail with 403. |
| `AP_INTERNAL_URL` | `http://ap.vec.run` — the internal URL the app sends to workers (HTTP, not HTTPS). Requires the network alias. |

**Never commit `.env`** — it's gitignored. If you need to regenerate secrets, use:
```bash
openssl rand -hex 16   # for AP_ENCRYPTION_KEY (32 hex chars)
openssl rand -hex 32   # for AP_JWT_SECRET
```

## Debugging

### Build fails

```bash
# Check the build output (last 30 lines)
docker build -f Dockerfile -t activepieces:custom . 2>&1 | tail -30

# Common causes:
#   - Out of memory (bun/turbo needs 3GB+ free)
#   - Network issue downloading bun
#   - Node version mismatch (needs node:24)
```

### Container starts but HTTP 502

```bash
# The docker-proxy process may have died (known issue)
# Force-recreate the container:
cd /root/infrastructure/server-provision/configs/activepieces
docker compose -p activepieces --env-file /root/projects/activepieces/.env \
  -f docker-compose.custom.yml up -d --force-recreate

# Check container logs:
docker logs activepieces-app --tail 50
```

### Postgres keeps restarting

```bash
# Check if the password was set correctly
docker logs activepieces-postgres 2>&1 | tail -10

# If "superuser password is not specified":
#   The --env-file flag is missing from docker compose.
#   ALWAYS use: --env-file /root/projects/activepieces/.env
```

### Piece bundle fetch fails with 403 Forbidden

Two causes, both must be fixed:

**1. Missing `AP_EXECUTION_DATA_RETENTION_DAYS`** — `generateEngineToken` throws `SYSTEM_PROP_NOT_DEFINED`, the worker gets no engine token, the bundle endpoint sees `principalType: UNKNOWN` → 403.

```bash
echo "AP_EXECUTION_DATA_RETENTION_DAYS=30" >> /root/projects/activepieces/.env
```

**2. Worker can't resolve `ap.vec.run`** — the app sends `PUBLIC_URL=https://ap.vec.run` to the worker via Socket.IO settings. The worker uses this URL for piece bundle fetches, but from inside Docker it can't reach the Tailscale IP. Fix: the `app` service has a network alias `ap.vec.run` in the compose file so Docker DNS resolves it to the app container:

```yaml
app:
  networks:
    activepieces:
      aliases:
        - ap.vec.run   # worker resolves ap.vec.run → app container
```

If the alias is missing, add it + restart:

```bash
cd /root/infrastructure/server-provision/configs/activepieces
docker compose -p activepieces --env-file /root/projects/activepieces/.env   -f docker-compose.custom.yml up -d --force-recreate
```

Verify the worker can resolve it:
```bash
docker exec activepieces-worker-1 getent hosts ap.vec.run
# should show: 172.x.x.x  ap.vec.run
```

### Changes don't appear on ap.vec.run

```bash
# 1. Did you rebuild the image? (changes to source require a rebuild)
docker build -f Dockerfile -t activepieces:custom .

# 2. Did you restart the container? (a running container uses the old image)
cd /root/infrastructure/server-provision/configs/activepieces
docker compose -p activepieces --env-file /root/projects/activepieces/.env \
  -f docker-compose.custom.yml up -d

# 3. Is the browser caching? (hard refresh: Cmd+Shift+R / Ctrl+Shift+F)
```

## What NOT to do

1. **Don't commit to `main`** — it's a pure upstream mirror. Customizations go on `lab`.
2. **Don't run `docker compose down -v`** — the `-v` flag deletes the postgres + redis data volumes.
3. **Don't change the port from `:8085`** without updating both the compose file AND the nginx `proxy_pass`.
4. **Don't forget `--env-file /root/projects/activepieces/.env`** when running compose manually — without it, postgres password interpolation fails + postgres won't start.
5. **Don't push to `upstream`** — only push to `origin` (KR78/activepieces). The `upstream` remote is read-only.
6. **Don't change `AP_ENCRYPTION_KEY`** after data exists — it encrypts all stored credentials. Changing it makes existing connections undecryptable.

## Related

- Infra repo: `scripts/deploy-activepieces.sh`
- Infra repo: `configs/activepieces/docker-compose.custom.yml`
- Infra repo: `configs/nginx/ap.vec.run`
- Upstream docs: https://www.activepieces.com/docs/install/options/docker-compose
