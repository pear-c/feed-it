# Workflow A — Collect (n8n 노드 설계서)

> n8n UI에서 그대로 구성 가능한 초안. 사용자가 setup.md 완료 후 이 문서를 따라 워크플로 만든 뒤, export → `workflows/collect.json` commit.
>
> 전제: `docker-compose.yml`에 `./channels.yaml` 및 `./prompts/` 마운트 완료 (컨테이너 내부 경로 `/data/feed-it/`).

---

## 개요

| 항목 | 값 |
|------|---|
| 워크플로명 | `collect` |
| 트리거 | Schedule Trigger (cron `0 9 * * 1,3,6`, timezone `Asia/Seoul`) |
| 목적 | channels.yaml 기반 채널 순회 → Firecrawl 스크래핑 → Claude URL 추출 → Notion 중복 필터 → upsert → Slack 알림 → Workflow B 호출 |
| credential 의존성 | Anthropic API, Firecrawl (HTTP Header Auth), Notion API, Slack API |
| 예상 실행 시간 | 5~15분 (채널 9개 기준) |

---

## 데이터 흐름

```
Schedule Trigger
  → Read channels.yaml          → 9개 채널 객체
  → Filter (enabled=true)       → 활성 채널만
  → Set today_kst               → 오늘 날짜 메타
  → SplitInBatches (5)
       ├── Firecrawl scrape (channel.url)
       │      → page_markdown
       ├── Read extract_urls.md (1회만)
       └── Claude messages.create (Sonnet 4.6)
              → JSON array
  → Parse + sanitize JSON       → URL 후보 N개
  → Normalize URL               → 정규화된 URL N개
  → Notion query (URL chunked) → 기존 URL set
  → Diff → 신규 URL만
  → Notion pages.create (loop)  → 신규 row M개 (IsMetadataFilled=false)
  → Aggregate counters          → 수집 N건 / 신규 M건 / Firecrawl 사용량
  → Slack chat.postMessage       → #feed-it-digest
  → Execute Workflow → summarize
```

---

## 노드별 상세 (14개)

### F-1. Schedule Trigger

| 필드 | 값 |
|------|---|
| Mode | Cron Expression |
| Expression | `0 9 * * 1,3,6` |
| Timezone | `Asia/Seoul` |

> `GENERIC_TIMEZONE=Asia/Seoul` 환경변수가 박혀있어도 노드 자체에 timezone 명시 (conventions §7.1 함정).

### F-2. Read channels.yaml

| 필드 | 값 |
|------|---|
| Node type | **Read/Write Files from Disk** (n8n 기본 노드) |
| Operation | Read From File |
| File Selector | `/data/feed-it/channels.yaml` |
| Output Property Name | `data` |

다음 노드(F-3)에서 YAML parse.

### F-3. Parse YAML

| 필드 | 값 |
|------|---|
| Node type | **Code** (JavaScript) |
| Code | 아래 박스 참조 |

```javascript
const yaml = require('js-yaml');
const raw = $input.first().binary.data;
const text = Buffer.from(raw.data, 'base64').toString('utf-8');
const channels = yaml.load(text);
return channels.map((c) => ({ json: c }));
```

> n8n 컨테이너에 `js-yaml`이 내장돼있지 않으면 Code Node 위에서 import 실패. 대안: Code Node에서 정규식·문자열 파싱 (channels.yaml 구조가 단순하므로 가능). 또는 docker-compose에 `NODE_FUNCTION_ALLOW_BUILTIN=*` + Community Node `n8n-nodes-yaml` 검토.

### F-4. Filter (enabled=true)

| 필드 | 값 |
|------|---|
| Node type | **Filter** |
| Conditions | `{{ $json.enabled }}` is `true` |

### F-5. Set today_kst

| 필드 | 값 |
|------|---|
| Node type | **Set** |
| Mode | Keep Only Set |
| Values | `today_kst` = `{{ $now.setZone('Asia/Seoul').toFormat('yyyy-MM-dd') }}` <br> `channel` = `{{ $json }}` |

### F-6. SplitInBatches

| 필드 | 값 |
|------|---|
| Node type | **Split In Batches** |
| Batch Size | 5 |

채널 9개를 5+4로 처리. Firecrawl·Anthropic rate limit 보호.

### F-7. Firecrawl scrape

옵션 A (HTTP Request, 권장 — PoC 단순화):

| 필드 | 값 |
|------|---|
| Node type | **HTTP Request** |
| Method | POST |
| URL | `https://api.firecrawl.dev/v1/scrape` |
| Authentication | Header Auth (credential: Firecrawl) |
| Body Content Type | JSON |
| Body | `{ "url": "{{ $json.channel.url }}", "formats": ["markdown"], "waitFor": 2000 }` |

응답: `{ success, data: { markdown, ... } }`. 다음 노드에서 `data.markdown` 사용.

옵션 B (MCP, 원글 동일):

| 필드 | 값 |
|------|---|
| Node type | **MCP Client** (Community Node) |
| Server | Firecrawl MCP (별도 docker container 또는 npx) |
| Tool | `firecrawl_scrape` |
| Arguments | `{ "url": "{{ $json.channel.url }}", "formats": ["markdown"] }` |

> MCP 노드가 안 잡히면 A로 폴백. 두 방식 출력 스키마 거의 동일.

### F-8. Read extract_urls.md (1회 fetch + cache)

| 필드 | 값 |
|------|---|
| Node type | **Read/Write Files from Disk** |
| Operation | Read From File |
| File Selector | `/data/feed-it/prompts/extract_urls.md` |

Static Data로 캐싱하면 매 채널마다 디스크 읽기 안 함. PoC 단계엔 매번 읽어도 부담 X.

### F-9. Claude messages.create (URL 추출)

| 필드 | 값 |
|------|---|
| Node type | **HTTP Request** |
| Method | POST |
| URL | `https://api.anthropic.com/v1/messages` |
| Authentication | Header Auth: `x-api-key: {{ $credentials.anthropic.apiKey }}` |
| Headers | `anthropic-version: 2023-06-01`<br>`content-type: application/json` |
| Body | 아래 박스 |

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 2000,
  "system": "{{ extract_urls.md 본문 (frontmatter 제외) }}",
  "messages": [
    {
      "role": "user",
      "content": "channel.name: {{ $json.channel.name }}\nchannel.type: {{ $json.channel.type }}\nchannel.domain: {{ $json.channel.domain }}\ntoday_kst: {{ $json.today_kst }}\n\npage_markdown:\n{{ $json.page_markdown }}"
    }
  ]
}
```

응답에서 `content[0].text`가 LLM 출력 (JSON 배열 string).

### F-10. Parse + sanitize JSON

| 필드 | 값 |
|------|---|
| Node type | **Code** |

```javascript
let text = $input.first().json.content[0].text.trim();
// 마크다운 코드블록 제거 (안전망 — conventions §1.6)
text = text.replace(/^```json\s*/i, '').replace(/^```\s*/i, '').replace(/```\s*$/i, '').trim();
try {
  const arr = JSON.parse(text);
  if (!Array.isArray(arr)) throw new Error('not array');
  return arr.map((item) => ({ json: { ...item, channel: $input.first().json.channel } }));
} catch (e) {
  // 파싱 실패: ErrorNote 적재용 더미 item으로 통과 + Slack alerts에 로그
  return [{ json: { _parse_error: e.message, raw: text.slice(0, 500), channel: $input.first().json.channel } }];
}
```

### F-11. Normalize URL

| 필드 | 값 |
|------|---|
| Node type | **Code** |

```javascript
const items = $input.all();
return items.map((it) => {
  let url = (it.json.url || '').trim();
  if (!url) return { json: { ...it.json, _skip: true } };
  // 1) https 강제
  url = url.replace(/^http:/, 'https:');
  // 2) host 소문자
  try {
    const u = new URL(url);
    u.host = u.host.toLowerCase();
    // 3) UTM 제거
    [...u.searchParams.keys()].filter((k) => k.toLowerCase().startsWith('utm_')).forEach((k) => u.searchParams.delete(k));
    // 4) trailing slash 제거 (pathname이 / 만이면 유지)
    if (u.pathname.length > 1 && u.pathname.endsWith('/')) u.pathname = u.pathname.slice(0, -1);
    url = u.toString();
  } catch (e) { /* 잘못된 URL은 그대로 */ }
  return { json: { ...it.json, url } };
});
```

### F-12. Notion query (URL chunked 중복 체크)

| 필드 | 값 |
|------|---|
| Node type | **Notion** |
| Resource | Database Page |
| Operation | Get Many |
| Database | (credential 등록된 feed-it DB) |
| Filter | `URL` `equals` `{{ $json.url }}` |
| Return All | false (단건 lookup) |

각 URL마다 1회 호출. 9 채널 × 평균 글 5개 = 45회 ≈ Notion rate (3/sec) 안. 안전 보강하려면 SplitInBatches(3) + Wait(1s).

### F-13. Diff + Notion pages.create

| 필드 | 값 |
|------|---|
| Node type | **IF** + **Notion** |

IF 조건: 위 F-12 결과가 0건이면 신규 → 다음 Notion 노드로.

Notion 노드 (페이지 생성):

| 필드 | 값 |
|------|---|
| Operation | Create a Database Page |
| Title | `{{ $json.title }}` |
| Properties |  ▾ |

| Property | Value |
|---------|-------|
| URL | `{{ $json.url }}` |
| Channel | `{{ $json.channel.name }}` |
| Domain | `{{ $json.channel.domain }}` |
| PublishedAt | `{{ $json.published_at || null }}` (null 허용) |
| CollectedAt | `{{ $now.setZone('Asia/Seoul').toISO() }}` |
| IsMetadataFilled | `false` |

> Author / Tags / Summary / ErrorNote는 비워둠 (Summarize 워크플로가 채움).

### F-14. Aggregate + Slack chat.postMessage

| 필드 | 값 |
|------|---|
| Node type | **Code** + **Slack** |

Code Node에서 channel별 카운트 집계:
```javascript
const items = $input.all();
const byChannel = {};
items.forEach((it) => {
  const name = it.json.channel?.name || 'unknown';
  byChannel[name] = (byChannel[name] || 0) + 1;
});
const total = items.length;
const lines = Object.entries(byChannel).map(([n, c]) => `• ${n}: ${c}건`);
return [{
  json: {
    text: `:inbox_tray: feed-it 수집 완료 (${$now.setZone('Asia/Seoul').toFormat('yyyy-MM-dd HH:mm')} KST)\n총 ${total}건 신규 수집\n${lines.join('\n')}`,
  },
}];
```

Slack 노드:
| 필드 | 값 |
|------|---|
| Resource | Message |
| Operation | Post |
| Channel | `{{ $env.SLACK_DIGEST_CHANNEL_ID }}` |
| Text | `{{ $json.text }}` |

### F-15. Execute Workflow → summarize

| 필드 | 값 |
|------|---|
| Node type | **Execute Workflow** |
| Workflow | summarize (Workflow B) |
| Mode | Each Item (또는 한 번 호출 후 B가 query) |

PoC 단계에선 B가 Notion query로 미요약 row를 가져오므로 **단순 1회 호출**로 충분.

---

## 에러 처리 정책

- **워크플로 settings → `Error Workflow` = error-monitor** (Workflow C)
- 노드별 `Continue On Fail` 적용 위치:
  - F-7 (Firecrawl): 실패 시 해당 채널 skip (다음 채널 진행)
  - F-9 (Claude): 실패 시 빈 배열로 처리 + 다음 채널
  - F-13 (Notion create): 실패 시 skip + ErrorNote는 별도 카운터
- 4xx 응답: 즉시 fail, 재시도 없음 (conventions §3 Anthropic·Firecrawl 응답 검증)
- 5xx / 네트워크: n8n 기본 재시도 (3회, 지수 백오프)

---

## Firecrawl quota 보호 (Q3 자동 중단 정책)

별도 노드 추가 (F-2와 F-3 사이 권장):

| 필드 | 값 |
|------|---|
| Node type | **HTTP Request** |
| Method | GET |
| URL | `https://api.firecrawl.dev/v1/team/credit-usage` |
| Authentication | Header Auth: Firecrawl |

응답에서 잔여 quota 추출. IF 조건:
- 잔여 < 10%: Slack `#feed-it-alerts` 경고 + 워크플로 계속
- 잔여 < 1%: Slack `#feed-it-alerts` 즉시 중단 알림 + 워크플로 종료 (throw error)

PoC 1주차엔 매번 호출하면 약간 비용. 일 1회 또는 매 수집 실행 시 1회만.

---

## credential 의존성 (n8n credential 화면에서 사전 등록)

- `Anthropic API` (HTTP Header Auth)
- `Firecrawl` (HTTP Header Auth, `Authorization: Bearer fc-...`)
- `Notion API` (OAuth2 또는 Internal Integration Token)
- `Slack API` (Bot User OAuth Token `xoxb-...`)

---

## 검증 (F-15 이후)

- [ ] 수동 실행 1회 → Notion DB에 신규 row 생기는지
- [ ] 1초 뒤 같은 워크플로 1회 더 실행 → 중복 row 0건 (멱등성 회귀)
- [ ] `#feed-it-digest`에 수집 다이제스트 도착
- [ ] Workflow B (summarize) 호출됐는지 (Executions 페이지 확인)

---

## Export + commit

```powershell
# n8n UI에서: workflow 우상단 ⋮ → Download → JSON 파일 저장
# 저장 위치: D:\workspace-bsh\feed-it\workflows\collect.json
git add workflows/collect.json
git commit -m "feat: Workflow A (collect) v1 — channels.yaml → Notion upsert"
git push
```

---

## 알려진 함정 (conventions §7 참조)

- F-1: timezone 노드 안에 명시 안 하면 UTC로 동작 (§7.1)
- F-7: Firecrawl이 빈 markdown 반환 케이스 (§7.2)
- F-9: Claude 마크다운 코드블록 wrap (§7.4) → F-10에서 안전망
- F-11: URL 정규화 안 하면 trailing slash·UTM으로 중복 인식 안 됨 (§7.3)
- F-12: Notion query rate limit (§5.1)
- F-14: Slack `not_in_channel` (§7.6) — 첫 실행 전 `/invite @feed-it` 확인

---

## TODO (다음 회차)

- [ ] Workflow B (summarize) 설계서 → `docs/workflow-summarize.md`
- [ ] Workflow C (error-monitor) 설계서 → `docs/workflow-error-monitor.md`
- [ ] Firecrawl quota 노드의 정확한 endpoint·필드 확인 (Firecrawl docs)
- [ ] F-3 YAML parse 방식 검증 (`js-yaml` 가용성 또는 대안)
