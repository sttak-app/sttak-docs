# AI-4-3. 구현 — 기능별 파이프라인 연결

> 각 기능 어댑터가 파이프라인을 어떻게 호출하고, 무엇을 잠그고, 실패를 어떻게 처리하는지.
> 공통 패턴을 먼저 정의하고 기능별 차이만 표로 남긴다.

## AI-4-3.1 공통 연결 패턴

```java
// OpenAiTradeRetrospectiveAdapter.generate() — 회고 예시 (다른 기능도 동형)
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

패턴 요소: ① 생성 → ② `apply(jobTag, text, slot, extraPatterns)` → ③ JSON 재파싱(재작성이
JSON 을 깨면 여기서 실패) → ④ 원본 대비 잠금 검증 → 도메인 매핑. 재작성 어댑터의 펜스
스트립이 ③의 흔한 실패(``` 감싸기)를 사전 차단한다.

## AI-4-3.2 기능별 차이

| 항목 | 뉴스 요약 | 뉴스 판단 | 퀴즈 | 차트 해설 | 회고 |
| --- | --- | --- | --- | --- | --- |
| jobTag | `news-summary` | `news-sentiment` | `quiz-generation` | `chart-signal-explanation` | `trade-retrospective` |
| 슬롯 | `SUMMARY_SLOT` | `SENTIMENT_SLOT` | `QUIZ_SLOT` | `CHART_SLOT` | `RETRO_SLOT` |
| extraPatterns | 없음 | 없음 | 없음 | 5패턴 (진입 기회/지금이 저점…) | 1패턴 (역방향 권유 — 코칭형은 통과) |
| 파이프라인 입력 | 요약 JSON (4필드) | 판단 JSON (perStock) | 문항 JSON | 해설 텍스트 (JSON 아님) | 회고 JSON |
| 코드 잠금 | — (텍스트) | stockCode+sentiment | correctChoice | — | evaluations 개수+type |
| 폴백 | 상태 유지 → 재시도 (기사 단위) | 〃 (기사 단위 공유) | 문항 폐기 → 재시도 ≤3 | 저장 안 함 → 다음 배치 | 저장 안 함 → 다음 정산 |
| 후처리 sanitize | 공백 정리만 (`\s{2,}`→' ') | 공백 정리만 | 공백 정리만 | strip | dropBlank |

- 구 금지어 치환/차단 후처리(sanitize 치환, `isBanned`, `sanitizeOrReject`)는 전부 제거됐다 —
  가드레일은 파이프라인 전담. `InvestmentGuardrail` 의 부분 문자열 경로는 챗봇(ChatGuardrail)만
  남아 @Deprecated (챗봇 파이프라인 합류 시 제거).

## AI-4-3.3 차트 해설 — 파라미터 날조 차단 장치

프롬프트 입력의 신호 설명을 Map 이 아닌 **switch 식**으로 고정한다:

```java
static String describe(ChartSignalKind kind) {
    return switch (kind) {
        case GOLDEN_CROSS -> "골든크로스 (5일 이동평균선이 20일 이동평균선을 상향 돌파)";
        case RSI_OVERSOLD -> "RSI 과매도 (RSI 14가 30을 하향 돌파 — 과매도 구간 진입)";
        // … 8종 전부. 새 신호 종류 추가 시 컴파일 에러로 설명문 작성을 강제
    };
}
```

- 실험 1라운드에서 모델이 임의 파라미터(20/200일선)를 지어내는 결함 발견 → 실제 감지
  파라미터를 명시한 고정 설명문 주입으로 해결. 누락 시 "신호: null" 프롬프트로 엉뚱한
  해설이 영구 저장되는 함정을 컴파일러가 막는다 (VOLUME_SPIKE 2차 대비).

## AI-4-3.4 뉴스 — 기능 2개, 트랙 2개, 저장 1건

요약과 호재/악재 판단은 어댑터 1개 안의 **독립 트랙 2개**다 (AI-FR2b, PR #53):

```java
public NewsAnalysis analyze(News news) {
    SummaryResult summary = generateSummary(news);      // v2 생성 → SUMMARY_SLOT 파이프라인
    SentimentResult sentiment = generateSentiment(news); // v4 생성 → SENTIMENT_SLOT 파이프라인 → 라벨 잠금
    return NewsAnalysisSupport.toNewsAnalysis(summary, sentiment);   // 병합 (기계적 조립)
}
```

- **왜 분리인가**: 2차 평가가 이 구성(v2/v4 단독 프롬프트 + 슬롯별 트랙)으로 실측됐다.
  운영 반영 때 병합됐던 것은 미검증 델타였고 v4 조항 3블록 누락까지 있었다 — 분리는
  검증-운영 정합의 복원이다. 교차 모순 우려는 스모크 100건 전수 검수 0건으로 해소.
- 요약 트랙 유저 프롬프트에는 종목코드를 넣지 않는다(역할 분리). 판단 트랙만 종목코드 포함.
- 실패·재시도는 기사 단위 — 한 트랙 실패 = 기사째 `NEWS_ANALYSIS_FAILED`, 부분 성공 없음.
- perStock 은 '분석 대상 종목코드'로 주어진 모든 종목에 대해 1항목씩 강제 — 잠금 검증도
  이 목록 기준으로 원본과 대조한다.
- 비용: 기사당 6호출(트랙 2 × 생성·재작성·판정) — 뉴스 일 ~185건 기준 통합 대비 +555호출/일.

## AI-4-3.5 퀴즈 — 파이프라인 밖의 품질 장치

파이프라인(표현)과 별개로 퀴즈 고유 검증이 이어진다 (ADR-019):
1. 구조 검증 — 4선지·정답 1..4·해설 존재 (`toGeneratedQuizzes` 가 무효 문항 드랍)
2. 임베딩 유사도 0.80 초과 시 중복 폐기 → 재생성 (문항당 최대 3회)
3. 회피 목록 — 직전 생성 문항을 userPrompt "반드시 회피" 섹션에 나열
