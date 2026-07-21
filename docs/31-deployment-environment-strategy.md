# BidTrace KR 배포·환경 전략

## 1. 목적

Manual MVP를 과도한 인프라 없이 안전하게 개발·검증·배포하기 위한 환경 구성을 정의한다.

구체 클라우드 공급자와 제품 버전은 구현 시작 시점에 보안·비용·운영성 검토 후 고정한다.

## 2. 환경 구분

```text
LOCAL
→ CI
→ SHARED_TEST
→ STAGING
→ MANUAL_MVP_PRODUCTION
```

환경 간 데이터와 자격증명을 공유하지 않는다.

### 2.1 LOCAL

목적:

- 개발
- 정적 validator
- 합성 fixture 하네스
- UI 개발
- 선택적 로컬 모델 실험

구성:

- Windows + WSL2
- Docker Compose
- Next.js Web
- Spring Boot Core
- Native Worker
- PostgreSQL
- S3-compatible local object store
- mock OIDC 또는 개발용 identity provider
- mock mail/notification

원칙:

- 운영 원문 반입 금지
- 개발용 비밀은 운영과 분리
- Docker volume 초기화 절차 제공
- 외부 모델 호출은 명시적 profile에서만 활성화

### 2.2 CI

목적:

- 빌드
- 단위·통합 테스트
- schema/workflow/fixture 정적검증
- 보안·dependency 검사
- 합성 회귀평가

CI는 기본적으로 외부 모델을 호출하지 않는다.

외부 모델 비교는 승인된 별도 평가 workflow에서 수동 또는 제한적으로 실행한다.

### 2.3 SHARED_TEST

목적:

- 여러 개발자의 통합검증
- UI/API/Worker 연동
- 합성·익명화 fixture
- 장애주입과 권한시험

제약:

- 실제 개인정보 기본 금지
- 테스트 사용자와 역할 분리
- 운영 API key 사용 금지
- 비용 quota 설정

### 2.4 STAGING

목적:

- 운영과 유사한 배포·권한·백업 검증
- 제한된 익명화 실제문서 검증
- 사용자 승인 UX 테스트
- restore drill

운영과 분리된 DB, Object Store, OIDC client, model key를 사용한다.

### 2.5 MANUAL_MVP_PRODUCTION

범위:

- 수동 사건 등록
- 문서 업로드
- 증거 인덱스
- 수동 및 에이전트 보조 분석
- 사람 승인
- 모의입찰 잠금
- 실제 결과 수동 입력
- 사후복기

제외:

- 자동 법원 크롤링
- 실제 자동입찰
- 송금·대출 신청
- 계약 또는 법원문서 제출
- 비인가 개인정보 수집

## 3. 배포 토폴로지

Manual MVP 권장:

```text
Internet
  ↓
TLS / Reverse Proxy or Managed Ingress
  ↓
Web Container
  ↓ private network
Core API Container
  ├─ Managed PostgreSQL
  ├─ Managed Object Store
  └─ Worker Container
         └─ Approved External Models/MCP
```

선택:

- Web과 Core는 같은 VM의 분리 컨테이너 또는 관리형 컨테이너
- Worker는 별도 프로세스·컨테이너
- DB와 Object Store는 관리형 서비스 우선

초기 Kubernetes는 사용하지 않는다.

## 4. 네트워크 정책

### Web

- 외부 inbound 허용
- Core API로만 outbound
- DB/Object Store 직접접근 금지

### Core

- Web/Admin/Worker의 인증된 요청만 허용
- DB, Object Store, OIDC, secret manager 접근
- 외부 모델 직접 호출 금지 또는 정책상 제한

### Worker

- inbound는 Core/dispatcher만
- 외부 egress는 LLM Gateway와 승인 MCP allowlist
- DB 직접접근 금지
- 일반 인터넷 접근은 Research Broker를 통해서만 허용

### Evaluation Harness

- 운영 네트워크와 분리
- fixture store만 기본 접근
- 운영 API write 금지

## 5. 설정 관리

설정 분류:

- 소스관리 가능 설정
- 환경별 일반 설정
- 비밀정보
- 운영 정책

예:

```text
config/
  application-defaults
  workflow-registry
  prompt-registry
  model-profile-defaults

secret manager
  database
  object-store
  oidc
  llm-provider
  mcp-provider
  signing-key
```

운영 정책은 코드와 분리하지만 승인·버전·감사 없이 변경하지 않는다.

## 6. Artifact 전략

- Web/Core/Worker 각각 immutable container image
- commit SHA와 build ID 태그
- SBOM 생성
- 취약점 검사
- 서명 또는 provenance 기록
- 동일 artifact를 Staging에서 Production으로 승격
- 운영 서버에서 직접 빌드 금지

## 7. Database Migration

원칙:

- migration 파일은 append-only
- destructive migration은 별도 승인
- expand/contract 방식 우선
- migration 전 백업과 rollback 또는 forward-fix 계획
- locked/approval/audit schema 변경은 추가 검토

배포 순서 예:

```text
호환 가능한 DB expand
→ Core 배포
→ Worker 배포
→ 데이터 backfill
→ 검증
→ 후속 contract migration
```

## 8. Object Store 배포

버킷 또는 prefix 분리:

- quarantine
- original
- derivative
- report
- audit-export
- evaluation

정책:

- public access 차단
- server-side encryption
- lifecycle policy
- 원본 overwrite 차단
- 접근로그
- presigned URL 짧은 만료

## 9. Feature Flag와 Kill Switch

Feature Flag 후보:

- 외부 LLM provider
- Hermes Runtime
- Research MCP
- 특정 Agent profile
- 자동 추적 scheduler
- 외부 전문가 portal

Kill Switch:

- 모든 Worker 실행 중지
- 특정 provider 중지
- 특정 MCP 중지
- 파일 처리 중지
- 신규 승인·잠금 중지

Kill Switch는 기존 데이터 읽기를 가능한 한 유지한다.

## 10. 배포 게이트

Production 배포 전 필수:

1. 빌드·단위·통합 테스트 통과
2. schema/workflow/fixture validator 통과
3. P0/P1 회귀 0건
4. dependency/secret scan 통과
5. migration dry run
6. backup 확인
7. Staging smoke test
8. approval/lock 무결성 테스트
9. 변경 승인

## 11. Rollback

애플리케이션:

- 이전 immutable image로 rollback
- DB schema가 역호환되는 범위에서 수행

DB:

- 일반 rollback migration보다 forward-fix 우선
- 데이터손상 시 point-in-time restore 검토

Worker/Model:

- Model Registry status를 suspended로 전환
- 이전 approved model/profile로 복귀
- 진행 중 task는 재시도정책에 따라 처리

Workflow/Prompt:

- 새 version 비활성화
- 기존 사건의 active execution plan은 명시적 invalidation 전 유지

## 12. 비용 통제

- 환경별 예산
- 모델별 월/일 quota
- task token/tool budget
- 비정상 비용 알림
- 합성평가 외부 호출 빈도 제한
- 사용하지 않는 Staging 자동 축소 검토

비용절감을 이유로 보안로그·백업·승인검사를 비활성화하지 않는다.

## 13. 구현 전 결정 필요

- 배포 공급자
- OIDC 공급자
- 관리형 PostgreSQL/Object Store 선택
- secret manager 선택
- 도메인과 TLS 운영
- 실제 사용자 수와 접근지역
- 외부 모델 데이터 처리조건
