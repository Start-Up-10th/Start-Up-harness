# qr-attendance — QR·출석 계획

## 담당자·API

- 담당자: 김성찬
- `POST /api/v1/qr` — QR 세션 생성(관리자)
- `POST /api/v1/qr/{sessionId}/heartbeat` — lease 연장과 현재 토큰 조회(세션 생성 관리자)
- `POST /api/v1/qr/{sessionId}/close` — 해당 세션만 종료(세션 생성 관리자)
- `POST /api/v1/qr/attendance` — QR 스캔 출석(로그인 학생)

## 구현

- 관리자 인증 후 자습실·기숙사 목적별 독립 QR 세션 생성
- 서버 만료 시각·15분 토큰 교체·페이지/기기별 세션 격리
- 관리자 화면은 heartbeat 하나로 lease 연장과 토큰 교체를 받는다. 토큰이 만료되면 다음 heartbeat 응답에 새 토큰을 담고, 이전 토큰의 원래 만료 시각은 바꾸지 않는다.
- 페이지 이탈(`sendBeacon`)·용도 탭 변경 시 close로 해당 세션만 종료한다. 이미 종료된 세션의 close도 성공으로 처리한다.
- heartbeat가 끊기면 lease 만료로 세션이 사라진다. 없는 세션의 heartbeat는 오류를 반환하고 웹이 새 세션을 만든다.
- 현재 로그인 학생 기준 QR 출석 처리, 가장 이른 유효 성공, 중복 안내
- 08:00 운영일 경계와 전날 QR/오프라인 이벤트 재사용 방지
- DB unique와 원자적 처리로 QR·얼굴·여러 기기의 동시 중복 방지

## QR 값 형식

- QR에는 웹 주소와 토큰을 담은 URL을 넣는다: `https://<웹 주소>/qr#t=<토큰>`
- 토큰은 32바이트 난수 base64url(43자)이며 학생 정보를 담지 않는다.
- 토큰은 URL fragment(`#t=`)에 둔다. fragment는 서버로 전송되지 않아 웹 서버·프록시 접근 로그에 남지 않는다.
- 휴대폰 일반 카메라로 찍으면 학생 웹 `/qr`이 열리고, 페이지가 `#t=`를 읽어 바로 스캔 API를 호출한다.
- 웹 내부 QR 카메라는 읽은 URL에서 `#t=` 뒤의 토큰만 꺼내 같은 스캔 API를 호출한다. 형식이 다르면 호출하지 않고 `INVALID`로 표시한다.

## 관리자 API 응답

- `POST /api/v1/qr` 요청: `{ "purpose": "DORMITORY" | "STUDY_ROOM" }`
- `POST /api/v1/qr`, `POST /api/v1/qr/{sessionId}/heartbeat` 응답:

| 필드 | 의미 |
| --- | --- |
| `sessionId` | 이 페이지의 QR 세션 ID. heartbeat·close에 쓴다. |
| `purpose` | 출석 용도 |
| `qrUrl` | QR로 그릴 값. 위 QR 값 형식의 URL이다. |
| `tokenExpiresAt` | 현재 토큰 만료 시각. `남은 유효 시간` 카운트다운 기준이다. |
| `leaseExpiresAt` | 다음 heartbeat가 없으면 세션이 끝나는 시각 |
| `serverTime` | 응답 시점 서버 시각. 브라우저 시계 오차 보정에 쓴다. |

- heartbeat에서 세션이 없거나 lease가 끝났거나 다른 관리자의 세션이면 404 `QR_SESSION_NOT_FOUND`를 반환한다. 웹은 `POST /api/v1/qr`로 새 세션을 만든다. 다른 관리자의 세션 존재 여부를 알리지 않도록 세 경우를 구분하지 않는다.
- 관리자가 아닌 계정이 관리자 API를 호출하면 403 `ADMIN_ONLY`다. 관리자 여부는 세션 권한이 아니라 요청마다 DB의 회원 역할로 판정한다.
- 오류 응답 본문은 공통 형식 `{ "code": "<ErrorCode>", "message": "<기본 메시지>" }`이다. 웹은 `code`로 분기한다.
- 시각 필드(`tokenExpiresAt`, `leaseExpiresAt`, `serverTime`)는 ISO-8601 UTC 문자열이다(예: `"2026-09-27T03:15:00Z"`). 숫자(unix ms)가 아니므로 웹은 `Date.parse` 등으로 변환해 계산한다.
- close는 성공·이미 종료 모두 204를 반환한다.
- 웹 주소는 서버 설정값으로 둔다. 관리자 웹은 `qrUrl`을 그대로 QR로 그리며 URL을 조립하지 않는다.

## 스캔 API

- `POST /api/v1/qr/attendance` 요청: `{ "token": "<토큰>" }`. 학생 ID·성공 여부는 받지 않고 로그인한 현재 학생으로 처리한다.
- 판정 결과는 모두 200과 `{ "result": "<결과>" }`로 반환한다. 웹은 결과별 문구만 표시한다.

| result | 조건 | 웹 문구 |
| --- | --- | --- |
| `APPROVED` | 새로 출석 처리됨 | `승인되었습니다.` 후 메인 이동 |
| `DUPLICATE` | 같은 학생·용도·운영일에 이미 출석 | `이미 출석 처리된 QR입니다.` |
| `EXPIRED` | 토큰의 발급 때 정한 만료 시각이 지남 | `만료된 QR입니다. 다시 스캔해 주세요.` |
| `CLOSED` | 토큰은 유효하지만 세션이 종료됐거나 lease가 끝남 | `지금은 출석 인증을 받고 있지 않습니다.` |
| `INVALID` | 발급하지 않았거나 기록이 남아 있지 않은 토큰 | `유효하지 않은 QR입니다.` |

- 판정 순서: 토큰 존재(`INVALID`) → 토큰 만료(`EXPIRED`) → 세션 활성(`CLOSED`) → 출석 저장(`APPROVED`/`DUPLICATE`)
- 만료된 토큰을 `INVALID`가 아닌 `EXPIRED`로 구분하도록, 토큰 기록은 만료 뒤에도 구현 설정 시간만큼 남긴다.
- 로그인하지 않으면 401, 학생 정보가 없는 계정(교사)은 403 `MISSING_STUDENT_INFO`다. 기숙사 자치위원은 학생이므로 스캔할 수 있다.
- `token`이 비어 있으면 400 `INVALID_REQUEST`다. 형식(43자 base64url)이 다른 토큰은 저장소를 조회하지 않고 `INVALID`로 판정한다.
- 출석 용도는 요청이 아니라 토큰을 발급한 QR 세션의 용도를 쓴다.

## 작업 분담

| 영역 | 담당 | 할 일 |
| --- | --- | --- |
| 서버 | 김성찬 | 세션·토큰, `qrUrl` 생성, 스캔 판정·출석 저장 |
| 학생 웹 `/qr` | 김성찬 | `#t=`가 있으면 카메라 없이 바로 스캔 API 호출, 내부 카메라는 URL에서 토큰 추출 |
| 관리자 웹 QR 화면 | 관리자 웹 담당 | `qrUrl`을 QR로 표시, 약 20초마다 heartbeat, 이탈·탭 변경 시 close |
| 로그인 복귀 | 인증 담당 | 미로그인 학생이 로그인 후 원래 `/qr#t=…`로 돌아오게 콜백 리다이렉트 처리 |

## 출석 테이블 (QR·얼굴·조회 공유)

QR이 출석을 처음 기록하는 기능이라 QR 작업에서 만들었다(서버 `V3__create_attendance.sql`, CheckUp-server#37). 학생·출석 조회와 얼굴 인식도 같은 테이블을 쓴다. 전체 ERD는 프로젝트 마무리 때 팀에서 맞춘다.

| 컬럼 | 타입 | 의미 |
| --- | --- | --- |
| `id` | `BIGSERIAL` | PK |
| `student_id` | `BIGINT NOT NULL` → `student(id)` `ON DELETE CASCADE` | 출석 대상 학생. 학생이 삭제되면 당일 출석도 함께 삭제된다. |
| `purpose` | `VARCHAR(20) NOT NULL` | `DORMITORY` / `STUDY_ROOM` |
| `operating_day` | `DATE NOT NULL` | 08:00 KST 기준 운영일(DEC-006) |
| `attended` | `BOOLEAN NOT NULL` | 현재 상태. 수동 미출석이면 false |
| `first_verified_at` | `TIMESTAMPTZ` | 최초 유효 인증 시각 |
| `method` | `VARCHAR(20)` | 최초 유효 인증 방식 `QR` / `FACE` / `MANUAL` |
| `manual_updated_at` | `TIMESTAMPTZ` | 마지막 수동 수정 시각(DEC-008) |

- `UNIQUE (student_id, purpose, operating_day)`: 학생·용도·운영일당 한 행이다(REQ-ATT-002).
- `member`가 아닌 `student`를 참조한다. 출석 대상은 학생뿐이다.
- `idx_attendance_operating_day`: 08:00 정리와 운영일별 조회용 인덱스다.
- 현재 상태(`attended`)와 최초 인증 시각(`first_verified_at`)을 분리한다(REQ-ATT-006).

저장 규칙:

- 출석은 서버 내부 `AttendanceService.markAttended(studentId, purpose, verifiedAt, method)`로만 기록한다. 이 기능 자체에는 HTTP API가 없고, QR 스캔·얼굴 인식이 인증 성공 뒤 호출한다.
- 운영일은 호출하는 쪽이 넘기지 않고 `verifiedAt`으로 계산한다.
- 자동 인증은 `INSERT … ON CONFLICT (student_id, purpose, operating_day)`로 원자적으로 처리한다. Redis만으로 중복을 막지 않는다.
- `first_verified_at`은 가장 이른 유효 인증 시각만 남긴다.

| 결과 | 조건 | QR 스캔 결과 |
| --- | --- | --- |
| `RECORDED` | 새로 출석 처리 | `APPROVED` |
| `ALREADY_ATTENDED` | 이미 출석 상태 | `DUPLICATE` |
| `SUPERSEDED_BY_MANUAL` | 수동 미출석(`manual_updated_at`) 이전에 발생해 늦게 도착한 인증 (DEC-008) | `DUPLICATE` |
| `STALE` | `verifiedAt`의 운영일이 오늘이 아님. 08:00 이전 이벤트 재전송으로 전날 기록을 되살리지 않는다 (REQ-ATT-007) | `EXPIRED` |
| `FUTURE` | `verifiedAt`이 서버 시각보다 5초 넘게 늦음. 5초 이내면 서버 현재 시각으로 낮춰 기록한다 (REQ-ATT-002 시계 보정) | `INVALID` |

- QR은 서버 현재 시각으로 기록하므로 `STALE`·`FUTURE`·`SUPERSEDED_BY_MANUAL`은 실제로 나오지 않는다. 얼굴 오프라인 동기화 대비다.
- 08:00 이후 전날 행 정리는 학생·출석 조회 계획(REQ-ATT-007)에서 담당한다.
- 관리자 수동 출석 저장 API는 아직 없다. 생기면 `manual_updated_at`을 채우고 같은 테이블을 쓴다.

## 계약 보완

- [x] `uuid`, `exp`에 목적·세션 격리·lease를 연결할 방법을 정한다. `sessionId`·`qrUrl`·`tokenExpiresAt`·`leaseExpiresAt`으로 대체했다.
- [x] 페이지 이탈 종료, heartbeat, 강제 종료 후 서버 lease 만료 계약을 보완한다. heartbeat·close 경로를 추가했고 lease·heartbeat 간격은 구현에서 설정값으로 정한다.
- [x] 스캔에서 인증 사용자·세션 상태·purpose·운영일을 재검증한다. 스캔 API 판정 순서로 정했다. 운영일은 토큰 만료가 다음 08:00을 넘지 않는 것으로 보장한다.
- [ ] 관리자 호실 수동 출석 저장 API 부재를 별도 계약 항목으로 남긴다.
- [x] 갱신 실패가 기존 QR의 원래 만료를 연장하지 않게 한다. 토큰 만료 시각은 발급 때 정하고 바꾸지 않으며, 교체는 새 토큰 발급으로만 한다.

## 기준·검증

- 요구사항: `REQ-AUTH-001`, `REQ-AUTH-003`, `REQ-ATT-001~007`, `REQ-UI-005~006`
- 수용 시나리오: `ACC-AUTH-001`, `ACC-AUTH-003`, `ACC-ATT-001~007`, `ACC-UI-005~006`
- 서로 다른 관리자·탭·기기의 QR이 서로 종료되지 않는지 확인한다.
- 15분 경계, 갱신 성공/실패, 만료 토큰, 페이지 이탈, 서버 재시작을 fake clock으로 확인한다.
- 위조 student ID/success를 보내도 현재 로그인 사용자 외의 출석이 생성되지 않게 한다.
