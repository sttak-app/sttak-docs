# 6. Observability — sTTak 백엔드

> 전체 관측성 설계(로그/트레이싱/`X-Request-Id`)는 추후 결정. 본 문서는 현재 발행 중인 메트릭 카탈로그만 우선 기록한다.

## 6.1 메트릭 카탈로그 (Micrometer)

배치 파이프라인 (SCRUM-56, ADR-014):

| 메트릭 | 타입 | 태그 | 의미 |
|---|---|---|---|
| `sttak.news.collected` | counter | — | 수집 회차에 벤더로부터 수신한 기사 수 |
| `sttak.news.deduped` | counter | — | 이미 적재돼 있어 저장을 skip 한 기사 수 |
| `sttak.news.saved` | counter | — | 실제 INSERT/UPDATE 된 기사 수 |
| `sttak.ai.tokens.input` | counter | `provider` | LLM 가공 입력 토큰 (비용 추적) |
| `sttak.ai.tokens.output` | counter | `provider` | LLM 가공 출력 토큰 (비용 추적) |

활용 예: `deduped / collected` 비율이 1 에 가까우면 정상(증분 없음), `saved` 급증은 뉴스 유입 급증 신호.
노출은 각 앱의 actuator(`/actuator/metrics`) 기준이며, 외부 수집기(CloudWatch/Prometheus) 연동은 추후 결정.
