# BidTrace KR 관찰성·신뢰성·백업·복구 설계

## 1. 목적

경매 분석 시스템은 단순 응답속도보다 다음을 우선한다.

- 어떤 자료와 모델이 결론을 만들었는지 추적
- 승인과 잠금이 우회되지 않았는지 확인
- 장기 작업 실패와 재시도를 복구
- 원본 증거와 분석 이력을 보존
- 장애 후 정확한 상태로 복원

## 2. 공통 추적 식별자

모든 요청과 작업은 가능한 범위에서 다음 ID를 공유한다.

```yaml
trace_id: string
request_id: string
case_id: UUID|null
execution_plan_id: UUID|null
task_id: UUID|null
attempt_id: UUID|null
model_run_id: UUID|null
decision_package_id: UUID|null
actor_id: string
```

민감정보와 원문 전체는 일반 로그에 기록하지 않는다.

## 3. 로그

### 3.1 구조화 로그

필수 필드:

- timestamp
- severity
- service
- environment
- trace_id
- actor_type
- action
- result
- reason_code
- duration_ms

### 3.2 보안 로그

- 인증 성공·실패
- 권한 거부
- 사건 간 접근 시도
- 승인과 잠금 시도
- 관리자 정책 변경
- 모델·MCP 활성화/중지
- 비밀 접근
- 파일 격리·차단

### 3.3 모델 실행 로그

- provider/model/profile
- prompt version
- task contract hash
- input evidence IDs
- tool call names와 결과코드
- token usage
- latency
- estimated cost
- validation outcome

원문 프롬프트·응답 보존 여부는 데이터등급 정책을 따른다.

## 4. 메트릭

### Core

- HTTP latency/error rate
- state transition success/denial
- stale propagation duration
- approval decision count
- bid lock success/denial
- DB connection and transaction health

### Worker

- queue depth
- ready/leased/running task
- lease timeout
- retry/dead-letter rate
- task duration
- model/provider failure rate
- tool denial rate
- token/cost usage

### Evidence

- upload count and size
- quarantine rejection rate
- hash duplicate rate
- derivative generation failure
- missing lineage count

### Evaluation

- P0/P1 count
- evidence linkage rate
- critical fact accuracy
- mandatory escalation rate
- calculation match rate
- regression trend

## 5. Trace

OpenTelemetry 호환 추적을 사용한다.

대표 span:

```text
http.request
case.command
workflow.compile
workflow.transition
outbox.publish
task.lease
worker.execute
llm.request
mcp.call
result.validate
analysis.persist
approval.decide
bidplan.lock
```

모델 공급자 응답에 trace ID를 직접 노출할 필요는 없으며 내부 correlation ID를 사용한다.

## 6. 알림

### 즉시 대응

- cross-case 접근 성공 또는 의심
- locked snapshot 무결성 실패
- approval hash 불일치
- audit write 실패
- credential leak 탐지
- 악성파일 격리 우회
- 실제 side-effect tool 호출 시도

### 업무시간 대응

- dead-letter 증가
- provider 오류율 상승
- 비용 이상
- backup 실패
- stale 분석 장기 미처리
- policy 만료 예정
- disk/object quota 임계치

알림에는 민감 원문을 포함하지 않는다.

## 7. 서비스 목표 초안

Manual MVP의 정확한 SLO는 부하측정 후 고정한다.

초기 목표 예시:

| 항목 | 목표 성격 |
|---|---|
| 사건 조회 | 일반 업무시간에 높은 가용성 |
| 증거 업로드 | 실패 시 재시도 가능 |
| 승인·잠금 | 강한 일관성, 실패 시 상태 미변경 |
| Agent 작업 | 비동기, 지연 허용, 중복결과 금지 |
| 감사로그 | 고위험 작업과 같은 트랜잭션 또는 실패 차단 |
| 평가 하네스 | 배포 전 반드시 완료 |

속도보다 정확성과 복구 가능성을 우선한다.

## 8. 오류 분류

```text
TRANSIENT
VALIDATION
POLICY_BLOCK
AUTHORIZATION
DATA_CONFLICT
DEPENDENCY
SECURITY
INTERNAL
```

재시도 가능:

- 일시 네트워크 오류
- provider rate limit
- lease 만료
- 일시 Object Store 오류

자동 재시도 금지:

- 권한 거부
- 필수 증거 부재
- 문서 간 실제 충돌
- policy block
- schema 반복 실패
- prompt injection 또는 보안 위반

## 9. Circuit Breaker

대상:

- LLM provider
- MCP server
- Object processing parser
- external public data source

동작:

- 연속 실패 또는 지연 임계치 초과 시 open
- 관련 task를 retry-later 또는 blocked 처리
- 다른 provider fallback은 데이터등급과 독립검토정책을 만족할 때만
- 자동 fallback으로 고위험 reviewer 독립성이 깨지지 않도록 검사

## 10. 백업

### PostgreSQL

- 자동 정기 백업
- point-in-time recovery 가능 구성 권장
- 백업 암호화
- 환경별 분리
- 복구 테스트

### Object Store

- versioning 또는 원본 overwrite 방지
- lifecycle와 보존정책
- 필요 시 교차 위치 복제 검토
- object inventory와 DB metadata 정합성 검사

### Configuration

- Workflow/Prompt/Schema는 Git
- 승인된 Model/Policy Registry는 DB export
- 비밀은 secret manager backup/rotation 정책

## 11. RPO/RTO 분류

실제 수치는 운영 승인 시 확정한다.

| 데이터 | RPO 우선도 | RTO 우선도 |
|---|---|---|
| 원본 증거 | 매우 높음 | 중간 |
| 승인·잠금 | 매우 높음 | 높음 |
| 사건 상태 | 매우 높음 | 높음 |
| 분석 결과 | 높음 | 중간 |
| 모델 실행 원문로그 | 정책에 따라 | 낮음 |
| 평가 결과 | 중간 | 낮음 |
| 캐시·파생 미리보기 | 낮음 | 낮음 |

## 12. 복구 시나리오

### DB 장애

```text
쓰기 중지
→ 장애시점 확인
→ 관리형 failover 또는 restore
→ outbox/task lease 정합성 검사
→ 승인·잠금 hash 검사
→ 제한적 재개
```

### Object Store 손실 또는 불일치

```text
증거 업로드 중지
→ DB inventory와 object inventory 대조
→ version/backup 복구
→ hash 재검증
→ 파생물 재생성
```

### Worker 오염

```text
Worker kill switch
→ credential revoke
→ 진행 task lease 만료/취소
→ 관련 결과 격리
→ clean image 재배포
→ 영향을 받은 task 재실행
```

### 모델 공급자 사고

```text
provider suspend
→ 외부 전송 중지
→ 영향 task/데이터 범위 식별
→ 공급자 정책에 따른 대응
→ 다른 approved provider 또는 수동절차
```

## 13. Restore Drill

운영 전, 이후 정기적으로 다음을 수행한다.

- 새 환경에 DB restore
- Object Store 원본 샘플 hash 검증
- 사건 하나의 evidence→analysis→approval 연결 확인
- locked snapshot hash 확인
- dead-letter와 outbox 재처리 확인
- 감사로그 chain 검사

복구 성공은 단순 서버 기동이 아니라 도메인 불변식 확인으로 판정한다.

## 14. 운영 Runbook 목록

- provider outage
- MCP outage
- task queue backlog
- dead-letter replay
- upload/parser failure
- malicious file
- cross-case incident
- approval/lock integrity failure
- database restore
- object restore
- policy expiration
- abnormal model cost

## 15. Postmortem

P0/P1 또는 사용자 의사결정에 영향을 준 장애는 사후분석을 작성한다.

```yaml
incident_id: string
impact: string
timeline: []
root_causes: []
contributing_factors: []
detection_gap: []
recovery: []
actions: []
fixture_updates: []
policy_updates: []
owners: []
due_dates: []
```

사후분석은 개인 비난보다 시스템 통제와 하네스 개선에 집중한다.
