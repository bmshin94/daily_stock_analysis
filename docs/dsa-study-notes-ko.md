# DSA 프로젝트 분석 노트 (한국어)

> 이 문서는 `daily_stock_analysis`(DSA) 저장소를 전수조사한 결과와, 활용·수익화 방향을 정리한 개인 학습 노트입니다.
> 투자 자문이 아니며, 어떤 수익도 보장하지 않습니다.

## 0. 저장소 정보

| 항목 | 내용 |
|---|---|
| 원본(upstream) | https://github.com/ZhuLinsen/daily_stock_analysis |
| 작업 저장소(fork) | https://github.com/bmshin94/daily_stock_analysis |
| 라이선스 | MIT (상업적 이용·수정·재배포 가능, 출처 표기 요청됨) |
| 관련 프로젝트 | https://github.com/ZhuLinsen/alphasift / https://github.com/ZhuLinsen/alphaevo |
| 규모 | 파일 약 1,165개 / Python 약 337,000줄 / TS·TSX 약 87,000줄 / 테스트 파일 280개 |
| 문서 언어 | 중국어 중심 (영어·번체 번역 일부 제공) |

## 1. 프로젝트 정체

AI 대규모 언어모델을 사용해 관심 종목(A주·홍콩·미국·일본·한국·대만·ETF)을 매일 자동 분석하고,
"결정 대시보드" 형태의 리포트를 메신저·메일로 푸시하는 풀스택 시스템.

핵심 파이프라인:

```
데이터 수집 -> 기술적 분석 + 뉴스/여론 검색 -> LLM 분석 -> 리포트 생성 -> 알림 푸시
```

오케스트레이터: `src/core/pipeline.py`

## 2. 구조 요약

| 경로 | 역할 | 특징 |
|---|---|---|
| `main.py` | 분석 작업 CLI 엔트리 | 단일 파일 약 73KB |
| `server.py`, `api/` | FastAPI REST 서버 | analysis / agent / portfolio / backtest / screening 등 엔드포인트 그룹 16개 |
| `src/core/` | 파이프라인, 백테스트 엔진, 설정, 거래일 캘린더, 대장주 복기 | 모듈 11개 |
| `src/services/` | 비즈니스 서비스 레이어 | 40개 이상 |
| `src/agent/` | AI 에이전트 엔진 | 서브에이전트 6개, 툴 5종, skills 라우터/스케줄러/숙의/합성 |
| `data_provider/` | 데이터 소스 어댑터 | 19개 + 우선순위 fallback |
| `src/notification_sender/` | 알림 채널 | 14개 |
| `strategies/` | 자연어 전략 YAML | 15개, 코드 작성 없이 전략 추가 가능 |
| `templates/` | Jinja2 리포트 템플릿 | markdown / wechat / brief |
| `apps/dsa-web/` | React + TS + Vite + Tailwind 웹 콘솔 | 페이지 10개, i18n 구조 보유 |
| `apps/dsa-desktop/` | Electron 데스크탑 앱 | 설치 패키지 빌드 포함 |
| `bot/` | 챗봇 접속부 | Discord / DingTalk / Lark, 명령어 11개 |
| `.github/workflows/` | 자동화 | 워크플로우 10개, 핵심은 `00-daily-analysis.yml` |
| `evals/agent_trajectory/` | 에이전트 품질 평가 | golden samples 기반 |
| `docs/` | 문서 | 50개 이상 |
| `.claude/skills/` | 개발 협업 스킬 | analyze-issue / analyze-pr / fix-issue |
| `SKILL.md` (루트) | `stock_analyzer` 스킬 명세 | `analyze_stock` / `analyze_stocks` / `perform_market_review` |

## 3. 설치 및 사용법

### 3.1 GitHub Actions (서버 불필요, 비용 0)

1. 저장소 Fork
2. `Settings > Secrets and variables > Actions`에 등록
   - AI 모델 키 1개 (`GEMINI_API_KEY`, `DEEPSEEK_API_KEY`, `ANSPIRE_API_KEYS` 등)
   - 알림 채널 1개 (`TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` 등)
   - `STOCK_LIST` (예: `005930.KS,000660.KS,AAPL,NVDA`)
3. Actions 활성화 후 `每日股票分析` 워크플로우 수동 실행으로 검증
4. 기본 스케줄은 `cron: '0 10 * * 1-5'`(UTC 10:00 = 베이징 18:00). 한국 장 마감 기준으로 받으려면 cron 수정 필요

### 3.2 로컬 실행

```bash
git clone https://github.com/bmshin94/daily_stock_analysis.git
cd daily_stock_analysis
pip install -r requirements.txt
cp .env.example .env
python main.py --dry-run
python main.py
python main.py --stocks 005930.KS
python main.py --market-review
python main.py --webui          # http://127.0.0.1:8000
python main.py --serve-only
python main.py --schedule
```

### 3.3 Docker / 데스크탑

```bash
cd docker && docker compose up -d

cd apps/dsa-web && npm ci && npm run build
cd ../dsa-desktop && npm install && npm run build
```

## 4. 플러그인 / 스킬 / MCP 구분

독립 애플리케이션이며, 외부 AI가 사용할 수 있는 통로를 3가지 제공한다.

| 통로 | 위치 | 상태 |
|---|---|---|
| Skill | 루트 `SKILL.md` (`name: stock_analyzer`) | 제공 |
| REST API | `api/v1/`, `docs/openclaw-skill-integration.md`, `docs/grok-bot-integration.md` | 제공 |
| Bot | `bot/platforms/` | 제공 |
| MCP 서버 | 없음 | **미구현 — 확장 기회** |

REST API가 이미 완성되어 있어 얇은 MCP 어댑터만 추가하면 Claude Desktop / Cursor 등에서 바로 호출 가능.

## 5. API 토큰 필요 여부

| 구분 | 필수 | 무료 대안 |
|---|:---:|---|
| AI 모델 | 사실상 필수 | Ollama 로컬 모델(키 불필요), Gemini 무료 티어, DeepSeek 저가 |
| 주가 데이터 | 선택 | AkShare / Baostock / YFinance 무료 내장 |
| 뉴스 검색 | 선택(권장) | SearXNG 자체 호스팅 |
| 알림 | 1개 필요 | Telegram 봇, Discord Webhook 무료 |

비용 참고: Ollama + 무료 데이터 + Telegram 조합은 0원 운영 가능. DeepSeek + 종목 10개 기준 월 수천 원 수준.
GitHub Actions 무료 한도(2,000분/월)와 토큰 사용량(`TokenUsagePage`, `src/services/usage.py`)을 함께 확인할 것.

## 6. AI 에이전트 구축에 참고할 패턴

| 패턴 | 참고 파일 |
|---|---|
| 멀티 에이전트 숙의·의견 충돌 조정 | `src/agent/agents/*`, `src/agent/skills/deliberation.py`, `src/agent/disagreement.py` |
| 툴 레지스트리 / 노출 범위 제어 | `src/agent/tools/registry.py`, `src/agent/tool_surface.py` |
| 자연어(YAML) 기반 노코드 확장 | `strategies/*.yaml`, `src/agent/skills/engine.py` |
| LLM 백엔드 추상화 + fallback | `src/llm/backend_factory.py`, `src/llm/backend_registry.py` |
| 스트리밍 이벤트 | `src/agent/stream_events.py`, `docs/agent-stream-events.md` |
| 가드레일 / 환각 교정 | `src/phase_decision_guardrail.py`, `src/agent/risk_override.py`, `src/daily_market_context_guardrail.py` |
| Trajectory 평가 | `evals/agent_trajectory/`, `golden_samples.json` |
| 메모리 / 세션 관리 | `src/agent/memory.py`, `src/agent/conversation.py` |

도메인(주식)은 교체 가능하며, 파이프라인·멀티에이전트·fallback·알림 구조는 다른 도메인에 재사용할 수 있다.

## 7. React / PHP 재구현 가능성

- **React**: 이미 `apps/dsa-web/`가 React + TypeScript 구성이므로 신규 구현 불필요. `src/i18n/`, `src/locales/`에 한국어 로케일 추가로 한글화 가능.
- **PHP**: 웹 UI, API 프록시, 회원·결제·구독 관리는 Laravel로 충분. 다만 데이터 수집(akshare/tushare/yfinance), 지표 계산(pandas/numpy), LLM 에이전트는 Python 의존성이 커서 재구현 비용이 매우 큼.
- **권장 구조**: 분석 엔진은 Python(DSA) 그대로 유지하고, 프런트엔드와 비즈니스 레이어만 원하는 스택으로 감싸는 하이브리드.

```
React 프런트  ->  Laravel / Next.js (회원·결제·과금)  ->  Python DSA (--serve-only)
```

## 8. 유튜브 강의 제작 검토

- 라이선스상 가능(MIT). 영상 설명란에 원본 저장소 출처 표기 권장.
- 리스크: 투자 자문 유사행위 규제, 수익 보장형 표현의 광고 관련 규제. "기술 튜토리얼" 포지셔닝과 면책 고지 필요.
- 차별화 요소: 한국어 콘텐츠 부재, 한국 주식 특화(KRX 어댑터 직접 구현), "AI 에이전트" 키워드 수요.
- 시리즈 구성안
  - 트랙 A(초보): 코딩 없이 자동 브리핑 구축 5편
  - 트랙 B(개발자): AI 에이전트 아키텍처 해부 7편
  - 트랙 C(수익형): 오픈소스를 서비스로 전환하기 4편

## 9. 수익화 아이디어

### 티어 1 — 즉시 시작

| 아이디어 | 내용 | 난이도 |
|---|---|---|
| 한국형 포크 공개 | 한글 i18n + KRX 데이터 어댑터 + 한국 거래일 캘린더. 제휴 링크·스폰서 수익 | 중하 |
| 유튜브 + 유료 강의 | 무료 영상으로 유입, 유료 강의로 전환 | 중 |
| MCP 서버 래퍼 | REST API 위 얇은 MCP 어댑터 배포, upstream PR | 중하 |
| 설치 대행 / 컨설팅 | 기본 셋업·커스텀·기업용 구간별 상품화 | 하 |

### 티어 2 — 제품화

| 아이디어 | 내용 | 비고 |
|---|---|---|
| 구독형 SaaS | 무료/베이직/프로/팀 4단 요금제 | 유사투자자문 규제 사전 검토 필수 |
| 도메인 전환 | 가격 모니터링, 채용 매칭, 부동산 브리핑, 기업 리스크 감시(B2B) | 규제 리스크 낮고 구조 재사용률 높음 |
| B2B 화이트라벨 | 리서치·커뮤니티 사업자에게 엔진 납품 | 규제 책임이 고객 측 라이선스로 이동 |

### 티어 3 — 콘텐츠·간접

전략 YAML 마켓플레이스, 뉴스레터, 기술블로그 연재, upstream 컨트리뷰션(KRX 어댑터·MCP·한국어 i18n).

### 권장 순서

1. MCP 래퍼 제작 및 upstream PR (1~2주)
2. 한국화 포크 공개 (2~4주)
3. 유튜브 트랙 A·B 제작 (1~2개월)
4. 유료 강의 패키징 (2~3개월)
5. 도메인 전환 SaaS 또는 B2B (3~6개월)

핵심 방향: 종목 추천으로 수익을 내는 대신, 시스템 구축 역량 자체를 상품화해 규제 리스크를 낮춘다.

## 10. 현재 저장소에서 발견된 이슈

`scripts/check_ai_assets.py` 실행 결과:

```
[ai-assets] ERROR: CLAUDE.md must be a symlink to AGENTS.md
```

- 원인: 이 fork의 `CLAUDE.md`가 `AGENTS.md` 심볼릭 링크가 아닌 일반 파일로 교체되어 페르소나 가이드가 추가된 상태.
- 영향: CI의 `ai-governance` 잡 실패. 기능 동작에는 영향 없음.
- 선택지
  1. 페르소나 내용을 `~/.claude/CLAUDE.md`(전역) 또는 별도 파일로 이동하고 루트 `CLAUDE.md`를 심볼릭 링크로 복원 (권장)
  2. fork 전용으로 `scripts/check_ai_assets.py` 검사 완화
  3. 개인 fork 범위에서는 CI 실패를 그대로 수용

## 11. 면책

본 문서와 이 저장소는 학습·연구 목적입니다. 투자 판단의 책임은 이용자에게 있으며, 어떤 손실에 대해서도 책임지지 않습니다.
