# Strategio workflows

Reusable GitHub Actions workflows that build images and deploy stacks to the Strategio Docker Swarm. They run on the self-hosted runners of the organization on `ops-1`, which hold the registry login and the Docker socket of a Swarm manager. The repository is public, so the private repositories of `strategio-digital`, `quackpocalypse` and `documan-ai` can call it. It holds no secrets: the jobs get them from the calling repository, and they run only on the runners of our organizations, which take no public repository.

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
    secrets: inherit
```

Inputs: `dockerfile` (default `docker/production/Dockerfile`), `context` (default `.`), `version`, the commit to build (default the commit of the run), and `image`, a name under the path of the repository. A workflow that commits and then builds in the same run, like a scheduled sync, passes the new commit as `version`, because the run itself still points to the commit it started on.

- **Several images from one repository:** one job per image, each with its `dockerfile` and `image`. The image `web` of `documan-ai/documan-source` is `registry.strategio.dev/documan-ai/documan-source/web:<commit>`, and the compose takes it as `${IMAGE_PATH}/web:${VERSION}`.

  ```yaml
  jobs:
    web:
      uses: strategio-digital/workflows/.github/workflows/build.yml@v1
      with:
        dockerfile: docker/web/Dockerfile.web
        image: web
      secrets: inherit
  ```

- **Build arguments and secrets come from the repository, as the Dockerfile names them.** A repository variable named by an `ARG` becomes a build argument of the same name. A repository secret named by the `id` of a `--mount=type=secret` becomes that build secret, which only the `RUN` with the mount reads and which never lands in a layer. The rest stays out of the build, and the summary of the run lists the names it passed. Secrets need `secrets: inherit` in the caller.

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

A repository in another organization (`documan-ai`, `quackpocalypse`) gets nothing from `secrets: inherit`, which GitHub passes only within one organization. It forwards its secrets as one secret instead, in Build and Deploy alike, and the workflow unpacks them and masks every value in the log:

```yaml
    secrets:
      REPO_SECRETS: ${{ toJSON(secrets) }}
```

Inputs: `version`, `compose`, `stack` (default the repository name, also under `schedule`, whose event carries no repository) and `role` (default `app`).

### What the compose gets

- `IMAGE` (`registry.strategio.dev/<owner>/<repo>:<commit>`), `IMAGE_PATH` (the same without the tag, for several images), `VERSION`, `STACK` and `ROLE` (place services with `node.labels.role == ${ROLE}`).
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
- `x-optional: true` next to `x-environment` makes the secret optional: without the repository secret, the deploy leaves it out of the stack and the variable stays unset.
- The entrypoint above exports every file of `/run/secrets` as a variable of the same name and runs the original command. It works for any image with `sh`, so the app needs no `_FILE` support. Overriding the entrypoint drops the image's command, so the wrapper names it.
- Secrets shared by all projects (the backup key) are created in the infra repo with the prefix `shared-` and declared as `external: true`.
- Networks shared by several stacks (a database of several sites) are created in the infra repo with the prefix `shared-`, like `edge`, and declared as `external: true`.
- Once a stack uses `x-environment`, the deploy fails when the value of any repository secret still appears in the rendered compose. Stacks without it get a warning.

### Backups

A Postgres service with the label `backup: pgbackrest` (under `deploy.labels`) and the image `registry.strategio.dev/strategio-digital/postgres` from the infra repo gets point-in-time backups to R2:

- the deploy sets `PGBACKREST_REPO1_PATH` to `/<stack>/<service>`, whatever the compose says, so no project writes into another project's path;
- it mounts the latest `shared-r2-backups-access-key-id-v<n>`, `shared-r2-backups-secret-access-key-v<n>` and `shared-backups-cipher-pass-v<n>` into that service only.

### What the deploy checks and changes

- Every image of the compose under `IMAGE_PATH` must be in the registry.
- Refused: external volumes, external networks other than `edge` and `shared-…`, external configs and secrets, bind mounts, `network_mode`, and names of volumes, networks, configs and secrets outside the stack prefix.
- Every network of the stack is an overlay with MTU 1400, which the private network between the servers needs. A service without networks gets the stack's `default` network.
- Configs and secrets from files get a name with the hash of their content, so a changed file deploys as a new config instead of failing.
- The run fails when a service rolled back or paused during this deploy, or runs fewer replicas than desired. It then prints the error and the last 60 lines of every task that ended during this deploy, at most 3 per service and without the tasks that ended with neither an error nor a line, and the last 20 of them in the summary of the run, with the values of repository secrets replaced by `***`. Give services a healthcheck and `update_config` with `failure_action: rollback`, so a broken version never replaces the running one.
- Every service gets the label `com.strategio.repository`. A stack of the same name that belongs to another repository refuses the deploy, so two projects never overwrite each other's stack, volumes or backups.
- One deploy per repository runs at a time.
