 # 3-4. ERD (Entity-Relationship Diagram) — sTTak 백엔드

본 문서는 sTTak 백엔드의 **데이터 모델 초안(ERD)** 을 정리한다.
도메인 구현을 시작하기 전 합의해야 할 테이블/컬럼/관계의 출발점이며,
컨벤션·타입·제약을 맞춰가는 과정에서 본 문서를 갱신한다.

연결 문서:

- 시스템 구조: `3-1-server-architecture.md`
- 디렉터리/모듈 구조: `3-2-directory.md` (도메인 그룹 → `sttak-domain` 패키지 매핑)
- API 계약: `3-3-contract.md`
- 에러 매핑: `5-2-error-handling.md` (제약 위반 → `ErrorType`)

> **상태: 초안 (draft).**
> 컬럼 타이포, `ENUM` 값 목록, 외래키/유니크 제약, `VARCHAR` 길이가 아직 확정되지 않았다.
> 정리 필요 사항은 §3-4.5 에 모아둔다 — v2 DDL을 만들 때 본 절을 체크리스트로 사용한다.

---

## 3-4.1 개요

총 25개 테이블. Bounded Context 단위로 묶으면 다음 8개 그룹이다 (`3-2-directory.md`의 도메인 분할과 같은 단위).

| 도메인 그룹 | 테이블 |
| --- | --- |
| 사용자/계좌 | `User`, `Account` (V1 구현: `members`, `accounts` — §3-4.2.1) |
| 시장/종목 | `Market`, `MarketSessions`, `Stock`, `Sector`, `Stock_Sector` |
| 주문/보유/관심 | `Order`, `Holdings`, `Watchlist` |
| 주문 회고 | `OrderReview`, `OrderReason`, `ReasonTemplate`, `ReasonEvaluation` |
| 차트/시그널 | `Stock_Candle`, `Chart_Terms`, `Stock_Chart_Signals`, `Chart_Signal_Explanations` |
| 뉴스/용어 | `News`, `term`, `News_Stock`, `News_Term` |
| 퀴즈 | `Quiz`, `Quiz_User` |
| 챗봇 | `ChatBot` (스텁) |

---

## 3-4.2 엔티티 / 관계 요약

각 그룹의 핵심 컬럼과 관계만 적는다 (전체 컬럼·타입은 §3-4.4 원본 DDL 참조).
`*` 가 붙은 컬럼은 외래키 의도(현재 DDL에는 FK 제약이 명시되어 있지 않다 — §3-4.5 참조).

### 3-4.2.1 사용자/계좌

> **구현됨 (V1 마이그레이션 — SCRUM-46 / ADR-007·008·009).** 이 그룹은 초안에서 확정되어 실제 스키마로 구현되었다.
> 운영 DB(PostgreSQL) 기준 실제 테이블명은 **복수형 `members` / `accounts`**, 회원 FK 컬럼은 `member_id`(모호한 `id`/`id2` 대신)다.
> 명명 확정 사유(복수형 채택, `member_id` FK)는 [ADR-009](../decisions/ADR-009-member-account-schema-naming.md).
> 소셜 로그인 요구로 `members` 에 `nickname`·`level`·`social_provider`·`social_id` 가 추가되었다.

- `members` (초안 `User`) — PK `id` (UUID), `role`(`USER`|`ADMIN`), 인증 정보(`email`·`password`·`phone_number` — 소셜 전용이라 nullable), 프로필(`nickname` NOT NULL, `level` INT DEFAULT 1), 소셜 신원(`social_provider` `KAKAO`|`GOOGLE`|`APPLE`, `social_id`), `created_at`/`updated_at`
  - 복합 UNIQUE `uq_members_social (social_provider, social_id)` — 공급자 간 ID 충돌 없이 중복 가입 방지
- `accounts` (초안 `Account`) — PK `account_id` (BIGSERIAL), `member_id*` → `members.id`, `cash_balance`(가입 시 `10,000,000`), `total_asset`, `account_status`(`ACTIVE`|`CLOSED`), `created_at`/`updated_at`
  - 회원 최초 가입과 **동일 트랜잭션**으로 자동 생성. 인덱스 `idx_accounts_member_id`.

관계:

- `members 1 ──< N accounts` (`accounts.member_id` → `members.id`; 현재 정책상 회원당 1계좌)

### 3-4.2.2 시장/종목

> **SCRUM-51 구현**: 복수형 `stocks`(ADR-009). **종목 마스터는 data.go.kr 금융위 KRX상장종목정보(`15094775`)로 동기화**([ADR-010](../decisions/ADR-010-market-data-source-datagokr.md)) — `stock_code`(단축코드)·`stock_name`·**`market_id`*(→`markets` FK)**·`isin_code`·`is_active`. **MVP 대상은 큐레이션 10종목**(삼성전자 005930, SK하이닉스 000660, 현대차 005380, SK스퀘어 402340, 삼성바이오로직스 207940, 삼성물산 028260, 삼성생명 032830, 한화에어로스페이스 012450, 현대모비스 012330, 한미반도체 042700).
> `Market`/`MarketSessions`(복수형 `markets`/`market_sessions`)는 **API에 없는 정적 참조데이터라 수동 시드**(V4): 시장 KOSPI/KOSDAQ/KONEX(KR/KRW), 정규장 세션 09:00~15:30. `stocks.market`(문자열)은 **`market_id` FK 로 전환**(V4).

- `Market` (구현 `markets`) — PK `market_id`, `market_code`(자연 유일키)·`market_name`·`country_code`·`currency_code`. 수동 시드.
- `MarketSessions` (구현 `market_sessions`) — PK `session_id`, `market_id*` → `markets`, `session_type`, `open_time`, `close_time`, `tradable`. `UNIQUE(market_id, session_type)`.
- `Stock` (구현 `stocks`) — PK `stock_id`, `market_id*` → `markets.market_id`, `stock_code`, `stock_name`, `isin_code`, `is_active` (`listed_date`/`stock_status`는 미도입).
- `Sector` — PK `sector_id`, `sector_name` — **SCRUM-51 구현(`sectors`): GICS 11섹터 자체 taxonomy 수동 시드([ADR-012](../decisions/ADR-012-sector-classification.md)). `sector_code`·`description` 추가.**
- `Stock_Sector` — PK `stock_sector_id`, `stokc_id*` → `Stock`, `sector_id2*` → `Sector` (M:N 매핑 테이블) — **구현(`stock_sectors`): `UNIQUE(stock_id, sector_id)`. 복합/지주 기업은 복수 섹터(예: SK스퀘어=IT+금융, 삼성물산=산업재+경기소비재).**

관계:

- `Market 1 ──< N MarketSessions`
- `Market 1 ──< N Stock`
- `Stock N >──< N Sector` (via `Stock_Sector`)

### 3-4.2.3 주문/보유/관심

> **구현됨 (V14 — SCRUM-32, [ADR-011](../decisions/ADR-011-mock-trade-execution-model.md) 접수·익일 정산 모델).**
> `order_status` 는 `PENDING`(접수) → `FILLED`(정산 체결) / `REJECTED`(최종 검증 미달, 사유 기록) / `CANCELLED`(사용자 취소 — 행 삭제 아님) 흐름.
> 체결가는 **매수=D 시가 / 매도=D 종가**, 정산은 별도 배치 잡(`tradeSettlementJob`, `--run-trade-settlement`)이 수행 —
> D 캔들 결측(공휴일·거래정지)은 이후 첫 캔들로 재귀속되며 그때까지 PENDING 유지. 초안의 `order_reasons` 는 MVP 제외
> (템플릿 선택도 텍스트로 `orders.rationale` 에 저장), 회고/근거평가는 AI 회고 티켓으로 이동.

- `orders` (V14) — PK `order_id`, `account_id*` → `accounts`, `stock_id*` → `stocks`(오타 정정), `order_side`(BUY|SELL), `quantity`(>0 CHECK), `rationale`(140), `order_status`(4종 CHECK), `trading_date`(체결 기준일 D — 주말 접수는 다음 영업일 귀속), `reference_price`(접수 시점 참고가), `filled_price`/`filled_at`, `rejected_reason`, `realized_profit`(매도 FILLED 만), `ordered_at`. 인덱스: `(account_id, ordered_at DESC)` 거래내역, `(order_status, trading_date)` 정산 스캔.
- `holdings` (V14) — PK `holdings_id`, `account_id*`·`stock_id*`, `quantity`(≥0 CHECK — 공매도 금지, 전량 매도 시 0 행 유지), `average_price`(가중평균·정수 절사는 도메인 소유), `total_buy_amount`, **`UNIQUE(account_id, stock_id)`** — 정산 upsert 키.
- `reason_templates` (V14) — PK `reason_template_id`, `reason_type`(BUY|SELL), `reason_label`, `UNIQUE(reason_type, reason_label)`. iOS presets 문구 6건 수동 시드(고정 동기화).
- `member_watchlist` (V10, 초안 `Watchlist`) — 기구현. 초안의 soft delete 정책 등은 V10 구현 기준.

관계:

- `accounts 1 ──< N orders`, `accounts 1 ──< N holdings` (계좌×종목당 1행)
- `stocks 1 ──< N orders`, `stocks 1 ──< N holdings`

### 3-4.2.4 주문 회고

> **구현됨 (V14 reason_templates — SCRUM-32 / V15 order_reviews·reason_evaluations — SCRUM-68, [ADR-011](../decisions/ADR-011-mock-trade-execution-model.md) §7).**
> 초안 대비 정정: `proofit_loss_amount` 오타 → `profit_loss_amount`, 모호한 `AI_feedback` 단일 컬럼 →
> iOS 표시 계약에 맞춘 구조 컬럼(`summary_line`·`good_points`·`watch_points` TEXT[]), `ReasonEvaluation` 의
> VARCHAR PK → BIGSERIAL, `model_name` 은 회고 본체로 이동. **`OrderReason` 은 미도입**(MVP 제외, SCRUM-32
> — 근거는 `orders.rationale` 자유 입력 + `reason_templates` 빠른 선택 문구) — 따라서 `ReasonEvaluation`
> 의 `order_reason_id` FK 도 제외되고 평가 대상은 `orders.rationale` 이다.

- `order_reviews` (V15) — PK `review_id`, `order_id*` → `orders` **UNIQUE**(1:1, 생성 배치 멱등 키), `summary_line`, `good_points`/`watch_points`(TEXT[], 각 1개 이상 — `cardinality` CHECK), `profit_loss_amount`(정수 원)·`profit_loss_rate`(DECIMAL(9,4) %)·`is_partial_sell`(생성 시점 스냅샷 — 주문 이력 기준 판정), `model_name`. **생성 후 불변**(updated_at 없음)
- `reason_templates` (V14) — PK `reason_template_id`, `reason_type`(`BUY`|`SELL`), `reason_label` — 정적 시드
- `reason_evaluations` (V15) — PK `reason_evaluation_id`(BIGSERIAL), `review_id*` → `order_reviews`, `evaluation_type`(`GOOD`|`BAD`|`IMPROVEMENT` CHECK), `evaluation_content`

관계:

- `orders 1 ── 0..1 order_reviews` (매도 FILLED 만 생성 대상)
- `order_reviews 1 ──< N reason_evaluations`

### 3-4.2.5 차트/시그널

> **구현됨 (V2·V12·V13 — SCRUM-51/31, [ADR-020](../decisions/ADR-020-chart-signal-server-ssot.md)).**
> 신호 감지·기록은 서버 SSOT — 배치(SCRUM-61 잡 후속 스텝)가 규칙 기반으로 감지·적재하고 API 는 읽기만 한다.
> 지표 곡선(오버레이)은 iOS 온디바이스 계산 유지, AI 해설 생성(LLM)은 별도 배치(SCRUM-65)가 채운다.

- `stock_candles` (V2, SCRUM-51) — PK `candle_id`, `stock_id*` → `stocks`, `candle_type`(`'1D'`만, [ADR-010](../decisions/ADR-010-market-data-source-datagokr.md)), `market_date`, `open_price`/`high_price`/`low_price`/`close_price`/`volume`, `UNIQUE(stock_id, candle_type, market_date)`. 주봉/월봉은 향후 일봉 집계로 파생. 이 일봉이 모의투자 체결 정산 트리거([ADR-011](../decisions/ADR-011-mock-trade-execution-model.md))이자 신호 감지 트리거.
- `chart_terms` (V12) — PK `chart_term_id`, `term_code` VARCHAR **UNIQUE**(`MA`/`RSI`/`BOLLINGER`/`SUPPORT`/`VOLUME`/`INFO` 6종 수동 시드), `term_name`, `easy_meaning`, `detail_meaning` (초안 `term_name` ENUM → `term_code` 자연키로 정리)
- `stock_chart_signals` (V12·V13) — PK `chart_signal_id`, `candle_id*` → `stock_candles`, `chart_term_id*` → `chart_terms`, `signal_kind` VARCHAR+CHECK(8종: 골든/데드크로스·RSI 과열/과매도·볼린저 상/하단 터치·지지 반등/저항 눌림), `subsequent_direction` VARCHAR+CHECK(`UP`/`DOWN`/`SIDEWAYS`, V13 — 신호 후 +7거래일 ±1.5% 기준), **`UNIQUE(candle_id, signal_kind)`** — 배치 멱등 upsert 키
- `chart_signal_explanations` (V12) — PK `chart_signal_explanation_id`, `chart_signal_id*` → `stock_chart_signals` **UNIQUE**(신호당 해설 1건), `explanation` TEXT(초안 오타 `exlpanation` 정정, 모호한 `Field` 컬럼 제거), `model_name`(생성 모델 기록)

관계:

- `stocks 1 ──< N stock_candles`
- `stock_candles 1 ──< N stock_chart_signals` (캔들당 신호 종류별 최대 1건)
- `chart_terms 1 ──< N stock_chart_signals`
- `stock_chart_signals 1 ── 0..1 chart_signal_explanations` (해설은 SCRUM-65 배치가 생성 — 미생성이면 API 가 null 반환)

### 3-4.2.6 뉴스/용어

- `News` — PK `news_id`, `original_title/content/link`, `news_summary`, `easy_content`, `source_name`, `published_at`
- `term` — PK `term_id`, `term_name`, `term_meaning` (일반 경제 용어 사전)
- `News_Stock` — PK `news_stock_id`, `news_id*` → `News`, `stokc_id*` → `Stock`, `factor`(ENUM, 호재/악재/중립 등 추정) — 뉴스 ↔ 종목 M:N
- `News_Term` — PK `news_term_id`, `news_id*` → `News`, `term_id*` → `term` — 뉴스 ↔ 용어 M:N

관계:

- `News N >──< N Stock` (via `News_Stock`, with `factor`)
- `News N >──< N term` (via `News_Term`)

> **AI 가공 확장 (ADR-003)** — 수집 원문을 iOS `/news` 카드 형태로 가공하기 위해 다음을 추가한다.
> 기사 공용 필드는 `News`, 종목별로 달라지는 필드는 `News_Stock` 에 둔다(계약 §3-3.15).
>
> - `News` +: `easy_title`(쉬운 제목), `easy_one_liner`(한 줄, ≤100자), `easy_detail`(상세 풀이),
>   `processing_status` ENUM(`COLLECTED`,`ENRICHED`,`FAILED`), `enriched_at`
>   - 기존 `news_summary`/`easy_content` 는 위 신규 필드로 대체·정리 (§3-4.5 v2 DDL 에서 반영).
> - `News_Stock` +: `reason`(왜 호재/악재인지 한 줄 근거). `factor` 는 수집 시 NULL, 가공 시 채움.
> - `News_Stock_Why_Point` (신규 자식): PK `why_point_id`, `news_stock_id*` → `News_Stock`, `seq`, `content`
>   — 카드의 `whyPoints[]`(왜 중요한지) 정규화, 순서 보존.
> - `lead`(원문 미리보기)는 `News.original_content` 앞부분에서 파생하며 별도 컬럼을 두지 않는다.
> - 자연 키 `News.original_link` 는 UNIQUE(중복 적재 방지). iOS 조회는 `processing_status='ENRICHED'` 만 노출.

### 3-4.2.7 퀴즈

- `Quiz` — PK `quiz_id`, `quiz_content`, `choice_a/b/c/d`, `correct_choice`(INT), `explanation`, `point`
- `Quiz_User` — PK `quiz_user_id`, `id*` → `users.id`, `quiz_id*` → `Quiz`, `start_at`, `end_at`, `is_solved`(ENUM), `selected_choice`

관계:

- `users N >──< N Quiz` (via `Quiz_User`)

### 3-4.2.8 챗봇 / RAG 지식 저장소

> **SCRUM-42 (V9__chat_knowledge.sql, ADR-016/017/018)** — 기존 `ChatBot` 스텁은 폐기한다.
> 챗 세션/메시지는 **저장하지 않는다**(클라이언트가 전체 히스토리를 매 호출 전달, apidocs §7).
> 대신 RAG 지식 저장소 2 테이블을 둔다.

- `term` — PK `id`(BIGSERIAL), `term_name` VARCHAR(200) **UNIQUE**, `term_meaning` TEXT, `created_at` TIMESTAMPTZ
  - 일반 경제/투자 용어 사전(§3-4.2.6 의 `term` 을 구체화). `news_term` 의 distinct (term, definition) 쌍을
    배치가 upsert 로 승격한다(ON CONFLICT DO NOTHING).
  - 인덱스: `idx_term_name_trgm` — GIN(`term_name gin_trgm_ops`), 질문 속 용어 exact-ish 매칭용(ADR-016).
- `knowledge_chunk` — PK `id`(BIGSERIAL), `source_type` VARCHAR(20)(`NEWS`|`TERM`), `source_id` BIGINT,
  `content` TEXT, `stock_codes` TEXT[], `published_at` TIMESTAMPTZ, `embedding` **vector(1536)**, `embedded_at`
  - **UNIQUE (source_type, source_id)** — 1뉴스=1청크·1용어=1청크(ADR-018), 임베딩 배치 멱등 키.
  - 인덱스: `idx_knowledge_chunk_embedding` — HNSW(`vector_cosine_ops`), `idx_knowledge_chunk_stock` — GIN(`stock_codes`).
  - `source_id` 는 논리 참조(뉴스/용어 두 테이블을 가리켜 FK 미설정). **JPA 미매핑** — `vector`/`TEXT[]` 는
    JdbcClient 네이티브 SQL 로만 접근한다(ADR-017).

관계:

- `knowledge_chunk N ──> 1 News` (source_type=NEWS, 논리 참조)
- `knowledge_chunk N ──> 1 term` (source_type=TERM, 논리 참조)

---

## 3-4.3 카디널리티 한눈에 보기

```
                ┌───────┐
                │ users │
                └───┬───┘
                   │ 1
        ┌──────────┼────────────────────────────────────────────┐
        │ N        │ N                                          │ N
   ┌────▼─────┐  ┌─▼─────────┐                              ┌───▼────┐
   │ Account  │  │ Watchlist │                              │QuizUser│──N→ Quiz
   └────┬─────┘  └─────┬─────┘                              └────────┘
        │ 1            │ N
        │ N            ▼
   ┌────▼────┐      ┌──────┐    N      ┌────────────┐
   │  Order  │ ─N──▶│Stock │ ◀──N──── │ Stock_Candle│ ──N──▶ Stock_Chart_Signals ──N──▶ Chart_Signal_Explanations
   └────┬────┘      └──┬───┘           └─────────────┘                ▲
        │ 1            │ N                                            │ N
        │ 1            │                                              │
   ┌────▼─────┐     ┌──▼──────┐  N    ┌─────┐                   ┌────▼─────┐
   │OrderRev. │     │ Market  │◀──1── │Stock│                   │ChartTerms│
   └────┬─────┘     └────┬────┘       └──┬──┘                   └──────────┘
        │ N              │ 1             │ N
        ▼                ▼               ▼
   OrderReason     MarketSessions     Stock_Sector ──N──▶ Sector
        │ 1
        ▼
   ReasonTemplate, ReasonEvaluation

   ┌──────┐  N      N  ┌─────┐
   │ News │──────────▶│Stock│   (News_Stock, factor)
   │      │  N      N  │term │   (News_Term)
   └──────┘──────────▶ └─────┘
```

(아스키 다이어그램은 가독성용 요약이며, 정확한 정의는 §3-4.4 DDL과 §3-4.2 관계 표를 본다.)

---

## 3-4.4 원본 DDL (Draft v1)

ERDCloud에서 추출된 초안을 원본 그대로 보존한다. 타이포·길이 누락·FK 부재는 §3-4.5 에서 추적한다.

> **사용자 테이블명 정합 (ADR-002):** 본 v1 원본 DDL 의 `CREATE TABLE `User`` / `ALTER TABLE `User`` 는 보존하지만, v2 부터는 `users` 로 정합되어 있다. 본 문서의 §3-4.1 / §3-4.2 / §3-4.5 본문은 이미 `users` 기준이다. PostgreSQL 예약어 `USER` 와의 충돌이 채택 사유 — `docs/decisions/ADR-002-user-table-naming.md` 참조.

```sql
CREATE TABLE `ChatBot` (
   `Key`   VARCHAR(255)   NOT NULL
);

CREATE TABLE `Order` (
   `order_id`   BIGINT   NOT NULL,
   `acount_id`   BIGINT   NOT NULL,
   `stokc_id`   BIGINT   NOT NULL,
   `order_side`   ENUM   NULL,
   `order_price`   DECIMAL(19,0)   NULL,
   `order_quantity`   BIGINT   NULL,
   `order_status`   ENUM   NULL,
   `order_at`   timestamp   NULL
);

CREATE TABLE `Watchlist` (
   `watchlist_id`   BIGINT   NOT NULL,
   `stokc_id`   BIGINT   NOT NULL,
   `id2`   UUID   NOT NULL,
   `created_at`   DATETIME   NULL,
   `is_deleted`   BOOLEAN   NULL,
   `deleted_at`   DATETIME   NULL
);

CREATE TABLE `ReasonEvaluation` (
   `reason_evaluation_id`   VARCHAR(255)   NOT NULL,
   `review_id`   BIGINT   NOT NULL,
   `order_reason_id`   BIGINT   NOT NULL,
   `model_name`   VARCHAR   NULL,
   `evaluation_type`   ENUM   NULL,
   `evaluation_content`   TEXT   NULL,
   `created_at`   DATETIME   NULL
);

CREATE TABLE `Stock_Chart_Signals` (
   `chart_signal_id`   BIGINT   NOT NULL,
   `candle_id`   BIGINT   NOT NULL,
   `chart_term_id`   BIGINT   NOT NULL,
   `created_at`   DATETIME   NULL,
   `updated_at`   DATETIME   NULL
);

CREATE TABLE `Stock_Candle` (
   `candle_id`   BIGINT   NOT NULL,
   `stokc_id`   BIGINT   NOT NULL,
   `open`   DECIMAL(19,0)   NULL,
   `high`   DECIMAL(19,0)   NULL,
   `low`   DECIMAL(19,0)   NULL,
   `close`   DECIMAL(19,0)   NULL,
   `volume`   DECIMAL(19,0)   NULL,
   `market_date`   DATETIME   NULL,
   `created_at`   DATETIME   NULL
);

CREATE TABLE `Chart_Terms` (
   `chart_term_id`   BIGINT   NOT NULL,
   `term_name`   ENUM   NULL,
   `easy_meaning`   TEXT   NULL,
   `detail_meaning`   TEXT   NULL,
   `created_at`   DATETIME   NULL,
   `updated_at`   DATETIME   NULL
);

CREATE TABLE `ReasonTemplate` (
   `reason_template_id`   BIGINT   NOT NULL,
   `reason_type`   ENUM   NULL,
   `reason_label`   VARCHAR   NULL,
   `created_at`   DATETIME   NULL,
   `updated_at`   DATETIME   NULL
);

CREATE TABLE `User` (
   `id`   UUID   NOT NULL,
   `role`   enum   NULL,
   `email`   VARCHAR   NULL,
   `password`   VARCHAR   NULL,
   `phone_number`   VARCHAR   NULL,
   `created_at`   timestamp   NULL,
   `updated_at`   timestamp   NULL
);

CREATE TABLE `term` (
   `term_id`   BIGINT   NOT NULL,
   `term_name`   TEXT   NULL,
   `term_meaning`   TEXT   NULL,
   `created_at`   DATE_TIME   NULL,
   `updated_at`   DATETIME   NULL
);

CREATE TABLE `News` (
   `news_id`   BIGINT   NOT NULL,
   `original_title`   TEXT   NULL,
   `original_content`   TEXT   NULL,
   `original_link`   TEXT   NULL,
   `news_summary`   TEXT   NULL,
   `easy_content`   VARCHAR   NULL,
   `source_name`   VARCHAR   NULL,
   `published_at`   DATETIME   NULL,
   `created_at`   DATETIME   NULL
);

CREATE TABLE `OrderReview` (
   `review_id`   BIGINT   NOT NULL,
   `order_id`   BIGINT   NOT NULL,
   `order_side`   BOOLEAN   NULL,
   `AI_feedback`   TEXT   NOT NULL,
   `profit_loss_rate`   DECIMAL(19,0)   NULL,
   `proofit_loss_amount`   DECIMAL(7,4)   NULL,
   `created_at`   DATETIME   NOT NULL
);

CREATE TABLE `Chart_Signal_Explanations` (
   `chart_signal_explanation_id`   BIGINT   NOT NULL,
   `chart_signal_id`   BIGINT   NOT NULL,
   `exlpanation`   TEXT   NULL,
   `Field`   VARCHAR(255)   NULL,
   `created_at`   DATETIME   NULL,
   `updated_at`   DATETIME   NULL
);

CREATE TABLE `Quiz` (
   `quiz_id`   BIGINT   NOT NULL,
   `quiz_content`   TEXT   NULL,
   `choice_a`   TEXT   NULL,
   `choice_b`   TEXT   NULL,
   `choice_c`   TEXT   NULL,
   `choice_d`   TEXT   NULL,
   `correct_choice`   INT   NULL,
   `explanation`   TEXT   NULL,
   `point`   BIGINT   NULL,
   `created_at`   DATETIME   NULL,
   `updated_at`   DATETIME   NULL
);

CREATE TABLE `Holdings` (
   `holdings_id`   BIGINT   NOT NULL,
   `acount_id`   BIGINT   NOT NULL,
   `stokc_id`   BIGINT   NOT NULL,
   `quantity`   BIGINT   NULL,
   `average_price`   DECIMAL(19,0)   NULL,
   `total_buy_amount`   DECIMAL(19,0)   NULL,
   `upldated_at`   DATETIME   NULL
);

CREATE TABLE `Stock` (
   `stokc_id`   BIGINT   NOT NULL,
   `market_id2`   BIGINT   NOT NULL,
   `stock_code`   VARCHAR   NULL,
   `stock_name`   VARCHAR   NULL,
   `listed_date`   DATE   NULL,
   `stock_status`   ENUM   NULL
);

CREATE TABLE `News_Stock` (
   `news_stock_id`   BIGINT   NOT NULL,
   `news_id`   BIGINT   NOT NULL,
   `stokc_id`   BIGINT   NOT NULL,
   `factor`   ENUM   NULL
);

CREATE TABLE `Quiz_User` (
   `quiz_user_id`   BIGINT   NOT NULL,
   `id`   UUID   NOT NULL,
   `quiz_id`   BIGINT   NOT NULL,
   `start_at`   DATETIME   NULL,
   `end_at`   DATETIME   NULL,
   `is_solved`   ENUM   NULL,
   `selected_choice`   INT   NULL,
   `created_at`   DATETIME   NULL,
   `updated_at`   DATETIME   NULL
);

CREATE TABLE `Market` (
   `market_id`   BIGINT   NOT NULL,
   `country_code`   enum   NULL,
   `currency_code`   enum   NULL,
   `market_name`   VARCHAR   NULL,
   `market_code`   VARCHAR   NULL
);

CREATE TABLE `Account` (
   `acount_id`   BIGINT   NOT NULL,
   `id`   UUID   NOT NULL,
   `cash_balance`   DECIMAL(19,0)   NULL,
   `total_asset`   DECIMAL(19,0)   NULL,
   `created_at`   timestamp   NULL,
   `updated_at`   timestamp   NULL,
   `account_status`   ENUM   NULL
);

CREATE TABLE `OrderReason` (
   `order_reason_id`   BIGINT   NOT NULL,
   `reason_template_id`   BIGINT   NULL,
   `order_id`   BIGINT   NOT NULL,
   `reason_content`   TEXT   NULL,
   `created_at`   DATTETIME   NULL,
   `updated_at`   DATETIME   NULL
);

CREATE TABLE `MarketSessions` (
   `session_id`   BIGINT   NOT NULL,
   `market_id`   BIGINT   NOT NULL,
   `session_type`   ENUM   NULL,
   `open_time`   TIME   NULL,
   `close_time`   TIME   NULL,
   `tradable`   BOOLEAN   NULL
);

CREATE TABLE `News_Term` (
   `news_term_id`   BIGINT   NOT NULL,
   `news_id`   BIGINT   NOT NULL,
   `term_id`   BIGINT   NOT NULL,
   `created_at`   DATETIME   NULL,
   `updated_at`   DATETIME   NULL
);

CREATE TABLE `Sector` (
   `sector_id`   BIGINT   NOT NULL,
   `sector_name`   ENUM   NULL
);

CREATE TABLE `Stock_Sector` (
   `stock_sector_id`   BIGINT   NOT NULL,
   `stokc_id`   BIGINT   NOT NULL,
   `sector_id2`   BIGINT   NOT NULL
);

ALTER TABLE `ChatBot` ADD CONSTRAINT `PK_CHATBOT` PRIMARY KEY (`Key`);
ALTER TABLE `Order` ADD CONSTRAINT `PK_ORDER` PRIMARY KEY (`order_id`);
ALTER TABLE `Watchlist` ADD CONSTRAINT `PK_WATCHLIST` PRIMARY KEY (`watchlist_id`);
ALTER TABLE `ReasonEvaluation` ADD CONSTRAINT `PK_REASONEVALUATION` PRIMARY KEY (`reason_evaluation_id`);
ALTER TABLE `Stock_Chart_Signals` ADD CONSTRAINT `PK_STOCK_CHART_SIGNALS` PRIMARY KEY (`chart_signal_id`);
ALTER TABLE `Stock_Candle` ADD CONSTRAINT `PK_STOCK_CANDLE` PRIMARY KEY (`candle_id`);
ALTER TABLE `Chart_Terms` ADD CONSTRAINT `PK_CHART_TERMS` PRIMARY KEY (`chart_term_id`);
ALTER TABLE `ReasonTemplate` ADD CONSTRAINT `PK_REASONTEMPLATE` PRIMARY KEY (`reason_template_id`);
ALTER TABLE `User` ADD CONSTRAINT `PK_USER` PRIMARY KEY (`id`);
ALTER TABLE `term` ADD CONSTRAINT `PK_TERM` PRIMARY KEY (`term_id`);
ALTER TABLE `News` ADD CONSTRAINT `PK_NEWS` PRIMARY KEY (`news_id`);
ALTER TABLE `OrderReview` ADD CONSTRAINT `PK_ORDERREVIEW` PRIMARY KEY (`review_id`);
ALTER TABLE `Chart_Signal_Explanations` ADD CONSTRAINT `PK_CHART_SIGNAL_EXPLANATIONS` PRIMARY KEY (`chart_signal_explanation_id`);
ALTER TABLE `Quiz` ADD CONSTRAINT `PK_QUIZ` PRIMARY KEY (`quiz_id`);
ALTER TABLE `Holdings` ADD CONSTRAINT `PK_HOLDINGS` PRIMARY KEY (`holdings_id`);
ALTER TABLE `Stock` ADD CONSTRAINT `PK_STOCK` PRIMARY KEY (`stokc_id`);
ALTER TABLE `News_Stock` ADD CONSTRAINT `PK_NEWS_STOCK` PRIMARY KEY (`news_stock_id`);
ALTER TABLE `Quiz_User` ADD CONSTRAINT `PK_QUIZ_USER` PRIMARY KEY (`quiz_user_id`);
ALTER TABLE `Market` ADD CONSTRAINT `PK_MARKET` PRIMARY KEY (`market_id`);
ALTER TABLE `Account` ADD CONSTRAINT `PK_ACCOUNT` PRIMARY KEY (`acount_id`);
ALTER TABLE `OrderReason` ADD CONSTRAINT `PK_ORDERREASON` PRIMARY KEY (`order_reason_id`);
ALTER TABLE `MarketSessions` ADD CONSTRAINT `PK_MARKETSESSIONS` PRIMARY KEY (`session_id`);
ALTER TABLE `News_Term` ADD CONSTRAINT `PK_NEWS_TERM` PRIMARY KEY (`news_term_id`);
ALTER TABLE `Sector` ADD CONSTRAINT `PK_SECTOR` PRIMARY KEY (`sector_id`);
ALTER TABLE `Stock_Sector` ADD CONSTRAINT `PK_STOCK_SECTOR` PRIMARY KEY (`stock_sector_id`);
```

---

## 3-4.5 초안에서 확인된 이슈 / 정리 필요 체크리스트

v2 DDL을 만들 때 일괄 반영한다.

### A. 컬럼/타입 타이포

| 위치 | 현재 | 수정안 |
| --- | --- | --- |
| `Account`, `Order`, `Holdings` | `acount_id` | `account_id` (✅ `accounts` 는 V1 에서 `account_id` 로 구현. `Order`/`Holdings` 는 미구현) |
| `Order`, `Holdings`, `Stock`, `Watchlist`, `News_Stock`, `Stock_Sector`, `Stock_Candle` | `stokc_id` | `stock_id` |
| `Chart_Signal_Explanations` | `exlpanation` | `explanation` |
| `Holdings` | `upldated_at` | `updated_at` |
| `OrderReview` | `proofit_loss_amount` | `profit_loss_amount` |
| `OrderReason` | `created_at DATTETIME` | `DATETIME` |
| `term` | `created_at DATE_TIME` | `DATETIME` |

### B. 명명 일관성

- ERDCloud 자동 생성으로 보이는 `2` 접미사 (`market_id2`, `sector_id2`, `id2`) 제거 — 각각 `market_id` / `sector_id` / `user_id` 로 정리.
- 테이블명 컨벤션 결정: snake_case + **복수형** 확정 ([ADR-009](../decisions/ADR-009-member-account-schema-naming.md) — 잠정 "단수형" 권고에서 변경; `user` 예약어 회피·집합 관례). 현재 `Order`, `term`, `User`, `Stock_Chart_Signals`, `MarketSessions` 등 혼재 → 구현 시 `members`/`accounts` 처럼 복수형으로 정합.
- `Watchlist.id2` 와 `Quiz_User.id` 와 `Account.id` 가 모두 `User.id`를 가리키는데 이름이 다르다. (✅ `accounts` 는 V1 에서 `member_id` → `members.id` 로 확정 — [ADR-009](../decisions/ADR-009-member-account-schema-naming.md). FK 규칙은 `{참조 엔티티}_id`, 즉 회원 FK 는 `member_id` 로 통일. 나머지 미구현 테이블도 이 규칙으로 정합.)
- `Chart_Signal_Explanations.Field` 의 대문자 시작 + `Field` 라는 모호한 이름 — 의미를 살린 이름으로 (예: `field_name`, `aspect`).

### C. PK 타입 일관성

- `ReasonEvaluation.reason_evaluation_id` 만 `VARCHAR(255)` — 나머지 PK는 모두 BIGINT. 의도된 ID 전략(예: 외부 모델 ID 그대로 사용)인지 확인.

### D. 길이/타입 누락

- `VARCHAR` 길이가 비어 있는 컬럼이 다수 (`users.email`, `users.password`, `users.phone_number`, `Market.market_name`, `Market.market_code`, `Stock.stock_code`, `Stock.stock_name`, `ReasonTemplate.reason_label`, `News.easy_content`, `News.source_name`, `ReasonEvaluation.model_name`) — MySQL에서는 길이 지정이 필요하다.
- `users.role`, `Market.country_code`, `Market.currency_code` 가 소문자 `enum` 으로 적혀 있음 — `ENUM(...)` 으로 정의 필요.
- `News.easy_content` 가 `VARCHAR` 인데, 의미상 가변 길이 본문이라면 `TEXT`가 적합.

### E. Enum 값 목록 미정

다음 `ENUM` 컬럼은 값 목록이 비어 있다. 각 값을 본 문서 또는 ADR로 명문화한다.

| 테이블 | 컬럼 | 추정 값(예시) |
| --- | --- | --- |
| `users` | `role` | `USER`, `ADMIN` |
| `Order` | `order_side` | `BUY`, `SELL` |
| `Order` | `order_status` | `PENDING`, `FILLED`, `CANCELLED`, … |
| `Account` | `account_status` | `ACTIVE`, `CLOSED`, … |
| `Stock` | `stock_status` | `LISTED`, `HALTED`, `DELISTED`, … |
| `MarketSessions` | `session_type` | `REGULAR`, `PRE`, `AFTER`, … |
| `Market` | `country_code` | `KR`, `US`, … |
| `Market` | `currency_code` | `KRW`, `USD`, … |
| `Sector` | `sector_name` | (업종 분류) |
| `Chart_Terms` | `term_name` | (차트 패턴 이름) |
| `ReasonTemplate` | `reason_type` | (매수/매도 사유 카테고리) |
| `ReasonEvaluation` | `evaluation_type` | (`STRENGTH`, `WEAKNESS`, … ?) |
| `News_Stock` | `factor` | `POSITIVE`, `NEGATIVE`, `NEUTRAL`, … |
| `Quiz_User` | `is_solved` | (왜 BOOLEAN이 아닌 ENUM인지 의도 확인 — `NOT_STARTED`/`SOLVED`/`SKIPPED` 등?) |

### F. 외래키 / 유니크 / 인덱스

- 현재 DDL에는 FK CONSTRAINT가 단 하나도 없다. §3-4.2 의 관계 표를 근거로 일괄 추가.
- 자연 유일성 제약 후보:
  - `users.email` UNIQUE
  - `Account(id, account_status='ACTIVE')` 부분 유니크 — 사용자별 활성 계좌 정책에 따라
  - `Stock(market_id, stock_code)` UNIQUE
  - `Stock_Candle(stock_id, market_date, ...)` UNIQUE (캔들 중복 방지)
  - `News_Stock(news_id, stock_id)` UNIQUE
  - `News_Term(news_id, term_id)` UNIQUE
  - `Stock_Sector(stock_id, sector_id)` UNIQUE
  - `Watchlist(user_id, stock_id)` 활성 레코드 부분 유니크 (soft delete 고려)
  - `Quiz_User(user_id, quiz_id)` UNIQUE (사용자당 퀴즈 1회 정책일 경우)
- 빈번한 조회 경로의 인덱스 후보: `Order(account_id, order_at desc)`, `Stock_Candle(stock_id, market_date desc)`, `News(published_at desc)`.

### G. 시점/숫자 표현

- `users`/`Account.created_at` 만 `timestamp`, 나머지는 `DATETIME`. 한 가지로 통일 권장 (JPA Auditing 적용 시 `TIMESTAMP WITH TIME ZONE` 또는 `DATETIME(6)` 일관 사용).
- `OrderReview.profit_loss_rate DECIMAL(19,0)` — 비율인데 소수점이 없다. `DECIMAL(7,4)` (예: `-99.9999` ~ `999.9999`) 등으로 재검토.
- `OrderReview.proofit_loss_amount DECIMAL(7,4)` — 금액인데 정밀도가 작다. `DECIMAL(19,0)` 또는 통화 단위에 맞춘 값으로 재검토 (위의 rate와 정의가 뒤바뀐 것으로 보임).

### H. 설계 미완 / 의도 확인

- `ChatBot` — 컬럼 1개(`Key`)만 정의된 스텁. 어떤 책임을 갖는 테이블인지(대화 세션? 프롬프트 캐시?) 확정 후 컬럼 추가.
- `OrderReview.order_side BOOLEAN` — `Order.order_side` 는 ENUM 인데 여기는 BOOLEAN. 회고 시점에 매수/매도 플래그를 재기록하는 의도인지, 혹은 ENUM 정합이 필요한지 결정.
- `OrderReason.reason_template_id` NULL 허용 — 자유 입력 사유 허용을 의미하는지 명시.
- `Watchlist` 의 soft delete (`is_deleted`, `deleted_at`) 와 다른 도메인의 삭제 정책 일관성 확인 (`Order`/`Holdings` 는 soft delete 없음).

---

## 3-4.6 다음 작업

1. §3-4.5 체크리스트를 반영한 **v2 DDL** 을 본 문서 §3-4.4 에 갱신 (원본은 git history에서 추적).
2. 각 `ENUM` 의 값 목록을 본 문서 §3-4.5.E 표에 채워넣고, 변경 비용이 큰 항목은 별도 ADR로 분리 (`docs/decisions/`).
3. 도메인 그룹별로 `sttak-domain` 패키지를 생성 — `3-2-directory.md §3-2.4` 의 Bounded Context 규약을 따른다.
4. JPA 엔티티 매핑 + Flyway/Liquibase 마이그레이션 스크립트 도입 (`7-deployment.md` 의 마이그레이션 방침 참조).

---

## 참고 문서

| 문서 | 의미 |
| --- | --- |
| `3-1-server-architecture.md` | 본 데이터 모델이 사용되는 시스템 구조 (실시간 vs 배치 경로) |
| `3-2-directory.md` | 본 도메인 그룹 → `sttak-domain` 패키지/모듈 매핑 |
| `3-3-contract.md` | 본 모델이 외부로 노출되는 API 시그니처 |
| `5-2-error-handling.md` | 제약 위반/무결성 오류 → `ErrorType` ↔ HTTP 매핑 |
| `7-deployment.md` | DB 마이그레이션 도구/절차 |
