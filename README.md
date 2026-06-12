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
| [`test.yml`](.github/workflows/test.yml) | `dotnet test` a test project or solution |
| [`backup.yml`](.github/workflows/backup.yml) | rsync `/var/<folder>/` from a server into a per-run backup dir on a host volume, prune to the newest N |
| [`deploy.yml`](.github/workflows/deploy.yml) | rsync a built artifact onto `root@<host>:/var/<folder>/` (keeps the server's `appsettings.json`). **No gate** — for non-gated envs like dev |
| [`deploy-approve.yml`](.github/workflows/deploy-approve.yml) | Same as `deploy.yml`, but **actor-gated** (`allowed_actors` + optional `allowed_ref`) — for protected envs like prod |

## Assumptions

- Jobs run on **self-hosted runners** (`runs-on: [self-hosted, <runner_label>]`).
  No `container:` — the toolchain (dotnet, rsync, ssh) is expected on the runner
  (e.g. the org's `dotnet-runner` image).
- `backup.yml` needs `BACKUP_DEST` set on the runner (a host path mounted into
  the runner container) — it is inherited from the runner env, not an input.
- SSH access to `root@<host>` is available from the runner.

## GitHub settings (one-time)

Two settings let other repos use these workflows. Set them once — new repos in
the org inherit them, no per-project setup.

### Org-level — Actions permissions
Org → Settings → Actions → General → Policies. Use a scoped allow-list rather
than "Allow all actions" (important: self-hosted runners mount SSH keys and the
Docker socket, so a malicious third-party action could reach internal hosts):

> **Allow `<org>`, and select non-`<org>` actions and reusable workflows**
> - ☑ Allow actions created by GitHub  → for `actions/checkout`, `upload-/download-artifact`
> - ☑ Allow actions by Marketplace verified creators  → for `docker/*` (e.g. ci-images)

This governs which actions/reusable workflows a **consumer** repo may call.

### This repo's visibility — access for consumers
- **Public** (current): no setup — any repo can `uses:` these workflows.
- **Private**: Settings → Actions → General → **Access** →
  *"Accessible from repositories in the `<org>` organization"*. Only same-org
  repos can then use them; repos outside the org cannot (use Public for that).

> Org policy = the consumer is *allowed to call*. Repo Access = this repo is
> *shared out* (private only). Private needs both; public needs only the org policy.

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

### test.yml
| Input | Required | Default | Notes |
|---|---|---|---|
| `project` | ✅ | — | test `.csproj` or `.sln` to run |
| `runner_label` | ✅ | — | extra runner label (e.g. `dev` / `prod`) |
| `config` | ➖ | `Debug` | build configuration |

> The runner must have the SDK for the test project's target framework
> (`dotnet test` builds it). E.g. a `netcoreapp3.1` test project needs the 3.1
> SDK on the runner.

### backup.yml
| Input | Required | Default | Notes |
|---|---|---|---|
| `host` | ✅ | — | source server (IP / hostname) |
| `folder` | ✅ | — | app folder under `/var` |
| `runner_label` | ✅ | — | extra runner label |
| `keep` | ➖ | `5` | how many recent backups to keep |

### deploy.yml (no gate)
| Input | Required | Default | Notes |
|---|---|---|---|
| `host` | ✅ | — | target server (IP / hostname) |
| `folder` | ✅ | — | app folder under `/var` |
| `runner_label` | ✅ | — | extra runner label |
| `artifact` | ➖ | `publish` | artifact name to download |

### deploy-approve.yml (actor-gated)
| Input | Required | Default | Notes |
|---|---|---|---|
| `host` | ✅ | — | target server (IP / hostname) |
| `folder` | ✅ | — | app folder under `/var` |
| `runner_label` | ✅ | — | extra runner label |
| `allowed_actors` | ✅ | — | space-separated GitHub logins allowed to deploy. **Works on any plan** |
| `allowed_ref` | ➖ | `""` | restrict to a ref, e.g. `refs/heads/release` |
| `artifact` | ➖ | `publish` | artifact name to download |

> **Soft control**: a user with write access could edit the workflow to bypass
> the actor gate. GitHub environment Required reviewers would be stronger but
> need a paid plan on private repos.
>
> **Gate timing**: the gate runs at the start of this deploy job — if called
> after build/backup, those run first. To block unauthorized runs before build,
> put an actor-gate job first in the caller instead.

## Changing a workflow

Decide first *what* you're changing:

- **Shared logic** (the build/backup/deploy steps themselves) → edit it **here**.
- **Project-specific** (triggers, hosts, folders, which project to build,
  approval gates) → edit the **caller** in the app repo, not this repo.

Consumers pin `@v1`, which points at a specific commit — **editing `main` alone
does nothing for them**. To roll a change out you must move (or cut) the tag:

```bash
# non-breaking change (add a step, fix a bug): move v1 forward
git add -A && git commit -m "fix: ..."
git push origin main
git tag -f v1 && git push -f origin v1     # <-- the step people forget

# then in the consumer repo: just re-run the job — it now picks up the new v1.
# No caller edit needed (the @v1 ref is unchanged; only what it points to moved).
```

> If you skip the `git tag -f v1` step, consumers keep running the OLD v1 even
> after re-running their jobs.

## Versioning

`v1` is a **rolling major tag** (like `actions/checkout@v4`): non-breaking
changes move it forward, so `@v1` consumers get them on the next run with no
caller edit.

| Change | Action here | Consumer |
|---|---|---|
| Non-breaking (add step, bug fix) | `git tag -f v1 && git push -f origin v1` | re-run job, picks it up |
| Breaking (rename/required input) | cut a new tag `v2` | bump `@v1` → `@v2` per repo, on their own schedule |

Breaking changes get a new major tag so the old one keeps working — consumers
migrate when they choose. Pin a full SHA instead of `@v1` if you need a frozen,
never-moving reference.
