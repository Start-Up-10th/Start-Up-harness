# auth — 인증 계획

## 담당자·API

- 담당자: 강민우
- `POST /api/v1/auth/oauth/callback`
- `GET /api/v1/auth/me`
- `POST /api/v1/auth/logout`
- `PUT /api/v1/auth/reissue`

## 구현

- 백엔드가 random `oauthState`와 PKCE `codeVerifier`를 생성·저장하고 callback에서 검증·일회 삭제
- OAuth state와 DataGSM 계정 `status`를 혼동하지 않고, `status`는 `accountStatus`로 매핑
- DataGSM 토큰 교환, `userinfo` 조회, 내부 Principal 매핑, 서버 세션/JWT 발급
- DataGSM 원본 DTO와 내부 모델을 분리한다. 원본 숫자형 ID는 `Long`으로 받고 기존 문자열 계약은 어댑터에서만 변환
- `/auth/me`의 canonical 학생 ID·이름·학번·호실·관리자 여부·얼굴 등록 상태 반환
- 학생 `student.role`(`DORMITORY_MANAGER` 또는 `STUDENT_COUNCIL`)와 교사 `teacher.department == DORMITORY`를 기준으로 서버 권한 판정
- 최상위 DataGSM `role`(`USER`/`ADMIN`)은 서비스 관리자 권한으로 자동 사용하지 않음
- `objectType`과 중첩 `student`/`teacher`의 일관성을 검증하고, 비활성·불완전·지원하지 않는 응답은 거부
- 전체 학생·호실 목록은 `userinfo`가 아니라 `DataGsmStudentClient`로 `GET https://openapi.datagsm.kr/v1/students`를 조회
- `DataGsmStudentClient`에서 `X-API-KEY`, `STUDENT_READ` 권한, 페이지네이션과 호실별 필터를 처리
- 로그아웃 시 로그인 세션과 사용자 인증 세션 정리
- Access Token 재발급 및 refresh token 만료·위조·재사용 거부
- QR 로그인 복귀 시 원래 QR의 만료·종료·용도 재검사

클라이언트 책임은 다음처럼 분리한다.

- `DataGsmOAuthClient`: authorization code 교환, `userinfo` 조회, 로그인 사용자 매핑
- `DataGsmStudentClient`: 전체 학생·호실 명단 조회, 페이지네이션, 졸업생·자퇴생 필터

## 계약 보완

- [ ] API 예시의 JSON 쉼표와 토큰 반환 타입을 정리한다.
- [ ] `RefreshToken` 헤더와 일반 `Authorization` 헤더 규칙을 공통 계약에 기록한다.
- [ ] 공통 오류 envelope와 `requestId`를 정한다.
- [ ] OAuth secret·refresh token을 프론트 번들·HTTP 로그·fixture에서 제외한다.

## 기준·검증

- 요구사항: `REQ-AUTH-001~005`
- 수용 시나리오: `ACC-AUTH-001~005`
- 학생 `student.id`와 표시용 `student.studentNumber`를 분리하고, 최상위 `role`과 내부 관리자 권한을 혼동하지 않는다.
- `STUDENT`/`TEACHER`의 중첩 객체 누락, `status=PENDING`, 지원하지 않는 `objectType`을 거부한다.
- `STUDENT`의 `DORMITORY_MANAGER`·`STUDENT_COUNCIL`, `TEACHER`의 `DORMITORY`만 관리자 권한을 갖는지 확인한다.
- 잘못된 `oauthState`/codeVerifier, 만료 code, DataGSM 장애, 권한 부족을 구분한다.
- 만료·위조·재사용 refresh token을 거부한다.
- 학생이 다른 학생의 `/auth/me`·보호 API 범위를 얻지 못하는지 확인한다.
- 전체 학생 조회가 `userinfo`가 아닌 `DataGsmStudentClient`와 학생 OpenAPI, 페이지네이션으로 수행되는지 확인한다.

선행 조건은 공통 인증·시간·오류 계약이다. 필드 매핑과 권한 판정은 SRC-DATAGSM의 최신 사용자 결정으로 고정하며,
학생 API의 실제 키 권한·페이지네이션 응답·졸업/전학/퇴사 신호는 연동 단계에서 검증한다.
