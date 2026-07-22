# BidTrace KR Design Freeze Audit

## 1. 결론

Stage A~F의 문서 설계는 **사람의 Design Freeze 승인 요청이 가능한 상태**까지 수렴했다.

```yaml
document_design: ready_for_human_approval
implementation: not_started
G0_start: no_go_until_human_approval
G1_start: no_go_until_static_validators_pass
runtime_validation: not_executed
operational_candidate: no_go
production: no_go
```

이 결론은 시스템이 운영 가능하다는 뜻이 아니다. 문서 간 P0/P1 설계모순을 해소하고, 남은 차이를 결정론적 migration과 validator 대상으로 고정했다는 뜻이다.

## 2. 검수 범위

- Foundation 생애주기와 PM 하네스
- Rights·Tenant·Special Rights
- Market·Cost·Profitability·Bid Plan
- Agent Operating Model
- Harness Deep Audit
- Workflow Registry 5종과 Special Rights Overlay
- CASE-001~012 Evaluation Program
- Human Approval UX AP-01~AP-07
- Final Architecture, Data, Security, Deployment, SRE·DR
- Draft PR 의존성과 병합 충돌

## 3. 발견한 주요 모순

### 3.1 사건 상태와 병렬 분석

기존 문서는 `RIGHTS_REVIEW`, `MARKET_REVIEW`, `BUILDING_REVIEW`, `FIELD_REVIEW`를 병렬로 설명하면서 단일 `current_state` enum으로 저장하려 했다.

조치:

- Macro State와 WorkflowStageRun 분리
- `ANALYSIS_IN_PROGRESS` 안에서 domain Stage 병렬 실행
- ADR-007, Lifecycle v2, Domain Model Amendment 반영

### 3.2 승인 게이트 불일치

- `HG-BID-PLAN`과 `AP-01~AP-07` 혼용
- Gate command와 generic decision 혼용
- AP-05 계획승인과 AP-06 잠금승인 축약
- AP-07의 직접 BID_READY 전환 가능성

조치:

- AP ID를 canonical gate로 고정
- generic decision 7종 고정
- AP-05와 AP-06 별도 Decision Package 강제
- AP-07을 change-control coordinator로 변경

### 3.3 평가 fixture 충돌

#7과 #15가 동일한 CASE-001/003 manifest 경로를 만들고, #8은 canonical catalog 밖의 CASE-020을 만들었다.

조치:

- #7의 초기 사례를 `evaluations/domain-seeds/rights/`로 이동
- #8의 초기 사례를 `evaluations/domain-seeds/investment/`로 이동
- #15의 CASE-001~012만 canonical evaluation manifest
- CASE-020 내용은 CASE-011 migration 입력으로 보존

### 3.4 Agent·Service ID

Workflow와 fixture에서 alias와 deterministic service가 혼재했다.

조치:

- canonical Agent/Service Registry 추가
- `finance-cost-analyst` 분리
- `profitability-engine`은 service로만 분류
- `post-auction-tracker`와 핵심 service acceptance amendment 추가

### 3.5 Workflow ID·Status·Action

- WF canonical ID와 slug 혼용
- Workflow status enum 불일치
- `APPROVE_WITH_CONDITIONS_ONLY`가 block action으로 사용
- 미정의 risk signal이 Overlay처럼 표현됨

조치:

- canonical contract registry 추가
- Workflow Definition Contract v2
- Workflow v1→v2 migration spec
- 정의된 Overlay는 `WF-SPECIAL-RIGHTS-OVERLAY`만 인정

## 4. Design Freeze Source of Truth

우선순위:

1. `architecture/canonical-contract-registry.yaml`
2. `architecture/system-manifest.yaml` v2
3. ADR-007, ADR-008
4. `docs/01-auction-lifecycle.md` v2
5. `schemas/workflow-definition.md` v2
6. `workflows/approval-gates.yaml` v2
7. `schemas/approval-decision.md` v2
8. `docs/37-domain-model-freeze-amendment.md`
9. migration specs
10. 이전 v1 설계 문서

이전 문서와 충돌하면 상위 순위 계약을 따른다.

## 5. Migration 정책

현재 5개 Base Workflow와 12개 Evaluation Manifest는 설계 초안 v1이다. 직접 운영하지 않으며 다음 규격으로 v2를 생성하거나 검증한다.

- `migrations/design/workflow-v1-to-v2.yaml`
- `migrations/design/evaluation-manifest-v1-to-v2.yaml`

G0 Validator는 migration 결과가 canonical registry와 일치하지 않으면 실패해야 한다.

## 6. 문서 Design Freeze 승인조건

다음은 충족됐다.

- P0 설계모순의 해결방향과 불변식 확정
- 사건 상태와 병렬 Stage 모델 확정
- 승인 게이트·결정·잠금·재승인 의미 확정
- canonical Workflow·Agent·Service·Gate ID 확정
- 평가 CASE 경로 충돌 제거
- Core/Worker/MCP 경계 확정
- 실제 입찰·송금·계약·법원제출 기능 제외
- G0~G8 구현 순서와 운영 승격 차단조건 확정

사람이 승인해야 하는 사항:

- Macro State + StageRun 모델 수용
- AP-05/AP-06 별도 승인 수용
- AP-07 재승인 체인 수용
- canonical registry와 migration 방식을 Design Freeze 기준으로 수용
- 첫 구현 범위를 G0 Validator·CI까지만 허용할지 결정

## 7. 구현 시작과 운영 승격을 구분한다

### G0 시작

사람의 Design Freeze 승인 후 가능하다.

범위:

- 저장소 skeleton
- schema와 registry loader
- v1→v2 migration validator
- DAG·ID·enum·approval path static validator
- CI baseline

### G1 시작

G0 validator가 모든 문서 계약을 통과한 후 가능하다.

### Worker/LLM 연결

Core·Evidence·Workflow·Approval·Lock의 수동 경로가 완성된 뒤 G5에서 시작한다.

### 운영 후보

실제 원문 fixture, 다른 공급자 모델 검토, MCP sandbox 공격시험과 외부 법률·세무·금융 보정 후에만 가능하다.

## 8. 최종 위험

- 법률·세금·대출 정책 Registry는 아직 실제 데이터가 아님
- 실제 경매 원문 fixture가 없음
- OCR·Vision 정확도 미측정
- 사용자 승인 UX 미검증
- 보존·삭제·Cloud IAM 구체화 미완료
- 백업·복구 훈련 미실행

따라서 Design Freeze 승인 이후에도 `production_status: no_go`를 유지한다.
