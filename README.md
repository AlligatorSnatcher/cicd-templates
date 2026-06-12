# cicd-templates

跨專案共用的 GitHub Actions reusable workflows(`uses:` 積木)。各 app repo 只留薄薄的
*caller* workflow(觸發條件、host、gate),實際的 build / backup / deploy 邏輯都呼叫這裡。

> 這個 repo 是 **public**,讓 org 外的專案也能引用。因此**不含**任何 host、IP、folder 名稱或
> secret —— 所有環境專屬的值都由 caller 以必填 input 傳入。

## Workflows

| Workflow | 做什麼 |
|---|---|
| [`build.yml`](.github/workflows/build.yml) | `dotnet publish` 一個專案,上傳成 artifact(排除 `appsettings*.json`) |
| [`test.yml`](.github/workflows/test.yml) | `dotnet test` 一個測試專案或 solution |
| [`backup.yml`](.github/workflows/backup.yml) | 從伺服器 rsync `/var/<folder>/` 到 host volume 的每次獨立備份目錄,只保留最新 N 份 |
| [`deploy.yml`](.github/workflows/deploy.yml) | rsync artifact 到 `root@<host>:/var/<folder>/`(保留伺服器的 `appsettings.json`)。**無 gate** —— 給 dev 這種不需核准的 |
| [`deploy-approve.yml`](.github/workflows/deploy-approve.yml) | 同 `deploy.yml`,但有 **actor gate**(`allowed_actors` + 選填 `allowed_ref`)—— 給 prod 這種受保護的 |

## 前提假設

- job 跑在 **self-hosted runner**(`runs-on: [self-hosted, <runner_label>]`)。不用 `container:` ——
  工具鏈(dotnet、rsync、ssh)預期已在 runner 上(例如 org 的 `dotnet-runner` image)。
- `backup.yml` 需要 runner 上有 `BACKUP_DEST`(掛進容器的 host 路徑)—— 由 runner 的環境變數繼承,不是 input。
- runner 能 SSH 到 `root@<host>`。

## GitHub 設定(一次性)

兩個設定讓其他 repo 能用這些 workflow。**設一次,org 內新 repo 自動繼承,不用每個專案重設。**

### Org 層級 —— Actions 權限
Org → Settings → Actions → General → Policies。用範圍化的 allow-list,**不要**用「Allow all actions」
(重要:self-hosted runner 掛了 SSH key 與 Docker socket,惡意第三方 action 可藉此打到內網):

> **Allow `<org>`, and select non-`<org>` actions and reusable workflows**
> - ☑ Allow actions created by GitHub → 給 `actions/checkout`、`upload-/download-artifact`
> - ☑ Allow actions by Marketplace verified creators → 給 `docker/*`(例如 ci-images)

這管的是「**consumer repo 被允許呼叫哪些** action / reusable workflow」。

### 這個 repo 的 visibility —— 對 consumer 的開放
- **Public**(目前):不用設 —— 任何 repo 都能 `uses:`。
- **Private**:Settings → Actions → General → **Access** →
  *「Accessible from repositories in the `<org>` organization」*。只有同 org 的 repo 能用;
  org 外的 repo 不行(要跨 org 就維持 Public)。

> Org policy = consumer「**有權限呼叫**」;Repo Access = 這個 repo「**對外分享**」(只 private 需要)。
> private 兩個都要,public 只要 org policy。

## 用法

Pin 到 tag(例如 `@v1`),**不要**用 `@main`:

```yaml
jobs:
  build:
    uses: COSCOMMS/cicd-templates/.github/workflows/build.yml@v1
    with:
      project: src/MyApp/MyApp.csproj
      runner_label: dev
      config: Debug
      artifact: publish-dev

  backup:
    needs: build
    uses: COSCOMMS/cicd-templates/.github/workflows/backup.yml@v1
    with:
      host: ${{ vars.DEV_HOST }}        # 真實 host 放在 CALLER 的 repo variables
      folder: ${{ vars.DEV_FOLDER }}
      runner_label: dev

  deploy:
    needs: backup
    uses: COSCOMMS/cicd-templates/.github/workflows/deploy.yml@v1
    with:
      host: ${{ vars.DEV_HOST }}
      folder: ${{ vars.DEV_FOLDER }}
      runner_label: dev
      artifact: publish-dev
```

> host / folder 放在各 caller repo 的 **variables / secrets**,不要寫進 YAML ——
> 內網值就只留在 private 的 app repo,不會進到這個 public repo。

## Inputs

### build.yml
| Input | 必填 | 預設 | 說明 |
|---|---|---|---|
| `project` | ✅ | — | 要 publish 的 `.csproj` 路徑 |
| `runner_label` | ✅ | — | runner label(例如 `dev` / `prod`) |
| `config` | ➖ | `Debug` | build 設定 |
| `artifact` | ➖ | `publish` | 上傳的 artifact 名稱 |

### test.yml
| Input | 必填 | 預設 | 說明 |
|---|---|---|---|
| `project` | ✅ | — | 要跑的測試 `.csproj` 或 `.sln` |
| `runner_label` | ✅ | — | runner label(例如 `dev` / `prod`) |
| `config` | ➖ | `Debug` | build 設定 |

> runner 必須有測試專案對應 framework 的 SDK(`dotnet test` 會 build 它)。
> 例如 `netcoreapp3.1` 的測試專案,runner 就要裝 3.1 SDK。

### backup.yml
| Input | 必填 | 預設 | 說明 |
|---|---|---|---|
| `host` | ✅ | — | 來源伺服器(IP / hostname) |
| `folder` | ✅ | — | `/var` 底下的 app 資料夾 |
| `runner_label` | ✅ | — | runner label |
| `keep` | ➖ | `5` | 保留最新幾份備份 |

### deploy.yml(無 gate)
| Input | 必填 | 預設 | 說明 |
|---|---|---|---|
| `host` | ✅ | — | 目標伺服器(IP / hostname) |
| `folder` | ✅ | — | `/var` 底下的 app 資料夾 |
| `runner_label` | ✅ | — | runner label |
| `artifact` | ➖ | `publish` | 要下載的 artifact 名稱 |

### deploy-approve.yml(actor gate)
| Input | 必填 | 預設 | 說明 |
|---|---|---|---|
| `host` | ✅ | — | 目標伺服器(IP / hostname) |
| `folder` | ✅ | — | `/var` 底下的 app 資料夾 |
| `runner_label` | ✅ | — | runner label |
| `allowed_actors` | ✅ | — | 允許部署的 GitHub 帳號(空白分隔)。**任何方案都可用** |
| `allowed_ref` | ➖ | `""` | 限定分支,例如 `refs/heads/release` |
| `artifact` | ➖ | `publish` | 要下載的 artifact 名稱 |

> **軟控制**:有 write 權限的人能改 workflow 繞過 actor gate。GitHub environment Required
> reviewers 更強,但 private repo 需付費方案。
>
> **Gate 時機**:gate 在 deploy job 開始時才檢查 —— 若在 build/backup 之後才呼叫,那兩個會先跑。
> 要讓未授權者連 build 都不准,就在 caller 放一個 authorize job 跑在最前面。

## 改流程怎麼做

先分清楚你改的是什麼:

- **共用邏輯**(build/backup/deploy 的步驟本身)→ 改**這裡**。
- **專案專屬**(觸發條件、host、folder、要 build 哪個專案、approval gate)→ 改 **app repo 的 caller**,不是這裡。

consumer pin 的是 `@v1`,它指向某個固定 commit —— **只改 `main` 對他們沒有任何作用**。要讓改動生效,必須移動(或新開)tag:

```bash
# 非破壞性改動(加 step、修 bug):把 v1 往前移
git add -A && git commit -m "fix: ..."
git push origin main
git tag -f v1 && git push -f origin v1     # <-- 大家最常忘的一步

# 然後在 consumer repo:重跑 job 就會吃到新的 v1。
# 不用改 caller(@v1 ref 沒變,只是它指向的 commit 變了)。
```

> 漏了 `git tag -f v1` 這步,consumer 重跑 job 還是吃舊的 v1。

## 版本策略

`v1` 是**滾動的大版本 tag**(像 `actions/checkout@v4`):非破壞性改動把它往前移,`@v1` 的 consumer 下次跑就拿到,不用改 caller。

| 改動 | 這裡要做 | consumer |
|---|---|---|
| 非破壞性(加 step、修 bug) | `git tag -f v1 && git push -f origin v1` | 重跑 job 就吃到 |
| 破壞性(改 input 名 / 必填) | 開新 tag `v2` | 各自 `@v1` → `@v2`,自行排程遷移 |

破壞性改動開新大版本,舊 tag 繼續可用,consumer 自己選時間遷移。需要永不變動的固定引用,就 pin 完整 SHA 而非 `@v1`。
