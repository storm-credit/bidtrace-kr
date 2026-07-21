# BidTrace KR 구현 전 최종 시스템 아키텍처

## 1. 문서 목적

이 문서는 Stage A~E에서 설계한 제품, PM 오케스트레이터, 전문 에이전트, 실행 하네스, Workflow Registry, 평가 사건과 사람 승인 UX를 실제 구현 가능한 하나의 시스템 구조로 통합한다.

이 단계에서는 애플리케이션 코드를 작성하지 않는다. 기술 경계, 데이터 소유권, 보안 원칙, 배포 단위와 구현 순서를 확정한다.

## 2. 최종 아키텍처 원칙

1. **Core가 유일한 시스템 오브 레코드다.**
2. **LLM Worker는 상태와 승인을 소유하지 않는다.**
3. **원본 증거와 파생 분석을 분리한다.**
4. **상태전환, 권한, 계산, 잠금과 감사는 결정론 코드가 담당한다.**
5. **에이전트는 작업계약으로만 실행한다.**
6. **고위험 결론은 독립 검토와 사람 승인을 요구한다.**
7. **Manual MVP에 필요 없는 분산 인프라를 도입하지 않는다.**
8. **실제 입찰, 송금, 계약과 법원 제출 기능을 만들지 않는다.**
9. **모델과 Worker Runtime은 교체 가능해야 한다.**
10. **평가 하네스 실패는 배포를 차단한다.**

핵심 문장:

> AI는 설명하고 의심하며, 계산기는 산출하고, Core는 통제하고, 사람은 승인한다.

## 3. 시스템 컨텍스트

```text
사용자 / 승인자 / 외부 전문가
            │
            ▼
      BidTrace Web
            │ HTTPS / OIDC
            ▼
      BidTrace Core API
  ┌─────────┼──────────────────────────────┐
  │         │                              │
  ▼         ▼                              ▼
PostgreSQL  Object Store              Task Dispatcher
  │         │                              │
  │         │                              ▼
  │         │                     Worker Runtime Adapter
  │         │                       ├─ Native Worker
  │         │                       └─ Hermes Worker
  │         │                              │
  │         │                              ▼
  │         │                       LLM / MCP Gateway
  │         │
  └─────────┴──── Audit / Outbox / Evaluation Export
```

외부 연계는 Manual MVP에서 최소화한다.

- 사용자가 직접 사건과 문서를 등록한다.
- 공개정보 검색은 승인된 Research MCP를 통해 읽기 전용으로 수행한다.
- 결과 추적은 수동 입력을 먼저 지원한다.
- 자동 수집과 예약 추적은 이후 단계에서 별도 승격한다.

## 4. 배포 단위

### 4.1 BidTrace Web

역할:

- 사건 보드와 대시보드
- 증거 업로드와 인덱스 탐색
- 권리·임차인·시장·비용 검토 화면
- Decision Package 표시
- 사람 승인과 모의입찰 잠금 요청
- 실제 결과와 사후복기 입력

기술 방향:

- Next.js와 TypeScript 기반 웹 애플리케이션
- 서버 컴포넌트와 API 프록시는 사용자 세션과 화면 조합에만 사용
- 핵심 업무규칙과 상태전환은 Web에 두지 않음
- 브라우저가 Object Store에 직접 접근하지 않고 짧은 유효기간의 제한된 업로드·다운로드 권한을 Core에서 발급

### 4.2 BidTrace Core API

최종 결정:

- Spring Boot 기반 모듈러 모놀리스
- Java LTS는 구현 시작 시점에 승인된 버전으로 고정
- 단일 배포물 안에서 모듈 경계를 엄격하게 유지
- 공개 API와 내부 Worker API를 분리

Core 모듈:

```text
identity-access
case-management
evidence-registry
workflow-orchestration
agent-task-control
analysis-registry
policy-registry
calculation-engine
approval-service
bid-plan-lock
tracking-postmortem
audit-observability
evaluation-export
```

Core가 소유하는 것:

- 사건 상태
- Workflow 실행계획
- 증거 메타데이터와 해시
- 에이전트 작업계약
- 모델·도구 권한정책
- 분석 버전과 stale 상태
- 계산 입력·공식·결과
- 승인 레코드
- 모의입찰 잠금 스냅샷
- 감사로그

### 4.3 Worker Runtime

Worker는 별도 프로세스 또는 컨테이너로 배포한다.

Worker 책임:

- Core에서 서명된 Agent Task Contract 수신
- 허용된 증거의 제한된 읽기
- 허용된 모델과 MCP 호출
- 구조화된 결과 반환
- 실행 로그와 도구 사용기록 반환

Worker 금지:

- 사건 상태 직접 변경
- 승인 레코드 생성·수정
- 잠긴 스냅샷 수정
- 임의 사건 자료 조회
- DB 직접 쓰기
- Object Store 원본 수정
- 실제 입찰·송금·계약·법원제출

기본 구현 순서:

1. Native Worker Adapter
2. Worker Contract Conformance Test
3. Hermes Adapter
4. 동일 fixture 교차실행
5. 평가 통과 후 선택 활성화

Hermes는 대체 가능한 실행 런타임이며 시스템 오브 레코드가 아니다.

### 4.4 Evaluation Harness

운영 서비스와 분리된 실행 경계를 가진다.

책임:

- Workflow와 fixture 정적검증
- 프롬프트·모델 비교
- P0/P1 회귀검사
- 보안·권한우회 fixture 실행
- 계산 결과 재현성 확인
- 운영 승격 후보 리포트

금지:

- 운영 사건 상태 변경
- 운영 승인 생성
- 운영 잠금 스냅샷 수정
- 실제 외부 부작용 실행

## 5. 데이터 저장 구조

### 5.1 PostgreSQL

PostgreSQL은 구조화된 시스템 오브 레코드다.

주요 스키마:

```text
identity
case_core
evidence
workflow
analysis
finance
approval
audit
evaluation
```

사용 방식:

- 정규화된 핵심 엔터티는 관계형 컬럼 사용
- 모델별 확장 출력은 검증된 JSONB 사용
- 모든 JSONB는 schema_name과 schema_version 필수
- 고위험 데이터는 자유형 JSON만으로 저장하지 않음
- 낙관적 잠금 또는 버전 컬럼으로 동시수정 방지

초기에는 별도 검색엔진을 도입하지 않는다.

- 사건번호, 주소, 상태, 날짜와 권리자는 인덱스 컬럼
- 문서 추출 텍스트는 PostgreSQL Full Text Search부터 사용
- 검색 부하와 정확도 근거가 생긴 뒤 별도 검색엔진 검토

### 5.2 Object Store

S3 호환 Object Store에 다음을 저장한다.

```text
original/      원본 업로드
normalized/    정규화 PDF·이미지
extracted/     OCR·표·텍스트 파생물
reports/       생성 보고서
exports/       평가·감사 내보내기
quarantine/    보안 검사 전 파일
```

원칙:

- 원본 객체는 내용 해시로 식별
- 원본 교체 금지, 새 버전으로 등록
- 파생물은 parent_evidence_id와 transform_version 연결
- 업로드 직후 격리영역에서 형식·악성파일 검사
- 사용자에게는 제한된 presigned URL만 제공
- 운영자도 원본을 직접 덮어쓰지 못함

### 5.3 Queue와 장기작업

Manual MVP에서는 별도 메시지 브로커를 먼저 도입하지 않는다.

초기 구조:

- PostgreSQL task table
- transactional outbox
- lease 기반 worker claim
- idempotency key
- retry policy
- dead-letter 상태
- scheduler heartbeat

외부 Queue 도입 조건:

- 동시 실행량이 DB 기반 큐의 운영한계를 지속적으로 초과
- 장기 추적 작업이 수천 단위로 증가
- 독립적인 재처리와 우선순위 큐가 필요
- 부하검증으로 병목이 확인

## 6. 실행 흐름

### 6.1 사건 등록과 증거 준비

```text
사용자 사건 등록
→ Case 생성
→ 증거 업로드 세션 발급
→ 격리 업로드
→ 해시·보안·형식 검사
→ Evidence Registry 등록
→ 필수자료 완성도 검사
→ DOCUMENTS_PENDING 또는 SCREENING
```

### 6.2 Workflow 실행

```text
PM Orchestrator
→ Base Workflow 선택
→ Risk Overlay 합성
→ DAG 정적검증
→ 실행계획 버전 저장
→ Agent Task Contract 생성
→ Outbox 기록
→ Worker 실행
→ 구조·증거·규칙 검증
→ 독립검토 또는 재시도
→ 분석 버전 저장
→ 사람 승인 패키지 생성
```

### 6.3 모의입찰 잠금

```text
필수 승인 확인
→ 정책·증거·분석 freshness 확인
→ 계산 재실행
→ Decision Package 해시 확인
→ 사람 AP-06 승인
→ immutable bid-plan snapshot 생성
→ BID_LOCKED 전환
```

잠금 후 새 증거가 들어오면 기존 스냅샷을 수정하지 않는다.

```text
새 증거 등록
→ 영향분석
→ 관련 분석 stale
→ 기존 승인 stale
→ 기존 잠금은 historical 유지
→ 새 계획 버전 생성 가능
```

## 7. LLM Gateway와 모델 라우팅

모델명을 업무코드에 하드코딩하지 않는다.

Model Registry 필드:

```yaml
model_profile: R3_REASONING
provider: string
model_id: string
status: candidate|approved|suspended|retired
allowed_data_classes: []
capabilities: []
context_limit: number
cost_policy: string
latency_policy: string
evaluation_run_id: string
approved_at: timestamp
expires_at: timestamp
```

LLM Gateway 책임:

- 공급자 Adapter
- 요청·응답 표준화
- schema-constrained output
- timeout과 retry
- token·비용 측정
- 민감정보 정책 검사
- prompt/model/tool 버전 기록
- 공급자별 장애격리

고위험 독립검토는 가능한 한 다른 공급자와 다른 프롬프트 계열을 사용한다.

같은 모델을 두 번 호출한 결과를 독립 합의로 취급하지 않는다.

## 8. MCP Gateway와 도구 권한

MCP 도구는 Agent Task Contract의 allowlist와 일치해야 한다.

도구 분류:

| 분류 | 예시 | 기본 정책 |
|---|---|---|
| Evidence Read | 사건 증거 읽기 | 사건범위 제한 |
| Calculator | 날짜·금액·수익 계산 | 결정론 함수만 |
| Research Read | 공식 법령·공개자료 | 읽기 전용 |
| Market Read | 실거래·호가 자료 | 출처·수집일 필수 |
| GitHub | 설계·평가 변경 | 별도 Git profile |
| Scheduler | 추적 작업 등록 | Core 승인 필요 |
| Side Effect | 송금·입찰·제출 | 등록 금지 |

권한 검사는 Worker의 선언에 의존하지 않는다.

```text
Worker 요청
→ Tool Permission Broker
→ task_id / case_id / agent_id 검증
→ allowlist 확인
→ 데이터등급 확인
→ 호출 예산 확인
→ 실행 또는 거부
→ 감사로그
```

## 9. 승인과 정책 격리

Approval Service는 Core 내부의 보호된 모듈이다.

- Worker용 DB 계정은 approval schema 쓰기 권한 없음
- API에서도 승인 endpoint는 사람 세션과 권한을 요구
- 승인 대상 패키지 해시가 바뀌면 승인 거부
- stale 분석이 있으면 잠금 승인 거부
- 외부전문가 필수 조건이 충족되지 않으면 승인 거부
- 승인 레코드는 append-only
- 정정은 원본 수정이 아니라 후속 decision record 생성

Policy Registry도 Worker와 분리한다.

- 법령·세금·대출 가정은 policy_version과 effective_date 보유
- 만료·확인 필요 정책은 자동 차단 또는 경고
- Worker는 정책을 읽을 수 있지만 승인된 정책을 변경하지 못함

## 10. API 경계

### Public API

- 사용자 사건·증거·분석 조회
- 업로드 세션
- 수동 입력
- 승인 요청과 결정
- 모의입찰 잠금 요청
- 결과·사후복기 입력

### Worker API

- 작업 lease 획득
- 제한된 Evidence Bundle 획득
- heartbeat
- 결과 제출
- 도구 호출 프록시
- 실행 실패 보고

### Admin API

- Model Registry 후보 관리
- 정책 버전 등록
- Workflow·Prompt 승격
- 평가 리포트 조회
- 사용자·권한 관리

Admin API는 일반 사용자 UI와 분리하고 강한 인증과 감사기록을 요구한다.

## 11. 신뢰 경계

```text
[Browser]
   │ untrusted input
   ▼
[Web / Core Edge]
   │ authenticated domain commands
   ▼
[Core Trust Zone]
   ├─ DB
   ├─ Approval
   ├─ Policy
   └─ Audit
   │ signed task contract
   ▼
[Worker Sandbox]
   │ brokered tools only
   ▼
[External Models / MCP / Public Web]
```

외부 문서와 모델 응답은 모두 신뢰하지 않는 입력으로 취급한다.

- 문서 내부 명령은 지시가 아니라 데이터
- 모델 출력은 검증 전 제안
- MCP 응답은 출처·시간·무결성을 검사
- 외부 URL과 파일은 격리 검사

## 12. 개발·배포 환경

### Local Development

- Windows + WSL2
- Docker Compose
- Web
- Core API
- Worker
- PostgreSQL
- S3-compatible local object store
- local mail/mock services
- 선택적 로컬 모델

### Shared Test

- 단일 테스트 환경
- 익명화·합성 fixture만 기본 사용
- 외부 모델 키는 비밀관리 시스템 사용
- 평가 하네스와 운영 테스트 DB 분리

### Manual MVP Production

- 관리형 PostgreSQL 권장
- 관리형 Object Store 권장
- Web/Core/Worker는 관리형 컨테이너 또는 소규모 VM
- TLS 종료
- OIDC
- 중앙 로그와 백업

Kubernetes 도입은 초기 요구사항이 아니다.

## 13. 확장 경로

다음 상황이 확인될 때만 분리한다.

### Core 서비스 분리

- 독립 배포가 반복적으로 필요
- 특정 모듈이 전체 배포를 방해
- 보안경계를 프로세스 단위로 강화해야 함
- 운영 부하가 명확하게 분리

첫 분리 후보:

1. Worker Runtime
2. Evaluation Harness
3. Document Processing
4. Tracking Scheduler

Approval과 Case State는 가장 늦게 분리하며 일관성 경계를 유지한다.

### 전용 Workflow Engine

도입 조건:

- 수백 단계 장기 워크플로
- 다수의 외부 callback
- 복잡한 보상 트랜잭션
- DB 기반 실행기가 반복 장애 원인

그 전까지는 명시적 상태머신, workflow tables와 outbox를 사용한다.

## 14. 금지 아키텍처

Manual MVP에서 채택하지 않는다.

- 에이전트가 DB에 직접 쓰는 구조
- 프론트엔드가 상태를 직접 계산하는 구조
- 모델 응답만으로 상태전환하는 구조
- 원본 파일을 분석결과로 덮어쓰는 구조
- 모델별로 서로 다른 비정형 저장소를 만드는 구조
- 모든 역할을 상시 LLM 프로세스로 띄우는 구조
- 초기부터 마이크로서비스와 Kubernetes 사용
- 평가 환경이 운영 DB에 쓰는 구조
- 자동입찰·송금·대출신청·법원제출 도구 등록

## 15. 최종 컴포넌트 책임표

| 컴포넌트 | 상태 변경 | 원본 증거 쓰기 | 분석 쓰기 | 승인 쓰기 | 외부 모델/MCP |
|---|---:|---:|---:|---:|---:|
| Web | 아니오 | 업로드 요청만 | 아니오 | 사용자 명령만 | 아니오 |
| Core API | 예 | 메타데이터 등록 | 예 | 예 | 작업 요청만 |
| Worker | 아니오 | 아니오 | 결과 제출만 | 아니오 | 예 |
| LLM Gateway | 아니오 | 아니오 | 아니오 | 아니오 | 예 |
| MCP Broker | 아니오 | 아니오 | 아니오 | 아니오 | 예 |
| Evaluation Harness | 운영 아니오 | fixture만 | fixture만 | 아니오 | 예 |
| Object Store | 아니오 | 객체 저장 | 파생물 저장 | 아니오 | 아니오 |
| PostgreSQL | 영속화 | 메타데이터 | 예 | 예 | 아니오 |

## 16. 아키텍처 완료 판정

다음 결정은 Stage F 기준으로 FIX한다.

- Next.js Web + Spring Boot Core
- 모듈러 모놀리스 Core
- 별도 Worker Runtime
- PostgreSQL 시스템 오브 레코드
- S3 호환 Object Store
- DB task/outbox 기반 초기 비동기 실행
- 공급자 중립 LLM Gateway
- MCP Permission Broker
- Native Worker 우선, Hermes 선택 Adapter
- 독립 Evaluation Harness
- 사람 승인과 잠금의 Core 소유

다음은 구현·평가 후 HARDEN한다.

- 구체 모델과 공급자
- 외부 MCP 서버 선정
- 배포 클라우드
- RPO/RTO 실제 수치
- Object Lock 사용 여부
- 전용 Queue와 Workflow Engine 도입
- 자동 추적 주기

## 17. 구현 시작 전 남은 게이트

아키텍처 문서만으로 구현 승인을 의미하지 않는다.

필수 선행:

1. 이 PR의 사람 리뷰
2. P0/P1 위협모델 확인
3. Manual MVP 범위 승인
4. 데이터 보존·삭제 정책 결정
5. 외부 모델로 전송 가능한 데이터등급 승인
6. 첫 구현 작업패키지 승인

그 전까지 Stage G는 `PLANNED_NOT_STARTED` 상태다.
