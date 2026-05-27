# Workflow B — Summarize (n8n 노드 설계서)

> Workflow A(collect)가 끝에서 호출. 또는 사용자가 수동 트리거 가능.
> 입력은 Notion DB의 `IsMetadataFilled=false` row들. 출력은 Notion 본문 + Slack 다이제스트.

---

## 개요

| 항목 | 값 |
|------|---|
| 워크플로명 | `summarize` |
| 트리거 | Execute Workflow Trigger (collect가 호출) + Manual Trigger (디버깅용) |
| 목적 | 미요약 글 → Firecrawl 본문 → Claude 한국어 요약 → Notion 본문/속성 업데이트 → Slack 다이제스트 |
| credential | Anthropic, Firecrawl, Notion, Slack |
| 예상 실행 시간 | 10~30분 (신규 글 20~30건 기준) |
| 처리 단위 | Notion row 1개 = Firecrawl 1 page + Claude 1 호출 |

---

## 데이터 흐름

```
Execute Workflow Trigger (from collect)
     │  (또는 Manual Trigger 디버깅용)
     ▼
Notion databases.query (IsMetadataFilled=false, page_size 20)
     │  → N건의 미요약 row
     ▼
SplitInBatches (size 3) + Wait (2s)   ← Anthropic rate limit 보호
     │
     ▼
Firecrawl scrape (row.url, formats=markdown)
     │  → body_markdown
     ▼
Read summarize_ko.md (1회 fetch)
     │
     ▼
Claude messages.create (Sonnet 4.6)
     │  → JSON {author, tags, summary_short, summary_long}
     ▼
Parse + sanitize JSON
     │
     ▼
Split summary_long (1500자 단위)
     │  → paragraph blocks N개
     ▼
Notion blocks.children.append (페이지 본문에 paragraph 추가)
     │
     ▼
Notion pages.update (Summary, Author, Tags, IsMetadataFilled=true)
     │
     ▼
(누적 후)
Aggregate digest
     │
     ▼
Slack chat.postMessage → #feed-it-digest (Block Kit 또는 단순 텍스트)
```

---

## 노드별 상세 (13개)

### G-1. Execute Workflow Trigger

| 필드 | 값 |
|------|---|
| Node type | **Execute Workflow Trigger** |

(Workflow A의 마지막 Execute Workflow 노드가 이 워크플로를 호출)

### G-1'. Manual Trigger (디버깅용 병렬)

| 필드 | 값 |
|------|---|
| Node type | **Manual Trigger** |
| 연결 | G-2 (같은 다음 노드) |

수동 실행으로 G-2 ~ G-13 단독 디버그 가능.

### G-2. Notion query (미요약 row 조회)

| 필드 | 값 |
|------|---|
| Node type | **Notion** |
| Resource | Database Page |
| Operation | Get Many |
| Database | (credential 등록된 feed-it DB) |
| Return All | false |
| Limit | 20 |
| Filter | `IsMetadataFilled` `equals` `false` |
| Sort | `CollectedAt` ASC (오래된 것부터) |

> 20건씩 처리. 더 많으면 다음 트리거에서 처리 (다음 cron 또는 collect 재호출).

### G-3. SplitInBatches + Wait

| 필드 | 값 |
|------|---|
| Node type | **Split In Batches** |
| Batch Size | 3 |

다음 노드 (Firecrawl) 후 **Wait Node 2초** 삽입.

### G-4. Firecrawl scrape (본문)

| 필드 | 값 |
|------|---|
| Node type | **HTTP Request** |
| Method | POST |
| URL | `https://api.firecrawl.dev/v1/scrape` |
| Authentication | Firecrawl |
| Body | `{ "url": "{{ $json.properties.URL.url }}", "formats": ["markdown"], "waitFor": 2000 }` |

응답에서 `data.markdown` 추출.

> Firecrawl 응답이 빈 markdown인 경우 (conventions §7.2):
> - IF 노드로 `data.markdown.length > 100` 체크
> - 빈 경우 → Notion pages.update에서 `ErrorNote: "본문 추출 실패"` + skip

### G-5. Read summarize_ko.md

| 필드 | 값 |
|------|---|
| Node type | **Read/Write Files from Disk** |
| Operation | Read From File |
| File Selector | `/data/feed-it/prompts/summarize_ko.md` |

### G-6. Claude messages.create (요약)

| 필드 | 값 |
|------|---|
| Node type | **HTTP Request** |
| Method | POST |
| URL | `https://api.anthropic.com/v1/messages` |
| Authentication | Anthropic |
| Headers | `anthropic-version: 2023-06-01` |
| Body | 아래 박스 |

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1500,
  "system": "{{ summarize_ko.md 본문 (frontmatter 제외) }}",
  "messages": [
    {
      "role": "user",
      "content": "channel.name: {{ $json.properties.Channel.select.name }}\nchannel.domain: {{ $json.properties.Domain.multi_select[0].name }}\ntitle: {{ $json.properties.Title.title[0].plain_text }}\nurl: {{ $json.properties.URL.url }}\n\nbody_markdown:\n{{ $node['Firecrawl_scrape'].json.data.markdown }}"
    }
  ]
}
```

응답: `content[0].text` = JSON `{author, tags, summary_short, summary_long}`.

### G-7. Parse + sanitize JSON

```javascript
let text = $input.first().json.content[0].text.trim();
text = text.replace(/^```json\s*/i, '').replace(/^```\s*/i, '').replace(/```\s*$/i, '').trim();
try {
  const obj = JSON.parse(text);
  // 필드 검증
  if (typeof obj.summary_short !== 'string' || obj.summary_short.length > 250) {
    obj.summary_short = (obj.summary_short || '').slice(0, 250);
  }
  if (!Array.isArray(obj.tags)) obj.tags = [];
  obj.author = obj.author || '';
  obj.summary_long = obj.summary_long || '';
  return [{ json: { ...obj, _pageId: $('Notion_query').first().json.id, _title: $('Notion_query').first().json.properties.Title.title[0].plain_text } }];
} catch (e) {
  return [{ json: { _parse_error: e.message, raw: text.slice(0, 500), _pageId: $('Notion_query').first().json.id } }];
}
```

### G-8. Split summary_long (1500자)

```javascript
const obj = $input.first().json;
if (obj._parse_error) return [{ json: obj }];

const long = obj.summary_long || '';
const chunks = [];
for (let i = 0; i < long.length; i += 1500) {
  chunks.push(long.slice(i, i + 1500));
}
return [{ json: { ...obj, _blocks: chunks.map((c) => ({
  object: 'block',
  type: 'paragraph',
  paragraph: { rich_text: [{ type: 'text', text: { content: c } }] },
})) } }];
```

### G-9. Notion blocks.children.append (본문 적재)

| 필드 | 값 |
|------|---|
| Node type | **HTTP Request** (Notion 노드의 `append` 기능이 제한적인 경우) |
| Method | PATCH |
| URL | `https://api.notion.com/v1/blocks/{{ $json._pageId }}/children` |
| Authentication | Notion API |
| Headers | `Notion-Version: 2022-06-28` |
| Body | `{ "children": {{ JSON.stringify($json._blocks) }} }` |

> n8n Notion 노드의 "Append Block" 기능이 지원되면 그 노드 사용. 안 되면 HTTP Request로 직접.

### G-10. Notion pages.update (속성 마감)

| 필드 | 값 |
|------|---|
| Node type | **Notion** |
| Resource | Database Page |
| Operation | Update |
| Page ID | `{{ $json._pageId }}` |
| Properties |  ▾ |

| Property | Value |
|---------|-------|
| Summary | `{{ $json.summary_short }}` |
| Author | `{{ $json.author }}` |
| Tags | `{{ $json.tags }}` (Multi-select, 배열) |
| IsMetadataFilled | `true` |
| ErrorNote | `{{ $json._parse_error || '' }}` |

> 파싱 실패 시 `IsMetadataFilled = false` 유지 + ErrorNote만 기록 → 다음 실행에서 재시도.
> 이렇게 분기하려면 IF 노드로 `_parse_error` 유무 체크.

### G-11. Aggregate digest

```javascript
const items = $input.all();
const success = items.filter((it) => !it.json._parse_error);
const failed = items.filter((it) => it.json._parse_error);

const blocks = success.slice(0, 30).map((it) => ({
  type: 'section',
  text: {
    type: 'mrkdwn',
    text: `*<{{ NOTION_PAGE_URL }}|${it.json._title}>* — ${it.json.summary_short}\n_<${it.json.url}|원문>_`,
  },
}));

return [{
  json: {
    digest_blocks: blocks,
    summary_text: `:newspaper: feed-it 요약 다이제스트 (${$now.setZone('Asia/Seoul').toFormat('yyyy-MM-dd HH:mm')} KST)\n신규 ${success.length}건 / 실패 ${failed.length}건`,
  },
}];
```

### G-12. Slack chat.postMessage (다이제스트)

| 필드 | 값 |
|------|---|
| Node type | **Slack** |
| Resource | Message |
| Operation | Post |
| Channel | `{{ $env.SLACK_DIGEST_CHANNEL_ID }}` |
| Text | `{{ $json.summary_text }}` |
| Blocks | `{{ $json.digest_blocks }}` (Q7 결정 후 — Block Kit 카드 vs 단순 텍스트) |

> Q7 (Slack 포맷)은 게이트 ③에서 sample 보고 결정. 기본은 위 Block Kit `section` 리스트.
> 30건 초과 시 메시지 분할 (Slack 50 blocks 제한, conventions §7.6).

### G-13. (선택) 실패 건 별도 알림

만약 G-7에서 파싱 실패가 있었다면:

| 필드 | 값 |
|------|---|
| Node type | **IF** + **Slack** |
| IF | `{{ $json._failed_count > 0 }}` |
| Slack channel | `{{ $env.SLACK_ALERTS_CHANNEL_ID }}` |
| Text | `:warning: 요약 실패 {{ count }}건. Notion 미요약 row로 남음 — 다음 실행에서 재시도.` |

---

## 에러 처리 정책

- **워크플로 settings → Error Workflow = error-monitor**
- 노드별 `Continue On Fail`:
  - G-4 (Firecrawl): 실패 시 ErrorNote 기록 후 다음 row
  - G-6 (Claude): 실패 시 IsMetadataFilled=false 유지 + 다음 row
  - G-9·G-10 (Notion write): 실패 시 alerts에 보고 + 다음 row
- 같은 row가 3회 연속 요약 실패하면 별도 처리: ErrorNote에 `[FAILED_3X]` prefix + 사용자 확인 요청 (Week 2+)

---

## 1500자 분할 검증 (정책 회귀 — conventions §1.4)

```
가장 긴 요약 1건 page block 개수·길이 확인:
1. Notion에서 가장 긴 요약 row 1개 열기
2. blocks 개수 = ceil(summary_long.length / 1500)
3. 각 block의 rich_text.content 길이 ≤ 1500
   (Notion API가 2000자 초과 거부 — 1500이 안전 마진)
```

PASS 시 게이트 ③ 통과.

---

## Slack 다이제스트 포맷 옵션 (Q7 — 게이트 ③에서 결정)

옵션 A — Block Kit `section` 리스트 (위 G-11 기본)
- 각 글마다 1 section: 제목·요약·원문 링크
- 깔끔. 모바일에서 약간 길어짐
- 최대 50 blocks

옵션 B — 단순 텍스트 리스트
```
:newspaper: feed-it 다이제스트 (2026-05-27)
신규 N건

• <https://...|글 제목 A> — 한줄 요약 (Channel)
• <https://...|글 제목 B> — ...
```
- 더 가벼움. 30건 넘어도 1 메시지에 들어갈 가능성
- Block Kit 카드보다 시각적 정보 적음

게이트 ③에서 sample 보고 선택 (A 권장).

---

## credential 의존성

- Anthropic, Firecrawl, Notion, Slack (Workflow A와 동일)

---

## 검증 (G-13 이후)

- [ ] 수동 트리거 (Manual) 1회 → Notion 미요약 row 1개 처리 확인
- [ ] Notion 페이지 본문에 한국어 요약 단락 적재됨
- [ ] Summary 속성 250자 한줄 채워짐
- [ ] Tags Multi-select 3~5개 정상
- [ ] IsMetadataFilled = true 로 마감
- [ ] Slack `#feed-it-digest`에 다이제스트 도착 + 포맷 결정 (Q7)
- [ ] 1500자 분할 회귀 검증 (위 §1500자 검증)

---

## Export + commit

```powershell
git add workflows/summarize.json
git commit -m "feat: Workflow B (summarize) v1 — Notion 미요약 → 한국어 요약 → Slack 다이제스트"
git push
```

---

## 알려진 함정

- G-2 Notion `IsMetadataFilled=false` 필터: Checkbox 속성 명칭 정확히 일치해야 함 (대소문자)
- G-6 max_tokens 1500: 너무 작으면 summary_long 잘림. 큰 본문일수록 더 큰 max_tokens 필요할 수 있음 — 게이트 ③ 후 조정
- G-7 한국어 글자 수 vs 바이트: `summary_short.length`는 JavaScript 문자 수 (한글 1자 = 1) — 250자 검증 OK
- G-9 Notion API 페이지 본문 append 시 한 번에 최대 100 children — 큰 분할은 chunked
- G-12 Slack Block Kit 50 blocks 한계: 30건 초과 시 메시지 분할

---

## TODO (다음 회차)

- [ ] Workflow C (error-monitor) 설계서 → `docs/workflow-error-monitor.md`
- [ ] G-12 Block Kit JSON 정확한 스키마 검증 (Slack API docs)
- [ ] G-11에서 NOTION_PAGE_URL 동적 생성 (workspace ID + DB ID + page ID 조합)
- [ ] G-13 "3회 연속 실패" 추적 로직 (Notion 별도 카운터 속성 추가 검토)
