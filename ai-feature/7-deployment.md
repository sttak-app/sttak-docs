# AI-7. 배포 (AI 기능)

> 배포 절차·인프라 전체는 `../7-deployment.md` 가 원본. 여기서는 AI 기능의 설정·키·롤아웃
> 유의점만 정의한다. (§6 모니터링은 논의 중 — 확정 후 `6-observability.md` 로 추가)

## AI-7.1 설정 카탈로그 (batch, 프로파일별 yml)

| 프리픽스 | 용도 | 주요 키 |
| --- | --- | --- |
| `sttak.ai.provider` | 생성 AI 스위치 — 구현은 `openai`(기본)뿐. 구현 없는 값 = 해당 배치 스킵 | `${AI_PROVIDER:openai}` |
| `sttak.openai.*` | 뉴스·퀴즈 공용 생성 | api-key, base-url, model, max-tokens |
| `sttak.chart.openai.*` | 차트 해설 생성 | 〃 (max-tokens 1024) |
| `sttak.trade.openai.*` | 회고 생성 | 〃 (max-tokens 2048) |
| `sttak.quiz.openai.*` | 퀴즈 생성 | 〃 (max-tokens 4096) |
| `sttak.guardrail.*` | 파이프라인 | **mode**(always-rewrite\|conditional), max-feedback-retries(2), api-key, model, max-tokens |
| `sttak.embedding.*` | 퀴즈 유사도·지식 임베딩 | text-embedding-3-small |

- 모든 api-key 는 `${OPENAI_API_KEY:}` 단일 환경변수 배선. 모델은 `${OPENAI_MODEL:gpt-5.6-luna}`.
- local/dev/prod 의 `sttak.*` 블록은 **100% 동일 유지** (드리프트 금지 — yml 헤더 주석 참조).
- `ANTHROPIC_API_KEY` 는 폐기됨(ADR-032) — Secrets Manager/task def 에서 제거 가능.

## AI-7.2 환경변수 (batch task def)

| 키 | 용도 | 필수 |
| --- | --- | --- |
| `OPENAI_API_KEY` | 생성 + 재작성 + 판정 + 임베딩 전부 | dev/prod (Secrets Manager) |
| `OPENAI_MODEL` | 모델 오버라이드 (기본 gpt-5.6-luna) | 선택 |
| `AI_PROVIDER` | 스위치 (기본 openai) | 선택 |

## AI-7.3 롤아웃 유의점 (파이프라인 첫 배포)

1. **차트 해설·회고가 처음 켜진다** — 종전 비활성이라 미생성 백로그가 쌓여 있다. 배포 다음
   회차부터 batch-size(차트 100/회 · 회고 50/회)만큼 소진 시작, 건당 3호출이므로 차트만
   회차당 최대 300호출. 백로그 규모를 먼저 확인하고 부담스러우면 env 로 batch-size 하향 시작.
2. **비용 관측 후 모드 결정** — 정상 운영 기준 파이프라인 +690호출/일 + 뉴스 트랙 분리
   +555호출/일 (기사당 3→6호출) 추정. `sttak.ai.tokens.*{job}` 로
   1~2주 집계 후 과다 판단 시 `sttak.guardrail.mode=conditional` 전환 (yml/env 변경 + 재배포,
   코드 무변경).
3. **경보 후보 등록** — `sttak.ai.guardrail.exhausted`(폴백 발생 = 위반 잔존 신호),
   `sttak.ai.guardrail.judge.refusal`(입력 이상). 임계·채널은 §6 논의에서 확정.
4. 차트 해설 백필 비용 참고치: 신호 5,350건 × (입력 ~850/출력 ~210 토큰) ≈ $10 안팎 1회성
   (ADR-025 §6) — 파이프라인 3호출 기준으로는 ×~2.2 환산.

## AI-7.4 배포 전 체크리스트 (AI 변경 포함 릴리스)

- [ ] 프롬프트 문구를 바꿨다면 실측 재검증 완료·기록했는가 (AI-NFR7, `AI-4-4.4`)
- [ ] 가드레일 범주·슬롯 변경이 코드 잠금(AI-FR9)과 모순되지 않는가
- [ ] 전 모듈 테스트 그린 (`./gradlew build` — Jacoco 포함)
- [ ] yml 3프로파일 동기화 확인 (local=dev=prod 의 sttak.* 블록)
- [ ] 신규 설정 키가 `@ConfigurationProperties` 와 일치하는가
- [ ] 키·시크릿이 diff 에 없는가 (env placeholder 만 허용)
- [ ] 계약에 닿는 변경이면 iOS 협의·apidocs 갱신 (`../3-3-contract.md §3-3.14`)
