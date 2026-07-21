# ADR-005: 모듈러 Core와 격리된 Worker Runtime

- 상태: Proposed for Stage F approval
- 날짜: 2026-07-21
- 결정자: PM Orchestrator 설계안, 사람 승인 대기

## Context

BidTrace KR은 사건 상태, 원본 증거, 분석 결과, 사람 승인과 모의입찰 잠금을 함께 관리한다. 동시에 여러 LLM 공급자, MCP 도구와 선택적 Hermes Runtime을 사용해야 한다.

LLM Worker에 상태와 데이터 소유권을 주면 다음 위험이 생긴다.

- 모델 오류가 사건 상태를 오염
- 승인 우회
- 사건 간 자료 혼합
- 잠긴 결과 수정
- Runtime 교체 불가능
- 감사 추적 단절

반대로 모든 기능을 하나의 프로세스에 넣으면 외부 모델·도구의 불신 경계가 Core 안으로 침투한다.

## Decision

### Core

Spring Boot 기반 모듈러 모놀리스로 시작한다.

Core는 다음을 유일하게 소유한다.

- 사건 상태와 Workflow 실행계획
- 증거 메타데이터와 해시
- 분석 버전
- 정책과 계산식 버전
- 사람 승인
- 모의입찰 잠금
- 감사로그

### Worker

Worker Runtime은 별도 프로세스 또는 컨테이너다.

Worker는 서명된 Agent Task Contract를 실행하고 구조화된 결과만 Core에 제출한다.

Worker는 DB와 Object Store에 직접 쓰지 않는다. 증거 읽기는 Core가 발급한 사건·작업 범위의 제한 권한으로만 허용한다.

### Adapter

Worker Runtime 인터페이스를 정의한다.

- Native Worker Adapter를 기준 구현으로 먼저 제작
- Hermes는 선택 Adapter
- Runtime별 conformance test 필수
- Runtime memory는 사실 저장소로 사용 금지

## Consequences

장점:

- Core 데이터 무결성
- Runtime 교체 가능
- Worker sandbox와 최소권한
- 모델 공급자 중립성
- 독립 평가 가능

비용:

- Core/Worker 계약 설계 필요
- 비동기 실행과 idempotency 필요
- 로그·trace 연계 필요
- 로컬 개발 구성 증가

## Rejected Alternatives

### 모든 기능을 Next.js에 구현

핵심 상태와 승인 규칙이 UI 배포와 결합되어 거부한다.

### Python agent framework가 시스템 전체 소유

LLM 실행 편의성은 높지만 상태·승인·감사 통제와 장기 유지보수 위험 때문에 거부한다.

### 초기 마이크로서비스

작은 팀과 Manual MVP에 과도하다. 분산 트랜잭션과 운영 복잡도를 정당화할 부하 근거가 없다.

### Hermes를 필수 Core Runtime으로 채택

선택 기술에 시스템 통제를 위임하므로 거부한다. Hermes는 Adapter 뒤의 Worker 옵션이다.

## Validation

- Worker가 DB 쓰기를 시도하면 거부
- 다른 case_id 증거 접근 거부
- approval API 호출 거부
- locked snapshot 수정 거부
- Native/Hermes 동일 fixture 결과 계약 검증
- Worker 장애가 Core 상태를 부분 전환하지 않는지 검증

## Revisit Triggers

- Core 모듈별 독립 배포 필요가 반복 발생
- 특정 모듈 부하가 전체 배포에 영향을 줌
- 프로세스 단위 보안격리가 추가로 필요
- Worker 계약이 Runtime 기능을 지나치게 제한
