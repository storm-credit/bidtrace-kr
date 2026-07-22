# PR 병합·Migration·Design Freeze Runbook

## 1. 목적

Draft PR을 안전하게 main에 통합하고 canonical v2 계약을 Design Freeze 기준선으로 만드는 절차다. 자동 merge와 main 직접 push는 사용하지 않는다.

## 2. 권장 병합순서

```text
#6  Foundation
#7  Rights Framework
#8  Market/Investment Framework
#10 Agent Operating Model
#12 Harness Deep Audit
#14 Workflow Registry v1 drafts
#15 Evaluation Fixtures v1 drafts
#16 Human Approval UX
#21 Final Architecture
#22 Design Freeze Review and Canonical Contracts
```

#7·#8은 Foundation에서 병렬 분기되었으므로 #6 다음에 별도로 병합한다.

## 3. #7·#8 충돌 제거 확인

### #7

정식 `evaluations/cases/CASE-001`과 `CASE-003` 경로를 더 이상 소유하지 않아야 한다.

허용 경로:

- `evaluations/domain-seeds/rights/general-apartment-seed.yaml`
- `evaluations/domain-seeds/rights/senior-tenant-suspected-seed.yaml`

### #8

정식 `CASE-020` manifest 경로를 더 이상 소유하지 않아야 한다.

허용 경로:

- `evaluations/domain-seeds/investment/missing-costs-unverified-loan-seed.yaml`

## 4. PR별 Retarget 절차

각 PR에서 반복한다.

```text
선행 PR main 병합
→ 다음 Draft PR base를 main으로 변경
→ changed files 확인
→ 예상 외 삭제·중복·재등장 확인
→ conflict 해결
→ 문서·schema 참조 수동검수
→ 사람 승인
→ 병합
```

한 번에 여러 PR을 main으로 retarget하지 않는다.

## 5. #22의 역할

#22는 v1 설계 산출물을 삭제하지 않는다. 다음을 canonical 기준으로 고정한다.

- Macro State + WorkflowStageRun
- canonical Workflow·Agent·Service·Gate ID
- Workflow·Approval enum
- AP-05/AP-06 분리
- AP-07 change control
- Workflow v1→v2 migration
- Evaluation Manifest v1→v2 migration
- Design Freeze blocker와 promotion gate

#22가 병합된 뒤 v1 YAML은 `DESIGN_DRAFT` 입력이며 운영 계약이 아니다.

## 6. Design Freeze 승인

모든 PR이 main에 병합된 후 다음을 확인한다.

- `architecture/canonical-contract-registry.yaml` 존재
- `architecture/system-manifest.yaml` v2
- ADR-007과 ADR-008 Accepted
- `evaluations/harness/design-freeze-blockers.yaml`의 open_document_design_blockers가 0
- #7·#8의 정식 CASE 경로 충돌 없음
- #15의 CASE-001~012만 canonical manifest
- #22 diff가 canonical contract 범위만 포함

사람 승인 레코드:

```yaml
approval_type: DESIGN_FREEZE
scope: Stage-A-through-F
approved_commit: main-sha
allowed_next_scope: G0-only
runtime_validation_complete: false
production_ready: false
```

## 7. Tag

승인 후에만 태그 후보를 만든다.

```text
design-freeze-v0.1
```

태그는 다음을 의미하지 않는다.

- 실제 모델 검증 완료
- MCP 보안검증 완료
- 법률·세무 정확성 보장
- 운영 또는 Production 준비 완료

## 8. G0 Branch

Design Freeze 승인 후 별도 branch에서 시작한다.

```text
agent/g0-contract-validators
```

허용 작업:

- repository skeleton
- canonical registry loader
- migration spec parser
- Workflow/Fixture v2 materializer
- identifier·enum·DAG·approval path validator
- CI checks

금지 작업:

- 실제 모델 호출
- Hermes 연결
- MCP 외부 도구 연결
- 자동수집
- 실제 입찰·송금·계약·법원제출

## 9. G0 완료조건

- 5개 Base Workflow v2 변환 성공
- Special Rights Overlay v2 변환 성공
- 12개 Evaluation Manifest v2 변환 성공
- unknown ID 0건
- prohibited control action 0건
- DAG cycle 0건
- AP-05 없는 BID_READY 경로 0건
- AP-06 없는 BID_LOCKED 경로 0건
- AP-07 직접 상태전환 0건
- duplicate canonical CASE manifest 0건
- CI 재실행 결과 결정론적

G0 완료 전 G1 Core 구현을 시작하지 않는다.
