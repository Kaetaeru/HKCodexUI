# HKCodexUI

> Codex를 **끝까지 일하는 코딩 에이전트**로 만드는 로컬 데스크톱 Workbench.

HKCodexUI는 Codex를 새로 만드는 프로젝트가 아니다.

**Codex는 코드 분석·설계·수정을 담당하고, HKCodexUI는 작업의 전체 수명주기를 관리한다.**

사용자는 목표를 한 번 입력하고, HKCodexUI는 구현 → 검증 → Git → CI → 실패 수정 → 최종 검증을 실제 완료 조건이 만족될 때까지 이어간다.

---

## 왜 만드는가

일반적인 코딩 에이전트는 다음과 같이 끝나기 쉽다.

```text
사용자 요청
   ↓
Codex 작업
   ↓
"완료했습니다"
   ↓
끝
```

하지만 실제 개발 작업은 그 뒤에도 남아 있다.

- 테스트가 정말 통과했는가?
- push한 정확한 commit이 CI를 통과했는가?
- CI가 실패하면 로그를 읽고 다시 고쳤는가?
- 앱을 껐다 켜도 작업을 이어갈 수 있는가?
- Codex가 완료했다고 말한 것이 아니라, 실제 acceptance criteria가 충족됐는가?

HKCodexUI는 이 부분을 담당한다.

```text
사용자 목표
   ↓
HKCodexUI Supervisor
   ↓
Codex 구현
   ↓
로컬 검증
   ├─ 실패 → Codex에게 실패 증거 전달 → 수정
   └─ 성공
        ↓
      Commit / Push
        ↓
      CI 대기
   ├─ 실패 → 실패 로그 수집 → Codex 수정
   └─ 성공
        ↓
   최종 Acceptance 검증
        ↓
      COMPLETE
```

CI나 quota를 기다리는 동안에는 Codex를 호출하지 않는다.

---

## 가장 중요한 원칙

### 1. Codex는 Worker다

Codex가 담당한다.

- 저장소와 코드 이해
- 구현 계획
- 코드 수정
- 테스트/CI 실패 원인 분석
- 의미적 최종 리뷰

### 2. HKCodexUI Supervisor가 작업의 주인이다

Supervisor가 담당한다.

- Task 상태 저장
- 작업 재개
- 로컬 검증 실행
- Git 상태 확인
- commit / push
- GitHub Actions 관찰
- 대기와 재개
- retry/stagnation 관리
- 실제 완료 여부 판정

### 3. `Codex가 완료했다고 말함 != Task 완료`

Task는 검증 가능한 증거가 모두 통과해야 완료된다.

```text
Local verification PASS
+ expected commit == pushed commit
+ required CI PASS
+ acceptance gates PASS
+ blocking review finding 없음
= COMPLETE
```

### 4. 기다리는 것은 모델의 일이 아니다

CI, rate limit, timer 등은 Supervisor가 기다린다.

```text
CI running
Codex state: sleeping
Codex calls: 0
```

이벤트가 발생했을 때만 Codex를 다시 깨운다.

### 5. 재시작해도 작업은 사라지지 않는다

Task, phase, commit SHA, verification 결과, CI run 등을 로컬 DB에 저장하고 앱 재실행 시 실제 Git/GitHub 상태와 대조해 이어간다.

---

## 사용자 경험

HKCodexUI의 중심은 `Chat`이 아니라 `Task`다.

```text
┌ Projects ─────────────────────────────────────────────────┐
│ SimpleVTT                                                 │
├──────────────┬───────────────────────────┬────────────────┤
│ TASKS        │ CURRENT TASK              │ STATE          │
│              │                           │                │
│ ● V1 release │ Complete SimpleVTT V1     │ ● CI           │
│ ✓ auth fix   │                           │                │
│ ✓ toolbar    │ Fixed MP-11 race          │ ✓ local        │
│              │ Added regression test     │ ✓ pushed       │
│              │                           │ ● GitHub CI    │
│              │ Codex sleeping            │                │
├──────────────┴───────────────────────────┴────────────────┤
│ Chat │ Timeline │ Diff │ CI │ Gates │ Skills │ Settings   │
└───────────────────────────────────────────────────────────┘
```

사용자는 항상 다음을 알 수 있어야 한다.

1. 지금 무엇을 하고 있는가?
2. Codex가 실제로 일하는 중인가, 대기 중인가?
3. 무엇이 통과했고 무엇이 실패했는가?
4. 왜 멈췄는가?
5. 완료까지 무엇이 남았는가?

---

## 실행 모드

### Chat

일반적인 Codex 대화와 단발성 작업.

### Autopilot

구현 → 로컬 검증 → 실패 수정까지 자동 반복.

### Ship

Autopilot에 더해 다음까지 관리한다.

- 격리 작업공간
- commit
- push
- GitHub Actions 대기
- CI 실패 자동 수정
- 최종 acceptance 검증

위험도가 높은 동작은 별도 권한으로 관리한다.

```text
자동 허용 가능
- task workspace 읽기/쓰기
- test/build/lint 실행
- git diff/status

Supervisor 전용
- commit
- push

기본적으로 사용자 승인 필요
- force push
- merge
- release / publish
- workspace 밖의 파괴적 변경
```

---

## V1 목표

> 사용자가 GitHub 저장소에 대한 코딩 목표를 입력하면, HKCodexUI가 Codex에게 구현시키고 로컬 테스트 및 GitHub CI 실패를 자동으로 되돌려주며, 모든 필수 검증이 통과한 정확한 commit SHA가 remote에 존재할 때까지 작업을 지속할 수 있다.

V1에 포함한다.

- Windows-first desktop app
- Codex App Server 연동
- 프로젝트 열기
- Task 생성/저장/재개
- Chat + 진행 이벤트 표시
- Skills 조회/활성화 관리
- 격리된 Git worktree
- 로컬 verification loop
- commit / push
- GitHub Actions wait/repair loop
- acceptance gates
- crash recovery
- 명확한 approval 정책

V1에 넣지 않는다.

- 여러 코딩 모델 지원
- 복잡한 멀티에이전트 orchestration
- 자동 merge
- 자동 release/publish
- cloud runner
- 완전한 IDE/코드 에디터
- 브라우저 자동화

---

## 기술 방향

```text
Electron + React + TypeScript
          │
          ▼
      Supervisor
   ├─ Task Engine
   ├─ Codex Adapter
   ├─ Verification Engine
   ├─ Git Manager
   ├─ GitHub/CI Watcher
   └─ Recovery Store (SQLite)
          │
          ▼
 codex app-server --stdio
```

Codex App Server API는 버전에 따라 변할 수 있으므로 **지원 Codex 버전을 pin**하고, 해당 버전의 `codex app-server generate-ts`로 생성한 schema를 기준으로 Adapter를 구현한다.

---

## 다음 문서

세부 상태 머신, Task lifecycle, UI, 데이터 모델, 실패 복구, milestone과 V1 acceptance criteria는 [`PLAN.md`](./PLAN.md)에 정리한다.
