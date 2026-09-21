# AF-4-1. 구현: 공통 가드레일 파이프라인

모든 AI 결과물은 저장하거나 전송하기 전에 이 검증 파이프라인을 거친다. 뉴스와 챗봇
기능(`../ai-chatbot/`)도 같은 파이프라인을 사용한다. 현재 구현은 `sttak-external`의
`support/guardrail/` 패키지에 있으며, 이 문서는 해당 구현을 설명한다. 검증에 실패했을 때
결과를 통과시킬지 차단할지는 `5-2`에서 다룬다.

## AF-4-1.1 구현된 구성 요소

| 클래스 | 역할 |
| --- | --- |
| `GuardrailPipeline` | 재생성 호출, 검수, 피드백 반복, 재시도 횟수 소진 처리를 조율한다 |
| `GuardrailRewriter` / `GuardrailJudge` | 외부 AI 제공자를 감추는 포트 인터페이스다 |
| `OpenAiGuardrailRewriteAdapter` | 재생성 호출. 실패 시 원문 유지. 마크다운 펜스(```) 감싸기를 벗겨낸다 |
| `OpenAiGuardrailJudgeAdapter` | strict `json_schema`로 검수 결과를 받는다. 판정할 수 없으면 통과시키고 refusal은 반려한다 |
| `GuardrailPrompts` | 재생성 공통 문안, 검수 기준, 기능별 출력 형식을 한곳에서 관리한다(`4-2`) |
| `GuardrailProperties` | `sttak.guardrail.*` 설정을 관리한다. 피드백 재시도 횟수의 기본값은 2이며 API 키, 모델, 최대 토큰 수도 포함한다 |
| `InvestmentGuardrail` | 공통 패턴 23개와 기능별 추가 패턴, 네 가지 예외 규칙으로 의심 표현을 찾는다 |

## AF-4-1.2 실제 처리 흐름

```java
public Outcome apply(String jobTag, String text, String formatSlot, List<Pattern> extraPatterns) {
    String rewritten = rewriter.rewrite(text, formatSlot, List.of());   // 무조건 재생성 (단일 경로)
    GuardrailVerdict verdict = judgeOf(rewritten, extraPatterns);
    if (verdict.pass()) {
        return new Outcome(rewritten, true, 1, List.of());
    }
    return rewriteLoop(jobTag, rewritten, formatSlot, extraPatterns, verdict.violations(), 1);
}

private GuardrailVerdict judgeOf(String text, List<Pattern> extraPatterns) {
    // 정규식 검출은 최종 판정이 아니라 검수 프롬프트에 싣는 후보 힌트다
    return judge.judge(text, InvestmentGuardrail.detect(text, extraPatterns));
}
```

피드백을 반영한 재생성을 두 번까지 시도했는데도 위반이 남으면 `rewriteLoop`를 종료한다.

```java
log.warn("가드레일 재작성 소진 job={} 시도={}회 잔존위반={} → 폴백", jobTag, rewriteCount, lastViolations);
meterRegistry.counter("sttak.ai.guardrail.exhausted", "job", jobTag).increment();
return new Outcome(current, false, rewriteCount, lastViolations);   // passed=false → 호출자가 폴백
```

각 단계를 이 방식으로 구현한 근거는 다음과 같다. 자세한 평가 결과는 `4-4`에서 확인할 수 있다.

| 단계 | 담당 | 근거 |
| --- | --- | --- |
| 재생성 | LLM, **항상 수행** | 조건부 방식과 위반 제거율은 같았지만, 항상 재생성한 방식에서만 문체 개선 효과가 있었다. 150건 중 5건이 개선되고 2건이 나빠졌으며 라벨 변경은 없었다. 나빠진 2건은 프롬프트 보완으로 막았다 |
| 후보 탐지 | 정규식 | 패턴에 걸린 표현이 검수 대상에서 빠지지 않게 한다. 탐지 결과가 없어도 검수는 실행한다 |
| 검수 | LLM | 경계 문장 판별 정확도는 규칙 기반 방식이 50%, LLM이 86%였다. 정상 문장 250건에서는 오탐과 반복 판정의 번복이 없었다 |
| 구조 보호 | 코드 | 라벨, 정답, 평가 타입처럼 바뀌면 안 되는 값은 검수 후 코드에서 원본과 같은지 확인한다(`3-2`의 코드 잠금) |

## AF-4-1.3 후보 탐지는 검수를 돕는 힌트

부분 문자열 방식의 금지어 필터는 정상 결과물 834건 중 68건을 잘못 차단했다. 모두 "다음에는
매수 전에" 같은 교육 문장이나 "곧 반등한다는 뜻이 아니므로" 같은 부정 문장이었다. 이를
보완하기 위해 `detect()`는 다음 규칙을 사용한다.

1. **권유로 완성된 문장 끝 표현만 찾는다.** `(매수|매도)\s*하(세요|시죠|십시오|…)` 형태를
   대상으로 한다.
   명사적 언급("매수 전에")은 매칭하지 않는다.
2. **네 가지 예외를 적용한다.** 탐지된 표현 뒤 20자 안에 부정 표현이 있거나, 인용·명사화
   표현("~라는", "~라고")이 있거나, 바로 앞에 여는 따옴표가 있거나, 같은 문장에 학습·개념
   표현이 있으면 후보에서 제외한다.
3. 탐지 결과는 "오탐일 수 있음"이라는 안내와 함께 검수 프롬프트에 넣는다. LLM이 힌트만
   보고 위반으로 단정하지 않게 하기 위해서다.

같은 데이터로 다시 검증한 결과, 정상 문장 834건은 모두 통과했고 합성 위반 문장 24건은
모두 탐지했다. 별도로 준비한 안전 문장 16건도 모두 통과했다.

## AF-4-1.4 검수할 수 없을 때의 처리

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

strict 스키마는 응답 **형식**만 보장하며 `pass` 값은 모델이 판단한다. 따라서 검수
프롬프트에는 판단이 확실하지 않으면 통과시키도록 명시한다. 정상 문장 250건에서 오탐이
없었던 결과를 근거로 삼았으며, refusal만 예외로 반려한다.

## AF-4-1.5 현재 허용한 한계

| 한계 | 수용 이유 |
| --- | --- |
| 검수·재생성 프롬프트에서 검사할 텍스트를 `<content>` 같은 태그로 감싸지 않는다 | 프롬프트 주입 실험 24건에서 우회 사례가 없었고 refusal 반려와 코드 잠금도 함께 적용하므로 현재 위험은 낮다고 판단했다. 태그 도입은 평가셋 재검증과 함께 추후 검토한다 |
| "특정 종목이 다른 종목보다 낫다"고 단정하는 표현은 정규식으로 찾기 어렵다 | 재생성·검수 프롬프트에서 판단하고 필요하면 정기적으로 표본을 검토한다 |
| 퀴즈는 정답 번호만 코드로 잠근다 | 현재는 문항을 한 건씩 생성해 순서가 바뀌어도 문제가 없다. 여러 문항을 한 번에 생성하게 되면 문항 식별 키를 추가한다 |
