

# Qwen2.5-1.5B vLLM 로컬 서빙 + OpenCode 연동 가이드

> **curl 없이 pip/npm만으로 설치하는 가이드입니다.**
> 회사 보안 정책으로 `curl` 설치가 차단된 환경에서도 사용 가능합니다.

## 개요

Qwen2.5-1.5B-Instruct 모델을 vLLM으로 로컬 서빙하고, OpenCode CLI 에이전트에서 해당 모델을 사용하여 tool call 기반 코딩 에이전트를 구성한다. 이후 OpenCode 플러그인(oh-my-opencode)까지 설치하여 동작을 검증한다.

## 사전 요구사항

- Linux (Ubuntu 등)
- NVIDIA GPU (VRAM 8GB 이상 권장)
- Python 3.10+ & pip
- Node.js 18+ & npm
- CUDA 드라이버 설치 완료

---

## Phase 1: Qwen2.5-1.5B-Instruct 모델 다운로드 및 vLLM 서빙

### 1-1. 모델 다운로드

```bash
pip install huggingface_hub

huggingface-cli download Qwen/Qwen2.5-1.5B-Instruct --local-dir ~/Qwen2.5-1.5B-Instruct
```

> 모델 ID가 `Qwen/Qwen2-1.5B-Instruct`가 아닌 **`Qwen/Qwen2.5-1.5B-Instruct`** 인지 확인할 것.

다운로드 확인:
```bash
ls ~/Qwen2.5-1.5B-Instruct/
# config.json  model.safetensors  tokenizer.json  등이 있어야 함
```

### 1-2. vLLM 설치 및 서빙 (tool call 활성화)

```bash
pip install vllm
```

```bash
vllm serve ~/Qwen2.5-1.5B-Instruct \
  --host 0.0.0.0 \
  --port 8000 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --max-model-len 32768 \
  --dtype auto
```

**핵심 옵션 설명:**

| 옵션 | 설명 |
|---|---|
| `--enable-auto-tool-choice` | 모델이 tool call을 자동으로 선택할 수 있도록 활성화 **(필수)** |
| `--tool-call-parser hermes` | Qwen2.5 계열에 적합한 tool call 파서 **(필수)** |
| `--max-model-len 32768` | 컨텍스트 길이. Qwen2.5-1.5B의 최대값 (아래 주의사항 참고) |
| `--dtype auto` | GPU에 맞는 데이터 타입 자동 선택 |

> **주의: `--max-model-len`은 반드시 32768로 설정할 것!**
>
> - `4096`으로 설정하면 OpenCode 시스템 프롬프트(~12,000 토큰)가 컨텍스트를 초과하여 에러 발생
> - `32768`은 Qwen2.5-1.5B의 `max_position_embeddings` 최대값
> - `32768`보다 크게 설정하면 vLLM이 `max_model_len > max_position_embeddings` 에러로 시작 불가
> - GPU VRAM 8GB 이상이면 32768 문제없이 동작 (모델 ~3GB + KV cache)

### 1-3. 서빙 확인

```bash
python3 -c "import urllib.request, json; print(json.dumps(json.loads(urllib.request.urlopen('http://localhost:8000/v1/models').read()), indent=2))"
```

응답 예시:
```json
{
  "data": [
    {
      "id": "/home/username/Qwen2.5-1.5B-Instruct",
      "object": "model"
    }
  ]
}
```

> **이 응답의 `id` 값이 opencode.json에서 사용할 모델 ID입니다. 정확히 기억하세요.**

---

## Phase 2: OpenCode 설치 및 설정

### 2-1. OpenCode 설치 (npm 사용)

```bash
npm install -g opencode-ai@latest
```

설치 확인:
```bash
opencode --version
```

> **패키지명은 `opencode`가 아니라 `opencode-ai`입니다.**

### 2-2. opencode.json 설정

프로젝트 루트 디렉토리에 `opencode.json` 파일을 생성한다.

> **Phase 3 (OpenCode 단독 테스트) 용 설정입니다. plugin은 아직 추가하지 않습니다.**

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
        "/home/username/Qwen2.5-1.5B-Instruct": {
          "name": "Qwen2.5-1.5B (vLLM local)",
          "limit": {
            "context": 32768,
            "output": 4096
          }
        }
      }
    }
  },
  "model": "vllm_qwen_local//home/username/Qwen2.5-1.5B-Instruct"
}
```

> `/home/username/Qwen2.5-1.5B-Instruct` 부분을 Phase 1-3에서 확인한 실제 모델 ID로 교체할 것.

**설정 포인트:**

| 항목 | 설명 |
|---|---|
| `provider.vllm_qwen_local` | 커스텀 provider 이름 (자유 지정 가능) |
| `npm` | `@ai-sdk/openai-compatible` — OpenAI 호환 API 사용 |
| `name` | UI에 표시되는 provider 이름 |
| `baseURL` | vLLM 서버 주소 (`http://localhost:8000/v1`) |
| `apiKey` | vLLM 기본값은 `"EMPTY"` |
| **`limit.context`** | **모델 최대 입력 토큰 (32768)** |
| **`limit.output`** | **모델 최대 출력 토큰 (4096). 이 값이 없으면 OpenCode가 기본 32000을 요청하여 에러 발생!** |
| `model` | `"provider명/모델ID"` 형식 |

### 2-3. 설정 검증

```bash
opencode models
```

출력 하단에 `vllm_qwen_local//home/username/Qwen2.5-1.5B-Instruct`가 보이면 설정 완료.

### 흔한 설정 실수와 해결

#### 실수 1: 최상위 `"options"` 키 사용 (에러 발생)

```jsonc
// 틀림 — "options"는 최상위 키로 유효하지 않음
{
  "options": {
    "maxCompletionTokens": 2048,
    "maxTokens": 2048
  }
}
// 에러: Unrecognized key: "options"
```

> 토큰 제한은 최상위 `options`가 아니라 **models 안의 `limit`**에서 설정한다.

#### 실수 2: `limit` 설정 누락 (에러 발생)

```jsonc
// 틀림 — limit 없으면 OpenCode가 max_tokens=32000 요청
"models": {
  "/home/username/Qwen2.5-1.5B-Instruct": {
    "name": "Qwen2.5-1.5B (vLLM local)"
  }
}
// 에러: max_tokens is too large: 32000
```

> 반드시 `"limit": {"context": 32768, "output": 4096}` 추가.

#### 실수 3: 모델 경로 불일치 (모델 못 찾음)

```bash
# vLLM에서 실제 모델 ID 확인
python3 -c "import urllib.request, json; print(json.loads(urllib.request.urlopen('http://localhost:8000/v1/models').read()))"
```

> opencode.json의 `models` 키와 `model` 값이 vLLM 응답의 `id`와 **글자 하나까지 정확히 일치**해야 한다.

---

## Phase 3: OpenCode 동작 검증 (Build Agent 모드)

### 3-1. test.txt 작성 테스트

```bash
opencode run "test.txt 파일에 도레미송 가사를 써서 저장해줘. write tool을 사용해줘."
```

> **팁:** 1.5B 모델은 tool call schema를 완벽하게 이해하지 못할 때가 있다.
> `"write tool을 사용해줘"` 같은 명시적 지시를 프롬프트에 포함하면 성공률이 높아진다.

검증:
```bash
cat test.txt
```

**기대 결과:** 에이전트가 `build` 모드에서 Write tool call을 통해 `test.txt`를 생성

> tool call이 작동하지 않는 경우: vLLM 서빙 시 `--enable-auto-tool-choice`와 `--tool-call-parser hermes` 옵션을 재확인할 것.

---

## Phase 4: oh-my-opencode 플러그인 설치 및 테스트

### 4-1. oh-my-opencode 설치 (npm 사용)

```bash
npm install -g oh-my-opencode
```

설치 확인:
```bash
oh-my-opencode --version
oh-my-opencode --help
```

### 4-2. oh-my-opencode 초기 설정

```bash
oh-my-opencode install --no-tui \
  --claude=no \
  --openai=no \
  --gemini=no \
  --copilot=no \
  --skip-auth
```

> 로컬 vLLM 모델만 사용하므로 외부 provider는 모두 `no`로 설정.

### 4-3. opencode.json에 plugin 등록 (중요!)

oh-my-opencode를 사용하려면 프로젝트 `opencode.json`에 **`plugin` 항목을 추가**한다.

최종 `opencode.json` 전체:

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
        "/home/username/Qwen2.5-1.5B-Instruct": {
          "name": "Qwen2.5-1.5B (vLLM local)",
          "limit": {
            "context": 32768,
            "output": 4096
          }
        }
      }
    }
  },
  "model": "vllm_qwen_local//home/username/Qwen2.5-1.5B-Instruct"
}
```

> **`"plugin": ["oh-my-opencode@latest"]`가 없으면 oh-my-opencode가 정상 동작하지 않는다.**

> **주의:** oh-my-opencode install 시 글로벌 설정 `~/.config/opencode/opencode.json`에도 plugin이 추가된다.
> 이 글로벌 설정이 있으면 `opencode run` 실행 시에도 기본 에이전트가 `build`가 아닌 `Sisyphus`로 변경된다.
> OpenCode 단독(build agent)으로 테스트하고 싶다면 글로벌 설정에서 plugin을 제거해야 한다:
> ```bash
> # 글로벌 설정 확인
> cat ~/.config/opencode/opencode.json
>
> # plugin 제거 시 (OpenCode 단독 테스트용)
> echo '{"$schema":"https://opencode.ai/config.json"}' > ~/.config/opencode/opencode.json
> ```

### 4-4. test2.txt 작성 테스트

```bash
oh-my-opencode run "write tool을 사용해서 ./test2.txt 경로에 다음 내용을 저장해줘: 도레미파솔라시도"
```

> **팁:** oh-my-opencode의 Sisyphus 에이전트는 1.5B 모델에겐 프롬프트가 복잡하다.
> 파일 경로(`./test2.txt`)와 내용(`도레미파솔라시도`)을 **명시적으로** 지정하면 성공률이 높아진다.
> 더 큰 모델(7B+)을 사용하면 이 문제는 해결된다.

검증:
```bash
cat test2.txt
```

**기대 결과:** Sisyphus 에이전트가 Write tool call을 통해 `test2.txt`를 생성

---

## 체크리스트

- [ ] Qwen2.5-1.5B-Instruct 모델 다운로드 완료
- [ ] vLLM 설치 완료 (`pip install vllm`)
- [ ] vLLM 서빙 시작 (`--max-model-len 32768`, tool call 옵션 활성화)
- [ ] 서빙 확인 및 모델 ID 확인 (`python3 -c "..."`)
- [ ] OpenCode 설치 완료 (`npm install -g opencode-ai@latest`)
- [ ] `opencode.json` 설정 완료 (`limit` 설정 포함!)
- [ ] `opencode models`에서 로컬 모델 확인
- [ ] OpenCode build agent 모드에서 `test.txt` 생성 성공
- [ ] oh-my-opencode 설치 완료 (`npm install -g oh-my-opencode`)
- [ ] oh-my-opencode 초기 설정 완료 (`oh-my-opencode install --no-tui ...`)
- [ ] `opencode.json`에 `"plugin": ["oh-my-opencode@latest"]` 추가
- [ ] oh-my-opencode 환경에서 `test2.txt` 생성 성공

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| `Unrecognized key: "options"` | opencode.json 최상위에 `options` 키 사용 | `options` 키 제거, 토큰 제한은 models 안의 `limit`에서 설정 |
| `max_tokens is too large: 32000` | models에 `limit.output` 미설정 | `"limit": {"context": 32768, "output": 4096}` 추가 |
| `input tokens exceed max context` (4096) | vLLM `--max-model-len`이 4096으로 너무 작음 | `--max-model-len 32768`로 vLLM 재시작 |
| `max_model_len > max_position_embeddings` | vLLM 컨텍스트가 모델 한계(32768) 초과 | `--max-model-len`을 32768 이하로 설정 |
| tool call이 발생하지 않음 | vLLM 옵션 누락 | `--enable-auto-tool-choice --tool-call-parser hermes` 확인 |
| 모델을 찾을 수 없음 | opencode.json 모델 경로와 vLLM 모델 ID 불일치 | vLLM `/v1/models` API로 정확한 ID 확인 후 일치시킴 |
| bash/write tool invalid arguments | 1.5B 모델의 tool schema 이해 부족 | 프롬프트에 `"write tool을 사용해줘"` 명시 또는 더 큰 모델(7B+) 사용 |
| `opencode run` 시 Sisyphus가 기본 에이전트 | 글로벌 설정에 oh-my-opencode plugin이 등록됨 | `~/.config/opencode/opencode.json`에서 plugin 배열 제거 |
| oh-my-opencode에서 파일 미저장 | plugin 미등록, limit 미설정, 또는 프롬프트가 모호함 | plugin 추가, limit 설정 확인, 파일경로+내용을 명시적으로 지정 |
| 연결 거부 | vLLM 서버가 실행 중이지 않음 | vLLM 프로세스 확인 후 재시작 |

---

## 설치 순서 요약 (curl 없이 pip/npm만 사용)

```
1.  모델 다운로드         pip install huggingface_hub && huggingface-cli download ...
2.  vLLM 설치            pip install vllm
3.  vLLM 서빙            vllm serve ... --max-model-len 32768 --enable-auto-tool-choice --tool-call-parser hermes
4.  서빙 확인            python3 -c "import urllib.request, json; ..." → 모델 ID 확인
5.  OpenCode 설치        npm install -g opencode-ai@latest
6.  opencode.json 작성   provider + models(limit 포함) + model 설정 (plugin은 아직 추가하지 않음)
7.  설정 검증            opencode models
8.  OpenCode 테스트      opencode run "..." → test.txt 생성 확인
9.  oh-my-opencode 설치  npm install -g oh-my-opencode
10. oh-my-opencode 설정  oh-my-opencode install --no-tui --claude=no --openai=no --gemini=no --copilot=no --skip-auth
11. plugin 등록          opencode.json에 "plugin": ["oh-my-opencode@latest"] 추가
12. oh-my-opencode 테스트 oh-my-opencode run "..." → test2.txt 생성 확인
```
