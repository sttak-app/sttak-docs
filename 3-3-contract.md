# 3-3. API Contract Conventions — sTTak 백엔드

본 문서는 sTTak 백엔드 API의 **개발자 간 공통 약속(컨벤션)** 을 정의한다.
URL/메서드/응답 모양/페이지네이션/에러 코드 등 “전 API가 따라야 하는 규칙”이 단일 출처다.

엔드포인트별 시그니처와 스키마는 별도 iosAPI 산출물(`docs/iosapi/` 예정)에서 관리한다.
**본 문서는 “어떻게 만들 것인가”에 답하고, OpenAPI는 “무엇을 만들 것인가”에 답한다.**

연결 문서:

- 시스템 구조: `3-1-server-architecture.md`
- 요구사항: `2-2-requirements.md`
- 에러 매핑: `5-2-error-handling.md`
- `X-Request-Id`·로깅·메트릭: `6-observability.md`

---

## 3-3.1 Base URL & 버저닝

- Base URL: `https://api.sttak.app`
- 모든 엔드포인트는 **`/api/v{N}`** 접두사를 갖는다. 현재 `v1`.
- 호환성 깨는 변경은 `v2` 신설로 처리. `v1`을 무단 수정하지 않는다.
- 단일 엔드포인트의 deprecate는 응답 헤더 `Deprecation: <date>` + `Sunset: <date>`로 알리고, 최소 1 메이저 버전 유예.

---

## 3-3.2 HTTP 메서드

| 메서드 | 용도 |
| --- | --- |
| `GET` | 조회. 부수효과 없음. 캐싱 가능. |
| `POST` | 생성, 행위(execute), 비-CRUD 명령. |
| `PUT` | 전체 교체. (현재 사용 안 함) |
| `PATCH` | 부분 변경. |
| `DELETE` | 삭제. |

규약:

- 조회는 `GET`만 사용. 검색 파라미터가 길어진다고 `POST` 우회 금지 (캐시·관측·로깅이 깨진다).
- 비-CRUD 행위(`/quizzes/{id}/submit`, `/auth/refresh`)는 `POST` + 명사+행위 경로.

---

## 3-3.3 URL 명명

- **명사 + 복수형** 사용: `/users`, `/trades`, `/quizzes`.
- 본인 자원은 `/users/me/...` 별칭으로 노출 (JWT의 `sub`를 사용해 식별).
- 행위는 자원의 하위 경로: `POST /quizzes/{id}/submit`, `POST /auth/refresh`.
- 자원 ID는 path parameter, 필터/정렬/페이징은 query parameter.
- snake_case / camelCase 혼용 금지 — **URL 세그먼트는 kebab-case, 쿼리/바디 필드는 camelCase**.
- 종목 등 자연 식별자가 있는 자원은 자연 식별자를 우선한다 (`/stocks/005930`). 도메인 ID(예: `t_01H...`)는 응답에 포함.

---

## 3-3.4 표준 헤더

| 헤더 | 방향 | 의미 |
| --- | --- | --- |
| `Authorization: Bearer <access-token>` | 요청 | 인증 토큰 (인증 필요 경로에서 필수) |
| `Content-Type: application/json; charset=utf-8` | 요청 | JSON 바디 |
| `Accept: application/json` 또는 `text/event-stream` | 요청 | SSE 엔드포인트는 SSE 헤더 |
| `X-Request-Id: <uuid>` | 요청(선택) / 응답(항상) | trace ID. 클라이언트가 보내면 그대로 echo, 없으면 서버가 생성 *(NFR-O1)* |
| `Idempotency-Key: <uuid>` | 요청(선택) | 쓰기 멱등 보장이 필요한 일부 엔드포인트에서 사용 |
| `Deprecation` / `Sunset` | 응답 | deprecated 엔드포인트 알림 |

규약:

- 비표준 헤더는 모두 **`X-` 접두사**.
- 인증 정보·키를 쿼리 파라미터로 받지 않는다 (로그/CDN에 새는 것을 막기 위해).

---

## 3-3.5 표준 응답 포맷 (`ApiResponse<T>`)

모든 응답은 다음 한 가지 모양을 따른다.

```json
{
  "message": "OK",
  "content": { ... }
}
```

- 성공 시 `content`는 페이로드, `message`는 `"OK"` 또는 사용자용 안내.
- 실패 시 `content`는 `null` 또는 보조 정보(`fieldErrors` 등), `message`는 사용자용 메시지.
- **이 모양 외의 응답 형태는 도입하지 않는다.** 별도 `code`/`errorCode` 필드를 응답 바디에 추가하지 않는다 (분류는 HTTP 상태코드 + `5-2-error-handling.md`).

> 자세한 에러 응답·상태코드 매핑은 `5-2-error-handling.md`.

---

## 3-3.6 페이지네이션 (커서 기반)

리스트 API는 **커서 기반**으로 통일.

쿼리:
- `cursor` (string, optional) — 마지막 항목의 커서. 첫 페이지는 생략.
- `size` (int, default=20, max=100) — 페이지 크기.

응답:
```json
{
  "message": "OK",
  "content": {
    "items": [ ... ],
    "nextCursor": "eyJpZCI6MTIzfQ==",
    "hasNext": true
  }
}
```

규약:

- offset/limit 기반은 도입하지 않는다 (대규모 페이지에서 일관성/성능 문제).
- 커서는 **불투명(opaque)** 으로 다룬다. 클라이언트는 디코딩하지 않는다.

---

## 3-3.7 정렬 / 필터

- 정렬: `?sort=<field>,<asc|desc>` (예: `?sort=executedAt,desc`).
- 필터: 쿼리 파라미터로. 복수 값은 `?status=OPEN&status=CLOSED` 또는 `?status=OPEN,CLOSED` (서버는 양쪽 모두 허용).
- 필터 키는 응답 필드명과 동일하게 맞춘다 (외부에 노출되는 이름이 곧 필터 키).

---

## 3-3.8 타임스탬프 / 통화 / 식별자

| 영역 | 규약 |
| --- | --- |
| 시각 | **ISO 8601 UTC** (`2026-06-23T15:04:05Z`). 타임존 정보 누락 금지. |
| 통화 (MVP) | KRW 단일. 금액 필드는 **정수(원)** 로 직렬화한다. (예: `"price": 71200`, `"cash": 8738000`) |
| 통화 (확장) | 다중 통화 도입 시 `{ "amount": 1234.56, "currency": "KRW" }` 객체로 전환한다 *(CON-S3)*. |
| 자원 ID | `<prefix>_<ULID>` 형식 권장 (예: `u_01H...`, `t_01H...`). 단조 증가 + 정렬 가능. |
| 내부 PK | 외부 응답에 노출하지 않는다. (도메인 ID/자연 식별자만 노출) |
| 종목 코드 | `stockCode` (string, 좌측 0 패딩 유지: `005930`). |
| 거래 시장 | `market` (enum, §3-3.15). MVP는 `KOSPI`, `KOSDAQ`. |
| enum | **대문자 SCREAMING_SNAKE_CASE 문자열**로 직렬화. 값 목록은 §3-3.15. |

---

## 3-3.9 SSE 스트리밍 컨벤션

LLM 응답 등 토큰 단위 응답:

- 응답 헤더: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`.
- 이벤트 종류:
  ```
  event: token   data: {"text":"..."}      ← 본문 토큰
  event: source  data: {"sources":[...]}    ← 사용된 출처(URL/제목)
  event: done    data: {}                   ← 정상 종료
  event: error   data: {"message":"..."}    ← 스트림 시작 후 오류
  ```
- 스트림 시작 **전** 오류 (인증 실패, 입력 검증 실패)는 일반 HTTP 4xx로 반환.
- 스트림 시작 **후** 오류는 `event: error` → `event: done` 순서로 종료.
- 클라이언트가 끊을 수 있도록 keepalive 코멘트(`:`)를 30초마다 전송.

---

## 3-3.10 멱등성 (Idempotency)

- 쓰기 호출이 **재시도되어도 같은 결과**가 되도록 권장.
- 다음 엔드포인트는 `Idempotency-Key` 헤더 수신 시 **24시간 내** 같은 키 + 같은 본문 요청을 **동일 응답으로 회신**한다:
  - 매매 실행
  - 퀴즈 채점
- 서버는 키 단위로 `(상태, 응답 페이로드)`를 저장한다.
- (MVP 우선순위 낮음 — 단계적 도입.)

---

## 3-3.11 인증 / 인가 (HTTP 표현 약속)

- 인증 토큰은 **`Authorization: Bearer <access-token>`** 헤더로만 받는다. 쿠키·세션 미사용.
- 토큰 만료/위조 → `401 UNAUTHORIZED`.
- 권한 부족 → `403 FORBIDDEN`.
- 토큰의 페이로드(클레임)는 클라이언트가 신뢰할 수 없다 — 서버는 항상 재검증한다.
- 소셜 로그인은 **공급자별 엔드포인트**(`POST /auth/kakao` · `/auth/google` · `/auth/apple`)로 노출한다. 공급자마다 자격증명 형식(OAuth Access Token / OIDC `idToken` / 애플 `identityToken`)이 달라 단일 엔드포인트로 묶지 않는다(근거: `docs/decisions/ADR-008-social-login.md`). 엔드포인트별 입출력은 `docs/iosapi/apidocs.md §1`.
- 토큰 갱신은 별도의 행위 경로(`POST /auth/refresh`)로 처리하고, 일반 API의 자동 재발급은 하지 않는다.

> 인증/인가의 시스템 구조는 `3-1-server-architecture.md §3-1.11`. JWT 설계는 `docs/decisions/ADR-005-jwt-design.md`.

---

## 3-3.12 캐싱

- `GET` 응답 중 외부 데이터를 가공한 것은 `fetchedAt`(ISO 8601)을 본문에 포함한다 *(NFR-D3)*.
- HTTP 캐시 헤더(`Cache-Control`, `ETag`)는 MVP에서 사용하지 않는다 (필요 시점에 도입).
- 클라이언트는 응답이 **지연 데이터**일 수 있음을 가정한다 *(CON-S2)*.

---

## 3-3.13 발생 가능한 에러 코드 (전 API 공통)

| HTTP | ErrorType | 발생 시나리오 (예) |
| --- | --- | --- |
| 400 | VALIDATION | 필수 필드 누락 / 형식 오류 / JSON 파싱 실패 |
| 401 | UNAUTHORIZED | access 만료, refresh 만료, 토큰 위조 |
| 403 | FORBIDDEN | 권한 부족 (사용자 토큰으로 admin 자원 접근 등) |
| 404 | NOT_FOUND | 미존재 자원 |
| 409 | CONFLICT | 현재 상태와 충돌 (Watchlist 5개 초과, 잔고 부족, 이미 채점된 퀴즈 등) |
| 415 | (자동) | Content-Type 미지원 |
| 500 | INTERNAL | 내부 오류 |
| 503 | INTERNAL | 외부 의존성 중단 + 폴백 불가 *(NFR-R1)* |

> 자세한 매핑·시나리오·메시지 예는 `5-2-error-handling.md`.

---

## 3-3.14 변경 정책

- **하위호환 깨는 변경**: 새 메이저(`v2`) 신설. 기존 클라이언트는 `v1`을 일정 기간 사용.
- **하위호환 변경 (필드 추가 등)**: `v1` 내에서 허용. 단, 응답 **필드 삭제는 금지** — 더 이상 사용하지 않으면 deprecated 표시 후 다음 메이저에서 제거.
- 응답 enum에 **새 값 추가는 깨는 변경으로 간주**한다. 클라이언트는 unknown 값을 graceful하게 다룰 책임을 진다.
- 모든 계약 변경은 PR 단위로 본 문서 + OpenAPI 산출물을 함께 갱신한다.

---

## 3-3.15 공통 Enum

응답/요청에 등장하는 enum 값은 모두 **대문자 SCREAMING_SNAKE_CASE 문자열**로 직렬화한다.
직렬화 시 변환(`lower`, `Pascal` 등)하지 않고 그대로 노출한다. 새 enum 값 추가는 §3-3.14에 따라
**깨는 변경으로 간주**되며, 클라이언트는 unknown 값을 graceful하게 다룬다.

### `AuthProvider` — 소셜 로그인 공급자

| 값 | 설명 |
| --- | --- |
| `KAKAO` | 카카오 |
| `GOOGLE` | 구글 |
| `APPLE` | 애플 |

### `Market` — 거래 시장

| 값 | 설명 |
| --- | --- |
| `KOSPI` | 유가증권시장 |
| `KOSDAQ` | 코스닥시장 |

### `TradeType` — 매매 구분

| 값 | 설명 |
| --- | --- |
| `BUY` | 매수 |
| `SELL` | 매도 |

### `Sentiment` — 뉴스의 호재/중립/악재 분류

같은 기사라도 종목 조합에 따라 값이 달라질 수 있다 (뉴스–종목 쌍 단위 평가).

| 값 | 설명 |
| --- | --- |
| `POSITIVE` | 호재 |
| `NEUTRAL` | 중립 |
| `NEGATIVE` | 악재 |

### `ChatContext` — 챗봇 호출 맥락

추천 질문/응답 톤 분기에 사용한다.

| 값 | 설명 |
| --- | --- |
| `NEWS` | 뉴스 기사 |
| `CHART_SEGMENT` | 차트 구간 |
| `FREE` | 자유 질문 |

### `ChatRole` — 챗 메시지 화자

| 값 | 설명 |
| --- | --- |
| `USER` | 사용자 발화 |
| `ASSISTANT` | 챗봇 발화 |

---

## 3-3.16 후속 문서

| 문서 | 연결 |
| --- | --- |
| `docs/iosapi/apidocs.md` | 엔드포인트별 입출력 명세 (본 문서의 공통 규약을 따른다) |
| `docs/openapi/` *(예정)* | OpenAPI 형식의 시그니처/스키마 |
| `2-2-requirements.md` | 본 컨벤션이 만족시키는 FR/NFR |
| `3-1-server-architecture.md` | API 뒤의 도메인/계층 |
| `5-2-error-handling.md` | 에러 응답 상세 매핑 |
| `6-observability.md` | `X-Request-Id`·로깅·메트릭 |
