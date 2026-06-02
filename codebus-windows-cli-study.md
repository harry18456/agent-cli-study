# codebus Windows CLI 研究

日期：2026-06-01；安全修訂：2026-06-02
目標路徑：`D:\side_project\codebus`
CLI 版本：Claude Code `2.1.159`，OpenAI Codex CLI `0.135.0`

## 範圍修正

前一版錯把 `D:\side_project\codebus\target` 當研究目標。正確目標是 `D:\side_project\codebus`。

`target` 只是 Rust/Tauri build output 與舊 PoC 產物；`codebus` 根目錄才是實際應用：

| 項目 | `D:\side_project\codebus` |
|---|---|
| repo root | `D:/side_project/codebus` |
| Rust workspace | `codebus-core`、`codebus-cli`、`codebus-app/src-tauri` |
| app source | `codebus-app` |
| CLI source | `codebus-cli` |
| core source | `codebus-core` |
| vault | `.codebus` |
| Claude project config | `.claude`、`CLAUDE.md` |
| build output | `target` |

本文件替代前一版 `codebus-target-windows-cli-study.md`。

## 結論

codebus 不是單純「把 Claude/Codex 開在 repo root」。它是一個 wrapper：

1. 使用者對 `codebus.exe` 下 `init/goal/query/fix/chat/quiz/lint`。
2. codebus 把 source repo 同步到 `.codebus/raw/code/`，並套 PII filter。
3. agent CLI 的 cwd 是 `<repo>/.codebus`，不是 source repo root。
4. Claude/Codex 只應讀 vault 內的 schema、skills、wiki 與 raw mirror。
5. Claude/Codex provider 透過不同 flag 組合實作隔離與輸出解析。

企業判斷：

| 面向 | Claude provider | Codex provider |
|---|---|---|
| codebus 內建整合成熟度 | 較高；工具 allowlist、Bash/Read PreToolUse hook、MCP/user setting 隔離較明確 | 可用；JSONL 與 sandbox 整合好，但 Windows read/network boundary 是 partial |
| Windows native 安全邊界 | 寫入較受限；讀取不是 vault hard boundary | 讀取不受限；寫入正常 ACL 多數可擋，但 ACL-writable 目錄例外；network 擋不完整 |
| codebus 目前防線 | `.codebus` cwd、PII mirror、tool allowlist、hooks、`--setting-sources project,local`、empty MCP | `.codebus` cwd、PII mirror、`--ignore-user-config`、feature disable、`-s` sandbox、`windows.sandbox=unelevated`、`web_search=disabled` |
| 主要風險 | `check-read` 是 denylist，非 vault allowlist；`Glob/Grep` 未掛 hook；父 shell env 會繼承 | Windows unelevated 不能硬擋讀 user home / sibling repo；`workspace-write` 允許 HTTP/80 與 loopback |
| 企業建議 | Windows 可作受控寫入工具；敏感讀取仍需外部 filesystem boundary 或 hook 改成 allowlist | 優先搬 Linux/WSL2 runner；Windows native 要加 AppContainer/VM/ACL/firewall |

codebus 在企業內使用時，不應把安全宣告寫成「CLI 已保證 repo 隔離」。正確宣告是：codebus 有 application-level 降風險設計，但企業級 hard boundary 必須由 runner/OS/network policy 補上。

## codebus 當前狀態

本機檢查：

| 檢查 | 結果 |
|---|---|
| `git -C D:\side_project\codebus status --short --branch` | 本次修訂時有使用者未提交變更，未納入本文件修改 |
| `codebus.exe --repo D:\side_project\codebus --help` | 可執行，列出 `init/goal/query/lint/fix/chat/quiz/config` |
| `codebus.exe --repo D:\side_project\codebus lint --format json` | `error_count=0`、`warn_count=0` |
| `.codebus/manifest.yaml` | `repo_root: \\?\D:\side_project\codebus`、`file_count: 1377`、`total_bytes: 15651384` |
| `~/.codebus/config.yaml` | `agent.active_provider: codex` |

目前 `~/.codebus/config.yaml` 的重要狀態：

| 項目 | 狀態 |
|---|---|
| active provider | `codex` |
| Claude profile | configured；active profile 為 Azure |
| Codex profile | configured；active profile 為 system |
| API key | 不在本文件輸出；codebus 設計以 keyring/env 注入 |
| `CODEBUS_HOME` | 可改變 config root，適合 CI/runner 隔離 |

## codebus 的 agent 執行模型

來源依據：

| 來源 | 證據 |
|---|---|
| `CLAUDE.md` | codebus 是 CLI + Tauri app，驅動 Claude Code 或 Codex 建置 `.codebus` wiki |
| `codebus-cli/src/main.rs` | verbs：`init/goal/query/lint/fix/chat/quiz/config` |
| `codebus-core/src/agent/claude_cli.rs:136` | agent child process `current_dir(vault_root)` |
| `codebus-core/src/agent/claude_cli.rs:428` | Claude argv 組裝集中在 `compose_claude_cmd` |
| `codebus-core/src/agent/codex_backend.rs:92` | Codex argv 組裝集中在 `CodexBackend::build_command` |
| `codebus-core/src/config/mod.rs:58` | `CODEBUS_HOME` 可覆蓋預設 config path |

核心設計：

```text
source repo: D:\side_project\codebus
  -> codebus init / goal / query / chat / quiz / fix
  -> vault: D:\side_project\codebus\.codebus
  -> raw mirror: .codebus\raw\code
  -> wiki: .codebus\wiki
  -> agent cwd: .codebus
  -> provider: Claude Code or Codex
```

安全意義：

| 設計 | 作用 | 限制 |
|---|---|---|
| cwd 改到 `.codebus` | agent 不直接在 source root 操作 | CLI/OS 若允許絕對路徑，仍可能越界讀 |
| `raw/code` mirror | agent 讀 PII-redacted copy | mirror 準確性與 PII scanner 是前置假設 |
| nested `.codebus` git | wiki 變更獨立追蹤 | agent 仍可能修改 vault 內 schema/wiki，靠 hooks/validator 管 |
| provider-neutral backend | Claude/Codex 可 swap | 兩個 CLI 的 sandbox 語意不等價 |

## Claude provider

codebus 的 Claude backend 會組出以下核心 argv：

| flag | codebus 用途 |
|---|---|
| `-p <slash_command>` | 觸發 codebus skill workflow |
| `--tools <csv>` | toolset hard gate |
| `--allowedTools <csv>` | auto-approval safety net |
| `--permission-mode acceptEdits` | 非互動 `-p` 模式必要權限 |
| `--output-format stream-json --verbose` | 事件解析、usage、audit |
| `--no-session-persistence` | 非 chat verb 不留下 session rollout |
| `--strict-mcp-config --mcp-config {"mcpServers":{}}` | 清空 MCP load layer |
| `--setting-sources project,local` | 排除 user/global settings |
| `--model` / `--effort` | per-verb config |

`.codebus/.claude/settings.json` 內有 PreToolUse hooks：

| matcher | hook |
|---|---|
| `Bash` | `codebus hook check-bash` |
| `Read` | `codebus hook check-read` |

讀取邊界修訂：

| 面向 | 結論 |
|---|---|
| `check-read` 性質 | denylist，不是 vault allowlist |
| 已擋項目 | image/binary extension、`*id_rsa*`、`*.pem`、`*.key`、`~/.ssh/`、`~/.aws/`、`~/.gnupg/`、`~/.config/gh/` |
| 未擋項目 | repo 外 source 絕對路徑、母 repo 原始檔、`~/.kube/`、`~/.docker/config.json`、家目錄 `.env`、其他未列入 denylist 的憑證 |
| hook 覆蓋 | `.codebus/.claude/settings.json` 只 match `Bash` 與 `Read`；`Glob/Grep` 不經 `check-read` |
| Windows native 讀取結論 | Claude Code native Windows 無 OS filesystem sandbox；headless `-p` 下讀取工具不會經互動批准，因此讀取不能宣稱 confine 在 vault |

寫入邊界修訂：

| 面向 | 結論 |
|---|---|
| cwd | codebus spawn cwd 是 `<repo>/.codebus` |
| toolset | read-only verbs 只給 `Read,Glob,Grep`；workspace verbs 才給 `Write,Edit` |
| headless `acceptEdits` | cwd 外寫入需要額外批准；headless 無互動批准時會被 Claude permission layer deny |
| Windows native 寫入結論 | Claude path 對 source repo 寫入相對受限；風險重點應放在讀取外洩與 vault 內可寫 |

注意：codebus 的 Claude spawn 不應簡單加 `--disable-slash-commands`，因為它需要 vault 內的 codebus skills/slash workflow。`--disable-slash-commands` 適合 raw Claude root-session 隔離測試，不適合 codebus wrapper 自己的 agent run。

### 根目錄 raw Claude 實測

工作目錄：`D:\side_project\codebus`

default command：

```powershell
claude --print "Reply exactly CODEBUS_ROOT_CLAUDE_OK" `
  --verbose `
  --output-format stream-json `
  --max-turns 1 `
  --tools=
```

摘要：

| 欄位 | 結果 |
|---|---|
| hook_started | `1`，`SessionStart:startup` |
| cwd | `D:\side_project\codebus` |
| tools_count | `9` |
| mcp_count | `4` |
| slash_commands_count | `105` |
| skills_count | `65` |
| plugins | `gopls-lsp`、`frontend-design`、`skill-creator`、`superpowers` |
| memory_auto | `C:\Users\harry\.claude\projects\D--side-project-codebus\memory\` |

結論：raw Claude root session 預設會載入 user/global hooks、plugins、skills、MCP、memory path。只加 `--tools=` 不足以隔離。

isolated command：

```powershell
$mcp = Join-Path $env:TEMP 'empty-mcp-agent-study.json'
Set-Content -LiteralPath $mcp -Encoding utf8 -Value '{"mcpServers":{}}'

claude --print "Reply exactly CODEBUS_ROOT_CLAUDE_ISOLATED_OK" `
  --verbose `
  --output-format stream-json `
  --max-turns 1 `
  --tools= `
  --setting-sources project,local `
  --disable-slash-commands `
  --strict-mcp-config `
  --mcp-config $mcp
```

摘要：

| 欄位 | 結果 |
|---|---|
| hook_started | `0` |
| tools_count | `0` |
| mcp_count | `0` |
| slash_commands_count | `0` |
| skills_count | `0` |
| plugins_count | `0` |
| memory_auto | 仍是 `C:\Users\harry\.claude\projects\D--side-project-codebus\memory\` |

結論：raw session 隔離 flags 能清掉 tools/MCP/slash/skills/plugins/hook surface，但仍不是 OS filesystem sandbox。

### Claude provider 優缺

優點：

| 項目 | 說明 |
|---|---|
| codebus wrapper 已排除 user setting source | `--setting-sources project,local` 不載入 user/global settings |
| MCP 清空 | `--strict-mcp-config` + empty MCP config |
| 工具面精準 | `--tools` 和 `--allowedTools` 明確對應 verb |
| hooks 可做 PreToolUse gate | Bash/Read 可用 codebus 自己的 hook；Bash gate 比 Read gate 強 |
| 寫入較受限 | cwd 是 `.codebus`，source repo 寫入在 headless 模式下不能自動批准 |
| stream-json 可 audit | init/tool/result/usage 可解析 |

缺點：

| 項目 | 說明 |
|---|---|
| native Windows hard sandbox 不成立 | Claude 官方 sandbox 支援 macOS/Linux/WSL2；native Windows 不應當 hard boundary |
| raw root session 風險高 | 不經 codebus wrapper 時 user/global surface 會載入 |
| Read hook 不是 allowlist | 絕對路徑讀取可到 vault 外；denylist 未涵蓋 `.kube`、`.docker`、`.env` 等 |
| `Glob/Grep` 沒有 hook | `settings.rs` 預設 PreToolUse 只 match `Bash` 與 `Read` |
| 讀取邊界仍需外部層 | 讀取能力本身不能替代 OS deny |
| hook 是 CLI 層 gate | 若 CLI/hook 路徑失效，不是 kernel-enforced boundary |

## Codex provider

codebus 的 Codex backend 會組出以下核心 argv：

| flag/config | codebus 用途 |
|---|---|
| `codex exec --json` | JSONL 事件解析 |
| `--ignore-user-config` | 不讀 user `config.toml` |
| `--disable apps/plugins/hooks/browser_use/browser_use_external/computer_use/in_app_browser` | 關閉未測 feature surface |
| `--ignore-rules` | 不讀 rules |
| `--skip-git-repo-check` | vault root 可獨立運作 |
| `-c project_root_markers=['.codebus-vault']` | 把 project root pin 到 vault |
| `-c windows.sandbox=unelevated` | Windows non-admin 路線 |
| `-c web_search=disabled` | 關閉 hosted web search |
| `--ephemeral` | 非 chat verb 不持久化 session |
| `-s read-only` / `-s workspace-write` | 依 verb permission 選 sandbox |

目前 codebus 使用 `codex exec --json ... -s read-only|workspace-write` route，不是 `codex sandbox --permissions-profile` route，也不是新版 managed permission profile route。這點很重要：Codex 官方新版 permission profiles、requirements、managed config 可以成為企業路線，但不能和 codebus 現行 `-s sandbox_mode` 混在一起當同一個保證。

### 根目錄 Codex direct sandbox 手動探針

工作目錄：`D:\side_project\codebus`

本節是外部手動測試，使用 `codex sandbox --permissions-profile`。它不是 codebus 實際 argv；只能作為 Windows unelevated sandbox 的輔助觀察，不能直接代表 codebus provider。

讀 user global：

```powershell
codex sandbox `
  -c 'windows.sandbox="unelevated"' `
  --permissions-profile ':workspace' `
  -C D:\side_project\codebus `
  powershell -NoProfile -NonInteractive -Command `
  "Get-Content -LiteralPath C:\Users\harry\.codex\AGENTS.md -TotalCount 1"
```

結果：`USER_AGENTS_READ_OK`。

讀 sibling repo：

```powershell
codex sandbox `
  -c 'windows.sandbox="unelevated"' `
  --permissions-profile ':workspace' `
  -C D:\side_project\codebus `
  powershell -NoProfile -NonInteractive -Command `
  "Get-Content -LiteralPath D:\side_project\agent-study\cli-enterprise-research.md -TotalCount 1"
```

結果：`SIBLING_REPO_READ_OK`。

read-only 寫入 codebus root：

```powershell
codex sandbox `
  -c 'windows.sandbox="unelevated"' `
  --permissions-profile ':read-only' `
  -C D:\side_project\codebus `
  powershell -NoProfile -NonInteractive -Command `
  "Set-Content -LiteralPath .\codex_root_readonly_write_probe.tmp -Value SHOULD_NOT_WRITE"
```

結果：`ROOT_WRITE_BLOCKED:Access to the path ... is denied.`，且 probe file 不存在。

network：

```powershell
codex sandbox `
  -c 'windows.sandbox="unelevated"' `
  --permissions-profile ':read-only' `
  -C D:\side_project\codebus `
  curl.exe -I --max-time 8 https://example.com
```

結果：sandbox 內失敗，嘗試經 `127.0.0.1` 連線；同一 `curl.exe` 在 sandbox 外可 HTTP 200。

結論：

| 能力 | Windows unelevated 結論 |
|---|---|
| workspace/root 內寫入 | `:read-only` 可擋 |
| workspace 外讀取 | 擋不住 |
| user home 讀取 | 擋不住 |
| sibling repo 讀取 | 擋不住 |
| network | `:read-only` 失敗是預期；不代表 `workspace-write` egress 安全 |

### codebus-style Codex exec workspace-write 補測

本節改用 codebus 實際 surface：

```text
codex.cmd exec --json
  --ignore-user-config
  --disable apps
  --disable plugins
  --disable hooks
  --disable browser_use
  --disable browser_use_external
  --disable computer_use
  --disable in_app_browser
  --ignore-rules
  --skip-git-repo-check
  -c "project_root_markers=['.codebus-vault']"
  -c windows.sandbox=unelevated
  -c web_search=disabled
  --ephemeral
  -s workspace-write
```

工作目錄：`D:\side_project\codebus\.codebus`。模型端用本機 mock Responses API 固定觸發單一 `shell_command`，未使用真實模型推論。

network：

| probe | 結果 |
|---|---|
| `curl.exe -I --max-time 8 https://example.com` | blocked；失敗點落在 443 連線 |
| `curl.exe --noproxy '*' -I --max-time 8 http://example.com` | allowed；取得 HTTP response headers |
| `curl.exe http://127.0.0.1:<port>` | allowed；本機 listener 收到真實 hit |
| `-s read-only` network | policy 擋下，這是 read-only 本來行為，不能推論 workspace-write |

寫入：

| probe | 結果 |
|---|---|
| 寫 `D:\side_project\agent-study\...tmp` | blocked；`Access to the path ... is denied` |
| 寫 `%TEMP%\codex_user_temp_write_probe_...tmp` | allowed；檔案存在且內容為 probe marker |
| 寫 `C:\Windows\Temp\...tmp` | 本機此次未重現；Access denied |
| codebus `docs/security.md` §5 | 記錄 `Everyone-writable (*S-1-1-0)` 目錄可寫，例：`C:\Windows\Temp`、ACL 鬆的 app 目錄 |

結論：

| 能力 | Windows unelevated `-s workspace-write` 結論 |
|---|---|
| 讀取 | 不擋 workspace 外讀取 |
| 正常 ACL 外部寫入 | 多數可擋 |
| ACL-writable 外部寫入 | 可寫；本機 `%TEMP%` 已重現，`security.md` §5 記錄 Everyone-writable 類例外 |
| 外部 HTTPS/443 | blocked |
| 外部 HTTP/80 | allowed when proxy bypassed |
| loopback | allowed |
| `web_search=disabled` | 只關 hosted web search；不能當 raw shell egress 防線 |

### Codex provider 優缺

優點：

| 項目 | 說明 |
|---|---|
| JSONL wrapper 友善 | 適合 codebus event parser / audit log |
| codebus 已關閉多個未測 feature | apps/plugins/hooks/browser/computer-use/in-app-browser |
| `project_root_markers` 可 pin vault | 降低 `.codex` / `AGENTS.md` parent discovery 影響 |
| `--ephemeral` 適合 single-shot verbs | 降低 session rollout 污染 |
| Linux/WSL2 企業路線較強 | 官方 Codex Linux sandbox 使用 OS sandbox primitives，前一份研究已有 Linux passing evidence |

缺點：

| 項目 | 說明 |
|---|---|
| Windows unelevated 不擋讀 | 實測可讀 user `AGENTS.md` 與 sibling repo |
| `workspace-write` network 擋不完整 | HTTPS/443 擋下，但 HTTP/80 與 loopback 放行 |
| `workspace-write` 寫入有 ACL 例外 | 正常 ACL 外部寫入被擋；`%TEMP%` 或 Everyone-writable/ACL-writable 目錄可寫 |
| `-s` route 與 permission profiles 不等價 | 企業新版控制面需另設計 migration |
| `--ignore-user-config` 不是完整隔離 | auth/HOME/plugin loader/feature flags 仍需專用 `CODEX_HOME` 與 disable set |
| codebus 目前 active provider 是 codex | 在 Windows 使用時，敏感讀取任務風險較高 |
| shell/network 邊界需外部政策 | 不得只根據 `web_search=disabled` 宣稱離線 |

## 額外風險查證

### agent env 繼承

codebus spawn Claude/Codex child process 時沒有看到 `env_clear` 或等效清空。Rust `Command` 預設繼承父 process env；codebus 目前只額外注入 provider 需要的 env：

| provider | 原始碼行為 | 風險 |
|---|---|---|
| Claude | `Command::new(claude_bin)` 後 `cmd.envs(...)` 注入 scoped env | 父 shell 的 `GITHUB_TOKEN`、`AWS_*`、`AZURE_*`、`KUBECONFIG` 等會進 agent |
| Codex | `Command::new(codex_bin)` 後必要時 `cmd.env(CODEBUS_CODEX_AZURE_KEY, key)` | 同上；PII mirror 不掃 env |

本機 mock Claude probe 已重現：父 shell 設 `GITHUB_TOKEN=CODEBUS_ENV_PROBE_GITHUB_TOKEN`、`AWS_ACCESS_KEY_ID=CODEBUS_ENV_PROBE_AWS_KEY` 後，codebus `query` 產生的 mock child log 讀得到兩者。

企業結論：agent runner 必須使用乾淨 env allowlist。只靠 `CODEBUS_HOME`、`CODEX_HOME` 不能處理 env secret。

### raw_sync gitignored 與非 UTF-8

`raw_sync.rs` 兩個 path 使用 `.git_ignore(true)`，且非 UTF-8 path 用 `fs::read_to_string(path).ok()` 判斷；讀失敗時 `matches = Vec::new()`，最後 byte-identical `fs::copy`。

| 類型 | 行為 | 風險 |
|---|---|---|
| gitignored 檔案 | 不進 raw mirror，也不掃描 PII | 機密未外洩到 mirror，但企業 audit 看不到該檔存在與是否含 secret |
| root `.env` | `ALWAYS_SKIP_AT_ROOT` 排除 | 同上 |
| UTF-16LE / 非 UTF-8 | 不掃 PII，byte-identical 複製進 raw mirror | secret 可未遮罩進 vault |

本機 probe 已重現：`.gitignore` 內 `secrets/` 未進 mirror；UTF-16LE 檔案內的 AWS-like marker byte-identical 進 raw mirror，解碼後仍含 secret。

企業結論：raw mirror 需要非 UTF-8 fail-closed 或 decode/scan；gitignored skip 需要 manifest/audit，不應靜默。

## codebus 內部使用建議

本機直接操作 codebus repo：

```powershell
D:\side_project\codebus\target\release\codebus.exe `
  --repo D:\side_project\codebus `
  lint --format json
```

Claude provider 測試建議：

```powershell
$env:CODEBUS_HOME = 'D:\agent-runner\codebus-home'
$env:CODEBUS_CLAUDE_BIN = 'claude'

D:\side_project\codebus\target\release\codebus.exe `
  --repo D:\side_project\codebus `
  query "說明目前 wiki 中的 agent sandbox model"
```

Codex provider 測試建議：

```powershell
$env:CODEBUS_HOME = 'D:\agent-runner\codebus-home'
$env:CODEBUS_CODEX_BIN = 'codex.cmd'

D:\side_project\codebus\target\release\codebus.exe `
  --repo D:\side_project\codebus `
  query "說明目前 wiki 中的 agent sandbox model"
```

企業 runner 不應使用開發者個人的 `~/.codebus`、`~/.claude`、`~/.codex` 作執行環境。應使用專用 service account、專用 `CODEBUS_HOME`、專用 `CODEX_HOME`、managed settings/requirements、短期憑證、乾淨 env allowlist 與可銷毀 workspace。

## swap 影響

| swap 面向 | 影響 |
|---|---|
| tool model | Claude 的 `--tools`/hooks 不是 Codex 的 permission profile |
| filesystem model | Codex `-s read-only` 在 Windows 不是讀取邊界；不能等價轉成 Claude read-only |
| read model | Claude 的 `Read` hook 目前是 denylist，`Glob/Grep` 未 hook；Codex Windows `-s` 不擋讀 |
| prompt surface | Claude 用 slash skill；Codex 用 `$codebus-<bundle>` skill invocation |
| session | Claude 非 chat 加 `--no-session-persistence`；Codex 非 chat 加 `--ephemeral` |
| user config | Claude 用 `--setting-sources project,local`；Codex 用 `--ignore-user-config`，但還需 feature disable |
| MCP/plugin | Claude 用 strict empty MCP；Codex 需 disable apps/plugins/hooks/browser/computer-use |
| provider config | codebus 用 `~/.codebus/config.yaml` 選 active provider；不是直接讀 Claude/Codex user config |
| audit | 兩者都轉成 codebus neutral `StreamEvent`，但 usage 語意不同：Claude delta、Codex cumulative |

安全 swap 原則：

1. 不得把 prompt 指令當安全邊界。
2. 不得把 Claude tool allowlist 等同 Codex OS sandbox。
3. 不得把 Codex `-s` 等同 Claude native Windows sandbox。
4. 不得把 `--ignore-user-config` 等同完整 HOME 隔離。
5. 不得在 Windows native Codex 上宣稱 hard read isolation。
6. 不得把 Claude `check-read` denylist 等同 vault allowlist。
7. 不得把 `web_search=disabled` 等同 shell egress deny。
8. 不得讓父 shell 的 token env 直接繼承進 agent。

## 企業上線高標準

最低 gate：

| Gate | 要求 |
|---|---|
| 版本固定 | `claude --version`、`codex --version`、`codebus --version` 固定並記錄 |
| config root | `CODEBUS_HOME`、`CODEX_HOME` 使用 runner 專用目錄 |
| credential | service account / short-lived token；不得使用個人 OAuth/session |
| env scrub | agent child process 使用 env allowlist；不得繼承 `GITHUB_TOKEN`、`AWS_*`、`AZURE_*`、`KUBECONFIG` 等父 shell 機密 |
| init surface | Claude/Codex init/doctor/event 必須符合 allowlist |
| filesystem smoke | workspace 內讀寫、workspace 外讀寫、user home secret、sibling repo、`.git`、`.codebus` 全部測 |
| network smoke | deny-all、provider allowlist、MCP allowlist、loopback、HTTP/80、HTTPS/443 全部測 |
| raw mirror smoke | gitignored secret、UTF-16/非 UTF-8 secret、oversized file、nested `.git`、PII mask/skip/warn 全部測 |
| persistence smoke | session/history/cache/memory 不跨任務 |
| audit | 保存 JSONL、exit code、sandbox denial、changed paths |
| upgrade gate | 每次 CLI/codebus 升級重跑 help diff + smoke |

建議架構：

```text
enterprise orchestrator
  -> disposable workspace
  -> dedicated service account
  -> CODEBUS_HOME / CODEX_HOME / Claude managed settings
  -> env allowlist / secret broker
  -> codebus.exe --repo <source>
  -> provider-specific wrapper policy
  -> OS sandbox: Linux/WSL2 container, VM, or Windows AppContainer/ACL broker
  -> egress firewall/proxy
  -> codebus event log + side-effect verifier
  -> destroy workspace / revoke token
```

Windows 判斷：

| 路線 | 判斷 |
|---|---|
| codebus + Claude native Windows | 寫入 source repo 較受限；讀取不是 hard boundary |
| codebus + Codex native Windows unelevated | 可做弱寫入控制；不能擋讀 user home/sibling repo；network/ACL 例外需外部控管 |
| codebus + Codex native Windows elevated | 需受控 runner 重測；本機先前 elevated backend 未通過 smoke |
| codebus + Codex WSL2/Linux | 最適合進企業 runner 驗證 |
| codebus + Windows AppContainer wrapper | 值得後續 spike；需跑真實 codex.exe last-mile |

## 官方來源

| 主題 | 來源 |
|---|---|
| Claude Code settings / managed policy | <https://code.claude.com/docs/en/settings> |
| Claude Code sandbox platform support | <https://code.claude.com/docs/en/sandboxing> |
| Claude Code CLI reference | <https://docs.anthropic.com/en/docs/claude-code/cli-reference> |
| OpenAI Codex Windows sandbox design | <https://openai.com/index/building-codex-windows-sandbox/> |
| OpenAI Codex CLI | <https://developers.openai.com/codex/cli> |
| OpenAI Codex permissions | <https://developers.openai.com/codex/permissions> |
| OpenAI Codex managed configuration | <https://developers.openai.com/codex/enterprise/managed-configuration> |

## 本地原始碼依據

| 主題 | 來源 |
|---|---|
| Claude cwd | `codebus-core/src/agent/claude_cli.rs:136` |
| Claude toolset | `codebus-core/src/agent/claude_backend.rs:33-37`、`codebus-core/src/verb/goal.rs:58` |
| Claude PreToolUse settings | `codebus-core/src/vault/settings.rs:35-59` |
| Claude `check-read` denylist | `codebus-cli/src/commands/hook.rs:313-435` |
| Claude env injection without clear | `codebus-core/src/agent/claude_cli.rs:442`、`:511` |
| Codex actual argv | `codebus-core/src/agent/codex_backend.rs:93-192` |
| Codex azure key env injection without clear | `codebus-core/src/agent/codex_backend.rs:247` |
| Codex Windows sandbox observations | `D:\side_project\codebus\docs\security.md` §5 |
| raw mirror gitignore and non-UTF-8 | `codebus-core/src/vault/raw_sync.rs:128`、`:214`、`:305`、`:334` |

## 本機證據摘要

| 證據 | 結果 |
|---|---|
| `claude --version` | `2.1.159 (Claude Code)` |
| `codex --version` | `codex-cli 0.135.0` |
| `codebus.exe --repo D:\side_project\codebus --help` | CLI 可執行 |
| `codebus.exe --repo D:\side_project\codebus lint --format json` | `error_count=0,warn_count=0` |
| raw Claude root default init | hooks/MCP/skills/plugins/user memory 會載入 |
| raw Claude root isolated init | tools/MCP/slash/skills/plugins/hook 清為 0 |
| codebus `check-read` probe | `~/.kube/config`、`~/.docker/config.json`、`~/.env`、母 repo `Cargo.toml` 允許；`~/.ssh/config`、`*.pem` 擋下 |
| Claude PreToolUse matcher | `.codebus/.claude/settings.json` 只有 `Bash`、`Read`；無 `Glob/Grep` |
| codebus env inheritance probe | mock Claude child 看得到父 shell `GITHUB_TOKEN` 與 `AWS_ACCESS_KEY_ID` |
| raw_sync gitignored secret probe | `.gitignore` 內 `secrets/` 不進 raw mirror，也不被 PII scanner 掃描 |
| raw_sync UTF-16 secret probe | UTF-16LE 檔案 byte-identical 進 raw mirror；PII scanner 未掃 |
| Codex root `:workspace` read user home | `USER_AGENTS_READ_OK` |
| Codex root `:workspace` read sibling repo | `SIBLING_REPO_READ_OK` |
| Codex root `:read-only` write root | `ROOT_WRITE_BLOCKED` |
| Codex direct sandbox probes | 使用 `codex sandbox --permissions-profile`，標記為非 codebus 實際 argv |
| Codex codebus-style `-s workspace-write` HTTPS/443 | blocked |
| Codex codebus-style `-s workspace-write` HTTP/80 | allowed with `--noproxy '*'` |
| Codex codebus-style `-s workspace-write` loopback | allowed；listener hit |
| Codex codebus-style `-s workspace-write` normal ACL 外部寫 | blocked；`Access is denied` |
| Codex codebus-style `-s workspace-write` `%TEMP%` 外部寫 | allowed；檔案存在且含 marker |

## 待補

1. 將 codebus agent spawn 改為 env allowlist 或 `env_clear` 等效機制，只注入 provider 必要 env。
2. 將 Claude `check-read` 從 denylist 改為 vault allowlist，或至少替 `Glob/Grep` 增加同等 path gate。
3. 修 raw_sync：gitignored secret 要有企業可見報告；非 UTF-8 檔案不得 byte-identical 帶入 raw mirror 而不掃描。
4. 用專用 `CODEBUS_HOME` 跑 codebus mock provider end-to-end，確認 `.codebus` cwd、hooks、event log、changed paths。
5. 在 Windows 受控 runner 部署 Claude managed settings，驗證 codebus wrapper 不受 user/global source 污染。
6. 在 Windows 受控 runner 重測 Codex elevated sandbox 與新版 permission profiles，不與舊 `-s` route 混用。
7. 在 WSL2/Linux runner 跑 codebus + Codex end-to-end，驗證 permission profile、network deny、fake SSH/AWS/GCloud 讀取。
8. 評估 codebus Codex backend 是否要從舊 `-s` route migration 到 managed permission profile route。
