# Headroom 전수조사 리포트 📊

> **작성일:** 2026-10-07
> **대상 레포:** [bmshin94/headroom](https://github.com/bmshin94/headroom) (fork)
> **원본 레포:** [headroomlabs-ai/headroom](https://github.com/headroomlabs-ai/headroom)
> **공식 문서:** https://docs.headroomlabs.ai
> **PyPI:** https://pypi.org/project/headroom-ai/ · **npm:** https://www.npmjs.com/package/headroom-ai
> **HF 모델:** https://huggingface.co/chopratejas/kompress-v2-base
> **Discord:** https://discord.gg/yRmaUNpsPJ
> **분석 버전:** `0.37.0` · **라이선스:** Apache 2.0

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [쉬운 설명 (비유)](#2-쉬운-설명-비유)
3. [전수조사 결과 — 내부 구조](#3-전수조사-결과--내부-구조)
4. [성능 데이터](#4-성능-데이터)
5. [설치 및 사용법](#5-설치-및-사용법)
6. [플러그인 / 스킬 / MCP — 정체는?](#6-플러그인--스킬--mcp--정체는)
7. [API 토큰이 필요한가?](#7-api-토큰이-필요한가)
8. [AI 에이전트 구축에 도움이 되는가?](#8-ai-에이전트-구축에-도움이-되는가)
9. [React / PHP로 만들 수 있는가?](#9-react--php로-만들-수-있는가)
10. [유튜브 강의 제작 가능성](#10-유튜브-강의-제작-가능성)
11. [수익화 아이디어 (상세)](#11-수익화-아이디어-상세)
12. [주의사항 및 발견된 이슈](#12-주의사항-및-발견된-이슈)
13. [참고 링크 모음](#13-참고-링크-모음)

---

## 1. 프로젝트 개요

### 한 줄 요약

> **AI 에이전트가 읽는 모든 것(툴 출력·로그·RAG 결과·파일·대화 이력)을 LLM에 도달하기 전에 로컬에서 압축해, 토큰 비용을 50~90% 줄여주는 미들웨어.**

| 항목 | 내용 |
|---|---|
| 이름 | Headroom (Headroom Labs) |
| 버전 | `0.37.0` |
| 라이선스 | Apache 2.0 (상업적 이용 가능) |
| 언어 | Python (메인) + Rust (고성능 코어) + TypeScript (SDK) |
| 배포 | PyPI `headroom-ai` / npm `headroom-ai` / `ghcr.io/headroomlabs-ai/headroom` |
| 요구사항 | Python 3.10+ (3.13 권장) |
| 개발 단계 | `Development Status :: 4 - Beta` |
| 수상 | Trendshift "Repository Of The Day #1" |

### 핵심 특징

- **로컬 실행** — 프롬프트/파일 내용이 압축을 위해 외부로 전송되지 않음
- **코드 수정 0** — 프록시 모드로 기존 에이전트를 그대로 사용
- **가역 압축(CCR)** — 원본을 로컬에 보관, 모델이 필요 시 복원 요청
- **정확도 유지** — GSM8K 압축 전/후 동일(0.870), BFCL 툴 호출 97% 유지
- **19종 에이전트 지원** — Claude Code, Codex, Cursor, Copilot, Aider 등

### 데이터 흐름

```
 에이전트 / 앱 (Claude Code · Cursor · Codex · LangChain · Agno · Strands · 자체 코드)
        │   프롬프트 · 툴 출력 · 로그 · RAG 결과 · 파일
        ▼
 ┌──────────────────────────────────────────────────┐
 │  Headroom  (로컬 실행 — 데이터가 머신을 벗어나지  │
 │             않음)                                 │
 │  CacheAligner → ContentRouter → CCR              │
 │                  ├─ SmartCrusher    (JSON)       │
 │                  ├─ CodeCompressor  (AST)        │
 │                  └─ Kompress-v2-base (텍스트/ML) │
 │                                                   │
 │  Cross-agent memory · headroom learn · MCP       │
 └──────────────────────────────────────────────────┘
        │   압축된 프롬프트 + 복원 툴
        ▼
 LLM 공급자 (Anthropic · OpenAI · Bedrock · Vertex · …)
```

---

## 2. 쉬운 설명 (비유)

### 비유 1 — 도시락 압축 서비스

무게당 요금을 내는 택배로 도시락을 보낸다고 하면:

- **압축 없음**: 밥·반찬·포장재·빈 공간·광고지 전부 → 1kg 요금
- **압축 있음**: 빈 공간 제거, 광고지 제거, 밥 압축. **단 중요한 반찬(에러 메시지)은 절대 제거 안 함** → 300g 요금
- **CCR**: 원본 도시락은 집 창고에 보관 → "원본 좀 볼 수 있나요?" 요청 시 즉시 제공

### 비유 2 — 수도관 중간에 끼우는 정수기

```
수돗물 (Claude Code) → [ 정수기 = Headroom ] → 마시는 물 (LLM)
```

수도관 공사도, 수도꼭지 교체도 필요 없음. 명령어 한 줄: `headroom wrap claude`

### 비유 3 — 영화 → 하이라이트 영상

2시간 영화(10,000줄 로그) 대신 5분 하이라이트(1,200줄)를 보내지만, 아무 장면이나 뽑는 게 아님:

| 콘텐츠 종류 | 처리 방식 | 비유 |
|---|---|---|
| JSON 데이터 | 통계로 평범한 항목 제거, 이상치 보존 | 반복 장면 스킵, 반전 장면 보존 |
| 코드 | 함수 시그니처/타입/import 보존, 본문만 요약 | 목차는 전부, 본문만 요약 |
| 일반 텍스트 | ML 모델이 압축 | AI 편집자가 편집 |
| 에러 / FATAL | **절대 건드리지 않음** | 범인 밝혀지는 장면은 풀영상 |

### 꼭 알아야 할 용어 4개

| 용어 | 쉬운 뜻 | 왜 중요한가 |
|---|---|---|
| **프록시(Proxy)** | 중간에 끼는 중개인 | 코드를 안 고치고 쓸 수 있는 이유 |
| **CCR** | 압축해도 원본은 로컬 창고에 보관 | 정보가 소실되지 않음 |
| **MCP** | AI에게 도구를 쥐어주는 표준 규격 | Claude가 직접 압축/복원 호출 가능 |
| **KV 캐시** | LLM이 앞부분을 기억해두는 메모장 | 깨지면 오히려 비용 증가 → CacheAligner가 보호 |

### CacheAligner를 가장 쉽게 설명하면

Claude 등은 **앞부분이 바이트 단위로 동일하면 할인**(prompt caching, 최대 90%)해 준다. 그래서 Headroom은:

```
[ 앞부분(frozen prefix) — 절대 변경 안 함 ]  ← 캐시 할인 유지
[ 뒷부분(live zone)     — 새 바이트만 압축 ]  ← 추가 절감
```

**압축과 캐시 할인을 동시에 챙기는 설계.** 이것이 가장 영리한 부분이다.

---

## 3. 전수조사 결과 — 내부 구조

### 3.1 규모

| 지표 | 수치 |
|---|---:|
| Python 파일 | 1,519 |
| Rust 파일 | 197 |
| TypeScript 파일 | 106 |
| Markdown 문서 | 93 |
| GitHub Actions 워크플로우 | 23 |

### 3.2 폴더별 역할

| 폴더 | 파일수 | 역할 |
|---|---:|---|
| `headroom/` | 545 | Python 본체 — 모든 핵심 로직 |
| `crates/` | 210 | Rust 코어 — 성능 critical 경로 |
| `tests/` | 1,123 | 테스트 (본체보다 많음) |
| `docs/` | 95 | Next.js 기반 문서 사이트 |
| `plugins/` | 60 | 에이전트별 플러그인 |
| `sdk/typescript/` | 61 | TS/JS SDK |
| `wiki/` | 42 | 아키텍처·벤치마크 문서 |
| `benchmarks/` | 34 | 성능 측정 스크립트 31종 |
| `scripts/` | 38 | 빌드·릴리즈 자동화 |
| `examples/` | 31 | LangChain · Strands · MCP 데모 |
| `sbom/` | 9 | SBOM(CycloneDX/SPDX) + 취약점 스캔 |
| `deploy/beacon/` | 8 | Cloudflare Worker 텔레메트리 수집기 |
| `sql/` | 5 | 텔레메트리 대시보드 스키마 |
| `e2e/` | 18 | 엔드투엔드 테스트 |

### 3.3 압축 엔진 4종 (`headroom/transforms/`, 35개 모듈)

**1. SmartCrusher** — `smart_crusher.py`
- JSON 전문. 배열/중첩 객체/혼합 타입 처리
- 키워드 매칭이 아닌 **필드 분산(variance) 통계**로 판단
- 에러 항목, 통계적 이상치, 첫/마지막 경계값은 무조건 보존
- JSON 툴 출력에서 70~90% 절감

**2. CodeCompressor** — `code_compressor.py`
- tree-sitter **AST 기반** (문자열 자르기가 아님)
- Python, JS/TS, Go, Rust, Java, C/C++, Perl 지원
- import, 함수 시그니처, 타입 보존 / 본문만 요약

**3. Kompress-v2-base** — `kompress_compressor.py`
- HuggingFace 자체 학습 모델 (ModernBERT 기반)
- 에이전트 실제 트레이스로 학습
- `chopratejas/kompress-v2-base`

**4. CacheAligner** — `cache_aligner.py`
- 공급자 KV 캐시 prefix를 깨뜨릴 변동성 콘텐츠 **탐지만** 수행
- 프롬프트를 절대 재작성하지 않음 (플래그만 세움)
- `live_zone_anthropic.rs` / `live_zone_openai.rs` / `live_zone_responses.rs` — 새 바이트만 압축

**기타 주요 transform 모듈:** `log_compressor`, `diff_compressor`, `search_compressor`, `config_compressor`, `html_extractor`, `text_crusher`, `thinking_compactor`, `cross_turn_dedup`, `recursive_json`, `tabular_ingest`, `spreadsheet_ingest`, `lossless_compaction`, `adaptive_sizer`, `anchor_selector`, `relevance_split`, `tag_protector`

### 3.4 CCR (Compress-Cache-Retrieve)

```
원본 → 로컬 SQLite 보관 → 압축본만 LLM 전송
                            ↓
              LLM: "원본이 필요하다"
                            ↓
          headroom_retrieve(hash) → 원본 복원
```

정보를 **버리는 게 아니라 접는** 구조. 관련 파일: `headroom/ccr/`, `headroom/cache/compression_store.py`, `headroom/cache/backends/sqlite.py`

### 3.5 프록시 (`headroom/proxy/` + `crates/headroom-proxy/`)

사실상 풀스펙 API 게이트웨이:

| 기능 | 구현 위치 |
|---|---|
| Anthropic `/v1/messages` | `sse/anthropic.rs`, `compression/anthropic.rs` |
| OpenAI `/v1/chat/completions` | `handlers/chat_completions.rs`, `sse/openai_chat.rs` |
| OpenAI `/v1/responses` | `handlers/responses.rs`, `sse/openai_responses.rs` |
| AWS Bedrock | `bedrock/` — SigV4 서명 직접 구현, eventstream→SSE 변환 |
| Google Vertex AI | `vertex/` — ADC 인증, rawPredict / streamRawPredict |
| WebSocket 프록시 | `websocket.rs` (Codex gpt-5.4+) |
| Prometheus 메트릭 | `observability/prometheus.rs` |
| 캐시 안정화 | `cache_stabilization/` — 7개 모듈 |

### 3.6 인증 모드 자동 판별 (`headroom/proxy/auth_mode.py`)

8단계 판별 (most-specific wins):

```
1. 구독 CLI User-Agent        → SUBSCRIPTION
2. Bearer sk-ant-oat*         → OAUTH   (Claude Pro/Max)
3. Bearer sk-ant-api* / sk-*  → PAYG    (Anthropic/OpenAI API 키)
4. Bearer <JWT 3조각>         → OAUTH   (Codex/Cursor/Copilot)
5. Authorization 비-Bearer    → OAUTH   (AWS4-HMAC-SHA256 = Bedrock)
6. x-api-key                  → PAYG
7. x-goog-api-key             → PAYG    (Gemini)
8. 기본값                     → PAYG    (가장 안전)
```

**설계 의도(코드 주석 기준):** 구독 계정이 오분류되어 공격적으로 압축되면 **계정 정지 위험**이 있으나, PAYG 오분류는 재실행 비용만 발생. 따라서 구독 계정 보호를 최우선으로 설계.

성능 목표: 호출당 10µs 미만 (`str.lower` 1회 외 zero-allocation).

### 3.7 Rust 워크스페이스 (`crates/`)

| 크레이트 | 역할 |
|---|---|
| `headroom-core` | 순수 압축 로직 |
| `headroom-proxy` | 고성능 프록시 |
| `headroom-py` | PyO3 바인딩 (maturin 빌드 필수) |
| `headroom-parity` | **Python ↔ Rust 결과 동등성 검증** |
| `headroom-simulators` | 테스트용 목 서버 |

Rust 1.80+, edition 2021. `headroom-parity`가 별도 크레이트로 존재하는 것이 특히 인상적 — 두 구현이 동일한 결과를 내는지 검증하는 전용 크레이트.

### 3.8 부가 기능

| 기능 | 위치 | 설명 |
|---|---|---|
| Cross-agent Memory | `headroom/memory/` (26 모듈) | Claude·Codex·Gemini·Grok 메모리 공유, 자동 중복제거, 프로젝트별 격리(GH #462) |
| `headroom learn` | `headroom/learn/` | 실패 세션 채굴 → `CLAUDE.local.md`에 교정사항 자동 작성 |
| Output Shaper | `HEADROOM_OUTPUT_SHAPER=1` | 모델이 **쓰는** 토큰 절감 (Opus급은 출력이 입력의 5배 비쌈) |
| SharedContext | `headroom/shared_context.py` | 멀티 에이전트 간 압축 컨텍스트 전달 |
| Image Compression | `headroom/image/` | ML 라우터로 40~90% 압축 |
| MCP 서버 | `headroom/ccr/mcp_server.py` | `headroom_compress` / `headroom_retrieve` / `headroom_stats` |
| Dashboard | `headroom/dashboard/` | 실시간 절감 조회 |
| Pricing | `headroom/pricing/` | 달러 환산 |

### 3.9 Output Shaper 상세

| 기법 | 동작 |
|---|---|
| **Verbosity steering** | 시스템 프롬프트 **끝**에 "간결하게" 지침 추가 → 프롬프트 캐시 유지 |
| **Effort routing** | 툴 결과 재개 턴은 thinking 하향, 새 질문/에러는 풀 effort |

- OpenAI: `reasoning_effort`
- Anthropic: `thinking.budget_tokens` / `output_config.effort`
- **clamp-only 불변식** — 올리지 않고 내리기만 함
- 양쪽 경로 동일한 `output_shaper:*` 라벨

### 3.10 파이프라인 라이프사이클 11단계

```
Setup → Pre-Start → Post-Start → Input Received → Input Cached
→ Input Routed → Input Compressed → Input Remembered
→ Pre-Send → Post-Send → Response Received
```

확장 seam 3종:
- **Pipeline extensions** — `on_pipeline_event(...)`로 단계 관찰/커스터마이즈
- **Compression hooks** — 라이프사이클 옆에 붙는 추가 seam
- **Proxy extensions** — ASGI 미들웨어 / 라우트 / 시작 정책

> 참고: IntelligentContext와 RollingWindow는 PR-B1에서 폐기됨.

### 3.11 엔터프라이즈급 운영 체계

```
.github/workflows/   23개 (ci, security, release, docker, rust, e2e, eval, docs...)
sbom/                CycloneDX + SPDX SBOM, 취약점 스캔, 라이선스 CSV
.gitleaks.toml       시크릿 스캔
.gitguardian.yaml    시크릿 스캔
deny.toml            Rust 의존성 정책
.pre-commit-config   커밋 전 검증
.release-please-*    자동 릴리즈
codecov.yml          커버리지
.devcontainer/       개발 컨테이너 (+ memory-stack: Qdrant/Neo4j)
.actrc               로컬 GitHub Actions 실행(act)
.commitlintrc.json   커밋 메시지 규약
```

### 3.12 공급망 보안 대응 사례

`pyproject.toml`:

```python
"ast-grep-cli>=0.30.0,!=0.44.1",
# 0.44.1 = info-stealer(sg.exe, Trojan:Win64/Lazy!MTB)가 섞인
# 악성 공급망 빌드 (GH #2332). != 로 해당 버전만 제외.
```

실제 공급망 사고를 코드에 기록으로 남긴 점이 신뢰도를 높인다.

---

## 4. 성능 데이터

### 4.1 토큰 절감 (재현 가능: `uv run python benchmarks/index_proof_table.py --seed 20260902`)

| 시나리오 | 전 | 후 | 절감 |
|---|---:|---:|---:|
| 코드 검색 (100 results) | 17,199 | 13,597 | **21%** |
| SRE 장애 디버깅 | 55,957 | 24,340 | **57%** |
| 코드베이스 탐색 | 58,801 | 33,895 | **42%** |
| GitHub 이슈 트리아지 | 46,067 | 32,429 | **30%** |
| 반복 JSON / 로그 라인 | — | — | **90%+** |

### 4.2 정확도 (`python -m headroom.evals suite --tier 1`)

| 벤치마크 | 분류 | N | 베이스라인 | Headroom | 차이 |
|---|---|---:|---:|---:|---|
| GSM8K | 수학 | 100 | 0.870 | 0.870 | ±0.000 |
| TruthfulQA | 사실성 | 100 | 0.530 | 0.560 | +0.030 |
| SQuAD v2 | QA | 100 | — | 97% | 19% 압축 시 |
| BFCL | 툴 호출 | 100 | — | 97% | 32% 압축 시 |

> N=100에서 ±0.03은 신뢰구간 내 → TruthfulQA는 "개선"이 아니라 "유의미한 차이 없음"으로 해석해야 함.

### 4.3 지연시간

| 입력 크기 | 압축 시간 |
|---|---|
| 10K 토큰 JSON | **0.21ms** (p50) |
| 100K 토큰 | **1.4ms** |

1ms 미만이라 에이전트 지연에 체감되지 않음.

### 4.4 측정 방식 주의

출력 토큰 절감은 **반사실(counterfactual)** 이므로 추정치로 제공:

```bash
headroom output-savings
# Reduction: 31.7%  (95% CI 27.7% … 35.7%)   [estimated]
```

실측치를 원하면 10% 대조군 설정:
```bash
export HEADROOM_OUTPUT_HOLDOUT=0.1
# → 대시보드 "Output Tokens Saved" 카드가 estimated → measured 로 변경
```

---

## 5. 설치 및 사용법

### 5.1 설치 4가지 방법

```bash
# 방법 1 (권장): uv — 독립 환경, 의존성 충돌 없음
uv tool install --python 3.13 "headroom-ai[all]"

# 방법 2: pip — CLI 포함
pip install "headroom-ai[all]"

# 방법 3: npm — ⚠️ TypeScript SDK 전용! CLI 명령어 없음
npm install headroom-ai

# 방법 4: Docker
docker pull ghcr.io/headroomlabs-ai/headroom:latest
docker run -p 8787:8787 ghcr.io/headroomlabs-ai/headroom:latest
```

> ⚠️ **중요:** npm 패키지에는 `headroom` CLI가 **없다.** CLI는 PyPI 패키지에만 포함.
> npm은 `import { compress } from 'headroom-ai'` 라이브러리 전용.

### 5.2 선택 설치 옵션 (extras)

| extra | 내용 |
|---|---|
| `[proxy]` | FastAPI 프록시 서버 (가장 많이 쓰임) |
| `[proxy-prod]` | gunicorn 포함 (Unix 전용) |
| `[mcp]` | MCP 서버 |
| `[ml]` | Kompress-v2-base ML 모델 |
| `[code]` | tree-sitter AST 코드 압축 |
| `[memory]` | 영속 메모리 |
| `[vector]` | HNSW 벡터 백엔드 (C++ 툴체인 필요, `[all]`에 없음) |
| `[relevance]` | 임베딩 관련성 |
| `[image]` | 이미지 압축 |
| `[evals]` | 벤치마크 평가 |
| `[pytorch-mps]` | Apple GPU 오프로드 (`HEADROOM_EMBEDDER_RUNTIME=pytorch_mps`) |
| `[langchain]` `[agno]` `[strands]` `[anyllm]` `[bedrock]` | 프레임워크 어댑터 — **별도 설치 필요** |

> ⚠️ `[all]`은 코어 스택만 커버하며 **프레임워크 어댑터는 포함하지 않는다.**

### 5.3 6가지 실행 모드

#### 모드 1 — 에이전트 래핑 (가장 쉬움)

```bash
headroom wrap claude          # Claude Code
headroom wrap codex           # OpenAI Codex (Claude와 메모리 공유)
headroom wrap cursor          # Cursor (수동 설정 안내 출력)
headroom wrap copilot         # GitHub Copilot CLI
headroom wrap vscode          # VS Code Copilot
headroom wrap vscode-claude   # VS Code의 Claude Code 확장
headroom wrap grok | aider | opencode | cline | continue | goose |
             openhands | openclaw | vibe | omp | zcode | kimi

headroom unwrap claude        # 되돌리기
```

**지원 에이전트 19종**

| 에이전트 | wrap | 비고 |
|---|:---:|---|
| Claude Code | ✅ | `--memory` · `--code-graph` · `--1m` · `--tool-search` |
| Codex | ✅ | Claude와 메모리 공유 |
| Grok CLI | ✅ | `GROK_MODELS_BASE_URL` 경유 |
| Cursor | 수동 | 프록시 시작 + base URL 출력 |
| Aider / Copilot CLI / Goose / OpenHands / Mistral Vibe | ✅ | 프록시 시작 + 실행 |
| VS Code Copilot | ✅ | 투명 프록시, 선택 모델 유지 |
| OpenClaw | ✅ | ContextEngine 플러그인으로 설치 |
| OpenCode / Cline / Continue / Oh My Pi | ✅ | 설정 주입 + 프록시 시작 |
| Cortex Code | 라이브러리만 | 60~65% 절감, wrap 없음 |
| Kimi CLI | ✅ | OAuth bearer 전달 |
| ZCode | ✅ | 프록시 시작 + base URL 출력 |

> ⚠️ `headroom wrap`은 Serena MCP를 **user scope**(`~/.claude.json`)에 설치하므로 다른 프로젝트에도 남는다. `headroom unwrap`으로 제거. `--code-memory none`으로 건너뛸 수 있음.

#### 모드 2 — 프록시 (코드 수정 0)

```bash
headroom proxy --port 8787
# 클라이언트 base_url → http://127.0.0.1:8787
```

#### 모드 3 — Python 라이브러리

```python
from headroom import compress
from openai import OpenAI

messages = [{"role": "user", "content": "Analyze these results"}]
result = compress(messages, model="gpt-4o")

client = OpenAI()
response = client.chat.completions.create(model="gpt-4o", messages=result.messages)
print(f"Saved {result.tokens_saved} tokens ({result.compression_ratio:.0%})")
```

#### 모드 4 — TypeScript

```typescript
import { compress, withHeadroom } from 'headroom-ai';

// A. 직접 압축
const result = await compress(messages, { model: 'gpt-4o' });

// B. SDK 래핑
const openai = withHeadroom(new OpenAI());
const anthropic = withHeadroom(new Anthropic());

// C. Vercel AI SDK 미들웨어
wrapLanguageModel({ model, middleware: headroomMiddleware() });
```

#### 모드 5 — MCP 서버

```bash
headroom mcp install     # 자동 설치
headroom mcp serve       # 직접 실행
```

PATH를 상속받지 못하는 클라이언트(Codex 등)는 절대경로 사용:

```toml
[mcp_servers.headroom]
command = "/Users/you/.local/bin/headroom"   # command -v headroom 결과
args = ["mcp", "serve"]
```

#### 모드 6 — 턴키 배포

```bash
headroom deploy          # 로컬 배포 + 에이전트 설정 일괄
```

### 5.4 CLI 명령어 전체

```bash
headroom doctor            # 헬스체크 — 라우팅 정상 확인
headroom perf              # 성능 측정
headroom dashboard         # 실시간 절감 대시보드 (프록시 실행 필요)
headroom savings           # 실제 트래픽 기준 절감액
headroom output-savings    # 출력 토큰 절감 (추정 + 신뢰구간)
headroom agent-savings     # 에이전트별 절감
headroom learn             # 실패 세션 채굴
headroom learn --verbosity # 적정 간결도 학습 (--apply 로 저장)
headroom memory            # 메모리 관리
headroom audit             # 감사
headroom capture           # 트래픽 캡처
headroom inspect           # 검사
headroom recover           # 복구
headroom rollout           # 롤아웃
headroom evals             # 평가 스위트
headroom copilot-auth      # Copilot OAuth
headroom update            # 업데이트 (pip/pipx/uv 자동 감지)
headroom init              # 초기화
headroom --version
```

### 5.5 주요 환경변수

```bash
export HEADROOM_OUTPUT_SHAPER=1      # 출력 토큰 절감 ON (기본 OFF)
export HEADROOM_OUTPUT_HOLDOUT=0.1   # 10% 대조군 → measured 제공
export HEADROOM_BEACON=off           # 텔레메트리 OFF (권장)
export DO_NOT_TRACK=1                # 동일 효과
export HEADROOM_UPDATE_CHECK=off     # 업데이트 체크 OFF
export HEADROOM_TLS_STRICT=0         # 회사 SSL 검사망 대응
export HF_HUB_OFFLINE=1              # HuggingFace 오프라인
export HF_ENDPOINT=https://mirror    # HF 미러
export ORT_STRATEGY=system           # ONNX Runtime 시스템 사용
export HEADROOM_EMBEDDER_RUNTIME=pytorch_mps   # Apple GPU
```

> 참고: 이 스위치들은 요청마다 live 로 읽힌다. `headroom wrap`이 **재사용한** 기존 프록시는 launch 시점 환경이 스냅샷되어 있으므로, wrap이 loopback `POST /admin/runtime-env`로 hot-sync 해준다. 공유 프록시에서는 전역 적용이며 마지막 설정이 이김.

### 5.6 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| `CERTIFICATE_VERIFY_FAILED` | 회사 SSL 검사(MITM) 프록시 | Rust 선설치 또는 `pip install --only-binary headroom-ai headroom-ai` |
| `Basic Constraints of CA cert not marked critical` | Python 3.13 + OpenSSL 3.x `VERIFY_X509_STRICT` (Zscaler 등) | `HEADROOM_TLS_STRICT=0` |
| Intel macOS 빌드 실패 (GH #941) | `ort-sys`가 x86_64-apple-darwin 프리빌드 없음 | `brew install onnxruntime` + `ORT_STRATEGY=system` + `ORT_LIB_LOCATION=$(brew --prefix onnxruntime)/lib` + `ORT_PREFER_DYNAMIC_LINK=1` |
| x86에서 ONNX 기능 폴백 | **AVX2 미지원 CPU** | BM25 relevance / 휴리스틱 검출로 자동 폴백 (크래시 없음) |
| `headroom` 명령어 없음 | npm으로 설치함 | PyPI로 설치 |
| MCP 클라이언트가 못 찾음 | PATH 상속 안 됨 | 절대경로 지정 |

**네트워크로 나가는 TLS 자산 2개:**
- `cdn.pyke.io` — Rust 코어용 ONNX Runtime (`ORT_STRATEGY=system`으로 회피)
- `huggingface.co` — `kompress-base` 모델 (`HF_HUB_OFFLINE=1` / `HF_ENDPOINT`로 회피)

> 압축을 끈 순수 게이트웨이 모드는 두 자산 모두 필요 없음.
> Windows는 회사 루트 인증서가 **machine** 저장소에 있어야 함.
> Rust 코어의 ONNX 다운로드는 별도 TLS 스택(rustls/OS trust store)이므로 `HEADROOM_TLS_STRICT` 영향을 받지 않음.

---

## 6. 플러그인 / 스킬 / MCP — 정체는?

### 결론: **전부 다 해당하지만, 본질은 "독립 실행 인프라"**

| 형태 | 해당? | 근거 |
|---|:---:|---|
| **독립 CLI** | ✅ 본질 | `pyproject.toml → [project.scripts]`: `headroom = "headroom.cli:main"`, `headroom-cache-ttl = "headroom.cache.ttl_estimator:main"` |
| **HTTP 프록시** | ✅ 본질 | `headroom/proxy/`, `crates/headroom-proxy/` |
| **라이브러리** | ✅ | `headroom/compress.py` → `compress()` |
| **MCP 서버** | ✅ | `server.json`, `headroom/ccr/mcp_server.py` |
| **Claude Code 플러그인** | ✅ | `.claude-plugin/marketplace.json` → `plugins/headroom-agent-hooks` |
| **Claude Skill** | ❌ | `SKILL.md` 없음 |

### 계층 구조

```
┌──────────────────────────────────────────────────────┐
│  본질: 로컬 압축 프록시 + 라이브러리 (인프라)          │
└──────────────────────────────────────────────────────┘
       ↑           ↑            ↑            ↑
  ┌────┴───┐ ┌────┴────┐ ┌─────┴────┐ ┌────┴────┐
  │ CLI    │ │ MCP서버 │ │ 플러그인  │ │ SDK     │
  │ wrap   │ │ 3 tools │ │ hooks    │ │ Py / TS │
  └────────┘ └─────────┘ └──────────┘ └─────────┘
```

### MCP 서버로서

`server.json` (MCP 공식 레지스트리 스키마 `2025-12-11`):

```json
{
  "name": "io.github.headroomlabs-ai/headroom",
  "version": "0.37.0",
  "packages": [{
    "registryType": "pypi",
    "identifier": "headroom-ai",
    "runtimeHint": "uvx",
    "runtimeArguments": [{ "type": "named", "name": "--from", "value": "headroom-ai[mcp]" }],
    "transport": { "type": "stdio" },
    "packageArguments": [
      { "type": "positional", "value": "headroom" },
      { "type": "positional", "value": "mcp" },
      { "type": "positional", "value": "serve" }
    ]
  }]
}
```

| MCP 툴 | 역할 |
|---|---|
| `headroom_compress` | 요청 시 압축 (프록시 없이도 가능) |
| `headroom_retrieve` | 해시로 원본 복원 (CCR) |
| `headroom_stats` | 압축 통계 조회 |

> 레지스트리 작성자는 산문에서 추측하지 말고 canonical `server.json`을 사용할 것 (README 권고).

### Claude Code 플러그인으로서

`.claude-plugin/marketplace.json`:

```json
{
  "name": "headroom-marketplace",
  "metadata": { "version": "0.37.0" },
  "plugins": [{
    "name": "headroom",
    "source": "./plugins/headroom-agent-hooks",
    "description": "Headroom startup hooks for Claude Code and GitHub Copilot CLI.",
    "keywords": ["headroom", "hooks", "claude-code", "copilot-cli"]
  }]
}
```

SessionStart hook 기반 — Claude Code 시작 시 프록시 자동 연결.

### `plugins/` 디렉토리 전체

| 플러그인 | 대상 | 언어 |
|---|---|---|
| `headroom-agent-hooks` | Claude Code, Copilot CLI | Shell/Python |
| `opencode` | OpenCode | TypeScript (tsup + vitest) |
| `openclaw` | OpenClaw (ContextEngine) | TypeScript |
| `headroom-oauth2` | OAuth2 인증 확장 | Python (SPEC.md 포함) |
| `hermes` | 범용 retrieve 어댑터 | Python |

**서드파티:** [Ship-Wright/headroom-plugin](https://github.com/Ship-Wright/headroom-plugin) — 상태줄에 실시간 절감 토큰 표시

### 기억할 한 문장

> **Headroom은 MCP도 되고 플러그인도 되지만, 본질은 "내 머신에서 돌아가는 압축 프록시 서버"다.**

---

## 7. API 토큰이 필요한가?

### 결론: **Headroom 자체는 토큰·가입·결제 불필요. 기존 LLM 인증을 통과만 시킴.**

### Headroom에 별도로 필요한 것

| 항목 | 필요? |
|---|:---:|
| 가입 | ❌ |
| API 키 발급 | ❌ |
| 로그인 | ❌ |
| 구독 | ❌ |

**완전 무료 + 로컬 실행 + Apache 2.0**

### 토큰 처리 방식 (코드 확인 결과)

| 동작 | 처리 |
|---|---|
| 토큰 읽기 | 인증 모드 판별만 (압축 강도 결정) |
| 토큰 저장 | ❌ 안 함 |
| 토큰 로깅 | ❌ `wire_debug_redaction_policy.py`에서 레다크션 |
| 토큰 전송 | 원래 목적지 LLM에만 (passthrough) |

**근거:**
- `headroom/proxy/wire_debug_redaction_policy.py` — `REDACT = ["authorization", "x-api-key", ...]`
- `headroom/proxy/helpers.py:369` 주석 — *"Never includes Authorization/x-api-key content or full body"*

### 예외 1 — GitHub Copilot 구독 모드

```bash
headroom copilot-auth login       # Headroom 전용 Copilot OAuth 토큰 저장
headroom wrap copilot --subscription -- --model gpt-4o
```

래퍼가 Headroom의 재사용 가능한 GitHub OAuth 토큰을 Copilot의 단기 API 토큰으로 교환하고, 시작 시 `COPILOT_PROVIDER_API_URL=...`을 출력한다. 일반 GitHub/Copilot CLI 토큰은 계정 메타데이터는 읽을 수 있어도 Copilot 토큰 교환 엔드포인트에서 거부되므로 전용 토큰이 필요.

GitHub Enterprise Server / 커스텀 도메인:
```bash
export GITHUB_COPILOT_ENTERPRISE_DOMAIN=ghe.example.com
export GITHUB_COPILOT_ENTERPRISE_URL=https://ghe.example.com   # 둘 다면 URL이 이김
```
> GitHub.com Enterprise Cloud(`github.com/enterprises/...`)는 **둘 다 설정하지 말 것** — 일반 토큰 교환 엔드포인트를 사용.

| 플랫폼 | 상태 |
|---|---|
| macOS | ✅ Keychain 재사용 라이브 테스트 |
| Windows | ✅ 디바이스 인증 (Copilot CLI 1.0.81은 레거시 Credential Manager 스키마 미노출 → 수동 로그인) |
| Linux | ⚠️ Secret Service / `secret-tool` 구현됐으나 실제 데스크톱 미검증 |
| Docker/CI | `GITHUB_COPILOT_TOKEN` / `GITHUB_COPILOT_GITHUB_TOKEN` 명시 전달 |

### 예외 2 — HuggingFace 모델 다운로드

`[ml]` extra 사용 시 `kompress-v2-base`를 HF에서 받는다. 보통 토큰 없이 가능하나 사내망 차단 시 `HF_ENDPOINT` 미러 또는 `HF_HUB_OFFLINE=1`.

### 네트워크 송신 전체

| 대상 | 용도 | 차단 방법 |
|---|---|---|
| LLM 공급자 | 실제 요청 | — |
| `cdn.pyke.io` | Rust 코어 ONNX Runtime | `ORT_STRATEGY=system` |
| `huggingface.co` | Kompress 모델 | `HF_HUB_OFFLINE=1` |
| **텔레메트리 비콘** | 압축 통계 (Cloudflare Worker, `deploy/beacon/worker.js`) | `HEADROOM_BEACON=off` |
| PyPI | 업데이트 확인 (1일 1회, 백그라운드, 논블로킹) | `HEADROOM_UPDATE_CHECK=off` |

전송 내용(README 기준): 압축 비율, 카운터, 공급자/모델 ID, OS, 아키텍처. **프롬프트·완성·코드·파일 경로는 전송하지 않음.**

### 한 문장 정리

> **Headroom은 무료이며 자체 토큰을 요구하지 않는다. 기존 Claude/OpenAI 키를 통과시키고, 그 키로 나가는 비용을 줄여준다.**

---

## 8. AI 에이전트 구축에 도움이 되는가?

### 결론: 두 가지 층위에서 크게 도움이 된다.

### 층위 A — 도구로 쓰기 ⭐⭐⭐⭐⭐

**1. 컨텍스트 한계 돌파 (에이전트 최대 병목)**
```
압축 없음: 툴 10회 호출 → 컨텍스트 포화 → 성능 저하
압축 있음: 같은 공간에 2~3배 → 30회 호출 가능
```

**2. 프레임워크 통합 완비**

| 프레임워크 | 코드 |
|---|---|
| LangChain | `HeadroomChatModel(your_llm)` |
| Agno | `HeadroomAgnoModel(your_model)` |
| Strands | 모델 래핑 + 훅 기반 툴 출력 압축 |
| LiteLLM | `litellm.callbacks = [HeadroomCallback()]` → **100+ 공급자 커버** |
| Vercel AI SDK | `wrapLanguageModel({ middleware: headroomMiddleware() })` |
| Anthropic / OpenAI SDK | `withHeadroom(client)` |
| ASGI (FastAPI) | `app.add_middleware(CompressionMiddleware)` |
| 멀티 에이전트 | `SharedContext().put / .get` |
| MCP 클라이언트 | `headroom mcp install` |

**3. 멀티 에이전트 컨텍스트 전달**
```python
from headroom import SharedContext
ctx = SharedContext()
ctx.put("research_result", 대용량_결과)   # 자동 압축
data = ctx.get("research_result")         # 필요 시 복원
```
멀티 에이전트의 "핸드오프 토큰 폭발" 문제를 직접 해결.

**4. Cross-agent Memory**
```
Claude Code ┐
Codex       ├→ 공유 메모리 (SQLite + 벡터 검색)
Gemini      │   · agent provenance 기록
Grok        ┘   · 자동 중복제거 · 프로젝트별 격리
```

**5. `headroom learn` — 자가 개선 루프**
```bash
headroom learn --verbosity --apply   # CLAUDE.local.md에 교정사항 자동 작성
```
self-improving agent 패턴의 실동작 구현.

**6. 가역성(CCR)이 에이전트에 결정적**
일반 요약은 정보 소실로 오판을 유발하지만, CCR은 "접어둠 → 필요 시 펼침". BFCL 툴 호출 97% 유지가 근거.

**7. 출력 토큰 절감** — Opus급은 출력이 입력의 5배 비쌈. Verbosity steering + Effort routing.

**8. 관측성 공짜** — Prometheus `/metrics`, `examples/grafana/headroom-dashboard.json`, `sql/create_dashboard_summary.sql`, `headroom savings` 달러 환산.

### 층위 B — 교과서로 쓰기 ⭐⭐⭐⭐⭐

| 배울 패턴 | 참고 파일 |
|---|---|
| 11단계 파이프라인 라이프사이클 + 3종 확장 seam | `headroom/pipeline.py` |
| 멀티 프로바이더 추상화 (전략 패턴) | `headroom/providers/registry.py` |
| 캐시 안정화 전략 (압축 vs prompt caching) | `crates/headroom-proxy/src/cache_stabilization/` (7 모듈) |
| SSE 스트리밍 프록시 | `crates/headroom-proxy/src/sse/` + `websocket.rs` |
| Rust ↔ Python 하이브리드 (PyO3/maturin) | `crates/headroom-py`, `crates/headroom-parity` |
| Bedrock SigV4 직접 구현 | `crates/headroom-proxy/src/bedrock/sigv4.rs` |
| 인증 모드 분류기 (10µs 목표) | `headroom/proxy/auth_mode.py` |
| 적대적/최악 케이스 테스트 문화 | `benchmarks/adversarial_ccr_tests.py`, `headroom_worst_case_benchmark.py` |
| 엔터프라이즈 운영 체계 | `.github/workflows/` (23개), `sbom/`, `deny.toml` |

**꼭 읽어볼 3개 파일:** `headroom/pipeline.py` · `headroom/providers/registry.py` · `crates/headroom-proxy/src/cache_stabilization/`

### 종합 평가

| 관점 | 점수 | 비고 |
|---|:---:|---|
| 즉시 도구로서 | ⭐⭐⭐⭐⭐ | 1~2줄 통합, 즉시 비용 절감 |
| 학습 자료로서 | ⭐⭐⭐⭐⭐ | 프로덕션 AI 인프라 교과서 |
| 에이전트 성능 향상 | ⭐⭐⭐⭐☆ | 컨텍스트 2~3배, 정확도 유지 |
| 프로덕션 투입 | ⭐⭐⭐☆☆ | Beta 단계 → 자기 워크로드 벤치마크 필수 |
| 의존성 부담 | ⭐⭐⭐☆☆ | Rust + ONNX + HF, AVX2 요구 |

---

## 9. React / PHP로 만들 수 있는가?

### 결론: **압축 엔진 자체는 불가. 그 위에 올리는 제품 레이어는 가능하며 오히려 권장.**

### 레이어별 가능성

| 레이어 | React/PHP | 이유 |
|---|:---:|---|
| Rust 압축 코어 | ❌ | AVX2 SIMD, ONNX, 네이티브 바인딩 |
| ML 텍스트 압축 | ❌ | ModernBERT 추론 (PyTorch/ONNX) |
| AST 코드 압축 | ⚠️ | tree-sitter WASM 존재하나 성능 저하 |
| SmartCrusher (JSON) | 🟡 | JS 포팅 가능하나 품질/속도 저하 |
| 프록시 서버 | 🟡 | Node로 가능하나 SSE/WebSocket/SigV4 전부 재구현 |
| MCP 클라이언트 | ✅ | TS SDK 공식 지원 |
| **대시보드 UI** | ✅✅ | React가 최적 |
| **관리자 포털** | ✅✅ | PHP(Laravel) 적합 |
| **결제/구독** | ✅✅ | Laravel Cashier / Next.js |
| 멀티테넌시 레이어 | ✅ | 비즈니스 로직은 언어 무관 |

### 왜 압축 엔진은 불가한가

```
필요 기술                     React/PHP
──────────────────────────────────────────
AVX2 SIMD 벡터 연산       →   없음
ONNX Runtime 추론         →   바인딩 없음
tree-sitter AST 파싱      →   WASM만, 느림
0.21ms 지연 목표          →   GC/인터프리터로 불가
tiktoken 정확한 카운팅    →   JS 포팅 있으나 느림
SSE 스트리밍 + 백프레셔   →   Node 가능 / PHP 매우 어려움
AWS SigV4 서명            →   라이브러리 있음
HNSW 벡터 검색            →   C++ 필요
```

**PHP는 특히 구조적으로 부적합** — 요청-응답 모델이라 장시간 실행 프록시에 맞지 않음. Swoole/RoadRunner로 가능하나 ROI가 나쁘다.

### 권장 아키텍처 — "Headroom은 엔진, 우리는 차체"

```
┌──────────────────────────────────────────────────┐
│  React / Next.js 프론트엔드  ← 직접 개발          │
│  · 실시간 절감 대시보드 (Recharts)                │
│  · 팀/프로젝트/모델별 비용 분석                   │
│  · 설정 관리 UI · 온보딩 위저드                   │
└──────────────────────────────────────────────────┘
                  ↕ REST / WebSocket
┌──────────────────────────────────────────────────┐
│  Laravel / Node 백엔드  ← 직접 개발               │
│  · 인증 / SSO · 결제·구독 (Stripe/토스페이먼츠)   │
│  · 멀티테넌시 / RBAC · 집계·리포트 · 알림          │
└──────────────────────────────────────────────────┘
                  ↕ HTTP / SQL
┌──────────────────────────────────────────────────┐
│  Headroom (Python/Rust)  ← 그대로 사용            │
│  headroom proxy --port 8787                      │
│  · GET  /metrics            (Prometheus)         │
│  · POST /admin/runtime-env  (런타임 설정 변경)    │
│  · sql/create_dashboard_summary.sql (집계)       │
└──────────────────────────────────────────────────┘
```

**이미 열려 있는 연동 포인트:**
- `GET /metrics` — Prometheus 포맷
- `POST /admin/runtime-env` — 재시작 없이 런타임 설정 변경 (loopback, CIDR 인증)
- `sql/create_dashboard_summary.sql` — 집계 쿼리
- `examples/grafana/headroom-dashboard.json` — Grafana 대시보드 샘플
- `headroom/dashboard/` — 기본 대시보드 (참고 구현)

### React로 만들면 좋은 것

1. **실시간 절감 대시보드** (최우선) — `/metrics` + SQL 집계 → WebSocket 스트리밍
2. **압축 Before/After 비주얼라이저** — 데모/마케팅 최강 무기
3. **설정 플레이그라운드** — JSON/코드 붙여넣고 실시간 미리보기
4. **크로스 에이전트 메모리 브라우저** — 메모리 그래프 시각화

### Laravel로 만들면 좋은 것

| 기능 | Laravel 강점 |
|---|---|
| 멀티테넌시 | `tenancy/tenancy` |
| 결제/구독 | **Laravel Cashier** (Stripe 완벽 통합) |
| 권한 관리 | `spatie/laravel-permission` |
| 관리자 UI | **Filament / Nova** (초고속 개발) |
| 큐/스케줄 | Horizon + Scheduler |
| 리포트 메일 | Mailable + Markdown |

### 권장 전략

```
하지 말 것: Headroom을 React/PHP로 재작성
  → 수개월 소요, 성능·품질 저하, Apache 2.0인데 무의미

할 것: Headroom을 엔진으로 쓰고 제품 레이어를 개발
  → 2~4주면 MVP
  → 차별화는 UX / 한국화 / 결제 / 리포팅에서
  → 업스트림 업데이트를 무료로 수용
```

**추천 스택:** Next.js(App Router) + Recharts + Laravel/Filament + Headroom 엔진

---

## 10. 유튜브 강의 제작 가능성

### 결론: 가능하며, 소재로서 최상급.

### 좋은 소재인 5가지 이유

| 이유 | 설명 |
|---|---|
| 숫자가 자극적 | "AI 요금 90% 절감" — 높은 CTR |
| 즉시 체감 | 명령어 1줄 → 대시보드 숫자 변화 → 영상화 완벽 |
| 비주얼 자산 제공 | 레포에 GIF 3개 + PNG 3개 |
| 재현 가능 | `--seed 20260902` 고정 벤치마크 |
| 한국어 콘텐츠 희박 | 블루오션 |

### 레포 내 활용 가능 자산

```
HeadroomDemo-Fast.gif        4.6MB   10,144 → 1,260 토큰 압축 데모
Headroom-2.gif               5.6MB   추가 데모
headroom_learn.gif          15.2MB   learn 명령어 동작
headroom-savings.png         1.2MB   절감 스크린샷
dashboard-cache-ttl-main.png 266KB   대시보드 화면
wiki/screenshots/*.png               캐시 TTL 대시보드 (live/history)
.github/assets/hero.svg              히어로 이미지
examples/07-context-compression.ipynb  주피터 노트북 데모
```

### 5부작 시리즈 기획

| EP | 제목 | 길이 | 핵심 |
|---|---|---|---|
| **1** | Claude Code 요금 반으로 줄이는 방법 (명령어 1줄) | 8~10분 | 훅 → 문제 → 설치 → 실측 → 정확도 검증 → 주의사항 |
| **2** | 토큰을 90% 줄이면서 답이 안 틀리는 원리 | 12~15분 | 4개 압축기 비유, CCR, CacheAligner, 애니메이션 |
| **3** | 같은 작업을 압축 ON/OFF로 해봤습니다 | 15분 | 동일 작업 2회, 토큰/비용/시간/품질 4지표 비교 |
| **4** | AI 에이전트 인프라는 이렇게 만든다 (코드 리딩) | 20분 | pipeline.py, providers/registry.py, cache_stabilization, Rust 하이브리드 |
| **5** | 오픈소스로 돈 버는 법 (Apache 2.0 활용) | 12분 | 라이선스 읽기, 수익화 전략 |

**EP.1 상세 타임라인**
```
00:00 훅 — "오늘 Claude에 쓴 토큰" (충격 숫자)
00:45 문제 — 로그 10,000줄 중 FATAL 1줄만 필요한데 전부 전송
02:00 솔루션 — Headroom (GIF 활용)
03:00 설치 — pip install → headroom wrap claude (라이브)
05:00 실측 — headroom dashboard (숫자 변화)
07:00 정확도 — "답이 나빠지나?" → GSM8K 0.870 동일
08:30 주의 — 텔레메트리 끄기, Beta 단계
09:30 CTA
```

### 쇼츠 아이디어

| # | 제목 | 길이 |
|---|---|---|
| 1 | AI 요금 90% 줄이는 1줄 | 30초 |
| 2 | 10,144 → 1,260 | 15초 |
| 3 | 압축했는데 답이 똑같은 이유 | 45초 |
| 4 | Claude 캐시 할인 지키는 방법 | 40초 |
| 5 | 이 오픈소스 README 보셨어요? (공급망 사고 대응) | 30초 |

### 저작권 체크리스트

| 항목 | 가능? | 비고 |
|---|:---:|---|
| 코드 설명/인용 | ✅ | Apache 2.0 |
| GIF/이미지 사용 | ✅ | 저작자 표시 권장 |
| 유료 강의 제작 | ✅ | 상업적 이용 허용 |
| "Headroom" 이름 **언급** | ✅ | 설명/리뷰 목적 |
| "Headroom" 을 **내 제품명**으로 | ❌ | 상표 문제 |
| 수정판 재배포 | ⚠️ | LICENSE + NOTICE 포함 + 변경사항 표시 |

**영상 설명란 권장 문구:**
```
Headroom is licensed under Apache License 2.0.
Repository: https://github.com/headroomlabs-ai/headroom
Docs: https://docs.headroomlabs.ai
This video is an independent review/tutorial and is not
affiliated with or endorsed by Headroom Labs.
```

### 제작 팁

**권장**
- 첫 15초에 절감 숫자 (이탈 방어)
- 실제 요금 명세서 모자이크 공개 (신뢰도)
- 터미널 폰트 18pt+ (모바일 가독성)
- `--seed` 고정 벤치마크로 "재현 가능" 강조
- 단점도 솔직히 (Beta, AVX2, 텔레메트리) → 신뢰도 상승
- 챕터 마커 필수

**금지**
- "무조건 90% 절감" 과장 — 산문은 거의 안 줄어듦
- API 키 노출 (환경변수 모자이크)
- Headroom 공식 채널처럼 보이게 하기
- 15MB GIF 원본 그대로 업로드

### 예상 성과

| 지표 | 보수적 | 낙관적 |
|---|---|---|
| EP.1 조회수 | 3,000~8,000 | 30,000~100,000 |
| 구독 전환 | 1~2% | 3~5% |
| 애드센스 (5편) | 10~30만원 | 100~300만원 |
| **실질 가치** | **강의/컨설팅 리드 확보** | |

---

## 11. 수익화 아이디어 (상세)

### 11.0 법적 기반 — Apache License 2.0

| 권리 | 가능? | 의미 |
|---|:---:|---|
| 상업적 이용 | ✅ | 유료 판매 가능 |
| 수정 | ✅ | 자유 수정 |
| 재배포 | ✅ | 배포 가능 |
| 특허 사용권 | ✅ | **명시적 부여** (MIT에 없는 장점) |
| 사유 소프트웨어화 | ✅ | 비공개 제품에 포함 가능 |
| **SaaS 제공** | ✅ | **소스 공개 의무 없음** |

**의무사항**
1. LICENSE 파일 포함
2. NOTICE 파일 포함 (레포에 존재)
3. 변경한 파일에 변경 표시
4. 원 저작권 표시 유지

**금지**
- "Headroom" 상표를 제품명/브랜드로 사용
- 원 저작자의 보증/후원 암시
- 라이선스/저작권 표시 제거

> **AGPL이 아니라 Apache 2.0인 것이 핵심.** AGPL이면 SaaS 제공 시 전체 소스 공개 의무가 있으나, Apache 2.0은 없다. 즉 "Headroom 기반 비공개 SaaS"가 완전히 합법.

**경쟁 구도:** README의 "Headroom for teams" 섹션 + `hello@headroomlabs.ai` → 원 저작자가 이미 기업 영업 중. 따라서 **글로벌 매니지드 SaaS 정면 승부는 불리**하고, **한국 시장 / 수직 특화 / 서비스 레이어** 차별화가 유리하다.

### 기회 지도

```
       수익 크기 ↑
  대  │  ④ 기업 컨설팅/MSP      ⑤ 매니지드 SaaS
      │     (고난도·고수익)         (고난도·최고수익)
  중  │  ② SaaS 대시보드         ③ 수직 통합 제품
      │     (중난도)                (중난도)
  소  │  ① 콘텐츠/강의           ⑥ 플러그인/템플릿
      │     (저난도·즉시)           (저난도)
      └────────────────────────────────────────→ 난이도
```

| # | 아이디어 | 난이도 | 예상 수익 | 리드타임 |
|---|---|:---:|---|---|
| ① | 콘텐츠·강의 | 🟢 낮음 | 월 50~500만 | 2~4주 |
| ② | 토큰 비용 SaaS 대시보드 | 🟡 중간 | MRR 300~3,000만 | 2~3개월 |
| ③ | 한국 시장 수직 통합 | 🟡 중간 | MRR 500만~ | 3~4개월 |
| ④ | 기업 컨설팅/MSP | 🔴 높음 | 건당 500~5,000만 | 즉시~2개월 |
| ⑤ | 매니지드 호스팅 | 🔴 높음 | ARR 1억~ | 4~6개월 |
| ⑥ | 플러그인·템플릿 | 🟢 낮음 | 월 30~300만 | 1~3주 |

---

### 아이디어 ① 콘텐츠 & 교육 사업 🟢

| 항목 | 내용 |
|---|---|
| 난이도 | 낮음 / 초기 투자 거의 0원 |
| 리드타임 | 2~4주 |
| 예상 수익 | 월 50만 ~ 500만원 |
| 리스크 | 매우 낮음 |

**A. 유튜브 채널 (무료 유입)** — 10번의 5부작 + 쇼츠 5개로 "AI 비용 최적화" 니치 선점. 애드센스 10~300만원 + **리드 생성 파이프라인**.

**B. 유료 온라인 강의 (핵심 수익원)**
```
플랫폼: 인프런 / 클래스101 / 패스트캠퍼스 / Udemy
강의명: "AI 에이전트 비용 최적화 실전"  (총 8시간)
  1장. LLM 토큰 비용 구조 완전 이해        (1.0h)
  2장. 컨텍스트 압축의 원리                (1.5h)
  3장. Headroom 설치부터 실전까지          (2.0h)
  4장. 프레임워크 통합 (LangChain/Agno)     (1.5h)
  5장. 비용 대시보드 직접 만들기 (React)    (1.5h)  ← 차별화
  6장. 기업 도입 전략 & 벤치마킹            (0.5h)
가격: 99,000 ~ 149,000원
목표: 월 50~200명 → 월 500만 ~ 2,000만원 (플랫폼 수수료 30~50% 차감)
```

**C. 유료 뉴스레터** — "월간 AI 비용 리포트" 월 9,900원 × 300명 = 월 297만원

**D. 기술 블로그 + 제휴** — SEO + 제휴 링크 + 스폰서 포스트, 월 30~100만원

**실행 로드맵**
```
Week 1: 직접 사용 + 실측 데이터 수집 (스크린샷/숫자 확보)  ← 가장 중요
Week 2: 블로그 3편 + 쇼츠 3개
Week 3: 유튜브 EP.1 업로드
Week 4: 반응 분석 → 강의 기획 확정
Month 2~3: 강의 녹화 & 출시
```

---

### 아이디어 ② 토큰 비용 관측 SaaS 🟡

| 항목 | 내용 |
|---|---|
| 난이도 | 중간 / 초기 투자 500~2,000만원 |
| 리드타임 | 2~3개월 |
| 예상 수익 | MRR 300만 ~ 3,000만원 |
| 스택 | Next.js + Laravel + Headroom |

**컨셉:** "Datadog for LLM Costs" — 팀이 AI에 얼마 쓰고 얼마 아꼈는지 한눈에

**Headroom OSS가 주지 않는 것 = 판매 포인트**

| Headroom OSS | 우리 제품 |
|---|---|
| 개인 로컬 대시보드 | 팀 클라우드 대시보드 |
| 환경변수 설정 | 웹 UI 중앙 설정 |
| 로컬 SQLite | 멀티테넌트 DB + 장기 히스토리 |
| 인증 없음 | SSO / RBAC |
| 알림 없음 | Slack/이메일 예산 알림 |
| 영어만 | 한국어 UI + 원화 환산 |
| 커뮤니티 지원 | SLA 지원 |

**가격 모델**

| 플랜 | 가격 | 대상 |
|---|---|---|
| Free | 0원 | 개발자 1명, 7일 히스토리 |
| Team | 월 29,000원/석 | 5~20명, 90일 히스토리, Slack 알림 |
| Business | 월 59,000원/석 | SSO, RBAC, 무제한 히스토리, API |
| Enterprise | 별도 견적 | 온프레미스, VPC, SLA, 세금계산서 |

```
손익 시나리오
 10팀 × 평균  8석 × 29,000원 = MRR   232만원
 50팀 × 평균 10석 × 35,000원 = MRR 1,750만원
100팀 × 평균 12석 × 40,000원 = MRR 4,800만원
```

**킬러 기능**
1. **절감 영수증** — 월말 "이번 달 847만원 절감" 리포트 자동 발송 (갱신율 최강 무기)
2. **ROI 계산기** — 반사실 추정 + 신뢰구간
3. **예산 가드레일** — 예산 80% 도달 시 Slack 알림 + 자동 압축 강화
4. **압축 Before/After 뷰어** — 데모/영업용
5. **팀 리더보드** — 게이미피케이션

---

### 아이디어 ③ 한국 시장 수직 통합 🟡

| 항목 | 내용 |
|---|---|
| 난이도 | 중간 / 리드타임 3~4개월 |
| 예상 수익 | MRR 500만원 ~ |
| 차별점 | 한국어 압축 품질 + 국내 결제/규정 |

**발견한 빈틈:** `benchmarks/i18n_compression_eval.py`로 다국어 평가는 있으나 **한국어 특화 최적화는 없음**. Kompress-v2-base도 영어 중심 학습.

| 영역 | 추가할 가치 |
|---|---|
| 한국어 압축 | 한국어 토크나이저 최적화, 조사/어미 처리, 한국어 로그 패턴 |
| 국내 LLM | HyperCLOVA X, SOLAR(Upstage), A.X(SKT), Gauss(삼성) |
| 결제 | 토스페이먼츠, 이니시스, **세금계산서 자동 발행** |
| 규정 | 개인정보보호법, 금융권 망분리, 공공 CSAP |
| 지원 | 한국어 문서, 카톡/채널톡, 한국 시간대 SLA |

**가장 큰 기회 — 망분리 환경**
```
금융권/공공기관은 외부 LLM API 직접 호출 불가
→ 폐쇄망 내 Headroom 배포 + 게이트웨이
→ 온프레미스 라이선스 = 건당 수천만원
→ 해외 업체는 국내 규정 대응 안 함 → 경쟁자 거의 없음
```

**가격:** SaaS 월 39,000원/석 · 온프레미스 연 2,000만~1억원 · 구축 지원 건당 500만~3,000만원

---

### 아이디어 ④ 기업 도입 컨설팅 / MSP 🔴

| 항목 | 내용 |
|---|---|
| 난이도 | 높음 (전문성 필요) / 초기 투자 거의 0원 |
| 리드타임 | 즉시 ~ 2개월 |
| 예상 수익 | **건당 500만 ~ 5,000만원** |
| 확장성 | 낮음 (사람 시간에 비례) |

**ROI가 명확해 의사결정이 빠름**
```
기업 월 LLM 지출 5,000만원 → 40% 절감 = 월 2,000만원
→ 연 2.4억 절감 → 컨설팅비 3,000만원은 즉시 승인 가능
```

**서비스 패키지**

| 패키지 | 가격 | 기간 | 산출물 |
|---|---|---|---|
| **A. 토큰 비용 진단** (입구 상품) | 300~500만원 | 2주 | 현행 지출 분석, 워크로드별 압축 가능성 벤치마크, 예상 절감액+신뢰구간, 도입 로드맵, 리스크 평가 |
| **B. 도입 구축** | 1,500~3,000만원 | 4~8주 | 사내 배포(K8s/VPC/에어갭), 중앙 설정+버전 롤아웃, Grafana/Prometheus 관측, CI 통합, 교육 2회, 운영 런북 |
| **C. MSP 운영 대행** (반복 수익) | 월 300~1,000만원 | — | 24/7 모니터링, 월간 절감 리포트, 업그레이드 대행, 압축 정책 튜닝, 긴급 대응 SLA |
| **D. 커스텀 압축기 개발** | 1,000~5,000만원 | — | 고객 데이터 포맷 전용 압축기 (`headroom/transforms/`에 모듈 추가) — 금융 거래 로그, 의료 EMR, 제조 센서 등 |

**타겟 고객 우선순위**
1. AI 코딩 에이전트를 팀 단위로 쓰는 스타트업 (Claude Code/Cursor 팀 라이선스 구매)
2. LLM 기반 SaaS 운영사 (API 비용이 원가)
3. CI에서 AI 에이전트 돌리는 기업 (토큰 폭발)
4. 금융/공공 (망분리 + 온프레미스)

**리드 발굴:** 채용 공고에 "LangChain/LLM" 있는 기업 / AWS Bedrock·Azure OpenAI 도입 사례 발표 기업 / 유튜브·블로그 인바운드(아이디어 ① 연계)

---

### 아이디어 ⑤ 매니지드 Headroom 호스팅 🔴

| 항목 | 내용 |
|---|---|
| 난이도 | 매우 높음 / 초기 투자 3,000만~1억원 |
| 리드타임 | 4~6개월 |
| 예상 수익 | ARR 1억원 ~ |
| 리스크 | **원 저작자와 정면 경쟁** |

**치명적 문제점**
1. 로컬 실행이 Headroom의 핵심 가치인데, 클라우드로 올리면 "내 코드가 남의 서버로" → 보안 거부감
2. 원 저작자가 이미 동일 사업 중 ("Headroom for teams")
3. 지연시간 추가 (로컬 0.21ms → 네트워크 왕복 수십 ms)

**가능한 변형 — VPC 전용 배포형**
```
공용 멀티테넌트 클라우드 (X)  →  고객 VPC 내 전용 배포 + 원격 관리 (O)
 · 데이터가 고객 VPC를 벗어나지 않음  (보안 해결)
 · Terraform/Helm으로 배포·운영      (확장성 확보)
 · 같은 리전이라 지연 최소
 · 가격: 월 300만 ~ 1,500만원
```

> 이 아이디어는 ②와 ④가 성공한 다음 단계로 미루는 것이 합리적.

---

### 아이디어 ⑥ 플러그인 & 템플릿 판매 🟢

| 항목 | 내용 |
|---|---|
| 난이도 | 낮음 / 리드타임 1~3주 |
| 예상 수익 | 월 30만 ~ 300만원 |

| 품목 | 내용 | 가격 |
|---|---|---|
| **A. Claude Code 플러그인** | 상태줄 실시간 절감액(원화), 세션별 비용 추적, 예산 초과 경고 | $9 일회 또는 $3/월 |
| **B. 대시보드 템플릿** | "Next.js LLM Cost Dashboard" 보일러플레이트, Recharts 차트 12종, 멀티테넌시 스캐폴딩 | $49 ~ $149 (Gumroad/Lemon Squeezy) |
| **C. 커스텀 압축기 마켓** | 금융 거래 로그 / 의료 EMR / 제조 센서 로그 압축기 | $99 ~ $499 each |
| **D. Notion/Obsidian 템플릿** | "AI 비용 관리 OS" — 월별 지출 추적, 모델별 단가 DB, ROI 계산 | $19 ~ $39 |

> A는 무료 경쟁자가 이미 존재([Ship-Wright/headroom-plugin](https://github.com/Ship-Wright/headroom-plugin)) → 차별화 필수.

---

### 11.7 종합 실행 로드맵

```
PHASE 1 (0~1개월) — 리스크 0, 학습 극대화
  ① 콘텐츠 시작
     · 직접 사용 + 실측 데이터 확보  ← 가장 중요
     · 블로그 3편 + 쇼츠 3개 + 유튜브 EP.1
     · 목표: 시장 반응 확인 + 리드 수집
  ⑥ 템플릿 1개 출시 (Gumroad) — 첫 매출 경험
                    ↓ 반응 양호 시
PHASE 2 (1~3개월) — 수익화 본격화
  ① 유료 강의 출시 (인프런)     → 월 500만 목표
  ④ 컨설팅 "진단 패키지" 영업   → 건당 300~500만
     · 유튜브 인바운드 리드 활용
                    ↓ 현금흐름 확보 시
PHASE 3 (3~6개월) — 제품화
  ② SaaS 대시보드 MVP (Next.js + Laravel)
     · Phase 2 컨설팅 고객을 베타 유저로
  ③ 한국어 최적화 레이어 추가
                    ↓ PMF 확인 시
PHASE 4 (6개월~) — 스케일
  ③ 망분리/온프레미스 영업 (금융/공공)
  ⑤ VPC 전용 배포형 매니지드
  ④ MSP 월 구독 전환 (반복 수익)
```

**한 줄 전략**
> 콘텐츠로 신뢰와 리드를 쌓고(①), 컨설팅으로 현금과 고객 인사이트를 얻고(④), 그것으로 제품을 만들어(②③) 스케일한다.

**수익 예상 (보수적)**

| 시점 | 월 수익 | 구성 |
|---|---|---|
| 1개월 | 30만원 | 템플릿 + 애드센스 |
| 3개월 | 500만원 | 강의 + 컨설팅 1건 |
| 6개월 | 1,500만원 | 강의 + 컨설팅 2건 + SaaS 초기 |
| 12개월 | 4,000만원 | SaaS MRR + MSP + 온프레미스 1건 |

**리스크 & 대응**

| 리스크 | 대응 |
|---|---|
| LLM 가격 급락 시 가치 하락 | 압축 외 관측/거버넌스로 가치 확장 |
| 공급자 네이티브 압축 강화 | Headroom은 크로스 공급자/크로스 에이전트가 차별점 |
| 원 저작자와 충돌 | 한국 시장 집중 + 파트너십 제안 고려 (`hello@headroomlabs.ai`) |
| Beta 단계 불안정 | 고객에게 벤치마크 먼저 제시, 단계적 도입 |
| 상표 문제 | 독자 브랜드 사용, "Headroom 기반"으로만 표기 |

---

## 12. 주의사항 및 발견된 이슈

### 12.1 텔레메트리 기본값 문서 모순 ⚠️

전수조사 중 발견한 **레포 내부 모순**:

| 파일 | 기술 |
|---|---|
| `README.md` | *"An anonymous beacon is **on by default**."* |
| `llms.txt` | *"Anonymous telemetry is **off by default** (opt-in); enable with `HEADROOM_TELEMETRY=on`."* |

`deploy/beacon/worker.js` (Cloudflare Worker 수집기)가 실제로 존재한다. 전송 내용은 압축 비율·카운터·공급자/모델 ID·OS·아키텍처이며 프롬프트/코드/파일 경로는 보내지 않는다고 명시되어 있으나, **기본값이 불명확하므로 명시적으로 끄는 것을 권장**:

```bash
export HEADROOM_BEACON=off
export DO_NOT_TRACK=1
# 또는 완전 차단
headroom proxy --offline
```

### 12.2 설치 난이도

- Rust 툴체인 + ONNX Runtime 필요 (sdist 폴백 시)
- x86/x86_64에서 **AVX2 CPU 필수** (없으면 BM25/휴리스틱 폴백)
- Intel macOS는 ONNX 프리빌드 없음 (GH #941)
- 네이티브 휠은 현재 macOS Apple Silicon + Linux 중심
- 회사 SSL 검사망 대응 필요

### 12.3 `headroom wrap`의 전역 영향

- Serena MCP를 **user scope**(`~/.claude.json`)에 설치 → 다른 프로젝트에도 잔존
- `headroom unwrap <tool>`로 반드시 복구
- `--code-memory none`으로 Serena 설치 건너뛰기 가능
- unwrap 지원: `claude`, `copilot`, `codex`, `grok`, `kimi`, `omp`, `opencode`, `openclaw`, `zcode`, `vscode-claude`

### 12.4 Beta 단계

`Development Status :: 4 - Beta` — 아직 1.0 미만. 프로덕션 투입 전 자기 워크로드로 벤치마크 필수.

### 12.5 압축이 잘 안 되는 경우 (과장 금지)

- 짧은 대화형 교환
- 산문(prose)
- 이미 밀도 높은 출력
- `min_input_words` 미만 블록은 **바이트 동일하게 반환**

→ "무조건 90% 절감"은 과장. `headroom savings`로 **자기 트래픽 기준 실측**이 정답.

### 12.6 긍정 신호 — 공급망 보안 대응

```python
"ast-grep-cli>=0.30.0,!=0.44.1",
# 0.44.1 = info-stealer(sg.exe) 섞인 악성 빌드 (GH #2332)
```

실제 사고를 코드 주석으로 기록하고 해당 버전만 정밀 제외. 대응 속도와 투명성이 신뢰도를 높인다.

### 12.7 상표 주의

Apache 2.0은 **상표권을 부여하지 않는다.** "Headroom" 이름을 제품명/브랜드로 사용할 수 없고, 원 저작자의 보증/후원을 암시해서도 안 된다. 설명·리뷰·비교 목적의 **언급(nominative use)** 은 가능.

---

## 13. 참고 링크 모음

### 레포지토리

| 구분 | URL |
|---|---|
| **이 레포 (fork)** | https://github.com/bmshin94/headroom |
| **원본 레포** | https://github.com/headroomlabs-ai/headroom |
| Issues | https://github.com/headroomlabs-ai/headroom/issues |
| CHANGELOG | https://github.com/headroomlabs-ai/headroom/blob/main/CHANGELOG.md |

### 배포 채널

| 구분 | URL |
|---|---|
| PyPI | https://pypi.org/project/headroom-ai/ |
| npm | https://www.npmjs.com/package/headroom-ai |
| Docker | `ghcr.io/headroomlabs-ai/headroom:latest` |
| HuggingFace 모델 | https://huggingface.co/chopratejas/kompress-v2-base |

### 문서

| 구분 | URL |
|---|---|
| 문서 홈 | https://docs.headroomlabs.ai |
| Quickstart | https://docs.headroomlabs.ai/docs/quickstart |
| 설치 가이드 | https://docs.headroomlabs.ai/docs/installation |
| 아키텍처 | https://docs.headroomlabs.ai/docs/architecture |
| 프록시 | https://docs.headroomlabs.ai/docs/proxy |
| MCP 툴 | https://docs.headroomlabs.ai/docs/mcp |
| CCR (가역 압축) | https://docs.headroomlabs.ai/docs/ccr |
| 압축 원리 | https://docs.headroomlabs.ai/docs/how-compression-works |
| SmartCrusher | https://docs.headroomlabs.ai/docs/smart-crusher |
| 코드 압축 | https://docs.headroomlabs.ai/docs/code-compression |
| 메모리 | https://docs.headroomlabs.ai/docs/memory |
| SharedContext | https://docs.headroomlabs.ai/docs/shared-context |
| 실패 학습 | https://docs.headroomlabs.ai/docs/failure-learning |
| 캐시 최적화 | https://docs.headroomlabs.ai/docs/cache-optimization |
| 출력 토큰 절감 | https://docs.headroomlabs.ai/docs/savings |
| 벤치마크 방법론 | https://docs.headroomlabs.ai/docs/benchmarks |
| 설정 | https://docs.headroomlabs.ai/docs/configuration |
| 한계 | https://docs.headroomlabs.ai/docs/limitations |
| 트러블슈팅 | https://docs.headroomlabs.ai/docs/troubleshooting |
| 영속 설치 | https://docs.headroomlabs.ai/docs/persistent-installs |
| VS Code Copilot | https://docs.headroomlabs.ai/docs/vscode-copilot |
| VS Code Claude Code | https://docs.headroomlabs.ai/docs/vscode-claude-code |
| LLM용 인덱스 | https://docs.headroomlabs.ai/llms.txt |
| LLM용 전체 문서 | https://docs.headroomlabs.ai/llms-full.txt |

### 커뮤니티 / 연계

| 구분 | URL |
|---|---|
| Discord | https://discord.gg/yRmaUNpsPJ |
| Trendshift | https://trendshift.io/repositories/20881 |
| 기업 문의 | hello@headroomlabs.ai |
| Serena (권장 동반 도구) | https://github.com/oraios/serena |
| 서드파티 상태줄 플러그인 | https://github.com/Ship-Wright/headroom-plugin |

### 레포 내 주요 파일 (학습용)

| 파일 | 왜 봐야 하는가 |
|---|---|
| `headroom/pipeline.py` | 11단계 파이프라인 라이프사이클 |
| `headroom/providers/registry.py` | 멀티 프로바이더 추상화 |
| `headroom/proxy/auth_mode.py` | 인증 모드 분류기 (10µs 목표) |
| `headroom/transforms/smart_crusher.py` | 통계 기반 JSON 압축 |
| `headroom/transforms/cache_aligner.py` | KV 캐시 보호 |
| `crates/headroom-proxy/src/cache_stabilization/` | 캐시 안정화 7개 모듈 |
| `crates/headroom-proxy/src/sse/` | SSE 스트리밍 프록시 |
| `crates/headroom-proxy/src/bedrock/sigv4.rs` | AWS SigV4 직접 구현 |
| `crates/headroom-parity/` | Python↔Rust 동등성 검증 |
| `benchmarks/index_proof_table.py` | 재현 가능한 벤치마크 |
| `benchmarks/adversarial_ccr_tests.py` | 적대적 테스트 |
| `examples/grafana/headroom-dashboard.json` | Grafana 대시보드 샘플 |
| `sql/create_dashboard_summary.sql` | 집계 쿼리 |
| `server.json` | MCP 레지스트리 canonical 정의 |
| `.claude-plugin/marketplace.json` | Claude Code 플러그인 정의 |

### 비교 도구

| 도구 | 범위 | 배포 | 로컬 | 가역 |
|---|---|---|:---:|:---:|
| **Headroom** | 모든 컨텍스트 (툴·RAG·로그·파일·이력) | 프록시·라이브러리·미들웨어·MCP | ✅ | ✅ |
| [Compresr](https://compresr.ai) | 자사 API로 보낸 텍스트 | 호스팅 API | ❌ | ❌ |
| [Token Co.](https://thetokencompany.ai) | 자사 API로 보낸 텍스트 | 호스팅 API | ❌ | ❌ |
| OpenAI Compaction | 대화 이력 | 공급자 네이티브 | ❌ | ❌ |

---

## 부록 — 핵심 요약 카드

```
┌────────────────────────────────────────────────────────────┐
│  HEADROOM 한 장 요약                                        │
├────────────────────────────────────────────────────────────┤
│  무엇    AI 에이전트 컨텍스트를 로컬에서 압축하는 미들웨어   │
│  효과    토큰 30~60% (반복 데이터는 90%+) 절감               │
│  정확도  GSM8K 동일(0.870), BFCL 툴 호출 97% 유지            │
│  속도    0.21ms (10K 토큰) — 체감 불가                       │
│  설치    pip install "headroom-ai[all]"                     │
│  사용    headroom wrap claude                               │
│  확인    headroom dashboard / headroom savings              │
│  정체    CLI + 프록시 + 라이브러리 + MCP + 플러그인          │
│  토큰    Headroom 자체 토큰 불필요 (기존 LLM 키 passthrough) │
│  라이선스 Apache 2.0 (상업적 이용·SaaS 가능, 상표는 불가)     │
│  주의    Beta · AVX2 필요 · 텔레메트리 기본값 불명확 → off   │
│  수익화  콘텐츠 → 컨설팅 → SaaS → 온프레미스 순서 권장        │
└────────────────────────────────────────────────────────────┘
```

---

*이 리포트는 레포지토리 `0.37.0` 시점의 소스 코드와 문서를 직접 읽고 작성되었습니다.*
*성능 수치는 레포 내 벤치마크 및 README 기재값이며, 실제 절감률은 워크로드에 따라 크게 달라집니다 — `headroom savings`로 자신의 트래픽을 실측하세요.*
