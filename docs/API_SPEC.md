# CareFlow Lite — API 명세

- 문서 버전: 0.1 (초안) · 작성일: 2026-10-22
- 화면은 서버 렌더링이 기본이다. 아래 API는 (1) 화면의 비동기 동작(AJAX), (2) 향후 모바일 입력 화면을 위한 내부 JSON API다. 외부 공개 API는 없다.

## 1. 공통 규약

| 항목 | 규약 |
|---|---|
| 기본 경로 | `/api/v1` |
| 인증 | 세션 쿠키 (`HttpOnly; Secure; SameSite=Lax`). 토큰 인증 없음 |
| CSRF | 모든 POST/PUT/PATCH/DELETE에 `X-CSRF-Token` 헤더 필수 |
| 형식 | 요청·응답 `application/json; charset=utf-8`, 업로드는 `multipart/form-data` |
| 날짜·시각 | `YYYY-MM-DD`, `YYYY-MM-DDTHH:MM:SS+09:00` |
| 금액 | 정수(원) |
| 페이지 | `?page=1&per_page=50` (최대 200) |
| 멱등성 | 생성 요청은 `Idempotency-Key` 헤더 선택 지원 (업로드·계산·Agent 실행은 필수) |

### 응답 형식

```json
{ "ok": true, "data": { }, "meta": { "page": 1, "total": 120 } }
```

```json
{ "ok": false, "error": { "code": "VALIDATION_FAILED", "message": "필수 항목이 누락되었습니다.", "fields": { "birth_date": "required" } } }
```

### 오류 코드

| HTTP | code | 의미 |
|---|---|---|
| 400 | VALIDATION_FAILED | 입력 검증 실패 |
| 401 | UNAUTHENTICATED | 로그인 필요 |
| 403 | FORBIDDEN / CSRF_INVALID | 권한 없음 / CSRF 실패 |
| 404 | NOT_FOUND | |
| 409 | CONFLICT / DUPLICATE / IMMUTABLE | 중복, 확정 데이터 수정 시도 |
| 422 | RULE_UNVERIFIED / CALC_HELD | 미검증 규칙, 계산 보류 |
| 423 | APPROVAL_REQUIRED | 승인 필요 작업 |
| 429 | RATE_LIMITED / AI_BUDGET_EXCEEDED | |
| 500 | INTERNAL | 상세는 로그에만 |

## 2. 권한 표기

`A`=admin, `S`=staff, `C`=caregiver(본인 데이터만). 상세는 [SECURITY §2](SECURITY.md).

## 3. 엔드포인트

### 3.1 인증

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| POST | `/auth/login` | 공개 | `{username, password}` → 세션 생성, 실패 횟수 제한 |
| POST | `/auth/logout` | 로그인 | |
| GET | `/auth/me` | 로그인 | 사용자·역할·CSRF 토큰 |

### 3.2 수급자·보호자

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| GET | `/clients` | A,S | 목록 (`status`, `q` 필터). 고유식별정보 미포함 |
| POST | `/clients` | A,S | 등록. 필수 항목 검사 결과 반환 |
| GET | `/clients/{id}` | A,S | 상세 (마스킹) |
| PATCH | `/clients/{id}` | A,S | 수정 → 감사 로그 |
| POST | `/clients/{id}/reveal-rrn` | A | 주민번호 원문 조회 (재인증 + 감사 로그) |
| GET/POST | `/clients/{id}/guardians` | A,S | 보호자 연결 |
| GET/POST | `/clients/{id}/assessments` | A,S | 욕구사정 목록·등록(새 버전) |
| GET/POST | `/clients/{id}/contracts` | A,S | 계약 |
| GET/POST | `/consultations` | A,S | 상담 |

### 3.3 직원

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| GET/POST | `/employees` | A,S | |
| GET/PATCH | `/employees/{id}` | A,S | |
| GET/POST | `/employees/{id}/credentials` | A,S | 자격증 (사본은 multipart) |

### 3.4 일정·기록

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| GET | `/schedules?date_from&date_to&employee_id&client_id` | A,S,C | C는 본인 일정만 |
| POST | `/schedules` | A,S | 응답에 충돌 경고 포함 (`warnings[]`) |
| PATCH | `/schedules/{id}` | A,S | 변경 시 이벤트 → Daily Ops 큐 |
| POST | `/schedules/{id}/cancel` | A,S | 사유 필수 |
| GET | `/records?...` | A,S,C | |
| POST | `/records` | A,S,C | |
| POST | `/records/{id}/amend-requests` | A,S,C | 정정 요청 → 승인 대기 생성 (L3). 직접 PATCH 없음 |
| POST | `/records/{id}/delete-requests` | A,S | 삭제 요청 → 승인 대기 |

### 3.5 가져오기

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| POST | `/imports` | A,S | 업로드 → 형식 검증 → `batch_id`, 미리보기 반환. 같은 해시면 409 |
| GET | `/imports/{batch_id}/preview` | A,S | 행·오류·매핑 실패 |
| POST | `/imports/{batch_id}/mappings` | A,S | 매핑 수동 지정 |
| POST | `/imports/{batch_id}/apply` | A | 반영 (트랜잭션) |
| POST | `/imports/{batch_id}/rollback` | A | 배치 되돌리기 |
| GET/POST | `/import-templates` | A | 열 매핑 템플릿 |

### 3.6 검증·규칙

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| POST | `/checks/run` | A,S | `{period_ym, scope}` → Claim Validation 실행 큐 (202) |
| GET | `/checks?period_ym&severity&status&source` | A,S | |
| PATCH | `/checks/{id}` | A,S | 상태 변경(확인/해결/기각) + 메모 |
| GET | `/rules` | A,S | 규칙과 버전, 검증 상태 |
| POST | `/rules/{rule_id}/versions` | A | 새 버전 등록 (상태 draft) |
| POST | `/rules/versions/{id}/test` | A | 규칙 테스트 케이스 실행 |
| POST | `/rules/versions/{id}/verify` | A | 근거 입력 + 테스트 통과 시에만 verified |

### 3.7 정산

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| POST | `/billing/{period_ym}/calculate` | A | 전체 또는 `client_ids[]` 계산 (멱등 키 필수) |
| GET | `/billing/{period_ym}` | A,S | 대상자별 결과, 보류 사유 |
| GET | `/billing/calculations/{id}` | A,S | 계산 근거 상세(항목, 규칙 버전, 입력 스냅샷) |
| POST | `/billing/calculations/{id}/approve-request` | A | 승인 요청 생성 |
| GET | `/billing/{period_ym}/export.csv` | A | CSV (감사 로그) |
| POST | `/billing/{period_ym}/notices/draft` | A | 보호자 안내 초안 생성 |
| GET/POST | `/payments` | A | 수납 기록 |

### 3.8 업무·대시보드·알림

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| GET | `/dashboard` | A,S | 위젯 데이터 묶음 |
| GET | `/briefing/today` | A,S | 최신 Daily Ops 결과 |
| GET/POST | `/tasks` | A,S | |
| PATCH | `/tasks/{id}` | A,S | 상태·담당자 |
| GET/POST | `/deadlines` | A,S | |
| GET | `/notifications` | 로그인 | |

### 3.9 Agent·승인·문서

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| POST | `/agents/{agent_name}/runs` | A | 수동 실행 (202, `run_id`) |
| GET | `/agents/runs?agent&status` | A | 실행 이력 |
| GET | `/agents/runs/{id}` | A | 결과, findings, ai_runs 요약 |
| POST | `/agents/runs/{id}/retry` | A | failed만 재시도 |
| POST | `/agents/runs/{id}/cancel` | A | queued만 취소 |
| GET | `/approvals?status=pending` | A | 승인 대기 |
| GET | `/approvals/{id}` | A | 원본·근거·규칙 버전·초안·diff |
| POST | `/approvals/{id}/approve` | A | 재확인 문구 입력. L4는 재인증 |
| POST | `/approvals/{id}/reject` | A | 사유 필수 |
| GET | `/documents/{id}` | A,S | 현재 버전 + 이력 |
| POST | `/documents/{id}/versions` | A,S | 사람이 수정한 새 버전 |
| POST | `/documents/{id}/submit-review` | A,S | review_required로 |
| GET | `/documents/{id}/print` | A,S | 인쇄용 HTML |

### 3.10 관리

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| GET/PATCH | `/settings` | A | AI 예산, 허용 오차 등 |
| GET | `/ai/usage?month` | A | 비용 대시보드 |
| GET | `/audit-logs` | A | 필터 조회 |

## 4. 예시

**POST `/api/v1/billing/2026-10/calculate`** (요청)
```json
{ "client_ids": [12, 15] }
```
응답 (예시 형식이며 실제 금액·규정이 아님)
```json
{
  "ok": true,
  "data": {
    "results": [
      { "client_id": 12, "calculation_id": 301, "status": "calculated", "copay_amount": 0, "rule_version_ids": [7, 9] },
      { "client_id": 15, "calculation_id": null, "status": "held", "hold_reasons": ["RULE_UNVERIFIED:COPAY_RATE", "MISSING:copay_category"] }
    ]
  }
}
```

## 5. CLI 명령 (cron)

| 명령 | 주기(안) | 설명 |
|---|---|---|
| `php bin/daily_jobs.php` | 매일 06:00 | Daily Ops, Staff Compliance 큐 생성 |
| `php bin/agent_worker.php --max-seconds=240` | 5분마다 | 큐 처리 |
| `php bin/monthly_jobs.php --period=YYYY-MM` | 매월 1일 07:00 | 상태변화 보고, 정산 준비 |
| `php bin/backup.sh` | 매일 02:00 | DB·파일 백업 |
| `php database/migrate.php [--dry-run]` | 배포 시 | 마이그레이션 |
