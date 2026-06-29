# 4. Git Workflow — sTTak 백엔드

본 문서는 sTTak 백엔드의 **브랜치 전략 · 커밋 메시지 · PR 프로세스**를 정의한다.
`sttak-ios`, `sttak-backend`, `sttak-infra` 세 저장소가 **동일한 전략**을 공유한다.

> 본 문서는 "어떻게 작업을 합치는가"에 답한다.
> 코드/모듈 구조는 `3-2-directory.md`, API 계약 변경 정책은 `3-3-contract.md` §3-3.14를 참조한다.

연결 문서:

- 디렉터리/모듈: `3-2-directory.md`
- API 변경 정책: `3-3-contract.md` §3-3.14
- PR 템플릿: `.github/PULL_REQUEST_TEMPLATE.md`
- Jira Epic 매핑: `CLAUDE.md` §7

---

## 4.1 브랜치 구성

| 브랜치 | 역할 | 분기 기준 | 직접 커밋 |
| --- | --- | --- | --- |
| `main` | 운영 배포 기준. 안정 코드만 포함. | — | **금지** |
| `develop` | 개발 통합 기준. | `main` | **금지** |
| `feature/*` | 기능 개발. | `develop` | 허용 |
| `release/*` | 배포 준비 (검증·버그 수정·문서). | `develop` | 허용 |
| `hotfix/*` | 운영 긴급 수정. | `main` | 허용 |

---

## 4.2 핵심 규칙

- 일반 기능 작업은 항상 `develop`에서 `feature/*` 브랜치를 따서 진행하고, 완료 후 `develop`으로 PR/병합한다.
- `main`, `develop`에는 **직접 커밋/푸시하지 않는다**. 변경은 반드시 PR을 거친다.
- 하나의 `feature/*` 브랜치에서는 **하나의 기능/작업 단위**만 다룬다.
- 병합 완료된 `feature/*` 브랜치는 삭제한다.
- `release/*`에서는 **신규 기능 개발 금지**. 검증·버그 수정·문서 수정만 수행한다.
- `hotfix/*`는 운영 긴급 이슈에만 사용한다.
- `release/*`, `hotfix/*`에서 발생한 수정 사항은 `main` 뿐 아니라 **`develop`에도 반드시 반영**한다.

---

## 4.3 브랜치 네이밍

- 소문자 영문, 숫자, 하이픈(`-`)만 사용. 공백·대문자·언더스코어 금지.
- 형식: `feature/{Jira키}-{기능명}`, `hotfix/{Jira키}-{수정내용}`, `release/{버전}`
  - Jira 키는 **대문자 원형 그대로**(예: `SCRUM-123`) 사용한다. Jira 연동 인식을 위해 변형하지 않는다.
  - 기능명/수정내용은 소문자 + 하이픈(kebab-case)으로 3~5단어 이내로 짧게 적는다.
  - `release/*`는 버전을 기준으로 하므로 Jira 키를 붙이지 않는다.
  - Jira 이슈가 없는 사소한 작업은 키를 생략하고 `feature/{기능명}` 형태로 둘 수 있다.

| 예시 | 판정 |
| --- | --- |
| `feature/SCRUM-123-apple-login` | ✅ |
| `hotfix/SCRUM-456-token-refresh-error` | ✅ |
| `release/v1.0.0` | ✅ (버전 기준, 키 없음) |
| `feature/apple-login` | ⚠️ Jira 이슈가 있으면 키 포함 권장 |
| `feature/SCRUM-123-Login` | ❌ 대문자 |
| `feature/SCRUM-123 login` | ❌ 공백 |
| `feature/scrum-123-new_login` | ❌ 언더스코어 |

---

## 4.4 분기 / 병합 흐름

### 분기 시작

```bash
# 기능 개발
git checkout develop && git pull origin develop
git checkout -b feature/SCRUM-3-news-feed

# 긴급 수정
git checkout main && git pull origin main
git checkout -b hotfix/SCRUM-789-login-error
```

### 병합 흐름

```text
# 기능 개발
develop → feature/* → develop → release/* → main

# 운영 긴급 수정
main → hotfix/* → main
              └→ develop
```

---

## 4.5 커밋 메시지 컨벤션

### 형식

```text
<type>(<scope>): SCRUM-{번호} {변경 내용 요약}

<본문 - 무엇을 왜 바꿨는지>

<푸터 - Jira Smart Commit, BREAKING CHANGE 등>
```

- 제목 줄은 [Conventional Commits](https://www.conventionalcommits.org/) 규칙을 따른다: `type`, 선택적 `scope`, 그리고 요약.
- 요약 앞에 **Jira 키를 포함**한다(예: `SCRUM-3`). Jira 이슈가 없으면 키를 생략 가능하다 (사소한 문서/설정 변경 등).
- 제목 줄은 50자 권장 · 72자를 넘기지 않는다. 마침표로 끝내지 않으며, 명령형 현재 시제(`add`, `fix`)로 쓴다.
- 두 번째 줄은 비우고, 필요하면 본문에 "왜" 이렇게 변경했는지를 적는다. 본문은 72자 내외에서 줄바꿈한다.

#### type 종류

| type | 용도 |
| --- | --- |
| `feat` | 새 기능 |
| `fix` | 버그 수정 |
| `refactor` | 동작 변화 없는 코드 개선 |
| `docs` | 문서 |
| `test` | 테스트 추가/수정 |
| `chore` | 빌드·설정 등 잡무 |
| `style` | 포맷팅 (로직 영향 없음) |

### 예시

```text
feat(news): SCRUM-3 뉴스 피드 페이지네이션 API 구현

홈 피드 응답 시간 단축(NFR-P1)을 위해
관심종목별 뉴스 조회에 커서 기반 페이지네이션을 도입.

SCRUM-3 #time 2h #comment 커서 기반 페이지네이션 적용
```

```text
feat(auth): SCRUM-2 카카오 OAuth 콜백 처리 추가
```

### 작성 규칙

- 커밋은 **자기완결적인 단위**로 쪼갠다. 무관한 변경을 한 커밋에 묶지 않는다.
- 빌드/테스트가 깨진 채 커밋하지 않는다. 작업 중간 저장이 필요하면 로컬에서 `git stash` 또는 squash 후 push.
- 시크릿/자격증명은 코드·커밋에 절대 포함하지 않는다 (`CLAUDE.md` §5).

---

## 4.6 PR 프로세스

### 템플릿

- 모든 PR은 `.github/PULL_REQUEST_TEMPLATE.md`의 섹션을 채워서 올린다.
  - 관련 Jira 이슈 · 개요 · PR 유형 · 상세 내용 · 테스트 내용 · 스크린샷 · 체크리스트 · 리뷰어 참고 사항
- 변경된 내용이 필요한 문서(`docs/specs/`, `docs/iosapi/`, `CLAUDE.md` 등)에 반영되었는지 체크리스트에서 확인한다.

### 제목

- 브랜치명과 별개로, PR 제목은 사람이 읽기 좋은 한 문장으로 작성한다.
- 형식: `<type>(<scope>): SCRUM-{번호} {변경 내용 요약}` — 첫 커밋 메시지와 동일하게 맞춘다.
  - 예: `feat(news): SCRUM-3 뉴스 피드 페이지네이션 API 구현`

### 머지 대상

| 브랜치 | 머지 대상 |
| --- | --- |
| `feature/*` | `develop` |
| `release/*` | `main` (그리고 `develop`에도 백머지) |
| `hotfix/*` | `main` (그리고 `develop`에도 백머지) |

### 머지 후

- 머지된 `feature/*` 브랜치는 삭제한다 (GitHub의 "Delete branch" 사용).
- `release/*`, `hotfix/*`에서 `main`으로 머지한 경우, `develop` 백머지를 잊지 말 것.

---

## 4.7 폴리레포 정합성

본 전략은 `sttak-ios` · `sttak-backend` · `sttak-infra` 모두에 동일하게 적용된다.
세 저장소에 걸친 변경(예: API 계약 변경 → iOS 동시 대응)은 다음을 따른다.

- 각 저장소에서 **동일한 Jira 키**를 사용해 추적할 수 있게 한다.
- API 계약 변경(`docs/iosapi/apidocs.md`, `docs/specs/3-3-contract.md`)이 포함된 PR은 iOS 측 PR 링크를 본문에 함께 적는다.

---

## 4.8 후속 문서

| 문서 | 연결 |
| --- | --- |
| `.github/PULL_REQUEST_TEMPLATE.md` | PR 본문 템플릿 |
| `3-2-directory.md` | 디렉터리/모듈 구조 |
| `3-3-contract.md` §3-3.14 | API 하위호환/버저닝 정책 |
| `CLAUDE.md` §7 | Jira Epic ↔ Sprint 매핑 |
| `7-deployment.md` | 배포 파이프라인 (브랜치 ↔ 환경 매핑) |
