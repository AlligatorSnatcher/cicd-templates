# cicd-templates

Reusable GitHub Actions workflows (the `uses:` building blocks) shared across
projects. Each app repo keeps its own thin *caller* workflows (triggers, hosts,
approval gates) and calls these for the actual build / backup / deploy logic.

> This repo is **public** so projects outside the org can reference it. It
> therefore contains **no** hosts, IPs, folder names, or secrets — every
> environment-specific value is a required input passed in by the caller.

## Workflows

| Workflow | What it does |
|---|---|
| [`build.yml`](.github/workflows/build.yml) | `dotnet publish` a project, upload the result as an artifact (excludes `appsettings*.json`) |
| [`backup.yml`](.github/workflows/backup.yml) | rsync `/var/<folder>/` from a server into a per-run backup dir on a host volume, prune to the newest N |
| [`deploy.yml`](.github/workflows/deploy.yml) | rsync a built artifact onto `root@<host>:/var/<folder>/` (keeps the server's `appsettings.json`) |

## Assumptions

- Jobs run on **self-hosted runners** (`runs-on: [self-hosted, <runner_label>]`).
  No `container:` — the toolchain (dotnet, rsync, ssh) is expected on the runner
  (e.g. the org's `dotnet-runner` image).
- `backup.yml` needs `BACKUP_DEST` set on the runner (a host path mounted into
  the runner container) — it is inherited from the runner env, not an input.
- SSH access to `root@<host>` is available from the runner.

## Usage

Pin to a tag (e.g. `@v1`), never `@main`:

```yaml
jobs:
  build:
    uses: AlligatorSnatcher/cicd-templates/.github/workflows/build.yml@v1
    with:
      project: src/MyApp/MyApp.csproj
      runner_label: dev
      config: Debug
      artifact: publish-dev

  backup:
    needs: build
    uses: AlligatorSnatcher/cicd-templates/.github/workflows/backup.yml@v1
    with:
      host: ${{ vars.DEV_HOST }}        # keep real hosts in the CALLER's repo vars
      folder: ${{ vars.DEV_FOLDER }}
      runner_label: dev

  deploy:
    needs: backup
    uses: AlligatorSnatcher/cicd-templates/.github/workflows/deploy.yml@v1
    with:
      host: ${{ vars.DEV_HOST }}
      folder: ${{ vars.DEV_FOLDER }}
      runner_label: dev
      artifact: publish-dev
```

> Keep hosts/folders in each caller repo's **variables/secrets**, not in YAML —
> so internal values stay in the private app repos, never in this public one.

## Inputs

### build.yml
| Input | Required | Default | Notes |
|---|---|---|---|
| `project` | ✅ | — | path to the `.csproj` to publish |
| `runner_label` | ✅ | — | extra runner label (e.g. `dev` / `prod`) |
| `config` | ➖ | `Debug` | build configuration |
| `artifact` | ➖ | `publish` | uploaded artifact name |

### backup.yml
| Input | Required | Default | Notes |
|---|---|---|---|
| `host` | ✅ | — | source server (IP / hostname) |
| `folder` | ✅ | — | app folder under `/var` |
| `runner_label` | ✅ | — | extra runner label |
| `keep` | ➖ | `5` | how many recent backups to keep |

### deploy.yml
| Input | Required | Default | Notes |
|---|---|---|---|
| `host` | ✅ | — | target server (IP / hostname) |
| `folder` | ✅ | — | app folder under `/var` |
| `runner_label` | ✅ | — | extra runner label |
| `artifact` | ➖ | `publish` | artifact name to download |
| `environment` | ➖ | `""` | GitHub environment for approval gating |

## Versioning

Tagged releases. Consumers pin `@v1` (or a SHA). Breaking changes get a new
major tag (`@v2`); the old tag keeps working so consumers migrate on their own
schedule.
