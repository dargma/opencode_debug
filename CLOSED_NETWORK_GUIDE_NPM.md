# Closed-Network Deployment Guide: OpenCode via NPM

> Transfer an NPM-installed OpenCode from an open network to a closed (air-gapped) network
> with a local vLLM backend serving GLM-4-9B-Chat.

---

## Prerequisites

| Component | Requirement |
|---|---|
| **Node.js** | v20.x LTS or later |
| **npm** | v10.x or later (bundled with Node.js) |
| **Python** | 3.10 – 3.12 |
| **GPU** | NVIDIA GPU with ≥ 24 GB VRAM (A100/A10G/L40S/4090 recommended) |
| **CUDA** | 12.1+ with compatible NVIDIA drivers |
| **OS** | Linux x86_64 (Ubuntu 22.04+ recommended) |
| **Disk** | ≥ 30 GB for model weights + 1 GB for OpenCode |

---

## Phase 1: Packaging on the Open Network

### Step 1: Install OpenCode

```bash
# Install OpenCode globally
npm install -g opencode-ai@latest

# Verify installation
opencode --version
# Expected: 1.2.15 or later
```

### Step 2: Bundle OpenCode for Transfer

```bash
# Option A: npm pack (recommended)
cd $(npm root -g)/opencode-ai
npm pack
# Creates opencode-ai-1.2.15.tgz

# Copy the tarball
cp opencode-ai-*.tgz ~/transfer/

# Option B: Direct directory copy
cp -r $(npm root -g)/opencode-ai ~/transfer/opencode-ai-package/
```

### Step 3: Download the vLLM Model

```bash
# Install huggingface-hub CLI
pip install huggingface-hub

# Download GLM-4-9B-Chat model weights
huggingface-cli download THUDM/glm-4-9b-chat --local-dir ~/transfer/glm-4-9b-chat

# Download vLLM wheel (for offline pip install)
pip download vllm -d ~/transfer/vllm-wheels/
```

### Step 4: Create the Chat Template File

Save the following as `~/transfer/hermes_glm4_template.jinja`:

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

### Step 5: Transfer Everything

Copy the `~/transfer/` directory to the closed network via USB drive, SCP, or approved transfer mechanism:

```
transfer/
├── opencode-ai-1.2.15.tgz          # OpenCode npm package
├── glm-4-9b-chat/                   # Model weights (~18 GB)
├── vllm-wheels/                     # Python wheels for vLLM
└── hermes_glm4_template.jinja       # Chat template
```

---

## Phase 2: Installation on the Closed Network

### Step 1: Install Node.js (if not present)

```bash
# If Node.js is pre-installed, verify:
node --version   # Must be v20+
npm --version    # Must be v10+

# If not, install from a transferred Node.js tarball:
tar -xzf node-v20.19.0-linux-x64.tar.gz -C /usr/local --strip-components=1
```

### Step 2: Install OpenCode

```bash
# From the transferred tarball
npm install -g ./opencode-ai-1.2.15.tgz

# Verify
opencode --version
```

### Step 3: Install vLLM

```bash
# From pre-downloaded wheels
pip install --no-index --find-links=./vllm-wheels/ vllm
```

### Step 4: Place the Chat Template

```bash
mkdir -p /opt/opencode/templates
cp hermes_glm4_template.jinja /opt/opencode/templates/
```

---

## Phase 3: Configuration

### Step 1: Start vLLM with Tool Calling Enabled

```bash
python3 -m vllm.entrypoints.openai.api_server \
  --model /path/to/glm-4-9b-chat \
  --served-model-name glm-4 \
  --trust-remote-code \
  --port 8000 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.85 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --chat-template /opt/opencode/templates/hermes_glm4_template.jinja
```

**Critical flags explained:**

| Flag | Purpose |
|---|---|
| `--enable-auto-tool-choice` | Allows the model to decide when to call tools |
| `--tool-call-parser hermes` | Parses `<tool_call>` XML tags from model output |
| `--chat-template` | Overrides GLM-4's native template with Hermes-compatible format |
| `--max-model-len 8192` | Must be ≥ 8192; OpenCode's tools use ~3000 tokens |
| `--served-model-name glm-4` | The model ID OpenCode references |

### Step 2: Create OpenCode Configuration

Create `opencode.json` in your project directory:

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

### Step 3: Create Authentication File

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

The API key value (`sk-local`) can be any non-empty string since vLLM doesn't require authentication by default.

---

## Phase 4: Verification

### Step 1: Verify vLLM is Running

```bash
curl -s http://localhost:8000/v1/models | python3 -m json.tool
# Should show: {"data": [{"id": "glm-4", ...}]}
```

### Step 2: Test Tool Calling Directly

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

**Expected:** Response contains `"tool_calls"` array with `"name": "write"` and correct camelCase arguments.

### Step 3: Test OpenCode End-to-End

```bash
cd /your/project/directory
opencode run "Write a file called test.txt with the content: hello from opencode"
```

**Expected:** File `test.txt` is created with the specified content.

---

## Troubleshooting

### Problem: "tool choice requires --enable-auto-tool-choice"

**Cause:** vLLM was started without the `--enable-auto-tool-choice` flag.

**Fix:** Restart vLLM with both `--enable-auto-tool-choice` and `--tool-call-parser hermes`.

### Problem: Tool calls return `"tool_calls": []` (empty)

**Cause:** The chat template is not in Hermes format, or the model is generating text instead of `<tool_call>` tags.

**Fix:**
1. Ensure `--chat-template` points to the Hermes template file
2. Verify the template file exists and is readable
3. Check vLLM logs: `tail -f /tmp/vllm.log`

### Problem: "Invalid Tool" / tool name mismatch

**Cause:** GLM-4 is hallucinating tool names (e.g., `write_file` instead of `write`).

**Fix:** The Hermes chat template includes strict instructions. If the problem persists:
1. Add more explicit examples to the chat template
2. Consider using Qwen2.5-7B-Instruct instead (has better tool-calling compliance)

### Problem: "invalid_type" error for filePath/content

**Cause:** GLM-4 is using snake_case parameter names (`file_path`) instead of camelCase (`filePath`).

**Fix:** The strict chat template should prevent this. If it persists:
1. Ensure you're using the exact template from this guide
2. Increase `--max-model-len` to ensure the template instructions aren't truncated

### Problem: "Free memory on device" error

**Cause:** Insufficient GPU memory or leftover processes from a previous run.

**Fix:**
```bash
# Check for zombie GPU processes
nvidia-smi
# Kill any leftover processes
kill -9 <PID>
# Reduce memory utilization
--gpu-memory-utilization 0.7
```

### Problem: OpenCode hangs or shows no output

**Cause:** vLLM is not running or not reachable.

**Fix:**
```bash
# Test connectivity
curl http://localhost:8000/v1/models
# If it fails, check vLLM process
ps aux | grep vllm
# Check vLLM logs
tail -50 /path/to/vllm.log
```

### Problem: Context length exceeded

**Cause:** `--max-model-len` is too small for OpenCode's system prompt + tool definitions.

**Fix:** Increase `--max-model-len` to at least 8192. For longer conversations, use 16384 or higher (requires more GPU memory).

---

## Recommended Alternative Models

If GLM-4-9B-Chat proves unreliable for tool calling, these models have better compliance:

| Model | Size | vLLM Parser | Notes |
|---|---|---|---|
| `Qwen/Qwen2.5-7B-Instruct` | 7B | `hermes` | Excellent tool calling |
| `Qwen/Qwen2.5-14B-Instruct` | 14B | `hermes` | Best quality for size |
| `THUDM/glm-4-9b-chat` | 9B | `hermes` + custom template | Requires this guide's fix |
| `mistralai/Mistral-7B-Instruct-v0.3` | 7B | `mistral` | Good baseline |

---

## systemd Service Files (Optional)

### vLLM Service

```ini
# /etc/systemd/system/vllm.service
[Unit]
Description=vLLM OpenAI-Compatible API Server
After=network.target

[Service]
Type=simple
User=root
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
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now vllm
```

---

## Quick Reference

```bash
# Start vLLM (one-liner)
python3 -m vllm.entrypoints.openai.api_server \
  --model /path/to/glm-4-9b-chat --served-model-name glm-4 \
  --trust-remote-code --port 8000 --max-model-len 8192 \
  --gpu-memory-utilization 0.85 --enable-auto-tool-choice \
  --tool-call-parser hermes --chat-template /opt/opencode/templates/hermes_glm4_template.jinja

# Test vLLM
curl http://localhost:8000/v1/models

# Run OpenCode
opencode

# Run OpenCode non-interactively
opencode run "your prompt here"
```
