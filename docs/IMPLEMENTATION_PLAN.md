# CareFlow Lite — 구현 계획 (4단계, 승인 대기)

- 문서 버전: 0.1 (초안) · 작성일: 2026-10-22
- **이 계획은 승인 전이다. 승인 전에는 코드를 작성하지 않는다.**
- 각 작업은 하나의 브랜치·PR 단위로 작게 진행하고, DB 변경은 마이그레이션으로만 한다.
- 완료 기준 공통: 관련 테스트를 실제로 실행해 통과 출력 첨부, `php -l` 통과, [보안 체크리스트](SECURITY.md#14-보안-점검-체크리스트-구현-각-단계-완료-시) 확인, 문서 갱신.

## M0. 사전 확인 (코드 없음)

| 항목 | 내용 |
|---|---|
| 범위 | [OPEN_QUESTIONS](OPEN_QUESTIONS.md)의 Q-ENV-01~06, Q-ARCH-01~03, Q-BIZ-01, Q-BIZ-03 답변 수집 |
| 산출물 | 문서 갱신, 잠정 버전 기준 확정 |
| 완료 기준 | Blocker(M1) 질문 모두 답변 |

## P0 작업

### W1. 프로젝트 골격

| 항목 | 내용 |
|---|---|
| 범위 | 디렉터리 구조, Front Controller, 라우터, 설정 로더(.env), 오류 처리, escape 헬퍼, `.htaccess`, `.gitignore`, `CLAUDE.md` 갱신, 테스트 러너 |
| 변경 파일 | `public/index.php`, `public/.htaccess`, `app/Core/*`, `config/config.php`, `composer.json`(승인 시), `tests/bootstrap.php` |
| DB 변경 | 없음 |
| 위험 | 공유호스팅 DocumentRoot 제약 |
| 테스트 | 라우팅 단위 테스트, `/config`, `/storage` 접근 차단 확인(T-SEC-10) |
| 완료 기준 | 빈 대시보드 렌더링, 테스트 러너 동작 |

### W2. 마이그레이션 러너 + 인증·세션·CSRF·RBAC·감사 로그

| 항목 | 내용 |
|---|---|
| 범위 | `database/migrate.php`, users/login_attempts/settings/audit_logs/job_locks/schema_migrations, 로그인·로그아웃, 미들웨어 |
| 변경 파일 | `database/migrations/0001~0003_*.sql`, `app/Security/*`, `app/Services/AuditLogger.php`, `app/Controllers/AuthController.php` |
| DB 변경 | 6개 테이블 생성 |
| 위험 | 세션 설정이 서버 php.ini와 충돌 |
| 테스트 | T-SEC-01, 02, 05, 06, 11, T-OPS-02 |
| 완료 기준 | admin 계정 생성 CLI, 로그인 후 대시보드 진입, 감사 로그 기록 |

### W3. 수급자·보호자·직원·자격증·계약·욕구사정·상담

| 항목 | 내용 |
|---|---|
| 범위 | CRUD, 필수항목 검증, 암호화 필드, 마스킹, 상담 후속 업무 |
| 변경 파일 | `app/Services/{Client,Guardian,Employee,Contract,Assessment,Consultation}Service.php`, Repositories, Views, `app/Security/Crypto.php` |
| DB 변경 | clients, guardians, client_guardians, employees, employee_credentials, service_contracts, assessments, consultations, files, tasks, task_events |
| 위험 | 암호화 키 관리 실수 → 데이터 복구 불가 (키 백업 절차 먼저) |
| 테스트 | T-BASE-01~06, T-SEC-03, 04, 07 |
| 완료 기준 | 모바일·PC에서 등록·조회, DB에 고유식별정보 평문 없음 |

### W4. 일정·서비스 기록·정정 요청

| 항목 | 내용 |
|---|---|
| 범위 | 일정(반복 포함), 충돌 탐지, 기록 입력(모바일), 정정·삭제 요청(승인 테이블 최소 구현) |
| 변경 파일 | `ScheduleService`, `ServiceRecordService`, `Rules/ScheduleConflictRule.php`, `Workflows/ApprovalWorkflow.php`(기본) |
| DB 변경 | service_schedules, service_records, record_revisions, agent_approvals |
| 위험 | 자정 넘는 방문, 분할 방문 정책 미확정(Q-BIZ-03) |
| 테스트 | T-REC-06~10, T-APPR-04 |
| 완료 기준 | 기록 직접 수정 불가, 정정 승인 후 이력 확인 |

### W5. 규칙 엔진 + 규칙 버전 관리

| 항목 | 내용 |
|---|---|
| 범위 | RuleRegistry, rule_versions, rule_test_cases, 관리 화면, integrity/operational 규칙 구현, regulatory 규칙은 `unverified` 틀만 |
| 변경 파일 | `app/Rules/*`, `app/Services/RuleVersionService.php` |
| DB 변경 | rule_versions, rule_test_cases, billing_rules, billing_checks |
| 위험 | 규칙을 법정 기준처럼 오인 표시 → 화면 문구로 유형 구분 |
| 테스트 | T-RULE-01~06, T-REC-02~05 |
| 완료 기준 | 근거 없는 규칙 verify 불가 |

### W6. CSV 업로드·태그 가져오기·매칭

| 항목 | 내용 |
|---|---|
| 범위 | 업로드 검증, 원본 보관, 템플릿 매핑, 미리보기, 매핑, 반영·롤백, 태그↔기록 비교 |
| 변경 파일 | `app/Services/Import/*`, `app/Validators/UploadValidator.php`, `Rules/TagTimeMatchRule.php` |
| DB 변경 | import_files, import_templates, tag_import_batches, tag_records |
| 위험 | 실제 공단 파일 형식 미확인(Q-BIZ-01) → 샘플 받기 전에는 템플릿을 확정하지 않음 |
| 테스트 | T-IMP-01~07, T-REC-01, T-SEC-12 |
| 완료 기준 | 합성 CSV로 전체 흐름 통과 |

### W7. 본인부담금 계산 엔진

| 항목 | 내용 |
|---|---|
| 범위 | CopaymentCalculator, 스냅샷, held 처리, diff, CSV·인쇄, 수납 기록 |
| 변경 파일 | `app/Services/Billing/*` |
| DB 변경 | copayment_calculations, copayment_calc_items, payment_records |
| 위험 | regulatory 규칙 미확인 → 모든 결과 held. 테스트는 가상 값으로만 |
| 테스트 | T-CALC-01~10 |
| 완료 기준 | 가상 규칙으로 재현성 확인, 실제 규칙 verified 전에는 "확정" 표시 안 됨 |

### W8. Agent 공통 엔진 + Daily Ops + Staff Compliance (규칙 기반) + 대시보드

| 항목 | 내용 |
|---|---|
| 범위 | AgentRunner, AgentInterface, PermissionGate, 큐 worker, cron 스크립트, 2개 Agent(LLM 없음), 대시보드 9개 위젯, 업무 목록, deadlines, 알림(in-app) |
| 변경 파일 | `app/Agents/*`, `bin/*.php`, `app/Controllers/DashboardController.php` |
| DB 변경 | agent_runs, agent_findings, deadlines, notifications |
| 위험 | cron 불가 환경 → LazyJobTrigger |
| 테스트 | T-AGT-C-01, 09, 10, T-AGT-01-*, T-AGT-06-*, T-OPS-01, T-UI-01~03 |
| 완료 기준 | 아침 브리핑이 LLM 없이 생성, 중복 실행 시 업무 중복 없음 |

### W9. 백업·운영 문서

| 항목 | 내용 |
|---|---|
| 범위 | `bin/backup.sh`, 복구 절차서, 배포 절차서 |
| 테스트 | T-OPS-03 (수동 리허설) |
| 완료 기준 | 복구 리허설 기록 |

**P0 종료 조건**: 합성 데이터로 시나리오 B·E 수행, 운영자 확인.

## P1 작업 (P0 파일럿 1개월 안정 후 착수, 상세 계획은 그때 재제출)

| 작업 | 범위 | 선행 조건 |
|---|---|---|
| W10 | LlmClient, Redactor, BudgetGuard, PromptRegistry, ai_runs, 비용 대시보드 | Q-LEGAL-01, Q-ARCH-04 |
| W11 | 승인 워크플로 완성, 문서·버전 관리, 승인 화면(diff) | W4 |
| W12 | Claim Validation Agent | W5, W6 |
| W13 | Care Plan Agent | W10, W11, Q-REG-07 |
| W14 | Condition Report Agent | W10, W11, Q-REG-08 |
| W15 | Billing & Copayment Agent, 안내 문구 초안, 발송 승인 | W7, W11, Q-OPS-03 |
| W16 | 상담 요약 | W10 |

## P2

자연어 업무 검색, 운영 지표·재무, 규정 변경 검토 보조, 직원 배정 보조, 방문목욕 모듈 — P1 정확성 확인 후 별도 계획.
