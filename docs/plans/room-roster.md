# room-roster — 호실 명단 계획

## 담당자·API

- 담당자: 임서하
- `GET /api/v1/room/student?dormitoryRoom={호실}&purpose={DORMITORY|STUDY_ROOM}` — 계약: [room.openapi.yaml](../../contracts/room.openapi.yaml)

## 구현

- 관리자는 모든 호실, 학생은 본인 호실만 조회한다. 다른 호실·호실 미배정 학생은 403 `FORBIDDEN`, 학생 정보 없음·빈 호실은 403 `MISSING_STUDENT_INFO`.
- 응답은 이름·학번 순 배열이고 항목은 `student_name`, `student_class`, `student_number`(표시용 학번), `attended`다.
- `attended`는 `purpose`(기본 `DORMITORY`)의 오늘 운영일(08:00 KST 경계) 출석 상태다. 행이 없거나 수동 미출석이면 false. 호실 학생 id 목록으로 한 번에 읽는다(`AttendanceRepository.findAttendedStudentIds`, [CheckUp-server#101](https://github.com/Start-Up-10th/CheckUp-server/pull/101)).
- 학생 홈은 본인 호실 번호(`/api/v1/auth/me`의 `student.dormitoryRoom`)로 `purpose=DORMITORY`를 조회해 명단·출석 인원을 보인다(DEC-023, [CheckUp-Client#118](https://github.com/Start-Up-10th/CheckUp-Client/pull/118)).
- 인증된 학생의 본인 호실 범위와 관리자 화면의 호실 학생 정보를 조회한다.
- `student_room_id` 입력을 서버 권한과 함께 검증한다.
- 학생 홈은 본인 호실만 읽기 전용으로 표시한다.
- `DataGsmStudentClient`가 학생 OpenAPI의 페이지를 모두 조회한 뒤 DataGSM 배정 인원 기준으로 방·층을 그룹화하며 정원을 4명으로 고정하지 않는다.

## 계약 보완

- [ ] GET body의 `student_room_id` 호환성을 확인한다.
- [ ] 단일 호실 조회 API가 관리자 층 전개도 전체 조회를 충족하는지 확인한다.
- [ ] `GET https://openapi.datagsm.kr/v1/students`의 `STUDENT_READ` 권한, API 키 주입, 페이지네이션 응답을 확인한다.
- [ ] 학생 OpenAPI의 졸업생·자퇴생 필터와 호실 변경 신호를 확인한다.
- [ ] 호실별 수동 출석 저장 API가 없어 `REQ-ATT-006`을 완료 처리하지 않는다.
- [x] 호실 명단에 출석 여부가 없었다. `attended`와 `purpose`를 추가했다(CheckUp-server#101).
- [ ] 관리자 호실 상세(CheckUp-Client#82)가 용도 탭에 맞춰 `purpose`를 넘기는지 확인한다.

## 기준·검증

- 요구사항: `REQ-AUTH-002~003`, `REQ-ATT-006`, `REQ-UI-001~003`, `REQ-UI-006`
- 수용 시나리오: `ACC-AUTH-002~003`, `ACC-ATT-006`, `ACC-UI-001~003`, `ACC-UI-006`
- 학생이 다른 호실을 조회하지 못하는지 확인한다.
- `attended`가 요청한 용도·오늘 운영일만 반영하고 다른 용도·지난 운영일·수동 미출석을 출석으로 세지 않는지 확인한다(서버 `AttendanceRepositoryTest`, 실제 PostgreSQL).
- 관리자와 학생의 응답 범위·읽기 전용/편집 권한을 구분한다.
- 호실 인원 분모가 DataGSM 배정 수와 일치하는지 확인한다.
