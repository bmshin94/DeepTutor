# DeepTutor 전수조사 분석 정리 (한국어)

> 이 문서는 DeepTutor 저장소를 전수조사하고 분석한 내용을 한국어로 정리한 자료입니다.
> 설치·사용법, 기술 구조, 활용 방안, 수익화 아이디어까지 담았습니다.

## 📎 관련 링크

| 구분 | 주소 |
|:---|:---|
| **이 저장소 (포크)** | https://github.com/bmshin94/DeepTutor |
| **원본 저장소 (upstream)** | https://github.com/HKUDS/DeepTutor |
| 공식 문서 | https://deeptutor.info |
| 논문 (arXiv) | https://arxiv.org/abs/2604.26962 |
| EduHub (스킬 허브) | https://eduhub.deeptutor.info |
| ClawHub (호환 허브) | https://clawhub.ai |
| Docker 이미지 | `ghcr.io/hkuds/deeptutor:latest` |
| Discord | https://discord.gg/eRsjPgMU4t |
| 로드맵 이슈 | https://github.com/HKUDS/DeepTutor/issues/498 |

- **분석 기준 버전:** v1.6.13 (2026-10-04 릴리즈)
- **라이선스:** Apache 2.0 (상업적 이용 가능)
- **개발 주체:** HKUDS (홍콩대학교 Data Intelligence Lab)
- **작성일:** 2026-10-07

---

## 1. 한 줄 요약

> **내 교재·논문·영상을 통째로 학습시켜, 나만을 위한 AI 과외 선생님을 내 컴퓨터에 설치하는 오픈소스 학습 플랫폼**

일반 챗봇과의 차이는 세 가지입니다.

1. **내 자료를 안다** — 지식베이스(RAG)로 내 PDF/EPUB/유튜브를 근거로 답변
2. **나를 기억한다** — L1/L2/L3 3계층 메모리로 약점과 선호를 누적
3. **스스로 도구를 쓴다** — 90여 개 도구를 에이전트 루프에서 자율 호출

---

## 2. 저장소 규모

| 항목 | 수치 |
|:---|---:|
| Python 파일 / 라인 | 1,818개 / 약 469,000줄 |
| TypeScript·TSX 파일 / 라인 | 1,237개 / 약 285,000줄 |
| 전체 | 약 75만 줄 |
| 지원 언어(UI) | 12개 (**한국어 미지원**) |
| GitHub 스타 | 40,000+ (9개월) |

토이 프로젝트가 아니라 제품급 모놀리식 플랫폼입니다.

---

## 3. 아키텍처

```
┌─────────────────────────────────────────────────┐
│  web/   Next.js 16 + React 19  (포트 3782)       │
│  브라우저는 이 오리진만 바라봄                     │
│  web/proxy.ts 가 /api/*, /ws/* 를 백엔드로 중계    │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│  deeptutor/api/   FastAPI  (포트 8001)           │
│  REST + WebSocket 스트리밍                       │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│  deeptutor/   핵심 엔진 (Python)                  │
│  agents · capabilities · tools · services        │
└─────────────────────────────────────────────────┘

  + deeptutor CLI — 웹 없이 전 기능 사용 / 타 에이전트가 조종 가능
```

### 폴더별 역할

| 폴더 | 파일 수 | 역할 |
|:---|---:|:---|
| `deeptutor/services/` | 523 | LLM·임베딩·RAG·검색·파싱·메모리·샌드박스·MCP·음성·이미지 등 모든 외부 연동 |
| `deeptutor/agents/` | 106 | 에이전트 루프 본체 (chat, loop, research, question, visualize, math_animator, vision_solver, notebook) |
| `deeptutor/capabilities/` | 102 | 학습 모드 플러그인 (reading, watching, mastery, course_study, solve, ask_questions, audio_overview, subagent, obsidian, ima, marginnote4) |
| `deeptutor/tools/` | 90 | 모델이 호출하는 도구 (rag, web_search, web_fetch, exec, paper_search, zotero, github, ask_user, imagegen/videogen, question_bank, cron) |
| `deeptutor/book/` | 72 | Book Engine — 내 자료로 "살아있는 교재" 컴파일 |
| `deeptutor/api/` | 56 | FastAPI 라우터 + 응답 계약(contracts) |
| `deeptutor/runtime/` | 51 | 턴 실행 런타임 (서버 재시작 후 세션 복구) |
| `deeptutor/learning/` | 46 | 학습 진도·숙달도·일일 연습 |
| `deeptutor/partners/` | 32 | IM 채널 봇 (15개 채널) |
| `deeptutor/reading/` | 23 | PDF/EPUB 몰입 독서 + 주석 |
| `deeptutor/multi_user/` | 20 | 다중 사용자 배포, 계정 격리, 관리자 권한 |
| `web/` | 1,237 | Next.js 프론트엔드 |

---

## 4. 핵심 개념 5가지

### 4.1 Capability (학습 모드)

하나의 에이전트 런타임 위에 목적별 루프를 올린 구조입니다.

| 모드 | 설명 |
|:---|:---|
| `chat` | 기본 대화 |
| `deep_solve` | 단계별 문제 풀이 |
| `deep_question` | 퀴즈·문제 자동 출제 |
| `deep_research` | 심층 리서치 → 리포트 생성 |
| `visualize` | 차트·SVG·인터랙티브 HTML |
| `math_animator` | Manim 기반 수학 애니메이션 |
| `mastery_path` | 숙달 게이트 (통과 전 진행 불가) |
| `immersive_reading` | 문서를 옆에 띄우고 페이지 인용 |
| `immersive_watching` | 유튜브 자막 동기화 + 타임스탬프 과외 |
| `course_study` | 코스 단위 학습 |
| `audio_overview` | 오디오 요약 |

### 4.2 Knowledge Base (RAG)

교체 가능한 RAG 엔진 **10종**:
`llamaindex`, `pageindex`, `graphrag`, `lightrag`, `lightrag_server`,
`weknora`, `ima`(텐센트), `kiwix`(오프라인 위키), `obsidian`, `marginnote4`

- 문서 파싱 엔진도 선택식: MinerU / Docling / Apache Tika / PyMuPDF4LLM / LiteParse
- 벡터스토어 FAISS 지원 (대용량 가속)
- `deeptutor kb eval` 로 Recall@k / Precision@k / nDCG@k / MRR / MAP / Hit@k 측정

### 4.3 Memory 3계층

```
L1 (trace)      원시 대화 로그
   ↓ 요약
L2 (surface)    표면별 요약 (chat, profile, reading ...)
   ↓ 종합
L3 (synthesis)  전역 종합 (학습자 특성)
```

Memory Graph가 L2 사실 ↔ L1 증거 ↔ L3 종합을 연결해,
**왜 그렇게 판단했는지 추적하고 직접 수정**할 수 있습니다.

### 4.4 Subagents & Partners

- **Subagents** — 채팅 중 로컬 코딩 CLI 실시간 호출
  (Claude Code, Codex, Grok CLI, Antigravity, Kimi, opencode, MiMo, Hermes, OpenClaw, DeepSeek)
- **Partners** — DeepTutor 두뇌를 IM 채널(15종)에 상주시키는 봇

### 4.5 Skills & EduHub

- 포맷: 개방형 **Agent-Skills** (`SKILL.md` = YAML frontmatter + Markdown)
- 기본 허브 **EduHub**, **ClawHub** 호환, 사용자 정의 허브 추가 가능
- 임포트 보안 게이트:
  레지스트리 보안 판정 확인 → 압축 해제 방어(path traversal·엔트리 수·크기·압축비·심볼릭링크) →
  실행 권한 제거 → `always:` 강제 주입 차단 → `.hub-lock.json`에 출처 기록

---

## 5. 보안 설계

- **샌드박스 3단 폴백**: Runner 사이드카(`Dockerfile.runner`) → Linux `bubblewrap` → 제한된 subprocess
- **워크스페이스 경계**: `outputs/` 밖은 읽기 전용, 파일 복사는 경로 조합별 "Allow once" 명시 승인
- **Content Workspace ↔ Runtime Home 분리**: 에이전트가 보는 폴더와 자격증명 폴더를 물리적으로 분리
- 정적 점검: `detect-secrets`(`.secrets.baseline`), `import-linter`(`.importlinter`), `pre-commit`

---

## 6. 설치 및 사용법

### 6.1 Docker (권장 — 가장 간단)

```bash
docker run --rm --name deeptutor \
  -p 127.0.0.1:3782:3782 \
  -v deeptutor-data:/app/data \
  ghcr.io/hkuds/deeptutor:latest
```

→ 브라우저에서 http://127.0.0.1:3782 접속. `Ctrl+C`로 종료.
→ 데이터는 `deeptutor-data` 볼륨에 보존.

호스트의 Ollama 등 로컬 모델을 쓰려면:

```bash
docker run --rm -p 127.0.0.1:3782:3782 \
  --add-host=host.docker.internal:host-gateway \
  -v deeptutor-data:/app/data \
  ghcr.io/hkuds/deeptutor:latest
# Settings → Providers → Base URL: http://host.docker.internal:11434/v1
```

### 6.2 PyPI 설치

요구: Python 3.11–3.14, Node.js 20+

```bash
mkdir my-deeptutor && cd my-deeptutor
pip install -U deeptutor
deeptutor init      # 포트 → LLM → 임베딩 → 검색 마법사
deeptutor start
```

### 6.3 소스 설치 (개발용)

```bash
git clone https://github.com/bmshin94/DeepTutor.git
cd DeepTutor
python3 -m venv .venv && source .venv/bin/activate
pip install -e .
( cd web && npm ci --legacy-peer-deps )
deeptutor init
deeptutor start --dev     # HMR
```

선택 extras: `.[rag-lightrag]`, `.[graphrag]`, `.[dev]`, `.[partners]`,
`.[matrix]`, `.[matrix-e2e]`, `.[math-animator]`, `.[video-learning]`

### 6.4 CLI 전용

```bash
pip install -e ./packaging/deeptutor-cli
deeptutor init --cli
deeptutor chat
```

### 6.5 자주 쓰는 명령어

```bash
deeptutor doctor --online                 # 런타임 진단
deeptutor kb create physics --doc ch1.pdf
deeptutor kb search physics "관성"
deeptutor kb eval physics --dataset qa.jsonl
deeptutor run deep_question "열역학" --kb physics --config num_questions=5
deeptutor run deep_research "RAG 2026" --format json   # NDJSON 스트림
deeptutor memory show L3
deeptutor plugin list
deeptutor workspace set /absolute/path
deeptutor skill search "socratic tutor"
deeptutor skill install eduhub:socratic-tutor@1.2.0
```

---

## 7. 플러그인? 스킬? MCP? — 정체 정리

**결론: 셋 다 아니고 "애플리케이션(플랫폼)". 다만 셋 모두를 품고 있음.**

| 구분 | DeepTutor의 위치 | 설명 |
|:---|:---|:---|
| 본체 | **독립 애플리케이션** | FastAPI + Next.js + CLI로 혼자 돌아가는 완성 제품 |
| Skill | **제공 + 소비** | 루트 `SKILL.md`로 타 AI에 자신을 노출 / EduHub·ClawHub에서 스킬 설치 |
| MCP | **클라이언트(호스트)만** | `deeptutor/services/mcp/`로 외부 MCP 서버 연결. **자신을 MCP 서버로 노출하지는 않음** |
| 플러그인 | **플러그인 호스트** | Tools/Capabilities 플러그인 구조, 서드파티 확장 지원 |

```
[Claude Code / Codex 등 AI 에이전트]
          │  SKILL.md 읽고 CLI 호출
          ▼
   ┌──────────────────────┐
   │   DeepTutor (본체)    │
   │  Tools / Capabilities │
   │  Skills (EduHub 등)   │
   └──────┬───────────────┘
          │  MCP 클라이언트
          ▼
   [외부 MCP 서버들]
```

> **틈새**: DeepTutor를 MCP **서버**로 래핑하면 Claude Desktop 등에서 바로 쓸 수 있음. 공식 미지원 → 선점 가능.

---

## 8. API 토큰이 꼭 필요한가?

**LLM은 반드시 필요하지만, 유료 토큰은 선택입니다.**

| 구성요소 | 필수 여부 | 무료 대안 |
|:---|:---:|:---|
| LLM | 필수 | Ollama / LM Studio / llama.cpp / vLLM / Lemonade (로컬 무료) |
| Embedding | KB 사용 시 | Ollama 임베딩 (`/api/embed`) |
| Web Search | 선택 | SearXNG 자체 호스팅 |
| TTS / STT | 선택 | 비활성화 가능 |
| Image / Video 생성 | 선택 | 비활성화 가능 |
| GitHub / Zotero / MCP | 선택 | 필요 시에만 |

### 비용 시나리오

| 플랜 | 구성 | 월 비용 | 비고 |
|:---|:---|:---|:---|
| 완전 로컬 | Ollama + Ollama 임베딩 + SearXNG | 0원 | GPU VRAM 12GB+ 권장, 품질 낮음 |
| **하이브리드(권장)** | 유료 LLM + 로컬 임베딩 + SearXNG | 1~3만원 | 가성비 최적 |
| 풀 클라우드 | 전부 유료 API | 5~20만원 | 사용량 비례 |

### 키 보관

- `data/user/settings/model_catalog.json`에 저장, 샌드박스에서 격리되어 생성 코드가 접근 불가
- `deeptutor provider login openai-codex` — OAuth 로그인(ChatGPT 구독 활용) 지원
- Keypool로 다중 키 로테이션 가능
- `.env.example`은 **포트 설정용**이지 API 키용이 아님

---

## 9. AI 에이전트 구축 레퍼런스로서의 가치

이 저장소의 최대 가치는 **프로덕션급 에이전트 구현 레퍼런스**라는 점입니다.

| 학습 주제 | 참고 위치 |
|:---|:---|
| 에이전트 루프 | `deeptutor/agents/loop/` |
| 도구 스키마 설계 (90개 예시) | `deeptutor/tools/` |
| 멀티 LLM 프로바이더 추상화 | `deeptutor/services/llm/provider_core/` |
| 장기 메모리 3계층 | `deeptutor/services/memory/` |
| RAG 엔진 10종 비교 구현 | `deeptutor/services/rag/pipelines/` |
| 코드 실행 샌드박스 | `deeptutor/services/sandbox/` |
| MCP 클라이언트 (OAuth·시크릿 분리) | `deeptutor/services/mcp/` |
| 스트리밍 UI (WS → React) | `web/components/chat/` |
| 서브에이전트 통합 | `deeptutor/capabilities/subagent/` |
| 크래시 복구형 턴 런타임 | `deeptutor/runtime/` |

### 실전 활용 패턴

```bash
# (1) 백엔드만 띄워 내 프론트에서 호출
deeptutor serve --port 8001

# (2) 다른 에이전트의 도구로 사용 (NDJSON)
deeptutor run deep_research "RAG 2026 서베이" --format json

# (3) 세션 체이닝으로 상태 유지
SID=$(deeptutor run deep_research "주제" --format json \
      | jq -r 'select(.type=="done").session_id')
deeptutor run deep_question "방금 내용 퀴즈" --session "$SID" --format json
```

### 추천 학습 순서

1. `deeptutor/agents/loop/` — 루프 원리
2. `deeptutor/tools/` 중 10개 — 도구 설계
3. `deeptutor/services/memory/` — 메모리 3계층
4. `deeptutor/services/rag/pipelines/` — RAG 심화
5. `deeptutor/services/sandbox/`, `services/mcp/` — 보안·확장

---

## 10. React / PHP로 만들 수 있나?

### React — 이미 React 기반

`web/`이 Next.js 16 + React 19 + TypeScript입니다.

| 작업 | 난이도 | 비고 |
|:---|:---:|:---|
| 테마·디자인 커스텀 | 쉬움 | `globals.css`, `shared/ui/` |
| 한국어 i18n 추가 | 쉬움 | `deeptutor/i18n/` + 웹 i18n |
| 새 페이지 추가 | 보통 | `web/app/(workspace)/` |
| 프론트 전면 재작성 | 보통 | 백엔드 유지, API 스키마는 `deeptutor/api/contracts/` 참고 |
| 모바일 앱(RN) | 어려움 | API 재사용, UI 신규 |

### PHP — 프론트/게이트웨이는 적합, 코어 포팅은 부적합

| 시도 | 평가 |
|:---|:---|
| Laravel로 프론트 + 회원·결제 + DeepTutor API 호출 | **권장** |
| 워드프레스 플러그인으로 연동 | 권장 |
| 멀티테넌시 게이트웨이 | 실용적 |
| 코어 전체를 PHP로 포팅 | **비권장** |

비권장 이유: 47만 줄 규모, `llama-index`·`faiss`·`PyMuPDF`·`Manim` 등 Python 전용 의존성,
주 1회 수준의 업스트림 릴리즈 속도, async/스트리밍 패턴 불일치.

### 권장 아키텍처

```
┌──────────────────────────────────────┐
│ PHP(Laravel) 또는 Next.js — 직접 개발  │
│ 랜딩·회원가입·결제·사용량 제한·관리자    │
└───────────────┬──────────────────────┘
                │ HTTP / WebSocket
┌───────────────▼──────────────────────┐
│ DeepTutor API (그대로 사용)           │
│ deeptutor serve --port 8001          │
└──────────────────────────────────────┘
```

**"껍데기는 직접, 엔진은 DeepTutor"** — 최소 노력 최대 효과.

---

## 11. 유튜브 강의 제작 가능성

### 제작 타당성

| 근거 | 내용 |
|:---|:---|
| 화제성 | 40k 스타, GitHub Trending 1위 |
| 경쟁 공백 | **한국어 콘텐츠 거의 없음** |
| 소재량 | 캐파 11종 × 모드별 → 영상 30편 이상 가능 |
| 타겟 | 학생·수험생(사용법) + 개발자(아키텍처) |
| 수익 연결 | 영상 → 강의 → 컨설팅 → SaaS |
| 법적 안전 | Apache 2.0 |

### 커리큘럼 설계안

**A. 입문 트랙 (일반·학생)**

| # | 제목 | 길이 |
|:--|:---|:--|
| EP1 | "ChatGPT는 내 교재를 모른다" — 왜 DeepTutor인가 | 5분 |
| EP2 | Docker 한 줄로 10분 설치 | 10분 |
| EP3 | 무료로 쓰기 — Ollama 연동 | 12분 |
| EP4 | 전공책 PDF 통째로 먹이기 | 15분 |
| EP5 | 유튜브 강의 링크로 공부하기 | 10분 |
| EP6 | 시험 2주 전, 자동 문제은행 | 15분 |
| EP7 | Mastery Path — 못 풀면 못 넘어감 | 12분 |
| EP8 | 수학 애니메이션 자동 생성 | 15분 |
| EP9 | 디스코드에 내 과외쌤 봇 띄우기 | 18분 |
| EP10 | Obsidian 노트 연결 | 12분 |

**B. 개발자 트랙 (코드 리딩)**

| # | 제목 | 길이 |
|:--|:---|:--|
| EP11 | 75만 줄 아키텍처 투어 | 20분 |
| EP12 | Agent Loop 코드 뜯어보기 | 25분 |
| EP13 | 도구(Tool) 직접 만들어 붙이기 | 25분 |
| EP14 | RAG 엔진 10개 비교 | 30분 |
| EP15 | 메모리 L1/L2/L3 설계 분석 | 22분 |
| EP16 | 샌드박스 — 코드 실행 안전하게 | 20분 |
| EP17 | MCP 서버 붙이기 | 18분 |
| EP18 | 한국어 i18n 기여 — 첫 PR | 20분 |

**C. 수익화 트랙**

| # | 제목 | 길이 |
|:--|:---|:--|
| EP19 | DeepTutor로 학원 SaaS 만들기 | 30분 |
| EP20 | Laravel + DeepTutor 연동 | 25분 |
| EP21 | EduHub에 내 스킬 퍼블리시 | 15분 |

### 제작 시 주의

- 주 1회 릴리즈 → 설치 영상은 빠르게 낡음. 개념 영상 위주 + 설치편 주기 갱신
- 공식 문서 스크린샷이 v1.4.6 기준 → 직접 캡처 필요
- **API 키 화면 노출 주의** (모자이크 필수)

---

## 12. 수익화 아이디어

> 전제: Apache 2.0 — 상업적 이용·수정·재배포 가능.
> 의무: 저작권 고지 유지, 라이선스 사본 포함, 변경사항 명시, NOTICE 유지.
> 상표는 별개이므로 "DeepTutor"를 제품명으로 쓰기보다 "Powered by DeepTutor" 표기 권장.

### TIER 1 — 즉시 실행 가능 (저비용·단독)

| # | 아이디어 | 수익 모델 | 비용 | 기간 | 난이도 |
|:--|:---|:---|:---|:---|:---|
| 1 | **한국어화 + 한국형 배포판** | 설치 대행(건당 10~30만), 후원, 인지도 → 타 상품 유입 | 거의 0 | 2~4주 | 하 |
| 2 | **유튜브 + 온라인 강의** | 애드센스, 인프런/클래스101 강의, 협찬, 유입 전환 | 장비+시간 | 3개월 | 하 |
| 3 | **EduHub 스킬 퍼블리싱** | 포트폴리오·브랜딩, 향후 유료 스킬 | 0 | 수일 | 하 |
| 4 | **MCP 서버 래퍼 개발** | 오픈소스 명성 → 컨설팅, GitHub Sponsors | 0 | 2~3주 | 중 |

> 1번이 최우선. 기여 실적이 이후 모든 수익 활동의 신뢰 기반이 됩니다.
> 4번은 공식 미지원 영역이라 선점 가치가 큽니다.

**스킬 아이디어 예시**: 수능 국어 지문분석기, 토익 LC 스크립트 훈련,
한국사 연표 암기, 코딩테스트 단계별 힌트

### TIER 2 — 중기 (수익 본격화)

| # | 아이디어 | 가격 | 목표 | 기간 | 난이도 |
|:--|:---|:---|:---|:---|:---|
| 5 | **학원·과외 SaaS** | 학원당 월 10~30만 | 10곳 = 월 100~300만 / 50곳 = 월 500~1,500만 | 3~6개월 | 상 |
| 6 | **기업 사내교육 구축 컨설팅** | 구축 500~3,000만 + 유지 월 50~200만 | 분기 1~2건 | 1~3개월/건 | 상 |
| 7 | **워드프레스 플러그인** | $49~199 또는 연간 라이선스 | 글로벌 교육 사이트 | 1~2개월 | 중 |
| 8 | **관리형 호스팅(Managed)** | 무료/월 9,900/월 29,900 | 구독 전환 | 3~6개월 | 상 |

- **5번이 가장 유망**: 학원은 이미 교재를 보유 → KB에 바로 투입 → "우리 교재로 24시간 조교"라는 즉시 가치
- **6번 셀링포인트**: 온프레미스 로컬 구동 = 자료 외부 유출 없음 (제조·금융·의료에 강력)
- **8번 리스크**: 업스트림이 직접 SaaS를 낼 가능성 → 한국 특화(한국어, 수능/공무원/자격증 템플릿)로 차별화 필수

### TIER 3 — 장기 (규모 확대)

| # | 아이디어 | 설명 | 가격 |
|:--|:---|:---|:---|
| 9 | **버티컬 특화 제품** | 공무원 시험 / 코딩테스트 / 법률 자격증 / 의료 국시 / 어학 중 하나에 집중 | B2C 월 19,900~49,900 |
| 10 | **교육청·대학 공공 조달** | 오픈소스 + 온프레미스 = 공공기관 선호 조합 | 건당 수천만~수억 |
| 11 | **AI 에이전트 개발 교육사업** | DeepTutor를 교보재로 활용한 부트캠프·기업 출강·유료 멤버십 | 수강 50~150만 / 출강 일 100~300만 |

> 9번 근거: 범용 튜터는 ChatGPT와 경쟁하기 어렵지만, 버티컬은 범용 모델이 따라오기 어렵습니다.

### 우선순위 매트릭스

```
수익 높음
   ↑
   │  [5.학원SaaS]        [9.버티컬]
   │  [6.기업컨설팅]       [10.공공조달]
   │
   │  [2.유튜브]          [8.관리형호스팅]
   │  [4.MCP래퍼]         [7.WP플러그인]
   │  [11.교육사업]
   │  [1.한국어화]
   │  [3.스킬퍼블리싱]
   └──────────────────────────────────→ 난이도 높음
```

### 12개월 로드맵

| 기간 | 할 일 |
|:---|:---|
| **1개월차** | Docker 설치·전 기능 체험 / 한국어 i18n PR / 유튜브 EP1~3 |
| **2~3개월차** | 유튜브 EP4~10 / MCP 서버 래퍼 공개 / EduHub 스킬 2~3개 |
| **4~6개월차** | 학원 SaaS MVP / 베타 학원 2~3곳 무료 운영 / 인프런 강의 1개 |
| **7~12개월차** | SaaS 유료 전환(목표 10곳) / 기업 컨설팅 1~2건 / 버티컬 제품 1개 집중 |

### 법적 체크리스트

- [x] `LICENSE`(Apache 2.0) 사본 포함
- [x] `THIRD_PARTY_NOTICES.md` 유지
- [x] 변경한 파일에 변경 사실 명시
- [x] "DeepTutor 기반" / "Powered by DeepTutor" 표기 (상표 회피)
- [ ] 서드파티 의존성 라이선스 확인 (GPL 혼입 여부)
- [ ] 교육 서비스 — 학원법 / 평생교육법 검토
- [ ] 학생 데이터 — 개인정보보호법, 미성년자 보호자 동의
- [ ] API 재판매 — OpenAI / Anthropic 이용약관 확인

---

## 13. 리스크 정리

| 수준 | 리스크 | 내용 | 대응 |
|:---:|:---|:---|:---|
| 높음 | 코드 규모 | 75만 줄 모놀리스 — 일부만 떼어내기 어려움 | API 레벨로 연동 |
| 높음 | API 비용 | 자체 모델 없음, RAG+멀티턴은 토큰 소모 큼 | 하이브리드(로컬 임베딩) |
| 중간 | 릴리즈 속도 | 거의 주 1회 — 포크 유지보수 부담 | upstream 정기 머지 |
| 중간 | 설치 난이도 | Python 3.11~3.14 + Node 20+ 필요 | Docker 경로 안내 |
| 중간 | 문서 스크린샷 | v1.4.6 기준 구버전 | 직접 캡처 |
| 기회 | 한국어 미지원 | 12개 언어 중 한국어 없음 | **i18n 기여 = 선점 기회** |

---

## 14. 결론

1. **학습 도구로서** — 내 교재를 아는 AI 과외 선생님을 로컬에 설치. 데이터 주권 확보.
2. **개발 레퍼런스로서** — 에이전트 루프, 도구 설계, 3계층 메모리, RAG 10종, 샌드박스, MCP를 한 저장소에서 학습 가능. **이 저장소의 최대 가치.**
3. **사업 기반으로서** — Apache 2.0으로 상업화 합법. 한국어 공백 + 학원 시장 + 온프레미스 수요가 명확한 진입점.

**권장 시작점:** Docker 설치 → 전 기능 체험 → 한국어 i18n 기여 → 유튜브 콘텐츠 → 학원 SaaS MVP

---

*이 문서는 저장소 전수조사 분석 결과를 정리한 자료입니다.*
*분석 기준: DeepTutor v1.6.13 / 2026-10-07*
