# Qwen2.5-1.5B + vLLM + OpenCode + oh-my-opencode 설치 가이드

> 실제 설치하면서 발생한 문제와 해결 방법을 포함한 실전 가이드입니다.

---

## 환경 요구사항

- Linux (Ubuntu 등)
- NVIDIA GPU (VRAM 8GB 이상 권장, 본 테스트: NVIDIA L4 23GB)
- Python 3.10+
- Node.js 18+ (npm 포함)
- CUDA 드라이버 설치 완료

---

## Step 1: 모델 다운로드

```bash
pip install huggingface_hub

huggingface-cli download Qwen/Qwen2.5-1.5B-Instruct \
  --local-dir ~/Qwen2.5-1.5B-Instruct
```

다운로드 확인:
```bash
ls ~/Qwen2.5-1.5B-Instruct/
# config.json  model.safetensors  tokenizer.json  등이 있어야 함
```

---

## Step 2: vLLM 설치 및 서빙

### 2-1. vLLM 설치

```bash
pip install vllm
```

### 2-2. vLLM 서버 시작

```bash
vllm serve ~/Qwen2.5-1.5B-Instruct \
  --host 0.0.0.0 \
  --port 8000 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --max-model-len 32768 \
  --dtype auto
```

> **주의:** `--max-model-len`은 반드시 **32768** 이상으로 설정해야 합니다.
> OpenCode의 시스템 프롬프트가 약 12,000 토큰을 차지하므로,
> 4096으로 설정하면 `input tokens exceed max context` 에러가 발생합니다.

| 옵션 | 설명 |
|---|---|
| `--enable-auto-tool-choice` | tool call 자동 선택 활성화 (필수) |
| `--tool-call-parser hermes` | Qwen2.5 계열용 tool call 파서 (필수) |
| `--max-model-len 32768` | 모델 최대 컨텍스트 (Qwen2.5-1.5B의 max_position_embeddings) |
| `--dtype auto` | GPU에 맞는 데이터 타입 자동 선택 |

### 2-3. 서빙 확인

```bash
curl http://localhost:8000/v1/models
```

응답에 모델 경로가 포함되면 정상입니다. 이 응답에 나오는 **모델 ID**를 기억하세요.
(예: `/home/username/Qwen2.5-1.5B-Instruct`)

---

## Step 3: OpenCode 설치

```bash
curl -fsSL https://opencode.ai/install | bash
```

설치 후 PATH 반영:
```bash
source ~/.bashrc
opencode --version
```

> 설치 경로는 보통 `~/.opencode/bin/opencode` 입니다.

---

## Step 4: opencode.json 설정 (핵심!)

프로젝트 루트 디렉토리에 `opencode.json` 파일을 생성합니다.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "vllm_qwen_local": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "vLLM Qwen Local",
      "options": {
        "baseURL": "http://localhost:8000/v1",
        "apiKey": "EMPTY"
      },
      "models": {
        "여기에_vLLM_모델ID": {
          "name": "Qwen2.5-1.5B (vLLM local)",
          "limit": {
            "context": 32768,
            "output": 4096
          }
        }
      }
    }
  },
  "model": "vllm_qwen_local/여기에_vLLM_모델ID"
}
```

**`여기에_vLLM_모델ID`** 부분을 Step 2-3에서 확인한 모델 ID로 교체하세요.

예시 (모델 경로가 `/home/sk2011cho/Qwen2.5-1.5B-Instruct`인 경우):
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "vllm_qwen_local": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "vLLM Qwen Local",
      "options": {
        "baseURL": "http://localhost:8000/v1",
        "apiKey": "EMPTY"
      },
      "models": {
        "/home/sk2011cho/Qwen2.5-1.5B-Instruct": {
          "name": "Qwen2.5-1.5B (vLLM local)",
          "limit": {
            "context": 32768,
            "output": 4096
          }
        }
      }
    }
  },
  "model": "vllm_qwen_local//home/sk2011cho/Qwen2.5-1.5B-Instruct"
}
```

### 흔한 실수 3가지

#### 실수 1: 최상위에 `"options"` 키 사용
```json
// 틀린 예
{
  "options": {
    "maxCompletionTokens": 2048,
    "maxTokens": 2048
  },
  ...
}
```
> `Unrecognized key: "options"` 에러 발생.
> 최상위에 `options`는 유효한 키가 아닙니다.

#### 실수 2: `limit` 설정 누락
```json
// 틀린 예 - limit 없음
"models": {
  "/home/user/Qwen2.5-1.5B-Instruct": {
    "name": "Qwen2.5-1.5B (vLLM local)"
  }
}
```
> OpenCode가 기본 `max_tokens=32000`을 요청하여
> `max_tokens is too large` 에러 발생.
> 반드시 `"limit": {"context": 32768, "output": 4096}`를 추가하세요.

#### 실수 3: 모델 경로 불일치
```bash
# vLLM이 서빙하는 모델 ID 확인
curl http://localhost:8000/v1/models | python3 -m json.tool
```
> `opencode.json`의 models 키와 model 값이
> vLLM에서 반환하는 모델 ID와 **정확히 일치**해야 합니다.

### 설정 검증

```bash
# 설정이 유효한지 확인
opencode models
# 목록 하단에 vllm_qwen_local/... 이 보이면 성공
```

---

## Step 5: OpenCode 테스트

```bash
opencode run "test.txt 파일에 도레미송 가사를 써서 저장해줘. write tool을 사용해줘."
```

결과 확인:
```bash
cat test.txt
```

> tool call이 실패하면 vLLM 서빙 시 `--enable-auto-tool-choice`와
> `--tool-call-parser hermes` 옵션을 재확인하세요.

---

## Step 6: oh-my-opencode 설치

```bash
npm install -g oh-my-opencode
```

설치 확인:
```bash
oh-my-opencode --version
```

### 6-1. oh-my-opencode 초기 설정

```bash
oh-my-opencode install --no-tui \
  --claude=no \
  --openai=no \
  --gemini=no \
  --copilot=no \
  --skip-auth
```

> 로컬 vLLM 모델만 사용하므로 외부 provider는 모두 `no`로 설정합니다.

### 6-2. 프로젝트 opencode.json에 plugin 추가

opencode.json에 `"plugin"` 항목을 추가합니다:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": [
    "oh-my-opencode@latest"
  ],
  "provider": {
    "vllm_qwen_local": {
      ...
    }
  },
  "model": "vllm_qwen_local/..."
}
```

### 6-3. 최종 opencode.json 전체 예시

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": [
    "oh-my-opencode@latest"
  ],
  "provider": {
    "vllm_qwen_local": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "vLLM Qwen Local",
      "options": {
        "baseURL": "http://localhost:8000/v1",
        "apiKey": "EMPTY"
      },
      "models": {
        "/home/sk2011cho/Qwen2.5-1.5B-Instruct": {
          "name": "Qwen2.5-1.5B (vLLM local)",
          "limit": {
            "context": 32768,
            "output": 4096
          }
        }
      }
    }
  },
  "model": "vllm_qwen_local//home/sk2011cho/Qwen2.5-1.5B-Instruct"
}
```

---

## Step 7: oh-my-opencode 테스트

```bash
oh-my-opencode run "test2.txt 파일에 도레미송 가사를 써서 저장해줘. write tool을 사용해줘."
```

결과 확인:
```bash
cat test2.txt
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| `Unrecognized key: "options"` | opencode.json 최상위에 `options` 키 사용 | `options` 키 제거, `limit`은 models 안에 설정 |
| `max_tokens is too large` | `limit.output` 미설정 | models 내 `"limit": {"context": 32768, "output": 4096}` 추가 |
| `input tokens exceed max context` | vLLM `--max-model-len`이 너무 작음 | `--max-model-len 32768`로 재시작 |
| `max_model_len > max_position_embeddings` | vLLM 컨텍스트가 모델 한계 초과 | Qwen2.5-1.5B는 최대 32768, 이 이상 설정 불가 |
| tool call 미발생 | vLLM 옵션 누락 | `--enable-auto-tool-choice --tool-call-parser hermes` 확인 |
| 모델 못 찾음 | 모델 경로 불일치 | `curl localhost:8000/v1/models`로 정확한 ID 확인 후 일치시킴 |
| oh-my-opencode에서 파일 미저장 | opencode.json에 plugin 미등록 또는 limit 미설정 | `"plugin": ["oh-my-opencode@latest"]` 추가 및 limit 설정 확인 |
| bash tool invalid arguments | 1.5B 모델의 tool schema 이해 부족 | 프롬프트에 "write tool을 사용해줘" 명시, 또는 더 큰 모델 사용 |

---

## 참고: 설치 순서 요약

```
1. 모델 다운로드       huggingface-cli download
2. vLLM 설치/서빙      pip install vllm && vllm serve ...
3. 서빙 확인           curl localhost:8000/v1/models
4. OpenCode 설치       curl -fsSL https://opencode.ai/install | bash
5. opencode.json 작성  limit 설정 포함 (핵심!)
6. OpenCode 테스트     opencode run "..."
7. oh-my-opencode 설치 npm install -g oh-my-opencode
8. oh-my-opencode 설정 oh-my-opencode install --no-tui ...
9. plugin 등록         opencode.json에 plugin 배열 추가
10. oh-my-opencode 테스트 oh-my-opencode run "..."
```
