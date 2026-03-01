# Closed-Network Deployment Guide: OpenCode Release Binary

> Deploy the pre-compiled OpenCode binary in a fully air-gapped network
> with a local vLLM backend serving GLM-4-9B-Chat.

---

## Prerequisites

| Component | Requirement |
|---|---|
| **Python** | 3.10 – 3.12 (for vLLM only) |
| **GPU** | NVIDIA GPU with ≥ 24 GB VRAM (A100/A10G/L40S/4090 recommended) |
| **CUDA** | 12.1+ with compatible NVIDIA drivers |
| **OS** | Linux x86_64 (Ubuntu 22.04+) / macOS (arm64/x64) / Windows x64 |
| **Disk** | ≥ 30 GB for model weights + 200 MB for OpenCode binary |

> **Note:** The release binary bundles its own Node.js runtime — you do **not** need to install Node.js separately.

---

## Phase 1: Download on the Open Network

### Step 1: Download the OpenCode Binary

Download the appropriate binary from the [OpenCode releases page](https://github.com/anomalyco/opencode/releases/latest):

| Platform | File | Size |
|---|---|---|
| Linux x64 | `opencode-linux-x64.tar.gz` | ~59 MB |
| Linux x64 (musl/Alpine) | `opencode-linux-x64-musl.tar.gz` | ~57 MB |
| Linux ARM64 | `opencode-linux-arm64.tar.gz` | ~59 MB |
| macOS ARM64 (Apple Silicon) | `opencode-darwin-arm64.zip` | ~42 MB |
| macOS x64 (Intel) | `opencode-darwin-x64.zip` | ~44 MB |
| Windows x64 | `opencode-windows-x64.zip` | ~60 MB |

```bash
# Example: Linux x64
curl -LO https://github.com/anomalyco/opencode/releases/latest/download/opencode-linux-x64.tar.gz
```

### Step 2: Download GLM-4-9B-Chat Model

```bash
pip install huggingface-hub
huggingface-cli download THUDM/glm-4-9b-chat --local-dir ./glm-4-9b-chat
```

### Step 3: Download vLLM Wheels

```bash
pip download vllm -d ./vllm-wheels/
```

### Step 4: Create the Chat Template

Save the following as `hermes_glm4_template.jinja`:

```jinja
{%- set system_message = "" -%}
{%- if messages[0]["role"] == "system" -%}
    {%- set system_message = messages[0]["content"] -%}
    {%- set loop_messages = messages[1:] -%}
{%- else -%}
    {%- set loop_messages = messages -%}
{%- endif -%}
{%- if tools -%}
    {%- set tool_str = "You are a function calling AI model. You are provided with function signatures within <tools></tools> XML tags. When the user asks you to perform an action, you MUST call the appropriate function. Do NOT explain what you are doing - just call the function directly.\n\nCRITICAL RULES:\n1. Use the EXACT function name as specified (e.g., \"write\" not \"write_file\")\n2. Use the EXACT parameter names as specified in the schema (e.g., \"filePath\" not \"file_path\")\n3. Output ONLY the <tool_call> XML tags with no other text\n4. Never wrap tool calls in markdown code blocks\n\nHere are the available tools:\n<tools>\n" -%}
    {%- for tool in tools -%}
        {%- if tool.type is defined and tool.type == "function" -%}
            {%- set tool_str = tool_str + '{\"type\": \"function\", \"function\": ' + tool.function | tojson + \"}\n\" -%}
        {%- else -%}
            {%- set tool_str = tool_str + tool | tojson + \"\n\" -%}
        {%- endif -%}
    {%- endfor -%}
    {%- set tool_str = tool_str + "</tools>\n\nFor each function call, return a JSON object with the function name and arguments within <tool_call></tool_call> XML tags:\n<tool_call>\n{\"name\": <function-name>, \"arguments\": <args-json-object>}\n</tool_call>" -%}
    {%- if system_message -%}
        {%- set system_message = tool_str + "\n\n" + system_message -%}
    {%- else -%}
        {%- set system_message = tool_str -%}
    {%- endif -%}
{%- endif -%}
[gMASK]<sop>
{%- if system_message -%}<|system|>
{{ system_message }}
{%- endif -%}
{%- for message in loop_messages -%}
    {%- if message["role"] == "user" -%}<|user|>
{{ message["content"] }}
    {%- elif message["role"] == "assistant" -%}
        {%- if message.tool_calls is defined and message.tool_calls -%}<|assistant|>
            {%- for tool_call in message.tool_calls -%}
<tool_call>
{"name": "{{ tool_call.function.name }}", "arguments": {{ tool_call.function.arguments }}}
</tool_call>
            {%- endfor -%}
        {%- else -%}<|assistant|>
{{ message["content"] }}
        {%- endif -%}
    {%- elif message["role"] == "tool" -%}<|observation|>
{{ message["content"] }}
    {%- endif -%}
{%- endfor -%}<|assistant|>
```

### Step 5: Transfer Package

Prepare the transfer directory:

```
transfer/
├── opencode-linux-x64.tar.gz        # OpenCode binary (~59 MB)
├── glm-4-9b-chat/                   # Model weights (~18 GB)
├── vllm-wheels/                     # Python wheels for vLLM
└── hermes_glm4_template.jinja       # Chat template
```

Transfer via USB drive, approved file transfer, or air-gap-approved mechanism.

---

## Phase 2: Installation on the Closed Network

### Step 1: Install the OpenCode Binary

```bash
# Extract the binary
tar -xzf opencode-linux-x64.tar.gz

# Move to a system-wide location
sudo mv opencode /usr/local/bin/opencode
sudo chmod +x /usr/local/bin/opencode

# Verify
opencode --version
# Expected: 1.2.15
```

### Step 2: Install vLLM

```bash
pip install --no-index --find-links=./vllm-wheels/ vllm
```

### Step 3: Place Model and Template

```bash
# Place model weights
sudo mkdir -p /opt/models
sudo mv glm-4-9b-chat /opt/models/

# Place chat template
sudo mkdir -p /opt/opencode/templates
sudo mv hermes_glm4_template.jinja /opt/opencode/templates/
```

---

## Phase 3: Configuration

### Step 1: Initialize OpenCode Without Internet

The binary works without internet access. On first run, it creates config directories automatically. To prevent any network attempts, set:

```bash
# Prevent any outbound network calls
export OPENCODE_DISABLE_TELEMETRY=1

# Skip plugin auto-install (requires internet)
export OPENCODE_SKIP_PLUGINS=1
```

### Step 2: Start vLLM

```bash
python3 -m vllm.entrypoints.openai.api_server \
  --model /opt/models/glm-4-9b-chat \
  --served-model-name glm-4 \
  --trust-remote-code \
  --port 8000 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.85 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --chat-template /opt/opencode/templates/hermes_glm4_template.jinja
```

**Critical flags — ALL THREE are required for tool calling:**

| Flag | Why it's needed |
|---|---|
| `--enable-auto-tool-choice` | Without this, vLLM rejects any request containing `tools` |
| `--tool-call-parser hermes` | Parses `<tool_call>` XML tags from model output into OpenAI-format `tool_calls` |
| `--chat-template /path/to/template.jinja` | Replaces GLM-4's native Chinese template with Hermes format that instructs exact parameter naming |

### Step 3: Create Project Configuration

In your project's root directory, create `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "vllm": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "vLLM (local)",
      "options": {
        "baseURL": "http://localhost:8000/v1"
      },
      "models": {
        "glm-4": {
          "name": "GLM-4-9B-Chat (local)",
          "limit": {
            "context": 8192,
            "output": 4096
          }
        }
      }
    }
  },
  "model": "vllm/glm-4",
  "small_model": "vllm/glm-4"
}
```

**Configuration keys explained:**

| Key | Value | Purpose |
|---|---|---|
| `provider.vllm.npm` | `@ai-sdk/openai-compatible` | Uses the bundled OpenAI-compatible SDK (no download needed) |
| `provider.vllm.options.baseURL` | `http://localhost:8000/v1` | Points to local vLLM |
| `provider.vllm.models.glm-4` | `{...}` | Model ID must match `--served-model-name` |
| `model` | `vllm/glm-4` | Format: `<provider-id>/<model-id>` |
| `small_model` | `vllm/glm-4` | Used for session titles and lightweight tasks |

### Step 4: Create Authentication File

```bash
mkdir -p ~/.local/share/opencode
cat > ~/.local/share/opencode/auth.json << 'EOF'
{
  "vllm": {
    "type": "api",
    "key": "sk-local"
  }
}
EOF
```

> The key value can be any non-empty string. vLLM does not validate API keys by default.

### Step 5 (Optional): Global Configuration

To apply the configuration system-wide (all projects), place it at the global level:

```bash
mkdir -p ~/.config/opencode
cp opencode.json ~/.config/opencode/opencode.json
```

---

## Phase 4: Verification

### Step 1: Verify vLLM Health

```bash
# Check model listing
curl -s http://localhost:8000/v1/models | python3 -m json.tool

# Expected output includes:
# "id": "glm-4"
# "max_model_len": 8192
```

### Step 2: Verify Tool Calling Works

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "glm-4",
    "messages": [{"role": "user", "content": "Write a file called test.txt with hello world"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "write",
        "description": "Write content to a file",
        "parameters": {
          "type": "object",
          "properties": {
            "filePath": {"type": "string"},
            "content": {"type": "string"}
          },
          "required": ["filePath", "content"]
        }
      }
    }],
    "tool_choice": "auto",
    "max_tokens": 200
  }' | python3 -m json.tool
```

**Expected:** Response contains:
```json
"tool_calls": [{
  "function": {
    "name": "write",
    "arguments": "{\"filePath\": \"test.txt\", \"content\": \"hello world\"}"
  }
}]
```

If `tool_calls` is empty `[]` or the tool name is wrong, check the chat template file.

### Step 3: Verify OpenCode End-to-End

```bash
cd /your/project/
opencode run "Write a file called verify.txt with the content: OpenCode is working"
```

**Expected:** File `verify.txt` is created. Check with `cat verify.txt`.

### Step 4: Verify No External Connections

```bash
# Monitor network connections during an OpenCode session
ss -tnp | grep opencode
# Should only show connections to 127.0.0.1:8000 (vLLM)

# Or use strace to verify
strace -e connect -f opencode run "Say hello" 2>&1 | grep -v "127.0.0.1"
# Should show NO external IP connections
```

---

## Troubleshooting

### Problem: "tool choice requires --enable-auto-tool-choice"

```
{"error": {"message": "\"auto\" tool choice requires --enable-auto-tool-choice..."}}
```

**Cause:** vLLM was started without `--enable-auto-tool-choice` and `--tool-call-parser`.

**Fix:** Restart vLLM with all three critical flags:
```bash
--enable-auto-tool-choice --tool-call-parser hermes --chat-template /path/to/template.jinja
```

### Problem: "Invalid Tool" / wrong tool name

**Cause:** GLM-4 is hallucinating tool names like `write_file` instead of `write`.

**Fix:**
1. Verify you're using the exact chat template from this guide
2. The template's "CRITICAL RULES" section prevents this in most cases
3. If persistent, consider switching to Qwen2.5-7B-Instruct (see Alternative Models below)

### Problem: "invalid_type" for filePath or content

```
"Invalid input: expected string, received undefined" for "filePath"
```

**Cause:** GLM-4 is using snake_case names (`file_path`) instead of camelCase (`filePath`).

**Fix:** This is prevented by the strict chat template. Ensure:
1. The template file path in `--chat-template` is correct
2. The file hasn't been modified or corrupted
3. `--max-model-len` is at least 8192 (template instructions get truncated if too short)

### Problem: vLLM "Free memory" error

```
ValueError: Free memory on device cuda:0 (...) is less than desired GPU memory utilization
```

**Cause:** GPU memory occupied by a previous process.

**Fix:**
```bash
# Check GPU usage
nvidia-smi

# Kill old processes
nvidia-smi --query-compute-apps=pid --format=csv,noheader | xargs kill -9

# Retry, or reduce utilization
--gpu-memory-utilization 0.7
```

### Problem: OpenCode can't connect to vLLM

**Cause:** vLLM isn't running, or the port/URL is misconfigured.

**Fix:**
```bash
# Verify vLLM is running
curl http://localhost:8000/v1/models

# Check opencode.json baseURL matches the port
grep baseURL opencode.json
# Must be: "http://localhost:8000/v1"
# NOT: "http://localhost:8000/v1/chat/completions"

# Check auth.json exists
cat ~/.local/share/opencode/auth.json
```

### Problem: "plugin loading" errors on startup

**Cause:** OpenCode tries to install npm plugins (e.g., `opencode-anthropic-auth`) on first run.

**Fix:** These plugins are optional and only needed for cloud providers. The errors are non-fatal. To suppress:
```bash
export OPENCODE_SKIP_PLUGINS=1
```

### Problem: Slow responses or timeouts

**Cause:** `max-model-len` is too large for available GPU memory, causing OOM or excessive KV cache allocation.

**Fix:**
- Use `--max-model-len 8192` (minimum viable)
- Use `--max-model-len 4096` if GPU has < 24 GB VRAM (tool calling may be less reliable)
- Add `--enforce-eager` to disable CUDA graph compilation (uses less memory)

---

## Alternative Models

If GLM-4-9B-Chat is unreliable for your use case, these models offer better tool calling:

| Model | VRAM | vLLM Parser | Chat Template Override? | Tool Calling Quality |
|---|---|---|---|---|
| `Qwen/Qwen2.5-7B-Instruct` | ~16 GB | `hermes` | Not needed | Excellent |
| `Qwen/Qwen2.5-14B-Instruct` | ~30 GB | `hermes` | Not needed | Excellent |
| `THUDM/glm-4-9b-chat` | ~20 GB | `hermes` | Yes (this guide) | Good with fix |
| `mistralai/Mistral-7B-Instruct-v0.3` | ~16 GB | `mistral` | Not needed | Good |

To switch models, update:
1. vLLM `--model` and `--served-model-name` flags
2. `opencode.json` model ID under `provider.vllm.models`
3. Remove `--chat-template` if the model natively supports the parser format

---

## systemd Service (Production Deployment)

```ini
# /etc/systemd/system/vllm-opencode.service
[Unit]
Description=vLLM Server for OpenCode (GLM-4)
After=network.target nvidia-persistenced.service

[Service]
Type=simple
User=root
Environment=CUDA_VISIBLE_DEVICES=0
ExecStart=/usr/bin/python3 -m vllm.entrypoints.openai.api_server \
  --model /opt/models/glm-4-9b-chat \
  --served-model-name glm-4 \
  --trust-remote-code \
  --port 8000 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.85 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --chat-template /opt/opencode/templates/hermes_glm4_template.jinja
Restart=on-failure
RestartSec=15
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now vllm-opencode
sudo systemctl status vllm-opencode
journalctl -u vllm-opencode -f  # View logs
```

---

## Quick Start Checklist

- [ ] OpenCode binary placed in PATH (`opencode --version`)
- [ ] GLM-4-9B-Chat model weights at `/opt/models/glm-4-9b-chat`
- [ ] Hermes chat template at `/opt/opencode/templates/hermes_glm4_template.jinja`
- [ ] vLLM started with **all three critical flags**
- [ ] `opencode.json` created in project directory
- [ ] `~/.local/share/opencode/auth.json` created
- [ ] `curl http://localhost:8000/v1/models` returns `glm-4`
- [ ] Tool calling test returns proper `tool_calls` with correct names/params
- [ ] `opencode run "Write test.txt with hello"` creates the file successfully
