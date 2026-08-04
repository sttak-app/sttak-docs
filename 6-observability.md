# 6. Observability — sTTak 백엔드

본 문서는 sTTak 백엔드의 **로깅 · 요청 추적(`X-Request-Id`) · 민감정보 마스킹 · 에러 트래킹(Sentry) ·
메트릭** 규약을 정의한다.
로깅/추적/마스킹/Sentry 설계의 결정 배경은 [ADR-020](../decisions/ADR-020-structured-logging-sentry.md)에 있다.

핵심 원칙:

1. **로그는 구조화 JSON 한 줄**로 남긴다.
2. **모든 HTTP 요청에 추적 ID**(`X-Request-Id`)를 부여해 MDC·응답 헤더로 전파한다 *(NFR-O1)*.
3. **PII/시크릿은 로그·Sentry 에 평문으로 남기지 않는다** *(NFR-S1, NFR-S5, CON-S1)*.
4. **에러 알림은 로그 레벨로 라우팅**한다 — ERROR 로그만 Sentry 이벤트가 된다.
5. **세 실행 앱이 한 벌의 로깅 설정을 공유**한다(`sttak-common`).

---

## 6.1 로깅 (Logging)

### 6.1.1 스택 & 파일 배치

| 요소 | 위치 | 비고 |
| --- | --- | --- |
| JSON 인코더 | `net.logstash.logback:logstash-logback-encoder` | `sttak-common` 의 `implementation` → 런타임 전이로 세 앱 상속 |
| 로그백 설정 | `sttak-common/src/main/resources/logback-spring.xml` | **단일 파일**, 세 앱이 클래스패스로 공유 |
| 마스킹/MDC 키 | `com.sttak.sttakcommon.logging.*` | `SensitiveDataMasker`, `MdcKeys` |

- 서비스 이름은 `spring.application.name`(`sttak-api`/`sttak-admin`/`sttak-batch`)을 JSON 필드로 넣는다.

### 6.1.2 출력 형식 (환경별)

| 프로파일 | 콘솔 형식 |
| --- | --- |
| `local` | **평문 패턴** (사람이 읽기 좋게, `requestId` 포함) |
| `dev` / `prod` | **한 줄 JSON** (`LogstashEncoder`) |

### 6.1.3 환경별 로그 레벨 분리

- `<springProfile>` 블록 + 환경변수 **`LOG_LEVEL_ROOT`**(기존 카탈로그, 기본 `INFO`)로 제어한다.

| 프로파일 | root | `com.sttak` |
| --- | --- | --- |
| `local` | `INFO` | `DEBUG` |
| `dev` | `INFO` | `DEBUG` |
| `prod` | `${LOG_LEVEL_ROOT:-INFO}` | `INFO` |

- 레벨은 코드에서 하드코딩하지 않는다.

---

## 6.2 요청 추적 ID (`X-Request-Id`)

- `sttak-api` 최상위 필터가 요청마다 추적 ID 를 부여한다.
  - `X-Request-Id` 헤더가 **있으면 그대로 echo**, 없으면 **서버가 UUID 생성**.
  - MDC 키 `requestId` 에 저장 → 모든 로그 줄에 자동 포함.
  - **응답 헤더 `X-Request-Id` 에 항상 반영**한다 *(NFR-O1)*.
  - 요청 종료 시 MDC 를 정리한다.

---

## 6.3 민감정보 마스킹

**1차 — 규율:** `NFR-S5` 대로 **PII(이메일·이름)를 애초에 로그에 넣지 않는다.** API 키/시크릿도
로그에 넣지 않는다 *(NFR-S1, CON-S1)*.

**2차 — 인코더 마스킹(안전망):** `MaskingJsonGeneratorDecorator` 로 자동 마스킹한다.

| 기준 | 대상 예 | 처리 |
| --- | --- | --- |
| 필드명 | `password`, `authorization`, `token`, `apiKey`, `serviceKey`, `clientSecret` 등 | 값 → 마스킹 |
| 값 정규식 | 이메일, `Bearer`/JWT, API 키 패턴 등 | 매칭 구간 마스킹 |

- 마스킹은 **안전망**일 뿐 1차 방어를 대체하지 않는다. 신규 민감 필드는 `SensitiveDataMasker` 에 추가한다.

---

## 6.4 에러 트래킹 (Sentry)

- 의존성: `io.sentry:sentry-spring-boot-starter-jakarta` + `io.sentry:sentry-logback`.
- **ERROR 로그만 Sentry 이벤트로** 보낸다(`sentry.logging.minimum-event-level=error`). 코드에서 Sentry 를
  직접 호출하지 않는다.

| 설정 | 값 |
| --- | --- |
| `sentry.dsn` | `${SENTRY_DSN:}` — **미주입(local) 시 SDK no-op** |
| `sentry.send-default-pii` | `false` |
| `sentry.logging.minimum-event-level` | `error` — ERROR → Issue(에러 이벤트) |
| `sentry.logs.enabled` | `true` — WARN+ 로그를 Sentry Logs 로도 전송 |
| `sentry.logging.minimum-level` | `warn` — Sentry Logs 전송 임계치 |

- **두 경로 모두 마스킹**: 에러 이벤트는 `beforeSend`, Sentry Logs 는 **별도** `Logs.beforeSend`
  (둘 다 `SensitiveDataMasker` 적용, `SttakSentry`). 로그는 인코더 마스킹을 거치지 않으므로 필수.
- **MDC 전파**: MDC 값은 Sentry 이벤트의 `contexts.MDC` 로 자동 첨부된다. `sttak-api` 는
  `sentry.context-tags: [requestId]` 로 `requestId` 를 **검색 가능한 태그**로 승격한다.
  `sttak-batch` 는 같은 방식으로 `jobName`/`jobExecutionId`(`JobMdcListener` 가 MDC 에 적재)를 태그로 승격해
  Sentry 이벤트를 잡·실행 단위로 필터할 수 있게 한다.

---

## 6.5 메트릭 카탈로그 (Micrometer)

배치 파이프라인 (SCRUM-56, ADR-014):

| 메트릭 | 타입 | 태그 | 의미 |
|---|---|---|---|
| `sttak.news.collected` | counter | — | 수집 회차에 벤더로부터 수신한 기사 수 |
| `sttak.news.deduped` | counter | — | 이미 적재돼 있어 저장을 skip 한 기사 수 |
| `sttak.news.saved` | counter | — | 실제 INSERT/UPDATE 된 기사 수 |
| `sttak.ai.tokens.input` | counter | `provider` | LLM 가공 입력 토큰 (비용 추적) |
| `sttak.ai.tokens.output` | counter | `provider` | LLM 가공 출력 토큰 (비용 추적) |

활용 예: `deduped / collected` 비율이 1 에 가까우면 정상(증분 없음), `saved` 급증은 뉴스 유입 급증 신호.

## 6.6 메트릭 수집 — Prometheus + Grafana (dev, ADR-022)

- 세 앱은 `micrometer-registry-prometheus` 로 메트릭을 `/actuator/prometheus` 에 노출한다. dev 에서는 이 엔드포인트를
  **관리 포트 9000**(`management.server.port`)으로 분리해 공개 8080/ALB 에 실리지 않게 한다 — 메트릭 외부 노출 차단.
  `sttak-batch` 는 웹 서버가 없었으므로 actuator 노출용으로 `spring-boot-starter-web` 을 더한다.
- dev 에는 자체 호스팅 **Prometheus + Grafana** 를 ECS Fargate 로 띄운다. Prometheus 가 Cloud Map(`sttak-dev.local`)
  A레코드를 `dns_sd` 로 받아 각 태스크의 `9000/actuator/prometheus` 를 Pull 수집하고, Grafana 로 시각화한다.
- Grafana 는 ALB 미노출(dns내부 전용) — 접속은 ECS Exec + SSM 포트포워딩(`sttak-infra/docs/runbook-dev.md`).
- 설계 배경·대안·배포 순서는 [ADR-022](../decisions/ADR-022-monitoring-prometheus-grafana-dev.md). prod 동형화·Alertmanager 알람은 후속.
