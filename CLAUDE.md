# CareFlow Lite — Claude Code 작업 지침

APM(Apache + PHP + MySQL/MariaDB) 기반 재가방문센터 ERP + AI Agent 업무 자동화 시스템.

## 현재 단계

- 설계 문서 단계. `docs/IMPLEMENTATION_PLAN.md`가 승인되기 전에는 애플리케이션 코드를 작성하지 않는다.
- 운영 서버의 PHP·DB 버전은 미확인(`docs/OPEN_QUESTIONS.md` Q-ENV-*). 확인 전까지 버전 의존 문법 사용 금지.
- 이 저장소(`angular`)에는 설계 문서만 둔다. 코드는 운영자가 만들 새 저장소에서 작성한다.
- cron 사용 가능. 예약 실행은 cron만 쓴다.
- **외부 LLM 호출은 보류**(개인정보 전송 검토 전). Agent는 규칙·템플릿 모드만 구현하고 `ai.enabled`는 false로 둔다.
- 서비스 기록·상태변화기록 구조는 `docs/FORMS.md`(운영자 제공 양식집 기반)를 따른다.
- 기록 입력은 `docs/DATA_ENTRY.md`의 단계(이중 입력 제거 → 모바일 직접 입력 → 사전 인쇄 시트·빠른 입력)를 따른다. 종이·전자 둘 다 받는 구조로 설계하고, 전자 서명·원본 보관 요건 확인 전에는 종이를 없애지 않는다. 지난 방문 값은 제안으로만 제공하고 자동 확정하지 않는다.
- 공단 제출·보고용 프로그램(요양사랑 등)은 계속 쓴다. CareFlow는 대체가 아니라 제출 전 점검·제출 후 대사·마감 관리를 맡는다 (`docs/INTEGRATION.md`). 제출 프로그램의 파일 형식은 운영자 샘플로 검증하기 전에 가정하지 않는다. 공단·제출 프로그램의 계정·인증서를 저장하거나 자동 조작하지 않는다.

## 문서

`docs/PRD.md`(요구사항·추적표) · `FORMS.md` · `DATA_ENTRY.md` · `INTEGRATION.md` · `ARCHITECTURE.md` · `DATA_MODEL.md` · `API_SPEC.md` · `AGENT_SPEC.md` · `RULE_ENGINE_SPEC.md` · `SECURITY.md` · `TEST_PLAN.md` · `OPEN_QUESTIONS.md` · `IMPLEMENTATION_PLAN.md`

## 반드시 지킬 것

- 금액 계산, 날짜 비교, 누락 탐지, 일정 충돌은 PHP로. AI가 금액을 만들거나 고치지 않는다.
- 장기요양 규정 수치(본인부담률, 수가, 한도 등)를 코드에 하드코딩하거나 추측으로 채우지 않는다. `rule_versions`에 근거와 함께 등록하고, `verified`만 확정에 쓴다.
- SQL은 Repository에서 PDO prepared statement로만. 출력은 escape 헬퍼로.
- Agent는 L0~L2만. L3 이상은 승인 요청만 만들고, 실행은 사람이 승인한 뒤 Service가 한다.
- LLM 입력은 Redactor를 거친다. 실제 수급자 식별정보를 전송하지 않는다. 로그에 개인정보·프롬프트 원문을 남기지 않는다.
- DB 변경은 `database/migrations/`로만.
- 테스트는 합성 데이터만 쓰고, 실제로 실행한 결과만 보고한다.
- 외부 시스템(국민건강보험공단 등) 자동 조작·제출 금지.
