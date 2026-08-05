# 3-1. Server Architecture — sTTak 백엔드

본 문서는 sTTak 백엔드 서버의 **시스템 아키텍처**를 정의한다.
원래 루트의 `ARCHITECTURE.md`에 있던 코드/모듈 레벨 규약과
스펙 디렉터리의 상위 아키텍처(두 갈래 처리, 외부 어댑터, LLM 토폴로지)를 **한 곳으로 통합**한다.

- 디렉터리/패키지 가시성 규약: 본 문서 §3-1.5, §3-1.10
- API 시그니처: `3-3-contract.md`
- 예외 → HTTP 매핑: `5-2-error-handling.md`
- 테스트 정책: `5-1-test-case.md` (루트 `TESTING.md`도 참조)

핵심 원칙은 세 가지다:

1. **DDD + Hexagonal + CQRS** — 도메인 모델은 인프라/HTTP를 모른다.
2. **두 갈래 처리 구조** — 사용자 응답 경로(`sttak-api`)와 데이터 파이프라인(`sttak-batch`)을 분리한다.
3. **외부 시스템은 Port 뒤로 숨긴다** — LLM/시세/뉴스 API는 `sttak-external`의 어댑터로만 호출한다.

---

## 3-1.1 시스템 컨텍스트 (System Context)

```
        ┌─────────────┐                ┌────────────────┐
        │ iOS Client  │ ──HTTPS/JWT──▶ │   sttak-api    │
        └─────────────┘                │ (REST + SSE)   │
                                       └────┬───────────┘
                                            │
        ┌──────────────────────────────────┐ │
        │ 관리자 (내부)                     │─▶│  sttak-admin
        └──────────────────────────────────┘   (관리자 콘솔 백엔드)
                                            │
                                            ▼
                          ┌──────────────────────────────────┐
                          │       sttak-domain (Port)        │
                          │  + sttak-common (공통 응답/예외)  │
                          └────────────────┬─────────────────┘
                                           │
                       ┌───────────────────┼───────────────────┐
                       ▼                   ▼                   ▼
              ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
              │   PostgreSQL   │  │    Vector DB   │  │  Graph 저장소  │
              │     (운영)     │  │   (임베딩)     │  │  (인과 그래프) │
              └────────────────┘  └────────────────┘  └────────────────┘
                                           ▲
                                           │
                                  ┌────────┴─────────┐
                                  │   sttak-batch    │
                                  │ (스케줄 / 파이프) │
                                  └────────┬─────────┘
                                           │
                          ┌────────────────┴────────────────┐
                          ▼                                 ▼
                  ┌───────────────────┐         ┌───────────────────┐
                  │ sttak-external    │         │  외부 API 벤더    │
                  │ (Port 구현 어댑터)│ ─────▶  │ NewsAPI · DART    │
                  └───────────────────┘         │ data.go.kr · LLM  │
                                                └───────────────────┘
```

흐름 요약:

- **실시간 경로(A)** — `iOS Client` → `sttak-api` → `sttak-domain` → 저장소.
  외부 API는 직접 호출하지 않고 **이미 가공된 데이터**(요약/임베딩/그래프)를 읽기만 한다.
- **비동기 경로(B)** — `sttak-batch`가 스케줄링되어 외부 API를 호출하고,
  정제·태깅·요약·임베딩·그래프 추출 결과를 같은 저장소에 적재한다.
- 두 경로는 **동일한 도메인 모델/Port를 공유**하지만 트리거가 다르다.

---

## 3-1.2 모듈 토폴로지

```
sttak-backend-demo (root, 실행 안 함)
├── sttak-apps/                  컨테이너 (자바 소스 없음)
│   ├── sttak-api      ─ Spring Boot 앱 (사용자용 REST API)
│   ├── sttak-admin    ─ Spring Boot 앱 (관리자 콘솔 백엔드)
│   └── sttak-batch    ─ Spring Boot 앱 (데이터 파이프라인/스케줄러)
├── sttak-domain        ─ 라이브러리 (도메인 모델 + JPA 영속 어댑터)
├── sttak-common        ─ 라이브러리 (응답/예외 등 횡단 관심사)
└── sttak-external      ─ 라이브러리 (외부 시스템 어댑터)
```

빌드 규약:

- 실행 모듈(`sttak-apps/*`)은 **`bootJar`만** 생성, plain `jar`는 비활성화.
- 라이브러리 모듈(`sttak-domain`, `sttak-common`, `sttak-external`)은 **`plain jar`만** 생성, `bootJar` 비활성화.
- `sttak-apps`는 자바 소스가 없는 컨테이너이므로 플러그인 적용 대상에서 제외.
- 공통: Java 21, Lombok, JUnit 5 / AssertJ / Mockito / Instancio, Jacoco(라인 커버리지 50% 이상 검증).

### 모듈 의존 그래프

```
       ┌─────────────────────┐
       │     sttak-common    │  ◄── 모든 모듈이 의존
       └──────────┬──────────┘
                  │
       ┌──────────▼──────────┐
       │     sttak-domain    │  ◄── 도메인 모델 + 영속 어댑터
       └──────────┬──────────┘
                  │
       ┌──────────▼──────────┐
       │    sttak-external   │  ◄── 외부 시스템 어댑터 (LLM/시세/뉴스)
       └──────────┬──────────┘
                  │
   ┌──────────────┼──────────────┐
   │              │              │
┌──▼────┐   ┌─────▼─────┐   ┌────▼────┐
│ api   │   │  admin    │   │  batch  │
└───────┘   └───────────┘   └─────────┘
```

방향은 항상 **위 → 아래로만** 흐른다. `sttak-domain`이 `sttak-apps/*`를 알지 못하고,
`sttak-common`은 어느 누구도 모른다 → 의존성 역전.

---

## 3-1.3 계층 매핑 (DDD 4 Layers)

DDD의 4계층을 모듈/패키지에 다음과 같이 매핑한다.

| 계층 | 위치 | 책임 |
| --- | --- | --- |
| **Presentation** | `sttak-api/.../<context>/controller`, `.../dto` | HTTP I/O, 직렬화, 입력 검증 |
| **Application** | `sttak-api/.../<context>/service`, `.../command`, `.../result` | 트랜잭션 경계, 유스케이스 조립 |
| **Domain** | `sttak-domain/.../<context>` (루트 패키지) | 애그리거트/엔티티/VO, 도메인 예외, Repository **Port** |
| **Infrastructure (Persistence)** | `sttak-domain/.../<context>/persistence`, `.../query` | JPA 어댑터 = Repository **Adapter** |
| **Infrastructure (External I/O)** | `sttak-external/.../<context>` | LLM/HTTP/메시지 큐 어댑터 |
| **Cross-cutting** | `sttak-common` | `ApiResponse`, `BaseException`, `BusinessException`, `ErrorType` |

특이점: **도메인 계층과 영속 어댑터를 같은 모듈(`sttak-domain`)에 둔다.**
영속 구현은 `*.persistence` 서브패키지에 **package-private**으로 캡슐화하여
모듈 외부에서 JPA 엔티티/리포지토리에 직접 접근할 수 없게 막는다.
(`UserJpaEntity`, `UserJpaRepository`, `UserRepositoryAdapter` 모두 `class`/`interface` 접근자 없음 = 패키지 전용)

---

## 3-1.4 패키지 구조 상세

### 3-1.4.1 sttak-domain

```
com.sttak.sttakdomain
├── config/
│   └── JpaConfig                  @EntityScan/@EnableJpaRepositories/@EnableJpaAuditing
└── user/                          ◄── Bounded Context: User
    ├── User                       도메인 모델 — Aggregate Root (정적 팩토리 create/restore)
    ├── Watchlist                  도메인 모델 — 1차 컬렉션 (불변식 보호: MAX_SIZE=5)
    ├── WatchlistItem              도메인 모델 — Aggregate 내부 Entity (식별자 보유, JPA @Entity 아님)
    ├── Name                       도메인 모델 — Value Object
    ├── UserRepository             Port (도메인이 정의)
    ├── exception/
    │   └── UserException          도메인 예외 enum (BaseException 구현)
    ├── persistence/               Adapter (package-private)
    │   ├── UserJpaEntity / WatchlistItemJpaEntity
    │   ├── UserJpaRepository / WatchlistItemJpaRepository
    │   ├── UserRepositoryAdapter        Command-side Port 구현
    │   ├── UserQueryRepositoryAdapter   Query-side Port 구현
    │   └── WatchlistItemQueryRepositoryAdapter
    └── query/                     CQRS Read 모델 (Port + View DTO)
        ├── UserQueryRepository / UserView
        └── WatchlistItemQueryRepository / WatchlistItemView
```

> 위 구조는 “신규 Bounded Context를 추가할 때 어떻게 만들어야 하는가”의 **레퍼런스 템플릿**이다.
> 실제 도메인(`Trade`, `Quiz`, `News`, `Retrospective` 등)도 같은 패턴을 따른다.

### 3-1.4.2 sttak-api

```
com.sttak.sttakapi
├── SttakApiApplication
├── exception/
│   └── GlobalExceptionHandler     BusinessException → HTTP 매핑
└── user/
    ├── controller/                Presentation (REST)
    ├── dto/                       Request / Response (HTTP 모델)
    ├── command/                   Application 입력 (record)
    ├── result/                    Application 출력 (record)
    └── service/                   Application Service (@Transactional)
```

### 3-1.4.3 sttak-common

```
com.sttak.sttakcommon
├── response/
│   └── ApiResponse<T>             success / error / inputError 팩토리
└── exception/
    ├── BaseException              interface { getErrorType(), getMessage() }
    ├── BusinessException          RuntimeException + BaseException 캡슐화
    └── ErrorType                  VALIDATION / UNAUTHORIZED / FORBIDDEN / NOT_FOUND / CONFLICT / INTERNAL
```

### 3-1.4.4 sttak-external

```
com.sttak.sttakexternal
└── <context>/
    ├── ClaudeNewsSummarizationAdapter      implements NewsSummarizationPort
    ├── DataGoKrStockPriceAdapter           implements StockQuotePort         (일봉, ADR-010)
    ├── DataGoKrKrxListingAdapter           implements StockListingPort       (종목 마스터, ADR-010)
    ├── ClaudeChatStreamAdapter             implements ChatStreamPort
    ├── DartDisclosureAdapter               implements DisclosurePort
    └── ...
```

외부 SDK/HTTP 응답 DTO는 **이 모듈 안에서만** 보인다. 도메인에는 Port 시그니처에 정의된 도메인 타입만 노출한다.

---

## 3-1.5 DDD 빌딩 블록 적용 규약

### 3-1.5.1 Aggregate / Entity / Value Object

- `<context>/` 바로 아래 클래스는 **전부 순수 도메인 모델**이다.
  JPA 어노테이션이 붙은 영속 객체(JPA `@Entity`)는 `<context>/persistence/`에만 있고 이름도 `*JpaEntity`로 구분한다 → **DDD Entity ≠ JPA Entity**.
- DDD 분류 (`User` 예시):
  - `User` — Aggregate Root
  - `WatchlistItem` — Aggregate 내부 Entity (자체 식별자 보유, `User`를 통해서만 변경됨)
  - `Watchlist` — 1차 컬렉션 (불변식 `MAX_SIZE=5` 보호)
  - `Name` — Value Object
- 도메인 객체는 **public 생성자 금지**. 정적 팩토리(`create` / `restore`)만 노출.
  - `create(...)`: 새로 만들 때 → 불변식 검증
  - `restore(...)`: 영속 어댑터가 DB → 도메인으로 복원할 때만 사용
- 컬렉션은 **노출하지 않는다**. `Watchlist` 같은 1차 컬렉션으로 감싸 불변식을 보호.
- 도메인 예외는 도메인이 소유한다 (`UserException` enum이 `BaseException` 구현).

### 3-1.5.2 Repository (Port & Adapter)

- 도메인이 인터페이스를 정의: `UserRepository`, `UserQueryRepository`, `WatchlistItemQueryRepository`.
- 어댑터는 `*.persistence` 패키지에 `package-private` 클래스로 구현(`@Repository`).
- 어댑터는 **JPA → 도메인** 매핑을 직접 책임진다. `toDomain()` / `newEntity()` / `update()`가 어댑터 안에서만 보인다.
- Aggregate 저장은 어댑터가 **묶음 일관성**까지 보장:
  `UserRepositoryAdapter.save()` → 도메인 watchlist 리스트와 DB를 비교해 **upsert + 삭제**까지 한 트랜잭션 안에서 처리(`reconcileWatchlist`).

### 3-1.5.3 CQRS (Command / Query 분리)

| 흐름 | Port | View 모델 | 트랜잭션 |
| --- | --- | --- | --- |
| **쓰기** | `UserRepository` | 도메인 객체 그대로 | `@Transactional` |
| **읽기** | `UserQueryRepository` / `WatchlistItemQueryRepository` | `UserView`, `WatchlistItemView` (record) | `@Transactional(readOnly = true)` |

- Command-side는 도메인을 통과하지만, Query-side는 도메인 객체를 거치지 않고 **읽기 전용 View record**를 바로 만든다 → 도메인 불변식 검증 부담 없이 조회 최적화 가능.
- `UserResult.from(...)`은 두 입력(도메인 `User`, View `UserView`)을 모두 받도록 **오버로드**되어 있어 Service 레이어 위쪽은 두 흐름을 동일하게 다룬다.
- **두 Port가 같은 DB를 본다.** 별도 read DB는 두지 않는다 (필요해지면 그 시점에 분리).

---

## 3-1.6 요청 흐름 (Application Service 경계)

### 3-1.6.1 등록 (Command)

```
HTTP POST /api/v1/users
   │
   ▼
UserController                                  (Presentation)
  └── request.toCommand() → UserCreateCommand
   │
   ▼
UserService.register(UserCreateCommand)         (Application, @Transactional)
  ├── User.create(...)                          (Domain — 불변식 검증)
  └── userRepository.save(user)                 (Port)
   │
   ▼
UserRepositoryAdapter.save(...)                 (Adapter — package-private)
  ├── UserJpaRepository.save(userEntity)
  └── reconcileWatchlist(...)                   (관심 종목 upsert/삭제 일관성)
   │
   ▼
UserJpaEntity → User.restore(...)               (도메인 복원)
   │
   ▼
UserResult.from(user)                           (Application 출력)
   │
   ▼
UserResponse.from(result) → ApiResponse.success (Presentation 직렬화)
```

### 3-1.6.2 조회 (Query)

```
HTTP GET /api/v1/users/{id}
   │
   ▼
UserService.findById(id)                            (@Transactional(readOnly = true))
  └── userQueryRepository.findById(id)              (Query Port)
   │
   ▼
UserQueryRepositoryAdapter
  └── JPA 엔티티 → UserView (record) 직접 매핑       (도메인 객체 우회)
   │
   ▼
UserResult.from(userView)                            (오버로드된 from)
```

Command-side와 Query-side가 **같은 DB**를 보지만 **모델/포트가 분리**되어 있다.

---

## 3-1.7 트랜잭션 정책

| 위치 | 정책 |
| --- | --- |
| Application Service의 변경 메서드 | `@Transactional` |
| Application Service의 조회 메서드 | `@Transactional(readOnly = true)` |
| Repository Adapter | 트랜잭션 시작/종료 없음. Service가 연 트랜잭션에 참여. |
| Controller | 트랜잭션 어노테이션 금지. |

→ **트랜잭션 경계는 Application Service**가 유일하게 소유한다.

---

## 3-1.8 도메인 이벤트 (Application Event)

부수 효과는 Spring `ApplicationEventPublisher`를 통해 도메인 이벤트로 발행한다.

```
TradeService.execute()
   ├── trade = Trade.create(...)
   ├── tradeRepository.save(trade)
   └── publisher.publishEvent(new TradeExecutedEvent(trade.id()))
                                           │
                                           ▼
                       (별도 컴포넌트가 구독 — 트랜잭션 외부)
                                           │
                                           ▼
                       RetrospectiveTriggerListener
                          └── retrospectiveService.generate(tradeId)
```

규약:

- 도메인 이벤트는 **현재 트랜잭션 커밋 후** 처리한다 (`@TransactionalEventListener(phase = AFTER_COMMIT)`).
- 비동기로 처리하고 싶다면 `@Async` 또는 메시지 큐로 위임한다 (MVP는 단일 JVM 내부 이벤트).
- **이벤트 실패가 매매 자체를 롤백하지 않는다.** 실패는 별도 재시도 큐에 적재한다.
- 본 메커니즘은 향후 `EX-N*` (푸시 알림) 확장 시 그대로 재사용한다.

---

## 3-1.9 외부 시스템 어댑터 (sttak-external)

LLM / 시세 / 뉴스 등 외부 시스템 호출은 **도메인 Port의 구현체**로 `sttak-external`에 들어온다.

```
sttak-domain                          sttak-external
─────────────────                     ───────────────────────────────────
NewsSummarizationPort   ◀── 구현 ─── ClaudeNewsSummarizationAdapter
StockQuotePort          ◀── 구현 ─── DataGoKrStockPriceAdapter   (일봉, ADR-010)
StockListingPort        ◀── 구현 ─── DataGoKrKrxListingAdapter   (종목 마스터, ADR-010)
ChatStreamPort          ◀── 구현 ─── ClaudeChatStreamAdapter
EmbeddingPort           ◀── 구현 ─── OpenAIEmbeddingAdapter
DisclosurePort          ◀── 구현 ─── DartDisclosureAdapter
NewsCollectionPort      ◀── 구현 ─── NewsApiCollectionAdapter
```

원칙:

- 외부 SDK/HTTP 응답 DTO는 **`sttak-external` 안에서만** 보인다. 도메인에는 Port 시그니처의 도메인 타입만 노출한다.
- 어댑터는 **재시도 / 타임아웃 / 서킷브레이커**를 자체적으로 적용한다 (Resilience4j 권장).
- LLM 토큰 사용량·비용 등은 어댑터가 메트릭으로 발행한다 *(NFR-O3)*.
- **API 키는 어댑터/환경변수에만** 존재하고, 도메인/응답/로그에 노출되지 않는다 *(CON-S1)*.

---

## 3-1.10 두 갈래 처리 구조 (Real-time vs Pipeline)

### (A) 실시간 경로 — `sttak-api`

- 트리거: HTTP 요청.
- **사용자 응답 시간이 SLA**. 외부 의존성(NewsAPI/DART/LLM) 호출이 사용자 응답을 막아서는 안 된다 *(NFR-R1)*.
- 외부 데이터가 필요한 경우, **이미 가공·저장된 결과**만 읽는다.
  - 100자 요약 → Postgres
  - 임베딩 검색 → Vector DB
  - 인과 노드 → Graph 저장소
- 챗봇/회고처럼 즉시 LLM 호출이 필요한 경로는 **SSE 스트리밍**으로 사용자 인지 지연을 줄인다.

### (B) 비동기 경로 — `sttak-batch`

- 트리거: 스케줄 (Cron / `@Scheduled`).
- 책임:
  1. **뉴스 / 공시 / 시세 수집** — NewsAPI, DART, **data.go.kr 금융위 오픈API(일봉 OHLCV·종목 마스터, [ADR-010](../decisions/ADR-010-market-data-source-datagokr.md))**.
     - 시세 일봉 수집분은 모의투자 체결의 정산 트리거가 된다([ADR-011](../decisions/ADR-011-mock-trade-execution-model.md)).
  2. **정제 / 종목 태깅** — 종목 코드 매칭, 클러스터링.
  3. **100자 요약 (LLM)** — 클러스터 대표 1건만 호출.
  4. **임베딩 적재** — Vector DB.
  5. **인과 그래프 추출** — Graph 저장소.
  6. **퀴즈 후보 풀 생성 (LLM)** — 6시간 주기, AI 랜덤 생성 *(FR-D4, FR-H5)*.
- 각 단계는 **재시도 / 스킵 정책**을 갖는다 *(FR-H2)*. 한 사용자 / 한 종목의 실패가 전체 배치를 막지 않는다.
- 사용자 트래픽과 **별도 워커**에서 실행 *(NFR-X2)*.

### 두 경로의 인터페이스

같은 도메인 모듈을 공유하므로 두 경로의 결과물은 **동일한 Port + 동일한 저장소**에 모인다.
배치가 만든 결과를 API가 읽는다.

---

## 3-1.11 인증 / 인가

- **인증(Authentication)** — 카카오/구글/애플 소셜 토큰을 백엔드에서 검증한 뒤 **자체 JWT**(access + refresh) 발급 *(FR-A1, NFR-S2)*.
- **인가(Authorization)** — JWT의 `sub`=사용자 ID, `roles`=권한. `ROLE_USER` / `ROLE_ADMIN`.
- 세션·쿠키 미사용. `sttak-api`는 stateless *(CON-T5, NFR-X1)*.
- `sttak-admin`은 동일한 JWT 메커니즘을 사용하되 `ROLE_ADMIN`만 허용한다.

---

## 3-1.12 데이터 저장소 토폴로지

| 저장소 | 역할 | MVP 사용 모듈 |
| --- | --- | --- |
| **PostgreSQL 17** | 정형 데이터 (사용자, 종목, 거래, 포트폴리오, 뉴스 요약, 퀴즈, 회고) | api / admin / batch |
| **Vector DB** | 뉴스/공시 임베딩 (예: pgvector, Pinecone — 결정 보류) | api(read), batch(write) |
| **Graph 저장소** | 인과 그래프 (Phase 2) | api(read), batch(write) |
| **캐시 (Redis)** | LLM 응답·요약 캐시, 세션 X | api, batch (선택) |

원칙:

- **단일 Postgres 인스턴스**로 시작, 필요 시점에 read replica / 샤딩 검토.
- Vector DB는 MVP에서 **Postgres + pgvector**로 시작해 운영 부담을 줄인다.
- Graph 저장소는 Phase 2까지는 인과 그래프 노드를 Postgres 일반 테이블로 모델링한다.

---

## 3-1.13 LLM 호출 토폴로지

```
sttak-api / sttak-batch ──▶ sttak-external (Port 구현)
                                   │
                                   ├── Claude Haiku  (분류 / 퀴즈 생성 / 매매 회고)
                                   ├── Claude Sonnet (챗봇)
                                   └── OpenAI Embeddings (임베딩)
```

전략:

- **모델 선택은 어댑터/설정**에서 결정한다. 도메인은 “요약을 만든다”/“챗봇 응답을 받는다” 수준만 안다.
- 매매 회고는 원안(Sonnet)에서 **Haiku 로 하향** — 신호 해설과 같은 "신규 이벤트당 1회 생성 + 조회 0회" 비용
  구조로 묶는 SCRUM-68 결정. 품질 미달이 관찰되면 상향하되 ADR 로 기록한다.
- 동일 입력에 대한 응답은 **캐시**한다 (특히 요약 / 임베딩).
- **가드레일(`FR-G3`)** 은 어댑터의 응답 후처리 단계에 위치한다. 추천성 표현 차단 + 출처 부착 + AI 생성 표시.
- 토큰 사용량·지연·비용 추정치를 메트릭으로 발행한다 *(NFR-O3)*.

---

## 3-1.14 예외 / 응답 체계 (요약)

### 3-1.14.1 예외 계층

```
BaseException (interface)                    ← sttak-common
   ▲
   │ implements
   │
UserException (enum)                         ← sttak-domain
   │   - USER_NOT_FOUND          (NOT_FOUND)
   │   - WATCHLIST_ITEM_NOT_FOUND (NOT_FOUND)
   │   - WATCHLIST_LIMIT_EXCEEDED (CONFLICT)
   │
   │ wrapped by
   ▼
BusinessException (RuntimeException)         ← sttak-common
```

- 도메인은 자기 예외 enum을 소유, 공통 모듈은 enum을 감쌀 `BusinessException`만 제공.
- 새 컨텍스트가 추가되면 `XxxException implements BaseException` enum을 해당 도메인 패키지에 추가하면 끝.

### 3-1.14.2 HTTP 매핑

`GlobalExceptionHandler` (`sttak-api/exception`)가 단일 진입점:

| ErrorType | HTTP |
| --- | --- |
| VALIDATION | 400 |
| UNAUTHORIZED | 401 |
| FORBIDDEN | 403 |
| NOT_FOUND | 404 |
| CONFLICT | 409 |
| INTERNAL | 500 |

- `BusinessException` → `ApiResponse.error(...)`
- `MethodArgumentNotValidException` → `ApiResponse.inputError(...)`

응답 포맷은 **`ApiResponse<T> { message, content }`** 하나로 통일.

> 상세 매핑/시나리오는 `5-2-error-handling.md` 참조.

---

## 3-1.15 패키지 가시성 규약

- `*.persistence` 하위의 모든 클래스/인터페이스/메서드는 **package-private**.
  - JPA 엔티티를 다른 모듈/패키지가 직접 알 수 없다.
  - 도메인 외부는 오직 `UserRepository` 같은 **Port 인터페이스**로만 영속을 다룬다.
- 도메인 모델의 생성자도 `private`. 외부는 정적 팩토리만 사용.
- DTO (`request/response`)는 HTTP 모듈 내부에서만 산다 → 도메인이 HTTP에 의존하지 않도록.
- `sttak-external`의 SDK/벤더 DTO도 어댑터 패키지 안에서만 보인다.

---

## 3-1.16 새 도메인을 추가할 때 (체크리스트)

새 Bounded Context `xxx`를 추가한다고 가정:

1. `sttak-domain/.../xxx/`에
   - `Xxx` (Aggregate Root, 정적 팩토리)
   - 필요한 VO / Entity
   - `XxxRepository` (Port)
   - `exception/XxxException` (enum implements `BaseException`)
   - `persistence/` 패키지에 `XxxJpaEntity`, `XxxJpaRepository`, `XxxRepositoryAdapter` (모두 package-private)
   - `query/XxxQueryRepository`, `XxxView` + `persistence/XxxQueryRepositoryAdapter`
2. `sttak-apps/sttak-api/.../xxx/`에
   - `controller/`, `dto/`, `command/`, `result/`, `service/`
3. 외부 시스템 호출이 필요하면 `sttak-domain/.../xxx/`에 **Port 인터페이스**를 정의하고,
   `sttak-external/.../xxx/`에 구현 어댑터를 추가한다.
4. 예외만 늘리면 되는 경우 — 도메인 패키지의 enum에 케이스 추가. `GlobalExceptionHandler`는 수정 불필요.

---

## 3-1.17 비기능 대응 매핑

| NFR | 대응 메커니즘 |
| --- | --- |
| **NFR-P1\~P5** (지연) | 두 갈래 처리, LLM 응답 캐시, SSE 스트리밍 |
| **NFR-R1** (장애 격리) | 어댑터 단의 타임아웃·서킷브레이커, 캐시 폴백 |
| **NFR-R3** (멱등성) | Application Service에서 도메인 ID 또는 Idempotency-Key 헤더 기반 처리 |
| **NFR-S1\~S5** (보안) | 키는 어댑터/환경변수만, JWT, HTTPS, PII 마스킹 |
| **NFR-D1\~D3** (일관성) | 트랜잭션 경계 = Service, 응답에 `fetchedAt` 명시 |
| **NFR-O1\~O3** (관측성) | trace ID 미들웨어, Micrometer 메트릭, LLM 어댑터 메트릭 |
| **NFR-X1\~X2** (확장성) | stateless API, 배치/API 분리, 워커 별도 |

---

## 3-1.18 의도적으로 남겨둔 결정 (TBD / TODO)

- `sttak-admin`, `sttak-batch`: 부트 클래스만 있고 사용 사례 미구현. 도메인 추가에 따라 점진적 채움.
- `sttak-external`: 의존성 선언만 있고 어댑터 없음. 외부 호출이 생기면 여기에 도메인 Port의 구현체로 추가.
- `sttak-api/.../<context>/exception`: 디렉터리만 존재 가능. 컨텍스트 고유 예외는 도메인으로 올리는 게 원칙이므로 보통 비워 둔다.
- Vector DB 구체 선택 (pgvector vs Pinecone) — 비용/운영 부담 평가 후.
- Graph 저장소 도입 시점 — Phase 2.
- 캐시 도입 시점 — LLM 호출 비용/지연 측정 후.
- 서킷브레이커 라이브러리 — Resilience4j 권장.
- 멱등 키 헤더 도입 시점 — 결제/충전 도입 시 (MVP는 가상 자본금만, 우선순위 낮음).

---

## 3-1.19 후속 문서

| 문서 | 내용 |
| --- | --- |
| `3-2-directory.md` | 모듈/패키지 트리와 명명 규약 |
| `3-3-contract.md` | API 시그니처 / 스키마 |
| `5-1-test-case.md` | 본 구조의 검증 전략 (루트 `TESTING.md` 함께 참조) |
| `5-2-error-handling.md` | 예외 → HTTP 상세 매핑·시나리오 |
| `6-observability.md` | 트레이싱·로깅·메트릭 |
| `7-deployment.md` | 배포 토폴로지 |
