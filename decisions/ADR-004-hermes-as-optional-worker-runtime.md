# ADR-004: Hermes Agent를 선택형 작업자 런타임으로 채택

- 상태: Proposed
- 결정일: 2026-07-21
- 결정 주체: PM Orchestrator / Human Owner

## Context

BidTrace KR은 다수의 전문 에이전트, MCP 도구, 반복 작업, 이벤트 훅과 장기 추적이 필요하다. Hermes Agent는 MCP 연결, 프로필, 스킬, 훅, 웹훅과 서브에이전트 실행을 제공하므로 작업자 런타임 후보로 적합하다.

그러나 경매 분석은 원본 증거 보존, 상태 전환 강제, 법률 위험, 모의입찰 잠금과 사람 승인이 필요하다. 이 기능을 범용 에이전트 런타임의 메모리나 프롬프트에만 맡길 수 없다.

## Decision

Hermes Agent를 **선택 가능한 Worker Runtime**으로 채택한다.

Hermes는 다음을 담당할 수 있다.

- 전문 에이전트 프로필 실행
- MCP 도구 연결
- 연구·문서화·검토 작업
- 이벤트 관찰과 메트릭 수집
- 예약·웹훅 기반 추적 작업 후보 생성

Hermes는 다음을 담당하지 않는다.

- BidTrace PM Orchestrator 자체
- 사건 상태의 원본 저장소
- 원본 증거 저장소
- 법률·세무 최종판정
- 모의입찰 잠금의 원장
- 사람 승인 대체
- 실제 입찰 또는 금전 집행

## Architecture Boundary

```text
BidTrace Core
  ├─ PM Orchestrator
  ├─ State Machine
  ├─ Evidence Registry
  ├─ Policy Engine
  ├─ Human Approval
  └─ Evaluation Harness
         ↓ adapter
Hermes Worker Runtime
  ├─ profiles
  ├─ skills
  ├─ MCP clients
  ├─ hooks
  └─ subagents
```

BidTrace Core는 `HermesAdapter` 계약을 통해 Hermes를 호출한다. 향후 다른 런타임으로 교체할 수 있어야 한다.

## Profiles

초기 프로필:

- `bidtrace-research`: 공개자료 조사, 쓰기 제한
- `bidtrace-extraction`: 문서 추출, 사건 원본 읽기 전용
- `bidtrace-rights`: 권리분석, 제한된 증거·규칙 도구
- `bidtrace-market`: 시세·지역 분석
- `bidtrace-review`: 독립 반대검토, 사건 결과 수정 금지
- `bidtrace-git`: 지정 브랜치 문서 작성, main 직접 쓰기 금지
- `bidtrace-tracker`: 사건 상태 후보 수집, 자동 확정 금지

Hermes 프로필은 상태·키·메모리를 분리하지만 자체적으로 파일시스템 샌드박스를 보장하지 않는다. 따라서 프로필 분리만으로 보안 경계를 충족했다고 간주하지 않는다.

## Skills Policy

- 운영 스킬 자동 생성·수정 금지
- 새 스킬은 `staging` 영역에서만 생성
- 스킬 변경은 Git diff와 평가 결과가 있어야 함
- 권리·세금·입찰 정책은 스킬 메모리가 아니라 버전 관리된 정책 파일에 저장
- 스킬은 규칙 엔진을 우회할 수 없음

## Hooks Policy

Hermes 훅은 관찰과 제한 보조에 사용한다.

허용:

- 호출·비용·도구·증거 ID 로깅
- 위험 도구 호출 차단을 위한 `pre_tool_call`
- 작업 종료 전 검증 계속 요청
- 경고·알림·리뷰 요청

금지:

- 훅만으로 모의입찰 잠금 보장
- 훅만으로 사건 상태 전환 보장
- 훅 실패 시에도 안전하다고 가정

훅 자체 오류는 에이전트 실행을 중단시키지 않을 수 있으므로 핵심 통제는 BidTrace Core와 GitHub 보호규칙에서 강제한다.

## MCP Policy

- 서버별 최소 도구만 노출
- 삭제·강제푸시·원본수정 도구는 기본 비노출
- 읽기 도구와 쓰기 도구를 분리
- 제한자료는 로컬 또는 승인된 서버만 사용
- MCP 결과도 증거가 아니라 수집 후보로 취급하고 검증

## Consequences

장점:

- MCP·프로필·도구 생태계를 빠르게 활용
- 작업자 런타임을 재사용
- 에이전트 역할별 환경과 키 분리

단점:

- BidTrace Core와 Hermes Adapter를 별도로 설계해야 함
- Hermes 업데이트에 대한 호환성 평가 필요
- 프로필이 보안 샌드박스가 아니므로 추가 통제 필요

## Acceptance Criteria

Hermes 통합은 다음이 검증된 후 활성화한다.

1. 지정 폴더 외 쓰기 차단 테스트
2. `main` 직접 push 차단 테스트
3. 원본 증거 수정 차단 테스트
4. 권리분석 스킬 자동변경 차단 테스트
5. 도구 허용목록 테스트
6. 훅 실패 시 Core 정책 유지 테스트
7. 동일 사건 재실행 재현성 테스트

## References

- https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp
- https://hermes-agent.nousresearch.com/docs/user-guide/profiles
- https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks
- https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks
