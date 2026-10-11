# Strategio workflows

Reusable GitHub Actions workflows that build images and deploy stacks to the Strategio Docker Swarm. They run on the self-hosted runners of the organization on `ops-1`, which hold the registry login and the Docker socket of a Swarm manager. The repository is private and shared with the repositories of `strategio-digital` (Settings → Actions → General → Access); repositories of other organizations can't call it.

## Build

Builds `docker/production/Dockerfile` on every push to `main` and pushes `registry.strategio.dev/<owner>/<repo>:<commit>`.

```yaml
name: Build

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read

jobs:
  build:
    uses: strategio-digital/workflows/.github/workflows/build.yml@v1
```

Inputs: `dockerfile` (default `docker/production/Dockerfile`) and `context` (default `.`).

- Builds run in the buildx builder `limited`, a BuildKit container on ops-1 capped at 3 CPUs and 5 GB of RAM, so a build never takes the last core and the memory of the registry and the Swarm manager. The first build creates it, every runner then shares it and its cache.
- After each build, cache older than a week goes, and the cache never grows past 20 GB.

## Deploy

Deploys the stack from `docker/production/compose.yaml` at the chosen commit, with the image of the same commit. The Run workflow button in the Actions tab starts it; an empty version deploys the head of the chosen branch, and an older commit rolls back.

```yaml
name: Deploy

on:
  workflow_dispatch:
    inputs:
      version:
        description: Commit to deploy, empty for the head of the branch
        required: false

permissions:
  contents: read

jobs:
  deploy:
    uses: strategio-digital/workflows/.github/workflows/deploy.yml@v1
    with:
      version: ${{ inputs.version }}
    secrets: inherit
```

Inputs: `version`, `compose`, `stack` (default the repository name) and `role` (default `app`).

### What the compose gets

- `IMAGE`, `VERSION`, `STACK` and `ROLE` (place services with `node.labels.role == ${ROLE}`).
- Every repository variable and secret, by its name, for `${…}` interpolation. Use it for configuration only: a value interpolated into `environment` ends up in the service spec, where `docker service inspect` shows it.

### Secrets

A repository secret becomes a Docker secret through `x-environment`:

```yaml
services:
  app:
    entrypoint: ["/bin/sh", "-c", "for f in /run/secrets/*; do [ -f \"$$f\" ] && export \"$${f##*/}=$$(cat \"$$f\")\"; done; exec /app/gokick"]
    secrets:
      - source: db-password
        target: DB_PASSWORD

secrets:
  db-password:
    x-environment: DB_PASSWORD
```

- The deploy creates the secret `<stack>_<name>-<hash of the value>`, so a new value deploys as a new secret.
- The entrypoint above exports every file of `/run/secrets` as a variable of the same name and runs the original command. It works for any image with `sh`, so the app needs no `_FILE` support. Overriding the entrypoint drops the image's command, so the wrapper names it.
- Secrets shared by all projects (the backup key) are created in the infra repo with the prefix `shared-` and declared as `external: true`.
- Once a stack uses `x-environment`, the deploy fails when the value of any repository secret still appears in the rendered compose. Stacks without it get a warning.

### Backups

A Postgres service with the label `backup: pgbackrest` (under `deploy.labels`) and the image `registry.strategio.dev/strategio-digital/postgres` from the infra repo gets point-in-time backups to R2:

- the deploy sets `PGBACKREST_REPO1_PATH` to `/<stack>/<service>`, whatever the compose says, so no project writes into another project's path;
- it mounts the latest `shared-r2-backups-access-key-id-v<n>`, `shared-r2-backups-secret-access-key-v<n>` and `shared-backups-cipher-pass-v<n>` into that service only.

### What the deploy checks and changes

- The image of the commit must be in the registry.
- Refused: external volumes, external networks other than `edge`, external configs and secrets, bind mounts, `network_mode`, and names of volumes, networks, configs and secrets outside the stack prefix.
- Every network of the stack is an overlay with MTU 1400, which the private network between the servers needs. A service without networks gets the stack's `default` network.
- Configs and secrets from files get a name with the hash of their content, so a changed file deploys as a new config instead of failing.
- The run fails when a service rolled back or paused during this deploy, or runs fewer replicas than desired. Give services a healthcheck and `update_config` with `failure_action: rollback`, so a broken version never replaces the running one.
- Every service gets the label `com.strategio.repository`. A stack of the same name that belongs to another repository refuses the deploy, so two projects never overwrite each other's stack, volumes or backups.
- One deploy per repository runs at a time.
