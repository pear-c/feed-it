# feed-it 셋업 가이드

> 0부터 첫 자동 실행까지. 처음 1회만 따라하면 됨.
> 예상 소요: 약 1시간 (계정 생성 포함 시 추가).

---

## 0. 사전 준비

| 항목 | 확인 |
|------|------|
| **Docker Desktop on Windows** | 설치·실행 중 (`docker --version` 동작) |
| **Anthropic 계정** | API 결제 가능 ([console.anthropic.com](https://console.anthropic.com)) |
| **Firecrawl 계정** | 무료 plan 가입 ([firecrawl.dev](https://firecrawl.dev)) |
| **Notion 워크스페이스** | 개인 무료 plan으로 충분 |
| **Slack 워크스페이스** | 개인 무료 plan으로 충분. 관리자 권한 필요 (App 설치용) |
| **GitHub `pear-c` 계정** | feed-it 레포 push용 ([github.com/pear-c](https://github.com/pear-c)) |

> **다른 GitHub 계정과 혼동 금지** — feed-it 레포는 `pear-c` 계정으로만 commit·push (자세한 사항: 메모리 `project_feed_it_git_account.md`)

---

## 1. git config 본인 설정 (Claude는 건드리지 않음)

```powershell
cd D:\workspace-bsh\feed-it

# pear-c GitHub 계정의 commit email로 설정 (GitHub > Settings > Emails 확인)
# privacy email 사용 권장:  <id>+pear-c@users.noreply.github.com
git config --local user.name "pear-c"
git config --local user.email "<pear-c email 주소>"

# 확인
git config --local --get user.name
git config --local --get user.email
```

> `--local`은 이 레포에만 적용. 다른 워크스페이스 프로젝트 영향 없음.

---

## 2. `.env` 작성 + 시크릿 생성

### 2.1 파일 복사

```powershell
cd D:\workspace-bsh\feed-it
Copy-Item .env.example .env
```

### 2.2 n8n 자체 키 3종 생성

PowerShell에서 32자 랜덤 문자열:

```powershell
function New-Secret { -join ((48..57)+(65..90)+(97..122) | Get-Random -Count 32 | ForEach-Object {[char]$_}) }

# 3번 실행해서 .env 의 다음 자리에 각각 붙여넣기:
#   N8N_BASIC_AUTH_PASSWORD
#   N8N_ENCRYPTION_KEY
New-Secret
New-Secret
```

`N8N_BASIC_AUTH_USER`는 본인 취향 (기본값 `admin` 그대로 OK).

> ⚠️ `N8N_ENCRYPTION_KEY`는 **한 번 정하면 절대 변경 금지** — 변경하면 등록된 모든 credential이 복호화 불가.

### 2.3 외부 API 키 6종은 §3~§6에서 발급 후 채움

---

## 3. Notion DB 셋업

### 3.1 DB 생성

1. Notion → 좌측 사이드바 `+ New page` → `Database — Full page` 선택
2. 페이지 제목: `feed-it`
3. 기본 속성 `Name` 을 `Title`로 두고 나머지 11속성 정의

### 3.2 속성 11종 정의

| 속성명 | 타입 | 옵션·기본값 |
|--------|------|------------|
| `Title` | Title | (기본) |
| `URL` | URL | — |
| `Channel` | Select | (수집되면서 자동 채워짐) |
| `Domain` | Multi-select | 옵션: `ai`, `devops`, `kr_tech` |
| `PublishedAt` | Date | (Include time 체크) |
| `CollectedAt` | Date | (Include time 체크) |
| `Author` | Text | — |
| `Tags` | Multi-select | (요약 단계에서 자동 채워짐) |
| `Summary` | Text | (250자 한줄) |
| `IsMetadataFilled` | Checkbox | 기본값 `unchecked` |
| `ErrorNote` | Text | — |

### 3.3 Integration 생성 + DB share

1. <https://www.notion.so/my-integrations> → `+ New integration`
   - Name: `feed-it`
   - Associated workspace: 해당 워크스페이스 선택
   - Type: `Internal`
2. 생성 후 `Internal Integration Secret` 복사 (`secret_xxxxxxxx...`)
   → `.env` 의 `NOTION_API_KEY` 자리에 붙여넣기
3. Notion DB 페이지로 돌아가서 우측 상단 `•••` → `Connect to` → `feed-it` Integration 선택

### 3.4 DB ID 추출

DB URL 형태:

```
https://www.notion.so/{workspace}/{32자ID}?v={view-id}
                                 ↑ 이 부분
```

32자 ID(하이픈 제거된 형태)를 `.env`의 `NOTION_DATABASE_ID`에 붙여넣기.

---

## 4. Slack Bot 셋업

### 4.1 App 생성

1. <https://api.slack.com/apps> → `Create New App` → `From scratch`
2. App Name: `feed-it`
3. Workspace 선택

### 4.2 권한 추가

좌측 메뉴 `OAuth & Permissions` → 하단 `Bot Token Scopes`에서 추가:
- `chat:write`

### 4.3 워크스페이스에 설치

같은 화면 상단 `Install to Workspace` → Allow.

설치 완료 후 `Bot User OAuth Token` 복사 (`xoxb-...`)
→ `.env`의 `SLACK_BOT_TOKEN` 자리에 붙여넣기.

### 4.4 채널 2개 생성

Slack 워크스페이스에서:
- `#feed-it-digest` — 다이제스트 도착용
- `#feed-it-alerts` — 에러 알림용

### 4.5 Bot 채널 invite

각 채널에서:

```
/invite @feed-it
```

### 4.6 채널 ID 추출

각 채널 이름 우클릭 → `View channel details` → 하단의 Channel ID (`C0XXXXXXXX`) 복사.

`.env`에 각각:
- `SLACK_DIGEST_CHANNEL_ID=C...`
- `SLACK_ALERTS_CHANNEL_ID=C...`

---

## 5. Anthropic API 키 발급

1. <https://console.anthropic.com> 로그인
2. 좌측 `Settings` → `API keys` → `Create Key`
3. Name: `feed-it`
4. Key 값(`sk-ant-...`) 복사 → `.env`의 `ANTHROPIC_API_KEY`
5. 결제 정보 등록 (대시보드 → Billing). PoC 단계 사용량 추정 < $5/월

---

## 6. Firecrawl API 키 발급

1. <https://firecrawl.dev/app> 로그인
2. 좌측 `API Keys` → `Create New Key`
3. Name: `feed-it`
4. Key 값(`fc-...`) 복사 → `.env`의 `FIRECRAWL_API_KEY`
5. 무료 quota 확인 (대시보드 → Usage). 500 page/month — 채널 5~10개로 시작하면 여유

---

## 7. n8n 띄우기

### 7.1 docker compose up

```powershell
cd D:\workspace-bsh\feed-it
docker compose up -d
docker compose logs -f n8n
```

`Editor is now accessible via:` 라인이 나오면 정상 기동.

### 7.2 브라우저 접속 + 로그인

<http://localhost:5678> → Basic Auth (`N8N_BASIC_AUTH_USER` / `N8N_BASIC_AUTH_PASSWORD`)

첫 접속 시 n8n 자체 owner 계정 생성 안내가 뜸. 본인 이메일·비밀번호 입력 (이건 n8n 내부 계정. 외부 노출 안 됨).

### 7.3 Community Node 설치 (선택 — Week 1엔 SKIP 가능)

Week 1 PoC는 Firecrawl·Anthropic 모두 **HTTP Request 노드로 직접 호출** (워크플로 docs 참조). 따라서 `n8n-nodes-mcp` Community Node는 **Week 1엔 불필요**. Week 2+ Python 이식·MCP 통합 검토 단계에서 도입.

지금 미리 설치해두고 싶다면:

1. 좌측 메뉴 `Settings` → `Community nodes`
2. `Install a community node` → npm 패키지명: `n8n-nodes-mcp`
3. `I understand the risks` 체크 → `Install`
4. 노드 목록에 `MCP Client` 등이 보이면 성공

> 안 보이면 `docker compose restart n8n` 후 재확인. 컨테이너 로그(`docker compose logs n8n`)에서 npm 설치 실패 메시지 확인.

---

## 8. n8n credential 4종 등록

좌측 `Credentials` → `+ Add Credential`

### 8.1 Anthropic

- Type: `Anthropic API`
- API Key: `.env`의 `ANTHROPIC_API_KEY` 값
- Save → `Test connection` PASS 확인

### 8.2 Notion

- Type: `Notion API`
- API Key: `.env`의 `NOTION_API_KEY`
- Save → `Test connection` PASS

### 8.3 Slack

- Type: `Slack API`
- Authentication: `Access Token`
- Access Token: `.env`의 `SLACK_BOT_TOKEN`
- Save → `Test connection` PASS

### 8.4 Firecrawl (HTTP Header Auth)

Firecrawl은 워크플로에서 **HTTP Request 노드로 직접 호출** (`https://api.firecrawl.dev/v1/scrape`). credential은 n8n 기본의 `Header Auth`로 등록.

- `+ Add Credential` → 검색에 `Header Auth` 입력 → `Header Auth` 선택
- **Name** (credential 이름): `Firecrawl API` (워크플로에서 이 이름으로 골라쓰게 됨)
- **Header Name**: `Authorization`
- **Header Value**: `Bearer fc-...` (`.env`의 `FIRECRAWL_API_KEY` 값에 `Bearer ` prefix 붙여서)
- Save

> Test 버튼은 Header Auth credential 자체에는 없음. §9 connectivity 테스트에서 임시 워크플로로 HTTP Request 1회 호출해 검증.

#### 왜 MCP 안 씀?

- Firecrawl이 **hosted MCP endpoint를 제공하지 않음** (2026-05 기준). self-host(`npx firecrawl-mcp`)는 stdio/SSE/HTTP transport 별도 설정 필요 — PoC 단계엔 과함
- HTTP Request 직접 호출이 더 단순·투명 (응답 JSON 디버깅 쉬움)
- Week 2+ Python 이식·MCP 통합 검토 단계에서 SSE 컨테이너 별도 띄워 재평가

---

## 9. 첫 connectivity 테스트 (워크플로 만들기 전)

n8n에서 임시 워크플로 1개를 만들어 4종 credential이 정말 동작하는지 확인.

1. `+ Workflow` → 빈 워크플로
2. Manual Trigger → Slack 노드 (`Post Message` to `#feed-it-alerts`, text: "셋업 테스트")
3. Execute → Slack 채널에 "셋업 테스트" 도착 확인
4. 같은 워크플로에 Notion 노드 (`Database — Get Many`) 추가 → 빈 DB에서 Execute → 에러 없이 0건 반환 확인
5. 임시 워크플로 삭제

---

## 10. 트러블슈팅

### 10.1 `docker compose up` 즉시 종료

- Docker Desktop이 안 켜진 경우. WSL2 기반이라 첫 부팅 30초 정도 걸림
- 확인: `docker ps` 동작 여부

### 10.2 `http://localhost:5678` 접속 무한 로딩

- Windows 방화벽이 5678 차단. Docker Desktop 첫 실행 시 권한 요청 허용 안 한 경우
- 임시 해결: `docker compose down && docker compose up -d` 후 방화벽 prompt 다시 받기

### 10.3 n8n credential Test 실패 (Notion `object_not_found`)

- DB가 Integration에 share 안 됨 (§3.3 다시 확인)
- DB ID 32자가 정확한지 (하이픈 X)

### 10.4 Slack 메시지 도착 안 함 (`not_in_channel`)

- Bot이 채널에 invite 안 됨
- 해당 채널에서 `/invite @feed-it` 다시 실행

### 10.5 Community Node 설치 후에도 MCP 노드 안 보임

- 컨테이너 재시작: `docker compose restart n8n`
- 환경변수 `N8N_COMMUNITY_PACKAGES_ENABLED=true` 확인 (docker-compose.yml에 이미 박혀있음)
- 로그 확인: `docker compose logs n8n | Select-String -Pattern "npm|n8n-nodes-mcp"`

### 10.6 Firecrawl 401 Unauthorized

- API Key 자리에 `fc-` prefix 누락
- 무료 계정도 키 발급 가능. 결제정보 등록 없이도 500/월 한도 안에서 동작

---

## 11. 셋업 완료 체크

- [ ] `.env` 6종 키 + 3종 n8n 시크릿 모두 실제 값
- [ ] `docker compose up -d` 후 <http://localhost:5678> 접속 OK
- [ ] n8n owner 계정 생성
- [ ] Community Node `n8n-nodes-mcp` 설치 + MCP 노드 보임
- [ ] credential 4종 모두 Test PASS
- [ ] §9 connectivity 테스트 (Slack 도착 + Notion 0건 반환) PASS
- [ ] git remote `origin` → `github.com/pear-c/feed-it.git` 확인 (`git remote -v`)
- [ ] `git config --local user.name` / `user.email` = pear-c 본인 정보

여기까지 OK면 **게이트 ① 진입 직전**.
다음 단계는 사용자와 함께 `channels.yaml`을 합의(D 서브태스크) — Claude 세션에서 진행.
