# CareFlow Lite — 제품 요구사항 정의서 (PRD)

- 문서 버전: 0.2 (초안)
- 작성일: 2026-10-22 · 개정: 2026-10-23 (운영자 답변 반영)
- 상태: **검토 대기** — 구현 착수 전 운영자 승인 필요
- 관련 문서: [ARCHITECTURE](ARCHITECTURE.md) · [DATA_MODEL](DATA_MODEL.md) · [API_SPEC](API_SPEC.md) · [AGENT_SPEC](AGENT_SPEC.md) · [RULE_ENGINE_SPEC](RULE_ENGINE_SPEC.md) · [SECURITY](SECURITY.md) · [TEST_PLAN](TEST_PLAN.md) · [OPEN_QUESTIONS](OPEN_QUESTIONS.md) · [IMPLEMENTATION_PLAN](IMPLEMENTATION_PLAN.md) · [FORMS](FORMS.md)

---

## 0. 저장소·환경 분석 결과 (1단계)

| 항목 | 확인 결과 | 비고 |
|---|---|---|
| 저장소 | `KimSehoon81/angular`, 브랜치 `claude/careflow-lite-prd-aoxk6b` | 커밋 없음, 파일 없음 (빈 저장소) |
| 기존 PHP 코드 | 없음 | 재사용 가능한 코드 없음 → 신규 구축 |
| 기존 DB 스키마 | 없음 | 신규 설계 ([DATA_MODEL](DATA_MODEL.md)) |
| 운영 서버 PHP 버전 | **미확인** | 분석용 컨테이너에는 PHP 8.3 CLI가 있으나 운영 서버와 무관 |
| 운영 서버 MySQL/MariaDB 버전 | **미확인** | |
| Apache 설정 (DocumentRoot, mod_rewrite, AllowOverride) | **미확인** | |
| 배포 방식 (FTP, git pull, rsync 등) | **미확인** | |
| cron 사용 가능 여부 | **가능** (운영자 답변 2026-10-23) | 최소 주기는 미확인 |
| 저장소 | **새 저장소를 만든다** (운영자 답변) | 현재 `angular` 저장소에는 설계 문서만 둔다. 새 저장소가 생기면 문서를 옮기고 코드는 거기서만 작성 → [Q-ENV-09](OPEN_QUESTIONS.md) |
| 서식 | 「재가장기요양기관 필수 서식 표준 양식집」 제공 | [FORMS.md](FORMS.md)로 데이터 모델에 매핑 |

### 0.1 범위 결정 (2026-10-23)

- **외부 LLM 호출 기능은 이번 범위에서 제외한다.** 개인정보 전송 검토(Q-LEGAL-01)를 보류했기 때문이다. Agent는 모두 PHP 규칙·템플릿으로 동작하도록 설계하고, LLM 연동부(`LlmClient`, `Redactor`, 비용 관리)는 인터페이스 자리만 남긴다. 설정 `ai.enabled`의 기본값은 `false`이며, 보류가 풀리기 전에는 켤 수 없다.
- 이 결정으로 Care Plan Agent의 자유 서술 초안, 상담 요약 같은 "자연어 생성" 기능은 보류되고, 구조화된 입력을 템플릿에 채우는 방식만 남는다.

**결론**: 기존 코드가 없어서 삭제·교체 위험은 없다. 대신 운영 환경을 확인하지 않았으므로 PHP 버전에 의존하는 기능(enum, readonly 속성 등)은 환경 확인 전까지 확정하지 않는다. 설계는 **PHP 8.1 이상, MySQL 8.0 / MariaDB 10.6 이상**을 잠정 기준으로 하며, 이 기준은 [Q-ENV-01~04](OPEN_QUESTIONS.md)가 답변되면 확정한다.

---

## 1. 배경과 문제 정의

국내 재가방문요양센터를 혼자 운영하는 사람은 다음 업무를 직접 반복한다.

- 수급자·보호자·요양보호사 정보와 방문 일정을 엑셀, 메모, 공단 시스템에 나눠서 관리한다.
- 태그(RFID/스마트폰) 기록, 실제 서비스 기록, 일정, 청구 자료를 눈으로 대조한다.
- 자격증 만료, 계약 갱신, 행정 마감일을 기억에 의존한다.
- 급여제공계획서, 월별 상태변화 기록 같은 반복 문서를 매번 새로 쓴다.
- 본인부담금 계산과 보호자 안내를 월말마다 직접 한다.

이 때문에 누락과 불일치가 청구 단계에서야 발견되고, 운영자 시간은 대부분 확인 작업에 쓰인다.

## 2. 제품 비전

> 운영자는 시스템이 찾아낸 **예외를 확인하고 승인**하는 데만 시간을 쓴다.

CareFlow Lite는 데이터를 저장하는 데서 그치지 않는다. 업무 이벤트를 감지하고, 결정론적 규칙으로 검증하고, 필요할 때만 AI로 문서 초안을 만들고, 사람의 승인을 받아 확정한다.

## 3. 목표와 비목표

### 3.1 목표 (측정 지표는 파일럿 운영 후 기준값 확정)

| ID | 목표 | 측정 지표(안) |
|---|---|---|
| G1 | 수급자·보호자·요양보호사·일정의 통합 관리 | 외부 엑셀 사용 횟수 감소 |
| G2 | 태그·기록·일정·청구 자료 불일치 사전 탐지 | 청구 후 발견 오류 건수 감소 |
| G3 | 행정 업무·마감일 자동 관리 | 마감 초과 건수 0 |
| G4 | 욕구사정 기반 급여제공계획서 초안 자동 생성 | 계획서 1건당 작성 시간 |
| G5 | 월별 상태변화 기록 초안 자동 생성 | 월말 문서 작업 시간 |
| G6 | 본인부담금 계산과 청구 전 점검 자동화 | 정산 작업 시간, 재계산 건수 |
| G7 | 매일 업무 브리핑 제공 | 브리핑 열람률, 누락 발견 건수 |
| G8 | 운영자는 예외 확인·승인에 집중 | 일일 수작업 시간 |

### 3.2 비목표 (초기 버전 범위 밖)

- 국민건강보험공단 시스템 자동 로그인·자동 제출·화면 조작 (명시적으로 금지)
- 공식 심사 결과 보장 또는 청구 가능 여부의 법적 확정
- 의료적 진단·처방
- 상시 실행되는 AI 서버, Node.js/Python 백엔드, 메시지 브로커
- 다중 센터(멀티 테넌트) 운영 — 확장 여지만 남긴다
- **외부 LLM API 호출** — 개인정보 전송 검토 전까지 보류 (§0.1)
- 급여(인건비) 계산·4대보험 신고

## 4. 사용자

| 페르소나 | 설명 | 주요 화면 |
|---|---|---|
| 운영자(센터장, admin) | 1인 운영자. 모든 권한. 최종 승인자 | 대시보드, 승인 대기, 정산, 문서 |
| 사무 보조(staff) | 선택. 입력·조회 위주, 승인 불가 | 수급자, 일정, 기록 입력 |
| 요양보호사(caregiver) | 선택(P2). 본인 일정·기록만 모바일로 조회·입력 | 내 일정, 기록 작성 |
| 시스템(cron/agent) | 예약 작업과 Agent 실행 주체 | — |

역할별 권한은 [SECURITY §2](SECURITY.md)에서 정의한다.

## 5. 설계 원칙

1. **정확성**: 계산과 규정 검증은 재현 가능한 PHP 규칙으로 처리한다.
2. **Agent 주도 업무 감지**: 운영자가 찾아보지 않아도 누락과 마감을 알려준다.
3. **승인 기반 자동화**: 초안과 검증은 자동, 확정은 사람.
4. **저비용**: 기존 APM 서버를 쓰고 AI 호출을 최소화한다 (목표: 추가 AI API 비용 월 10,000원 이내, 보장 아님).
5. **추적 가능성**: 모든 결과는 근거, 규칙 버전, 변경 이력을 남긴다.
6. **보안**: 개인정보는 최소 수집·사용·전송한다.
7. **단순성**: 1인 운영자가 이해하고 관리할 수 있어야 한다.
8. **확장성**: 방문목욕 등은 모듈로 추가한다.
9. **불확실성 표시**: 근거가 없으면 추측하지 않고 "확인 필요"로 표시한다.
10. **시간 절감**: 모든 기능은 반복 작업 감소, 오류 예방, 현황 파악 중 하나 이상에 기여해야 한다.

### 5.1 역할 분리 (필수)

| 업무 | 담당 |
|---|---|
| 본인부담금·비용 계산 | PHP 계산 엔진 |
| 태그 시간 비교, 일정 충돌 탐지 | PHP 검증 엔진 |
| 자격증 만료일, 서류 필수 항목 확인 | PHP 규칙 엔진 |
| 상담 요약, 계획서 초안, 상태변화 요약, 예외 설명 | AI Agent (LLM) |
| 규정 근거·적용 버전 연결 | 규정 데이터 + PHP 검증 |
| 최종 문서 확정, 청구 확정·제출 | 사람 |
| 외부 문자 발송 | 승인 정책을 거친 발송 모듈 |

## 6. 기능 요구사항

우선순위: **P0** 기본 ERP·규칙 자동화 / **P1** 문서·청구 보조 Agent / **P2** 고도화.
각 요구사항의 구현 모듈·테이블·테스트·완료 조건은 [§9 추적표](#9-요구사항-추적표-3단계)에 정리한다.

### 6.1 인증·권한 (AUTH)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-AUTH-01 | 아이디·비밀번호 로그인, `password_hash()` 저장, 로그인 실패 횟수 제한 | P0 |
| FR-AUTH-02 | 역할 기반 접근 제어(admin/staff/caregiver), 화면·API 단위 권한 검사 | P0 |
| FR-AUTH-03 | 세션 보안(HttpOnly, Secure, SameSite, 재생성, 유휴 만료) | P0 |
| FR-AUTH-04 | 모든 상태 변경 요청에 CSRF 토큰 검증 | P0 |
| FR-AUTH-05 | 2단계 인증(TOTP) 선택 기능 | P2 |

### 6.2 기본 관리 (BASE)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-BASE-01 | 수급자 등록·조회·수정·종료 처리. 필수 항목 누락 검사 | P0 |
| FR-BASE-02 | 보호자 등록, 수급자와 다대다 연결, 주 연락 보호자 지정 | P0 |
| FR-BASE-03 | 요양보호사(직원) 등록·조회·수정·퇴사 처리 | P0 |
| FR-BASE-04 | 직원 자격증 등록(종류, 번호, 발급일, 만료일, 사본 파일) | P0 |
| FR-BASE-05 | 상담 기록 등록, 후속 연락일 지정 → 후속 업무 자동 생성 | P0 |
| FR-BASE-06 | 서비스 계약 등록(기간, 급여 종류, 장기요양등급, 인정 유효기간) | P0 |
| FR-BASE-07 | 욕구사정 기록 등록(구조화 항목 + 자유 서술), 버전 관리 | P0 |
| FR-BASE-08 | 주민등록번호 등 고유식별정보는 암호화 저장, 화면에서 마스킹 | P0 |

### 6.3 일정·서비스 기록 (SVC)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-SVC-01 | 방문 일정 등록(수급자, 직원, 날짜, 시작·종료 예정 시각, 급여 종류), 반복 일정 | P0 |
| FR-SVC-02 | 같은 직원의 시간 중복, 같은 수급자의 시간 중복 탐지 | P0 |
| FR-SVC-03 | 서비스 기록 입력: 실제 시각, 4개 영역(신체활동·인지관리·일상생활·정서지원) 완료 여부와 소요시간, 상태 관찰 4개 항목(신체·건강 / 식사·복약 / 인지·정서 / 배변). 형식은 [FORMS §1~2](FORMS.md) | P0 |
| FR-SVC-07 | 기록지 필수 항목 누락 검사 (`FORM.HOMECARE.REQUIRED`), 영역별 소요시간 합계와 방문 시간 비교 | P0 |
| FR-SVC-08 | 주간 상태변화기록 작성(점검 항목별 상태·변화 내용과 조치 사항), 일별 관찰값으로 자동 채운 초안 제공 | P1 |
| FR-SVC-04 | 서비스 기록 정정·삭제는 승인 요청을 거치고 변경 이력을 남김 | P0 |
| FR-SVC-05 | 일정 대비 기록 누락 탐지 ("기록 없음"과 "미제공"을 구분) | P0 |
| FR-SVC-06 | 모바일 반응형 기록 입력 화면 | P0 |

### 6.4 외부 자료 가져오기 (IMP)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-IMP-01 | CSV·Excel(xlsx) 업로드. 확장자·MIME·크기·실제 형식 검증 | P0 |
| FR-IMP-02 | 원본 파일을 `storage/private_files`에 해시와 함께 보관 (웹 직접 접근 불가) | P0 |
| FR-IMP-03 | 열 매핑 템플릿 관리 (공단 내려받기 파일 형식은 실제 파일 확인 후 등록) | P0 |
| FR-IMP-04 | 미리보기 → 필수 열·형식·중복 검사 → 오류 목록 → 담당자 확인 후 반영 | P0 |
| FR-IMP-05 | 수급자·직원 식별 매핑, 매핑 실패 목록 제공 | P0 |
| FR-IMP-06 | 같은 파일(해시 동일) 재업로드 차단, 배치 단위 되돌리기 | P0 |

### 6.5 검증·규칙 엔진 (RULE)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-RULE-01 | 규칙 등록·버전 관리(근거 문서, 조항, 출처 URL, 확인일, 시행 기간, 검증 상태) | P0 |
| FR-RULE-02 | 서비스 제공일 기준으로 적용 규칙 버전 선택 (소급 적용 금지) | P0 |
| FR-RULE-03 | `verified`가 아닌 규칙은 청구·정산 확정에 쓰지 않고 결과를 "확인 필요"로 표시 | P0 |
| FR-RULE-04 | 태그 기록 ↔ 서비스 기록 시간 비교 (허용 오차는 설정값, 법정 기준으로 단정하지 않음) | P0 |
| FR-RULE-05 | 검증 결과를 오류/경고/정보/확인 필요로 분류해 저장 | P0 |
| FR-RULE-06 | 규칙별 테스트 케이스 등록, 테스트 통과 전 활성화 금지 | P0 |

### 6.6 정산·본인부담금 (BILL)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-BILL-01 | PHP 결정론적 계산 엔진으로 수급자별 월 본인부담금 계산 | P0 |
| FR-BILL-02 | 본인부담률·감경 조건·수가는 시행 기간이 있는 규칙 데이터로 관리 (하드코딩 금지) | P0 |
| FR-BILL-03 | 계산 입력값·규칙 버전·결과를 스냅샷으로 저장해 재현 가능 | P0 |
| FR-BILL-04 | 자료나 규칙이 빠지면 "계산 보류" 상태와 사유 표시 | P0 |
| FR-BILL-05 | 전월 대비, 이전 계산 대비 차이 표시 | P0 |
| FR-BILL-06 | 청구액·수납액·미수금을 별도 항목으로 관리 | P0 |
| FR-BILL-07 | 정산 내역 CSV 내보내기, 인쇄용 화면 | P0 |
| FR-BILL-08 | 보호자 안내 문구 초안 생성 (템플릿 기반) | P1 |
| FR-BILL-09 | 정산 확정은 운영자 승인 필수 | P1 |

### 6.7 업무·알림·대시보드 (TASK / DASH)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-TASK-01 | 업무(task) 생성·배정·상태 변경·이력. 출처(규칙 엔진/AI/수동) 표시 | P0 |
| FR-TASK-02 | 행정 마감일 등록, D-30/D-7/D-1 알림 (기간은 설정값) | P0 |
| FR-TASK-03 | 시스템 내 알림. 외부 발송은 승인 후에만 | P0 |
| FR-DASH-01 | 메인 대시보드: 오늘 방문, 미완료 업무, 기록 누락, 태그 불일치, 청구 전 오류·경고, 만료 예정, 정산 현황, Agent 결과, 승인 대기 | P0 |
| FR-DASH-02 | 업무 목록 항목에 유형·대상·발견 시각·중요도·설명·원본 기록·적용 규칙·권장 조치·상태·담당자·이력 표시 | P0 |
| FR-DASH-03 | AI 발견 항목과 규칙 엔진 판정 항목을 시각적으로 구분 (배지·색상) | P0 |

### 6.8 AI Agent (AGT)

상세 명세는 [AGENT_SPEC](AGENT_SPEC.md).

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-AGT-00 | 공통 실행 엔진(큐, 상태, 멱등성, 스키마 검증, 감사 로그) | P0 |
| FR-AGT-01 | Daily Operations Agent — 규칙 기반 | P0 |
| FR-AGT-02 | Claim Validation Agent — 규칙 기반 | P1 |
| FR-AGT-03 | Care Plan Agent — **템플릿 모드**: 욕구사정 구조화 항목을 계획서 템플릿에 채우고 근거 연결·누락 표시. 자유 서술 생성은 보류 | P1 |
| FR-AGT-04 | Monthly Condition Report Agent — **템플릿 모드**: 주간 상태변화기록과 일별 관찰값을 PHP로 집계해 월간 보고서 초안 생성 | P1 |
| FR-AGT-05 | Billing & Copayment Agent — 계산은 엔진, 안내 문구는 템플릿 | P1 |
| FR-AGT-06 | Staff & Compliance Agent — 규칙 기반 (P0 일부: 자격증 만료) | P0/P1 |
| FR-AGT-07 | 상담 요약 (LLM 필요) | **보류** |
| FR-AGT-09 | LLM 모드 (Agent 3·4의 자연어 초안, 예외 설명, 안내 문구 다듬기) | **보류** (Q-LEGAL-01) |
| FR-AGT-08 | Agent 권한 레벨 L0~L4 적용, L3 이상은 PHP 서비스 + 사람 승인 | P0 |

### 6.9 승인·문서 (APPR / DOC)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-APPR-01 | 승인 대기 목록과 승인 화면(원본 자료, 계산 근거, 규칙 버전, AI 초안, 변경 diff 함께 표시) | P1 |
| FR-APPR-02 | 승인 필수 업무: 계획서 확정, 상태변화 문서 확정, 정산 확정, 외부 발송, 기록 정정·삭제, 청구 자료 확정, 규정 예외 처리 | P0(기록 정정)/P1 |
| FR-DOC-01 | 문서 상태 `draft → review_required → approved`, 승인 후 변경 시 새 버전 | P1 |
| FR-DOC-02 | 문서 화면: 원본 자료, AI 초안, 변경 이력, 근거 기록, 누락 항목, 승인 상태, 인쇄·내보내기 | P1 |

### 6.10 AI 비용 관리 (COST) — 보류

LLM 기능 보류(§0.1)에 따라 아래 요구사항도 보류한다. LLM 기능을 재개할 때 함께 구현한다.

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-COST-01 | 월 예산, 일 호출 수, 최대 입력·출력 토큰, 재시도 횟수 설정 | 보류 |
| FR-COST-02 | 호출별 토큰·모델·추정 비용 기록, 관리자 비용 대시보드 | 보류 |
| FR-COST-03 | 입력 해시 + 프롬프트 버전 기반 결과 재사용 | 보류 |
| FR-COST-04 | 예산 초과 시 비필수 AI 기능 차단, 핵심 ERP 기능은 계속 동작 | 보류 |

### 6.11 감사·운영 (AUD / OPS)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-AUD-01 | 중요 작업(로그인, 개인정보 조회·수정, 승인, 정산, 내보내기) 감사 로그 | P0 |
| FR-AUD-02 | 로그에 개인정보·전체 프롬프트를 남기지 않음 | P0 |
| FR-OPS-01 | cron + PHP CLI 일일·월간 작업, 중복 실행 잠금 | P0 |
| FR-OPS-02 | DB 백업·복구 절차 문서화와 복구 리허설 | P0 |
| FR-OPS-03 | DB 마이그레이션 도구(버전 테이블 기반 자체 러너) | P0 |

### 6.12 재무·고도화 (FIN / P2)

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-FIN-01 | 매출·비용·수납 기록, 월별 요약 | P2 |
| FR-P2-01 | 자연어 업무 검색 | P2 |
| FR-P2-02 | 규정 변경 검토 보조 (변경 후보 제시만, 자동 반영 금지) | P2 |
| FR-P2-03 | 직원 배정 보조 | P2 |
| FR-P2-04 | 방문목욕 확장 모듈 | P2 |
| FR-P2-05 | 방문간호 확장 모듈 (바이탈·간호 처치·지시서 이행, [FORMS §3](FORMS.md)). 수치는 기록만, 의료 판정 없음 | P2 |

## 7. 비기능 요구사항

| ID | 범주 | 요구사항 |
|---|---|---|
| NFR-01 | 성능 | 대시보드 첫 화면 2초 이내 (수급자 100명, 월 일정 3,000건 기준, 잠정) |
| NFR-02 | 가용성 | AI API 장애 시 ERP 기능 100% 유지 |
| NFR-03 | 신뢰성 | cron 중복 실행 시에도 결과 중복 없음 (멱등성) |
| NFR-04 | 재현성 | 과거 정산 결과를 저장된 입력·규칙 버전으로 동일하게 재계산 가능 |
| NFR-05 | 보안 | [SECURITY](SECURITY.md)의 필수 통제 전부 적용 |
| NFR-06 | 호환성 | 운영 서버 PHP/DB 버전에서 동작 (확인 후 확정) |
| NFR-07 | 반응형 | 360px 모바일부터 PC까지 |
| NFR-08 | 유지보수 | 외부 의존성 최소화. Composer 패키지는 승인된 것만 |
| NFR-09 | 시간대 | `Asia/Seoul` 기준. DB에는 로컬 시각 + 명시적 타임존 설정 |
| NFR-10 | 비용 | 현재 범위의 추가 AI API 비용 0원 (LLM 보류). 재개 시 월 10,000원 이내 목표 |

## 8. 핵심 시나리오

| 시나리오 | 흐름 요약 | 관련 요구사항 |
|---|---|---|
| A. 신규 수급자 등록 | 등록 → 필수항목 검사 → 욕구사정 → 계획서 초안(템플릿) → 검토·승인 → 버전 저장 → 후속 업무 생성 | BASE-01~07, AGT-03, AGT-07, DOC-01 |
| B. 매일 방문 관리 | 당일 일정 확인 → 일정·기록 비교 → 태그 검증 → 불일치 업무 등록 → 확인·정정(승인) → 이력 | SVC-*, IMP-*, RULE-04, AGT-01, AGT-02 |
| C. 월말 청구 준비 | 자료 수집 → 중복·누락·불일치 검사 → 규칙 점검 → 본인부담금 계산 → 오류·경고 표시 → 수정 → 재검증 → 승인 → 사람이 공식 시스템 제출 | AGT-02, AGT-05, BILL-*, APPR-01 |
| D. 월별 상태변화 보고 | 일별 관찰값 → 주간 상태변화기록 → 월간 취합(PHP 집계) → 변화·근거 연결 → 근거 부족 표시 → 검토·확정 | SVC-08, AGT-04, DOC-* |
| E. 매일 업무 브리핑 | 로그인 또는 아침 cron → 오늘 방문·누락·불일치·행정 업무·만료 예정·상담 후속·청구 전 항목·전날 미완료 | AGT-01, DASH-01 |

## 9. 요구사항 추적표 (3단계)

완료 조건(DoD)의 공통 항목: 코드 리뷰 완료, 관련 테스트 통과(실제 실행 결과 첨부), 감사 로그 확인, 문서 갱신.

| 요구사항 | 구현 모듈 | 관련 테이블 | Agent / 엔진 | 테스트 | 완료 조건 (개별) |
|---|---|---|---|---|---|
| FR-AUTH-01~04 | `Security/Auth`, `Security/Csrf`, `Controllers/AuthController` | users, login_attempts, audit_logs | — | T-SEC-01~06 | 권한 없는 접근 403, CSRF 누락 거부 |
| FR-BASE-01~02 | `Services/ClientService` | clients, guardians, client_guardians | 규칙: 필수항목 | T-BASE-01~03 | 필수 누락 시 저장 거부·안내 |
| FR-BASE-03~04 | `Services/EmployeeService` | employees, employee_credentials | 규칙: 자격 만료 | T-BASE-04, T-STAFF-01 | 만료 D-30 업무 생성 |
| FR-BASE-05 | `Services/ConsultationService` | consultations, tasks | — | T-BASE-05 | 후속일에 업무 자동 생성 |
| FR-BASE-06~07 | `Services/ContractService`, `AssessmentService` | service_contracts, assessments | — | T-BASE-06 | 버전 증가 확인 |
| FR-BASE-08 | `Security/Crypto` | clients(rrn_enc) | — | T-SEC-07 | DB 평문 없음, 화면 마스킹 |
| FR-SVC-01~02 | `Services/ScheduleService`, `Rules/ScheduleConflictRule` | service_schedules | 검증 엔진 | T-REC-08~09 | 중복 일정 경고 |
| FR-SVC-03~04 | `Services/ServiceRecordService` | service_records, record_revisions, agent_approvals | — | T-REC-10, T-APPR-04 | 정정 시 승인·이력 |
| FR-SVC-07 | `Rules/HomecareFormRule` | service_records, billing_checks | 규칙 엔진 | T-FORM-01~04 | 누락 항목 경고 |
| FR-SVC-08 | `Services/ConditionObservationService` | condition_observations | — | T-FORM-05~06 | 일별 관찰값으로 주간 초안 채움 |
| FR-SVC-05 | `Rules/MissingRecordRule` | service_schedules, service_records, billing_checks | 검증 엔진 | T-REC-02 | "기록 없음" 표시, 미제공 단정 없음 |
| FR-IMP-01~06 | `Services/Import/*`, `Validators/UploadValidator` | tag_import_batches, tag_records, import_files | 검증 엔진 | T-IMP-01~07 | 동일 해시 차단, 배치 롤백 |
| FR-RULE-01~03,06 | `Rules/RuleRegistry`, `Services/RuleVersionService` | rule_versions, rule_test_cases | 규칙 엔진 | T-RULE-01~06 | 미검증 규칙 확정 사용 불가 |
| FR-RULE-04~05 | `Rules/TagTimeMatchRule` | tag_records, service_records, billing_checks | 검증 엔진 | T-REC-01~07 | 분류·근거 저장 |
| FR-BILL-01~07 | `Services/Billing/CopaymentCalculator` | billing_rules, copayment_calculations, payment_records | 계산 엔진 | T-CALC-01~08 | 스냅샷 재계산 일치 |
| FR-BILL-08~09 | `Agents/BillingCopaymentAgent` | report_drafts, agent_approvals | Agent 5 | T-AGT-05-* | 승인 전 발송 불가 |
| FR-TASK-01~03 | `Services/TaskService`, `NotificationService` | tasks, task_events, notifications | — | T-TASK-01~03 | 출처 표시 |
| FR-DASH-01~03 | `Controllers/DashboardController` | (조회) | — | T-UI-01~03 | 9개 위젯 표시, AI/규칙 구분 |
| FR-AGT-00 | `Agents/AgentRunner`, `Agents/AgentRegistry` | agent_runs, agent_findings, ai_runs | 공통 엔진 | T-AGT-C-01~10 | 멱등·스키마 검증 |
| FR-AGT-01 | `Agents/DailyOperationsAgent` | tasks, agent_findings | Agent 1 | T-AGT-01-* | LLM 없이 브리핑 생성 |
| FR-AGT-02 | `Agents/ClaimValidationAgent` | billing_checks | Agent 2 | T-AGT-02-* | 원본 자동 수정 없음 |
| FR-AGT-03 | `Agents/CarePlanAgent` (템플릿 모드) | care_plans, documents | Agent 3 | T-AGT-03-* | 근거 링크, 누락 시 blocked |
| FR-AGT-04 | `Agents/ConditionReportAgent` (템플릿 모드) | condition_observations, report_drafts | Agent 4 | T-AGT-04-* | 집계값마다 근거 기록 링크 |
| FR-AGT-06 | `Agents/StaffComplianceAgent` | employee_credentials, tasks | Agent 6 | T-AGT-06-* | 자료 부족 시 "판정 불가" |
| FR-AGT-08 | `Agents/PermissionGate` | agent_approvals | 공통 엔진 | T-AGT-C-09, T-APPR-* | L3+ 우회 불가 |
| FR-APPR-01~02, FR-DOC-* | `Workflows/ApprovalWorkflow`, `Services/DocumentService` | agent_approvals, documents, document_versions | — | T-APPR-01~06 | 승인 후 수정 시 새 버전 |
| FR-COST-01~04 | (보류) | — | — | — | — |
| FR-AUD-01~02 | `Services/AuditLogger` | audit_logs | — | T-SEC-08~09 | 로그에 개인정보 없음 |
| FR-OPS-01~03 | `bin/*.php`, `database/migrate.php` | job_locks, schema_migrations | — | T-OPS-01~03 | 동시 실행 시 1회만 처리 |

> **규정 근거 확인 상태**: 본 문서는 장기요양 관련 법령·고시의 구체적 수치(본인부담률, 감경 비율, 수가, 시간 인정 기준 등)를 **하나도 확정하지 않았다**. 모든 규정 값은 [RULE_ENGINE_SPEC §6 규칙 후보 목록](RULE_ENGINE_SPEC.md)에 `unverified` 상태로 등록하고, 공식 출처를 확인한 뒤에만 `verified`로 바꾼다.

## 10. 릴리스 계획

| 단계 | 범위 | 진입 조건 |
|---|---|---|
| M0 | 환경 확인, 문서 승인 | 본 문서 승인, OPEN_QUESTIONS의 Blocker 답변 |
| M1 (P0-a) | 기반: 마이그레이션, 인증, 감사 로그, 수급자·직원·보호자 | M0 |
| M2 (P0-b) | 일정·기록·업로드·태그 비교·규칙 버전 | M1 |
| M3 (P0-c) | 계산 엔진, 업무·대시보드, Agent 공통 엔진, Daily Ops(규칙), Staff Compliance(규칙) | M2 + 본인부담 규칙 1건 이상 verified |
| M4 (P1) | Agent 2~5(규칙·템플릿 모드), 주간 상태변화기록, 승인 워크플로, 문서 버전 | M3 파일럿 1개월 안정 운영 |
| (보류) | LLM 연동, 비용 관리, 상담 요약, Agent 3·4 LLM 모드 | Q-LEGAL-01 답변 |
| M5 (P2) | 자연어 검색, 재무 분석, 방문목욕 등 | P1 정확성 확인 |

상세 작업 분해는 [IMPLEMENTATION_PLAN](IMPLEMENTATION_PLAN.md).

## 11. 위험

| 위험 | 영향 | 대응 |
|---|---|---|
| 운영 서버 환경 미확인 | 코드 비호환 | M0에서 확인. 확인 전 버전 의존 문법 사용 금지 |
| 규정 해석 오류 | 잘못된 청구 안내 | 규칙 검증 상태 관리, 미검증 규칙은 확정에 사용 안 함 |
| AI 환각 | 잘못된 문서 | 현재 범위에서는 LLM 미사용. 재개 시 근거 링크 필수, 스키마 검증, 사람 승인 |
| 개인정보 유출 | 법적 책임 | 가명처리, 최소 전송, 암호화, 접근 통제 |
| 공단 파일 형식 변경 | 가져오기 실패 | 열 매핑 템플릿을 데이터로 관리, 형식 검사 실패 시 중단 |
| 1인 운영자의 백업 누락 | 데이터 손실 | 자동 백업 cron + 백업 실패 알림 |
| 양식집이 공식 서식과 다를 가능성 | 필수 항목 누락 | 양식집 기반 규칙은 operational로 두고, 공식 필수 항목은 확인 후 regulatory로 별도 등록 |
| LLM 없이 문서 초안 품질 부족 | 시간 절감 효과 감소 | 관찰값을 선택형으로 구조화해 PHP 집계 품질 확보, 서술은 담당자가 보완 |
