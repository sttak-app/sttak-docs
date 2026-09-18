# CB-7. 배포·인프라 (챗봇 RAG)

> 절차 원본은 `../7-deployment.md`. ES 는 **첫 상시 검색 인프라**라 프로비저닝·비용·컷오버가
> 배포 설계의 중심이다. (인프라 실작업은 sttak-infra 레포)

## CB-7.1 저장소 선택 (결정 — ADR 기록)

| 후보 | 판단 |
| --- | --- |
| **AWS OpenSearch Service (관리형)** | 권장 — 패치·복구 자동, nori 내장, VPC/IAM 통합. RRF 는 직접 구현 or 2.19+ 네이티브 |
| self-host (EC2/ECS) | 비추 — JVM·디스크·스냅샷 운영 부담을 팀이 짐 |
| Elastic Cloud | AWS 밖 SaaS — 경로·과금 별도, 굳이 |

## CB-7.2 환경 구성 — 정할 것

- dev: 최소 노드 1 (t3.small.search 급) — 설계·평가·듀얼런 개발용, 먼저 프로비저닝
- ❓ **prod 노드 구성 (팀 논의)**: 멀티AZ 2노드(가용성, 비용↑) vs 단일 노드 + "PG 재색인
  복구" 수용(비용↓ — SSOT 원칙이 방어 근거). 폴백 정책(C-NFR3)과 세트로 결정
- 네트워크: VPC 내 + SG 로 api/batch 태스크만 접근. fine-grained access control 검토
- 스냅샷: 관리형 자동 스냅샷으로 충분 (재색인 복구가 1차 수단)

## CB-7.3 설정·시크릿 카탈로그 (안)

| 키 | 용도 |
| --- | --- |
| `OPENSEARCH_ENDPOINT` (❓명명) | 도메인 엔드포인트 — Secrets/env |
| `OPENSEARCH_USERNAME`/`PASSWORD` 또는 IAM | 자격증명 — Secrets Manager (NFR-S1) |
| `sttak.rag.store` (❓명명) | pgvector \| es — 듀얼런 검색 스위치 |
| 인덱스 alias 명 | 설정화 여부 ❓ (기본 상수 + env 오버라이드 권장) |

yml 은 프로파일 3종 동일 유지 규약 그대로.

## CB-7.4 롤아웃 순서 (확정 골격)

1. dev 도메인 + nori 토큰화 검증 (사용자 사전 포함)
2. PG 청크 테이블 마이그레이션 + 색인 파이프라인 (컨텍스트 생성 포함)
3. **백필 1회** — 기존 청크 전량: 컨텍스트 생성 + (재)임베딩 + ES 색인 (비용 1회성, 사전 견적)
4. 검색 어댑터 + 평가 A/B (§CB-4-4) → 파라미터·rerank·임베딩 모델 확정
5. **듀얼런**: 색인 양쪽 유지, dev 에서 `es` 전환 → 안정화 관찰(드리프트·지연·0건률)
6. prod 도메인 프로비저닝 → prod 듀얼런 → 전환 → 구 경로(pgvector 검색) 제거 + knowledge_chunk 처분 실행

롤백: 어느 단계든 플래그를 `pgvector` 로 되돌리면 즉시 복귀 (구 경로 제거 전까지).

## CB-7.5 비용 (견적 항목)

- 도메인 월 고정비 dev+prod — **첫 상시 인프라 비용**, 팀/멘토 공유 후 확정
- 백필 1회성: 컨텍스트 생성 LLM + 전량 임베딩 (모델 결정에 따라 변동 — large 면 ~2배)
- 상시: 컨텍스트 생성(일 수 센트) + (rerank 채택 시) 쿼리당 LLM +1
- 듀얼런 기간 저장 이중화 (코퍼스 작아 미미)

## CB-7.6 문서·절차 세트

착수 시: **ADR 신설**(ADR-016/017/018 supersede — 저장소·청크·임베딩·rerank·폴백·가드레일
방식 결정 전부 포함) + 본 디렉터리 ❓ 항목 확정 갱신 + `../6-observability`·`../7-deployment`
env 카탈로그 동기화 + Jira 티켓. 계약에 닿는 결정(0건 폴백 (b)안·스트리밍 (a)안)은 iOS 협의
+ apidocs 갱신 세트.
