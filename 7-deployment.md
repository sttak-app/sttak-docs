
본 문서는 sTTak 백엔드의 **배포 파이프라인과 운영 환경**을 정의한다.

원칙:

1. **운영 배포는 GitHub Actions 단일 파이프라인을 통해서만** 이루어진다 *(CON-O3)*.
2. **환경 프로파일은 `local` / `dev` / `prod` 셋** *(CON-O1)*.
3. **이미지 = 동일, 환경 = 설정으로 분기.** 환경별로 새 빌드하지 않는다.
4. **롤백은 한 번의 명령으로** 가능해야 한다 (이전 이미지 태그 재배포).

---

## 7.1 환경 모델

| 환경 | 목적 | 데이터 | 트래픽 |
| --- | --- | --- | --- |
| `local` | 개발자 PC, IDE 실행 | H2 (in-memory) / 로컬 Postgres 17 | 본인만 |
| `dev` | 통합 검증, 클라이언트 개발 | 운영과 동일한 스키마, 시드 데이터 | 내부 / 테스트 클라이언트 |
| `prod` | 실 서비스 | 운영 Postgres 17 (RDS) | 사용자 |

규약:

- `dev`와 `prod`의 인프라 스택은 **모양이 같다** (스케일만 차이). 운영 전용 컴포넌트가 dev에 없으면 운영에서 처음 만나는 버그가 생긴다.
- 새 환경 추가는 명시 승인 *(CON-O1)*.

---

## 7.2 인프라 토폴로지 (MVP, AWS 기준)

```
        ┌──────────────────────────────────────────────────┐
        │                CloudFront (HTTPS)                │
        └────────────────────────┬─────────────────────────┘
                                 │
                                 ▼
                          ALB (Application LB)
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
         ┌─────────┐        ┌──────────┐       ┌──────────┐
         │ECS Task │  …     │ ECS Task │  …    │ ECS Task │  ← sttak-api (수평 확장)
         │ sttak-  │        │ sttak-   │       │ sttak-   │
         │  api    │        │  admin   │       │  batch   │  ← sttak-batch (스케줄)
         └────┬────┘        └────┬─────┘       └────┬─────┘
              └─────────────┬────┴─────────────────┘
                            ▼
                      ┌─────────────┐         ┌──────────────┐
                      │  RDS Postgres17│      │   S3 (정적/감사)│
                      └─────────────┘         └──────────────┘
                            │
                            ▼
                  ┌──────────────────┐
                  │ CloudWatch Logs  │  + Prometheus/Grafana (별도)
                  └──────────────────┘
```

구성:

- **컨테이너 오케스트레이션**: ECS Fargate (MVP는 가장 운영 부담 적음). 향후 EKS 검토.
- **DB**: RDS for PostgreSQL **17.x**, Multi-AZ는 prod에서만, dev는 Single-AZ.
- **로드밸런서**: ALB (HTTPS 종단), 헬스체크 `/actuator/health/readiness`.
- **CDN/엣지**: CloudFront (정적 자산 + WAF).
- **Secrets**: AWS Secrets Manager (LLM/시세/뉴스 API 키, JWT 서명 키).

---

## 7.3 컨테이너 / 빌드

### 7.3.1 빌드 단위

각 실행 모듈이 각자의 컨테이너 이미지를 만든다. **레지스트리는 ECR** *(ADR-005, GHCR 에서 변경)*.

| 모듈 | 이미지 |
| --- | --- |
| `sttak-apps/sttak-api` | `<acct>.dkr.ecr.<region>.amazonaws.com/sttak-api:<env>-<git-sha>` |
| `sttak-apps/sttak-admin` | `<acct>.dkr.ecr.<region>.amazonaws.com/sttak-admin:<env>-<git-sha>` |
| `sttak-apps/sttak-batch` | `<acct>.dkr.ecr.<region>.amazonaws.com/sttak-batch:<env>-<git-sha>` |

규약:

- 이미지 태그는 **`<env>-git SHA(짧은 7자)`**. 예: `prod-a1b2c3d`, `dev-9f8e7d6`.
- **`latest` 태그는 사용하지 않는다.** 자동 변경되는 태그는 롤백을 어렵게 만든다. (ECR 레포는 `IMMUTABLE`.)
- ECS Task 는 execution role 로 ECR 을 인증 없이 pull, CI 는 OIDC 로 push *(ADR-005)*.

### 7.3.2 Dockerfile (개념)

운영용 이미지의 골격(각 모듈에서 동일 패턴 적용):

```dockerfile
# build stage
FROM gradle:8-jdk21 AS build
WORKDIR /workspace
COPY . .
RUN ./gradlew :sttak-apps:sttak-api:bootJar --no-daemon

# runtime stage
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /workspace/sttak-apps/sttak-api/build/libs/*.jar /app/app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

JVM 옵션은 `JAVA_OPTS` 환경변수로 주입한다.

> **실제 구현** *(ADR-007)*: 위는 개념도다. CI 가 gradle 로 bootJar 를 **1회** 빌드(캐시)하고, 단일 파라미터화 `deploy/Dockerfile` 은 그 jar 만 COPY 한다(Docker 안에서 gradle 재빌드 안 함). 3앱이 `deploy/Dockerfile` 하나를 공유한다.

---

## 7.4 CI/CD (GitHub Actions)

### 7.4.1 파이프라인 단계

```
PR ─► ci.yml ──► _build-test.yml (재사용)
                  ├─► build  (Java 21 + Gradle 캐시)
                  ├─► test   (Unit + Repo 통합 with Testcontainers)
                  └─► jacoco verification (≥ 50%)           ← 머지 게이트(테스트는 항상 전체)
                              │
push(develop) ─► deploy-dev.yml (auto)
                  ├─► _build-test.yml (동일 빌드 재사용)
                  ├─► detect-affected  (변경 경로 → 영향 앱 집합)
                  ├─► build & push to ECR   (affected 앱만, dev-<sha7>)
                  └─► ECS service update (rolling, affected 앱만)
                              │
push(main)    ─► deploy-prod.yml (manual approval)          ← 다음 단계(stub)
                  └─► DB migration → ECS service update (blue/green)
```

> **멀티모듈 전략** *(ADR-007)*: 테스트는 항상 전체, **이미지 빌드/배포는 affected 앱만**. 공유 라이브러리/빌드설정 변경 ⇒ 3앱 전부, 특정 앱만 변경 ⇒ 그 앱만(path filter + matrix).

### 7.4.2 트리거

| 파이프라인 | 트리거 |
| --- | --- |
| `ci.yml` | 모든 push / PR |
| `deploy-dev.yml` | `develop` 브랜치 머지 후 자동 |
| `deploy-prod.yml` | `main` 브랜치 머지 후 **수동 승인** |

### 7.4.3 비밀 / Secrets

- GitHub Actions Secrets: **`AWS_OIDC_ROLE_ARN` 만** (ECR 채택으로 `GHCR_TOKEN` 폐기 — `ADR-005`).
- 런타임 비밀 (LLM/시세/JWT 키)은 **GitHub Secrets에 두지 않는다.** AWS Secrets Manager에서 ECS Task가 직접 읽는다 *(NFR-S1)*.
- AWS 접근은 **OIDC**로 단기 토큰 발급. 정적 액세스 키 금지.

### 7.4.4 머지 게이트

`main` 머지를 위해 다음이 모두 통과되어야 한다.

- `ci.yml` 빌드/테스트/Jacoco 통과.
- **코드 리뷰 1인 이상 승인**.
- 컨벤셔널 커밋 형식 (`type(scope): subject`).

---

## 7.5 브랜치 전략

루트 `README.md`의 정책을 정식 출처로 한다.

| 브랜치 | 목적 |
| --- | --- |
| `main` | 운영 배포 기준 |
| `develop` | 개발 통합 |
| `feature/*` | 기능 개발 |
| `release/*` | 배포 준비 (MVP는 거의 사용 X) |
| `hotfix/*` | 운영 긴급 수정 |

규약:

- `main`에 직접 push 금지. PR + 승인 + CI 통과만.
- Squash merge (히스토리 노이즈 최소화).
- 머지 후 feature 브랜치 삭제.

---

## 7.6 환경별 설정 / 프로파일

### 7.6.1 Spring 프로파일

- `local` — 개발자 PC. H2 + show-sql + devtools.
- `dev` — 통합 검증.
- `prod` — 실 서비스.

활성화는 환경변수 `SPRING_PROFILES_ACTIVE=prod`.

### 7.6.2 설정 우선순위

```
환경변수 (ECS Task Definition / .env)
      │
      ▼
application-<profile>.yml  (in classpath)
      │
      ▼
application.yml            (공통)
```

규약:

- **모든 비밀은 환경변수**. yml 파일에는 빌드 시점에 알려진 비-비밀 값만.
- `application-local.yml`의 `password`처럼 평문 비밀이 들어간 파일은 **운영 이미지에 포함되지 않도록** 한다.

### 7.6.3 환경 변수 카탈로그 (요약)

| 키 | 의미 | 환경별 |
| --- | --- | --- |
| `SPRING_PROFILES_ACTIVE` | 프로파일 | 모두 |
| `SPRING_DATASOURCE_URL` | JDBC URL | dev/prod |
| `SPRING_DATASOURCE_USERNAME` / `_PASSWORD` | DB 자격 | dev/prod (Secrets Manager) |
| `JWT_SIGNING_KEY` | JWT 서명 키 | dev/prod (Secrets Manager) |
| `CLAUDE_API_KEY` | LLM | dev/prod (Secrets Manager) |
| `OPENAI_API_KEY` | 임베딩 | dev/prod (Secrets Manager) |
| `NEWS_API_KEY` | NewsAPI | dev/prod (Secrets Manager) |
| `DART_API_KEY` | DART | dev/prod (Secrets Manager) |
| `KIS_API_KEY` / `KIS_API_SECRET` | KIS Developers | dev/prod (Secrets Manager) |
| `LOG_LEVEL_ROOT` | 로그 레벨 | 선택 (기본 INFO) |
| `JAVA_OPTS` | JVM 옵션 | 선택 |

---

## 7.7 DB 마이그레이션

- 도구: **Flyway** *(CON-O2)*. (Liquibase 도입 시 본 문서 갱신.)
- > **현황(잠정)**: Flyway 는 **아직 미도입**(로드맵 §7.14 5단계). 그 전까지 `local`·`dev` 는 `spring.jpa.hibernate.ddl-auto=update` 로 스키마를 맞춘다. Flyway 도입 시 `ddl-auto` 를 `validate` 로 전환하고 아래 실행 시점 정책을 적용한다.
- 위치: `sttak-domain/src/main/resources/db/migration/V<번호>__<설명>.sql`
- 명명: `V1__init_user.sql`, `V2__add_trade.sql`. 한 번 커밋된 마이그레이션은 **수정 금지**.
- 실행 시점:
  - `local` — 앱 부팅 시 자동 (`spring.flyway.enabled=true`).
  - `dev` — 배포 직전 GitHub Actions에서 마이그레이션 잡 실행.
  - `prod` — 배포 직전 GitHub Actions에서 **수동 승인 후** 마이그레이션 잡 실행.
- 롤백:
  - 데이터 손실 가능한 마이그레이션은 **별 두 개 PR** (스키마 추가 → 데이터 백필 → 컬럼 제거를 분리).
  - `Flyway repair`는 수동 운영 절차 (런북에 명시).

---

## 7.8 배포 전략

### 7.8.1 dev

- **Rolling update.** 다운타임 허용 X, 1 분 단위 빠른 이터레이션.
- 헬스체크 `/actuator/health/readiness` 통과 후 트래픽 전환.

### 7.8.2 prod

- **Blue/Green.** 새 태스크 셋을 띄우고 readiness 확인 → ALB 가중치 100% 전환 → 이전 셋 종료.
- 전환 후 **5분 모니터링 윈도우**: 5xx 비율 > 1% 또는 p95 위배 시 자동 롤백.

### 7.8.3 batch

- **상시 Fargate 서비스**(`desired_count=1`, ALB 없음)로 배포 *(ADR-006)*. 앱이 in-process `@Scheduled`(`@EnableScheduling`)로 cron 을 돌리므로, 주기마다 컨테이너를 새로 띄우는 ECS Scheduled Task 모델 대신 상시 실행한다.
- 동시 실행 방지(`max concurrent: 1`): **인스턴스 1개 = 스케줄러 1개**로 중첩을 원천 차단. 2개 이상으로 스케일 시 분산 락 필요(후속 ADR).
- 배포 동작은 api/admin 과 동일(새 task def revision 등록 → `update-service` rolling).
- > 진짜 Scheduled Task(필요 시에만 실행, 비용↓)로 전환하려면 batch 를 "부팅 → 잡 1회 → 종료"로 재설계해야 한다 *(ADR-006 Consequences)*.

---

## 7.9 롤백

- “이전 이미지 태그를 다시 배포”가 표준 절차.
- 작업: `deploy-prod.yml` 워크플로를 **`IMAGE_TAG`** 입력으로 실행.
- 마이그레이션이 들어간 배포의 롤백은 **스키마 호환** 여부를 확인. 호환되지 않으면 데이터 복구 절차 필요 (런북).

---

## 7.10 비밀 / 서명 관리

- **AWS Secrets Manager** 단일 출처.
- ECS Task가 IAM Role을 통해 읽는다.
- JWT 서명 키 회전: 분기별 1회, 권장. 회전 시 검증 키 두 개 병행(이전+신규) → 일정 기간 후 이전 제거.
- API 키 회전은 벤더 정책 + 비밀 변경 → 무중단 hot-reload는 MVP 범위 외 (재배포 허용).

---

## 7.11 모니터링 / 알람 연동

운영 환경의 메트릭/로그/알람 정책은 `6-observability.md`.
배포 관점에서는 다음을 본다:

- 배포 직후 **A1(5xx 폭증)** / **A2(NFR-P1 위배)** 알람 윈도우 5분.
- 알람 발생 시 자동 롤백 트리거 (옵션, 점진 도입).

---

## 7.12 로컬 실행 (개발자 빠른 시작)

```bash
# 1) Postgres 17 띄우기 (로컬)
docker run -d --name sttak-pg \
  -e POSTGRES_USER=dionisos198 \
  -e POSTGRES_PASSWORD=haruka198^^ \
  -e POSTGRES_DB=sttaklocal \
  -p 5432:5432 \
  postgres:17

# 2) API 실행
./gradlew :sttak-apps:sttak-api:bootRun

# 3) 배치 실행 (선택)
./gradlew :sttak-apps:sttak-batch:bootRun

# 4) 전체 빌드 (테스트 + Jacoco)
./gradlew build
```

로컬 프로파일은 `application-local.yml`이 자동 활성화된다 (`spring.profiles.active: local`).

---

## 7.13 디렉터리/파일 — 운영 산출물

```
sttak-backend/
├── .github/workflows/
│   ├── ci.yml                 PR 머지 게이트 (재사용 워크플로 호출)
│   ├── _build-test.yml        재사용: build/test/jacoco + bootJar 아티팩트
│   ├── deploy-dev.yml         develop → affected 빌드/푸시/배포
│   └── deploy-prod.yml        (stub) main + 수동승인 + blue/green
├── deploy/
│   ├── Dockerfile             3앱 공용 단일 파라미터화 (ADR-007)
│   ├── render-taskdef.sh      task-def 템플릿 placeholder 치환
│   └── task-definitions/
│       ├── api.dev.json / admin.dev.json / batch.dev.json   (prod.*.json 은 후속)
└── sttak-domain/src/main/resources/db/migration/   (Flyway 도입 시 — §7.7)

sttak-infra/                   별도 리포 — Terraform IaC (ADR-007)
└── terraform/
    ├── modules/{iam-oidc,ecr,network,rds,secrets,ecs-cluster,ecs-service}
    └── envs/dev/              S3 backend + 모듈 조립
```

> 위 산출물은 **도입 시점에 함께 생성**한다. dev 범위는 구비 완료(prod 워크플로/task-def 은 후속).
> 레지스트리는 ECR *(ADR-005)*, batch 는 상시 서비스 *(ADR-006)*, IaC·멀티모듈 전략은 *(ADR-007)*.

---

## 7.14 단계적 도입 순서 (Roadmap)

1. **로컬 Postgres 17 + Testcontainers** — 이미 가능.
2. **CI(`ci.yml`)** — 빌드/테스트/Jacoco. (1순위 도입)
3. **ECR 이미지 빌드/푸시** *(ADR-005)*.
4. **dev ECS 서비스 + 자동 배포**.
5. **Flyway 도입 + 마이그레이션 잡**.
6. **prod ECS Blue/Green + 수동 승인**.
7. **Secrets Manager + OIDC**.
8. **모니터링/알람 연동** (`6-observability.md`).
9. **부하 테스트 stage 환경** (`5-1-test-case.md` §5-1.9).

---

## 7.15 의도적으로 남겨둔 결정 (TBD)

- ECS vs EKS — MVP는 ECS Fargate. 멀티 워크로드 늘면 EKS 재평가.
- Redis 도입 시점 — LLM 캐시 / 잠금 필요 시점.
- Vector DB 분리 시점 — pgvector 한계 도달 시.
- Canary 배포 도입 시점 — 사용자 트래픽 충분히 누적된 이후.
- 비밀 회전 자동화 — 1차 수동, 2차 자동.

---

## 7.16 후속 문서

| 문서 | 연결 |
| --- | --- |
| `3-1-server-architecture.md` | 모듈 구조 / 빌드 단위 |
| `5-1-test-case.md` | CI에서 돌릴 테스트 / 부하 테스트 |
| `6-observability.md` | 배포 후 모니터링 |
| `2-2-requirements.md` | NFR-X (확장성) / CON-O (운영) |
