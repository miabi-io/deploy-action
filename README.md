<p align="center">
  <img src="assets/miabi-icon.svg" alt="Miabi" width="96" height="96">
</p>

<h1 align="center">Deploy to Miabi — GitHub Action</h1>

Deploy an application to a [Miabi](https://github.com/miabi-io/miabi) control
panel from GitHub Actions. It installs the [`miabi`
CLI](https://github.com/miabi-io/miabi-cli) and runs `miabi apps deploy` against
your panel's public `/api/v1` API — no SSH, no Docker socket, no server access.

The typical flow: build and push your image, then point Miabi at the new tag and
wait for the rollout to finish (the job fails if the deploy fails).

## Quick start

1. Create a **workspace-bound API token** in your Miabi panel (Settings → API
   tokens) with deploy permission for the app.
2. Add two secrets/vars to your repository:
   - `MIABI_URL` (repository **variable**) — your panel URL, e.g. `https://miabi.example.com`
   - `MIABI_TOKEN` (repository **secret**) — the API token
3. Add the action to a workflow:

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # ... build & push your image tagged with ${{ github.sha }} first ...
      - name: Deploy to Miabi
        uses: miabi-io/deploy-action@v1
        with:
          app: web
          tag: ${{ github.sha }}
          url: ${{ vars.MIABI_URL }}
          token: ${{ secrets.MIABI_TOKEN }}
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app` | no | — | Application to deploy (its name/handle). Omit to deploy the app a workspace-bound token points at. |
| `tag` | no | `${{ github.sha }}` | Image tag to deploy. |
| `strategy` | no | — | Deploy strategy: `recreate` \| `rolling` \| `canary`. Omit to use the app's configured default. |
| `url` | no | — | Panel URL. Falls back to the `MIABI_URL` env var. |
| `token` | no | — | API token. Falls back to the `MIABI_TOKEN` env var. |
| `workspace` | no | — | Workspace name or id. Only needed when the token is not bound to one workspace. |
| `wait` | no | `true` | Wait for the deployment and fail the job if it fails. |
| `timeout` | no | `10m` | Max wait when `wait` is true (Go duration, e.g. `10m`, `300s`). |
| `version` | no | `latest` | miabi CLI version to install (e.g. `1.4.0`) or `latest`. |

`url`/`token` can also be provided via the `MIABI_URL` / `MIABI_TOKEN`
environment variables (job- or step-level `env:`), which is handy when several
steps share them:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      MIABI_URL: ${{ vars.MIABI_URL }}
      MIABI_TOKEN: ${{ secrets.MIABI_TOKEN }}
    steps:
      - uses: miabi-io/deploy-action@v1
        with:
          app: web
```

## Examples

### Build, push, then deploy

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Deploy to Miabi
        uses: miabi-io/deploy-action@v1
        with:
          app: web
          tag: ${{ github.sha }}
          url: ${{ vars.MIABI_URL }}
          token: ${{ secrets.MIABI_TOKEN }}
          timeout: 15m
```

### Fire-and-forget (don't block on the rollout)

```yaml
      - uses: miabi-io/deploy-action@v1
        with:
          app: web
          wait: 'false'
          url: ${{ vars.MIABI_URL }}
          token: ${{ secrets.MIABI_TOKEN }}
```

### Deploy on a release tag

```yaml
on:
  release:
    types: [published]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: miabi-io/deploy-action@v1
        with:
          app: web
          tag: ${{ github.event.release.tag_name }}
          url: ${{ vars.MIABI_URL }}
          token: ${{ secrets.MIABI_TOKEN }}
```

A ready-to-copy workflow lives in [`examples/deploy.yml`](examples/deploy.yml).

## Notes

- **Runners:** Linux and macOS (`ubuntu-*`, `macos-*`). Windows runners are not
  supported — run this step on a Linux runner.
- **Security:** always pass the token via `secrets`, never inline. The CLI never
  logs the token; this action passes it to the CLI via the environment only.
- **What it runs:** effectively `miabi apps deploy <app> --tag <tag> --wait`. See
  the [CLI docs](https://github.com/miabi-io/miabi-cli) for the full command set
  (`apply`, `rollback`, `env`, …) if you need more than a deploy.

## License

[Apache-2.0](LICENSE).
