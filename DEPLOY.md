# Plane — Deploy & Upgrade Runbook (Manual)

How to ship changes from this fork (`knbl-ai/plane`, branch `preview`) to the
production server, and how to upgrade to a new upstream Plane version.
Deployment is **manual**: build images locally → push to GHCR → pull & restart
on the server.

---

## 1. Architecture at a glance

| Piece | Value |
|---|---|
| Repo / branch | `git@github.com-knbl:knbl-ai/plane.git`, branch **`preview`** |
| Server | `ssh knbl_plane` → `deploy@IPv4_ADDRESS`, app dir **`~/plane`** |
| Orchestration | `docker-compose.yml` + `docker-compose.override.yml` (image overrides) |
| Custom images | `ghcr.io/USER/plane-{web,admin,space,backend,live,proxy}:preview` |
| Stock images | `postgres:15.7-alpine`, `valkey`, `rabbitmq`, `minio/minio` (pulled directly) |
| Database | PostgreSQL 15, db/user `plane`. Schema migrations run automatically by the `migrator` container. |
| File storage | Local **MinIO**, Docker volume `plane_uploads`, bucket `uploads` |
| Public URL | `https://knbl-plane.giize.com` (Caddy + Let's Encrypt). Toggle to IP/HTTP with `./toggle-access.sh`. |
| Secrets | `~/plane/.env` and `~/plane/apps/api/.env` on the server — **git-ignored, not in the repo.** |

The 4 frontend images (web/admin/space/live) and proxy build from the repo root /
their dirs; api/worker/beat-worker/migrator all share the **backend** image.

### The `docker-compose.override.yml` file

`docker-compose.yml` only defines how to **build** each service. The override
file maps every service to a **prebuilt GHCR image** so we pull instead of build.
Docker Compose merges any file named `docker-compose.override.yml` automatically
(no `-f` flag needed) when you run compose from `~/plane`.

```yaml
services:
  web:         { image: ghcr.io/USER/plane-web:preview }
  admin:       { image: ghcr.io/USER/plane-admin:preview }
  space:       { image: ghcr.io/USER/plane-space:preview }
  api:         { image: ghcr.io/USER/plane-backend:preview }
  worker:      { image: ghcr.io/USER/plane-backend:preview }
  beat-worker: { image: ghcr.io/USER/plane-backend:preview }
  migrator:    { image: ghcr.io/USER/plane-backend:preview }
  live:        { image: ghcr.io/USER/plane-live:preview }
  proxy:       { image: ghcr.io/USER/plane-proxy:preview }
```

> **You MUST replace `USER` with your own GHCR namespace** — your GitHub
> username (e.g. `nazar165`) or an org you can publish to. Rules:
> - **Lowercase only** — GHCR rejects uppercase (`Nazar165` → `nazar165`).
> - The **same `USER`** must be used in three places: where you `docker compose
>   push` (your build machine), this override file, and on the server that pulls.
> - This file must exist **both on your build machine and on the server** (it's
>   not the same as the repo's build config). Keep them identical.
> - Your token needs `write:packages` for that namespace to push, and the server
>   needs `read:packages` (or make the packages public) to pull.

---

## 2. Prerequisites (one-time per developer)

- Read/write access to the repo.
- A GHCR token (`write:packages`) for the namespace you publish to
  (`ghcr.io/USER/...`), and `read:packages` on the server.
  - **Where the token lives:** the current token is stored in **Passbolt**. On
    GitHub it's the Personal Access Token under the **`nazar@igentity.ai`** account
    (Settings → Developer settings → Personal access tokens), saved with the
    **"access tokens"** note. Use it as `GHCR_PAT`; if it's expired or revoked,
    regenerate it there and update the Passbolt entry.
- SSH access to the server (`ssh knbl_plane` working).
- Docker + Docker Compose v2 locally.
- Building on a non‑amd64 machine? You'll cross-build for the server's amd64 (see §5).

---

## 3. The big picture

```
  edit code on `preview`
        │
        ▼
  build 6 images locally  ──►  push to GHCR (:preview and :<git-sha>)
        │
        ▼
  on server: back up DB ─► docker compose pull ─► up -d (migrator auto-runs) ─► verify
        │
        ▼
  rollback = re-pull previous :<git-sha> + (if needed) restore DB backup
```

---

## 4. Upgrading to a new upstream Plane version

Do this when makeplane releases a new version you want.

```bash
# one-time: add the upstream remote
git remote add upstream https://github.com/makeplane/plane.git

# fetch upstream and merge the version you want onto our fork
git fetch upstream --tags
git checkout preview && git pull
git checkout -b upgrade/<version>           # e.g. upgrade/v0.30.0
git merge upstream/preview                  # or: git merge <upstream-tag>
# → resolve conflicts (see "Our custom patches" below)
```

### Our custom patches — verify they survive every upgrade
These are the fork's deliberate changes; re-check them after a merge:
- **Dockerfile pnpm fix** in `apps/{web,admin,space,live}/Dockerfile.*` — the
  `PATH` must include `$PNPM_HOME/bin` (otherwise `pnpm add -g turbo` fails the
  build; see §8).
- **Proxy config** changes (`apps/proxy/`).
- **200 MB upload limit** (`FILE_SIZE_LIMIT` and proxy `request_body max_size`).
- **Email notification** fixes in the API.

Then push the branch, open a PR into `preview`, review, and merge:
```bash
git push -u origin upgrade/<version>
```

---

## 5. Build & publish images (locally)

From the repo root:
```bash
export COMPOSE_FILE=docker-compose.yml:docker-compose.override.yml

# log in to GHCR (token needs write:packages)
echo $GHCR_PAT | docker login ghcr.io -u <your-username> --password-stdin

# the server is amd64 — set this if your machine is arm64 (e.g. Apple Silicon)
export DOCKER_DEFAULT_PLATFORM=linux/amd64

docker compose build      # builds web, admin, space, backend, live, proxy
docker compose push       # pushes ghcr.io/USER/plane-*:preview
```

Confirm the packages updated at **github.com/USER?tab=packages**.

> Tip: also tag a build with the git SHA so you can roll back to a specific
> version. Either add a second tag in `docker-compose.override.yml` temporarily,
> or `docker tag` + `docker push` each image as `...:<git-sha>` after building.

---

## 6. Deploy to the server

```bash
ssh knbl_plane
cd ~/plane

# 1. ALWAYS back up the database first
mkdir -p ~/backups
docker compose exec -T plane-db pg_dump -U plane -d plane --clean --if-exists --no-owner \
  > ~/backups/plane-db-$(date +%Y%m%d-%H%M%S).sql

# 2. pull the new images
docker compose pull

# 3. apply — the migrator runs DB migrations automatically on startup
docker compose up -d --no-build

# 4. verify
docker compose ps
docker compose logs --tail=50 migrator   # should finish cleanly; container Exited (0)
docker compose logs --tail=30 api        # gunicorn workers booting, no traceback
```

Then check the site (`https://knbl-plane.giize.com`) — log in, open a project,
confirm issues/images load.

> **DB migrations** are baked into the image. The `migrator` service runs
> `manage.py migrate` once on `up`, then exits `0`. You never run migrations by
> hand. If `migrator` logs an error, the schema step failed — see §8.

---

## 7. Rollback

Code rollback is fast if you tagged images with the git SHA:
```bash
cd ~/plane
# edit docker-compose.override.yml to point each image at the previous good SHA:
#   image: ghcr.io/USER/plane-<svc>:<previous-sha>
docker compose pull
docker compose up -d --no-build
```

If the new version ran **schema migrations** you must undo, restore the
pre-upgrade dump (§6 step 1):
```bash
docker compose down
docker compose up -d plane-db && sleep 10
docker compose exec -T plane-db psql -U plane -d plane < ~/backups/plane-db-<timestamp>.sql
docker compose up -d --no-build
```
> Forward-fixes are usually safer than restoring over a migrated DB. Restore only
> if a migration genuinely broke data and you accept losing changes since the dump.

---

## 8. Automating with CI/CD (optional)

Everything above is manual. If you later want to remove the local build/push and
the by-hand server steps, fold them into a pipeline (e.g. GitHub Actions). In a
nutshell:

- **Trigger** on push/merge to `preview`, plus a manual "run" button for ad-hoc deploys.
- **Build job:** check out the code, build all six images, and push them to GHCR
  under the **organization namespace** (`ghcr.io/knbl-ai/plane-*`), tagged with both
  `preview` and the commit SHA. Authenticate with the workflow's built-in
  **organization token** (`GITHUB_TOKEN` with `packages: write`) — **not** a
  personal/user token. This way the images are owned by the **organization** rather
  than an individual, and no developer needs their own `write:packages` PAT or
  package-create rights.
- **Image namespace:** because CI publishes to the org, point
  `docker-compose.override.yml` at `ghcr.io/knbl-ai/...` (instead of a personal
  `ghcr.io/USER/...`), and have the server pull using an **org-scoped read token**
  (or make the packages public).
- **Deploy job** (runs after build): connect to the server over SSH, back up the
  database, pull the new images, and run `up -d` so the migrator applies
  migrations; then poll container health and fail the run if anything is unhealthy.
- **Secrets:** keep the SSH key, server host/user, and any env values in the CI
  provider's encrypted secrets — never in the repo.
- **Rollback** stays the same: images are tagged by commit SHA, so you can redeploy
  a previous SHA, and the deploy job's DB backup covers schema rollbacks.
