# CB-4-1. 구현: 뉴스 가공과 색인

`3-1`에서 설명한 색인 파이프라인(뉴스 수집 → AI 가공 → 저장 → 청크 조립 → 색인)을
구체화한 문서다. 이미 구현된 뉴스 가공은 현재 동작을 기준으로 설명하고, 아직 구현하지 않은
청크 저장과 색인은 확정된 계획을 정리한다. 재생성과 검수의 공통 동작은
`../ai-feature/4-1`을 참고한다.

## CB-4-1.1 현재 뉴스 가공 방식

- `OpenAiNewsAnalysisAdapter` 하나에서 요약과 판단을 별도 작업으로 처리한다. 각 작업은
  검증된 단독 프롬프트(`NewsAnalysisSupport`의 요약 v2, 판단 v4)로 생성, 재생성, 검수를
  독립적으로 수행한 뒤 결과를 합친다.

  ```java
  public NewsAnalysis analyze(News news) {
      SummaryResult summary = generateSummary(news);       // v2 생성 → SUMMARY_SLOT 파이프라인
      SentimentResult sentiment = generateSentiment(news); // v4 생성 → SENTIMENT_SLOT 파이프라인 → 라벨 잠금
      return NewsAnalysisSupport.toNewsAnalysis(summary, sentiment);   // 병합 (기계적 조립)
  }
  ```

- 요약 작업에는 종목 코드를 전달하지 않는다. 판단 작업만 종목 목록을 받으며, `perStock`에는
  입력으로 받은 **모든 종목이 하나씩** 들어가야 한다. 잠금 검증(`requireSameSentiments`)도
  이 목록을 기준으로 원본과 대조한다.
- 작업별 메트릭은 `jobTag`의 `news-summary`와 `news-sentiment`로 구분한다.
- 어느 한 작업이라도 실패하면 기사 전체를 `NewsException.NEWS_ANALYSIS_FAILED`로 처리한다.
  부분 성공은 허용하지 않는다. 상태 머신(COLLECTED→ENRICHED)이 유지되므로 다음 실행에서
  다시 시도하며, 재시도 횟수도 함께 기록한다.
- 저장: `news_analysis`(요약·terms) + `news_stock`(판단) + `news_stock_why_point`.
  배치 진입점은 매시 실행되는 `NewsEnrichmentJobConfig`의 가공 단계다.
- 기사 한 건을 처리할 때 총 6회 호출한다(작업 2개 × 생성·재생성·검수).

## CB-4-1.2 청크 저장소 구현 계획(PG 신규 테이블)

기사 한 건은 요약과 종목 판단을 합쳐 하나의 청크로 만들고, 용어도 한 건당 하나의 청크로
저장한다. 청크가 작아 얻는 이점이 크지 않으므로 맥락문 생성(Contextual Retrieval)은
적용하지 않는다.

| 컬럼 | 내용 |
| --- | --- |
| chunk_id (PK) | 인덱스 문서 `_id` 와 동일 |
| 원천 참조 | news_id 또는 (원천 타입 + ID) — 용어 청크 포함 |
| raw_text | 표시·인용용 원문 청크 |
| search_text 구성 요소 | 요약 + 종목 판단 + **원문 제목·스니펫** (기사 표현 그대로 묻는 질문(U-C4)을 어휘 검색이 잡으려면 원문 표현이 검색 필드에 있어야 한다. 표시용 아님) |
| embedding | **`text-embedding-3-large`(3072차원)**로 만든 벡터. 검색 품질을 위해 약 2배의 비용을 수용하며, 퀴즈 유사도에는 small 모델을 계속 사용한다. 인덱스 유실, 매핑 변경, 전체 재색인 시 다시 임베딩하지 않고 복구할 수 있도록 **PG에도 보관**한다(NFR4) |
| **source_url** | 출처 구성에 사용할 기사 원문 URL(`3-2` 원칙 6) |
| 생성 시각 | — |

- 기존 `knowledge_chunk` 테이블은 신규 테이블의 백필을 검증한 뒤 **DROP**한다. 전환 기간의
  대체 검색도 신규 테이블을 사용하므로 기존 테이블을 유지하지 않는다.

## CB-4-1.3 검색 인덱스 구현 계획(ES/OpenSearch 매핑)

| 필드 | 타입·내용 |
| --- | --- |
| search_text | nori로 분석할 요약·판단·원문 제목·스니펫의 통합 텍스트 |
| embedding | 질문 임베딩과 같은 모델과 차원을 사용하는 `dense_vector`, cosine, dims 3072(`text-embedding-3-large`) |
| raw_text | 표시용 |
| stock_codes | keyword — 종목 필터 |
| published_at | date — 기간 필터 |
| source_type | NEWS / TERM. 뉴스와 용어를 **한 인덱스에 통합**해 한 번의 하이브리드 검색으로 함께 조회한다 |
| source_url | 출처 구성용 |
| _id | chunk_id |

- 매핑을 바꾸거나 전체를 다시 색인할 때 서비스를 중단하지 않도록 **alias로만 접근한다**.
  새 인덱스를 만들고 색인을 마친 뒤 alias를 전환한다.
- nori는 기본 설정으로 시작한다. 종목명 등의 사용자 사전과 동의어 사전은 토큰화 검증에서
  문제가 확인될 때 평가 결과를 근거로 도입한다.

## CB-4-1.4 색인 배치 구현 계획

- 새 잡을 만들지 않고 기존 임베딩 배치를 확장한다. 처리 단계를 [청크 조립 → 임베딩(신규
  테이블) → 인덱스 색인]으로 바꾸고 기존 스케줄, 트리거, 관측 구성을 재사용한다.
- PG를 기준으로 anti-join해 "PG에는 있지만 인덱스에는 없는 데이터"만 처리한다. 실패한
  데이터는 다음 실행 때 다시 대상에 포함된다.
- 쓰기 순서는 **PG 커밋 → 인덱스 색인**으로 고정하며, 하나의 트랜잭션으로 묶지 않는다.
  두 저장소의 데이터 차이는 건수 비교 메트릭으로 감시한다(`6`).
- 용어의 뜻이 갱신되면(upsert) 해당 청크의 재임베딩·재색인이 전파되어야 한다.
- 기존 지식 전체를 한 차례 임베딩하고 색인한다. 이때 발생하는 일회성 비용은 미리
  산정한다(`7`).
