# feed-it 운영 가이드

> 셋업 끝난 뒤 일상 운영에서 자주 마주칠 절차들. 매일 보는 문서.
>
> 셋업이 아직이면: [`setup.md`](setup.md)

---

## 일상 흐름

```
[월/수/토 09:00 KST]   PC가 켜져 있고 docker n8n이 running
  ↓
  Workflow A (collect) 자동 실행
  → 5~15분
  → Slack #feed-it-digest: 수집 결과
  ↓
  Workflow B (summarize) 자동 실행 (A가 호출)
  → 10~30분
  → Slack #feed-it-digest: 요약 다이제스트
```

> ⚠️ **PC가 꺼져 있으면 그 회차는 누락**. 다음 월/수/토까지 대기 (cron은 보류 실행 안 함).

---

## 자주 하는 작업

### 1. 수동으로 한 번 더 돌리기

n8n UI에서:

1. 좌측 `Workflows` → `collect` 선택
2. 우상단 `Execute Workflow` 클릭
3. 진행 상황은 우측 `Executions` 패널 또는 좌측 `Executions` 페이지

또는 PowerShell에서 n8n CLI:

```powershell
docker exec -it feed-it-n8n n8n execute --id <workflow-id>
```

### 2. 특정 채널만 시험 실행

`channels.yaml`에서 다른 채널들을 일시 비활성화:

```yaml
- name: anthropic-news
  enabled: true   # 이 채널만 true

- name: openai-news
  enabled: false  # 나머지 false
```

수동 실행 → 결과 확인 → 원상 복구.

### 3. Firecrawl quota 확인

브라우저: <https://firecrawl.dev/app> → Usage

또는 PowerShell:
```powershell
$env:FIRECRAWL_API_KEY = "fc-..."
curl.exe -H "Authorization: Bearer $env:FIRECRAWL_API_KEY" https://api.firecrawl.dev/v1/team/credit-usage
```

> 80% 도달 시 Slack 경고 자동 (Workflow A의 quota 보호 노드). 100% 도달 시 워크플로 자동 중단.

### 4. Anthropic 사용량 / 비용 확인

브라우저: <https://console.anthropic.com> → Usage

> Sonnet 4.6 한국어 요약 1건 평균 ≈ 입력 5K + 출력 1K 토큰 ≈ $0.018. 주 3회 × 신규 30건 = 월 ~$2.

### 5. Notion DB 정리

#### 오래된 row 아카이브

Notion DB → Filter `CollectedAt` is before `3 months ago` → 선택 → 우클릭 → `Move to` (별도 아카이브 DB) 또는 삭제.

#### 요약 실패 row 재시도

Filter `IsMetadataFilled = false` AND `CollectedAt > 1 day ago` → row가 있으면 자동 재시도 대상 (Workflow B가 다음 실행 시 다시 처리). 1일 넘으면 수동 검토 권장:

- ErrorNote에 메시지가 있는지
- URL이 유효한지 (방문해서 확인)
- 영구 실패면 row 삭제 또는 `IsMetadataFilled = true` 수동 토글 (skip 처리)

### 6. 신규 채널 추가

1. `channels.yaml` 끝에 새 entry 추가 (`enabled: false`로 시작)
2. robots.txt + 이용약관 확인 (이 사이트가 자동 수집 허용하는지)
3. n8n에서 수동 실행 1회 (해당 채널만 enabled: true) → 결과 확인
4. 정상이면 `enabled: true` 유지하고 commit·push
5. (newsletter 채널이면) `extract_urls.md`의 newsletter 분기가 잘 처리하는지 확인

### 7. 채널 일시 비활성화

`channels.yaml`에서 `enabled: false`로 변경 → commit. 다음 실행부터 skip.

n8n 재시작 불필요 (Read From File이 매 실행마다 디스크에서 새로 읽음).

### 8. Slack 채널 무음

`#feed-it-digest`가 시끄러우면:
- 채널 옆 종 아이콘 → `Mute channel` → `For 24 hours` / `Until I turn it on`
- 또는 `Notification preferences` → 키워드만 알림

`#feed-it-alerts`는 절대 mute 금지 (에러 누락).

### 9. 워크플로 일시 중단

n8n UI → Workflow `collect` → 우상단 토글 OFF.

다음 cron 시각에 자동 실행 안 됨. 다시 켜려면 토글 ON.

### 10. 워크플로 변경 후 export

```powershell
# n8n UI에서 워크플로 수정 → 우상단 ⋮ → Download → JSON
# 저장: D:\workspace-bsh\feed-it\workflows\<workflow-name>.json
git -C D:\workspace-bsh\feed-it add workflows/<workflow-name>.json
git -C D:\workspace-bsh\feed-it commit -m "feat: <workflow-name> v<N> — <변경 요약>"
git -C D:\workspace-bsh\feed-it push
```

> conventions.md §7.10: 운영 중 워크플로 수정하고 export 잊으면 stale. 변경 후 즉시 export·commit.

---

## 트러블슈팅

### 워크플로가 cron 시각에 안 돌았다

체크 순서:
1. PC가 그 시각에 켜져 있었나? (`Get-WinEvent -LogName System | Where-Object {$_.Id -eq 1 -or $_.Id -eq 12}` 등)
2. Docker Desktop 동작 중? (`docker ps`)
3. n8n 컨테이너 살아있나? (`docker ps | findstr feed-it-n8n`)
4. 워크플로가 active 상태인가? (n8n UI 토글)
5. Schedule Trigger 노드 timezone이 `Asia/Seoul`인가? (UTC면 9시간 차이)
6. n8n Executions 페이지 로그 확인 (실행 시도는 있었는데 실패했는지)

### Notion에 중복 row가 생겼다

URL 정규화 안 됐을 가능성:
1. F-11 (URL 정규화) 코드 노드 동작 확인
2. 중복 row의 URL 직접 비교 — trailing slash·UTM·host 대소문자 차이?
3. 다른 점이 있으면 정규화 로직에 케이스 추가

### Slack 다이제스트가 안 온다

1. Workflow B 실행 됐나? (Executions 페이지)
2. Slack credential Test 통과?
3. Bot이 채널에 invite 됐나? (`/invite @feed-it` 다시)
4. 채널 ID 정확한가? (`.env`의 `SLACK_DIGEST_CHANNEL_ID`)
5. 워크플로 안의 `$env.SLACK_DIGEST_CHANNEL_ID` 참조가 정상 평가되나? n8n 환경변수 expression 문법 확인

### Firecrawl이 400/403 응답

- 400: URL 형식 오류. 정규화 단계 결과 확인
- 403: Cloudflare 등 봇 차단. Firecrawl이 우회 못 한 케이스. 해당 채널 일시 disable

### Claude가 매번 다른 형식 반환

- 시스템 프롬프트 self-check 섹션이 잘 작동 안 함
- 임시: `max_tokens` 늘려서 self-correction 여유 더 주기
- 영구: 프롬프트 v2 작성 — 더 강한 형식 강제 + few-shot 예시 추가

### n8n 컨테이너 갑자기 죽음

```powershell
docker compose logs n8n --tail 100
docker compose restart n8n
```

자주 발생하면:
- Docker Desktop 메모리 할당 늘리기 (Settings → Resources → 최소 4GB)
- n8n 영구 볼륨 디스크 사용량 확인 (`./data` 폴더 크기)

---

## 정기 점검 (월 1회)

- [ ] Firecrawl 잔여 quota + 결제 정보 확인
- [ ] Anthropic 누적 사용량 + 비용 추세
- [ ] Notion DB row 수 + 가장 오래된 row 날짜 (아카이브 검토)
- [ ] Slack `#feed-it-digest` 마지막 메시지 시각 (워크플로 정상 동작 증거)
- [ ] `channels.yaml` 채널별 수집 빈도 — 0건 채널은 enable 조정 검토
- [ ] git log 1주일 — 운영 중 워크플로 수정이 commit 안 된 것 있는지

---

## 백업

- **n8n 영구 볼륨**: `./data/` 폴더 (credential·실행 이력 포함). 정기 백업 권장
  ```powershell
  Compress-Archive -Path D:\workspace-bsh\feed-it\data\* -DestinationPath D:\workspace-bsh\feed-it-backups\data-$(Get-Date -Format yyyyMMdd).zip
  ```
- **Notion DB**: Notion 자체 export (Settings → Export workspace) — 분기별
- **`.env`**: 별도 password manager (1Password · Bitwarden 등)에 저장. 절대 git에 X

---

## Week 2 진입 결정 (PLAN §10)

운영 1~2주 후 다음 질문에 답:

| 질문 | 진입 OK 신호 |
|------|--------------|
| 도착하는 콘텐츠 중 실제 읽는 비율은? | ≥ 50% |
| n8n UI 운영에서 가장 불편한 지점은? | 명확한 답 있음 (테스트성·diff·재실행 분기 등) |
| 추가하고 싶은 단계가 떠올랐나? | yes (태깅·번역·중요도 점수 등) |
| 채널 30~50개 확장 시 n8n 부담은? | 구체적 추정 가능 |

답이 모이면 `.claude/tasks/feed-it-week2/`에 PLAN/CONTEXT/CHECKLIST 신설.
모이지 않으면 Week 1.5로 채널 확장·프롬프트 튜닝만.

---

## 참고

- [setup.md](setup.md) — 0부터 첫 실행까지
- [workflow-collect.md](workflow-collect.md) — Workflow A 설계서
- [workflow-summarize.md](workflow-summarize.md) — Workflow B 설계서
- [workflow-error-monitor.md](workflow-error-monitor.md) — Workflow C 설계서
- 벤치마크 원글: <https://insight.infograb.net/blog/2025/12/17/n8n-devops-contents/>
