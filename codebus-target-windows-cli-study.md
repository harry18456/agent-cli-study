# codebus target Windows CLI 研究

日期：2026-06-01
目標路徑：`D:\side_project\codebus\target`
CLI 版本：Claude Code `2.1.159`，OpenAI Codex CLI `0.135.0`

## 結論

`D:\side_project\codebus\target` 不應當成一般 source workspace。它是 `codebus` 的 build artifact 目錄，包含 `release\codebus.exe`、`release\codebus-app.exe`、`release\mock-claude.exe`、debug/release 產物、舊 PoC、nested plugin/MCP/AGENTS 測試 fixture。從這裡啟動 Claude Code 或 Codex，工作目錄是 `target`，但 git root 仍是父層 `D:\side_project\codebus`。

企業用法判斷：

| 項目 | Claude Code on Windows | Codex on Windows |
|---|---|---|
| 從 `target` 啟動可用性 | 可用；`--print --verbose --output-format stream-json` 可觀察 init surface | 可用；`codex sandbox` 可在 `target` 跑 direct sandbox smoke |
| 預設污染 | 高；本機預設載入 user hook、skills、plugins、claude.ai MCP auth tools | 高；一般 user `CODEX_HOME` 可含 config、rules、plugins、skills、MCP、auth |
| `target` 邊界 | 無 OS 級讀寫邊界；允許 Read/Write/Bash 時可碰絕對路徑 | Windows unelevated 可做弱寫入控制，但不能當 `target` 讀取邊界 |
| 清掉 global surface | `--setting-sources project,local --disable-slash-commands --strict-mcp-config --mcp-config <empty> --tools=` 可清空 tools/MCP/skills/plugins surface | 專用 `CODEX_HOME` 加 `--ignore-user-config --ignore-rules --disable ...` 才接近乾淨 |
| 企業硬邊界 | 需 managed settings 加外部 OS sandbox/VM/ACL | 需 permission profile 加 Windows elevated sandbox/WSL2/Linux/AppContainer/VM；本機 Windows elevated 未通過 |
| 本機 Windows 高標準結論 | 未達企業級安全邊界 | 未達企業級安全邊界 |

實務建議：

1. 若目標是操作 codebus 應用，不要讓 agent 直接把 `target` 當 repo。應執行 `target\release\codebus.exe --repo <真正 source repo>`。
2. 若目標是研究 build output，可把 `target` 設為只讀資料集，用受控 flags 啟動 agent，但不要宣稱只能讀 `target`。
3. Windows 上要達企業級，CLI flags 只能當第二層。第一層必須是受控 runner、專用 HOME、外部 filesystem sandbox、network egress control、audit。
4. Codex 在 Linux/WSL2 上比 native Windows 更適合做標準企業 runner；Windows 若要硬隔離，優先研究 AppContainer 或 VM/WSL2。

## target 目錄事實

本機檢查結果：

| 檢查 | 結果 |
|---|---|
| `git rev-parse --show-toplevel` from `target` | `D:/side_project/codebus` |
| `target` top-level `CLAUDE.md` | 不存在 |
| `target` top-level `AGENTS.md` | 不存在 |
| `target` top-level `.mcp.json` | 不存在 |
| 父層 `D:\side_project\codebus\CLAUDE.md` | 存在 |
| 父層 `D:\side_project\codebus\.claude` | 存在 |
| 父層 `D:\side_project\codebus\.codebus` | 存在 |
| `target\release\codebus.exe --help` | 可執行，列出 `init/goal/query/lint/fix/chat/quiz/config` |
| `target\release\mock-claude.exe --help` | exit `0`，無輸出 |

`target` 內含 nested fixture：

| 類型 | 例子 | 風險 |
|---|---|---|
| plugin clone | `target\codex-hook-spike\run*\home\.tmp\plugins...` | 掃描整個 `target` 會讀到測試 plugin、skills、MCP |
| `.mcp.json` | nested plugin fixture 內存在 | 不能把 `target` 遞迴內容視為乾淨 policy source |
| `AGENTS.md` | nested plugin skill fixture 內存在 | agent 掃描或 grep 可能讀到非專案指令 |
| PoC scripts | `poc_route0_exec.py`、`appcontainer_poc.ps1` | 是安全研究材料，不是產品 runtime |

## codebus.exe 使用方式

`target\release\codebus.exe` 是已建置的 CLI。從 `target` 直接執行時，預設 repo 會是 `target`，這通常不是想要的行為。

建議使用：

```powershell
D:\side_project\codebus\target\release\codebus.exe `
  --repo D:\side_project\codebus `
  query "檢查 wiki 狀態"
```

研究 build output 時才使用：

```powershell
D:\side_project\codebus\target\release\codebus.exe `
  --repo D:\side_project\codebus\target `
  query "分析 release 產物與 PoC 檔案"
```

判斷：

| 用法 | 結論 |
|---|---|
| `codebus.exe` cwd=`target`、不指定 `--repo` | 容易誤把 build output 當 source repo |
| `codebus.exe --repo D:\side_project\codebus` | 正確操作 codebus source repo |
| `codebus.exe --repo D:\side_project\codebus\target` | 只適合做 artifact 研究，不適合一般 codebus 工作流 |

## Claude Code 從 target 啟動

### 本機實測

命令：

```powershell
claude --print --verbose --output-format stream-json --max-turns 1 --tools= `
  "Reply exactly TARGET_CLAUDE_OK"
```

工作目錄：`D:\side_project\codebus\target`

觀察：

| 欄位 | 結果 |
|---|---|
| `cwd` | `D:\side_project\codebus\target` |
| `tools` | 即使 `--tools=`，仍出現 LSP 與 claude.ai MCP auth tools |
| `mcp_servers` | claude.ai Google Drive/Figma/Gmail/Calendar `needs-auth` |
| `slash_commands` | 大量 user/plugin/builtin commands |
| `skills` | 大量 user/plugin skills |
| `plugins` | user plugins，包括 `gopls-lsp`、`frontend-design`、`skill-creator`、`superpowers` |
| `memory_paths.auto` | `C:\Users\harry\.claude\projects\D--side-project-codebus\memory\` |
| hook | `SessionStart` hook 執行並注入 user plugin context |

關鍵結論：從 `target` 啟動，Claude Code 仍把 session 綁到父層 `D:\side_project\codebus` 的專案記憶路徑。`--tools=` 只清工具清單，不會阻止 user hooks、plugins、skills、MCP、memory path。

隔離 flags 實測：

```powershell
$mcp = Join-Path $env:TEMP 'empty-mcp-agent-study.json'
Set-Content -LiteralPath $mcp -Encoding utf8 -Value '{"mcpServers":{}}'

claude --print "Reply exactly TARGET_CLAUDE_ISOLATED_OK" `
  --verbose `
  --output-format stream-json `
  --max-turns 1 `
  --tools= `
  --setting-sources project,local `
  --disable-slash-commands `
  --strict-mcp-config `
  --mcp-config $mcp

Remove-Item -LiteralPath $mcp -Force
```

結果：

| 欄位 | 結果 |
|---|---|
| `tools` | `[]` |
| `mcp_servers` | `[]` |
| `slash_commands` | `[]` |
| `skills` | `[]` |
| `plugins` | `[]` |
| `memory_paths.auto` | 仍是 `C:\Users\harry\.claude\projects\D--side-project-codebus\memory\` |

隔離 flags 可清空 tool/MCP/skill/plugin surface，但不等於 OS sandbox，也不等於完全不碰 user-level product state。

### Claude Code 建議啟動模板

無工具、觀察 init surface：

```powershell
claude --print "回覆固定字串" `
  --verbose `
  --output-format stream-json `
  --max-turns 1 `
  --tools= `
  --setting-sources project,local `
  --disable-slash-commands `
  --strict-mcp-config `
  --mcp-config $env:TEMP\empty-mcp.json `
  --no-session-persistence
```

只允許讀取的研究模式：

```powershell
claude --print "只分析 target 內檔案，不得修改" `
  --verbose `
  --output-format stream-json `
  --tools=Read,Grep,Glob `
  --allowedTools=Read,Grep,Glob `
  --permission-mode dontAsk `
  --setting-sources project,local `
  --disable-slash-commands `
  --strict-mcp-config `
  --mcp-config $env:TEMP\empty-mcp.json `
  --no-session-persistence
```

`--bare` 模式：

```powershell
$env:ANTHROPIC_API_KEY = '<enterprise-key-or-short-lived-token>'

claude --bare --print "只分析 target 內檔案" `
  --verbose `
  --output-format stream-json `
  --tools=Read,Grep,Glob `
  --allowedTools=Read,Grep,Glob `
  --permission-mode dontAsk
```

限制：

| 控制 | 效果 | 不足 |
|---|---|---|
| `--tools=` | 清空工具清單 | 不清 hooks/plugins/MCP/skills |
| `--setting-sources project,local` | 排除 user settings | 仍不是 OS sandbox |
| `--disable-slash-commands` | 清 skills/slash commands surface | 不限制 Read/Write 的 filesystem scope |
| `--strict-mcp-config --mcp-config <empty>` | 清 MCP servers | 不管 plugins/hooks |
| `--no-session-persistence` | 不持久化該 session | init 仍可能使用 product state path |
| `--bare` | 官方說會跳過 hooks、LSP、plugin sync、auto-memory、keychain、`CLAUDE.md` discovery 等 | 本機 OAuth 登入下不能直接用；需 API key 或 apiKeyHelper |

### Claude Code 優缺

優點：

| 項目 | 說明 |
|---|---|
| init event 可觀察 | `stream-json --verbose` 可看 tools/MCP/skills/plugins/hook surface |
| tool allowlist 直觀 | `--tools`、`--allowedTools`、permission mode 易於包成 wrapper |
| settings 隔離能力較成熟 | `--setting-sources`、managed settings、managed MCP、permission rules 可形成治理層 |
| 適合 code review / artifact research | 在沒有 OS sandbox claim 的前提下，生產力較高 |

缺點：

| 項目 | 說明 |
|---|---|
| native Windows 無內建 Bash sandbox | 官方 sandbox 文件列 macOS/Linux/WSL2，不包含 native Windows |
| 預設 global 污染強 | user hooks/plugins/skills/MCP 會進 local session |
| Read/Write/Bash 不是 workspace 邊界 | 允許工具後可碰絕對路徑，需外部隔離 |
| `target` 會被父層 project state 影響 | 本機 memory path 對應 `D:\side_project\codebus` |

## Codex 從 target 啟動

### 本機 direct sandbox 實測

從 `target` 啟動，git root：

```powershell
git rev-parse --show-toplevel
```

結果：`D:/side_project/codebus`

讀父層 `CLAUDE.md`：

```powershell
codex sandbox `
  -c 'windows.sandbox="unelevated"' `
  --permissions-profile ':workspace' `
  -C D:\side_project\codebus\target `
  powershell -NoProfile -NonInteractive -Command `
  "Get-Location; Get-Content -LiteralPath D:\side_project\codebus\CLAUDE.md -TotalCount 1"
```

結果：成功輸出 `D:\side_project\codebus\target` 與父層 `# CLAUDE.md`。

讀 user global `AGENTS.md`：

```powershell
codex sandbox `
  -c 'windows.sandbox="unelevated"' `
  --permissions-profile ':workspace' `
  -C D:\side_project\codebus\target `
  powershell -NoProfile -NonInteractive -Command `
  "Get-Content -LiteralPath C:\Users\harry\.codex\AGENTS.md -TotalCount 1"
```

結果：可讀取。內容不列入文件，避免保存 user/global 指令文本。

寫入 target，`:read-only`：

```powershell
codex sandbox `
  -c 'windows.sandbox="unelevated"' `
  --permissions-profile ':read-only' `
  -C D:\side_project\codebus\target `
  powershell -NoProfile -NonInteractive -Command `
  "try { Set-Content -LiteralPath .\codex_readonly_write_probe.tmp -Value SHOULD_NOT_WRITE -ErrorAction Stop; 'WRITE_OK' } catch { 'WRITE_BLOCKED:' + $_.Exception.Message }"
```

結果：`WRITE_BLOCKED:Access to the path ... is denied.`

網路 probe：

```powershell
codex sandbox `
  -c 'windows.sandbox="unelevated"' `
  --permissions-profile ':read-only' `
  -C D:\side_project\codebus\target `
  curl.exe -I --max-time 8 https://example.com
```

結果：sandbox 內失敗，且 stderr 顯示嘗試經 `127.0.0.1` 連線；sandbox 外同一 `curl.exe` 可得 HTTP 200。此結果只證明本命令在 sandbox 內未成功連線，不足以單獨宣稱完整企業 egress policy 已完成。

### Codex 建議啟動模板

清掉一般 user config / plugin surface 的 agent run：

```powershell
$env:CODEX_HOME = 'D:\agent-runner\codex-home'

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
  -C D:\side_project\codebus\target `
  -c 'web_search="disabled"' `
  -c 'default_permissions=":workspace"' `
  -c 'windows.sandbox="unelevated"' `
  "只分析 target 目錄，不得修改 workspace 外檔案"
```

direct sandbox smoke：

```powershell
codex sandbox `
  -c 'windows.sandbox="unelevated"' `
  --permissions-profile ':workspace' `
  -C D:\side_project\codebus\target `
  powershell -NoProfile -NonInteractive -Command Get-Location
```

自訂 permission profile 不應和舊 `-s/--sandbox` 混用。若使用新版 permission profile，企業 runner 應用 managed config/requirements 管控；若使用舊 `-s workspace-write`，它只能視為舊 sandbox route，不是 deny-read profile route。

### Codex 優缺

優點：

| 項目 | 說明 |
|---|---|
| JSONL 易整合 | `codex exec --json` 適合應用包裝與 audit |
| `--ephemeral` | 降低 session 持久化 |
| permission profiles | 官方新版控制面可描述 filesystem/network/profile policy |
| Linux/WSL2 路線較強 | 官方 Linux sandbox 使用 `bwrap` 加 `seccomp`；本 repo 既有 Linux 實測已通過多項 boundary |
| direct sandbox | 可不經模型先測 OS sandbox backend |

缺點：

| 項目 | 說明 |
|---|---|
| Windows native 本機未達標 | 本機 elevated backend 先前有 `spawn setup refresh` 問題；unelevated 不能當讀取邊界 |
| `--ignore-user-config` 不等於完全隔離 | 只是不讀 user config；auth、HOME、plugin/skill loader 仍需專用 HOME 與 feature disable |
| `target` 讀取邊界不存在 | `:workspace` 可讀父層 `CLAUDE.md` 與 user `AGENTS.md` |
| direct sandbox 和 `exec` 需分開驗證 | direct sandbox 通過不等於 agent run 的 prompt/tool/session/auth surface 合格 |
| Windows deny-read 需特別驗證 | 既有 PoC 顯示 unelevated deny-read 不能可靠關閉 user-profile secret read |

## 既有 PoC 對 target 的意義

`target\poc_route0_exec.py`：

| 項目 | 含義 |
|---|---|
| Case A | codebus legacy recipe：`--ignore-user-config`、`-s workspace-write`、`windows.sandbox=unelevated` |
| Case B | real `config.toml` deny-read profile，`default_permissions`，`windows.sandbox=unelevated`，不加 `--ignore-user-config` |
| synthetic secret | `%USERPROFILE%\.codebus-sandbox-read-poc\synthetic-credentials.txt` |
| 測試目的 | 驗證 Windows unelevated 下 Codex `exec` 是否能擋讀 user-profile synthetic secret |
| 文件化結果 | deny-read profile 在 Windows unelevated 路線仍漏讀；不能作企業硬邊界 |

`target\appcontainer_poc.ps1`：

| 項目 | 含義 |
|---|---|
| AppContainer/LowBox | 非 admin 建立 LowBox token |
| workspace grant | 明確 ACL grant 後可讀 workspace |
| user-profile secret | AppContainer child 被拒絕讀取 |
| 企業價值 | Windows native 最值得繼續研究的 no-admin hard read boundary |
| 未完成 | 尚未把真實 `codex.exe`、node/git/rg/ConPTY child process 全部跑進 AppContainer |

判斷：若企業堅持 Windows native，不應只調 Codex flags。應把 AppContainer/VM/WSL2 納入 runner 層。

## swap 影響

| 面向 | Claude -> Codex | Codex -> Claude |
|---|---|---|
| 工具 gating | Claude `--tools/--allowedTools` 語意要轉成 Codex feature disable + permission profile；不能只轉 prompt | Codex `-s/default_permissions` 不能等價轉成 Claude `--tools`，因 Claude native Windows 沒 OS sandbox |
| global state | Claude user hooks/plugins/skills/MCP 需用 setting-sources、slash、MCP、managed settings 清掉 | Codex user config/plugins/skills/MCP 需用專用 `CODEX_HOME`、ignore config、disable features、managed policy 清掉 |
| filesystem boundary | Claude 需要外部 sandbox | Codex Windows 也需要外部 sandbox；Linux/WSL2 才能把 permission profile 當主力候選 |
| output parsing | Claude stream-json event 結構和 hooks/init event 可見 | Codex JSONL 更適合 event audit，但 stderr 仍要分流 |
| enterprise acceptance | Claude 強在工具面治理與可觀察性 | Codex 強在 Linux sandbox/permission profile 路線 |

不能做的 swap：

1. 不得把 Claude `--tools=Read` 等同 Codex `:read-only`。
2. 不得把 Codex `:workspace` 等同 Claude workspace filesystem policy。
3. 不得把 `--ignore-user-config` 等同 Claude `--setting-sources project,local`。
4. 不得把 `--ephemeral` 等同「不讀全域狀態」。
5. 不得把 `target` 當 repo root；兩者都可能受父層 repo 或 user/global state 影響。

## 企業上線 gate

最低可接受 gate：

| Gate | Claude Code | Codex |
|---|---|---|
| 版本固定 | `claude --version` | `codex --version` |
| init surface | `stream-json init` 必須 `tools/mcp/skills/plugins` 符合 allowlist | `codex doctor --json`、`codex exec --json` header/event 符合 allowlist |
| user/global 隔離 | managed settings + `--setting-sources` + empty managed MCP | 專用 `CODEX_HOME` + managed config/requirements + `--disable` feature set |
| filesystem smoke | workspace 內讀、workspace 內寫、workspace 外讀、workspace 外寫、user secret read | 同左，direct sandbox 和 `exec` 都要跑 |
| network smoke | deny-all、provider allowlist、MCP allowlist | deny-all、provider allowlist、MCP/proxy allowlist |
| persistence smoke | session/history/memory/cache 不跨任務污染 | session/history/memory/cache 不跨任務污染 |
| upgrade gate | help diff + smoke rerun | help diff + permission profile smoke rerun |

高標準架構：

```text
企業 orchestrator
  -> 專用 service account / short-lived credentials
  -> 專用 HOME / CODEX_HOME / CLAUDE config root
  -> managed settings / requirements
  -> CLI wrapper 編譯 policy 到 flags/config
  -> OS sandbox: Linux/WSL2 container、VM、AppContainer、ACL broker
  -> egress firewall / proxy allowlist
  -> JSON event audit + side-effect verifier
  -> cleanup / workspace destroy
```

Windows 現階段判斷：

| 路線 | 判斷 |
|---|---|
| Claude native Windows only | 不足；可當受控工具面，但缺 OS boundary |
| Codex native Windows unelevated only | 不足；可擋部分寫入，不能當讀取邊界 |
| Codex native Windows elevated | 理論上較完整，但本機未通過 smoke；需受控 runner 重測 |
| Codex on WSL2/Linux | 目前最適合進企業標準化驗證 |
| Windows AppContainer wrapper | 值得 spike；PoC 已證明 primitive，但 real Codex last-mile 未完成 |

## 官方來源

| 主題 | 來源 |
|---|---|
| Claude Code CLI reference | <https://docs.anthropic.com/en/docs/claude-code/cli-reference> |
| Claude Code settings / managed settings | <https://docs.anthropic.com/en/docs/claude-code/settings> |
| Claude Code hooks | <https://docs.anthropic.com/en/docs/claude-code/hooks> |
| Claude Code sandbox environments | <https://docs.anthropic.com/en/docs/claude-code/sandbox> |
| OpenAI Codex CLI | <https://developers.openai.com/codex/cli> |
| OpenAI Codex CLI reference | <https://developers.openai.com/codex/cli/reference> |
| OpenAI Codex permissions | <https://developers.openai.com/codex/permissions> |
| OpenAI Codex Windows sandbox | <https://developers.openai.com/codex/windows> |
| OpenAI Codex config | <https://developers.openai.com/codex/config> |
| OpenAI Codex managed configuration | <https://developers.openai.com/codex/enterprise/managed-configuration> |

## 本機證據

| 證據 | 結果 |
|---|---|
| `claude --version` | `2.1.159 (Claude Code)` |
| `codex --version` | `codex-cli 0.135.0` |
| `target\release\codebus.exe --help` | CLI 可執行，列出 verbs |
| `git rev-parse --show-toplevel` from `target` | 父層 repo root：`D:/side_project/codebus` |
| Claude default `stream-json` from `target` | 載入 user hook、MCP auth tools、skills、plugins，memory path 指向父層 codebus project |
| Claude isolated flags from `target` | `tools=[]`、`mcp_servers=[]`、`slash_commands=[]`、`skills=[]`、`plugins=[]` |
| Codex `:workspace` direct sandbox from `target` | 可讀父層 `D:\side_project\codebus\CLAUDE.md` |
| Codex `:workspace` direct sandbox from `target` | 可讀 user `C:\Users\harry\.codex\AGENTS.md` |
| Codex `:read-only` direct sandbox from `target` | 寫入 `target` 被拒絕 |
| Codex `:read-only` network probe | sandbox 內 `curl https://example.com` 失敗；sandbox 外成功 |

## 待補

1. 在受控 Windows runner 部署 Claude managed settings 後，驗證 plugins、managed MCP、permission rules 是否完全符合 allowlist。
2. 在受控 Windows runner 重跑 Codex elevated backend smoke，不用一般 user `CODEX_HOME`。
3. 把真實 `codex.exe` 包入 AppContainer，驗證 node/git/rg/ConPTY child process、workspace ACL、network capability。
4. 在 WSL2 runner 重跑 Linux permission profile、fake SSH/AWS/GCloud、network deny、`codex exec` agent-level smoke。
5. 對 `codebus.exe --repo D:\side_project\codebus` 跑 dry-run/mock provider 測試，確認 app wrapper 不會把 `target` 當預設 repo。
