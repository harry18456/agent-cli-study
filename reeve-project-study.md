# Reeve 專案研究

日期：2026-06-02
目標路徑：`/home/asus/Reeve`
參考邏輯：`/home/asus/agent-cli-study/codebus-windows-cli-study.md`

## 結論

Reeve 不是一般 bot，也不是單純把 Codex 對著 PR diff 跑。它是一個 Azure DevOps 內部 advisory code reviewer：

1. `reeve serve` 收 Azure DevOps webhook 或 `/reeve review` 留言。
2. Reeve 用 Service Principal 讀 PR、clone / worktree 出受審版本、產生 diff context。
3. Reeve 準備 agent-facing ephemeral workspace，放入 `.reeve/context/diff.json`、rules、`reeve-review` skill。
4. Codex agent 只能把 findings 寫到 `.reeve/queue/actions.jsonl`。
5. Reeve server 端再 dequeue、重驗 finding schema、重算 dedup key、scrub、過 gate chain。
6. 真正 Azure DevOps 寫回只由 Reeve 發 REST：inline threads、summary、advisory status、reviewer vote。

嚴格判斷：

| 面向 | 現況 | 判斷 |
|---|---|---|
| 功能 MVP | `go test ./...`、`go build ./...`、`go vet ./...` 通過；README 宣稱 `0.1.0 MVP COMPLETE` | 功能面已接近 MVP，可做 demo / controlled pilot 前置 |
| 寫回權限 | Azure DevOps client 只實作 thread/comment/status/vote，沒有 merge / policy mutation / delete ref API | advisory-only 架構合理，是目前最強的一層 |
| agent containment | Azure key proxy、env denylist、trusted rules、Linux egress probe、bwrap read isolation primitive 都存在 | 方向正確，但 production gate 尚未完成 |
| 上線安全 | webhook 無認證；Codex security control plane active change `0/54 tasks`；live proofs 預設皆 skip | 不應宣稱可直接接公司內部真 repo 廣泛上線 |
| 規格可信度 | `openspec validate --all` 通過 | 工具綠燈不足；主 spec 仍殘留 C0 `not implemented` 舊需求 |
| 併發品質 | 一般測試通過 | `go test -race ./...` 失敗，queue 也缺 cross-process flock |

企業結論：Reeve 的產品方向是對的，尤其是「agent 輸出不可信、server 端重驗後才寫回」這個核心模型。但它目前更像「MVP + 多層安全硬化正在收斂」，不是「已可安心暴露 webhook 並審任意內部 repo」。下一階段應把重點放在 production gate，而不是再堆更多 review 規則或 UI。

## 本機現況

| 檢查 | 結果 |
|---|---|
| `git -C /home/asus/Reeve status --short --branch` | `master...origin/master`，但有未提交變更與 untracked 檔案 |
| dirty tracked files | `cmd/reeve/main.go`、`internal/llm/codex_backend.go`、`internal/llm/codex_live_test.go`、`openspec/specs/agent-backend-spawn/spec.md` 等 7 檔 |
| untracked | `config.json`、`internal/llm/codex_preflight.go`、`internal/llm/codex_preflight_test.go`、`docs/...2026-06-02...roadmap.md`、active / archived OpenSpec changes |
| latest commit | `b5f4287 Add Codex spawn read isolation` |
| Go | `go version go1.24.6 linux/amd64`；`go.mod` 宣告 `go 1.24.1` |
| Codex | `codex-cli 0.136.0` |
| bwrap | `/usr/bin/bwrap` |
| Linux Codex 0.136.0 補測 | `cli-enterprise-research.md:571` 記錄 direct sandbox 9/9、agent-level `codex exec --json --ephemeral` 9/9、暫時 `/etc/codex` managed policy 載入與 enforce 均通過 |
| `reeve --version` | `reeve version 0.1.0` |
| `go build ./...` | PASS |
| `go vet ./...` | PASS |
| `go test ./...` | PASS |
| `go test -race ./...` | FAIL：`internal/daemon` warning log buffer data race |
| `openspec validate --all` | PASS |
| `openspec list` | `harden-codex-security-control-plane 0/54 tasks` |
| live tests | Azure DevOps、Codex、Codex preflight、read isolation、systemd live tests 均因 env gate 未設而 skip |

目前 untracked `config.json` 只做結構性檢視，未輸出實值。它含 `azure_devops`、`codex.provider_mode=azure`、`codex.azure_openai`、`secrets.sources=["env"]`、`daemon.listen_addr`，但沒有看到 `codex.read_isolation.enabled`。

## Reeve 執行模型

來源依據：

| 來源 | 證據 |
|---|---|
| `README.md` | 定義 Reeve 為 Azure DevOps 內部 AI code reviewer，advisory-only |
| `cmd/reeve/main.go:188` | Cobra root command，加入 `review/serve/systemd/pipeline/cli/cred` |
| `cmd/reeve/main.go:275` | `review [pr-url]` 同時支援本機 self-review 與 remote PR |
| `cmd/reeve/main.go:104` | `serve` 建 daemon、worker、GC、PR review pipeline |
| `internal/actions/review/pipeline.go:123` | 準備 agent-facing working dir |
| `internal/actions/review/pipeline.go:170` | spawn backend 並 drain events |
| `internal/actions/review/pipeline.go:188` | server 端 dequeue queue |
| `internal/reporters/gates.go:60` | C3 chain：RedLine、RepoBlacklist、Permission、Audit |
| `internal/azdoreporter/azdo_pr.go:38` | 真正 Azure DevOps writeback 在 reporter 層 |
| `internal/llm/codex_backend.go:396` | Codex argv 組裝集中於 backend |

核心資料流：

```text
Azure DevOps webhook / CLI review
  -> diff.RemotePRSource / WorkingDirSource
  -> workspace.PrepareWorkingDir
  -> .reeve/context/diff.json + rules.json
  -> CodexBackend.Spawn
  -> reeve cli submit-finding / summary / abort
  -> .reeve/queue/actions.jsonl
  -> cli.Dequeue validates schema + recomputes dedup key
  -> reporter gate chain
  -> Azure DevOps thread / summary / advisory status / vote
```

這個模型的優點是權限集中：agent 產出的 stdout / markdown 不會直接貼 PR，只有 queue 中通過 Reeve server 端驗證的 finding 會進 reporter。

## CLI 現況

`go run ./cmd/reeve --help` 顯示：

| command | help 現況 | 判斷 |
|---|---|---|
| `review [pr-url]` | `run local self-review` | 實作已支援 PR URL，但 help 仍偏本機描述 |
| `serve` | `run webhook daemon` | 可用，但 help 沒暴露 config / listen 等 operator flags |
| `cli` | `submit-finding/summary/vote/abort` | agent-facing internal command |
| `cred` | `set/status/codex-azure` | SP 與 Azure key onboarding |
| `systemd unit` | print service unit | print-only，方向合理 |
| `pipeline` | `not implemented` | 仍是 stub，README command list 沒強調此限制 |

問題不是 `pipeline` stub 本身，而是產品表面與文件敘述需要對齊。若 `pipeline` 沒有近期用途，應隱藏或標成 experimental；若保留，就要避免讓 operator 以為這是可用功能。

## 安全現況

### 已做得好的地方

| 控制 | 證據 | 評價 |
|---|---|---|
| AzDO credential 不給 agent | `filterSpawnEnv` deny `REEVE_AZDO_SP_SECRET`、`REEVE_AZDO_PAT`、`AZDO_PAT` | 強控制，方向正確 |
| Azure OpenAI key proxy | `azure_openai_proxy.go` 由 Reeve loopback proxy 注入 upstream `api-key` | 比 key env / config.toml 安全 |
| subscription mode 不宣稱 production contained | `validateProvider` 在 `RequireProductionContainment` 下拒絕 subscription | 好 |
| Codex feature surface hardening | spawn 加 `--disable apps/plugins/hooks/browser_use/.../multi_agent` | 好，但需 preflight / managed policy 證明 |
| Linux egress probe | `verifyLinuxSandboxEgress` 對 `1.1.1.1:80` 做 probe，reachable 則 fail loud | 比只設 config 強 |
| bwrap read isolation primitive | `read_isolation.go` 用 bwrap 只 bind workdir、Codex home、必要 runtime paths | 方向正確 |
| advisory-only writeback | Azure client 僅有 safe REST write methods | 結構性優點 |

### 主要缺口

| 缺口 | 嚴重度 | 證據 | 風險 |
|---|---:|---|---|
| webhook 無請求認證 | P0 | `handleWebhook` 只檢 method、body parse、comment content | 任意可達者可打 `/webhook` 觸發 review queue |
| Codex production control plane 未完成 | P0 | active change `harden-codex-security-control-plane` 為 `0/54 tasks` | 不能把目前 spawn flags 當最終企業邊界 |
| read isolation 非 production fail-closed | P0 | config 可省略 `codex.read_isolation.enabled`；roadmap task 才要補 | Linux same-user host reads 仍是已知高風險 |
| Codex 版本 gate 未併入 Reeve | P1 | Linux `0.136.0` direct / agent-level / `/etc/codex` 補測已通過，但 Reeve 程式仍 `maxTestedCodexVersion=0.135` | 外部 smoke 證據已更新；Reeve 自身仍需把 0.136.0 gate / evidence 收進 production preflight |
| queue 缺 cross-process flock | P1 | `AppendAction` 建 `.lock` 但未使用 `gofrs/flock`；測試只跑 goroutine | 多個 `reeve cli` 子行程併發 append 可能 interleave / lost |
| race test 失敗 | P1 | `go test -race ./...` 在 daemon warning buffer 讀寫 race | 併發驗證不完整 |
| workspace copy 含 `.git` | P1 | `copyTree` 全量 copy regular files；`Sandbox` 只 remove origin + chmod `.git/config` | agent 可見 git metadata / history / worktree pointer，與 ephemeral isolation 敘述有落差 |
| OpenSpec 主規格 drift | P1 | `agent-cli-queue-bridge` 仍寫 C0 subcommands 必須 print `not implemented` | SDD ground truth 會誤導後續變更 |
| review 規則覆蓋有限 | P2 | 11 條 builtin rule 只有 5 條 enabled | 審查品質仍靠模型，需 eval / false-positive 管理 |

## Codex 控制面判斷

目前 Reeve Azure-mode spawn 大致會做：

| 類別 | 現況 |
|---|---|
| argv | `codex exec --json --strict-config --disable ... --skip-git-repo-check --ephemeral -c project_root_markers=['.reeve-workspace'] -c web_search=disabled -s workspace-write` |
| Linux network | `-c sandbox_workspace_write.network_access=false` + probe |
| Windows | `-c windows.sandbox=unelevated` |
| Azure provider | 透過 Reeve loopback proxy base URL，不把 raw key 交給 agent env |
| rules | Azure-mode 不加 `--ignore-rules`，改 seed Reeve-owned trusted rules |
| preflight | 新增 `codex_preflight.go`，檢 help、doctor、features、MCP |

這比早期版本強很多，但還不是最終 production control plane。官方 Codex 文件把控制面拆成 sandbox / approval、managed requirements、managed defaults、permission profiles、deny-read、feature pins、MCP allowlist、managed command rules、hooks。Reeve 現在的 active roadmap 也同意：只加 permission profile 不夠，必須形成完整 security control plane。

本機 `codex doctor --json` 顯示 user Codex config 仍有多個 feature 預設 enabled，例如 `hooks`、`multi_agent`、`apps`、`plugins`、`browser_use`、`computer_use`、`image_generation`。Reeve spawn 有用 `--disable` 壓住這些功能，這是必要的；但 production 要求應由 managed requirements / managed config 與 preflight evidence 證明，而不是只相信 argv。

補充新證據：`cli-enterprise-research.md:571` 已補測 Linux `codex-cli 0.136.0`。direct sandbox 自訂 profile 9/9 通過；agent-level `codex exec --strict-config --ephemeral --json` 由 `command_execution` event 執行 boundary script，9/9 通過；暫時部署 root-owned `/etc/codex/requirements.toml` / `/etc/codex/managed_config.toml` 後，危險 user config `approval_policy="never"`、`sandbox_mode="danger-full-access"` 被 requirements 拉回 OnRequest / restricted sandbox，JSONL event 明確標示 fallback 來源為 `/etc/codex/requirements.toml`。測後 `/etc/codex`、臨時 workspace、臨時 `CODEX_HOME` 已清理。這使「0.136.0 Linux sandbox/system policy 是否退化」可視為已補測通過；但它仍不是 Reeve production daemon 的長駐 service user / lifecycle / audit gate。

特別注意：本機 `which codex` 是 `/home/asus/.local/bin/codex`。`read_isolation.go` 的 path validator 會拒絕暴露 `/home/` 下的 executable directory / PATH entry。這代表若在此 host 直接把 `codex.read_isolation.enabled=true` 套進 production path，可能需要把 Codex 安裝到 service-controlled `/opt` 或 `/usr/local`，再用專用 service user 驗證。現有 `TestLiveReadIsolationProof` 預設 skip，且 live proof 是 primitive proof，不等於完整 Codex production E2E。

## Agent Workspace 問題

`workspace.PrepareWorkingDir` 目前是全量 copy：

```text
source repo/worktree
  -> os.MkdirTemp("", "reeve-workingdir-*")
  -> copyTree(root, dir)
  -> materialize skills + AGENTS.md + .reeve-workspace
  -> write .reeve/context + queue
  -> Sandbox(dir)
```

`Sandbox` 做兩件事：`git remote remove origin`，以及若 `.git/config` 存在就 chmod `0`。

問題：

1. local repo 會把 `.git` directory 的 objects / refs / logs 一起 copy 給 agent-facing workspace。
2. remote git worktree 的 `.git` 常是 pointer file，copy 後可能指向 cache bare repo 的 gitdir。
3. `chmod .git/config` 不是完整 git metadata redaction，也不是 workspace-only read boundary。
4. 若 bwrap read isolation 開啟，`.git` pointer 指向外部 cache 會使 git command 行為變得不穩；若未開啟，agent 可能讀到 cache 路徑。

優化方向：agent-facing workspace 應明確選一種策略，不要半隔離。

| 策略 | 建議 |
|---|---|
| 最安全 | 產生 no-git source snapshot，不帶 `.git`，只帶工作樹內容、diff context、必要 rules |
| 折衷 | copy 後移除 `.git`，另用 `.reeve/context` 提供 base/head SHA 與 patch |
| 若需 git metadata | 只掛 read-only sanitized git metadata，不能指向外部 cache |

## 規格與文件漂移

OpenSpec 工具驗證是綠的，但人工檢查有明顯 drift：

| 檔案 | 問題 |
|---|---|
| `openspec/specs/agent-cli-queue-bridge/spec.md` | 仍保留 C0「`review/serve/pipeline/cli` 印 `not implemented`」的 normative requirement |
| 同檔後段 | 又要求 `serve` 啟動 webhook listener，和前段 C0 requirement 衝突 |
| `cmd/reeve/main.go` | `pipeline` 仍 stub，`review` help 與實際 PR URL 能力不一致 |
| README | 說 `0.1.0 MVP COMPLETE`，但同時列出曝露前安全 gate；需要更清楚區分 MVP / production ready |
| active roadmap | `harden-codex-security-control-plane` 已建立但 0/54，README 應避免讓人以為這些 gate 已完成 |

這類 drift 對 Spectra / OpenSpec 專案傷害很大。若 spec 不能作為真相，後續 agent 會依過期 requirement 寫出「符合 spec、違反現況」的變更。

## 測試與驗證

目前測試量不低：去掉 archive 後約 113 個 Go 檔、48 個 test file、約 20k Go LOC。`internal/llm/codex_backend_test.go` 與 `cmd/reeve/main_test.go` 覆蓋很多安全 flag / CLI wiring。

但測試分層還有缺口：

| 層級 | 現況 | 建議 |
|---|---|---|
| unit | `go test ./...` 綠 | 保持 |
| static | `go vet ./...` 綠 | CI 已跑，保持 |
| race | `go test -race ./...` 失敗 | 至少修 `internal/daemon`，再把關鍵 packages 加進 CI |
| OpenSpec | `openspec validate --all` 綠 | 增加人工 / script 檢查 stale C0/TBD/not implemented |
| live Azure | 預設 skip | production gate 前必須在目標 tenant / repo 跑 |
| live Codex preflight | 預設 skip | Codex 升級或 host 變更必跑 |
| Linux 0.136 sandbox smoke | direct / agent-level / `/etc/codex` 補測已通過 | 還要併入 Reeve production preflight 與長駐 runner lifecycle |
| live read isolation | 預設 skip | 必須以 production service user + actual Codex binary 跑 |
| direct sandbox smoke | roadmap 尚未完成 | 應用 non-secret sentinel，不依賴模型 final text |
| queue concurrency | 只測 goroutine | 補 subprocess cross-process append test |

## 優化建議

### Gate 0：先把 ground truth 穩住

1. 修 `go test -race ./...` 的 daemon race。最低限度：測試用 thread-safe writer；更好是 production `WarningLog` 包一層 synchronized writer，避免不同 goroutine 寫任意 `io.Writer`。
2. `internal/cli/queue.go` 改用 `gofrs/flock` 或等價跨 process lock。既然 audit 已有成熟 pattern，queue 應直接沿用。
3. 清理 OpenSpec 主規格，把 C0 stub requirement 標成 archived/historical，不可留在 active normative spec。
4. CI 增加 `openspec validate --all`，並至少對 `internal/daemon`、`internal/cli`、`internal/llm` 跑 focused `-race`。
5. 釐清 dirty worktree：目前 uncommitted hardening 變更應拆成一個明確 PR / change，不要讓 `master` 工作樹長期處於「半完成 production gate」狀態。

### Gate A：daemon 暴露前必補

1. `/webhook` 加認證：Azure DevOps Service Hooks Basic Auth、HMAC shared secret、或反向 proxy mTLS，至少一種。
2. `handleWebhook` 加 body size limit，例如 `http.MaxBytesReader`，避免任意大 body。
3. webhook replay / redelivery model 寫清楚：目前 dedup 是 PR + SHA；comment trigger 的 key 是 `comment:<PRKey>`，要確認 `/reeve review` 重跑語意。
4. health / readiness 分離：`/healthz` 只能代表 process alive，不能代表 Codex policy、Azure credential、read isolation ready。
5. systemd live check 要驗 production service user 的 Codex policy evidence，而不只是 service active + healthz。

### Gate B：Codex production security control plane

1. 完成 `harden-codex-security-control-plane`，不要只停在 spec/proposal。
2. Azure-mode production 改走 managed profile route：`/etc/codex/requirements.toml`、`/etc/codex/managed_config.toml`、`default_permissions="reeve-review"`。
3. 移除 production 對 legacy `-s workspace-write` / `sandbox_workspace_write.*` 的依賴；若保留 local developer mode，必須明確不宣稱 contained。
4. requirements 裡 pin feature：`apps/plugins/hooks/browser_use/browser_use_external/computer_use/in_app_browser/image_generation/multi_agent`。
5. MCP deny-all：`mcp_servers` present but empty，或明確 allowlist。
6. deny-read 規則：`.env`、`~/.codex`、`~/.reeve`、daemon secret path、host credential path。
7. managed command rules：host read、network tools、destructive git/file commands 禁止；正常 `reeve cli submit-finding/summary/abort` 允許。
8. 每次 production spawn 前輸出 redacted evidence：Codex version、help capabilities、doctor verdict、MCP verdict、feature verdict、policy verdict、argv summary、cwd、CODEX_HOME path、env keys only。

### Gate C：read isolation 與 workspace

1. Linux Azure production 預設要求 `codex.read_isolation.enabled=true`，否則 fail closed。
2. production Codex binary 不應位於 service user home。放 `/opt/reeve/bin/codex` 或 `/usr/local/bin/codex`，再跑 bwrap proof。
3. read isolation proof 要用 actual Codex production path，不只 `/bin/sh` sentinel。
4. agent-facing workspace 不帶 `.git`。若 review 需要 commit metadata，透過 `.reeve/context` 給 sanitized SHA / patch。
5. remote bare cache 只給 Reeve host process 使用，不給 agent workspace pointer 到 cache。

### Gate D：review 品質

1. 建立小型 golden PR corpus：security、bug、drift、false positive、no finding。
2. 對 11 條 builtin rules 設定啟用策略。目前只有 5 條 enabled，若其他規則刻意關閉，要在 README 說明。
3. 加 noise budget：同 PR 每輪 max inline、summary grouping、severity threshold、重審 delta 策略。
4. agent finding suppression 是本質風險，不能靠 prompt 解決。應用 eval 追蹤漏報率，並把 Reeve 定位為 advisory，不可替代 reviewer。

### Gate E：文件與產品表面

1. README 把狀態分成三層：`MVP functional complete`、`controlled pilot requirements`、`production go-live requirements`。
2. CLI help 改成 operator 可理解的文字：`review [pr-url]` 不只是 local self-review。
3. `pipeline` 若無用途就 hidden；若是 roadmap，就標 experimental。
4. Linux runbook 加入 Codex managed policy、bwrap、service user、live proofs、exact commands。
5. 每次 Codex 版本升級都要更新 max-tested version 或明確保留 warning，並附 smoke 結果；Linux 0.136.0 smoke 已在 `cli-enterprise-research.md` 補上，下一步是把它轉成 Reeve 可重跑的 gate。

## 建議上線分級

| 階段 | 可做 | 不可宣稱 |
|---|---|---|
| Local dev | `reeve review` 自審、fake / mocked pipeline、規則調整 | 不代表 production safety |
| Controlled pilot | 修 race、queue flock、webhook auth、B0 preflight evidence 後，可挑低敏 repo 試跑 | 不代表不可信 repo 安全 |
| Internal production | 完成 Codex security control plane、read isolation fail-closed、service user live proof、systemd readiness gate | 不代表能審任意 third-party code |
| 不可信 repo / 高敏 repo | 建議再加容器 / VM / ephemeral CI runner，只掛 workspace | 不應只靠 Codex sandbox + prompt |

## 官方來源

本文件使用 OpenAI Codex manual helper 於 2026-06-02 取得 current manual，路徑為 `/tmp/openai-docs-cache/codex-manual.md`。相關官方來源：

| 主題 | 來源 |
|---|---|
| sandbox / approvals / network | <https://developers.openai.com/codex/agent-approvals-security> |
| sandbox platform prerequisites | <https://developers.openai.com/codex/concepts/sandboxing> |
| managed requirements / managed config | <https://developers.openai.com/codex/enterprise/managed-configuration> |
| permission profiles | <https://developers.openai.com/codex/permissions> |
| command rules / execpolicy | <https://developers.openai.com/codex/rules> |
| hooks | <https://developers.openai.com/codex/hooks> |

## 本機證據摘要

| 證據 | 結果 |
|---|---|
| `go test ./...` | PASS |
| `go build ./...` | PASS |
| `go vet ./...` | PASS |
| `go test -race ./...` | FAIL，daemon warning log race |
| `go run ./cmd/reeve --version` | `reeve version 0.1.0` |
| `go run ./cmd/reeve --help` | 顯示 `pipeline not implemented`、`review run local self-review` |
| `codex --version` | `codex-cli 0.136.0` |
| Linux 0.136 direct sandbox | `enterprise-linux` profile 9/9 通過 |
| Linux 0.136 agent-level `codex exec` | boundary script `SUMMARY pass=9 fail=0`，workspace 外寫入未建立 |
| Linux 0.136 temporary `/etc/codex` | `requirements.toml` / `managed_config.toml` 可載入並 enforce；dangerous user config 被 fallback |
| `codex features list` | 多個 feature 預設 enabled；Reeve spawn 需 disable |
| `codex mcp list --json` | `[]` |
| `codex doctor --json` | overall ok；user `CODEX_HOME=/home/asus/.codex`；sandbox restricted；features 多項 enabled |
| `openspec list` | `harden-codex-security-control-plane 0/54 tasks` |
| `openspec validate --all` | PASS |
| live tests | Azure / Codex / preflight / read isolation / systemd live 均 skip |

## 待補

1. 在目標 production service user 上跑 `REEVE_LIVE_READ_ISOLATION=1`，且使用 actual Codex binary path。
2. 把已通過的 Linux 0.136 `/etc/codex` 補測轉成 Reeve production runner 的長駐 lifecycle / ownership / audit gate。
3. 補 webhook auth 後跑 daemon E2E，不只 `/healthz`。
4. 補 queue cross-process flock test。
5. 修 race test 後決定是否把 `-race` 加 CI。
6. 清除 active spec 裡的 C0 historical requirements。
7. 把 README 的 MVP / pilot / production-ready 狀態拆清楚。
8. WSL2 跑同套 Linux 0.136 sandbox / `/etc/codex` 測試；擴充 `.git` / `.codex` metadata deny 驗證；Windows elevated sandbox 仍需確認。
