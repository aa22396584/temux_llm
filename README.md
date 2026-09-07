# temux_llm — local LLM on Android, ollama-style

[English](README.md) · [繁體中文](README.zh-TW.md)

[![build](https://github.com/ImL1s/temux_llm/actions/workflows/build.yml/badge.svg)](https://github.com/ImL1s/temux_llm/actions/workflows/build.yml)
[![release](https://img.shields.io/github/v/release/ImL1s/temux_llm?include_prereleases)](https://github.com/ImL1s/temux_llm/releases)
[![license](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)
[![android](https://img.shields.io/badge/android-13%2B-3DDC84?logo=android&logoColor=white)](https://developer.android.com/about/versions/13)
[![runtime](https://img.shields.io/badge/runtime-LiteRT--LM%200.11.0--rc1-yellow)](https://github.com/google-ai-edge/LiteRT-LM)

A self-contained Android app + Termux client that runs Google's **Gemma 4** (or
Qwen3) on your phone's GPU and exposes it as
`http://127.0.0.1:11434/api/generate`. No network. No cloud. No data leaves
the device.

```
$ litertlm "Reply with just: hi."
hi.

[gpu  total=2676ms  tokens=62  decode=23.2 t/s]
```

Tested on **Galaxy S21+** (SD 888 / Adreno 660, Android 15), **Galaxy S24 Ultra**
(SD 8 Gen 3 / Adreno 750, Android 16), and **Galaxy S25** (SD 8 Elite / Adreno
830, Android 16). Should work on any arm64-v8a Android 13+ Snapdragon device.
See [Performance](#performance) for measured CPU + GPU decode rates
across both paths.

## Use cases / 用途

- **No network or cloud:** Run LLM prompts on airplane mode, in subways, in
  sensitive contexts where queries cannot leave the device.
- **Ollama-compatible endpoint:** Apps and scripts expecting
  `http://127.0.0.1:11434` (Ollama's standard port) work without modification.
- **Termux scripting:** Pipe LLM output into shell pipelines:
  `curl -s ... | jq .response` for structured queries on the phone.
- **Offline assistant:** Gemma-class model resident in memory, sub-second
  replies for short queries.
- **Interchangeable models:** Load any `.litertlm` model (Gemma, Qwen, future
  releases) and swap freely.
- **Private prototyping:** Build against a local endpoint during early
  dev — no API keys, no token costs, no prompts sent to third parties.

**Not for:** production traffic (single device, single tenant), long-context
work (8k context max), multi-user serving (localhost only by design).

---

## Open box use

One command from a fresh clone, with the phone plugged in via USB:

```bash
bash scripts/install.sh
```

The installer:

1. Verifies host has `adb`, JDK 17+, and a configured Android SDK.
2. Auto-picks your phone (or set `DEVICE_SERIAL=...` for a specific one).
3. Auto-picks a model based on your SoC, or set `MODEL=e2b|e4b|qwen3`:
    - **`e2b`** (default for older flagships): `gemma-4-E2B-it.litertlm`, 2.4 GB.
       Good on S21+ class hardware.
    - **`e4b`** (default for SD 8 Elite class — Fold7, S25): `gemma-4-E4B-it.litertlm`,
       3.4 GB. Smarter, slower, needs ≥10 GB RAM phone.
    - **`qwen3`**: `Qwen3-0.6B.litertlm`, 614 MB. Smallest fallback; rambles a lot.
4. Downloads the binary + accelerators + chosen model (sha256-verified).
5. Builds the debug APK with the bundled Gradle wrapper.
6. Pushes everything to the device, installs the APK, starts the service.
7. Smoke-tests `/api/generate` and prints next steps.

After it finishes, install Termux ([F-Droid build][termux-fdroid] — the Play Store
version is unmaintained) and add the wrapper to your PATH **once**:

```sh
echo 'export PATH="/data/local/tmp/bin:$PATH"' >> ~/.bashrc
```

Then in any Termux session:

```sh
litertlm "你好"
litertlm --backend cpu "Reply OK in 3 words."
litertlm --json "what is 2+2?"            # raw JSON for scripting
litertlm --help
```

[termux-fdroid]: https://f-droid.org/packages/com.termux/

---

## Use with CLI coding agents (v0.6.0 verified matrix)

> **v0.6.0 highlights:** vision input on `/v1/messages` /
> `/v1/chat/completions` / `/api/chat` / `/v1/responses` (Galaxy S25 read
> "STOP" from a 64×48 PNG — no fork required); tool-call success rate on
> Gemma 4 E4B raised from **77 % → 100 %** (n=30) via a stack-based
> JSON repair port from Ollama; `/api/probe/native_tools` endpoint hits
> 100 % using the SDK's native tool path 30 % faster, opt-in for v0.7
> via `native_tools=on` in `temuxllm.conf`; codex `web_search` is now
> filtered out of `/v1/responses` ingress so the model stops fabricating
> hosted-tool calls.



`temux_llm` impersonates Ollama closely enough that the popular coding-agent
CLIs talk to it without a proxy. The matrix below is what we **actually
re-ran end-to-end on a Galaxy S25 (12 GB / 16 k context, default backend
CPU)** for v0.5.1 — not what should work in theory.

| CLI | Verified | Endpoint | Notes |
|---|---|---|---|
| **Claude Code** (`claude --bare`) | ✅ | `/v1/messages` (Anthropic SSE) | works directly; `--bare` strips ~16 k system prompt |
| **OpenAI Codex** 0.125+ | ✅ | `/v1/responses` (OpenAI SSE) | needs custom provider; `wire_api="chat"` is dropped, `--oss` forces gpt-oss:20b |
| **llxprt-code** (Gemini CLI fork) | ✅ | `/v1/chat/completions` | `npm i -g @vybestack/llxprt-code`; native OpenAI provider |
| **OpenCode** 1.14+ | ✅ via bridge | `/v1/chat/completions` | needs LiteLLM bridge to inject `backend=cpu` (12 GB devices OOM in GPU mode) |
| **stock Gemini CLI** | ❌ | — | no env var redirects to local; cached OAuth bypasses any base URL override (g-g/gemini-cli #1605, #5945, #24166) |
| **OpenClaw** | not retested in v0.5.1 | `/api/chat` (Ollama native) | wire is correct; flagged for v0.6.0 retest |

### One-shot launcher

Once the service is running on the phone, forward port 11434 from your host:

```sh
adb forward tcp:11434 tcp:11434
```

Then use the bundled launcher (does what `ollama launch <cli>` does in
Ollama 0.15+ — probe service, pick active model, set the right env / config,
exec the CLI):

```sh
scripts/temuxllm launch claude          # Claude Code, --bare
scripts/temuxllm launch codex           # Codex CLI (custom temuxllm provider)
scripts/temuxllm launch opencode        # OpenCode (auto-spawns LiteLLM cpu bridge)
scripts/temuxllm launch gemini          # llxprt-code (Gemini CLI fork)
scripts/temuxllm launch openclaw        # Ollama-native, untested in v0.5.1

scripts/temuxllm launch claude --config-only   # print env block, do not exec
scripts/temuxllm launch codex --model gemma-4-E2B-it
```

### Manual setup per CLI

If you'd rather wire the env / config yourself, here are the **exact
configs the launcher uses**. Each was re-run on v0.5.1 against the real
service before being committed.

#### Claude Code

```sh
export ANTHROPIC_BASE_URL=http://127.0.0.1:11434
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_API_KEY=
export ANTHROPIC_MODEL=model
export ANTHROPIC_DEFAULT_OPUS_MODEL=model
export ANTHROPIC_DEFAULT_SONNET_MODEL=model
export ANTHROPIC_DEFAULT_HAIKU_MODEL=model
claude --bare           # add --print "your prompt" for one-shot
```

`--bare` is required on 12 GB / 16 k devices: the full Claude Code agent
system prompt is ~16 k tokens which leaves zero output room. Drop it on
≥16 GB devices with `TEMUXLLM_MAX_TOKENS=24576+` set.

#### Codex CLI 0.125+

Codex 0.125 broke the simple `--oss` shorthand for non-stock models:
`wire_api="chat"` was dropped, `--oss --local-provider ollama` overrides
`-c model=` and forces `gpt-oss:20b`, and `model_providers.ollama.*` is
reserved as a built-in name. Codex's default `reasoning_effort="xhigh"`
also adds ~5 k tokens of `<think>` trace per turn, blowing 12 GB / 16 k
device budgets. We use a custom `temuxllm` provider with a tiny
instructions file and `reasoning_effort="minimal"`:

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

**Important:** on 12 GB devices, drop `max_tokens` to **12288** (16384
OOMs under codex's prompt prefill + GPU init). Push the conf:

```sh
adb shell 'echo max_tokens=12288 > /data/local/tmp/litertlm/temuxllm.conf'
adb shell 'am force-stop dev.temuxllm.service'
# next request restarts the engine with the new ceiling
```

Override the launcher's defaults with `TEMUXLLM_CODEX_REASONING=high|low`
or `TEMUXLLM_CODEX_INSTRUCTIONS_FILE=/path/to/your.md`.

#### llxprt-code (Gemini CLI fork)

**Stock `gemini` CLI cannot be redirected to a local OpenAI-compatible
endpoint** — verified empirically against gemini 0.36 with a LiteLLM
bridge. The `GOOGLE_GEMINI_BASE_URL` and `GEMINI_API_KEY` env vars are
ignored when cached OAuth credentials exist; traffic goes to Google's
servers. The relevant feature requests
([#1605](https://github.com/google-gemini/gemini-cli/issues/1605),
[#5945](https://github.com/google-gemini/gemini-cli/discussions/5945),
[#24166](https://github.com/google-gemini/gemini-cli/discussions/24166))
remain open as of v0.5.1.

Use the [`@vybestack/llxprt-code`](https://github.com/vybestack/llxprt-code)
fork, which adds first-class OpenAI / Anthropic / Ollama providers:

```sh
npm install -g @vybestack/llxprt-code

# one-shot
llxprt --provider openai \
       --baseurl http://127.0.0.1:11434/v1 \
       --key ollama \
       --model model \
       --prompt "your prompt"

# interactive REPL — set provider once at the prompt
llxprt
# /provider openai
# /baseurl http://127.0.0.1:11434/v1/
# /key ollama
# /model model
```

#### OpenCode 1.14+ (via LiteLLM bridge)

OpenCode 1.14's bare-mode agent still ships a ~2.3 k-token system prompt.
On 12 GB devices, GPU mode + that prompt regularly trips
`lowmemorykiller`. We route OpenCode through a tiny LiteLLM proxy that
injects `extra_body: {backend: "cpu"}` into every call so the on-device
service runs the request on CPU instead. The launcher auto-spawns the
bridge; for a manual setup:

```sh
pip install 'litellm[proxy]'

# bridge config
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

# OpenCode config (writes to a temp file; cd to scratch dir to dodge
# AGENTS.md / CLAUDE.md walk-up that re-bloats the prompt)
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

On ≥16 GB devices set `TEMUXLLM_OPENCODE_BACKEND=gpu` to skip the bridge.

### What works vs what doesn't (v0.5.1)

**Works:**
- All four wire envelopes (Anthropic SSE, OpenAI SSE, OpenAI Responses,
  Ollama NDJSON) round-trip plain text and tool calls.
- Tool calling via prompt-injection + brace-balance JSON parser (no
  Conversation::Clone yet — single-turn only).
- Memory observability: `MemoryProbe` samples PSS / VmHWM / oom_score_adj
  during inference. After an LMK kill, `AutoFallback` downshifts
  `maxNumTokens` for the next start (device-fingerprinted).
- Auto CPU/GPU default via `default_backend=` in
  `/data/local/tmp/litertlm/temuxllm.conf` or `TEMUXLLM_DEFAULT_BACKEND=`.

**Doesn't work / known limitations:**
- **Multi-turn agent loops** on 12 GB / 16 k devices once the prompt
  prefill grows past ~10 k tokens. Memory pressure trips LMK before
  decode starts. Mitigation: use `e2b` model + `--bare` flags + the CPU
  default backend.
- **Image inputs.** `image_url` / `input_image` blocks return 400.
- **Stock Gemini CLI redirect.** No env-var path; use llxprt-code.

### Tool calling, MCP, skills, web search

`--bare` modes strip auto-discovered tools by design. The matrix of
which CLI keeps which capability — and how to get web search / MCP /
skills back when you need them — lives in
[`docs/cli-tool-matrix.md`](docs/cli-tool-matrix.md). Short version:

| CLI | What `--bare` keeps | How to add web search / MCP back |
|---|---|---|
| `claude --bare` | Bash, Read, Edit only | `--mcp-config <file>` + `--strict-mcp-config` |
| `codex exec` (minimal) | All 29 built-ins + MCP + skills | already on; add MCP servers in `~/.codex/config.toml` |
| `llxprt` | All built-ins + MCP + skills (no bare knob) | `EXA_API_KEY=...` for web search; `llxprt mcp add` for MCP |
| `opencode run --agent bare` | Nothing — every tool off | swap to `--agent build` and re-enable `webfetch` / `websearch` |

Single-shot tool calling works on all four wire envelopes
(`/api/chat`, `/v1/chat/completions`, `/v1/messages`, `/v1/responses`).
Multi-turn agent loops are unreliable on Gemma 4 today — fixed in
v0.6.0 (KV reuse via `Conversation::Clone()`).

For a plain reference of every endpoint we expose and which CLI each one is
for, see [`docs/ollama-compat.md`](docs/ollama-compat.md).

---

## Termux-native (no APK)

Don't want to sideload the APK? Run the LiteRT-LM binary directly inside
Termux — no USB cable, no Android service, no host machine required.

**Inside Termux on the phone:**

```sh
# download installer from your clone, or copy it manually
bash install-termux-native.sh          # default: gemma-4-E2B-it (2.4 GB)
MODEL=qwen3 bash install-termux-native.sh   # 614 MB fallback
MODEL=e4b   bash install-termux-native.sh   # 3.4 GB, high-end SoC only
```

The installer writes everything to `~/.litertlm/` and drops the wrapper at
`~/.local/bin/litertlm-native`. Add `~/.local/bin` to PATH once, then:

```sh
litertlm-native "你好"
litertlm-native --backend cpu "Reply OK in 3 words."
litertlm-native --json "what is 2+2?"
litertlm-native --help
```

**When to pick which path:**

| | APK path (`install.sh`) | Native path (`install-termux-native.sh`) |
|---|---|---|
| Setup | host machine + USB (one-time) | inside Termux only |
| Sideloading APK | required | none |
| Per-call init | sub-second (engine resident) | 3-8 s warm / 10-20 s first-ever |
| Steady-state decode | depends on backend (see matrix) | **CLI CPU rivals or beats APK GPU on Snapdragon** |
| GPU acceleration | works on Adreno (v0.1.2+) | blocked by Termux's vendor namespace |
| Best for | short interactive turns, sub-second replies | long generation, no-USB workflows, scripts |

**Counter-intuitive but verified:** the CLI's raw CPU decode rate beats the
APK's GPU end-to-end on every Snapdragon device we tested
(see [Performance](#performance)). Why: the CLI is a tight native binary
with multi-threaded CPU inference, while the APK pays JNI + service +
foreground-service-notification overhead. The APK still wins on per-call
latency because its engine is resident; the CLI pays init per call.

Both paths use LiteRT-LM 0.11.0-rc.1 (the APK depends on the Maven
artifact `com.google.ai.edge.litertlm:litertlm-android:0.11.0-rc1`,
the Termux-native path uses the matching `litert_lm_main` CLI binary
from the `v0.11.0-rc.1` GitHub release) and the same models.

---

## What it ships

```
scripts/install.sh                  one-shot installer (above)
scripts/install-termux-native.sh    Termux-only installer (no APK)
scripts/litertlm-native-wrapper.sh  source for the Termux-native CLI wrapper
scripts/fetch_artifacts.sh          sha256-verified host downloader
scripts/litertlm-termux-wrapper.sh  Termux client for the APK service
scripts/preflight.sh                host + device readiness probe
scripts/setup_litertlm_android.sh   manual push (used by install.sh)
scripts/run_{cpu,gpu}_smoke.sh      raw adb-shell smokes (skip APK)
scripts/run_litertlm_benchmark.sh   long-prompt benchmark
scripts/parse_litertlm_logs.sh      extract BenchmarkInfo to logs/summary.txt
scripts/sha256_manifest.txt         pinned hashes for binary + .so + models

android/                            Android Studio project (Kotlin)
  app/src/main/kotlin/dev/temuxllm/
    LlmService.kt                   foreground service holding the engine
    LlmEngine.kt                    in-process Engine + Conversation wrapper
    HttpServer.kt                   NanoHTTPD on 127.0.0.1:11434
    LauncherActivity.kt             tappable icon → starts service, finishes
    BootReceiver.kt                 auto-start on BOOT_COMPLETED

docs/specs/                         phase-2 architecture spec
docs/plans/                         implementation plans (phase-1 / phase-2a)
docs/findings-*.md                  test results with real numbers per device
docs/screenshots/                   phone-screen captures of Termux running it
```

---

## API

```
GET  /healthz       -> "ok"
GET  /api/version   -> {service, phase, runtime, default_backend,
                        model_path, source_model_path, engine_loaded}
GET  /api/tags      -> {models: [{name, path, size_bytes}, ...]}
POST /api/generate  -> NDJSON stream of {response, done} (or one JSON if stream=false)
```

`/api/generate` request body:

```json
{
  "prompt":  "Hi",
  "backend": "cpu" | "gpu",   // default: see Tunables below
  "stream":  true             // default: true (NDJSON one-line-per-token)
}
```

### Tunables (v0.5.1+)

`/data/local/tmp/litertlm/temuxllm.conf` is read at every engine init:

```
max_tokens=16384         # KV-cache ceiling. Drop to 8192 on 8 GB devices
default_backend=cpu      # cpu|gpu. Default for /v1 calls when the body
                         # omits a `backend` field. CLIs like Codex and
                         # OpenCode never send one — set this to `cpu`
                         # on 12 GB / 16 k devices to avoid LMK kills.
```

`TEMUXLLM_DEFAULT_BACKEND=cpu` env var is read as a fallback when the
file is absent.

Streaming response: one JSON object per line (`application/x-ndjson`):

```
{"response":"Hi","done":false}
{"response":"!","done":false}
{"response":"","done":true,"backend":"gpu","total_duration_ms":820,"output_tokens":2,"output_chars":3}
```

Non-streaming (`stream=false`) returns one document with `model`, `backend`,
`response`, `done`, `total_duration_ms`, `output_tokens`.

The service binds only to `127.0.0.1`. Verified after every restart with
`ss -tnlp | grep 11434` showing `[::ffff:127.0.0.1]:11434` (never `0.0.0.0`).

---

## Constraints (kept from the original brief)

- No root.
- arm64-v8a only.
- Localhost-only HTTP. Never exposed beyond loopback.
- Runtime is LiteRT-LM loading `.litertlm` (not the older `.tflite` / `.task`).

---

## Performance

`gemma-4-E2B-it` (2.4 GB), 50-word output prompt, warm engine.

| Device | SoC | APK CPU<br>(end-to-end) | APK GPU<br>(end-to-end) | CLI CPU<br>(decode-only) |
|---|---|---|---|---|
| Galaxy S21+ | SD 888 / Adreno 660 | 8.0 t/s | 10.5 t/s | **12.1 t/s** |
| Galaxy S24 Ultra | SD 8 Gen 3 / Adreno 750 | 10.3 t/s | 22.0 t/s | 21.5 t/s |
| Galaxy S25 | SD 8 Elite / Adreno 830 | 12.4 t/s | 23.2 t/s | **35.4 t/s** |

The two metrics measure different things and are not directly comparable as
a percentage. Reported separately so each is honest for its use case:

- **APK end-to-end** = `output_tokens / (total_duration_ms / 1000)`, the full
  cycle of one warm `/api/generate` request — includes prefill, decode, and
  the loopback HTTP/JNI overhead the user actually feels. Excludes the
  resident engine's one-time init.
- **CLI decode-only** = `Decode Speed` reported by the binary's
  `BenchmarkInfo` block — sustained tokens/sec once the engine is producing
  output, measured in isolation from prefill and per-call init.

For a CLI end-to-end view: the warm wall-time-per-call is **3-8 s** (engine
init dominates short prompts; long generation amortizes it).
A 60-token output on S25 takes ~3 s wall ≈ 20 t/s end-to-end, vs the APK GPU's
~3 s for 62 tokens ≈ 23 t/s end-to-end — roughly tied for short outputs.
The CLI's decode-only advantage (35 t/s on S25) shows up when generation is
long enough for init to amortize. See "Why CLI CPU beats APK GPU" below for
the architectural reason.

**Per-call latency** is a different axis: APK keeps `Engine` resident, so a
warm `/api/generate` returns in 200-1000 ms for short answers. CLI rebuilds
the engine each call, so per-call wall is **3-8 s warm / 10-20 s first ever**.
Pick the path based on whether you optimize for per-call latency
(APK) or sustained decode throughput (CLI).

`gemma-4-E4B-it` (3.4 GB) — APK only:

| Device | E4B CPU | E4B GPU |
|---|---|---|
| Galaxy S21+ | 4.1 t/s | (not measured) |
| Galaxy S24 Ultra | 5.7 t/s | (not measured) |
| Galaxy S25 | 7.8 t/s | 15.0 t/s |
| Galaxy Note 9 | unsupported (minSdk=33 / Android 13+) | |

**First-call init cost:**

- **APK GPU:** OpenCL kernel-compile takes 8-22 s the first time the engine
  starts. The SDK writes `model.litertlm_*_mldrift_program_cache.bin` into
  the app's filesDir; subsequent service starts skip the compile and reuse
  it. If you re-push a model, the cache is invalidated and you pay this
  cost again.
- **APK CPU:** XNNPack weight cache build takes 3-7 s on first call; warm
  init is 0.5-0.8 s.
- **CLI CPU:** First-ever call builds the xnnpack cache file next to the
  model (~10-20 s, one-time). Subsequent calls pay only engine init
  (~2-3 s). The `install-termux-native.sh` script auto-runs a warmup at
  the end of install so the user's first interactive call is already warm.

**Why CLI CPU beats APK GPU:** The CLI is a tight native binary doing
multi-threaded CPU inference — no JNI, no service framework overhead, no
foreground-service notification, no per-request socket round-trip. The
APK GPU pays for the SDK's JVM↔native bridge plus the localhost HTTP
hop on every call. For long generation (where decode dominates init),
CLI's raw throughput wins. For short interactive replies (where init
dominates), APK's resident engine wins.

---

## Known issues

### Note 9 / Android < 13

`minSdk=33` (Android 13). Older devices like Galaxy Note 9 (Android 10) reject
the APK install with `INSTALL_FAILED_OLDER_SDK`. There is no plan to lower the
floor — the LiteRT-LM SDK targets API 33+.

### Non-Snapdragon SoCs (Tensor, Exynos)

GPU is verified on Adreno (Qualcomm Snapdragon) only. Other SoC families have
their own LiteRT-LM upstream issues:

- **Pixel / Tensor G3+:** OpenCL not exposed by Tensor — see
  [LiteRT-LM #1860](https://github.com/google-ai-edge/LiteRT-LM/issues/1860).
- **Exynos (S26 / Xclipse):** Clspv kernel bug under ANGLE-CL — see
  [LiteRT-LM #2114](https://github.com/google-ai-edge/LiteRT-LM/issues/2114).

If GPU init returns `INTERNAL ERROR ... compiled_model_executor.cc:1928` on
your device, fall back to CPU per request:

```bash
curl -s http://127.0.0.1:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{"prompt":"hi","backend":"cpu","stream":false}'
```

---

## Uninstall

```bash
adb shell am force-stop dev.temuxllm.service
adb uninstall dev.temuxllm.service
adb shell rm -rf /data/local/tmp/litertlm /data/local/tmp/bin/litertlm
```

---

## Support

If this project saved you some time, you can [buy me a coffee](https://buymeacoffee.com/iml1s).
