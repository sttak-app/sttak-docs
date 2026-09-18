# CB-3-1. 아키텍처 (챗봇 RAG)

## CB-3-1.1 전체 그림 — 두 갈래에 걸친 파이프라인

```
[경로 B — sttak-batch]                          [경로 A — sttak-api]
뉴스 수집 → 가공(요약·판단)                      사용자 질문 (SSE)
    → 청크 조립                                     → 검색: BM25 ┐
    → 컨텍스트 생성(luna)                                kNN     ├→ RRF → (rerank?)
    → PG 청크 테이블 저장 (SSOT)                        필터(종목·기간)┘
    → 임베딩                                        → 프롬프트 조립([참고 자료] ≤N청크)
    → ES 색인 (파생)                                 → 생성(luna) + 가드레일(방식 미정)
                                                    → SSE 스트림 + source 이벤트
```

- **색인은 배치, 검색·생성은 실시간** — 배치 5기능과 달리 사용자 지연이 직접 보이는 경로
- ES 는 검색 전용 — 저장의 진실은 PG (C-NFR2)

## CB-3-1.2 저장소 배치

| 저장소 | 보관 내용 | 비고 |
| --- | --- | --- |
| PG 신규 청크 테이블 | chunk_id, news_id(원천 참조), raw_text(원문 청크), context(맥락문), ❓embedding 보관 여부, 생성 시각 | 기존 운영 테이블 무변경. **embedding 을 PG 에도 둘지 = 재색인 비용 vs 저장 이중화 결정 필요** |
| ES 인덱스 | search_text(nori 합본 = context+raw+스니펫), embedding(dense_vector, cosine), raw_text, stock_codes(keyword — 노션 안에 보강), published_at(date), source(NEWS/TERM), _id=chunk_id | **alias(`chunk-v1`) 접근** — 무중단 재색인 전제 |
| 기존 knowledge_chunk | ❓처분 결정: 폴백용 유지 / context 만 이전 후 폐기 | 듀얼런 종료 시점과 연동 |

## CB-3-1.3 정합성 (PG↔ES)

- 쓰기 순서 고정: **PG 커밋 → ES 색인** (한 트랜잭션에 안 묶음)
- 색인은 멱등 배치 스텝 — "PG 에 있는데 ES 에 없는 것" anti-join, 실패 건 다음 회차 재등장
- term 은 upsert 갱신 데이터 — 뜻 갱신 시 재컨텍스트/재임베딩/재색인 전파 경로 필요
- 드리프트 감시: PG 청크 수 vs ES 문서 수 비교 메트릭 (§CB-6)

## CB-3-1.4 포트·모듈 배치 (헥사고날)

| 책임 | 포트 (sttak-domain) | 구현 위치 | 비고 |
| --- | --- | --- | --- |
| 검색 (읽기, api) | 현행 검색 포트 시그니처 **유지** | sttak-external ES 어댑터 | 시그니처 유지 시 챗 Service·프롬프트 조립·SSE 무변경. **Spring AI VectorStore 로는 불가**(RRF 하이브리드가 추상화 밖) — ES 클라이언트 직접 |
| 색인 (쓰기, batch) | 색인 포트 신설 | sttak-external ES 어댑터 (검색과 분리) | |
| 컨텍스트 생성 | 배치 스텝 → luna | 임베딩 스텝에 선행 단계로 | 멱등: context IS NULL 대상 |
| 임베딩 | `EmbeddingPort` (Spring AI, ADR-034) | 기존 공용 어댑터 | 모델 변경 시(❓large) 프로퍼티만 |
| 스토어 스위치 | — | `sttak.rag.store=pgvector\|es` 류 플래그 | 듀얼런 컷오버 (C-NFR7) |

## CB-3-1.5 인프라 (요약 — 상세는 §CB-7)

관리형 OpenSearch(nori 내장) 권장 · VPC 내 + SG(앱 태스크만) · dev 먼저, prod 는 컷오버 직전 ·
**prod 노드 구성(단일+재색인 복구 vs 멀티AZ)은 비용 결정으로 팀 논의**
