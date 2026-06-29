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
| 사용자/계좌 | `User`, `Account` |
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

- `User` — PK `id` (UUID), 인증 정보(`email`, `password`, `phone_number`), `role`
- `Account` — PK `acount_id`, `id*` → `User.id`, `cash_balance`, `total_asset`, `account_status`

관계:

- `User 1 ──< N Account` (`Account.id` → `User.id`)

### 3-4.2.2 시장/종목

- `Market` — PK `market_id`, `country_code`, `currency_code`, `market_name`, `market_code`
- `MarketSessions` — PK `session_id`, `market_id*` → `Market`, `session_type`, `open_time`, `close_time`, `tradable`
- `Stock` — PK `stokc_id`, `market_id2*` → `Market.market_id`, `stock_code`, `stock_name`, `listed_date`, `stock_status`
- `Sector` — PK `sector_id`, `sector_name`
- `Stock_Sector` — PK `stock_sector_id`, `stokc_id*` → `Stock`, `sector_id2*` → `Sector` (M:N 매핑 테이블)

관계:

- `Market 1 ──< N MarketSessions`
- `Market 1 ──< N Stock`
- `Stock N >──< N Sector` (via `Stock_Sector`)

### 3-4.2.3 주문/보유/관심

- `Order` — PK `order_id`, `acount_id*` → `Account`, `stokc_id*` → `Stock`, `order_side`(ENUM), `order_price`, `order_quantity`, `order_status`(ENUM), `order_at`
- `Holdings` — PK `holdings_id`, `acount_id*` → `Account`, `stokc_id*` → `Stock`, `quantity`, `average_price`, `total_buy_amount`
- `Watchlist` — PK `watchlist_id`, `id2*` → `User.id`, `stokc_id*` → `Stock`, `is_deleted`, `deleted_at` (soft delete)

관계:

- `Account 1 ──< N Order`, `Account 1 ──< N Holdings`
- `Stock 1 ──< N Order`, `Stock 1 ──< N Holdings`, `Stock 1 ──< N Watchlist`
- `User 1 ──< N Watchlist`

### 3-4.2.4 주문 회고

- `OrderReview` — PK `review_id`, `order_id*` → `Order` (1:1), `AI_feedback`, `profit_loss_rate`, `proofit_loss_amount`
- `OrderReason` — PK `order_reason_id`, `order_id*` → `Order`, `reason_template_id*` → `ReasonTemplate` (nullable: 자유 입력 허용 가정), `reason_content`
- `ReasonTemplate` — PK `reason_template_id`, `reason_type`(ENUM), `reason_label`
- `ReasonEvaluation` — PK `reason_evaluation_id`, `review_id*` → `OrderReview`, `order_reason_id*` → `OrderReason`, `model_name`, `evaluation_type`(ENUM), `evaluation_content`

관계:

- `Order 1 ── 1 OrderReview`
- `Order 1 ──< N OrderReason`, `ReasonTemplate 1 ──< N OrderReason`
- `OrderReview 1 ──< N ReasonEvaluation`, `OrderReason 1 ──< N ReasonEvaluation`

### 3-4.2.5 차트/시그널

- `Stock_Candle` — PK `candle_id`, `stokc_id*` → `Stock`, OHLCV(`open`/`high`/`low`/`close`/`volume`), `market_date`
- `Chart_Terms` — PK `chart_term_id`, `term_name`(ENUM), `easy_meaning`, `detail_meaning`
- `Stock_Chart_Signals` — PK `chart_signal_id`, `candle_id*` → `Stock_Candle`, `chart_term_id*` → `Chart_Terms` (특정 캔들에 어떤 차트 용어가 잡혔는지)
- `Chart_Signal_Explanations` — PK `chart_signal_explanation_id`, `chart_signal_id*` → `Stock_Chart_Signals`, `exlpanation`, `Field`

관계:

- `Stock 1 ──< N Stock_Candle`
- `Stock_Candle 1 ──< N Stock_Chart_Signals`
- `Chart_Terms 1 ──< N Stock_Chart_Signals`
- `Stock_Chart_Signals 1 ──< N Chart_Signal_Explanations`

### 3-4.2.6 뉴스/용어

- `News` — PK `news_id`, `original_title/content/link`, `news_summary`, `easy_content`, `source_name`, `published_at`
- `term` — PK `term_id`, `term_name`, `term_meaning` (일반 경제 용어 사전)
- `News_Stock` — PK `news_stock_id`, `news_id*` → `News`, `stokc_id*` → `Stock`, `factor`(ENUM, 호재/악재/중립 등 추정) — 뉴스 ↔ 종목 M:N
- `News_Term` — PK `news_term_id`, `news_id*` → `News`, `term_id*` → `term` — 뉴스 ↔ 용어 M:N

관계:

- `News N >──< N Stock` (via `News_Stock`, with `factor`)
- `News N >──< N term` (via `News_Term`)

### 3-4.2.7 퀴즈

- `Quiz` — PK `quiz_id`, `quiz_content`, `choice_a/b/c/d`, `correct_choice`(INT), `explanation`, `point`
- `Quiz_User` — PK `quiz_user_id`, `id*` → `User.id`, `quiz_id*` → `Quiz`, `start_at`, `end_at`, `is_solved`(ENUM), `selected_choice`

관계:

- `User N >──< N Quiz` (via `Quiz_User`)

### 3-4.2.8 챗봇

- `ChatBot` — PK `Key` (VARCHAR(255)) — 현재 컬럼 1개만 정의된 스텁. 설계 미완.

---

## 3-4.3 카디널리티 한눈에 보기

```
                ┌──────┐
                │ User │
                └──┬───┘
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
| `Account`, `Order`, `Holdings` | `acount_id` | `account_id` |
| `Order`, `Holdings`, `Stock`, `Watchlist`, `News_Stock`, `Stock_Sector`, `Stock_Candle` | `stokc_id` | `stock_id` |
| `Chart_Signal_Explanations` | `exlpanation` | `explanation` |
| `Holdings` | `upldated_at` | `updated_at` |
| `OrderReview` | `proofit_loss_amount` | `profit_loss_amount` |
| `OrderReason` | `created_at DATTETIME` | `DATETIME` |
| `term` | `created_at DATE_TIME` | `DATETIME` |

### B. 명명 일관성

- ERDCloud 자동 생성으로 보이는 `2` 접미사 (`market_id2`, `sector_id2`, `id2`) 제거 — 각각 `market_id` / `sector_id` / `user_id` 로 정리.
- 테이블명 컨벤션 결정: snake_case + 단수형 (`order`, `order_review`, `chart_term`) 권장. 현재 `Order`, `term`, `User`, `Stock_Chart_Signals`, `MarketSessions` 등이 혼재.
- `Watchlist.id2` 와 `Quiz_User.id` 와 `Account.id` 가 모두 `User.id`를 가리키는데 이름이 다르다 → `user_id`로 통일.
- `Chart_Signal_Explanations.Field` 의 대문자 시작 + `Field` 라는 모호한 이름 — 의미를 살린 이름으로 (예: `field_name`, `aspect`).

### C. PK 타입 일관성

- `ReasonEvaluation.reason_evaluation_id` 만 `VARCHAR(255)` — 나머지 PK는 모두 BIGINT. 의도된 ID 전략(예: 외부 모델 ID 그대로 사용)인지 확인.

### D. 길이/타입 누락

- `VARCHAR` 길이가 비어 있는 컬럼이 다수 (`User.email`, `User.password`, `User.phone_number`, `Market.market_name`, `Market.market_code`, `Stock.stock_code`, `Stock.stock_name`, `ReasonTemplate.reason_label`, `News.easy_content`, `News.source_name`, `ReasonEvaluation.model_name`) — MySQL에서는 길이 지정이 필요하다.
- `User.role`, `Market.country_code`, `Market.currency_code` 가 소문자 `enum` 으로 적혀 있음 — `ENUM(...)` 으로 정의 필요.
- `News.easy_content` 가 `VARCHAR` 인데, 의미상 가변 길이 본문이라면 `TEXT`가 적합.

### E. Enum 값 목록 미정

다음 `ENUM` 컬럼은 값 목록이 비어 있다. 각 값을 본 문서 또는 ADR로 명문화한다.

| 테이블 | 컬럼 | 추정 값(예시) |
| --- | --- | --- |
| `User` | `role` | `USER`, `ADMIN` |
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
  - `User.email` UNIQUE
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

- `User`/`Account.created_at` 만 `timestamp`, 나머지는 `DATETIME`. 한 가지로 통일 권장 (JPA Auditing 적용 시 `TIMESTAMP WITH TIME ZONE` 또는 `DATETIME(6)` 일관 사용).
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
