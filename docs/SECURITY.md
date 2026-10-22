# CareFlow Lite — 보안 및 개인정보 보호

- 문서 버전: 0.1 (초안) · 작성일: 2026-10-22
- 재가센터 데이터에는 건강 정보(민감정보)와 주민등록번호 등 고유식별정보가 포함될 수 있다. 아래 통제는 **P0 필수**다.
- 이 문서는 기술적 통제를 정의한다. 개인정보 보호법 등 법적 의무의 해석·이행 여부는 운영자가 확인해야 한다 ([OPEN_QUESTIONS §4](OPEN_QUESTIONS.md)).

## 1. 데이터 분류

| 등급 | 예 | 저장 | 화면 | AI 전송 |
|---|---|---|---|---|
| S1 고유식별정보 | 주민등록번호, 장기요양인정번호, 자격증 번호 | 암호화 + HMAC | 마스킹, 원문은 admin 재인증 | **금지** |
| S2 민감정보 | 건강 상태, 관찰 기록, 욕구사정, 진단명 | 평문(DB 접근 통제) | 권한별 | 가명처리 후 필요 항목만 |
| S3 개인정보 | 이름, 전화, 주소, 생년월일 | 전화·주소 암호화 | 권한별 | 이름 → 임시 토큰, 나머지 제거 |
| S4 업무정보 | 일정, 업무, 규칙 | 평문 | | 필요 시 |

## 2. 접근 제어 (RBAC)

| 기능 | admin | staff | caregiver |
|---|---|---|---|
| 수급자 조회·등록 | ✔ | ✔ | 배정된 수급자 최소 정보 |
| 고유식별정보 원문 | ✔(재인증) | ✘ | ✘ |
| 일정 | ✔ | ✔ | 본인 일정 |
| 서비스 기록 작성 | ✔ | ✔ | 본인 기록 |
| 기록 정정·삭제 요청 | ✔ | ✔ | 본인 기록 정정 요청 |
| 승인 | ✔ | ✘ | ✘ |
| 정산·CSV 내보내기 | ✔ | 조회만 | ✘ |
| 규칙·설정·AI 예산 | ✔ | ✘ | ✘ |
| 감사 로그 | ✔ | ✘ | ✘ |

- 객체 단위 검사: 모든 조회·수정에서 `id`만 믿지 않고 소유·배정 관계를 확인한다 (IDOR 방지).
- 권한 검사는 Controller 진입 미들웨어 + Service 내부 이중 검사.

## 3. 인증·세션

- `password_hash(PASSWORD_DEFAULT)` / `password_verify`, `password_needs_rehash`.
- 로그인 실패: 같은 계정 5회/15분 → 15분 잠금, IP 기준 추가 제한 (값은 설정).
- 세션: `session.cookie_httponly=1`, `cookie_secure=1`, `cookie_samesite=Lax`, `use_strict_mode=1`, 로그인 시 `session_regenerate_id(true)`, 유휴 30분 만료.
- 중요 작업(L4 승인, 원문 조회, 설정 변경)은 비밀번호 재확인.
- HTTPS 전용 + HSTS. HTTP 요청은 리다이렉트.

## 4. 웹 취약점 방어

| 위협 | 통제 |
|---|---|
| SQL Injection | PDO prepared statements만 사용, `EMULATE_PREPARES=false`. 동적 정렬·컬럼은 화이트리스트 |
| XSS | 모든 출력 `htmlspecialchars(ENT_QUOTES, 'UTF-8')` 헬퍼, JSON은 `json_encode(JSON_HEX_TAG\|JSON_HEX_AMP\|...)`. CSP 헤더(`default-src 'self'`). AI 생성 텍스트도 동일하게 escape |
| CSRF | 세션별 토큰, 상태 변경 요청 전부 검증, `SameSite` 쿠키 |
| 클릭재킹 | `X-Frame-Options: DENY` / CSP `frame-ancestors 'none'` |
| 헤더 | `X-Content-Type-Options: nosniff`, `Referrer-Policy: same-origin` |
| 오류 노출 | 운영 `display_errors=Off`, 사용자에게 일반 메시지, 상세는 로그 |
| 경로 노출 | DocumentRoot=`public/`. `config/`, `storage/`, `prompts/`, `.env`, `.git` 웹 접근 불가 |

## 5. 파일 업로드

1. 확장자 화이트리스트: csv, xlsx (자격증 사본: pdf, jpg, png)
2. 크기 제한: 10MB (설정)
3. 실제 유형 검사: `finfo_file` MIME + 매직 바이트, xlsx는 ZIP 구조 확인
4. 저장 이름은 무작위(UUID), 원본 이름은 DB에만
5. 저장 위치 `storage/private_files/YYYY/MM/` (웹 루트 밖), 권한 0640
6. 다운로드는 권한 확인 후 PHP가 스트리밍 (`Content-Disposition: attachment`)
7. CSV 내보내기 시 수식 인젝션 방지: `=`, `+`, `-`, `@`로 시작하는 셀 앞에 `'` 추가
8. 엑셀 파서는 외부 엔티티·매크로 무시, 행 수 상한

## 6. AI API 전송 통제

- **기본 원칙**: 식별 가능한 수급자 정보는 전송하지 않는다.
- `Redactor` 단계 ([AGENT_SPEC §1.6](AGENT_SPEC.md)):
  - S1: 제거 / S3: 이름 → 임시 토큰, 연락처·주소 제거, 생년월일 → 연령대 / 날짜 → 상대 일자
  - S2: 해당 작업에 필요한 필드만 화이트리스트로 추출
  - 정규식 2차 검사(주민번호·전화번호·이메일 패턴) 적중 시 전송 중단
- 전송 로그(`ai_runs`)에는 입력 해시와 제거 항목 **개수**만 저장, 프롬프트 원문·응답 원문 저장 안 함 (디버그 모드도 개인정보 없는 합성 데이터에서만).
- 가명처리했다고 개인정보 관련 의무가 모두 사라진다고 보지 않는다. 다음은 운영 전 확인 필요: 국외 이전 해당 여부, 처리 위탁 고지, LLM 제공자의 데이터 보존·학습 정책, 계약 조건 ([Q-LEGAL-01](OPEN_QUESTIONS.md)).
- 운영자 승인 없는 외부 데이터 전송 금지. AI 기능은 `settings.ai.enabled`로 전체 차단 가능.

## 7. 암호화와 키 관리

- 필드 암호화: `sodium_crypto_secretbox` (libsodium, PHP 7.2+ 기본 포함 — 서버 확장 확인 필요) 또는 `openssl` AES-256-GCM.
- 검색용: `hash_hmac('sha256', 정규화값, HMAC_KEY)`.
- 키는 웹 루트 밖 `.env` 또는 서버 환경변수. Git·DB·백업과 분리 보관.
- 키 교체 절차: 새 키 버전으로 재암호화 배치 (암호문에 키 버전 접두).

## 8. API 키·비밀값

- `.env`(Git 제외), 권한 0600, 소유자 웹 서버 사용자.
- `.gitignore`와 pre-commit 검사(키 패턴 grep)로 커밋 방지.
- 로그·오류 메시지에 키 출력 금지.

## 9. DB 계정

- 애플리케이션 계정: 해당 DB에 SELECT/INSERT/UPDATE/DELETE만. DDL 권한 없음.
- 마이그레이션 계정: 별도, 배포 시에만 사용.
- `audit_logs`, `record_revisions`: 가능하면 애플리케이션 계정의 UPDATE/DELETE 권한 회수 (MySQL 테이블 단위 GRANT).

## 10. 감사 로그

기록 대상: 로그인 성공·실패, 로그아웃, 고유식별정보 원문 조회, 수급자·직원 생성·수정·종료, 기록 정정·삭제 요청과 승인, 정산 계산·승인, CSV·인쇄 내보내기, 업로드 반영·롤백, 규칙 상태 변경, 설정 변경, Agent 실행, 외부 발송.
- 값 대신 변경 필드명만 기록. 보존 기간은 [Q-LEGAL-02](OPEN_QUESTIONS.md) 답변 후 설정.

## 11. 백업·복구

- 매일 `mysqldump --single-transaction` + `storage/private_files` 증분 백업.
- 백업 파일 암호화(gpg 등) 후 서버 밖 저장소로 전송 (위치는 [Q-OPS-01](OPEN_QUESTIONS.md)).
- 보관: 일 7개, 주 4개, 월 6개 (안).
- 분기 1회 복구 리허설, 결과 기록.
- 백업 실패 시 대시보드 알림.

## 12. 보유·파기

- 데이터 종류별 보유 기간은 법령 확인 후 `settings.retention.*`에 등록.
- 파기 배치: 대상 조회 → 운영자 확인(승인) → 물리 삭제·파일 삭제 → 감사 로그.

## 13. 외부 발송

- 문자·메시지는 `notifications(pending_approval)` → admin 승인(L4) → 발송 모듈.
- 수신 동의(`guardians.sms_consent`) 없으면 발송 차단.
- 발송 서비스·계약은 [Q-OPS-03](OPEN_QUESTIONS.md).

## 14. 보안 점검 체크리스트 (구현 각 단계 완료 시)

- [ ] 새 SQL이 전부 prepared statement인가
- [ ] 새 출력이 escape 헬퍼를 거치는가
- [ ] 새 POST 경로에 CSRF·RBAC가 있는가
- [ ] 객체 접근에 소유 관계 검사가 있는가
- [ ] 로그에 개인정보가 없는가
- [ ] 업로드·다운로드 경로가 웹 루트 밖인가
- [ ] AI 전송 데이터가 Redactor를 거치는가
