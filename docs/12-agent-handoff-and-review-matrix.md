# 에이전트 전달·검토·승격 매트릭스

## 1. 목적

전문 에이전트가 많아질수록 가장 큰 위험은 전문성 부족이 아니라 다음과 같은 운영 실패다.

- 같은 자료를 서로 다른 형식으로 해석
- 선행 결과가 완성되지 않았는데 후속 에이전트 실행
- 작성자와 검토자가 사실상 같은 맥락과 결론을 공유
- 미확인 값이 다음 단계에서 확정값으로 변환
- 결과 충돌을 PM이 임의로 평균 처리
- 최신 자료가 들어와도 기존 결론이 그대로 유지

이 문서는 사건 단계별 책임자, 입력, 산출물, 검토자, 전환조건과 무효화 조건을 정의한다.

---

## 2. 공통 전달 원칙

모든 에이전트 전달은 자유형 대화가 아니라 `Agent Task Envelope`를 사용한다.

```yaml
task_id: TASK-20260721-0001
case_id: CASE-SEOUL-2026-001
parent_task_id: null
stage: RIGHTS_REVIEW
agent_role: rights_analyst
objective: 말소·인수 위험 후보와 추가 확인항목 도출
risk_tier: high
input_contract_version: agent-task-v1
output_contract_version: analysis-result-v1
prompt_version: rights-analyst-v1.1
model_profile: R3_PRIMARY
allowed_evidence_ids:
  - EVD-0001
  - EVD-0002
  - EVD-0003
allowed_tools:
  - evidence_read
  - rights_rule_read
  - policy_read
prohibited_tools:
  - source_write
  - git_write
  - bid_execute
preconditions:
  - registry_extraction_validated
  - sale_document_extraction_validated
  - evidence_freshness_checked
expected_outputs:
  - rights_timeline_review
  - extinction_candidate
  - assumable_right_candidates
  - unknowns
  - requested_evidence
time_budget_seconds: 300
cost_budget_class: high
retry_budget: 1
human_escalation_rules:
  - senior_tenant_suspected
  - special_right_detected
  - unresolved_date_conflict
```

### 필수 원칙

- 허용되지 않은 증거는 참조하지 않는다.
- 이전 에이전트의 결론문 전체보다 구조화된 사실과 증거를 우선 전달한다.
- 검토자에게 작성자의 신뢰도 점수는 숨길 수 있다.
- `unknown`은 다음 단계에서 자동으로 `false` 또는 `0`이 되지 않는다.
- 모든 후속 작업은 입력 버전을 기록한다.

---

## 3. 전체 단계 매트릭스

| 단계 | 주 책임 | 주요 입력 | 병렬 작업 | 필수 검토 | 완료 게이트 |
|---|---|---|---|---|---|
| S0 사건 접수 | PM / Case Manager | 사건번호·사용자 목적 | 사건상태, 자료목록 | PM | case_id와 범위 확정 |
| S1 증거 등록 | Evidence Curator | 원본 문서 | 해시, 품질검사, 민감도 | Evidence Auditor | 필수자료 목록과 누락 확정 |
| S2 사실 추출 | Extractor Pod | 검증 가능한 원본 | 등기·매각·대장·시장 추출 | Evidence Auditor | 위치 연결과 스키마 통과 |
| S3 기본 필터 | PM / Screening Rules | 구조화 사실 | 물건유형·자동제외 신호 | Governance | 분석 지속 여부 결정 |
| S4 권리·절차 | Rights Pod | 등기·임차·절차 사실 | 권리, 임차, 특수권리 | Counter + Legal Safety | 인수위험과 unknown 정리 |
| S5 물건·현장 | Property Pod | 대장·감정·사진 | 건축, 점유, 정비, 입지 | Counter / Legal Safety | 현장·건축 위험 정리 |
| S6 시장·유동성 | Market Pod | 정규화 사례 | 시장, 출구, 임대 | Market Counter | 가격범위와 표본한계 정리 |
| S7 자금·비용·수익 | Finance Pod | 시세·비용·사용자 조건 | 금융, 세금, 수리, 운영 | Investment Red Team | 재현 가능한 시나리오 |
| S8 모의입찰안 | Decision Pod | 모든 검증 결과 | 입찰안, 스트레스안 | Evidence Judge + Red Team | 사람 승인패키지 완성 |
| S9 잠금 | PM / Human | 승인된 계획 | 해시·버전·조건 | Governance | immutable snapshot |
| S10 결과추적 | Tracker Pod | 법원결과·공개정보 | 결과, 허가, 잔금 | PM | 실제 결과 구조화 |
| S11 사후복기 | Evaluation Pod | 잠금안·실제결과 | 오차, 원인, 개선 | Governance | 변경제안과 평가사례 |

---

## 4. 단계별 상세 전달

## S0. 사건 접수

### 입력

- 사건번호 또는 물건 식별정보
- 사용자 목적: 실거주, 임대, 매도, 학습용 모의입찰
- 투자 가능한 자기자본 범위
- 관심 지역·물건 유형
- 분석 기준일

### PM 작업

1. 사건의 범위를 하나의 `case_id`로 고정한다.
2. 사용자 목적과 실제 행동 범위를 구분한다.
3. 법률·금융 고위험 사건인지 초기 신호를 확인한다.
4. 자료 목록과 수집 우선순위를 생성한다.

### Gate

- 사건 식별 가능
- 사용자 목적 기록
- 실제 자동입찰 범위 제외 확인

---

## S1. 증거 등록

### Evidence Curator → Document Quality Inspector

전달:

- 원본 파일 참조
- 문서 유형 후보
- 수집일과 사용자 제공 설명

반환:

- 페이지 수
- 읽기 가능성
- 누락·회전·잘림
- 표·도장·이미지 페이지
- 재업로드 요청

### Document Quality Inspector → Evidence Curator

`quality_pass=false`이면 Extractor 호출을 차단할 수 있다.

### Evidence Curator → Evidence Auditor

- 문서 해시
- 메타데이터
- 중복 후보
- 기준일
- 민감도

### Gate

- 원본과 파생파일 분리
- 해시 생성
- 기준일 명시
- 필수문서 누락 상태 표시

---

## S2. 사실 추출

### 병렬 실행군

```text
Registry Extractor
Sale Document Extractor
Building Document Extractor
Market Data Normalizer
```

### 병렬 허용 조건

- 서로 다른 원본을 사용
- 동일 필드의 최종 소유자가 명확
- 공통 주소·사건번호는 Case Manager 값 사용

### 교차검사 필드

- 주소
- 면적
- 물건 유형
- 점유자
- 임차 보증금
- 감정가
- 대지권
- 위반건축물 표시

### 충돌 처리

문서 A와 B의 값이 다르면 Extractor가 하나를 선택하지 않는다.

```yaml
conflict:
  field: occupied_by
  values:
    - value: 소유자
      evidence_id: EVD-0002
    - value: 임차인 김OO
      evidence_id: EVD-0003
  resolution_status: unresolved
```

### Gate

- 스키마 통과
- 핵심 필드에 증거 위치
- 충돌 목록 생성
- Evidence Auditor 표본검사 통과

---

## S3. 기본 필터

### 목적

고비용 분석 전에 명백한 제외 또는 보류 사건을 식별한다.

### 자동 제외가 아니라 경고 대상으로 두는 항목

- 자료 부족
- 점유 미확인
- 특수권리 신호
- 감정가와 실거래 괴리
- 위반건축물 의심

### 즉시 중단 가능 항목

- 사용자가 명시한 투자한도와 구조적으로 불일치
- 분석에 필요한 원본이 존재하지 않거나 접근 불가
- 개인정보·접근권한 정책 위반
- 시스템 범위 밖의 실제 법률대리 또는 자동입찰 요청

### Gate

PM이 `CONTINUE`, `HOLD`, `EXCLUDE`, `EXPERT_REVIEW_REQUIRED` 중 하나를 기록한다.

---

## S4. 권리·경매절차

### 실행 DAG

```text
Auction Procedure Analyst ─────────────┐
Registry Timeline Analyst ── Rights Analyst ── Rights Counter-Reviewer ─┐
Tenant Facts ───────────── Tenant & Distribution Analyst ────────────────┼─ Evidence Judge
Special Rights Detector ── Legal Source Researcher ─ Legal Safety ───────┘
```

### Rights Analyst 입력

- 구조화된 등기 사실
- 날짜 정렬 결과
- 점유·임차 사실
- 매각물건명세서 관련 필드
- 정책 버전

### Rights Counter-Reviewer 입력

- 원본 사실 패키지
- Rights Analyst 주장과 evidence_id
- 작성자의 추론과 가정

검토자에게 제공하지 않아도 되는 정보:

- 작성자의 자기 신뢰도
- PM의 예상 결론
- 사용자가 기대하는 입찰 의사

### Evidence Judge 판단값

- `AGREED`
- `AGREED_WITH_LIMITATIONS`
- `MORE_EVIDENCE_REQUIRED`
- `EXPERT_REVIEW_REQUIRED`
- `UNRESOLVED_CONFLICT`

### Gate

다음 중 하나라도 존재하면 자동 `BID_READY` 불가:

- assumable_right_candidate
- senior_tenant_suspected
- distribution_input_missing
- special_right_detected
- unresolved_date_conflict

---

## S5. 물건·현장·점유

### 병렬 실행군

- Building & Land Analyst
- Field Inspection Planner
- Urban Planning & Redevelopment Analyst
- Location & Infrastructure Analyst

### 후속 순차 작업

사용자 현장자료가 들어온 후:

- Field Evidence Analyst
- Occupancy & Eviction Analyst

### 무효화 규칙

새 현장사진이나 관리사무소 확인으로 점유·위반·사용상태가 달라지면 다음 결과를 무효화한다.

- 명도 시나리오
- 수리비
- 임대 가능시점
- 금융가정
- 입찰계획

### Gate

- 공식 문서와 현장추정 분리
- 방문 불가 영역은 unknown 유지
- 위반·점유 위험의 사람 확인항목 작성

---

## S6. 시장·입지·유동성

### Market Analyst 입력

- 정규화된 비교사례
- 물건 특성
- 공식 개발계획 사실
- 현재 점유·수리 상태

### Market Counter-Reviewer 검사

- 비교사례 거리
- 전용면적 차이
- 층·향·승강기·주차
- 거래시점
- 특수거래 가능성
- 호가 비중
- 개발기대 과대반영

### Liquidity & Exit Analyst 출력

- 정상 매도기간 범위
- 급매 할인범위
- 임대 전환 가능성
- 대체 출구전략
- 출구전략 실패조건

### Gate

- 최소 비교사례 기준 충족 또는 표본부족 명시
- 실거래와 호가 분리
- 상단가격이 상한가 계산의 단독 근거가 아님

---

## S7. 금융·세금·비용·수익

### 순서

```text
Finance Analyst ─────┐
Tax & Cost Analyst ──┼─ Cost Completeness Validator ─ Profitability Engine
Renovation Analyst ──┤                                  │
Rental Operations ───┘                                  ▼
                                                Investment Red Team
```

### 결정론 입력 원칙

- `confirmed`: 사용자·기관·공식자료로 확인
- `scenario`: 분석용 가정
- `unknown`: 계산에서 0으로 대체 금지

### 필수 시나리오

- downside
- base
- upper
- finance_stress
- delayed_exit

### Investment Red Team 검사

- 취득비용 누락
- 수리·명도·공실 누락
- 미확인 대출을 확정으로 사용
- 매도기간 과소평가
- 호가를 매도가로 사용
- 세금 정책 기준일 누락

### Gate

- 계산 fixture 검증
- 비용 완성도 통과
- unknown 입력이 결과에 미치는 영향 표시
- downside에서 손실 가능성 표시

---

## S8. 모의입찰안

### Bid Plan Drafter 입력

검증 완료된 결과만 사용한다.

- 권리 위험
- 점유·명도 위험
- 시장 가격범위
- 비용·수익 시나리오
- 자금조달 상태
- 사람 확인조건

### 출력

- 전략별 가격 후보
- 절대 상한 후보
- 상한 산정 근거
- 포기조건
- 잠금 전 체크리스트
- 미확인값이 해소될 때의 영향

### Evidence Judge

권리·시장·비용의 충돌을 증거 수준으로 다시 확인한다.

### 사람 승인패키지

```yaml
approval_summary:
  proposed_status: APPROVE_WITH_CONDITIONS
  critical_facts: []
  critical_unknowns: []
  opposing_views: []
  downside_loss: null
  maximum_price_candidate: null
  conditions_before_action: []
  external_expert_questions: []
```

### Gate

- 사람 승인
- 조건 기록
- 실제 행동과 모의분석 구분

---

## S9. 잠금

잠금 대상:

- 증거 목록과 해시
- 추출 결과 버전
- 모델·프롬프트·도구 버전
- 계산 입력과 공식
- 분석 결과
- 사람 승인과 조건
- 모의입찰 가격과 포기조건

잠금 후 변경은 수정이 아니라 새 버전이다.

---

## S10~S11. 결과추적과 사후복기

### 결과추적

- 예상값과 실제값을 같은 필드로 보관
- 공개 확인값과 사용자 입력값 구분
- 추정 데이터는 별도 상태

### 사후복기 오류 분류

- evidence_missing
- extraction_error
- reasoning_error
- policy_stale
- calculation_error
- optimistic_assumption
- external_change
- human_override

### 개선 제안 승격

Postmortem Analyst가 직접 프롬프트를 수정하지 않는다.

```text
Postmortem finding
→ Learning Curator change proposal
→ Prompt & Model Evaluator regression test
→ Governance Review
→ PM / Human approval
→ Git commit and PR
```

---

## 5. 충돌 해결 매트릭스

| 충돌 유형 | 1차 처리 | 2차 처리 | 최종 |
|---|---|---|---|
| 문서 값 충돌 | Evidence Auditor | 추가원본 요청 | PM 보류 |
| 날짜·순위 충돌 | 결정론 정렬기 | Rights Counter | Evidence Judge |
| 권리 해석 충돌 | Counter Reviewer | Legal Safety | 외부전문가/사람 |
| 시세 범위 충돌 | Market Counter | Liquidity Analyst | PM 시나리오 병기 |
| 비용 추정 충돌 | 견적자료 요청 | Investment Red Team | 보수값 사용/보류 |
| 모델 결과 충돌 | 증거·규칙 비교 | 다른 공급자 재검토 | 사람 판정 |
| 정책 기준 충돌 | Freshness Guard | 공식자료 확인 | 해당 분석 무효화 |

다수결, 평균값, 더 자신 있게 말한 모델을 기준으로 해결하지 않는다.

---

## 6. 재시도와 승격 예산

### 자동 재시도 가능

- JSON 또는 스키마 오류: 동일 모델 1회
- 일부 필드 누락: 수정 지시 1회
- 일시적 MCP 실패: 1회

### 상위 모델 승격

- 저비용 모델이 핵심 표를 읽지 못함
- 문서 간 관계 해석 필요
- 고위험 권리 후보 발생

### 사람 또는 전문가 승격

- 2회 검토 후에도 핵심 결론 충돌
- 원문 자체가 불명확
- 법률·세무·대출의 최신 기준 확인 필요
- 실제 현장·기관 확인만 가능한 사실

### 호출 제한

- 동일 task lineage 최대 3개 분석 세션
- Reviewer 최대 2개
- 새로운 증거 없이 세 번째 재분석 금지

---

## 7. 무효화 규칙

다음 이벤트가 발생하면 영향받는 분석을 `STALE`로 변경한다.

| 이벤트 | 무효화 대상 |
|---|---|
| 새 등기사항증명서 | 권리·임차·입찰계획 |
| 매각물건명세서 변경 | 권리·점유·명도·입찰계획 |
| 매각기일 변경 | 절차·자금일정·추적 |
| 현장 점유 변경 | 명도·비용·임대개시 |
| 새 실거래 자료 | 시장·수익·입찰계획 |
| 대출 조건 변경 | 금융·수익·상한가 |
| 세법·정책 변경 | 비용·수익·입찰계획 |
| 수리견적 확보 | 비용·수익·입찰계획 |

PM은 무효화된 결과를 최종 보고서에서 제거하는 대신, 이전 버전과 무효화 사유를 보존한다.

---

## 8. 포드별 완료 선언

에이전트는 `완료`를 자유문장으로 선언하지 않는다.

```yaml
completion:
  status: complete|complete_with_limits|blocked|expert_review_required
  required_outputs_present: true
  schema_valid: true
  evidence_coverage: 1.0
  critical_unknowns: []
  conflicts: []
  reviewer_status: passed
  next_stage_eligible: false
  reason: 사람 승인 필요
```

`next_stage_eligible`은 에이전트가 아니라 PM과 정책엔진이 최종 결정한다.
