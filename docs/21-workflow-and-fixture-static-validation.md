# Workflow 및 Fixture 정적검증 설계

## 1. 목적

실제 모델을 실행하기 전에 Workflow Registry, Evaluation Case Manifest, Agent Task Contract 사이의 구조적 오류를 탐지한다. 정적검증은 법률·시장 판단의 정확성을 검증하지 않지만, 잘못된 참조와 위험한 실행경로가 운영 하네스로 들어가는 것을 막는다.

## 2. 검증 대상

```text
workflows/*.yaml
evaluations/cases/**/manifest.yaml
evaluations/cases/catalog.yaml
evaluations/harness/*.yaml
schemas/*.md 또는 구현 JSON Schema
prompts/system/*
```

## 3. 검증 단계

### S0. YAML 및 기본 구조

- YAML 파싱 가능
- 필수 필드 존재
- ID 형식 준수
- 중복 `case_id`, `workflow_id`, `stage_id` 없음
- 상태 enum 유효

### S1. 참조 무결성

- `base_workflow`가 Registry에 존재
- Overlay가 존재하고 Overlay 타입으로 등록됨
- Agent ID가 Agent Registry에 존재
- Reviewer가 작성자와 다른 역할
- Schema와 Harness Layer 참조가 존재
- 카탈로그의 디렉터리와 manifest의 `case_id`, `slug`가 일치

### S2. DAG 무결성

- 순환 의존성 없음
- 모든 단계가 시작점에서 도달 가능
- 종료되지 않는 고립 단계 없음
- `depends_on` 대상이 존재
- Map-Reduce의 Reduce 단계는 모든 Map 결과 또는 명시적 차단상태에 의존
- Debate의 Judge는 Primary와 Counter 결과 모두에 의존

### S3. 위험 Overlay 완전성

다음 위험 신호는 지정 Overlay 또는 동등한 전문가·사람 게이트를 요구한다.

| 위험 신호 | 최소 요구 |
|---|---|
| senior_tenant_suspected | Tenant Analyst + Rights Counter + 사람 승인 |
| assumable_deposit_unknown | BID_READY 차단 |
| owner_mismatch | Special Rights Overlay + 외부 법률검토 |
| statutory_surface_right_suspected | Special Rights Overlay |
| partial_share | Share Workflow + 외부 법률검토 |
| lien_claimed | Special Rights Overlay 전체 경로 |
| provisional_registration | Special Rights Overlay |
| injunction | Special Rights Overlay |
| illegal_extension_suspected | Building Gate + 현장/행정 확인 |
| loan_unverified | Finance Gate + BID_READY 차단 |
| new_evidence_after_lock | Snapshot/Invalidation 서비스 |

위험 신호가 있는데 요구 단계가 없으면 P0 구성 오류다.

### S4. 사람 승인 경로

- `BID_READY` 직전에 사람 승인 게이트가 있어야 한다.
- `BID_LOCKED`는 별도의 잠금 승인과 스냅샷 해시를 요구한다.
- 특수권리·선순위 임차인·지분·토지/건물 불일치·미확인 대출은 자동승인 불가다.
- 어떤 Workflow도 실제 입찰·송금·계약·법원 제출 작업을 포함할 수 없다.

### S5. 독립 검토

- 권리·임차인·특수권리·입찰계획 고위험 단계는 Reviewer가 있어야 한다.
- Reviewer는 작성자와 동일 agent ID일 수 없다.
- 운영 설정에서는 다른 공급자 또는 독립 프로필이어야 한다.
- Evidence Judge는 모든 사건에서 상시 호출하지 않고 결론 충돌 또는 중요 증거 충돌 조건에서 호출한다.

### S6. 도구 및 보안

- Agent Task Contract의 허용 도구는 역할 Allowlist의 부분집합이어야 한다.
- 실제 쓰기·제출·송금 도구는 허용목록에 존재할 수 없다.
- 사건별 증거 범위는 단일 `case_id`로 제한한다.
- 문서 내부 명령은 도구 권한 변경에 영향을 줄 수 없다.
- 잠금 스냅샷 수정은 금지하고 후속버전 생성만 허용한다.

### S7. 증거·정책·계산

- 모든 `critical_findings`는 필요한 증거 타입을 가져야 한다.
- 날짜에 따라 변하는 정책은 기준일과 정책 버전을 요구한다.
- 비용 누락을 기본 0으로 처리하는 설정은 금지한다.
- 계산단계는 언어모델 단독 산술이 아니라 결정론 계산기 참조를 요구한다.
- 비교사례는 실거래·호가·임대등록가 타입을 구분해야 한다.

### S8. Fixture 상태 일관성

- 증거파일이 없으면 `DRAFT_MANIFEST`만 허용한다.
- Expected Facts 검토 전 `EXPECTED_RESULT_REVIEWED` 이상 불가
- 실행 가능한 원문과 정답 fixture 전 `HARNESS_EXECUTABLE` 불가
- 외부 전문가 검토 전 고위험 사건의 `EXPERT_CALIBRATED` 불가
- 활성 회귀세트 승격 전 P0/P1 0건을 요구한다.

### S9. 무효화 규칙

각 새 증거 유형은 영향받는 산출물을 명시해야 한다.

- 새 등기 → 권리, 특수권리, 입찰계획
- 새 임차인·점유자료 → 임차인, 명도, 비용, 수익성, 입찰계획
- 새 건축자료 → 건축, 수리, 시장조정, 금융, 입찰계획
- 새 시장자료 → 시장, 수익성, 입찰계획
- 새 대출·세금자료 → 금융/세금, 수익성, 입찰계획

과거 잠금 스냅샷은 무효화 대상이 아니라 보존 대상이다.

## 4. 오류 등급

### P0

- 사람 승인 우회
- 특수권리 Overlay 누락
- 다른 사건 증거 접근 허용
- 잠금 스냅샷 변조 허용
- 실제 입찰·금전·법원 제출 도구 등록
- 필수 위험 단계가 DAG에서 도달 불가능

### P1

- 독립 검토자 누락
- 비용·정책·계산 검증 누락
- 필수 증거 없는 완료 게이트
- 영향 결과 무효화 누락
- Map-Reduce 일부 임차인 누락 가능

### P2

- 용어·상태명 불일치
- 불필요한 고비용 에이전트 상시 호출
- 추적성과 관찰성 필드 누락

### P3

- 설명·문서 포맷·비핵심 메타데이터 문제

## 5. 정적검증 결과 계약

```yaml
validation_run_id: STATIC-20260721-001
registry_version: 1.0.0
fixture_catalog: bidtrace-core-cases-v1
result: pass|fail
errors:
  - rule_id: S3-RISK-OVERLAY
    severity: P0
    file: evaluations/cases/CASE-009-lien-claim/manifest.yaml
    path: classification.overlays
    message: lien_claimed requires special-rights-overlay
warnings: []
summary:
  files_checked: 0
  workflows_checked: 0
  cases_checked: 0
  p0: 0
  p1: 0
  p2: 0
  p3: 0
```

## 6. CI 승격 게이트

```text
PR 생성
→ YAML/Schema 검사
→ 참조 무결성
→ DAG 검사
→ 위험 Overlay 검사
→ 권한·사람 승인 검사
→ Fixture 상태 검사
→ 결과 리포트
```

P0 또는 P1이 있으면 PR 승격과 Workflow/Fixture 운영등록을 차단한다. P2는 승인된 예외가 없으면 차단하고, P3는 경고로 허용할 수 있다.

## 7. 현재 한계

현재 문서는 정적검증 규칙 설계이며 validator 코드는 작성하지 않는다. 실제 파서·DAG 검사·JSON Schema·GitHub Actions는 최종 아키텍처 승인 이후 구현한다.
