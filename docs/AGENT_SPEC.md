# CareFlow Lite — AI Agent 명세

- 문서 버전: 0.2 (초안) · 작성일: 2026-10-22 · 개정: 2026-10-23
- Agent는 챗봇이 아니라 **업무 자동화 실행 단위**다. 6개 Agent는 6개의 LLM이 아니며, 하나의 PHP 애플리케이션 안에서 모듈로 분리되고 필요할 때만 같은 LLM API를 공유한다.

> **2026-10-23 범위 결정 — 외부 LLM 호출 보류.** 개인정보 전송 검토(Q-LEGAL-01)를 이번 범위에서 제외했다. 따라서 모든 Agent는 `llm_required=false`로 동작하는 **규칙·템플릿 모드**만 구현한다. 아래 §1.5~1.6(LLM 정책, 가명처리)과 각 Agent의 "LLM 모드" 항목은 재개 시를 위한 설계로 남겨 두며, 지금은 구현하지 않는다. `AgentRunner`의 LLM 단계는 `ai.enabled=false`면 항상 건너뛴다.

---

## 1. 공통 실행 엔진

### 1.1 실행 절차

```
이벤트 발생 (cron / 데이터 변경 / 수동)
  → agent_runs INSERT (status=queued, idempotency_key UNIQUE — 중복이면 기존 run 반환)
  → worker가 잠금 획득 (status=running)
  → PermissionGate: permission_scope 확인
  → 입력 수집 (읽기 전용 Service) + input_schema 검증  ── 부족 → blocked + missing_inputs
  → rule_checks 실행 (PHP 규칙 엔진)
  → LLM 필요 판단 (llm_required + 규칙 결과 + 예산)
       └ 필요 시: Redactor(가명처리) → 캐시 조회(input_hash+prompt_version)
                 → BudgetGuard → LlmClient(타임아웃) → JSON 파싱
  → output_schema 검증 (실패 → 재시도 1회 → invalid_output, needs_review 또는 failed)
  → 근거 검증 (AI가 인용한 record_id가 실제 입력 집합에 있는지)
  → 트랜잭션: findings/drafts/tasks 저장 + audit_logs
  → approval_required면 agent_approvals(pending) 생성 + 알림
  → status = completed / needs_review / blocked / failed
```

### 1.2 Agent 인터페이스 (PHP)

```php
interface AgentInterface
{
    public function name(): string;                 // agent_name
    public function triggers(): array;              // trigger
    public function inputSchema(): array;           // input_schema (JSON Schema 배열)
    public function permissionScope(): PermissionScope;
    public function ruleChecks(): array;            // 규칙 ID 목록
    public function llmRequired(AgentContext $ctx): bool;
    public function outputSchema(): array;
    public function approvalRequired(): bool;
    public function retryPolicy(): RetryPolicy;
    public function idempotencyKey(AgentRequest $req): string;
    public function auditPolicy(): AuditPolicy;
    public function execute(AgentContext $ctx): AgentResult;  // 저장은 Runner가 수행
}
```

Agent는 DB에 직접 쓰지 않는다. `AgentResult`를 반환하면 `AgentRunner`가 검증 후 저장한다.

### 1.3 상태

`queued → running → completed`
추가: `needs_review`(사람 확인 필요), `blocked`(필수 자료 부족·정책상 불가), `failed`(실패), `cancelled`(취소).

| 전이 | 조건 |
|---|---|
| queued → cancelled | 관리자 취소 |
| running → blocked | 필수 입력 누락, 예산 초과(LLM 필수 Agent), 미검증 규칙만 존재 |
| running → failed | 예외, 타임아웃, 재시도 소진 |
| failed → queued | 수동 재시도 (`attempts < max_attempts`) |
| running → needs_review | 결과에 사람 확인 항목 또는 승인 요청 존재 |

`running` 상태로 `locked_at + 10분`이 지나면 worker가 고아 실행으로 보고 `failed(STALE)`로 바꾼다.

### 1.4 공통 출력 스키마

```json
{
  "agent_name": "claim_validation",
  "status": "needs_review",
  "summary": "확인할 항목이 있습니다.",
  "findings": [
    {
      "type": "string",
      "severity": "error|warning|info|needs_confirmation",
      "source": "rule_engine|ai",
      "subject": { "type": "client|employee", "id": 0 },
      "related_ids": [0],
      "rule_version_id": null,
      "message": "string",
      "action": "string"
    }
  ],
  "drafts": [ { "doc_type": "string", "document_id": 0, "version": 0 } ],
  "missing_inputs": [ { "field": "string", "reason": "string" } ],
  "rule_references": [ { "rule_id": "string", "version": 0, "status": "verified|unverified" } ],
  "requires_approval": true
}
```

- 위 JSON은 형식 예시이며 실제 데이터·판정 결과가 아니다.
- 검증: `Validators/JsonSchemaValidator`(경량 자체 구현 또는 승인된 라이브러리)로 필수 필드, 타입, 허용값, 문자열 최대 길이를 검사한다.
- `source=ai` finding의 `severity`는 `error`가 될 수 없다. 오류 판정은 규칙 엔진만 한다.

### 1.5 LLM 호출 공통 정책 (보류 — 재개 시 적용)

| 항목 | 기본값(안) | 설정 키 |
|---|---|---|
| 타임아웃 | 60초 | `ai.timeout_sec` |
| 재시도 | 네트워크 오류 2회(지수 백오프), 스키마 오류 1회 | `ai.max_retries` |
| 최대 입력 | 8,000 토큰 | `ai.max_input_tokens` |
| 최대 출력 | 2,000 토큰 | `ai.max_output_tokens` |
| 월 예산 | 10,000원 (목표, 보장 아님) | `ai.monthly_budget_krw` |
| 일 호출 수 | 50회 | `ai.daily_call_limit` |
| 캐시 | `input_hash + prompt_id + prompt_version + model` 일치 시 재사용 | |
| 모델 | 비용 효율 모델 기본, 문서 초안만 상위 모델 선택 가능 | `ai.model.*` |

- 프롬프트는 `prompts/{prompt_id}/v{n}.md`에 버전 관리한다. 결과에 `prompt_version`과 생성 시각을 기록한다.
- 시스템 프롬프트 공통 지시: "입력에 없는 사실을 만들지 말 것, 모든 주장에 근거 record_id를 붙일 것, 근거가 없으면 `insufficient_evidence`로 표시할 것, 계산하지 말 것, 진단하지 말 것, 지정된 JSON만 출력할 것."
- 입력 데이터(서비스 기록 텍스트 등)는 프롬프트 안에서 명확한 구분자로 감싸 **데이터로만** 취급하게 하고, 그 안의 지시문은 따르지 않도록 지시한다(프롬프트 인젝션 대응).

### 1.6 가명처리 (Redactor) (보류 — 재개 시 적용)

LLM 전송 전 다음을 수행한다. 상세는 [SECURITY §6](SECURITY.md).

1. 이름 → `수급자A`, `요양보호사1`, `보호자1` 같은 실행 단위 임시 토큰 (매핑표는 메모리에만, 저장 안 함)
2. 주민번호·인정번호·전화번호·주소·이메일 → 제거
3. 생년월일 → 연령대
4. 날짜 → 기간 시작일 기준 상대 일자(`D+3`)
5. 정규식 2차 검사로 남은 식별 패턴이 있으면 **전송 중단** → `blocked(PII_DETECTED)`

응답의 임시 토큰은 PHP에서 원래 이름으로 되돌려 화면에 표시한다.

### 1.7 권한 레벨

| 레벨 | 허용 | 정책 |
|---|---|---|
| L0 | 조회·요약 | 제한된 데이터만 |
| L1 | 검증 결과·업무·알림 생성 | 자동 저장 |
| L2 | 문서·문자 초안 생성 | 초안으로만 저장 |
| L3 | 내부 데이터 수정 | 사용자 명시 승인 필요 |
| L4 | 외부 발송·공식 제출·확정 | 명시 승인 + 별도 검증(재인증) |

Agent는 L0~L2에서만 동작한다. L3 이상은 Agent가 `agent_approvals`에 **요청**만 만들고, 실행은 사람이 승인한 뒤 해당 PHP Service가 권한을 다시 검사하고 수행한다. Agent 코드에는 쓰기용 Repository와 원시 PDO가 주입되지 않는다.

### 1.8 공통 실패 처리

| 실패 | 처리 |
|---|---|
| LLM 타임아웃/네트워크 | 재시도 후 `failed(AI_UNAVAILABLE)`. 규칙 결과는 저장하고 AI 부분만 비움 |
| 잘못된 JSON | 1회 재요청, 다시 실패하면 `needs_review(AI_INVALID_OUTPUT)`, 원문은 저장하지 않음 |
| 근거 없는 인용 | 해당 항목 폐기 + `review_flags`에 기록 |
| 예산 초과 | LLM 단계 건너뜀 → 규칙 결과만으로 `completed` 또는 `blocked(AI_BUDGET_EXCEEDED)` |
| 필수 입력 부족 | `blocked` + `missing_inputs` + 운영자 업무 생성 |
| 중복 실행 | 같은 idempotency_key는 새 실행을 만들지 않음 |
| PII 탐지 | 전송 중단, `blocked(PII_DETECTED)` |

---

## 2. Agent별 명세

표기: **LLM** = LLM 호출 여부, **승인** = 결과 확정에 사람 승인 필요 여부.

### Agent 1. Daily Operations Agent (`daily_operations`)

| 항목 | 내용 |
|---|---|
| 목적 | 매일 놓치기 쉬운 업무를 찾아 우선순위 목록 생성 |
| trigger | cron 매일 06:00 / 대시보드 열기(당일 결과 없거나 6시간 경과 시) / 일정 변경·기록 누락·신규 상담 이벤트 / 마감 임박 |
| input_schema | `{ date: DATE }` + 읽기: 당일·전일 일정, 기록, 태그 매칭 결과, 열린 업무, deadlines, 자격 만료, 상담 후속일, 월말 정산 현황 |
| permission_scope | L0(조회), L1(findings·tasks·notifications 생성) |
| rule_checks | `SCHED.MISSING_RECORD`, `REC.ENTRY_LATE`, `TAG.TIME_MISMATCH`, `TASK.OVERDUE`, `DEADLINE.UPCOMING`, `STAFF.CREDENTIAL_EXPIRY`, `CONSULT.FOLLOWUP_DUE`, `BILL.MONTH_END_READINESS` |
| llm_required | **아니오(P0)**. P1에서 선택: 브리핑 문장 다듬기 (집계 수치·우선순위는 규칙 결과 그대로) |
| output_schema | 공통 출력 + `items[]`(type, priority, entity_id, message, action) |
| approval_required | 아니오 (업무 생성만) |
| retry_policy | 규칙 단계 실패 시 최대 2회. LLM 실패 시 규칙 결과로 완료 |
| idempotency_key | `daily_operations:{date}:{trigger}:{event_id?}` |
| audit_policy | 실행 메타데이터, 생성한 task id 목록 |
| 제약 | "기록 없음" ≠ "미제공". 원인 단정 금지, 확인 업무만 생성. 업무 자동 완료 금지 |
| 우선순위 | 규칙 데이터(`settings.priority_rules`)로 결정. AI는 순서를 바꾸지 않음 |

**테스트 기준**: T-AGT-01-01 일정 있음·기록 없음 → `missing_record` 업무 1건 / T-AGT-01-02 같은 날 두 번 실행 → 업무 중복 없음 / T-AGT-01-03 LLM 비활성 상태에서 브리핑 생성 / T-AGT-01-04 메시지에 "미제공" 같은 단정 표현 없음.

### Agent 2. Claim Validation Agent (`claim_validation`)

| 항목 | 내용 |
|---|---|
| 목적 | 월 청구 전 오류·누락 최대한 탐지 |
| trigger | 수동(월말 화면) / 태그 배치 반영 직후 / 재검증 요청 |
| input_schema | `{ period_ym, scope: all\|client_ids[], import_batch_id? }` + 읽기: 일정, 기록, 태그, 계약·대상자, 문서 상태, 적용 규칙 버전 |
| permission_scope | L0, L1(billing_checks, tasks) |
| 처리 순서 | ① 원본 파일 확인 ② 형식·필수 열 ③ 형식·중복 ④ 수급자·직원 매핑 ⑤ 일정↔기록 ⑥ 태그↔기록 ⑦ 규칙 엔진 조건 ⑧ 오류/경고/정보/확인 필요 분류 ⑨ 검토 목록 ⑩ 수정 후 재검증 |
| rule_checks | `REC.MISSING`, `REC.DUPLICATE`, `REC.TIME_MISMATCH`, `TAG.RECORD_MISMATCH`, `MAP.CLIENT_FAILED`, `MAP.EMPLOYEE_FAILED`, `DOC.REQUIRED_MISSING`, `RULE.VERSION_UNVERIFIED`, `SUM.MONTHLY_MISMATCH` + verified 상태의 청구 규칙 |
| llm_required | 기본 아니오. 선택: 경고 항목의 쉬운 설명 문장 생성(L0, 개수 제한) |
| output_schema | 공통 출력. finding 필드: 대상 식별자, check_code, 문제 유형, 관련 기록 ID, 규칙 ID·버전, 발생 일시, 설명, 권장 절차, 처리 상태, 검토자·검토 시각 |
| approval_required | 검증 자체는 아니오. 청구 자료 확정(L4)은 별도 승인 |
| retry_policy | 2회 |
| idempotency_key | `claim_validation:{period_ym}:{scope_hash}:{data_version}` (data_version = 관련 테이블 최종 수정 시각 해시 → 데이터 변경 시 재실행 허용) |
| audit_policy | 실행, 결과 건수(심각도별), 적용 규칙 버전 |
| 제약 | 금액 산출 금지, 청구 가능 여부 확정 금지, 자동 제출 금지, 원본 자동 수정 금지. 미검증 규칙 관련 항목은 `needs_confirmation`. 결과는 공식 심사 결과를 보장하지 않음(화면에 고지) |

**테스트 기준**: T-AGT-02-01 중복 기록 → `REC.DUPLICATE` error / 02-02 미검증 규칙 → needs_confirmation만 / 02-03 실행 후 service_records 변경 0건 / 02-04 같은 데이터 재실행 → 결과 중복 없음 / 02-05 기록 수정 후 재실행 → 해결 항목 resolved.

### Agent 3. Care Plan Agent (`care_plan`)

| 항목 | 내용 |
|---|---|
| 목적 | 욕구사정 기반 급여제공계획서 초안 |
| trigger | 욕구사정 final 저장 / 욕구사정 변경 / 수동 버튼 |
| input_schema | `{ client_id, assessment_id }` + 읽기: 욕구사정(구조화·서술·목표), 계약·급여 종류, 기존 승인 계획서, 확인된 양식 정의 |
| permission_scope | L0, L2(document_versions draft) |
| rule_checks | `CAREPLAN.REQUIRED_INPUTS`(필수 입력 존재), `CAREPLAN.FORM_FIELDS`(양식 필수 항목 — 공식 양식 확인 후 등록, Q-REG-07) |
| llm_required | **아니오 (템플릿 모드)**. LLM 모드는 보류 |
| 처리 (템플릿 모드) | ① 필수 입력 확인 → 부족하면 blocked + 요청 업무 ② 욕구사정 구조화 항목을 계획서 섹션에 매핑(매핑표는 데이터) ③ 수급자 목표·희망 사항은 원문 그대로 인용 ④ 지원 필요 영역 → 4개 서비스 영역(신체활동·인지관리·일상생활·정서지원, [FORMS §1.2](FORMS.md)) 후보 제시 ⑤ 섹션별 근거(assessment 필드) 연결 ⑥ 매핑 안 된 항목·빈 섹션에 review_flag ⑦ draft 저장 → review_required. 서술 문장은 담당자가 작성 |
| 처리 (LLM 모드, 보류) | 위 ②~④에서 자연어 요약·초안 문장 생성 |
| output_schema | 공통 출력 + `plan.sections[]: { key, text, evidence: [{source: "assessment", id, field}], review_flag?: string }` |
| approval_required | **예** (최종 확정, 서명은 사람이 오프라인 수행) |
| retry_policy | LLM 2회, 스키마 1회 |
| idempotency_key | `care_plan:{assessment_id}:{assessment_version}:{prompt_version}` |
| audit_policy | 실행, ai_run_id, 생성 문서 버전 |
| 제약 | 템플릿 모드는 입력 원문 외 문장을 만들지 않는다. 자료에 없는 진단·기능 상태·가족 상황 생성 금지. 근거 없는 섹션은 review_flag. 목표↔서비스 연결 불명확 시 확인 요청. 양식 필수 항목은 PHP 검증기가 검사 |
| 문서 상태 | `draft → review_required → approved`. 승인 후 변경 시 새 버전 + 이력 |

**테스트 기준**: 03-01 충분한 입력 → 모든 섹션에 evidence 존재 / 03-02 필수 입력 누락 → blocked / 03-03 매핑되지 않은 욕구사정 항목 → review_flag / 03-04 승인 문서 수정 → 새 버전 생성. (LLM 모드 테스트는 재개 시 추가)

### Agent 4. Monthly Condition Report Agent (`condition_report`)

| 항목 | 내용 |
|---|---|
| 목적 | 일별 관찰값과 주간 상태변화기록을 취합해 월간 상태변화보고서 초안 생성 ([FORMS §2](FORMS.md)) |
| trigger | 매월 지정일 cron / 월말 화면 수동 / 특정 수급자 선택 실행 |
| input_schema | `{ client_id, period_ym }` + 읽기: 해당 월 서비스 기록의 관찰값(4개 점검 항목), 주간 상태변화기록(`condition_observations`), 직전 월 승인 보고서 |
| permission_scope | L0, L2 |
| rule_checks | `REC.MISSING`, `REC.DUPLICATE`(기간 내), `REPORT.MIN_RECORDS`(기록 수 최소값, 설정값), `REPORT.WEEKLY_MISSING`(주간 기록 누락 주) |
| llm_required | **아니오 (템플릿 모드)**. LLM 모드는 보류 |
| 처리 (템플릿 모드) | ① 기간 기록·주간 기록 수집 ② 중복·누락 확인 ③ 점검 항목별 지표 집계 (예: 식사량 '감소' 기록 횟수, 통증 호소 횟수, 실금 횟수, 수면 '불량' 횟수, 체중 측정값 첫·끝) ④ 직전 월 지표와 비교 → 변화 판정은 설정된 임계값 규칙으로만 ⑤ 지표마다 근거 기록 ID 연결 ⑥ 주간 기록의 '조치 사항 및 특이사항'은 원문 그대로 나열 ⑦ 초안 생성 ⑧ 원본 기록 링크 |
| 처리 (LLM 모드, 보류) | 집계 결과와 주간 서술을 자연어 요약 문장으로 정리 |
| output_schema | 공통 출력 + `report.items[]: { domain: physical_health\|meal_medication\|cognition_emotion\|elimination, metrics: [{key, value, prev_value}], observation: string, interpretation?: string, change: improved\|worsened\|unchanged\|not_comparable\|insufficient_evidence, actions_notes: [string], evidence_record_ids: [] }` |
| approval_required | 예 |
| idempotency_key | `condition_report:{client_id}:{period_ym}:{records_hash}:{prompt_version}` |
| 제약 | 기록에 없는 건강 상태·사건 생성 금지. `observation`(사실)과 `interpretation`(해석) 필드 분리 — 템플릿 모드에서 `interpretation`은 비워 두고 담당자가 작성. 이전 보고서 없으면 `not_comparable`. 변화 임계값은 센터 설정값이며 의학적 기준이 아님을 화면에 표시. 의료 진단 금지 |

**테스트 기준**: 04-01 이전 보고서 없음 → 모든 change가 not_comparable / 04-02 evidence 없는 change=worsened → insufficient_evidence로 강등 / 04-03 기록 수 부족 → blocked / 04-04 식사량 감소 횟수가 임계값 이상 증가 → worsened + 근거 기록 ID / 04-05 주간 기록 누락 주 → finding.

### Agent 5. Billing & Copayment Agent (`billing_copayment`)

| 항목 | 내용 |
|---|---|
| 목적 | 월별 정산 작업 조율과 안내 자료 생성 |
| trigger | 월말 화면 수동 / monthly cron(계산 준비까지만) |
| input_schema | `{ period_ym, client_ids? }` |
| permission_scope | L0, L1, L2(안내 문구 초안). 계산은 `CopaymentCalculator` Service 호출(결정론) |
| 처리 | ① 대상자 조회 ② 원본 자료·계산 조건 확인 ③ 계산 엔진 실행 ④ 이전 결과와 차이 표시 ⑤ 누락 시 held ⑥ 수급자별 정산 내역 ⑦ 보호자 안내 문구 초안 ⑧ 승인 후 발송/내보내기 |
| rule_checks | `BILL.INPUT_COMPLETE`, `BILL.RULE_VERIFIED`, `BILL.DIFF_THRESHOLD` |
| llm_required | 아니오. 안내 문구는 템플릿으로 생성 (AI 다듬기는 보류) |
| output_schema | 공통 출력 + `calculations[]: {client_id, calculation_id, status, hold_reasons[]}` + `notice_drafts[]` |
| approval_required | 예 (정산 확정 L3, 외부 발송 L4) |
| idempotency_key | `billing:{period_ym}:{client_id}:{input_hash}:{engine_version}` |
| 제약 | AI 금액 계산·수정 금지. 미검증 규칙이면 확정 금액으로 표시 안 함. 청구액·수납액·미수금 분리. 승인 전 외부 발송 금지 |

**테스트 기준**: 05-01 안내 초안의 금액이 계산 결과와 일치 / 05-02 미검증 규칙 → held / 05-03 승인 전 발송 시도 → 423.

### Agent 6. Staff & Compliance Agent (`staff_compliance`)

| 항목 | 내용 |
|---|---|
| 목적 | 직원 자격·필수 문서·마감일 지속 관리 |
| trigger | cron 매일 / 직원 정보·자격증·서류 변경 / 마감일 등록·변경 |
| input_schema | `{ date }` 또는 `{ employee_id }` |
| permission_scope | L0, L1 |
| rule_checks | `STAFF.CREDENTIAL_EXPIRY`, `STAFF.REQUIRED_DOCS`(필수 문서 목록은 확인 후 규칙 등록), `STAFF.MONTHLY_HOURS`(집계만), `SCHED.EMPLOYEE_CONFLICT`, `STAFF.TRAINING_DUE`, `CONTRACT.RENEWAL_DUE` |
| llm_required | 아니오 |
| output_schema | 공통 출력. 판정값 `ok / expiring / expired / missing / undeterminable` |
| approval_required | 아니오 |
| idempotency_key | `staff_compliance:{date}` / `staff_compliance:emp:{employee_id}:{updated_at}` |
| 제약 | 자료 부족 시 `undeterminable`(판정 불가). 근무시간·방문 횟수를 법적 인정 기준과 같다고 표시하지 않음("집계값" 라벨). 법령 변경 여부 자동 확정 금지 |

**테스트 기준**: 06-01 만료 D-30 → expiring 업무 / 06-02 만료일 없는 자격 → undeterminable 아님(만료 없음 처리), 자격 기록 자체 없음 → missing / 06-03 같은 날 재실행 중복 없음.

### 보조 작업: Consultation Summary (`consultation_summary`, 보류)

시나리오 A의 상담 요약. LLM이 필요하므로 보류한다. 재개 시 L2, 승인 불필요(초안으로만 저장), 6개 Agent와 같은 실행 엔진 사용.

---

## 3. 승인 절차

1. Agent 또는 사용자가 `agent_approvals(pending)` 생성
2. 운영자 승인 화면: 원본 자료, 계산 근거, 규칙 버전, AI 초안, 변경 diff 동시 표시
3. 승인 시 `ApprovalWorkflow`가 권한 재검사 → 대상 Service 실행(트랜잭션) → `executed`
4. L4는 비밀번호 재입력 후 실행
5. 대상 데이터가 승인 요청 이후 바뀌었으면(버전 불일치) 승인 거부, 재요청 필요

반드시 사람이 승인: 계획서 확정, 상태변화 문서 확정, 정산 확정, 보호자 메시지 발송, 기록 정정·삭제, 청구 자료 확정·제출, 규정 해석이 필요한 예외 처리.

## 4. 공통 엔진 테스트 기준

현재 범위: T-AGT-C-01 중복 idempotency_key / C-02 출력 스키마 위반 거부 / C-09 Agent의 L3 직접 실행 불가 / C-10 고아 running 정리 / C-13 `ai.enabled=false`에서 LLM 단계 미호출.
보류(LLM 재개 시): C-03 타임아웃 / C-04 재시도 상한 / C-05 예산 초과 / C-06 캐시 / C-07 PII 차단 / C-08 근거 ID 위조 / C-11 프롬프트 인젝션 / C-12 AI severity=error 거부. 세부는 [TEST_PLAN](TEST_PLAN.md).
