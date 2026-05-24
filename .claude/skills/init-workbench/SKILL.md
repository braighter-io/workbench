---
name: init-workbench
description: Bootstrap a product workbench — clone repos, install dependencies, start Docker, run migrations and seed.
allowed-tools: Read, Bash, Glob, Grep, Edit, AskUserQuestion, TodoWrite
tags: [tooling]
---

# Init Workbench

Full local environment setup for a product workbench. Run this from the workbench root
directory after cloning it.

## Prerequisites

- Git configured with access to the GitHub org
- Node.js and npm installed
- Docker Desktop running

## Process

### Step 1 — Discover repos

Read `repos.yaml` in the workbench root. It lists repos grouped by cluster:

```yaml
repos:
  services:
    - name: exercir-service
      type: fullstack
  platform:
    - name: exercir-platform
      type: infra
```

Also check for a specs repo by convention: if directory `specs/` exists but is empty,
the specs repo name is `{product}-specs` (derive product name from the workbench
directory name, e.g. `exercir-workbench` → `exercir`).

### Step 2 — Determine GitHub org and clone URL base

Read the workbench's `origin` remote:

```bash
git remote get-url origin
```

Extract the org and protocol from the URL. Examples:
- `https://github.com/braighter-io/exercir-workbench.git` → base: `https://github.com/braighter-io/`
- `git@github.com-braighter:braighter-io/exercir-workbench.git` → base: `git@github.com-braighter:braighter-io/`

Use the same protocol and host for cloning child repos.

### Step 3 — Clone repos

For each repo in `repos.yaml` plus the specs repo:

1. Check if the target directory already exists and has a `.git/` folder — **skip if yes**
2. Clone into the correct cluster directory:
   - `services/{repo-name}`
   - `platform/{repo-name}`
   - `specs/{product}-specs`

```bash
git clone {base-url}{repo-name}.git {cluster}/{repo-name}
```

Report each clone result (success, skipped, or error).

### Step 4 — Install dependencies

For each service repo with `type: fullstack` or that has a `package.json`:

```bash
cd services/{repo-name}
npm install
```

If `build:themes` script exists in package.json, also run:

```bash
npm run build:themes
```

### Step 5 — Docker infrastructure

Ask the user: "Should I start Docker infrastructure (PostgreSQL, etc.)?"

If yes:

1. Find `docker-compose.infra.yml` in the service repo
2. Verify the referenced Docker Compose file exists at the path specified in the `include` directive
3. Start infrastructure:

```bash
cd services/{service-name}
docker compose -f docker-compose.infra.yml up -d
```

4. Wait a few seconds for PostgreSQL to be ready:

```bash
docker compose -f docker-compose.infra.yml exec postgres pg_isready -U postgres
```

### Step 6 — Database setup

If Docker was started:

```bash
cd services/{service-name}
npx prisma migrate deploy
npx prisma db seed
```

### Step 7 — Verify

Run a build check to confirm everything is wired up:

```bash
cd services/{service-name}
npx nx build
```

### Step 8 — Summary

Print a summary table:

| Step | Status |
|------|--------|
| Clone repos | list each with status |
| npm install | success/failed |
| Docker infra | started/skipped |
| DB migrations | success/skipped |
| DB seed | success/skipped |
| Build themes | success/skipped |
| Build check | success/failed |

If any step failed, list the errors with guidance on how to fix them manually.

## Error handling

- Do not abort on a single failure — continue with remaining steps
- Collect all errors and report them in the summary
- If Docker is not running, skip steps 5-6 and note it in the summary
- If a repo fails to clone, skip its downstream steps (install, migrate, etc.)

## Safeguards

- Never delete or overwrite existing directories
- Never run `prisma migrate reset` — only `migrate deploy` (non-destructive)
- Ask before starting Docker services
