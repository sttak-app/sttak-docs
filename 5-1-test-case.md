# 5-1. Test Strategy — sTTak 백엔드

본 문서는 sTTak 백엔드의 **테스트 전략과 TDD 규약**을 정의한다.
실무 디테일(Fixture, InMemory 패턴, Testcontainers 운용 팁)은 루트 `TESTING.md`를 참조.

원칙:

1. **TDD 강제** — 프로덕션 코드는 **실패하는 테스트가 먼저 존재해야** 작성한다.
2. **빠른 슬라이스를 많이, E2E는 핵심 시나리오에만.**
3. **수용 기준(`2-2-requirements.md`의 FR/NFR)은 테스트가 직접 검증**한다.
4. **JaCoCo 라인 커버리지 ≥ 50%** *(CON-A6)*. 미달 시 `./gradlew build` 실패.

---

## 5-1.1 TDD 규약 (강제)

본 저장소의 모든 프로덕션 코드 변경은 **Red → Green → Refactor** 사이클을 따른다.

### 5-1.1.1 사이클

1. **Red** — 추가/변경할 행위를 표현하는 **실패하는 테스트**를 먼저 작성한다. 컴파일 에러도 Red로 간주.
2. **Green** — 그 테스트를 통과시키는 **최소한의** 프로덕션 코드만 작성한다. 통과 외 코드는 금지.
3. **Refactor** — 테스트가 모두 초록인 상태에서 중복 제거/이름 정리/구조 개선. 새로운 행위 추가 금지.

### 5-1.1.2 커밋/PR 규약

- **테스트 없는 프로덕션 변경 PR은 머지 금지.** 리뷰어는 Diff에서 테스트가 같은 변경에 포함되어 있는지 먼저 확인한다.
- 가능한 한 **테스트 커밋을 별도로 분리** (`test: ...` → `feat: ...` 순). 한 커밋에 묶더라도 메시지에 테스트가 먼저 작성되었음을 명시.
- 버그 수정은 **재현 테스트(Red) 먼저, 수정(Green) 나중**. 재현 테스트 없는 버그 수정 PR은 머지 금지.

### 5-1.1.3 예외 (테스트가 선행되지 않아도 되는 경우)

다음만 예외로 인정한다. 그 외는 모두 TDD 대상.

- 설정 파일 (`application*.yml`, `build.gradle`, IDE 설정).
- 정적 리소스 / 문서 (`docs/**`, `README.md`).
- 자동 생성 코드 (Lombok 어노테이션 추가만 하는 경우).
- 커버리지 제외 패턴(§5-1.10)에 해당하는 보일러플레이트.

### 5-1.1.4 선행 테스트가 어느 레이어인지

행위의 위치에 따라 **가장 안쪽 레이어부터** 시작한다.

| 추가/변경 대상 | 시작 레이어 |
| --- | --- |
| 도메인 불변식·계산·정책 | Unit (도메인) |
| Application Service 분기 | Unit (Service + Mock Port) |
| JPA 쿼리/매핑/reconcile | Repository 통합 |
| 컨트롤러 입력 검증/응답 모양 | Web 슬라이스 |
| 사용자 가치 시나리오 | E2E (마지막에 1건) |

여러 레이어가 동시에 필요하면 안쪽부터 차례로 Red→Green을 반복한다. **E2E를 먼저 짜서 안쪽을 끌고 가는 방식 금지** — 안쪽 단위가 비어 있는 채로 E2E만 통과하는 상태는 인정하지 않는다.

---

## 5-1.2 테스트 피라미드

| 레이어 | 비율 (테스트 수) | 실행 시간 목표 |
| --- | --- | --- |
| Unit | ~70% | 모듈 전체 < 30s |
| Repository 통합 | ~15% | 클래스당 < 10s |
| Web 슬라이스 | ~10% | 클래스당 < 3s |
| E2E | ~5% | 시나리오당 < 5s |
| Load | (PR 외 별도) | — |

---

## 5-1.3 레이어별 선택

| 종류 | 어노테이션 | DB | 외부 의존 | 주 용도 |
| --- | --- | --- | --- | --- |
| **Unit** | `@ExtendWith(MockitoExtension.class)` | 없음 | Mock | 도메인 VO/엔티티, 서비스 분기 |
| **Repository 통합** | `@DataJpaTest` + `@Testcontainers` | Postgres (Testcontainers) | 없음 | JPA 매핑/쿼리/트랜잭션 |
| **Web 슬라이스** | `@WebMvcTest(Controller.class)` | 없음 | 서비스 `@MockitoBean` | URL/직렬화/예외 매핑 |
| **E2E** | `@SpringBootTest + @AutoConfigureMockMvc` | H2 | 실제 빈 + Stub 어댑터 | 컨트롤러 → DB 관통 |

휴리스틱:

- 분기 로직만 본다 → **Unit**.
- JPA 동작이 의심된다 → **Repository 통합** (H2 대체 금지).
- 컨트롤러 변환/예외만 본다 → **Web 슬라이스**.
- 컨트롤러부터 DB까지 관통한다 → **E2E**.

---

## 5-1.4 단위 테스트

### 도메인
- 정적 팩토리(`create`) 입력 검증 — 빈 값/음수/길이 초과.
- 1차 컬렉션 불변식 — `Watchlist.MAX_SIZE = 5` 등.
- 도메인 행위 — `Trade.execute()`, `Portfolio.applyTrade(...)` 의 계산.

### Application Service
- 도메인/Port 협업 분기.
- 입력 검증·예외 변환 (`VALIDATION` → `BusinessException`).
- 기본: `@Mock Port` + `@InjectMocks Service`. 시나리오가 길면 InMemory Repository (`TESTING.md` §3-2).

### 순수 계산/지표/유틸
ROI가 가장 높다. 분기·경계값 위주로 촘촘히.

---

## 5-1.5 Repository 통합 테스트

대상: `*RepositoryAdapter`, `*QueryRepositoryAdapter`.

검증:
- JPA 엔티티 ↔ 도메인 매핑 (`toDomain` / `newEntity` / `update`).
- 1차 컬렉션 reconcile (upsert + 삭제 일관성).
- 트랜잭션 경계 — 어댑터는 트랜잭션을 열지 않으므로 테스트가 연다.

운용:
- `@DataJpaTest` + `@Testcontainers`, `@Container static final`로 클래스당 1회.
- 이미지: `postgres:17`.
- **H2로 대체하지 않는다** — JSON/B, UPSERT, 윈도우 함수 등 방언 차이를 못 잡는다.

---

## 5-1.6 Web 슬라이스

대상: `*Controller`. `@WebMvcTest(XxxController.class)` + `@MockitoBean Service`.

검증:
- URL 매핑.
- `ApiResponse<T>` 직렬화 모양.
- 입력 검증 → `400 VALIDATION`.
- `BusinessException` → 정확한 `ErrorType`/HTTP 매핑 (→ `5-2-error-handling.md`).

---

## 5-1.7 E2E 테스트

대상: **사용자 가치 시나리오**가 한 번에 깨지지 않는지.

| ID | 시나리오 | 추적 |
| --- | --- | --- |
| **E2E-1** | 소셜 로그인 → JWT → `/users/me` 200 | U1, FR-A1 |
| **E2E-2** | Watchlist 5개 등록 → 6개째 409 | U2, FR-A3 |
| **E2E-3** | 매수(rationale 포함) → 포트폴리오/거래내역 갱신 | U9, FR-C2/C3 |
| **E2E-4** | 매수 직후 회고 자동 생성 → 아카이브 노출 | U10, FR-C4 |
| **E2E-5** | 퀴즈 조회 → 채점 → 가상 자본금 충전 → 잔고 증가 | U11/U12, FR-D2/D3 |
| **E2E-6** | 랭킹 조회 시 본인 순위 별도 노출 | U13, FR-E2 |

구성: `@SpringBootTest` + `@AutoConfigureMockMvc` + `application-test.yml` (H2). 외부 어댑터는 `@TestConfiguration` Stub.

---

## 5-1.8 SSE / 데이터 파이프라인 / 부하

- **SSE** — 단위: 토큰 인코더/이벤트 빌더. 통합: 가짜 `ChatStreamPort`로 `event: token/source/done` 순서 검증. E2E는 1–2건.
- **sttak-batch** — 단위: 분기 케이스. 통합: 스텝 입출력 스키마. 외부 API(NewsAPI/DART/KIS)는 **반드시 stub**.
- **부하** — k6/Gatling 권장. PR 게이트 아님. 주 1회 stage 자동 실행. 시나리오:
  - L-1: `GET /home/briefing` VU 500 — p95 ≤ 1500ms (NFR-P1).
  - L-2: `POST /trades` 동시 100 — 잔고 일관성 (NFR-D2).
  - L-3: SSE 동시 200 — TTFB ≤ 2s (NFR-P4).
  - L-4: LLM 강제 지연 — 폴백 작동 (NFR-R1).
  - L-5: 30분 soak — 에러율 < 0.1%, 메모리 누수 없음 (NFR-R4).

---

## 5-1.9 Fixture / InMemory 패턴

표준 출처는 `TESTING.md` §2, §3.

- 도메인 객체는 **Fixture 정적 팩토리** (`UserFixture.aUser()`).
- Instancio는 **원시값만** 생성, 도메인 팩토리에 흘려 보냄 — 객체 통째 생성 금지.
- 시나리오가 길어지면 `ConcurrentHashMap` 기반 InMemory Repository.
- 위치: `src/test/java/.../support/`.

---

## 5-1.10 커버리지 정책

- 게이트: **라인 커버리지 ≥ 50%** (`build.gradle` `jacocoTestCoverageVerification`).
- 리포트: `<module>/build/reports/jacoco/test/html/index.html`.
- 제외: `**/*Application*`, `**/config/**`, `**/dto/**`, `**/result/**`, `**/command/**`, `**/query/*View*`, `**/exception/**`, `**/persistence/*JpaEntity*`.
- 분기 커버리지를 추가하려면 `counter = 'BRANCH'` 룰을 더한다.

---

## 5-1.11 안티패턴

- **TDD 우회** — 프로덕션 코드부터 짜고 사후에 테스트를 끼워 맞춤.
- **E2E로 안쪽 레이어 대체** — 안쪽 단위 테스트가 비어 있는 채 E2E만 통과.
- **H2로 Repository 통합 테스트 대체** — 방언 차이를 못 잡는다.
- **`@MockitoBean` 남발로 슬라이스가 사실상 Unit이 되는 경우** — 그럴 거면 Unit으로.
- **도메인 객체를 Instancio로 통째 생성** — 정적 팩토리 우회 = 불변식 검증 우회.
- **테스트에서 `Date.now()` / `Math.random()` 직접 사용** — 비결정적. 클럭/난수원을 주입.

---

## 5-1.12 요구 추적 매트릭스 (요약)

| 요구 | 주된 검증 레이어 |
| --- | --- |
| FR-A3 (Watchlist ≤ 5) | Unit + Web 슬라이스 + E2E-2 |
| FR-C2 (rationale 필수) | Web 슬라이스 + E2E-3 |
| FR-C3 (잔고 일관성) | Repo 통합 + L-2 |
| FR-D1 (6h 쿨다운) | Unit + Web 슬라이스 |
| FR-D3 (정답 → 자본금 충전) | Unit + E2E-5 |
| FR-G2 (SSE) | 통합(스트리밍) + L-3 |
| NFR-P1 | L-1 |
| NFR-R1 (장애 격리) | L-4 |
| NFR-D2 (트랜잭션 일관성) | Repo 통합 + L-2 |

---

## 5-1.13 후속 문서

| 문서 | 내용 |
| --- | --- |
| `5-2-error-handling.md` | 예외 → HTTP 매핑 검증 |
| `6-observability.md` | 부하 테스트 결과의 메트릭 대시보드 |
