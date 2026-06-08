# Hướng dẫn cài đặt và cấu hình Codex-Shim

## Codex-Shim là gì?

Codex-Shim là local proxy (Python/aiohttp) cho phép dùng **BYOK (Bring Your Own Key)** models với Codex Desktop/CLI. Codex mặc định chỉ cho phép models được OpenAI whitelist qua server-side config. Shim bypass limitation này bằng cách:

1. Bind local server trên `127.0.0.1:8765`
2. Translate Codex Responses API → upstream APIs (OpenAI chat completions, Anthropic Messages, generic chat)
3. Route requests đến models bạn config trong `~/.codex-shim/models.json`

## Yêu cầu

- Python 3.11+
- Codex CLI hoặc Codex Desktop (VS Code extension) đã cài
- Git
- API keys của các models bạn muốn dùng (Anthropic, DeepSeek, OpenRouter, etc.)
- (Optional) Tài khoản ChatGPT để dùng ChatGPT passthrough

---

## Bước 1: Clone và cài đặt

### Windows (PowerShell)

```powershell
git clone https://github.com/0xSero/codex-shim $HOME\codex-shim
cd $HOME\codex-shim
py -3.11 -m pip install --user -e .
```

### macOS / Linux / WSL / Git Bash

```bash
git clone https://github.com/0xSero/codex-shim ~/codex-shim
cd ~/codex-shim
python3 -m pip install --user -e .
```

### Kiểm tra cài đặt

```powershell
codex-shim --help
```

Nếu `codex-shim` không tìm thấy, dùng module form:

```powershell
py -3.11 -m codex_shim.cli --help
```

Trên Windows, Python user Scripts thường ở:
```
C:\Users\<username>\AppData\Roaming\Python\Python312\Scripts
```
Thêm vào PATH nếu cần.

---

## Bước 2: Cấu hình models

### Default config file

Shim mặc định đọc `~/.codex-shim/models.json`. Nếu file không tồn tại, shim vẫn generate catalog — và thêm `gpt-5.5` ChatGPT passthrough entry **chỉ khi** `~/.codex/auth.json` chứa valid `tokens.access_token`.

### Custom config file

Bạn có thể dùng file config tùy ý:

```powershell
codex-shim --settings /path/to/my-models.json generate
codex-shim --settings /path/to/my-models.json start
```

### Schema chuẩn

Tạo file `~/.codex-shim/models.json` (hoặc file custom):

```json
{
  "models": [
    {
      "model": "gpt-5.5",
      "provider": "openai",
      "base_url": "https://api.openai.com/v1",
      "api_key": "sk-...",
      "display_name": "OpenAI GPT-5.5",
      "max_context_limit": 400000
    },
    {
      "model": "claude-opus-4-7-20251109",
      "provider": "anthropic",
      "base_url": "https://api.anthropic.com/v1",
      "api_key": "sk-ant-...",
      "display_name": "Claude Opus 4.7"
    },
    {
      "model": "deepseek-v4-pro",
      "provider": "anthropic",
      "base_url": "https://api.deepseek.com/anthropic",
      "api_key": "...",
      "display_name": "DeepSeek V4 Pro",
      "no_image_support": true
    }
  ]
}
```

Loader cũng chấp nhận camelCase aliases (`baseUrl`, `apiKey`, `displayName`, `maxContextLimit`, `maxOutputTokens`, `noImageSupport`, `extraHeaders`) và legacy top-level `customModels` array.

**Shim KHÔNG copy API keys vào generated catalog.** Keys stay trong settings file và được read fresh mỗi request.

### Các provider được hỗ trợ

| Provider | Upstream API |
|---|---|
| `openai` | OpenAI `/v1/chat/completions` |
| `generic-chat-completion-api` | OpenAI-shaped chat completions (DeepSeek, OpenRouter, local proxy, etc.) |
| `anthropic` | Anthropic `/v1/messages` |

### Các field hữu ích

| Field | Mô tả |
|---|---|
| `display_name` | Tên hiển thị trong Codex picker |
| `max_context_limit` | Context window của model |
| `max_output_tokens` | Max output tokens khi translate sang Anthropic |
| `no_image_support` | Set `true` nếu model không hỗ trợ ảnh |
| `extra_headers` | Headers tùy chỉnh gửi kèm request |

### Ollama / Local models

```json
{
  "model": "llama3.2",
  "display_name": "Ollama Llama 3.2",
  "provider": "generic-chat-completion-api",
  "base_url": "http://127.0.0.1:11434/v1",
  "api_key": "ollama"
}
```

---

## Bước 3: Login ChatGPT (optional, cho passthrough)

Nếu bạn có tài khoản ChatGPT và muốn dùng ChatGPT Codex backend qua slug `gpt-5.5`:

```powershell
codex login
```

Trình duyệt sẽ mở trang OAuth. Đăng nhập và cấp quyền. File `~/.codex/auth.json` sẽ được tạo với `tokens.access_token`.

**Không cần** nếu bạn chỉ dùng BYOK models.

---

## Bước 4: Generate catalog và start shim

```powershell
# Generate catalog từ models.json (không start daemon, không modify config.toml)
codex-shim generate

# Start shim daemon (background trên 127.0.0.1:8765)
codex-shim start

# Enable (generate + start + write managed blocks vào ~/.codex/config.toml)
codex-shim enable
```

### Kiểm tra

```powershell
codex-shim status
# Shim is running on http://127.0.0.1:8765 with pid XXXX (N models).

codex-shim list
# Hiển thị danh sách models và routes
```

### Các lệnh quản lý shim

Theo README chính thức:

```powershell
codex-shim generate          # Regenerate catalog/config, không start daemon
codex-shim start             # Regenerate catalog và start daemon
codex-shim enable            # Start daemon + write managed blocks vào config.toml
codex-shim status            # Health check + model count
codex-shim stop              # Stop daemon
codex-shim disable           # Remove managed blocks + stop daemon
codex-shim restart           # Stop, regenerate, start daemon
codex-shim list              # List generated slugs và routes
codex-shim model list        # List slugs khả dụng trong picker
codex-shim model use <slug>  # Set default model trong managed config
codex-shim codex -- <args>   # Exec codex CLI với inline shim overrides
codex-shim app [path]        # Launch Codex Desktop qua managed shim config
```

Shortcuts:
```powershell
codex-app [path]             # Shortcut cho codex-shim app
codex-model [list|<slug>]    # Shortcut cho codex-shim model
```

Global flags:
- `--settings <path>`: Dùng cho catalog/model/start/app/codex flows
- `--port <port>`: Dùng cho daemon/provider flows

**Lưu ý quan trọng về config behavior:**
- `codex-shim generate`, `start`, `stop`, `restart`, `list`, `status`, và `codex-shim codex -- ...` **KHÔNG** modify `~/.codex/config.toml`
- `codex-shim enable`, `codex-shim app`, và `codex-shim model use <slug>` **CÓ** write managed blocks vào `~/.codex/config.toml`
- `codex-shim disable` removes managed blocks, restores displaced keys, và stops daemon

---

## Bước 5: Cấu hình Codex

### Option A: Dùng Codex CLI (recommended)

```powershell
# Chạy codex với shim
codex-shim codex -- "task của bạn"

# Hoặc dùng inline overrides
codex --model-provider codex_shim --model claude-sonnet-4-5 "task của bạn"
```

### Option B: Dùng Codex Desktop / VS Code Extension

`codex-shim enable` sẽ tự động write managed blocks vào `~/.codex/config.toml`:

```toml
# >>> codex-shim managed >>>
model = "gpt-5.5"
model_provider = "codex_shim"
model_catalog_json = "C:\\Users\\<username>\\codex-shim\\.codex-shim\\custom_model_catalog.json"
# <<< codex-shim managed >>>

# >>> codex-shim managed >>>
[model_providers.codex_shim]
name = "GPT-5.5"
base_url = "http://127.0.0.1:8765/v1"
wire_api = "responses"
experimental_bearer_token = "dummy"
request_max_retries = 3
stream_max_retries = 3
stream_idle_timeout_ms = 600000
# <<< codex-shim managed <<<
```

Restart VS Code để extension đọc config mới.

### Đổi model đang dùng

```powershell
codex-model list              # Xem models khả dụng
codex-model claude-sonnet-4-5 # Đổi sang Claude
codex-model deepseek-v4-pro   # Đổi sang DeepSeek
codex-model gpt-5.5           # Đổi sang ChatGPT passthrough
```

Hoặc dùng web UI picker:
```
http://127.0.0.1:8765/picker
```

---

## Bước 6: Verify cấu hình

```powershell
codex doctor
```

Output mong đợi:

```
✓ config       loaded
    model                    claude-sonnet-4-5 · codex_shim
    config.toml              C:\Users\<username>\.codex\config.toml
    config.toml parse        ok

✓ auth         OpenAI auth is not required for the active model provider
    auth storage mode        File
    auth file                C:\Users\<username>\.codex\auth.json
    model provider requires OpenAI auth false

✓ reachability active provider endpoints are reachable over HTTP
    codex_shim API base URL  http://127.0.0.1:8765/v1 reachable (HTTP 404)
    codex_shim API route probe http://127.0.0.1:8765/v1/<redacted> route exists (HTTP 200)
```

---

## Troubleshooting

### Lỗi encoding trên Windows

Nếu gặp `UnicodeDecodeError: 'charmap' codec can't decode byte`, file `codex_shim/settings.py` cần patch để thêm `encoding="utf-8"` vào tất cả `read_text()` calls. Đã được fix trong repo này.

### Lỗi `ImportError: cannot import name 'available_model_slugs'`

File `codex_shim/settings.py` bị modify làm mất function. Restore:

```powershell
cd $HOME\codex-shim
git restore codex_shim/settings.py
```

### Lỗi `401 Unauthorized: Incorrect API key provided: sk_9router`

File `~/.codex/auth.json` chứa API key giả. Xóa và login lại:

```powershell
Remove-Item "$env:USERPROFILE\.codex\auth.json" -Force
codex login
```

### Lỗi `FileNotFoundError: [WinError 2] The system cannot find the file specified`

Codex CLI chưa cài hoặc không có trong PATH. Dùng bundled binary từ VS Code extension:

```powershell
& "c:\Users\ADMIN\.vscode\extensions\openai.chatgpt-<version>\bin\windows-x86_64\codex.exe" doctor
```

### Windows proxy chặn loopback traffic

Nếu có system proxy (Clash, V2Ray), set bypass:

**Temporary (current session only):**
```powershell
$env:NO_PROXY = "127.0.0.1,localhost,::1"
$env:no_proxy = "127.0.0.1,localhost,::1"
```

**Persistent (theo README):**
```powershell
setx NO_PROXY "127.0.0.1,localhost,::1"
setx no_proxy "127.0.0.1,localhost,::1"
```

`codex-shim app ...` và `codex-shim codex -- ...` tự động set entries cho child process. Set globally nếu bạn chạy `codex.exe` trực tiếp.

### Codex Desktop (MSIX) rewrite config.toml

Windows Store/MSIX Codex Desktop builds stricter hơn CLI. Chúng có thể:
- Treat custom local/BYOK slugs as unavailable
- Rewrite `model = "<custom-slug>"` back to `gpt-5.5`
- Add `[tui.model_availability_nux]` entries on launch

Đây là Desktop allowlist behavior, không phải shim routing behavior. `codex exec`, TUI, và shim endpoint vẫn dùng configured model slug.

**Giải pháp:**
1. Đóng Codex Desktop
2. Chạy `codex-shim enable`
3. Mở lại Codex Desktop ngay

Hoặc dùng Codex CLI thay vì Desktop: `codex-shim codex -- "task"`

**Idempotent behavior:** `codex-shim enable`, `codex-shim app`, và `codex-shim model use ...` là idempotent. Shim-managed top-level keys và `[model_providers.codex_shim]` block được remove trước khi write block mới, nên duplicate keys không accumulate.

**Note:** Codex có thể make small background calls đến OpenAI model slugs như `gpt-5.4-mini` cho product behavior. Đây không phải Ollama routing failures. Dùng shim request log để confirm actual selected model.

### Model không hiện trong picker

Windows MSIX Desktop có server-side allowlist ẩn custom models. Giải pháp:

1. Dùng Codex CLI: `codex-shim codex -- "task"`
2. Dùng inline overrides: `codex --model-provider codex_shim --model <slug> "task"`
3. Dùng web picker: `http://127.0.0.1:8765/picker`

### Tool calls bị flatten thành text

BYOK providers có quality tool-call khác nhau. Test với ChatGPT passthrough trước:

```powershell
codex-model gpt-5.5
codex "task cần tool calls"
```

Nếu passthrough OK nhưng BYOK không, upstream model không hỗ trợ native tool calls hoặc emit malformed streamed arguments. Check `.codex-shim/shim.log` cho requested model và tool count.

### Reset generated shim state

```powershell
codex-shim stop
# Xóa .codex-shim/ manually nếu muốn fresh state hoàn toàn
Remove-Item -Recurse -Force .codex-shim
codex-shim generate
codex-shim start
```

---

## Kiến trúc routing

```
Codex Desktop/CLI
    │
    ▼ /v1/responses
codex-shim (127.0.0.1:8765)
    │
    ├── slug "gpt-5.5"
    │       └── chatgpt.com/backend-api/codex/responses
    │           (Authorization: Bearer <auth.json access_token>)
    │
    ├── provider "openai" / "generic-chat-completion-api"
    │       └── baseUrl/chat/completions
    │           (Authorization: Bearer apiKey)
    │
    └── provider "anthropic"
            └── baseUrl/messages
                (x-api-key: apiKey, anthropic-version: ...)
```

---

## File layout

Theo README chính thức:

```text
codex_shim/             Python source (server + cli + translation)
bin/codex-shim          Main entrypoint
bin/codex-app           Shortcut wrapping codex-shim app
bin/codex-model         Shortcut wrapping codex-shim model
.codex-shim/            Generated catalog, config, logs, pid (gitignored)
tests/                  Pytest suite
```

### Files quan trọng

| File | Mô tả |
|---|---|
| `~/.codex-shim/models.json` | Config BYOK models (hoặc dùng `--settings` để chỉ định file khác) |
| `~/.codex/auth.json` | ChatGPT auth tokens (cho passthrough) |
| `~/.codex/config.toml` | Codex config (có managed blocks của shim) |
| `.codex-shim/custom_model_catalog.json` | Generated catalog cho Codex picker |
| `.codex-shim/config.toml` | Generated config template |
| `.codex-shim/shim.pid` | PID của shim daemon |
| `.codex-shim/shim.log` | Log của shim (stdout/stderr + request summaries) |

### Generated runtime files

Sau khi `codex-shim generate` hoặc `start`, các files sau được tạo trong `.codex-shim/` (repo-local, gitignored):

```text
.codex-shim/custom_model_catalog.json   # Model picker catalog cho Codex
.codex-shim/config.toml                  # Opt-in Codex provider config
.codex-shim/shim.pid                     # Daemon PID
.codex-shim/shim.log                     # Stdout/stderr + request summaries
```

---

## Auto Router (optional)

Thêm smart routing vào `models.json`:

```json
{
  "models": [...],
  "router": {
    "enabled": true,
    "slug": "codex-auto",
    "classifier": "minimax-m3",
    "threshold": 0.7,
    "default": "minimax-m3",
    "cache": true,
    "candidates": [
      {
        "slug": "minimax-m3",
        "cost": 0.3,
        "supports_images": false,
        "card": "Cheap, fast. Single-file edits, codegen."
      },
      {
        "slug": "claude-sonnet-4-5",
        "cost": 3.0,
        "supports_images": true,
        "card": "Frontier. Complex refactors, debugging, images."
      }
    ]
  }
}
```

Auto Router sẽ tự động route tasks đơn giản sang model rẻ, tasks phức tạp sang model mạnh.

---

## Prompt Catching Proxy (advanced)

Put local proxy trước shim để inspect/modify requests:

```python
from aiohttp import ClientSession, web

UPSTREAM = "http://127.0.0.1:8765"

async def responses(request):
    body = await request.json()
    body = catch_prompt(body)  # modify payload
    async with ClientSession() as s:
        async with s.post(f"{UPSTREAM}/v1/responses", json=body, headers=request.headers) as r:
            return web.Response(body=await r.read(), status=r.status, headers=r.headers)

def catch_prompt(body):
    # Inject system prompt, strip boilerplate, etc.
    return body

app = web.Application()
app.router.add_post("/v1/responses", responses)
web.run_app(app, host="127.0.0.1", port=8766)
```

Rồi point Codex vào `http://127.0.0.1:8766/v1` thay vì `8765`.

---

## Security notes

Theo README chính thức:

- **Loopback binding**: Shim bind `127.0.0.1` by default. Là local loopback adapter, không phải Internet-facing proxy.
- **Host header validation**: Shim validate `Host` header trên mọi request và reject nếu không phải loopback name (`127.0.0.1`, `localhost`, `::1`), configured bind host, hoặc entry trong `CODEX_SHIM_ALLOWED_HOSTS`. Blocks DNS-rebinding attacks.
- **API keys**: Stay trong settings file, generated catalog không chứa keys.
- **Request logs**: Summary-level by default, tránh full prompt/API-key dumps.
- **ChatGPT passthrough**: Read `~/.codex/auth.json` at request time, forward access token chỉ đến ChatGPT's Codex endpoint.
- **Prompt-catching proxy**: Nếu put proxy trước shim, proxy đó controls what nó logs. Redact hoặc hash large/private prompt bodies.

### CODEX_SHIM_ALLOWED_HOSTS

Nếu bạn deliberately bind shim vào non-loopback host, thêm host(s) vào env var:

```powershell
$env:CODEX_SHIM_ALLOWED_HOSTS = "192.168.1.100,myserver.local"
```

Comma-separated list.

---

## Limitations

Theo README chính thức:

- **Codex internals change**: Model-picker bundles thay đổi theo version. ASAR patch là version-sensitive.
- **ChatGPT passthrough endpoint**: Có thể move hoặc change shape trong future Codex release.
- **BYOK tool-call quality**: Providers vary wildly. Shim translates shapes nhưng không thể make upstream model emit valid tool-call JSON.
- **Hosted Responses-only tools**: Highest fidelity trên ChatGPT passthrough path. BYOK routes get normal function-tool translation.
- **bin/ shortcuts**: `bin/codex-app` và `bin/codex-model` là POSIX shell scripts. Trong native Windows shells, dùng `codex-shim app` và `codex-shim model` thay thế.

---

## Development checks

Theo README, nếu bạn modify code:

```bash
python3 -m pytest tests/
python3 -m compileall codex_shim/ -q
```

Tests cover settings/catalog generation, request translation, server routing, và CLI settings-file UX. Add regression tests khi changing translation behavior.

---

## Tham khảo

- Repo chính thức: https://github.com/0xSero/codex-shim
- Auto Router docs: `docs/AUTO_ROUTER.md`
- Subscription integration: `docs/subscription-integration.md`
- Cursor/Composer passthrough: `docs/subscription-integration.md`


---

## Cấu hình OpenCode GO & ZEN

[OpenCode](https://opencode.ai) cung cấp hai nhóm models qua API gateway: **ZEN** và **GO**. Cả hai đều dùng OpenAI-compatible endpoint nhưng khác base URL và API key.

### So sánh ZEN vs GO

| Tiêu chí | OpenCode ZEN | OpenCode GO |
|---|---|---|
| Base URL | `https://opencode.ai/zen/v1` | `https://opencode.ai/zen/go/v1` |
| API key (ví dụ) | `sk-TBjnz...` | `sk-cBNmter...` |
| Context limit | 131K-200K | 131K |
| Provider chính | `generic-chat-completion-api` + Anthropic | `anthropic` + generic |
| Models nổi bật | Claude Sonnet 4.5 (OpenCode Zen), Claude Opus 4.5 (OpenCode Zen), Claude Haiku 4.5 (OpenCode Zen), Kimi K2.5 (OpenCode Zen), Kimi K2.6 (OpenCode Zen), MiniMax M2.7 (OpenCode Zen) | Qwen 3.7 Plus (OpenCode GO), Qwen 3.7 Max (OpenCode GO), GLM 5 (OpenCode Go), MiMo V2.5 (OpenCode Go), MiMo V2.5 Pro (OpenCode Go), DeepSeek V4 Pro (OpenCode Go) |

### Models JSON cho OpenCode ZEN

```json
{
  "models": [
    { "model": "claude-sonnet-4-5", "provider": "anthropic", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "Claude Sonnet 4.5 (OpenCode Zen)", "max_context_limit": 200000, "max_output_tokens": 8192 },
    { "model": "claude-opus-4-5", "provider": "anthropic", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "Claude Opus 4.5 (OpenCode Zen)", "max_context_limit": 200000, "max_output_tokens": 8192 },
    { "model": "claude-haiku-4-5", "provider": "anthropic", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "Claude Haiku 4.5 (OpenCode Zen)", "max_context_limit": 200000, "max_output_tokens": 8192 },
    { "model": "kimi-k2.5", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "Kimi K2.5 (OpenCode Zen)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "kimi-k2.6", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "Kimi K2.6 (OpenCode Zen)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "minimax-m2.7", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "MiniMax M2.7 (OpenCode Zen)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "glm-5.1", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "GLM 5.1 (OpenCode Zen)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "grok-build-0.1", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "Grok Build 0.1 (OpenCode Zen)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "deepseek-v4-flash-free", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "DeepSeek V4 Flash Free (OpenCode Zen)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "mimo-v2.5-free", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "MiMo V2.5 Free (OpenCode)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "nemotron-3-ultra-free", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "Nemotron 3 Ultra Free (OpenCode)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "big-pickle", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/v1", "api_key": "sk-...", "display_name": "Big Pickle Free (OpenCode)", "max_context_limit": 131072, "max_output_tokens": 8192 }
  ]
}
```

### Models JSON cho OpenCode GO

```json
{
  "models": [
    { "model": "qwen3.7-plus", "provider": "anthropic", "base_url": "https://opencode.ai/zen/go/v1", "api_key": "sk-...", "display_name": "Qwen 3.7 Plus (OpenCode GO)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "qwen3.7-max", "provider": "anthropic", "base_url": "https://opencode.ai/zen/go/v1", "api_key": "sk-...", "display_name": "Qwen 3.7 Max (OpenCode GO)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "glm-5", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/go/v1", "api_key": "sk-...", "display_name": "GLM 5 (OpenCode Go)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "mimo-v2.5", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/go/v1", "api_key": "sk-...", "display_name": "MiMo V2.5 (OpenCode Go)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "mimo-v2.5-pro", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/go/v1", "api_key": "sk-...", "display_name": "MiMo V2.5 Pro (OpenCode Go)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "deepseek-v4-pro", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/go/v1", "api_key": "sk-...", "display_name": "DeepSeek V4 Pro (OpenCode Go)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "deepseek-v4-flash-free", "provider": "generic-chat-completion-api", "base_url": "https://opencode.ai/zen/go/v1", "api_key": "sk-...", "display_name": "DeepSeek V4 Flash Free (OpenCode Go)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "minimax-m3", "provider": "anthropic", "base_url": "https://opencode.ai/zen/go/v1", "api_key": "sk-...", "display_name": "MiniMax M3 (OpenCode Go)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "minimax-m2.5", "provider": "anthropic", "base_url": "https://opencode.ai/zen/go/v1", "api_key": "sk-...", "display_name": "MiniMax M2.5 (OpenCode Go)", "max_context_limit": 131072, "max_output_tokens": 8192 },
    { "model": "qwen3.6-plus", "provider": "anthropic", "base_url": "https://opencode.ai/zen/go/v1", "api_key": "sk-...", "display_name": "Qwen 3.6 Plus (OpenCode Go)", "max_context_limit": 131072, "max_output_tokens": 8192 }
  ]
}
```

### Lưu ý khi dùng OpenCode GO / ZEN với codex-shim

1. **Chọn provider đúng**: Models dùng giao thức Anthropic (`"provider": "anthropic"`) cần shim translate Responses API––> Messages API. Models dùng `"provider": "generic-chat-completion-api"` thì shim translate Responses API––> Chat Completions API.

2. **Hỗ trợ ảnh**: Claude models (Sonnet/Opus/Haiku) hỗ trợ ảnh. Các models còn lại nên set `"no_image_support": true` nếu gặp lỗi.

3. **API key riêng**: OpenCode ZEN và GO dùng hai API key **khác nhau**:
   - ZEN: key bắt đầu với `sk-TBjnz...`
   - GO: key bắt đầu với `sk-cBNmter...`

   Không dùng nhầm key cho sai base URL.

4. **Context limit**: Claude models trên ZEN có context 200K tokens. Các models còn lại (cả ZEN và GO) đều 131K.

5. **Copy models.json**: Copy file `~/.codex-shim/models.json` từ thông tin trên, thay `api_key` bằng key thật của bạn. Chạy `codex-shim generate && codex-shim start` để apply.

6. **Đổi model nhanh**:
   ```powershell
   codex-model qwen3.7-plus       # Qwen 3.7 Plus (OpenCode GO)
   codex-model claude-sonnet-4-5  # Claude Sonnet 4.5 (OpenCode Zen)
   codex-model deepseek-v4-pro    # DeepSeek V4 Pro (GO)
   codex-model kimi-k2.5          # Kimi K2.5 (ZEN)
   ```