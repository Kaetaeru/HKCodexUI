# HKCodexUI Product Vision

> 이 문서는 HKCodexUI의 **장기 제품 방향성**을 정의한다.
>
> [`PLAN.md`](./PLAN.md)는 V1 구현 계약이고, 이 문서는 V1 범위를 넘어서 **궁극적으로 어떤 개발환경을 만들 것인가**를 정의한다.

---

# 1. North Star

HKCodexUI의 최종 목표는 "Codex용 예쁜 UI"가 아니다.

목표는 다음에 가깝다.

> **개발자가 코딩 에이전트를 계속 지켜보고 조종하지 않아도 되는 Local-first Agent Workbench / Agent OS.**

사용자는 가능한 한 결과와 중요한 결정만 본다.

```text
사용자
  ↓
목표를 맡긴다
  ↓
HKCodexUI가 작업을 구조화한다
  ↓
필요한 Codex 작업을 호출한다
  ↓
검증 / CI / 외부 이벤트는 Supervisor가 관리한다
  ↓
실패하면 적절한 증거만 다시 Codex에 준다
  ↓
위험하거나 애매한 결정만 사용자에게 올린다
  ↓
완료 증거를 보여준다
```

궁극적인 UX는 다음 질문을 없애는 것이다.

```text
"지금 얘가 뭐 하고 있지?"
"끝난 건가?"
"CI 확인해야 하나?"
"실패했는데 다시 시켜야 하나?"
"이전 대화 어디 갔지?"
"어느 branch였지?"
"테스트 진짜 돌린 거 맞나?"
"왜 또 권한을 묻지?"
"이 작업을 집에 가서도 볼 수 없나?"
```

---

# 2. 경쟁 기준

HKCodexUI는 단순히 현재 Codex UI보다 편한 것을 목표로 하지 않는다.

최소 경쟁 기준은 현대적인 agentic coding 환경이 제공하는 다음 경험이다.

```text
background agents
parallel sessions
session resume
plan mode
skills
rules
hooks
subagents
permission profiles
checkpoint / rewind
remote control
long-running goals
CI / PR workflows
```

이 기능들은 **Parity Baseline**이다.

HKCodexUI의 차별화는 그 위에 다음을 더하는 것이다.

```text
session이 아니라 durable Task가 중심
외부 증거 기반 completion
zero-token wait / wake
crash-safe lifecycle
Task graph / dependency scheduling
acceptance compiler
repository-aware project brain
exception-only human attention
CI / PR babysitting
remote/event-driven operation
replayable audit trail
```

---

# 3. 제품 원칙

## Principle A — Chat보다 Task가 상위 개념이다

Conversation은 Task를 수행하기 위한 인터페이스 중 하나다.

```text
Project
 └─ Task
     ├─ Goal
     ├─ Constraints
     ├─ Gates
     ├─ Workspace
     ├─ Agent Thread(s)
     ├─ Verification
     ├─ Git / PR / CI
     ├─ Timeline
     ├─ Checkpoints
     └─ Completion Evidence
```

Chat history가 사라져도 Task의 의미와 상태는 남아야 한다.

---

## Principle B — 사람은 예외를 관리한다

잘 되는 작업에는 사용자의 관심을 요구하지 않는다.

사용자에게 올라오는 것은 예를 들어 다음뿐이어야 한다.

```text
요구사항이 서로 충돌함
보안/데이터 손실 위험이 큼
merge/release/deploy 필요
외부 credential이 없음
여러 제품 결정 중 하나를 골라야 함
반복 실패로 진척이 없음
```

즉 목표는 `human in every loop`가 아니라:

> **human on important exceptions**

이다.

---

## Principle C — 기다리는 동안 모델을 쓰지 않는다

다음은 AI 판단이 아니다.

```text
CI 대기
rate-limit reset 대기
PR review 대기
scheduled start 대기
artifact 생성 대기
외부 webhook 대기
```

Supervisor가 sleep하고 이벤트가 발생하면 깨운다.

---

## Principle D — 완료는 주장보다 증거다

```text
Agent says done     → 정보
Test says PASS       → 증거
CI says PASS         → 증거
Artifact digest      → 증거
Remote SHA match     → 증거
Acceptance gate PASS → 증거
```

완료 화면은 항상 증거를 보여준다.

---

## Principle E — 위험도에 따라 UX가 달라진다

작은 수정과 production release를 같은 permission flow로 다루지 않는다.

```text
Low risk
→ 거의 무중단 자동 진행

Medium risk
→ task/project policy 안에서 자동 진행

High risk
→ 실행 직전 명확한 사용자 승인
```

---

## Principle F — 사용자가 작업을 잊어도 시스템은 기억한다

앱 재시작, PC 재부팅, CI 장기 실행, quota reset을 거쳐도 Task가 복구되어야 한다.

---

# 4. 가장 이상적인 첫 사용 경험

앱을 처음 켰을 때 복잡한 설정부터 요구하지 않는다.

```text
[ Open Repository ]
```

저장소를 열면 HKCodexUI가 자동으로 탐색한다.

```text
✓ Git repository
✓ default branch
✓ package/runtime
✓ repository instructions
✓ test commands
✓ build commands
✓ CI workflows
✓ required checks
✓ GitHub remote
✓ Codex availability
✓ Skills
```

그리고 한 화면으로 보여준다.

```text
Project ready

Repository       SimpleVTT
Branch           main
Tests            npm test
Build            npm run build
CI               GitHub Actions · 4 workflows
Codex            Connected
Environment      Ready

[ Start Task ]
```

설정할 것이 없으면 설정 화면에 들어갈 필요가 없어야 한다.

---

# 5. Universal Task Inbox

Task 생성 진입점을 프롬프트 입력창 하나로 제한하지 않는다.

사용자는 다음을 그냥 HKCodexUI에 던질 수 있다.

```text
자연어 요청
GitHub issue URL
PR URL
CI run URL
error log
stack trace
파일
폴더
스크린샷
짧은 동영상/GIF
TODO text
release checklist
```

예:

```text
<GitHub issue URL 붙여넣기>
```

HKCodexUI가 자동으로:

```text
Issue 읽기
→ 관련 repository 식별
→ branch 전략 결정
→ 요구사항 추출
→ acceptance 초안 생성
→ Task 생성
```

Drag & Drop도 같은 방식으로 동작한다.

---

# 6. Goal / Acceptance Compiler

사용자는 보통 완벽한 acceptance criteria를 작성하지 않는다.

예:

```text
로그인 버그 고쳐줘
```

HKCodexUI는 이 요청을 내부적으로 구조화한다.

```text
Objective
- 로그인 세션 만료 시 발생하는 오류 해결

Constraints
- 기존 정상 로그인 동작 유지
- 인증 보안 완화 금지

Suggested gates
- relevant tests PASS
- full auth test suite PASS
- lint/typecheck PASS
- required CI PASS
```

중요한 점:

- 작은 작업에서는 이 화면을 굳이 사용자에게 막지 않는다.
- 큰/위험한 작업에서는 `Task Contract`로 보여준다.
- 사용자가 나중에 요구사항을 추가하면 Gate도 다시 계산한다.

---

# 7. Smart Task Modes

단순한 `Chat / Autopilot / Ship`에서 장기적으로 더 자연스러운 개념으로 발전시킨다.

사용자는 대부분 모드를 고르지 않아도 된다.

HKCodexUI가 작업 성격을 판단해 추천한다.

```text
Ask
- 설명/분석 중심

Edit
- 작은 로컬 수정

Build
- 구현 + 로컬 검증

Ship
- Git + CI + PR까지

Watch
- 외부 상태를 계속 관찰

Runbook
- 반복 가능한 운영 절차 수행
```

고급 사용자는 직접 고정할 수 있다.

---

# 8. Project Brain

매 Task마다 저장소 전체를 처음부터 다시 이해하게 만들지 않는다.

HKCodexUI는 프로젝트에 대한 **검증 가능한 구조적 지식**을 유지한다.

예:

```text
repository map
build/test commands
CI topology
required checks
code ownership
high-risk directories
common generated files
known flaky tests
runtime versions
package manager
branch conventions
release process
repository instructions
frequently used Skills
```

중요:

- 이 정보는 언제든 실제 repository 상태와 비교해 invalidation한다.
- AI가 임의로 추측한 사실을 canonical memory로 승격하지 않는다.
- stale 정보는 명확히 stale 처리한다.

이것은 모델의 개인 기억이 아니라 **Project Metadata Cache**다.

---

# 9. Context Engine

모델에게 무조건 많은 context를 던지는 것이 좋은 UX가 아니다.

HKCodexUI가 context budget을 관리한다.

```text
Task objective
+ repository instructions
+ relevant files
+ relevant previous decisions
+ current failure evidence
+ relevant Skill
```

만 우선 제공한다.

사용자는 필요하면 다음도 볼 수 있다.

```text
Why is this in context?
How many tokens did this phase consume?
What was omitted?
```

긴 작업에서는 Task-level summary와 evidence를 사용해 불필요한 history 재전송을 줄인다.

---

# 10. Adaptive Effort

모든 turn을 최고 effort로 실행하지 않는다.

예:

```text
repository scan       → low/medium
simple formatting     → low
bug root-cause        → medium/high
architecture decision → high
final semantic review → high
```

사용자는 다음 정책을 선택할 수 있다.

```text
Fast
Balanced
Thorough
Custom
```

단, 모델 routing은 Task correctness를 희생하지 않는다.

---

# 11. Background Task Center

여러 Task를 동시에 관리할 수 있어야 한다.

```text
TASKS

● auth refresh bug       CODING
● Windows release        CI 7m
◐ dependency upgrade     WAITING REVIEW
○ docs cleanup           QUEUED
! payment migration      NEEDS YOU
✓ toolbar fix            COMPLETE
```

각 Task row에서 즉시 보여준다.

```text
현재 상태
마지막 의미 있는 이벤트
사용자 입력 필요 여부
다음 예정 행동
최근 progress
```

사용자는 terminal tab을 여러 개 기억할 필요가 없다.

---

# 12. Queue / Overnight Mode

개발자가 여러 작업을 줄 수 있다.

```text
1. auth issue
2. dependency bump
3. docs sync
4. Windows build
```

Scheduler가 다음을 고려한다.

```text
priority
dependency
repository conflict
branch ownership
CPU-heavy local tests
Codex quota
risk policy
```

## Overnight Mode

퇴근 전에 Task 여러 개를 넣는다.

```text
[ Run overnight ]
```

정책 예:

```text
자동 수정/검증/push는 허용
merge/release/deploy는 금지
막히면 다음 independent Task로 이동
아침에 digest 제공
```

아침:

```text
Overnight Summary

3 completed
1 waiting for approval
1 blocked

✓ auth issue        PR #142 ready
✓ dependency bump   CI PASS
✓ docs sync         local complete
! Windows release   signing credential required
× flaky-test task   no progress after 3 repairs
```

---

# 13. Task Graph

큰 목표를 억지로 한 agent thread에 넣지 않는다.

필요한 경우 Task를 dependency graph로 분해한다.

```text
Release V2
│
├─ API migration ──────┐
├─ Frontend migration ─┼→ Integration tests → Release build
└─ Docs update ────────┘
```

중요:

- 작은 작업을 쓸데없이 분해하지 않는다.
- 독립적인 경우만 병렬 실행한다.
- merge 충돌 가능성까지 고려한다.
- 부모 Task는 child Task의 evidence를 종합한다.

---

# 14. Agent Fleet

장기적으로 한 Codex thread만 사용하지 않아도 된다.

단, 무조건적인 agent swarm은 목표가 아니다.

필요할 때 역할을 분리한다.

```text
Main Implementer
├─ Repository Researcher
├─ Test Investigator
├─ Security Reviewer
├─ UX Reviewer
└─ Release Auditor
```

UI에서는 agent chatter가 아니라 **결론과 책임 영역**을 보여준다.

```text
Implementation   PASS
Tests            PASS
Security Review  1 finding
Release Audit    waiting
```

사용자는 내부 agent 수를 알지 않아도 작업을 이해할 수 있어야 한다.

---

# 15. Smart Steering

작업 중 사용자가 메시지를 보내면 의미를 구분한다.

```text
Steer now
- 현재 진행 중인 방향을 즉시 수정

Next instruction
- 현재 turn 종료 후 적용

Add requirement
- Task Contract와 acceptance에 영구 반영

Ask
- 작업을 바꾸지 않고 질문만 함
```

메시지 하나가 현재 작업을 실수로 망가뜨리지 않아야 한다.

UI에서 전송 전에 간단히 의미를 보여줄 수 있다.

```text
This will modify the task objective.
```

---

# 16. Checkpoints / Time Travel

Git commit만 undo 수단으로 사용하지 않는다.

Task 진행 중 의미 있는 시점에 checkpoint를 만든다.

```text
Checkpoint 1 · before implementation
Checkpoint 2 · local tests passing
Checkpoint 3 · before API redesign
Checkpoint 4 · CI candidate
```

사용자 행동:

```text
View
Compare
Restore files
Fork from here
Create branch from here
```

## Approach Fork

예:

```text
"A안이랑 B안 둘 다 한번 해봐"
```

```text
Task
├─ Approach A → worktree A
└─ Approach B → worktree B
```

검증 결과를 비교한다.

```text
              A        B
Tests         PASS     PASS
Files changed 14       6
Perf          +12%     +9%
Complexity    higher   lower
```

그 뒤 선택된 쪽만 이어간다.

---

# 17. Diff UX

Raw diff만 보여주지 않는다.

세 레벨을 제공한다.

```text
Summary
Semantic changes
Raw diff
```

예:

```text
Changed behavior
- expired sessions now refresh once before logout

Tests
- added regression test for concurrent refresh

Files
4 changed · +118 -43
```

사용자는 diff에 직접 질문할 수 있다.

```text
"왜 이 파일까지 바꿨어?"
"이 변경 없어도 되지 않아?"
"이 부분만 원래대로 돌려"
```

---

# 18. Visual / UI Verification

웹/GUI 프로젝트에서는 테스트 PASS만으로 충분하지 않다.

향후 HKCodexUI는 preview 환경을 열고 시각 검증을 수행할 수 있어야 한다.

```text
start dev server
→ target page open
→ screenshot
→ requested UI state verify
→ console errors check
→ accessibility smoke check
```

가능한 경우 before/after screenshot을 함께 보여준다.

```text
[ Before ] [ After ]
```

사용자는 이미지에 표시해서 다시 지시할 수 있다.

---

# 19. Environment Doctor

Task가 시작도 못 하는 환경 문제를 agent에게 계속 던지지 않는다.

HKCodexUI가 먼저 진단한다.

```text
runtime missing
wrong Node/Python version
package manager missing
Git auth missing
GitHub remote inaccessible
Codex unavailable
required env var absent
Docker unavailable
SDK/workload missing
```

화면:

```text
Environment issue

Node 22 required
Node 20.11 detected

[ Fix automatically ]
[ Show steps ]
[ Ignore for this task ]
```

위험한 설치는 사용자 승인을 요구한다.

---

# 20. Verification Profiles

매번 모든 검증을 같은 방식으로 돌리지 않는다.

Project에 profile을 저장한다.

```text
Quick
- targeted tests
- typecheck

Standard
- tests
- lint
- typecheck

Release
- full tests
- build
- integration
- packaging
- security checks
```

HKCodexUI가 Task 위험도에 따라 적절한 profile을 추천한다.

---

# 21. Failure Intelligence

실패를 그냥 로그 덩어리로 보여주지 않는다.

분류한다.

```text
Code failure
Test expectation failure
Environment failure
Infrastructure failure
CI configuration failure
Permission failure
Flaky suspicion
External service failure
Agent stagnation
```

## Flake handling

동일 commit에서 외부/flake 가능성이 높다면 제한된 횟수로 deterministic rerun을 먼저 할 수 있다.

코드 실패 증거가 분명할 때는 Codex를 깨운다.

즉 모든 빨간 CI가 곧바로 새로운 AI 수정 turn이 되는 것을 막는다.

---

# 22. CI / PR Babysitter

Task는 CI PASS로 무조건 끝나지 않을 수 있다.

Watch Task는 다음 이벤트를 계속 관찰한다.

```text
new CI run
review requested
review comment
requested changes
branch behind
merge conflict
required check changed
PR merged/closed
```

예:

```text
PR #142

✓ CI passed
! Reviewer requested change in auth/session.ts

HKCodexUI
→ comment 읽음
→ requirement에 반영
→ worktree resume
→ 수정
→ verification
→ push
→ 다시 CI 대기
```

사람이 PR을 계속 새로고침하지 않아도 된다.

---

# 23. External Event Triggers

Task 시작은 사용자 버튼뿐일 필요가 없다.

장기적으로:

```text
GitHub issue assigned
PR review comment
CI failure
scheduled time
webhook
filesystem event
custom HTTP event
chat integration
```

등을 trigger로 사용할 수 있다.

예:

```text
When:
required CI fails on release/*

Do:
create diagnostic Task

Policy:
do not push automatically
```

---

# 24. Remote Control

HKCodexUI는 local-first를 유지하면서 원격 관리를 지원할 수 있어야 한다.

PC에서 Supervisor와 작업은 계속 실행된다.

외부 기기에서는:

```text
Task 상태 보기
승인
간단한 지시
pause/resume/cancel
실패 요약 보기
완료 알림 확인
```

만 안전하게 수행한다.

예:

```text
Phone

HKCodexUI
Windows release needs approval

Action:
Create GitHub release v1.4.0

[ Approve ] [ Reject ] [ Open details ]
```

원격 사용을 위해 source tree 전체를 별도 cloud로 복제하는 것이 기본 전제가 되어서는 안 된다.

---

# 25. Notifications

앱을 계속 보고 있을 필요가 없어야 한다.

알림 우선순위:

```text
Info
- Task completed

Attention
- user decision required
- permission required

Failure
- blocked
- repeated failure
- environment unavailable
```

지원 후보:

```text
Windows notification
system tray
mobile/remote notification
Slack / Discord / Telegram
email
```

같은 원인 알림을 반복 발송하지 않는다.

---

# 26. System Tray / Mini Control

창을 닫아도 background service는 유지할 수 있다.

Tray에서:

```text
3 active · 1 needs attention

Pause all
Open task center
Quiet mode
Exit after tasks finish
```

를 제공한다.

---

# 27. Permission UX

permission은 매번 `Allow?`를 묻는 팝업이 되어서는 안 된다.

Capability 기반 정책으로 관리한다.

```text
Allow once
Allow for this Task
Allow for this Project
Always ask
Always deny
```

예:

```text
npm test                Project allow
npm install             Ask
push task/*             Project allow
push main               Ask
force push              Deny
release                 Ask
production deploy       Ask every time
```

여러 pending approval이 있으면 가능한 경우 한 번에 묶어 보여준다.

---

# 28. Safety Preview

큰 side effect 직전에는 무엇이 일어날지 보여준다.

```text
About to publish

Repository       HKCodexUI
Tag              v1.2.0
Commit           a91d381
Artifacts        3
Target           GitHub Release

This action is externally visible.

[ Publish ]
```

추상적인 `Allow command?`보다 의미 기반 승인을 우선한다.

---

# 29. Secrets / Privacy Boundary

Project에서 secret을 발견해도 모델 context에 무조건 보내지 않는다.

```text
.env
credentials
private keys
tokens
known secret patterns
```

은 기본 redaction 대상이다.

HKCodexUI는 가능하면:

```text
secret value
```

대신:

```text
SECRET_X is present / missing
```

상태만 agent에게 제공한다.

---

# 30. Skills, Rules, Hooks, MCP를 하나의 Workbench 개념으로 통합

사용자는 파일 위치를 외울 필요가 없다.

```text
Project Behavior

Rules
Skills
Hooks
Tools / MCP
Permissions
```

UI에서 다음을 관리한다.

```text
적용 범위
project/task/global
언제 로드되는가
무엇을 할 수 있는가
context cost
최근 사용
```

## Hooks

AI가 판단할 필요 없는 deterministic automation에 사용한다.

예:

```text
After file edit → formatter
Before commit   → secret scan
After CI fail   → collect annotations
Task complete   → notification
```

Lifecycle core는 여전히 Supervisor에 남긴다.

---

# 31. Skill Studio

장기적으로 Skill을 파일 편집 없이 만들 수 있게 한다.

```text
[ New Skill ]

Name
When to use
Instructions
Allowed tools
References
Verification
```

Skill 테스트 기능:

```text
Test against sample prompt
Why was this Skill selected?
What context was loaded?
```

Project에서 자주 반복하는 성공 workflow는 Skill 후보로 제안할 수 있다.

단, 사용자 승인 없이 project rules를 스스로 영구 변경하지 않는다.

---

# 32. Reusable Runbooks

반복 운영 작업은 Task template보다 강한 Runbook으로 만든다.

예:

```text
Dependency Upgrade
Release Candidate
Hotfix
Database Migration Check
Weekly Maintenance
PR Review
```

Runbook은:

```text
inputs
steps
gates
permissions
rollback strategy
```

를 가진다.

자연어 flexibility와 deterministic workflow를 섞는다.

---

# 33. Command Palette / Keyboard-first UX

고급 사용자는 마우스로 모든 탭을 찾아다닐 필요가 없다.

```text
Ctrl+K

Start Task
Open Task
Pause Task
Run Release Verification
Switch Project
Open Diff
Approve Pending
Create Checkpoint
```

Task switcher는 매우 빨라야 한다.

---

# 34. Quick Actions

Task 상태에 따라 가장 가능성 높은 action을 바로 제공한다.

예:

```text
CI failed

[ Let Codex fix ]
[ Retry job ]
[ View failure ]
[ Mark as flaky ]
```

또는:

```text
Task complete

[ Open diff ]
[ Create PR ]
[ Merge... ]
[ Keep branch ]
[ Cleanup ]
```

---

# 35. Evidence Explorer

왜 PASS/FAIL인지 추적할 수 있어야 한다.

Gate 하나를 클릭하면:

```text
Gate: Windows package builds
Status: PASS

Evidence
Command       npm run package:win
Exit code     0
Artifact      dist/HKCodexUI.exe
SHA           a91d381
Run time      4m 12s
Timestamp     ...
```

Semantic gate도 근거를 보관한다.

---

# 36. Decision Log

긴 작업에서 중요한 설계 결정만 별도로 축약한다.

예:

```text
Decision
Use SQLite instead of JSON state file

Reason
Concurrent task updates and crash-safe transactions are required.

Source
Task #18 / architecture turn
```

모든 agent message를 다시 읽지 않아도 된다.

---

# 37. Task Replay / Export

Task를 재현 가능한 bundle로 내보낼 수 있다.

예:

```text
objective
constraints
key decisions
final diff
commit SHAs
verification results
CI evidence
agent summaries
```

용도:

```text
audit
bug report
handoff
team review
reproduction
```

---

# 38. Repository Timeline

Task 단위뿐 아니라 프로젝트 전체 변화를 볼 수 있다.

```text
Today
✓ Auth race fixed
✓ React upgraded
● Windows release validating

Yesterday
✓ PR #132 review repaired
× migration experiment cancelled
```

어떤 agent가 어떤 branch를 건드렸는지 한눈에 보여준다.

---

# 39. Cross-repository Tasks

최종적으로 하나의 제품 변경이 여러 repo를 건드릴 수 있다.

```text
Feature X
├─ API repo
├─ Web repo
└─ SDK repo
```

각 repository는 별도 workspace와 verification을 갖고 부모 Task가 dependency를 관리한다.

이 기능은 single-repo reliability가 충분히 안정된 뒤 도입한다.

---

# 40. Release Workbench

Release는 단순 push보다 별도 lifecycle이다.

```text
version check
changelog
full verification
artifact build
artifact digest
signing
release notes
Git tag
GitHub release
deployment
post-release smoke
```

각 외부 side effect에 별도 approval policy를 적용한다.

Rollback 가능한 제공자는 rollback action을 명확히 제공한다.

---

# 41. Artifact Center

Task 결과물이 코드만은 아니다.

```text
executables
installers
archives
coverage reports
screenshots
benchmark results
logs
release notes
```

Task에 생성된 artifact를 한 화면에서 보고 exact SHA와 연결한다.

---

# 42. Performance / Cost Dashboard

사용자가 agent 사용 패턴을 이해할 수 있다.

```text
Task duration
Codex active time
waiting time
verification time
CI time
tokens per phase
repair count
first-pass success rate
```

목표는 vanity metric이 아니라 병목 발견이다.

예:

```text
Task took 42m
Codex active       8m
Local verification 6m
CI waiting         27m
User waiting        1m
```

이 경우 agent가 느린 것이 아니라 CI가 병목임을 바로 알 수 있다.

---

# 43. Rate-limit-aware Scheduler

Quota가 부족할 때 무작정 실패시키지 않는다.

```text
Task A · requires Codex
Task B · only waiting CI
Task C · deterministic verification available
```

이면 Scheduler가 가능한 작업을 계속 진행한다.

reset 예정 시각이 알려져 있으면:

```text
Waiting for Codex capacity
Resumes automatically at ...
```

으로 표시한다.

---

# 44. Focus / Quiet Mode

사용자가 코딩 중일 때 autonomous agents가 계속 팝업을 띄우면 안 된다.

```text
Normal
Focus
Do not disturb
Overnight
```

Focus mode에서는 critical decision만 방해한다.

나머지는 Task Center에 쌓는다.

---

# 45. Remote Event Channels

향후 원하는 경우 다음 인터페이스로 Task를 맡길 수 있다.

```text
Slack
Discord
Telegram
mobile app
web dashboard
custom webhook
```

예:

```text
"HKCodex, issue #421 고쳐"
```

→ 로컬 Workbench에 Task 생성
→ 완료 후 해당 채널에 결과 요약

실제 실행 권한은 로컬 HKCodexUI policy가 결정한다.

---

# 46. Agent Attention Model

UI에서 Task를 단순 status color로만 표현하지 않는다.

사용자 관점의 attention state를 별도로 둔다.

```text
No attention needed
Working
Waiting externally
Needs decision
Needs permission
Blocked
Ready for review
Complete
```

가장 중요한 질문은:

> **내가 지금 뭘 해야 하는가?**

이다.

Task Center 상단에:

```text
Needs You (2)
```

를 명확히 제공한다.

---

# 47. Smart Cleanup

작업이 끝난 뒤 branch/worktree가 계속 쌓이지 않게 한다.

하지만 자동 삭제도 위험하다.

완료 후:

```text
Task complete

Branch task/auth-refresh
Worktree clean
PR merged

[ Cleanup now ]
[ Keep 7 days ]
[ Keep ]
```

Project policy로 자동 cleanup 보존기간을 정할 수 있다.

---

# 48. Recovery Center

문제가 생겼을 때 일반 error dialog 대신 복구 가능한 선택지를 제공한다.

예:

```text
Recovery conflict

Database expected SHA: 91a...
Local branch SHA:       c82...
Remote branch SHA:      91a...

Local branch changed outside HKCodexUI.

[ Inspect changes ]
[ Adopt local state ]
[ Restore task state ]
[ Detach task ]
```

시스템이 모르는 상태를 억지로 자동 해결하지 않는다.

---

# 49. Compatibility Doctor

Codex/App Server가 빠르게 변할 수 있으므로 HKCodexUI 시작 시 capability를 진단한다.

```text
Codex version
protocol version
supported methods
Skills support
review support
multi-agent support
usage support
```

지원되지 않는 기능은 UI에서 명확히 비활성화한다.

장기적으로 단순 version string보다 capability detection을 우선한다.

---

# 50. Platform Extension Model

HKCodexUI 자체가 모든 integration을 하드코딩하지 않는다.

장기 extension point 후보:

```text
Task source
Event trigger
CI provider
SCM provider
Verification adapter
Artifact provider
Notification provider
Skill / Rule / Hook
Remote runner
```

하지만 V1에서 plugin framework부터 만들지는 않는다.

먼저 실제 extension point가 반복해서 필요해질 때 추출한다.

---

# 51. 무엇이 Claude 이상이어야 하는가

HKCodexUI가 정말 더 편하다고 느끼려면 다음 차이가 분명해야 한다.

## 1. Session Manager가 아니라 Work Manager

다른 도구가 여러 agent session을 잘 보여준다면, HKCodexUI는 그보다 한 단계 위에서 **실제 개발 Task와 완료 조건**을 보여준다.

## 2. Goal continuation보다 강한 durable lifecycle

모델이 계속 스스로 turn을 만드는 방식이 아니라:

```text
agent needed      → wake
CI waiting        → sleep
review arrived    → wake
quota unavailable → sleep
```

로 동작한다.

## 3. 정확한 completion evidence

`goal evaluator says done`보다 강한:

```text
exact SHA
local verification
CI
artifact
acceptance
```

를 묶어 완료를 증명한다.

## 4. Crash가 작업을 끝내지 않는다

앱/PC 재시작이 Task 종료가 되어서는 안 된다.

## 5. 사용자는 세션을 관리하지 않는다

`agent 1`, `agent 2`, `agent 3`보다:

```text
Auth bug
Release 1.4
Dependency upgrade
```

를 관리한다.

필요한 agent topology는 시스템 내부 문제다.

## 6. CI와 PR이 agent 바깥에 있지 않다

실제 software delivery loop 전체를 같은 Task 안에서 관리한다.

## 7. 요구사항이 완료 조건으로 변환된다

자연어 goal과 실제 검증 사이의 간극을 줄인다.

## 8. Attention UX

사용자는 "진행 중인 모든 것"보다 **지금 내가 개입해야 하는 것**을 먼저 본다.

---

# 52. 최종 UI 방향

장기적으로 메인 화면은 채팅앱보다 운영 콘솔에 가깝다.

```text
┌ HKCodexUI ───────────────────────────────────────────────────────┐
│  Needs You 2     Working 4     Waiting 3     Complete Today 7   │
├──────────────┬───────────────────────────────────┬───────────────┤
│ PROJECTS     │ TASKS                             │ TASK DETAIL   │
│              │                                   │               │
│ SimpleVTT    │ ● Release V1      CI              │ Release V1    │
│ HKCodexUI    │ ! Auth bug        Needs approval  │               │
│ StayOps      │ ◐ Dependency      Working         │ Phase: CI     │
│              │ ✓ Toolbar fix     Complete        │ Agent: asleep │
│              │                                   │               │
│              │                                   │ Gates 68/72   │
│              │                                   │ CI 3/4        │
├──────────────┴───────────────────────────────────┴───────────────┤
│ Overview | Chat | Timeline | Diff | Tests | CI | Gates | Files  │
└─────────────────────────────────────────────────────────────────┘
```

Chat은 여전히 강력해야 하지만 제품의 유일한 중심은 아니다.

---

# 53. Roadmap Horizons

이 문서는 구현 순서를 강제하지 않지만 큰 방향은 다음처럼 나눈다.

## Horizon 0 — Reliable Autonomous Core

현재 `PLAN.md`의 V1.

```text
Task persistence
Codex App Server
local verification
worktree
Git / push
CI wait / repair
completion gates
crash recovery
```

## Horizon 1 — Best Daily Driver

```text
Task Center
project auto-discovery
checkpoints / rewind
better diff UX
permission profiles
command palette
tray/background
notifications
Task templates
environment doctor
verification profiles
```

## Horizon 2 — Long-running Workbench

```text
queue
Overnight Mode
Watch Tasks
PR babysitter
external event triggers
rate-limit scheduler
remote control
Task graph
```

## Horizon 3 — Agent OS

```text
multi-agent specialists
cross-repo Task graphs
release workbench
artifact center
remote workers
team/shared Task handoff
extension SDK
```

---

# 54. Scope Rule

Vision이 넓다고 해서 V1을 한 번에 거대하게 만들지 않는다.

새 기능은 다음 질문을 통과해야 한다.

```text
1. 실제 사용자의 반복적인 마찰을 없애는가?
2. 모델이 할 일을 줄이거나 더 정확하게 만드는가?
3. Task lifecycle을 더 신뢰할 수 있게 만드는가?
4. 기존 단순한 메커니즘으로 해결할 수 없는가?
```

아니면 보류한다.

즉:

> **제품의 야심은 크게, 한 번의 구현 diff는 작게.**

---

# 55. 최종 제품 한 문장

> **HKCodexUI는 사용자가 코딩 에이전트 세션을 관리하는 도구가 아니라, 개발 목표를 맡기면 필요한 Agent·Workspace·검증·Git·CI·대기·복구를 스스로 조립하고 중요한 예외만 사람에게 올리는 local-first autonomous development workbench다.**
