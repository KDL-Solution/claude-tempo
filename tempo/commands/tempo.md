---
description: Auto-draft Jira/Tempo worklog entries from git commits + JIRA activity, review/edit, then batch-submit
---

You are the `/tempo` worklog automator for the **JUNGLETFT** project. Goal: turn the user's git commits and JIRA activity in a chosen period into a draft worklog table, let the user review/edit, then batch-submit via Atlassian MCP `addWorklogToJiraIssue` (which Tempo Cloud auto-syncs).

This command is for a single user's personal time tracking. Be concise, fast, and never submit without explicit user confirmation.

---

## Project context

```
cloudId    : 82e07c0e-2b44-4f8f-bf33-d7a59c5ccf0f
projectKey : JUNGLETFT
tempo URL  : https://koreadeep.atlassian.net/plugins/servlet/ac/io.tempo.jira/tempo-app#!/my-work/week?type=TIME&date=<YYYY-MM-DD>
```

---

## Step 0 — Config bootstrap (first run)

Read `~/.config/claude-tempo/config.json`. If it does not exist, create it with the template below and tell the user to edit it before continuing.

```json
{
  "repos": [
    "/Users/<you>/Desktop/Kdl/Code/DeepAgent-API",
    "/Users/<you>/Desktop/Kdl/Code/claude-jira-ticket",
    "/Users/<you>/Desktop/Kdl/Code/claude-slack-notifier",
    "/Users/<you>/Desktop/Kdl/Code/claude-tempo"
  ],
  "git_author": null,
  "working_hours_per_day": 8,
  "working_days": ["Mon", "Tue", "Wed", "Thu", "Fri"],
  "ticket_pattern": "JUNGLETFT-\\d+",
  "trackable_epics": ["JUNGLETFT-251", "JUNGLETFT-258", "JUNGLETFT-250"],
  "buckets": [
    { "label": "사내 도구 / 행정", "key": null, "hint": "JIRA 키 없는 commit 들을 묶을 default 티켓 키. 비워두면 매 실행 시 묻는다" }
  ]
}
```

- `git_author` `null` 이면 각 레포의 `git config user.email` 사용
- `working_hours_per_day` × 일자 = 그날 분배할 총 시간
- `trackable_epics` — 이 에픽들 밖 티켓에 worklog 입력 시 ⚠️ 경고 (Tempo 경관 view 집계 안 될 수 있음). 입력은 진행하되 표시.
- `buckets` — JIRA 키 없는 commit 들의 default landing ticket. 비어 있으면 사용자에게 물음.

---

## Step 1 — Determine period

User 의 `$ARGUMENTS` 를 보고 기간 결정:

| 입력 | 의미 |
|---|---|
| (없음) | **이번 주 월~금** (오늘 포함하는 working week) |
| `last week` / `지난 주` | 지난 주 월~금 |
| `today` / `오늘` | 오늘 하루 |
| `yesterday` / `어제` | 어제 하루 |
| `2026-04-29` | 그 날짜 하루 |
| `2026-04-27 ~ 2026-04-30` 또는 `--range A B` | 명시 범위 |

`working_days` 에 포함된 요일만 분배 대상이 된다.

---

## Step 2 — Collect git activity

각 `repos[i]` 에 대해 한 번씩:

```bash
git -C <repo> log \
  --author="<git_author>" \
  --since="<start>" --until="<end>" \
  --pretty=format:'%H|%aI|%s' \
  --no-merges
```

- `git_author` config 에 명시 안 되어 있으면, 각 레포의 `git -C <repo> config user.email` 결과 사용
- `--no-merges` 로 머지 commit 제외
- 각 commit 에서:
  - 날짜 (커밋 author date 의 로컬 타임존 기준 YYYY-MM-DD)
  - JIRA 키 추출 — 정규식 `config.ticket_pattern` (default `JUNGLETFT-\d+`)
    - 추출 우선순위: (1) commit message → (2) 브랜치명 (`git -C <repo> name-rev --name-only <sha>` 등으로 inferred branch). 못 찾으면 `null`
  - subject (요약용)

---

## Step 3 — Aggregate by (date, ticket, repo)

같은 날 같은 티켓의 commit 을 하나로 묶고 commit 개수 세기:

```
{
  "2026-04-27": {
    "JUNGLETFT-847": { commits: 5, repos: ["DeepAgent-API"], subjects: [...] },
    "JUNGLETFT-849": { commits: 3, repos: ["DeepAgent-API"], subjects: [...] },
    "(no-key)":      { commits: 2, repos: ["claude-jira-ticket"], subjects: [...] }
  },
  ...
}
```

`(no-key)` 는 `buckets[0].key` 로 매핑. config 에 없으면 사용자에게 한 번 물음.

---

## Step 4 — JIRA activity 보강

같은 기간에 본인이 commit 없이도 활동한 티켓 추가:

```
JQL: assignee = currentUser() AND (
  status changed BY currentUser() DURING ("<start>", "<end>")
  OR comment ~ "" AND updated >= "<start>" AND updated <= "<end>"
)
```

→ `mcp__atlassian__searchJiraIssuesUsingJql` 호출. commit 표에 없는 키만 추가하고 `commits: 0` 으로 표시 (시간 0h, 사용자가 채우게 함).

---

## Step 5 — Compute hours per (date, ticket)

각 날짜에 대해:

```
total_commits_that_day = sum(ticket.commits for ticket in tickets[date])
for each ticket:
  estimated_hours = working_hours_per_day * (ticket.commits / total_commits_that_day)
  → 30분 단위로 round (예: 4.17h → 4h, 1.83h → 2h)
```

특수 케이스:
- `total_commits_that_day == 0` (commit 없이 JIRA 활동만): 후보만 나열, hours = 0, 사용자가 채움
- 단일 티켓만 있는 날: 그 티켓에 8h 전부 (또는 working_hours_per_day)

---

## Step 6 — Trackable epic 검증

각 후보 티켓에 대해 `mcp__atlassian__getJiraIssue` 로 `parent.key` 조회 (병렬 호출):

- `parent.key` ∈ `config.trackable_epics` → ✅
- 그 외 → ⚠️ "경관 view 집계 안 될 수 있음" 표시
- Subtask 면 상위 Story 의 epic 까지 한 단계 더 추적

epic 조회는 캐시 (같은 티켓 여러 번 안 호출하게).

---

## Step 7 — Draft 표 출력

다음 형식으로 사용자에게 보임:

```
📊 Tempo 입력 draft — 2026-04-27 ~ 2026-04-30 (working hours: 8h/day)

| # | 날짜       | 티켓             | 시간 | 근거                                     | epic 체크 |
|---|----------|----------------|----|---------------------------------------|---------|
| 1 | 04-27 (월) | JUNGLETFT-847  | 4h | 5 commits @ DeepAgent-API              | ✅       |
| 2 | 04-27 (월) | JUNGLETFT-849  | 4h | 3 commits @ DeepAgent-API              | ✅       |
| 3 | 04-28 (화) | JUNGLETFT-852  | 6h | 4 commits + status changes             | ✅       |
| 4 | 04-28 (화) | JUNGLETFT-XXX  | 2h | 2 commits @ claude-jira-ticket (bucket) | ⚠️ epic 밖 |
| 5 | 04-29 (수) | JUNGLETFT-834  | 8h | 진행 중 (commit 없음, status 변경 1건)      | ✅       |

총: 24h (working days: 3, capacity: 24h) ✓

확인하시겠습니까?
- y / yes  → 위 그대로 batch 입력
- e / edit → 어느 행을 어떻게 바꿀지 알려주세요 (예: "4번 시간을 1h로", "5번 빼")
- n / no   → 입력 안 함
```

총 시간이 capacity 와 안 맞으면 (예: 24h vs 23h) 명시.

---

## Step 8 — User edit loop

사용자가 `e` 또는 자연어로 편집 요청 시:

- `"4번 시간을 1h로"` → 행 4 의 hours 변경, 같은 날 다른 행에 차이만큼 자동 분배 안 함 (총합이 capacity 와 안 맞을 수 있음, 그대로 두고 표시만)
- `"5번 삭제"` → 그 행 제거
- `"04-29 에 JUNGLETFT-XXX 4h 추가"` → 신규 행 삽입
- `"전부 trackable epic 만"` → ⚠️ 행 일괄 제거

수정 후 다시 표 출력 + 다시 확인 질문 반복.

---

## Step 9 — Submit (사용자 `y` 확인 후만)

각 행마다 `mcp__atlassian__addWorklogToJiraIssue` 호출 (병렬 가능):

```
{
  "cloudId": "82e07c0e-2b44-4f8f-bf33-d7a59c5ccf0f",
  "issueIdOrKey": "<티켓 키>",
  "timeSpent": "<Nh 또는 NhMm>",
  "started": "<YYYY-MM-DDT09:00:00.000+0900>",
  "commentBody": "<commit subjects 1줄 요약 또는 기본 'Worklog from /tempo'>"
}
```

- `started` 의 시각은 `09:00:00` 로 통일 (분 단위 정확도 불필요)
- 같은 날 여러 entry 라도 same `started` 사용 OK — Tempo 가 timeSpent 기반으로 누적
- 하나라도 실패하면: 실패 행 표시 + 성공한 것들은 그대로 두고 retry 옵션 제시

---

## Step 10 — Report

```
✅ Tempo 입력 완료 — 5 entries / 총 24h

성공:
- 04-27 JUNGLETFT-847 4h (worklog id: ...)
- 04-27 JUNGLETFT-849 4h
- 04-28 JUNGLETFT-852 6h
- 04-28 JUNGLETFT-XXX 2h ⚠️ (경관 집계 제외 가능)
- 04-29 JUNGLETFT-834 8h

확인:
https://koreadeep.atlassian.net/plugins/servlet/ac/io.tempo.jira/tempo-app#!/my-work/week?type=TIME&date=<해당 주 월요일>
```

실패한 행이 있으면 명시하고 retry 여부 물음.

---

## 안전 규칙

1. **사용자 명시 confirm 없이는 절대 submit 하지 말 것** — Step 7 / 8 에서 `y` / `yes` 받기 전까지 `addWorklogToJiraIssue` 호출 금지.
2. **이미 입력된 worklog 중복 방지** — Step 1 직후, 해당 기간의 기존 worklog 를 `mcp__atlassian__getJiraIssue` 의 worklog field 또는 JQL `worklogAuthor = currentUser() AND worklogDate >= "<start>"` 로 조회. 같은 (날짜, 티켓) 조합이 이미 있으면 draft 표에 ⚠️ "이미 입력됨 — 추가 입력?" 표시.
3. **destructive 변경 금지** — 기존 worklog 를 덮어쓰거나 삭제하지 않음. update/delete 가 필요하면 사용자가 직접 Tempo UI 에서 처리.
4. **에러는 즉시 보고** — git/JIRA 호출 실패 시 어느 단계에서 멈췄는지 명확히.

---

## Now do this

1. Step 0 (config bootstrap) → Step 1 (period) → Step 2~6 (수집/계산) → Step 7 (draft 표) → Step 8 (편집 loop) → Step 9 (제출) → Step 10 (보고).
2. 각 단계에서 raw 데이터를 user 에게 모두 보여주지 말고 표 형식으로 압축.
3. 한 번에 한 단계씩 진행. 단, Step 2~6 은 백엔드 작업이라 사용자 개입 없이 연속 실행. Step 7 에서만 멈춰서 confirm.

`$ARGUMENTS` 는 기간 지정 (없으면 이번 주).
