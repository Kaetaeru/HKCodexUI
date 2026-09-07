# HKCodexUI Canonical Product Specification

> Status: **Normative**
>
> 이 문서는 HKCodexUI의 제품 동작에 대한 최상위 구현 계약이다.
> `VISION.md`는 방향을 설명하고, `PLAN.md`는 초기 V1 설계를 설명하지만, 실제 구현에서 제품 동작이 충돌하면 이 문서를 우선한다.

---

# 0. 문서 목적

이 문서의 목표는 Codex가 저장소를 읽고 다음과 같은 질문을 다시 제품 결정으로 되묻지 않아도 되게 하는 것이다.

```text
어떤 화면을 만들어야 하는가?
Task가 정확히 언제 완료인가?
Pause와 Cancel은 어떻게 다른가?
CI가 없으면 어떻게 되는가?
CI가 PR에서만 돌면 어떻게 되는가?
앱이 죽었다 켜지면 어디서 이어가는가?
Codex turn 중 사용자가 새 지시를 보내면 어떻게 되는가?
무엇을 자동 승인하고 무엇을 사용자에게 묻는가?
어떤 기능이 V1이고 이후 기능은 어떻게 확장하는가?
```

코드 레벨의 작은 선택은 구현자가 할 수 있다. 제품 상태, 사용자 의미, 안전 경계, 완료 조건은 이 문서가 결정한다.

---

# 1. 제품 정의

HKCodexUI는 **local-first autonomous development workbench**다.

Codex는 코드 분석/수정/판단을 수행하는 Worker다. HKCodexUI Supervisor는 Task 수명주기와 외부 세계를 관리한다.

```text
User Goal
   ↓
Task Contract
   ↓
Workspace
   ↓
Codex work
   ↓
Independent verification
   ↓
Review
   ↓
Git / Remote
   ↓
CI / PR / external waits
   ↓
Repair when evidence changes
   ↓
Completion evidence
```

제품의 중심 객체는 Conversation이 아니라 **Task**다.

---

# 2. 대상 사용자

H0~H2의 주 대상은 다음과 같다.

```text
single user
single Windows PC
one or more local Git repositories
GitHub repositories optional but first-class
Codex CLI already usable or installable
```

팀 공유 상태/분산 worker는 H3 전에는 제품 전제가 아니다.

---

# 3. 절대 Invariants

아래는 모든 Horizon에서 유지한다.

## I-01 — Agent self-report is not completion

`Codex: Done`은 상태 정보일 뿐 완료 증거가 아니다.

## I-02 — Completion belongs to an exact candidate

검증/CI/artifact 결과는 가능하면 항상 exact commit SHA 또는 exact workspace snapshot에 귀속한다.

## I-03 — Waiting is deterministic

CI, PR review, quota, timer, webhook 등 외부 대기 중에는 새 Codex turn을 생성하지 않는다.

## I-04 — One writer per workspace

한 worktree에는 동시에 하나의 writer Task/Agent만 존재한다.

## I-05 — External side effects are reconciled

commit, push, PR 생성, release 등은 재시도 전에 실제 외부 상태를 확인한다.

## I-06 — Task survives process death

앱 crash/종료/PC 재부팅 이후 Task를 실제 Git/remote 상태와 reconciliation해 복구할 수 있어야 한다.

## I-07 — Renderer is not trusted execution code

Renderer가 raw shell, credential, unrestricted filesystem write에 직접 접근하지 않는다.

## I-08 — Untrusted repositories do not execute code

새 repository를 열었다는 이유만으로 package script, build, Codex write session을 실행하지 않는다. 사용자가 repository trust를 승인해야 한다.

## I-09 — Secrets are not casual context

알려진 secret 파일/패턴의 값은 UI event나 Codex context로 자동 전송하지 않는다.

## I-10 — Human attention is a first-class state

내부 status와 별개로 사용자가 지금 해야 할 일이 있는지를 명확히 계산한다.

---

# 4. Horizon 정의

## H0 — Reliable Autonomous Core

첫 출시 가능한 제품.

```text
Project onboarding
Repository trust
Codex App Server chat
Durable Task
Task worktree
Local verification
Repair loop
Review
Commit / push
GitHub Actions observation
Exact-SHA completion
Crash recovery
Skills
Approval policy
Task-centered UI
Windows package
```

## H1 — Best Daily Driver

```text
Global Task Center
Needs You attention queue
Project Brain
Context inspection
Verification profiles
Environment Doctor
Checkpoint / rewind
Semantic diff
Command palette
Tray / notifications
Smart cleanup
Universal Task Inbox
Acceptance Compiler v1
```

## H2 — Long-running Workbench

```text
Task queue
Overnight Mode
Quota-aware scheduler
Watch Tasks
PR Babysitter
Event triggers
Task Graph
Runbooks
Remote Control over user-owned network
```

## H3 — Agent OS

```text
Agent specialists
Cross-repository Task Graph
Release Workbench
Artifact Center
Task export/replay
Provider extension points
Shared/team handoff
Performance/cost analytics
```

Horizon은 기능의 의존성과 안정성 순서다. 최종 Vision을 축소하는 의미가 아니다.

---

# 5. 애플리케이션 Shell

## 5.1 기본 레이아웃

Desktop 폭이 충분할 때 기본은 3-column이다.

```text
┌──────────────── HKCodexUI ────────────────────────────────────────┐
│ Needs You 2   Working 3   Waiting 2        Search / Ctrl+K        │
├────────────┬──────────────────────────┬─────────────────────────────┤
│ PROJECTS   │ TASKS                    │ TASK DETAIL                 │
│            │                          │                             │
│ SimpleVTT  │ ! Auth bug              │ Auth bug                    │
│ HKCodexUI  │ ● Release V1            │ Build · local_verify        │
│ StayOps    │ ◐ Dependency upgrade    │                             │
│            │ ✓ Toolbar fix            │ Next: run full test suite   │
│            │                          │                             │
├────────────┴──────────────────────────┴─────────────────────────────┤
│ Overview | Chat | Timeline | Diff | Tests | CI | Gates | Skills   │
└────────────────────────────────────────────────────────────────────┘
```

H0에서는 Projects sidebar + selected project Task list + Task detail을 구현한다. H1에서 global cross-project Task Center를 완성한다.

## 5.2 Dark / Light

- 첫 실행은 OS theme를 따른다.
- Dark, Light, System을 Settings에서 선택할 수 있다.
- 테이블/카드/선택 항목 경계는 배경색 차이에만 의존하지 않고 1px border 또는 명확한 separator를 사용한다.
- 상태는 색만으로 표현하지 않는다. 아이콘 + 텍스트 badge를 같이 사용한다.

## 5.3 Keyboard

H0:

```text
Ctrl+N          New Task
Ctrl+P          Quick project/task switcher
Ctrl+,          Settings
Ctrl+Enter      Send current message
Esc             Close modal / dismiss non-blocking panel
```

H1에서 `Ctrl+K` command palette를 추가한다.

---

# 6. First Run / Project Onboarding

## 6.1 Welcome

repository가 하나도 없으면 중앙 primary action은 하나다.

```text
HKCodexUI
Make Codex finish the whole job.

[ Open Repository ]
```

보조 action:

```text
Open recent
Settings
```

## 6.2 Repository 선택 후 Read-only Scan

아직 repository code를 실행하지 않는다.

scan 항목:

```text
canonical path
Git repository 여부
current HEAD
current branch
default remote
origin URL
GitHub owner/repo 추출
package/runtime markers
AGENTS.md / repository instructions
CI workflow 존재 여부
Codex binary/version
Git binary/version
gh binary/auth availability
```

결과 상태:

```text
ready
warning
blocked
```

## 6.3 Repository Trust

처음 여는 repository에는 한 번만 trust를 묻는다.

```text
Trust this repository?

HKCodexUI and Codex may run repository-defined scripts and modify isolated
worktrees created from this repository.

Path: C:\work\SimpleVTT
Remote: github.com/Kaetaeru/SimpleVTT

[ Trust and continue ] [ Open read-only ]
```

`Open read-only`에서는 Chat의 read-only 분석까지만 허용하고 repository script와 write Task를 비활성화한다.

trust는 canonical path와 repository identity를 로컬 DB에 저장한다.

## 6.4 Project Ready 화면

```text
Project ready

Repository      SimpleVTT
Branch          main
GitHub          Connected
Codex           0.153.4 · Compatible
Environment     Ready
Verification    4 checks discovered
CI              GitHub Actions detected

[ Start Task ]
```

문제가 있으면 같은 화면에서 `Environment issues`를 펼쳐 보여준다.

---

# 7. Task 생성 UX

## 7.1 기본 Composer

```text
What should be done?
┌────────────────────────────────────────────┐
│ Fix the auth refresh race and ship it.     │
└────────────────────────────────────────────┘

Mode: Auto      Verification: Standard

[ Start ]
```

H0에서는 user-facing mode를 다음으로 제공한다.

```text
Chat
Autopilot
Ship
```

H1부터 `Auto`를 기본으로 추가하고 Autopilot의 표시 이름을 `Build`로 바꿀 수 있다. DB enum `autopilot`은 호환성을 위해 유지한다.

## 7.2 Advanced options

기본은 접혀 있다.

```text
Base branch
Task branch name override
Verification profile
Remote delivery policy
Review on/off
Permission profile
```

## 7.3 Task title

사용자가 별도 title을 입력하지 않으면 objective 첫 문장을 최대 72자로 축약해 생성한다. 사용자가 언제든 수정할 수 있다.

## 7.4 Attachments / URL

H0: plain text only.

H1 Universal Inbox에서 다음을 composer에 drop/paste할 수 있다.

```text
GitHub issue URL
PR URL
CI URL
log file
text file
image
```

입력은 source card로 Task에 보존한다.

---

# 8. Task Contract

Task는 최소 다음 의미를 가진다.

```text
Objective
Constraints
Acceptance Gates
Mode
Base candidate
Verification profile
Permission profile
```

H0에서는 Objective + mechanical verification + final review를 중심으로 구성한다.

H1 Acceptance Compiler는 자연어를 구조화한다.

## 8.1 Contract 변경

사용자가 `Add requirement`로 요구사항을 추가하면:

1. Task revision 증가
2. 관련 gate를 `stale` 처리
3. 이미 PASS한 verification이 새 요구사항에 영향을 받을 수 있으면 다시 검증
4. 현재 Codex turn이 active면 요구사항 자체는 즉시 DB에 저장하지만 다음 safe boundary에서 agent context에 반영

Task가 completed 후 requirement를 추가하면 기존 Task를 변경하지 않고 `Continue as new Task`를 제안한다.

---

# 9. Task 상태 모델

## 9.1 status

```text
queued
running
waiting
paused
blocked
completed
failed
cancelled
```

## 9.2 phase

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
watch
```

## 9.3 waitReason

```text
null
ci
quota
approval
user
external
timer
pr_review
```

## 9.4 attentionState

```text
none
working
waiting_external
needs_decision
needs_permission
blocked
ready_for_review
complete
```

`attentionState`는 사용자 UX용 파생/저장 상태다. `status`를 대체하지 않는다.

---

# 10. H0 Task Lifecycle

Ship의 정상 경로:

```text
prepare
  ↓
analyze / implement
  ↓
local_verify
  ├─ fail → implement(repair) → local_verify
  └─ pass
       ↓
review
  ├─ blocking finding → implement(repair) → local_verify
  └─ pass
       ↓
commit
       ↓
push
       ↓
ci
  ├─ code failure → implement(repair) → local_verify
  ├─ external wait → sleep
  └─ pass / n.a.
       ↓
acceptance
       ↓
finalize
       ↓
completed
```

Autopilot은 `local_verify` PASS + optional review PASS에서 완료한다. push/CI는 수행하지 않는다.

Chat은 autonomous lifecycle을 수행하지 않는다.

---

# 11. Pause / Stop / Cancel / Retry

## Pause

- `pauseRequested=true`를 저장한다.
- 새 Codex turn, 새 verification command, 새 external side effect를 시작하지 않는다.
- 이미 실행 중인 Codex turn/verification은 기본적으로 끝까지 둔다.
- 현재 step이 끝나면 `status=paused`.

## Stop current step

별도 action이다.

- active Codex turn → `turn/interrupt`
- active verification process → process tree 종료
- CI wait → watcher 해제
- Task 자체는 cancelled가 아니다.
- 이후 paused 상태가 된다.

## Resume

reconciliation 후 안전한 next action을 계산하고 재개한다.

## Cancel

- Task `status=cancelled`
- active Codex/process를 best-effort 중단
- 자동 lifecycle 종료
- worktree/branch는 삭제하지 않음
- Cleanup은 별도 action

## Retry

현재 failure evidence를 그대로 다시 실행할 가치가 있을 때만 보여준다. Code failure에서 의미 없는 blind retry를 기본 action으로 하지 않는다.

---

# 12. User Steering

Task 실행 중 메시지는 네 가지 의미를 갖는다.

```text
Ask
Steer now
Next instruction
Add requirement
```

H0 UI에서는 `Ask`와 `Instruction` 두 개로 시작해도 되지만 내부 모델은 네 의미를 지원할 수 있게 설계한다.

## Active Codex turn 중

- `Steer now` → active turn에 `turn/steer(expectedTurnId)`.
- `Next instruction` → Task mailbox에 queue하고 turn 완료 뒤 새 turn.
- `Add requirement` → Contract revision 저장 + safe boundary에서 새 turn.
- `Ask` → active autonomous writer turn을 오염시키지 않도록 별도 read-only/forked conversation을 사용하거나 active turn 종료 후 수행. H0에서는 active turn 중 Ask를 queue한다.

Background event가 active human-owned turn에 섞이지 않도록 자동 repair는 항상 idle boundary에서 시작한다.

---

# 13. Task Detail UI

## Header

항상 표시:

```text
title
status + attention badge
mode
phase
branch
short SHA
Codex: active / sleeping / unavailable
Pause/Resume
More
```

## Overview

카드 순서:

1. `Next action`
2. `Needs you` — 없으면 숨김
3. `Agent`
4. `Verification`
5. `Remote / CI`
6. `Acceptance`
7. `Last failure` — 없으면 숨김

## Chat

- user/agent message
- tool/command event는 접힌 activity card
- raw internal reasoning을 제품 필수 정보처럼 노출하지 않음
- command/file changes는 Timeline/Diff와 연결

## Timeline

의미 있는 사건만 기본 노출.

```text
Task started
Codex turn started/completed
files changed summary
verification pass/fail
review findings
commit
push
CI state change
user intervention
recovery
completion
```

Debug toggle에서 low-level event를 볼 수 있다.

## Diff

H0:

```text
file list
unified diff
staged/untracked markers
```

H1:

```text
Summary
Semantic changes
Raw diff
```

## Tests

verification run별:

```text
profile
command
status
duration
exit code
failure summary
log
```

## CI

```text
candidate SHA
workflow/check list
queued/running/pass/fail/skipped
last refresh
```

CI 대기 중에는 `Codex sleeping while CI runs`를 명확히 표시한다.

## Gates

각 gate:

```text
name
required?
status
candidate SHA/snapshot
evidence source
last evaluated
```

## Skills

H0:

```text
Available
Enabled for current Task
Disabled
Refresh
```

H1 이후 scope/global/project 관리.

---

# 14. Failure UX

모든 failure는 다음 구조를 가진다.

```text
category
summary
evidence
recoverable
next actions
first seen
last seen
fingerprint
```

category:

```text
agent_failure
verification_failure
review_failure
ci_failure
git_failure
permission_failure
quota_wait
environment_failure
external_failure
stagnation
recovery_conflict
compatibility_failure
```

오류 dialog 하나로 끝내지 않는다. Task Overview에 recovery action을 남긴다.

---

# 15. Stagnation

자동 repair가 다음 중 하나면 progress 없음으로 판단할 수 있다.

```text
same failure fingerprint 3회 연속
repair 후 tracked diff 변화 없음 2회 연속
failed gate count가 3 repair 동안 감소하지 않음
Codex가 동일 blocker를 반복 보고
```

기본 정책:

```text
soft warning after repair 2
block after repair 3 without progress
```

Project/Task에서 limit을 늘릴 수 있다.

block 시 자동 진행을 멈추고 `Needs You`로 올린다.

---

# 16. Repository / Worktree UX

사용자의 현재 checkout을 autonomous writer가 직접 수정하지 않는다.

Ship과 Autopilot writer Task는 기본적으로 별도 worktree를 사용한다.

Task 화면에서 항상 다음을 볼 수 있다.

```text
Base branch
Base SHA
Task branch
Worktree path
Current HEAD
```

branch 이름 기본:

```text
hk/<task-id-8>-<slug>
```

예:

```text
hk/3f19ab42-auth-refresh-race
```

사용자가 branch 이름을 override할 수 있다.

---

# 17. Local Verification UX

Profile:

```text
Quick
Standard
Release
Custom
```

H0에서는 Standard가 기본이다.

Project가 명시한 command가 있으면 그것을 우선한다.

검증을 시작하기 전에 UI에 실행될 command 목록을 확인할 수 있지만, trusted project + 허용된 profile에서는 매번 승인받지 않는다.

한 command 실패 시 profile 정책에 따라 나머지를 계속 실행해 failure를 모을 수 있다. 기본은 `continue collecting independent checks`이며 build가 prerequisite인 check는 skip 처리한다.

---

# 18. Review UX

Ship 기본은 final semantic review ON.

Review는 다음을 확인한다.

```text
objective 충족
constraints 위반 여부
불필요한 scope 확대
명백한 regression
verification 약화 여부
```

finding:

```text
blocking
non_blocking
```

blocking이 하나라도 있으면 candidate는 push 대상이 아니다.

사용자는 non-blocking finding을 completion을 막지 않는 note로 볼 수 있다.

---

# 19. Git / Commit UX

Commit은 Supervisor가 수행한다.

기본 commit subject는 Task title이다.

Project setting으로 template을 지정할 수 있다.

예:

```text
{title}
HKCodexUI: {title}
fix: {title}
```

commit 전 필수:

```text
required local verification PASS
review blocking finding = 0 (review enabled 시)
intended changes exist
no unresolved conflict
```

commit 후 exact SHA를 Task candidate로 저장한다.

---

# 20. Push / GitHub CI semantics

## 20.1 Push

push 직전 remote ref를 조회한다.

이미 expected SHA라면 중복 push하지 않는다.

remote ref가 예상과 다르게 외부에서 움직였으면 자동 force push하지 않고 `recovery_conflict`.

## 20.2 CI가 없는 repository

`.github/workflows`가 없고 commit checks도 없으며 Project setting에서 required CI를 지정하지 않았다면 remote CI gate는 `not_applicable`이다.

Ship은 local/review/push/acceptance가 모두 통과하면 완료할 수 있다.

## 20.3 CI가 있는 repository

push 후 exact SHA의 checks/runs를 발견한다.

required check source 우선순위:

```text
1. Project explicit requiredChecks
2. readable GitHub required-check configuration
3. exact SHA에서 발견된 non-neutral checks 전체
```

## 20.4 PR-only CI

workflow 파일이 존재하지만 push SHA에 CI가 생성되지 않고 PR event가 필요한 것으로 판단되면 H0는 Task를 잘못 완료하지 않는다.

상태:

```text
waiting/user
Needs You: CI requires a pull request
```

H1에서 `[Create draft PR]`을 제공하고, Project policy가 허용하면 자동 draft PR을 만들 수 있다.

## 20.5 CI failure

먼저 failure를 분류한다.

```text
code likely
flake suspected
infrastructure
permission
configuration
unknown
```

동일 SHA에서 flake/infrastructure 가능성이 높고 provider가 rerun을 지원하면 제한된 1회 deterministic rerun을 먼저 허용한다.

코드 failure이면 실패 증거만 Codex repair context에 넣는다.

---

# 21. Completion UI

완료는 banner 하나가 아니라 evidence summary다.

```text
COMPLETE

Objective            Auth refresh race fixed
Candidate SHA        a91d381
Remote SHA           a91d381
Local verification   PASS · 4/4
Review               PASS · 0 blocking
GitHub Actions       PASS · 6/6
Acceptance gates     PASS · 8/8

Duration             28m
Codex active          7m
External waiting      16m
Repairs               2

[ Open diff ] [ Open commit ] [ Cleanup ]
```

Autopilot처럼 remote commit이 없는 mode는 workspace snapshot/diff 기준 evidence를 보여준다.

---

# 22. Close / Background Behavior

H0:

- active/waiting Task가 있으면 window close는 tray/background로 숨긴다.
- 처음 한 번 `HKCodexUI is still working in the background` notification.
- active Task가 없으면 일반 close는 앱 종료.
- `Exit` 명령은 active Task가 있으면 확인 후 app-server/process를 정리하고 Task를 persisted 상태로 남긴다.
- 다음 실행에서 reconciliation 후 재개 여부를 보여주고 자동-resume policy가 켜져 있으면 재개한다.

H1에서 `Always keep running`, `Quit on close` setting을 추가한다.

---

# 23. Approval UX

Capability 단위다.

```text
read_task_workspace
write_task_workspace
run_repository_commands
network_access
install_dependencies
commit_task_branch
push_task_branch
create_pr
merge_pr
force_push
create_release
publish_package
deploy
write_outside_workspace
```

결정 scope:

```text
Allow once
Allow for Task
Allow for Project
Always ask
Deny
```

H0 기본 policy:

```text
read/write trusted task worktree     Project allow
run discovered verification          Project allow
network used by repository command   Ask when Codex/native policy asks
install dependency                    Ask
commit task branch                    Supervisor allow
push hk/* task branch                 Task/Project allow after Ship start
push protected/base branch            Ask/Deny by default
force push                            Deny by default
merge/release/publish/deploy          Ask every time or unavailable in H0
outside workspace destructive write   Deny
```

UI는 raw command보다 의미를 우선한다.

---

# 24. Compatibility UX

Project Ready 전에 Codex Compatibility Doctor 결과를 보여준다.

상태:

```text
Compatible
Compatible with limitations
Unsupported
Unavailable
```

H0 최초 검증 기준은 Codex CLI/App Server `0.153.4`.

정확한 버전 문자열만으로 모든 기능을 판단하지 않는다. core RPC capability smoke test 결과를 함께 사용한다.

Experimental API는 H0 기본 off.

---

# 25. Skills UX

H0:

- App Server `skills/list`를 기준으로 표시
- refresh 가능
- Task에 명시적으로 invoke할 Skill을 선택 가능
- Skill 파일 변경 notification을 받으면 list invalidate

Lifecycle 자체(CI polling, retry timer, recovery)는 Skill로 만들지 않는다.

H1:

```text
scope: global/project/task
skill config enable/disable
why loaded
context footprint
```

H3 Skill Studio는 VISION대로 확장한다.

---

# 26. H1 Daily Driver 상세 계약

## Global Task Center

앱을 열면 기본 필터 순서:

```text
Needs You
Active
Waiting
Recently Completed
```

## Project Brain

AI 추측 memory가 아니라 repository metadata cache다.

최소 데이터:

```text
repo map
runtime/package manager
commands
CI topology
instructions
high-risk paths
known local overrides
```

HEAD/manifests/workflows/instructions 변경 시 관련 cache를 stale 처리한다.

## Environment Doctor

문제를 `Fix automatically / Show steps / Ignore`로 표현한다. 자동 설치/변경이 system-wide이면 approval 필요.

## Checkpoints

의미 있는 safe boundary에서 생성:

```text
before first implementation
local verification pass
before large repair
CI candidate
```

restore는 파일 상태를 되돌리되 기존 evidence를 stale 처리한다.

## Semantic Diff

Summary가 raw diff를 대체하지 않는다. 항상 raw diff 접근 가능.

## Notifications

기본 알림:

```text
Needs decision
Needs permission
Blocked
Complete
```

동일 fingerprint 알림은 dedupe한다.

## Universal Inbox

source를 Task에서 제거하지 않고 provenance로 보존한다.

## Acceptance Compiler

생성된 gate는 `derived` 표시. 사용자가 고정한 gate는 `user_locked`이며 자동 삭제하지 않는다.

---

# 27. H2 Long-running 상세 계약

## Queue / Overnight

Overnight 기본 policy:

```text
code/edit/test       allow
commit task branch   allow
push task branch     allow
create draft PR      allow if Project enabled
merge                deny
release              deny
deploy               deny
```

한 Task가 blocked돼도 dependency 없는 다음 Task를 진행한다.

아침 digest는 완료/blocked/needs-you와 실제 active/wait time을 보여준다.

## Watch Task

외부 event만 관찰하는 Task. 이벤트 없을 때 Codex call 0.

## PR Babysitter

PR review comment를 원 requirement와 구분해 provenance와 함께 Task Contract에 반영한다. requested changes만 자동 repair 대상이며 단순 discussion comment는 자동 코드변경하지 않는다.

## Task Graph

DAG만 허용한다. cycle은 생성 시 reject.

writer child Task는 각자 별도 worktree/branch.

부모는 child evidence가 PASS한 뒤 integration 단계로 간다.

## Runbook

Runbook은 자연어 step과 deterministic step을 함께 가질 수 있다.

```text
inputs
steps
gates
permissions
rollback notes
```

## Remote Control

첫 구현은 cloud relay가 아니라 **user-owned network에서 접근하는 local companion web surface**다.

- default bind: localhost only
- explicit opt-in으로 LAN/Tailscale interface bind
- one-time pairing token
- paired device token revoke 가능
- remote에서 raw shell 제공하지 않음
- Task status/approval/pause/resume/instruction만 제공

---

# 28. H3 Agent OS 상세 계약

## Agent Fleet

Agent 수 자체를 UI 중심에 놓지 않는다.

Writer는 worktree당 한 명. Specialist reader/reviewer는 stable checkpoint 또는 별도 snapshot/worktree에서 동작한다.

기본 역할 후보:

```text
implementer
repository researcher
test investigator
security reviewer
UX reviewer
release auditor
```

## Cross-repo Task

부모 Task 아래 repository별 child Task. 각 repo는 자체 gate/branch/worktree를 가진다.

## Release Workbench

외부 side effect 순서와 approval을 명시한다.

```text
version
changelog
full verification
artifact
hash/sign
Git tag
GitHub release
deploy
post-release smoke
```

## Artifact Center

모든 artifact는 source SHA/snapshot, producer step, digest, timestamp를 가진다.

## Task Replay / Export

민감값 redaction 후 다음을 export 가능:

```text
contract
decisions
events
final diff
SHAs
verification
CI evidence
artifacts
```

---

# 29. 접근성 / 품질

- 모든 primary action keyboard accessible.
- focus indicator 제거 금지.
- icon-only button은 accessible label 필수.
- status는 색만으로 구분하지 않음.
- destructive action은 의미가 드러나는 label 사용.
- 긴 로그는 UI를 block하지 않고 virtualized/streamed rendering.
- empty/loading/error/recovery state를 모든 주요 view에 구현.

---

# 30. 주요 Empty / Loading / Error 상태

## No project

`Open Repository`.

## Project scanning

scan step과 현재 항목을 보여주고 spinner만 무기한 표시하지 않는다.

## Codex unavailable

Task history는 계속 열 수 있다. 새 agent turn만 disabled. `[Diagnose]` 제공.

## No Tasks

`Start your first Task`.

## Waiting CI

명시적으로 agent sleeping 상태와 next poll/event를 표시.

## No CI configured

`No remote CI configured · Not required for this project`.

## Recovery conflict

자동으로 덮어쓰지 않고 local/DB/remote SHA를 비교해서 action 제시.

## Offline

local Chat/Build가 가능한 경우 계속 가능. Ship remote phase는 waiting/external.

---

# 31. Product-level Non-goals

아래는 명시적 요구가 생기기 전까지 하지 않는다.

```text
source code를 HKCodexUI 자체 cloud에 자동 업로드
자체 Git hosting
자체 CI runner farm
사용자 승인 없는 production deployment
사용자 승인 없는 base branch force push
모든 기능을 plugin framework로 추상화
agent chatter를 메인 UX로 노출
```

---

# 32. Definition of Product Done

기능은 다음을 모두 만족해야 `implemented`로 본다.

```text
specified user path exists
normal state works
empty/loading/error state exists where applicable
restart/recovery semantics are defined and tested when stateful
permission semantics are enforced
observability/timeline event exists for meaningful autonomous action
unit/integration test covers core behavior
no acceptance-critical TODO/mock remains
```

Horizon release gate는 `ROADMAP.md`가 정의한다.
