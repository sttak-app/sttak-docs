# AI-7. 배포 (AI 기능)

> 배포 절차·인프라 전체는 `../7-deployment.md` 가 원본. 여기서는 AI 기능의 설정·키·롤아웃
> 유의점만 정의한다. 관측(트레이스·메트릭·경보)은 `6-observability.md` 참조.

## AI-7.1 설정 분류 (batch, 프로파일별 yml)

| 프리픽스 | 용도 | 주요 키 |
| --- | --- | --- |
| `sttak.ai.provider` | 생성 AI 스위치 — 구현은 `openai`(기본)뿐. 구현 없는 값 = 해당 배치 스킵 | `${AI_PROVIDER:openai}` |
| `sttak.openai.*` | 뉴스·퀴즈 공용 생성 | api-key, base-url, model, max-tokens |
| `sttak.chart.openai.*` | 차트 해설 생성 | 〃 (max-tokens 1024) |
| `sttak.trade.openai.*` | 회고 생성 | 〃 (max-tokens 2048) |
| `sttak.quiz.openai.*` | 퀴즈 생성 | 〃 (max-tokens 4096) |
| `sttak.guardrail.*` | 파이프라인 | max-feedback-retries(2), api-key, model, max-tokens |
| `sttak.embedding.*` | 퀴즈 유사도·지식 임베딩 | text-embedding-3-small |
| `spring.ai.model.*` | **전부 `none`** — Spring AI 자동설정 차단. 빠지면 기본 temperature 0.7 주입 → luna 전 호출 400 (ADR-033) | chat/embedding/image 등 |
| `spring.ai.chat.observations.*` | 트레이스 본문 기록 정책 — local/dev true, **prod false** | `6-observability.md` |

- 모든 api-key 는 `${OPENAI_API_KEY:}` 단일 환경변수 배선. 모델은 `${OPENAI_MODEL:gpt-5.6-luna}`.
- local/dev/prod 의 `sttak.*` 블록은 **100% 동일 유지** (드리프트 금지 — yml 헤더 주석 참조).
- `ANTHROPIC_API_KEY` 는 폐기됨(ADR-032) — Secrets Manager/task def 에서 제거 가능.

## AI-7.2 환경변수 (batch task def)

| 키 | 용도 | 필수 |
| --- | --- | --- |
| `OPENAI_API_KEY` | 생성 + 재작성 + 판정 + 임베딩 전부 | dev/prod (Secrets Manager) |
| `OPENAI_MODEL` | 모델 오버라이드 (기본 gpt-5.6-luna) | 선택 |
| `AI_PROVIDER` | 스위치 (기본 openai) | 선택 |
| `LANGFUSE_OTLP_ENDPOINT` / `LANGFUSE_OTLP_AUTH` | Langfuse 트레이스 전송 (ADR-033) — 미주입 시 전송만 실패, 기능 무영향 | dev/prod (Secrets Manager) |
| `LANGFUSE_TRACING_SAMPLING` | 트레이스 샘플링 (기본 1.0) | 선택 |

