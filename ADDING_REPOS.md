# Adding a new repo to the Orchestrator

Step-by-step workflow to onboard a new repo under [repos/](repos/) so it works both standalone (`docker compose up` from inside the repo) and orchestrated (`npm run docker:<alias>` from the root). Examples use generic placeholders (`AP.REPO-1`, `AP.REPO-2`, ...) — substitute your actual repo names.

**Naming convention in this doc**:
- `<Repo>` = name of the folder under `repos/` (e.g. `AP.REPO-1`).
- `<alias>` = short label used in profiles and npm scripts (e.g. `web`, `worker`, `auth`).

---

## Choosing an alias for the repo

The alias is the **same short string** appearing in 3 places. Choose it once and use it identically in all three to avoid ambiguity.

Example: repo `my-repo` under `repos/my-repo/`, alias **`aliasDemo`**.

**1. Repo standalone compose** — `repos/my-repo/docker-compose.yml`:

```yaml
name: my-repo

services:
  aliasDemo:                        # ← alias goes here, as the service key
    build:
      context: .
      dockerfile: Dockerfile
    image: my-repo:dev              # (optional) image tag
    container_name: my-repo-aliasDemo # (optional) visible container label
    ports:
      - "${MYREPO_PORT:-3000}:3000"
    volumes:
      - .:/src
    restart: unless-stopped
```

**2. Orchestrator base compose** — `docker-compose.yml` at the root:

```yaml
services:
  aliasDemo:                        # ← same alias
    build:
      context: ./repos/my-repo
      dockerfile: Dockerfile
    profiles: ["all", "aliasDemo"]  # ← alias in the profile
    ports:
      - "${MYREPO_PORT:-3000}:3000"
    volumes:
      - ./repos/my-repo:/src
    networks:
      - rif
```

**3. Orchestrator npm scripts** — `package.json`:

```json
"docker:aliasDemo": "docker compose --profile aliasDemo up --build"
```

### Why does it matter that they match in all 3?

- **Standalone compose (`services.aliasDemo`)**: internal container DNS and log prefix. If another container calls `http://aliasDemo:3000`, docker resolves it by that name.
- **Orchestrator compose (`services.aliasDemo` + `profiles: ["aliasDemo"]`)**: same internal DNS + enables `docker compose --profile aliasDemo up` to bring up only that service.
- **`package.json` (`docker:aliasDemo`)**: npm wrapper that translates to `--profile aliasDemo`. By convention it matches the other two.

### On alias / service key divergence

You can technically use different strings for the service key (in both composes) and the profile alias (in `package.json`). It works, but it makes things less obvious when debugging or when someone reads the compose for the first time. **Recommendation for a new repo**: use the same string in all 3 places unless you have a concrete reason to split them.

---

## 1) Prepare the repo under `repos/<Repo>/`

These files live **inside the repo** so it also works standalone.

### 1.1 `Dockerfile` (dev image)

Place it where your build context makes sense (root of the `.sln`, of `package.json`, `pyproject.toml`, etc). Typical structures per stack:

- **.NET solution**: Dockerfile at the solution folder (`repos/<Repo>/<Repo>/Dockerfile`) — SDK image, cached `dotnet restore`, `dotnet run` as CMD.
- **Node / Vite**: Dockerfile at the repo root (`repos/<Repo>/Dockerfile`) — `node:22-alpine` (or your target), `npm ci` with cache mount, `npm run dev -- --host 0.0.0.0` as CMD.
- **Python**: Dockerfile at the repo root — slim base, `pip install` in a layer separate from source, `uvicorn` / `gunicorn` / etc. as CMD binding to `0.0.0.0`.

Points to respect regardless of stack:
- CMD binds to `0.0.0.0` (or equivalent), not `localhost`. Without this the port published by the container is not reachable from the host.
- Cache the restore/install step in a layer separate from the source code (COPY only the manifests → restore → COPY the rest). Even though the source is overridden by the volume mount at runtime, the layer serves as a fallback and speeds up the first build.

### 1.2 `.dockerignore`

At minimum: build artifacts (`bin/`, `obj/`, `node_modules/`, `target/`, `__pycache__/`, `dist/`, ...), IDE files (`.vs/`, `.vscode/`, `.idea/`), `.git`, and the `Dockerfile` / `docker-compose*.yml` themselves to keep them out of the image context.

### 1.3 `docker-compose.yml` standalone

At the repo root. Template:

```yaml
name: ap-<repo>

services:
  <alias>:
    build:
      context: ./<subpath-to-Dockerfile-or-.>
      dockerfile: Dockerfile
    image: ap-<repo>:dev
    container_name: ap-<repo>
    ports:
      - "${<REPO>_PORT:-<default>}:<internal-port>"
    environment:
      # Docker-specific values only (do NOT duplicate config already in a repo file)
      <ENV_VAR>: <value>
    volumes:
      - ./<source-folder>:/src
      # Anonymous volumes over build artifacts (prevents clobbering the host):
      - /src/<sub>/bin
      - /src/<sub>/obj
      # or /app/node_modules, /src/target, /src/__pycache__, etc.
      - <named-volume>:/<cache-path>
    extra_hosts:
      - "host.docker.internal:host-gateway"   # required on Linux; no-op on Docker Desktop
    restart: unless-stopped

volumes:
  <named-volume>:
```

### 1.4 `package.json` (optional but recommended)

To have a uniform `npm run docker` inside the repo. Minimal `package.json` — 3 scripts and no dependencies:

```json
{
  "name": "ap-<repo>-docker",
  "private": true,
  "scripts": {
    "docker": "docker compose up --build",
    "docker:down": "docker compose down",
    "docker:logs": "docker compose logs -f"
  }
}
```

If the stack already ships a `package.json` (Node), just add the `docker*` scripts to the existing block.

### 1.5 Source patches (only if unavoidable)

If the repo has hardcoded URLs pointing to `localhost` to consume other services, allow an env var override. Without this the container has no way to reach another container on the docker network (where the other service is addressed by its service name, not `localhost`).

**Example scenario**: a Vite dev server has its proxy target hardcoded to `https://localhost:7094` — not reachable from inside a container. The fix is to add support for a `VITE_PROXY_TARGET` env var so the Orchestrator's compose can override the target to `http://<other-service>:<port>` when both services run on the same docker network.

**Rule of thumb**: keep the patch minimal and additive (env var with fallback to the original hardcoded value). Don't rewrite the repo's proxy logic.

If the repo does NOT have this problem (consumes configurable URLs from a repo file), no patch is needed.

---

## 2) Register in the Orchestrator

All of this is done at the Orchestrator root — it does not touch repo files.

### 2.1 [docker-compose.yml](docker-compose.yml) (base)

Add the service replicating what's in the standalone compose, but with:
- `profiles: ["all", "<alias>"]` — the base compose only brings up the service if the profile is active.
- `build.context` pointing at `./repos/<Repo>/<subpath>`.
- `volumes:` with the same paths but prefixed with `./repos/<Repo>/...`.
- `networks: [rif]` so it lives on the same network as the other services.

Example (extrapolate from the standalone template above):

```yaml
services:
  <alias>:
    build:
      context: ./repos/<Repo>/<subpath>
      dockerfile: Dockerfile
    image: ap-<repo>:dev
    container_name: rif-<alias>
    profiles: ["all", "<alias>"]
    ports:
      - "${<REPO>_PORT:-<default>}:<internal-port>"
    environment:
      # same as the standalone
    volumes:
      - ./repos/<Repo>/<source-subpath>:/src
      # ... anonymous volumes identical to the standalone
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      - rif
    restart: unless-stopped
```

### 2.2 Overrides `docker-compose.{all,local,remote}.yml` (if applicable)

Only if the service has variants (e.g. a repo with `dev` / `dev:local` / `dev:remote` variants pointing at different upstream targets). Each override adds only what changes on top of the base (e.g. a different `command:`, or different env vars). Everything else is inherited from the base.

### 2.3 Ports

The service's `ports:` mapping uses the `${<REPO>_PORT:-<default>}` fallback. So:

- If the default doesn't collide on the host, nothing to do.
- If it collides, each dev can override with a one-off shell env var:
  ```powershell
  $env:<REPO>_PORT=<other>; npm run docker
  ```

There's no `.env` at the Orchestrator level — each repo's runtime config lives inside the repo and is read via the source volume mount.

### 2.4 [package.json](package.json) (Orchestrator)

Add scripts:

```json
"docker:<alias>": "docker compose --profile <alias> up --build"
```

If the service participates in the `npm run docker` full-stack (profile `all`), you don't need to add anything else — it's included by the profile. If it also has its own local/remote variants, add the equivalent scripts.

Also update `docker:down` to include the new profile, listing every registered alias:

```json
"docker:down": "docker compose --profile all --profile <alias-1> --profile <alias-2> --profile <alias> down"
```

---

## 3) Smoke test

In this order — if something fails you isolate layer by layer:

```powershell
# 3.1 Standalone of the new repo
cd repos\<Repo>
npm run docker
# → verify startup log, service health, main URL

# 3.2 Only the new repo from the Orchestrator
cd ..\..
npm run docker:<alias>
# → same behavior as 3.1 but running from the root

# 3.3 Full stack
npm run docker
# → all registered repos + <alias> running on the same rif network
# → test interconnection: curl from one container to another by service name

# 3.4 Tear everything down
npm run docker:down
```

Mental checklist:
- [ ] The published port is reachable at `http://localhost:<REPO_PORT>` on the host.
- [ ] The service reaches other services by name (e.g. `http://<other-alias>:<other-port>`, not `http://localhost:<other-port>`).
- [ ] Code changes on the host reflect inside the container (bind mount OK).
- [ ] The container's build artifacts (`bin/`/`obj/`, `node_modules`, `target/`, etc.) don't clobber the host's (anonymous volumes OK).

---

## 4) Networking between services — cheatsheet

Inside the Orchestrator's `rif` network, each service is reachable by its **service name** (not `container_name`, not `localhost`).

| From | To reach | URL |
|---|---|---|
| container `repo-1` | container `repo-2` | `http://repo-2:<internal-port>` |
| container `repo-1` | host DB / broker / other service | `host.docker.internal:<port>` |
| host (browser, curl) | container `repo-1` | `http://localhost:${REPO1_PORT}` |
| host (browser, curl) | container `repo-2` | `http://localhost:${REPO2_PORT}` |

If a repo hardcodes `localhost:<x>` to consume another service, it won't work in containerized mode — introduce an env var override (see 1.5).

---

## Minimal summary (TL;DR)

1. Under `repos/<Repo>/`: Dockerfile + .dockerignore + standalone `docker-compose.yml` + (optional) `package.json` with `docker*` scripts.
2. In the Orchestrator: add the service to the base [docker-compose.yml](docker-compose.yml) with `profiles: ["all", "<alias>"]`, add a script to [package.json](package.json).
3. Smoke test: standalone → orchestrated individually → full stack.
4. If the repo hardcodes `localhost` to consume other services, add an env var override (minimal additive patch).
