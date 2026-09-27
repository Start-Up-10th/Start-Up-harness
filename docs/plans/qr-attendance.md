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

- heartbeat에서 세션이 없거나 lease가 끝났거나 다른 관리자의 세션이면 404를 반환한다. 웹은 `POST /api/v1/qr`로 새 세션을 만든다.
- close는 성공·이미 종료 모두 204를 반환한다.
- 웹 주소는 서버 설정값으로 둔다. 관리자 웹은 `qrUrl`을 그대로 QR로 그리며 URL을 조립하지 않는다.

## 계약 보완

- [ ] `uuid`, `exp`에 목적·세션 격리·lease를 연결할 방법을 정한다.
- [x] 페이지 이탈 종료, heartbeat, 강제 종료 후 서버 lease 만료 계약을 보완한다. heartbeat·close 경로를 추가했고 lease·heartbeat 간격은 구현에서 설정값으로 정한다.
- [ ] 스캔에서 인증 사용자·세션 상태·purpose·운영일을 재검증한다.
- [ ] 관리자 호실 수동 출석 저장 API 부재를 별도 계약 항목으로 남긴다.
- [ ] 갱신 실패가 기존 QR의 원래 만료를 연장하지 않게 한다.

## 기준·검증

- 요구사항: `REQ-AUTH-001`, `REQ-AUTH-003`, `REQ-ATT-001~007`, `REQ-UI-005~006`
- 수용 시나리오: `ACC-AUTH-001`, `ACC-AUTH-003`, `ACC-ATT-001~007`, `ACC-UI-005~006`
- 서로 다른 관리자·탭·기기의 QR이 서로 종료되지 않는지 확인한다.
- 15분 경계, 갱신 성공/실패, 만료 토큰, 페이지 이탈, 서버 재시작을 fake clock으로 확인한다.
- 위조 student ID/success를 보내도 현재 로그인 사용자 외의 출석이 생성되지 않게 한다.
