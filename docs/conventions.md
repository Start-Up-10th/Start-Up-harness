# Git·PR 컨벤션

하네스와 이 하네스를 서브모듈로 쓰는 서비스 레포에 공통으로 적용한다. 결정 근거는 [DEC-015](decisions.md), [DEC-017](decisions.md).

## PR 제목·커밋 메시지

- 커밋 메시지: `type: 설명`
- PR 제목: `[type] 설명`

두 형식은 같은 type 표를 쓴다.

| type | 용도 |
| --- | --- |
| `feat` | 새 기능 |
| `fix` | 버그 수정 |
| `refactor` | 동작 변화 없는 구조 개선 |
| `test` | 테스트 추가·수정 |
| `docs` | 문서·명세·계획 |
| `ci` | CI/CD 워크플로 |
| `chore` | 설정·의존성·서브모듈 갱신 등 기타 |

- type은 소문자로 쓴다.
- 설명은 한국어로 무엇을 바꿨는지 짧게 쓰고 마침표를 붙이지 않는다.
- 커밋 예: `feat: QR 세션 발급 API 추가`, `fix: 운영일 경계 계산 오류 수정`, `chore: 하네스 최신화`
- PR 제목 예: `[feat] QR 세션 발급 API`, `[chore] 하네스 최신화`
- 이미 머지되거나 push된 다른 형식(`[type] 설명` 커밋, `[CHORE]` 등)의 이력은 고치지 않는다.

## 브랜치

- `type/짧은-영문-설명`을 쓴다. 예: `feat/qr-session`, `chore/harness-update`, `ci/server-test-db`
- type은 위 표와 같다.

## PR 대상

- 서비스 레포: 기능 PR은 `develop`으로 보낸다. approve 1개와 CI 통과가 필요하다. `main`은 배포할 때만 `develop`에서 머지한다.
- 하네스 레포: `main`으로 보낸다.
- PR 본문은 각 레포의 PR 템플릿을 따른다.

## 서버 테스트 작성

Spring 서버(JUnit 5) 테스트는 아래 형식을 따른다. 결정 근거는 [DEC-022](decisions.md).

```java
import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

/**
 * 운영일이 Asia/Seoul 08:00 경계로 계산되는지 검증한다(DEC-006).
 */
class OperatingDayCalculatorTest {

    @Test
    @DisplayName("오전 8시 직전은 전날 운영일이다")
    void justBefore8IsPreviousOperatingDay() {
        // given / when / then
    }
}
```

- 메서드명은 영문 camelCase로 쓰고, 한국어 설명은 `@DisplayName`에 쓴다. 한글 메서드명은 쓰지 않는다.
- 테스트 클래스 위 Javadoc에 무엇을 검증하는지 한 문장으로 쓴다. 관련 REQ·DEC가 있으면 괄호로 함께 적는다.
- `@Test`·`@ParameterizedTest` 바로 아래에 `@DisplayName`을 둔다.
- `static` import는 맨 위에 모으고, 한 줄을 띄운 뒤 나머지 import를 쓴다.
- 이미 머지된 테스트도 수정할 때 이 형식으로 맞춘다. 다른 PR이 고치고 있는 테스트는 충돌을 피하려고 그 PR이 머지된 뒤에 맞춘다.
