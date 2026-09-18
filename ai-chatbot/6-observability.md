# CB-6. 관측성 (챗봇 RAG)

> 체계는 `../ai-feature/6-observability.md` 공유 (Langfuse 트레이스 / 메트릭 / 로그).
> 챗봇은 Spring AI 스트리밍 전환 제외(ADR-034)라 **자동 span 이 없다** — 수동 Observation
> 배선이 개편 스코프에 포함된다. 노션 설계의 로깅 항목이 본 절의 원안이다.

## CB-6.1 트레이스 — 질의 1건의 단계별 기록 (노션 안 확정)

질의당 하나의 트레이스에 단계별 span:

| span | input / output |
| --- | --- |
| query | 사용자 질문 (+ 맥락 newsId) |
| embedding | 질문 임베딩 (모델·토큰) |
| bm25 / knn | 각 검색의 히트 청크 ID·스코어 |
| rrf | 융합 후 순위 |
| rerank (채택 시) | 통과/탈락 청크와 사유 |
| generation | 프롬프트 조립 결과(청크 목록)·출력·토큰 |
| (전체) | 단계별·총 지연 |

- 구현: 수동 Micrometer Observation → 기존 OTel/OTLP 배선 재사용 (신규 인프라 없음)
- ❓ 본문 기록 정책: ai-feature 와 동일하게 **prod 는 본문 미기록**(질문은 사용자 입력).
  청크 ID·스코어 같은 메타는 prod 도 기록
- ❓ 샘플링: 사용자 트래픽 경로라 전수 수집 비용 확인 후 결정 (배치와 달리 물량 가변)

## CB-6.2 메트릭 — 정할 것 (후보)

| 후보 | 태그 | 용도 |
| --- | --- | --- |
| 검색 지연 histogram | stage(bm25/knn/rrf/rerank/total) | C-NFR1 p95 검증·경보 |
| 검색 0건 카운터 | — | 폴백 발생률 — 급증 시 색인·평가 점검 |
| rerank 탈락률 (채택 시) | — | 과필터 감시 |
| **PG↔ES 드리프트** | — | PG 청크 수 vs ES 문서 수 차 — 색인 침묵 실패 감지 |
| 색인 처리·실패 카운터 | — | 배치 상태 |
| 토큰 카운터 | job=chat / chat-rerank / chunk-context | 기존 `sttak.ai.tokens.*` 체계 합류 |

## CB-6.3 경보 후보 (임계는 팀 확정)

드리프트 지속 / 검색 0건률 급증 / 검색 p95 예산 초과 / ES 클러스터 상태(관리형 CloudWatch) —
`../ai-feature/6 §AI-6.5` 표에 합류시켜 함께 논의.
