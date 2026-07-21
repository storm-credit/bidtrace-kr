# BidTrace KR Workflow Registry 설계

## 1. 목적

Workflow Registry는 PM Orchestrator가 사건 유형과 위험 신호에 따라 어떤 작업을 어떤 순서로 실행할지 결정하는 선언형 설계 원장이다.

이 문서는 애플리케이션 구현 코드가 아니다. 구현 전 다음을 고정하기 위한 기획·설계 계약이다.

- 사건 유형별 필수 단계
- 단계별 입력과 출력
- 호출할 전문 에이전트 또는 결정론 모듈
- 병렬 실행 가능 범위
- 완료 게이트
- 중단·보류·전문가 승격 조건
- 새 자료 유입 시 무효화 범위
- 사람 승인 위치

Workflow Registry는 PM의 판단을 없애지 않는다. PM이 임의로 작업 순서를 바꾸거나 필수 검증을 생략하지 못하도록 기본 경로와 최소 게이트를 제공한다.

---

## 2. 설계 원칙

### 2.1 사건 유형과 위험 신호를 분리한다

사건 유형은 기본 워크플로를 선택한다.

- apartment_standard
- villa_multiunit
- multifamily_dagagu
- land_or_building
- share_auction
- special_rights

위험 신호는 기본 워크플로에 추가 작업을 삽입한다.

예:

- suspected_senior_tenant
- multiple_tenants
- illegal_extension_signal
- land_building_owner_mismatch
- lien_claim
- statutory_superficies_signal
- unregistered_land_share
- redevelopment_signal
- missing_or_stale_document

즉, `다가구`와 `선순위 임차인 의심`을 같은 분류축으로 취급하지 않는다.

### 2.2 모든 단계는 계약을 가진다

각 단계는 다음을 반드시 정의한다.

- stage_id
- objective
- required_inputs
- optional_inputs
- executor
- reviewer
- deterministic_checks
- allowed_tools
- completion_gate
- block_conditions
- escalation_conditions
- invalidated_by
- next_stages

### 2.3 완료와 안전을 분리한다

작업 산출물이 생성됐다고 단계가 완료되는 것은 아니다.

예:

- 권리분석 보고서가 존재함: 산출물 생성
- 모든 핵심 주장에 evidence_id가 연결됨: 근거 게이트
- 독립 반대검토가 완료됨: 검토 게이트
- 인수 가능 위험이 미확인으로 남지 않음: 안전 게이트
- 사람이 승인함: 승인 게이트

### 2.4 조건부 에이전트를 기본 실행하지 않는다

비용과 복잡도를 줄이기 위해 다음은 신호가 있을 때만 호출한다.

- Special Rights Detector
- Legal Safety Reviewer
- Occupancy & Eviction Analyst
- Urban Planning & Redevelopment Analyst
- Renovation Cost Analyst
- Rental Operations Analyst
- Evidence Judge

### 2.5 결정론 작업을 LLM에게 맡기지 않는다

다음은 규칙·상태·계산 모듈이 수행한다.

- 문서 해시와 버전 비교
- 날짜·접수순위 정렬
- 필수자료 존재 검사
- 상태 전환 가능 여부
- 비용 항목 완성도 검사
- 수익성 산술
- 잠금 스냅샷 무결성 검사
- 정책 만료와 기준일 검사

---

## 3. 공통 상위 워크플로

모든 사건은 다음 상위 단계를 공유한다.

```text
W00 CASE_INTAKE
  → W10 EVIDENCE_PREPARATION
  → W20 EARLY_SCREENING
  → W30 DOMAIN_ANALYSIS
  → W40 CROSS_VALIDATION
  → W50 INVESTMENT_SCENARIOS
  → W60 HUMAN_DECISION
  → W70 BID_PLAN_LOCK
  → W80 RESULT_TRACKING
  → W90 POSTMORTEM
```

### W00 CASE_INTAKE

목적:

- 사건 식별
- 기본 유형 분류
- 매각기일·상태 기록
- 분석 목표와 사용자 전략 확인

완료 게이트:

- case_id 존재
- 법원·사건번호 또는 동등 식별자 존재
- property_type 후보 존재
- 사용 목적이 실거주·임대·매도·복합 중 하나로 기록

### W10 EVIDENCE_PREPARATION

목적:

- 필수 문서 목록 생성
- 원본 등록과 해시
- 구조화 추출
- 문서 상충과 최신성 검사

완료 게이트:

- 물건 유형별 최소 문서 충족
- 원본과 추출물 연결
- 치명적 필드의 출처 위치 존재

### W20 EARLY_SCREENING

목적:

- 명백한 제외 사유 조기 탐지
- 불필요한 고비용 분석 방지
- 위험 신호 생성

대표 제외·보류 신호:

- 물건 동일성 불명확
- 핵심 원문 미확보
- 권리 또는 점유관계가 자동화 범위를 크게 초과
- 사용자의 자금·목표와 물건 규모가 명백히 불일치

W20은 법률적 안전을 확정하지 않는다. 상세 분석 필요성을 분류한다.

### W30 DOMAIN_ANALYSIS

사건 유형별 워크플로가 실행되는 핵심 구간이다.

병렬 가능 예:

- 시장분석
- 건축·토지분석
- 입지분석
- 금융 사전조건

순차 필요 예:

- 문서추출 → 권리 타임라인 → 권리해석 → 반대검토
- 임차인 구조화 → 임차인별 분석 → 배당 시나리오

### W40 CROSS_VALIDATION

목적:

- 에이전트 간 모순 탐지
- 증거 누락 확인
- 독립 모델 반대검토
- 규칙·계산 재검증

Evidence Judge 호출 조건:

- 권리분석과 임차인분석 충돌
- 감정평가서와 건축물대장 충돌
- 시장분석과 현장자료가 처분가에 중대한 차이를 유발
- 작성자와 반대검토자의 위험등급이 2단계 이상 차이

### W50 INVESTMENT_SCENARIOS

목적:

- 보수·기준·상단 시나리오 작성
- 총투자비와 자금소요 계산
- 손익분기점과 민감도 계산
- 모의입찰 계획 초안 작성

완료 게이트:

- 계산 입력값과 출처 존재
- 미확인 값이 별도 표시
- 비용 카탈로그 누락 검사 통과
- 투자 레드팀 완료

### W60 HUMAN_DECISION

사람에게 제공할 최소 패키지:

- 확인된 사실
- 핵심 위험
- 반대해석
- 미확인 자료
- 추가 확인 행동
- 세 가지 시나리오
- 입찰 계획 초안과 포기 조건
- 전문가 확인 필요항목

선택:

- APPROVE
- APPROVE_WITH_CONDITIONS
- REQUEST_MORE_EVIDENCE
- REQUEST_EXPERT_REVIEW
- HOLD
- EXCLUDE

### W70 BID_PLAN_LOCK

잠금 조건:

- 사람 승인 존재
- P0/P1 미해결 없음
- 필수 정책 버전 유효
- 입력·출력·모델·프롬프트·증거 버전 기록

잠금 후 수정은 새 버전으로만 생성한다.

### W80 RESULT_TRACKING

- 실제 낙찰가
- 입찰자 수
- 유찰·변경·취하
- 매각허가·잔금·재매각 상태

### W90 POSTMORTEM

- 예상과 실제 차이
- 권리·시장·비용·기간 오차
- 놓친 위험과 과대평가한 위험
- 정책·프롬프트·fixture 개선 후보

---

## 4. 위험 신호 라우팅

| 위험 신호 | 추가 호출 | 강제 게이트 |
|---|---|---|
| suspected_senior_tenant | Tenant Analyst, Rights Counter, Legal Safety | 사람/전문가 승인 |
| multiple_tenants | Tenant Map-Reduce, Distribution Rule Check | 임차인 전원 처리 |
| lien_claim | Special Rights Detector, Legal Safety | 자동 BID_READY 금지 |
| statutory_superficies_signal | Land/Building Analyst, Special Rights, Legal Safety | 외부 전문가 검토 |
| share_auction | Share Workflow, Liquidity & Exit | 권리·점유·환금성 승인 |
| illegal_extension_signal | Building Analyst, Field Agent, Cost Analyst | 현장·원상복구 조건 |
| redevelopment_signal | Urban Planning Analyst | 공식 단계·불확실성 표시 |
| stale_document | Evidence Curator | 최신자료 확보 전 잠금 금지 |
| model_conflict | Evidence Judge | 충돌 해결 전 상태전환 금지 |
| permission_violation | Governance Reviewer | 즉시 실행 중지 |

---

## 5. 결과 무효화 규칙

새 자료가 들어오면 모든 분석을 다시 돌리지 않는다. 영향 그래프에 따라 무효화한다.

| 새 자료 | 기본 무효화 범위 |
|---|---|
| 최신 등기 | 권리 타임라인, 권리분석, 입찰계획 |
| 수정 매각물건명세서 | 임차인, 권리, 명도, 입찰계획 |
| 새 현황조사서 | 점유, 임차인, 명도, 비용 |
| 새 건축물대장 | 건축, 수리, 시세조정, 입찰계획 |
| 새 실거래 자료 | 시장 시나리오, 수익성, 입찰계획 |
| 대출 실제 확인 | 금융, 수익성, 입찰계획 |
| 세금 정책 변경 | 세금, 수익성, 잠금 유효성 |
| 실제 현장사진 | 건축·현장, 수리, 명도, 시장조정 |

잠긴 계획은 삭제하지 않고 `superseded`로 표시한 후 새 버전을 만든다.

---

## 6. 워크플로 선택 정책

PM은 다음 순서로 Workflow Registry를 선택한다.

```text
1. 기본 property_type 분류
2. 소유 구조 분류
3. 점유·임차인 복잡도 분류
4. 특수권리 신호 검사
5. 개발·건축 신호 검사
6. 사용자 전략 반영
7. 기본 워크플로 + 위험 오버레이 합성
8. 비용·시간 예산 확인
9. 사람에게 실행계획 표시
```

분류가 불명확하면 가장 안전한 상위 워크플로를 선택하고, 분류 확인 작업을 먼저 실행한다.

---

## 7. Registry 버전 관리

각 Workflow 정의는 다음 버전을 가진다.

```yaml
workflow_id: WF-APARTMENT-STANDARD
workflow_version: 1.0.0
policy_bundle_version: 2026-07-draft
agent_contract_version: 1.0.0
schema_version: 1.0.0
status: design_draft
```

상태:

- design_draft
- fixture_ready
- harness_tested
- expert_calibrated
- operational_candidate
- deprecated

현재 작성되는 Workflow는 모두 `design_draft`다. 실제 사건 fixture와 실행 하네스를 통과하기 전 운영 워크플로로 표시하지 않는다.

---

## 8. 다음 산출물

- `schemas/workflow-definition.md`
- `workflows/apartment-standard.yaml`
- `workflows/villa-multiunit.yaml`
- `workflows/multifamily-dagagu.yaml`
- `workflows/land-building.yaml`
- `workflows/share-auction.yaml`
- `workflows/special-rights-overlay.yaml`
- `evaluations/harness/workflow-coverage-matrix.yaml`
