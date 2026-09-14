# AI-5-2. 에러 처리 (AI 기능)

> 예외 계층·ErrorType↔HTTP 매핑은 `../5-2-error-handling.md` 가 원본. AI 기능은 전부 배치
> 경로라 HTTP 매핑보다 **"어디까지 흡수하고 무엇을 살리는가"** 가 핵심이다.

## AI-5-2.1 에러 분류와 처리 방향

| 분류 | 예 | 처리 | 이유 |
| --- | --- | --- | --- |
| 생성 호출 실패 | RestClientException, 타임아웃, 429/5xx | 재시도(`HttpRetrySupport`) → 소진 시 기능 예외로 변환 → **해당 건만 실패** | 벤더 일시 장애가 배치를 죽이면 안 됨 |
| 응답 형식 불량 | choices 비어있음, content null, JSON 파싱 실패 | 기능 예외 (해당 건 실패) | 저장 불가능한 결과 |
| 가드레일 소진 | 피드백 2회 후에도 위반 잔존 | passed=false → 기능 예외 + `exhausted{job}` 카운터 | 위반 콘텐츠는 저장하지 않는다 (CON-S5) |
| 잠금 위반 | 재작성이 라벨·정답·평가 구조 변경 | 기능 예외 (해당 건 실패) | 데이터 오염 차단 — 표현 정제가 판정을 바꾸면 안 됨 |
| 판정 불능 | 판정 호출 실패·파싱 실패 | **통과** (fail-open) | 오탐 0/250 실측의 "확신 없으면 통과". 정상 콘텐츠 보호 우선 |
| 판정 refusal | 모델이 응답 거부 | **반려** (fail-closed) + `judge.refusal` 카운터 | 입력 이상 신호 — fail-open 이중 겹침 차단 |
| 재작성 실패 | 재작성 호출 불능 | 원문 유지 → 판정 진행 | 판정이 최종 저지선 |
| 시스템 장애 | Error(OOM), JobRepository 메타 실패 | 흡수하지 않음 — 잡 실패 | 경보가 정당한 재해 상황 |

## AI-5-2.2 기능별 예외 타입

| 기능 | 예외 | ErrorType | 배치에서의 결과 |
| --- | --- | --- | --- |
| 뉴스 요약·판단 | `NewsException.NEWS_ANALYSIS_FAILED` (트랙 분리 — 실패는 기사 단위 공유) | INTERNAL | 해당 기사 상태 유지 → 다음 회차 재시도 (retry_count 상한, V8) |
| 퀴즈 | `QuizException.QUIZ_GENERATION_FAILED` | INTERNAL | 해당 문항 폐기 → 재시도 ≤3 → 그래도 실패면 그 회차 수량 감소 |
| 차트 해설 | `ChartException.EXPLANATION_GENERATION_FAILED` | INTERNAL | 해설 미저장 → 다음 배치 anti-join 재등장. 수집·감지 무영향 |
| 회고 | `TradeException.REVIEW_GENERATION_FAILED` | INTERNAL | 회고 미저장 → 다음 정산 배치 재등장. 정산 무영향 |

- 전부 기존 enum 재사용 — 파이프라인 도입으로 새 ErrorType/HTTP 매핑은 없다.
- 사용자 요청 경로에는 이 예외들이 도달하지 않는다 (배치 전용). api 는 "없으면 null/폴백
  카피" 계약 (`AI-3-2.3`).

## AI-5-2.3 격리 규약

1. **건별 격리** — 한 건의 생성 실패가 같은 회차의 다른 건을 막지 않는다. 저장은 건별
   트랜잭션(REQUIRES_NEW 또는 리포지토리 자체 트랜잭션).
2. **스텝 격리** — 해설/회고 스텝의 전면 실패도 선행 스텝(수집·감지·정산)의 커밋을 되돌리지
   않는다.
3. **중복 격리** — 동시 실행·재실행으로 인한 UNIQUE 충돌(DuplicateKey)은 멱등 승리로 간주,
   흡수한다.
4. **head-of-line 리스크 인지** — 오래된 순 + LIMIT 선정이라 결정적 실패 건이 매 회차
   앞자리를 차지할 수 있다. 같은 건의 실패 로그 반복 관찰 시 retry_count 패턴 도입
   (ADR-024/025 공통 수용 조건).

## AI-5-2.4 로그 규약

- 실패 로그는 **식별자 중심** — orderId/chartSignalId/newsId + 원인. 생성 콘텐츠 전문을 남기지
  않는다 (위반 인용문 요약 수준까지만).
- API 키·Authorization 헤더는 어떤 로그·예외 메시지에도 싣지 않는다 (NFR-S1).
- 소진 로그(`가드레일 재작성 소진 job=… 잔존위반=…`)는 warn — 폴백이 정상 동작한 것이므로
  error 가 아니다. error 는 잡 자체가 죽는 상황에 예약.
