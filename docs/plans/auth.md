# auth — 인증 계획

## 담당자·API

- 담당자: 강민우
- `GET /auth/login` — DataGSM 인가 URL로 302 이동
- `GET /auth/callback?code=&state=` — state 검증, 토큰 교환, 회원 저장, 세션 발급
- `GET /auth/me` — 세션 사용자 조회. 비로그인은 401
- `POST /auth/logout` — 세션 무효화, `SESSION` 쿠키 삭제, 204

인증은 서버 세션 방식이다([DEC-016](../decisions.md)). JWT·refresh token·재발급 API는 두지 않는다.
다른 기능 API의 `/api/v1` prefix 적용 여부는 아직 정하지 않았다. 정해지면 인증 경로도 함께 맞춘다.

## 구현 현황 (`feat/datagsm-oauth`)

- DataGSM 공식 SDK `datagsm-oauth-sdk-java:1.6.0`의 `DataGsmOAuthClient`로 인가 URL 생성, 토큰 교환, `userinfo` 조회
- `oauthState`는 UUID로 만들고 PKCE `codeVerifier`와 함께 Redis `oauth:state:{state}`에 5분 저장. callback에서 `getAndDelete`로 한 번만 사용
- state가 없거나 만료·재사용이면 400
- `status != ACTIVE`, 학생인데 `student` 또는 `student.role`이 없음, 기숙사부가 아닌 교사, 지원하지 않는 `objectType`은 403
- 역할 판정: 학생 `DORMITORY_MANAGER`(기숙사 자치위원)와 교사 `DORMITORY`는 `ADMIN`, 학생회(`STUDENT_COUNCIL`)를 포함한 그 외 활성 학생은 `STUDENT`
- 로그인 시 `member`(`datagsm_id`=최상위 `id`, 이름, 역할)와 `student`(학년, 반, 번호, 학번, 호실)를 저장·갱신
- 로그인 시 기존 세션을 무효화하고 새 세션에 회원 id와 역할을 저장해 세션 고정을 막음
- `/auth/me`는 현재 `name`, `role`만 반환

## 명세와 다른 부분

- [ ] `student.id`(canonical `studentId`)를 저장하지 않는다. REQ-AUTH-002 매핑과 맞춰야 한다.
- [ ] 권한 부족 문구가 `이용 권한이 없는 계정입니다.`다. REQ-AUTH-003 문구는 `관리자 권한이 없는 계정입니다.`다.
- [ ] `/auth/me`에 학생 ID·학번·호실·동의/얼굴 등록 상태가 없다.
- [ ] callback이 JSON을 반환한다. 로그인 후 웹 화면으로 복귀하는 흐름을 정해야 한다.

## 남은 작업

- [ ] 공통 오류 envelope와 `requestId`. 현재는 `ResponseStatusException` 기본 응답
- [ ] QR 로그인 복귀 시 원래 QR의 만료·종료·용도 재검사
- [ ] 로그아웃 시 그 사용자가 운영하던 QR·인식 세션 정리
- [ ] `DataGsmStudentClient`로 `GET https://openapi.datagsm.kr/v1/students` 조회(`X-API-KEY`, `STUDENT_READ`, 페이지네이션, 호실 필터)
- [ ] SDK `UserInfo`를 서비스 계층에 직접 넘기지 않도록 내부 모델로 분리
- [ ] 운영 CORS 설정과 프론트·API same-site 배치 확인
- [ ] OAuth secret을 프론트 번들·HTTP 로그·fixture에서 제외하는지 확인

## 기준·검증

- 요구사항: `REQ-AUTH-001~005`
- 수용 시나리오: `ACC-AUTH-001~005`
- 학생 `student.id`와 표시용 `student.studentNumber`를 분리하고, 최상위 `role`과 내부 관리자 권한을 혼동하지 않는다.
- `STUDENT`/`TEACHER`의 중첩 객체 누락, `status=PENDING`, 지원하지 않는 `objectType`을 거부한다.
- `STUDENT`의 `DORMITORY_MANAGER`, `TEACHER`의 `DORMITORY`만 관리자 권한을 갖는지 확인한다.
- 잘못된·만료·재사용 `oauthState`, 만료 code, DataGSM 장애, 권한 부족을 구분한다.
- 로그아웃 후 같은 세션 쿠키로 `/auth/me`가 401인지 확인한다.
- 학생이 다른 학생의 `/auth/me`·보호 API 범위를 얻지 못하는지 확인한다.
- 전체 학생 조회가 `userinfo`가 아닌 `DataGsmStudentClient`와 학생 OpenAPI, 페이지네이션으로 수행되는지 확인한다.
- 현재 서버에 인증 자동 테스트는 없다. 테스트 추가 전에는 위 항목을 검증 완료로 보고하지 않는다.

선행 조건은 공통 인증·시간·오류 계약이다. 필드 매핑과 권한 판정은 SRC-DATAGSM의 최신 사용자 결정으로 고정하며,
학생 API의 실제 키 권한·페이지네이션 응답·졸업/전학/퇴사 신호는 연동 단계에서 검증한다.
