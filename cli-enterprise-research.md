# Claude Code CLI 與 Codex CLI 企業內部承載研究

研究日期：2026-06-01  
工作區：`D:\side_project\agent-study`  
原則：本文件以本機 CLI 實測為主，官方文件為輔。若本機版本與文件不一致，以本機版本作為直接承載應用時的決策依據。

證據標記：

| 標記 | 意義 |
|---|---|
| 本機實測 | 已在目前 Windows 工作站執行並觀察 stdout/stderr、init event、檔案狀態或 exit code |
| 官方確認 | 已對照官方文件，但未必在本機完成部署驗證 |
| 官方 schema 對齊 | 設定鍵與值型別已對照官方文件；尚未把 managed policy 實際部署到系統層驗證 |
| 待目標環境驗證 | 需要搬到 Linux/WSL2/企業 runner/managed policy 環境後才能確認 |
| 推論 | 由實測與官方語意推導，不能視為已驗證安全邊界 |

## 1. 研究目的

目標是判斷未來內部應用是否能直接架設在 `claude` 與 `codex` CLI 上，並定義安全、可審計、可替換的呼叫方式。

核心問題：

1. 兩個 CLI 的互動與非互動行為。
2. flag、config、env var、權限與 sandbox 的實際效果。
3. Claude Code CLI 與 Codex CLI 互相 swap 時的語意落差。
4. 企業內部使用時的安全預設、封鎖面、審計面、wrapper 設計。
5. 哪些行為不能相信模型文字回覆，必須由 wrapper 以檔案狀態、exit code、事件流重新驗證。

## 2. 本機實測環境

| 項目 | Claude Code CLI | Codex CLI |
|---|---:|---:|
| 指令 | `claude.exe` | `codex.ps1` / native `codex.exe` |
| 版本 | `2.1.159 (Claude Code)` | `codex-cli 0.135.0` |
| 安裝來源 | `%USERPROFILE%\.local\bin\claude.exe` | npm package `@openai/codex` |
| 本機 auth | 已登入 claude.ai，帳號資訊已省略 | 已登入 ChatGPT |
| 本機 user config 狀態 | `~/.claude/settings.json` 存在，含 hooks、plugins、model、permission 設定 | `~/.codex/config.toml` 存在，含 model、approval、sandbox、MCP、plugins 設定 |
| 重要風險 | 預設會讀使用者 hooks/plugins/MCP/skills | 使用者 config 目前為 `sandbox_mode = "danger-full-access"`、`approval_policy = "never"` |

本機狀態不是企業安全預設。企業 wrapper 不得直接繼承一般使用者 home 目錄中的設定。

## 3. 本機 CLI 實測摘要

### 3.1 版本與 help

| 驗證命令 | 結果 |
|---|---|
| `claude --version` | exit `0`，輸出 `2.1.159 (Claude Code)` |
| `claude --help` | 預設互動 session；`-p/--print` 才是非互動輸出 |
| `codex --version` | exit `0`，輸出 `codex-cli 0.135.0` |
| `codex --help` | 無 subcommand 時進互動 CLI；`codex exec` 是非互動模式 |
| `codex exec --help` | 支援 `--json`、`--ephemeral`、`--ignore-user-config`、`--ignore-rules`、`--sandbox` |
| `codex exec -a never ...` | exit `1`，`unexpected argument '-a'` |
| `codex exec --ask-for-approval never ...` | exit `1`，`unexpected argument '--ask-for-approval'` |

結論：Codex root help 顯示的 `-a/--ask-for-approval` 不可直接套到本機 `codex exec`。wrapper 必須以目標 subcommand 的 help 為準，不得只讀 root help。

### 3.2 Claude Code 非互動輸出

| 驗證命令 | 結果 |
|---|---|
| `claude -p --tools "" --output-format json ...` | 因 PowerShell 空字串與 variadic flag 行為，後續 flags 可能未如預期套用；曾得到純文字輸出 |
| `claude --print --model sonnet --effort low --output-format json ...` | exit `0`，輸出單一 JSON result |
| `claude --print --output-format stream-json ...` | exit `1`，錯誤：`--output-format=stream-json requires --verbose` |
| `claude --print --verbose --output-format stream-json ...` | exit `0`，輸出 system/init、assistant、result 等 JSONL 事件 |
| `claude --bare --print ...` | exit `1`，JSON result 內 `is_error=true`，內容 `Not logged in · Please run /login` |

Claude 的 `--bare` 在本機 claude.ai OAuth 登入下不可直接使用；它不讀 keychain/OAuth，需 `ANTHROPIC_API_KEY` 或 `--settings` 中的 `apiKeyHelper` 類設定。

PowerShell 下要把工具清空，實測可用：

```powershell
"Respond exactly: NOTOOLS_INIT" |
  claude --print `
    --model sonnet `
    --effort low `
    --verbose `
    --output-format stream-json `
    --no-session-persistence `
    --setting-sources project,local `
    --disable-slash-commands `
    --strict-mcp-config `
    --mcp-config .\empty-mcp.json `
    --max-budget-usd 0.25 `
    --tools=
```

其中 `empty-mcp.json`：

```json
{"mcpServers":{}}
```

實測 init 事件：

```json
{
  "tools": [],
  "mcp_servers": [],
  "slash_commands": [],
  "skills": [],
  "plugins": []
}
```

### 3.3 Claude Code 工具權限實測

| 場景 | 命令核心 | 實測結果 |
|---|---|---|
| 無工具可用 | `--tools=` | init 顯示 `tools: []`，要求寫檔時檔案不存在 |
| 允許 Write | `--tools=Write --allowedTools=Write` | CLI 真的建立檔案，stream-json 出現 `tool_use` 與 `tool_result` |
| Write 被 disallow | `--tools=Write --disallowedTools=Write` | init 顯示 `tools: []`，檔案未建立；但模型最後文字偽造 `<function_calls>` 並聲稱成功 |

關鍵結論：不能信任最終文字中的成功聲明。wrapper 必須以結構化 tool event、檔案系統實際狀態、git diff、exit code 作為事實來源。

### 3.4 Claude Code 設定隔離實測

| 命令 | 實測效果 |
|---|---|
| 不加隔離 flags 的 `stream-json --verbose` | 載入使用者 hooks、plugins、MCP、skills，hook 內容也進入輸出事件 |
| `--setting-sources project,local` | 使用者 plugins/hooks 不再載入，但仍有內建 tools、內建 claude.ai connectors、skills、memory path |
| `--disable-slash-commands` | slash commands / skills 可清空 |
| `--strict-mcp-config --mcp-config empty-mcp.json` | MCP servers 可清空 |
| `--tools=` | tool surface 可清空 |

最低干擾模式不是單一 flag，而是多個 flag 疊加。

### 3.5 Codex 非互動輸出

| 驗證命令 | 結果 |
|---|---|
| `codex exec --ephemeral --ignore-rules --skip-git-repo-check --json -s read-only ...` | exit `0`，stdout 為 JSONL |
| 同命令用 .NET Process 分離 stdout/stderr | JSONL 在 stdout；`Reading additional input from stdin...` 在 stderr |
| `codex exec --ephemeral --ignore-user-config ...` | auth 仍可用，因 help 明示 auth 仍使用 `CODEX_HOME` |

Codex `--json` 可以由 stdout 逐行解析；stderr 仍可能有非 JSON 診斷訊息。wrapper 必須分流 stdout/stderr，不得把合併輸出當 JSONL。

### 3.6 Codex sandbox 實測

| 場景 | 命令核心 | 實測結果 |
|---|---|---|
| read-only，要求 shell `Get-Location` | `-s read-only` | `command_execution` 失敗：`windows sandbox: spawn setup refresh` |
| workspace-write，要求 shell `Get-Location` | `-s workspace-write` | 同樣失敗：`windows sandbox: spawn setup refresh` |
| danger-full-access，要求 shell `Get-Location` | `-s danger-full-access` | shell 執行成功，exit code `0` |
| read-only，要求建立檔案 | `-s read-only` | file patch 被拒，檔案不存在 |
| workspace-write，要求建立工作區檔案 | `-s workspace-write` | file_change 成功，檔案存在且內容正確 |
| `codex sandbox --permissions-profile read-only ...` | direct sandbox 子命令 | exit `1`，`default_permissions requires a [permissions] table` |
| `codex sandbox --permissions-profile ':read-only' ...` | direct sandbox 子命令 | exit `1`，`windows sandbox failed: spawn setup refresh` |
| `codex sandbox --permissions-profile ':workspace-write' ...` | direct sandbox 子命令 | exit `1`，`unknown built-in profile`；本版 built-in 名稱不是 `:workspace-write` |
| `codex sandbox --permissions-profile ':danger-full-access' ...` | direct sandbox 子命令 | exit `1`，`only managed permission profiles can be enforced by the Windows sandbox` |

本機 Codex 0.135.0 在 Windows sandbox 下，read-only/workspace-write 的 shell 啟動失敗，但 workspace-write 的 file_change 可以成功。這代表：

1. Windows sandbox 不能在本機直接視為已通過企業安全驗證。
2. `danger-full-access` 能執行 shell，但失去 sandbox。
3. 若企業應用需要 shell，應優先用外部 VM/container/runner 隔離，再決定是否給 CLI `danger-full-access`。
4. 若只需要受控檔案改寫，可用 `workspace-write`，但仍需 wrapper 驗證實際 diff。
5. Codex permission profile 與舊 sandbox 模式同時存在時會產生取捨；本機 `codex sandbox` 對 built-in profile 的行為與文件語意不完全一致，企業部署必須在目標 runner 上重跑 smoke test。

### 3.7 workspace 外讀寫邊界實測

補測隔離目錄：

```text
D:\side_project\agent-study\.tmp_boundary_probe\workspace
D:\side_project\agent-study\.tmp_boundary_probe\outside
```

`outside\outside_read.txt` 內容為：

```text
OUTSIDE_BOUNDARY_SECRET
```

Codex 實測：

| 場景 | 命令核心 | 結果 |
|---|---|---|
| workspace 內寫入 | `codex exec -s workspace-write` | 成功建立 `workspace\inside_write_codex_noignore.txt` |
| workspace 外寫入 | `codex exec -s workspace-write` | 被拒，回報可寫範圍僅含 workspace，目標檔不存在 |
| workspace 外寫入 + `--add-dir outside` | `codex exec -s workspace-write --add-dir outside` | 成功建立 `outside\outside_write_codex_adddir_noignore.txt` |
| workspace 外讀取 | `codex exec --ignore-user-config -s workspace-write` | 成功讀出 `OUTSIDE_BOUNDARY_SECRET` |
| workspace 外讀取 | `codex exec --ignore-user-config -s read-only` | 成功讀出 `OUTSIDE_BOUNDARY_SECRET` |
| workspace 外寫入 | `codex exec --ignore-user-config -s workspace-write` | 未建立檔案；模型回報 read-only / approval never |

Codex 結論：`workspace-write` 是寫入邊界，不是讀取邊界。本機實測證明 `read-only` 與 `workspace-write` 仍可透過 shell 讀 workspace 外檔案。`--add-dir` 會擴大可寫邊界。

Claude 實測：

| 場景 | 命令核心 | 結果 |
|---|---|---|
| workspace 外讀取 | `claude --tools=Read --allowedTools=Read --permission-mode dontAsk` | 成功讀出 `OUTSIDE_BOUNDARY_SECRET` |
| workspace 外寫入 | `claude --tools=Write --allowedTools=Write --permission-mode dontAsk` | 成功建立 `outside\outside_write_claude.txt` |

Claude 結論：`Read` / `Write` 工具本身不以目前 cwd 作硬性 workspace 邊界。沒有 managed deny、sandbox filesystem policy 或外部 OS 隔離時，允許 `Read` / `Write` 等同允許模型操作絕對路徑。

企業結論：不能把 CLI 的「workspace」語意視為完整資料邊界。workspace 邊界必須由外部 runner、OS ACL、container mount、唯讀 bind mount、managed denyRead/denyWrite、wrapper path verifier 共同形成。

### 3.8 Codex 0.135.0 新版補測

補測時間：2026-06-01。`npm view @openai/codex version` 與 `codex doctor --json` 均顯示目前最新版本為 `0.135.0`。本機實際執行檔是 npm 安裝版，Windows app 版也存在於 PATH，但目前 shell 呼叫的是 npm 版。

官方文件確認：

1. Permission profiles 是 beta。
2. `default_permissions` / `[permissions.<name>]` 不應與舊 `sandbox_mode`、`sandbox_workspace_write.*`、CLI `--sandbox` 混用。
3. built-in profile 是 `:read-only`、`:workspace`、`:danger-full-access`。
4. Windows native sandbox 首選 `elevated`，fallback 是 `unelevated`；`unelevated` 網路隔離較弱。
5. `requirements.toml` 可鎖 `allowed_sandbox_modes`、`allowed_approval_policies`、web search、MCP allowlist、feature flags、network requirements 與 deny-read。
6. 官方明寫 Windows managed `deny_read` 只套用 direct file tools；shell subprocess reads 不走這條 sandbox rule。

本機關鍵狀態：

| 項目 | 結果 |
|---|---|
| `codex --version` | `codex-cli 0.135.0` |
| `npm view @openai/codex version` | `0.135.0` |
| 使用者 config | `approval_policy = "never"`、`sandbox_mode = "danger-full-access"`、`[windows] sandbox = "elevated"` |
| 系統 requirements | `%ProgramData%\OpenAI\Codex\requirements.toml` 不存在 |
| managed config | `%USERPROFILE%\.codex\managed_config.toml` 不存在；官方 Windows/non-Unix 路徑是 `~/.codex/managed_config.toml` |
| `features.network_proxy` | experimental，未啟用 |

新版 permission profile 實測：

| 測試 | 結果 |
|---|---|
| `codex sandbox --permissions-profile ':read-only'` | exit `1`，`windows sandbox failed: spawn setup refresh` |
| `codex sandbox --permissions-profile ':workspace'` | exit `1`，`windows sandbox failed: spawn setup refresh` |
| `codex sandbox --permissions-profile ':danger-full-access'` | exit `1`，`only managed permission profiles can be enforced by the Windows sandbox` |
| 隔離 `CODEX_HOME` + 自訂 `[permissions.enterprise-test]` | config parse ok，`doctor` 顯示 filesystem/network restricted |
| 自訂 profile direct sandbox | exit `1`，`Restricted read-only access requires the elevated Windows sandbox backend` |
| 改 `[windows] sandbox = "unelevated"` 後 direct sandbox | 仍 exit `1`，同樣要求 elevated Windows sandbox backend |
| `codex exec -c 'default_permissions=":read-only"'`，要求 workspace 內寫檔 | 被 policy block，檔案未建立 |
| `codex exec -c 'default_permissions=":workspace"'`，要求 workspace 外寫檔 | shell 啟動失敗：`windows sandbox: spawn setup refresh`，檔案未建立 |

Trust / config isolation 補測：

| 測試 | 結果 |
|---|---|
| 不帶 `--ignore-user-config`、不帶 sandbox flag | header 顯示 `sandbox: danger-full-access`，原因是使用者 config 已設定 |
| 不帶 `--ignore-user-config`、加 `-s workspace-write` | header 顯示 `sandbox: workspace-write` |
| 帶 `--ignore-user-config`、加 `-s workspace-write` | header 仍顯示 `sandbox: read-only` |
| 非 Git 目錄加 `--skip-git-repo-check` | 仍偏向 `read-only`，即使傳 `-s workspace-write` |

新版結論：

1. Codex 不是沒有控制方式；0.135.0 官方控制面比舊版更完整，核心是 permission profiles、requirements、managed defaults、Windows sandbox backend。
2. 本機 Windows native sandbox 尚未通過可用性 smoke test；`spawn setup refresh` 代表不能把此 runner 當企業安全執行面。
3. `--ignore-user-config` 會隔離使用者 config，但也會移除本機 trust 狀態；自動化若需要 workspace-write，必須用受控 runner 建立可信任工作區與 managed policy，而不是只靠 CLI flag。
4. 使用者層 `sandbox_mode = "danger-full-access"` 會讓無 flag 執行直接落到 full access；企業部署不可使用一般使用者 `CODEX_HOME`。
5. Windows 上若要用 deny-read 控制 secret，不能只依賴 managed `deny_read`，因為官方說 shell subprocess reads 不走這條規則；必須外加 OS ACL、container/VM mount、DLP/proxy、wrapper path verifier。
6. 企業標準路線是：專用 `CODEX_HOME` + system/cloud `requirements.toml` + managed defaults + elevated Windows sandbox smoke test；若 elevated 不穩，改 WSL2/Linux runner 或外部 VM/container。

### 3.9 Codex Linux / WSL 官方行為

官方文件對 Linux / WSL 在新版控制面有明確描述：

1. Linux sandbox 使用 `bwrap` 加 `seccomp`。
2. Windows 上的 WSL2 使用 Linux sandbox implementation；WSL1 只支援到 Codex `0.114`，從 Codex `0.115` 起 Linux sandbox 改為 `bwrap`，所以 WSL1 不再支援。
3. `codex sandbox linux [--permissions-profile <name>] [COMMAND]...` 是官方列出的本機 sandbox 測試入口。
4. permission profiles 的 deny-read glob 預展開行為明確適用於 Linux、WSL、native Windows；`glob_scan_max_depth` 用於限制啟動前掃描深度。
5. permission profiles 的 direct write rule 若 active runtime 無法 enforce，Codex 會拒絕該 rule。
6. network profile 的 `enabled = true` 只改 sandbox network policy，不會自動啟動 network proxy；domain allow/deny 邏輯同樣適用於 sandboxed networking。
7. Linux/Unix 的 system requirements 路徑是 `/etc/codex/requirements.toml`，managed defaults 路徑是 `/etc/codex/managed_config.toml`。

Linux / WSL 的企業判斷：

1. 官方語意上，Linux/WSL 比 native Windows 更適合做新版 permission profiles 的標準 runner，因為官方直接指定 `bwrap + seccomp` 作 sandbox backend。
2. 仍必須實測，不可只靠文件：`codex sandbox linux --permissions-profile ':read-only' ...`、`:workspace`、自訂 profile、workspace 外讀寫、network allowlist 都要在目標 distro/kernel 上跑。
3. WSL2 可視為 Linux sandbox 路線；WSL1 不應進入企業支援矩陣。
4. 若企業要嚴格阻擋 secret，被測 profile 必須同時驗證 direct file tools 與 shell subprocess reads；Linux 文件沒有 Windows 那條「managed deny_read 只套 direct file tools」的限制，但仍要用實測確認。

### 3.10 Linux 搬遷後實測計畫

Linux 上先跑 direct sandbox，再跑 `codex exec`。原因是 direct sandbox 不需要模型與 auth，可單獨驗證 OS sandbox backend；`codex exec` 才驗證 agent wrapper、prompt、tool routing、session、auth 與 config layering。

前置檢查：

```bash
set -euo pipefail

codex --version
codex --help
codex exec --help
codex sandbox --help
codex doctor --json
codex features list

command -v bwrap
uname -a
cat /etc/os-release
```

建立隔離測試資料：

```bash
set -euo pipefail

ROOT="$(pwd)/.tmp_codex_linux_probe"
WORKSPACE="$ROOT/workspace"
OUTSIDE="$ROOT/outside"
CODEX_HOME_TEST="$ROOT/codex-home"

test ! -e "$ROOT" || { printf 'probe root already exists: %s\n' "$ROOT" >&2; exit 1; }
mkdir -p "$WORKSPACE" "$OUTSIDE" "$CODEX_HOME_TEST"

printf 'INSIDE_BOUNDARY_DATA\n' > "$WORKSPACE/inside_read.txt"
printf 'OUTSIDE_BOUNDARY_SECRET\n' > "$OUTSIDE/outside_read.txt"
printf 'ENV_SECRET\n' > "$WORKSPACE/.env"
```

最小企業 profile：

```bash
cat > "$CODEX_HOME_TEST/config.toml" <<EOF
default_permissions = "enterprise-linux"

[permissions.enterprise-linux]
extends = ":workspace"

[permissions.enterprise-linux.filesystem]
":minimal" = "read"
"$OUTSIDE" = "deny"

[permissions.enterprise-linux.filesystem.":workspace_roots"]
"." = "write"
".env" = "deny"
".ssh" = "deny"
".aws" = "deny"
".config/gcloud" = "deny"

[permissions.enterprise-linux.network]
enabled = false
EOF
```

注意：本機 Linux `codex-cli 0.135.0` 實測中，`"**/*.env"` 會觸發 Linux glob expansion warning，需要設定 `glob_scan_max_depth` 或改列明確深度。將 `.git` / `.codex` 直接列入 deny 時，direct sandbox 會在建立 protected workspace metadata path 階段失敗；這兩個 metadata path deny 需另用目標 managed profile 驗證，不應混入六項 smoke test。

Direct sandbox smoke test：

```bash
export CODEX_HOME="$CODEX_HOME_TEST"

expect_success() {
  name="$1"
  shift
  if "$@"; then
    printf 'PASS expected success: %s\n' "$name"
  else
    printf 'FAIL expected success: %s\n' "$name"
    exit 1
  fi
}

expect_fail() {
  name="$1"
  shift
  if "$@"; then
    printf 'FAIL expected block: %s\n' "$name"
    exit 1
  else
    printf 'PASS expected block: %s\n' "$name"
  fi
}

expect_success "workspace read" \
  codex sandbox --permissions-profile enterprise-linux -C "$WORKSPACE" \
  bash -lc 'cat inside_read.txt'

expect_success "workspace write" \
  codex sandbox --permissions-profile enterprise-linux -C "$WORKSPACE" \
  bash -lc 'printf OK > inside_write.txt && cat inside_write.txt'

expect_fail "outside read" \
  codex sandbox --permissions-profile enterprise-linux -C "$WORKSPACE" \
  bash -lc "cat '$OUTSIDE/outside_read.txt'"

expect_fail "outside write" \
  codex sandbox --permissions-profile enterprise-linux -C "$WORKSPACE" \
  bash -lc "printf BAD > '$OUTSIDE/outside_write.txt'"

expect_fail ".env read" \
  codex sandbox --permissions-profile enterprise-linux -C "$WORKSPACE" \
  bash -lc 'cat .env'

expect_fail "fake SSH key read" \
  codex sandbox --permissions-profile enterprise-linux -C "$WORKSPACE" \
  bash -lc 'cat .ssh/id_rsa'

expect_fail "fake AWS credentials read" \
  codex sandbox --permissions-profile enterprise-linux -C "$WORKSPACE" \
  bash -lc 'cat .aws/credentials'

expect_fail "fake GCloud ADC read" \
  codex sandbox --permissions-profile enterprise-linux -C "$WORKSPACE" \
  bash -lc 'cat .config/gcloud/application_default_credentials.json'

expect_fail "network egress" \
  codex sandbox --permissions-profile enterprise-linux -C "$WORKSPACE" \
  bash -lc 'curl -fsS https://example.com >/tmp/codex_net_probe'
```

預期結果：

| 測試 | 預期 |
|---|---|
| workspace 內讀取 | 成功 |
| workspace 內寫入 | 成功 |
| workspace 外讀取 | 失敗 |
| workspace 外寫入 | 失敗 |
| `.env` 讀取 | 失敗 |
| fake SSH / AWS / GCloud config 讀取 | 失敗 |
| network egress | 失敗 |

`codex exec` smoke test：

```bash
export CODEX_HOME="$CODEX_HOME_TEST"

codex exec \
  --strict-config \
  --ephemeral \
  -C "$WORKSPACE" \
  -c 'default_permissions="enterprise-linux"' \
  -o "$ROOT/exec_read_inside.txt" \
  "Use shell exactly once to run: cat inside_read.txt. Then report the exact output."

codex exec \
  --strict-config \
  --ephemeral \
  -C "$WORKSPACE" \
  -c 'default_permissions="enterprise-linux"' \
  -o "$ROOT/exec_read_outside.txt" \
  "Use shell exactly once to run: cat '$OUTSIDE/outside_read.txt'. Then report whether it was blocked."
```

`codex exec` 使用隔離 `CODEX_HOME` 時需要該 home 具備可用 auth。若只要先測 sandbox backend，先跑 direct sandbox；不要為了通過 `exec` 測試改用一般使用者 `~/.codex`。

通過門檻：

1. `codex doctor --json` 顯示 config parse ok，sandbox helper 可用。
2. direct sandbox 六項 smoke test 全部符合預期。
3. `codex exec` header 顯示使用目標 profile / sandbox，不得落到 `danger-full-access`。
4. `codex exec` 的實際檔案結果與 direct sandbox 一致。
5. network disabled 時，shell subprocess 無法出站。
6. `.env`、SSH、cloud config、workspace 外敏感檔不可由 shell subprocess 讀取。

### 3.11 Linux 目標環境本機實測（2026-06-01）

本次在 Linux runner 實測，不是 WSL2。後續補測包含暫時部署 `/etc/codex` system-level managed policy；測完已恢復原本不存在 `/etc/codex` 的狀態。

| 項目 | 結果 |
|---|---|
| repo root | `/home/asus/agent-cli-study`，存在 `cli-enterprise-research.md` |
| distro / kernel | Ubuntu 22.04.5 LTS，kernel `6.8.0-111-generic` |
| `bwrap` | `/usr/bin/bwrap` |
| Codex | `codex-cli 0.135.0`，standalone linux-x86_64 |
| Claude Code | `2.1.159 (Claude Code)` |
| Codex MCP | `codex mcp list --json` 輸出 `[]` |
| Claude MCP / plugin | `claude mcp list` 顯示 `plugin:figma:figma` connected；`claude plugin list` 顯示 user scope `figma@claude-plugins-official` enabled |

Codex direct sandbox 使用專用 `CODEX_HOME=/home/asus/agent-cli-study/.tmp_linux_cli_probe/codex-home`。本機 `codex sandbox --help` 顯示 Linux sandbox 沒有 `linux` subcommand；實測可用命令是 `codex sandbox --permissions-profile enterprise-linux -C "$WORKSPACE" bash -lc ...`。

| 測試 | exit / stdout / stderr | 判定 |
|---|---|---|
| workspace 內讀 `cat inside_read.txt` | exit `0`，stdout `INSIDE_BOUNDARY_DATA` | 通過 |
| workspace 內寫 `inside_write.txt` | exit `0`，stdout `OK`，檔案存在 | 通過 |
| workspace 外讀 `outside/outside_read.txt` | exit `1`，`Permission denied` | 通過 |
| workspace 外寫 `outside/outside_write.txt` | exit `1`，`Permission denied`，檔案未建立 | 通過 |
| `.env` 讀取 | exit `1`，`cat: .env: Permission denied` | 通過 |
| fake SSH key `.ssh/id_rsa` 讀取 | exit `1`，`cat: .ssh/id_rsa: Permission denied` | 通過 |
| fake AWS credentials `.aws/credentials` 讀取 | exit `1`，`cat: .aws/credentials: Permission denied` | 通過 |
| fake GCloud ADC `.config/gcloud/application_default_credentials.json` 讀取 | exit `1`，`Permission denied` | 通過 |
| network egress | exit `6`，`curl: (6) Could not resolve host: example.com` | 通過 |

直接套用原 smoke profile 時的差異：

| profile 項目 | 本機結果 | 後續要求 |
|---|---|---|
| `"**/*.env" = "deny"` | `codex doctor --json` config parse ok，但警告 Linux non-macOS sandbox 不支援 unbounded `**` native expansion | enterprise profile 應設定 `glob_scan_max_depth` 或列舉明確深度 |
| `".codex" = "deny"` | direct sandbox 啟動失敗：`bwrap: Can't create file at .../workspace/.codex: Is a directory` | 需在目標 managed profile 另測 metadata path deny |
| `".git" = "deny"` | direct sandbox 啟動失敗：`sandbox blocked creation of protected workspace metadata path .../workspace/.git` | 需在目標 managed profile 另測 metadata path deny |

最初使用無 auth 的專用 `CODEX_HOME` 時，`codex exec` 因缺少 credentials 回 `401 Unauthorized`，未進入 shell tool 測試。後續使用另一個專用 `CODEX_HOME=/home/asus/agent-cli-study/.tmp_linux_cli_probe/exec-auth-home` 完成測試帳號登入後，`codex doctor --json` overall `ok`，顯示 `auth is configured`、`config.toml parse: ok`、sandbox helpers 為 restricted filesystem / restricted network。

Codex `exec` agent-level profile 補測：

| 測試 | JSON event / 檔案狀態 | 判定 |
|---|---|---|
| workspace 內讀 | `command_execution` 執行 `/bin/bash -lc 'cat inside_read.txt'`，exit `0`，stdout `INSIDE_BOUNDARY_DATA` | 通過 |
| agent boundary script | `command_execution` 執行 `bash agent_boundary_probe.sh`，stdout 顯示 `inside=INSIDE_BOUNDARY_DATA`，`.env`、outside、fake SSH/AWS/GCloud 全部 status `1` 且 `Permission denied` | 通過 |
| agent network script | `command_execution` 執行 `bash agent_network_probe.sh`，stdout `network_status=6`，stderr `curl: (6) Could not resolve host: example.com` | 通過 |
| agent outside write | `agent_inside_write.txt` 實際建立且內容 `AGENT_OK`；`outside/agent_outside_write.txt` 未建立 | 通過 |

注意：直接 prompt 要求 `cat .env` 或讀 outside path 時，模型 final text 可能回報 `Permission denied`，但 JSON stream 沒有 `command_execution` event。這不能作為事實來源。因此 agent-level blocked boundary 採用 workspace 內 probe script，由 event 中的 `command_execution` 與實際檔案狀態驗證。

System-level managed policy 補測：

| 項目 | 結果 |
|---|---|
| 初始狀態 | `/etc/codex` 不存在 |
| 暫時部署 | root-owned `/etc/codex/requirements.toml` 與 `/etc/codex/managed_config.toml` |
| `requirements.toml` 載入證據 | `codex doctor --json` / `codex exec --json` 產生 startup event，明確標示 `set by /etc/codex/requirements.toml`，並把 disallowed `approval_policy=Never` fallback 到 `OnRequest`、把 disallowed `web_search_mode=Cached` fallback 到 `Disabled` |
| `managed_config.toml` profile 證據 | 只用空專用 `CODEX_HOME=.tmp_linux_cli_probe/system-managed-home`，`codex sandbox --permissions-profile enterprise-system ...` 可解析 `/etc/codex/managed_config.toml` 中的 profile |
| managed profile direct sandbox | workspace 內讀成功；workspace 外讀、`.env`、fake SSH/AWS/GCloud config、network egress 失敗 |
| managed profile `codex exec` | 暫時部署 `/etc/codex` 後，用已登入的專用 `CODEX_HOME` 加 `--ignore-user-config` 跑 `codex exec`；event 顯示 requirements fallback 來源為 `/etc/codex/requirements.toml`，並有 `command_execution` 驗證 boundary script 與 network script |
| direct sandbox default 行為 | `codex sandbox` 不帶 `--permissions-profile` 直接 exit `2`，要求提供 profile；direct sandbox 不使用 `default_permissions` 自動選 profile |
| `:danger-full-access` caveat | `codex sandbox --permissions-profile ':danger-full-access'` 仍可執行 harmless `pwd`，即使加 `--include-managed-config` 也未被 `allowed_permissions` 擋下 |
| 恢復狀態 | 補測後移除 `/etc/codex`，恢復此機原本無 system policy 的狀態 |

企業判斷：`requirements.toml` 對 `codex doctor` / `codex exec` config load 有實測 enforcement 證據；`managed_config.toml` 中的 permission profile 可被 direct sandbox 與 `codex exec` agent shell enforce。`codex exec` JSON event 未提供明確 active profile header，但 blocked reads/writes/network 的實測行為證明沒有落到 `danger-full-access`。不可把 direct `codex sandbox` 當作 requirements compliance gate，因為本機 `0.135.0` 對 explicit `:danger-full-access` 未被 `allowed_permissions` 擋下。

Codex local-only 測試未完成。`command -v ollama` 與 `command -v lms` 都無結果，`ollama list` 是 command not found；`codex exec --oss --local-provider ollama ...` exit `1`，回報 `No running Ollama server detected`。本機也未配置 deny-all outbound / process network audit，因此不能把 local-only 視為已驗證。

Claude Code Linux init 比較：

| 模式 | init event 重點 | 結果 |
|---|---|---|
| default + `--tools=` | `tools=[]`，但 `mcp_servers=[plugin:figma:figma]`、`plugins=[figma]`、多個 slash commands / skills，`memory_paths.auto=/home/asus/.claude/projects/.../memory/`，`analytics_disabled=false`，`product_feedback_disabled=false` | init surface 可觀察；API auth 之後 `401 Invalid authentication credentials` |
| isolated：`--setting-sources project,local --disable-slash-commands --strict-mcp-config --mcp-config empty --tools=` | `tools=[]`、`mcp_servers=[]`、`slash_commands=[]`、`skills=[]`、`plugins=[]`；仍有 `memory_paths.auto=/home/asus/.claude/projects/.../memory/` | 隔離 flags 可清空工具/MCP/skill/plugin surface；API auth 仍 `401` |
| `--bare --strict-mcp-config --mcp-config empty --tools=` | `tools=[]`、`mcp_servers=[]`，但 init event 仍列出 slash commands / skills / `plugins=[figma]`；無 `memory_paths`；結果 `Not logged in · Please run /login` | `--bare` 不讀 OAuth/keychain；本機無 API key 時不可完成推理，且不應單獨視為 plugin/skill 清空證據 |

### 3.12 新版控制面啟用後的舊資訊處理

官方規則：permission profiles 不和舊 sandbox settings 組合使用。只要任何 active config layer 出現 `sandbox_mode`、`sandbox_workspace_write`，或啟動時傳入 CLI `--sandbox`，Codex 會使用舊 sandbox settings，而不是 `default_permissions`。

因此，使用新版控制面時，舊資訊必須處理，不可保留在 active config 中。

Windows 0.135.0 補測：

| 測試 | 結果 | 判讀 |
|---|---|---|
| 使用目前 user config，無 flag | header `sandbox: danger-full-access`，shell 寫入成功 | 舊 `sandbox_mode = "danger-full-access"` 會污染無 flag 執行 |
| `--ignore-user-config -c 'sandbox_mode="danger-full-access"'` | header `sandbox: danger-full-access`，shell 寫入成功 | 舊 key 單獨存在時仍有效 |
| `-c 'sandbox_mode="danger-full-access"' -c 'default_permissions=":read-only"'` | header `sandbox: read-only`，shell 寫入被 policy block | CLI override 混用時，本機 0.135.0 實作不應被當成穩定安全契約 |
| `-c 'default_permissions=":read-only"' -c 'sandbox_mode="danger-full-access"'` | header `sandbox: read-only`，shell 寫入被 policy block | 即使反向順序，仍未落到 danger；但官方仍禁止組合使用 |

Windows 補測結論：舊資訊必須處理。雖然本機 CLI `-c default_permissions` 在混用時壓過了 `sandbox_mode`，但官方文件要求不要組合，且無 flag 執行已證明會吃到使用者層舊 `danger-full-access`。企業判定應採 fail-closed：只要 active config 或 wrapper 同時出現新舊控制面，即判定 runner 不合格。

必須移除或隔離：

| 舊資訊 | 處理 |
|---|---|
| `sandbox_mode = "read-only"` | 移除；改用 `default_permissions = ":read-only"` 或自訂 profile |
| `sandbox_mode = "workspace-write"` | 移除；改用 `default_permissions = ":workspace"` 或自訂 profile |
| `sandbox_mode = "danger-full-access"` | 移除；不得存在企業 runner active config |
| `[sandbox_workspace_write]` | 移除；改寫到 `[permissions.<name>.filesystem]` / `[permissions.<name>.network]` |
| `sandbox_workspace_write.network_access = true` | 移除；改用 `[permissions.<name>.network] enabled = true` 與 domain allowlist |
| CLI `--sandbox` / `-s` | 新版 profile 路線禁用；改由 config / managed config 選 `default_permissions` |
| wrapper 內舊 `--full-auto` / `--yolo` 相容旗標 | 禁用；`--yolo` 等同無 sandbox / 無 approvals |
| user `~/.codex/config.toml` 中 sandbox 舊 key | 不載入；使用專用 `CODEX_HOME` |
| project `.codex/config.toml` 中 sandbox 舊 key | 不信任；企業 runner 不應載入未審核 project-local config |

需要保留但改用途：

| 舊資訊 | 新用途 |
|---|---|
| `allowed_sandbox_modes` | 仍屬 `requirements.toml` 的舊 sandbox mode 限制；若主控改成 permission profiles，不可把它當作 profile allowlist |
| `[permissions.filesystem] deny_read` in `requirements.toml` | 可保留，這是 admin-enforced deny-read，不是舊 sandbox key |
| `[rules]` in `requirements.toml` | 可保留，用於強制 `prompt` / `forbidden` command rules |
| `mcp_servers` allowlist | 可保留，用 identity 鎖 MCP |
| `allow_managed_hooks_only` | 可保留，用於排除 user/project/session/plugin hooks |
| `[experimental_network]` in `requirements.toml` | 可保留，作管理層 network requirements |

遷移檢查命令：

```bash
grep -RInE 'sandbox_mode|sandbox_workspace_write|--sandbox|full-auto|yolo' \
  /etc/codex "$CODEX_HOME" .codex 2>/dev/null || true

codex doctor --json
codex exec --strict-config --ephemeral -C "$WORKSPACE" \
  -o "$ROOT/profile_mode_check.txt" \
  "Do not use tools. Report only the sandbox or permission profile shown in the session header."
```

判定標準：

1. 生產 wrapper 不傳 `--sandbox`。
2. active `config.toml` / managed config 不含 `sandbox_mode`。
3. active config 不含 `[sandbox_workspace_write]`。
4. `default_permissions` 是唯一主要 sandbox/profile selector。
5. `codex exec` header 不得顯示 `danger-full-access`。
6. smoke test 證明 `default_permissions` 的 profile 實際生效。

## 4. Claude Code CLI 行為模型

### 4.1 啟動模式

| 模式 | 指令 | 用途 |
|---|---|---|
| 互動 | `claude` | TUI session |
| 非互動 | `claude -p` / `claude --print` | pipeline、CI、wrapper |
| 繼續 session | `claude -c` / `--continue` | 讀最近 conversation |
| 指定 session | `--resume` / `--session-id` | 會引入歷史狀態，不適合無狀態企業 API |
| minimal | `--bare` | 跳過 hooks、LSP、plugin sync、auto-memory、CLAUDE.md auto-discovery 等；但本機 claude.ai OAuth 不可用 |

企業 wrapper 預設應使用 `--print`，禁止 `--continue`、`--resume`、`--session-id`，除非產品明確要求可追蹤長 session。

### 4.2 輸出格式

| 格式 | Flag | 實務處理 |
|---|---|---|
| text | `--output-format text` 或預設 | 不可作為自動化事實來源 |
| JSON result | `--output-format json` | 適合單回合 wrapper；讀 `result`、`is_error`、`permission_denials`、`usage` |
| JSONL stream | `--verbose --output-format stream-json` | 適合 UI/審計；需處理 tool_use、tool_result、system init、result |

注意：`stream-json` 沒有 `--verbose` 會直接失敗。

### 4.3 權限模式

官方文件定義的 permission modes：

| Mode | 行為重點 | 企業適用性 |
|---|---|---|
| `default` | 讀取自動，寫入/命令要問 | 人工互動適用 |
| `acceptEdits` | 可自動改工作區內檔案 | 半自動開發適用 |
| `plan` | 讀取與規劃，不編輯 | 安全分析、review 前置 |
| `auto` | 背景 classifier 判斷安全 | 需企業信任環境設定 |
| `dontAsk` | 只執行預先允許工具，其餘拒絕 | CI / locked-down wrapper 適用 |
| `bypassPermissions` | 跳過權限層 | 只能在外部隔離環境使用 |

企業預設應偏向：

```text
--permission-mode dontAsk
```

並搭配明確 `--tools`、`--allowedTools`、`--disallowedTools`。

### 4.4 工具控制

Claude Code 有兩層語意：

1. `--tools`：控制模型上下文中有哪些工具可見。
2. `--allowedTools` / `--disallowedTools`：控制哪些工具呼叫不需詢問或被拒絕。

實測指出 `disallowedTools` 把工具移出 context 後，模型仍可能在純文字中偽造 function call。wrapper 必須只接受結構化事件，不接受文字中的工具宣稱。

### 4.5 MCP、hooks、plugins、skills

預設 `claude --print --verbose --output-format stream-json` 會載入使用者層設定，包含：

1. hooks。
2. plugins。
3. MCP servers。
4. skills / slash commands。
5. memory path。

企業安全預設：

```powershell
--setting-sources project,local `
--disable-slash-commands `
--strict-mcp-config `
--mcp-config .\empty-mcp.json `
--tools=
```

若企業要部署 MCP，應用 `managed-mcp.json` 或 managed settings 的 `allowedMcpServers` / `deniedMcpServers`，且不要只用 `serverName` 當安全控制，因為名稱可由使用者指定。

### 4.6 sandbox

Claude Code sandbox 是 Bash 子程序層級的 OS isolation；不是所有工具都進 sandbox。

官方文件指出：

1. sandbox 限制 Bash 與其 child process。
2. Read/Edit/Write 等內建 file tools 走 permission system，不是 Bash sandbox。
3. macOS 使用 Seatbelt，Linux/WSL2 使用 bubblewrap/socat。
4. Native Windows sandbox support 仍非主要可依賴路徑。
5. sandbox 不可用時預設可能警告後改跑 unsandboxed；企業應設 `sandbox.failIfUnavailable = true`。

企業結論：Claude 的安全模型需要「permission deny + sandbox + 外部 runner 隔離」三層。只開 sandbox 不足以限制 file tools。

## 5. Codex CLI 行為模型

### 5.1 啟動模式

| 模式 | 指令 | 用途 |
|---|---|---|
| 互動 | `codex` | TUI |
| 非互動 | `codex exec` | automation / CI / wrapper |
| review | `codex review` / `codex exec review` | code review |
| resume | `codex resume` / `codex exec resume` | session continuation |
| MCP server | `codex mcp-server` | 把 Codex 暴露成 MCP server |

企業 wrapper 預設應使用 `codex exec --ephemeral`，除非要明確保留 thread。

### 5.2 輸出格式

`codex exec --json` 實測輸出 JSONL 至 stdout，事件類型包含：

1. `thread.started`
2. `turn.started`
3. `item.started`
4. `item.completed`
5. `turn.completed`

檔案變更事件使用 `type: "file_change"`；shell 使用 `type: "command_execution"`；最終文字使用 `type: "agent_message"`。

stderr 可能出現非 JSON 訊息，例如：

```text
Reading additional input from stdin...
```

wrapper 解析規則：

1. 只把 stdout 當 JSONL。
2. stderr 進診斷 log。
3. `turn.completed` 不等同 side effect 成功，需再驗證檔案、git diff、命令 exit code。

### 5.3 config 與狀態

本機 help 與 doctor 顯示：

1. `~/.codex/config.toml` 是主要 config。
2. `--config key=value` 可覆蓋設定。
3. `--profile` 可疊加 `$CODEX_HOME/<name>.config.toml`。
4. `--ignore-user-config` 不讀 `$CODEX_HOME/config.toml`，但 auth 仍使用 `CODEX_HOME`。
5. `--ephemeral` 不持久化 session files。
6. `--ignore-rules` 不讀 user/project execpolicy `.rules`。

企業應使用專用 `CODEX_HOME`，避免載入使用者個人 plugins、MCP、history、state、model 偏好。

### 5.4 sandbox

本機 `codex exec --help` 支援：

```text
-s, --sandbox <read-only|workspace-write|danger-full-access>
```

實測語意：

| sandbox | shell | file_change | 建議 |
|---|---|---|---|
| `read-only` | 本機 Windows shell 啟動失敗 | 寫入被拒 | 可作只讀分析，但需驗證 shell 可用性 |
| `workspace-write` | 本機 Windows shell 啟動失敗 | 工作區寫入成功 | 可作受控改檔，但不可假設 shell 可用 |
| `danger-full-access` | shell 成功 | 完全開放 | 只能放在外部隔離 runner |

本段是舊 `--sandbox` 模式研究資料。若採新版 permission profiles，不能把這些 flag 當生產 wrapper 控制點。

`codex sandbox` 子命令的新版名稱是 `--permissions-profile ':read-only'` / `':workspace'`，不是舊名 `read-only` / `workspace-write`。本機 0.135.0 已能解析 built-in profile 名稱，但 native Windows sandbox 啟動失敗，主要錯誤是 `windows sandbox failed: spawn setup refresh`；自訂 profile 也因 elevated Windows sandbox backend 未可用而失敗。

官方文件把新版 permission profile 視為 beta 控制面，並明確指出它不與舊 sandbox 設定組合使用。也就是說，以下兩套只能選一套作為主控：

1. 舊模式：`sandbox_mode`、`sandbox_workspace_write.*`、CLI `--sandbox`。
2. 新模式：`default_permissions` 與 `[permissions.<name>]`。

企業內部要穩定承載應用時，不能只看 profile 語法是否 parse 成功。必須在目標 runner 上驗證 shell 可啟動、workspace 內寫入可控、workspace 外讀寫被拒、network egress 符合 allowlist。若 native Windows elevated sandbox 不能穩定通過，應改用 WSL2/Linux runner 或外部 VM/container。

新版 profile 的企業方向：

```toml
default_permissions = "project-edit"

[permissions.project-edit.filesystem]
":minimal" = "read"

[permissions.project-edit.filesystem.":workspace_roots"]
"." = "write"
"**/*.env" = "deny"
".git" = "deny"
".codex" = "deny"

[permissions.project-edit.network]
enabled = false
```

若需要網路，使用 domain allowlist，不使用全域 `*`：

```toml
default_permissions = "readonly-net"

[permissions.readonly-net.filesystem]
":minimal" = "read"

[permissions.readonly-net.filesystem.":workspace_roots"]
"." = "read"

[permissions.readonly-net.network]
enabled = true

[permissions.readonly-net.network.domains]
"api.internal.example.com" = "allow"
```

### 5.5 approvals

Codex root CLI 有 `--ask-for-approval`，但本機 `codex exec` 不接受該 flag。以下是舊 sandbox 模式下的 `exec` 自動化控制點：

1. `--sandbox`
2. `--dangerously-bypass-approvals-and-sandbox`
3. `--ignore-rules`
4. `--ignore-user-config`
5. `--config approval_policy=...`（需依目標 shell 正確 quoting，仍應以本機測試為準）

新版 permission profiles 路線下，企業 wrapper 不應傳 `--sandbox`。舊 flag 只保留於相容性測試與回歸比較。

企業 wrapper 不應把文件或 root help 的 `--full-auto`、`--ask-for-approval` 直接套入 `codex exec`，必須啟動時檢查 `codex exec --help`。

官方文件目前仍描述 `codex exec` 可用預設 sandbox / approval 設定，也說 `--full-auto` 是 deprecated compatibility flag。這與本機 `codex exec --help` 的可用 flag 不完全一致。企業系統必須把「文件宣稱」與「本機 help 實測」分開記錄，並在版本升級時重新產生 allowlist。

### 5.6 network 與 managed configuration

Codex 官方安全文件指出，CLI / IDE / App 的舊 `workspace-write` 預設網路關閉；若開啟 `sandbox_workspace_write.network_access = true`，必須同時用 `features.network_proxy` 或 managed network policy 限制目的地。企業內部預設應保持 network off。

以下 `sandbox_workspace_write` 範例只適用舊 sandbox mode；若採新版 permission profiles，必須改用 `[permissions.<name>.network]`，不可混用。

禁止設定：

```toml
[sandbox_workspace_write]
network_access = true

[features.network_proxy.domains]
"*" = "allow"
```

最低允許形態：

```toml
[sandbox_workspace_write]
network_access = true

[features.network_proxy]
enabled = true

[features.network_proxy.domains]
"api.internal.example.com" = "allow"
"*.exfil.example.com" = "deny"
```

Codex managed configuration 還應使用 `requirements.toml` 鎖定組織政策。以下 `allowed_sandbox_modes` 是舊 sandbox mode 的 requirements 限制；若主控改成 permission profiles，不可把它當 profile allowlist：

```toml
allow_managed_hooks_only = true

allowed_sandbox_modes = ["read-only", "workspace-write"]
allowed_approval_policies = ["untrusted", "on-request"]

[features]
browser_use = false
browser_use_external = false
in_app_browser = false
computer_use = false

[experimental_network]
enabled = true
managed_allowed_domains_only = true
allowed_domains = ["api.internal.example.com"]
denied_domains = ["*.exfil.example.com"]
```

新版 permission profiles 的 network 設定要拆清楚：

1. `[permissions.<name>.network] enabled = true` 只改該 profile 的 sandbox network policy，不會自動啟動 network proxy。
2. `[permissions.<name>.network.domains]` 沒有 allow rule 時，domain request 應被阻擋；deny rule 勝過 allow rule。
3. `requirements.toml` 的 `[experimental_network]` 是管理層網路要求，和使用者 `features.network_proxy` toggle 分離。
4. 本機 `codex features list` 顯示 `network_proxy` 仍是 experimental 且未啟用，因此不能把本機視為已通過新版 domain allowlist egress 測試。
5. `--search` 啟用的是模型側 web search tool，不等於 shell / subprocess network 被允許，也不等於本地資料不會送到模型 provider。

實際 key 名稱需以目標版本 `codex doctor --json`、`codex features list`、官方 config reference 與本機 schema 同步驗證。`requirements.toml` 屬管理層，不應由專案 repo 或一般使用者寫入。

## 6. Swap 影響

### 6.1 不能只 swap 指令名

Claude 與 Codex 的語意差異集中在：

1. 輸出事件 schema 不同。
2. 權限模型不同。
3. sandbox 覆蓋範圍不同。
4. tools 可見性控制不同。
5. session persistence 與 state 位置不同。
6. hooks/plugins/MCP 的載入與隔離方式不同。
7. 錯誤時 exit code 與 JSON result 的語意不同。

應設計 adapter，而不是在程式裡寫：

```text
if provider == "claude": command = "claude ..."
else: command = "codex ..."
```

### 6.2 Adapter 共通介面

建議 wrapper 對上層應用只暴露以下抽象：

```ts
type AgentRunRequest = {
  provider: "claude" | "codex";
  prompt: string;
  cwd: string;
  mode: "read_only" | "workspace_write" | "no_tools" | "external_sandbox_full_access";
  model?: string;
  maxCostUsd?: number;
  outputSchemaPath?: string;
  allowedTools?: string[];
  deniedTools?: string[];
  allowedWriteRoots?: string[];
  timeoutMs: number;
  envPolicy: "minimal" | "managed";
  persistSession: boolean;
};

type AgentRunResult = {
  provider: "claude" | "codex";
  exitCode: number;
  finalText: string;
  structuredEvents: unknown[];
  toolCalls: ToolCall[];
  fileChanges: FileChange[];
  commandExecutions: CommandExecution[];
  usage: Usage | null;
  verifiedSideEffects: VerifiedSideEffect[];
  stderr: string;
};
```

上層不得直接讀 CLI 原始 final text 判斷成功。

### 6.3 Mapping 表

| 抽象需求 | Claude Code CLI | Codex CLI |
|---|---|---|
| 非互動 | `claude --print` | `codex exec` |
| 單一 JSON result | `--output-format json` | 無單一 result；解析 JSONL 或 `--output-last-message` |
| streaming events | `--verbose --output-format stream-json` | `--json` |
| 無 session persistence | `--no-session-persistence` | `--ephemeral` |
| 隔離使用者 config | `--setting-sources project,local`、`--bare` | `--ignore-user-config`、專用 `CODEX_HOME` |
| 禁用工具 | `--tools=` | 無等價 CLI flag；靠 sandbox / config / 外部隔離 |
| 禁用 slash/skills | `--disable-slash-commands` | `--ignore-user-config`、feature/config 控制 |
| 禁用 MCP | `--strict-mcp-config --mcp-config empty-mcp.json` | 專用 `CODEX_HOME`、空 MCP config、managed policy |
| 只讀 | `--permission-mode dontAsk --tools=Read` 或工具清空 | `-s read-only` |
| 工作區寫入 | `--tools=Write,Edit --allowedTools=...` 或 `acceptEdits` | `-s workspace-write` |
| 完全開放 | `--dangerously-skip-permissions` / `bypassPermissions` | `--dangerously-bypass-approvals-and-sandbox` 或 `-s danger-full-access` |
| 外部 sandbox 必要性 | bypass 時必要；Windows native sandbox 不作主保護 | danger-full-access 時必要；本機 Windows sandbox shell 未通過 |
| auth | claude.ai subscription 或 API key；`--bare` 需 API key/key helper | ChatGPT login 或 API key；auth 使用 `CODEX_HOME` |
| model | `--model sonnet` / full model id | `-m gpt-5.5` / config `model` |
| reasoning/effort | `--effort low|medium|high|xhigh|max` | config `model_reasoning_effort` / model settings |
| 成本限制 | `--max-budget-usd` | 未在本機 `exec --help` 看到等價 flag |

### 6.4 Swap 風險等級

| 類別 | 風險 | 原因 |
|---|---:|---|
| final text | 高 | Claude 實測可在無工具時文字宣稱已寫檔 |
| file edit | 中高 | Claude file tools 不受 Bash sandbox；Codex file_change 與 shell sandbox 行為不同 |
| shell | 高 | Codex Windows sandbox shell 實測失敗；Claude sandbox native Windows 不應依賴 |
| workspace 外讀取 | 高 | Claude `Read` 可讀 workspace 外；Codex `read-only/workspace-write` 可透過 shell 讀 workspace 外 |
| workspace 外寫入 | 高 | Claude `Write` 可寫 workspace 外；Codex `--add-dir` 會擴大可寫邊界 |
| MCP | 高 | 兩者都能載入外部工具，需企業 allowlist |
| hooks/plugins/skills | 高 | 預設使用者層設定會污染執行面 |
| JSON parser | 中 | Claude JSON result vs stream-json；Codex JSONL stdout + stderr 診斷 |
| session | 中 | resume/continue 會引入不可預測歷史 |
| cost | 中 | Claude 預設 Opus 在小 budget 下直接失敗；需指定 model/budget |

## 7. 企業安全預設

### 7.1 通用規則

1. 不用一般使用者 home 作生產執行環境。
2. 每次任務建立獨立工作目錄。
3. workspace 不等於資料邊界；workspace 外讀寫必須由 OS/container/managed policy 阻擋。
4. 預設不允許網路，除非任務需要且目的網域已 allowlist。
5. 預設不允許 shell，除非外部 runner 隔離。
6. 預設不允許 MCP，除非企業管理 allowlist。
7. 預設不讀使用者 hooks/plugins/skills。
8. 禁止 CLI resume/continue，除非 session 被產品資料模型管理。
9. wrapper 必須驗證 side effects。
10. wrapper 必須記錄：版本、完整 argv、cwd、env allowlist、stdout JSON、stderr、exit code、git diff、檔案 hash。
11. CLI 更新前必須跑 smoke test，不得自動套新版。

### 7.2 Claude locked-down read-only / no-tool 模板

```powershell
$env:DISABLE_TELEMETRY = "1"
$env:DO_NOT_TRACK = "1"
$env:CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC = "1"

$prompt |
  claude --print `
    --model sonnet `
    --effort low `
    --output-format json `
    --no-session-persistence `
    --setting-sources project,local `
    --disable-slash-commands `
    --strict-mcp-config `
    --mcp-config .\empty-mcp.json `
    --permission-mode dontAsk `
    --max-budget-usd 0.25 `
    --tools=
```

用途：分類、摘要、純推理、無檔案操作。  
驗證要求：init 事件需顯示 tools/MCP/skills/plugins 清空；若使用 JSON result，仍需獨立確認無檔案變更。

### 7.3 Claude controlled-write 模板

```powershell
$prompt |
  claude --print `
    --model sonnet `
    --effort low `
    --verbose `
    --output-format stream-json `
    --no-session-persistence `
    --setting-sources project,local `
    --disable-slash-commands `
    --strict-mcp-config `
    --mcp-config .\empty-mcp.json `
    --permission-mode dontAsk `
    --max-budget-usd 0.50 `
    --tools=Read,Write,Edit `
    --allowedTools=Read,Write,Edit `
    --disallowedTools="Bash(*)"
```

用途：受控檔案改寫。  
驗證要求：

1. stream-json init tools 只能包含 `Read`、`Write`、`Edit`。
2. 實際 git diff 必須只落在允許路徑。
3. final text 不作為成功依據。

### 7.4 Claude external-sandbox full access 模板

```powershell
$prompt |
  claude --print `
    --model sonnet `
    --effort low `
    --verbose `
    --output-format stream-json `
    --no-session-persistence `
    --setting-sources project,local `
    --strict-mcp-config `
    --mcp-config .\empty-mcp.json `
    --permission-mode bypassPermissions
```

只允許在外部 container/VM/ephemeral runner 中使用。不得在員工主機或共享內部伺服器直接使用。

### 7.5 Codex locked-down read-only 模板

```powershell
$env:CODEX_HOME = "C:\ProgramData\Company\codex-home"

codex exec `
  --ephemeral `
  --ignore-user-config `
  --ignore-rules `
  --skip-git-repo-check `
  --json `
  -s read-only `
  -m gpt-5.5 `
  "Analyze this repository without modifying files."
```

用途：只讀分析。  
本機限制：Windows sandbox 下 shell 啟動實測失敗；若任務需要 shell，必須先在目標 runner smoke test。

### 7.6 Codex controlled-write 模板

```powershell
$env:CODEX_HOME = "C:\ProgramData\Company\codex-home"

codex exec `
  --ephemeral `
  --ignore-user-config `
  --ignore-rules `
  --skip-git-repo-check `
  --json `
  -s workspace-write `
  -m gpt-5.5 `
  "Modify only files under the current working directory."
```

用途：受控工作區改寫。  
驗證要求：

1. parse stdout JSONL。
2. stderr 另存。
3. 檢查 `file_change` event。
4. 以 git diff / filesystem whitelist 驗證實際改動。
5. 不信任 final `agent_message`。

### 7.7 Codex external-sandbox full access 模板

```powershell
codex exec `
  --ephemeral `
  --ignore-user-config `
  --ignore-rules `
  --json `
  -s danger-full-access `
  -m gpt-5.5 `
  "Run the requested task inside this disposable runner."
```

只允許在外部 sandbox 已強制隔離的 runner 中使用。若直接在內部主機使用，等同把 shell 與檔案系統交給模型。

## 8. 企業 wrapper 必備檢查

### 8.1 啟動前檢查

1. 讀 `claude --version` / `codex --version`。
2. 讀目標 subcommand 的 `--help`，確認必要 flags 存在。
3. 檢查 forbidden flags 沒有出現：
   - Claude：`--dangerously-skip-permissions`、`--permission-mode bypassPermissions`。
   - Codex：`--dangerously-bypass-approvals-and-sandbox`、`-s danger-full-access`。
4. 建立空白或 managed config home。
5. 檢查 cwd 為任務專屬目錄。
6. 檢查 env allowlist，不把整個 parent env 傳入 CLI。
7. 若要 sandbox，跑 smoke test：
   - shell read。
   - allowed write。
   - denied write。
   - denied network。

### 8.2 執行中檢查

1. stdout/stderr 分流。
2. stdout JSON / JSONL schema 驗證。
3. tool call allowlist 驗證。
4. shell command prefix allowlist 驗證。
5. 檔案變更路徑即時檢查。
6. timeout 強制殺掉 process tree。
7. 記錄 token/cost/usage。

### 8.3 執行後檢查

1. exit code。
2. final structured result。
3. 實際 filesystem diff。
4. git status。
5. 是否產生未允許新檔。
6. 是否碰觸 protected paths：
   - `.git`
   - `.claude`
   - `.codex`
   - `.mcp.json`
   - shell profile
   - `.env*`
   - credential/config directories
7. 若任一檢查失敗，丟棄工作區，不把結果合併回主 repo。

## 9. 審計資料模型

每次 CLI invocation 應保存：

```json
{
  "run_id": "uuid",
  "provider": "claude|codex",
  "cli_version": "string",
  "argv": ["string"],
  "cwd": "string",
  "env_keys": ["string"],
  "started_at": "iso8601",
  "ended_at": "iso8601",
  "exit_code": 0,
  "stdout_events_path": "path",
  "stderr_path": "path",
  "final_text_path": "path",
  "tool_calls": [],
  "command_executions": [],
  "file_changes": [],
  "git_diff_path": "path",
  "policy_decision": "accepted|rejected|quarantined",
  "policy_violations": [],
  "usage": {}
}
```

審計目的不是保存越多越好，而是能重建「模型看到了什麼、工具做了什麼、wrapper 接受了什麼」。

## 10. 企業政策建議

### 10.1 最小可接受政策

1. CLI 版本 pin。
2. 專用執行帳號。
3. 專用 `CODEX_HOME` / Claude managed settings。
4. 預設 no-tools 或 read-only。
5. 所有寫入都在 ephemeral branch/worktree。
6. 所有結果由 wrapper 驗證後才進入內部系統。
7. 禁止使用者自帶 MCP / hooks / plugins。
8. 禁止直接繼承使用者 auth 與 config。
9. 禁止在未隔離主機使用 full access / bypass。
10. 所有 prompt 與輸出進入內部審計儲存，並套 secrets redaction。

### 10.2 Claude 管理設定方向

使用 managed settings，至少設定：

```json
{
  "disableBypassPermissionsMode": "disable",
  "forceRemoteSettingsRefresh": true,
  "allowedMcpServers": [
    { "serverUrl": "https://mcp.internal.example.com/*" },
    { "serverCommand": ["python", "C:\\enterprise\\mcp\\approved-server.py"] }
  ],
  "deniedMcpServers": [
    { "serverUrl": "https://*.untrusted.example.com/*" },
    { "serverName": "dangerous-server" }
  ],
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Bash(curl *)",
      "Bash(iwr *)",
      "Bash(Invoke-Expression *)"
    ]
  },
  "allowManagedPermissionRulesOnly": true,
  "allowManagedMcpServersOnly": true,
  "allowManagedHooksOnly": true,
  "strictPluginOnlyCustomization": ["skills", "hooks", "mcp"],
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "allowUnsandboxedCommands": false,
    "network": {
      "allowManagedDomainsOnly": true,
      "allowedDomains": ["api.internal.example.com"],
      "deniedDomains": ["*.exfil.example.com"]
    },
    "filesystem": {
      "allowManagedReadPathsOnly": true,
      "denyRead": ["~/.ssh", "~/.aws", "~/.kube", "~/.config"],
      "denyWrite": ["~", "/"]
    }
  }
}
```

MCP allowlist 必須用 `serverUrl` 或 `serverCommand` 作主要控制。`serverName` 是使用者指定的標籤，只能作為輔助 deny，不可作為安全 allowlist。

此 managed settings 範例屬於官方 schema 對齊，尚未在本機以 `C:\Program Files\ClaudeCode\managed-settings.json` 實際部署驗證。上線前必須用 `/status`、`/doctor`、`/mcp`、`/hooks`、`/permissions` 或非互動 init event 驗證生效。

若要完全禁用 Claude MCP，部署 managed `managed-mcp.json`：

```json
{"mcpServers":{}}
```

管理部署位置依平台而定；Windows 應使用 `C:\Program Files\ClaudeCode\managed-settings.json` 與 `C:\Program Files\ClaudeCode\managed-mcp.json`，不要使用已淘汰的 `C:\ProgramData\ClaudeCode\managed-settings.json`。

### 10.3 Codex 管理方向

Codex 企業化時應優先採外部 wrapper 控制：

1. 服務帳號專用 `CODEX_HOME`。
2. 不使用一般使用者 `~/.codex/config.toml`。
3. 啟動加 `--ignore-user-config --ignore-rules --ephemeral`。
4. 禁用或清空 MCP servers。
5. 用 OS/container/VM 形成真正 sandbox。
6. 每次執行前檢查 `codex exec --help`，確認本版 flags。
7. 不依賴 root help 的 approval flags。
8. 對 Windows runner，必跑 sandbox smoke test；本機已觀察到 `windows sandbox: spawn setup refresh`。

管理層應拆成兩份：

1. `requirements.toml`：不可被使用者覆寫的限制。
2. `managed_config.toml`：每次啟動套用的預設值，使用者可能在 session 中暫時調整，但下次啟動會回到管理預設。

Windows 路徑方向：

```text
%ProgramData%\OpenAI\Codex\requirements.toml
%USERPROFILE%\.codex\managed_config.toml
```

`requirements.toml` 基線：

```toml
allowed_approval_policies = ["untrusted", "on-request"]
allowed_sandbox_modes = ["read-only", "workspace-write"]
allowed_web_search_modes = []
allow_managed_hooks_only = true

[features]
hooks = true
browser_use = false
browser_use_external = false
in_app_browser = false
computer_use = false

[experimental_network]
enabled = true
managed_allowed_domains_only = true
allowed_domains = ["api.internal.example.com"]
denied_domains = ["*.exfil.example.com"]

[permissions.filesystem]
deny_read = [
  "~/.ssh",
  "~/.aws",
  "~/.kube",
  "/**/*.env"
]

[rules]
prefix_rules = [
  { pattern = [{ token = "rm" }], decision = "forbidden", justification = "Blocked destructive deletion." },
  { pattern = [{ token = "git" }, { any_of = ["push", "commit"] }], decision = "prompt", justification = "Require review before repository mutation." },
  { pattern = [{ any_of = ["curl", "iwr", "Invoke-WebRequest", "Invoke-Expression"] }], decision = "forbidden", justification = "Block unreviewed network/script execution." }
]
```

若允許 MCP，必須在 `requirements.toml` 中用 identity allowlist；若不允許，保留空的 `mcp_servers`：

```toml
[mcp_servers]
```

`managed_config.toml` 基線分兩種，不可混用。

舊 sandbox route：

```toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = false

[analytics]
enabled = false

[otel]
environment = "prod"
log_user_prompt = false
```

新版 permission profile route：

```toml
approval_policy = "on-request"
default_permissions = ":workspace"

[analytics]
enabled = false

[otel]
environment = "prod"
log_user_prompt = false
```

採新版 permission profile route 時，active config 不得同時存在 `sandbox_mode` 或 `[sandbox_workspace_write]`。本文件先前把 `managed_config.toml` 基線只寫成舊 sandbox route，容易誤導；已改成兩條路徑分離。

`CODEX_API_KEY` 只用於單次 `codex exec`，不得設成整個 job 或整個 runner 的長生命週期環境變數。若使用 ChatGPT / Codex access token，應透過專用服務帳號與短期憑證注入，並把 `CODEX_HOME` 權限限制到該服務帳號。

## 11. 風險矩陣

| 風險 | Claude | Codex | 控制 |
|---|---:|---:|---|
| 使用者設定污染 | 高 | 高 | 專用 home、ignore user config、managed settings |
| hooks 執行任意命令 | 高 | 中高 | managed hooks only、禁用 hooks、審計 |
| MCP 擴權 | 高 | 高 | managed MCP allowlist、空 MCP config |
| 模型文字誤報成功 | 高 | 高 | 只信 tool event + filesystem |
| sandbox 不可用 fallback | 高 | 高 | fail-closed smoke test |
| Windows sandbox 差異 | 中 | 高 | 禁把 Windows sandbox 當唯一保護 |
| secrets 讀取 | 高 | 高 | deny read、env allowlist、redaction |
| 網路外洩 | 高 | 高 | egress proxy、domain allowlist、無網 runner |
| bypass/danger flags | 極高 | 極高 | policy 禁用，只允許外部隔離 |
| CLI 更新破壞 wrapper | 中高 | 中高 | version pin、help diff、smoke tests |

## 12. 資安 Review 補強結論

Review 後直接補強的缺口：

1. Claude managed settings 範例修正為 top-level `disableBypassPermissionsMode`，避免把 bypass 禁用錯放在 `permissions` 下。
2. Claude MCP 控制補上 `serverUrl` / `serverCommand` allowlist；明確禁止只用 `serverName` 作安全控制。
3. Claude managed MCP 補上空 `managed-mcp.json` 的完全禁用策略。
4. Codex 補上 `requirements.toml` 與 `managed_config.toml` 的分工，避免只靠 wrapper flags。
5. Codex 補上 permission profile 與舊 sandbox 模式不能混用的限制。
6. Codex 補上 network proxy / managed network policy，並移除全域 `*` 允許網路的落地風險。
7. 補上 `CODEX_API_KEY` 僅限單次 `codex exec` 注入，不作長生命週期 runner env。
8. 補上本機 `codex sandbox --permissions-profile` 的實測失敗結果，避免把官方 profile 語意直接視為 Windows 可用。

仍不能放寬的項目：

1. 不在員工主機上使用 `danger-full-access` / `bypassPermissions`。
2. 不把使用者個人 `~/.claude` 或 `~/.codex` 當企業服務執行 home。
3. 不使用模型 final text 判斷檔案、shell、MCP、網路動作成功。
4. 不允許 repo 內 `.claude`、`.codex`、`.mcp.json`、hooks、skills 自動進入生產執行面。
5. 不允許 CLI 自動更新進入生產；升級必須跑版本 help diff、sandbox smoke test、side-effect verifier regression。

### 12.1 上線 Gate

企業內部上線前必須全部通過：

| Gate | 驗證方式 | 未通過處置 |
|---|---|---|
| 版本 pin | `claude --version`、`codex --version` 與核准清單一致 | 禁止啟動 |
| flag schema | `claude --help`、`codex exec --help`、`codex sandbox --help` diff 無未審核變更 | 禁止啟動 |
| config isolation | Claude init event / Codex doctor 顯示未載入使用者 MCP、hooks、plugins | 隔離 home 重建 |
| sandbox smoke | shell read、allowed write、denied write、denied network 均符合預期 | 禁止使用該 runner |
| workspace boundary | workspace 內讀寫可控；workspace 外讀寫被 OS/container/managed policy 阻擋 | 禁止使用該 runner |
| network egress | 無網任務不能連外；有網任務只到 allowlist domain | 丟棄結果並封存審計 |
| secret boundary | `.env*`、SSH、cloud config、vault token 路徑不可讀 | 停止任務 |
| output parser | stdout JSON/JSONL schema 驗證通過，stderr 分流 | 標記 runner 不健康 |
| side-effect verifier | git diff、檔案 hash、路徑 whitelist 全部一致 | 丟棄 workspace |
| audit | argv、cwd、env keys、events、stderr、diff、usage 都入庫 | 禁止回傳成功 |

### 12.2 內部威脅模型

企業承載 CLI 時，主要敵手不是單一模型失誤，而是多層組合：

1. 惡意 repo 透過 `AGENTS.md`、`CLAUDE.md`、`.claude/settings.json`、`.codex/config.toml`、`.mcp.json` 改變 agent 行為。
2. prompt injection 要求讀取 secrets、開網路、安裝套件、執行遠端 script。
3. 供應鏈套件透過 postinstall、build script、test script 執行外連或 persistence。
4. MCP server 假冒合法名稱，或以 stdio command 啟動任意本機程式。
5. hooks / plugins / skills 在使用者 home 或專案中預先植入。
6. 模型輸出偽造成功，誘導上層系統跳過實際驗證。
7. 長 session / resume 把舊任務、舊授權、舊機密上下文帶入新任務。

對應控制：

1. 生產任務不讀 repo 內 agent config，除非先經靜態審核與簽章。
2. wrapper 只傳入必要 prompt 與必要檔案，不把整個 home、整個 repo、整個 env 暴露給 CLI。
3. package install / test / build 類命令在無網或 allowlist proxy 下跑。
4. MCP 以 URL / command identity 控制，不以 server name 控制。
5. hooks / plugins / skills 僅能從 managed source 載入。
6. 所有成功條件以 event、filesystem、git diff、exit code、hash 驗證。
7. 預設 `--no-session-persistence` / `--ephemeral`，禁止跨任務 resume。

### 12.3 推薦預設矩陣

| 用例 | Claude 預設 | Codex 預設 | 說明 |
|---|---|---|---|
| 純文字分析 | `--tools= --permission-mode dontAsk` | `codex exec --ephemeral --ignore-user-config --ignore-rules -s read-only` | 不允許 side effect |
| 程式碼 review | `--tools=Read,Grep --permission-mode dontAsk` | `-s read-only` | 只讀 repo，不跑測試 |
| 自動修 patch | `--tools=Read,Edit,Write --permission-mode dontAsk` | `-s workspace-write` | 寫入只准 workspace，完成後 verifier 檢查 diff |
| 需跑測試 | 外部 runner + sandbox fail-closed | 外部 runner + workspace-write smoke passed | 測試可能執行 repo script，必須外部隔離 |
| 需網路 | managed domain allowlist | `network_proxy` / managed network allowlist | 禁止直接全網 |
| 維運/部署 | 不由 CLI 自動執行 | 不由 CLI 自動執行 | 只能產生 plan/diff，由現有 CI/CD 或人審系統執行 |
| full access | 只在 disposable VM/container | 只在 disposable VM/container | 禁止員工主機與共享主機 |

### 12.4 聯網限制與本地資料處理

必須分成三層，不可混為一談：

1. 工具層聯網：Bash、PowerShell、MCP、WebFetch、WebSearch、browser、package manager、test/build script 是否能連外。
2. CLI/產品層聯網：telemetry、update check、model catalog、auth refresh、plugin sync、remote control 是否連外。
3. 模型推理層聯網：prompt、檔案片段、工具輸出是否送到模型 provider。

限制 shell/MCP/WebSearch 只能解決第 1 層；不代表資料沒有送到模型 provider。若要求「所有資料只在本地處理」，必須同時滿足第 1、2、3 層都不連外。

Claude Code 判斷：

1. 可限制工具層聯網：`--tools=`、`--strict-mcp-config --mcp-config empty-mcp.json`、`--disable-slash-commands`、managed sandbox network policy。
2. 可減少產品層背景行為：`--bare`、`--no-chrome`、`DISABLE_TELEMETRY=1`、`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`。
3. 不能把 Claude 模型推理變成本地推理。只要使用 Claude 模型，prompt 與被讀入上下文的資料會送到 Anthropic 或企業指定的雲端 provider。`--bare` 不改變模型推理位置。

Claude local-only 結論：若法規或公司政策要求「原始資料、程式碼、prompt 不得離開本機或內網」，Claude Code CLI 不應作為該 workload 的模型執行器。它只能用在資料可送往核准模型 provider 的場景，或只產生不含敏感內容的任務外殼。

Codex 判斷：

1. 可限制工具層聯網：`-s read-only` / `-s workspace-write`、`sandbox_workspace_write.network_access = false`、managed network policy、OS firewall。
2. 可限制產品層背景行為：專用 `CODEX_HOME`、`--ignore-user-config`、`--ephemeral`、禁用 plugins/MCP/browser/computer-use、關閉 telemetry/update 相關設定。
3. 可能可用本地模型路徑：本機 `codex --help` 顯示 `--oss --local-provider <lmstudio|ollama>`。這是目前兩者中較接近本地模型推理的路徑，但仍必須在無外網 runner 上實測確認不會觸發 OpenAI auth、model catalog、telemetry、plugin sync 或其他背景連線。

Codex local-only 最小驗證：

```powershell
codex exec `
  --oss `
  --local-provider ollama `
  --ephemeral `
  --ignore-user-config `
  --ignore-rules `
  --json `
  -s read-only `
  "Respond exactly: LOCAL_ONLY_PROBE"
```

驗證方式不是看 CLI 成功輸出，而是看 OS/network 層：

1. runner 預設 deny all outbound。
2. 只允許連到本機 `127.0.0.1` / `::1` 的 Ollama 或 LM Studio。
3. 抓取 process-level network event，確認沒有對 OpenAI、Anthropic、GitHub、npm、plugin marketplace、telemetry endpoint 的連線。
4. 檢查 stdout/stderr，確認沒有 auth fallback、model download、plugin sync。
5. 檢查任務結束後沒有新 credentials、history、session、cache 寫到非專用目錄。

local-only 強制政策：

```text
Default deny outbound at OS/firewall layer
Allow 127.0.0.1:11434 only when using Ollama
Allow 127.0.0.1:<LMStudioPort> only when using LM Studio
Block DNS except internal resolver if needed
Block package registries, GitHub raw URLs, public webhook/request-bin domains
Disable MCP unless MCP server is local and signed
Disable browser/chrome/computer-use
Disable WebSearch/WebFetch tools
Disable resume/session persistence
Redact or block .env, SSH, cloud config, vault tokens before prompt construction
```

企業結論：限制 CLI 聯網必須由 CLI flags、managed config、OS firewall、egress proxy、runner 隔離共同完成。若要求全本地資料處理，Claude Code 不符合模型本地推理條件；Codex 需改用 `--oss --local-provider ollama|lmstudio` 並通過無外網實測。

## 13. 最終架構建議

建議架構：

```text
Internal App
  -> Agent Adapter API
    -> Policy Compiler
    -> Provider Runner
      -> Ephemeral Workspace
      -> Dedicated CLI Home
      -> External Sandbox / Runner
      -> claude or codex process
    -> Event Parser
    -> Side-effect Verifier
    -> Audit Store
    -> Result Normalizer
```

不可讓應用直接 spawn：

```text
claude "do task"
codex "do task"
```

必須經過 policy compiler 與 verifier。

## 14. 採用判斷

### 14.1 Claude Code CLI

可用於企業內部承載，但需：

1. 明確使用 `--print`。
2. 禁止預設讀使用者 hooks/plugins/MCP/skills。
3. 使用 managed settings 鎖住 bypass、MCP、hooks、permissions。
4. 若要 minimal mode，處理 `--bare` 的 API key/key helper 認證。
5. 永遠驗證 tool events 與檔案結果。

### 14.2 Codex CLI

可用於企業內部承載，但需：

1. 使用 `codex exec --json --ephemeral`。
2. 使用專用 `CODEX_HOME`。
3. 避免依賴一般使用者 config。
4. 在 Windows 上不要假設 read-only/workspace-write shell 可用；必跑 smoke test。
5. 若需 shell，優先跑在外部隔離 runner。
6. 以 stdout JSONL 為事件來源，stderr 作診斷來源。
7. 若採新版 permission profiles，禁止同時使用舊 `sandbox_mode` / CLI `--sandbox` 作同一層控制。
8. 對 Windows runner，必須先修通 elevated sandbox；本機 0.135.0 仍出現 `spawn setup refresh`。

### 14.3 Swap 可行性

可 swap，但只能 swap adapter，不可 swap prompt 與 flags。

最低必要 normalization：

1. final text normalization。
2. event schema normalization。
3. file change normalization。
4. command execution normalization。
5. permission/sandbox mode normalization。
6. auth/config isolation normalization。
7. side effect verification normalization。

## 15. 官方資料來源

Claude Code：

1. CLI reference：<https://code.claude.com/docs/en/cli-usage>
2. Permission modes：<https://code.claude.com/docs/en/permission-modes>
3. Permissions：<https://code.claude.com/docs/en/permissions>
4. Settings：<https://code.claude.com/docs/en/settings>
5. Server-managed settings：<https://code.claude.com/docs/en/server-managed-settings>
6. Sandboxing：<https://code.claude.com/docs/en/sandboxing>
7. Environment variables：<https://code.claude.com/docs/en/env-vars>
8. Managed MCP：<https://code.claude.com/docs/en/managed-mcp>
9. Hooks：<https://docs.anthropic.com/en/docs/claude-code/hooks>
10. Memory / `CLAUDE.md`：<https://docs.anthropic.com/en/docs/claude-code/memory>
11. Skills：<https://code.claude.com/docs/en/skills>
12. Plugins：<https://code.claude.com/docs/en/plugins>
13. MCP：<https://code.claude.com/docs/en/mcp>
14. `.claude` directory / `CLAUDE_CONFIG_DIR`：<https://code.claude.com/docs/en/claude-directory>
15. Debug configuration：<https://code.claude.com/docs/en/debug-your-config>

Codex：

1. OpenAI Codex non-interactive mode：<https://developers.openai.com/codex/noninteractive>
2. OpenAI Codex agent approvals & security：<https://developers.openai.com/codex/agent-approvals-security>
3. OpenAI Codex sandboxing：<https://developers.openai.com/codex/concepts/sandboxing>
4. OpenAI Codex permissions：<https://developers.openai.com/codex/permissions>
5. OpenAI Codex managed configuration：<https://developers.openai.com/codex/enterprise/managed-configuration>
6. OpenAI Codex configuration reference：<https://developers.openai.com/codex/config-reference>
7. OpenAI Codex environment variables：<https://developers.openai.com/codex/environment-variables>
8. OpenAI Codex GitHub README：<https://github.com/openai/codex/blob/main/README.md>
9. Codex config schema：<https://github.com/openai/codex/blob/main/codex-rs/core/config.schema.json>
10. OpenAI Codex Windows：<https://developers.openai.com/codex/windows>
11. OpenAI Codex config schema JSON：<https://developers.openai.com/codex/config-schema.json>
12. OpenAI Codex hooks：<https://developers.openai.com/codex/hooks>
13. OpenAI Codex plugins：<https://developers.openai.com/codex/plugins>
14. OpenAI Codex skills：<https://developers.openai.com/codex/skills>
15. OpenAI Codex MCP：<https://developers.openai.com/codex/mcp>
16. OpenAI Codex rules：<https://developers.openai.com/codex/rules>

## 16. 本機重跑驗證命令

### Claude

```powershell
claude --version
claude --help
claude auth status
claude mcp list
claude auto-mode defaults
claude auto-mode config

"Respond exactly: MINIMAL_INIT" |
  claude --print `
    --model sonnet `
    --effort low `
    --verbose `
    --output-format stream-json `
    --no-session-persistence `
    --setting-sources project,local `
    --disable-slash-commands `
    --strict-mcp-config `
    --mcp-config .\empty-mcp.json `
    --max-budget-usd 0.25 `
    --tools=
```

### Codex Windows / current host

```powershell
codex --version
codex --help
codex exec --help
codex doctor --json
codex features list
codex login status
codex mcp list --json
codex sandbox --help
codex update --help
npm view @openai/codex version
codex sandbox --permissions-profile ':read-only' -C . powershell -NoProfile -Command Get-Location
codex sandbox --permissions-profile ':workspace' -C . powershell -NoProfile -Command Get-Location
codex sandbox --permissions-profile ':danger-full-access' -C . powershell -NoProfile -Command Get-Location

codex exec `
  --ephemeral `
  --strict-config `
  -C . `
  -o .\codex-mode-check.txt `
  "Do not use tools. Report only the sandbox mode from the header."

codex exec `
  --ephemeral `
  --ignore-user-config `
  --ignore-rules `
  --skip-git-repo-check `
  --json `
  -s read-only `
  -m gpt-5.5 `
  "Respond exactly: CODEX_OK"
```

### Codex Linux / WSL2

```bash
set -euo pipefail

codex --version
codex --help
codex exec --help
codex doctor --json
codex features list
codex login status
codex mcp list --json
codex sandbox --help
codex sandbox linux --help
npm view @openai/codex version

command -v bwrap
uname -a
cat /etc/os-release

ROOT="$(pwd)/.tmp_codex_linux_probe"
WORKSPACE="$ROOT/workspace"
OUTSIDE="$ROOT/outside"
CODEX_HOME_TEST="$ROOT/codex-home"

rm -rf "$ROOT"
mkdir -p "$WORKSPACE" "$OUTSIDE" "$CODEX_HOME_TEST"
printf 'INSIDE_BOUNDARY_DATA\n' > "$WORKSPACE/inside_read.txt"
printf 'OUTSIDE_BOUNDARY_SECRET\n' > "$OUTSIDE/outside_read.txt"
printf 'ENV_SECRET\n' > "$WORKSPACE/.env"

cat > "$CODEX_HOME_TEST/config.toml" <<EOF
default_permissions = "enterprise-linux"

[permissions.enterprise-linux]
extends = ":workspace"

[permissions.enterprise-linux.filesystem]
":minimal" = "read"
"$OUTSIDE" = "deny"

[permissions.enterprise-linux.filesystem.":workspace_roots"]
"." = "write"
"**/*.env" = "deny"
".git" = "deny"
".codex" = "deny"

[permissions.enterprise-linux.network]
enabled = false
EOF

export CODEX_HOME="$CODEX_HOME_TEST"

expect_success() {
  name="$1"
  shift
  if "$@"; then
    printf 'PASS expected success: %s\n' "$name"
  else
    printf 'FAIL expected success: %s\n' "$name"
    exit 1
  fi
}

expect_fail() {
  name="$1"
  shift
  if "$@"; then
    printf 'FAIL expected block: %s\n' "$name"
    exit 1
  else
    printf 'PASS expected block: %s\n' "$name"
  fi
}

expect_success "workspace read" codex sandbox linux --permissions-profile enterprise-linux -C "$WORKSPACE" bash -lc 'cat inside_read.txt'
expect_success "workspace write" codex sandbox linux --permissions-profile enterprise-linux -C "$WORKSPACE" bash -lc 'printf OK > inside_write.txt && cat inside_write.txt'
expect_fail "outside read" codex sandbox linux --permissions-profile enterprise-linux -C "$WORKSPACE" bash -lc "cat '$OUTSIDE/outside_read.txt'"
expect_fail "outside write" codex sandbox linux --permissions-profile enterprise-linux -C "$WORKSPACE" bash -lc "printf BAD > '$OUTSIDE/outside_write.txt'"
expect_fail ".env read" codex sandbox linux --permissions-profile enterprise-linux -C "$WORKSPACE" bash -lc 'cat .env'
expect_fail "network egress" codex sandbox linux --permissions-profile enterprise-linux -C "$WORKSPACE" bash -lc 'curl -fsS https://example.com >/tmp/codex_net_probe'
```

## 17. 立即落地結論

1. 兩者都能作為企業應用底層執行器。
2. 兩者都不應被直接暴露為「任意 prompt 進 CLI」。
3. Claude 的強項是工具/權限可見性控制較直接；風險是預設載入使用者 hooks/plugins/MCP/skills。
4. Codex 的強項是 `exec --json --ephemeral` 與新版 permission profiles / requirements；風險是本機 Windows elevated sandbox 未通過，且使用者 config 可把無 flag 執行推到 `danger-full-access`。
5. 安全 swap 的核心不是 prompt engineering，而是 adapter、policy compiler、external sandbox、side-effect verifier、audit store。

## 18. Global 設定對 Local 專案的影響

本節結論：Claude Code 與 Codex 都會讓 global/user 層影響 local 專案。local 專案沒有設定檔時，不代表乾淨執行；只是更直接繼承使用者 home 目錄中的設定、技能、外掛、MCP、hooks、rules、記憶與狀態。

本機專案狀態：

| 項目 | Claude Code | Codex |
|---|---|---|
| local config 目錄 | `.claude/` 不存在 | `.codex/` 不存在 |
| local MCP | `.mcp.json` 不存在 | local MCP config 不存在 |
| local memory/rules | `CLAUDE.md` 不存在 | `AGENTS.md` 不存在 |
| user/global config | `C:\Users\harry\.claude\settings.json` 存在 | `C:\Users\harry\.codex\config.toml` 存在 |
| user/global memory | `C:\Users\harry\.claude\CLAUDE.md` 存在 | `C:\Users\harry\.codex\AGENTS.md` 存在 |
| user/global plugins | 4 個 user plugin enabled | 4 個 plugin enabled |
| user/global skills | 28 個 user skill 目錄 | system/plugin skills enabled |
| user/global MCP | 4 個 claude.ai connector 顯示 needs-auth | 2 個 MCP server enabled |
| user/global hooks/rules | hooks、rules 目錄存在 | hooks feature enabled；user config loaded |

### 18.1 Claude Code global 影響面

官方設定層級可分為 managed、command line、local project、shared project、user。user scope 是 `~/.claude/settings.json`，會套用到所有專案；project scope 是 `.claude/settings.json`；local project scope 是 `.claude/settings.local.json`。managed settings 優先級最高，適合企業管控。

預設 `claude --print` 在本專案啟動，即使使用 `--tools=` 清掉內建工具，仍會載入 global/user 層：

```json
{
  "tools": [
    "LSP",
    "mcp__claude_ai_Figma__authenticate",
    "mcp__claude_ai_Gmail__authenticate",
    "mcp__claude_ai_Google_Calendar__authenticate",
    "mcp__claude_ai_Google_Drive__authenticate"
  ],
  "mcp_servers": [
    {"name": "claude.ai Google Drive", "status": "needs-auth"},
    {"name": "claude.ai Figma", "status": "needs-auth"},
    {"name": "claude.ai Gmail", "status": "needs-auth"},
    {"name": "claude.ai Google Calendar", "status": "needs-auth"}
  ],
  "slash_commands": "user skills + plugin skills + built-ins loaded",
  "skills": "user skills + plugin skills loaded",
  "plugins": [
    "gopls-lsp@claude-plugins-official",
    "frontend-design@claude-plugins-official",
    "skill-creator@claude-plugins-official",
    "superpowers@claude-plugins-official"
  ]
}
```

實測命令：

```powershell
"Respond exactly: DEFAULT_INIT_OK" |
  claude --print `
    --model sonnet `
    --effort low `
    --verbose `
    --output-format stream-json `
    --no-session-persistence `
    --max-budget-usd 0.10 `
    --tools=
```

結果：exit `0`，但 init event 仍含 LSP、claude.ai MCP auth tools、4 個 needs-auth MCP servers、大量 slash commands、skills、4 個 user plugins。`--tools=` 不是 global 隔離旗標，只是限制內建 tool list。

隔離後實測：

```powershell
$emptyMcp = "$env:TEMP\empty-mcp.json"
Set-Content -LiteralPath $emptyMcp -Value '{"mcpServers":{}}' -Encoding UTF8

"Respond exactly: GLOBAL_ISOLATION_OK" |
  claude --print `
    --model sonnet `
    --effort low `
    --verbose `
    --output-format stream-json `
    --no-session-persistence `
    --setting-sources project,local `
    --disable-slash-commands `
    --strict-mcp-config `
    --mcp-config $emptyMcp `
    --max-budget-usd 0.10 `
    --tools=
```

init event：

```json
{
  "tools": [],
  "mcp_servers": [],
  "slash_commands": [],
  "skills": [],
  "plugins": [],
  "memory_paths": {
    "auto": "C:\\Users\\harry\\.claude\\projects\\D--side-project-agent-study\\memory\\"
  }
}
```

結論：

1. `--setting-sources project,local` 可排除 user settings。
2. `--disable-slash-commands` 可清掉 skills/slash commands。
3. `--strict-mcp-config --mcp-config empty-mcp.json` 可清掉 MCP servers。
4. `--tools=` 可清掉可用工具，但不會單獨清掉 MCP、skills、plugins。
5. `--no-session-persistence` 不等於不碰 global state；init 仍顯示 auto memory path 在 `~/.claude/projects/...`。
6. 需要最小化背景行為時，使用 `--bare`；但 `--bare` 不讀 claude.ai OAuth/keychain，本機實測會因未提供 `ANTHROPIC_API_KEY` 或 `apiKeyHelper` 而失敗。
7. `--bare` 的官方 help 明確指出會跳過 hooks、LSP、plugin sync、attribution、auto-memory、background prefetches、keychain reads、`CLAUDE.md` auto-discovery；skills 仍可透過 `/skill-name` 明確解析。

企業規則：

```powershell
claude --print `
  --bare `
  --settings .\enterprise-claude-settings.json `
  --system-prompt "..." `
  --mcp-config .\empty-mcp.json `
  --strict-mcp-config `
  --disable-slash-commands `
  --tools= `
  --no-session-persistence
```

若不能使用 `--bare`，最低限度使用：

```powershell
claude --print `
  --setting-sources project,local `
  --strict-mcp-config `
  --mcp-config .\empty-mcp.json `
  --disable-slash-commands `
  --tools= `
  --no-session-persistence `
  --no-chrome
```

但此模式仍可能保留產品層 memory/state path。企業級控制仍需 managed settings 與外部 runner 隔離。

### 18.2 Claude Code managed 防污染設定

Windows managed settings 位置應使用：

```text
C:\Program Files\ClaudeCode\managed-settings.json
C:\Program Files\ClaudeCode\managed-mcp.json
```

企業建議：

```json
{
  "disableBypassPermissionsMode": "disable",
  "allowManagedHooksOnly": true,
  "allowManagedMcpServersOnly": true,
  "allowManagedPermissionRulesOnly": true,
  "disableAllHooks": true,
  "disableSkillShellExecution": true,
  "strictKnownMarketplaces": [],
  "claudeMdExcludes": ["**/CLAUDE.md", "~/.claude/CLAUDE.md"],
  "permissions": {
    "allow": [],
    "deny": [
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(powershell *Invoke-WebRequest*)",
      "Bash(powershell *Invoke-RestMethod*)",
      "Read(C:\\Users\\*\\.ssh\\**)",
      "Read(C:\\Users\\*\\.aws\\**)",
      "Read(C:\\Users\\*\\.azure\\**)",
      "Read(C:\\Users\\*\\.config\\**)",
      "Read(**\\.env)",
      "Write(C:\\Users\\**)",
      "Write(D:\\**)"
    ]
  }
}
```

此範例只使用已對照官方文件的 managed settings key。`strictKnownMarketplaces` 的官方型別是 marketplace source object 陣列；空陣列表示不允許新增 allowlisted marketplace。若要停用既有 installed plugin，不能用萬用字元推測，需列出實際 plugin id 並用 `enabledPlugins` 設為 `false`，再以 `claude plugin list` / init event 驗證。

空 `managed-mcp.json`：

```json
{
  "mcpServers": {}
}
```

注意：

1. managed settings 是企業策略，不應交由專案 repo 自行提供。
2. `allowManagedHooksOnly` 可避免 user/project hook 進入執行面。
3. `allowManagedMcpServersOnly` 可避免 user/project MCP 進入執行面。
4. `allowManagedPermissionRulesOnly` 可避免 user/project permission rule 擴權。
5. `claudeMdExcludes` 應排除 user/global 與專案自帶 `CLAUDE.md`，再由 wrapper 用 `--system-prompt` 明確注入允許內容。
6. Windows native Claude Code 不應被視為 OS filesystem sandbox；要限制讀寫必須靠 permission deny、managed policy、外部 runner ACL / container / VM。
7. 本節 managed settings 範例是官方 schema 對齊，不是本機 managed deployment 實測結果；需在企業 Windows/Linux/WSL2 runner 上部署後驗證。

### 18.3 Codex global 影響面

Codex user config 位於 `CODEX_HOME`，本機為：

```text
C:\Users\harry\.codex
```

本機 `codex doctor --json` 顯示：

```json
{
  "CODEX_HOME": "C:\\Users\\harry\\.codex",
  "config.toml": "C:\\Users\\harry\\.codex\\config.toml",
  "mcp servers": "2",
  "model": "gpt-5.5",
  "filesystem sandbox": "unrestricted",
  "approval policy": "Never",
  "enabled feature flags": "hooks, apps, plugins, in_app_browser, browser_use, browser_use_external, computer_use, ..."
}
```

本機 global 來源：

| 類型 | 路徑 / 狀態 |
|---|---|
| user config | `C:\Users\harry\.codex\config.toml` |
| user instructions | `C:\Users\harry\.codex\AGENTS.md` |
| user skills | `C:\Users\harry\.codex\skills` |
| plugins cache | `C:\Users\harry\.codex\plugins`、runtime cache |
| MCP servers | `node_repl`、`reeve-spike` enabled |
| state DB | `state_5.sqlite`、`logs_2.sqlite`、`memories_1.sqlite`、`goals_1.sqlite` |
| sessions/history | `sessions`、`history.jsonl` |

預設 `codex exec` 在本專案執行時，stderr 出現多筆：

```text
WARN codex_core_skills::loader: ignoring interface.icon_small ...
WARN codex_core_skills::loader: ignoring interface.icon_large ...
```

這代表 plugin/skill loader 仍在 local 專案中啟動。實測命令：

```powershell
codex exec `
  --json `
  --ephemeral `
  -m gpt-5.5 `
  "Do not use tools. Reply exactly: CODEX_DEFAULT_GLOBAL_CHECK"
```

`--ignore-user-config --ignore-rules` 不足以隔離 plugin/skill loader。實測仍出現同類 loader warnings：

```powershell
codex exec `
  --json `
  --ephemeral `
  --ignore-user-config `
  --ignore-rules `
  -m gpt-5.5 `
  "Do not use tools. Reply exactly: CODEX_IGNORE_ONLY_CHECK"
```

加上 feature disable 後，plugin/skill loader warnings 消失，只剩 PowerShell shell snapshot 診斷：

```powershell
codex exec `
  --json `
  --ephemeral `
  --ignore-user-config `
  --ignore-rules `
  --disable plugins `
  --disable apps `
  --disable hooks `
  --disable browser_use `
  --disable browser_use_external `
  --disable computer_use `
  --disable in_app_browser `
  -m gpt-5.5 `
  "Do not use tools. Reply exactly: CODEX_ISOLATED_GLOBAL_CHECK"
```

結論：

1. `--ignore-user-config` 只是不讀 `$CODEX_HOME/config.toml`；auth 仍使用 `CODEX_HOME`。
2. `--ignore-rules` 只是不讀 user/project execpolicy `.rules`。
3. `--ephemeral` 只是不持久化該次 session files；不代表不讀 global config、skills、plugins、MCP 或 state DB。
4. user-installed plugins/skills 可能不完全由 `config.toml` 控制；只用 `--ignore-user-config` 仍會載入 plugin/skill loader。
5. 企業隔離必須用乾淨 `CODEX_HOME`、feature disable、managed config / requirements、外部 runner 隔離共同完成。

### 18.4 Codex 防污染設定

最低執行旗標：

```powershell
$env:CODEX_HOME = "D:\agent-runner\codex-home-clean"

codex exec `
  --json `
  --ephemeral `
  --ignore-user-config `
  --ignore-rules `
  --strict-config `
  --disable plugins `
  --disable apps `
  --disable hooks `
  --disable browser_use `
  --disable browser_use_external `
  --disable computer_use `
  --disable in_app_browser `
  --disable image_generation `
  --disable multi_agent `
  -C . `
  -m gpt-5.5 `
  "..."
```

更穩定的企業模式：

```text
Dedicated Windows/Linux service account
Dedicated CODEX_HOME per runner/job class
No personal ~/.codex reuse
Managed %USERPROFILE%\.codex\managed_config.toml
Managed %ProgramData%\OpenAI\Codex\requirements.toml
Disable plugins/apps/hooks/browser/computer-use unless explicitly required
Allowlist MCP servers by managed policy only
External filesystem sandbox / container / VM for hard read/write boundary
```

`CODEX_HOME` 內容應由部署系統產生，不應複製開發者個人 `~/.codex`。若需要 auth，使用服務帳號與最小權限 token，不混用個人 sessions/history/memories/plugins。

### 18.5 Global 與 Local 的覆寫規則

| 需求 | Claude Code | Codex |
|---|---|---|
| 避免 user config | `--setting-sources project,local` 或 `--bare` | `--ignore-user-config` + 專用 `CODEX_HOME` |
| 避免 user memory | `--bare`；或 managed `claudeMdExcludes` + explicit `--system-prompt` | 不使用個人 `CODEX_HOME`；排除 global `AGENTS.md` |
| 避免 user hooks | `--bare`；managed `allowManagedHooksOnly` / `disableAllHooks` | `--disable hooks`；managed requirements |
| 避免 user rules | managed `allowManagedPermissionRulesOnly` | `--ignore-rules`；managed requirements |
| 避免 user MCP | `--strict-mcp-config --mcp-config empty-mcp.json`；managed MCP | 專用 `CODEX_HOME`；managed MCP allowlist；disable related features |
| 避免 user plugins | `--bare`；`--setting-sources project,local`；managed plugin blocklist | `--disable plugins --disable apps`；專用 `CODEX_HOME` |
| 避免 user skills | `--disable-slash-commands`；`--bare` 降低 auto 行為 | `--disable plugins --disable apps`；專用 `CODEX_HOME` |
| 避免 session/history | `--no-session-persistence`；隔離 `CLAUDE_CONFIG_DIR` | `--ephemeral`；隔離 `CODEX_HOME` |
| 限制 workspace 外讀寫 | permissions deny + external sandbox | permission profile + external sandbox；Windows 需實測 |

### 18.6 最終判斷

不可避免 global 影響的前提：

```text
直接執行開發者機器上的 claude / codex
沿用使用者 home 目錄
不指定 settings sources / CODEX_HOME
不禁用 plugins / skills / hooks / MCP
不使用 managed policy
```

可避免 global 影響的前提：

```text
企業 wrapper 統一產生 CLI command
每個 runner 使用專用 home/config dir
每次任務使用 ephemeral workspace
Claude 使用 --bare 或 --setting-sources project,local + strict MCP + disable slash commands
Codex 使用專用 CODEX_HOME + --ignore-user-config + --ignore-rules + --disable feature set
兩者都用 managed settings / requirements 作不可覆寫策略
兩者都用外部 OS sandbox 作最後讀寫邊界
每次啟動解析 init/doctor/plugin/mcp 輸出作合規 gate
```

企業落地規則：

1. 不允許 production runner 使用工程師個人 `~/.claude` 或 `~/.codex`。
2. 不允許以專案 repo 中的 local 設定作最高信任來源。
3. 不允許只靠 prompt 要求模型不要使用 global skills/plugins/MCP。
4. 不允許只靠 `--tools=`、`--ignore-user-config`、`--ephemeral` 任一單點旗標宣稱隔離完成。
5. 啟動前必跑設定審計；啟動後必解析 init/doctor 事件；執行後必做 side-effect verifier。

## 19. 驗證審核紀錄

本節是針對「是否有未驗證就寫入」的文件審核結果。

### 19.1 已修正

| 位置 | 原問題 | 修正 |
|---|---|---|
| 3.8 Codex 本機關鍵狀態 | `managed_config.toml` 寫成 `%ProgramData%\OpenAI\Codex\managed_config.toml` | 改為官方 Windows/non-Unix 路徑 `%USERPROFILE%\.codex\managed_config.toml` |
| 10.2 Claude managed settings | `strictPluginOnlyCustomization` 使用 `mcpServers`，官方 surface 名稱是 `mcp` | 改為 `["skills", "hooks", "mcp"]` |
| 10.2 Claude managed settings | 範例未標明未部署驗證 | 補上「官方 schema 對齊，尚未本機 managed deployment 實測」 |
| 10.3 Codex `managed_config.toml` | 同一基線把舊 sandbox route 當成唯一建議，容易與 permission profile 路線混用 | 拆成舊 sandbox route 與新版 permission profile route，並明示不可混用 |
| 18.2 Claude managed settings | `disableBypassPermissionsMode` 寫成布林值 | 改為官方值 `"disable"` |
| 18.2 Claude plugin marketplace | `strictKnownMarketplaces` 寫成布林值、`blockedMarketplaces` 寫成 `["*"]`，皆不符合官方型別 | 改為 `strictKnownMarketplaces: []`，並補充既有 plugin 必須用實際 plugin id 驗證 |
| 18.4 Codex 防污染設定 | `managed_config.toml` 路徑寫成 `%ProgramData%` | 改為 `%USERPROFILE%\.codex\managed_config.toml` |

### 19.2 已驗證可保留

| 陳述 | 證據 |
|---|---|
| Claude Code user/global 設定會影響 local | 本機 `claude --print --verbose --output-format stream-json --tools=` init event 顯示 LSP、claude.ai MCP auth tools、MCP servers、slash commands、skills、plugins 仍載入 |
| Claude Code 疊加 `--setting-sources project,local --disable-slash-commands --strict-mcp-config --mcp-config empty --tools=` 可清空 tool/MCP/skill/plugin surface | 本機 init event 顯示 `tools=[]`、`mcp_servers=[]`、`slash_commands=[]`、`skills=[]`、`plugins=[]` |
| Claude Code `--bare` 不讀 keychain/OAuth，本機 OAuth 登入下會失敗 | 本機 `claude --bare --print` 回報 `Not logged in`；`claude --help` 說明 `--bare` 僅用 `ANTHROPIC_API_KEY` 或 `apiKeyHelper` |
| Codex `--ignore-user-config --ignore-rules` 不足以隔離 plugin/skill loader | 本機 stderr 仍出現 `codex_core_skills::loader` warnings |
| Codex 加上 feature disable 後 plugin/skill loader warnings 消失 | 本機 `codex exec --disable plugins --disable apps --disable hooks ...` 只剩 PowerShell shell snapshot 診斷 |
| Codex Windows sandbox 尚未通過本機 smoke test | 本機 `codex sandbox --permissions-profile ':read-only'` / `':workspace'` 均出現 `windows sandbox failed: spawn setup refresh` |
| Codex Linux direct sandbox 可 enforce 自訂 permission profile | Linux 本機 `codex sandbox --permissions-profile enterprise-linux` 六項 smoke test 通過：workspace 內讀寫成功，workspace 外讀寫、`.env` 讀取、network egress 均失敗 |
| Codex Linux direct sandbox 可阻擋 fake SSH / cloud config 讀取 | Linux 本機在 repo 內建立假 `.ssh/id_rsa`、`.aws/credentials`、`.config/gcloud/application_default_credentials.json`，shell `cat` 全部 exit `1` 且 `Permission denied` |
| Codex Linux `codex sandbox` 入口與原測試計畫不同 | 本機 `codex sandbox --help` 沒有 `linux` subcommand；實測命令為 `codex sandbox --permissions-profile ...` |
| Codex 專用 `CODEX_HOME` 若無 auth，`codex exec` 不可完成 agent 測試 | 專用 home `codex doctor --json` 顯示 config parse ok 且 sandbox helpers restricted，但 auth fail；`codex exec --strict-config --ephemeral --json` 回 `401 Unauthorized` |
| Codex 專用 `CODEX_HOME` 有 auth 後，`codex exec` agent-level sandbox 可 enforce profile | 專用 `exec-auth-home` 登入後，`codex exec --strict-config --ephemeral --json` 的 `command_execution` event 顯示 workspace 內讀成功，`.env`、outside、fake SSH/AWS/GCloud、network egress 被阻擋 |
| Codex Linux `/etc/codex/requirements.toml` 會被載入並 enforce config fallback | 暫時部署 root-owned `/etc/codex/requirements.toml` 後，`doctor` / `exec` event 明確標示 `set by /etc/codex/requirements.toml`，並把 disallowed approval / web search 設定 fallback |
| Codex Linux `/etc/codex/managed_config.toml` 中的 permission profile 可被 direct sandbox enforce | 空專用 `CODEX_HOME` 下，`codex sandbox --permissions-profile enterprise-system` 可解析 system managed profile，並阻擋 workspace 外、`.env`、fake SSH/AWS/GCloud、network egress |
| Codex Linux `/etc/codex/managed_config.toml` 中的 permission profile 可被 `codex exec` enforce | 暫時部署 `/etc/codex`，使用已登入專用 `CODEX_HOME` 加 `--ignore-user-config`，`command_execution` event 顯示 boundary script 與 network script 被 sandbox policy 阻擋 |
| Codex direct `sandbox` 不能當作 requirements allowlist gate | 本機 `codex sandbox --permissions-profile ':danger-full-access'` 可執行 harmless `pwd`；加 `--include-managed-config` 仍未被 `allowed_permissions` 擋下 |
| Codex permission profiles 不能與舊 `sandbox_mode` / `[sandbox_workspace_write]` 混用 | 官方 Codex permissions/config reference 明確寫明 |
| Claude Code native Windows 不支援 Bash sandbox | 官方 Claude sandbox 文件明確寫 sandbox runs on macOS/Linux/WSL2，native Windows not supported |
| Claude Code Linux isolated init flags 可清空 MCP / slash commands / skills / plugins surface | Linux 本機 `--setting-sources project,local --disable-slash-commands --strict-mcp-config --mcp-config empty --tools=` init event 顯示 `tools=[]`、`mcp_servers=[]`、`slash_commands=[]`、`skills=[]`、`plugins=[]` |
| Claude Code Linux `--bare` 不能單獨當作 plugin/skill 清空證據 | Linux 本機 `--bare --strict-mcp-config --mcp-config empty --tools=` 因無 API key 回 `Not logged in`，且 init event 仍列出 slash commands、skills 與 `plugins=[figma]` |

### 19.3 仍屬待驗證，不得當完成結論

| 主題 | 目前狀態 | 需要的驗證 |
|---|---|---|
| Claude managed settings 實際部署 | 只做官方 schema 對齊，未寫入 `C:\Program Files\ClaudeCode\managed-settings.json` | 在受控 runner 部署後，用 `/status`、`/doctor`、`/permissions`、`/hooks`、`/mcp` 或 init event 驗證 |
| Claude managed MCP 空檔案 | 官方確認可禁 MCP，但本機未部署 system-level `managed-mcp.json` | 在受控 runner 部署後，驗證 `claude mcp list` 只顯示 managed policy 允許項或空集合 |
| Claude plugin 完全禁用 | 已確認 marketplace 可用 `strictKnownMarketplaces` 管控，但既有 installed plugin 需逐一處理 | 用目標 runner 的 `claude plugin list` 建立 deny/disable 清單，再驗證 init event `plugins=[]` |
| Codex managed defaults / requirements 實際部署 | Linux 本機已暫時部署 `/etc/codex` 並驗證 requirements 載入、managed profile direct sandbox enforce、managed profile `codex exec` enforce；尚未在長駐受控 runner 部署 | 在正式受控 Linux/WSL2 runner 重跑同一套 smoke test，並保留 `/etc/codex` lifecycle / ownership / update audit |
| Codex Linux / WSL2 sandbox | Linux direct sandbox、fake SSH/cloud config boundary、system managed profile direct sandbox、agent-level `codex exec` 已本機實測通過；WSL2、`.git`/`.codex` metadata deny 尚未完成 | 在 WSL2 / enterprise runner 跑 direct sandbox 與 `codex exec` 擴充測試 |
| Codex `--oss --local-provider` 全本地資料處理 | Linux 本機無 Ollama / LM Studio；`codex exec --oss --local-provider ollama` exit `1`，`No running Ollama server detected`；未做無外網封包驗證 | 在 deny-all outbound runner 上只允許 localhost model endpoint，抓 process network event |
| 企業 wrapper side-effect verifier | 目前是架構要求，尚未實作 | 實作 event parser、filesystem diff verifier、network audit、exit-code gate |

### 19.4 審核結論

文件目前仍包含「待目標環境驗證」內容，但已加上證據標記並修正已發現的錯誤 schema / 錯誤路徑。Codex Linux direct sandbox、fake SSH/cloud boundary、system-level requirements 載入、managed profile direct sandbox、agent-level `codex exec` 已移入本機實測；不可把 WSL2、Codex local-only 路線視為已完成驗證。
