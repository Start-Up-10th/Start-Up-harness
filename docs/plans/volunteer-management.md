# volunteer-management — 봉사 관리 계획

## 담당자·API

- 담당자: 강민우
- 서버: [CheckUp-server#76](https://github.com/Start-Up-10th/CheckUp-server/pull/76) (리뷰 중)
- 경로의 `{studentId}`는 DataGSM 학생 id다.

| API | 설명 |
| --- | --- |
| `GET /api/v1/volunteer?q=&floor=&minCount=&onDuty=` | 봉사 관리 명단. 전체 학생, 호실순. 항목에 오늘 지정 상태(`todayDuty`) 포함 |
| `PATCH /api/v1/volunteer/{studentId}/count/increase` | 봉사 `+1`. `Idempotency-Key` 헤더(선택) |
| `PATCH /api/v1/volunteer/{studentId}/count/decrease` | 봉사 `−1`(사감 감면). 0이면 409 |
| `POST /api/v1/volunteer/{studentId}/duty` | 오늘 당일 봉사자 지정 + 봉사 알림. 봉사 0회면 409 `봉사가 없습니다.` |
| `DELETE /api/v1/volunteer/{studentId}/duty` | 지정 취소 + 알림 삭제. 완료 후에는 409 |
| `POST /api/v1/volunteer/{studentId}/duty/complete` | 봉사 완료 + `−1`. 한 번만 반영 |

## 구현

- 모든 API는 관리자(사감·자치위원)만 사용한다. 명단은 DataGSM 학생 id가 있는 저장 학생 전체이고 추가·제외는 없다(DEC-021).
- 봉사 횟수는 앞으로 해야 할 횟수다(DEC-020). `+`/`−`는 확인 없이 즉시 반영하고 조정 기록(`volunteer_adjustment`)에 날짜를 남긴다. 최근 활동은 마지막 조정 시각이다.
- 차감은 `volunteer_count > 0` 조건 한 쿼리로 0 미만을 막는다. 같은 `Idempotency-Key`의 재시도는 한 번만 반영하고, 그 키가 다른 학생·방향에 쓰였으면 409로 거부한다.
- 검색은 숫자면 호실·학번 정확 일치, 그 밖에는 이름 포함 → 초성 → 자음·모음 오타 1개 순이다. 외부 검색 엔진 없이 서버에서 거르고 정렬한다.
- 당일 봉사자(`volunteer_duty`, 학생·운영일 unique)는 봉사 1회 이상인 학생만 지정한다. 지정할 때 같은 트랜잭션에서 `VOLUNTEER` 알림(원본 `duty:<운영일>`)을 만들고 취소하면 지운다.
- 완료는 지정을 완료로 바꾸고 `−1` 기록·차감을 한 트랜잭션에서 한다. 완료한 지정은 취소할 수 없다.
- 학생 본인 남은 횟수 조회는 `GET /api/v1/users/{studentId}/volunteer`(CheckUp-server#69)이다.

## 계약 보완

- [x] `student_id` path는 서버에서 존재 학생·관리자 권한을 검증한다.
- [x] 같은 클릭의 재시도는 idempotency로 막고 새 클릭은 별도 1회로 처리한다.
- [x] 차감은 0회 미만이 되지 않도록 원자적으로 검증한다.
- [x] 당일 봉사자 지정 테이블(학생·운영일 unique)과 지정·취소·완료 API 경로를 정한다.
- [ ] 서버 PR 머지 후 `contracts/`에 OpenAPI를 추가한다.
- [ ] 명단에는 로그인한 적 있는 학생만 저장돼 있다. 전교생을 보여주려면 DataGSM 목록으로 학생을 미리 저장하는 방법을 정한다.
- [ ] 공지 CRUD API는 현재 명세에 없으므로 임의 구현하지 않는다.

## 기준·검증

- 요구사항: `REQ-AUTH-003`, `REQ-COM-001~006`, `REQ-UI-004~006`
- 수용 시나리오: `ACC-AUTH-003`, `ACC-COM-001~006`, `ACC-UI-004~006`
- 일반 학생이 봉사 명단·타인 횟수·관리 mutation을 호출하지 못하게 한다.
- 두 번의 새 클릭, 동일 요청 재전송, 다른 요청에 재사용한 키를 구분한다.
- 봉사 횟수와 당일 봉사자 지정은 08:00 출석 정리 때 삭제하지 않는다.
- 지정·중복 지정·취소 때 알림이 한 번 생기고 취소 후 사라지는지, 완료가 한 번만 차감되는지 확인한다.
