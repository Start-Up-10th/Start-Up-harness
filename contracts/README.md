# 서비스 간 계약

제품 정책은 [명세](../docs/spec/index.md), 기술 경계는 [아키텍처](../docs/architecture.md)를 따른다.
서비스 간 내부 REST 계약은 각 기능이 구현될 때 OpenAPI로 기록한다. DataGSM 외부 계약은 사용자가 제공한 경로와 실제 연동 확인 범위로 제한한다.

백엔드 기능별로 실제 OpenAPI 계약을 이 폴더에 추가하고 웹/AI 담당자와 공유한다. 첫 계약은 호실 명단이다.
빈 OpenAPI 파일로 계약이 완료됐다고 처리하지 않는다.

## 인증 (구현됨)

- 로그인 상태는 `SESSION` 쿠키로 전달한다(DEC-016). 웹은 `credentials: 'include'`로 요청한다. `Authorization`·`RefreshToken` 헤더는 쓰지 않는다.
- `GET /api/v1/auth/login`, `GET /api/v1/auth/callback`, `GET /api/v1/auth/me`, `POST /api/v1/auth/logout`. 세부는 [인증 계획](../docs/plans/auth.md).
- 상태 코드: 400 = `oauthState` 없음·만료·재사용, 401 = 비로그인, 403 = 권한 없음·비활성 계정, 로그아웃 성공 = 204.
- 인증 필터의 비로그인 401에는 응답 본문이 없다. 서비스에서 발생한 오류는 기존 `ErrorResponse` 형식을 사용하며 모든 API에 대한 전역 envelope는 아직 정하지 않았다.

## 호실 명단 (구현됨)

- [호실 명단 OpenAPI](room-roster.openapi.yaml): `GET /api/v1/room/student?dormitoryRoom=301`.
- `SESSION` 쿠키 인증. 관리자는 모든 호실, 학생은 DB에 배정된 본인 호실만 조회한다. 비로그인은 401, 잘못된 호실 파라미터는 400, 권한 부족·학생 정보/호실 명단 없음은 403이다.
- 응답은 `student_name`, 반 번호 정수 `student_class`, 표시용 학번 정수 `student_number`를 담은 배열이다.
- 제공 데이터는 현재 서버 PostgreSQL에 저장된 학생 범위다. 전체 명단은 별도 DataGSM 학생 API 동기화가 완료되기 전까지 제공되지 않을 수 있다.

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
