# AF-7. 배포 (배치 AI 기능)

전체 배포 절차와 인프라 구성은 `../7-deployment.md`를 따른다. 여기에는 AI 기능의 설정
구조와 키 관리를 정리한다. 관측 설정은 `6-observability`를 참고한다.

## AF-7.1 배치 프로파일에 구현된 설정

| 프리픽스 | 용도 | 주요 키 |
| --- | --- | --- |
| `sttak.ai.provider` | 생성 AI 기능을 켜고 끈다. 현재 구현은 기본값인 `openai`뿐이며, 구현되지 않은 값을 사용하면 해당 생성 단계만 건너뛴다 | `${AI_PROVIDER:openai}` |
| `sttak.openai.*` | 뉴스·퀴즈 공용 생성 | api-key, base-url, model |
| `sttak.chart.openai.*` | 차트 해설 생성 | 〃 + max-tokens 1024 |
| `sttak.trade.openai.*` | 회고 생성 | 〃 + max-tokens 2048 |
| `sttak.quiz.openai.*` | 퀴즈 생성 | 〃 + max-tokens 4096 |
| `sttak.guardrail.*` | 검증 파이프라인 | max-feedback-retries(2), api-key, model, max-tokens |
| `sttak.embedding.*` | 퀴즈 유사도·지식 임베딩 | `text-embedding-3-small` |
| `spring.ai.model.*` | Spring AI 자동 설정을 막기 위해 **모두 `none`으로 설정한다.** 이 설정이 빠지면 기본 `temperature` 0.7이 전달되어 모든 호출이 400 응답으로 실패한다 | chat/embedding 등 |
| `spring.ai.chat.observations.*` | 트레이스 본문 기록 여부를 정한다. local/dev는 `true`, **prod는 `false`**다 | `6-observability` |

- 모든 `api-key`는 `${OPENAI_API_KEY:}` 환경 변수 하나로 주입한다. 생성·재생성·검수에는
  `${OPENAI_MODEL:gpt-5.6-luna}`로 지정한 **luna 모델만 사용한다.** AI 제공자나 모델을 다시
  나누려면 별도의 설계 결정이 필요하다.
- local, dev, prod의 `sttak.*` 설정 항목은 모두 같게 유지하고 값만 환경 변수로 구분한다.
  환경마다 설정 구조가 달라 운영에서만 다른 동작이 생기는 일을 막기 위해서다.

## AF-7.2 배치 태스크의 환경 변수

| 키 | 용도 | 필수 |
| --- | --- | --- |
| `OPENAI_API_KEY` | 생성, 재생성, 검수, 임베딩에 공통으로 사용 | dev/prod(시크릿 저장소) |
| `OPENAI_MODEL` | 기본값 `gpt-5.6-luna`를 다른 모델로 바꿀 때 사용 | 선택 |
| `AI_PROVIDER` | 기본값 `openai`인 기능 스위치 | 선택 |
| `LANGFUSE_OTLP_ENDPOINT` / `LANGFUSE_OTLP_AUTH` | 트레이스 전송에 사용. 설정하지 않으면 전송만 실패하고 기능은 계속 동작한다 | dev/prod(시크릿 저장소) |
| `LANGFUSE_TRACING_SAMPLING` | 기본값이 1.0인 샘플링 비율 | 선택 |

## AF-7.3 롤아웃 유의점

- 프롬프트·검증 기준 변경이 포함된 배포는 평가 재검증(`4-4`) 완료가 선행 조건이다.
- 한 번에 생성할 수 있는 양은 차트 해설 100건, 회고 50건, 퀴즈 10건이다. 백필이 필요하면
  환경 변수로 이 값을 잠시 높여 한 번 처리한 뒤 원래 값으로 되돌린다.
- 배포 후 첫 배치가 끝나면 재시도 소진 카운터(`exhausted`)와 토큰 사용량을 확인한다.
