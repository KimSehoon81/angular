# CareFlow Lite — 아키텍처

- 문서 버전: 0.2 (초안) · 작성일: 2026-10-22 · 개정: 2026-10-23 (cron 가능 확인, 새 저장소, LLM 보류)
- 전제: 저장소에 기존 코드가 없음 ([PRD §0](PRD.md)). 운영 서버 버전은 미확인이며, 잠정 기준은 PHP 8.1+, MySQL 8.0+ / MariaDB 10.6+.

## 1. 기술 스택

| 계층 | 선택 | 근거 |
|---|---|---|
| Web Server | Apache 2.4 (mod_rewrite) | APM 고정 |
| Backend | PHP (프레임워크 없음, 경량 자체 구조) | 1인 유지보수, 공유호스팅 호환. 프레임워크 도입은 [Q-ARCH-01](OPEN_QUESTIONS.md) 결정 후 |
| DB | MySQL / MariaDB, InnoDB, utf8mb4 | APM 고정, 트랜잭션·외래키 |
| DB 접근 | PDO + Prepared Statements (`ATTR_EMULATE_PREPARES=false`) | SQL Injection 방어 |
| Frontend | 서버 렌더링 PHP 템플릿 + 바닐라 JS + CSS | 빌드 도구 불필요 |
| Background | Linux cron + PHP CLI (`bin/`) | 상시 프로세스 없음 |
| AI | 외부 LLM HTTP API (cURL) | **보류** — 이번 범위에서 호출하지 않음 (`ai.enabled=false`) |
| 테스트 | PHPUnit (Composer, dev 의존성) | 환경 비호환 시 자체 경량 러너 ([Q-ARCH-02](OPEN_QUESTIONS.md)) |
| 엑셀 읽기 | 1안: PhpSpreadsheet / 2안: CSV만 지원 | [Q-ARCH-03](OPEN_QUESTIONS.md) |

**도입하지 않는 것**: Node.js/Python 백엔드, 상시 AI 서버, Redis·RabbitMQ 등 메시지 브로커, SPA 프레임워크.

## 2. 시스템 구성도

```
[브라우저(PC/모바일)]
        │ HTTPS
        ▼
[Apache] ── DocumentRoot = /public  (그 밖의 디렉터리는 웹 접근 불가)
        │
        ▼
public/index.php (Front Controller)
        │
  Router → Middleware(Session, Auth, CSRF, RBAC) → Controller
        │
        ▼
  Services ───────────┬─────────────┬──────────────┐
        │             │             │              │
  Repositories   Rules(규칙엔진)  Workflows      Agents(AgentRunner)
        │             │          (승인)            │
        ▼             ▼                            ├─ PermissionGate
     [MySQL] ◄────────┴────────────────────────────┤
                                                   └─ LlmClient ─► [외부 LLM API]
                                                         ▲          (가명처리된 입력만)
                                                    BudgetGuard
                                                    Redactor

[cron] ─► bin/daily_jobs.php / bin/monthly_jobs.php / bin/agent_worker.php
           (job_locks로 중복 실행 방지, agent_runs 큐 소비)
```

## 3. 디렉터리 구조

```
/
├── public/                 # 유일한 웹 루트
│   ├── index.php           # Front Controller
│   ├── .htaccess           # 모든 요청 → index.php, 디렉터리 목록 금지
│   └── assets/             # css, js, 아이콘 (정적 파일만)
├── app/
│   ├── Controllers/        # HTTP 입출력만 담당, 비즈니스 로직 없음
│   ├── Services/           # 업무 로직, 트랜잭션 경계
│   │   ├── Billing/        # CopaymentCalculator 등 결정론적 계산
│   │   ├── Import/         # 업로드·파싱·매핑
│   │   └── Ai/             # (보류) LlmClient, BudgetGuard, Redactor, PromptRegistry — 인터페이스만
│   ├── Repositories/       # SQL은 여기에만
│   ├── Models/             # 값 객체·DTO
│   ├── Rules/              # 규칙 구현 클래스 + RuleRegistry
│   ├── Agents/             # AgentRunner, AgentInterface, 6개 Agent
│   ├── Workflows/          # ApprovalWorkflow, DocumentWorkflow
│   ├── Validators/         # 입력·업로드·JSON 스키마 검증
│   ├── Security/           # Auth, Csrf, Rbac, Crypto, Session
│   └── Views/              # PHP 템플릿 (출력 시 escape 헬퍼 필수)
├── config/                 # config.php (비밀값은 .env 또는 서버 밖 파일에서 읽음)
├── database/
│   ├── migrations/         # 0001_create_users.sql ...
│   ├── seeds/              # 개발용 가짜 데이터만
│   └── migrate.php
├── prompts/                # 버전 붙은 프롬프트 템플릿 (웹 접근 불가)
├── storage/
│   ├── logs/
│   ├── private_files/      # 업로드 원본, 자격증 사본
│   ├── exports/            # 일시 보관 후 삭제
│   └── agent_runs/         # 디버그용 (개인정보 없는 메타데이터만)
├── bin/
│   ├── daily_jobs.php
│   ├── monthly_jobs.php
│   ├── agent_worker.php
│   └── backup.sh
├── tests/
│   ├── Unit/
│   ├── Integration/
│   └── fixtures/           # 가짜 데이터만
├── docs/
└── CLAUDE.md
```

공유호스팅처럼 DocumentRoot를 바꿀 수 없으면, 루트 `.htaccess`로 `public/` 외 경로를 전부 `Require all denied` 처리하는 대안을 쓴다 ([Q-ENV-03](OPEN_QUESTIONS.md)).

## 4. 계층 규칙

1. Controller → Service → Repository 방향으로만 호출한다.
2. SQL은 Repository에만 둔다. Agent와 Controller는 SQL을 쓰지 않는다.
3. 트랜잭션은 Service에서 시작·종료한다.
4. Agent는 **읽기 전용 Service 인터페이스**와 `FindingWriter`, `DraftWriter`만 주입받는다. 쓰기 권한이 있는 Service는 `PermissionGate`를 거친다.
5. LLM 응답은 `Validators/JsonSchemaValidator`를 통과해야 DTO로 바뀐다.
6. 금액 계산은 `Services/Billing`만 한다. 금액은 원 단위 정수(`int`)로 다루고 비율은 정수 basis point 또는 문자열 십진수로 저장한다(부동소수점 금지).

## 5. 요청 흐름 (HTTP)

```
요청 → Session 시작·검증 → 인증 확인 → CSRF 확인(POST/PUT/DELETE)
     → RBAC 확인 → Controller → Service(트랜잭션) → 감사 로그 → 응답(escape 출력)
```

## 6. 배치·Agent 실행 흐름

```
cron(매일 06:00)  → bin/daily_jobs.php
                     ├─ job_locks 획득 (실패 시 종료)
                     ├─ 이벤트 수집 → agent_runs에 queued 행 생성 (idempotency_key UNIQUE)
                     └─ 잠금 해제
cron(5분마다)     → bin/agent_worker.php
                     ├─ queued 행을 SELECT ... FOR UPDATE SKIP LOCKED (MariaDB 10.6+/MySQL 8.0+)
                     │   (미지원 시 UPDATE ... SET status='running', locked_by=? WHERE status='queued' LIMIT 1)
                     ├─ AgentRunner.run()
                     └─ 실행 시간 한도(예: 240초) 초과 전 종료
cron(매월 1일)    → bin/monthly_jobs.php  (상태변화 보고, 정산 준비)
cron(매일 02:00)  → bin/backup.sh
```

운영 서버에서 cron 사용이 가능함을 확인했다(2026-10-23). 따라서 cron을 유일한 예약 실행 방식으로 쓰고, 접속 시 지연 실행(LazyJobTrigger)은 만들지 않는다. 대시보드는 마지막 일일 작업 실행 시각을 표시하고, 26시간 이상 실행이 없으면 경고를 띄운다. 최소 주기는 [Q-ENV-10](OPEN_QUESTIONS.md).

이벤트 기반 실행(일정 변경, 기록 저장 등)은 Service가 같은 트랜잭션 안에서 `agent_runs`에 행을 넣는 방식(아웃박스 패턴)으로 처리한다.

## 7. 설정과 비밀값

- `config/config.php`는 Git에 포함, 비밀값은 포함하지 않는다.
- 비밀값(DB 비밀번호, LLM API 키, 암호화 키)은 웹 루트 밖 `.env`(Git 제외) 또는 Apache `SetEnv`로 주입한다.
- `.gitignore`: `.env`, `storage/*`, `vendor/`(배포 방식에 따라), 백업 파일.

## 8. 확장 방식 (방문목욕 등)

- 급여 종류를 `service_type` 코드로 관리하고, 급여 종류별 규칙·계산기·서비스 기록 항목을 플러그인 형태 클래스로 등록한다 (`Rules/`, `Services/Billing/Calculators/`).
- 새 급여 종류 추가 시 기존 테이블 구조는 바꾸지 않고 `service_type` 값과 규칙 데이터만 추가하는 것을 목표로 한다.

## 9. 저장소

- 운영자가 새 저장소를 만든다(2026-10-23 결정). 현재 `angular` 저장소에는 설계 문서만 있고, 새 저장소가 생기면 `docs/`와 `CLAUDE.md`를 옮긴 뒤 애플리케이션 코드는 새 저장소에서만 작성한다.

## 10. 배포

- 1안: 서버에서 `git pull` + `php database/migrate.php` (SSH 가능 시)
- 2안: 로컬 빌드 후 rsync/FTP, 마이그레이션은 관리자 화면의 보호된 실행 버튼(admin + 재인증)
- 배포 전 DB 백업 필수. 마이그레이션은 앞으로만 진행하고, 되돌리기 SQL을 함께 작성한다.

배포 방식은 [Q-ENV-04](OPEN_QUESTIONS.md) 답변 후 확정한다.
