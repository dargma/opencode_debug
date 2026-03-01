# oh-my-opencode 폐쇄망 배포 가이드

> 오픈 네트워크에서 npm으로 oh-my-opencode를 설치하고,
> 폐쇄(에어갭) 환경에서 로컬 vLLM + GLM-4 모델과 함께 사용하는 방법

---

## 목차

1. [개요 및 주의사항](#1-개요-및-주의사항)
2. [오픈 네트워크에서 패키징](#2-오픈-네트워크에서-패키징)
3. [폐쇄망에서 설치](#3-폐쇄망에서-설치)
4. [폐쇄망 전용 설정](#4-폐쇄망-전용-설정)
5. [진단된 잠재 문제 및 해결책](#5-진단된-잠재-문제-및-해결책)
6. [GLM-4.7-Flash 업그레이드 가이드](#6-glm-47-flash-업그레이드-가이드)
7. [검증 절차](#7-검증-절차)
8. [트러블슈팅](#8-트러블슈팅)

---

## 1. 개요 및 주의사항

### oh-my-opencode란?

oh-my-opencode(OmO)는 OpenCode용 대규모 플러그인으로, 다음을 추가합니다:

| 구성요소 | 수량 | 설명 |
|---|---|---|
| 에이전트 | 11개 | Sisyphus(오케스트레이터), Hephaestus(딥워커), Prometheus(플래너) 등 |
| 도구 | 26개+ | LSP 통합(11), AST-Grep(2), 태스크 위임, 백그라운드 관리 등 |
| 훅 | 44개 | PreToolUse, PostToolUse, 가드, 변환기 등 |
| MCP 서버 | 3개 | Exa 웹검색, Context7, grep.app 코드검색 |
| 스킬 | 5개 | playwright, agent-browser, frontend-ui-ux 등 |

### 폐쇄망 + 로컬 모델 사용 시 핵심 주의사항

> **경고:** oh-my-opencode는 Claude, GPT-5 등 대형 클라우드 모델에 최적화되어 있습니다.
> GLM-4-9B-Chat 같은 소형 로컬 모델에서는 아래 문제가 **실제 확인**되었으며,
> 반드시 해결책을 적용해야 합니다.

**확인된 문제 요약:**

| # | 문제 | 심각도 | 원인 |
|---|---|---|---|
| P1 | 컨텍스트 초과 (4097 > 4096) | **치명적** | 26+ 도구 정의로 입력 토큰 ~6800개 (기본 ~3000의 2배+) |
| P2 | Hashline Edit 호환 불가 | **높음** | write 차단 → edit 리다이렉트 → GLM-4가 edit 스키마 이해 불가 |
| P3 | MCP 서버 연결 실패 | **중간** | Exa/Context7/grep.app 모두 외부 API 필요 |
| P4 | 다중 에이전트 모델 할당 실패 | **중간** | 11개 에이전트 각각 다른 클라우드 모델 지정됨 |
| P5 | 빈 파라미터 도구 호출 | **높음** | 도구 수 과다로 모델 혼란, `input: {}` 반복 |

---

## 2. 오픈 네트워크에서 패키징

### Step 1: oh-my-opencode 설치

```bash
# 전역 설치
npm install -g oh-my-opencode@latest

# 버전 확인
oh-my-opencode --version
# 예상 출력: 3.9.0

# 또는 프로젝트 로컬 설치 (권장 - 패키징이 쉬움)
mkdir -p ~/opencode-package && cd ~/opencode-package
npm init -y
npm install oh-my-opencode@latest
```

### Step 2: 의존성 포함 패키징

```bash
cd ~/opencode-package

# node_modules 전체를 포함하여 tarball 생성
tar -czf oh-my-opencode-offline.tar.gz \
  node_modules/oh-my-opencode \
  node_modules/oh-my-opencode-linux-x64-baseline \
  node_modules/@ast-grep \
  node_modules/@opencode-ai \
  node_modules/@modelcontextprotocol \
  node_modules/@clack \
  node_modules/zod \
  node_modules/diff \
  node_modules/js-yaml \
  node_modules/jsonc-parser \
  node_modules/picocolors \
  node_modules/picomatch \
  node_modules/commander \
  node_modules/detect-libc \
  node_modules/vscode-jsonrpc \
  package.json

# 또는 전체 node_modules를 묶기 (더 안전)
tar -czf oh-my-opencode-full.tar.gz node_modules/ package.json
```

### Step 3: OpenCode와 함께 묶기

```bash
# OpenCode npm 패키지도 함께 준비
cd $(npm root -g)/opencode-ai && npm pack
cp opencode-ai-*.tgz ~/transfer/

# 전체 전송 패키지
mkdir -p ~/transfer
cp ~/opencode-package/oh-my-opencode-offline.tar.gz ~/transfer/
```

### Step 4: 전송 목록

```
transfer/
├── opencode-ai-1.2.15.tgz               # OpenCode
├── oh-my-opencode-offline.tar.gz          # oh-my-opencode + 의존성
├── glm-4-9b-chat/                        # 모델 가중치 (~18 GB)
├── vllm-wheels/                          # vLLM Python 패키지
├── hermes_glm4_template.jinja            # 도구 호출 템플릿
└── oh-my-opencode.jsonc                  # 폐쇄망 전용 설정 (아래 참조)
```

---

## 3. 폐쇄망에서 설치

### Step 1: OpenCode 설치

```bash
npm install -g ./opencode-ai-1.2.15.tgz
opencode --version
```

### Step 2: oh-my-opencode 설치

```bash
# 프로젝트 디렉토리에서
cd /your/project
tar -xzf oh-my-opencode-offline.tar.gz

# node_modules가 프로젝트 루트에 풀림
ls node_modules/oh-my-opencode/
# dist/ bin/ package.json 등 확인
```

### Step 3: 바이너리 권한 설정

```bash
# oh-my-opencode 네이티브 바이너리에 실행 권한 부여
chmod +x node_modules/oh-my-opencode-linux-x64-baseline/bin/oh-my-opencode 2>/dev/null
chmod +x node_modules/oh-my-opencode-linux-x64/bin/oh-my-opencode 2>/dev/null

# ast-grep 바이너리에도 권한 부여
chmod +x node_modules/@ast-grep/cli-linux-x64/sg 2>/dev/null
```

### Step 4: vLLM 시작 (도구 호출 지원)

```bash
python3 -m vllm.entrypoints.openai.api_server \
  --model /path/to/glm-4-9b-chat \
  --served-model-name glm-4 \
  --trust-remote-code \
  --port 8000 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --chat-template /path/to/hermes_glm4_template.jinja
```

> **중요:** oh-my-opencode 사용 시 `--max-model-len`은 반드시 **32768 이상**이어야 합니다.
> 26+ 도구 정의로 인해 입력 토큰이 ~6800개로 증가하기 때문입니다.
> 8192에서는 "input tokens exceeded" 에러가 발생합니다.

---

## 4. 폐쇄망 전용 설정

### 4-1. opencode.json

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
            "context": 32768,
            "output": 8192
          }
        }
      }
    }
  },
  "model": "vllm/glm-4",
  "small_model": "vllm/glm-4",
  "plugin": ["oh-my-opencode"]
}
```

### 4-2. oh-my-opencode.jsonc (핵심 — 반드시 생성)

`~/.config/opencode/oh-my-opencode.jsonc` 또는 프로젝트 루트의 `.opencode/oh-my-opencode.jsonc`:

```jsonc
{
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-opencode/master/assets/oh-my-opencode.schema.json",

  // ========================================
  // 1. Hashline Edit 비활성화 (필수!)
  // ========================================
  // GLM-4는 해시 앵커 포맷(11#VK|)을 이해하지 못함
  // 기본 Edit 도구를 유지해야 함
  "hashline_edit": false,

  // ========================================
  // 2. 외부 MCP 서버 비활성화 (필수!)
  // ========================================
  // 폐쇄망에서 Exa/Context7/grep.app 접근 불가
  "disabled_mcps": ["websearch", "context7", "grep_app"],

  // ========================================
  // 3. 모델 폴백 비활성화
  // ========================================
  // 외부 API로 폴백 시도 방지
  "model_fallback": false,

  // ========================================
  // 4. 불필요한 도구 비활성화 (토큰 절약)
  // ========================================
  // GLM-4-9B는 도구가 많을수록 혼란
  // 핵심 도구만 유지, 나머지 비활성화
  "disabled_tools": [
    "lsp_goto_definition",
    "lsp_find_references",
    "lsp_symbols",
    "lsp_diagnostics",
    "lsp_prepare_rename",
    "lsp_rename",
    "lsp_hover",
    "lsp_code_action",
    "lsp_completion",
    "lsp_signature_help",
    "lsp_document_highlight",
    "ast_grep_search",
    "ast_grep_replace",
    "background_output",
    "background_cancel"
  ],

  // ========================================
  // 5. 불필요한 에이전트 비활성화
  // ========================================
  // 모든 에이전트가 외부 모델을 요구하므로 비활성화
  "disabled_agents": [
    "hephaestus",
    "prometheus",
    "atlas",
    "oracle",
    "librarian",
    "explore",
    "multimodal-looker",
    "metis",
    "momus",
    "sisyphus-junior"
  ],

  // ========================================
  // 6. 에이전트 모델을 로컬 vLLM으로 오버라이드
  // ========================================
  "agents": {
    "build": {
      "model": "vllm/glm-4"
    },
    "sisyphus": {
      "model": "vllm/glm-4"
    }
  },

  // ========================================
  // 7. 인터넷 필요 스킬 비활성화
  // ========================================
  "disabled_skills": [
    "playwright",
    "agent-browser",
    "dev-browser"
  ],

  // ========================================
  // 8. 문제 유발 훅 비활성화
  // ========================================
  "disabled_hooks": [
    "write-existing-file-guard"
  ]
}
```

### 4-3. 인증 파일

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

---

## 5. 진단된 잠재 문제 및 해결책

### P1: 컨텍스트 초과 (치명적)

**증상:**
```
You passed 4097 input tokens and requested 4096 output tokens.
However, the model's context length is only 8192 tokens.
```

**원인:** oh-my-opencode가 26+ 도구 정의를 시스템 프롬프트에 추가하여
입력 토큰이 ~6800개로 증가 (기본 OpenCode ~3000개의 2배+).

**해결책:**
```bash
# vLLM의 --max-model-len을 32768 이상으로 설정
--max-model-len 32768

# opencode.json의 context limit도 맞춰서 변경
"limit": { "context": 32768, "output": 8192 }
```

**추가 최적화:** oh-my-opencode.jsonc에서 `disabled_tools`로 불필요한 도구를 비활성화하면
입력 토큰을 줄일 수 있음 (위 설정 참조).

---

### P2: Hashline Edit 호환 불가 (높음)

**증상:**
```
write → "File already exists. Use edit tool instead."
edit → "edits parameter must be a non-empty array"
```

**원인:** oh-my-opencode의 `write-existing-file-guard` 훅이 기존 파일에 대한 `write`를
차단하고 `edit` 사용을 강제함. 동시에 Hashline Edit 도구는 해시 앵커 포맷(예: `11#VK|`)을
요구하는데, GLM-4는 이 포맷을 이해하지 못함.

**해결책:**
```jsonc
// oh-my-opencode.jsonc
{
  "hashline_edit": false,                    // Hashline 비활성화 → 기본 Edit 유지
  "disabled_hooks": ["write-existing-file-guard"]  // write 차단 훅 비활성화
}
```

---

### P3: MCP 서버 연결 실패 (중간)

**증상:**
- OpenCode 시작 시 MCP 연결 타임아웃
- 웹검색/문서조회 도구 사용 불가

**원인:** oh-my-opencode에 내장된 3개 MCP 서버가 모두 외부 API 필요:
- `websearch` → Exa API (exa.ai)
- `context7` → Context7 API
- `grep_app` → grep.app

**해결책:**
```jsonc
{
  "disabled_mcps": ["websearch", "context7", "grep_app"]
}
```

---

### P4: 다중 에이전트 모델 할당 실패 (중간)

**증상:**
- 에이전트 위임 시 외부 API 호출 시도 → 실패
- 타임아웃 또는 연결 거부 에러

**원인:** oh-my-opencode의 11개 에이전트 각각에 서로 다른 클라우드 모델이 기본 할당됨:
| 에이전트 | 기본 모델 | 폐쇄망 상태 |
|---|---|---|
| sisyphus | claude-4-opus | 접근 불가 |
| hephaestus | gpt-5.3-codex | 접근 불가 |
| prometheus | gpt-5.2 | 접근 불가 |
| oracle | gpt-5.2 | 접근 불가 |
| multimodal-looker | gemini-3-flash | 접근 불가 |

**해결책:**
```jsonc
{
  // 사용 에이전트만 로컬 모델로 오버라이드
  "agents": {
    "build": { "model": "vllm/glm-4" },
    "sisyphus": { "model": "vllm/glm-4" }
  },
  // 나머지 에이전트 비활성화
  "disabled_agents": [
    "hephaestus", "prometheus", "atlas", "oracle",
    "librarian", "explore", "multimodal-looker",
    "metis", "momus", "sisyphus-junior"
  ]
}
```

---

### P5: 빈 파라미터 도구 호출 (높음)

**증상:**
```json
"tool": "write", "input": {}
// filePath, content 모두 undefined
```

**원인:** 26+ 도구 정의가 GLM-4-9B-Chat(9B 파라미터)의 도구 호출 능력을 초과.
모델이 어떤 도구를 어떤 파라미터로 호출할지 혼란.

**해결책 (복합 적용):**
1. `disabled_tools`로 불필요한 도구 대량 비활성화 (LSP 11개, AST 2개 등)
2. `--max-model-len 32768` 확보하여 도구 정의가 잘리지 않도록
3. Hermes 채팅 템플릿의 CRITICAL RULES 유지 (정확한 함수명/파라미터명 강제)
4. **궁극적 해결:** GLM-4.7-Flash로 업그레이드 (아래 섹션 참조)

---

## 6. GLM-4.7-Flash 업그레이드 가이드

### 왜 GLM-4.7-Flash인가?

GLM-4-9B-Chat은 9B 파라미터로 도구 호출 능력이 제한적입니다.
**GLM-4.7-Flash**는 30B(활성 3B) MoE 아키텍처로 동일한 GPU에서
훨씬 뛰어난 도구 호출을 제공합니다.

| 비교 항목 | GLM-4-9B-Chat | GLM-4.7-Flash |
|---|---|---|
| 파라미터 | 9B (Dense) | 30B total / 3B active (MoE) |
| VRAM | ~20 GB | ~32 GB (FP8), ~60 GB (BF16) |
| A100 80GB 적합 | O | O |
| vLLM 파서 | hermes (커스텀 필요) | **glm47 (네이티브!)** |
| 커스텀 템플릿 필요 | O (필수) | X (불필요) |
| 도구 호출 품질 | 보통 (파라미터명 오류 빈번) | **우수 (네이티브 지원)** |
| 컨텍스트 | 128K | **200K** |
| tau2-Bench (도구 사용) | 미보고 | **87.4% (오픈소스 SOTA)** |

### GLM-4.7-Flash 설치

```bash
# 1. 모델 다운로드 (오픈 네트워크에서)
huggingface-cli download zai-org/GLM-4.7-Flash --local-dir ./GLM-4.7-Flash

# 2. vLLM 시작 (네이티브 파서 사용 — 커스텀 템플릿 불필요!)
python3 -m vllm.entrypoints.openai.api_server \
  --model /path/to/GLM-4.7-Flash \
  --served-model-name glm-4.7-flash \
  --trust-remote-code \
  --port 8000 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90 \
  --enable-auto-tool-choice \
  --tool-call-parser glm47
# 주의: --chat-template 불필요 (네이티브 지원)
```

### GLM-4.7-Flash용 opencode.json

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
        "glm-4.7-flash": {
          "name": "GLM-4.7-Flash (local)",
          "limit": {
            "context": 32768,
            "output": 8192
          }
        }
      }
    }
  },
  "model": "vllm/glm-4.7-flash",
  "small_model": "vllm/glm-4.7-flash",
  "plugin": ["oh-my-opencode"]
}
```

### GLM-4.7-Flash용 oh-my-opencode.jsonc (완화된 설정)

GLM-4.7-Flash는 도구 호출 능력이 우수하므로, 더 많은 기능을 활성화할 수 있습니다:

```jsonc
{
  // Hashline Edit: GLM-4.7-Flash에서 테스트 후 결정
  // 대형 모델이므로 동작할 가능성 있음, 실패 시 false로 변경
  "hashline_edit": false,

  // MCP: 폐쇄망이므로 여전히 비활성화
  "disabled_mcps": ["websearch", "context7", "grep_app"],

  // 모델 폴백: 여전히 비활성화
  "model_fallback": false,

  // 도구: LSP 도구는 활성화 가능 (GLM-4.7은 처리 가능)
  // AST-Grep도 사용 가능
  "disabled_tools": [],

  // 에이전트: 여전히 로컬 모델로 오버라이드
  "agents": {
    "build": { "model": "vllm/glm-4.7-flash" },
    "sisyphus": { "model": "vllm/glm-4.7-flash" }
  },
  "disabled_agents": [
    "hephaestus", "prometheus", "atlas", "oracle",
    "librarian", "explore", "multimodal-looker",
    "metis", "momus", "sisyphus-junior"
  ],

  "disabled_skills": ["playwright", "agent-browser", "dev-browser"]
}
```

### GLM-4.7-Flash 알려진 이슈

| 이슈 | 상태 | 해결 방법 |
|---|---|---|
| 빈 파라미터 도구 호출 시 파서 크래시 (vLLM #32436) | **수정됨** (PR #32321) | vLLM 최신 버전 사용 |
| 스트리밍 시 도구 인자 일괄 전송 (vLLM #32829) | **수정됨** (PR #33218) | vLLM 최신 버전 사용 |
| `<think>` 태그 누락 (vLLM #31319) | 보고됨 | reasoning-parser 비사용 시 무관 |
| GLM-4.7-Flash 아키텍처 미인식 (vLLM #34098) | 오픈 | `pip install transformers --upgrade` 필요 |
| 중복 `<tool_call>` 태그 출력 (SGLang #15721) | 부분 수정 | vLLM 사용 시 해당 없음 |

> **권장:** vLLM **0.16.0 이상** 사용 (위 수정 사항 포함).

---

## 7. 검증 절차

### Step 1: vLLM 상태 확인

```bash
curl -s http://localhost:8000/v1/models | python3 -m json.tool
# "id": "glm-4" (또는 "glm-4.7-flash") 확인
```

### Step 2: 도구 호출 직접 테스트

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "glm-4",
    "messages": [{"role": "user", "content": "Write test.txt with hello"}],
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

**확인 사항:**
- `tool_calls` 배열에 항목이 있는지
- `name`이 `"write"` (정확한 이름)인지
- `arguments`에 `filePath`(camelCase)가 사용되었는지

### Step 3: OpenCode + oh-my-opencode 통합 테스트

```bash
# oh-my-opencode 플러그인 로드 확인
opencode run "Write verify.txt with: plugin loaded" --format json 2>&1 | \
  grep -E '"tool"|"status"|"error"'
```

**성공 기준:**
- `"tool": "write"`, `"status": "completed"` 출력
- `verify.txt` 파일 생성 확인

### Step 4: oh-my-opencode doctor 실행

```bash
node node_modules/oh-my-opencode/bin/oh-my-opencode.js doctor
```

---

## 8. 트러블슈팅

### "input tokens exceeded" / 컨텍스트 초과

```bash
# vLLM의 max-model-len 확인
curl -s http://localhost:8000/v1/models | python3 -c "
import sys,json
d = json.load(sys.stdin)
print(f'max_model_len: {d[\"data\"][0][\"max_model_len\"]}')
"
# 32768 이상이어야 함

# 부족하면 vLLM 재시작
--max-model-len 32768
```

### "File already exists. Use edit tool instead."

```jsonc
// oh-my-opencode.jsonc에 추가
{ "disabled_hooks": ["write-existing-file-guard"] }
```

### MCP 타임아웃 / 연결 실패

```jsonc
// oh-my-opencode.jsonc에 추가
{ "disabled_mcps": ["websearch", "context7", "grep_app"] }
```

### oh-my-opencode 플러그인 로드 실패

```bash
# node_modules에 설치되었는지 확인
ls node_modules/oh-my-opencode/dist/index.js

# 바이너리 권한 확인
chmod +x node_modules/oh-my-opencode-linux-x64-baseline/bin/oh-my-opencode

# opencode.json에 plugin 배열 확인
grep -A1 '"plugin"' opencode.json
# ["oh-my-opencode"] 이어야 함
```

### GLM-4.7-Flash 아키텍처 미인식

```bash
# transformers를 최신 버전으로 업그레이드
pip install transformers --upgrade

# 또는 GitHub에서 직접 설치
pip install git+https://github.com/huggingface/transformers.git
```

### 도구 호출은 되지만 파라미터가 빈 객체

```bash
# 1. disabled_tools로 도구 수 줄이기 (위 설정 참조)
# 2. max-model-len이 충분히 큰지 확인 (도구 정의가 잘리면 안됨)
# 3. Hermes 채팅 템플릿의 CRITICAL RULES 확인
# 4. GLM-4.7-Flash로 업그레이드 고려
```

### 모든 설정을 적용했는데도 도구 호출 실패

**최종 수단 — oh-my-opencode 없이 실행:**
```json
{
  "model": "vllm/glm-4",
  "small_model": "vllm/glm-4"
  // "plugin" 줄 제거
}
```

OpenCode의 기본 도구(write, read, edit, bash, glob, grep)만으로도
충분한 코딩 에이전트 기능을 제공합니다. oh-my-opencode의 추가 기능이
반드시 필요한 경우에만 사용하세요.

---

## 부록: 설정 파일 위치 요약

| 파일 | 경로 | 용도 |
|---|---|---|
| OpenCode 설정 | `./opencode.json` (프로젝트) | 프로바이더, 모델, 플러그인 |
| OmO 설정 | `~/.config/opencode/oh-my-opencode.jsonc` (글로벌) | 도구/훅/MCP 비활성화 |
| OmO 설정 | `.opencode/oh-my-opencode.jsonc` (프로젝트) | 프로젝트별 오버라이드 |
| 인증 | `~/.local/share/opencode/auth.json` | API 키 |
| 채팅 템플릿 | `/opt/opencode/templates/hermes_glm4_template.jinja` | GLM-4용 도구 호출 포맷 |

---

## 부록: 권장 구성 매트릭스

| 구성 | GLM-4-9B-Chat | GLM-4.7-Flash | Qwen2.5-14B |
|---|---|---|---|
| vLLM 파서 | `hermes` + 커스텀 템플릿 | `glm47` (네이티브) | `hermes` (네이티브) |
| max-model-len | 32768+ | 32768+ | 32768+ |
| hashline_edit | `false` (필수) | `false` (권장) | `false` (권장) |
| disabled_tools | LSP+AST 전체 | 선택적 | 선택적 |
| disabled_agents | 대부분 | 대부분 | 대부분 |
| 도구 호출 신뢰도 | 낮음 (~60%) | 높음 (~90%) | 높음 (~85%) |
| VRAM | ~20 GB | ~32 GB (FP8) | ~30 GB |
