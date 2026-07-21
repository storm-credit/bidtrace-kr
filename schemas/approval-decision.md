# Approval Decision Record Contract

## 1. 목적

사람의 승인·조건부 승인·보류·제외·재승인 결정을 변경 불가능한 감사 레코드로 저장한다. 이 계약은 분석 결과와 승인 결과를 분리하며, 에이전트는 승인 레코드를 직접 생성·수정할 수 없다.

## 2. 구조

```yaml
schema_version: approval-decision-v1
decision_id: APR-000001
case_id: CASE-001
decision_package_id: DPK-0001
approval_type: AP-05

request:
  requested_at: 2026-07-21T10:00:00+09:00
  requested_by: pm-orchestrator
  current_state: BID_CANDIDATE
  requested_transition: BID_READY
  package_hash: sha256:...

actor:
  actor_type: user|internal_reviewer|external_expert
  actor_id: USER-001
  display_name: string
  authority_scope: bid-plan-review

decision:
  value: APPROVED|APPROVED_WITH_CONDITIONS|REJECTED|HOLD|REQUEST_MORE_EVIDENCE|REQUEST_EXPERT_REVIEW|EXCLUDE
  decided_at: 2026-07-21T10:10:00+09:00
  rationale: string
  acknowledged_warnings: []

conditions:
  - condition_id: COND-001
    statement: string
    owner: user
    due_before_state: BID_LOCKED
    evidence_type: lender_confirmation
    status: OPEN|SATISFIED|FAILED|WAIVED
    on_failure: HOLD

scope:
  approved_analysis_versions: []
  approved_evidence_ids: []
  approved_policy_versions: []
  excluded_claim_ids: []

transition:
  allowed: true
  from_state: BID_CANDIDATE
  to_state: BID_READY
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

## 3. 핵심 규칙

1. `package_hash`가 현재 Decision Package와 일치해야 한다.
2. 승인자가 해당 승인 유형에 대한 권한을 가져야 한다.
3. 승인 레코드는 append-only다.
4. 수정이 필요하면 새 결정 레코드를 만들고 이전 레코드를 `SUPERSEDED` 또는 `STALE`로 연결한다.
5. 조건부 승인에서 필수조건이 충족되지 않으면 지정 상태 이후로 전환할 수 없다.
6. 승인과 실제 입찰·송금·계약 실행 권한을 연결하지 않는다.
7. 고위험 승인에는 `rationale`이 필수다.
8. 외부전문가 검토가 필요한 경우 해당 검토 레코드가 없으면 승인 전환을 차단한다.

## 4. 승인 유형별 권한

| 승인 유형 | 최소 권한 |
|---|---|
| AP-01 자료 완성도 | case-reviewer |
| AP-02 권리·임차인 | user + 필요 시 legal-expert |
| AP-03 건축·현장 | user + 필요 시 building-expert |
| AP-04 금융·비용 | user + 필요 시 lender/tax review |
| AP-05 모의입찰 계획 | user |
| AP-06 모의입찰 잠금 | user + governance gate |
| AP-07 새 증거 재승인 | 기존 승인자 또는 동등 권한자 |

## 5. 전환 검사

상태머신은 승인 결과를 그대로 신뢰하지 않고 다음을 검사한다.

```text
Decision exists
AND Package hash matches
AND Actor has authority
AND Required conditions satisfied
AND Required expert reviews exist
AND No P0/P1 blocker
AND Decision is not stale
```

모두 참일 때만 상태 전환을 적용한다.

## 6. 조건 Waive

조건을 면제하려면 별도 권한과 이유가 필요하다.

- P0 안전조건은 면제 불가
- 법률전문가 필수검토는 PM 또는 일반 사용자가 면제할 수 없음
- 미확인 대출·인수 보증금·특수권리 조건은 잠금 전 면제 불가

## 7. 무효화

다음 변경은 관련 승인을 stale 처리한다.

- 의존 증거의 새 버전
- 권리·임차인·금융·수익 분석의 새 버전
- 정책 만료 또는 새 정책 버전
- 승인 조건 실패
- 스냅샷 해시 불일치

무관한 메타데이터 변경은 승인을 무효화하지 않는다.

## 8. 감사 검증

- 레코드 해시 체인 검증
- 승인자와 권한 검증
- 패키지 버전 검증
- 상태전환 로그와 대조
- stale 결정 사용 탐지
- 에이전트 또는 Worker Runtime의 직접 수정 시도 탐지

## 9. 개인정보

승인자 식별정보는 감사에 필요한 최소 범위로 저장한다. 외부전문가 연락처·자격정보·검토원문은 별도 접근등급으로 관리한다.
