# 1. Background — sTTak 백엔드

본 문서는 sTTak(딱) 서비스의 **백엔드 작업자가 본격적인 스펙·설계 문서를 읽기 전에**
다.
세부 스펙(요구사항, 아키텍처, 디렉터리, 계약, 테스트, 운영)은 본 디렉터리의 후속 문서를 참조한다.

---

## 1.1 프로젝트 개요

### 한 줄 정의

> **sTTak** — 감으로 투자하는 초보자를 위한 **AI 뉴스 요약 · 모의투자 학습 플랫폼**.

### 해결하려는 문제

20\~30대 초보 투자자(투자 경력 3년 미만)는 시장 진입 직후 세 가지 벽을 동시에 마주친다.

1. **정보 과부하** — 뉴스/공시/커뮤니티/유튜브 등 출처가 파편화되어 핵심을 잡기 어렵다.
2. **용어 지옥** — `공시`, `볼린저밴드`, `커버드콜` 같은 기초 용어부터 막히고,
   한 용어를 검색하면 그 안에 모르는 단어가 또 등장한다.
3. **인과관계 판단의 한계** — “이란 분쟁 → 해협 봉쇄 → 원유가 ↑ → 항공주 ↓” 같은
   다단계 거시 인과를 스스로 그려내지 못한다.

결과적으로 초보 투자자는 **“감”으로 매수**하고 손실 후 시장에서 이탈한다.

### 해결 전략 (네 가지 동사)

| 전략 | 설명 | 매핑되는 핵심 기능 |
| --- | --- | --- |
| **모아주기** | 관심 종목 단위로 뉴스/공시를 LLM이 100자 요약 | 100자 요약 피드 |
| **풀어주기** | 모르는 용어는 차트 위 시각화 + 챗봇으로 즉시 학습 | 차트 인터랙티브 용어 학습 |
| **연결해주기** | 거시 이슈는 지식 그래프 인과 체인으로, 상반된 해석은 두 LLM 토론으로 | 인과/토론 시각화 |
| **체화시키기** | 포인트 기반 모의투자로 매매 직접 체험 + AI 회고 코칭 | 퀴즈/포인트/모의투자 |

### 결과물 형태

- **모바일 앱** — iOS.
- **관리자 대시보드** — 데이터 파이프라인 상태·LLM 호출량·사용자 지표 모니터링(내부용).

> 본 레포지토리(`sttak-backend-demo`)는 위 결과물 중 **백엔드 서버** 부분만 담당한다.
> 프론트엔드/모바일 앱은 별도 레포지토리에서 진행한다.

---

## 1.2 기술 스택

### 1.2.1 백엔드 (이 레포지토리)

| 구분 | 선택 | 비고 |
| --- | --- | --- |
| 언어 | **Java 21** | `toolchain { languageVersion = JavaLanguageVersion.of(21) }` |
| 프레임워크 | **Spring Boot 3.5.15** | 루트에서 버전만 고정, 각 모듈이 적용 |
| 빌드 | **Gradle (Groovy DSL)** 멀티모듈 | `settings.gradle`에 6개 모듈 |
| ORM | **Spring Data JPA + Hibernate** | `sttak-domain` 모듈에 한정 |
| DB (운영) | **PostgreSQL 17** | `runtimeOnly 'org.postgresql:postgresql'` |
| DB (로컬·테스트) | **H2 / Testcontainers Postgres 17** | 통합 테스트는 Testcontainers Postgres 고정 |
| 보일러플레이트 | **Lombok** | 모든 모듈 공통 |
| 테스트 | **JUnit 5 + AssertJ + Mockito + Instancio 5.3 + Testcontainers 1.20.4** | Instancio는 도메인 팩토리에 주입할 **원시값만** 생성 |
| 커버리지 | **Jacoco 0.8.12 (최소 50%)** | 미달 시 `./gradlew build` 실패 (CI 차단) |

### 1.2.2 백엔드 외부 인접 기술 (참고)

백엔드가 직접 호출하거나 데이터를 주고받는 외부 요소다. 구현은 `sttak-external` 모듈 또는
`sttak-batch`에서 점진적으로 추가된다.

| 구분 | 항목 |
| --- | --- |
| 데이터 소스 | NewsAPI, DART OpenAPI, KRX/한국거래소 공개 데이터, 모의투자용 시세 |
| LLM | OpenAI GPT-4o-mini, Anthropic Claude (요약/설명/챗봇 용도별 분기) |
| LLM 오케스트레이션 | LangChain 기반 RAG, Function Calling, Multi-Agent(토론형) |
| 저장소 (예정) | Vector DB(임베딩), Graph 저장소(지식 그래프), 캐시(Redis 등) |
| 인프라 | AWS (EC2/ECS, RDS, S3), CloudFront, GitHub Actions(CI/CD) |
| 협업 | GitHub, Jira, Postman, IntelliJ |

> 위 외부 요소들은 **도메인 Port의 구현체**로 `sttak-external`에 들어온다.
> 도메인은 외부 SDK/HTTP를 직접 알지 않는다.

---

## 1.3 도메인 용어 (Ubiquitous Language)

코드, 스펙, 회의에서 동일한 단어를 동일한 의미로 사용하기 위한 사전이다.
**한국어 표기를 1차로 사용**하고, 코드/식별자에는 괄호 안 영문을 사용한다.

### 1.3.1 사용자/계정

| 용어 | 정의 |
| --- | --- |
| **사용자 (User)** | sTTak에 가입해 식별자를 갖는 주체. 코드의 `User` 애그리거트와 1:1 대응. |
| **관심 종목 (Watchlist Item)** | 한 사용자가 등록한 종목. **최대 5개**의 불변식을 가진다. |

### 1.3.2 정보 컨텐츠

| 용어 | 정의 |
| --- | --- |
| **종목 (Stock / Ticker)** | 거래 단위. 종목 코드(예: `005930`) 기반으로 매칭. |
| **뉴스 (News)** | NewsAPI 등에서 수집한 기사 원문. |
| **공시 (Disclosure)** | DART에서 수집한 전자공시 원문. |
| **요약 카드 (Summary Card)** | 뉴스/공시 원문을 LLM이 **100자**로 요약한 카드. 피드의 최소 단위. |
| **요약 클러스터** | 동일 이슈에 대한 중복 기사를 임베딩으로 묶은 그룹. 대표 요약 1건만 생성한다. |
| **인과 그래프 (Causal Graph)** | “분쟁 → 유가↑ → 항공주↓” 같은 다단계 인과를 표현하는 지식 그래프. GraphRAG의 소스. |
| **토론형 멀티에이전트 (Debate Multi-Agent)** | 강세론자/약세론자 두 LLM이 동일 이슈에 대해 상반된 시나리오를 구성하는 응답 패턴. |

### 1.3.3 학습/모의투자

| 용어 | 정의 |
| --- | --- |
| **퀴즈 (Quiz)** | 6시간마다 갱신되는 추천 퀴즈 + 관심 종목 기반 맞춤 퀴즈. |
| **포인트 (Point)** | 퀴즈 정답·연속 학습 보상으로 적립되는 학습 보상 단위. |
| **가상 자본금 (Virtual Capital)** | 포인트를 충전해 사용하는 모의투자용 자본금. **실제 화폐 가치 없음.** |
| **모의 매매 (Mock Trade)** | 가상 자본금으로 수행하는 체결. 실거래 X. |
| **매수 근거 (Trade Rationale)** | 모의 매매 직전에 사용자가 직접 입력하는 텍스트. AI 회고의 입력값이 된다. |
| **회고 코칭 (Retrospective Coaching)** | 모의 매매 결과와 매수 근거를 비교해 LLM이 제공하는 학습 피드백. |
| **랭킹 (Ranking / Leaderboard)** | 친구·전체 사용자 간 학습/모의투자 성과 비교. |

### 1.3.4 AI·데이터 처리

| 용어 | 정의 |
| --- | --- |
| **RAG (Retrieval-Augmented Generation)** | LLM 응답 전에 Vector DB에서 관련 컨텍스트를 검색해 프롬프트에 주입하는 패턴. 환각 방지의 1차 장치. |
| **GraphRAG** | Vector 검색 + 지식 그래프 탐색을 결합한 RAG. 인과 설명에 사용. |
| **Function Calling** | 차트 지표 계산 함수(이동평균/볼린저/RSI/MACD 등)를 LLM이 직접 호출해 좌표를 반환하게 하는 방식. |
| **데이터 파이프라인 (Data Pipeline)** | 외부 뉴스/공시/시세를 주기적으로 수집·정제·태깅·요약·임베딩하는 비동기 배치. `sttak-batch`가 담당 예정. |
| **LLM 가드레일 (Guardrail)** | “사야 한다”, “무조건 오른다” 같은 투자 추천성 표현을 차단하는 응답 후처리. |

---

## 1.4 작동 원리 (How it Works)

### 1.4.1 두 갈래 처리 구조

sTTak 백엔드는 **두 개의 처리 경로**로 분리되어 동작한다.

```
┌──────────────────────────────────────────────────────────────────┐
│ (A) 실시간 사용자 응답 경로                                       │
│                                                                  │
│   Client ──HTTP──▶  sttak-api  ──▶  sttak-domain (Port)          │
│                         │              │                         │
│                         │              ▼                         │
│                         │       Repository Adapter ──▶ Postgres  │
│                         ▼                                        │
│                  (RAG/Function Calling) ──▶ LLM Provider         │
└──────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────┐
│ (B) 비동기 데이터 파이프라인 경로                                 │
│                                                                  │
│   Scheduler ──▶ sttak-batch ──▶ NewsAPI / DART / KRX             │
│                      │                                           │
│                      ▼                                           │
│         정제 · 종목 태깅 · 클러스터링 ──▶ LLM 요약 / 임베딩      │
│                      │                                           │
│                      ▼                                           │
│         Postgres  +  Vector DB  +  Graph 저장소                  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              └─▶ (A)가 RAG/GraphRAG로 읽어감
```

- **(A) 실시간 경로** — 사용자 요청은 지연 없이 응답해야 하므로 **이미 가공된 데이터**만 본다.
- **(B) 배치 경로** — 비싼 작업(LLM 요약, 임베딩, 그래프 추출)은 사용자 트래픽과 분리.
- 두 경로는 **같은 Postgres + Vector DB + Graph 저장소**를 공유한다.

### 1.4.2 단일 요청의 흐름 (예: 사용자 등록)

`3-1-server-architecture.md`의 흐름을 압축하면 다음과 같다.

```
HTTP POST /api/v1/users
   ▼
UserController            (Presentation: 직렬화 + 입력 검증)
   ▼
UserService.register()    (Application: @Transactional 경계)
   ├── User.create(...)               (Domain: 불변식 검증)
   └── userRepository.save(user)      (Port)
        ▼
   UserRepositoryAdapter              (Adapter: package-private)
        └── JPA upsert + reconcile
   ▼
UserResult.from(user) → UserResponse → ApiResponse.success
```

핵심 규약:

- **트랜잭션 경계는 Application Service만 소유**한다. Controller·Adapter는 트랜잭션을 시작하지 않는다.
- **도메인 객체의 `public 생성자` 금지** — 정적 팩토리 `create`(신규)/`restore`(복원)만 사용.
- **JPA 엔티티는 도메인 모듈 밖에 노출되지 않는다** — `*.persistence` 패키지는 `package-private`.

### 1.4.3 LLM 요청의 흐름 (RAG 한 사이클)

```
1) Client가 종목/이슈/차트 컨텍스트와 함께 질문 전송
2) sttak-api 가 도메인 Port를 통해 컨텍스트 조립
     ├── 시계열/지표 값        (Postgres)
     ├── 최근 100자 요약        (Postgres)
     └── 임베딩 검색 결과       (Vector DB)
3) (필요 시) GraphRAG로 인과 노드 확장 (Graph 저장소)
4) 가드레일 프롬프트 + 컨텍스트 → LLM Provider 호출
5) 응답 후처리 (추천성 표현 차단, 출처 부착)
6) Client에 응답 + 응답 카드별 👍/👎 피드백 수집
```

- **반복되는 요약 설명은 캐싱**해 LLM 비용을 줄인다.
- **요약 파이프라인(GPT-4o-mini)** 과 **챗봇 응답** 모델을 분리해 비용/지연을 별도 최적화.
- 변동성이 큰 이슈에만 토론형 Multi-Agent를 제한적으로 적용한다.

---

## 1.5 후속 문서

본 문서를 다 읽었다면 다음 순서로 진행한다.

| 문서 | 내용 |
| --- | --- |
| `2-1-user-story.md` | 사용자 스토리 |
| `2-2-requirements.md` | 요구사항 명세 |
| `3-1-server-architecture.md` | 서버 아키텍처 상세 |
| `3-2-directory.md` | 디렉터리/모듈 규약 |
| `3-3-contract.md` | API 계약 |
| `5-1-test-case.md` | 테스트 케이스 |
| `5-2-error-handling.md` | 에러 처리 정책 |
| `6-observability.md` | 관측성 (로그/메트릭/트레이싱) |
| `7-deployment.md` | 배포 / CI·CD |

세부 백엔드 구조와 패키지 가시성 규약은 `3-1-server-architecture.md`,
테스트 규칙은 [`TESTING.md`](../../TESTING.md)를 함께 참고한다.
