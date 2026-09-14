# AI-5-1. 테스트 (AI 기능)

> 테스트 레이어·Fixture·Testcontainers 규약은 `../5-1-test-case.md` 와 루트 `TESTING.md` 가 원본.
> 여기서는 AI 기능 특유의 테스트 전략 — "LLM 은 목으로, 판단 로직은 실코드로" — 를 정의한다.

## AI-5-1.1 원칙

1. **테스트는 LLM 을 호출하지 않는다.** HTTP 는 `MockRestServiceServer`, 파이프라인 협력자는
   함수형 페이크(`(text, slot, feedback) -> ...`)로 대체한다. LLM 품질 자체의 검증은 테스트가
   아니라 평가(`AI-4-4`)의 몫이다 — 둘을 섞지 않는다.
2. **실측에서 발견된 사례는 테스트로 이식한다.** 룰 엔진 오탐 5유형·경계 문장이 대표 —
   회귀를 CI 가 막는다.
3. **프롬프트는 존재·핵심 조항만 검증한다** (`contains`/`doesNotContain`). 전문 스냅샷 비교는
   문구 개선을 막으므로 하지 않는다.

## AI-5-1.2 레이어별 대상

| 레이어 | 대상 | 방식 | 대표 케이스 |
| --- | --- | --- | --- |
| 단위 (external) | `InvestmentGuardrail` | 순수 함수 | 32케이스 — 종결형 검출, 예외 4규칙(부정·인용·따옴표·교육), 실측 오탐 문장 통과 |
| 단위 (external) | `GuardrailPipeline` | 페이크 rewriter/judge | 15케이스 — always/conditional 분기, 피드백 루프, 소진 시 passed=false + 카운터, 판정 불능 통과 |
| 단위 (external) | 판정/재작성 어댑터 | MockRestServiceServer | strict 스키마 파싱, 힌트 전달, 응답 불능 시 원문 유지/통과, **refusal 반려**, 펜스 스트립 |
| 단위 (external) | 기능 어댑터 4종 | Mock HTTP + 페이크 파이프라인 | 통과본 매핑, 소진 시 기능 예외, JSON 깨짐 반려, **잠금 위반 반려**(라벨 반전·정답 변경·평가 구조 변경) |
| 단위 (external) | `<기능>Support` | 순수 함수 | 프롬프트 v2 조항, describe() 8종 비지 않음, sanitize 공백 정리만 |
| 단위 (batch) | Tasklet/Processor | Mockito | Port 부재 시 스킵, 개별 실패 흡수, DuplicateKey 흡수 |
| 통합 (batch) | 잡 E2E | `@SpringBootTest` + `@MockitoBean`(AI Port 스텁) + Testcontainers | 시드 → Reader→Processor→Writer → 상태 전이. 실 LLM 어댑터가 컨텍스트에 남지 않게 항상 mock |

## AI-5-1.3 잠금 검증 테스트 — 반드시 있어야 하는 것

이중 잠금(AI-FR9)은 기능마다 "재작성이 값을 바꾸는 페이크"로 반려를 확인한다:

```java
// 회고 예 — type 반전(GOOD→BAD)은 표현 재작성 범위를 벗어난 구조 변경
GuardrailRewriter rewriter = (text, slot, feedback) ->
        text.replace("\"type\":\"GOOD\"", "\"type\":\"BAD\"");
assertThatThrownBy(() -> target.generate(candidate()))
        .isInstanceOf(BusinessException.class);
```

같은 패턴: 뉴스(sentiment 또는 stockCode 변경), 퀴즈(correctChoice 변경). 새 기능이 파이프라인에
합류하면 이 3종 세트(통과/소진/잠금 위반)가 최소 요건이다.

## AI-5-1.4 하지 않는 것

- 실 API 호출 테스트 (비용·비결정성·키 노출) — CI 에서 키는 존재하지 않는 게 정상.
- LLM 출력 품질의 assert (요약이 좋은지, 판정이 옳은지) — 평가 파이프라인 소관.
- 프롬프트 전문 고정 — 조항 존재 검증까지만.
