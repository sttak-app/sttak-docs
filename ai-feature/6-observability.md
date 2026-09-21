# AF-6. 관측성 (배치 AI 기능)

서비스 전체의 로그, 요청 ID, 메트릭 구성은 `../6-observability.md`를 따른다. AI 기능은
트레이스, 메트릭, 로그의 세 가지 방식으로 관측하며 각 방식은 서로 보완한다. 예를 들어
메트릭에서 재시도 횟수를 모두 사용한 건이 3개 보이면 트레이스에서 각 건의 재생성과 검수
과정을 확인한다.

| 질문 | 층 | 현재 사용 중인 도구 |
| --- | --- | --- |
| "이 콘텐츠가 왜 이렇게 나왔지?"(건별 추적) | 트레이스 | Langfuse에서 프롬프트, 응답, 단계별 처리 시간을 확인한다 |
| "비용과 처리량은 얼마나 되지?"(집계) | 메트릭 | Micrometer → Prometheus/Grafana |
| "어떤 작업이 실패했지?"(사건) | 로그·경보 | `log.warn` + Sentry(배치 MDC의 `jobName` 태그) |

## AF-6.1 Langfuse 트레이스 자동 기록

트레이스는 Spring AI GenAI Observation → Micrometer Observation → OTel 브리지 → OTLP →
Langfuse 순서로 전달된다. 어댑터에서 span을 직접 만들지 않는다.
`OpenAiChatModelSupport`와 `OpenAiEmbeddingModelSupport`가 `ObservationRegistry`를 연결하면
각 호출의 span이 자동으로 생성된다.

- **모든 LLM 호출을 건별 span으로 기록한다.** 기능별 생성, 재생성, 검수와 피드백 반복,
  임베딩, 퀴즈 유사도 벡터 검색을 모두 포함한다. 벡터 검색의 질문과 결과는
  `VectorStoreContentObservationFilter`가 기록한다. 가드레일 호출을 빼면 비용과 지연의
  상당 부분을 파악할 수 없으므로 예외 없이 기록한다.
- **본문 기록 정책**(`spring.ai.chat.observations.*`): local과 dev에서는 디버깅과 품질
  검수를 위해 프롬프트와 응답 본문을 기록한다. **prod에서는 모델, 토큰, 지연 시간 같은
  메타데이터만 기록한다.** 사용자가 작성한 `rationale`과 생성된 콘텐츠는 외부 SaaS에 남기지
  않는다. 본문 기록이 필요하면 제한된 기간에만 활성화한다.
- **설정·시크릿**: `LANGFUSE_OTLP_ENDPOINT` · `LANGFUSE_OTLP_AUTH`(Basic 인증, 시크릿
  저장소), `LANGFUSE_TRACING_SAMPLING`(기본 1.0)을 사용한다. 배치 처리량은 하루 수백 건
  수준이므로 모든 호출을 기록한다. 설정하지 않으면 트레이스 전송만 실패하며 기능 동작에는
  영향을 주지 않는다.

## AF-6.2 구현된 메트릭

| 메트릭 | 태그 | 의미 |
| --- | --- | --- |
| `sttak.ai.tokens.input` / `.output` | `provider`, `job` | LLM 토큰 사용량과 비용을 추적한다. `job`으로 기능과 가드레일 단계를 구분한다 |
| `sttak.ai.guardrail.exhausted` | `job` | 재생성 횟수를 모두 사용해 대체 처리가 실행된 횟수다. 위반이 끝까지 남았다는 뜻이므로 우선 확인한다 |
| `sttak.ai.guardrail.judge.refusal` | 없음 | 검수 refusal로 결과를 반려한 횟수다. 갑자기 늘면 프롬프트 주입 시도나 데이터 오염을 점검한다 |
| `sttak.quiz.saved` / `.rejected` / `.slot.skipped` | 없음 | 퀴즈 저장, 유사도 기준 초과로 반려, 생성 슬롯 포기 횟수를 각각 기록한다 |

**job 태그 값**(Langfuse span 식별자와 같은 체계): `quiz-generation` ·
`chart-signal-explanation` · `trade-retrospective` · `guardrail-rewrite` · `guardrail-judge`
(뉴스 작업은 `news-summary`와 `news-sentiment`를 사용한다. 자세한 내용은 `../ai-chatbot/6` 참고).

## AF-6.3 비용 관측

1. `sttak.ai.tokens.*`를 job별로 주기적으로 집계하고(Grafana, input/output 분리) 단가를 곱해
   비용으로 환산한다.
2. 전체 비용에서 검증 단계(`rewrite`+`judge`)가 차지하는 비율을 확인한다. 설계 단계에서는
   생성 토큰의 약 60%로 예상한다.
3. 상한(NFR2)보다 높은 사용량이 계속되면 팀에서 원인을 확인하고 절감 방안을 정한다.

## AF-6.4 경보 후보 (임계·채널은 팀 확정)

| 신호 | 조건(안) | 의미 |
| --- | --- | --- |
| `exhausted` 발생 | 하루 1건 이상 | 프롬프트나 판정 기준의 품질이 떨어졌을 수 있으므로 Langfuse에서 해당 건을 확인한다 |
| `refusal` 급증 | 기준선 대비 | 프롬프트 주입 시도 등 입력 이상 여부를 점검한다 |
| 일일 토큰 합계 급증 | 추정치 대비 | 반복 처리나 재시도가 비정상적으로 늘었는지 확인한다 |
| 배치 작업 실패 | 기존 배치 Slack 경보 | LLM 장애 등으로 배치 전체가 실패했는지 확인한다 |

## AF-6.5 로그와의 경계

- 식별자 중심 기록, 본문 미기록, 키 노출 금지 등 실패 로그 규칙은 `5-2`를 따른다.
- 본문이 필요한 조사는 트레이스(local/dev)로 하고, 같은 내용을 로그에 중복해서 남기지 않는다.
- 배치 로그는 MDC `jobName`/`jobExecutionId` 가 Sentry 태그로 승격된다.
