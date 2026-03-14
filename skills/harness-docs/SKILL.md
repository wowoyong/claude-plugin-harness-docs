# harness-docs Skill

에이전트를 위한 레포 내 문서 관리 시스템. "에이전트에게 백과사전이 아니라 지도를 줘라"는 원칙 하에, 모든 지식을 레포 안에 버전 관리되는 아티팩트로 유지한다.

## 트리거

- `/docs` — 전체 워크플로 안내
- `/harness-docs` — 전체 워크플로 안내
- `/docs-diagnose` — 문서 품질 진단 실행
- `/docs-extract` — 소스코드에서 문서 자동 추출
- `/agents-md` — AGENTS.md 생성/검증

---

## Phase 1: 초기화 (Initialization)

레포에 문서 인프라를 구축한다.

### 실행 순서

1. **레포 스캔**: 프로젝트 루트에서 기존 `.md` 파일, 디렉토리 구조, 패키지 매니저 설정 파일을 탐색한다.
2. **docs/ 구조 생성**: 아래 하위 디렉토리를 생성한다.
   ```
   docs/
   ├── architecture/    # 시스템 아키텍처 문서
   ├── design/          # 설계 결정 문서 (ADR 포함)
   ├── specs/           # API 및 기능 스펙
   ├── glossary/        # 도메인 용어집
   ├── usecases/        # 유스케이스 문서
   ├── standards/       # 코딩 표준, 컨벤션
   ├── plans/           # 프로젝트 계획, 마일스톤
   ├── quality/         # 품질 리포트, 테스트 전략
   └── index.yml        # 중앙 문서 인덱스
   ```
3. **index.yml 생성**: 기존 마크다운 파일을 분석하여 초기 인덱스를 구축한다.
4. **설정 파일 생성**: `.harness/config.yml`에 프로젝트별 설정을 저장한다.
5. **Pre-commit 훅 제안**: 문서 품질 검사를 커밋 시점에 실행할 수 있도록 훅 설정을 제안한다.

### docs/ 디렉토리 규칙

- 모든 문서는 `docs/` 하위에 위치해야 한다.
- 각 문서는 YAML frontmatter를 포함해야 한다.
- 파일명은 kebab-case를 사용한다 (예: `auth-flow.md`, `payment-api-spec.md`).
- 이미지는 `docs/_assets/` 디렉토리에 저장한다.

---

## Phase 2: 진단 (Diagnostics)

4계층 점수 체계로 문서 품질을 측정한다.

### 4계층 점수 체계

#### Layer 1: Index Integrity (인덱스 정합성) — 가중치 40%

| 검사 항목 | 설명 | 심각도 |
|-----------|------|--------|
| 경로 유효성 | index.yml의 모든 `path`가 실제 파일을 가리키는지 | error |
| 고아 문서 | `docs/` 내 존재하지만 인덱스에 없는 파일 | warning |
| 유령 항목 | 인덱스에 있지만 파일이 없는 항목 | error |
| 중복 등록 | 동일 경로가 인덱스에 2번 이상 등록 | error |
| ID 고유성 | 모든 문서 ID가 고유한지 | error |
| 필수 필드 | id, path, type, status 필드가 모두 존재하는지 | error |
| 타입 유효성 | type 값이 허용된 값인지 | warning |

점수 계산:
```
layer1_score = (1 - error_count / total_checks) * 100
```
에러가 하나라도 있으면 이 레이어의 최대 점수는 70점으로 제한된다.

#### Layer 2: Agent Guide Hygiene (에이전트 가이드 위생) — 가중치 30%

| 검사 항목 | 설명 | 심각도 |
|-----------|------|--------|
| AGENTS.md 존재 | 프로젝트 루트에 AGENTS.md가 있는지 | error |
| 라인 수 제한 | AGENTS.md가 100줄 이하인지 | warning |
| 점진적 공개 | AGENTS.md가 상세 문서로의 포인터를 포함하는지 | warning |
| 외부 참조 없음 | Notion, Google Docs 등 외부 링크가 없는지 | error |
| 디렉토리 구조 | 프로젝트 디렉토리 구조를 포함하는지 | info |
| 코딩 컨벤션 | 코딩 규칙에 대한 포인터가 있는지 | info |

점수 계산:
```
layer2_score = base_score - (error_count * 20) - (warning_count * 10) - (info_missing * 5)
```

#### Layer 3: Harness Principle Alignment (하네스 원칙 준수) — 가중치 20%

| 검사 항목 | 설명 | 심각도 |
|-----------|------|--------|
| 설계 문서 검증 상태 | design/ 문서에 `status: validated` 비율 | warning |
| 계획 진행 추적 | plans/ 문서에 progress 섹션이 있는지 | warning |
| 기술 부채 추적 | tech-debt 관련 문서가 존재하는지 | info |
| 품질 등급 존재 | quality/ 디렉토리에 등급 문서가 있는지 | info |
| 버전 관리 | 문서에 `updated` 날짜가 있는지 | warning |
| 참조 그래프 | 문서 간 `references`가 양방향인지 | info |

점수 계산:
```
layer3_score = (validated_ratio * 40) + (progress_exists * 20) + (debt_tracked * 15) + (quality_graded * 15) + (versioned_ratio * 10)
```

#### Layer 4: Link Quality (링크 품질) — 가중치 10%

| 검사 항목 | 설명 | 심각도 |
|-----------|------|--------|
| 내부 참조 해결 | 문서 내 `[link](path)` 가 실제 파일을 가리키는지 | error |
| 깨진 링크 없음 | 404가 되는 링크가 없는지 | error |
| 양방향 참조 | A가 B를 참조하면 B도 A를 참조하는지 | info |
| 앵커 유효성 | `#section` 앵커가 실제 헤딩에 대응하는지 | warning |

점수 계산:
```
layer4_score = (valid_links / total_links) * 100
```

### 종합 점수

```
total_score = (layer1 * 0.4) + (layer2 * 0.3) + (layer3 * 0.2) + (layer4 * 0.1)
```

### 건강 등급

| 등급 | 점수 범위 | 의미 |
|------|-----------|------|
| A | 90-100 | 우수 — 에이전트가 즉시 활용 가능 |
| B | 80-89 | 양호 — 사소한 개선 필요 |
| C | 70-79 | 보통 — 주요 개선 권장 |
| D | 60-69 | 미흡 — 상당한 작업 필요 |
| F | 0-59 | 불량 — 문서 인프라 재구축 필요 |

---

## Phase 3: 자동 추출 (Auto-extraction)

소스코드를 분석하여 문서를 자동 생성한다.

### 추출 대상

#### TypeScript / JavaScript
- **인터페이스 및 타입**: `export interface`, `export type` → 용어집 항목
- **API 라우트**: Express/Fastify/Next.js 라우트 핸들러 → UseCase 문서
- **Zod/Joi 스키마**: 검증 스키마 → 스펙 문서
- **에러 타입**: `class XxxError extends Error` → 에러 코드 문서
- **설정 스키마**: `.env` 패턴, config 객체 → 설정 스펙 문서

#### Python
- **클래스 정의**: `class Xxx:` + docstring → 용어집 항목
- **FastAPI/Flask 라우트**: `@app.get`, `@app.post` → UseCase 문서
- **dataclass/Pydantic 모델**: 데이터 모델 → 스펙 문서
- **예외 클래스**: `class XxxError(Exception)` → 에러 코드 문서

#### Go
- **구조체**: `type Xxx struct` → 용어집 항목
- **HTTP 핸들러**: `func XxxHandler(w http.ResponseWriter, r *http.Request)` → UseCase 문서
- **인터페이스**: `type Xxx interface` → 스펙 문서

### 추출 프로세스

1. **소스 스캔**: 지정된 경로(기본: 프로젝트 루트)에서 소스 파일을 탐색한다.
2. **패턴 매칭**: 언어별 AST 패턴을 적용하여 문서화 대상을 식별한다.
3. **컨텍스트 수집**: JSDoc/docstring/GoDoc 주석, 타입 정보, 사용 예시를 수집한다.
4. **마크다운 생성**: 수집된 정보로 구조화된 마크다운 문서를 생성한다.
5. **인덱스 등록**: 생성된 문서를 `docs/index.yml`에 자동 등록한다.
6. **중복 검사**: 이미 존재하는 문서와 중복되는지 확인하고, 중복 시 업데이트를 제안한다.

### 생성되는 문서 형식

```markdown
---
id: "extracted-user-dto"
type: glossary
status: draft
extracted_from: "src/models/user.ts"
extracted_at: "2026-03-14"
---

# User DTO

`src/models/user.ts`에서 자동 추출된 용어집 항목.

## 정의

| 필드 | 타입 | 설명 |
|------|------|------|
| id | string | 사용자 고유 식별자 |
| email | string | 이메일 주소 |
| role | UserRole | 사용자 역할 (admin, user, guest) |

## 사용처

- `src/controllers/auth.ts` — 로그인 응답
- `src/controllers/profile.ts` — 프로필 조회

## 관련 문서

- [UserRole 열거형](glossary/user-role.md)
- [인증 API 스펙](specs/auth-api.md)
```

---

## Phase 4: AGENTS.md 관리

AGENTS.md를 생성하고 유지 관리한다.

### AGENTS.md 원칙

1. **100줄 이하**: 에이전트가 빠르게 파악할 수 있어야 한다.
2. **지도 역할**: 상세 내용은 포함하지 않고, 어디에 무엇이 있는지만 알려준다.
3. **점진적 공개**: 개요 → 구조 → 포인터 순서로 정보를 공개한다.
4. **레포 내부만 참조**: 외부 링크(Notion, Google Docs, Confluence)를 절대 포함하지 않는다.
5. **자동 갱신 가능**: index.yml로부터 자동으로 재생성할 수 있어야 한다.

### AGENTS.md 구조

```markdown
# AGENTS.md

## 프로젝트 개요
[1-2문장 프로젝트 설명]

## 기술 스택
[주요 기술 나열]

## 디렉토리 구조
[핵심 디렉토리만 트리로 표시]

## 핵심 문서
[docs/ 내 주요 문서 포인터]

## 코딩 컨벤션
[규칙 요약 또는 standards/ 포인터]

## 작업 시 참고사항
[에이전트가 작업 전 알아야 할 핵심 사항]
```

### 생성 프로세스

1. **프로젝트 분석**: `package.json`, `pyproject.toml`, `go.mod` 등에서 프로젝트 메타데이터를 추출한다.
2. **디렉토리 분석**: 프로젝트 구조를 분석하여 핵심 디렉토리를 식별한다.
3. **index.yml 활용**: 등록된 문서 중 상태가 `validated`인 핵심 문서를 선별한다.
4. **컨벤션 탐지**: ESLint, Prettier, EditorConfig 등에서 코딩 규칙을 추출한다.
5. **AGENTS.md 조립**: 위 정보를 100줄 이내로 조합하여 AGENTS.md를 생성한다.

### 검증 규칙

- 총 라인 수가 100줄을 초과하면 경고를 출력한다.
- 외부 URL(`https://notion.so`, `https://docs.google.com` 등)이 포함되면 에러를 발생시킨다.
- `docs/` 내 문서를 참조하는 링크가 최소 3개 이상이어야 한다.
- 프로젝트 개요, 디렉토리 구조, 핵심 문서 섹션이 반드시 존재해야 한다.

---

## Phase 5: 유지보수 (Maintenance)

문서를 지속적으로 최신 상태로 유지한다.

### 오래된 문서 탐지

소스코드 변경 대비 문서가 오래된 경우를 탐지한다.

**탐지 기준:**
- 문서의 `updated` 날짜가 관련 소스 파일의 마지막 커밋보다 30일 이상 오래된 경우
- 문서가 참조하는 파일 경로가 더 이상 존재하지 않는 경우
- 문서가 참조하는 함수/클래스 시그니처가 변경된 경우

**탐지 프로세스:**
1. `docs/index.yml`에서 각 문서의 `updated` 날짜를 확인한다.
2. `git log`를 통해 관련 소스 파일의 마지막 변경 일시를 조회한다.
3. 문서 내 코드 참조(`src/xxx.ts` 등)가 여전히 유효한지 확인한다.
4. 오래된 문서 목록을 생성하고 `status: stale`로 변경을 제안한다.

### 문서 가드닝

정기적인 문서 정리 작업을 지원한다.

- **미사용 태그 정리**: 어떤 문서에도 사용되지 않는 태그를 제거한다.
- **참조 그래프 최적화**: 끊어진 참조를 수정하고, 누락된 양방향 참조를 추가한다.
- **상태 갱신**: `draft` 상태가 30일 이상 지속된 문서에 대해 리뷰를 제안한다.
- **중복 탐지**: 유사한 내용의 문서를 식별하고 통합을 제안한다.

### Pre-commit 훅

커밋 시 자동으로 문서 품질을 검사한다.

```yaml
# .harness/config.yml
pre_commit:
  enabled: true
  checks:
    - index_integrity    # 인덱스 정합성 검사
    - broken_links       # 깨진 링크 검사
    - agents_md_size     # AGENTS.md 라인 수 검사
  fail_on: error         # error | warning | info
  skip_patterns:
    - "*.test.*"
    - "node_modules/**"
```

---

## MCP 서버 도구

이 플러그인은 다음 MCP 도구를 제공한다. 에이전트는 작업 중 필요에 따라 이 도구를 호출할 수 있다.

### docs_lookup

문서 ID 또는 키워드로 문서를 검색한다.

**사용 시점**: 에이전트가 특정 개념, API, 아키텍처에 대한 문서를 찾아야 할 때.

**파라미터:**
- `query` (string, required): 문서 ID 또는 검색 키워드
- `type` (string, optional): 문서 타입 필터 (architecture, design, spec, glossary, usecase, standard, plan, quality)
- `status` (string, optional): 문서 상태 필터 (draft, review, validated, stale)

**반환값:**
```json
{
  "results": [
    {
      "id": "auth-flow",
      "path": "docs/architecture/auth-flow.md",
      "type": "architecture",
      "status": "validated",
      "title": "인증 플로우",
      "snippet": "JWT 기반 인증 플로우의 전체 구조..."
    }
  ],
  "total": 1
}
```

### docs_register

새 문서를 인덱스에 등록한다.

**사용 시점**: 에이전트가 새 문서를 작성한 후 인덱스에 추가해야 할 때.

**파라미터:**
- `id` (string, required): 문서 고유 ID (kebab-case)
- `path` (string, required): docs/ 기준 상대 경로
- `type` (string, required): 문서 타입
- `tags` (string[], optional): 태그 목록
- `references` (string[], optional): 참조 문서 ID 목록

**동작:**
1. ID 중복 검사를 수행한다.
2. 파일 존재 여부를 확인한다.
3. index.yml에 새 항목을 추가한다.
4. `status: draft`, `updated: <today>` 를 자동 설정한다.

### docs_validate

인덱스 정합성과 문서 링크를 검증한다.

**사용 시점**: 에이전트가 문서 변경 후 전체 정합성을 확인하고 싶을 때.

**파라미터:**
- `scope` (string, optional): 검증 범위 — `full` (전체), `index` (인덱스만), `links` (링크만). 기본값: `full`
- `fix` (boolean, optional): 자동 수정 가능한 문제를 바로 수정할지. 기본값: `false`

**반환값:**
```json
{
  "valid": false,
  "errors": [
    { "type": "phantom_entry", "id": "old-doc", "message": "인덱스에 등록되어 있으나 파일이 존재하지 않음" }
  ],
  "warnings": [
    { "type": "orphan_doc", "path": "docs/design/new-feature.md", "message": "파일이 존재하나 인덱스에 미등록" }
  ],
  "fixed": []
}
```

### docs_stale_check

코드 변경 대비 오래된 문서를 탐지한다.

**사용 시점**: 에이전트가 코드 변경 작업을 완료한 후, 관련 문서가 최신 상태인지 확인할 때.

**파라미터:**
- `since` (string, optional): 기준 날짜 (ISO 8601). 기본값: 30일 전
- `paths` (string[], optional): 검사할 소스 경로 목록. 기본값: 전체 프로젝트

**반환값:**
```json
{
  "stale_docs": [
    {
      "id": "payment-api",
      "path": "docs/specs/payment-api.md",
      "doc_updated": "2025-12-01",
      "source_changed": "2026-03-10",
      "related_sources": ["src/controllers/payment.ts", "src/models/payment.ts"],
      "staleness_days": 100
    }
  ],
  "total_checked": 42,
  "stale_count": 3
}
```

---

## index.yml 스키마

모든 문서를 추적하는 중앙 인덱스 파일.

```yaml
version: "1.0"
project:
  name: "my-project"
  description: "프로젝트 설명"
  updated: "2026-03-14"

documents:
  - id: "arch-overview"           # 고유 ID (kebab-case, 필수)
    path: "docs/architecture/overview.md"  # docs/ 기준 상대 경로 (필수)
    type: architecture            # 문서 타입 (필수)
    status: validated             # 문서 상태 (필수)
    updated: "2026-03-14"         # 마지막 업데이트 날짜 (자동 관리)
    tags: ["system", "layers"]    # 태그 목록 (선택)
    references: ["arch-layers"]   # 참조 문서 ID 목록 (선택)
    title: "아키텍처 개요"          # 문서 제목 (선택, frontmatter에서 추출)
    extracted_from: null          # 자동 추출 출처 (자동 추출 시 설정)
```

### 허용되는 type 값

| 타입 | 설명 | docs/ 하위 디렉토리 |
|------|------|---------------------|
| architecture | 시스템 아키텍처 | architecture/ |
| design | 설계 결정 (ADR) | design/ |
| spec | API/기능 스펙 | specs/ |
| glossary | 도메인 용어 | glossary/ |
| usecase | 유스케이스 | usecases/ |
| standard | 코딩 표준 | standards/ |
| plan | 계획/마일스톤 | plans/ |
| quality | 품질 리포트 | quality/ |

### 허용되는 status 값

| 상태 | 설명 |
|------|------|
| draft | 초안 — 작성 중 |
| review | 검토 중 — 리뷰 필요 |
| validated | 검증 완료 — 신뢰 가능 |
| stale | 오래됨 — 업데이트 필요 |

---

## 설정 파일

`.harness/config.yml`에 프로젝트별 설정을 정의한다.

```yaml
# .harness/config.yml
project:
  name: "my-project"
  languages: ["typescript", "python"]
  source_dirs: ["src/", "lib/"]
  exclude_dirs: ["node_modules/", "dist/", ".next/"]

docs:
  root: "docs/"
  index: "docs/index.yml"
  agents_md: "AGENTS.md"
  max_agents_md_lines: 100

extraction:
  auto_register: true            # 추출 후 자동으로 인덱스 등록
  glossary_from_types: true      # 타입/인터페이스에서 용어집 자동 추출
  usecase_from_routes: true      # 라우트에서 유스케이스 자동 추출
  spec_from_schemas: true        # 스키마에서 스펙 자동 추출

diagnostics:
  weights:
    index_integrity: 40
    agent_guide_hygiene: 30
    harness_alignment: 20
    link_quality: 10
  stale_threshold_days: 30       # 오래된 문서 판별 기준 일수
  draft_review_days: 30          # draft 상태 유지 최대 일수

pre_commit:
  enabled: true
  checks:
    - index_integrity
    - broken_links
    - agents_md_size
  fail_on: error
  skip_patterns:
    - "*.test.*"
    - "node_modules/**"
```

---

## 워크플로 예시

### 새 프로젝트에서 시작

```
사용자: /docs-init
→ docs/ 구조 생성, index.yml 초기화, .harness/config.yml 생성

사용자: /docs-extract --type all
→ 소스코드 스캔, 용어집/유스케이스/스펙 문서 자동 생성

사용자: /agents-md --generate
→ AGENTS.md 자동 생성

사용자: /docs-diagnose
→ 4계층 품질 진단 실행, 개선 사항 리포트
```

### 기존 프로젝트에서 진단

```
사용자: /docs-diagnose --json
→ JSON 형식으로 진단 결과 출력

사용자: /docs-diagnose --fix
→ 자동 수정 가능한 문제 즉시 수정

사용자: /docs-diagnose --ci
→ CI 파이프라인용 종료 코드 반환 (에러 시 exit 1)
```

### 코드 변경 후 문서 유지보수

```
사용자: (코드 변경 작업 완료 후)
→ docs_stale_check 도구로 오래된 문서 확인
→ 해당 문서 업데이트
→ docs_validate 도구로 정합성 확인
→ index.yml의 updated 날짜 갱신
```
