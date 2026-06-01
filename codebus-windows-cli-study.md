# codebus Windows CLI 研究

日期：2026-06-01
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
| codebus 內建整合成熟度 | 較高；工具 allowlist、PreToolUse hook、MCP/user setting 隔離較明確 | 可用；JSONL 與 sandbox 整合好，但 Windows read boundary 弱 |
| Windows native 安全邊界 | 不足；Claude native Windows Bash sandbox 不支援 | 不足；unelevated 可擋部分寫入，但可讀 workspace 外 |
| codebus 目前防線 | `.codebus` cwd、PII mirror、tool allowlist、hooks、`--setting-sources project,local`、empty MCP | `.codebus` cwd、PII mirror、`--ignore-user-config`、feature disable、`-s` sandbox、`windows.sandbox=unelevated`、`web_search=disabled` |
| 主要風險 | 若允許 raw Claude root session，user/global hooks/plugins/skills/MCP 會污染 | Windows unelevated 不能硬擋讀 user home / sibling repo |
| 企業建議 | Windows 可作受控人工輔助；硬邊界需外部 runner | 優先搬 Linux/WSL2 runner；Windows native 要加 AppContainer/VM/ACL |

codebus 在企業內使用時，不應把安全宣告寫成「CLI 已保證 repo 隔離」。正確宣告是：codebus 有 application-level 降風險設計，但企業級 hard boundary 必須由 runner/OS/network policy 補上。

## codebus 當前狀態

本機檢查：

| 檢查 | 結果 |
|---|---|
| `git -C D:\side_project\codebus status --short --branch` | `main...origin/main [ahead 4]`，無未提交變更 |
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
| hooks 可做 PreToolUse gate | Bash/Read 可用 codebus 自己的 hook |
| stream-json 可 audit | init/tool/result/usage 可解析 |

缺點：

| 項目 | 說明 |
|---|---|
| native Windows hard sandbox 不成立 | Claude 官方 sandbox 支援 macOS/Linux/WSL2；native Windows 不應當 hard boundary |
| raw root session 風險高 | 不經 codebus wrapper 時 user/global surface 會載入 |
| filesystem 邊界仍需外部層 | Read/Write/Bash 能力本身不能替代 OS deny |
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

目前 codebus 使用舊 `-s` sandbox route，不是新版 managed permission profile route。這點很重要：Codex 官方新版 permission profiles、requirements、managed config 可以成為企業路線，但不能和舊 `-s sandbox_mode` 混在一起當同一個保證。

### 根目錄 Codex direct sandbox 實測

工作目錄：`D:\side_project\codebus`

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
| network | 此次 `:read-only` probe 失敗；仍需以 enterprise egress smoke 覆蓋 HTTP/80、HTTPS/443、loopback、provider allowlist |

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
| `-s` route 與 permission profiles 不等價 | 企業新版控制面需另設計 migration |
| `--ignore-user-config` 不是完整隔離 | auth/HOME/plugin loader/feature flags 仍需專用 `CODEX_HOME` 與 disable set |
| codebus 目前 active provider 是 codex | 在 Windows 使用時，敏感讀取任務風險較高 |
| shell/network 邊界需外部證明 | 不得只根據 `web_search=disabled` 宣稱離線 |

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

企業 runner 不應使用開發者個人的 `~/.codebus`、`~/.claude`、`~/.codex` 作執行環境。應使用專用 service account、專用 `CODEBUS_HOME`、專用 `CODEX_HOME`、managed settings/requirements、短期憑證與可銷毀 workspace。

## swap 影響

| swap 面向 | 影響 |
|---|---|
| tool model | Claude 的 `--tools`/hooks 不是 Codex 的 permission profile |
| filesystem model | Codex `-s read-only` 在 Windows 不是讀取邊界；不能等價轉成 Claude read-only |
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

## 企業上線高標準

最低 gate：

| Gate | 要求 |
|---|---|
| 版本固定 | `claude --version`、`codex --version`、`codebus --version` 固定並記錄 |
| config root | `CODEBUS_HOME`、`CODEX_HOME` 使用 runner 專用目錄 |
| credential | service account / short-lived token；不得使用個人 OAuth/session |
| init surface | Claude/Codex init/doctor/event 必須符合 allowlist |
| filesystem smoke | workspace 內讀寫、workspace 外讀寫、user home secret、sibling repo、`.git`、`.codebus` 全部測 |
| network smoke | deny-all、provider allowlist、MCP allowlist、loopback、HTTP/80、HTTPS/443 全部測 |
| persistence smoke | session/history/cache/memory 不跨任務 |
| audit | 保存 JSONL、exit code、sandbox denial、changed paths |
| upgrade gate | 每次 CLI/codebus 升級重跑 help diff + smoke |

建議架構：

```text
enterprise orchestrator
  -> disposable workspace
  -> dedicated service account
  -> CODEBUS_HOME / CODEX_HOME / Claude managed settings
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
| codebus + Claude native Windows | 可做受控工具面；不是 hard filesystem sandbox |
| codebus + Codex native Windows unelevated | 可做弱寫入控制；不能擋讀 user home/sibling repo |
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

## 本機證據摘要

| 證據 | 結果 |
|---|---|
| `claude --version` | `2.1.159 (Claude Code)` |
| `codex --version` | `codex-cli 0.135.0` |
| `codebus.exe --repo D:\side_project\codebus --help` | CLI 可執行 |
| `codebus.exe --repo D:\side_project\codebus lint --format json` | `error_count=0,warn_count=0` |
| raw Claude root default init | hooks/MCP/skills/plugins/user memory 會載入 |
| raw Claude root isolated init | tools/MCP/slash/skills/plugins/hook 清為 0 |
| Codex root `:workspace` read user home | `USER_AGENTS_READ_OK` |
| Codex root `:workspace` read sibling repo | `SIBLING_REPO_READ_OK` |
| Codex root `:read-only` write root | `ROOT_WRITE_BLOCKED` |
| Codex root `:read-only` network | sandbox 內 `curl https://example.com` 失敗，sandbox 外成功 |

## 待補

1. 用專用 `CODEBUS_HOME` 跑 codebus mock provider end-to-end，確認 `.codebus` cwd、hooks、event log、changed paths。
2. 在 Windows 受控 runner 部署 Claude managed settings，驗證 codebus wrapper 不受 user/global source 污染。
3. 在 Windows 受控 runner 重測 Codex elevated sandbox 與新版 permission profiles，不與舊 `-s` route 混用。
4. 在 WSL2/Linux runner 跑 codebus + Codex end-to-end，驗證 permission profile、network deny、fake SSH/AWS/GCloud 讀取。
5. 評估 codebus Codex backend 是否要從舊 `-s` route migration 到 managed permission profile route。
