# BidTrace KR 최종 설계 통합검수

## 1. 검수 범위

검수 대상:

- 제품 제안과 사건 생애주기
- PM Orchestrator와 Agent Harness
- 전문가 조직·모델·MCP 배정
- 권리·임차인·특수권리 프레임워크
- 시장·비용·수익·모의입찰
- 사건 유형별 Workflow Registry
- 평가 사건 CASE-001~012
- 사람 승인·대시보드 UX
- 최종 시스템 아키텍처
- 데이터·보안·배포·관찰성·복구
- Manual MVP 구현 로드맵

검수 방식:

```text
용어 정합성
→ 책임 소유권
→ 상태와 데이터 불변식
→ 고위험 경로
→ 실패·재시도·무효화
→ 사람 승인
→ 보안 경계
→ MVP 과잉설계 점검
→ 구현 가능성
```

## 2. 최종 구조 요약

```text
사용자
  ↓
Next.js Web
  ↓
Spring Boot Modular Core
  ├─ Case State
  ├─ Evidence Registry
  ├─ Workflow / Policy / Calculation
  ├─ Analysis Registry
  ├─ Approval / Bid Lock
  └─ Audit
       ↓ signed Agent Task Contract
Worker Runtime Adapter
  ├─ Native Worker
  └─ Hermes Optional Worker
       ↓ brokered access
LLM Gateway / MCP Permission Broker
```

Evaluation Harness는 운영 쓰기경계 밖에 둔다.

## 3. 해결된 주요 모순

### 3.1 PM이 전문가인가 제어자인가

초기 위험:

PM이 자체적으로 권리·시장·입찰 결론을 만들 가능성.

해결:

- PM은 Workflow 계획·라우팅·검증·상태·승인을 조정
- 도메인 결론은 전문 Agent와 결정론 모듈
- PM도 증거 없는 사실을 생성할 수 없음

### 3.2 31개 역할이 31개 실행 Agent인가

초기 위험:

비용·지연·책임 중복과 인위적인 다수결.

해결:

- 논리 역할과 실행 세션 분리
- 상시 Core 서비스, 조건부 LLM, 독립 Reviewer, 결정론 모듈로 재편
- 사건별 최소 실행세트와 Risk Overlay

### 3.3 상태머신을 Agent가 운영하는가

해결:

- 상태전환은 Core 결정론 코드만 수행
- Agent는 transition suggestion만 가능
- 승인·잠금은 사람 명령과 Core guard 필요

### 3.4 Workflow Engine을 즉시 도입하는가

해결:

- 명시적 Workflow Registry + DB 실행계획 + Transactional Outbox
- 전용 Engine은 실제 복잡도·부하 근거 후 검토

### 3.5 Hermes가 PM 또는 시스템 오브 레코드인가

해결:

- Hermes는 선택 Worker Runtime
- Core가 상태·증거·승인·감사를 소유
- Native Worker와 동일 conformance 요구

### 3.6 LLM이 수익과 날짜를 계산하는가

해결:

- 날짜 정렬·금액·수익·정책 만료는 결정론 모듈
- LLM은 입력 후보와 해석을 제공
- 계산 trace와 formula version 저장

### 3.7 원본과 분석결과의 경계

해결:

- Object Store original과 derivative 분리
- EvidenceItem hash 불변
- ExtractedFact, Claim, AnalysisVersion 분리

### 3.8 승인과 실제 입찰의 경계

해결:

- 승인 대상은 분석과 모의입찰 계획
- 실제 입찰·송금·계약·법원제출 기능 제외
- `BID_LOCKED`는 기록 잠금이지 실제 입찰 실행이 아님

### 3.9 새 증거와 잠긴 계획

해결:

- 영향받는 분석과 승인만 stale
- 기존 locked snapshot은 historical 보존
- 새 계획은 새 version

## 4. 책임 단일성 검수

| 책임 | 최종 소유자 | 중복 여부 |
|---|---|---|
| 사건 상태 | Case Management | 없음 |
| 증거 원본 메타데이터 | Evidence Registry | 없음 |
| Object bytes | Object Store | 없음 |
| Workflow 정의 | Workflow Registry | 없음 |
| 실행계획 | Workflow Orchestration | 없음 |
| 도구 권한 | Tool Permission Broker | 없음 |
| 모델 승인상태 | Model Registry | 없음 |
| 정책 | Policy Registry | 없음 |
| 계산 | Calculation Engine | 없음 |
| 분석 버전 | Analysis Registry | 없음 |
| 승인 | Approval Service | 없음 |
| 모의입찰 잠금 | Bid Plan Lock | 없음 |
| 감사 | Audit Module | 없음 |
| Agent 실행 | Worker Runtime | 상태 소유 없음 |
| 운영 승격 평가 | Evaluation Harness + 사람 | 운영 쓰기 없음 |

판정: 책임 소유권은 구현 가능한 수준으로 수렴했다.

## 5. 고위험 경로 검수

### 권리·임차인

필수 경로:

```text
원문 추출
→ 사실·날짜 정규화
→ Rights/Tenant 분석
→ 다른 공급자 Counter Review
→ 규칙검사
→ 필요 시 Evidence Judge
→ Legal Safety
→ 사람/외부전문가
```

차단 조건:

- 필수 날짜·보증금·점유 미확인
- 인수 가능 권리 누락
- 임차인 completeness 미달
- 특별권리 외부검토 미완료

### 투자·입찰계획

필수 경로:

```text
시장 비교
→ 비용·금융·세금 입력
→ 결정론 수익계산
→ Bid Plan Draft
→ Investment Red Team
→ 사람 승인
→ Snapshot Lock
```

차단 조건:

- 미확인 대출 확정값 사용
- 비용 누락을 0원 처리
- 호가를 실거래로 사용
- stale 분석
- 승인 패키지 hash 불일치

판정: 고위험 경로에 단일 모델 또는 자동 승인 지점이 없다.

## 6. 하네스 완성도 검수

### 설계 완료

- H0 계약·정적검증 규칙
- H1 보안 위협 시나리오
- H2 결정론 계산·상태 규칙
- H3 Agent 정확도 평가형식
- H4 독립 검토 요구
- H5 Workflow end-to-end 계획
- H6 외부 전문가 보정 위치
- H7 회귀승격 정책

### 미실행

- Validator 코드
- 실제 모델 비교
- 실제 MCP sandbox 공격시험
- 실제 원문 evidence fixture
- 전문가 정답 보정
- 사용자 테스트

판정:

> 하네스의 설계는 완료에 가깝지만 실행 검증은 시작되지 않았다.

## 7. MVP 과잉설계 검수

제외 또는 연기한 항목:

- Microservices
- Kubernetes
- Temporal 등 전용 Workflow Engine
- Kafka/RabbitMQ 등 외부 broker
- Elasticsearch
- Vector DB 기본 도입
- 모든 Agent 동시 실행
- 자동 크롤링
- 낙찰가 예측 ML
- 실제 입찰·금전 기능

유지한 필수 복잡성:

- Core/Worker 프로세스 경계
- PostgreSQL과 Object Store 분리
- 승인과 잠금 불변성
- Workflow Registry
- Evaluation Harness
- 감사·보안·백업

판정: 안전에 필요한 복잡성은 유지하고 운영규모가 증명되지 않은 인프라는 제거했다.

## 8. 잔여 위험

### R1. 실제 법률 정확도 미보정

현재 공식자료 기반 설계이나 사건별 판례·실무 예외를 포괄하지 않는다.

대응:

- 외부 법률전문가 fixture 보정
- 특수권리 자동 확정 금지
- 사람 승격

### R2. 실제 문서 추출 품질 미측정

대응:

- 익명화 원문 fixture
- 페이지·표·도장·손상 스캔 평가
- 낮은 품질 시 수동 입력 fallback

### R3. 최신 세금·금융정책

대응:

- policy version/effective/expiry
- 확정 세액·대출 승인 표현 금지
- 전문가 확인 질문

### R4. 외부 모델 개인정보 처리

대응:

- 외부전송 데이터등급 승인 전 운영 사용 금지
- redaction과 provider allowlist

### R5. 사용자 승인 과신

대응:

- 반대해석·unknown·조건 표시
- 고위험은 외부전문가 요구
- 위험점수 단독 승인 금지

### R6. 설계 PR 누적과 병합순서

현재 PR이 Foundation 위에 순차적으로 쌓였다.

대응:

- base 순서대로 리뷰·병합
- merge 전 각 PR diff 재검수
- rebase 시 schema/workflow validator 필요

## 9. 문서 상태 판정

| 영역 | 판정 |
|---|---|
| 제품 제안 | FIX 후보 |
| 생애주기·상태 | FIX 후보 |
| PM·Agent 운영모델 | FIX 후보 |
| 권리·임차인 프레임워크 | HARDEN 필요 |
| 시장·비용·수익 | HARDEN 필요 |
| Workflow Registry | FIX 후보 |
| 하네스 계획 | FIX 후보 |
| 12종 manifest | DRAFT, evidence 미작성 |
| 승인 UX | 정보구조 FIX 후보 |
| 최종 아키텍처 | FIX 후보 |
| 보안·복구 | 설계 완료, 실행 미검증 |
| Manual MVP WBS | READY FOR APPROVAL |

## 10. 최종 설계 준비도

```yaml
product_scope: 0.95
lifecycle_and_state: 0.93
pm_and_agent_governance: 0.92
workflow_registry: 0.90
rights_domain: 0.82
market_investment: 0.80
evaluation_design: 0.86
human_approval_ux: 0.85
system_architecture: 0.91
security_and_recovery_design: 0.86
runtime_validation: 0.00
implementation: 0.00
```

숫자는 관리용 상대평가이며 객관적 인증점수가 아니다.

## 11. 최종 판정

### 설계

`SUBSTANTIALLY_COMPLETE_PENDING_HUMAN_REVIEW`

- 구현 가능한 경계와 순서가 정의됨
- 치명적 자동화 위험이 명시적으로 차단됨
- 구현 전 필요한 평가와 승인 위치가 정의됨

### 구현

`NO_GO_UNTIL_EXPLICIT_APPROVAL`

- 실제 코드는 작성하지 않음
- 첫 승인 범위는 G0 Validator/CI 기반만 권장

### 운영

`NO_GO`

- 실제 모델·MCP·원문·전문가·복구 검증 미실행

## 12. 사람에게 필요한 최종 결정

1. Stage F 설계 동결 승인 여부
2. Manual MVP가 수동 사건 등록 중심이라는 범위 승인
3. Web Next.js + Core Spring Boot 결정 승인
4. Native Worker 우선·Hermes 선택 Adapter 승인
5. 외부 모델에 전송 가능한 데이터등급 정책 수립 진행 승인
6. 첫 구현을 G0 Validator/CI로 제한할지 승인

이 결정 전까지 모든 구현 이슈는 계획 상태로 유지한다.
