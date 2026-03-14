# docs-init 명령어

프로젝트에 문서 인프라를 초기화한다. `docs/` 구조, `index.yml`, `.harness/config.yml`을 생성하고 기존 마크다운 파일을 자동으로 인덱싱한다.

## 사용법

```
/docs-init [options]
```

## 옵션

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `--force` | 기존 docs/ 구조가 있어도 덮어쓰기 | false |
| `--no-scan` | 기존 .md 파일 스캔 건너뛰기 | false |
| `--no-hook` | pre-commit 훅 설정 제안 건너뛰기 | false |
| `--lang <languages>` | 프로젝트 주요 언어 지정 (쉼표 구분) | 자동 탐지 |

## 실행 절차

### Step 1: 프로젝트 분석

프로젝트 루트를 스캔하여 다음 정보를 수집한다.

1. **패키지 매니저 탐지**: `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `build.gradle` 등의 존재 여부를 확인한다.
2. **언어 탐지**: 소스 파일의 확장자 분포를 분석하여 주요 언어를 식별한다. `--lang` 옵션으로 직접 지정할 수도 있다.
3. **기존 문서 탐지**: 프로젝트 내 `.md` 파일 목록을 수집한다. 이미 `docs/` 디렉토리가 있다면 그 구조도 분석한다.
4. **프레임워크 탐지**: Express, Next.js, FastAPI, Gin 등 사용 중인 프레임워크를 파악하여 추출 설정에 반영한다.

수집된 정보는 `.harness/config.yml` 생성에 활용된다.

### Step 2: docs/ 디렉토리 구조 생성

아래 디렉토리 구조를 프로젝트 루트에 생성한다.

```
docs/
├── architecture/    # 시스템 아키텍처, 컴포넌트 다이어그램, 레이어 구조
├── design/          # 설계 결정 기록 (ADR), 기술 선택 근거
├── specs/           # API 스펙, 기능 스펙, 인터페이스 정의
├── glossary/        # 도메인 용어, 타입 정의, 열거형 설명
├── usecases/        # 유스케이스 문서, 사용자 시나리오
├── standards/       # 코딩 표준, 네이밍 규칙, 커밋 컨벤션
├── plans/           # 프로젝트 계획, 마일스톤, 로드맵
├── quality/         # 품질 리포트, 테스트 전략, 커버리지 목표
├── _assets/         # 이미지, 다이어그램 등 정적 리소스
└── index.yml        # 중앙 문서 인덱스
```

이미 `docs/` 디렉토리가 존재하는 경우:
- `--force` 옵션 없이: 누락된 하위 디렉토리만 추가한다.
- `--force` 옵션 사용: 기존 구조를 유지하면서 누락된 디렉토리를 추가하고, `index.yml`을 재생성한다.

### Step 3: 기존 문서 인덱싱

`--no-scan` 옵션이 없다면, 프로젝트 내 기존 `.md` 파일을 분석하여 `index.yml`에 등록한다.

**인덱싱 규칙:**

1. `docs/` 하위의 `.md` 파일을 모두 수집한다.
2. 각 파일에서 YAML frontmatter를 파싱한다.
   - frontmatter에 `id`가 있으면 그대로 사용한다.
   - 없으면 파일 경로에서 ID를 생성한다 (예: `docs/architecture/overview.md` → `arch-overview`).
3. 문서 타입을 결정한다.
   - frontmatter에 `type`이 있으면 그대로 사용한다.
   - 없으면 디렉토리명에서 추론한다 (`architecture/` → `architecture`, `specs/` → `spec` 등).
4. 상태를 설정한다.
   - frontmatter에 `status`가 있으면 그대로 사용한다.
   - 없으면 `draft`로 설정한다.
5. 태그를 추출한다.
   - frontmatter의 `tags`가 있으면 사용한다.
   - 없으면 파일 내 헤딩에서 키워드를 추출한다.

**인덱싱 결과 출력:**
```
[docs-init] 프로젝트 분석 완료
  언어: TypeScript, Python
  프레임워크: Next.js, FastAPI
  기존 문서: 12개 발견

[docs-init] docs/ 구조 생성 완료
  새로 생성: architecture/, design/, quality/
  기존 유지: specs/, glossary/

[docs-init] 인덱싱 완료
  등록: 12개 문서
  타입 분포: architecture(3), spec(4), glossary(2), design(2), usecase(1)
  상태 분포: draft(8), validated(4)
```

### Step 4: .harness/config.yml 생성

프로젝트 분석 결과를 기반으로 설정 파일을 생성한다.

```yaml
# .harness/config.yml — harness-docs 설정
# 자동 생성: 2026-03-14

project:
  name: "my-project"              # package.json의 name 또는 디렉토리명
  languages: ["typescript"]       # 탐지된 주요 언어
  source_dirs: ["src/"]           # 소스 디렉토리
  exclude_dirs: ["node_modules/", "dist/", ".next/", "coverage/"]

docs:
  root: "docs/"
  index: "docs/index.yml"
  agents_md: "AGENTS.md"
  max_agents_md_lines: 100

extraction:
  auto_register: true
  glossary_from_types: true
  usecase_from_routes: true
  spec_from_schemas: true

diagnostics:
  weights:
    index_integrity: 40
    agent_guide_hygiene: 30
    harness_alignment: 20
    link_quality: 10
  stale_threshold_days: 30
  draft_review_days: 30

pre_commit:
  enabled: false                  # Step 5에서 활성화 제안
  checks:
    - index_integrity
    - broken_links
    - agents_md_size
  fail_on: error
```

### Step 5: Pre-commit 훅 제안

`--no-hook` 옵션이 없다면, pre-commit 훅 설정을 제안한다.

**제안 내용:**

커밋 시 다음 검사를 자동 실행하는 훅을 설정할 수 있다:
- `index_integrity`: index.yml의 모든 경로가 유효한지 확인
- `broken_links`: 문서 내 깨진 링크 탐지
- `agents_md_size`: AGENTS.md가 100줄을 초과하지 않는지 확인

사용자가 동의하면:
1. `.harness/config.yml`의 `pre_commit.enabled`를 `true`로 설정한다.
2. `.husky/pre-commit` 또는 `.git/hooks/pre-commit`에 검사 스크립트를 등록한다.
3. 검사 실패 시 커밋을 차단하도록 설정한다.

## 주의사항

- `docs/index.yml`이 이미 존재하고 `--force` 없이 실행하면, 기존 인덱스에 새로 발견된 문서만 추가한다.
- `.harness/config.yml`이 이미 존재하면 덮어쓰지 않고, 누락된 필드만 추가한다.
- `node_modules/`, `.git/`, `dist/`, `build/` 등 빌드 아티팩트 디렉토리는 스캔에서 제외한다.
- 대규모 프로젝트(파일 1000개 이상)에서는 스캔 시간이 길어질 수 있으며, 진행 상황을 표시한다.

## 다음 단계

초기화가 완료되면 다음 작업을 권장한다:

1. `/docs-extract --type all` — 소스코드에서 문서 자동 추출
2. `/agents-md --generate` — AGENTS.md 생성
3. `/docs-diagnose` — 초기 품질 진단 실행
