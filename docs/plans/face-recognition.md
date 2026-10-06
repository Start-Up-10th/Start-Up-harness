# face-recognition — 얼굴 등록·인식 계획

## 현재 구현 상태

- Spring 얼굴 API는 [`face.openapi.yaml`](../../contracts/face.openapi.yaml)에 기록한다. 학생 상태·동의·최초 등록과 관리자 세션·프레임 인식·종료 경로가 구현돼 있다.
- Spring은 AI 호출에 `FACE_SERVICE_TOKEN` Bearer만 쓰고 브라우저 `SESSION`이나 DataGSM 토큰을 전달하지 않는다.
- AI의 canonical DataGSM 학생 ID를 현재 Spring 세션 후보와 대조한 뒤 CheckUp DB 학생 키로 출석을 기록한다. unknown은 학생 이름·학번·출석 결과를 만들지 않는다.
- 원본 등록 영상과 인식 프레임은 처리 뒤 요청 메모리 버퍼에서 비우며, 학생 이탈 시 얼굴 템플릿과 관련 인식 세션 후보를 정리한다.
- 실제 배포 보호 AI 경로, 웹 카메라 제품 흐름, 승인된 얼굴 데이터의 정확도는 이 코드 상태만으로 검증 완료 처리하지 않는다.

## 담당자·API

- 담당자: 임서하
- Spring 공개 API: `GET /api/v1/face/me`, `POST /api/v1/face/consent`, `POST /api/v1/face/enrollments`, `POST /api/v1/face/sessions`, `POST /api/v1/face/sessions/{sessionId}/frames`, `DELETE /api/v1/face/sessions/{sessionId}`
- `GET /api/v1/face/recognitions?purpose=&limit=` — 최근 인식 목록(관리자 본인 세션, 오늘, 최신순, limit 1~100·기본 50). 응답 `[{ recognizedAt, result(SUCCESS/FAILED), studentName, studentNumber }]`, 못 알아본 얼굴은 이름·학번 null(DEC-029)
- AI 내부 API: `POST /internal/v1/face/enrollments/extract`, `PUT /internal/v1/face/sessions/{session_id}`, `POST /internal/v1/face/sessions/{session_id}/frames`, `DELETE /internal/v1/face/sessions/{session_id}`
- 브라우저는 Spring `SESSION` 쿠키로 인증하며 AI endpoint나 `FACE_SERVICE_TOKEN`을 직접 사용하지 않는다.

## 구현

- 동의 완료 학생의 휴대폰 최초 얼굴 등록
- 진입 즉시 카메라 자동 실행, 카운트다운 → 촬영 중 → 완료 3단계로 진행; 셔터 버튼 없음
- `다시 찍기`는 촬영본을 폐기하고 카운트다운부터 반복, `완료`는 등록 요청 후 학생 홈으로 이동
- 약 100프레임 품질 평가와 약 20개 대표 벡터 저장
- 원본 영상·프레임을 성공·실패·취소·오류 후 즉시 폐기
- MediaPipe 검출/랜드마크와 신원 임베딩 모델 분리·실측
- 관리자 카메라의 목적별 인식 세션, 다수 얼굴 트랙, unknown, 실패 안내
- AI 결과만 반환하고 권위 있는 출석 확정은 Spring이 담당
- 얼굴 출석 확정은 QR과 같은 출석 테이블·저장 규칙을 쓴다. 스키마 제안은 [qr-attendance.md](qr-attendance.md) "출석 테이블 제안".

## 구현 확인 및 남은 검증

- [x] Spring은 원본 `video/webm` 또는 `video/mp4` body를 AI에 전달하고 모델 메타데이터 및 정규화 벡터를 저장한다.
- [x] Spring→AI 요청은 서비스 Bearer 인증을 사용하고 브라우저 cookie·DataGSM OAuth token을 전달하지 않는다.
- [x] 다수 얼굴의 track, quality, known/unknown/not-attempted, 출석 결과를 Spring 응답으로 표현한다.
- [x] AI 결과는 Spring 세션의 후보 목록과 대조하며 unknown에 학생 신원을 할당하지 않는다.
- [x] 온라인 얼굴 출석은 Spring `AttendanceService`의 용도·운영일 단위 저장 및 중복 방지를 사용한다.
- [ ] 실제 배포 AI의 protected enrollment/session/frame 호출을 같은 `FACE_SERVICE_TOKEN`으로 확인한다. 실제 자격증명이 필요하고 공개 HTTP 주소에는 토큰을 보내지 않는다.
- [ ] 브라우저→Spring→AI 제품 E2E와 카메라 이탈 시 자원 종료를 확인한다. 이번 작업에서 제외한 항목이며 acceptance는 미검증 상태로 둔다.
- [ ] 오프라인 인식·임시 기록·복구 후 동기화는 기기 신뢰 및 시계 정책 결정 뒤 구현한다. 온라인 경로를 오프라인 지원으로 간주하지 않는다.
- [ ] 승인된 실제 얼굴 데이터로 동일인·타인·unknown·저조도·다수 얼굴·임계값을 검증한다.

## 기준·검증

- 요구사항: `REQ-FACE-001~008`, `REQ-ATT-002`, `REQ-ATT-007`, `REQ-UI-005~006`
- 수용 시나리오: `ACC-FACE-001~008`, `ACC-ATT-002`, `ACC-ATT-007`, `ACC-UI-005~006`
- 같은 사람·다른 사람·미등록·저조도·다수 얼굴·점수 경계를 승인된 테스트 데이터로 확인한다.
- 성공·실패 얼굴의 이름·실패 횟수·출석 상태를 섞지 않는다.
- raw 얼굴·프레임이 DB·캐시·로그·임시 파일·백업에 남지 않는지 확인한다.
- 벡터 삭제 후 기기 캐시와 백업 복원에서 재출현하지 않는지 확인한다.

## 연결 설정

Spring `FACE_AI_BASE_URL`에는 AI 서비스 origin을 설정한다. readiness 주소가 `http://service.gsmsv.site:32200/health/ready`라면 base URL은 `http://service.gsmsv.site:32200`이다. Spring이 `/health/ready` 경로를 붙인다.

`FACE_SERVICE_TOKEN`은 AI와 Spring에 같은 secret으로 주입한다. public HTTP 주소에는 이 토큰을 보내지 않는다. 보호 호출은 HTTPS 또는 신뢰된 사설망에서만 시험하고, AI 세션은 메모리에 있으므로 단일 인스턴스 또는 `session_id` 기준 고정 라우팅을 사용한다.

## 2026-10-05 검증 기록

- Spring 얼굴 패키지 테스트 22개 통과, live integration test 1개는 실행 환경 변수가 없어 skip. `FaceStudentLeftListenerTest`는 고유 Postgres schema에서 실행하고 종료 뒤 해당 schema만 삭제한다.
- `npm.cmd run harness:check`는 39 REQ/39 ACC의 문서 검사와 하네스 테스트 20/20을 통과했다. acceptance 시나리오 0/39를 verified로 표시했다.
- 전체 `gradlew build`는 컴파일·패키징 뒤 테스트에서 실패했다(333개 중 52 fail, 1 skip). 여러 DB 통합 테스트가 공유 `checkup` 스키마의 Flyway V3 checksum mismatch를 만났고, QR controller 테스트는 `PUBLIC_ORIGIN` 미설정, 기존 미추적 `RoomMapDatabaseIntegrationTest`는 404를 받았다. 로컬 DB migration history와 해당 미추적 파일은 수정하지 않았다.
