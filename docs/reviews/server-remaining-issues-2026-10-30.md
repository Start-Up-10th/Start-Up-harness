# 서버 #269·#270 검증 — 2026-10-30

## 변경과 정책

사용자 승인(DEC-035·036)에 따라 출입 시각 없는 출석 요약을 지난 30개 운영일까지 보관하고, 기숙사 얼굴·QR 인증을 연속 봉사 자동 지정과 연결했다. 원본 인증 시각·방식·수동 수정 시각은 기존 08:00 정리 대상으로 유지한다. 요약은 학생·운영일·용도·호실·출석 여부 5개 열뿐이다. 이름·학번은 현재 학생에서 읽고 학생 삭제 시 요약도 FK cascade로 삭제한다. 당시 호실은 해당 운영일에 처음 요약을 만들 때의 배정값이다.

## 실행 환경과 명령

- Windows, Microsoft JDK 25.0.4.1, PostgreSQL 17, Redis 7, Node.js 24.19.0.
- 로컬 Docker context desktop-linux, Compose 프로젝트 checkup-remaining-issues. 운영 환경·개인 데이터는 사용하지 않았다.
- CI와 같은 개발용 DATAGSM_CLIENT_ID, DATAGSM_CLIENT_SECRET, DATAGSM_API_KEY, DATAGSM_WEBHOOK_SECRET, FACE_SERVICE_TOKEN 및 PUBLIC_ORIGIN=http://localhost:3000 설정을 사용했다. 실제 외부 API는 호출하지 않았다.
- 서버 루트에서: ./gradlew.bat build --console=plain
- 하네스 루트에서: npm run harness:check
- 이전 검증에서 확인된 Windows JDK C2 오류를 피하기 위해 로컬 JAVA_TOOL_OPTIONS=-XX:TieredStopAtLevel=1만 사용했다. CI·운영 JVM 옵션은 변경하지 않았다.

## 결과

- 기존 V22 DB에서 새 V23·V24 마이그레이션 적용 후 관련 출석·호실·봉사 테스트 통과.
- 별도 빈 PostgreSQL DB(checkup_remaining_clean)에서 모든 마이그레이션과 전체 빌드 통과. 총 694개 중 692 통과, 실패·오류 0, 기존 실서버 AI·부하 테스트 2개 비활성.
- 최종 코드에서 전체 빌드를 다시 실행해 같은 결과를 확인했다.
- 하네스 검사와 검사기 테스트 20개 통과. 제품 수용 시나리오 전체의 verified 승격은 하지 않았다.

## 근거 테스트

- AttendanceServiceTest: 가짜 clock으로 08:00 원본 폐기·최종 수동 상태 요약·미출석 명단·30일 경계·반복 정리·학생 삭제·과거 이벤트 거부 확인. summary 열에 인증 시각이나 방식이 없음을 실제 DB metadata로 확인.
- RoomMapApiIntegrationTest: date로 층 집계·층 학생·호실 조회, 용도 분리, 당시 호실 유지, 401/403, 잘못된/미래/31일 전 날짜 거부, 두 수동 저장 경로의 과거 날짜 거부.
- ConsecutiveVolunteerIntegrationTest: 일반/연속 잔여 분리, 기숙사 얼굴/QR만 자동 지정, 수동·자습실·과거·미래 인증 제외, 기존 일반 지정 유지, 건너뛰기·0회 종료, 조정 재시도/종류 충돌·입력/권한·응답 검증. 실제 독립 트랜잭션 6개를 동시에 실행해 지정·알림·완료 차감이 각각 한 번임을 확인. 잔여 0 완료 실패는 지정 상태·조정 기록도 롤백함을 확인.

## 미검증과 후속

- 웹 날짜 선택·연속 봉사 화면(Client#395·396)의 서버 연결, 실기기 얼굴·QR E2E, 학교 동의 최종 전문의 운영 적용, 운영 배포는 이번 서버 자동 테스트의 범위가 아니다.
- 이미 폐기된 도입 전 출석과 기록이 전혀 없던 서버 중단일의 명단은 복원하지 않는다. 과거 조회 결과가 비어 있을 수 있다.
- 요약은 백업 제외 대상이다. 현재 저장소에는 운영 백업 구현이 없어 실제 백업·복원 검증은 수행하지 않았다.
