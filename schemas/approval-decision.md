# Approval Decision Record Contract v2

## 1. 목적

사람의 승인·조건부 승인·보류·제외·재검토 결정을 변경 불가능한 감사 레코드로 저장한다. 분석 결과와 승인 결과를 분리하며, 에이전트와 Worker Runtime은 승인 레코드를 직접 생성·수정할 수 없다.

정식 식별자와 enum은 `architecture/canonical-contract-registry.yaml`을 따른다.

## 2. 구조

```yaml
schema_version: approval-decision-v2
decision_id: APR-000001
case_id: CASE-001
decision_package_id: DPK-0001
approval_gate_id: AP-05

request:
  requested_at: 2026-07-22T10:00:00+09:00
  requested_by: approval-service
  macro_state: BID_CANDIDATE
  affected_stage_run_ids: []
  requested_transition:
    from: BID_CANDIDATE
    to: BID_READY
  package_hash: sha256:...

actor:
  actor_type: user|internal_reviewer|external_expert
  actor_id: USER-001
  display_name: string
  authority_scope: bid-plan-review

decision:
  value: APPROVED|APPROVED_WITH_CONDITIONS|REJECTED|HOLD|REQUEST_MORE_EVIDENCE|REQUEST_EXPERT_REVIEW|EXCLUDE
  gate_command: APPROVE_PLAN|null
  decided_at: 2026-07-22T10:10:00+09:00
  rationale: string
  acknowledged_warnings: []

conditions:
  - condition_id: COND-001
    statement: string
    owner: user
    due_before_gate: AP-06
    evidence_type: lender_confirmation
    status: OPEN|SATISFIED|FAILED|WAIVED
    on_failure: HOLD

scope:
  approved_analysis_versions: []
  approved_stage_run_ids: []
  approved_evidence_ids: []
  approved_policy_versions: []
  excluded_claim_ids: []

transition:
  transition_type: none|domain_gate|macro_state
  allowed: true
  from_state: BID_CANDIDATE|null
  to_state: BID_READY|null
  applied_at: null
  applied_by_service: state-machine

invalidation:
  events:
    - evidence_type: registry_document
      action: MARK_STALE
  stale_at: null
  stale_reason: null
  successor_decision_id: null

audit:
  previous_record_hash: sha256:...
  record_hash: sha256:...
  created_by_service: approval-service
  immutable: true
```

## 3. Generic Decision과 Gate Command 분리

저장되는 정식 결정값은 다음 7개뿐이다.

- `APPROVED`
- `APPROVED_WITH_CONDITIONS`
- `REJECTED`
- `HOLD`
- `REQUEST_MORE_EVIDENCE`
- `REQUEST_EXPERT_REVIEW`
- `EXCLUDE`

`LOCK_PLAN`, `APPROVE_PLAN`, `REQUEST_LENDER_CONFIRMATION` 같은 표현은 화면과 게이트별 명령이다. 감사 레코드에는 `gate_command`로 보존하되, 상태머신은 canonical generic decision으로 정규화한 뒤 처리한다.

예:

```text
AP-06 + LOCK_PLAN → APPROVED
AP-05 + APPROVE_PLAN_WITH_CONDITIONS → APPROVED_WITH_CONDITIONS
AP-04 + REQUEST_LENDER_CONFIRMATION → REQUEST_MORE_EVIDENCE
```

## 4. 전환 유형

### domain_gate

AP-02·AP-03처럼 특정 분석 도메인의 검토완료를 승인한다. 사건 상위상태를 직접 변경하지 않는다.

### macro_state

AP-01·AP-05·AP-06처럼 명시된 사건 상위상태 전환을 요청한다. State Machine이 전제조건을 다시 확인한 경우에만 적용한다.

### none

AP-07 변경통제처럼 무효화·재계산·재승인 체인을 시작하지만 직접 상태전환은 하지 않는다.

## 5. 핵심 규칙

1. `package_hash`가 현재 Decision Package와 일치해야 한다.
2. 승인자가 해당 게이트 권한을 가져야 한다.
3. 승인 레코드는 append-only다.
4. 정정은 새 레코드로 생성하고 이전 레코드를 `SUPERSEDED` 또는 `STALE`로 연결한다.
5. 조건부 승인의 필수조건이 충족되지 않으면 지정 게이트 이후로 진행할 수 없다.
6. 승인과 실제 입찰·송금·계약 실행 권한을 연결하지 않는다.
7. 고위험 승인에는 `rationale`이 필수다.
8. 외부전문가 필수검토가 없으면 상태전환을 차단한다.
9. AP-05와 AP-06은 서로 다른 Decision Package와 ApprovalDecision을 요구한다.
10. AP-07은 과거 승인을 재사용하거나 BID_READY/BID_LOCKED로 직접 전환할 수 없다.

## 6. 승인 유형별 최소 권한

| 게이트 | 최소 권한 |
|---|---|
| AP-01 자료 완성도 | case-reviewer |
| AP-02 권리·임차인 | user + 필요 시 legal-expert |
| AP-03 건축·현장 | user + 필요 시 building-expert |
| AP-04 금융·비용 | user + 필요 시 lender/tax review |
| AP-05 모의입찰 계획 | user |
| AP-06 모의입찰 잠금 | user + governance gate |
| AP-07 새 증거 변경통제 | 기존 승인자 또는 동등 권한자 |

## 7. 상태머신 전환 검사

```text
Decision exists
AND Generic decision permits requested effect
AND Package hash matches
AND Actor has authority
AND Required conditions satisfied
AND Required expert reviews exist
AND Required domain stage runs completed
AND No P0/P1 blocker
AND Decision is not stale
AND AP-07 bypass is not attempted
```

모두 참일 때만 domain gate 완료 또는 macro state 전환을 적용한다.

## 8. 조건 Waive

- P0 안전조건은 면제 불가
- 법률전문가 필수검토는 PM 또는 일반 사용자가 면제할 수 없음
- 미확인 대출·인수 보증금·특수권리 조건은 AP-06 전 면제 불가
- 면제는 별도 권한, 이유와 감사 레코드를 요구

## 9. 무효화

다음 변경은 관련 승인을 `STALE`로 만든다.

- 의존 증거의 새 버전
- 권리·임차인·금융·수익 분석의 새 버전
- 정책 만료 또는 새 정책 버전
- 승인 조건 실패
- 스냅샷 해시 불일치

AP-07은 영향범위를 계산하고 관련 AP-02~AP-06을 다시 요구한다. 과거 잠금 스냅샷은 수정하지 않고 historical로 보존한다.

## 10. 감사 검증

- 레코드 해시 체인 검증
- 승인자와 권한 검증
- package 및 subject version 검증
- domain gate와 상태전환 로그 대조
- stale 결정 사용 탐지
- AP-05/AP-06 동일 결정 재사용 탐지
- 에이전트 또는 Worker Runtime의 직접 수정 시도 탐지

## 11. 개인정보

승인자 식별정보는 감사에 필요한 최소 범위로 저장한다. 외부전문가 연락처·자격정보·검토원문은 별도 접근등급으로 관리한다.
