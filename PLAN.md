# HKCodexUI Plan

이 문서는 HKCodexUI V1의 구현 기준이다.

README가 "무엇을 만드는가"를 설명한다면, 이 문서는 **어떻게 동작해야 하는가**를 정의한다.

---

# 1. 제품 목표

HKCodexUI는 Codex를 직접 대체하지 않는다.

Codex는 **코딩 Worker**이고, HKCodexUI는 **Supervisor**다.

목표는 사용자가 코딩 작업을 한 번 맡긴 뒤 다음 과정을 사람이 계속 지켜보지 않아도 되게 만드는 것이다.

```text
Goal
 ↓
Implement
 ↓
Verify
 ↓
Commit
 ↓
Push
 ↓
Wait for CI
 ↓
Repair failures
 ↓
Verify acceptance
 ↓
Complete
```

핵심 성공 조건:

> Codex의 자연어 응답이 아니라, 외부에서 확인 가능한 증거를 기준으로 Task 완료 여부를 판정한다.

---

# 2. Source of Truth

## Task 상태의 유일한 권위자는 HKCodexUI Supervisor다

다음은 보조 정보일 뿐 canonical Task state가 아니다.

- Codex thread state
- Codex의 마지막 메시지
- Codex Goal/Queue
- GitHub Actions UI의 branch 상태
- 이전 실행의 메모

Supervisor는 로컬 DB와 실제 repository/GitHub 상태를 reconciliation하여 현재 상태를 결정한다.

이 원칙은 crash recovery와 autonomous retry의 기반이다.

---

# 3. 주요 구성요소

```text
HKCodexUI
│
├─ Desktop UI
│   ├─ Projects
│   ├─ Tasks
│   ├─ Chat
│   ├─ Timeline
│   ├─ Diff
│   ├─ CI
│   ├─ Gates
│   ├─ Skills
│   └─ Settings
│
├─ Supervisor
│   ├─ Task Engine
│   ├─ Scheduler
│   ├─ Codex Adapter
│   ├─ Verification Engine
│   ├─ Git Manager
│   ├─ GitHub/CI Watcher
│   ├─ Approval Policy
│   ├─ Recovery Engine
│   └─ Event Log
│
├─ Persistence
│   └─ SQLite
│
└─ Codex Runtime
    └─ codex app-server --stdio
```

---

# 4. Task 모델

Task는 하나의 자연어 프롬프트가 아니라 **지속되는 작업 객체**다.

최소 필드:

```ts
Task {
  id
  projectId
  title
  objective
  mode

  status
  phase
  waitReason

  baseBranch
  taskBranch
  worktreePath

  codexThreadId

  localHeadSha
  pushedHeadSha
  verifiedSha

  attemptCount
  repairCount

  createdAt
  updatedAt
  completedAt
}
```

## status

```text
queued
running
waiting
blocked
completed
failed
cancelled
```

## phase

```text
prepare
analyze
implement
local_verify
review
commit
push
ci
acceptance
finalize
```

## waitReason

```text
null
ci
quota
approval
user
external
timer
```

상태와 phase를 분리한다.

예:

```text
status = waiting
phase = ci
waitReason = ci
```

이렇게 해야 UI와 recovery가 현재 의미를 정확하게 해석할 수 있다.

---

# 5. 핵심 Invariants

아래 규칙은 V1에서 깨지면 안 된다.

## Invariant A — Codex self-report는 완료 증거가 아니다

```text
Codex: "Done"
```

만으로 Task는 `completed`가 될 수 없다.

## Invariant B — CI 결과는 정확한 commit SHA에 귀속된다

`branch CI passed`가 아니라 다음을 증명해야 한다.

```text
required CI passed for expected SHA
```

## Invariant C — Waiting 중에는 불필요한 Codex turn을 만들지 않는다

CI, quota, timer를 기다리는 것은 Supervisor 책임이다.

## Invariant D — 한 worktree는 동시에 한 active Task만 소유한다

동일 작업공간에 두 Codex Task를 동시에 쓰지 않는다.

## Invariant E — retry는 idempotent해야 한다

앱이 중간에 죽었다가 같은 phase를 재개해도 다음을 중복 생성해서는 안 된다.

- 동일 commit
- 동일 push 의도
- 동일 worktree
- 불필요한 Codex repair turn

## Invariant F — 실제 상태가 DB보다 우선한다

예:

DB:
```text
phase = push
```

실제 remote:
```text
expected SHA already pushed
```

이면 다시 push하지 않고 다음 phase로 진행한다.

---

# 6. 실행 모드

## Chat

목적:

- 일반적인 Codex 사용
- 탐색
- 질문
- 작은 수정

특징:

- autonomous lifecycle 없음
- 사용자가 turn을 직접 제어

## Autopilot

목적:

- Codex가 구현하고 로컬 검증 실패를 스스로 수정

흐름:

```text
Codex implementation
      ↓
Local verification
   ├─ PASS → done
   └─ FAIL
        ↓
 Failure evidence
        ↓
 Codex repair
        ↓
 verification
```

## Ship

목적:

- repository 작업을 실제 remote/CI 완료까지 수행

```text
Prepare worktree
 ↓
Codex
 ↓
Local verification
 ↓
Review
 ↓
Commit
 ↓
Push
 ↓
CI wait
 ├─ failure → repair
 └─ pass
 ↓
Acceptance
 ↓
Complete
```

V1의 대표 기능은 Ship mode다.

---

# 7. Workspace 전략

Ship Task는 기본적으로 Git worktree를 사용한다.

예:

```text
repository/

.hkcodexui/worktrees/
  task-<id>/
```

Task 생성 시:

```text
base branch 확인
 ↓
task branch 생성
 ↓
worktree 생성
 ↓
Codex cwd = worktree
```

장점:

- 현재 사용자 workspace 보호
- 병렬 Task 가능
- rollback/cleanup 단순화
- Codex의 변경 범위 명확화

V1에서는 **같은 task branch의 동시 active Task를 허용하지 않는다.**

---

# 8. Codex Adapter

Codex는 `codex app-server --stdio`를 통해 연결한다.

Supervisor가 app-server process를 소유한다.

```text
Electron Main
    ↓ spawn
codex app-server --stdio
```

## Adapter 책임

- app-server startup/shutdown
- initialize
- thread start/resume
- turn start/interrupt
- streaming event 변환
- command/file change approval 처리
- Skills 조회/설정
- review 요청
- usage/rate-limit 정보 수집

UI가 raw JSON-RPC event를 직접 이해하지 않게 한다.

```text
Codex event
  ↓
CodexAdapter
  ↓
Workbench event
```

예:

```ts
AgentMessage
CommandStarted
CommandFinished
FileChanged
TurnCompleted
ApprovalRequested
UsageUpdated
```

---

# 9. Codex 버전 정책

App Server interface 변화에 대비한다.

V1 정책:

1. 지원 Codex 버전을 pin한다.
2. 해당 버전의 `codex app-server generate-ts` 결과를 사용한다.
3. Adapter 외부에서는 generated protocol type을 직접 참조하지 않는다.

```text
Generated Codex Types
       ↓
CodexAdapter
       ↓
HKCodexUI stable types
```

이렇게 해야 Codex 업데이트가 UI 전체를 깨뜨리지 않는다.

---

# 10. 첫 Codex 프롬프트

Supervisor는 lifecycle을 Codex에게 떠넘기지 않는다.

예:

```text
Objective:
<user objective>

Work only inside the provided task workspace.
Inspect repository instructions and relevant code before editing.
Implement the requested behavior and run relevant local verification.
Do not commit, push, merge, publish, or wait for remote CI.
Report any genuine blocker with concrete evidence.
```

Codex는 코드 작업만 한다.

Git/CI lifecycle은 Supervisor가 이어받는다.

---

# 11. Verification Engine

Codex가 테스트를 실행했다고 말해도 Supervisor가 독립적으로 검증한다.

## Verification source priority

1. repository-defined commands
2. project configuration
3. user-configured commands
4. 안전하게 추론 가능한 standard command

예:

```text
package.json scripts
pyproject.toml
Cargo.toml
*.sln
Makefile
CI workflow
```

검증 종류 예:

```text
test
lint
typecheck
build
repository-specific gates
```

## 실패 시

Supervisor가 실패 결과를 구조화한다.

```text
command
exit code
failed test names
relevant stderr/stdout
```

그 증거만 Codex repair turn에 전달한다.

전체 로그를 무조건 context에 넣지 않는다.

---

# 12. Repair Loop

```text
VERIFY FAIL
    ↓
Create repair evidence
    ↓
Resume Codex thread
    ↓
Repair
    ↓
VERIFY
```

하지만 무한 반복하면 안 된다.

## Stagnation detection

다음과 같은 패턴을 감지한다.

- 동일한 테스트가 반복 실패
- 동일한 error fingerprint 반복
- 여러 repair 후 diff에 실질적 변화 없음
- 동일 commit 상태 반복

발생 시:

```text
status = blocked
```

으로 전환하고 사용자에게 다음을 보여준다.

```text
왜 멈췄는지
마지막 실패
시도 횟수
Codex의 blocker 설명
사용자가 취할 수 있는 행동
```

V1에서는 repair 횟수를 무한으로 허용하지 않는다.

---

# 13. Review

Ship mode에서는 remote push 전에 의미적 검토 단계를 둘 수 있다.

목적:

- 테스트는 통과하지만 요구사항을 잘못 구현한 경우 탐지
- 불필요한 범위 확대 탐지
- regression 가능성 탐지

Review 결과는 두 종류로 나눈다.

```text
blocking
non-blocking
```

blocking finding이 있으면 Codex에게 repair를 요청하고 다시 verification한다.

V1에서 review는 configurable이지만 Ship mode 기본 ON을 목표로 한다.

---

# 14. Git Manager

Codex는 V1 Ship mode에서 Git lifecycle의 권위자가 아니다.

Supervisor가 관리한다.

```text
git status
git diff
git diff --check
git add
git commit
git push
```

각 단계 전후에 SHA를 기록한다.

```text
localHeadSha
pushedHeadSha
verifiedSha
```

## commit 생성 전 최소 조건

```text
local verification PASS
blocking review 없음
working tree에 의도된 변경 존재
```

## push 후

remote branch SHA가 예상 SHA와 일치하는지 확인한다.

---

# 15. CI Watcher

V1 remote CI 대상은 GitHub Actions다.

Supervisor가 정확한 commit SHA에 연결된 workflow/check를 추적한다.

```text
PUSH SHA
  ↓
Discover runs/checks
  ↓
WAIT
```

WAIT 동안:

```text
Codex 호출 없음
```

## CI failure

다음 정보만 우선 추출한다.

- workflow
- failed job
- failed step
- annotations
- exit code
- 오류 주변 로그

그 후 Codex에게 repair 요청한다.

```text
CI failed for commit <sha>.

Workflow: ...
Job: ...
Step: ...

Relevant failure evidence:
...

Fix the failure. Do not weaken required verification.
```

수정 후 전체 local verification부터 다시 시작한다.

---

# 16. Acceptance Gates

Gate는 Task 완료 조건이다.

두 종류가 있다.

## Mechanical Gate

Supervisor가 직접 판정할 수 있다.

예:

```text
command exits 0
file exists
artifact exists
expected SHA pushed
required CI success
working tree clean
```

## Semantic Gate

코드 의미를 판단해야 한다.

예:

```text
requested UX preserved
compatibility requirement met
specified behavior actually implemented
```

Semantic gate는 Codex review 또는 명시된 repository test/evidence를 이용한다.

가능한 요구사항은 mechanical gate로 만드는 것을 우선한다.

---

# 17. Completion Gate

Ship Task는 다음 조건을 모두 만족할 때만 `completed`가 된다.

V1 기본 모델:

```text
required local verification PASS
AND
verified SHA == current intended SHA
AND
remote SHA == intended SHA
AND
required CI for intended SHA PASS
AND
all required acceptance gates PASS
AND
no blocking review findings
```

완료 화면에는 "완료"만 쓰지 않는다.

증거를 보여준다.

예:

```text
COMPLETE

Local verification   PASS
GitHub Actions       PASS
Acceptance gates     14 / 14
Blocking findings    0
Final SHA            abc1234
Remote SHA           abc1234
```

---

# 18. Approval Policy

두 층으로 관리한다.

```text
Codex native approval
+
HKCodexUI policy
```

## 기본 자동 허용 범위

Task worktree 안에서:

- 읽기
- 쓰기
- 테스트
- lint/build/typecheck
- git status/diff

## Supervisor가 수행

- commit
- task branch push

## 기본 사용자 승인 필요

- force push
- merge
- tag
- release
- package publish
- deployment
- task workspace 밖의 destructive operation

V1은 자동 merge/release를 제품 목표에 넣지 않는다.

---

# 19. Crash Recovery

이 기능은 V1 필수다.

Autonomous 작업 도구는 앱을 다시 켰을 때 이어지지 않으면 가치가 크게 떨어진다.

모든 중요한 transition은 SQLite에 저장한다.

예:

```text
Task
Attempt
AgentTurn
VerificationRun
GitSnapshot
CIRun
GateResult
Approval
Event
```

앱 시작 시 active Task에 대해 reconciliation을 수행한다.

```text
DB snapshot
   +
actual worktree
   +
actual git SHA
   +
remote SHA
   +
GitHub CI state
   ↓
recover current state
```

단순히 DB의 마지막 phase부터 다시 실행하지 않는다.

---

# 20. Scheduler

Scheduler는 active Task를 실행 가능한 상태에서만 진행시킨다.

예:

```text
running + implement
    → Codex turn

waiting + ci
    → CI watcher only

waiting + quota
    → reset time까지 sleep

blocked
    → 자동 실행 없음
```

V1에서는 single-machine, single-user를 기준으로 한다.

분산 scheduler는 만들지 않는다.

---

# 21. Timeline / Observability

사용자가 autonomous loop를 신뢰하려면 현재 상황을 쉽게 이해할 수 있어야 한다.

Raw internal event를 모두 보여주는 대신 의미 있는 Timeline을 생성한다.

예:

```text
11:03  Repository analyzed
11:07  Codex changed 4 files
11:09  Local verification failed: 2 tests
11:12  Repair #1 started
11:16  Local verification passed
11:17  Commit a92df37 created
11:18  Pushed
11:18  Waiting for GitHub Actions
11:26  CI failed: windows-integration
11:27  Repair #2 started
11:35  Waiting for CI
11:43  Required checks passed
11:44  Acceptance passed
11:44  Task complete
```

사용자는 최소한 항상 다음 질문에 답을 얻을 수 있어야 한다.

```text
지금 뭘 하는가?
왜 기다리는가?
무엇이 실패했는가?
Codex가 지금 호출되고 있는가?
몇 번 고쳤는가?
완료에 무엇이 남았는가?
```

---

# 22. UI Information Architecture

## Left Sidebar

```text
Projects
└─ Tasks
```

## Main Task Header

```text
Task title
status
phase
branch
current SHA
Codex active/sleeping
```

## Main tabs

```text
Overview
Chat
Timeline
Diff
CI
Gates
Skills
```

### Overview

가장 중요한 상태만 보여준다.

```text
Current phase
Local checks
Remote CI
Acceptance
Last failure
Next action
```

### Chat

Codex와의 실제 interaction.

Task 실행 중에도 추가 instruction을 줄 수 있다.

단, 새 instruction이 현재 Task objective와 충돌하면 autonomous execution을 잠시 멈추고 반영 여부가 명확해야 한다.

### Timeline

작업 이력.

### Diff

현재 Task가 만든 변경.

### CI

workflow/job/check 상태.

### Gates

완료 조건과 증거.

### Skills

현재 Task/Codex에 적용되는 Skill 관리.

---

# 23. User Intervention

Autopilot 중에도 사용자가 개입할 수 있어야 한다.

행동:

```text
Pause
Resume
Send instruction
Retry
Cancel
```

## Pause

새 Codex turn과 새 lifecycle transition을 시작하지 않는다.

현재 외부 process를 무조건 kill한다는 뜻은 아니다.

## Cancel

Task를 `cancelled`로 바꾸고 자동 진행을 종료한다.

기존 branch/worktree 삭제 여부는 별도 cleanup action으로 둔다.

사용자가 cancel했다고 자동으로 작업물을 삭제하면 안 된다.

---

# 24. Skills

Skills는 코딩 판단과 규칙을 보강한다.

예:

```text
plan-critic
minimal-engineering
repository-specific skill
release-auditor
```

Skills에 넣지 않을 것:

```text
CI polling
retry timer
push lifecycle
crash recovery
```

그것들은 deterministic Supervisor 기능이다.

UI에서는:

```text
Enabled for task
Available
Disabled
```

정도를 먼저 제공한다.

Skill marketplace/editor는 V1 필수가 아니다.

---

# 25. Failure Categories

실패를 전부 `error` 하나로 표현하지 않는다.

최소 분류:

```text
agent_failure
verification_failure
ci_failure
git_failure
permission_failure
quota_wait
external_failure
stagnation
recovery_conflict
```

각 실패에는:

```text
human-readable summary
machine evidence
recoverable?
recommended next action
```

을 둔다.

---

# 26. Retry Policy

무조건 `while (!done)` 하지 않는다.

Retry에는 항상 다음이 있어야 한다.

```text
reason
attempt number
previous evidence fingerprint
new evidence fingerprint
progress signal
```

진척의 예:

```text
failed tests 8 → 3
new implementation diff
failure fingerprint changed
acceptance gates 6/10 → 9/10
```

진척이 없는 반복은 stagnation으로 전환한다.

---

# 27. Persistence 초안

V1 SQLite table 후보:

```text
projects
tasks
task_attempts
agent_threads
agent_turns
verification_runs
git_snapshots
ci_runs
ci_jobs
gates
gate_results
approvals
events
settings
```

처음부터 event sourcing 시스템을 만들지는 않는다.

현재 상태는 일반 table에 저장하고, `events`는 observability/audit 용도로 둔다.

---

# 28. 기술 스택

V1 권장:

```text
Electron
React
TypeScript
SQLite
Node child_process
Git CLI
GitHub API / gh integration
Codex app-server
```

Windows-first로 만든다.

Electron Main이 Supervisor와 process lifecycle을 소유한다.

Renderer는 직접 shell/Git/Codex process를 실행하지 않는다.

```text
Renderer
   ↓ typed IPC
Electron Main
   ↓
Supervisor
```

---

# 29. 보안/Trust Boundary

Renderer를 신뢰 경계 밖으로 취급한다.

민감한 실행은 Main/Supervisor에서만 한다.

다음은 Renderer에 직접 노출하지 않는다.

- raw arbitrary shell execution API
- GitHub credential
- Codex auth material
- unrestricted filesystem write

IPC는 목적별 command로 제한한다.

예:

```text
startTask
pauseTask
resumeTask
sendTaskInstruction
approveAction
```

---

# 30. V1 Milestones

## M0 — Foundation

목표:

- Electron + React 실행
- SQLite
- project 등록
- Codex process startup
- app-server initialize

완료 기준:

- 앱에서 프로젝트를 열 수 있다.
- Codex thread를 시작하고 응답을 streaming 표시할 수 있다.

## M1 — Task + Codex

목표:

- Task persistence
- thread mapping
- Chat/Timeline
- crash 후 Task 복원

완료 기준:

- 앱 재실행 후 이전 Task와 Codex conversation을 다시 열 수 있다.

## M2 — Local Autopilot

목표:

- worktree
- verification discovery/config
- verify/repair loop
- stagnation detection

완료 기준:

- 실패하는 local test를 Codex가 수정하고 Supervisor 독립 검증으로 PASS까지 갈 수 있다.

## M3 — Ship / GitHub CI

목표:

- commit
- push
- exact SHA tracking
- GitHub Actions watcher
- CI failure evidence
- CI repair loop

완료 기준:

- CI 실패 후 Codex를 자동 재개하고 새 commit을 push해서 CI PASS까지 갈 수 있다.
- CI 대기 중 Codex turn이 생성되지 않는다.

## M4 — Completion + Recovery

목표:

- gates
- final review
- robust reconciliation
- approvals
- completion evidence UI

완료 기준:

- 앱 중간 종료 후 재실행해도 active Ship Task를 실제 상태에서 안전하게 이어간다.
- 완료 화면이 exact final SHA와 검증 증거를 보여준다.

## M5 — V1 UX polish

목표:

- Overview
- CI view
- Gates view
- Skills view
- useful errors
- pause/resume/cancel

완료 기준:

- 사용자가 Terminal이나 GitHub 웹사이트를 열지 않고도 Task가 왜 실행/대기/실패/완료 상태인지 이해할 수 있다.

---

# 31. V1 Acceptance Criteria

V1 release 전 아래 시나리오를 실제로 검증한다.

### Basic

- 프로젝트를 등록할 수 있다.
- Codex와 대화할 수 있다.
- Task를 생성/재개/취소할 수 있다.

### Local Autopilot

- Codex가 파일을 수정한다.
- Supervisor가 독립적으로 verification한다.
- verification 실패가 Codex repair로 전달된다.
- repair 후 PASS가 확인된다.

### Git

- Task worktree가 격리된다.
- 검증 전 commit되지 않는다.
- commit SHA가 기록된다.
- push 후 remote SHA를 확인한다.

### CI

- exact pushed SHA의 GitHub Actions를 추적한다.
- CI 실행 중 Codex는 대기 호출을 하지 않는다.
- CI 실패 job/step/log evidence를 수집한다.
- Codex가 CI failure를 수정한다.
- 수정 commit의 CI를 새로 추적한다.

### Completion

- Codex의 `Done`만으로 completed가 되지 않는다.
- required local/CI/acceptance gate가 모두 PASS해야 한다.
- final SHA와 remote SHA가 일치한다.

### Recovery

다음 각 지점에서 앱을 강제 종료한 뒤 복구 테스트한다.

- Codex turn 후
- local verification 후
- commit 후 push 전
- push 후 CI 대기 중
- CI failure 후 repair 전

중복 commit/push/repair 없이 올바른 phase를 복원해야 한다.

### Safety

- force push는 기본 자동 수행되지 않는다.
- auto merge/release/publish는 기본 수행되지 않는다.
- task workspace 밖의 destructive operation은 자동 허용되지 않는다.

---

# 32. V1에서 의도적으로 하지 않는 것

다음은 좋은 아이디어일 수 있지만 V1 핵심 목표와 무관하므로 보류한다.

```text
Claude/Gemini 등 multi-model adapter
multi-agent swarm
cloud execution
built-in full code editor
custom terminal IDE
issue triage automation
auto merge
release automation
deployment automation
plugin marketplace
team collaboration
remote web dashboard
```

먼저 **Codex 한 명을 끝까지 일하게 만드는 것**을 완성한다.

---

# 33. V1 이후 후보

V1이 안정화된 후 검토한다.

```text
multiple parallel Tasks
PR creation/review lifecycle
optional auto merge
release gates
remote notifications
scheduled tasks
issue → autonomous task
multi-agent reviewer/implementer
other model adapters
cloud worker
```

---

# 34. 제품 한 문장

최종적으로 HKCodexUI가 제공해야 하는 경험은 이것이다.

> **코딩 목표를 입력하면, Codex는 필요한 순간에만 일하고 HKCodexUI가 나머지 실행·대기·검증·복구를 맡아 실제 증거가 모두 통과할 때까지 작업을 끝까지 관리한다.**
