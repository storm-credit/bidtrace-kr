# 사람 승인 UX와 Decision Package

## 1. 목적

BidTrace KR의 사람 승인은 단순한 `승인/거절` 버튼이 아니다. 사용자가 다음 내용을 확인하고 책임 있는 결정을 내리도록 하는 검토 절차다.

- 어떤 사실이 확인됐는가
- 어떤 해석이 적용됐는가
- 어떤 반대해석이 존재하는가
- 어떤 자료가 없거나 충돌하는가
- 승인하면 어떤 상태와 산출물이 변경되는가
- 새 자료가 들어오면 어떤 승인이 무효화되는가

## 2. 승인 원칙

1. 결론보다 근거를 먼저 보여준다.
2. 위험 점수 하나로 안전 여부를 표현하지 않는다.
3. `확인`, `해석`, `가정`, `미확인`을 시각적으로 분리한다.
4. 고위험 승인에서는 반대검토 결과를 반드시 함께 표시한다.
5. 승인자는 조건부 승인과 추가자료 요청을 할 수 있다.
6. 승인 결과와 이유는 변경 불가능한 감사기록으로 남긴다.
7. 새 증거가 승인 전제를 깨면 승인 상태를 자동으로 `STALE` 처리한다.
8. 실제 입찰·송금·계약 실행과 분석 승인을 분리한다.

## 3. 승인 유형

### AP-01. 자료 완성도 승인

목적: 분석을 시작할 최소자료가 확보됐는지 확인한다.

승인자가 확인하는 것:

- 필수문서 목록
- 기준일과 최신성
- 누락·읽기불가·상충 자료
- 개인정보 및 접근권한

가능한 결정:

- `APPROVE_ANALYSIS_START`
- `REQUEST_MORE_EVIDENCE`
- `HOLD`

### AP-02. 권리·임차인 위험 승인

목적: 권리분석과 임차인 분석의 미확인·인수 가능 위험을 확인한다.

필수 표시:

- 권리 타임라인
- 말소기준권리 후보
- 인수 가능 권리 후보
- 임차인별 점유·전입·확정일자·배당요구 상태
- Primary와 Counter Reviewer의 차이
- 외부전문가 질문 목록

가능한 결정:

- `APPROVE_WITH_CONDITIONS`
- `REQUEST_MORE_EVIDENCE`
- `REQUEST_EXPERT_REVIEW`
- `EXCLUDE`

### AP-03. 건축·현장·명도 승인

목적: 현장·건축·점유 위험이 비용과 일정에 적절히 반영됐는지 확인한다.

필수 표시:

- 공식자료와 현장관찰 차이
- 위반·증축·면적 불일치
- 수리·명도 기간과 비용 범위
- 내부 미확인 영역
- 확인할 기관·전문가

### AP-04. 자금·비용 가정 승인

목적: 대출, 세금, 비용과 자기자본 가정을 확인한다.

필수 표시:

- 확인된 자금과 미확인 자금
- 정책 버전과 기준일
- 누락비용 검사 결과
- 금리·대출비율·보유기간 민감도
- 자금부족 발생 조건

가능한 결정:

- `APPROVE_ASSUMPTIONS`
- `APPROVE_WITH_LIMITS`
- `REQUEST_LENDER_CONFIRMATION`
- `REQUEST_TAX_REVIEW`
- `HOLD`

### AP-05. 모의입찰 계획 승인

목적: 실제 실행과 분리된 모의입찰 계획을 검토한다.

필수 표시:

- 투자 목적
- 목표가와 절대 상한가 후보
- 가격 산출식과 입력값
- 보수·기준·상단 시나리오
- 손익분기점
- 포기 조건
- Investment Red Team의 최악 시나리오
- 미해결 위험과 조건

가능한 결정:

- `APPROVE_PLAN`
- `APPROVE_PLAN_WITH_CONDITIONS`
- `REVISE_PLAN`
- `HOLD`
- `EXCLUDE`

### AP-06. 모의입찰 잠금 승인

목적: 특정 시점의 계획을 수정 불가능한 비교 기준으로 보존한다.

잠금 전 필수조건:

- 필수 승인 완료
- P0/P1 차단 오류 없음
- 사람 승인 이유 입력
- 스냅샷 해시 생성
- 근거·정책·프롬프트·모델 버전 기록

잠금은 실제 입찰 지시가 아니다.

### AP-07. 새 증거 후 재승인

목적: 잠금 이후 새 자료가 들어왔을 때 영향받는 결과와 계획을 새 버전으로 검토한다.

필수 표시:

- 기존 잠금 계획
- 새 증거
- 무효화된 분석
- 유지 가능한 분석
- 이전값과 새값 차이
- 새 승인 필요 항목

## 4. Decision Package

PM Orchestrator는 사람에게 자유형 보고서를 던지지 않고 표준 Decision Package를 생성한다.

```yaml
decision_package_id: DPK-0001
case_id: CASE-001
approval_type: AP-05
requested_at: 2026-07-21T10:00:00+09:00
requested_by: pm-orchestrator
current_state: BID_CANDIDATE
requested_transition: BID_READY

executive_summary:
  recommendation_status: review_required
  one_sentence: string
  why_now: string

confirmed_facts:
  - fact_id: F-001
    statement: string
    evidence_ids: [EVD-001]

key_interpretations:
  - interpretation_id: I-001
    statement: string
    basis: string
    confidence: medium

counter_review:
  agreements: []
  disagreements: []
  unresolved: []

unknowns:
  - unknown_id: U-001
    impact: high
    required_action: string

financial_snapshot:
  confirmed_inputs: []
  assumptions: []
  scenario_results: []
  missing_costs: []

conditions:
  required_before_approval: []
  required_after_approval: []
  automatic_invalidation_events: []

choices:
  - APPROVE_PLAN
  - APPROVE_PLAN_WITH_CONDITIONS
  - REVISE_PLAN
  - HOLD
  - EXCLUDE
```

## 5. 화면 구성

### 5.1 상단 고정 영역

- 사건번호·물건명·법원·매각기일
- 현재 상태
- 승인 요청 유형
- 자료 기준일
- 새 자료 또는 stale 경고
- 실제 입찰 실행 기능이 아님을 표시

### 5.2 핵심 판단 영역

다음 4개 칸을 같은 화면에 배치한다.

| 확인된 사실 | 해석 | 반대해석 | 미확인 |
|---|---|---|---|
| 원문 직접 확인 | 규칙·전문가 해석 | 독립 검토자의 반론 | 추가자료 필요 |

### 5.3 위험·영향 영역

위험을 단일 총점만으로 표현하지 않는다.

- 권리·임차인
- 건축·토지
- 점유·명도
- 시장·유동성
- 자금·세금·비용
- 실행·운영

각 영역에는 다음을 표시한다.

```text
상태: 확인 / 조건부 / 미확인 / 충돌 / 전문가검토
근거: evidence_id
영향: 금액 / 기간 / 진행차단
다음 행동: 자료요청 / 검토 / 제외
```

### 5.4 금액 영역

- 감정가·최저가·시세범위
- 총투자금
- 자기자본
- 미확인 비용
- 손익분기점
- 목표가와 절대 상한 후보
- 최악 시나리오

모든 금액에는 `확인값`, `계산값`, `가정값`, `범위` 라벨을 붙인다.

### 5.5 승인 영역

승인 버튼을 누르기 전에 다음 체크를 요구한다.

- 근거와 미확인 사항을 확인함
- 조건과 무효화 사건을 확인함
- 실제 입찰 실행 승인이 아님을 확인함
- 필요한 경우 외부 전문가 검토 결과를 확인함

고위험 승인에서는 승인 이유를 필수 입력으로 받는다.

## 6. 조건부 승인

조건은 기계 판독 가능하게 저장한다.

```yaml
conditions:
  - condition_id: COND-001
    statement: 금융기관 사전확인에서 LTV 60% 이상 확보
    owner: user
    due_before_state: BID_LOCKED
    evidence_type: lender_confirmation
    on_failure: HOLD
```

조건이 충족되지 않았는데 PM이 다음 상태로 전환하면 P0다.

## 7. 승인 무효화

승인은 전제와 연결한다.

```yaml
approval_dependencies:
  evidence_ids: [EVD-001, EVD-002]
  analysis_versions: [rights-v3, finance-v2]
  policy_versions: [tax-policy-2026-07]
```

의존 항목이 바뀌면 승인 상태를 자동으로 `STALE` 처리하고 재승인 패키지를 만든다. 과거 승인 기록은 삭제하지 않는다.

## 8. 승인 결정 상태

```text
PENDING
→ APPROVED
→ APPROVED_WITH_CONDITIONS
→ REJECTED
→ HOLD
→ STALE
→ SUPERSEDED
```

`STALE`은 과거 승인이 틀렸다는 뜻이 아니라 승인 전제가 변경됐다는 뜻이다.

## 9. 외부 전문가 결과 표시

외부 전문가 의견은 다음과 같이 분리한다.

- 전문가 유형과 검토일
- 제공받은 자료 범위
- 질문과 답변
- 확정이 아닌 조건부 의견
- 후속 확인사항

전문가 의견을 LLM 요약만으로 대체하지 않고 원문·사용자 확인을 연결한다.

## 10. 접근성과 초보자 지원

- 용어에 짧은 설명 제공
- 위험이 왜 중요한지 금액·기간·권리 영향으로 설명
- `안전`, `확실`, `무조건` 같은 표현 제한
- 모바일에서도 승인 전 핵심 위험이 접히지 않게 표시
- 색상 외 아이콘·텍스트로 상태 구분

## 11. 감사 및 보안

저장 항목:

- 승인자
- 결정 시각
- 결정 이유
- 확인한 Decision Package 버전
- 조건
- 승인 전후 상태
- IP·기기 등은 필요성과 개인정보 정책 검토 후 최소 수집

에이전트나 Hermes Worker가 사람 승인 레코드를 직접 수정할 수 없다.

## 12. MVP 범위

MVP에서는 다음 승인 화면을 우선 구현한다.

1. 권리·임차인 검토
2. 자금·비용 가정 검토
3. 모의입찰 계획 검토
4. 모의입찰 잠금
5. 새 증거 후 재승인

기타 승인은 공통 Decision Package를 재사용한다.
