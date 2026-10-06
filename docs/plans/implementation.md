# 개발 계획과 인계 — API 기능 기준

## 현재 상태

- 하네스와 제품 명세 정리는 완료됐다.
- 제품 웹·Spring·FastAPI·운영 서비스와 실제 API 구현은 아직 시작하지 않았다.
- `API명세서/API명세서`의 API 문서는 모두 `시작 전`이다.
- 계획은 API 그룹별 파일로 나누고, 파일명은 기능을 설명하는 이름으로 정한다.

## 하네스 개선과 CI 선구축

초기 하네스 정리 이후의 결함과 개선 과제는 [2026-09-23 평가](../reviews/harness-assessment-2026-09-23.md) 및 [하네스 개선 계획](harness-improvements.md)에서 관리한다.
CI/CD 구성·정적 검사는 서비스 개발 전에 준비할 수 있고, 테스트·빌드는 실행 가능한 골격·명령부터 연결한다(DEC-013).
이 작업 흐름은 아래 제품 API 개발과 병행할 수 있다. 하네스 개선 전체 완료를 제품 개발의 새 선행 조건으로 만들지 않는다.

## API 그룹별 계획

| API 그룹 | 계획 파일 | API 범위 | 담당자 | 상태 |
| --- | --- | --- | --- | --- |
| 인증 | [auth.md](auth.md) | `/api/v1/auth/*` | 강민우 | 진행 중 (`feat/datagsm-oauth`) |
| 상태 확인 | [health-check.md](health-check.md) | `/api/v1/health` | 김준수 | 미착수 |
| 학생·출석 | [student-attendance.md](student-attendance.md) | `/api/v1/student`, `/api/v1/attend` | 김준수 | 미착수 |
| 호실 명단 | [room-roster.md](room-roster.md) | `/api/v1/room/student` | 임서하 | 미착수 |
| QR 출석 | [qr-attendance.md](qr-attendance.md) | `/api/v1/qr*` | 김성찬 | 서버·웹 연동 완료, PC 실기 확인(로그인 복귀 포함), 휴대폰 카메라 확인 전 |
| 얼굴 인식 | [face-recognition.md](face-recognition.md) | `/api/v1/face/*` | 임서하 | Spring·AI 코드 구현, 배포 AI protected API·브라우저 제품 E2E 미검증 |
| 봉사 관리 | [volunteer-management.md](volunteer-management.md) | `/api/v1/volunteer/*` | 강민우 | 서버 PR 리뷰 중(CheckUp-server#76), 웹 연결 전 |
| DataGSM 동기화 | [datagsm-webhook.md](datagsm-webhook.md) | `/api/v1/webhook` | 강민우 | 미착수 |
| 알림 | [notification.md](notification.md) | `/api/v1/notifications*` | 강민우 | 서버 완료(조회·읽음 API, 출석 완료 알림, 08:00 폐기), 웹 연결 전, 봉사·공지 알림 연결 전 |

## 공통 선행 작업

- [ ] 인증 헤더, 사용자 식별자 타입, 공통 오류 envelope, `requestId`를 정한다.
- [ ] 서버 UTC 시각과 Asia/Seoul 08:00 운영일 계산을 공통화한다.
- [x] `DataGSM student.id → studentId`, `student.studentNumber → studentNumber`, `student.dormitoryRoom → dormitoryRoom` 매핑을 고정한다.
- [x] `student.role`·`teacher.department`로 서비스 관리자 권한을 판정하고 최상위 `role`은 계정 역할로만 취급한다.
- [x] `status → accountStatus`, OAuth callback 검증값 `state → oauthState`를 분리한다.
- [ ] DB migration, 테스트용 clock, 민감정보 없는 로그·fixture를 준비한다.
- [ ] 실제 구현 뒤 비어 있지 않은 OpenAPI 계약을 `contracts/`에 생성한다. QR은 [qr.openapi.yaml](../../contracts/qr.openapi.yaml)로 생성했다.
- [ ] 관리자·본인·본인 호실 권한을 서버에서 검증한다.

## DataGSM 연동 구현 기준

- `DataGsmOAuthClient`는 authorization code 교환, `userinfo` 조회, 로그인 사용자 매핑만 담당한다.
- `DataGsmStudentClient`는 `GET https://openapi.datagsm.kr/v1/students`를 `X-API-KEY`와 `STUDENT_READ` 권한으로 호출하고 페이지네이션을 처리한다.
- DataGSM 원본 DTO와 내부 Principal/Student 모델을 분리한다. 원본 숫자형 `id`는 `Long`으로 받고, 내부 계약이 문자열이면 어댑터에서만 변환한다.
- 내부 매핑은 다음을 따른다.
  - `externalUserId ← userinfo.id`
  - `studentId ← userinfo.student.id`
  - `studentNumber ← userinfo.student.studentNumber`
  - `name ← student.name | teacher.name`
  - `dormitoryRoom ← student.dormitoryRoom`
  - `accountStatus ← userinfo.status`
  - `subjectType ← userinfo.objectType`
- `STUDENT`는 `student.role == DORMITORY_MANAGER`, `TEACHER`는 `teacher.department == DORMITORY`일 때만 서비스 관리자다.
- `status != ACTIVE`, `objectType`와 중첩 객체 불일치, 지원하지 않는 사용자 유형은 인증 실패로 처리한다.
- AI `recognition.studentId`는 백엔드 canonical `studentId` 계약을 유지하고, `UNKNOWN`은 `studentId: null`로 전달한다.
- `userinfo`를 전체 학생 명단으로 사용하지 않는다. 학생 OpenAPI의 실제 권한·페이지네이션·졸업/전학/퇴사 신호를 연동 검증으로 남긴다.

## 기능 간 의존성

인증과 DataGSM 동기화가 먼저다. 이후 학생·출석, 호실, QR, 얼굴, 봉사 기능을 병렬 진행하고 마지막에 웹·운영·수용 검증을 연결한다.

## API 문서와 제품 명세의 보완 목록

1. QR 발급 문서에 `purpose`, 독립 세션 ID, lease/heartbeat, 종료, 15분 갱신이 없다. → [qr-attendance.md](qr-attendance.md) 관리자 API 응답에 반영했다.
2. QR 출석 문서에 운영일·중복 결과·현재 사용자 범위가 없다. → [qr-attendance.md](qr-attendance.md) 스캔 API에 반영했다.
3. Spring 얼굴 API 계약은 [face.openapi.yaml](../../contracts/face.openapi.yaml)에, AI 내부 계약은 AI 저장소 `contracts/ai-face.openapi.yaml`에 기록한다. 실제 제품 흐름 검증은 남아 있다.
4. 온라인 얼굴 인식 출석은 Spring에서 구현했다. 오프라인 임시 기록과 복구 후 동기화 계약·정책은 아직 확인되지 않았다.
5. 관리자 호실 수동 출석 저장 API가 없다.
6. 공지 CRUD API가 없다. 내부 알림 API는 [알림 계획](notification.md)으로 정했다(DEC-019).
7. 호실 API는 단일 호실 조회만 정의해 관리자 층 전개도 전체 조회를 직접 지원하지 않는다.
8. webhook의 event 값, 서명 방식, old/new 실제 필드, 재전송 idempotency가 미정이다.
9. 공통 오류 envelope가 정해지지 않았다. 모든 API는 `/api/v1` prefix를 붙인다. 인증은 세션 쿠키 방식으로 정해져 `RefreshToken` 헤더는 쓰지 않는다(DEC-016).
10. 봉사 증가·차감 API의 재시도 idempotency와 0회 하한 검증을 구현 계약에 반영한다. UI 노출은 SRC-NOTION-CHECKUPZIP 승인으로 `+ / −` 모두 확정됐다.

없는 경로를 임의로 구현하지 않고, 제공자·소비자·관련 REQ·수용 시나리오를 정한 뒤 `contracts/`에 반영한다.

얼굴 AI 연결의 상세 진척과 미검증 항목은 [얼굴 인식 계획](face-recognition.md)을 따른다. browser→Spring→AI 제품 E2E를 구현 완료로 집계하지 않는다.

## 웹·운영 통합

- [ ] OAuth, 학생 홈·마이·QR, 관리자 홈·QR·얼굴·봉사 화면을 각 API와 연결한다.
- [ ] 로딩·빈 상태·권한 부족·만료·중복·일시 장애를 오류 코드와 분리한다.
- [ ] 카메라 track·타이머·구독을 이탈/로그아웃 때 정리한다.
- [ ] Docker Compose, PostgreSQL/Redis 볼륨, health check, secret 주입을 구성한다.
- [ ] GSM SV 권한·VM 만료·자원·포트/TLS·OAuth callback을 확인한다.
- [ ] 원본 얼굴·당일 출석·임시 기록을 백업에서 제외하고 삭제 복원을 검사한다.

## 완료 기준

- API 요청·응답·권한·오류가 각 계획과 `contracts/`에 연결된다.
- 출석은 학생+용도+운영일 DB 원자성과 가장 이른 유효 기록 규칙을 갖는다.
- 얼굴 원본·프레임·벡터·당일 출석의 수명을 DB·캐시·로그·백업까지 검증한다.
- 실제 제품 테스트 전에는 수용 시나리오를 `verified`로 표시하지 않는다.
- 실제 서비스 테스트 후 `npm run harness:check`를 실행한다.

## 실행 기록

| 시점 | 실행 | 결과 |
| --- | --- | --- |
| 2026-09-21 | `npm run harness:sync` | 공통 스킬 동기화 완료 |
| 2026-09-21 | `npm run harness:check` | 하네스·명세·스킬 검사 통과, 제품 서비스는 미구현 |
| 2026-09-22 | API 명세 폴더 대조 | 8개 API 그룹별 기능 계획으로 재편 |
| 2026-09-23 | CI 선구축 지침 반영·공식 문서/공개 저장소 비교·로컬 진단 | DEC-013 추가, 평가와 개선 계획 작성. 기준 검사 20/20 통과와 별개로 훅 입력/경로 및 검증 증빙의 허점을 재현. 실제 에이전트 새 세션·원격 CI·제품 실행은 미검증. |
| 2026-09-24 | Notion ZIP 항목 3개 사용자 승인 반영 | 얼굴 자동 촬영, 관리자 휴대폰 5탭, 봉사 횟수 `+ / −`를 출처·명세·수용 시나리오·분야별 계획에 반영. 제품 코드는 미구현. 이번 변경 뒤 자동 검사는 실행하지 않음. |
| 2026-09-26 | DataGSM 연동 구조 수정안 반영 | userinfo 중첩 매핑·권한 판정·OAuth state 분리·학생 OpenAPI 경계와 수용 시나리오 갱신 |
| 2026-09-26 | `npm run harness:check` | 하네스 문서 검사 및 테스트 통과, 제품 서비스는 미구현 |
| 2026-09-27 | DataGSM 인증 구현 내용 반영 | 서버 세션 방식 DEC-016, 인증 경로·계약·환경변수·도메인 모델 갱신. 명세와 다른 코드 부분은 auth.md에 기록. 서버 인증 자동 테스트는 없음 |
| 2026-09-27 | 서버 인증 테스트·CI 수정 | 서버 `AuthServiceTest` 역할 판정 11개 통과, Server CI에 테스트용 DataGSM 환경변수 추가 후 통과. 웹훅 권한 반영(#25)과 로그인 후 복귀(#26)는 서버 이슈로 분리 |
| 2026-10-04 | 전교생 미리 저장 결정 반영 | 로그인 전 재학생을 계정 없이 저장하고 로그인 때 연결하는 결정과 동의 범위(얼굴 정보)를 REQ-AUTH-002·004, ACC-AUTH-002, DEC-026, SRC-PRESTORE-STUDENTS, room-roster 계획에 반영. 서버 구현은 CheckUp-server#119에서 테스트, 실제 DataGSM 동기화 확인은 없음 |
| 2026-10-04 | 학생 화면 서버 연동 내용 반영 | `/api/v1/auth/me`의 `student`(CheckUp-server#90)를 auth 계획·계약에, 호실 명단 `attended`·`purpose`(CheckUp-server#101)를 room-roster 계획·`contracts/room.openapi.yaml`에 기록. 학생 홈 출석 용도를 기숙사 입소로 정한 사용자 결정을 REQ-UI-003·ACC-UI-003·DEC-023·SRC-STUDENT-HOME-PURPOSE에 반영. 웹 레포에만 있던 2026-09-25 결정(학생 홈 호실은 목록 대신 Figma 배치 그림, SRC-FIGMA-USER)도 REQ-UI-003·ACC-UI-003·SRC-CORRECTIONS에 맞춤. 서버 테스트는 각 PR에서 실행, 실서버 확인은 없음 |
| 2026-10-04 | 학생 봉사 활동 결정 반영 | 사용자 결정(남은 횟수 + 완료 내역)을 REQ-COM-003·ACC-COM-003·DEC-024·SRC-VOLUNTEER-REMAINING에, 완료 내역 API(CheckUp-server#120)를 봉사 계획·계약에 기록. 이전 "남은 횟수만, 활동 목록 없음"을 대체 |
| 2026-10-05 | 관리자 허용 목록 결정 반영 | 개발·테스트용 관리자 허용 목록(`CHECKUP_ADMIN_DATAGSM_IDS`)을 REQ-AUTH-003, ACC-AUTH-003, DEC-025, SRC-ADMIN-ALLOWLIST에 반영. 서버 구현은 CheckUp-server#133 테스트로 확인, 배포 설정은 #134, 실서버 확인은 없음 |

현재 변경은 명세·계획 문서뿐이며 제품 API·웹·AI·배포 구현은 수행하지 않았다.
