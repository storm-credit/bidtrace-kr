# 권리분석 프레임워크

- 상태: Draft
- 기준일: 2026-07-21
- 주의: 법률 자문이 아니라 분석 절차와 승격 정책 설계

## 1. 목적

권리분석을 단일 결론 생성 작업이 아니라 다음의 검증 가능한 단계로 분해한다.

```text
원문 수집
 → 사실 추출
 → 권리·점유 타임라인
 → 말소기준권리 후보
 → 권리별 말소·인수 후보 분류
 → 임차인·배당 영향
 → 특수권리 검사
 → 독립 반대검토
 → 사람·전문가 승인
```

공식 생활법령 안내는 등기기록과 현장조사를 통해 권리를 확인하고, 이를 시간순으로 배열한 뒤 말소기준권리를 찾고 인수되는 권리가 있는지 확인하는 접근을 설명한다. 등기기록에 드러나지 않을 수 있는 유치권·분묘기지권 등은 현장 확인이 필요하다.

## 2. 분석 입력

### 필수

- 최신 등기사항증명서
- 매각물건명세서
- 현황조사서
- 감정평가서
- 매각기일·경매개시 관련 정보
- 물건과 점유 기본정보

### 조건부 필수

- 배당요구 관련 자료
- 임차인 전입·확정일자·점유 확인자료
- 건축물대장과 토지자료
- 현장사진과 조사메모
- 선순위 권리 원인서류
- 법정지상권·유치권 등 특수쟁점 관련 자료

자료가 없으면 `UNKNOWN` 또는 `EXPERT_REVIEW_REQUIRED`를 사용한다.

## 3. 사실 추출 계층

해석 전에 다음 필드를 원문 그대로 구조화한다.

```yaml
right_id: string
registry_section: gapgu|eulgu|other
registration_purpose: string
receipt_date: date|null
cause_date: date|null
right_holder: string|null
obligor: string|null
amount: number|null
maximum_claim_amount: number|null
related_registration: string|null
status: active|changed|transferred|cancelled|unknown
source_evidence_id: string
source_location: string
```

### 금지

- 접수일과 원인일을 임의로 동일하게 처리
- 변경·이전 권리를 별도 신규 권리로 중복 계산
- 말소 표시가 불명확한데 자동으로 inactive 처리
- 원문에 없는 금액·권리자·날짜 생성

## 4. 타임라인 계층

타임라인은 등기만이 아니라 다음 사건을 함께 표시한다.

- 소유권 변동
- 담보권·압류·가압류·가등기
- 임차인 점유·전입·확정일자
- 경매개시결정
- 배당요구
- 현장조사 기준일
- 문서 발급·작성 기준일

예시:

```yaml
- event_id: EVT-001
  effective_date: 2020-07-01
  recorded_date: 2020-07-02
  event_type: mortgage_registration
  subject_ids: [RIGHT-001]
  evidence_ids: [EVD-001]
  certainty: verified
```

날짜가 서로 다른 의미를 갖는 경우 `effective_date`, `recorded_date`, `observed_date`를 구분한다.

## 5. 말소기준권리 후보 탐색

공식 생활법령 안내는 저당권·근저당권·압류·가압류·담보가등기·경매개시결정등기 중 가장 앞선 권리를 말소기준권리 후보로 설명한다. 다만 사건별 예외와 원인관계를 확인해야 하므로 시스템은 후보와 검토근거를 반환한다.

```yaml
reference_right_candidates:
  - right_id: RIGHT-001
    type: mortgage
    recorded_date: 2020-07-02
    basis_version: rights-policy-2026-07
    confidence: 0.92
    unresolved_conditions: []
```

### 후보 선정 실패 조건

- 등기 페이지 누락
- 날짜 식별 실패
- 담보가등기 여부 불명확
- 경매개시 원인 또는 중복 사건 불명확
- 복수 권리의 선후관계 확인 불가

이 경우 자동분류를 중단한다.

## 6. 권리 분류

각 권리는 다음 상태 중 하나를 가진다.

| 상태 | 의미 |
|---|---|
| likely_extinguished | 현재 자료와 정책상 말소 가능성이 높음 |
| possible_assumption | 매수인 인수 가능성 검토 필요 |
| survives_by_condition | 특정 조건에 따라 존속 가능 |
| special_review_required | 일반 순위규칙으로 처리 금지 |
| unknown | 자료 부족 또는 충돌 |

분류 결과에는 반드시 조건과 반대 가능성을 적는다.

```yaml
classification:
  right_id: RIGHT-004
  result: possible_assumption
  reasons: []
  conditions: []
  counter_interpretations: []
  evidence_ids: []
  human_review_required: true
```

## 7. 일반 순위규칙의 예외 검사

공식 안내에서 제시하는 예외 신호를 별도 검사한다.

### 전세권

선순위 여부뿐 아니라 배당요구 여부가 결과에 영향을 줄 수 있으므로 다음 필드가 필요하다.

- 설정일·순위
- 범위
- 배당요구 여부
- 배당으로 전액 변제 가능성
- 매각물건명세서 기재

### 등기되지 않은 임차권

- 실제 점유
- 주민등록·사업자등록 관련 사실
- 확정일자
- 배당요구
- 예상 배당과 미변제 가능성

이 영역은 Tenant & Distribution Analyst가 별도 분석한다.

### 가처분·가등기

권리의 목적과 원인을 확인하지 않고 단순히 후순위라는 이유로 말소 분류하지 않는다.

### 유치권·법정지상권·분묘기지권

등기 순위만으로 처리하지 않고 무조건 `special_review_required`로 보낸다.

## 8. 임차인 분석 연결

Rights Analyst는 임차인의 법률효과를 단독 확정하지 않고 필요한 사실과 위험 신호를 Tenant Analyst에 전달한다.

```yaml
tenant_handoff:
  tenant_id: TENANT-001
  confirmed_facts: []
  conflicts: []
  questions:
    - occupancy_start
    - resident_registration_date
    - fixed_date
    - distribution_demand
    - deposit_amount
```

Tenant Analyst 결과가 없으면 권리분석 완료상태는 `conditional` 이하로 제한한다.

## 9. 특수권리 승격

다음은 자동 완료 금지다.

- 유치권 주장 또는 점유 신호
- 토지와 건물 소유자가 달라 법정지상권 검토가 필요한 경우
- 지분매각
- 대지권 미등기·불명확
- 철거·인도·처분금지 가처분
- 가등기의 성격 불명확
- 매각 제외 부분과 현황 사용부분 충돌
- 종물·부합물·제시외 건물 문제
- 분묘 존재 가능성

승격 결과:

```yaml
escalation:
  category: legal_special_right
  severity: critical
  required_professional: attorney|judicial_scrivener|surveyor|architect
  questions: []
  bid_state_effect: block_bid_ready
```

## 10. 독립 검토

1차 권리분석 후 다른 공급자 모델 또는 독립 전문가 역할이 다음을 다시 검사한다.

- 타임라인 재구성
- 말소기준권리 후보
- 누락된 선순위·특수권리
- 원문 간 충돌
- 과도한 안전 표현
- 사람 검토 누락

의견 충돌은 다수결로 해결하지 않고 증거 직접성·최신성·규칙 적합성을 비교한다.

## 11. 완료 등급

### COMPLETE_HIGH_CONFIDENCE

- 필수 원문 모두 확보
- 타임라인과 순위 검증
- 임차인 분석 완료
- 특수권리 신호 없음 또는 전문가 검토 완료
- 독립 검토 통과

### COMPLETE_WITH_CONDITIONS

- 핵심 구조는 확인됐으나 입찰 전 확인 조건 존재

### INCOMPLETE

- 필수자료 누락 또는 핵심 충돌

### EXPERT_REVIEW_REQUIRED

- 일반 분석범위를 넘어서는 법률쟁점

## 12. 출력 보고서

```text
1. 분석 범위와 기준일
2. 사용 증거
3. 권리 타임라인
4. 말소기준권리 후보
5. 권리별 분류표
6. 임차인·배당 연계
7. 특수권리 신호
8. 반대해석
9. 미확인 사항
10. 추가자료·전문가 질문
11. 사람 승인 필요성
```

## 13. 평가 사례

- 선순위 근저당 이후 후순위 권리
- 선순위 전세권과 배당요구 여부
- 선순위 임차인 의심
- 후순위 가처분이지만 목적 검토가 필요한 사례
- 유치권 주장
- 토지·건물 소유자 불일치
- 다가구 다수 임차인
- 대지권 미등기

## 14. 공식 참고자료

- 찾기쉬운 생활법령정보, 매수로 인해 말소·인수되는 권리의 확인
  - https://www.easylaw.go.kr/CSP/CnpClsMain.laf?ccfNo=2&cciNo=1&cnpClsNo=2&csmSeq=306
- 관련 법령은 사건 분석 시 국가법령정보센터의 최신 시행본과 판례를 별도 확인
  - https://www.law.go.kr

정책 규칙은 법령·판례 변경 가능성이 있으므로 `basis_version`과 확인일을 반드시 기록한다.
