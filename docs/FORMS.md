# CareFlow Lite — 서식 매핑

- 문서 버전: 0.1 · 작성일: 2026-10-23
- 출처: 운영자 제공 「재가장기요양기관 필수 서식 표준 양식집」(Google Docs, 2026-10-23 확인)
- 성격: 운영자가 국민건강보험공단 서식을 **기반으로 표준화한 센터 내부 템플릿**이다. 공단 공식 양식 원본과 같은지는 확인하지 않았으므로, 규칙 엔진에서는 `operational`(센터 정책)으로 다루고 공식 필수 항목 규칙(`regulatory`)은 별도로 확인한다 ([Q-REG-07, 08](OPEN_QUESTIONS.md)).

## 1. 방문요양 급여제공기록지 → `service_records`

### 1.1 기본 정보

| 서식 항목 | 저장 위치 | 비고 |
|---|---|---|
| 수급자명 | `clients.name` | 기록에는 `client_id`만 저장 |
| 장기요양등급 | `clients.ltc_grade` | 코드: `G1`~`G5`, `COG`(인지지원등급). 코드 목록은 `settings.ltc_grades` 데이터로 관리 |
| 장기요양인정번호 | `clients.ltc_number_enc` / `ltc_number_hmac` | 서식상 `L`로 시작. 형식 검증 정규식은 실제 번호 형식 확인 후 확정 |
| 장기요양기관기호 | `settings.center.institution_code` | 센터 단위 1개 |
| 요양보호사 성명 | `employees.name` | 기록에는 `employee_id` |
| 제공일자 및 시간 | `service_date`, `actual_start`, `actual_end` | |

### 1.2 세부 제공 서비스 (4개 영역)

| 영역 코드 | 서식 구분 | 세부 예시 (선택 항목) | 기록 |
|---|---|---|---|
| `physical` | 신체활동지원 | 세면, 세수, 양치, 식사 수발, 체위변경, 이동도움, 화장실 이용 도움 | 완료 여부 + 소요시간(분) |
| `cognitive` | 인지관리지원 | 인지자극활동, 행동변화 대응, 회상요법 | 〃 |
| `daily_living` | 일상생활지원 | 외출동행, 일상업무 대행, 청소, 세탁, 식사 준비 | 〃 |
| `emotional` | 정서지원 | 말벗, 격려, 위로, 생활상담 | 〃 |

저장 형식 (`service_records.content` JSON):

```json
{
  "form_version": "home_care_v1",
  "areas": [
    { "code": "physical", "done": true, "minutes": 40, "items": ["meal_assist", "position_change"] },
    { "code": "cognitive", "done": false, "minutes": 0, "items": [] },
    { "code": "daily_living", "done": true, "minutes": 30, "items": ["cleaning"] },
    { "code": "emotional", "done": true, "minutes": 20, "items": ["conversation"] }
  ]
}
```
(형식 예시이며 실제 기록이 아님)

세부 항목 코드 목록은 `settings.service_items`에 데이터로 둔다. 서식이 바뀌면 `form_version`을 올리고 이전 기록은 이전 버전으로 해석한다.

### 1.3 이 서식에서 나오는 검증 (규칙 엔진)

| 규칙 | 유형 | 내용 |
|---|---|---|
| `FORM.HOMECARE.REQUIRED` | operational | 기본 정보 누락, 4개 영역 중 완료 0개, 완료인데 소요시간 0 |
| `FORM.HOMECARE.MINUTES_SUM` | operational | 영역별 소요시간 합계와 (종료−시작) 차이 > 설정값 → 경고 (법정 인정 시간 판정 아님) |
| `TAG.TIME_MISMATCH` | operational | 기록 시각과 RFID 태그 시각 비교. 양식집의 "공단 기준 이탈 시 알림"은 공단 기준 수치를 확인하기 전까지 센터 허용 오차(설정값)로 동작하고, 공단 기준은 `BENEFIT.TIME_RULES`(regulatory, 미확인)로 별도 등록 |

## 2. 수급자 상태변화기록지 (주간/월간) → `condition_observations`, `report_drafts`

| 점검 항목 코드 | 서식 점검 항목 | 관찰 예시 (구조화 필드) |
|---|---|---|
| `physical_health` | 신체 및 건강 상태 | 피부 상태(발적·상처 유무), 체중 변화, 통증 호소 여부 |
| `meal_medication` | 식사 및 복약 | 식사량(정상/감소), 섭취 형태, 약 복용 여부·시간 |
| `cognition_emotion` | 인지 및 정서 상태 | 기억력 변화, 배회, 수면 상태, 감정 기복 |
| `elimination` | 배변 상태 | 배변 횟수, 실금 여부, 기저귀 교체 횟수 |

각 항목은 **상태 및 변화 내용**과 **조치 사항 및 특이사항** 두 칸을 가진다.

데이터 흐름:
```
일별 서비스 기록의 관찰값 (service_records.observations, 위 4개 항목 구조)
   → 주간 상태변화기록 (condition_observations, period_type=weekly)  ← 담당자 작성 또는 자동 집계 초안
   → 월간 상태변화보고서 초안 (report_drafts, kind=condition_monthly) ← Agent 4가 주간 기록 취합
   → 담당자 검토·승인
```

`service_records.observations` 저장 형식:

```json
{
  "physical_health": { "skin": "normal|redness|wound", "pain": false, "weight_kg": null, "note": "" },
  "meal_medication": { "intake": "normal|decreased|refused", "form": "regular|soft|liquid|tube", "med_taken": true, "med_time": "08:30", "note": "" },
  "cognition_emotion": { "memory_change": false, "wandering": false, "sleep": "normal|poor", "mood_swing": false, "note": "" },
  "elimination": { "bowel_count": 1, "incontinence": false, "diaper_changes": 0, "note": "" }
}
```
(형식 예시이며 실제 기록이 아님. 선택값 목록은 구현 시 운영자와 확정)

선택형 값으로 받으면 LLM 없이도 PHP가 "식사량 감소 기록 n회 → 전주 대비 증가" 같은 집계를 결정론적으로 만들 수 있다. 이것이 [AGENT_SPEC](AGENT_SPEC.md)의 Agent 4 **템플릿 모드**의 기반이다.

## 3. 방문간호 급여제공기록지 → 확장 모듈 (P2)

| 구분 | 항목 | 저장 형식 |
|---|---|---|
| 바이탈 체크 | 혈압(mmHg), 체온(℃), 맥박, 혈당(mg/dL) | 숫자 필드 + 측정 시각 |
| 간호 처치 | 욕창 관리, 투약 지도, 카테터 관리, 상태 관찰 등 | 선택 항목 + 서술 |
| 지시서 확인 | 방문간호 지시서 기재 사항 이행 여부 | `done` / `not_applicable` |

- 방문간호는 별도 급여 종류(`service_type=home_nursing`)로, 방문목욕과 같이 확장 모듈로 설계만 해 둔다. 현재 센터가 방문간호를 제공하는지는 [Q-BIZ-08](OPEN_QUESTIONS.md).
- 바이탈 수치는 기록·표시만 한다. 정상 범위 판정이나 의료적 해석은 하지 않는다.

## 4. 양식집의 자동화 연동 포인트 반영

| 양식집 항목 | 반영 위치 |
|---|---|
| 기록 대조 자동화 (작성 시각 vs RFID 태그, 기준 이탈 알림) | `TAG.TIME_MISMATCH`, Agent 1·2. 알림은 시스템 내 알림. 외부 발송은 승인 후 |
| 문서 생성 자동화 (주간 데이터 → 월간 상태변화보고서 요약) | Agent 4 (템플릿 모드 P1, LLM 모드는 보류) |
| 청구 검증 연동 (기록지 누락 항목 → 청구 전 오류 알림) | `FORM.HOMECARE.REQUIRED`, Agent 2 |
