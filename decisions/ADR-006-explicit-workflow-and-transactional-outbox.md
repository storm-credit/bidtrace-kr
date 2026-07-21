# ADR-006: 명시적 Workflow Registry와 Transactional Outbox

- 상태: Proposed for Stage F approval
- 날짜: 2026-07-21

## Context

BidTrace KR의 사건은 수일에서 수개월 동안 진행될 수 있고, 새 증거·매각기일 변경·사람 승인·외부전문가 검토에 따라 중단과 재개가 반복된다.

일반적인 대화형 Agent loop만으로 이를 제어하면 다음 문제가 발생한다.

- 실행순서가 프롬프트마다 변함
- 어떤 단계가 완료됐는지 재현하기 어려움
- 장애 후 재시작 시 중복 실행
- 사람 승인 전후 경계가 불명확
- 새 증거가 미치는 영향 범위를 계산하기 어려움

전용 분산 Workflow Engine은 강력하지만 Manual MVP에는 운영비용이 과도하다.

## Decision

### Workflow Registry

Workflow는 YAML 기반 선언형 정의와 DB에 저장된 실행계획으로 관리한다.

정의 내용:

- base workflow
- risk overlay
- stage DAG
- executor/reviewer
- required evidence
- deterministic checks
- completion/block/escalation gate
- invalidation dependencies
- human approval gate

실행 전 정적검증을 통과해야 한다.

### State Machine

사건 상태전환은 Core의 결정론 상태머신만 수행한다.

- transition command
- current state/version 확인
- guard 검사
- 상태 변경
- domain event와 outbox를 같은 트랜잭션에 기록

### Transactional Outbox

Core가 Worker 작업이나 후속 처리를 요청할 때 DB 트랜잭션 안에서 outbox event를 기록한다.

Dispatcher가 outbox를 읽어 task queue에 등록한다.

필수 속성:

- event_id
- aggregate_id
- aggregate_version
- event_type
- payload_schema_version
- idempotency_key
- occurred_at
- publish_status
- retry_count

### DB-backed Task Queue

Manual MVP는 PostgreSQL task table과 lease 방식으로 시작한다.

- `READY → LEASED → RUNNING → SUCCEEDED|FAILED|DEAD_LETTER`
- lease timeout 후 재획득 가능
- 결과 제출은 task_id와 attempt_id 검증
- 같은 idempotency_key 중복 성공 금지

## Consequences

장점:

- 실행 재현성
- 사람 승인 경계 명확화
- 부분 장애 복구
- 새 증거에 따른 선택적 무효화
- 전용 Queue 없이 시작 가능

비용:

- 상태와 task 처리 코드 필요
- lease·retry·dead-letter 운영 필요
- long-running task heartbeat 필요
- Workflow 정의 버전관리 필요

## Rejected Alternatives

### LLM이 다음 단계를 자유롭게 선택

재현성과 안전경계를 보장할 수 없어 거부한다.

### 처음부터 Temporal 등 전용 Workflow Engine 도입

장기적으로 가능한 선택이지만 Manual MVP의 규모와 팀 운영능력에 비해 과도하다.

### 애플리케이션 메모리 Queue

재시작 시 작업 손실과 수평확장 문제로 거부한다.

### 상태 변경 후 별도 메시지 발행

DB commit과 메시지 발행 사이의 유실 가능성 때문에 거부한다.

## Validation

- DAG 순환 참조 거부
- 도달 불가능 단계 거부
- 사람 승인 우회 경로 거부
- 같은 task 중복 결과 제출 검증
- dispatcher 장애 후 outbox 재처리
- lease timeout 후 안전한 재획득
- 새 증거 유입 시 영향받는 분석만 stale 처리
- locked snapshot은 stale 처리 없이 historical 보존

## Revisit Triggers

- 장기 task 수가 DB queue 운영한계를 지속 초과
- 외부 callback과 보상흐름이 크게 증가
- 수백 단계 Workflow와 고빈도 스케줄이 필요
- 운영복구 복잡도가 전용 Engine 도입비용보다 커짐
