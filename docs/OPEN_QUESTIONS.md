# CareFlow Lite — 미결 질문

- 문서 버전: 0.1 (초안) · 작성일: 2026-10-22
- **Blocker**: 답변 전 해당 단계 구현 불가 / **High**: 해당 기능 설계 확정에 필요 / **Normal**: 구현 중 결정 가능
- 추측으로 채우지 않은 항목이다. 답변을 받으면 이 문서와 관련 문서를 함께 갱신한다.

## 1. 환경 (ENV)

| ID | 질문 | 중요도 | 영향 |
|---|---|---|---|
| Q-ENV-01 | 운영 서버의 PHP 버전과 설치된 확장(pdo_mysql, mbstring, sodium, openssl, bcmath, zip, intl, fileinfo, curl)은? | Blocker(M1) | 문법·라이브러리 선택 |
| Q-ENV-02 | MySQL/MariaDB 정확한 버전은? (`SKIP LOCKED`, JSON 타입 지원 여부) | Blocker(M1) | 큐 구현, 컬럼 타입 |
| Q-ENV-03 | VPS인가 공유호스팅인가? DocumentRoot 변경, `.htaccess` AllowOverride, SSH 가능 여부는? | Blocker(M1) | 디렉터리 배치, 배포 |
| Q-ENV-04 | 배포 방식(git pull / FTP / rsync)과 Composer 사용 가능 여부는? | High | 의존성·배포 절차 |
| Q-ENV-05 | 저장소 이름이 `angular`인데 이 저장소에 PHP 프로젝트를 둘 것인가, 새 저장소(예: `careflow-lite`)를 만들 것인가? | High | 저장소 구조 |
| Q-ENV-06 | cron 등록 가능 여부와 최소 주기는? | High | Agent 실행 방식(cron vs 접속 시 지연 실행) |
| Q-ENV-07 | HTTPS 인증서와 도메인은 준비되어 있는가? | High | 세션 보안 |
| Q-ENV-08 | 서버 시간대 설정은 Asia/Seoul인가? | Normal | 시각 처리 |

## 2. 아키텍처 (ARCH)

| ID | 질문 | 중요도 | 기본안 |
|---|---|---|---|
| Q-ARCH-01 | 프레임워크 없이 경량 자체 구조로 갈 것인가, Slim 같은 마이크로 프레임워크를 쓸 것인가? | High | 프레임워크 없음 |
| Q-ARCH-02 | 테스트에 PHPUnit(Composer)을 써도 되는가? | High | PHPUnit |
| Q-ARCH-03 | Excel(xlsx) 지원이 꼭 필요한가? 필요하면 PhpSpreadsheet 도입 허용? | High | 1차 CSV, 2차 xlsx |
| Q-ARCH-04 | 사용할 LLM 제공자와 모델은? (비용·데이터 처리 정책 비교 필요) | High (P1) | 미정 |

## 3. 업무·규정 (BIZ / REG) — 공식 근거 확인 필요

| ID | 질문 | 중요도 | 비고 |
|---|---|---|---|
| Q-REG-01 | 본인부담 구분별 비율과 시행 기간의 공식 근거(법령·고시 조항, URL)는? | Blocker(정산 확정) | `COPAY.RATE` |
| Q-REG-02 | 감경·면제 대상 조건과 증빙은? | Blocker(정산 확정) | `COPAY.REDUCTION` |
| Q-REG-03 | 본인부담금 원 단위 처리(절사·반올림 단위) 근거는? | Blocker(정산 확정) | `COPAY.ROUNDING` |
| Q-REG-04 | 적용 연도의 방문요양 수가표(시간 구간별)와 고시 번호는? | Blocker(정산 확정) | `FEE.SCHEDULE` |
| Q-REG-05 | 등급별 월 한도액과 초과분 처리 기준은? | High | `BENEFIT.MONTHLY_LIMIT` |
| Q-REG-06 | 방문 시간 인정·산정 기준(태그 시각과 인정 시간의 관계)은? | High | 현재 태그 비교는 내부 허용 오차로만 처리 |
| Q-REG-07 | 급여제공계획서 공식 양식과 필수 항목은? | Blocker(Care Plan Agent) | `CAREPLAN.FORM_FIELDS` |
| Q-REG-08 | 월별 상태변화 기록의 공식 양식·필수 항목·작성 주기 근거는? | High | Agent 4 출력 형식 |
| Q-REG-09 | 직원 필수 서류와 필수 교육의 종류·주기는? | High | `STAFF.REQUIRED_DOCS`, `STAFF.TRAINING_DUE` |
| Q-REG-10 | 수급자 필수 서류(계약서, 동의서 등) 목록은? | High | `DOC.REQUIRED_MISSING` |
| Q-REG-11 | 평가(정기평가) 대비 관리해야 할 항목이 있는가? | Normal (P2) | |
| Q-BIZ-01 | 공단 시스템에서 내려받는 태그·청구 관련 파일의 실제 형식(샘플, 개인정보 제거본)은? | Blocker(M2 가져오기) | `import_templates` |
| Q-BIZ-02 | 태그 허용 오차(분)의 센터 내부 기준은? | Normal | 설정값, 기본 10분(안) |
| Q-BIZ-03 | 하루 같은 수급자에게 분할 방문(여러 번)이 있는가? | High | `service_records` UNIQUE 제약 |
| Q-BIZ-04 | 사무 보조, 요양보호사 계정이 실제로 필요한가? | Normal | RBAC 범위 |
| Q-BIZ-05 | 현재 수급자·직원 규모와 월 일정 건수는? | Normal | 성능 기준 |
| Q-BIZ-06 | 기존 엑셀 등 이관할 데이터가 있는가? 형식은? | High | 이관 도구 필요 여부 |
| Q-BIZ-07 | 서비스 기록 관찰 항목(식사·이동·위생·인지·정서 외)과 입력 방식(선택형/서술형)은? | High | Agent 4 품질 |

## 4. 법무·개인정보 (LEGAL)

| ID | 질문 | 중요도 |
|---|---|---|
| Q-LEGAL-01 | 가명처리된 건강 관련 기록을 외부 LLM API로 보내는 것에 대한 처리 근거, 국외 이전·위탁 고지 필요 여부, 제공자 약관(보존·학습 미사용) 검토 결과는? | Blocker(P1 AI 기능) |
| Q-LEGAL-02 | 데이터 종류별 보유 기간(서비스 기록, 계약서, 정산, 감사 로그)의 근거는? | High |
| Q-LEGAL-03 | 주민등록번호 수집이 필요한 법적 근거가 있는가? 없으면 수집하지 않는다 | High |
| Q-LEGAL-04 | 보호자 문자 발송 동의 양식과 절차는? | Normal |

## 5. 운영 (OPS)

| ID | 질문 | 중요도 |
|---|---|---|
| Q-OPS-01 | 백업을 보관할 서버 밖 저장소는? (다른 서버, 클라우드 스토리지 등) | High |
| Q-OPS-02 | 월 AI 예산 10,000원 목표 유지 여부, 초과 시 정책(차단 vs 알림만)은? | Normal |
| Q-OPS-03 | 문자 발송 서비스(제공자, 발신번호 등록)는? | Normal (P1) |
| Q-OPS-04 | 장애 시 연락·복구 담당은 운영자 1인인가? | Normal |
