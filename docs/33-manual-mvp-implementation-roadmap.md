# BidTrace KR Manual MVP 구현 로드맵과 WBS

## 1. 상태

```yaml
stage: G
status: PLANNED_NOT_STARTED
code_allowed: false
start_condition: Stage F human approval
```

이 문서는 구현 작업을 정의하지만 구현을 시작하지 않는다.

## 2. MVP 목표

사용자가 경매 사건을 수동 등록하고 문서를 업로드한 뒤, 증거 기반 분석을 검토하고 모의입찰 계획을 잠그며 실제 결과를 수동 비교할 수 있어야 한다.

MVP 성공 기준:

- 사건과 증거 이력 보존
- 권리·임차인 핵심 위험 누락 방지
- 모든 핵심 주장 증거 연결
- 계산 재현
- 사람 승인 우회 불가
- 잠금 스냅샷 변경 불가
- 대표 fixture 회귀검증

## 3. MVP 제외

- 법원 자동 크롤링
- 자동 입찰
- 송금·대출 신청
- 계약·법원 제출
- 모든 특수권리 자동 확정
- 낙찰가 예측 모델
- 대규모 멀티테넌트 SaaS
- Kubernetes
- 전용 검색엔진
- 전용 분산 Workflow Engine

## 4. 구현 순서 원칙

1. Validator와 불변식을 먼저 만든다.
2. Core 상태·증거·승인을 Worker보다 먼저 만든다.
3. 수동 입력 경로를 먼저 완성한다.
4. Agent는 수동 기능을 대체하지 않고 보조한다.
5. 계산은 LLM과 분리한다.
6. 평가 fixture를 구현과 동시에 확장한다.
7. 각 Work Package는 작은 Draft PR로 진행한다.

## 5. Epic 구조

### EPIC G0 — Repository and Quality Foundation

목표:

- 구현 저장소 구조
- 빌드·테스트·검증의 최소 기반

작업패키지:

#### G0-WP01 Repository Layout

- `apps/web`
- `apps/core`
- `apps/worker`
- `packages/contracts`
- `evaluation`
- `infra/local`

수용기준:

- 각 앱 독립 빌드
- 공통 계약 버전관리
- 생성물 source control 제외

#### G0-WP02 Static Validators

- Workflow YAML validator
- Agent Task Contract validator
- fixture manifest validator
- Prompt metadata validator
- Model/Tool registry validator

수용기준:

- 잘못된 참조, DAG 순환, 승인 우회, side-effect tool을 CI에서 차단
- CASE-001~012 manifest 정적검증

#### G0-WP03 CI Security Baseline

- dependency scan
- secret scan
- SBOM
- test report
- branch protection 제안

완료 게이트:

- P0 정적 fixture가 의도대로 실패
- 정상 fixture 통과

### EPIC G1 — Core Case and Evidence

#### G1-WP01 Identity and Access Skeleton

- OIDC 연동
- 사용자·역할
- case membership
- Worker service identity

수용기준:

- cross-case 접근 거부 테스트
- Worker human endpoint 접근 거부

#### G1-WP02 Case Management

- 사건 생성·수정 가능한 메타데이터
- 상태 조회
- 법원 일정 수동 입력
- 상태전환 command

수용기준:

- state_version 충돌 방지
- 허용되지 않은 transition 거부

#### G1-WP03 Evidence Upload

- quarantine upload
- hash
- MIME/size 검사
- original/derivative lineage
- evidence index

수용기준:

- 원본 overwrite 불가
- 악성/비정상 파일 차단
- presigned URL 만료 검증

#### G1-WP04 Audit Foundation

- 구조화 감사 이벤트
- actor/resource/trace
- append-only 권한

완료 게이트:

- 사건 등록부터 증거 조회까지 수동 end-to-end
- 다른 사건 증거 접근 P0 테스트 통과

### EPIC G2 — Workflow and Manual Analysis

#### G2-WP01 Workflow Registry Runtime

- approved Workflow 로딩
- Base + Overlay 합성
- DAG compile
- execution plan version

수용기준:

- CASE 유형별 예상 단계 구성
- 특수권리 Overlay 누락 차단

#### G2-WP02 Manual Fact and Claim Entry

- 원문 위치와 사실 등록
- claim 분류
- unknown/disputed 처리
- 분석 버전

수용기준:

- evidence 없는 high-risk claim 승인 불가
- fact와 interpretation 분리

#### G2-WP03 Rights and Tenant Screens

- 권리 타임라인
- 임차인별 테이블
- 다가구 completeness
- counter interpretation

완료 게이트:

- CASE-001~006을 사람 입력만으로 표현 가능
- 임차인 누락 시 reduce 완료 불가

### EPIC G3 — Deterministic Calculation

#### G3-WP01 Policy Registry

- policy version
- effective/expiry date
- approval status
- stale propagation

#### G3-WP02 Cost Catalog

- verified/calculated/assumption/range/unknown
- 취득·보유·수리·명도·처분 비용

#### G3-WP03 Profitability Engine

- 공식 버전
- 입력 snapshot
- conservative/base/upper
- calculation trace

수용기준:

- fixture 계산 100% 일치
- unknown을 0원 처리 금지
- 정책 만료 시 관련 결과 stale

### EPIC G4 — Approval and Bid Plan Lock

#### G4-WP01 Decision Package

- AP-01~AP-07 package 생성
- facts/claims/unknowns/counter/conditions
- package hash

#### G4-WP02 Approval Decision

- 승인·조건부승인·자료요청·전문가요청·보류·제외
- actor role
- append-only

#### G4-WP03 Bid Plan Snapshot

- 입력 분석·시나리오 hash
- AP-06 요구
- immutable version
- 새 증거 후 historical 유지

수용기준:

- Worker 승인 불가
- package hash mismatch 거부
- stale 분석 잠금 거부
- CASE-012 통과

### EPIC G5 — Worker and Model Gateway

선행조건:

- G1~G4 Core 불변식 완료
- 수동 분석 경로 완료

#### G5-WP01 Task Queue and Outbox

- outbox
- DB task queue
- lease/heartbeat
- retry/dead-letter
- idempotency

#### G5-WP02 Native Worker

- task contract 수신
- scoped evidence bundle
- structured result
- execution logs

#### G5-WP03 LLM Gateway

- provider adapter
- model registry
- schema output
- budgets
- sensitive data policy

#### G5-WP04 MCP Broker

- tool allowlist
- case/task scope
- network/path restrictions
- audit

#### G5-WP05 Core Agents

초기 활성화:

1. Document Extractor
2. Rights Analyst
3. Rights Counter-Reviewer
4. Market Analyst
5. Finance & Cost Analyst
6. Bid Plan Drafter
7. Investment Red Team

Special Rights는 조건부·수동 검토 중심으로 시작한다.

완료 게이트:

- CASE-001~012 중 활성 agent 대상 fixture 실행
- P0/P1 0건
- 외부 모델 전송정책 승인

### EPIC G6 — Dashboard and Operational Flow

#### G6-WP01 Case Dashboard

- 진행률
- 차단·stale
- 다음 작업
- 승인 대기

#### G6-WP02 Evidence Viewer

- 원문 위치 이동
- 추출 사실 연결
- 버전·lineage

#### G6-WP03 Analysis Views

- 권리·임차인
- 건축·현장
- 시장
- 비용·수익

#### G6-WP04 Approval UX

- Decision Package
- 반대해석
- 조건부 승인
- 접근성

#### G6-WP05 Bid Lock UX

- 변경 전후 비교
- hash/버전
- 잠금 후 수정 불가

완료 게이트:

- 사용자 walkthrough
- 승인 실수 방지 usability 검증
- 모바일은 조회 중심, 핵심 승인은 데스크톱 우선 검토

### EPIC G7 — Result Tracking and Learning

#### G7-WP01 Auction Result Entry

- 낙찰·유찰·취하·변경
- 실제 낙찰가·입찰자 수

#### G7-WP02 Post-auction Cost and Outcome

- 명도·수리·금융·임대·매도 수동 기록

#### G7-WP03 Postmortem

- 예상/실제 비교
- 오차 원인
- fixture 후보 생성

#### G7-WP04 Evaluation Dashboard

- model/prompt regression
- P0/P1
- 비용·지연
- promotion status

### EPIC G8 — Production Readiness

#### G8-WP01 Security Harness

- H1-SEC-001~010
- malicious upload
- prompt injection
- cross-case
- locked mutation

#### G8-WP02 Observability

- logs/metrics/traces
- alerts
- cost monitoring

#### G8-WP03 Backup and Restore

- DB backup/PITR
- Object inventory
- restore drill

#### G8-WP04 Runbooks

- provider outage
- queue backlog
- data incident
- approval integrity

#### G8-WP05 External Review

- 법률·세무·금융 전문가 표본 보정
- 개인정보·보안 검토

완료 게이트:

- Go/No-Go 체크리스트 충족
- 제한 사용자 Manual MVP 승인

## 6. 의존관계

```text
G0
 ↓
G1 Core/Evidence
 ↓
G2 Workflow/Manual Analysis
 ├─────────→ G3 Calculation
 └─────────→ G4 Approval/Lock
                 ↓
              G5 Worker/LLM
                 ↓
              G6 Dashboard
                 ↓
              G7 Tracking
                 ↓
              G8 Production Readiness
```

G6 UI 일부는 병렬 시작 가능하지만 G4 승인 계약보다 먼저 확정하지 않는다.

## 7. 구현 PR 규칙

각 Work Package:

- 별도 branch
- 범위 1개
- 설계 문서 링크
- schema migration 명시
- fixture 추가 또는 업데이트
- 보안 영향
- rollback 계획
- Draft PR
- 자동검증
- 사람 승인

금지:

- 여러 Epic을 한 PR에 혼합
- main 직접 push
- 평가 없이 prompt/model 변경
- migration과 대규모 리팩터링 혼합
- 테스트를 비활성화해 merge

## 8. 정의된 완료

`Done`은 코드 작성만 의미하지 않는다.

```text
코드
+ 테스트
+ fixture
+ 감사로그
+ 권한검사
+ 관찰성
+ 문서
+ rollback
+ 사람 승인
```

## 9. 구현 시작 승인안

첫 구현 묶음은 G0만 권장한다.

- repository skeleton
- contract package
- static validators
- CI baseline

G0는 실제 경매 의사결정 기능을 만들지 않으며 이후 구현의 안전 기반을 검증한다.

다만 사용자 명시 승인 전까지 G0도 시작하지 않는다.
