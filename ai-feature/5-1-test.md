# AF-5-1. 테스트 (배치 AI 기능)

테스트 계층과 Fixture 작성 규칙은 `../5-1-test-case.md`와 루트의 `TESTING.md`를 따른다.
여기에는 AI 기능에만 해당하는 테스트 구현을 정리한다.

## AF-5-1.1 원칙

1. **테스트에서는 LLM을 호출하지 않는다.** 전송 계층은 ChatModel 목으로, 파이프라인의 협력
   객체는 함수형 페이크(`(text, slot, feedback) -> …`)로 대체한다. LLM 출력의 품질은
   테스트가 아니라 평가(`4-4`)에서 확인한다.
2. **평가에서 발견한 문제는 테스트 사례로 추가한다.** 대표적으로 오탐 유형과 경계 문장을
   테스트에 넣어 같은 문제가 다시 생기면 CI에서 찾을 수 있게 한다.
3. **프롬프트는 핵심 조항의 존재만 검증한다**(`contains`/`doesNotContain`). 전문 스냅샷
   비교는 문구 개선을 막으므로 하지 않는다.

## AF-5-1.2 구현된 테스트 대상

| 대상 | 방식 | 대표 케이스 |
| --- | --- | --- |
| `InvestmentGuardrail` | 순수 함수 | 32개 사례로 문장 끝의 권유 표현 탐지, 네 가지 예외 규칙, 평가에서 발견한 오탐 문장의 정상 통과를 확인한다 |
| `GuardrailPipeline` | 페이크 rewriter/judge | 피드백 반복, 재시도 소진 시 `passed=false`와 카운터 증가, 검수 불능 시 통과를 확인한다 |
| 검수/재생성 어댑터 | ChatModel 목 | strict 스키마 파싱, 힌트 전달, 응답 불능 시 원문 유지 또는 통과, **refusal 반려**, 코드 펜스 제거를 확인한다 |
| 기능 어댑터 3종 | ChatModel 목 + 페이크 파이프라인 | 통과한 결과의 매핑, 재시도 소진 시 기능 예외, 잘못된 JSON과 잠금 위반 반려, 요청 규격 보호를 확인한다(§AF-5-1.4) |
| `<기능>Support` | 순수 함수 | 프롬프트 핵심 조항, `describe()` 8종 비지 않음, 후처리 검증 |
| 배치 Tasklet/Processor | Mockito | Port 부재 시 스킵, 개별 실패 흡수, DuplicateKey 흡수 |
| 잡 E2E | `@SpringBootTest` + AI Port 스텁 + Testcontainers | 테스트 데이터를 넣고 배치 단계를 실행해 상태 전이를 확인한다. 실제 LLM 어댑터가 컨텍스트에 남지 않도록 항상 목으로 대체한다 |

## AF-5-1.3 기능별 필수 잠금 테스트

새 기능에 파이프라인을 적용할 때는 최소한 **정상 통과, 재시도 횟수 소진, 잠금 값 위반**을
각각 테스트한다. 잠금 값 위반 테스트에서는 재생성 결과가 값을 바꾸도록 만든 뒤 해당 결과가
반려되는지 확인한다.

```java
// 회고 예 — type 반전(GOOD→BAD)은 표현 재작성 범위를 벗어난 구조 변경
GuardrailRewriter rewriter = (text, slot, feedback) ->
        text.replace("\"type\":\"GOOD\"", "\"type\":\"BAD\"");
assertThatThrownBy(() -> target.generate(candidate()))
        .isInstanceOf(BusinessException.class);
```

퀴즈는 `correctChoice`가 바뀌면 반려하고, 뉴스는 판단 라벨이 바뀌면 반려하는 방식으로 같은
패턴을 적용한다. 뉴스 관련 내용은 `../ai-chatbot/`에서 다룬다.

## AF-5-1.4 요청 규격을 보호하는 테스트

AI 제공자 호출 규격이 바뀌지 않도록 CI에서 확인한다. 직접 작성한 strict 스키마를 사용하고
`temperature`를 보내지 않는지 모든 어댑터 테스트에서 다음과 같이 검증한다.

```java
OpenAiChatOptions opt = (OpenAiChatOptions) promptCaptor.getValue().getOptions();
assertThat(opt.getTemperature()).isNull();     // 자동설정 기본값 0.7 주입 사고 차단
assertThat(opt.getResponseFormat().getJsonSchema().getName()).isEqualTo("...");
```

`temperature`가 전송되지 않는 것은 자동 설정을 끄고 빌더에 기본값을 두지 않았기 때문이다.
설정 변경이나 프레임워크 업데이트로 이 동작이 바뀔 수 있다. 현재 모델은 `temperature`
파라미터를 받지 않으므로 값이 전송되면 모든 호출이 400 응답으로 실패한다. 새 어댑터에도
같은 테스트를 반드시 추가한다.

## AF-5-1.5 하지 않는 것

- 실제 API를 호출하는 테스트는 작성하지 않는다. 비용이 들고 결과가 매번 달라질 수 있으며,
  키가 노출될 위험이 있다. CI에는 API 키가 없는 상태가 정상이다.
- 해설의 품질이나 판정의 정확성처럼 LLM 출력 품질을 테스트 코드로 단정하지 않는다. 이는
  평가에서 확인한다.
- 프롬프트 전문을 스냅샷으로 고정하지 않고 핵심 규칙이 포함됐는지만 확인한다.
