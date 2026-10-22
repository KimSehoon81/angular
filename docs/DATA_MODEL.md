# CareFlow Lite — 데이터 모델

- 문서 버전: 0.1 (초안) · 작성일: 2026-10-22
- 기존 스키마 없음 → 신규 설계. 실제 DDL은 구현 단계에서 `database/migrations/`로 작성한다.
- 공통: InnoDB, `utf8mb4` / `utf8mb4_unicode_ci`, 기본키 `BIGINT UNSIGNED AUTO_INCREMENT`, 시각은 `DATETIME`(세션 타임존 `+09:00` 고정), 날짜는 `DATE`.

## 1. 공통 규칙

| 규칙 | 내용 |
|---|---|
| 공통 컬럼 | `created_at`, `updated_at`, `created_by`, `updated_by` (users.id) |
| 삭제 | 업무 데이터는 물리 삭제 대신 `deleted_at` (보유기간 경과 후 파기 배치에서만 물리 삭제) |
| 금액 | `INT` 또는 `BIGINT`, 원 단위 정수 |
| 비율 | `INT` basis point (예: 1.5% → 150). 부동소수점 금지 |
| 상태값 | `VARCHAR(32)` + 애플리케이션 상수 검증 (ENUM은 마이그레이션 부담으로 지양) |
| 개인정보 | 고유식별정보는 `VARBINARY` 암호문 + 검색용 HMAC 컬럼 |
| 버전 | 문서·규칙은 행 단위 버전, 확정본은 수정 불가 |
| 원본/정규화 분리 | 업로드 원본은 파일 + `import_files`, 파싱 결과는 정규화 테이블 |
| 외래키 | 업무 무결성에 필요한 곳은 FK 설정, 감사·로그 테이블은 FK 없이 ID만 저장 |

## 2. ERD 개요

```
users ─┬─< audit_logs
       └─< tasks(assignee)

clients ─< client_guardians >─ guardians
   │
   ├─< service_contracts
   ├─< consultations
   ├─< assessments ─< care_plans ─< document_versions
   ├─< service_schedules ─< service_records ─< record_revisions
   │          ▲                    ▲
   │          └──── tag_records ───┘ (매칭 결과)
   ├─< copayment_calculations ─< copayment_calc_items
   ├─< payment_records
   └─< report_drafts

employees ─< employee_credentials
    └─< service_schedules

import_files ─< tag_import_batches ─< tag_records

rule_versions ─< rule_test_cases
billing_rules (rule_versions의 정산용 파라미터)

agent_runs ─< agent_findings
    ├─< ai_runs
    └─< agent_approvals
tasks ─< task_events
notifications
settings, job_locks, schema_migrations, login_attempts
```

## 3. 테이블 정의

### 3.1 인증·시스템

**users**
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | BIGINT PK | |
| username | VARCHAR(64) UNIQUE | |
| password_hash | VARCHAR(255) | `password_hash()` 결과 |
| role | VARCHAR(32) | admin / staff / caregiver |
| employee_id | BIGINT NULL FK | caregiver 계정 연결 |
| totp_secret_enc | VARBINARY NULL | P2 |
| is_active | TINYINT | |
| last_login_at | DATETIME | |

**login_attempts** — username_hash, ip, attempted_at, success. 인덱스 (username_hash, attempted_at).

**settings** — `key` VARCHAR(64) PK, `value` JSON, updated_by, updated_at. (예: AI 월 예산, 태그 허용 오차 분, 알림 D-일수)

**job_locks** — name PK, locked_at, locked_by, expires_at.

**schema_migrations** — version PK, applied_at, checksum.

### 3.2 기본 관리

**clients** (수급자)
| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | BIGINT PK | |
| client_code | VARCHAR(32) UNIQUE | 내부 코드 (AI 전송 시 가명 ID 생성에 사용하지 않음) |
| name | VARCHAR(64) | |
| birth_date | DATE | |
| gender | VARCHAR(8) | |
| rrn_enc | VARBINARY(255) NULL | 주민등록번호 암호문 |
| rrn_hmac | CHAR(64) NULL, INDEX | 중복 확인용 |
| ltc_number_enc | VARBINARY NULL | 장기요양인정번호 암호문 |
| ltc_number_hmac | CHAR(64) NULL, INDEX | 공단 파일 매핑용 |
| ltc_grade | VARCHAR(16) | 장기요양등급 (값 목록은 규칙 데이터로 관리) |
| grade_valid_from / grade_valid_to | DATE | 인정 유효기간 |
| copay_category | VARCHAR(32) NULL | 본인부담 구분 코드 (값과 비율은 billing_rules) |
| address, phone_enc | | |
| status | VARCHAR(16) | active / paused / terminated |
| deleted_at | DATETIME NULL | |

**guardians** — id, name, relation, phone_enc, email_enc, sms_consent(TINYINT), sms_consent_at.

**client_guardians** — client_id, guardian_id, is_primary. PK(client_id, guardian_id).

**employees** — id, employee_code UNIQUE, name, birth_date, phone_enc, hire_date, resign_date, employment_type, status.

**employee_credentials**
| 컬럼 | 설명 |
|---|---|
| id, employee_id FK | |
| credential_type | 자격 종류 코드 |
| credential_number_enc | 암호문 |
| issued_at, expires_at | 만료일 NULL 허용(만료 없는 자격) |
| file_id FK NULL | 사본 (`files`) |
| verified_by, verified_at | 원본 확인자 |

인덱스: (expires_at), (employee_id, credential_type).

**consultations** — id, client_id NULL(예비 수급자 가능), prospect_name, channel, consulted_at, content(TEXT), summary_draft_id NULL, follow_up_date, status.

**assessments** (욕구사정)
| 컬럼 | 설명 |
|---|---|
| id, client_id FK | |
| version | 같은 수급자 내 증가 |
| assessed_at, assessor_id | |
| structured_data JSON | 신체·인지·일상생활 등 구조화 항목 (양식 확인 후 스키마 확정) |
| narrative TEXT | 자유 서술 |
| goals TEXT | 수급자 목표·희망 |
| status | draft / final |

**service_contracts** — id, client_id, contract_start, contract_end, service_type, monthly_limit_note, file_id, status. 인덱스 (client_id, contract_end).

### 3.3 서비스·청구

**service_schedules**
| 컬럼 | 설명 |
|---|---|
| id, client_id FK, employee_id FK | |
| service_type | 방문요양 등 코드 |
| scheduled_date, planned_start, planned_end | |
| recurrence_group_id NULL | 반복 일정 묶음 |
| status | planned / cancelled / done |
| cancel_reason | |

인덱스: (scheduled_date), (employee_id, scheduled_date), (client_id, scheduled_date).

**service_records**
| 컬럼 | 설명 |
|---|---|
| id, schedule_id FK NULL, client_id, employee_id | 일정 없는 기록 허용 (경고 대상) |
| service_date, actual_start, actual_end | |
| content JSON | 제공 서비스 항목 |
| observations JSON | 식사·이동·위생·인지·정서 등 구조화 관찰 |
| notes TEXT | 특이사항 |
| revision | 정정 시 증가 |
| status | draft / submitted / locked(청구 확정 후) |

UNIQUE(schedule_id) — 일정당 기록 1건 (분할 방문 정책은 [Q-BIZ-03](OPEN_QUESTIONS.md)).

**record_revisions** — id, record_id, revision, before JSON, after JSON, reason, requested_by, approval_id FK, created_at. (정정·삭제 이력, 수정 불가)

**import_files** — id, original_name, stored_path, sha256 UNIQUE, mime, size, kind(tag/claim/other), uploaded_by, uploaded_at, retention_until.

**tag_import_batches** — id, import_file_id FK, template_id, period_from, period_to, row_count, error_count, status(previewed/applied/rolled_back), applied_by, applied_at.

**tag_records**
| 컬럼 | 설명 |
|---|---|
| id, batch_id FK | |
| row_no | 원본 행 번호 |
| raw JSON | 원본 행 (원본 보존) |
| client_id NULL, employee_id NULL | 매핑 결과 |
| tag_date, tag_in, tag_out | |
| mapping_status | mapped / unmapped / ambiguous |
| matched_record_id NULL | 매칭된 서비스 기록 |

인덱스: (client_id, tag_date), (batch_id).

**import_templates** — id, kind, name, column_map JSON, required_columns JSON, version, verified_with_file_id. (공단 파일 형식은 실제 파일로 확인 후 등록)

**billing_rules** (정산 파라미터; rule_versions와 1:1 연결)
| 컬럼 | 설명 |
|---|---|
| id, rule_version_id FK | |
| param_key | 예: `copay_rate.general` (예시 키, 값은 미확정) |
| param_value | 문자열 십진수 / 정수 |
| applies_to JSON | 대상 조건 |
| effective_from, effective_to | |

**copayment_calculations**
| 컬럼 | 설명 |
|---|---|
| id, client_id, period_ym CHAR(7) | |
| calc_version | 같은 기간 재계산 시 증가 |
| status | pending / held / calculated / approved / superseded |
| hold_reasons JSON | 보류 사유 |
| total_benefit_cost, copay_amount, insurer_amount | INT 원 |
| input_snapshot JSON | 계산 입력 전체 (재현용) |
| input_hash CHAR(64) | |
| rule_version_ids JSON | 적용 규칙 버전 |
| engine_version | 계산 엔진 코드 버전 |
| diff_from_prev JSON | 전월·이전 계산 대비 |
| approved_by, approved_at | |

UNIQUE(client_id, period_ym, calc_version). 확정(approved) 행은 수정 불가, 재계산은 새 버전.

**copayment_calc_items** — calculation_id, line_no, description, quantity, unit_cost, amount, rule_version_id.

**billing_checks** (검증 결과; Claim Validation 및 규칙 엔진 공용)
| 컬럼 | 설명 |
|---|---|
| id, run_id FK NULL | agent_runs |
| period_ym | |
| subject_type / subject_id | client / employee |
| check_code | 검증 항목 코드 |
| severity | error / warning / info / needs_confirmation |
| related_ids JSON | 관련 기록 ID |
| rule_version_id NULL | |
| message | 사람이 읽는 설명 (개인정보 최소) |
| recommended_action | |
| source | rule_engine / ai |
| status | open / acknowledged / resolved / dismissed |
| reviewed_by, reviewed_at, review_note | |
| fingerprint CHAR(64) UNIQUE | 같은 문제 중복 생성 방지 |

### 3.4 재무

**payment_records** — id, client_id, period_ym, billed_amount, received_amount, received_at, method, note. 미수금 = 청구 − 수납 (조회 시 계산, 저장하지 않음).

**revenue_records**, **expenses** — P2. 일자, 분류, 금액, 증빙 file_id.

### 3.5 문서·업무

**files** — id, stored_path, sha256, mime, size, owner_type, owner_id, retention_until.

**documents** — id, doc_type(care_plan/condition_report/notice 등), subject_type, subject_id, current_version_id, status(draft/review_required/approved/archived).

**document_versions**
| 컬럼 | 설명 |
|---|---|
| id, document_id, version | |
| body JSON | 섹션 구조 |
| evidence_links JSON | 섹션 → 근거 레코드 ID 목록 |
| missing_items JSON | 누락 항목 |
| review_flags JSON | 검토 필요 표시 |
| source | ai_draft / human_edit |
| ai_run_id NULL | |
| created_by, created_at | |
| approved_by, approved_at | 승인 후 불변 |

**care_plans** — id, client_id, assessment_id, document_id, period_from, period_to, status. (문서 본문은 document_versions)

**report_drafts** — id, kind(condition_monthly/guardian_notice), client_id, period_ym, document_id, run_id.

**tasks**
| 컬럼 | 설명 |
|---|---|
| id, task_type, priority(high/medium/low) | |
| subject_type, subject_id | |
| title, description | |
| due_date | |
| source | rule_engine / ai / manual / system |
| finding_id NULL, check_id NULL | 출처 |
| assignee_id | |
| status | open / in_progress / done / dismissed |
| fingerprint CHAR(64) UNIQUE NULL | 자동 생성 업무 중복 방지 |

**task_events** — task_id, event, from_status, to_status, note, actor_id, created_at.

**deadlines** — id, title, category, due_date, recurrence, remind_days JSON, linked_task_id.

**notifications** — id, user_id, channel(in_app/sms/email), payload JSON, status(pending_approval/approved/sent/failed), approval_id, sent_at.

### 3.6 AI·감사

**agent_runs**
| 컬럼 | 설명 |
|---|---|
| id, agent_name | |
| trigger_type | cron / event / manual |
| idempotency_key VARCHAR(191) UNIQUE | |
| input_ref JSON | 입력 레코드 ID 범위 (데이터 자체 아님) |
| status | queued / running / completed / needs_review / blocked / failed / cancelled |
| attempts, max_attempts | |
| locked_by, locked_at | |
| started_at, finished_at | |
| output JSON | 검증된 공통 출력 |
| error_code, error_message | 개인정보 없음 |

**agent_findings** — id, run_id, finding_type, severity, subject_type, subject_id, related_ids JSON, rule_version_id NULL, message, action, source(rule_engine/ai), status, fingerprint UNIQUE.

**agent_approvals**
| 컬럼 | 설명 |
|---|---|
| id, run_id NULL | |
| action_type | finalize_care_plan / finalize_report / approve_copay / send_message / amend_record / delete_record / finalize_claim / rule_exception |
| permission_level | L3 / L4 |
| target_type, target_id | |
| payload JSON | 변경 내용 |
| status | pending / approved / rejected / expired / executed |
| requested_by, decided_by, decided_at, decision_note | |
| executed_at | |

**ai_runs**
| 컬럼 | 설명 |
|---|---|
| id, agent_run_id | |
| provider, model | |
| prompt_id, prompt_version | |
| input_hash CHAR(64) | 가명처리 후 입력 해시 (재사용 키) |
| input_tokens, output_tokens | |
| cost_estimate_krw | INT (추정치) |
| latency_ms | |
| status | success / invalid_output / timeout / error / budget_blocked / cache_hit |
| redaction_report JSON | 제거·치환된 항목 종류와 수 (값은 저장 안 함) |

인덱스: (created_at), (input_hash, prompt_version).

**rule_versions** — [RULE_ENGINE_SPEC §3](RULE_ENGINE_SPEC.md) 참조. **rule_test_cases** — rule_version_id, name, input JSON, expected JSON, last_result, last_run_at.

**audit_logs**
| 컬럼 | 설명 |
|---|---|
| id, occurred_at | |
| actor_type(user/system/agent), actor_id | |
| action | 예: client.view_rrn, copay.approve |
| target_type, target_id | |
| ip, user_agent_hash | |
| detail JSON | 변경 필드명 등. 개인정보 값 저장 금지 |

추가 전용(INSERT만). 애플리케이션 DB 계정에서 UPDATE/DELETE 권한 제거를 권장 ([SECURITY §9](SECURITY.md)).

## 4. 인덱스·무결성 요약

| 목적 | 방법 |
|---|---|
| 중복 실행 방지 | `agent_runs.idempotency_key` UNIQUE, `job_locks` |
| 중복 결과 방지 | `billing_checks.fingerprint`, `agent_findings.fingerprint`, `tasks.fingerprint` UNIQUE |
| 중복 업로드 방지 | `import_files.sha256` UNIQUE |
| 정산 재현 | `copayment_calculations.input_snapshot` + `rule_version_ids` + `engine_version` |
| 확정 데이터 불변 | 애플리케이션 레벨 검사 + 상태 `approved`/`locked` 행 UPDATE 거부 (가능하면 트리거) |

## 5. 보유·파기

보유 기간은 법적 근거 확인 후 `settings`에 등록한다 ([Q-LEGAL-02](OPEN_QUESTIONS.md)). 파기 배치는 `retention_until`이 지난 파일과 행을 처리하고, 파기 사실을 감사 로그에 남긴다.
