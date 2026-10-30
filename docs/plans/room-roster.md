# room-roster — 호실 명단 계획

## 담당자·API

- 담당자: 임서하
- `GET /api/v1/room/student?dormitoryRoom={호실}&purpose={DORMITORY|STUDY_ROOM}` — 계약: [room.openapi.yaml](../../contracts/room.openapi.yaml)
- `GET /api/v1/room/floor/students?floor={층}&purpose={DORMITORY|STUDY_ROOM}` — 관리자 층별 학생 일괄 조회, 같은 계약에 정의(CheckUp-server#268, Client#394).
- `PUT /api/v1/room/{dormitoryRoom}/attendance?purpose={DORMITORY|STUDY_ROOM}` — 관리자가 호실 학생 출석을 일괄 저장한다. 본문 `students`에는 DataGSM 학생 id와 `attended`를 보낸다. `date`는 생략하거나 오늘 운영일만 쓸 수 있다.
- `PUT /api/v1/room/student/{studentId}/attendance` — 관리자가 한 학생의 기숙사 출석 상태를 수정한다. 상세 계약은 `room.openapi.yaml`에 둔다.

## 구현

- 관리자는 모든 호실, 학생은 본인 호실만 조회한다. 다른 호실·호실 미배정 학생은 403 `FORBIDDEN`, 학생 정보 없음·빈 호실은 403 `MISSING_STUDENT_INFO`.
- 명단에는 학생 동기화로 미리 저장한, 아직 로그인하지 않은 학생도 포함한다(DEC-026, [CheckUp-server#119](https://github.com/Start-Up-10th/CheckUp-server/pull/119)).
- 응답은 학생 이름·학번 순 배열이고 항목은 `student_name`, `student_class`, `student_number`(표시용 학번), `attended`다.
- `attended`는 `purpose`(기본 `DORMITORY`)의 오늘 운영일(08:00 KST 경계) 출석 상태다. 행이 없거나 수동 미출석이면 false. 호실 학생 id 목록으로 한 번에 읽는다(`AttendanceRepository.findAttendedStudentIds`, [CheckUp-server#101](https://github.com/Start-Up-10th/CheckUp-server/pull/101)).
- 학생 홈은 본인 호실 번호(`/api/v1/auth/me`의 `student.dormitoryRoom`)로 `purpose=DORMITORY`를 조회해 명단·출석 인원을 보인다(DEC-023, [CheckUp-Client#118](https://github.com/Start-Up-10th/CheckUp-Client/pull/118)).
- 인증된 학생의 본인 호실 범위와 관리자 화면의 호실 학생 정보를 조회한다.
- `student_room_id` 입력을 서버 권한과 함께 검증한다.
- 학생 홈은 본인 호실만 읽기 전용으로 표시한다.
- `DataGsmStudentClient`가 학생 OpenAPI의 페이지를 모두 조회한 뒤 DataGSM 배정 인원 기준으로 방·층을 그룹화하며 정원을 4명으로 고정하지 않는다.
- 관리자 수동 출석 저장은 오늘 운영일만 허용한다. 지난 날짜는 요약 조회만 가능하다(DEC-035).

## 계약 보완

- [ ] GET body의 `student_room_id` 호환성을 확인한다.
- [ ] 단일 호실 조회 API가 관리자 층 전개도 전체 조회를 충족하는지 확인한다.
- [x] 호실별 반복 요청을 피하는 별도 `/room/floor/students` 계약을 정의한다. 기존 `/room/floor` 집계는 유지한다. 층 명단은 단일 scalar projection 쿼리로 읽고 호실 번호·학생 이름·학번·내부 id 순으로 정렬한다. 호실 상세와 같은 DataGSM 학생 id 기준과 학생 필드를 쓴다.
- [ ] `GET https://openapi.datagsm.kr/v1/students`의 `STUDENT_READ` 권한, API 키 주입, 페이지네이션 응답을 확인한다.
- [ ] 학생 OpenAPI의 졸업생·자퇴생 필터와 호실 변경 신호를 확인한다.
- [x] 호실별 수동 출석 저장은 `PUT /api/v1/room/{dormitoryRoom}/attendance`로 제공한다. 호실 범위·중복 학생을 검증하고 요청 전체를 원자적으로 저장한다. 개인 수정 경로도 `PUT /api/v1/room/student/{studentId}/attendance`로 제공한다.
- [x] 호실 명단에 출석 여부가 없었다. `attended`와 `purpose`를 추가했다(CheckUp-server#101).
- [ ] 관리자 호실 상세(CheckUp-Client#82)가 용도 탭에 맞춰 `purpose`를 넘기는지 확인한다.

## 기준·검증

- 요구사항: `REQ-AUTH-002~003`, `REQ-ATT-006`, `REQ-UI-001~003`, `REQ-UI-006`
- 수용 시나리오: `ACC-AUTH-002~003`, `ACC-ATT-006`, `ACC-UI-001~003`, `ACC-UI-006`
- 학생이 다른 호실을 조회하지 못하는지 확인한다.
- `attended`가 요청한 용도·오늘 운영일만 반영하고 다른 용도·지난 운영일·수동 미출석을 출석으로 세지 않는지 확인한다(서버 `AttendanceRepositoryTest`, 실제 PostgreSQL).
- 관리자와 학생의 응답 범위·읽기 전용/편집 권한을 구분한다.
- 호실 인원 분모가 DataGSM 배정 수와 일치하는지 확인한다.

## 층별 학생 조회 구현·검증 (2026-10-30)

- 관련 REQ: `REQ-UI-001~002`, `REQ-AUTH-003`, `REQ-ATT-001·007`. 관련 수용 시나리오: `ACC-UI-001~002`, `ACC-AUTH-003`, `ACC-ATT-007`.
- 서버 구현: `RoomController.getFloorStudents`, `RoomService.getFloorStudents`, `AttendanceRepository.findFloorStudents`, `FloorStudentAttendance`, `RoomFloorStudentsResponse`.
- 서버 테스트: `RoomControllerTest`(응답·기본/명시 용도·401/403·잘못된 입력), `RoomFloorStudentsServiceTest`(정렬·빈 층·권한 선검증·08:00 경계), `FloorStudentAttendanceRepositoryTest`(PostgreSQL 범위·용도·운영일·수동 미출석·미로그인 학생·SQL 한 문장과 엔티티 추가 로딩 없음).
- 검증: Windows, Microsoft JDK 25.0.4.1, 격리된 로컬 PostgreSQL 17·Redis 7에서 `./gradlew.bat build --console=plain` 통과(682개 중 680 통과, 실패 0, 기존 실서버 AI·부하 테스트 2개 비활성). 새 테스트 12개는 모두 실행·통과했다. CI와 같은 개발용 DataGSM/AI 환경 값과 `PUBLIC_ORIGIN=http://localhost:3000`을 사용했으며 실제 외부 API는 호출하지 않았다.
- 첫 실행에서 새 테스트 전용 Spring 컨텍스트가 DB 연결 한도를 넘겨 통계 활성화를 메서드 안에서만 켜고 복구하도록 수정했다. 재실행 중 Windows JDK의 C2 CompilerThread가 access violation으로 종료되어 로컬 실행에만 `JAVA_TOOL_OPTIONS=-XX:TieredStopAtLevel=1`을 사용했다. 서버/CI JVM 설정은 변경하지 않았다.
- `npm run harness:check` 통과(하네스 테스트 20개). 프런트 연동·브라우저 전개도는 이번 서버 변경 범위가 아니므로 수용 시나리오 전체를 verified로 올리지 않는다.
- 다음 작업: Client#394에서 이 API로 호실별 반복 요청을 대체하고 실제 전개도 표시·새로고침 동선을 확인한다. 지난 날짜 보관(#269)과 연속 봉사(#270)는 정책 결정이 필요해 이 변경에 포함하지 않는다.


## 2026-10-30 — #269·#270 구현

- 사용자 승인: DEC-035(출석 요약 30일), DEC-036(출석일 연속 봉사).
- [x] 명세·출처·수용 시나리오 갱신.
- [x] #269: 원본 08:00 폐기 유지, 요약 저장·만료 삭제·관리자 날짜 조회.
- [x] #270: 별도 잔여 조정·기숙사 얼굴/QR 자동 지정·완료 차감·응답.
- [x] PostgreSQL 통합·권한·경계·동시성 테스트, 전체 서버 빌드(692 통과·2 비활성), 하네스 검사(20 통과).
- 실제 웹 연동·최종 학교 동의 전문 적용·운영 배포는 이 서버 작업에서 검증하지 않는다.

검증 환경·명령·구현 근거·미검증 범위: [서버 #269·#270 검증](../reviews/server-remaining-issues-2026-10-30.md).
