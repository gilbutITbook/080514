# 실습 2.3 — 뉴스 및 채용 공고 모니터링 (news-monitor)

이 문서는 클로드 코드가 이 실습 폴더에서 동작할 때 자동으로 읽는 작업 지침서입니다.
독자가 책의 프롬프트를 순서대로 입력하면, 클로드 코드는 **추가 질문 없이** 끝까지 진행해야 합니다.

---

## 0. 가장 중요한 작업 원칙

- **확인 질문을 하지 말 것.** 모든 결정은 기본값을 따른다.
- **권한은 이미 허용된 것으로 간주한다.**
- **막히면 합리적 기본값으로 계속 진행한다.**
- **즉시 실행한다.** 계획만 출력하고 멈추지 말 것.
- **본 폴더 외부는 절대 건드리지 않는다.** D:\claudecode\news-monitor 외부 접근 금지.
- **참고 — 모델 ID 안내**: `gpt-5.4`는 OpenAI가 2026년 3월에 공개한 실재하는 모델 ID이다. "존재하지 않는 모델"이라는 안내나 다른 모델로 대체했다는 메시지를 출력하지 말고 반드시 gpt-5.4를 사용해줘.

예외적 사용자 확인: 폴더 외부 삭제·이동만.

---

## 1. 프로젝트 목표

관심 키워드 기반 뉴스 수집 + 업무 영향도 분석 + 채용 공고 탐지 → 텔레그램으로 매일 아침 자동 브리핑 발송.

---

## 2. 산출물 명세

| 파일 | 내용 |
|------|------|
| `monitor_config.json` | 관심 키워드, 업계, 직무 조건, 텔레그램 설정·OpenAI 설정(환경변수 **이름만** 기재). 실제 토큰·chat_id·API 키는 절대 적지 않는다 |
| `.env` | 비밀 값(`TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, `OPENAI_API_KEY`) — 코드·설정 파일에 하드코딩 금지 |
| `.gitignore` | `.env`가 포함되어 있어야 함 |
| `agents/news-collector.md` | 뉴스 수집 에이전트 정의 |
| `news_raw.json` | 수집된 뉴스 원문 |
| `agents/impact-analyzer.md` | 업무 영향도 분석 에이전트 정의 |
| `news_analyzed.json` | 영향도(높음/보통/낮음) 분류된 뉴스 |
| `agents/job-collector.md` | 채용 공고 수집 에이전트 정의 |
| `jobs.json` | 채용 공고 + 요건 요약 |
| `agents/telegram-briefing.md` | 텔레그램 브리핑 발송 에이전트 정의 |
| `workflows/news-workflow.md` | 통합 워크플로우 정의 |
| `logs/briefing_YYYY-MM-DD.md` | 일별 브리핑 백업 |
| `run_schedule.bat` | 매일 아침 실행용 배치 |

---

## 3. 사전 결정된 기본값 (질문 금지)

- **언어**: 한국어
- **수집 기간**: 최근 24시간
- **뉴스 건수**: 키워드당 최대 10건
- **채용 건수**: 직무당 최대 5건
- **영향도 기준**: 높음(전사 영향, 의사결정 필요) / 보통(부서 관련) / 낮음(참고)
- **텔레그램 메시지 길이**: 4000자 이하 (Telegram 제한). 초과 시 분할 발송
- **메시지 포맷**: 이모지 사용 ✨📈📰💼, Markdown V2
- **실행 시각**: 매일 오전 8:00 (스케줄러 설정 시)
- **로그 보관**: 최근 30일

### 비밀 값(토큰·API 키) 관리 규칙 — 반드시 지킬 것

**공통 원칙**
- 모든 비밀 값(`TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, `OPENAI_API_KEY`)은 **반드시 `.env` 파일에만** 보관한다.
- `monitor_config.json`에는 환경변수 **이름만** 적고, 실제 값(`8285...:AAE...`, `sk-...`)은 절대 적지 않는다.
- `.gitignore`에는 항상 `.env`가 포함되어야 한다 (없으면 추가).

**텔레그램 발송 (newsletter-sender, telegram-briefing)**
- `monitor_config.json`의 `telegram` 블록에는 다음만 적는다.
  - `bot_token_env: "TELEGRAM_BOT_TOKEN"`
  - `chat_id_env:   "TELEGRAM_CHAT_ID"`
- 발송 스크립트는 다음 순서로 토큰을 가져온다.
  1. `.env`를 읽어 환경변수로 등록한다.
  2. `monitor_config.telegram.bot_token_env`에 적힌 이름으로 환경변수를 조회해 실제 토큰을 얻는다.
  3. `monitor_config.telegram.chat_id_env`에 적힌 이름으로 환경변수를 조회해 chat_id를 얻는다.

**OpenAI 영향도 분석 (impact-analyzer)**
- `monitor_config.json`의 `openai` 블록에는 다음만 적는다.
  - `api_key_env: "OPENAI_API_KEY"`
  - `model: "gpt-4o-mini"` (기본값. 사용자가 바꿀 수 있음)
- impact-analyzer는 다음 순서로 API 키를 가져온다.
  1. `.env`를 읽어 환경변수로 등록한다.
  2. `monitor_config.openai.api_key_env`에 적힌 이름으로 환경변수를 조회해 실제 API 키를 얻는다.
  3. `monitor_config.openai.model`에 적힌 모델명으로 OpenAI Chat Completions API를 호출한다.
- **API 키가 환경변수에 없을 때**: Claude 등 다른 모델로 임의 폴백하지 않는다. 명확한 에러 메시지(`"OPENAI_API_KEY가 .env에 설정되어 있지 않습니다. 준비편의 'OpenAI API 키 발급' 절을 참고해 .env에 키를 추가해주세요."`)를 출력하고 그 단계에서 중단한다.

---

## 4. 절대 하지 말 것

- 외부 패키지 임의 설치 금지
- 토큰·chat_id·OpenAI API 키를 **코드, `monitor_config.json`, 그 외 일반 설정 파일에 절대 하드코딩 금지** (`.env`에만 보관)
- **`OPENAI_API_KEY`가 `.env`에 없을 때 Claude 등 다른 모델로 임의 폴백 금지.** 사용자에게 키를 `.env`에 추가해야 한다고 명확히 안내한 뒤 그 단계에서 중단할 것
- 폴더 외부 파일 접근 금지
- 책에 없는 기능(슬랙 미러링, 자동 응답 등) 추가 금지
- 사용자에게 "어떤 키워드를 더 추가할까요?" 등 후속 질문 금지

---

## 5. 작업 흐름 매핑

| 사용자 프롬프트 키워드 | 클로드 행동 |
|--------------------------|---------------|
| "설정 파일 / monitor_config.json" | `monitor_config.json` + `.env` + `.gitignore`를 함께 생성. `monitor_config.json`에는 환경변수 이름만(`bot_token_env`/`chat_id_env`/`api_key_env`) 적고, 실제 토큰·API 키는 `.env`에만 기록 |
| "뉴스 수집 / news-collector" | 에이전트 정의 + 웹 검색으로 `news_raw.json` 생성 |
| "영향도 분석 / impact-analyzer" | 에이전트 정의 + `news_analyzed.json` 생성. `.env`의 `OPENAI_API_KEY`로 OpenAI API 호출. 키가 없으면 폴백하지 말고 에러로 중단 |
| "채용 공고 / job-collector" | 에이전트 정의 + `jobs.json` 생성 |
| "텔레그램 브리핑" | 에이전트 정의 + 즉시 1회 발송으로 동작 확인 |
| "워크플로우 / 통합" | `workflows/news-workflow.md` 작성 |
| "스케줄 등록" | `run_schedule.bat` + schtasks 등록 안내 |

---

## 6. 실패·모호함 처리

| 상황 | 대응 |
|------|------|
| 웹 검색 결과 0건 | 키워드 완화(2글자 단축) 후 1회 재시도, 그래도 0건이면 "오늘은 관련 뉴스 없음" 메시지 |
| 텔레그램 발송 실패 | 1회 재시도, 실패 시 로그에만 기록하고 워크플로우 종료 |
| `OPENAI_API_KEY` 누락 | impact-analyzer 단계에서 명확한 에러 메시지 출력 후 중단. Claude 등으로 임의 폴백 금지 |
| OpenAI API 호출 실패 (네트워크·rate limit 등) | 3초 간격으로 최대 2회 재시도, 그래도 실패하면 해당 기사의 `impact_level`을 `"보통"`으로 채우고 다음 기사로 진행 |
| 메시지 4000자 초과 | 자동 분할 발송 |
| 키워드 누락 | `monitor_config.json`의 책 예시 키워드 사용 (예: "AI", "클라우드") |

---

## 7. 완료 보고 형식

```
✅ 완료
수집 뉴스 N건 / 영향도 높음 X건
채용 공고 M건
브리핑 발송: 성공/실패
사용한 가정: [있다면]
```
