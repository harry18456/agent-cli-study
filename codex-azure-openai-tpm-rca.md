# Codex + Azure OpenAI TPM 429 RCA

日期：2026-06-04
範圍：Reeve / Codex CLI 透過 Azure OpenAI `gpt-5.4` deployment 呼叫 `/openai/responses` 失敗

## 結論

Codex + Azure OpenAI 失敗不是 key、endpoint、API version、Codex 版本或內容過濾問題。直接原因是 Azure OpenAI `gpt-5.4` deployment 的 effective token rate limit 為 `4000 TPM / 60s`，而 Codex agentic request 單發會超過這個 bucket，Azure OpenAI `/openai/responses` 會在 admission 階段直接回 `429 too_many_requests`。串流模式下 Codex CLI 只顯示 generic `response.failed`，沒有把 Azure SSE 裡的 `too_many_requests` surface 出來。

更精準地說，這不是「所有 Azure Foundry model 只要 TPM 低就不能送大 prompt」。同一個 Azure resource/domain 下，不同 API surface 的限額執行語意不同：

| API surface | deployment header | 實測請求 | 結果 | 判讀 |
|---|---:|---:|---|---|
| `/openai/responses` for `gpt-5.4` | `4000 TPM`, `40 RPM` | 約 `6K+` estimated tokens / Codex request | `429` / `response.failed` | OpenAI Responses endpoint 會把 TPM bucket 當作硬 admission gate |
| `/anthropic/v1/messages` for Claude | `2000 TPM`, `2 RPM` | `23,006` fresh input tokens, cache `0` | `200` | Anthropic endpoint 不用同一種「單請求 input > TPM bucket 就拒絕」語意 |

所以 Claude 在同一個 domain 能通，不反證 `gpt-5.4` / Codex 的 TPM admission failure。它們共享 domain，但不共享相同的 provider adapter / quota enforcement semantics。

## 已驗證事實

### Azure OpenAI resource / key / deployment 可用

同一把 key、同一個 resource：

```text
endpoint: https://2026msf13.cognitiveservices.azure.com
model: gpt-5.4
```

小請求直接打 Azure Responses API 成功：

| Endpoint | 結果 |
|---|---|
| `/openai/v1/responses` | HTTP `200`, output `pong` |
| `/openai/responses?api-version=2025-04-01-preview` | HTTP `200`, output `pong` |

回應 header 顯示 deployment effective limit：

```text
x-ratelimit-key: gpt-5.4
x-ratelimit-limit-requests: 40
x-ratelimit-renewalperiod-requests: 60
x-ratelimit-limit-tokens: 4000
x-ratelimit-renewalperiod-tokens: 60
```

也測過官方 Codex manual 字面上的 Azure OpenAI hostname：

```text
https://2026msf13.openai.azure.com/openai
https://2026msf13.cognitiveservices.azure.com/openai
```

兩者對小請求都成功，且 rate limit header 相同。

### 大 payload 第一發即 429

為避免小請求先消耗 quota，先等一個 rate-limit window reset，再直接送大 payload。

結果：

```text
body_bytes: 35113
HTTP: 429
error.type: too_many_requests
error.message: Too Many Requests
x-ratelimit-limit-tokens: 4000
x-ratelimit-remaining-tokens: 0
```

再等一個 window 後，送中等 payload：

```text
body_bytes: 15165
HTTP: 200
usage.total_tokens: 3031
output: pong
x-ratelimit-limit-tokens: 4000
x-ratelimit-remaining-tokens: 244
```

這證明問題不是 key / resource / endpoint，而是 request size 與 `4000 TPM` admission gate 的關係。

### Codex 官方 Azure provider config 仍失敗

使用 OpenAI Codex manual 的 Azure provider 形狀建立乾淨 `CODEX_HOME/config.toml`：

```toml
model = "gpt-5.4"
model_provider = "azure"
model_reasoning_effort = "medium"

[model_providers.azure]
name = "Azure"
base_url = "https://2026msf13.cognitiveservices.azure.com/openai"
env_key = "AZURE_OPENAI_API_KEY"
query_params = { api-version = "2025-04-01-preview" }
wire_api = "responses"
request_max_retries = 4
stream_max_retries = 10
stream_idle_timeout_ms = 300000
```

再跑最小命令：

```bash
CODEX_HOME=/home/asus/agent-cli-study/.tmp-codex-official-home \
  codex exec 'Return only pong.'
```

Codex 正確讀到：

```text
model: gpt-5.4
provider: azure
reasoning effort: medium
```

但仍失敗：

```text
ERROR: Reconnecting... 1/10
...
ERROR: stream disconnected before completion: response.failed event received
```

透過本機 proxy 記錄不含 key 的 request 摘要，Azure SSE 真正錯誤是：

```text
event: error
data: {"type":"error","error":{"type":"too_many_requests","code":"too_many_requests","message":"Too Many Requests"}}
```

Codex 官方 config 下實際 request 摘要：

```text
path: /openai/responses?api-version=2025-04-01-preview
content_length: 30952
instructions_len: 14732
tools_len: 11
stream: true
reasoning.effort: medium
```

同樣測過 current user Codex `0.137.0` 與 Reeve production Codex `0.136.0`，都失敗。降版無效。

## 為什麼 Claude 同 domain 卻能通

codebus 側補測 Claude `/anthropic/v1/messages`：

```text
deployment TPM header: 2000
RPM header: 2
input_tokens: 23006
cache_creation: 0
cache_read: 0
HTTP: 200
```

這排除了幾個錯誤解釋：

| 說法 | 判斷 |
|---|---|
| Claude 是另一個 domain 所以不同 | 不成立。它可以是同一個 Azure resource/domain |
| Claude 是靠 prompt cache 才通 | 不成立。fresh request cache `0` 仍成功 |
| Claude prompt 比 Codex 小 | 不成立。Claude fresh input 可達 `23K` tokens |
| 只要 deployment TPM 低就一定擋大 request | 不成立。Claude header `2000 TPM` 仍可接受 `23K` fresh input |

目前最合理且被實測支持的解釋：

```text
同一 Azure resource/domain 下，/openai/responses 和 /anthropic/v1/messages 是不同 API surface。
它們後面接的 provider adapter 與 quota enforcement semantics 不等價。
OpenAI Responses 會把 TPM bucket 當作單次 request admission gate。
Anthropic Messages 不會因單發 fresh input tokens 超過 header TPM 就拒絕。
```

因此，「Claude + Azure 可用」不能用來判定 `gpt-5.4` Azure OpenAI deployment 對 Codex 也應該可用。

## 可能的公司配置背景

使用者提供的預算對應 token 表：

| 類別 | 預算可用 token | 預算小時 | TPM |
|---|---:|---:|---:|
| OpenAI `gpt-5.4` | `12,000,000` | `50` | `4,000` |
| Kimi-K2.5 | `126,000,000` | `50` | `35,000` |
| Claude Opus 4.6 | `6,000,000` | `50` | `2,000` |

`gpt-5.4` 與 Claude 的 TPM 正好等於：

```text
12,000,000 / 50 / 60 = 4,000
6,000,000 / 50 / 60 = 2,000
```

Azure Retail Prices API / Microsoft Foundry 公開價格顯示：

| Model | input price |
|---|---:|
| `gpt-5.4` Global input | 約 `$2.50 / 1M tokens` |
| `gpt-5.4` Data Zone input | 約 `$2.75 / 1M tokens` |
| Claude Opus 4.6 standard input | 約 `$5 / 1M tokens` |
| Kimi K2.5 input | 約 `$0.60-$0.66 / 1M tokens` |

因此 `gpt-5.4` 與 Claude 很像是用每個 model bucket 約 `$30` 反推：

```text
gpt-5.4: $30 / $2.50 * 1M = 12,000,000 tokens
Claude:  $30 / $5.00 * 1M = 6,000,000 tokens
```

再 spread 到 `50h`：

```text
12,000,000 / 50h / 60m = 4,000 TPM
6,000,000 / 50h / 60m = 2,000 TPM
```

如果每人總額約 NT$3000，拆成 3 個 model bucket，則每 bucket 約 US$30，總額約 US$90，約 NT$2700-3000。這與觀察到的數字相符。

Kimi 這列仍需另外確認，因為：

```text
126,000,000 / 50 / 60 = 42,000 TPM
```

不是表中的 `35,000 TPM`。Kimi 可能有不同折扣、不同供應商價格、保留係數，或表格另有換算邏輯。

## 為什麼之前可能能用

目前最合理的歷史解釋：

```text
公司原本可能已有每人預算 / model bucket / 50h pacing 的設計，
但 gpt-5.4 deployment 的 4000 TPM hard limit 之前沒有正確套用，
或 Reeve/Codex 之前實際走到另一個較高 TPM deployment。
最近幾天設定被修正或同步後，4000 TPM 真的成為 Azure OpenAI endpoint 的 effective hard limit，
Codex 單發 agent request 因此開始穩定 429。
```

其他可能但較次要：

1. Codex / Reeve 最近 request payload 變大，例如 tools、instructions、managed policy 或 feature surface 增加。
2. Azure OpenAI `/openai/responses` 最近 enforcement 變嚴。
3. Reeve retry 行為使 throttling 更容易被放大。

但因為 response header 的 `4000 TPM` 正好吻合公司預算換算表，最優先應檢查公司 quota 配置是否剛被套用或修正。

## 已排除項目

| 假設 | 結論 |
|---|---|
| Azure key 無效 | 排除。小請求 HTTP `200` |
| endpoint / API version 錯誤 | 排除。`/openai/v1/responses` 與 preview endpoint 都成功 |
| model deployment 不存在 | 排除。小請求成功，header `x-ratelimit-key: gpt-5.4` |
| Codex 版本 regression | 排除。`0.136.0`、`0.137.0` 都同樣失敗 |
| 官方 Codex Azure provider config 用錯 | 排除。照官方 config.toml 型態仍失敗 |
| 內容過濾 | 不支持。無害大 payload 也 429；小 payload content filter clean |
| Claude cache 幫忙 | 排除。fresh Claude input `23,006` tokens、cache `0` 仍成功 |
| Claude 成功代表 Azure resource 沒 quota 問題 | 不成立。不同 API surface quota semantics 不同 |

## 建議修法

### 1. 不要把成本 pacing 直接落成過低 Azure OpenAI hard TPM

成本預算應由上層 budget controller / usage meter 控制，不應直接把 `US$30 / 50h` 換算成 `4000 TPM` 套到 Azure OpenAI deployment hard limit。對 `/openai/responses` 來說，這會變成單請求 admission ceiling，而不是平滑的花費節流。

### 2. 調高 `gpt-5.4` deployment TPM

`gpt-5.4` deployment 的 effective TPM 必須高於 Codex 單發 request 的 estimated tokens。

本次觀察到 Codex 官方 config request：

```text
content_length: 30952 bytes
instructions_len: 14732 chars
tools_len: 11
```

實務建議：

| 用途 | 建議 TPM |
|---|---:|
| 最小可用驗證 | `20K-30K TPM` |
| Reeve / Codex controlled pilot | `60K+ TPM`，視 concurrent review 與 retry policy 調整 |
| 多使用者或多 PR 併發 | 依 peak concurrency 設計，不應只用 50h average pacing |

### 3. Reeve / codebus 應 surface Azure SSE error

目前 Codex CLI 對上游 SSE `event: error` 的 `too_many_requests` 顯示成：

```text
stream disconnected before completion: response.failed event received
```

這會誤導排查方向。wrapper 應盡可能保留 provider error code：

```text
provider=azure-openai
endpoint=/openai/responses
error.type=too_many_requests
error.code=too_many_requests
request_id=<x-request-id/apim-request-id>
rate_limit_tokens=4000
```

### 4. 降低 retry storm

當錯誤是 `too_many_requests` 且單發 request 本身大於 bucket 時，立即重試不會成功，只會持續打滿 window。應調整為：

1. 讀 `retry-after` / `retry-after-ms`。
2. 若 request estimated tokens > observed `x-ratelimit-limit-tokens`，直接 fail fast，提示需要提高 deployment TPM。
3. 避免短時間 5-10 次 streaming reconnect。

### 5. 減少 payload 只能作輔助

可以檢查是否有不必要 tools / apps / web_search / instructions，但這不是根本修法。Codex agentic request 天然包含 system/developer instructions 與 tool schemas，不能期待它穩定低於 `4000 TPM`。

## 建議給 owner 的一句話

```text
這不是 key 或 Codex 版本問題。Azure OpenAI gpt-5.4 deployment 目前 effective limit 是 4000 TPM，且 /openai/responses 會把這個 TPM 當成單次 request admission gate；Codex agent request 單發超過 bucket，所以必然 429。Claude 雖同 domain 可用，但 /anthropic/v1/messages 的 quota enforcement semantics 不同，不能類比。請把 gpt-5.4 deployment TPM 調高，並把成本 pacing 從 provider hard TPM 拆到上層 budget controller。
```

## 參考來源

- OpenAI Codex manual, Azure provider config: https://developers.openai.com/codex/codex-manual.md
- Microsoft Azure OpenAI Responses API: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/responses
- Microsoft Azure OpenAI quota management: https://learn.microsoft.com/en-us/azure/foundry-classic/openai/how-to/quota
- Microsoft Azure OpenAI quotas and limits: https://learn.microsoft.com/en-us/azure/foundry/openai/quotas-limits
- Azure Retail Prices API: https://learn.microsoft.com/en-us/rest/api/cost-management/retail-prices/azure-retail-prices
- Microsoft Foundry cost management: https://learn.microsoft.com/en-us/azure/foundry/concepts/manage-costs
- Microsoft Foundry Claude model announcement / pricing note: https://devblogs.microsoft.com/foundry/whats-new-in-microsoft-foundry-feb-2026/
- Microsoft Foundry Kimi K2.5 announcement / pricing note: https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/kimi-k2-5-now-in-microsoft-foundry/4492321
