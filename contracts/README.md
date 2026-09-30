# 서비스 간 계약

제품 정책은 [명세](../docs/spec/index.md), 기술 경계는 [아키텍처](../docs/architecture.md)를 따른다.
구현된 내부 REST API의 계약은 이 폴더의 OpenAPI 파일로 둔다. 아직 OpenAPI가 없는 API는 각 계획 문서를 따른다.

백엔드 첫 기능 구현 시 실제 OpenAPI 계약을 이 폴더에 생성하고 웹/AI 담당자와 공유한다.
빈 OpenAPI 파일로 계약이 완료됐다고 처리하지 않는다.

## 인증 (구현됨)

- 로그인 상태는 `SESSION` 쿠키로 전달한다(DEC-016). 웹은 `credentials: 'include'`로 요청한다. `Authorization`·`RefreshToken` 헤더는 쓰지 않는다.
- `GET /api/v1/auth/login`, `GET /api/v1/auth/callback`, `GET /api/v1/auth/me`, `POST /api/v1/auth/logout`. 세부는 [인증 계획](../docs/plans/auth.md).
- 상태 코드: 400 = `oauthState` 없음·만료·재사용, 401 = 비로그인, 403 = 권한 없음·비활성 계정, 로그아웃 성공 = 204.
- 공통 오류 형식: `{ "code": "<ErrorCode>", "message": "<기본 메시지>", "errors": [{ "field", "reason" }] }`. `errors`는 요청 값 검증 실패 때만 있다. 웹은 `code`로 분기한다. Spring Security 필터가 막는 401·403도 같은 형식이다(`UNAUTHORIZED`·`FORBIDDEN`, [CheckUp-server#55](https://github.com/Start-Up-10th/CheckUp-server/pull/55)).

## 동의 (구현됨)

- `POST /api/v1/consent`: 본문 `{ "privacy": boolean, "face": boolean, "noticeAlarm": boolean }`, 성공 204. 필수 두 항목이 true가 아니거나 빠지면 400 `INVALID_REQUEST`, 미로그인 401, 학생이 아닌 계정 403 `MISSING_STUDENT_INFO`.
- `GET /api/v1/auth/me`의 `consented`: 필수 동의 두 항목 완료 여부. 웹은 이 값으로 로그인 후 동의 화면을 거칠지 정한다.
- 제공자: CheckUp-server. 소비자: 학생 웹 `/consent`, `/login/complete`. 정책은 REQ-AUTH-004, 세부는 [인증 계획](../docs/plans/auth.md).

## QR 출석 (구현됨)

- [qr.openapi.yaml](qr.openapi.yaml): `POST /api/v1/qr`, `POST /api/v1/qr/{sessionId}/heartbeat`, `POST /api/v1/qr/{sessionId}/close`, `POST /api/v1/qr/attendance`
- 제공자: CheckUp-server. 소비자: 관리자 웹 QR 화면, 학생 웹 `/qr`. 정책은 [QR 계획](../docs/plans/qr-attendance.md), DEC-018.
- 시각 필드는 ISO-8601 UTC 문자열이다.

계약에 반드시 표현할 내용:

- OAuth 세션의 사용자 역할, 동의/등록 완료 상태, 본인 데이터 범위.
- 자습실/기숙사 purpose와 운영일, 서버 UTC 시각·08:00 KST 경계.
- 페이지별 독립 QR/카메라 session ID, 토큰 만료 시각, 종료/갱신 오류.
- 출석 단일 처리의 event ID/중복 판정, 원래 발생 시각과 서버 수신 시각.
- 수동 수정과 늦은 동기화의 순서 규칙.
- AI 결과의 얼굴별 track ID, known/unknown, 모델 버전/점수. unknown 학생 ID는 null.
- 대표 벡터의 모델/차원/정규화 호환성. 벡터 API는 일반 학생에게 공개하지 않음.
- 봉사 명단 소속과 +1 적립, 재시도 idempotency, 공지 알림 동의.
- 오류 코드와 UI 문구를 분리하여 권한·만료·중복·일시 장애를 구별.

계약 변경 시 제공자/소비자와 시나리오를 같은 변경에서 갱신한다. 예제에는 실제 이름·학번·벡터·secret을 넣지 않는다.
