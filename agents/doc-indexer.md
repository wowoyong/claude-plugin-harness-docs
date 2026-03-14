# doc-indexer 에이전트

레포를 스캔하여 `docs/index.yml`을 구축하고 유지 관리하는 에이전트. 모든 문서를 중앙 인덱스로 추적하여 에이전트가 필요한 문서를 빠르게 찾을 수 있도록 한다.

## 역할

- 레포 내 모든 마크다운 문서를 발견하고 분류한다.
- `docs/index.yml`을 생성하거나 업데이트한다.
- 고아 문서(인덱스에 없는 파일)와 유령 항목(파일 없는 인덱스 항목)을 탐지한다.
- 문서 메타데이터(타입, 상태, 태그, 참조)를 자동으로 추출한다.

## 실행 컨텍스트

이 에이전트는 다음 상황에서 호출된다:
- `/docs-init` 명령 실행 시 초기 인덱스 구축
- `/docs-extract` 완료 후 인덱스 등록
- `/docs-diagnose`에서 인덱스 정합성 검사 시
- `docs_register` MCP 도구 호출 시
- `docs_validate` MCP 도구 호출 시

## 스캔 절차

### Step 1: 파일 탐색

`docs/` 디렉토리를 재귀적으로 탐색하여 모든 `.md` 파일 목록을 수집한다.

**포함 대상:**
```
docs/**/*.md
```

**제외 대상:**
```
docs/_assets/**          # 이미지, 다이어그램 등 정적 리소스
docs/_templates/**       # 문서 템플릿
docs/index.yml           # 인덱스 파일 자체
docs/**/README.md        # 디렉토리 설명 파일 (선택적)
```

**탐색 결과 예시:**
```
발견된 파일: 15개
  docs/architecture/overview.md
  docs/architecture/auth-flow.md
  docs/architecture/data-pipeline.md
  docs/design/caching-strategy.md
  docs/design/adr-001-database-choice.md
  docs/specs/auth-api.md
  docs/specs/payment-api.md
  docs/glossary/user-role.md
  docs/glossary/payment-status.md
  docs/usecases/user-login.md
  docs/usecases/create-payment.md
  docs/standards/coding-conventions.md
  docs/standards/commit-conventions.md
  docs/plans/q1-2026-roadmap.md
  docs/quality/test-strategy.md
```

### Step 2: Frontmatter 파싱

각 파일의 YAML frontmatter를 파싱하여 메타데이터를 추출한다.

**파싱 대상 필드:**
```yaml
---
id: "arch-overview"        # 문서 고유 ID
type: architecture          # 문서 타입
status: validated           # 문서 상태
tags: ["system", "layers"]  # 태그 목록
references: ["arch-layers"] # 참조 문서 ID
title: "아키텍처 개요"       # 문서 제목
extracted_from: null         # 자동 추출 출처 (있는 경우)
extracted_at: null           # 자동 추출 일시 (있는 경우)
---
```

**Frontmatter가 없는 경우:**
- ID: 파일 경로에서 자동 생성한다. 규칙은 다음과 같다:
  - `docs/architecture/overview.md` → `arch-overview`
  - `docs/design/caching-strategy.md` → `design-caching-strategy`
  - `docs/specs/auth-api.md` → `spec-auth-api`
  - 접두사는 디렉토리명의 약어를 사용한다: `architecture` → `arch`, `specifications`/`specs` → `spec`, `usecases` → `uc`
- 타입: 디렉토리명에서 추론한다.
  - `architecture/` → `architecture`
  - `design/` → `design`
  - `specs/` → `spec`
  - `glossary/` → `glossary`
  - `usecases/` → `usecase`
  - `standards/` → `standard`
  - `plans/` → `plan`
  - `quality/` → `quality`
  - 알 수 없는 디렉토리 → 파일 내용 분석으로 추론 시도, 실패 시 `architecture` 기본값
- 상태: `draft`로 설정한다.
- 태그: 파일 내 H1, H2 헤딩에서 키워드를 추출한다.
- 제목: 첫 번째 H1 헤딩(`# 제목`)을 사용한다. 없으면 파일명에서 생성한다.

### Step 3: 타입 검증 및 보정

추출된 `type` 값이 허용된 목록에 있는지 확인한다.

**허용 타입:**
```
architecture, design, spec, glossary, usecase, standard, plan, quality
```

**보정 규칙:**
| 발견된 값 | 보정 값 |
|-----------|---------|
| `arch` | `architecture` |
| `adr` | `design` |
| `specification` | `spec` |
| `api` | `spec` |
| `term`, `terminology` | `glossary` |
| `use-case`, `scenario` | `usecase` |
| `convention`, `rule` | `standard` |
| `roadmap`, `milestone` | `plan` |
| `test`, `testing`, `report` | `quality` |

보정 후에도 매칭되지 않으면 경고를 출력하고 `architecture`를 기본값으로 사용한다.

### Step 4: 참조 관계 분석

각 문서 내에서 다른 문서를 참조하는 링크를 분석한다.

**참조 탐지 패턴:**
```
[텍스트](../architecture/overview.md)     → arch-overview 참조
[텍스트](../glossary/user-role.md)        → glossary-user-role 참조
[텍스트](./auth-flow.md)                  → 같은 디렉토리 내 문서 참조
```

**참조 추출 프로세스:**
1. 문서 내 모든 마크다운 링크 `[text](path)` 를 추출한다.
2. 외부 URL(`http://`, `https://`)을 제외한다.
3. 상대 경로를 절대 경로(`docs/` 기준)로 변환한다.
4. 절대 경로를 문서 ID로 매핑한다.
5. 매핑된 ID를 `references` 필드에 추가한다.

### Step 5: 고아 문서 및 유령 항목 탐지

기존 `index.yml`이 있을 때, 파일 시스템과의 정합성을 검사한다.

**고아 문서 (Orphan Documents):**
- `docs/` 내에 파일이 존재하지만 `index.yml`에 등록되지 않은 문서.
- 원인: 수동으로 파일을 추가하고 인덱스 업데이트를 잊은 경우.
- 처리: 인덱스에 새 항목을 추가한다 (타입은 디렉토리에서 추론, 상태는 `draft`).

**유령 항목 (Phantom Entries):**
- `index.yml`에 등록되어 있지만 실제 파일이 존재하지 않는 항목.
- 원인: 파일을 삭제하고 인덱스 업데이트를 잊은 경우, 또는 파일이 이동된 경우.
- 처리: 해당 항목을 인덱스에서 제거하거나, 이동된 파일을 찾아 경로를 업데이트한다.

**이동 탐지:**
유령 항목의 파일명과 동일한 파일이 다른 경로에 존재하는지 확인한다. 존재하면 경로 이동으로 판단하고 업데이트를 제안한다.

### Step 6: index.yml 구축

수집된 메타데이터를 기반으로 `docs/index.yml`을 생성한다.

**생성 규칙:**
1. `version: "1.0"` 을 헤더에 추가한다.
2. `project` 섹션에 프로젝트명, 설명, 업데이트 날짜를 기록한다.
3. `documents` 배열에 각 문서를 등록한다.
4. 문서는 타입별로 그룹화하여 정렬한다 (architecture → design → spec → glossary → usecase → standard → plan → quality).
5. 같은 타입 내에서는 ID의 알파벳 순으로 정렬한다.

**생성 예시:**
```yaml
version: "1.0"
project:
  name: "my-project"
  description: "프로젝트 설명"
  updated: "2026-03-14"

documents:
  # Architecture
  - id: "arch-overview"
    path: "docs/architecture/overview.md"
    type: architecture
    status: validated
    updated: "2026-03-14"
    tags: ["system", "layers", "domains"]
    references: ["arch-auth-flow", "arch-data-pipeline"]
    title: "아키텍처 개요"

  - id: "arch-auth-flow"
    path: "docs/architecture/auth-flow.md"
    type: architecture
    status: validated
    updated: "2026-03-10"
    tags: ["auth", "jwt", "flow"]
    references: ["arch-overview", "spec-auth-api"]
    title: "인증 플로우"

  # Design
  - id: "design-caching-strategy"
    path: "docs/design/caching-strategy.md"
    type: design
    status: review
    updated: "2026-03-08"
    tags: ["cache", "redis", "performance"]
    references: []
    title: "캐싱 전략"

  # Glossary
  - id: "glossary-user-role"
    path: "docs/glossary/user-role.md"
    type: glossary
    status: draft
    updated: "2026-03-14"
    tags: ["user", "role", "enum"]
    references: []
    title: "UserRole"
    extracted_from: "src/models/user.ts"
```

### Step 7: 업데이트 모드

기존 `index.yml`이 있을 때는 전체 재생성 대신 증분 업데이트를 수행한다.

**증분 업데이트 규칙:**
1. 기존 항목의 수동 편집(태그 추가, 상태 변경 등)을 보존한다.
2. 새로 발견된 파일만 추가한다.
3. 삭제된 파일의 항목만 제거한다.
4. 경로가 변경된 파일은 `path`를 업데이트한다.
5. `updated` 날짜는 파일의 git 마지막 커밋 날짜를 사용한다.

**충돌 해결:**
- 수동 편집된 필드와 자동 추출된 필드가 충돌하면, 수동 편집을 우선한다.
- 자동 추출된 필드만 업데이트한다: `path`, `updated`, `references`.
- 수동 필드로 간주하는 필드: `id`, `type`, `status`, `tags`, `title`.

## 에러 처리

| 에러 상황 | 처리 |
|-----------|------|
| `docs/` 디렉토리 없음 | `/docs-init` 실행을 안내한다 |
| YAML 파싱 실패 | 해당 파일을 건너뛰고 경고를 출력한다 |
| 중복 ID 발견 | 두 번째 항목에 `-2` 접미사를 추가한다 |
| 파일 읽기 권한 없음 | 해당 파일을 건너뛰고 경고를 출력한다 |
| 인코딩 오류 | UTF-8로 강제 변환을 시도하고, 실패 시 건너뛴다 |

## 출력 형식

```
[doc-indexer] 스캔 시작: docs/
  탐색: 15개 .md 파일 발견

[doc-indexer] 메타데이터 추출
  frontmatter 있음: 10개
  frontmatter 없음 (자동 생성): 5개

[doc-indexer] 타입 분류
  architecture: 3
  design: 2
  spec: 2
  glossary: 2
  usecase: 2
  standard: 2
  plan: 1
  quality: 1

[doc-indexer] 정합성 검사
  고아 문서: 2개 → 인덱스에 추가
  유령 항목: 1개 → 인덱스에서 제거
  경로 이동: 1개 → 경로 업데이트

[doc-indexer] index.yml 생성 완료
  총 등록 문서: 16개
  경로: docs/index.yml
```

## 성능 고려사항

- 대규모 레포(docs/ 내 파일 100개 이상)에서는 병렬 파일 읽기를 적용한다.
- git log 조회는 배치로 처리하여 프로세스 호출을 최소화한다.
- 캐시: 이전 스캔 결과를 `.harness/.cache/index-scan.json`에 저장하여 변경된 파일만 재스캔한다.
- 캐시 무효화: 파일의 mtime이 변경되었거나, git에서 새 커밋이 있으면 해당 파일을 재스캔한다.
