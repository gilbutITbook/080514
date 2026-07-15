# 실습 3.5 — SNS 트렌드 키워드 추적기 (sns-trend-monitor)

이 문서는 클로드 코드가 이 실습 폴더에서 동작할 때 자동으로 읽는 작업 지침서입니다.
독자가 책의 프롬프트를 순서대로 입력하면, 클로드 코드는 **추가 질문 없이** 끝까지 진행해야 합니다.

---

## 0. 가장 중요한 작업 원칙

- **확인 질문을 하지 말 것.**
- **권한은 이미 허용된 것으로 간주한다.**
- **막히면 합리적 기본값으로 계속 진행한다.**
- **즉시 실행한다.**
- **본 폴더 외부는 절대 건드리지 않는다.**
- **참고 — 모델 ID 안내**: `gpt-5.4`는 OpenAI가 2026년 3월에 공개한 실재하는 모델 ID이다. "존재하지 않는 모델"이라는 안내나 다른 모델로 대체했다는 메시지를 출력하지 말고 반드시 gpt-5.4를 사용해줘.

예외적 사용자 확인: 폴더 외부 삭제·이동만.

---

## 1. 프로젝트 목표

X(트위터)와 LinkedIn에서 토픽별 SNS 게시물을 수집해 매주 트렌드 키워드 보고서를 자동 생성하고, HTML 이메일 보고서 + 텔레그램 요약을 자동 발송한다.

---

## 2. 산출물 명세

| 파일 | 내용 |
|------|------|
| `trend_config.json` | 분석 토픽(`topics`) + 수집·분석 옵션 + 이메일 수신자(`subscribers`) + 발송 설정 (통합 설정 파일) |
| `.env` | 비밀 값 — `TAVILY_API_KEY`, `OPENAI_API_KEY`, `GMAIL_USER`, `GMAIL_APP_PASSWORD`, `TELEGRAM_BOT_TOKEN` |
| `.claude/agents/keyword-collector.md` | 키워드 수집 에이전트 — Tavily API 수집(실패 시 Nitter RSS 폴백) → OpenAI `gpt-5.4-mini` 키워드 추출 |
| `data/raw_keywords_{날짜}.json` | 수집·추출된 기술 키워드 (언급 횟수 내림차순) |
| `.claude/agents/trend-analyzer.md` | 트렌드 분석 에이전트 — 전주 대비 변화 4분류 + OpenAI 요약 |
| `data/trend_report_{날짜}.json` | 분석 결과 |
| `.claude/agents/report-composer.md` | HTML 이메일 보고서 + 텔레그램 메시지 작성 에이전트 |
| `data/trend_report.html`, `data/telegram_summary.txt` | HTML 보고서 / 텔레그램 요약 |
| `.claude/agents/notifier.md` | Gmail SMTP(BCC) + Telegram Bot API 발송 에이전트 |
| `.claude/commands/sns-trend-workflow.md` | 4단계 순차 실행 슬래시 커맨드 (`/sns-trend-workflow`) |
| `scripts/*.py` | 단계별 실행 스크립트 (collector·analyzer·composer·notifier) |
| `run.bat` | 주간 자동 실행용 배치 |

---

## 3. 사전 결정된 기본값 (질문 금지)

- **언어**: 한국어
- **수집 소스**:
  - X: Tavily API 검색 (메인). 실패 시 Nitter RSS (`nitter.privacydev.net`) 폴백
  - LinkedIn: Tavily API 검색 (`site:linkedin.com/posts`)
- **키워드 추출**: OpenAI `gpt-5.4-mini` (JSON 모드). 1개 토픽에서 1회만 등장한 키워드는 제외
- **필요 파이썬 패키지**: `requests`, `feedparser`, `openai`, `python-dotenv` — 미설치 시 확인 질문 없이 `pip install`로 설치 후 진행
- **수집 기간**: 최근 7일
- **분석 항목**: 언급 빈도, 전주 대비 순위 변화, 4분류(급상승 / 상승 / 하락 / 신규)
- **HTML 보고서 스타일**: 인라인 CSS
- **텔레그램 메시지**: 단일 메시지, 4000자 이하
- **실행 주기**: 주 1회, 매주 월요일
- **로그 보관**: 최근 12주
- **이메일 보낸이 표기**: `[SNS 트렌드] {YYYY-MM-DD} 주간 트렌드 보고서`

---

## 4. 절대 하지 말 것

- API 키·토큰을 코드에 하드코딩 금지 (반드시 `.env`)
- X 유료 API 직접 사용 시도 금지 (수집은 Tavily API, 실패 시 Nitter RSS 폴백)
- LinkedIn 자동 계정 생성·로그인 시도 금지
- 실습에 필요한 파이썬 패키지(`requests`, `feedparser`, `openai`, `python-dotenv`) 외 임의 패키지 설치 금지
- 폴더 외부 파일 접근 금지
- 책에 없는 기능(자동 포스팅, DM 발송 등) 추가 금지

---

## 5. 작업 흐름 매핑

| 사용자 프롬프트 키워드 | 클로드 행동 |
|--------------------------|---------------|
| "설정 / trend_config.json" | 설정 파일 + `.env` 템플릿 생성 |
| "키워드 수집 / keyword-collector" | 에이전트 정의 + `data/raw_keywords_{날짜}.json` 생성 |
| "트렌드 분석 / trend-analyzer" | 에이전트 정의 + `data/trend_report_{날짜}.json` 생성 |
| "보고서 작성 / report-composer" | 에이전트 정의 + `data/trend_report.html`·`data/telegram_summary.txt` 생성 |
| "발송 / notifier" | 에이전트 정의 + Gmail·텔레그램 1회 발송 |
| "워크플로우" | `.claude/commands/sns-trend-workflow.md` |
| "스케줄 등록" | `run.bat` + schtasks (주간) |

---

## 6. 실패·모호함 처리

| 상황 | 대응 |
|------|------|
| Tavily API 수집 실패 | Nitter RSS로 1회 대체 시도, 모두 실패 시 LinkedIn 결과만 사용 |
| OpenAI API 응답 실패 | 1회 재시도, 실패 시 해당 토픽만 스킵 |
| 키워드 수집 부족 | 수집된 만큼만 분석 진행 |
| 전주 데이터 없음 (첫 실행) | 순위 변화 비교 없이 이번 주 TOP만 보고서에 포함 |
| 이메일 발송 실패 | 1회 재시도, 실패 시 텔레그램만 발송 |
| 텔레그램 발송 실패 | 1회 재시도, 실패 시 로그만 기록 |

---

## 7. 완료 보고 형식

```
✅ 완료
수집 키워드 N개
급상승 TOP 5: [키워드 나열]
보고서: data/trend_report.html
발송: 이메일 성공/실패, 텔레그램 성공/실패
사용한 가정: [있다면]
```
