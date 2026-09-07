# HKCodexUI

> **Codex를 대화 상대가 아니라 끝까지 일하는 개발 Worker로 사용하는 local-first autonomous development workbench.**

HKCodexUI는 Codex 자체를 다시 만드는 프로젝트가 아니다.

- **Codex** — 코드 읽기, 판단, 수정, failure 분석
- **HKCodexUI Supervisor** — Task 상태, workspace, 검증, Git, CI, 대기, 권한, 복구, 완료 판정

사용자는 개발 목표를 맡기고 중요한 예외만 처리한다.

```text
Goal
 ↓
HKCodexUI Task Contract
 ↓
Codex implementation
 ↓
Independent verification
 ├─ FAIL → evidence → Codex repair
 └─ PASS
      ↓
    Review
      ↓
   Commit / Push
      ↓
   CI / external wait
 ├─ FAIL → evidence → Codex repair
 └─ PASS
      ↓
 Acceptance evidence
      ↓
   COMPLETE
```

**Codex가 `Done`이라고 말하는 것은 완료 조건이 아니다.**

---

## 핵심 철학

### Task > Chat

Conversation은 Task를 수행하기 위한 인터페이스 중 하나다.

```text
Project
 └─ Task
     ├─ Objective
     ├─ Constraints
     ├─ Acceptance Gates
     ├─ Workspace
     ├─ Agent Thread(s)
     ├─ Verification
     ├─ Git / PR / CI
     ├─ Timeline
     └─ Completion Evidence
```

### 기다리는 동안 모델을 쓰지 않는다

```text
CI running
Agent: sleeping
Codex calls: 0
```

CI, PR review, quota, timer, webhook은 Supervisor가 기다리고 event가 생길 때만 Codex를 깨운다.

### 완료는 증거다

```text
Local verification
Exact candidate SHA/snapshot
Remote SHA
CI
Review
Acceptance gates
Artifacts where applicable
```

### Crash가 Task를 끝내지 않는다

앱/PC가 재시작되면 DB만 믿고 그대로 실행하지 않는다.

```text
Persisted state
+ actual worktree
+ actual Git SHA
+ remote SHA
+ CI/external state
= reconciled next action
```

### 사람은 예외를 관리한다

메인 UX의 가장 중요한 질문은 다음이다.

> **내가 지금 뭘 해야 하는가?**

장기적으로 `Needs You`가 모든 프로젝트의 중요한 decision/permission/blocker를 한곳에 모은다.

---

## 제품 방향

HKCodexUI는 단순 Codex UI보다 훨씬 넓은 제품을 목표로 한다.

### H0 — Reliable Autonomous Core

```text
Project onboarding / trust
Codex App Server
Durable Tasks
isolated worktree
local verification
repair loop
semantic review
commit / push
GitHub Actions
exact-SHA completion
crash recovery
Skills / permissions
Windows package
```

### H1 — Best Daily Driver

```text
Global Task Center / Needs You
Project Brain
Environment Doctor
Verification Profiles
Checkpoints / rewind
Semantic Diff
Command Palette
Notifications / tray
Universal Task Inbox
Acceptance Compiler
```

### H2 — Long-running Workbench

```text
Task queue
Overnight Mode
quota-aware scheduler
Watch Tasks
PR Babysitter
event triggers
Task Graph
Runbooks
remote control over user-owned network
```

### H3 — Agent OS

```text
specialist agents
cross-repository Tasks
visual verification
Release Workbench
Artifact Center
Task replay/export
provider extension layer
reliability/performance dashboard
```

상세 장기 아이디어는 [`VISION.md`](./VISION.md)에 있다.

---

## UI 방향

채팅앱보다 **Task 운영 콘솔**에 가깝다.

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
├────────────┴──────────────────────────┴─────────────────────────────┤
│ Overview | Chat | Timeline | Diff | Tests | CI | Gates | Skills   │
└────────────────────────────────────────────────────────────────────┘
```

사용자는 항상 다음을 알 수 있어야 한다.

```text
지금 무엇을 하는가?
Codex가 active인가 sleeping인가?
왜 기다리는가?
무엇이 실패했는가?
내가 개입해야 하는가?
완료까지 무엇이 남았는가?
```

---

## H0 기술 방향

```text
Electron + React + TypeScript
          │
          ▼
      Electron Main
          │
      Supervisor
   ├─ Task Engine
   ├─ Scheduler
   ├─ Codex Adapter
   ├─ Verification Engine
   ├─ Git Manager
   ├─ GitHub/CI Watcher
   ├─ Approval Engine
   ├─ Recovery Engine
   └─ SQLite
          │
          ▼
 codex app-server --stdio
```

Renderer는 shell/Git/Codex/DB를 직접 실행하지 않는다.

H0 Codex App Server tested baseline은 `0.153.4`. stable protocol surface를 우선하고 experimental API에 핵심 correctness를 의존하지 않는다.

---

# 구현을 시작하는 Codex에게

이 저장소는 이제 단순 아이디어 문서가 아니라 **실행 가능한 구현 계약**을 가진다.

먼저 [`AGENTS.md`](./AGENTS.md)를 읽는다.

그 다음:

```text
1. SPEC.md
2. ARCHITECTURE.md
3. ROADMAP.md
4. PLAN.md
5. VISION.md
6. README.md
```

순으로 읽는다.

현재 handoff는 [`STATUS.md`](./STATUS.md)에 있다.

현재 Next Exact Action:

> **`ROADMAP.md`의 `WP-00 — Repository Bootstrap`부터 구현한다.**

---

## 문서 역할

| Document | 역할 |
|---|---|
| [`AGENTS.md`](./AGENTS.md) | Codex/코딩 에이전트 실행 규칙 |
| [`SPEC.md`](./SPEC.md) | **제품 동작의 canonical source of truth** |
| [`ARCHITECTURE.md`](./ARCHITECTURE.md) | **기술 구조·DB·IPC·프로세스의 canonical contract** |
| [`ROADMAP.md`](./ROADMAP.md) | **Work Packet 구현 순서·테스트·release gates** |
| [`STATUS.md`](./STATUS.md) | 현재 구현 handoff / next WP |
| [`PLAN.md`](./PLAN.md) | 초기 H0/V1 lifecycle 설명 |
| [`VISION.md`](./VISION.md) | Claude 이상을 목표로 하는 장기 제품 방향 |

문서가 충돌하면 제품 동작은 `SPEC.md`, 기술 경계는 `ARCHITECTURE.md`, 구현 순서는 `ROADMAP.md`가 우선한다.

---

## 현재 상태

```text
Planning / specification     READY
Application implementation  NOT STARTED
Next                         WP-00
```

즉 지금부터는 제품 기획을 다시 발명하는 단계가 아니라, Work Packet을 하나씩 구현하고 실제 acceptance를 통과시키는 단계다.
