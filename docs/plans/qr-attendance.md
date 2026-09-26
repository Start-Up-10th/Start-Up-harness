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
