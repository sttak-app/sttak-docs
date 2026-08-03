# 5-2. Error Handling & HTTP Status Codes — sTTak 백엔드

본 문서는 sTTak 백엔드의 **HTTP 상태코드 사용 기준**과 **예외 처리 정책**을 정리한다.
헷갈릴 때마다 표를 보고 한 줄짜리 매핑을 그대로 따라가면 된다.

전체 구조의 출처:

- `sttak-common`의 `BaseException` / `BusinessException` / `ErrorType`
- `sttak-api`의 `GlobalExceptionHandler`
- 응답 포맷 `ApiResponse<T> { message, content }` — `3-3-contract.md` §3-3.1.5

---

## 5-2.1 한눈에 보는 표 (Cheat Sheet)

| HTTP | 의미 한 줄 | 본 서비스에서 발생하는 대표 케이스 | ErrorType |
| --- | --- | --- | --- |
| **200 OK** | 성공 + 응답 본문 있음 | 조회 / 갱신 성공 | — |
| **201 Created** | 새 리소스가 생성됨 | `POST /trades`, `POST /retrospectives` | — |
| **204 No Content** | 성공했지만 본문 없음 | `DELETE /users/me/watchlist/{code}` (응답에 갱신 Watchlist를 같이 주면 200) | — |
| **400 Bad Request** | 요청 자체가 형식상 잘못됨 | 필수 필드 누락, 형식 오류, JSON 파싱 실패 | `VALIDATION` |
| **401 Unauthorized** | 너 누구냐 — **인증** 실패 | 토큰 없음 / 만료 / 위조 | `UNAUTHORIZED` |
| **403 Forbidden** | 너 인줄 알겠는데 못함 — **인가** 실패 | 사용자 토큰으로 `admin` 자원 접근 | `FORBIDDEN` |
| **404 Not Found** | 자원이 없음 | 미존재 사용자 / 뉴스 / 거래 / 종목 | `NOT_FOUND` |
| **409 Conflict** | 자원의 **현재 상태와 충돌** | Watchlist 5개 초과, 잔고 부족, 이미 채점된 퀴즈 | `CONFLICT` |
| **500 Internal Server Error** | 우리 잘못 | 예상치 못한 NPE / 외부 의존 무관 내부 오류 | `INTERNAL` |

> 빨리 결정할 때 룰: **인증 = 401, 인가 = 403, 자원 없음 = 404, 상태 충돌 = 409, 입력 잘못 = 400.**

---

## 5-2.2 200번대 (Success) 사용 기준

| 코드 | 언제 |
| --- | --- |
| **200 OK** | 조회 / 갱신 성공. 응답 본문이 있다 (`ApiResponse<T>` 그대로). |
| **201 Created** | 새 리소스 생성. 본문에 생성된 자원 또는 식별자 포함. `Location` 헤더는 선택. |
| **204 No Content** | 성공했지만 본문이 없음. **본 서비스에서는 거의 안 쓴다.** 클라이언트가 응답을 활용해야 하면 200 + 본문이 낫다. |

규약:

- 삭제(`DELETE`)는 **갱신된 상태**(예: 남은 Watchlist 전체)를 함께 돌려준다 → **200**.
- 진짜 “받을 게 없다” 만 204.

---

## 5-2.3 400 vs 401 vs 403 vs 404 vs 409 — 자주 헷갈리는 짝

### 400 vs 422

본 서비스는 **400으로 통일**한다. 형식 오류든 의미 오류든 클라이언트가 입력을 고쳐야 한다는 점에서 동일.
나중에 분리가 필요해지면 `ErrorType.VALIDATION_SEMANTIC`를 추가해 422로 매핑한다.

### 401 vs 403

| 질문 | 답 | 코드 |
| --- | --- | --- |
| 토큰이 아예 없거나, 만료/위조? | 너 누구냐 모름 | **401** |
| 토큰은 유효한데 이 자원에 권한 없음? | 너 인줄 알겠는데 못함 | **403** |

> Spring Security의 “Remember-Me”는 **인증** 연장이지 인가가 아니다. *(1-background.md 1.4.3 참조)*

### 404 vs 403

- 존재하지 않는 자원 → **404**.
- 존재하지만 접근 권한이 없는 자원 → **403** *(but 보안상 404로 숨기는 정책을 쓰는 곳도 있다. 본 서비스는 403 그대로 노출 — 운영 정보 노출이 크게 우려되는 자원은 없다.)*

### 404 vs 409

- 자원이 **없다** → 404.
- 자원이 있는데 **현재 상태가 요청과 충돌**한다 → 409.
  - “이 사용자의 Watchlist는 이미 5개” → 409
  - “이미 채점된 퀴즈를 다시 채점” → 409
  - “잔고 부족 매수” → 409

### 400 vs 409

- 입력 자체가 **형식·필수**상 잘못됨 → 400.
- 입력은 맞는데 **현재 상태와 양립 불가** → 409.

---

## 5-2.4 예외 계층 (코드 ↔ HTTP)

```
BaseException (interface)                    ← sttak-common
   ▲ implements
   │
<Context>Exception (enum)                    ← sttak-domain
   │   - 각 케이스가 (ErrorType, 메시지)를 보유
   │
   │ wrapped by
   ▼
BusinessException (RuntimeException)         ← sttak-common
   │
   ▼ caught by
GlobalExceptionHandler                       ← sttak-api
   │
   ▼
ApiResponse<T> + HTTP 상태코드
```

### 5-2.4.1 도메인 예외 enum 작성 규약

```java
public enum UserException implements BaseException {
    USER_NOT_FOUND          (ErrorType.NOT_FOUND, "사용자를 찾을 수 없습니다."),
    WATCHLIST_ITEM_NOT_FOUND(ErrorType.NOT_FOUND, "관심 종목을 찾을 수 없습니다."),
    WATCHLIST_LIMIT_EXCEEDED(ErrorType.CONFLICT,  "관심 종목은 최대 5개까지 등록할 수 있습니다."),
    EMAIL_FORMAT_INVALID    (ErrorType.VALIDATION,"이메일 형식이 올바르지 않습니다.");

    private final ErrorType errorType;
    private final String message;
    /* getter ... */
}
```

- 새 도메인 추가 시 항상 `<Context>Exception` enum을 함께 만든다.
- enum 케이스명은 `<자원>_<상황>` (예: `USER_NOT_FOUND`).
- **메시지는 사용자에게 그대로 노출 가능한 한국어.** 내부 로그용은 별도 컨텍스트에 담는다.

### 5-2.4.2 도메인에서 throw, 어딘가에서 wrap

도메인 객체는 enum을 **그대로 throw**하지 않는다 (`throw UserException.X`가 컴파일 안 됨).
대신 도메인 내부 헬퍼나 Application Service가 `throw new BusinessException(UserException.X)` 로 감싼다.

```java
// 도메인 객체 안
if (watchlist.size() >= MAX_SIZE) {
    throw new BusinessException(UserException.WATCHLIST_LIMIT_EXCEEDED);
}
```

`BusinessException`은 두 가지 생성자를 갖는다. **원인(cause) 예외를 넘길지는 "그 자리에 잡은 예외가 있느냐"로 갈린다.**

| 생성자 | 언제 | 판별 |
| --- | --- | --- |
| `BusinessException(BaseException)` | 도메인 규칙 위반을 **스스로 처음** 던질 때 (넘길 하위 예외가 없음) | `if` 검증에서 던진다 |
| `BusinessException(BaseException, Throwable cause)` | 하위 예외를 **`catch` 해서** 도메인 예외로 감쌀 때 | `catch (X e)` 안에서 던진다 |

```java
// (1) if 검증 — 넘길 e 가 없다
if (watchlist.size() >= MAX_SIZE) {
    throw new BusinessException(UserException.WATCHLIST_LIMIT_EXCEEDED);
}

// (2) 인프라/기술 예외를 감쌀 때 — 잡은 e 를 반드시 cause 로 넘긴다
try {
    response = openAiRestClient.post()...;
} catch (RestClientException e) {
    throw new BusinessException(QuizException.QUIZ_GENERATION_FAILED, e);
}
```

- `catch` 블록 안에서 던지는데 `e`를 안 넘기면 **근본 원인이 로그·Sentry에서 소실**된다 (§5-2.9의 "원인까지 적재"가 성립하려면 cause 체이닝이 필수).
- 두 생성자 모두 `super(baseException.getMessage())`를 호출해 예외 메시지를 채운다 — message가 `null`이면 관측 도구에서 예외 식별이 어렵다.

### 5-2.4.3 GlobalExceptionHandler 매핑

`sttak-api/.../exception/GlobalExceptionHandler` 단일 진입점.

| 예외 | 처리 | 결과 |
| --- | --- | --- |
| `BusinessException` | `ApiResponse.error(errorType, message)` | `ErrorType` → HTTP 매핑표 |
| `MethodArgumentNotValidException` (Bean Validation) | `ApiResponse.inputError(fieldErrors)` | **400 VALIDATION**, 필드별 메시지 노출 |
| `HttpMessageNotReadableException` (JSON 파싱 실패) | `ApiResponse.error(VALIDATION, ...)` | **400** |
| `MethodArgumentTypeMismatchException` (쿼리/패스 타입 오류) | `ApiResponse.error(VALIDATION, ...)` | **400** |
| `AccessDeniedException` (Spring Security) | `ApiResponse.error(FORBIDDEN, ...)` | **403** |
| `AuthenticationException` (Spring Security) | `ApiResponse.error(UNAUTHORIZED, ...)` | **401** |
| `HttpRequestMethodNotSupportedException` | `ApiResponse.error(...)` | **405** |
| `HttpMediaTypeNotSupportedException` | `ApiResponse.error(...)` | **415** |
| 그 외 `Exception` | `ApiResponse.error(INTERNAL, "잠시 후 다시 시도해주세요.")` + 스택트레이스 로그 | **500** |

### 5-2.4.4 ErrorType → HTTP 매핑

| ErrorType | HTTP |
| --- | --- |
| `VALIDATION` | 400 |
| `UNAUTHORIZED` | 401 |
| `FORBIDDEN` | 403 |
| `NOT_FOUND` | 404 |
| `CONFLICT` | 409 |
| `INTERNAL` | 500 |

---

## 5-2.5 에러 응답 모양

성공:
```json
{ "message": "OK", "content": { ... } }
```

도메인/비즈니스 오류:
```json
{ "message": "관심 종목은 최대 5개까지 등록할 수 있습니다.", "content": null }
```

입력 검증 오류 (필드별):
```json
{
  "message": "입력값이 올바르지 않습니다.",
  "content": {
    "fieldErrors": [
      { "field": "rationale", "reason": "매수 근거는 필수입니다." },
      { "field": "quantity",  "reason": "수량은 1 이상이어야 합니다." }
    ]
  }
}
```

내부 오류:
```json
{ "message": "잠시 후 다시 시도해주세요.", "content": null }
```

규약:

- **스택트레이스/내부 메시지는 응답에 절대 노출하지 않는다.**
- 추적은 응답 헤더 `X-Request-Id`로 한다 *(NFR-O1)*.

---

## 5-2.6 도메인별 대표 시나리오

| 컨텍스트 | 케이스 | HTTP / ErrorType | 메시지 예 |
| --- | --- | --- | --- |
| 인증 | access 만료 | 401 UNAUTHORIZED | "로그인이 필요합니다." |
| 인증 | refresh 만료/위조 | 401 UNAUTHORIZED | "다시 로그인해주세요." |
| 인가 | 사용자 토큰으로 admin 자원 | 403 FORBIDDEN | "권한이 없습니다." |
| User | 미존재 사용자 ID 조회 | 404 NOT_FOUND | "사용자를 찾을 수 없습니다." |
| Watchlist | 6번째 등록 | 409 CONFLICT | "관심 종목은 최대 5개까지 등록할 수 있습니다." |
| Watchlist | 미지원 종목 | 404 NOT_FOUND | "지원하지 않는 종목입니다." |
| Watchlist | 중복 등록 | 409 CONFLICT | "이미 등록된 종목입니다." |
| Trade | rationale 빈 값 | 400 VALIDATION | "매수 근거는 필수입니다." |
| Trade | 잔고 부족 | 409 CONFLICT | "현금 잔고가 부족합니다." |
| Trade | 음수 수량 | 400 VALIDATION | "수량은 1 이상이어야 합니다." |
| Trade | 미지원 종목 | 404 NOT_FOUND | "지원하지 않는 종목입니다." |
| Quiz | 미존재 문항 채점 | 404 NOT_FOUND | "퀴즈를 찾을 수 없습니다." |
| Quiz | 쿨다운 중 응답 제출 (`POST /quizzes/{quizId}/submit`) | 409 CONFLICT | "다음 퀴즈는 쿨다운 후 응시할 수 있습니다." |
| Quiz | 이미 응답한 문항 재제출 | 409 CONFLICT | "이미 응시한 퀴즈입니다." |
| Quiz | 쿨다운 중 다음 문항 조회 (`GET /quizzes/next`) | 200 OK (응답 본문에 `cooldown=true`, `nextAvailableAt`) | "다음 퀴즈는 …후에 받을 수 있어요." |
| Quiz | 풀에 남은 미응시 문항 0개 | 409 CONFLICT | "응시 가능한 퀴즈가 없습니다." |
| News | 미존재 뉴스 | 404 NOT_FOUND | "뉴스를 찾을 수 없습니다." |
| Chat | 미지원 mode | 400 VALIDATION | "지원하지 않는 모드입니다." |
| 외부 의존 | LLM 어댑터 서킷 OPEN + 폴백 불가 | 503 (메시지로 안내) | "지금은 답변을 드리기 어려워요. 잠시 후 다시 시도해주세요." |

> “쿨다운 중”처럼 **정상 시나리오**는 에러로 처리하지 않는다. 200 + 본문 안내가 더 적절.

---

## 5-2.7 SSE 스트림에서의 에러

챗봇 등 SSE 응답 도중 오류:

```
event: error
data: {"message":"답변 생성 중 문제가 발생했어요. 잠시 후 다시 시도해주세요."}

event: done
data: {}
```

- 스트림 시작 **전** 오류 (예: 인증 실패, 모드 오류)는 일반 HTTP 응답으로 4xx를 반환.
- 스트림 시작 **후** 오류는 `event: error` → `event: done` 순서로 종료.

---

## 5-2.8 외부 의존 장애 시의 정책

| 상황 | 정책 |
| --- | --- |
| LLM 타임아웃 | 1회 재시도 → 실패 시 캐시된 직전 응답 / 안내 메시지로 폴백 |
| 시세 API 장애 | 마지막 캐시된 시세 + `fetchedAt` 표시 |
| 뉴스 API 장애 | 마지막 적재 시점 데이터 노출 + `fetchedAt` 표시 |
| 서킷브레이커 OPEN | 폴백이 있으면 200/안내, 없으면 503 |

**핵심:** 외부 장애가 사용자 경로의 4xx/5xx로 그대로 전이되지 않도록 **어댑터 단에서 흡수**한다 *(NFR-R1)*.

---

## 5-2.9 로그 / 추적과의 연계

- 모든 에러 응답은 `X-Request-Id`와 함께 로깅된다 *(6-observability.md)*.
- 5xx만 ERROR 레벨로 적재, 4xx는 WARN 또는 INFO (양에 따라).
- `BusinessException`은 **스택트레이스를 짧게**(원인까지만) 적재해 로그 비용을 줄인다.
- `INTERNAL` 분류 예외는 **전체 스택트레이스 + 사용자 ID + 요청 본문 요약**을 함께 남긴다 (단, PII는 마스킹).

---

## 5-2.10 자주 하는 실수

| 실수 | 올바른 처리 |
| --- | --- |
| 인증 실패에 403 사용 | **401** |
| 인가 실패에 401 사용 | **403** |
| 잔고 부족에 400 사용 | **409 CONFLICT** (입력은 맞고 상태가 문제) |
| “이미 채점됨”에 400 사용 | **409 CONFLICT** |
| 사용자가 처리 불가한 내부 오류 메시지를 응답에 노출 | "잠시 후 다시 시도해주세요." + `X-Request-Id`만 노출 |
| 도메인 enum을 throw | 항상 `BusinessException(enum)`로 감싸 throw |
| 컨트롤러에서 try/catch | `GlobalExceptionHandler`에 위임 |
| 다른 응답 모양 (`{ "code": ... }`) 도입 | 모든 응답은 **`ApiResponse<T>`** 한 가지 모양 |

---

## 5-2.11 후속 문서

| 문서 | 연결 |
| --- | --- |
| `3-1-server-architecture.md` | 예외 계층의 원본 정의 |
| `3-3-contract.md` | 엔드포인트별 에러 코드 표 |
| `6-observability.md` | 에러 로그/메트릭/`X-Request-Id` |
