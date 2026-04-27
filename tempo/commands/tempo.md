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

## Step 0 — Config bootstrap

Read `~/.config/claude-tempo/config.json`.

### Case A — Config 가 이미 있음
그대로 사용. 아래 Step 1 로 진행.

단, `$ARGUMENTS` 에 `reconfigure` / `--reconfigure` / `--reset` / `setup` 이 포함되면 → Case B 의 부트스트랩을 강제 재실행.

### Case B — Config 가 없음 (또는 reconfigure 요청) — **자동 감지 + 인터랙티브 부트스트랩**

사용자가 JSON 을 직접 편집하지 않게 한다. 다음을 자동 수행:

**B-1. KDL 레포 자동 감지**

다음 위치들을 스캔 (`shopt -s nullglob` 효과를 위해 안전 처리):

```bash
for parent in ~/Desktop/Kdl/Code ~/Code ~/Workspace ~/projects ~/work ~/dev ~/Documents/Code; do
  [ -d "$parent" ] || continue
  for git_dir in "$parent"/*/.git; do
    [ -e "$git_dir" ] || continue
    repo=$(dirname "$git_dir")
    remote=$(git -C "$repo" config --get remote.origin.url 2>/dev/null)
    case "$remote" in
      *KDL-Solution*|*koreadeep*) echo "$repo" ;;
    esac
  done
done | sort -u
```

→ KDL-Solution 또는 koreadeep 이 origin URL 에 포함된 git 레포 목록.

**B-2. git author 자동 감지**

```bash
git config --global user.email
```

→ 비어 있으면 (드물게) 각 레포의 `git config user.email` 중 가장 자주 등장하는 값 사용. 그것도 없으면 사용자에게 한 번 물음.

**B-3. 감지 결과 표 출력 + 한 번 확인**

```
🔍 Auto-detected:

git author: jude@koreadeep.com  (from ~/.gitconfig)

KDL repos found (3):
  1. /Users/jude/Desktop/Kdl/Code/DeepAgent-API
  2. /Users/jude/Desktop/Kdl/Code/claude-jira-ticket
  3. /Users/jude/Desktop/Kdl/Code/claude-tempo

Defaults:
- working hours/day:    8h
- working days:          Mon~Fri
- ticket pattern:        JUNGLETFT-\d+
- trackable epics:       JUNGLETFT-251 / 258 / 250
- bucket (no-key):       (없음 — JIRA 키 없는 commit 은 매번 묻습니다)

이 설정으로 진행하시겠습니까?
- y / yes        → 그대로 저장하고 이번 주 draft 시작
- e / edit       → 자연어로 변경 사항 알려주세요 (예: "5번 빼", "/Users/jude/Code/foo 추가", "bucket key 를 JUNGLETFT-900 으로")
- m / manual     → JSON 편집기로 직접 (기본 template 만 만들고 안내 메시지)
```

**B-4. 사용자 응답 처리**

- `y` → B-3 에서 보여준 값 그대로 `~/.config/claude-tempo/config.json` 에 저장 (`mkdir -p ~/.config/claude-tempo` 먼저)
- `e` → 자연어 편집 받아서 반영 후 다시 B-3 표 출력 + 재확인 루프
- `m` → 기본 template (placeholder 포함) 만 저장하고 "편집 후 다시 /tempo 실행" 안내. 종료.

**B-5. 저장 후 진행**

저장 완료 메시지 (1줄) + Step 1 으로 자동 이어짐. "다시 /tempo 실행하라" 같은 redirect 금지 — 한 turn 안에 부트스트랩 + 본 작업까지 끝낸다.

저장 포맷 (참고):
```json
{
  "repos": ["/Users/jude/Desktop/Kdl/Code/DeepAgent-API", ...],
  "git_author": "jude@koreadeep.com",
  "working_hours_per_day": 8,
  "working_days": ["Mon", "Tue", "Wed", "Thu", "Fri"],
  "ticket_pattern": "JUNGLETFT-\\d+",
  "trackable_epics": ["JUNGLETFT-251", "JUNGLETFT-258", "JUNGLETFT-250"],
  "buckets": [
    { "label": "사내 도구 / 행정", "key": null }
  ]
}
```

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
