# HKCodexUI Architecture

> Status: **Normative technical contract**
>
> 제품 동작은 `SPEC.md`가 우선한다. 이 문서는 그 동작을 구현하기 위한 기술 경계와 데이터 계약을 잠근다.

---

# 1. Locked Decisions

초기 구현에서 아래를 제품 결정으로 다시 논의하지 않는다.

```text
Desktop shell        Electron
Renderer             React + TypeScript
Build/dev            electron-vite
Packaging            electron-builder
Package manager      npm + committed package-lock.json
Renderer state       Zustand
Runtime validation   Zod
Database             node:sqlite (Electron bundled Node)
Unit/integration     Vitest
UI/Electron E2E      Playwright Electron
Git                  installed git CLI
GitHub                installed gh CLI in H0
Codex                 codex app-server --stdio
Primary OS           Windows 11 x64
```

Dependency patch versions은 bootstrap 시 compatible stable을 설치한 뒤 `package-lock.json`으로 고정한다.

Reference runtime baseline as of 2026-09-07:

```text
Electron 44.x
Codex CLI/App Server 0.153.4
```

Codex 0.153.4는 **tested baseline**이지 영구 hard-coded maximum이 아니다. Compatibility Doctor가 다른 버전의 core capability를 검사한다.

---

# 2. Repository Layout

첫 구현부터 아래 구조를 따른다.

```text
/
├─ AGENTS.md
├─ README.md
├─ SPEC.md
├─ ARCHITECTURE.md
├─ ROADMAP.md
├─ PLAN.md
├─ VISION.md
├─ STATUS.md
│
├─ package.json
├─ package-lock.json
├─ tsconfig.json
├─ electron.vite.config.ts
├─ electron-builder.yml
│
├─ src/
│  ├─ main/
│  │  ├─ index.ts
│  │  ├─ lifecycle/
│  │  ├─ ipc/
│  │  ├─ db/
│  │  ├─ supervisor/
│  │  ├─ codex/
│  │  ├─ process/
│  │  ├─ project/
│  │  ├─ git/
│  │  ├─ verification/
│  │  ├─ github/
│  │  ├─ security/
│  │  └─ notifications/
│  │
│  ├─ preload/
│  │  └─ index.ts
│  │
│  ├─ renderer/
│  │  ├─ main.tsx
│  │  ├─ app/
│  │  ├─ components/
│  │  ├─ features/
│  │  ├─ stores/
│  │  └─ styles/
│  │
│  └─ shared/
│     ├─ contracts/
│     ├─ domain/
│     └─ utils/
│
├─ migrations/
│  └─ 0001_initial.sql
│
├─ tests/
│  ├─ unit/
│  ├─ integration/
│  ├─ e2e/
│  ├─ fixtures/
│  └─ helpers/
│
├─ scripts/
│  ├─ codex-schema.ts
│  ├─ compatibility-smoke.ts
│  └─ package-smoke.ts
│
└─ vendor/
   └─ codex-schema/
```

규칙:

- `shared/`는 Node/Electron-only API를 import하지 않는다.
- Renderer는 `main/`을 import하지 않는다.
- Main은 Renderer component를 import하지 않는다.
- protocol generated types는 `src/main/codex/` 밖으로 새지 않는다.

---

# 3. Process Model

```text
Electron Renderer
      │ typed preload API
      ▼
Electron Main
      │
      ├─ Supervisor
      ├─ SQLite
      ├─ Git / gh child processes
      ├─ verification child processes
      └─ Codex App Server child process
              │ JSONL stdio
              ▼
          Codex Core
```

## Main process owns

```text
Task canonical state
all child process lifecycle
DB
filesystem writes outside renderer bundle
Git
GitHub/gh
Codex RPC
permissions
notifications
tray
```

## Renderer owns only

```text
presentation state
selected project/task/tab
filters
unsent composer text
modal open/closed
optimistic UI that is always reconciled with Main snapshot
```

---

# 4. Local Data Paths

Windows:

```text
Roaming state DB
%APPDATA%\HKCodexUI\state.db

Local durable runtime data
%LOCALAPPDATA%\HKCodexUI\
  wt\
  logs\
  artifacts\
  cache\
  protocol\
```

fallback는 Electron `userData` path.

Task worktree는 source repository 내부에 만들지 않는다.

```text
%LOCALAPPDATA%\HKCodexUI\wt\<projectId8>\<taskId8>
```

이 결정은 repo pollution, recursive scanning, Windows path length 위험을 줄이기 위한 것이다.

---

# 5. Domain Types

`src/shared/domain/`에 framework-independent type을 둔다.

## Project

```ts
type Project = {
  id: string
  name: string
  rootPath: string
  canonicalPath: string
  trusted: boolean
  gitRemoteUrl: string | null
  githubRepo: string | null
  defaultBranch: string | null
  currentHeadSha: string | null
  scanState: 'idle' | 'scanning' | 'ready' | 'warning' | 'blocked'
  metadataRevision: number
  createdAt: string
  updatedAt: string
}
```

## Task

```ts
type TaskStatus =
  | 'queued'
  | 'running'
  | 'waiting'
  | 'paused'
  | 'blocked'
  | 'completed'
  | 'failed'
  | 'cancelled'

type TaskPhase =
  | 'prepare'
  | 'analyze'
  | 'implement'
  | 'local_verify'
  | 'review'
  | 'commit'
  | 'push'
  | 'ci'
  | 'acceptance'
  | 'finalize'
  | 'watch'

type WaitReason =
  | 'ci'
  | 'quota'
  | 'approval'
  | 'user'
  | 'external'
  | 'timer'
  | 'pr_review'
  | null

type AttentionState =
  | 'none'
  | 'working'
  | 'waiting_external'
  | 'needs_decision'
  | 'needs_permission'
  | 'blocked'
  | 'ready_for_review'
  | 'complete'

type TaskMode = 'chat' | 'autopilot' | 'ship' | 'watch' | 'runbook'

type Task = {
  id: string
  projectId: string
  title: string
  objective: string
  contractRevision: number
  contract: TaskContract

  mode: TaskMode
  status: TaskStatus
  phase: TaskPhase
  waitReason: WaitReason
  attentionState: AttentionState
  pauseRequested: boolean

  baseBranch: string | null
  baseSha: string | null
  taskBranch: string | null
  worktreePath: string | null

  codexThreadId: string | null
  activeTurnId: string | null

  localHeadSha: string | null
  pushedHeadSha: string | null
  verifiedSha: string | null

  attemptCount: number
  repairCount: number
  lastFailure: FailureRecord | null

  revision: number
  createdAt: string
  updatedAt: string
  completedAt: string | null
}
```

## TaskContract

```ts
type TaskContract = {
  objective: string
  constraints: ContractConstraint[]
  gateIds: string[]
  verificationProfile: 'quick' | 'standard' | 'release' | 'custom'
  reviewRequired: boolean
  permissionProfileId: string
  remotePolicy: 'none' | 'push_branch' | 'pull_request'
  sourceRefs: TaskSourceRef[]
}
```

H0에서는 constraints/sourceRefs가 비어 있어도 된다. schema는 처음부터 유지한다.

---

# 6. Canonical State Machine

Task state 변경은 UI component나 service가 직접 임의로 mutate하지 않는다.

```text
TaskCommand
   ↓
TaskEngine
   ↓
validate current state
   ↓
perform pure transition decision
   ↓
transaction persist
   ↓
queue side effect / next action
   ↓
WorkbenchEvent
```

## Allowed normal phase transitions

```text
prepare       -> analyze | implement | blocked | cancelled
analyze       -> implement | blocked | cancelled
implement     -> local_verify | blocked | paused | cancelled
local_verify  -> implement | review | commit | acceptance | blocked | paused | cancelled
review        -> implement | commit | acceptance | blocked | paused | cancelled
commit        -> push | blocked | paused | cancelled
push          -> ci | blocked | paused | cancelled
ci            -> implement | acceptance | waiting(ci) | blocked | paused | cancelled
acceptance    -> implement | finalize | blocked | paused | cancelled
finalize      -> completed | blocked
watch         -> waiting(external) | running | blocked | cancelled
```

Recovery는 phase를 직접 임의 jump하지 않고 `ReconcileResult`에서 안전한 next phase를 산출한다.

---

# 7. Task Runtime Serialization

각 Task는 **serialized mailbox**를 가진다.

```text
TaskRuntime(taskId)
  queue: RuntimeMessage[]
  processing: boolean
```

같은 Task에서 두 lifecycle transition이 동시에 실행되지 않는다.

메시지 예:

```text
START
CODEX_TURN_COMPLETED
VERIFICATION_COMPLETED
CI_UPDATED
USER_PAUSE
USER_CANCEL
USER_INSTRUCTION
APPROVAL_RESOLVED
TIMER_FIRED
RECOVER
```

Task 간에는 병렬 가능하지만, H0 기본 `maxActiveWriters=1`. H1에서 설정 가능.

---

# 8. Side Effect Journal / Idempotency

DB `effects` table로 외부 side effect 의도를 기록한다.

```ts
type EffectKind =
  | 'create_worktree'
  | 'codex_thread_start'
  | 'codex_turn_start'
  | 'verification_run'
  | 'git_commit'
  | 'git_push'
  | 'ci_rerun'
  | 'create_pr'
  | 'cleanup_worktree'
```

각 effect:

```text
id
Task id
kind
logical key
status: planned | started | completed | uncertain | failed
input JSON
result JSON
startedAt
completedAt
```

unique:

```text
(task_id, kind, logical_key)
```

예:

```text
git_push logicalKey = <taskBranch>:<expectedSha>
verification logicalKey = <candidateDigest>:<profileRevision>
codex repair logicalKey = repair:<failureFingerprint>:<repairNumber>
```

외부 call timeout 후 성공 여부가 불명확하면 `uncertain`. 자동 재실행 전에 reconcile한다.

---

# 9. SQLite Schema

`node:sqlite`의 synchronous transaction API를 Main process에서 사용한다. Renderer는 DB를 직접 열지 않는다.

H0 최소 tables:

## schema_meta

```text
key TEXT PRIMARY KEY
value TEXT NOT NULL
```

## projects

```text
id TEXT PRIMARY KEY
name TEXT NOT NULL
root_path TEXT NOT NULL UNIQUE
canonical_path TEXT NOT NULL UNIQUE
trusted INTEGER NOT NULL DEFAULT 0
git_remote_url TEXT
github_repo TEXT
default_branch TEXT
current_head_sha TEXT
scan_state TEXT NOT NULL
metadata_json TEXT NOT NULL DEFAULT '{}'
metadata_revision INTEGER NOT NULL DEFAULT 0
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

## tasks

```text
id TEXT PRIMARY KEY
project_id TEXT NOT NULL REFERENCES projects(id)
title TEXT NOT NULL
objective TEXT NOT NULL
contract_revision INTEGER NOT NULL DEFAULT 1
contract_json TEXT NOT NULL
mode TEXT NOT NULL
status TEXT NOT NULL
phase TEXT NOT NULL
wait_reason TEXT
attention_state TEXT NOT NULL
pause_requested INTEGER NOT NULL DEFAULT 0
base_branch TEXT
base_sha TEXT
task_branch TEXT
worktree_path TEXT
codex_thread_id TEXT
active_turn_id TEXT
local_head_sha TEXT
pushed_head_sha TEXT
verified_sha TEXT
attempt_count INTEGER NOT NULL DEFAULT 0
repair_count INTEGER NOT NULL DEFAULT 0
last_failure_json TEXT
revision INTEGER NOT NULL DEFAULT 0
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
completed_at TEXT
```

## gates

```text
id TEXT PRIMARY KEY
task_id TEXT NOT NULL REFERENCES tasks(id)
kind TEXT NOT NULL
name TEXT NOT NULL
required INTEGER NOT NULL
origin TEXT NOT NULL
config_json TEXT NOT NULL
status TEXT NOT NULL
candidate_key TEXT
evidence_json TEXT
updated_at TEXT NOT NULL
```

status:

```text
pending | running | pass | fail | stale | not_applicable | blocked
```

## agent_turns

```text
id TEXT PRIMARY KEY
task_id TEXT NOT NULL
thread_id TEXT
turn_id TEXT
purpose TEXT NOT NULL
client_message_id TEXT NOT NULL
status TEXT NOT NULL
input_summary TEXT
result_summary TEXT
started_at TEXT
completed_at TEXT
UNIQUE(task_id, client_message_id)
```

purpose:

```text
initial | repair | review | user_instruction | acceptance | analysis
```

## verification_runs

```text
id TEXT PRIMARY KEY
task_id TEXT NOT NULL
candidate_key TEXT NOT NULL
profile TEXT NOT NULL
profile_revision INTEGER NOT NULL
status TEXT NOT NULL
results_json TEXT NOT NULL
started_at TEXT NOT NULL
completed_at TEXT
```

## git_snapshots

```text
id TEXT PRIMARY KEY
task_id TEXT NOT NULL
kind TEXT NOT NULL
head_sha TEXT
branch TEXT
dirty INTEGER NOT NULL
diff_digest TEXT
metadata_json TEXT NOT NULL
created_at TEXT NOT NULL
```

## ci_runs

```text
id TEXT PRIMARY KEY
task_id TEXT NOT NULL
provider TEXT NOT NULL
candidate_sha TEXT NOT NULL
provider_run_id TEXT
status TEXT NOT NULL
jobs_json TEXT NOT NULL
failure_json TEXT
first_seen_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

## approvals

```text
id TEXT PRIMARY KEY
task_id TEXT
capability TEXT NOT NULL
scope TEXT NOT NULL
status TEXT NOT NULL
request_json TEXT NOT NULL
resolution_json TEXT
created_at TEXT NOT NULL
resolved_at TEXT
```

## effects

위 Side Effect Journal 정의.

## events

```text
seq INTEGER PRIMARY KEY AUTOINCREMENT
id TEXT NOT NULL UNIQUE
task_id TEXT
project_id TEXT
type TEXT NOT NULL
level TEXT NOT NULL
summary TEXT NOT NULL
payload_json TEXT NOT NULL
created_at TEXT NOT NULL
```

## settings

```text
key TEXT PRIMARY KEY
value_json TEXT NOT NULL
updated_at TEXT NOT NULL
```

Migration은 numbered SQL file + transaction으로 수행한다. migration 실패 시 앱은 DB를 더 쓰지 않고 Recovery/backup 안내를 보여준다.

---

# 10. Workbench Events

Renderer와 Timeline은 raw service log가 아니라 normalized event를 받는다.

```ts
type WorkbenchEvent = {
  seq: number
  id: string
  projectId?: string
  taskId?: string
  type: WorkbenchEventType
  level: 'debug' | 'info' | 'attention' | 'warning' | 'error'
  summary: string
  payload: unknown
  createdAt: string
}
```

H0 type 최소:

```text
project.scan.started
project.scan.completed
project.environment.warning
project.trust.changed

task.created
task.started
task.state.changed
task.paused
task.resumed
task.cancelled
task.blocked
task.completed

agent.thread.started
agent.turn.started
agent.turn.completed
agent.turn.interrupted
agent.message
agent.command.started
agent.command.completed
agent.files.changed

verification.started
verification.step.completed
verification.completed

review.started
review.completed

git.worktree.created
git.commit.created
git.push.completed
git.conflict

ci.discovered
ci.updated
ci.failed
ci.passed

approval.requested
approval.resolved

recovery.started
recovery.completed
recovery.conflict
```

UI가 처음 연결할 때 snapshot을 받고 그 snapshot의 `lastEventSeq` 이후 event만 subscribe한다.

---

# 11. IPC Contract

Preload는 `window.hk` 하나만 노출한다.

```ts
window.hk = {
  app: {
    getBootstrap(): Promise<BootstrapSnapshot>
    chooseRepository(): Promise<RepositoryChoice | null>
    getSettings(): Promise<Settings>
    updateSettings(patch): Promise<Settings>
  },

  projects: {
    list(): Promise<ProjectSummary[]>
    open(path): Promise<Project>
    rescan(projectId): Promise<Project>
    setTrust(projectId, trusted): Promise<Project>
    get(projectId): Promise<ProjectDetail>
  },

  tasks: {
    list(query): Promise<TaskSummary[]>
    get(taskId): Promise<TaskDetail>
    create(input): Promise<Task>
    start(taskId): Promise<Task>
    pause(taskId): Promise<Task>
    stopCurrentStep(taskId): Promise<Task>
    resume(taskId): Promise<Task>
    cancel(taskId): Promise<Task>
    sendInstruction(input): Promise<void>
    cleanup(taskId): Promise<CleanupResult>
  },

  approvals: {
    resolve(input): Promise<void>
  },

  codex: {
    getCompatibility(): Promise<CodexCompatibility>
    listSkills(projectId): Promise<SkillSummary[]>
  },

  events: {
    subscribe(afterSeq, handler): Unsubscribe
  }
}
```

모든 input/output은 shared Zod schema로 runtime validate한다.

Main IPC handler는 stack trace/credential을 Renderer로 그대로 던지지 않고 sanitized error envelope을 사용한다.

---

# 12. Renderer State

Zustand store는 DB mirror가 아니다.

```text
bootstrap snapshot
+ selected IDs
+ current entity cache
+ ordered WorkbenchEvent stream
+ local UI state
```

Main event가 들어오면 affected entity를 patch하거나 필요 시 refetch한다.

Task revision이 local cache보다 낮은 event는 stale로 무시한다.

---

# 13. ProcessRunner

모든 child process는 하나의 `ProcessRunner`를 통한다.

API 의미:

```ts
run({
  executable,
  args,
  cwd,
  env,
  timeoutMs,
  cancellationSignal,
  category,
  redact,
}): ProcessResult
```

category:

```text
git
github
verification
utility
packaging
```

Codex App Server는 persistent process라 별도 `CodexProcess`가 관리한다.

## Defaults

```text
git utility timeout          120s
GitHub API command timeout   120s
project scan command          30s
verification step             30m
packaging step                45m
```

Project profile로 verification timeout override 가능.

## Output

- stdout/stderr streaming
- Renderer에는 sanitized chunk만 전송
- UI memory는 최근 line window만 유지
- persisted log도 secret scrubber를 거친 text만 저장
- log path는 Task event에 연결

## Windows cancellation

1. graceful signal/stdio close 가능한 경우 시도
2. grace period 3초
3. process tree 종료
4. 종료 결과를 event로 기록

Task Cancel/Stop 후 orphan process가 남지 않는 테스트를 작성한다.

---

# 14. Secret Scrubber

모든 agent context/log/UI event 전에 scrub 가능해야 한다.

기본 sensitive path pattern:

```text
.env
.env.*
*.pem
*.key
id_rsa*
credentials*
secrets*
```

기본 value pattern:

```text
GitHub tokens
common API-key prefixes
private-key blocks
Authorization/Bearer header values
```

Project scan은 `.gitignore`도 참고하지만 `.gitignore`만 secret boundary로 신뢰하지 않는다.

secret file은 기본적으로 `present/missing/path` metadata만 context에 전달한다.

---

# 15. Codex Process / Protocol Adapter

## 15.1 Process ownership

한 HKCodexUI Main process가 기본적으로 한 `codex app-server --stdio`를 소유한다.

```text
spawn
initialize
initialized notification
keep stdout reader alive
route JSONL messages by request id / thread id
```

app-server crash 시:

```text
mark Codex unavailable
active turn -> uncertain/failed evidence
restart with backoff
resume loaded Task threads where possible
reconcile before new turn
```

## 15.2 Stable surface only in H0

H0 startup은 `experimentalApi=false`.

필수 capability:

```text
initialize
thread/start
thread/resume
thread/read or equivalent history read
turn/start
turn/steer
turn/interrupt
turn + item notifications
review/start
skills/list
```

`command/exec`는 utility에 사용할 수 있지만 Project verification canonical runner는 ProcessRunner다.

Experimental remote control, goal continuation, multi-agent mode 등에 H0 correctness를 의존하지 않는다.

## 15.3 Generated protocol

개발 기준 Codex에서:

```text
codex app-server generate-ts --out vendor/codex-schema
codex app-server generate-json-schema --out vendor/codex-schema/json
```

generated type을 `CodexAdapter` 안에서만 사용한다.

## 15.4 Initialization

client metadata:

```text
name: hkcodexui
Title: HKCodexUI
version: app version
experimentalApi: false in H0
```

## 15.5 Thread creation

Thread는 trusted Task worktree cwd에서 시작한다.

새 Task에서 thread/start 성공 응답을 받은 즉시 DB에 thread id를 저장한다.

zero-turn thread durability에 의존하지 않도록 일반 Task는 곧바로 첫 user turn을 시작한다.

thread/start request 결과가 timeout되어 thread 생성 여부가 불명확하면 blind retry하지 않는다.

복구 순서:

```text
restart/reconnect
list/read recent candidate threads if API supports
match exact cwd + creation window
exactly one match -> adopt
0 or >1 -> recovery_conflict
```

## 15.6 Turn identity

모든 HK 생성 user turn은 `clientUserMessageId` 사용.

형식:

```text
hk:<taskId>:<purpose>:<ordinal>:<uuid8>
```

DB unique 저장.

turn/start timeout 시 새 turn을 바로 재시작하지 않고 thread history에서 client id 존재를 먼저 확인한다.

## 15.7 No concurrent start

TaskRuntime의 `activeTurnId`가 non-null이면 background scheduler는 `turn/start`를 호출하지 않는다.

사용자 immediate steering만:

```text
turn/steer(expectedTurnId = activeTurnId)
```

를 사용한다.

이 규칙으로 자동 event가 active human turn에 실수로 합쳐지는 race를 막는다.

## 15.8 Turn event projection

raw items를 다음 stable internal event로 변환한다.

```text
AgentMessageDelta
AgentMessageCompleted
CommandStarted
CommandOutput
CommandCompleted
FileChangeStarted
FileChangeCompleted
ApprovalRequested
TurnCompleted
TurnFailed
```

UI는 raw JSON-RPC를 알지 못한다.

## 15.9 Review

H0 final review는 `review/start` inline delivery 사용.

이유:

- implementer history에 review finding이 남아 repair context가 자연스럽다.
- 별도 review thread lifecycle이 필요 없다.

H1/H3에서 independent detached review가 필요한 경우 추가한다.

---

# 16. Compatibility Doctor

startup에서 다음을 검사한다.

```text
codex executable exists
codex --version
app-server can spawn
initialize succeeds
core methods respond
skills/list available
review/start available
```

결과:

```ts
type CodexCompatibility = {
  installed: boolean
  version: string | null
  baseline: 'exact' | 'newer' | 'older' | 'unknown'
  coreReady: boolean
  features: Record<string, 'supported' | 'unsupported' | 'unknown'>
  warnings: string[]
}
```

0.153.4 exact은 H0 tested path.

newer version은 stable API smoke가 통과하면 `Compatible with untested version` warning으로 사용 가능.

core RPC가 빠지면 Unsupported.

---

# 17. Project Scanner

Read-only scan은 repository script를 실행하지 않는다.

허용 command 예:

```text
git rev-parse --show-toplevel
git status --porcelain=v2 -b
git remote get-url origin
git symbolic-ref refs/remotes/origin/HEAD
git rev-parse HEAD
git ls-files
codex --version
gh auth status
```

파일 metadata read:

```text
AGENTS.md
package.json
pyproject.toml
Cargo.toml
*.sln / *.csproj
Makefile
.github/workflows/*.yml|yaml
.hkcodexui/project.json
```

H0 scan metadata는 DB JSON에 저장.

---

# 18. Project Configuration

portable optional file:

```text
.hkcodexui/project.json
```

H0 schema:

```json
{
  "verification": {
    "quick": [],
    "standard": [],
    "release": []
  },
  "requiredChecks": [],
  "commitMessageTemplate": "{title}",
  "stagnationLimit": 3
}
```

우선순위:

```text
Task explicit override
> local Project setting in HK DB
> repository .hkcodexui/project.json
> deterministic auto-discovery
```

잘못된 config는 project warning. 무시하고 추측값으로 silently override하지 않는다.

---

# 19. Verification Discovery

H0 built-in adapters:

## Node / TypeScript

manifest: `package.json`

standard candidates if script exists:

```text
lint
typecheck
test
build
```

실행은 `npm run <script>`.

`CI=1` env를 추가한다.

package-lock 존재 시 `npm`; 다른 lockfile만 존재하면 해당 package manager binary가 있는 경우 사용하고 없으면 Environment warning.

## Python

markers:

```text
pyproject.toml
pytest.ini
setup.cfg
```

available/configured 기준:

```text
pytest
ruff check .
mypy
```

설정 파일에 실제 사용 흔적이 있는 tool만 auto-add.

## Rust

```text
cargo fmt --check
cargo test
cargo clippy -- -D warnings
```

clippy는 project/CI에서 사용 흔적이 있을 때 standard에 포함.

## .NET

solution/project 발견 시:

```text
dotnet build
dotnet test
```

## Unknown project

자동으로 임의 명령을 만들지 않는다.

Codex가 구현 중 실행한 성공적인 verification 후보를 UI에 제안할 수 있지만, Supervisor canonical profile에는 사용자/Project 승인 후 저장한다.

---

# 20. Verification Runner

candidate key:

- committed candidate: commit SHA
- dirty worktree: deterministic digest of tracked+untracked intended diff + base SHA

같은 candidate key + same profile revision에서 PASS 결과는 재사용 가능.

파일 변경 발생 시 이전 dirty candidate verification은 stale.

step result:

```ts
type VerificationStepResult = {
  id: string
  label: string
  commandDisplay: string
  status: 'pass' | 'fail' | 'skipped' | 'cancelled' | 'timeout'
  exitCode: number | null
  durationMs: number
  failureFingerprint: string | null
  summary: string
  logPath: string | null
}
```

Failure fingerprint는 최소:

```text
normalized command + exit code + normalized top error/test identifiers
```

line number/timestamp/random temp path는 normalization에서 제거한다.

---

# 21. Repair Evidence Builder

Codex에 전체 log를 무조건 전송하지 않는다.

우선순위:

```text
failed command
exit code
failed test names
compiler diagnostic lines
stack trace top relevant frames
CI annotations
stderr tail around failure
```

최대 context budget은 설정 가능. H0 기본 failure evidence text cap 20k characters.

초과 시 full sanitized log path를 알려주고 Codex가 필요하면 읽게 한다.

---

# 22. Git Manager

모든 Git command는 ProcessRunner.

## Task base

Ship:

1. origin available 확인
2. safe `git fetch --prune origin`
3. base branch remote tip을 base SHA로 선택
4. task branch를 immutable base SHA에서 생성

Autopilot:

- default는 현재 local HEAD를 base SHA로 사용
- user가 다른 base를 지정할 수 있음

fetch 실패는 Ship에서 `environment/external` block. stale remote를 silently 사용하지 않는다.

## Worktree

```text
git worktree add -b <taskBranch> <worktreePath> <baseSha>
```

이미 logical effect가 completed면 실제 worktree/branch를 검사하고 재생성하지 않는다.

## Commit

commit author는 user git config 사용.

`user.name/email` 없으면 commit 전에 Environment issue.

commit message는 Task contract template.

## Push

기본:

```text
git push -u origin <taskBranch>
```

push 전/후:

```text
git ls-remote origin refs/heads/<taskBranch>
```

exact SHA 확인.

force push는 H0 automatic path에 없음.

---

# 23. GitHub Adapter H0

H0은 `gh` CLI auth를 재사용한다. HKCodexUI가 별도 GitHub token 저장 UI를 만들지 않는다.

Project Scanner:

```text
gh auth status
```

GitHub repo identity는 origin URL을 parse하고 `gh repo view`로 확인 가능.

CI access는 `gh api`를 사용하며 JSON parsing은 Main에서 한다.

최소 operations:

```text
commit checks/status read
workflow runs by SHA read
workflow jobs read
failed job log/annotations read
allowed rerun when policy permits
```

H1에서 PR create/read/review event를 추가한다.

---

# 24. CI Watcher

Task별 watcher는 DB state가 아니다. DB의 Task/CI row에서 재생성 가능한 runtime object다.

poll 기본:

```text
first 2 minutes      15s
2-10 minutes         30s
10+ minutes          60s
```

네트워크 실패 시 exponential backoff max 2m.

CI 기다리는 동안 Codex turn 없음.

## Discovery grace

push 후 60초 동안 exact SHA의 run/check 발견을 기다린다.

없으면:

1. workflow 존재 여부 확인
2. Project requiredChecks 확인
3. PR-only 가능성 분류
4. no CI / PR required / external failure 중 하나 결정

무한 spinner 금지.

## Result normalization

```text
queued
in_progress
pass
fail
cancelled
skipped
neutral
unknown
```

required `neutral/cancelled/unknown`은 자동 PASS로 취급하지 않는다.

---

# 25. Flake / CI Retry

AI repair 전에 동일 SHA rerun 가능한 조건:

```text
failure classified infra/flake suspected
AND rerun count for same SHA < 1
AND Project policy allows rerun
```

rerun 뒤 같은 failure fingerprint면 code/unknown으로 승격하고 Codex repair 또는 user attention.

---

# 26. Gate Engine

Gate evaluator interface는 내부 코드 수준에서 고정한다.

```ts
interface GateEvaluator {
  supports(gate: Gate): boolean
  evaluate(context: GateContext): Promise<GateEvaluation>
}
```

H0 evaluator:

```text
verification_profile
review
remote_sha_match
ci_checks
file_exists
command_exit
working_tree_clean
custom_manual
```

Semantic objective gate는 final review 결과를 evidence로 사용.

모든 Gate는 `candidate_key`를 저장. candidate가 바뀌면 영향 받는 gate를 stale.

---

# 27. Completion Engine

Ship 기본:

```text
candidateSha != null
verifiedSha == candidateSha
remote branch sha == candidateSha
all required local gates pass
review gate pass if enabled
CI gate pass OR not_applicable
all acceptance gates pass/not_applicable
no unresolved required approval
no active turn/process
```

Autopilot:

```text
dirtyCandidateDigest verified
required local gates pass
review pass if enabled
```

Completion transaction에서:

```text
status=completed
attentionState=complete
completedAt=now
completion evidence snapshot event
```

를 한 transaction으로 저장한다.

---

# 28. Recovery Engine

앱 시작 시 `status in (running, waiting, paused, blocked)` Task를 로드한다.

## Reconciliation order

```text
1. worktree exists?
2. branch exists?
3. actual HEAD
4. dirty state / diff digest
5. DB candidate SHAs
6. remote branch SHA if relevant
7. exact SHA CI state if relevant
8. Codex thread resumability
9. unfinished/uncertain effects
10. derive safe next action
```

## Examples

DB says push planned, remote already exact SHA:

```text
mark push effect completed
phase -> ci
```

DB says commit completed SHA A, worktree HEAD B externally changed:

```text
recovery_conflict
blocked
Needs You
```

DB says waiting CI, CI already pass:

```text
phase -> acceptance
resume without Codex
```

DB says active Codex turn but app restarted:

```text
resume/read thread
if turn terminal -> project completion event
if no reliable state -> mark prior turn uncertain and require safe inspection before new turn
```

복구는 destructive overwrite를 자동 선택하지 않는다.

---

# 29. Approval Engine

Capability policy resolution 순서:

```text
hard safety deny
> Task explicit
> Project policy
> global setting
> default
```

Approval request는 semantic action으로 정규화한다.

```ts
type ApprovalRequest = {
  id: string
  taskId?: string
  capability: Capability
  risk: 'low' | 'medium' | 'high' | 'critical'
  summary: string
  details: Record<string, unknown>
  allowedScopes: ApprovalScope[]
}
```

Codex native approval을 HK policy가 자동 승인할 수 있는 경우에도 event를 기록한다.

Managed/system Codex policy를 우회하지 않는다.

---

# 30. Scheduler

H0:

```text
maxActiveWriters = 1
maxConcurrentVerificationProcesses = 1
CI watchers unlimited within sane provider poll cap
```

H1 settings로 CPU/agent concurrency 조절.

Scheduler candidate order:

```text
needs safe resume
> user-started foreground Task
> queued priority
> overnight queue
```

waiting-only Task는 writer slot을 점유하지 않는다.

---

# 31. H1 Architecture Additions

## Project Brain

`project_metadata`를 별도 module로 확장. cache invalidation key:

```text
HEAD SHA
manifest fingerprints
workflow fingerprints
instruction fingerprints
local config revision
```

## Checkpoints

table:

```text
checkpoints(id, task_id, kind, base_sha, diff_bundle_path, metadata_json, created_at)
```

H0 state schema에 미리 넣지 않아도 migration으로 추가.

## Notifications / tray

Main lifecycle service. Renderer 없어도 Task update 가능.

## Acceptance Compiler

별도 Codex planning turn을 사용할 수 있지만 결과는 Zod-validated structured JSON으로 파싱하고 실패 시 기존 Objective-only contract로 fallback. user_locked gate는 자동 변경 금지.

---

# 32. H2 Architecture Additions

## Queue

Task dependency table 추가:

```text
task_edges(parent_task_id, child_task_id, relation)
```

DAG validation.

## Watch sources

runtime provider:

```ts
interface EventSource {
  poll(cursor): Promise<EventBatch>
}
```

GitHub polling부터 시작. webhook은 remote endpoint가 있을 때만.

## Remote Control

별도 local HTTP/WebSocket server를 Main service가 소유.

remote API는 기존 IPC command의 안전한 subset만 call한다.

raw DB/shell endpoint 없음.

---

# 33. H3 Architecture Additions

Agent Fleet도 동일 `TaskRuntime` 규칙을 따른다.

writer lease:

```text
workspace_writer_lease(worktreePath) -> max 1
```

specialist는 stable snapshot 또는 별도 worktree.

Cross-repo parent는 repository child Task의 evidence를 aggregate하고 직접 파일을 수정하지 않는다.

Provider plugin SDK는 실제 두 번째 provider가 생겼을 때 public extension surface로 추출한다. H0부터 plugin framework를 만들지 않는다.

---

# 34. Error Boundary

Main service error는:

```text
internal error object
→ classify
→ redact
→ FailureRecord / WorkbenchError
→ DB event
→ Renderer
```

Renderer가 Rust/Node stack trace를 일반 사용자 화면에 바로 표시하지 않는다.

Debug details에서 sanitized stack 접근 가능.

---

# 35. Logging

두 종류:

## Event log

사용자 의미 사건. DB `events`.

## Diagnostic log

app process diagnostics. file.

기본 retention:

```text
diagnostic logs 14 days
completed Task sanitized logs 30 days
Task metadata/evidence keep until user deletes Task
```

Settings에서 변경 가능.

---

# 36. Packaging

Windows H0 release artifact:

```text
NSIS installer
portable/unpacked directory for test
```

installer는 Codex/Git/gh를 묵시적으로 bundled 설치하지 않는다.

첫 실행 Environment Doctor가 dependency availability를 진단한다.

H1에서 Codex installer helper를 추가할 수 있다.

---

# 37. Test Architecture

## Unit

```text
Task state reducer
attention derivation
failure fingerprint
secret scrubber
branch naming
config merge
Gate evaluation
CI normalization
```

## Integration

실제 temp Git repo 사용:

```text
worktree create
commit
push to local bare remote
reconcile after interrupted effect
verification process timeout
```

Fake binaries/shims:

```text
fake codex app-server JSONL process
fake gh
fixture verification command
```

## E2E

Playwright Electron:

```text
first run
open repo
trust
start Task
stream fake Codex
verification failure -> repair -> pass
CI wait -> fail -> repair -> pass
crash/relaunch recovery
pause/cancel
approval
```

Real Codex/GitHub smoke는 credential-dependent라 CI required test와 분리한다.

---

# 38. Fault Injection

Recovery를 나중에 수동으로만 검증하지 않는다.

integration test에서 다음 boundary에 fault를 주입한다.

```text
after worktree created before DB complete
after Codex turn accepted before response recorded
after verification pass before phase transition
after commit before SHA persisted
after push before effect completed
while waiting CI
after CI fail before repair turn
```

각 경우 재실행 후 duplicate destructive effect가 없어야 한다.

---

# 39. Performance Requirements

H0 target:

```text
cold UI shell visible              < 3s on normal SSD PC
stored Task list initial render     < 500ms after DB ready
Task UI event update                < 250ms after Main normalized event
large log rendering                 must not freeze renderer
CI wait CPU                         near idle between polls
```

절대값보다 UI freeze 방지와 background idle 효율을 우선한다.

---

# 40. Security Requirements

- `contextIsolation=true`
- `nodeIntegration=false` in Renderer
- CSP 설정
- preload API whitelist only
- external URL open은 allowlisted http/https + system browser
- no credential value in Renderer store
- no shell command built through unescaped string concatenation; use executable + args arrays
- repository trust before code execution
- secret scrub before AI context/event persistence
- force push/base branch write hard deny by default

---

# 41. Definition of Architectural Done

Work packet이 architecture layer를 추가할 때:

```text
one canonical owner exists
runtime validation exists at trust boundary
state is recoverable if durable
side effects are idempotent/reconciled
normalized event exists
unit/integration test exists
Renderer does not bypass Main
no duplicate competing service implements same responsibility
```

`ROADMAP.md`의 순서대로 구현한다.
