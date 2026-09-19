# AI-6. 관측성 (AI 기능)

> 서비스 전체 관측성(로그/`X-Request-Id`/Prometheus 스택)은 `../6-observability.md` 가 원본.
> 여기서는 AI 기능 특유의 세 층 — **건별 트레이스(Langfuse) · 집계 메트릭(Micrometer) ·
> 로그** — 를 정의한다. 근거: ADR-033/034 (Spring AI 전환), ADR-022 (Prometheus).

## AI-6.1 관측의 세 층 — 무엇을 어디서 보나

| 질문 | 층 | 도구 |
| --- | --- | --- |
| "이 콘텐츠가 왜 이렇게 나왔지?" (건별 추적) | 트레이스 | **Langfuse** — 프롬프트·응답·단계 타임라인 |
| "비용·양이 얼마나 되지?" (집계) | 메트릭 | Micrometer → Prometheus/Grafana (관리 포트 9000, ADR-022) |
| "뭐가 실패했지?" (사건) | 로그 | log.warn + Sentry (배치 MDC jobName 태그) |

세 층은 대체가 아니라 보완이다 — 메트릭이 "소진이 3건 있었다"를 알려주면, Langfuse 에서
그 3건의 재작성·판정 내용을 연다.

## AI-6.2 Langfuse 트레이스 (ADR-033/034)

**경로**: Spring AI GenAI Observation → Micrometer Observation → OTel 브리지 → OTLP exporter
→ Langfuse. 어댑터 코드는 span 을 직접 만들지 않는다 — `OpenAiChatModelSupport` 가
ObservationRegistry 를 배선하면 호출마다 자동 발행된다.

**무엇이 뜨나** — 배치 LLM 호출 전부:

| span | 발생 지점 |
| --- | --- |
| 생성 (기능 5종) | 뉴스 요약·판단 각 트랙, 퀴즈, 차트 해설, 회고 |
| 재작성 / 판정 | 가드레일 파이프라인 — 피드백 루프 반복까지 건별로 보임 |
| 임베딩 | 퀴즈 유사도 + 지식/챗봇 RAG (공용 EmbeddingModel) |
| 벡터 검색 | 퀴즈 유사도 검색 — 질의/결과를 span input/output 으로 승격 (`VectorStoreContentObservationFilter`) |

**안 뜨는 것 (알려진 공백)**: 챗봇 스트리밍(`OpenAiChatStreamAdapter`) — SSE+스트리밍 필터의
리액티브 재작성 위험 때문에 Spring AI 전환에서 의도적으로 제외(ADR-034). 챗봇 관측은 RAG
개편의 단계별 로깅 설계(query → BM25/kNN/RRF/rerank 청크 → 생성 출력 → 토큰·지연)와 함께
수동 Observation 으로 별도 정의한다.

**프로파일 정책** (`spring.ai.chat.observations.*`):

| 프로파일 | log-prompt / log-completion | 이유 |
| --- | --- | --- |
| local / dev | true — 프롬프트·응답 본문 포함 | 디버깅·품질 검수용 |
| **prod** | **false — 메타(모델·토큰·지연)만** | 사용자 유래 입력(rationale)·생성 콘텐츠를 외부 SaaS 에 남기지 않는다. 필요 시 한시적 true 전환 |

**설정·시크릿**: `LANGFUSE_OTLP_ENDPOINT`(기본 cloud.langfuse.com OTLP) ·
`LANGFUSE_OTLP_AUTH`(Basic 인증 — Secrets Manager, NFR-S1) ·
`LANGFUSE_TRACING_SAMPLING`(기본 1.0 — 물량이 배치 수백 건/일 수준이라 전수 수집, 비용 문제
시 하향). 미주입 시 전송만 실패하고 기능은 무영향(관측은 기능 가용성을 깨지 않는다).

## AI-6.3 메트릭 분류 (AI 기능분)

`../6-observability.md §6.5` 전체 표의 AI 부분. 전부 counter, 관리 포트 9000 으로 노출.

| 메트릭 | 태그 | 의미 |
| --- | --- | --- |
| `sttak.ai.tokens.input` / `.output` | `provider`, `job` | LLM 토큰 (비용 추적, NFR-O3). job 으로 기능·가드레일 단계 구분 |
| `sttak.ai.guardrail.exhausted` | `job` | 재작성 소진 → 폴백 발생. **경보 후보 1순위** — 위반 잔존 신호 |
| `sttak.ai.guardrail.judge.refusal` | — | 판정 모델 refusal → 반려. 입력 이상 신호 — 급증 시 주입 시도·데이터 오염 점검 |
| `sttak.quiz.saved` / `.rejected` / `.slot.skipped` | — | 퀴즈 저장/유사도 반려/슬롯 포기 |
| `sttak.news.collected` / `.deduped` / `.saved` | — | 뉴스 수집 파이프라인 (ADR-014) |

**job 태그 값** (= Langfuse span 식별자와 동일 체계):
`news-summary` · `news-sentiment` · `quiz-generation` · `chart-signal-explanation` ·
`trade-retrospective` · `guardrail-rewrite` · `guardrail-judge`

## AI-6.4 비용 관측 (AI-NFR5 의 실행법)

1. `sttak.ai.tokens.*` 를 job 별로 1~2주 집계 (Grafana — input/output 분리, 단가 곱해 비용 환산)
2. 구성비 확인: 가드레일 단계(rewrite+judge)가 전체의 어느 비중인지 — 설계 추정은 생성 대비
   ~60% 토큰(ADR-032)
3. 상한(AI-NFR5) 대비 과다가 지속되면 팀 논의 — 비용 절감 방안은 그때 결정한다

## AI-6.5 경보 (제안 — 임계·채널은 팀 논의로 확정)

| 신호 | 조건(안) | 의미 |
| --- | --- | --- |
| `sttak.ai.guardrail.exhausted` | 1건 이상/일 | 폴백 발생 — 프롬프트·판정 기준의 회귀 가능성. Langfuse 로 해당 건 추적 |
| `sttak.ai.guardrail.judge.refusal` | 급증 (기준선 대비) | 입력 이상 — rationale 주입 시도 등 |
| `sttak.ai.tokens.*` 일 합계 | 추정치(파이프라인 +690 + 뉴스 분리 +555 호출/일) 대비 급증 | 루프 폭주·재시도 폭증 |
| 배치 잡 실패 | 기존 Jenkins Slack 경보(SCRUM-87) | LLM 장애로 인한 잡 레벨 실패 |

침묵 감지(잡이 아예 안 도는 것)는 별도 팀 논의 사안 — 본 문서 범위 밖.

## AI-6.6 로그 (5-2 와의 경계)

- 실패 로그 규약(식별자 중심·본문 미기록·키 금지)은 `5-2-error-handling.md §AI-5-2.4` 가 원본
- 관측 층위 정리: **본문이 필요한 조사는 Langfuse(local/dev)** 로, 로그는 식별자·원인까지만 —
  같은 정보를 두 곳에 남기지 않는다
- 배치 로그는 MDC `jobName`/`jobExecutionId` 로 Sentry 태그 승격 (`../6-observability.md §6.4`)

## AI-6.7 후속 (미확정)

- 경보 임계·채널 확정 (§AI-6.5 는 제안 상태)
- 챗봇 관측 설계 — RAG 개편(ES 전환)과 세트: 단계별 청크·지연 로깅, 수동 Observation 배선
- PG↔ES 드리프트 메트릭 — ES 전환 시 신설 (체크리스트 참조)
- Langfuse self-host 여부 — 현재 cloud SaaS, prod 본문 미기록으로 리스크 완화 중. 본문까지
  남기려면 self-host 검토
