# temux_llm — 在 Android 上跑本地 LLM，仿 Ollama

**開發、Issues 與 PR：** https://github.com/aa22396584/temux_llm  
**鏡像：** [GitLab](https://gitlab.com/aa22396584/temux_llm) · [Codeberg](https://codeberg.org/ImL1s/temux_llm)


[English](README.md) · [繁體中文](README.zh-TW.md)

[![build](https://github.com/aa22396584/temux_llm/actions/workflows/build.yml/badge.svg)](https://github.com/aa22396584/temux_llm/actions/workflows/build.yml)
[![release](https://img.shields.io/github/v/release/aa22396584/temux_llm?include_prereleases)](https://github.com/aa22396584/temux_llm/releases)
[![license](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)
[![android](https://img.shields.io/badge/android-13%2B-3DDC84?logo=android&logoColor=white)](https://developer.android.com/about/versions/13)
[![runtime](https://img.shields.io/badge/runtime-LiteRT--LM%200.11.0--rc1-yellow)](https://github.com/google-ai-edge/LiteRT-LM)

一個 self-contained 的 Android app + Termux 客戶端，用你手機的 GPU 跑 Google
**Gemma 4**（或 Qwen3），透過 `http://127.0.0.1:11434/api/generate` 提供服務。
不連網。不上雲。資料不離開裝置。

```
$ litertlm "Reply with just: hi."
hi.

[gpu  total=2676ms  tokens=62  decode=23.2 t/s]
```

實機驗證：**Galaxy S21+**（SD 888 / Adreno 660，Android 15）、
**Galaxy S24 Ultra**（SD 8 Gen 3 / Adreno 750，Android 16）、
**Galaxy S25**（SD 8 Elite / Adreno 830，Android 16）。
理論上任何 arm64-v8a Android 13+ Snapdragon 機都能跑。CPU + GPU 實測數字見
[效能](#效能)。

## 用途 / Use cases

- **離線、不上雲：** 飛航模式、地下鐵、不能讓 prompt 離開裝置的場景，照樣跑 LLM。
- **Ollama-相容 endpoint：** 預設綁 `http://127.0.0.1:11434`（Ollama 標準埠），既有
  腳本/app 不用改就能接。
- **Termux 腳本：** 把 LLM 輸出 pipe 進 shell：`curl -s ... | jq .response` 就能在
  手機上做結構化查詢。
- **離線助理：** Gemma-class 模型常駐記憶體，短 prompt 不到 1 秒回。
- **可換模型：** 任何 `.litertlm` 格式（Gemma、Qwen、未來釋出的）都能換上去。
- **隱私 prototyping：** 早期開發直接打本地 endpoint，不用 API key、不燒 token、
  prompt 不會送到第三方。

**不適合：** 正式 production 流量（單機單租戶）、長 context 工作（最多 8k）、
多人服務（設計上只綁 localhost）。

---

## 開箱即用

從乾淨 clone 出發，手機接上 USB，一行：

```bash
bash scripts/install.sh
```

安裝腳本會：

1. 檢查主機有 `adb`、JDK 17+、設定好的 Android SDK。
2. 自動挑你的手機（或指定 `DEVICE_SERIAL=...`）。
3. 依 SoC 自動挑模型，或設 `MODEL=e2b|e4b|qwen3`：
    - **`e2b`**（舊機預設）：`gemma-4-E2B-it.litertlm`，2.4 GB。
       S21+ 等級的硬體跑得動。
    - **`e4b`**（SD 8 Elite 等級預設 — Fold7、S25）：`gemma-4-E4B-it.litertlm`，
       3.4 GB。比較聰明、慢一點，需要 ≥10 GB RAM 的手機。
    - **`qwen3`**：`Qwen3-0.6B.litertlm`，614 MB。最小的退路；話多。
4. 下載 binary + accelerators + 選定模型（sha256 驗證）。
5. 用內建 Gradle wrapper 編出 debug APK。
6. 把所有東西推到裝置、安裝 APK、啟動 service。
7. 對 `/api/generate` 做 smoke test，印下一步指引。

跑完之後，安裝 Termux（請用 [F-Droid 版][termux-fdroid] — Play Store 那個沒在維護），
把 wrapper 加進 PATH **一次**：

```sh
echo 'export PATH="/data/local/tmp/bin:$PATH"' >> ~/.bashrc
```

之後任何 Termux session 都能用：

```sh
litertlm "你好"
litertlm --backend cpu "Reply OK in 3 words."
litertlm --json "what is 2+2?"            # 輸出原始 JSON 供腳本使用
litertlm --help
```

[termux-fdroid]: https://f-droid.org/packages/com.termux/

---

## 跟 CLI coding agent 一起用（v0.6.0 實測矩陣）

> **v0.6.0 重點**：vision 圖片輸入接到 `/v1/messages` /
> `/v1/chat/completions` / `/api/chat` / `/v1/responses` 四條 wire
> （Galaxy S25 讀出 64×48 PNG 裡的 "STOP" — 不需要 fork）；Gemma 4 E4B
> tool-call 通過率從 **77 % → 100 %**（n=30），靠 port 自 Ollama 的
> stack-based JSON repair；新 endpoint `/api/probe/native_tools` 走
> SDK 原生 tool 路徑也 100 %、快 30 %，v0.7 透過 `temuxllm.conf` 的
> `native_tools=on` 切過去；codex 的 `web_search` 在 `/v1/responses`
> 入口被過濾掉，model 不再瞎掰 hosted-tool 結果。



`temux_llm` 對外協定模仿 Ollama 0.13+，常見 coding agent CLI 可以直接接，
不需要 proxy。下面這張表是 **v0.5.1 在 Galaxy S25（12 GB / 16k context、
default backend GPU）上實機重跑** 的結果，不是「理論上應該能跑」：

| CLI | 實測 | Endpoint | 備註 |
|---|---|---|---|
| **Claude Code** (`claude --bare`) | ✅ | `/v1/messages` (Anthropic SSE) | 直接連；`--bare` 砍掉 ~16k system prompt |
| **OpenAI Codex** 0.125+ | ✅ | `/v1/responses` (OpenAI SSE) | 需要自訂 provider；`wire_api="chat"` 已被廢、`--oss` 強制 gpt-oss:20b。需把 `max_tokens` 降到 12288 |
| **llxprt-code**（Gemini CLI fork） | ✅ | `/v1/chat/completions` | `npm i -g @vybestack/llxprt-code`；原生 OpenAI provider |
| **OpenCode** 1.14+ | ✅ via bridge | `/v1/chat/completions` | 要架 LiteLLM bridge 把 `backend=cpu` 注進去（12 GB 機在 GPU 模式會 OOM） |
| **stock Gemini CLI** | ❌ | — | 找不到能 redirect 到本地端的 env var；Cached OAuth 會 bypass 任何 base URL 蓋掉（見 g-g/gemini-cli #1605, #5945, #24166） |
| **OpenClaw** | v0.5.1 沒重測 | `/api/chat` (Ollama 原生) | wire 是對的；標記到 v0.6.0 重測 |

### 一行 launcher

手機上 service 起來後，host 端先 forward 11434：

```sh
adb forward tcp:11434 tcp:11434
```

然後跑 launcher（行為對標 Ollama 0.15+ 的 `ollama launch <cli>`：
probe 服務、選 model、設好 env / config、`exec` 目標 CLI）：

```sh
scripts/temuxllm launch claude          # Claude Code, --bare
scripts/temuxllm launch codex           # Codex CLI（自訂 temuxllm provider）
scripts/temuxllm launch opencode        # OpenCode（自動起 LiteLLM cpu bridge）
scripts/temuxllm launch gemini          # llxprt-code（Gemini CLI fork）
scripts/temuxllm launch openclaw        # Ollama 原生，v0.5.1 沒重測

scripts/temuxllm launch claude --config-only   # 只印 env block，不 exec
scripts/temuxllm launch codex --model gemma-4-E2B-it
```

### 各 CLI 手動設定

如果你想自己接 env / config，這裡是 **launcher 內部用的精確設定**，
每一份都在 v0.5.1 對著實機跑過才寫進來。

#### Claude Code

```sh
export ANTHROPIC_BASE_URL=http://127.0.0.1:11434
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_API_KEY=
export ANTHROPIC_MODEL=model
export ANTHROPIC_DEFAULT_OPUS_MODEL=model
export ANTHROPIC_DEFAULT_SONNET_MODEL=model
export ANTHROPIC_DEFAULT_HAIKU_MODEL=model
claude --bare           # 一次性：加 --print "your prompt"
```

12 GB / 16k 機必須 `--bare`：完整 Claude Code agent system prompt
~16k token，把 context 整碗吃光、output 沒空間。≥16 GB 機可以拿掉
（並且 set `TEMUXLLM_MAX_TOKENS=24576+`）。

#### Codex CLI 0.125+

Codex 0.125 把舊的 `--oss` 短路給打死了：`wire_api="chat"` 被廢、
`--oss --local-provider ollama` 蓋掉 `-c model=` 強塞 `gpt-oss:20b`、
`model_providers.ollama.*` 是 reserved built-in 不准蓋。Codex 預設
`reasoning_effort="xhigh"` 還會塞 ~5k token 的 `<think>` trace，
12 GB / 16k 裝不下。所以我們用自訂 `temuxllm` provider + 小的
instructions 檔 + `reasoning_effort="minimal"`：

```sh
echo "You are a helpful coding assistant. Reply concisely." > /tmp/tiny.md

codex exec \
  -c project_doc_max_bytes=0 \
  -c 'model_reasoning_effort="minimal"' \
  -c 'model_instructions_file="/tmp/tiny.md"' \
  -c 'model_provider="temuxllm"' \
  -c 'model_providers.temuxllm.name="temuxllm"' \
  -c 'model_providers.temuxllm.base_url="http://127.0.0.1:11434/v1"' \
  -c 'model_providers.temuxllm.wire_api="responses"' \
  -c 'model="model"' \
  "your prompt"
```

**重要**：12 GB 機要把 `max_tokens` 降到 **12288**（16384 在 codex 的
prompt prefill + GPU init 下會 OOM）。直接 push conf：

```sh
adb shell 'echo max_tokens=12288 > /data/local/tmp/litertlm/temuxllm.conf'
adb shell 'am force-stop dev.temuxllm.service'
# 下一個 request 會用新的 ceiling 重啟 engine
```

要蓋 launcher 預設可以用 `TEMUXLLM_CODEX_REASONING=high|low` 或
`TEMUXLLM_CODEX_INSTRUCTIONS_FILE=/path/to/your.md`。

#### llxprt-code（Gemini CLI fork）

**stock `gemini` CLI 沒辦法 redirect 到本地的 OpenAI-compatible
endpoint** — 我們對著 gemini 0.36 + LiteLLM bridge 實測過。
`GOOGLE_GEMINI_BASE_URL` 跟 `GEMINI_API_KEY` 在有 cached OAuth 的時候
會被 ignore，流量還是送到 Google 那邊。對應 feature request
([#1605](https://github.com/google-gemini/gemini-cli/issues/1605)、
[#5945](https://github.com/google-gemini/gemini-cli/discussions/5945)、
[#24166](https://github.com/google-gemini/gemini-cli/discussions/24166))
到 v0.5.1 為止都還沒 merge。

請改用 [`@vybestack/llxprt-code`](https://github.com/vybestack/llxprt-code)
fork，它原生支援 OpenAI / Anthropic / Ollama provider：

```sh
npm install -g @vybestack/llxprt-code

# 一次性
llxprt --provider openai \
       --baseurl http://127.0.0.1:11434/v1 \
       --key ollama \
       --model model \
       --prompt "your prompt"

# 互動式 REPL — 在 prompt 裡設一次 provider
llxprt
# /provider openai
# /baseurl http://127.0.0.1:11434/v1/
# /key ollama
# /model model
```

#### OpenCode 1.14+（透過 LiteLLM bridge）

OpenCode 1.14 的 bare-agent 還是會夾帶 ~2.3k token system prompt。
12 GB 機在 GPU 模式下這個 prompt 加上模型常常會被 `lowmemorykiller` 砍掉。
所以我們把 OpenCode 走過一個 LiteLLM proxy，proxy 會在每個 call 注入
`extra_body: {backend: "cpu"}`，讓裝置端用 CPU 跑這個 request。
launcher 會自動起這個 bridge；手動的話：

```sh
pip install 'litellm[proxy]'

cat > /tmp/tmuxllm-litellm.yaml <<'YAML'
model_list:
  - model_name: model
    litellm_params:
      model: openai/model
      api_base: http://127.0.0.1:11434/v1
      api_key: ollama
      extra_body: { backend: cpu }
litellm_settings: { drop_params: true }
YAML

litellm --config /tmp/tmuxllm-litellm.yaml --port 4000 &

# OpenCode config（寫到一個暫存檔；cd 到 scratch dir 避免 OpenCode
# 走出去找 AGENTS.md / CLAUDE.md 來把 prompt 撐肥回來）
cat > /tmp/opencode.json <<'JSON'
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "temuxllm": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "temux_llm (local)",
      "options": { "baseURL": "http://127.0.0.1:4000/v1", "apiKey": "ollama" },
      "models": { "model": { "name": "model" } }
    }
  },
  "model": "temuxllm/model",
  "default_agent": "bare",
  "agent": {
    "bare": {
      "mode": "primary",
      "model": "temuxllm/model",
      "prompt": "You are a helpful assistant. Reply concisely.",
      "tools": { "write": false, "edit": false, "bash": false, "read": false,
                 "glob": false, "grep": false, "list": false, "patch": false,
                 "todowrite": false, "todoread": false, "webfetch": false,
                 "task": false, "lsp_diagnostics": false, "lsp_hover": false }
    }
  }
}
JSON

(cd "$(mktemp -d)" && \
  OPENCODE_DISABLE_CLAUDE_CODE=1 OPENCODE_CONFIG=/tmp/opencode.json \
  opencode run --agent bare "your prompt")
```

≥16 GB 機可以 `TEMUXLLM_OPENCODE_BACKEND=gpu` 直接連，不走 bridge。

### v0.5.1 work / 不 work 範圍

**會 work**：
- 四種 wire envelope（Anthropic SSE、OpenAI SSE、OpenAI Responses、
  Ollama NDJSON）的純文字跟 tool calling round-trip。
- Tool calling：透過 prompt-injection + brace-balance JSON parser
  （沒有 Conversation::Clone — 單輪 only）。
- Memory observability：`MemoryProbe` inference 時取樣 PSS / VmHWM /
  oom_score_adj。LMK 砍服務後，`AutoFallback` 用 device fingerprint 把
  下次的 `maxNumTokens` 降階。
- 自動 CPU/GPU default：`/data/local/tmp/litertlm/temuxllm.conf` 裡
  `default_backend=` 或 `TEMUXLLM_DEFAULT_BACKEND=` env var。

**不 work / 已知限制**：
- **多輪 agent loop** 在 12 GB / 16k 機上一旦 prompt prefill 超過
  ~10k token，inference 還沒開始就被 LMK 砍。緩解：用 `e2b` 模型 +
  `--bare` 旗標 + CPU default backend。
- **Image input**：`image_url` / `input_image` block 一律回 400。
- **stock Gemini CLI 重 base URL**：沒有 env 路徑；改用 llxprt-code。

### Tool calling、MCP、skills、web search

各 CLI 的 `--bare` 模式設計上會把 auto-discovery 的 tool 砍掉。完整
矩陣（誰留了什麼、怎麼把 web search / MCP / skills 加回來）寫在
[`docs/cli-tool-matrix.md`](docs/cli-tool-matrix.md)。簡版：

| CLI | `--bare` 保留 | 加回 web search / MCP |
|---|---|---|
| `claude --bare` | 只剩 Bash、Read、Edit | `--mcp-config <檔案>` + `--strict-mcp-config` |
| `codex exec` (minimal) | 29 個 built-in + MCP + skills 全在 | 預設都開；`~/.codex/config.toml` 加 MCP server |
| `llxprt` | 全部 built-in + MCP + skills（沒 bare 開關） | `EXA_API_KEY=...` 開 web search；`llxprt mcp add` 接 MCP |
| `opencode run --agent bare` | 零工具（我們的 launcher 把 14 個全關了） | 換成 `--agent build`，把 `webfetch` / `websearch` 開回來 |

單輪 tool calling 4 個 wire envelope 都過（`/api/chat`、
`/v1/chat/completions`、`/v1/messages`、`/v1/responses`）。多輪 agent loop
在 Gemma 4 上不可靠 — v0.6.0 用 `Conversation::Clone()` 加 KV reuse 後
才會穩。

完整 endpoint 對照與 wire format 細節：
[`docs/ollama-compat.md`](docs/ollama-compat.md)。

---

## Termux-native（不用 APK）

不想 sideload APK？那就直接在 Termux 裡跑 LiteRT-LM binary — 不用 USB、
不用 Android service、不用主機。

**在手機的 Termux 裡：**

```sh
# 從你的 clone 複製 installer 過來，或手動 copy
bash install-termux-native.sh          # 預設：gemma-4-E2B-it (2.4 GB)
MODEL=qwen3 bash install-termux-native.sh   # 614 MB 退路
MODEL=e4b   bash install-termux-native.sh   # 3.4 GB，高階 SoC 才能跑
```

Installer 會把所有東西寫到 `~/.litertlm/`，wrapper 放在
`~/.local/bin/litertlm-native`。把 `~/.local/bin` 加進 PATH 一次，然後：

```sh
litertlm-native "你好"
litertlm-native --backend cpu "Reply OK in 3 words."
litertlm-native --json "what is 2+2?"
litertlm-native --help
```

**怎麼選哪一條路：**

| | APK 路徑（`install.sh`） | Native 路徑（`install-termux-native.sh`） |
|---|---|---|
| 安裝環境 | 主機 + USB（一次性） | 只在 Termux 裡 |
| 需要 sideload APK | 要 | 不用 |
| 每次 call 啟動成本 | 不到 1 秒（engine 常駐） | 3-8 秒（暖）/ 10-20 秒（首次） |
| 穩態 decode 速度 | 看 backend（見矩陣） | **CLI CPU 在 Snapdragon 上贏過或追平 APK GPU** |
| GPU 加速 | Adreno 上可用（v0.1.2 起） | 被 Termux 的 vendor namespace 擋掉 |
| 適合場景 | 短互動、要秒回 | 長段生成、無 USB 工作流、腳本 |

**反直覺但實測過：** CLI 的純 CPU decode 在所有測過的 Snapdragon 機上都贏過或
追平 APK 的 GPU 端到端速度（見[效能](#效能)）。原因：CLI 是緊湊的 native
binary，多執行緒 CPU 推理；APK 走 JNI + service + foreground 通知 +
loopback HTTP，每個 call 都要付這些 overhead。APK 還是贏在「每次 call 啟動
延遲」（engine 常駐）；CLI 每次 call 要重新 init engine。

兩條路都用 LiteRT-LM 0.11.0-rc.1（APK 走 Maven artifact
`com.google.ai.edge.litertlm:litertlm-android:0.11.0-rc1`，Termux-native
走 GitHub release `v0.11.0-rc.1` 的 `litert_lm_main` CLI binary），
模型也是同一批。

---

## 包含什麼

```
scripts/install.sh                  一鍵安裝（上面那個）
scripts/install-termux-native.sh    Termux-only 安裝（不用 APK）
scripts/litertlm-native-wrapper.sh  Termux-native CLI wrapper 的原始碼
scripts/fetch_artifacts.sh          sha256 驗證的主機端下載
scripts/litertlm-termux-wrapper.sh  Termux 客戶端（給 APK service 用）
scripts/preflight.sh                主機 + 裝置就緒檢查
scripts/setup_litertlm_android.sh   手動推送（install.sh 內部用）
scripts/run_{cpu,gpu}_smoke.sh      原生 adb-shell smoke（跳過 APK）
scripts/run_litertlm_benchmark.sh   長 prompt benchmark
scripts/parse_litertlm_logs.sh      把 BenchmarkInfo 抽到 logs/summary.txt
scripts/sha256_manifest.txt         binary + .so + 模型的 sha256

android/                            Android Studio 專案（Kotlin）
  app/src/main/kotlin/dev/temuxllm/
    LlmService.kt                   foreground service（持有 engine）
    LlmEngine.kt                    in-process Engine + Conversation 包裝
    HttpServer.kt                   NanoHTTPD on 127.0.0.1:11434
    LauncherActivity.kt             可點的 icon → 啟動 service 然後 finish()
    BootReceiver.kt                 開機自動拉起 service

docs/specs/                         phase-2 架構規格
docs/plans/                         實作 plans（phase-1 / phase-2a）
docs/findings-*.md                  各裝置實測結果
docs/screenshots/                   手機螢幕截圖（Termux 跑起來的樣子）
```

---

## API

```
GET  /healthz       -> "ok"
GET  /api/version   -> {service, phase, runtime, default_backend,
                        model_path, source_model_path, engine_loaded}
GET  /api/tags      -> {models: [{name, path, size_bytes}, ...]}
POST /api/generate  -> NDJSON 串流 {response, done}（stream=false 則回單一 JSON）
```

`/api/generate` request body：

```json
{
  "prompt":  "Hi",
  "backend": "cpu" | "gpu",   // 預設：見下面 Tunables
  "stream":  true             // 預設：true（NDJSON 一 token 一行）
}
```

### Tunables（v0.5.1+）

`/data/local/tmp/litertlm/temuxllm.conf`，每次 engine 重啟都會讀：

```
max_tokens=16384         # KV-cache 上限。8 GB 機建議降到 8192；
                         # 12 GB 機跑 codex 建議 12288
default_backend=cpu      # cpu|gpu。當 request body 沒帶 backend
                         # 欄位時的預設。Codex 跟 OpenCode 都不會
                         # 帶這個欄位 — 12 GB / 16k 機把這個調 cpu
                         # 可以避免 LMK 砍服務。
```

也可以用 `TEMUXLLM_DEFAULT_BACKEND=cpu` env var（檔案不存在時 fallback）。

串流 response：每行一個 JSON object（`application/x-ndjson`）：

```
{"response":"Hi","done":false}
{"response":"!","done":false}
{"response":"","done":true,"backend":"gpu","total_duration_ms":820,"output_tokens":2,"output_chars":3}
```

非串流（`stream=false`）：回一個 document，含 `model`、`backend`、`response`、
`done`、`total_duration_ms`、`output_tokens`。

Service 只綁 `127.0.0.1`。每次重啟後驗證：`ss -tnlp | grep 11434` 必須顯示
`[::ffff:127.0.0.1]:11434`（絕不會是 `0.0.0.0`）。

---

## 限制（沿用最初的 brief）

- 不需要 root。
- 只支援 arm64-v8a。
- HTTP 只綁 localhost。絕不對外暴露。
- Runtime 是 LiteRT-LM 載入 `.litertlm`（不是舊的 `.tflite` / `.task`）。

---

## 效能

`gemma-4-E2B-it`（2.4 GB），50 字 output prompt，暖機後。

| 裝置 | SoC | APK CPU<br>(end-to-end) | APK GPU<br>(end-to-end) | CLI CPU<br>(decode-only) |
|---|---|---|---|---|
| Galaxy S21+ | SD 888 / Adreno 660 | 8.0 t/s | 10.5 t/s | **12.1 t/s** |
| Galaxy S24 Ultra | SD 8 Gen 3 / Adreno 750 | 10.3 t/s | 22.0 t/s | 21.5 t/s |
| Galaxy S25 | SD 8 Elite / Adreno 830 | 12.4 t/s | 23.2 t/s | **35.4 t/s** |

兩個欄位量的東西不一樣，不能直接拿百分比比。各自報自己場景的誠實值：

- **APK end-to-end** = `output_tokens / (total_duration_ms / 1000)`，一次暖
  `/api/generate` 整個來回 — 含 prefill、decode、loopback HTTP/JNI
  overhead，就是使用者實際感受到的延遲。不算常駐 engine 的一次性 init。
- **CLI decode-only** = binary 的 `BenchmarkInfo` 報的 `Decode Speed` —
  engine 開始產 token 之後的穩態吞吐量，跟 prefill / 每次 call init 分開算。

看 CLI 端到端：暖機後每次 call wall time **3-8 秒**（短 prompt 是 init
主導；長段生成把 init 攤掉）。
S25 上 60 token 輸出大約 3 秒 wall ≈ 20 t/s 端到端，vs APK GPU
3 秒 / 62 token ≈ 23 t/s — 短輸出大致打平。CLI 的 decode-only 優勢
（S25 35 t/s）要在生成夠長、init 被攤掉時才看得出來。架構原因見下面
「為什麼 CLI CPU 贏過 APK GPU」。

**每次 call 啟動延遲**是另一個面向：APK 把 `Engine` 常駐記憶體，所以暖機後
`/api/generate` 短回答 200-1000 毫秒。CLI 每次 call 都要重新 init engine，
所以每次 wall 大概是 **3-8 秒（暖）/ 10-20 秒（首次）**。要選哪條路看你
要優化「每次 call 延遲」（APK）還是「穩態 decode 吞吐量」（CLI）。

`gemma-4-E4B-it`（3.4 GB）— 只有 APK 路徑測過：

| 裝置 | E4B CPU | E4B GPU |
|---|---|---|
| Galaxy S21+ | 4.1 t/s | （未測） |
| Galaxy S24 Ultra | 5.7 t/s | （未測） |
| Galaxy S25 | 7.8 t/s | 15.0 t/s |
| Galaxy Note 9 | 不支援（minSdk=33 / Android 13+） | |

**第一次 call 的初始化成本：**

- **APK GPU：** OpenCL kernel-compile 第一次要 8-22 秒。SDK 把
  `model.litertlm_*_mldrift_program_cache.bin` 寫進 app filesDir；之後
  重啟 service 跳過 compile 直接用 cache。重推模型會讓 cache 失效。
- **APK CPU：** XNNPack weight cache 第一次 3-7 秒；暖機降到 0.5-0.8 秒。
- **CLI CPU：** 首次 call 在模型旁邊建 xnnpack cache（~10-20 秒、一次性）。
  之後每 call 只付 engine init（~2-3 秒）。`install-termux-native.sh` 安裝
  最後會自動跑一次暖機，所以使用者第一次互動已經是暖機狀態。

**為什麼 CLI CPU 贏過 APK GPU：** CLI 是緊湊的 native binary，多執行緒
CPU 推理 — 沒 JNI、沒 service framework 開銷、沒 foreground 通知、沒
per-call socket round-trip。APK GPU 要付 SDK 的 JVM↔native 橋接 + 每次
loopback HTTP。長段生成（decode 主導）CLI 的原始吞吐量贏；短互動
（init 主導）APK 的常駐 engine 贏。

---

## 已知問題

### Note 9 / Android < 13

`minSdk=33`（Android 13）。Galaxy Note 9 這種更舊的裝置，APK 安裝會被
`INSTALL_FAILED_OLDER_SDK` 擋下來。沒有計畫降低門檻 — LiteRT-LM SDK 本身就要 API 33+。

### 非 Snapdragon SoC（Tensor、Exynos）

GPU 只在 Adreno（Qualcomm Snapdragon）驗證過。其他 SoC 系列各自有 LiteRT-LM 上游 issue：

- **Pixel / Tensor G3+：** Tensor 沒暴露 OpenCL — 見
  [LiteRT-LM #1860](https://github.com/google-ai-edge/LiteRT-LM/issues/1860)。
- **Exynos（S26 / Xclipse）：** ANGLE-CL 下的 Clspv kernel bug — 見
  [LiteRT-LM #2114](https://github.com/google-ai-edge/LiteRT-LM/issues/2114)。

如果你的裝置 GPU init 回 `INTERNAL ERROR ... compiled_model_executor.cc:1928`，
就在 request 裡 fallback 到 CPU：

```bash
curl -s http://127.0.0.1:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{"prompt":"hi","backend":"cpu","stream":false}'
```

---

## 解除安裝

```bash
adb shell am force-stop dev.temuxllm.service
adb uninstall dev.temuxllm.service
adb shell rm -rf /data/local/tmp/litertlm /data/local/tmp/bin/litertlm
```
