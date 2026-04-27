# claude-tempo

매주 매일 Tempo 에 시간 입력하기 귀찮은 사람을 위한 Claude Code 플러그인.

`/tempo` 한 번이면 git commit + JIRA 활동 → 일자/티켓별 worklog draft 자동 생성 → 본인 검토/편집 → batch 입력. Jira native worklog 로 입력하면 Tempo Cloud 가 자동 동기화하므로 별도 Tempo API 토큰 불필요.

---

## Quick Start

### 1. 사전 요구사항

- **Atlassian MCP 연결** (Claude Code → ⌘⇧P → `MCP: Add Server` → Atlassian)
- **로컬 git 레포** — 작업 commit 들이 있는 곳
- (Tempo 경관 view 집계용으로) 작업 티켓이 **특정 에픽** 의 자식이어야 함 — 아래 config `trackable_epics` 참고

### 2. 설치

```
/plugin marketplace add https://github.com/KDL-Solution/claude-tempo.git
/plugin install tempo@claude-tempo
```

재시작 후 `/tempo` 자동완성 확인.

### 3. 첫 실행 + config 작성

```
/tempo
```

처음 실행 시 `~/.config/claude-tempo/config.json` 이 자동 생성됨. 본인 환경에 맞게 편집:

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
    { "label": "사내 도구 / 행정", "key": null, "hint": "JIRA 키 없는 commit 들을 묶을 default 티켓 키" }
  ]
}
```

| 키 | 설명 |
|---|---|
| `repos` | 매주 commit 스캔할 로컬 git 레포 목록 (절대 경로) |
| `git_author` | `null` 이면 각 레포의 `git config user.email` 사용. 명시하면 그 author 로 필터 |
| `working_hours_per_day` | 하루 분배할 총 시간 (default 8h) |
| `working_days` | 분배 대상 요일. 주말 제외하려면 default 그대로 |
| `ticket_pattern` | commit message / 브랜치명에서 티켓 키 추출 정규식 |
| `trackable_epics` | 이 에픽들 밖 티켓에 worklog 입력 시 ⚠️ 경고. 입력은 진행하되 "경관 view 집계 안 될 수 있음" 표시 |
| `buckets[0].key` | JIRA 키 없는 commit 들의 default landing ticket. 비어 있으면 매번 물음 |

---

## 사용

### 기본 — 이번 주 (월~금)

```
/tempo
```

### 다른 기간

```
/tempo last week         # 지난 주
/tempo today             # 오늘 하루
/tempo yesterday         # 어제 하루
/tempo 2026-04-29        # 그 날짜 하루
/tempo 2026-04-27 ~ 2026-04-30   # 명시 범위
```

---

## 동작 흐름

1. **수집** — 각 레포에서 `git log --author=<나> --since=<start> --until=<end>` 실행, commit message / 브랜치명에서 `JUNGLETFT-XXX` 추출
2. **JIRA 보강** — 같은 기간에 commit 없이 status 만 바꾼 / 코멘트만 단 티켓도 후보에 추가
3. **시간 산정** — 그날 commit 개수 비례로 working_hours_per_day 분배 (30분 단위 round)
4. **에픽 검증** — 각 티켓의 parent epic 이 `trackable_epics` 에 있는지 확인
5. **Draft 표 출력** — 일자 × 티켓 × 시간 × 근거 + 에픽 체크 마크
6. **사용자 검토/편집** — `e` 로 자연어 편집 ("4번 시간을 1h로", "5번 삭제", "04-29 에 -XXX 4h 추가")
7. **`y` 확인 후 batch 입력** — `addWorklogToJiraIssue` 병렬 호출 (Tempo Cloud 가 자동 동기화)
8. **결과 보고** + Tempo URL

---

## 출력 예시

```
📊 Tempo 입력 draft — 2026-04-27 ~ 2026-04-30 (working hours: 8h/day)

| # | 날짜       | 티켓             | 시간 | 근거                                     | epic |
|---|----------|----------------|----|---------------------------------------|------|
| 1 | 04-27 (월) | JUNGLETFT-847  | 4h | 5 commits @ DeepAgent-API              | ✅   |
| 2 | 04-27 (월) | JUNGLETFT-849  | 4h | 3 commits @ DeepAgent-API              | ✅   |
| 3 | 04-28 (화) | JUNGLETFT-852  | 6h | 4 commits + status changes             | ✅   |
| 4 | 04-28 (화) | JUNGLETFT-900  | 2h | 2 commits @ claude-jira-ticket (bucket)| ⚠️    |
| 5 | 04-29 (수) | JUNGLETFT-834  | 8h | 진행 중 (commit 없음, status 변경 1건)     | ✅   |

총: 24h (working days: 3, capacity: 24h) ✓

확인? y / e / n
```

---

## 안전장치

- **명시 confirm 없이 입력하지 않음** — 표 검토 후 `y` 받아야만 submit
- **중복 방지** — 같은 (날짜, 티켓) 조합에 이미 worklog 있으면 ⚠️ 표시
- **수정/삭제 안 함** — 기존 worklog 는 절대 덮어쓰지 않음. 변경/삭제 필요하면 Tempo UI 에서 직접
- **에러 보고** — git/JIRA 호출 실패 시 어느 단계에서 멈췄는지 명확히

---

## 한계 (솔직)

- **시간 산정은 추정** — commit 개수 비례 분배. 실제 작업 시간과 차이날 수 있음 → 표 검토 시 본인이 편집
- **JIRA 키 없는 commit** — `buckets[0].key` 로 자동 매핑하거나 매번 사용자가 지정 (회의 / 문서 / 행정 등을 묶을 default ticket)
- **회의 / 문서 작업** — git 활동에 안 잡힘. `e` 편집으로 행 추가
- **Tempo 전용 필드** (Account, Activity 등) — 이 도구는 입력하지 않음. 필요하면 Tempo UI 에서 사후 보강

---

## 트러블슈팅

| 증상 | 원인 / 해결 |
|---|---|
| `~/.config/claude-tempo/config.json not found` | 첫 실행이라 자동 생성됨 — 안내 메시지대로 편집 후 다시 실행 |
| `git author 가 commit 못 찾음` | `git_author` 가 `null` 인지, 또는 본인 이메일이 정확한지 확인. 각 레포의 `git config user.email` 출력 비교 |
| `티켓 키가 추출 안 됨` | commit message 끝에 `(JUNGLETFT-XXX)` suffix 있는지 확인. 없으면 브랜치명 fallback. 둘 다 없으면 bucket 으로 |
| `addWorklogToJiraIssue 권한 없음` | Atlassian MCP 연결 재확인 + JIRA 권한(worklog 작성 가능 role) 확인 |
| `Tempo 경관 view 에 안 잡힘` | 그 티켓의 parent epic 이 `trackable_epics` 밖. 티켓을 적절한 에픽 하위로 옮기거나 다른 티켓에 입력 |

---

## 라이선스

MIT
