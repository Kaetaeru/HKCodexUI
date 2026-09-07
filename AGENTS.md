# HKCodexUI Agent Instructions

이 저장소에서 구현 작업을 수행하는 Codex/코딩 에이전트는 이 문서를 따른다.

## 1. Source of truth order

제품/구현 판단이 충돌하면 다음 순서다.

1. `SPEC.md` — 제품 동작과 사용자 의미
2. `ARCHITECTURE.md` — 기술 경계, 데이터/프로세스 계약
3. `ROADMAP.md` — 구현 순서와 acceptance
4. `PLAN.md` — 초기 V1 설명/배경
5. `VISION.md` — 장기 방향과 아이디어
6. `README.md` — 프로젝트 소개

`STATUS.md`는 진행 상황만 기록하며 요구사항을 변경하지 않는다.

## 2. 시작 전

항상 현재 저장소 전체 구조와 위 source-of-truth 문서를 읽고, 이미 구현된 코드를 기준으로 다음 Work Packet을 결정한다.

`STATUS.md`가 stale하면 실제 코드/테스트 상태를 우선하고 STATUS를 바로잡는다.

## 3. Work Packet discipline

`ROADMAP.md`의 Work Packet을 기본 작업 단위로 사용한다.

- 가능한 한 한 WP를 완성한 뒤 다음으로 간다.
- 이전 WP acceptance가 깨진 상태에서 다음 WP를 쌓지 않는다.
- 사용자가 여러 WP를 한 번에 완료하라고 해도 내부적으로 WP boundary마다 검증한다.
- 이미 통과한 기능을 이유 없이 재작성하지 않는다.

## 4. Implementation style

- 가장 작은 충분한 구현을 선택한다.
- `ARCHITECTURE.md`에 없는 추상화/plugin framework를 미리 만들지 않는다.
- 같은 책임을 두 service가 소유하게 만들지 않는다.
- Renderer에서 shell/Git/Codex/DB를 직접 호출하지 않는다.
- generated Codex protocol type을 Codex Adapter 밖으로 노출하지 않는다.
- 사용자 workspace를 autonomous writer가 직접 수정하지 않는다.
- acceptance-critical code에 placeholder/mock/TODO를 남기고 완료 처리하지 않는다.

## 5. Product decisions

명세에 답이 있으면 질문하지 말고 그대로 구현한다.

명세에 없는 작은 구현 세부는 다음 순서로 결정한다.

1. 기존 repository pattern
2. 가장 단순한 안전한 방법
3. 기존 dependency/표준 library 재사용
4. 새 dependency는 실질적 이득이 있을 때만

정말로 제품 의미를 바꾸는 미정 사항이 발견되면 임의의 큰 기능을 만들지 않는다. 가장 보수적인 동작으로 막고 `STATUS.md` Known blockers에 구체적으로 기록한다.

## 6. Verification

WP 완료 전 해당 WP의 acceptance와 tests를 실제로 실행한다.

WP-00 이후 기본:

```text
npm run typecheck
npm test
npm run build
```

관련 layer가 존재하면:

```text
npm run test:integration
npm run test:e2e
```

H0 release 후보는 `npm run verify:h0`와 Windows packaging까지 통과해야 한다.

테스트를 통과시키기 위해 test/gate를 약화하거나 제거하지 않는다.

## 7. Recovery is not optional

stateful/external side effect 기능을 구현할 때 normal path만 만들지 않는다.

다음을 함께 구현한다.

```text
persist
restart/reconcile
idempotency or side-effect journal
uncertain outcome handling
normalized event
failure/recovery UI where applicable
```

## 8. Codex App Server integration

H0는 stable App Server API를 사용한다.

- experimental API에 core correctness를 의존하지 않는다.
- tested baseline은 Codex 0.153.4.
- generated schema는 `vendor/codex-schema`에 둔다.
- 한 Task thread에서 automated turn start를 serialize한다.
- active turn에 background repair를 섞지 않는다.
- CI/PR/quota wait 중 모델을 polling 용도로 호출하지 않는다.

## 9. Git safety

- force push 기본 금지.
- base/protected branch 직접 push 기본 금지.
- task worktree/branch를 사용한다.
- commit/push/release 등의 side effect는 실제 상태를 reconcile한 뒤 수행한다.
- Cancel이 worktree/branch 삭제를 의미하지 않는다.

## 10. Security

- untrusted repository code 실행 금지.
- secret value를 Renderer event/Codex context에 자동 포함하지 않는다.
- credential을 UI state에 저장하지 않는다.
- shell command는 executable + args로 실행하고 명령 문자열 결합을 피한다.
- `contextIsolation=true`, Renderer `nodeIntegration=false` 유지.

## 11. UX rules

새 주요 화면/상태에는 필요한 경우 다음을 함께 구현한다.

```text
normal
empty
loading
error
blocked/recovery
```

상태를 색만으로 표현하지 않는다.

사용자가 항상 알 수 있어야 한다.

```text
지금 무엇을 하는가?
Codex가 active인가 sleeping인가?
왜 기다리는가?
내가 지금 할 일이 있는가?
완료까지 무엇이 남았는가?
```

## 12. STATUS update

한 WP를 실제로 완료한 뒤에만 `STATUS.md`를 갱신한다.

기록:

```text
Last completed WP
Next WP
Verification run/results
Current important SHA if applicable
Known blockers
```

검증하지 않은 일을 completed라고 기록하지 않는다.

## 13. Completion response

실제 구현 작업 완료 응답에는 최소 다음을 포함한다.

```text
completed WP(s)
what changed
verification actually run
pass/fail result
remaining next WP or blocker
```

설계 문서를 다시 길게 반복하기보다 실제 변경과 증거를 중심으로 보고한다.
