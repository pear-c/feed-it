# Workflow C — ErrorMonitor (n8n 노드 설계서)

> 다른 모든 워크플로(collect, summarize, 미래 추가될 워크플로)의 실패를 한 곳에서 받아 Slack `#feed-it-alerts`에 알림. 노드 4개로 끝나는 짧은 워크플로.

---

## 개요

| 항목 | 값 |
|------|---|
| 워크플로명 | `error-monitor` |
| 트리거 | **Error Trigger** (전역 — 다른 워크플로 settings에서 `Error Workflow`로 지정) |
| 목적 | 워크플로 실패 컨텍스트 수집 + Slack alerts 전송 |
| credential | Slack |
| 예상 실행 시간 | < 5초 |

---

## 데이터 흐름

```
Error Trigger
   │  (다른 워크플로 실패 시 자동 발화)
   ▼
Format error message     (Code Node)
   │  → 워크플로명·실패 노드·에러 메시지·실행 URL 포맷팅
   ▼
Slack chat.postMessage   → #feed-it-alerts
```

---

## 노드별 상세 (3개)

### H-1. Error Trigger

| 필드 | 값 |
|------|---|
| Node type | **Error Trigger** |

이 노드 자체는 별도 파라미터 없음. n8n이 자동으로 입력 데이터에 다음을 채워줌:

```json
{
  "execution": {
    "id": "12345",
    "url": "https://localhost:5678/workflow/abc/executions/12345"
  },
  "workflow": {
    "id": "abc",
    "name": "collect"
  },
  "trigger": {
    "module": "Schedule Trigger"
  },
  "lastNodeExecuted": "Firecrawl_scrape",
  "executionError": {
    "message": "Connection timeout",
    "stack": "..."
  }
}
```

### H-2. Format error message

| 필드 | 값 |
|------|---|
| Node type | **Code** |

```javascript
const data = $input.first().json;
const workflowName = data.workflow?.name || 'unknown';
const failedNode = data.lastNodeExecuted || 'unknown';
const errMsg = data.executionError?.message || '(no message)';
const executionUrl = data.execution?.url || '';

const text = [
  `:rotating_light: *feed-it 워크플로 실패*`,
  `*워크플로*: \`${workflowName}\``,
  `*실패 노드*: \`${failedNode}\``,
  `*시각*: ${new Date().toLocaleString('ko-KR', { timeZone: 'Asia/Seoul' })} KST`,
  `*에러*: \`${errMsg.slice(0, 500)}\``,
  executionUrl ? `*디버깅*: <${executionUrl}|n8n Execution>` : '',
].filter(Boolean).join('\n');

return [{ json: { text } }];
```

> stack trace는 길고 민감 정보 포함 가능. 첫 줄 메시지만 사용 (500자 cap).

### H-3. Slack chat.postMessage

| 필드 | 값 |
|------|---|
| Node type | **Slack** |
| Resource | Message |
| Operation | Post |
| Channel | `{{ $env.SLACK_ALERTS_CHANNEL_ID }}` |
| Text | `{{ $json.text }}` |
| Other Options | `mrkdwn: true` |

---

## 다른 워크플로에 적용

Workflow A·B 각각의 settings에서:

1. n8n UI → Workflow 우상단 ⋮ → **Settings**
2. **Error Workflow** → `error-monitor` 선택
3. Save

이렇게 하면 A·B에서 어떤 노드가 실패하든 자동으로 H-1이 발화.

> Error Trigger를 가진 워크플로가 본인 자체에서 에러 나면 무한 루프? n8n은 보호 — Error Workflow가 실패하면 두 번째 알림은 안 발화하고 로그에만 기록.

---

## 검증

- [ ] 의도적 실패 1건 발생 (예: collect의 Firecrawl 노드 URL을 잘못된 값으로 임시 변경 → 실행)
- [ ] `#feed-it-alerts`에 위 포맷의 메시지 도착
- [ ] 메시지 안의 "n8n Execution" 링크 클릭 → 실패한 Execution 페이지 열림
- [ ] 원래 URL 복원 후 정상 실행 확인

---

## credential 의존성

- Slack (Workflow A·B와 공유)

---

## Export + commit

```powershell
git add workflows/error-monitor.json
git commit -m "feat: Workflow C (error-monitor) v1 — 전역 에러 → Slack alerts"
git push
```

---

## 알려진 함정

- **첫 1회 호출**: Error Workflow를 다른 워크플로에 등록한 직후 첫 실패는 종종 누락될 수 있음 — n8n 재시작 1회 권장 (`docker compose restart n8n`)
- **자기 자신 hook**: error-monitor 자체가 실패하면? 이건 n8n이 내부 로그에만 기록. 워크플로 디자인 후 1회 의도적 실패 테스트로 정상 동작 확인 필수
- **민감 정보 노출**: H-2 Code Node에서 stack trace·환경변수가 Slack에 노출될 수 있음 → `.slice(0, 500)` + stack 제외로 보호 (위 코드 적용됨)
- **Slack rate**: 동일 워크플로가 짧은 시간에 다수 실패 시 알림 폭주 가능 — Week 2+에 throttle 검토 (예: 같은 워크플로·노드 조합은 5분 내 1회만)

---

## TODO (다음 회차)

- [ ] Error 분류별 채널 분리 검토 (4xx vs 5xx vs network — Week 2+)
- [ ] 같은 에러 반복 시 throttle (예: 같은 (workflow, node) 5분 내 1회만)
- [ ] PagerDuty 또는 이메일 연동 (out-of-hours 알림 — Week 3+)
