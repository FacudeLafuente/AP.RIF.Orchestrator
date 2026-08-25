# AP.RIF.Orchestrator

Dev orchestrator to work with **separate dockerized repos as if they were a monorepo**, without actually being one.

## The problem it solves

When your product lives in N separate repos you get the upsides of multi-repo (independent git, per-repo CI/CD, isolated deploys, clear ownership) but also the day-to-day pain:

- Bringing up the full stack in dev = cloning N repos, following N READMEs, starting N services by hand.
- Agent / AI instructions (`AGENTS.md`, `.github/copilot-instructions.md`, skills, prompts, etc.) live replicated — or divergent — in each repo. There's no shared context for features that cross repos.
- Cross-repo changes require jumping between folders and coordinating mentally.

A monorepo solves the orchestration but brings other costs (tooling, permissions, blast radius, blurry ownership). The Orchestrator gives you the middle ground.

## What you get

Dropping your repos under `repos/` (gitignored, each keeps its own git) and registering each one with an alias unlocks:

- **One command spins up whatever you want**: `npm run docker` for the full stack, or `npm run docker:<alias>` for a single repo. Each repo still runs standalone if you focus on just one.
- **Config and secrets are not duplicated**: each repo keeps reading its own files (`.env`, `appsettings.*.json`, config templates, etc.), mounted as a volume in the container.
- **Single agentic context**: agent instructions and skills (Copilot, Cursor, Claude, etc.) placed at the Orchestrator level apply to every repo below. Add a new repo and it inherits the context — no need to reconfigure agents per repo every time.
- **Zero lock-in**: every repo remains a separate repo. Leaving the Orchestrator = removing its entry from the compose. No friction, no history migration.

The best of both worlds between monorepo and multi-repo, without adopting either one fully.

> **New to the Orchestrator?**
>
> → [GET STARTED!](GET_STARTED.md) with a guided setup and a visual diagram of the model.
>
> → For the technical detail of adding a new repo: [How to add new repos?](ADDING_REPOS.md).

## Model

- Each repo under `repos/` is registered in the Orchestrator with an **alias** (`web`, `worker`, `auth`, `payments`, ...) and participates in docker compose profiles.
- `npm run docker` brings up every repo with profile `all`. `npm run docker:<alias>` brings up just one.
- Every repo also ships its own `docker-compose.yml` to run standalone (`cd repos/<Repo>; docker compose up --build`), independent of the Orchestrator.
- The Orchestrator **does not duplicate secrets or app config**. Each repo keeps reading its own files (`.env`, `appsettings.*.json`, config templates, etc.), mounted as a volume in the container.

## Layout

```
AP.RIF.Orchestrator/
├── docker-compose.yml           # base: services for each repo with profiles
├── docker-compose.<variant>.yml # optional overrides (e.g. local/remote variants)
├── package.json                 # npm scripts: docker, docker:<alias>, docker:down, ...
├── ADDING_REPOS.md              # detailed guide to onboard a new repo
└── repos/                       # gitignored — each subfolder is a repo with its own git
    ├── <Repo-1>/
    │   ├── Dockerfile           # (or under a subpath depending on the stack)
    │   ├── .dockerignore
    │   └── docker-compose.yml   # standalone
    └── <Repo-2>/
        └── ...
```

## Prerequisites

- Docker Desktop (Windows/Mac) or Docker + Compose v2 (Linux).
- Node.js — optional; only if you want to use the `npm run docker*` scripts. Alternative: call `docker compose` directly.
- Any external infra your repos consume in dev (DB, brokers, cache, etc.) must be running on your host or reachable over the network. See **Config and networking** below.

## Usage from the root (Orchestrator)

```powershell
npm run docker            # brings up every repo with profile "all"
npm run docker:<alias>    # brings up only that repo
npm run docker:down       # tears everything down
npm run docker:logs       # follows logs
```

Each repo's published port has a fallback in `docker-compose.yml` (`${<REPO>_PORT:-<default>}`). If one collides on your host:

```powershell
$env:<REPO>_PORT=<other>; npm run docker
```

**Scripts currently registered in this workspace** → see [package.json](package.json). Each `docker*` entry documents what it brings up and with which options.

## Standalone usage (from inside a repo)

When you're working on a single repo, no need to go through the Orchestrator:

```powershell
cd repos\<Repo>
docker compose up --build
```

If the repo has its own `package.json` with `docker*` scripts:

```powershell
cd repos\<Repo>
npm run docker
```

## Config and networking

- **Runtime config**: the compose mounts each repo's source as a volume. Config files the repo already uses locally are read from the bind mount — nothing is duplicated at the Orchestrator level.
- **Consuming host services** (local DB, broker, etc.): inside the container, use `host.docker.internal` instead of `localhost`. Docker Desktop maps it on Windows/Mac; the compose adds `extra_hosts: host-gateway` so it resolves on Linux too.
  - The host service **must accept TCP connections on the interface reachable from the docker network** (not just `127.0.0.1`), and the host firewall must allow the port.
  - **SQL Server on Windows**: by default it listens on Shared Memory only. Enable the **TCP/IP** protocol in *SQL Server Configuration Manager → SQL Server Network Configuration → Protocols for MSSQLSERVER*, restart the `MSSQLSERVER` service, and add an inbound rule for port `1433` in Windows Firewall. Without this, the container gets `connection refused` even when the hostname resolves correctly.
  - If your connection string uses your machine's NetBIOS name (e.g. `Server=YOURPCNAME`) instead of `host.docker.internal`, map it in `extra_hosts`:
    ```yaml
    extra_hosts:
      - "YOURPCNAME:host-gateway"
    ```
- **Consuming another container** (repo A → repo B): use the service name (`http://<service>:<port>`), NOT `localhost`. If your repo hardcodes `localhost` to reach an internal service, an additive patch is needed (env var override) — see [ADDING_REPOS.md](ADDING_REPOS.md#15-source-patches-only-if-unavoidable).
- **Build artifacts** (`bin/obj`, `node_modules`, `target/`, `__pycache__/`, etc.): the compose isolates them with anonymous/named volumes so container binaries don't clobber your host working copy.
- **Hot reload**: depends on each repo's Dockerfile CMD. If the CMD uses a native watcher (`vite`, `dotnet watch`, `nodemon`, `air`, etc.), changes are picked up without restart. Otherwise, restart manually: `docker compose restart <alias>`.
