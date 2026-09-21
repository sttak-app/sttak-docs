# AF-4-3. 구현: 기능별 파이프라인 연결

이 문서는 세 기능의 어댑터가 검증 파이프라인(`4-1`)을 호출하는 방법과 보호할 값, 실패 처리
방법을 설명한다. 어댑터는 모두 `sttak-external`에 있다. 전송에는 Spring AI의
ChatModel과 EmbeddingModel을 사용하며, 경로·타임아웃·재시도·관측 설정은
`OpenAiChatModelSupport`에서 공통으로 관리한다.

## AF-4-3.1 실제 코드의 공통 연결 방식

```java
// OpenAiTradeRetrospectiveAdapter.generate()
String json = callOpenAi(candidate);                          // ① 1차 생성 (strict schema)
GuardrailPipeline.Outcome outcome = guardrailPipeline.apply(
        JOB, json, GuardrailPrompts.RETRO_SLOT, TradeRetrospectiveSupport.EXTRA_PATTERNS);
if (!outcome.passed()) {                                      // ② 소진 → 폴백
    throw new BusinessException(TradeException.REVIEW_GENERATION_FAILED);
}
RetrospectiveResult original  = objectMapper.readValue(json, ...);          // ③ 재파싱
RetrospectiveResult rewritten = objectMapper.readValue(outcome.text(), ...);
requireSameEvaluations(original, rewritten, candidate);       // ④ 코드 잠금
return TradeRetrospectiveSupport.toGeneratedRetrospective(rewritten, model);
```

위 코드는 다음 순서로 동작한다.

1. strict 스키마를 적용해 첫 결과를 생성한다.
2. `apply(jobTag, text, slot, extraPatterns)`로 재생성과 검수를 실행한다.
3. 재생성된 JSON을 다시 파싱한다. JSON 구조가 깨졌다면 이 단계에서 실패한다. 재생성
   어댑터는 흔한 오류를 줄이기 위해 마크다운 코드 펜스를 먼저 제거한다.
4. 잠금 값이 원본과 같은지 확인한 뒤 도메인 객체로 변환한다.

## AF-4-3.2 기능별 차이

| 항목 | 퀴즈 | 차트 해설 | 회고 |
| --- | --- | --- | --- |
| 어댑터 | `OpenAiQuizGenerationAdapter` | `OpenAiChartSignalExplanationAdapter` | `OpenAiTradeRetrospectiveAdapter` |
| jobTag | `quiz-generation` | `chart-signal-explanation` | `trade-retrospective` |
| 형식 슬롯 | `QUIZ_SLOT` | `CHART_SLOT` | `RETRO_SLOT` |
| 기능별 후보 패턴 | 없음 | "진입 기회", "지금이 저점"처럼 신호를 매매 시점으로 받아들이게 하는 표현 5개 | "다음에는 매수하세요" 같은 직접 권유 표현 1개. 코칭 목적의 질문은 통과 |
| 파이프라인 입력 | 문항 JSON | 해설 텍스트 (JSON 아님) | 회고 JSON |
| 코드 잠금 | `correctChoice`(`requireSameAnswers`) | 구조화된 값 없음 | `evaluations`의 개수와 `type` 순서(`requireSameEvaluations`) |
| 실패 시 처리 | 문항을 버리고 최대 3회 재생성 | 저장하지 않고 다음 배치에서 재시도 | 저장하지 않고 다음 정산 배치에서 재시도 |
| 후처리 | 공백 정리만 | strip | dropBlank |
| 배치 진입점 | `QuizGenerationTasklet`(일 배치) | `ChartExplanationTasklet`(신호 감지 후속 단계) | `TradeReviewTasklet`(정산 후속 단계) |

- 과거에 사용하던 금지어 치환·차단 후처리(`sanitizeOrReject`, `isBanned` 등)는 모두
  제거했다. 표현 검증은 공통 파이프라인에서만 처리한다.
- 배치 단계는 Port를 `ObjectProvider`로 주입받는다. 구현이 없는 환경에서는 해당 단계만
  건너뛰고 수집·감지·정산은 계속 실행한다.

## AF-4-3.3 잘못된 차트 파라미터 생성 방지

프롬프트에 넣는 신호 설명은 Map 대신 **switch 식**으로 정의한다.

```java
static String describe(ChartSignalKind kind) {
    return switch (kind) {
        case GOLDEN_CROSS -> "골든크로스 (5일 이동평균선이 20일 이동평균선을 상향 돌파)";
        case RSI_OVERSOLD -> "RSI 과매도 (RSI 14가 30을 하향 돌파 — 과매도 구간 진입)";
        // … 8종 전부. 새 신호 종류를 추가하면 컴파일 에러로 설명문 작성이 강제된다
    };
}
```

초기 실험에서는 모델이 실제 감지 조건과 다른 20일선·200일선 조합을 만들어 내는 문제가
있었다. 실제 파라미터를 담은 고정 설명문을 프롬프트에 넣어 해결했다. Map을 사용하면 새
신호를 추가할 때 설명문이 빠져 "신호: null"이 전달될 수 있다. switch 식을 사용하면 모든
신호를 처리하지 않은 코드가 컴파일되지 않으므로 설명문 누락을 미리 막을 수 있다.

## AF-4-3.4 퀴즈에 추가로 적용하는 품질 검사

공통 파이프라인의 표현 검증을 마친 뒤 퀴즈에 필요한 검사를 추가로 수행한다.

1. **구조 검증**: 선지가 4개인지, 정답 번호가 1부터 4 사이인지, 해설이 있는지 확인한다.
   조건을 만족하지 않는 문항은 `toGeneratedQuizzes`에서 제외한다.
2. **유사도 중복 검사**: `text-embedding-3-small`로 새 문항을 임베딩하고 기존 문항과 코사인
   유사도를 비교한다. **0.80을 넘으면 해당 문항을 버리고 문항당 최대 3회 다시 생성한다.**
   벡터는 Spring AI VectorStore 전용 `quiz_vector` 테이블에 저장한다. 도메인은
   `QuizSimilarityPort`만 참조하므로 벡터 저장소가 바뀌어도 구현체만 교체하면 된다.
3. **회피 목록**: 직전에 만든 문항을 사용자 프롬프트의 "반드시 회피" 영역에 넣어 같은 생성
   회차에서 중복이 생길 가능성을 줄인다.
