# 3-2. Directory & Module Structure — sTTak 백엔드

본 문서는 sTTak 백엔드의 **디렉터리/패키지/모듈 구조와 명명 규약**을 정리한다.
시스템 아키텍처(왜 이렇게 자르는가)는 `3-1-server-architecture.md`를 참조.

핵심 패턴은 다음과 같다 — 새 디렉터리를 만들기 전에 본 문서의 표를 확인한다.

- **Gradle 멀티모듈** (라이브러리 3 + 실행 앱 3 = 6 모듈)
- **DDD + Hexagonal + CQRS** — 도메인/Port/Adapter 분리
- **Bounded Context 단위 패키지** — `<context>/`, `<context>/persistence/`, `<context>/query/`, `<context>/exception/`
- **`*.persistence`는 package-private** — 외부 노출은 Port만

---

## 3-2.1 루트 구조

```
sttak-backend-demo/
├── build.gradle                root 빌드 (apply false 전략으로 버전만 고정)
├── settings.gradle             include 'sttak-common', ...
├── gradlew / gradle/           Gradle Wrapper
├── README.md
├── TESTING.md                  테스트 가이드 (5-1과 함께 참조)
├── docs/
│   └── specs/                  본 스펙 디렉터리
│       ├── 1-background.md
│       ├── 2-1-user-story.md
│       ├── 2-2-requirements.md
│       ├── 3-1-server-architecture.md
│       ├── 3-2-directory.md     ◄── 이 문서
│       ├── 3-3-contract.md
│       ├── 3-4-erd.md
│       ├── 4-git-workflow.md
│       ├── 5-1-test-case.md
│       ├── 5-2-error-handling.md
│       ├── 6-observability.md
│       └── 7-deployment.md
├── sttak-apps/                 컨테이너 (소스 없음)
│   ├── sttak-api/              사용자용 REST API (bootJar)
│   ├── sttak-admin/            관리자 콘솔 백엔드 (bootJar)
│   └── sttak-batch/            데이터 파이프라인/스케줄러 (bootJar)
├── sttak-common/               응답·예외 등 횡단 관심사 (plain jar)
├── sttak-domain/               도메인 모델 + JPA 영속 어댑터 (plain jar)
└── sttak-external/             외부 시스템 어댑터 (plain jar)
```

규약:

- 모든 Spring Boot 실행 모듈은 `sttak-apps/` 아래에만 둔다.
- 라이브러리 모듈은 루트 직속에 둔다.
- `docs/specs/` 외 위치에 새 문서를 두려면 본 문서에 한 줄 추가한다.

---

## 3-2.2 모듈별 책임 요약

| 모듈 | 종류 | 책임 | 의존 |
| --- | --- | --- | --- |
| `sttak-common` | 라이브러리 | 응답 포맷, 공통 예외, 횡단 관심사 | (없음) |
| `sttak-domain` | 라이브러리 | 도메인 모델, Port, JPA 영속 어댑터 (package-private) | `sttak-common` |
| `sttak-external` | 라이브러리 | 외부 시스템(HTTP/SDK) 어댑터 = Port 구현 | `sttak-common`, `sttak-domain` |
| `sttak-apps/sttak-api` | 실행 (bootJar) | 사용자 REST API + SSE | `sttak-common`, `sttak-domain`, `sttak-external` |
| `sttak-apps/sttak-admin` | 실행 (bootJar) | 관리자 콘솔 백엔드 | `sttak-common`, `sttak-domain`, `sttak-external` |
| `sttak-apps/sttak-batch` | 실행 (bootJar) | 데이터 파이프라인 / 스케줄러 | `sttak-common`, `sttak-domain`, `sttak-external` |

빌드 규약:

- 실행 모듈 → `bootJar` ON, plain `jar` OFF.
- 라이브러리 모듈 → `bootJar` OFF, plain `jar` ON.
- `sttak-apps` 컨테이너는 플러그인 적용 대상에서 제외 (자바 소스 없음).

---

## 3-2.3 Bounded Context 패키지 템플릿

모든 도메인은 다음 패턴으로 한다. `<context>`는 `user`, `trade`, `quiz`, `news`, `retrospective`, `ranking`, `portfolio`, `chat` 등이 될 수 있다.

### 3-2.3.1 `sttak-domain` — Bounded Context 한 개의 트리

```
com.sttak.sttakdomain.<context>/
├── <Context>                          Aggregate Root            (정적 팩토리 create/restore)
├── <SubEntity>                        Aggregate 내부 Entity     (식별자 보유, JPA @Entity 아님)
├── <FirstClassCollection>             1차 컬렉션                (예: Watchlist — MAX_SIZE 보호)
├── <ValueObject>                      Value Object              (불변, 식별자 없음)
├── <Context>Repository                Port (Command-side)
├── exception/
│   └── <Context>Exception             도메인 예외 enum implements BaseException
├── persistence/                       Adapter — package-private
│   ├── <Context>JpaEntity
│   ├── <SubEntity>JpaEntity
│   ├── <Context>JpaRepository
│   ├── <SubEntity>JpaRepository
│   ├── <Context>RepositoryAdapter           Command-side Port 구현
│   └── <Context>QueryRepositoryAdapter      Query-side Port 구현
└── query/
    ├── <Context>QueryRepository       Port (Query-side)
    └── <Context>View                  Read-only record DTO
```

### 3-2.3.2 `sttak-apps/sttak-api` — 같은 컨텍스트의 API 트리

```
com.sttak.sttakapi.<context>/
├── controller/<Context>Controller     Presentation (REST)
├── dto/
│   ├── <Context>CreateRequest         HTTP 입력 DTO
│   └── <Context>Response              HTTP 출력 DTO
├── command/<Context>CreateCommand     Application 입력 (record)
├── result/<Context>Result             Application 출력 (record)
└── service/<Context>Service           Application Service (@Transactional)
```

### 3-2.3.3 `sttak-external` — 같은 컨텍스트의 어댑터 트리 (필요 시)

```
com.sttak.sttakexternal.<context>/
├── <Vendor><Capability>Adapter        implements <Capability>Port (sttak-domain)
└── dto/                               벤더 SDK/HTTP DTO (package-private)
```

---

## 3-2.4 명명 규약 (Naming)

| 종류 | 규약 | 예시 |
| --- | --- | --- |
| Bounded Context 디렉터리 | 단수 명사, 소문자 | `user`, `trade`, `quiz`, `news` |
| Aggregate Root 클래스 | 컨텍스트 명사 (대문자 시작) | `User`, `Trade`, `Quiz` |
| 1차 컬렉션 | 복수형 또는 도메인 용어 | `Watchlist`, `TradeHistory` |
| Value Object | 의미 명사 | `Name`, `Money`, `TickerCode` |
| 도메인 예외 enum | `<Context>Exception` | `UserException`, `TradeException` |
| Repository Port (쓰기) | `<Context>Repository` | `UserRepository` |
| Repository Port (읽기) | `<Context>QueryRepository` | `UserQueryRepository` |
| Read 모델 record | `<Context>View` | `UserView` |
| JPA 엔티티 | `<Context>JpaEntity` (접미사 필수) | `UserJpaEntity` |
| JPA 리포지토리 | `<Context>JpaRepository` | `UserJpaRepository` |
| Port 구현 어댑터 | `<Context>RepositoryAdapter`, `<Context>QueryRepositoryAdapter` | `UserRepositoryAdapter` |
| 외부 시스템 어댑터 | `<Vendor><Capability>Adapter` | `ClaudeChatStreamAdapter`, `DataGoKrStockPriceAdapter` |
| Application Service | `<Context>Service` | `UserService`, `TradeService` |
| Application 입력 record | `<Action>Command` | `UserCreateCommand`, `TradeExecuteCommand` |
| Application 출력 record | `<Context>Result` | `UserResult`, `TradeResult` |
| Controller | `<Context>Controller` | `UserController` |
| HTTP 입력 DTO | `<Action><Object>Request` (동사 + 목적어 + `Request`) | `CreateMemberRequest`, `SubmitQuizAnswerRequest` |
| HTTP 출력 DTO | `<Action><Object>Response` (동사 + 목적어 + `Response`) | `CreateMemberResponse`, `GetRankingResponse` |
| 도메인 이벤트 | `<Context><PastAction>Event` | `TradeExecutedEvent` |

> **DDD Entity ≠ JPA Entity.** 영속 객체는 항상 `*JpaEntity` 접미사로 구분한다. 도메인 모델은 접미사 없음.

---

## 3-2.4.1 메서드 네이밍 컨벤션

- 컨트롤러, 서비스
    - 생성할 때 : `create`
    - 조회할 때 : 컨트롤러에서는 `get`, 서비스는 `find`
        - 단건 조회 : `get` + 목적어 + `By` + 필드
        - 여러 건 조회 : `get` + 목적어s + `By` + 필드
        - 전체 조회 : `getAll` + 목적어
    - 수정할 때 : `update`
    - 삭제할 때 : `delete`
    - 조회/생성/수정/삭제에 해당하지 않는 도메인 행위는 그 행위를 표현하는 동사를 그대로 쓴다. 컨트롤러/서비스 모두 같은 동사로 통일하며, 조회 패턴과 동일하게 `동사 + 목적어 + By + 필드` 형태를 따른다. 예: `submitQuizAnswerByUserId`, `cancelOrderById`, `publishPostById`, `refundPaymentById`.
    - 엔티티 변경감지 관련 메소드는 `set` 말고 행위 명시. 예: `updateName(String name)`
- DTO 클래스 이름도 메서드와 같은 `동사 + 목적어` 구조를 따른다: HTTP 입력은 `<Action><Object>Request`, HTTP 출력은 `<Action><Object>Response`. 예: `CreateMemberRequest`, `SubmitQuizAnswerRequest`, `GetRankingResponse`.
- DTO/Request에서도 `@Setter`를 쓰지 않는다.
- 메서드 본문의 첫 줄 앞에는 한 줄을 띈다 (서명의 `{` 와 첫 코드 라인 사이에 빈 줄 한 개). 비어 있는 메서드/한 줄짜리 람다는 해당 없음. 예:
    ```java
    @GetMapping("/last")
    public ApiResponse<GetLastQuizResultResponse> getLastQuizResultByUserId(@CurrentUserId UUID userId) {

        return quizService.findLastQuizResultByUserId(userId)
                .map(GetLastQuizResultResponse::from)
                .map(ApiResponse::success)
                .orElseGet(() -> ApiResponse.success(null));
    }
    ```

---

## 3-2.4.2 Lombok 사용 규약

필드 접근자·생성자를 손으로 쓰지 않고 Lombok 으로 대체할 수 있는 경우엔 **항상 Lombok 을 쓴다.**
목적은 코드 반복 제거와 리뷰 시야 확보다. 아래 규약은 위 §3-2.4.1 의 네이밍/`@Setter` 금지 조항과 짝을 이룬다.

### 3-2.4.2.1 기본 원칙

1. **필드 → getter 는 클래스 레벨 `@Getter` 로만 노출한다.**
    - 클래스의 모든 필드에 게터가 필요하면 클래스 레벨 `@Getter`.
    - **일부 필드만 노출해야 할 때만 손으로 쓴 게터를 허용한다.** 필드 레벨 `@Getter` 는 쓰지 않는다 (선택적 노출은 손 게터가 더 명확).
    - 그 외의 경우 `public T getFoo() { return foo; }` 형태의 손 게터는 새로 쓰지 않는다.
2. **`@Setter` 는 어디에도 쓰지 않는다.** 도메인/JPA/DTO/Request 어느 쪽이든 상태 변경은 의미 있는 동사 메서드로만 (§3-2.4.1: `updateName(String name)`).
3. **정적 팩토리 / 도메인 생성자를 대체하는 목적으로 `@AllArgsConstructor(access = PUBLIC)`, `@Builder` 를 쓰지 않는다.**
    - 도메인 객체(Aggregate/VO)는 여전히 `private` 생성자 + `create` / `restore` 정적 팩토리 (`§3-2.5`, `§3-2.10`).
    - `@Builder` 는 도메인 객체 안티패턴 (`§3-2.10` 표).
4. **enum 파라미터의 반복 대입은 `@RequiredArgsConstructor` 로 없앤다.** 도메인 예외 enum(`<Context>Exception`) 처럼 `final` 필드만 담는 enum 은 아래 §3-2.4.2.3 참조.
5. **JPA `*JpaEntity` / `@Embeddable` 은 클래스 레벨 `@Getter` 를 기본으로 한다** (아래 §3-2.4.2.2).
    - 실제 인자 생성자(package-private)는 손으로 정의한다 — Lombok 생성자 애노테이션은 필드 순서·널 처리 요구사항과 자주 충돌하므로 사용하지 않는다.

### 3-2.4.2.2 JPA `*JpaEntity` 표준 형태

```java
@Entity
@Table(name = "…")
@EntityListeners(AuditingEntityListener.class)   // 필요 시
@Getter                                          // ◄ 모든 필드가 노출 대상이면 클래스 레벨
class FooJpaEntity {                             // ◄ persistence 패키지는 package-private

    @Id ...
    private Long id;

    // 필드 …

    protected FooJpaEntity() {                   // ◄ JPA 요구 — 손으로 두거나 @NoArgsConstructor(access = PROTECTED)
    }

    FooJpaEntity(...) {                          // ◄ 상태 초기화 생성자 — 손으로 정의
        ...
    }

    void updateXxx(...) {                        // ◄ 변경감지 — set 대신 동사
        ...
    }
}
```

- `*JpaEntity` 는 `persistence` 패키지 안에서만 쓰이므로(§3-2.5) `@Getter` 가 만드는 `public` 게터라도 외부로 새지 않는다.
- 일부 필드만 노출해야 한다면 클래스 레벨 `@Getter` 를 붙이지 말고 필요한 필드에 대해 손으로 게터를 정의한다. 예: 비밀번호 필드(`password`)를 게터 없이 유지하고 나머지만 노출해야 한다면 클래스 레벨 `@Getter` 를 빼고 필요한 게터만 손으로 쓴다.
- `Persistable<T>` 를 구현하는 엔티티도 클래스 레벨 `@Getter` 로 충분하다. 필드 `boolean isNew` 는 Lombok 이 `isNew()` 로 매핑해 `Persistable#isNew()` 를 그대로 만족한다. `@Override` 를 남기고 싶으면 손게터를 명시 정의해도 되지만, 기본은 `@Getter` 위임.

### 3-2.4.2.3 도메인 예외 enum 표준 형태

`<Context>Exception implements BaseException` 은 다음 하나의 형태로 통일한다.

```java
@Getter
@RequiredArgsConstructor
public enum FooException implements BaseException {

    FOO_NOT_FOUND(ErrorType.NOT_FOUND, "…"),
    …;

    private final ErrorType errorType;
    private final String message;
}
```

- `BaseException` 은 `getErrorType()`·`getMessage()` 를 요구한다. 필드명 `errorType`·`message` 를 그대로 두면 `@Getter` 가 시그니처를 정확히 매치한다.
- 생성자·손게터를 함께 유지하지 않는다 — Lombok 애노테이션 두 개로 충분.

### 3-2.4.2.4 어디에 쓰고, 어디에 쓰지 않는가

| 대상 | 게터 | 생성자 애노테이션 |
| --- | --- | --- |
| `*JpaEntity`, `@Embeddable` (전 필드 노출) | 클래스 레벨 `@Getter` | 실제 생성자는 손으로. 필요 시 `@NoArgsConstructor(access = PROTECTED)` 만. |
| `*JpaEntity` 중 일부 필드만 노출 | 클래스 레벨 `@Getter` 없이 필요한 게터만 손으로 | 위와 동일 |
| 도메인 예외 enum (`<Context>Exception`) | `@Getter` | `@RequiredArgsConstructor` |
| 도메인 Aggregate/VO/1차 컬렉션 | 전 필드면 `@Getter`, 부분 노출이면 손 게터 | 애노테이션 금지 — `private` 생성자 + 정적 팩토리 (§3-2.5) |
| Application `<Action>Command` / `<Context>Result` | (record 이므로 불필요) | (record) |
| HTTP `<Action><Object>Request` / `<Action><Object>Response` | (record 우선) — 클래스라면 `@Getter` | `@AllArgsConstructor` 대신 정적 팩토리(`from(...)`) |
| Spring `@ConfigurationProperties` / Adapter 내부 상태 | 필요 시 `@Getter` | `@RequiredArgsConstructor` 로 의존성 주입 |

**금지 조합**

- `@Setter` (모든 곳)
- 도메인 객체에 `@Builder` (§3-2.10)
- 전 필드 노출이 필요한데도 게터를 손으로 다시 쓰기 (`public T getFoo() { return foo; }`)
- 필드 레벨 `@Getter` — 부분 노출이 필요하면 손 게터로 명시.
- `@Data`, `@Value` — 도메인/JPA/DTO 모두 필요 이상으로 노출되므로 쓰지 않는다.

### 3-2.4.2.5 리뷰 체크리스트

- [ ] 새로 추가한 클래스에 `public T getX() { return x; }` 형태의 손 게터가 있는가? → 전 필드 노출이면 클래스 레벨 `@Getter` 로 치환. 부분 노출이면 그대로 유지.
- [ ] enum 안에 반복적인 `this.foo = foo` 대입 생성자가 있는가? → `@RequiredArgsConstructor`.
- [ ] `*JpaEntity` 에 `@Getter` 가 빠져 있는가? → 전 필드 노출이면 추가. 아니라면 필요한 손 게터만 남긴다.
- [ ] `@Setter` / `@Data` / `@Builder`(도메인) / 필드 레벨 `@Getter` 가 들어왔는가? → 리뷰 거절.

---

## 3-2.5 패키지 가시성 규약

`*.persistence` 하위는 **전부 package-private**.

| 위치 | 접근자 |
| --- | --- |
| `com.sttak.sttakdomain.<context>.persistence.*` | (없음 = package-private) |
| `com.sttak.sttakdomain.<context>.<Aggregate>` | `public` |
| `com.sttak.sttakdomain.<context>.<Port>` | `public` |
| 도메인 객체 생성자 | `private` (정적 팩토리만 노출) |
| 도메인 객체 정적 팩토리 `create` / `restore` | `public` |
| `sttak-external/.../<context>.dto.*` | (없음 = package-private) |
| HTTP DTO (`sttak-api/.../<context>.dto.*`) | `public` (모듈 외부 비공개) |

위 규약을 깨려는 코드(예: 컨트롤러가 JPA 엔티티를 직접 import)는 **리뷰에서 거절한다.**

---

## 3-2.6 cross-cutting 모듈 (`sttak-common`) 구조

```
com.sttak.sttakcommon/
├── response/
│   └── ApiResponse<T>           success(content) / error(errorType, message) / inputError(...) 팩토리
└── exception/
    ├── BaseException            interface { ErrorType getErrorType(); String getMessage(); }
    ├── BusinessException        RuntimeException + BaseException 캡슐화
    └── ErrorType                enum: VALIDATION / UNAUTHORIZED / FORBIDDEN / NOT_FOUND / CONFLICT / INTERNAL
```

원칙:

- `sttak-common`은 **다른 모듈을 모른다.** 가장 안쪽 레이어.
- `ErrorType`에 새 항목을 추가하면 `GlobalExceptionHandler`의 HTTP 매핑도 함께 갱신해야 한다 → `5-2-error-handling.md`.
- 어떤 도메인도 `BusinessException`을 **직접 throw하지 않는다.** 도메인 enum을 throw하면 어댑터/Service 경계에서 `BusinessException`으로 감싸진다.

---

## 3-2.7 실행 모듈별 패키지 루트

각 실행 모듈의 패키지 루트와 `@SpringBootApplication`은 다음과 같다.

| 모듈 | 패키지 루트 | 진입점 |
| --- | --- | --- |
| `sttak-api` | `com.sttak.sttakapi` | `SttakApiApplication` |
| `sttak-admin` | `com.sttak.sttakadmin` | `SttakAdminApplication` |
| `sttak-batch` | `com.sttak.sttakbatch` | `SttakBatchApplication` |
| `sttak-domain` | `com.sttak.sttakdomain` | (없음, 라이브러리) |
| `sttak-common` | `com.sttak.sttakcommon` | (없음, 라이브러리) |
| `sttak-external` | `com.sttak.sttakexternal` | (없음, 라이브러리) |

JPA 스캔 / Repository 스캔은 `sttak-domain`의 `config/JpaConfig`에서
`@EntityScan("com.sttak.sttakdomain")`, `@EnableJpaRepositories("com.sttak.sttakdomain")`로 처리한다.

---

## 3-2.8 새 도메인 추가 체크리스트

새 Bounded Context `xxx`를 만들 때:

1. `sttak-domain/src/main/java/com/sttak/sttakdomain/xxx/`
   - `Xxx`, 필요한 VO/Entity, 1차 컬렉션
   - `XxxRepository`, `query/XxxQueryRepository`, `query/XxxView`
   - `exception/XxxException` (enum implements `BaseException`)
   - `persistence/` — `XxxJpaEntity`, `XxxJpaRepository`, `XxxRepositoryAdapter`, `XxxQueryRepositoryAdapter` *(모두 package-private)*

2. `sttak-apps/sttak-api/src/main/java/com/sttak/sttakapi/xxx/`
   - `controller/`, `dto/`, `command/`, `result/`, `service/`

3. 외부 시스템 호출이 필요하면:
   - `sttak-domain/.../xxx/`에 **Port 인터페이스** 정의 (`<Capability>Port`)
   - `sttak-external/.../xxx/`에 **구현 어댑터** 추가 (`<Vendor><Capability>Adapter`)

4. 테스트:
   - `sttak-domain/src/test/.../xxx/` — 도메인 단위 테스트
   - `sttak-domain/src/test/.../xxx/persistence/` — Repository 통합 테스트 (`@DataJpaTest` + Testcontainers Postgres)
   - `sttak-api/src/test/.../xxx/` — Web 슬라이스 + E2E 테스트
   - Fixture는 `src/test/java/.../xxx/support/XxxFixture` 정적 팩토리로 만든다 (`TESTING.md` §2 참조)

5. 예외만 늘리는 경우 — `XxxException` enum에 케이스 추가. `GlobalExceptionHandler`는 손대지 않는다.

---

## 3-2.9 빌드/실행 명령

```bash
# 전체 빌드 (테스트 + Jacoco 검증 + bootJar)
./gradlew build

# 특정 모듈만
./gradlew :sttak-apps:sttak-api:bootRun
./gradlew :sttak-apps:sttak-batch:bootRun

# 테스트만
./gradlew :sttak-domain:test
./gradlew test                # 모든 모듈

# Jacoco 리포트 위치
<module>/build/reports/jacoco/test/html/index.html
```

---

## 3-2.10 안 만드는 디렉터리 (Anti-patterns)

다음 디렉터리는 만들지 않는다. 이름이 보이면 리뷰에서 거절한다.

| 안티패턴 | 이유 |
| --- | --- |
| `util/`, `common/` (도메인 모듈 안) | 모듈 간 공유 유틸은 `sttak-common`. 도메인 안 “common”은 의도 불명. |
| `helper/`, `manager/` | 책임이 모호한 이름. 도메인 객체나 Service로 분해. |
| `Xxx.Builder` 별도 클래스 | 정적 팩토리 우선. Lombok `@Builder`는 도메인 객체에 쓰지 않는다. |
| `Xxx.Impl` 접미사 | 인터페이스 1:1 구현 표시. 도메인은 Port + Adapter 명명을 따른다. |
| `dto/` (도메인 모듈 안) | 도메인 모듈은 DTO를 갖지 않는다. View record는 `query/`에. |
| `service/` (도메인 모듈 안) | Application Service는 `sttak-api/.../service/`. 도메인 모듈은 도메인 서비스만 허용. |

---

## 3-2.11 후속 문서

| 문서 | 연결 |
| --- | --- |
| `3-1-server-architecture.md` | 본 구조의 의도 / 시스템 흐름 |
| `3-3-contract.md` | API 시그니처 |
| `5-1-test-case.md` | 테스트 디렉터리/패턴 |
