# HKCodexUI Executable Roadmap

> Status: **Normative implementation order**
>
> 이 문서는 Codex가 HKCodexUI를 실제로 제작할 때 따르는 작업 순서와 각 단계의 완료 조건을 정의한다.

---

# 0. 실행 규칙

모든 구현은 Work Packet(WP) 단위다.

```text
Read current repository
→ confirm prior WP evidence
→ implement one WP
→ run WP verification
→ fix until PASS
→ update STATUS.md
→ proceed to next WP
```

한 WP가 FAIL인 상태에서 그 위의 기능을 쌓지 않는다.

큰 작업 요청을 받더라도 내부적으로 WP boundary를 유지한다.

---

# 1. 공통 Quality Gate

WP-00 이후 모든 코드 변경은 가능한 범위에서 다음을 유지한다.

```text
npm run typecheck
npm test
npm run build
```

integration/e2e가 추가된 뒤에는 해당 layer를 건드린 WP에서 반드시 해당 suite도 실행한다.

Acceptance-critical `TODO`, dummy handler, fake PASS는 허용하지 않는다.

---

# 2. Horizon 0 — Reliable Autonomous Core

H0 목표:

> trusted local Git repository를 열고 Task를 생성하면, Codex가 isolated worktree에서 수정하고 HKCodexUI가 독립 검증, review, commit, push, GitHub CI, failure repair, crash recovery까지 관리하여 exact evidence를 가지고 완료할 수 있다.

---

## WP-00 — Repository Bootstrap

### Scope

```text
Electron
React
TypeScript
electron-vite
electron-builder
Vitest
Playwright Electron
Zod
Zustand
basic CSS tokens
```

### Required files

```text
package.json
package-lock.json
tsconfig.json
electron.vite.config.ts
electron-builder.yml
src/main/index.ts
src/preload/index.ts
src/renderer/main.tsx
src/renderer/styles/*
src/shared/*
```

### Required npm scripts

```text
dev
build
typecheck
test
test:integration
test:e2e
package:win
verify:h0
```

처음에는 존재하지만 아직 suite가 없는 integration/e2e script도 정상적으로 empty PASS하게 구성하되, 실제 feature WP부터 test가 추가된다.

### UI

창에 최소 shell 렌더:

```text
HKCodexUI
No project open
[ Open Repository ]
```

### Acceptance

- Windows development launch 성공.
- Renderer nodeIntegration off, contextIsolation on.
- preload whitelist API만 존재.
- `npm run typecheck` PASS.
- `npm test` PASS.
- `npm run build` PASS.

---

## WP-01 — Database, Migrations, Typed IPC

### Scope

`ARCHITECTURE.md` H0 DB schema의 foundational subset 구현.

```text
schema_meta
projects
tasks
gates
agent_turns
verification_runs
git_snapshots
ci_runs
approvals
effects
events
settings
```

### Required modules

```text
src/main/db/database.ts
src/main/db/migrations.ts
src/main/ipc/*
src/shared/contracts/*
```

### Behavior

- first launch DB 생성.
- migration transaction.
- bootstrap snapshot IPC.
- normalized error envelope.
- event sequence persistence.

### Tests

- empty DB migration.
- reopen DB preserves rows.
- broken migration fixture rolls back.
- Zod rejects invalid IPC input.
- Renderer cannot invoke arbitrary channel name.

### Acceptance

앱 재시작 후 fixture Project/Task record가 유지되는 integration test PASS.

---

## WP-02 — Project Open, Read-only Scan, Trust

### Scope

```text
folder picker
canonical path
Git detection
remote/default branch
runtime markers
Codex/git/gh binary availability
repository trust
Project Ready screen
```

### UX

```text
Open Repository
→ Scanning
→ Trust prompt
→ Project Ready / warnings
```

read-only open 지원.

### Tests

fixtures:

```text
not-a-git-folder
plain-git-repo
node-github-repo
repo-with-bad-config
```

- untrusted repo에서 repository script 실행 0회.
- path reopen은 duplicate Project 생성 안 함.
- HTTPS/SSH GitHub origin parser.

### Acceptance

실제 temp Git repo를 열고 scan → trust → reopen까지 integration PASS.

---

## WP-03 — ProcessRunner + Secret Scrubber

### Scope

```text
spawn executable + args
stream output
timeout
cancel
Windows process-tree cleanup
sanitized logging
secret path/value scrub
```

### Tests

- stdout/stderr streaming.
- timeout command terminated.
- child-spawns-child fixture에서 tree cleanup.
- bearer token/private key fixture redacted.
- no raw secret in persisted event/log.

### Acceptance

ProcessRunner 외부에서 feature service가 직접 `child_process.spawn`하지 않도록 lint/code rule 또는 review로 유지.

---

## WP-04 — Codex App Server Adapter + Compatibility Doctor

### Scope

```text
codex --version
spawn codex app-server --stdio
initialize handshake
request routing
notification routing
thread/start
thread/resume
turn/start
turn/steer
turn/interrupt
review/start
skills/list
compatibility result
```

H0 experimental API OFF.

### Test infrastructure

`tests/helpers/fake-codex-app-server`를 만든다.

JSONL로:

```text
initialize response
thread/start
turn started
message delta
command item
file item
turn completed
approval request
review
skills
process crash
```

을 재현한다.

### Critical concurrency tests

- activeTurnId가 있을 때 automated `turn/start` 금지.
- user steer만 expectedTurnId로 active turn에 들어감.
- queued background repair가 active turn 종료 후에 시작.
- duplicate clientUserMessageId 방지.

### Compatibility

0.153.4 baseline schema를 `vendor/codex-schema`에 생성/커밋.

### Acceptance

fake server로 full streaming conversation을 UI-independent adapter test에서 PASS.

---

## WP-05 — Chat Experience

### Scope

trusted Project에서 Chat session.

```text
new thread
resume thread
message streaming
command activity card
file change event
interrupt
skills list
Codex unavailable state
```

### UI

Task-less exploratory chat도 Project 안에서 생성 가능하지만 내부적으로 `mode=chat` Task record를 사용한다.

### Tests

Playwright Electron fake Codex:

```text
send message
see streaming message
see command activity
interrupt
restart app
open same Chat Task
```

### Acceptance

Codex UI replacement의 최소 chat experience가 usable.

---

## WP-06 — Durable Task Engine

### Scope

```text
Task create
status/phase/waitReason/attention
serialized mailbox
Task commands
normalized events
Task list/detail
Pause/Resume/Cancel
```

아직 autonomous Codex repair는 연결하지 않는다.

### Unit tests

state transition matrix 전부.

특히 invalid transitions:

```text
completed -> running
cancelled -> resume
commit -> implement without explicit repair/recovery
```

등 reject.

### UI

```text
Projects sidebar
Task list
Task header
Overview
Timeline
```

### Acceptance

Task를 생성하고 여러 transition 후 앱 재시작해 동일 상태/Timeline 복원.

---

## WP-07 — Worktree + Git Manager

### Scope

```text
base SHA resolution
branch name
external durable worktree path
create/reconcile worktree
git snapshots
diff
git status
commit
remote ref check
push
```

H0는 실제 GitHub 없이 local bare remote로 integration test 가능해야 한다.

### Tests

- worktree가 source repo 내부에 생기지 않음.
- duplicate effect 재실행 시 worktree 중복 생성 안 됨.
- dirty user checkout 영향 없음.
- external branch movement detection.
- commit SHA persisted.
- local bare remote exact push SHA match.

### UI

Diff tab H0 raw diff/file list.

### Acceptance

fixture repo에서 Task branch/worktree → edit → commit → bare remote push PASS.

---

## WP-08 — Verification Discovery + Runner

### Scope

```text
Project config merge
auto discovery
Quick/Standard/Release profile model
candidate digest
verification run persistence
failure fingerprint
Tests tab
```

### Built-in adapters H0

```text
Node/TypeScript
Python
Rust
.NET
custom configured commands
```

### Tests

각 language fixture에서 command discovery.

- same candidate PASS reuse.
- file edit → previous evidence stale.
- timeout/cancel.
- multi-step independent failure collection.

### Acceptance

broken Node fixture의 failing test를 정확한 structured failure evidence로 저장.

---

## WP-09 — Autopilot Repair Loop

### Scope

```text
initial Codex implementation
independent verification
failure evidence builder
repair turn
re-verify
stagnation
```

### Scenario test

fake Codex가:

```text
turn 1 -> fixture file을 잘못 수정
verification -> fail
repair evidence 수신
turn 2 -> 올바르게 수정
verification -> pass
```

### Stagnation scenarios

- same fingerprint 3회 -> blocked.
- no diff progress -> blocked.
- progress fingerprint 변화 -> 계속 허용.

### Acceptance

Autopilot Task가 사람 추가 입력 없이 FAIL → repair → PASS.

CI/Git remote는 아직 없음.

---

## WP-10 — Review + Gates + Completion for Autopilot

### Scope

```text
review/start inline
blocking/non-blocking classification
Gate Engine
candidate staleness
Autopilot completion evidence
Gates tab
```

### Tests

- blocking review -> repair loop.
- non-blocking -> completion 가능.
- candidate changed -> old gate stale.
- Codex `Done` alone never completed.

### Acceptance

Autopilot completion UI에 verification/review evidence가 존재.

---

## WP-11 — Ship: GitHub/gh + CI Watcher

### Scope

```text
gh auth diagnosis
GitHub repo resolution
push task branch
exact SHA checks/runs
adaptive polling
jobs/failure extraction
CI sleep
CI repair
flake one-rerun rule
PR-only detection
no-CI N/A
```

### Fake gh

`tests/helpers/fake-gh` scenario files:

```text
ci-pass
ci-fail-then-pass
ci-queued-long
no-ci
pr-only
network-error
flake-rerun-pass
remote-sha-moved
```

### Critical tests

- CI waiting 동안 Codex turn count 변화 0.
- CI code fail 후 repair turn 정확히 1개.
- exact SHA가 바뀌면 이전 CI PASS 재사용 안 함.
- PR-only는 false completion 금지.
- no CI project는 CI gate `not_applicable`.

### Acceptance

fake end-to-end Ship:

```text
implement
verify pass
review pass
commit
push
CI fail
repair
verify
new commit
push
CI pass
complete
```

---

## WP-12 — Recovery + Side Effect Journal

### Scope

`ARCHITECTURE.md` fault injection 전부.

### Required recovery scenarios

```text
after worktree OS effect before DB complete
after turn accepted before recorded
after verify pass before phase write
after commit before SHA persist
after push before effect completion
while CI waiting
after CI failure before repair
external worktree HEAD changed
remote branch changed
```

### Behavior

- known state → auto reconcile.
- ambiguous/destructive conflict → blocked + Recovery UI.
- duplicate commit/push/turn 금지.

### Acceptance

모든 fault-injection integration test PASS.

이 WP 전에는 H0 autonomous reliability를 완료했다고 부르지 않는다.

---

## WP-13 — Approval, Skills, Security Boundaries

### Scope

```text
capability policy
approval queue
Codex native approval mapping
Skills tab
skills changed invalidation
secret scrub enforcement
external URL policy
CSP
```

### Tests

- force push default deny.
- outside-workspace destructive action deny.
- project-scoped allow persistence.
- secret never appears in Renderer event fixture.
- Skill refresh after changed event.

### Acceptance

Security requirements in `ARCHITECTURE.md` automated/inspectable.

---

## WP-14 — H0 UX Completion + Background + Windows Package

### Scope

```text
all empty/loading/error states
active Task close-to-tray
system notifications
Task completion evidence
useful error/recovery cards
responsive 3-column layout
System/Dark/Light theme
keyboard basics
NSIS package
```

### E2E journeys

1. first run → open repo → trust → Chat.
2. Autopilot failing test → repair → complete.
3. Ship CI fail → repair → complete.
4. close window during CI → background → reopen.
5. force app kill during CI → relaunch → recover.
6. recovery conflict → user action available.

### H0 Release Gate

```text
npm run typecheck
npm test
npm run test:integration
npm run test:e2e
npm run build
npm run package:win
npm run verify:h0
```

`verify:h0`는 위 required automated suites를 조합하는 script다.

Windows installer를 fresh test user profile에서 launch smoke한다.

---

# 3. Horizon 1 — Best Daily Driver

H1은 H0 lifecycle을 변경하기보다 사람의 마찰을 줄인다.

---

## WP-15 — Global Task Center + Attention UX

### Features

```text
cross-project Task list
Needs You section
Working/Waiting/Recent filters
quick project/task switcher
search
attention badge rules
```

### Acceptance

사용자가 프로젝트를 클릭하지 않아도 현재 개입 필요한 Task를 찾을 수 있다.

---

## WP-16 — Environment Doctor + Better Project Scan

### Features

```text
runtime/version problems
package manager
Git/gh/Codex diagnosis
Fix automatically when safe
Show steps
ignore per Task
```

system-wide change는 approval.

### Acceptance

대표 missing Node/package manager/Git auth/Codex unavailable fixture를 actionable state로 표현.

---

## WP-17 — Project Brain + Context Inspection

### Features

```text
metadata cache
invalidation fingerprints
repository map
verification/CI topology
instructions summary
Why is this in context?
context source display
```

AI 추측을 canonical metadata로 자동 저장 금지.

### Acceptance

manifest/workflow/AGENTS change 후 affected cache stale/rebuild.

---

## WP-18 — Checkpoints / Rewind / Fork

### Features

```text
auto checkpoints
manual checkpoint
compare
restore files
fork Task from checkpoint
```

### Acceptance

restore 후 old verification/CI gate stale.

기존 original Task evidence는 history로 보존.

---

## WP-19 — Semantic Diff + Command Palette + Quick Actions

### Features

```text
diff summary
behavior summary
raw diff
ask about diff
Ctrl+K actions
state-aware quick actions
```

Summary가 raw diff를 감추지 않는다.

---

## WP-20 — Universal Inbox + Acceptance Compiler v1

### Sources

```text
GitHub issue
PR
CI URL
text/log file
image metadata attachment
```

H1 image는 저장/표시/Task source 제공까지. full visual agent verification은 별도 WP.

### Compiler

```text
objective
constraints
derived gates
risk hints
```

structured output validation 실패 시 Task 생성 자체를 막지 않고 raw objective fallback.

user_locked gate 보호.

---

## WP-21 — Smart Cleanup + Rich Notifications + Usage View

### Features

```text
cleanup now / keep days / keep
log retention
notification dedupe
active/wait/Codex time breakdown
repair count
```

### H1 Release Gate

H0 suites + H1 E2E:

```text
Needs You
checkpoint restore
universal issue source
Project Brain invalidation
cleanup retention
```

---

# 4. Horizon 2 — Long-running Workbench

---

## WP-22 — Queue + Overnight + Quota-aware Scheduler

### Features

```text
Task priority
dependencies
writer slot
Overnight policy
continue independent Tasks when one blocks
quota wait/resume
morning digest
```

### Acceptance

3 Task fixture 중 하나 blocked여도 독립 두 Task 완료.

quota wait 동안 Codex call 0.

---

## WP-23 — Watch Tasks + GitHub Event Polling

### Features

```text
scheduled poll source
CI change
issue assignment
PR state
review state
cursor persistence
```

이벤트 없을 때 agent call 0.

---

## WP-24 — PR Lifecycle / Babysitter

### Features

```text
create draft PR
PR status
review requested changes
comment provenance
branch behind
conflict
push repair
CI re-wait
```

### Rules

- `REQUEST_CHANGES`/inline actionable review → repair candidate.
- ordinary comment → user-visible note, 자동 code change 금지 unless explicitly classified instruction.
- merge automatic 기본 off.

### Acceptance

fake PR comment → requirement revision → repair → new CI → ready for review.

---

## WP-25 — Task Graph

### Features

```text
DAG model
child Task
parallel independent work
parent progress
integration child
evidence aggregation
cycle prevention
```

### Acceptance

API + frontend independent child → integration Task example.

writer conflict 없는 병렬 실행.

---

## WP-26 — Runbooks + Scheduled Tasks

### Features

```text
Runbook schema
input form
deterministic step
agent step
gate
permission
schedule
```

첫 built-in examples:

```text
Dependency Upgrade
Release Candidate Check
PR Review
```

---

## WP-27 — Local Remote Control

### Scope

```text
optional local HTTP/WebSocket companion
localhost default
explicit network bind
pairing token
revoke
status
approve/reject
pause/resume
simple instruction
```

raw shell/source browser 제공하지 않는다.

### Security acceptance

- unpaired client reject.
- token revoke immediate.
- remote capability subset enforced server-side.
- LAN bind explicit approval.

### H2 Release Gate

72h soak scenario를 자동화 가능한 수준으로 축약해 반복 테스트:

```text
many waits/restarts
queued Tasks
PR events
quota simulation
no duplicate effects
```

---

# 5. Horizon 3 — Agent OS

---

## WP-28 — Agent Specialists

### Features

```text
implementer
researcher
test investigator
security reviewer
UX reviewer
release auditor
```

agent topology는 Task 내부 detail이고 메인 UI는 responsibility/evidence 중심.

writer lease 위반 불가.

---

## WP-29 — Cross-repository Task Graph

### Features

```text
parent Goal
repo child Tasks
cross-repo dependencies
integration order
per-repo gates
```

한 repo failure가 다른 independent child의 worktree를 변경하지 않음.

---

## WP-30 — Visual Verification

### Features

```text
dev server lifecycle
browser target
screenshot evidence
console error capture
accessibility smoke
before/after
user annotation source
```

Visual result도 exact candidate key와 연결.

---

## WP-31 — Release Workbench + Artifact Center

### Features

```text
release Task type
version/changelog
release verification
artifact capture
digest
signing hook
tag
GitHub release
deploy approval
post-release smoke
```

Artifact metadata:

```text
path/name
digest
size
candidate SHA
producer step
createdAt
```

---

## WP-32 — Task Replay / Export + Decision Log

### Features

```text
contract
important decisions
sanitized timeline
SHAs
diff
verification
CI
artifacts
export bundle
```

secret scrub mandatory.

---

## WP-33 — Provider Extension Layer

이 시점에만 실제 반복된 provider boundary를 public interface로 추출한다.

후보:

```text
SCM
CI
Task source
Event source
Notification
Verification
Artifact
Remote worker
```

plugin framework를 위해 기존 단순 코드를 대규모 rewrite하지 않는다.

---

## WP-34 — Performance / Cost / Reliability Dashboard

### Metrics

```text
wall time
agent active
verification
CI wait
user wait
tokens where available
repairs
first-pass rate
recovery count
flake reruns
```

metric 수집이 Task execution correctness를 block하면 안 된다.

### H3 Release Gate

- H0/H1/H2 regression suites PASS.
- Cross-repo + agent specialist + release artifact scenario PASS.
- remote control security regression PASS.
- Task export redaction PASS.
- one-writer invariant stress PASS.

---

# 6. Manual Compatibility Matrix

각 major release에서 최소 확인:

```text
Windows 11 current
Git HTTPS remote
Git SSH remote
GitHub repo with push CI
GitHub repo with PR-only CI
repo without CI
Node/TS project
Python project
Rust project
.NET project
Codex tested baseline
Codex current stable
```

가능한 것은 fixture automated test로 승격한다.

---

# 7. STATUS.md 규칙

Codex는 각 WP 완료 후 `STATUS.md`만 현재 실행 상태로 갱신한다.

`ROADMAP.md`에 완료 체크박스를 대량 추가해 merge conflict를 만들지 않는다.

STATUS 최소:

```text
Current horizon
Last completed WP
Next WP
Current branch/SHA
Verification last result
Known blockers
```

문서의 acceptance 기준을 낮추기 위해 STATUS를 수정하면 안 된다.

---

# 8. 전체 제품 완료 의미

`VISION.md`의 모든 아이디어를 무조건 구현했다는 뜻이 아니다.

HKCodexUI가 H3까지 제작 완료라고 부르려면 최소:

```text
H0 reliable lifecycle
H1 daily-driver UX
H2 unattended long-running operation
H3 specialist/cross-repo/release foundation
```

가 각 Release Gate를 통과해야 한다.

추가 Vision 아이디어는 동일 Work Packet 규칙으로 이후 확장한다.
