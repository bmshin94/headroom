# 🎀 Headroom 전수조사 & 활용 전략 정리

> 작성일: 2026-09-28
> 작성: 카리나 (Claude Code)
> 대상 저장소: **https://github.com/bmshin94/headroom**
> 원본(upstream): **https://github.com/headroomlabs-ai/headroom**

---

## 📑 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [쉽게 이해하는 Headroom](#2-쉽게-이해하는-headroom)
3. [폴더 구조 전수조사](#3-폴더-구조-전수조사)
4. [작동 원리](#4-작동-원리)
5. [성능 지표](#5-성능-지표)
6. [설치 및 사용법](#6-설치-및-사용법)
7. [플러그인 vs 스킬 vs MCP](#7-플러그인-vs-스킬-vs-mcp)
8. [API 토큰 필요 여부](#8-api-토큰-필요-여부)
9. [GitHub에서 유명한 이유](#9-github에서-유명한-이유)
10. [로컬 에이전트 구축 활용법](#10-로컬-에이전트-구축-활용법)
11. [React / PHP 구현 가능성](#11-react--php-구현-가능성)
12. [수익화 아이디어 12선](#12-수익화-아이디어-12선)
13. [참고 링크 모음](#13-참고-링크-모음)

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **이름** | Headroom (헤드룸) |
| **내 저장소** | https://github.com/bmshin94/headroom |
| **원본 저장소** | https://github.com/headroomlabs-ai/headroom |
| **버전** | v0.37.0 |
| **라이선스** | Apache 2.0 (상업적 이용 허용) |
| **규모** | 파일 2,399개 / Python 218,879줄(535 파일) + Rust 79,651줄(197 파일) |
| **테스트** | 테스트 파일 887개 |
| **CI** | GitHub Actions 워크플로 22개 |
| **수상** | Trendshift "#1 Repository Of The Day" |
| **배포** | PyPI `headroom-ai` / npm `headroom-ai` / `ghcr.io/headroomlabs-ai/headroom` / HuggingFace |
| **문서** | https://docs.headroomlabs.ai/docs |
| **커뮤니티** | https://discord.gg/yRmaUNpsPJ |
| **모델** | https://huggingface.co/chopratejas/kompress-v2-base |

### 한 줄 요약

> **AI 에이전트가 읽는 모든 것(툴 출력·로그·RAG 청크·파일·대화 히스토리)을 LLM에 도달하기 전에 로컬에서 압축하는 컨텍스트 압축 레이어**

핵심 수치: **10,144 토큰 → 1,260 토큰** (동일한 `FATAL` 라인 탐지 성공)

---

## 2. 쉽게 이해하는 Headroom

### 비유 1 — 여행 가방 싸주는 친구
- 오빠의 옷 = AI에게 보낼 내용
- 캐리어 = LLM 컨텍스트 윈도우
- 짐 싸주는 친구 = **Headroom**
- "이 정장은 구기면 안 돼" = FATAL 에러/이상값 → **바이트 단위 보존**
- 집에 두고 온 옷 = CCR 캐시 (필요하면 즉시 조달)

### 비유 2 — 회의록 요약 비서
A4 50장 녹취록을 3장으로 줄이되,
- 결정사항·에러·이상값은 **한 글자도 안 빼고** 보존
- 반복 인사말·잡담은 과감히 압축
- **"원본 50장은 제 책상에 있습니다"** ← 이것이 CCR

### 비유 3 — 멀티탭처럼 중간에 끼우기

```
[이전]  Claude Code ──────────────────→ Anthropic  (55,957 토큰)
[이후]  Claude Code ──→ Headroom ─────→ Anthropic  (24,340 토큰)
                       (내 노트북)
```

Headroom이 Anthropic/OpenAI API 규격을 그대로 흉내내므로 **클라이언트는 변화를 인지하지 못함**.

### 압축 예시 — JSON (SmartCrusher)

**전:**
```json
[{"id":1,"name":"kim","status":"ok","ms":12}, ... 100개 ...]
```

**후:**
```
100개 항목 | 필드: id, name, status, ms
status: 99개 "ok", 1개 "ERROR"
ms: 평균 12, 범위 9~15

⚠️ 이상값 (원문 보존):
{"id":77,"name":"park","status":"ERROR","ms":9999}

첫 항목: {"id":1,...}  마지막 항목: {"id":100,...}
```

> 키워드 목록이 아니라 **필드별 분산 통계**로 이상값을 탐지 → `error` 단어가 없어도 특이값을 살림.

### 압축 예시 — 코드 (CodeCompressor)

AST(문법 트리) 기반이라 구조가 깨지지 않음. import / 시그니처 / 타입은 보존, 본문만 압축.
지원 언어: **Python, JS/TS, Go, Rust, Java, C/C++, Perl**

### 가장 영리한 설계 — "냉동실 vs 도마" (Live-zone 압축)

```
[냉동실 = 얼어붙은 앞부분]  → 절대 안 건드림 (바이트 동일)
[도마   = 방금 들어온 내용]  → 여기만 압축
```

프롬프트 캐시(최대 90% 할인)가 깨지지 않도록 **프리픽스를 바이트 단위로 보존**.
`CacheAligner`는 캐시를 깨뜨릴 휘발성 콘텐츠를 **탐지해서 경고만 하고, 프롬프트를 절대 재작성하지 않음**.

---

## 3. 폴더 구조 전수조사

### `headroom/` — Python 본체 (535 파일)

| 폴더 | 역할 |
|---|---|
| **`transforms/`** ⭐ | 압축 엔진 총집합 (40개 모듈) — `smart_crusher.py`(JSON), `code_compressor.py`(AST), `log_compressor.py`, `diff_compressor.py`, `cache_aligner.py`, `content_router.py`, `kompress_compressor.py`, `text_crusher.py`, `thinking_compactor.py`, `cross_turn_dedup.py` 등 |
| **`proxy/`** ⭐ | 로컬 HTTP 프록시 (OpenAI/Anthropic 호환). 인증·CCR 정책·메모리 핸들러·이미지 압축 결정 등 |
| **`ccr/`** ⭐ | Compress-Cache-Retrieve — 원본 로컬 보관 + 온디맨드 복원. MCP 서버 포함 |
| **`memory/`** | 크로스 에이전트 메모리. SQLite + HNSW 벡터스토어, 자동 dedup, 프로젝트별 격리 |
| **`providers/`** | 에이전트/제공자 어댑터 25종 (claude, codex, cursor, copilot, grok, kimi, aider, opencode, openclaw, omp, zcode, cortex_code, mistral_vibe, gemini, vertex, anthropic, openai …) |
| **`integrations/`** | LangChain, LiteLLM, Agno, Strands, CrewAI, AutoGen, ASGI |
| **`cli/`** | CLI 명령 27개 (`wrap`, `proxy`, `doctor`, `learn`, `savings`, `mcp`, `install`, `update`, `evals` …) |
| **`learn/`** | 실패 세션 마이닝 → `CLAUDE.local.md` 자동 교훈 작성 |
| **`evals/`** | GSM8K / TruthfulQA / SQuAD v2 / BFCL 정확도 평가 |
| **`dashboard/`** | 실시간 절감 대시보드 |
| **`image/`** | 이미지 압축 (ML 라우터, 40~90% 감소) |
| **`observability/`** | 메트릭·Prometheus |
| **`tokenizers/`, `pricing/`** | 토크나이저 + 모델별 단가 계산 |
| **`audit/`, `capture/`, `telemetry/`** | 감사 로그, 트래픽 캡처, 익명 비콘 |
| **`subscription/`, `copilot_auth.py`** | Copilot 구독 모드 OAuth (macOS Keychain / Linux Secret Service / Windows) |

### `crates/` — Rust 코어 (197 파일)

| 크레이트 | 역할 |
|---|---|
| `headroom-proxy` | Rust 프록시 — SSE 스트리밍(Anthropic/OpenAI Chat/Responses), Bedrock SigV4 + EventStream, Vertex AI ADC, 캐시 안정화(`cache_stabilization/`), 관측성 |
| `headroom-core` | 압축 코어 |
| `headroom-py` | PyO3 바인딩 (Python ↔ Rust) |
| `headroom-parity` | Python/Rust **동작 일치 검증** |
| `headroom-simulators` | 시뮬레이터 |

### `plugins/` — 플러그인 4종

| 플러그인 | 설명 |
|---|---|
| **`headroom-agent-hooks`** ⭐ | **Claude Code / Copilot CLI 플러그인**. `SessionStart`(startup\|resume) + `PreToolUse`(Bash\|PowerShell) 훅에서 `headroom init hook ensure` 실행 |
| `opencode` | OpenCode용 TypeScript 플러그인 |
| `openclaw` | OpenClaw ContextEngine 플러그인 |
| `headroom-oauth2` | OAuth2 인증 플러그인 (provider + middleware) |
| `hermes` | `headroom_retrieve` 플러그인 |

### 기타 디렉터리

| 경로 | 내용 |
|---|---|
| `sdk/typescript/` | TS SDK — `compress()`, OpenAI/Anthropic/Gemini/Vercel AI 어댑터, hooks, SharedContext |
| `docs/` | Next.js 기반 문서 사이트 (`docs.headroomlabs.ai`) + `llms.txt` / `llms-full.txt` 자동 생성 |
| `wiki/` | 마크다운 문서 40여 개 (architecture, benchmarks, limitations, proxy, memory, ccr …) |
| `benchmarks/` | 벤치마크 30종 (`index_proof_table.py`, `bench_latency.py`, `adversarial_ccr_tests.py`, `i18n_compression_eval.py` …) |
| `tests/` | 테스트 887개 |
| `e2e/` | E2E 테스트 (init / wrap / docker-native / copilot live) |
| **`REALIGNMENT/`** 💎 | Python→Rust 마이그레이션 로드맵 Phase A~I + 의사결정 기록 |
| `.github/workflows/` | CI 22개 (ci, rust, docker, security, release-please, eval, e2e …) |
| `sbom/` | SBOM(SPDX/CycloneDX) + 취약점 스캔 + 라이선스 목록 |
| `deploy/beacon/` | Cloudflare Worker 텔레메트리 수집기 |
| `sql/` | 대시보드/텔레메트리 스키마 |
| `scripts/` | 빌드·릴리스·검증 스크립트 |
| `examples/` | LangChain / Strands / MCP / Vertex 데모, Jupyter 노트북, Grafana |
| `.claude-plugin/marketplace.json` | Claude Code 플러그인 마켓플레이스 매니페스트 |
| `server.json` | MCP 레지스트리 표준 매니페스트 |

---

## 4. 작동 원리

```
 에이전트 / 앱 (Claude Code, Cursor, Codex, LangChain, 자체 코드…)
      │  프롬프트 · 툴 출력 · 로그 · RAG 결과 · 파일
      ▼
 ┌────────────────────────────────────────────────────┐
 │  Headroom   (로컬 실행 — 데이터가 나가지 않음)      │
 │  ────────────────────────────────────────────────  │
 │  CacheAligner  →  ContentRouter  →  CCR            │
 │                    ├─ SmartCrusher   (JSON)        │
 │                    ├─ CodeCompressor (AST)         │
 │                    └─ Kompress-v2-base (text, HF)  │
 │                                                    │
 │  Cross-agent memory  ·  headroom learn  ·  MCP     │
 └────────────────────────────────────────────────────┘
      │  압축된 프롬프트 + 원본 조회 도구
      ▼
 LLM 제공자 (Anthropic · OpenAI · Bedrock · Vertex …)
```

### 파이프라인 라이프사이클 (11단계)

`Setup` → `Pre-Start` → `Post-Start` → `Input Received` → `Input Cached` →
`Input Routed` → `Input Compressed` → `Input Remembered` → `Pre-Send` →
`Post-Send` → `Response Received`

### 핵심 원칙 3가지

1. **로컬 전용** — 압축이 내 기기에서 일어나며 프롬프트·파일 내용은 외부로 전송되지 않음
2. **가역성(CCR)** — 원본을 로컬 캐시에 보관, 모델이 `headroom_retrieve`로 복원 가능
3. **캐시 보존(Live-zone)** — 새 바이트만 압축, 프리픽스는 바이트 동일 → 프롬프트 캐시 유지

### 출력 토큰 절감 (별도 기능)

Opus급 모델은 **출력이 입력의 5배 비쌈**. Headroom은 프록시에서:
- **Verbosity steering** — "간결하게, 컨텍스트 재진술 금지" 지시를 **시스템 프롬프트 맨 끝**에 추가 (프리픽스 캐시 보존)
- **Effort routing** — 툴 결과 이어받는 단순 턴은 thinking 예산 하향, 새 질문/에러는 풀 파워
  - OpenAI: `reasoning_effort` / Anthropic: `thinking.budget_tokens`, `output_config.effort`

```bash
export HEADROOM_OUTPUT_SHAPER=1   # 기본 off
headroom proxy --port 8787
headroom output-savings
# Reduction: 31.7%  (95% CI 27.7% … 35.7%)   [estimated]
```

측정값(estimated 아닌 measured)을 원하면 `export HEADROOM_OUTPUT_HOLDOUT=0.1`

---

## 5. 성능 지표

### 토큰 절감 (재현 가능: `uv run python benchmarks/index_proof_table.py --seed 20260902`)

| 시나리오 | Before | After | 절감 |
|---|---:|---:|---:|
| 코드 검색 (결과 100개) | 17,199 | 13,597 | **21%** |
| SRE 장애 디버깅 | 55,957 | 24,340 | **57%** |
| 코드베이스 탐색 | 58,801 | 33,895 | **42%** |
| GitHub 이슈 분류 | 46,067 | 32,429 | **30%** |

반복적인 JSON 배열/로그 라인은 `bench_latency.py`에서 90% 이상도 나옴. 산문·고밀도 출력은 거의 안 줄어듦.

### 지연시간

| 입력 크기 | 압축 시간 |
|---|---|
| 10K 토큰 JSON | **0.21 ms (p50)** |
| 100K 토큰 | **1.4 ms** |

→ 에이전트 체감 지연에 나타나지 않는 수준.

### 정확도 (`python -m headroom.evals suite --tier 1`)

| 벤치마크 | 카테고리 | N | Baseline | Headroom | Delta |
|---|---|---:|---:|---:|---|
| GSM8K | Math | 100 | 0.870 | 0.870 | ±0.000 |
| TruthfulQA | Factual | 100 | 0.530 | 0.560 | +0.030 |
| SQuAD v2 | QA | 100 | — | 97% | 19% 압축에서 |
| BFCL | Tools | 100 | — | 97% | 32% 압축에서 |

> N=100에서 ±0.03은 신뢰구간 내 → TruthfulQA는 "향상"이 아니라 "차이 없음"으로 해석.

---

## 6. 설치 및 사용법

### 설치

```bash
# 추천: uv (격리 환경)
uv tool install --python 3.13 "headroom-ai[all]"

# pip
pip install "headroom-ai[all]"
pip install "headroom-ai[proxy]"   # 프록시만
pip install "headroom-ai[mcp]"     # MCP만

# TypeScript (라이브러리만! CLI 없음)
npm install headroom-ai

# Docker
docker pull ghcr.io/headroomlabs-ai/headroom:latest
```

> ⚠️ `headroom` **CLI는 PyPI 패키지에만** 존재. npm은 TS SDK 라이브러리.
> 요구사항: **Python 3.10+** (권장 3.13)

**Extras**: `[proxy]` `[mcp]` `[ml]` `[code]` `[memory]` `[vector]` `[relevance]` `[image]` `[agno]` `[langchain]` `[strands]` `[anyllm]` `[bedrock]` `[evals]` `[pytorch-mps]`

> `[all]`은 코어 스택만 포함. 프레임워크 어댑터는 별도 설치 필요.

### 모드 A — 에이전트 래핑 (가장 쉬움) ⭐

```bash
headroom wrap claude          # Claude Code
headroom wrap codex
headroom wrap cursor          # 설정용 base URL 출력
headroom wrap copilot
headroom wrap vscode          # VS Code Copilot
headroom wrap vscode-claude   # VS Code Claude Code 확장
headroom unwrap claude        # 되돌리기
```

동작: ① 로컬 프록시 시작 ② **Serena**(시맨틱 코드 탐색 MCP) 설치 — `--code-memory none`으로 생략 가능 ③ 에이전트를 프록시 경유로 설정 후 실행

**지원 에이전트 18종**: Claude Code, Codex, Grok CLI, Cursor, Aider, Copilot CLI, VS Code Copilot, OpenClaw, OpenCode, Cline, Continue, Goose, OpenHands, Mistral Vibe, Oh My Pi, Cortex Code, Kimi CLI, ZCode

### 모드 B — 프록시 (코드 수정 0줄)

```bash
headroom proxy --port 8787
curl http://localhost:8787/health

ANTHROPIC_BASE_URL=http://localhost:8787 claude
OPENAI_BASE_URL=http://localhost:8787/v1 python my_app.py
```

### 모드 C — 라이브러리

```python
from headroom import compress
from openai import OpenAI

result = compress(messages, model="gpt-4o")
client = OpenAI()
resp = client.chat.completions.create(model="gpt-4o", messages=result.messages)
print(f"Saved {result.tokens_saved} tokens ({result.compression_ratio:.0%})")
```

```typescript
import { compress } from 'headroom-ai';
const result = await compress(messages, { model: 'gpt-4o' });
```

### 모드 D — MCP 서버

```bash
headroom mcp install   # Claude Code에 자동 등록
claude
```

도구 3종: `headroom_compress`, `headroom_retrieve`, `headroom_stats`

MCP 클라이언트가 PATH를 상속 못 하면 절대경로 지정:
```toml
[mcp_servers.headroom]
command = "/Users/you/.local/bin/headroom"
args = ["mcp", "serve"]
```

### 운영 명령어

```bash
headroom doctor           # 건강검진 (라우팅 확인)
headroom perf             # 성능 측정
headroom dashboard        # 실시간 절감 대시보드
headroom savings          # 실제 트래픽 기준 절감액
headroom output-savings   # 출력 토큰 절감액
headroom learn            # 실패 세션 분석 (dry run)
headroom learn --apply    # 적용
headroom learn --verbosity --apply  # 간결도 자동 학습
headroom update           # 업데이트 (pip/pipx/uv 자동 감지)
headroom deploy           # 턴키 로컬 배포
headroom install apply --preset persistent-service --providers auto
```

### 알려진 설치 이슈

| 증상 | 해결 |
|---|---|
| `CERTIFICATE_VERIFY_FAILED` (사내 SSL 검사) | Rust 선설치 or `pip install --only-binary headroom-ai headroom-ai` |
| `Basic Constraints of CA cert not marked critical` | `HEADROOM_TLS_STRICT=0` |
| Intel macOS ONNX 빌드 실패 (#941) | `brew install onnxruntime` + `ORT_STRATEGY=system` + `ORT_PREFER_DYNAMIC_LINK=1` |
| AVX2 없는 x86 | 자동 폴백 (BM25 relevance, 휴리스틱 탐지) |

---

## 7. 플러그인 vs 스킬 vs MCP

### 결론: **셋 다 있지만, 본질은 "로컬 프록시 + 라이브러리"**

| 형태 | 제공 | 설명 |
|---|:---:|---|
| **로컬 프록시** | ✅ | 🌟 **본체** — `headroom proxy` |
| **라이브러리(SDK)** | ✅ | Python `compress()`, TS `compress()` |
| **MCP 서버** | ✅ | `headroom mcp serve` — 도구 3종 |
| **Claude Code 플러그인** | ✅ | `plugins/headroom-agent-hooks` (SessionStart / PreToolUse 훅) |
| **Claude Skill** | ❌ | **스킬 아님** |

```
        ┌─────────────────────────────┐
        │   압축 엔진 (진짜 본체)      │
        │  transforms/ + ccr/ + Rust  │
        └─────────────────────────────┘
          ↑     ↑      ↑      ↑     ↑
       프록시  라이브러리  MCP  플러그인  CLI
```

| 모드 | 커버 범위 | 추천 상황 |
|---|---|---|
| 프록시 | 🟢 모든 트래픽 자동 | **대부분 이것** |
| MCP | 🟡 모델이 필요할 때만 수동 호출 | 프록시 불가 환경 |
| 플러그인(훅) | 🟡 런타임 준비만 담당 | 프록시 보조 |
| 라이브러리 | 🟢 직접 제어 | 자체 앱 내장 |

> MCP는 **선택적 보조**. 완전 자동 압축을 원하면 **프록시**를 써야 함.

---

## 8. API 토큰 필요 여부

### 결론: **Headroom 자체는 토큰/가입 불필요. 완전 무료 오픈소스.**

| 토큰 | 필요? | 언제 |
|---|:---:|---|
| Headroom 계정/API 키 | ❌ | 존재하지 않음 (로컬 전용) |
| 내 LLM API 키 (`ANTHROPIC_API_KEY` 등) | ✅ | 원래 쓰던 것 그대로. 프록시가 그대로 전달 |
| `HEADROOM_PROXY_TOKEN` | ⚠️ | 프록시를 `0.0.0.0`으로 **외부 노출할 때만**. 로컬(127.0.0.1)은 불필요. 생성: `openssl rand -hex 32` |
| GitHub OAuth | ⚠️ | Copilot 구독 모드 (`headroom copilot-auth login`) |
| HuggingFace 토큰 | ❌ | 모델이 공개 |
| `NEO4J_AUTH` / `NEO4J_PASSWORD` | ⚠️ | 그래프 메모리 백엔드 사용 시 |

### 보안 비교

```
❌ 경쟁 서비스: 내 코드 → 그들 API → 압축 → 반환   (외부 전송)
✅ Headroom:   내 코드 → 내 기기에서 압축 → LLM     (로컬)
```

### ⚠️ 텔레메트리 (주의)

익명 비콘이 **기본 ON** (Cloudflare Worker 수집).
- 보내는 것: 압축률, 카운터, provider/model ID, OS, 아키텍처
- **보내지 않는 것: 프롬프트, 응답, 코드, 파일 경로**

끄는 법:
```bash
export HEADROOM_BEACON=off
export DO_NOT_TRACK=1
headroom proxy --offline
```

> 사내/기업 환경이라면 **도입 전 반드시 끄기** 권장.
> (참고: `llms.txt`에는 "opt-in, 기본 off"로, README에는 "기본 on"으로 기술되어 있어 문서 간 불일치가 있음. 실제로는 README 기준으로 **기본 on**으로 보고 명시적으로 끄는 것이 안전.)

---

## 9. GitHub에서 유명한 이유

1. **가장 아픈 곳을 찌름** — "Claude Code 토큰값 폭탄"이라는 2025~2026 개발자 공통 고민 직격
2. **진입장벽 제로** — `headroom wrap claude` 한 줄. README 60초 안에 결과
3. **"그냥 자르는 거 아냐?" 방어 완벽** — 재현 가능 벤치마크(시드 고정·오프라인), 정확도 eval 공개, CCR 가역성, `LIMITATIONS.md` 당당 공개
4. **로컬 우선 = 보안팀 승인 가능** — 기업 도입 검토 시 결정적
5. **호환성** — 에이전트 18종, 프레임워크 10종+, 제공자 5종+ → "내가 쓰는 거 지원함" 확률 거의 100%
6. **엔지니어링 퀄리티** — 테스트 887개, CI 22개, SBOM + 취약점 스캔 + GitGuardian + Gitleaks + Dependabot, Rust 재작성 로드맵 공개, Python↔Rust parity 검증 크레이트
7. **Apache 2.0 + 상업적 이용 허용** — 유료는 "팀 배포"로 분리, OSS는 기능 제한 없음
8. **마케팅** — GIF 데모 3개, `llms.txt` 제공(AI가 읽고 추천), Discord, PyPI/npm/Docker/HF 4중 배포, Trendshift #1

---

## 10. 로컬 에이전트 구축 활용법

### 차원 1 — 그대로 갖다 쓰기

```python
from headroom import compress

tool_result = run_tool(...)          # 5만 토큰
compressed = compress([{"role": "tool", "content": tool_result}], model="...")
messages.append(compressed.messages[0])   # 2만 토큰
```

→ 컨텍스트 **2~3배 오래 버팀**

### 차원 2 — 아키텍처 교과서 ⭐

| 필요한 것 | 참고 위치 |
|---|---|
| LLM 프록시 구현 | `headroom/proxy/` + `crates/headroom-proxy/` |
| SSE 스트리밍 중계 | `crates/headroom-proxy/src/sse/` (anthropic / openai_chat / openai_responses / framing) |
| MCP 서버 구현 | `headroom/ccr/mcp_server.py`, `headroom/memory/mcp_server.py`, `server.json` |
| Claude Code 훅/플러그인 | `plugins/headroom-agent-hooks/hooks/hooks.json` |
| 에이전트별 설정 주입 | `headroom/providers/` (18종 실사례) |
| 로컬 벡터 메모리 | `headroom/memory/` (SQLite + HNSW, 프로젝트 격리) |
| 에이전트 간 컨텍스트 공유 | `headroom/shared_context.py` |
| 프롬프트 캐시 최적화 | `crates/headroom-proxy/src/cache_stabilization/` (volatile_detector, drift_detector, tool_def_normalize, beta_sticky) |
| 토크나이저 / 비용 계산 | `headroom/tokenizer.py`, `headroom/pricing/` |
| Bedrock SigV4 / Vertex ADC | `crates/headroom-proxy/src/bedrock/`, `vertex/` |
| 에이전트 평가 | `headroom/evals/`, `benchmarks/` |
| 파이프라인 라이프사이클 설계 | `headroom/pipeline.py` (11단계) |
| Python + Rust 하이브리드 | `crates/headroom-py` (PyO3), `headroom-parity` |
| 실패 학습 시스템 | `headroom/learn/` |

> 💎 **`REALIGNMENT/` 폴더가 보물** — Phase A~I로 Python→Rust 이전 과정과 설계 의사결정이 문서화되어 있음.

### 차원 3 — 인프라로 깔기

```
┌──────────────────────────────────┐
│  내 로컬 에이전트 (React + API)  │
└────────────┬─────────────────────┘
       ┌─────▼──────┐
       │ Headroom   │ ← 압축 + 메모리 + CCR + 토큰계산
       └─────┬──────┘
       ┌─────▼──────┐
       │ Claude/GPT │
       └────────────┘
```

### 주의할 점

- 규모가 큼(2,399 파일) → **필요한 부분만** 보기
- Python↔Rust 이중 구조라 초기 학습 곡선 있음
- 의존성 무거움 (`[all]` = ONNX, tree-sitter, HNSW …)
- x86 ONNX 기능은 **AVX2 CPU** 필요 (없으면 자동 폴백)
- 짧은 대화/고밀도 산문엔 효과 미미

---

## 11. React / PHP 구현 가능성

### 해석 A — "Headroom을 React/PHP로 재구현?"

#### React(JS/TS) → 부분적으로 가능 🟡

| 기능 | 가능? | 비고 |
|---|:---:|---|
| SmartCrusher (JSON) | ✅ | 순수 로직 |
| 로그/검색결과 압축 | ✅ | 정규식·통계 |
| CCR 캐시 | ✅ | SQLite / IndexedDB |
| 프록시 | ✅ | Node/Express/Fastify |
| MCP 서버 | ✅ | 공식 TS SDK 존재 |
| CodeCompressor (AST) | 🟡 | `web-tree-sitter`(WASM) 필요 |
| Kompress-v2 ML | 🟡 | `onnxruntime-node` 필요, 느림 |
| 정확한 토크나이저 | 🟡 | `tiktoken` WASM 가능하나 무거움 |
| 브라우저 React 직접 | ❌ | API 키 노출 위험 → 서버 필수 |

> 참고: 공식 TS SDK(`sdk/typescript/`)는 이미 존재하지만, **압축 자체는 로컬 Headroom에 위임**하는 클라이언트임.

#### PHP → 가능하지만 비효율 🟠

| 기능 | 가능? |
|---|:---:|
| JSON 통계 압축 | ✅ |
| 로그/텍스트 압축 | ✅ |
| CCR 캐시 (MySQL/Redis) | ✅ |
| 프록시 | 🟡 SSE 스트리밍이 고통스러움 |
| AST 압축 | ❌ tree-sitter PHP 바인딩 빈약 |
| ML 모델 | ❌ 사실상 불가 |

### 해석 B — "React/PHP 앱에 Headroom을 붙이기?" → **100% 가능** 👑

OpenAI/Anthropic 호환 HTTP 프록시이므로 **언어 무관**.

#### React + Node 백엔드
```javascript
import OpenAI from 'openai';

const client = new OpenAI({
  baseURL: 'http://127.0.0.1:8787/v1',   // ← 이 한 줄
  apiKey: process.env.OPENAI_API_KEY,
});
```

#### PHP (Laravel)
```php
$response = Http::withHeaders([
        'x-api-key'         => env('ANTHROPIC_API_KEY'),
        'anthropic-version' => '2023-06-01',
        'content-type'      => 'application/json',
    ])
    ->post('http://127.0.0.1:8787/v1/messages', [   // ← 여기만 교체
        'model'      => 'claude-sonnet-4-5-20250929',
        'max_tokens' => 4096,
        'messages'   => $messages,
    ]);
```

### 권장 아키텍처

```
┌────────────────────────────────────────┐
│  React 프론트엔드                       │
│   · 채팅 UI                             │
│   · 토큰 절감 대시보드                   │
│   · 압축 Before/After 뷰어              │
└──────────────┬─────────────────────────┘
               │ REST / SSE
┌──────────────▼─────────────────────────┐
│  PHP(Laravel) or Node 백엔드            │
│   · 인증 / 사용자 / 과금                 │
│   · API 키 보관                         │
└──────────────┬─────────────────────────┘
               │ baseURL 교체 한 줄
┌──────────────▼─────────────────────────┐
│  Headroom 프록시 (Docker)               │
│   · 압축 · CCR · 메모리 · 절감 측정      │
└──────────────┬─────────────────────────┘
       ┌───────▼────────┐
       │  Claude / GPT  │
       └────────────────┘
```

> **핵심: 압축 엔진을 재개발하지 말 것.** Docker로 띄우고 React/PHP는 UI·비즈니스 로직에 집중하는 것이 훨씬 빠르고 안정적.

---

## 12. 수익화 아이디어 12선

### ⚖️ 법적 체크 (Apache 2.0)

상업적 이용은 **완전 허용**. 단:

| 항목 | 내용 |
|---|---|
| LICENSE 사본 포함 | 배포물에 Apache 2.0 전문 포함 |
| NOTICE 파일 유지 | 레포의 `NOTICE` 그대로 |
| 변경사항 명시 | 수정 시 "수정함" 표기 |
| **상표 주의** ⚠️ | **"Headroom" 이름/로고는 Apache 2.0 §6 상표 조항에서 제외** → 자체 서비스는 **다른 이름** + "Powered by Headroom" 표기 |
| 특허 보복 조항 | 특허 소송 제기 시 라이선스 종료 |

---

### 🥇 TIER 1 — 즉시 가능 (난이도 ⭐)

#### 1. AI 토큰 비용 절감 컨설팅 🇰🇷
- **초기비용** 거의 0원 / **타겟** Claude Code·Cursor 쓰는 국내 팀
- **수익** 진단 100~300만원 + 절감액의 20~30% 성과급 + 월간 리포트 구독 50만원
- **실행** `headroom savings`로 고객사 트래픽 분석 → Before/After 리포트 → 세팅 + 교육
- **근거** 한글 문서 부재. 설치 대행만으로도 가치

#### 2. 한국어 압축 최적화 패키지
- **수익** 유료 플러그인 월 2~5만원/인
- **근거** Kompress-v2는 영어 트레이스 학습 → 한국어 효율 낮을 가능성. 한글은 토큰 소모도 큼
- **도구** 레포에 `benchmarks/i18n_compression_eval.py` 이미 존재 → 측정 가능
- **차별점** 원본 팀이 절대 안 할 영역

#### 3. 한글 강의 / 전자책 / 유료 커뮤니티
- **수익** 강의 5~15만원, 전자책 3만원, 멤버십 월 2만원
- **주제** "Claude Code 토큰값 절반 줄이기" / "LLM 프록시 직접 만들기" / "MCP 서버 개발" / "로컬 AI 에이전트 A to Z"
- **부가효과** 브랜딩 → 컨설팅 리드 확보

---

### 🥈 TIER 2 — 제품화 (난이도 ⭐⭐⭐)

#### 4. React 기반 팀 대시보드 SaaS ⭐ 최우선 추천
- **수익** 팀당 월 $29~199 (좌석 과금)
- **스택** React + Node/PHP + Headroom Docker
- **OSS에 없는 기능**: 팀 전체 사용량 집계 / 개발자·프로젝트·모델별 비용 분해 / 절감액 위젯 / 예산 초과 알림(Slack) / 압축 Before-After 뷰어 / 주간·월간 PDF 리포트 / SSO(Google·MS·SAML) / RBAC
- **참고** `headroom/dashboard/`, `sql/` 폴더에 스키마 존재

#### 5. 워드프레스 / PHP 생태계 플러그인 🐘
- **수익** 플러그인 $49~199 + 연간 갱신
- **제품** "AI Token Saver for WordPress" — 기존 AI 플러그인의 엔드포인트를 Headroom으로 라우팅 + 관리자 절감액 표시. Laravel 패키지 버전도
- **근거** 워드프레스 = 전 세계 웹의 43%, PHP 진영엔 이런 도구 전무 (블루오션)

#### 6. Claude Code 플러그인 마켓 진출
- **근거** 레포에 `.claude-plugin/marketplace.json` 구조 존재. 배포 채널 확보됨
- **아이디어** 실시간 절감액 상태바(@Ship-Wright 선례 있음), 압축 품질 경고, 한국어 특화 훅, 팀 정책 강제

#### 7. AI 비용 관리(FinOps) 게이트웨이
- **수익** 엔터프라이즈 연 2,000만~1억원
- **기능** 부서별 예산 할당·차단 / 프로젝트별 비용 태깅 → 회계 연동 / 정책 엔진("Opus는 승인 필요") / 감사 로그(`headroom/audit/` 활용) / ISMS-P·ISO27001 대응 문서 / DLP 훅
- **근거** 대기업은 "절감"보다 **"통제 가능성"**에 더 지불

---

### 🥉 TIER 3 — 큰 그림 (난이도 ⭐⭐⭐⭐⭐)

#### 8. 온프렘 / 폐쇄망 어플라이언스 🏢
- **수익** 구축 5,000만~2억 + 연 유지보수 20%
- **타겟** 금융, 공공, 국방, 병원, 대기업
- **패키징** 폐쇄망 설치 패키지 / 국내 클라우드 인증(NHN·네이버·KT) / SBOM + 취약점 리포트(`sbom/` 활용) / 24-7 지원 SLA
- **근거** 금융·공공은 "코드가 밖으로 안 나감"이 계약 조건 → 로컬 우선 설계와 완벽 부합

#### 9. 버티컬 특화 압축기

| 도메인 | 압축 대상 | 고객 |
|---|---|---|
| 법률 | 판례·계약서 (구조 보존) | 로펌, 리걸테크 |
| 의료 | 진료기록·논문 (수치 절대 보존) | 병원, 헬스케어 |
| 금융 | 재무제표·공시 (숫자 무손실) | 증권사, 핀테크 |
| 제조 | 센서 로그·설비 데이터 | 스마트팩토리 |
| 이커머스 | 상품 카탈로그 JSON | 쇼핑몰 |

- **방법** `transforms/compressor_registry.py`에 커스텀 압축기 등록
- **수익** 도메인 압축기 라이선스 월 100~500만원

#### 10. "압축 품질 보증" 서비스 🔬
- **근거** 도입 최대 불안 = "정보 손실"
- **방법** `evals/`, `benchmarks/adversarial_ccr_tests.py`로 고객 실데이터 정확도 검증 리포트 + 릴리스별 회귀 테스트
- **수익** 검증 1회 500만원 + 월간 모니터링 100만원

#### 11. 멀티 에이전트 오케스트레이션 플랫폼
- `SharedContext` + 크로스 에이전트 메모리를 핵심 자산으로
- Claude(설계) → Codex(구현) → Gemini(리뷰) → Grok(문서) 파이프라인
- **수익** SaaS 월 $99~499/팀

#### 12. 듀얼 라이선스 파생 제품
- OSS 코어(Apache 2.0) + 자체 엔터프라이즈 모듈(상용)
- 원본 팀과 동일 전략이며 합법. 단 **파생 부분만** 상용화 가능

---

### 🎯 추천 로드맵

```
[1~3개월] 씨앗 뿌리기
  ├─ 한글 블로그/유튜브 콘텐츠 (무료, 브랜딩)
  ├─ 한글 설치 가이드 배포
  └─ 컨설팅 1~2건 수주 (아이디어 1)
        ↓ 리드 확보 + 시장 이해
[4~9개월] 제품화
  ├─ React 팀 대시보드 MVP (아이디어 4) ⭐
  ├─ 한국어 압축 최적화 (아이디어 2)
  └─ 강의 런칭 (아이디어 3)
        ↓ MRR 발생
[10~24개월] 확장
  ├─ FinOps 게이트웨이 → 엔터프라이즈 (아이디어 7)
  ├─ 온프렘 패키지 (아이디어 8)
  └─ 버티컬 특화 (아이디어 9)
```

### 🏆 최우선 조합

> **"한국어 특화 + React 대시보드 + 컨설팅"**

1. 원본 팀이 **절대 안 할 영역** (한국 시장)
2. 압축 엔진 **재개발 불필요** → 개발 시간 대폭 절약
3. 컨설팅으로 **현금 흐름** 확보하며 SaaS 개발
4. 핵심이 **UI 레이어(React/PHP)** → 기존 기술 스택과 부합

---

## 13. 참고 링크 모음

### 저장소
- **내 저장소**: https://github.com/bmshin94/headroom
- **원본 저장소**: https://github.com/headroomlabs-ai/headroom

### 패키지 / 배포
- PyPI: https://pypi.org/project/headroom-ai/
- npm: https://www.npmjs.com/package/headroom-ai
- Docker: `ghcr.io/headroomlabs-ai/headroom:latest`
- HuggingFace 모델: https://huggingface.co/chopratejas/kompress-v2-base

### 문서
- 문서 사이트: https://docs.headroomlabs.ai/docs
- Quickstart: https://docs.headroomlabs.ai/docs/quickstart
- Architecture: https://docs.headroomlabs.ai/docs/architecture
- CCR: https://docs.headroomlabs.ai/docs/ccr
- Proxy: https://docs.headroomlabs.ai/docs/proxy
- MCP: https://docs.headroomlabs.ai/docs/mcp
- Memory: https://docs.headroomlabs.ai/docs/memory
- Benchmarks: https://docs.headroomlabs.ai/docs/benchmarks
- Limitations: https://docs.headroomlabs.ai/docs/limitations
- llms.txt: https://docs.headroomlabs.ai/llms.txt
- llms-full.txt: https://docs.headroomlabs.ai/llms-full.txt

### 커뮤니티 / 관련 프로젝트
- Discord: https://discord.gg/yRmaUNpsPJ
- Serena (권장 동반 MCP): https://github.com/oraios/serena
- Claude Code 상태바 플러그인(커뮤니티): https://github.com/Ship-Wright/headroom-plugin
- Trendshift: https://trendshift.io/repositories/20881

### 문의
- 팀 배포/managed 문의: hello@headroomlabs.ai

---

## 📌 한 페이지 요약

| 질문 | 답 |
|---|---|
| **뭐 하는 거야?** | AI 에이전트가 읽는 모든 걸 LLM 도달 전에 로컬에서 압축 |
| **얼마나 줄어?** | 실사용 21~57%, 반복 JSON/로그는 90%+ |
| **느려지지 않아?** | 10K 토큰에 0.21ms — 체감 불가 |
| **정보 잃어?** | CCR로 원본 보관 + 온디맨드 복원. eval상 정확도 손실 거의 없음 |
| **설치는?** | `uv tool install --python 3.13 "headroom-ai[all]"` |
| **제일 쉬운 사용법?** | `headroom wrap claude` |
| **플러그인? MCP?** | 둘 다 제공하지만 **본체는 로컬 프록시**. 스킬은 아님 |
| **토큰 필요해?** | Headroom 자체는 불필요. 내 LLM API 키만 있으면 됨 |
| **보안은?** | 로컬 전용. 단 텔레메트리 기본 ON → `HEADROOM_BEACON=off` 권장 |
| **React/PHP로 쓸 수 있어?** | ✅ baseURL 한 줄 교체로 연동. 재구현은 비추천 |
| **수익화 1순위?** | 한국어 특화 + React 팀 대시보드 SaaS + 컨설팅 |
| **라이선스 주의점?** | Apache 2.0 상업 이용 OK, 단 **"Headroom" 이름은 사용 금지**(상표 조항) |
