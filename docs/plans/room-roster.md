# room-roster — 호실 명단 계획

## 담당자·API

- 담당자: 임서하
- 브랜치: `feat/room-roster`
- `GET /api/v1/room/student?dormitoryRoom={room}`
- OpenAPI: [room-roster.openapi.yaml](../../contracts/room-roster.openapi.yaml)

## 구현

- 인증된 학생의 본인 호실 범위와 관리자 화면의 호실 학생 정보를 조회한다.
- 호실 요청은 필수 양의 정수 `dormitoryRoom` 쿼리 파라미터다. GET body는 사용하지 않는다.
- 학생 홈은 본인 호실만 읽기 전용으로 표시한다.
- 응답은 `student_name`, 반 번호 정수 `student_class`, 표시용 학번 `student_number`의 배열이다.
- 현재 DB 역할로 관리자를 판정하고, 관리자는 임의 호실, 학생은 배정된 본인 호실만 읽을 수 있다.
- Java 학생 필드·파라미터는 `dormitoryRoom`, DB 컬럼은 `dormitory_room`, 파생 Java 층 속성은 `dormitoryFloor`다. 층 필드와 별도 호실 테이블을 만들지 않는다.
- 현재 응답 범위는 PostgreSQL에 저장된 학생이다. 로그인 학생 이외를 채우는 전체 명단 동기화는 별도 DataGSM 작업이다.

## 계약 보완

- [x] 쿼리 방식과 SESSION 인증, 성공·오류 schema를 OpenAPI에 기록한다.
- [ ] `GET https://openapi.datagsm.kr/v1/students`의 `STUDENT_READ` 권한, API 키 주입, 페이지네이션 응답을 확인한다.
- [ ] 학생 OpenAPI의 졸업생·자퇴생 필터와 호실 변경 신호를 확인한다.
- 호실별 수동 출석 저장 API는 별도 범위다. 해당 API 없이 `REQ-ATT-006`을 완료 처리하지 않는다.

## 기준·검증

- 요구사항: `REQ-AUTH-002~003`, `REQ-ATT-006`, `REQ-UI-001~003`, `REQ-UI-006`
- 수용 시나리오: `ACC-AUTH-002~003`, `ACC-ROOM-001`, `ACC-ATT-006`, `ACC-UI-001~003`, `ACC-UI-006`
- 학생이 다른 호실을 조회하지 못하는지 확인한다.
- 관리자와 학생의 응답 범위·읽기 전용/편집 권한을 구분한다.
- 조회 순서가 이름·학번·내부 ID 순이고 이름 관계를 추가 쿼리 없이 가져오는지 확인한다.
- 301→3층, 425→4층, 미배정 호실→층 없음과 DataGSM 로그인 호실 저장·갱신을 확인한다.

## 구현·검증 기록

- 서버 구현: `feat/room-roster`
- 하네스 계약·명세 브랜치: `docs/room-roster`
- `./gradlew.bat build` — Java 25, PostgreSQL 17, Redis 7에서 통과. 컨트롤러·서비스·JPA 저장소·DataGSM 저장/갱신 및 전체 회귀 테스트 통과.
- `npm run harness:check` — 38 REQ, 39 시나리오, 하네스 자체 검사 20/20 통과. 제품 시나리오는 실행 증빙과 분리하여 `specified` 상태를 유지한다.
- 전체 DataGSM 명단 동기화 전에는 조회 범위가 서버 DB에 저장된 학생으로 제한된다.
- 2026-09-28 PR #41 CI 통합 검증: `develop`의 출석 테이블 migration과 호실 컬럼 변경이 모두 `V3`여서 병합 결과에서 `Found more than one migration with version 3`를 재현했다. 최신 `develop`을 반영하고 호실 migration을 `V4`로 옮겼다. SQL 내용과 기존 출석 migration은 유지했다.
- 수정 후 별도 PostgreSQL 17·Redis 7에서 `./gradlew.bat clean build` 통과: 17개 테스트 클래스, 91개 테스트, 실패·오류·건너뜀 0개. 기존 로컬 DB의 migration 이력은 변경하지 않았다.
- 2026-09-28 PR #41 리뷰 반영: 이번 기능에서 추가한 5개 테스트 클래스의 17개 테스트에 한국어 `@DisplayName`을 추가하고 `Student.getDormitoryFloor()`의 JavaDoc을 한국어로 작성했다. PR 제목을 `[feat] 호실 학생 명단 조회 API`로 맞추고 작성자 `Ims2oha`를 담당자로 지정했다.
- 리뷰 반영 후 새 PostgreSQL 17·Redis 7 환경에서 `./gradlew.bat clean build`를 실행해 91개 테스트가 통과했다(실패·오류·건너뜀 0개).
- 2026-09-29 PR #41 머지 충돌 해결: 서버 `develop`의 QR 스캔용 `findByMemberId`와 호실 명단 조회 메서드를 함께 유지했다. 하네스 `main`의 QR·출석 문서도 `docs/room-roster`에 병합해 양쪽 문서를 보존했다.
- 통합 후 새 PostgreSQL 17·Redis 7에서 `./gradlew.bat clean build` 통과: 20개 클래스, 118개 테스트, 실패·오류·건너뜀 0개. 호실 권한·정렬 조회와 실제 DB·Redis를 사용하는 QR 출석 흐름을 포함한다.
