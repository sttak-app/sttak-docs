# AI-4-1. 구현 — 가드레일 파이프라인

> 코드 위치: `sttak-external/src/main/java/com/sttak/sttakexternal/support/guardrail/`
> 결정 기록: ADR-032. 실측 산출물: 노션 "가드레일 A/B 검수" 3부작.

## AI-4-1.1 구성 요소

| 클래스 | 역할 |
| --- | --- |
| `GuardrailPipeline` | 오케스트레이션 — 모드 분기, 판정, 피드백 루프, 소진 처리 |
| `GuardrailRewriter` / `GuardrailJudge` | 포트 인터페이스 (파이프라인은 벤더를 모름) |
| `OpenAiGuardrailRewriteAdapter` | 재작성 호출. 실패 시 원문 유지. 마크다운 펜스 스트립 |
| `OpenAiGuardrailJudgeAdapter` | 판정 호출. strict json_schema `{pass, violations[{category,quote}]}`. 불능 시 통과, refusal 은 반려 |
| `GuardrailPrompts` | 프롬프트 단일 출처 — 재작성 코어·판정 기준·기능별 형식 슬롯 (`AI-4-2`) |
| `GuardrailProperties` | `sttak.guardrail.*` — mode / max-feedback-retries(2) / api-key / model / max-tokens |
| `InvestmentGuardrail` (support/) | 룰 후보 탐지 엔진 — BASE_PATTERNS 23 + 기능별 extraPatterns, 예외 4규칙 |

## AI-4-1.2 핵심 흐름 (실코드)

```java
public Outcome apply(String jobTag, String text, String formatSlot, List<Pattern> extraPatterns) {

    if (properties.isConditionalMode()) {                       // 비용 절감 모드
        GuardrailVerdict verdict = judgeOf(text, extraPatterns);
        if (verdict.pass()) {
            return new Outcome(text, true, 0, List.of());
        }
        return rewriteLoop(jobTag, text, formatSlot, extraPatterns, verdict.violations(), 0);
    }
    String rewritten = rewriter.rewrite(text, formatSlot, List.of());   // 기본: 무조건 재작성
    GuardrailVerdict verdict = judgeOf(rewritten, extraPatterns);
    if (verdict.pass()) {
        return new Outcome(rewritten, true, 1, List.of());
    }
    return rewriteLoop(jobTag, rewritten, formatSlot, extraPatterns, verdict.violations(), 1);
}

private GuardrailVerdict judgeOf(String text, List<Pattern> extraPatterns) {
    // 룰 엔진 검출은 최종 판정이 아니라 판정 프롬프트에 싣는 후보 힌트다
    return judge.judge(text, InvestmentGuardrail.detect(text, extraPatterns));
}
```

소진 시 (`rewriteLoop` 종료):

```java
log.warn("가드레일 재작성 소진 job={} 시도={}회 잔존위반={} → 폴백", jobTag, rewriteCount, lastViolations);
meterRegistry.counter("sttak.ai.guardrail.exhausted", "job", jobTag).increment();
return new Outcome(current, false, rewriteCount, lastViolations);   // passed=false → 호출자가 폴백
```

## AI-4-1.3 룰 엔진 — 후보 탐지의 설계

부분 문자열 방식(구 `containsBanned`)은 정상 834건 중 68건을 오탐했다(회고 59·차트 9 —
"다음에는 매수 전에", "곧 반등한다는 뜻이 아니므로"). 개선 엔진 `detect()`:

1. **종결형·행위형만 매칭** — `(매수|매도)\s*하(세요|시죠|십시오|…)` 처럼 권유로 완성된
   형태만. 명사적 언급("매수 전에")은 아예 매칭하지 않는다.
2. **예외 4규칙** — 매치돼도 다음이면 제외: ① 직후 20자 내 부정 표지 ② 직후 인용·명사화
   표지("~라는/~라고") ③ 직전 여는 따옴표 ④ 같은 문장에 학습·개념 표지.
3. 결과는 최종 판정이 아니라 **후보** — LLM 판정에 "오탐일 수 있음" 단서와 함께 전달된다.

검증: 같은 834건에서 오탐 0, 합성 위반 24/24 검출, 안전 문장 16/16 통과.

## AI-4-1.4 판정 어댑터 — 불능 처리의 경계

```java
if (message.refusal() != null && !message.refusal().isBlank()) {
    log.warn("가드레일 판정 거부(refusal) — 반려로 처리");     // fail-closed
    refusals.increment();
    return new GuardrailVerdict(false, List.of(new GuardrailViolation("판정 거부", "")));
}
if (message.content() == null) {
    log.warn("가드레일 판정 응답 없음 — 통과로 처리");          // fail-open
    return GuardrailVerdict.passed();
}
```

- strict 스키마가 **형식**을 잠그지만 pass **값**은 모델 판단 — 그래서 "확신 없으면 통과"
  원칙을 판정 프롬프트에 명시하고(오탐 0/250 실측), refusal 만 예외로 반려한다.

## AI-4-1.5 알려진 한계 (의도적 수용)

| 한계 | 수용 이유 | 대응 계획 |
| --- | --- | --- |
| 판정·재작성 프롬프트에 심사 대상이 펜싱 없이 인라인 | 1차 생성 주입 방어 실측 0/24 + refusal 반려 + 코드 잠금 3종으로 위험도 낮음 | `<content>` 펜싱 — 실측 세트 재검증과 묶어 후속 |
| 코어 11범주 중 "종목 비교 우위 단정"은 룰 패턴화 불가 | 재작성·판정 프롬프트가 커버 | 필요 시 주기적 LLM 샘플 감사 |
| 퀴즈 잠금이 correctChoice 번호만 비교 | 현재 단건 생성(count=1)이라 순서 뒤바뀜 실해 없음 | 다건 생성 도입 시 정체 키 추가 |
