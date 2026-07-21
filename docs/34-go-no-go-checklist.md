# BidTrace KR Go/No-Go 체크리스트

## 1. 목적

설계 완료, 구현 시작, 외부 모델 활성화와 Manual MVP 운영 승격을 서로 다른 승인으로 관리한다.

`GO`는 모든 위험이 사라졌다는 의미가 아니다. 지정된 범위에서 위험이 식별되고 통제·승인됐다는 의미다.

## 2. Gate D — 설계 동결

목표: Stage A~F 문서를 구현 기준선으로 승인한다.

### 필수

- [ ] 제품 범위와 Not Now 명시
- [ ] 사건 상태와 Workflow 용어 일치
- [ ] 전문가 역할·검토자·금지행동 정의
- [ ] Workflow Base/Overlay 구조 정의
- [ ] Agent Task Contract 정의
- [ ] 평가 사건 CASE-001~012 manifest 존재
- [ ] 승인 AP-01~07 정의
- [ ] Core/Worker/LLM/MCP 경계 정의
- [ ] 데이터 소유권과 버전·무효화 정의
- [ ] 보안 위협모델과 P0/P1 정의
- [ ] Manual MVP WBS 정의
- [ ] 실제 입찰·송금·법원제출 제외 확인

### 차단

- 상태 또는 승인 소유자가 둘 이상
- Worker가 DB·Approval·Lock 직접 수정
- 사람 승인 없는 BID_LOCKED 경로
- 원본 증거 덮어쓰기 허용
- 사건 간 격리정책 부재
- 설계 PR 리뷰 없이 동결

판정:

```yaml
gate: DESIGN_FREEZE
status: PENDING_HUMAN_REVIEW
```

## 3. Gate I0 — 구현 시작

목표: G0 구현을 시작할 수 있다.

### 필수 결정

- [ ] Stage F PR 사람 승인
- [ ] 구현 저장소 구조 승인
- [ ] Java/Node/Python 지원 버전 결정
- [ ] 라이선스 정책 결정
- [ ] 테스트·CI 실행환경 결정
- [ ] 공개 repository에 저장 가능한 정보 재확인

### 범위

허용:

- repository skeleton
- contract package
- validator
- CI baseline

금지:

- 실제 사건 원문
- 운영 비밀
- 외부 모델에 개인정보 전송
- 실제 입찰 관련 side effect

## 4. Gate I1 — Core/Evidence 통합

### 필수

- [ ] OIDC와 case scope
- [ ] 사건 상태머신
- [ ] Evidence hash와 lineage
- [ ] quarantine
- [ ] 감사로그
- [ ] cross-case P0 테스트
- [ ] 원본 overwrite 차단
- [ ] backup 기본설정

### 실패 시

Worker/LLM 구현으로 진행하지 않는다.

## 5. Gate I2 — 승인·잠금

### 필수

- [ ] Decision Package hash
- [ ] Approval append-only
- [ ] Worker approval 거부
- [ ] stale 분석 승인 거부
- [ ] locked snapshot update 거부
- [ ] CASE-012 통과
- [ ] 복구 후 hash 검증

## 6. Gate A0 — 외부 모델 시험

운영 데이터가 아닌 합성 fixture에서만 시작한다.

### 필수

- [ ] Model Registry
- [ ] provider 계약·데이터 처리조건 검토
- [ ] secret manager
- [ ] token/cost budget
- [ ] prompt injection fixture
- [ ] restricted data detector
- [ ] 로그 redaction
- [ ] kill switch

### 차단

- RESTRICTED 데이터 전송 가능
- 공급자별 삭제·학습정책 미확인
- 모델 응답이 상태를 직접 변경

## 7. Gate A1 — Agent 기능 통합

### 필수

- [ ] Native Worker conformance
- [ ] Agent Task Contract 검증
- [ ] Tool Permission Broker
- [ ] cross-case denial
- [ ] output schema validation
- [ ] evidence linkage 100%
- [ ] high-risk independent review
- [ ] P0/P1 0건

### Hermes

- [ ] Native Worker와 동일 계약 통과
- [ ] profile isolation 확인
- [ ] hooks를 hard security gate로 오해하지 않음
- [ ] memory에 사건 사실 미저장
- [ ] main push·force push 금지

Hermes 실패는 전체 MVP를 차단하지 않고 Hermes Adapter만 비활성화한다.

## 8. Gate U0 — 사용자 승인 UX 시험

### 필수

- [ ] 근거 원문 위치 이동
- [ ] 사실·해석·가정·미확인 구분
- [ ] 반대해석 표시
- [ ] 조건부 승인 이해 가능
- [ ] stale 승인 경고
- [ ] 잠금 전 최종 차이 확인
- [ ] 접근성 기본검사
- [ ] 실수 취소가 원본 수정이 아닌 후속결정으로 처리

## 9. Gate P0 — 제한 Manual MVP 운영

### 필수

- [ ] Stage G G0~G8 필수범위 완료
- [ ] 보안 H1 테스트 100%
- [ ] CASE-001~012 정적·실행 하네스 통과
- [ ] P0 0건
- [ ] P1 0건
- [ ] calculation fixture 100%
- [ ] mandatory escalation 100%
- [ ] 외부 법률·세무·금융 표본검토
- [ ] 개인정보·보존·삭제정책 승인
- [ ] backup restore drill
- [ ] 사고대응 runbook
- [ ] 제한 사용자 교육

### 운영 제한

- 사용자 수 제한
- 사건 수 제한
- 모델 비용 quota
- 자동수집 비활성화
- 실제 입찰·금전 side effect 없음

## 10. Gate P1 — 범위 확대

운영데이터와 사용자 피드백을 바탕으로 별도 승인한다.

후보:

- 자동 사건 상태 추적
- 공공데이터 연계
- 외부 전문가 portal
- 상가·공장·농지 Workflow
- 전용 Queue
- 확장된 멀티사용자 권한

필수 근거:

- 실제 병목
- 반복 사용성 문제
- 운영 사고와 회귀 추세
- 비용·성능 데이터
- 법률·보안 재검토

## 11. 즉시 No-Go 조건

어느 단계에서도 다음은 즉시 중단한다.

- 실제 입찰 또는 송금 동작이 자동화됨
- 다른 사건 개인정보 접근
- 잠긴 스냅샷 수정
- 사람 승인 위조·우회
- 원본 증거 hash 불일치
- 외부 모델에 승인되지 않은 민감정보 전송
- 평가 실패를 숨기거나 테스트 비활성화
- 법률·세무 확정 보장 표현

## 12. 예외 관리

예외는 다음 필드를 가져야 한다.

```yaml
exception_id: string
scope: string
risk: string
reason: string
compensating_controls: []
owner: string
approved_by: string
approved_at: timestamp
expires_at: timestamp
verification: string
```

만료일과 책임자 없는 예외는 허용하지 않는다.

P0 통제 예외는 원칙적으로 허용하지 않는다.

## 13. 현재 판정

```yaml
design_documents: substantially_complete
design_freeze: pending_human_review
implementation: not_started
runtime_validation: not_executed
production_readiness: no_go
```
