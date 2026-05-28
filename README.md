# feed-it

> AI · DevOps · 한국 IT 분야의 새 글을 주 3회 자동으로 모아 한국어로 요약해서 Notion에 적재하고 Slack으로 알림 보내는 **개인 자동화 시스템**.

<sub>Status: 🚧 Week 1 WIP · Personal use only · Korean output</sub>

---

## 무엇

기술 업계의 흐름을 따라가려면 매일 수십 개 채널을 직접 봐야 한다. feed-it은 그 수고를 n8n + Claude 한국어 요약 + Notion DB로 옮긴다.

```
n8n (로컬 Docker)
 ├─ Collect       cron 0 9 * * 1,3,6 (KST)
 │   채널 → Firecrawl → Claude → URL 추출 → Notion upsert → Slack
 │       └─ Execute Workflow → Summarize
 ├─ Summarize
 │   Notion(미요약) → Firecrawl → Claude 한국어 요약
 │   → Notion 본문 업데이트 → Slack 다이제스트
 └─ ErrorMonitor  (Error Trigger 전역) → Slack alerts
```

핵심 패턴 두 가지:
- **수집·요약 분리** — 둘 사이를 Notion DB가 버퍼로 받는다. 한쪽이 실패해도 다른 쪽은 살아있음
- **URL UNIQUE 멱등성** — 같은 워크플로를 N번 돌려도 중복 적재 없음

---

## 왜

수십 개의 기술 블로그·뉴스레터를 매일 직접 챙기는 건 비현실적이다. 한 번 못 보면 며칠 분이 쌓이고, 영문 본문은 빠르게 훑기에도 부담스럽다. feed-it은 그 부담을 다음 셋업으로 옮긴다:

- **수집은 정해진 시각에만** — 매일 알림이 오면 노이즈. 주 3회(월/수/토 09:00 KST)로 충분
- **요약은 한국어로** — 영문 원문을 다 읽지 않아도 핵심 파악. 더 볼 가치가 있는 것만 원문 클릭
- **Notion이 저장소** — Slack은 알림용. 검색·태깅·아카이브는 Notion에 위임
- **수집·요약을 분리** — 한쪽이 실패해도 다른 쪽은 살아있고, Notion이 자연스러운 버퍼

처음엔 5~10개 채널로 시작해 가치 검증 후 확대한다. n8n 워크플로 위에서 출발하되, 후속으로 LangGraph 코드 이식이 가능하도록 설계.

## 도메인

| 축 | 예시 |
|----|------|
| AI / LLM / 에이전트 | Anthropic, OpenAI, Simon Willison, Hugging Face |
| DevOps / 인프라 / 클라우드 | CNCF, HashiCorp |
| 한국 IT / 채용 / 스타트업 | 우아한형제들, 토스 |

---

## 기술 스택

| 영역 | 도구 |
|------|------|
| 오케스트레이션 | n8n (self-hosted Docker, KST timezone) |
| AI | Claude Sonnet 4.6 (Anthropic API) |
| 웹 스크래핑 | Firecrawl MCP (Cloudflare 우회) |
| 저장소 | Notion DB (11 속성) |
| 알림 | Slack Bot (`chat:write`) — `#feed-it-digest` / `#feed-it-alerts` |
| 시크릿 | `.env` + n8n credential (Vault 미사용) |

---

## 디렉토리

```
feed-it/
├── docker-compose.yml      n8n self-hosted
├── .env.example            6 키 placeholder + 생성 명령
├── .gitignore .gitattributes
├── README.md               (이 파일)
├── docs/
│   ├── setup.md            0부터 첫 실행까지 (11 섹션)
│   └── operations.md       재실행·quota·정리 (예정)
├── prompts/                Claude 프롬프트 (예정)
│   ├── extract_urls.md
│   └── summarize_ko.md
├── workflows/              n8n export JSON (예정)
│   ├── collect.json
│   ├── summarize.json
│   └── error-monitor.json
├── channels.yaml           수집 대상 (예정)
└── data/                   n8n 영구 볼륨 (gitignored)
```

---

## 빠른 시작

```powershell
# 0. 사전: Docker Desktop, Anthropic/Firecrawl/Notion/Slack 계정
# 자세한 절차는 docs/setup.md 참조

git clone https://github.com/pear-c/feed-it.git
cd feed-it

Copy-Item .env.example .env
# .env 안 replace-me 자리를 모두 실제 값으로

docker compose up -d
start http://localhost:5678
```

상세 가이드: [`docs/setup.md`](docs/setup.md)

---

## 로드맵

| Week | 목표 | 상태 |
|------|------|------|
| 1 | n8n PoC — 수집·요약·에러 모니터링 워크플로 동작 | 🚧 진행 중 |
| 2 | 1~2주 운영 회고 → LangGraph 이식 판단 | ⏳ |
| 3+ | LangGraph 그래프화 · 채널 확장 · MCP 노출 등 | ⏳ |

Week 2 진입 판단 기준은 `.claude/tasks/feed-it-week1/PLAN.md` §10 참조.

---

## 정책

- **개인 사용 한정.** 외부 배포 안 함
- **수집 대상은 공개 콘텐츠만** — robots.txt + 이용약관 확인 후 등록
- **본문 전체 복사 금지** — 한국어 요약 + 원문 URL 동반 표시
- **시크릿은 `.env`만** — 코드·docs·워크플로 export에 평문 키 박지 않음

---

## 라이선스

미정 (개인 사용 한정. 외부 배포 결정 시 추가).
