# CareFlow Lite — 규칙 엔진 명세

- 문서 버전: 0.2 (초안) · 작성일: 2026-10-22 · 개정: 2026-10-23 (서식 규칙 추가)
- 원칙: **AI가 장기요양 규정을 안다고 가정하지 않는다.** 규정에 근거한 판정은 공식 출처를 확인해 등록한 규칙으로만 한다.

## 1. 구성

| 구성요소 | 역할 |
|---|---|
| `RuleRegistry` | 규칙 ID → 구현 클래스 매핑, 날짜별 활성 버전 조회 |
| `RuleInterface` | `evaluate(RuleContext): RuleResult[]` |
| `rule_versions` | 규칙 메타데이터·근거·시행 기간·검증 상태 |
| `billing_rules` | 정산 파라미터(비율·단가·조건) — 코드가 아닌 데이터 |
| `rule_test_cases` | 규칙 버전별 테스트 |
| `RuleVersionService` | 등록·테스트·검증 상태 변경 (admin) |

규칙의 **로직**은 PHP 클래스, **수치·조건**은 데이터. 수치가 바뀌면 새 데이터 버전을 등록하고, 로직이 바뀌면 새 클래스 버전(`TagTimeMatchRuleV2`)을 추가한다.

## 2. 규칙 유형

| 유형 | 규정 근거 필요 | 예 |
|---|---|---|
| `integrity` (데이터 무결성) | 아니오 | 중복 기록, 필수 입력 누락, 시간 형식 오류, 매핑 실패 |
| `operational` (운영 정책) | 아니오 (센터 내부 정책) | 태그 허용 오차 분, 알림 D-일수, 우선순위 |
| `regulatory` (법령·고시) | **예** | 본인부담률, 감경 조건, 수가, 인정 시간 기준, 필수 서류 |

`regulatory` 규칙만 `verified` 상태가 있어야 확정 판정에 쓴다. `integrity`·`operational` 규칙은 내부 정의이므로 테스트 통과 후 활성화한다. 다만 operational 규칙 결과를 법정 기준처럼 표현하지 않는다.

## 3. 규칙 메타데이터 (`rule_versions`)

| 필드 | 설명 |
|---|---|
| id | PK |
| rule_id | 예: `TAG.TIME_MISMATCH` |
| version | 정수, rule_id 내 증가 |
| name | 규칙명 |
| category | integrity / operational / regulatory |
| applies_to | 적용 업무 (billing, record, staff, document …) |
| description | 설명 |
| target | 적용 대상 (예: service_type=방문요양) |
| effective_from / effective_to | 시행 기간 (NULL=무기한) |
| source_title | 근거 문서명 |
| source_clause | 근거 조항·항목 |
| source_url | 공식 출처 URL |
| checked_at | 확인 날짜 |
| checked_by | 확인자 |
| verification_status | draft / unverified / verified / deprecated |
| impl_class | PHP 클래스명 |
| params | JSON (또는 billing_rules 참조) |
| test_status | not_run / passed / failed |
| created_at | |

## 4. 생명주기

```
draft ──(근거 일부 입력)──► unverified ──(근거 전체 + 테스트 통과 + admin 확인)──► verified
                                                               │
                                     (새 버전 등록 또는 폐지) ──► deprecated
```

- `verified` 전환 조건: `source_title`, `source_clause`, `source_url`, `checked_at` 모두 있음 + 모든 `rule_test_cases` 통과 + admin 확인 (감사 로그).
- `verified` 버전은 수정할 수 없다. 변경은 새 버전으로.

## 5. 버전 선택과 소급 금지

- 판정 기준일 = 해당 기록의 **서비스 제공일** (정산은 서비스 제공 월의 각 일자).
- `effective_from <= 기준일 AND (effective_to IS NULL OR 기준일 <= effective_to)`인 버전 중 가장 높은 `verified` 버전 선택.
- 한 달 안에서 규칙이 바뀌면 일자별로 다른 버전을 적용하고, 계산 결과에 사용한 버전 목록을 모두 기록한다.
- 기준일에 맞는 규칙이 없으면 AI가 보충하지 않고 `RULE_MISSING` → 결과 `needs_confirmation`("규칙 확인 필요").
- `unverified`만 있으면 `RULE_UNVERIFIED` → 참고 결과로만 표시, 정산은 `held`.

## 6. 규칙 후보 목록 (초기 등록 대상)

> 아래 regulatory 규칙의 **수치·조건은 하나도 확정하지 않았다**. 모두 `unverified`로 시작하며, 공식 출처(법령, 보건복지부 고시, 국민건강보험공단 안내 등)를 운영자가 확인해야 한다. 확인 항목은 [OPEN_QUESTIONS §3](OPEN_QUESTIONS.md).

| rule_id | 유형 | 내용 | 확인 상태 |
|---|---|---|---|
| `REC.MISSING` | integrity | 일정(planned/done)이 있으나 서비스 기록 없음 → "기록 없음" | 내부 정의 |
| `REC.DUPLICATE` | integrity | 같은 수급자·같은 직원·시간 겹침 기록 | 내부 정의 |
| `REC.TIME_FORMAT` | integrity | 시각 누락·형식 오류·종료<시작 | 내부 정의 |
| `REC.TIME_MISMATCH` | operational | 예정 대비 실제 시각 차이 > `settings.schedule_tolerance_min` | 내부 정책 |
| `TAG.TIME_MISMATCH` / `TAG.RECORD_MISMATCH` | operational | 태그 시각과 기록 시각 차이 > `settings.tag_tolerance_min`, 태그만 있음/기록만 있음 | 내부 정책 (법정 인정 기준 아님) |
| `MAP.CLIENT_FAILED` / `MAP.EMPLOYEE_FAILED` | integrity | 업로드 행 매핑 실패·중복 후보 | 내부 정의 |
| `SCHED.EMPLOYEE_CONFLICT` / `SCHED.CLIENT_CONFLICT` | integrity | 일정 시간 겹침 | 내부 정의 |
| `SUM.MONTHLY_MISMATCH` | integrity | 일정·기록·태그 월 합계 불일치 | 내부 정의 |
| `STAFF.CREDENTIAL_EXPIRY` | operational | 만료일 D-N 알림 | 내부 정책 |
| `STAFF.REQUIRED_DOCS` | regulatory | 직원 필수 서류 목록 | **미확인** |
| `STAFF.TRAINING_DUE` | regulatory | 필수 교육 종류·주기 | **미확인** |
| `REC.ENTRY_LATE` | operational | 방문일과 입력일 차이 > 설정 일수 (법정 기한 아님, [DATA_ENTRY §5](DATA_ENTRY.md)) | 센터 정책 |
| `REC.REPEATED_CONTENT` | operational | 같은 수급자의 연속 방문 기록 내용이 동일. 판정이 아니라 확인 요청 | 센터 정책 |
| `REC.SOURCE_CONFLICT` | integrity | 같은 방문에 입력 출처가 다른 기록이 2건 이상 | 내부 정의 |
| `FORM.HOMECARE.REQUIRED` | operational | 급여제공기록지 필수 항목 누락, 완료 영역 0개, 완료인데 소요시간 0 ([FORMS §1.3](FORMS.md)) | 센터 양식 기준 |
| `FORM.HOMECARE.MINUTES_SUM` | operational | 영역별 소요시간 합 vs 방문 시간 차이 > 설정값 (법정 인정 시간 판정 아님) | 센터 양식 기준 |
| `REPORT.WEEKLY_MISSING` | operational | 주간 상태변화기록 누락 주 (주간 작성 여부는 Q-BIZ-11) | 센터 정책 |
| `CONDITION.CHANGE_THRESHOLD` | operational | 월간 지표 변화 판정 임계값 (예: 식사량 감소 기록 횟수 증가폭). 의학적 기준 아님 | 센터 정책 |
| `CLIENT.GRADE_VALIDITY` | integrity | 인정 유효기간 밖 일정 | 내부 정의 (유효기간 데이터 기준) |
| `DOC.REQUIRED_MISSING` | regulatory | 계약서·계획서 등 필수 서류 존재 | **미확인** |
| `CAREPLAN.FORM_FIELDS` | regulatory | 급여제공계획서 필수 항목 | **미확인** (공식 양식 필요) |
| `COPAY.RATE` | regulatory | 본인부담 구분별 비율 | **미확인** |
| `COPAY.REDUCTION` | regulatory | 감경·면제 조건 | **미확인** |
| `COPAY.ROUNDING` | regulatory | 금액 절사·반올림 단위 | **미확인** |
| `FEE.SCHEDULE` | regulatory | 급여 종류·시간별 수가 (연도별) | **미확인** |
| `BENEFIT.MONTHLY_LIMIT` | regulatory | 등급별 월 한도액 | **미확인** |
| `BENEFIT.TIME_RULES` | regulatory | 방문 시간 인정·산정 기준 | **미확인** |

## 7. 결과 형식 (`RuleResult`)

```php
final class RuleResult
{
    public string $ruleId;
    public int $ruleVersionId;
    public string $severity;      // error | warning | info | needs_confirmation
    public string $subjectType;
    public int $subjectId;
    public array $relatedIds;
    public string $messageKey;    // 메시지 템플릿 키 (개인정보 미포함)
    public array $messageParams;
    public string $recommendedAction;
    public string $fingerprint;   // sha256(ruleId|subject|relatedIds|period)
}
```

심각도 결정 규칙:
- `integrity` 위반 → `error`
- `operational` 위반 → `warning`
- `regulatory` + verified 위반 → `error`
- `regulatory` + unverified/missing → `needs_confirmation`
- 정상 처리 요약 → `info`

## 8. 본인부담금 계산 엔진 (`CopaymentCalculator`)

```
입력: client_id, period_ym
1. 입력 수집: 대상자(본인부담 구분, 등급, 유효기간), 해당 월 확정 서비스 기록, 적용 수가 데이터
2. 필수값 점검 → 누락 시 held(사유 목록)
3. 일자별 적용 규칙 버전 선택 (§5) → unverified/missing 있으면 held
4. 항목별 급여비용 = 정수 연산 (수가 × 수량)
5. 월 합계, 한도 비교 (규칙 존재 시)
6. 본인부담금 = 비율(basis point) 적용 → COPAY.ROUNDING 규칙으로 정수화
7. 결과 + input_snapshot + rule_version_ids + engine_version 저장 (새 calc_version)
8. 이전 버전·전월과 diff 계산
```

- 부동소수점 금지: `intdiv`와 정수 곱셈, 필요 시 `bcmath`(서버 확장 확인 필요, [Q-ENV-06](OPEN_QUESTIONS.md)).
- 같은 `input_hash + engine_version`이면 같은 결과가 나와야 한다(재현성 테스트).

## 9. 규칙 테스트

- 규칙 버전마다 `rule_test_cases`(정상·경계·위반·누락) 최소 3건.
- PHPUnit 데이터 제공자로 `rule_test_cases`를 읽어 실행하는 공용 테스트 + 관리자 화면 "테스트 실행" 버튼.
- regulatory 규칙의 테스트 기대값은 공식 자료의 예시(있다면)로 작성하고 출처를 테스트 케이스에 기록한다.

## 10. 규정 변경 관리

- 운영자가 공식 공지를 확인하면 새 규칙 버전을 `unverified`로 등록 → 근거 입력 → 테스트 → verified.
- P2 "규정 변경 검토 보조"는 공지 문서를 요약해 **변경 후보**만 제시한다. 규칙 데이터를 자동으로 바꾸지 않는다.
