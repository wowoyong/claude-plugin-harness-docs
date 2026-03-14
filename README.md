# harness-docs

에이전트를 위한 레포 내 문서 관리 시스템. OpenAI Harness Engineering과 MyRealTrip의 docs-tree-tools에서 영감을 받아, 모든 지식을 레포 안에 버전 관리되는 아티팩트로 유지한다.

## 핵심 개념

> "에이전트에게 백과사전이 아니라 지도를 줘라."

- AGENTS.md는 100줄 이내의 프로젝트 지도 역할을 한다.
- 모든 문서는 `docs/` 하위에 구조화되어 위치한다.
- 중앙 `index.yml`이 모든 문서를 추적한다.
- 4계층 진단 시스템으로 문서 품질을 정량적으로 관리한다.
- 소스코드에서 문서를 자동 추출하여 코드와 문서의 동기화를 유지한다.

## 주요 기능

| 기능 | 명령어 | 설명 |
|------|--------|------|
| 초기화 | `/docs-init` | docs/ 구조 생성, index.yml 초기화, 설정 파일 생성 |
| 품질 진단 | `/docs-diagnose` | 4계층 점수 체계로 문서 품질 진단 (등급 A~F) |
| 자동 추출 | `/docs-extract` | 소스코드에서 용어집, 유스케이스, 스펙 문서 자동 생성 |
| AGENTS.md | `/agents-md` | AGENTS.md 생성, 검증, 갱신 |

## 설치

```bash
# Claude Code 플러그인으로 설치
git clone https://github.com/jojaeyong/claude-plugin-harness-docs.git
```

프로젝트의 `.claude/plugins/` 또는 글로벌 플러그인 경로에 클론한다.

## 사용법

### 1. 문서 인프라 초기화

```
/docs-init
```

프로젝트에 `docs/` 구조, `index.yml`, `.harness/config.yml`을 생성한다. 기존 마크다운 파일이 있으면 자동으로 인덱싱한다.

```
/docs-init --force --lang typescript,python
```

기존 구조를 덮어쓰고, 언어를 직접 지정할 수도 있다.

### 2. 소스코드에서 문서 추출

```
/docs-extract --type all
```

소스코드를 스캔하여 타입 정의(용어집), API 라우트(유스케이스), 스키마(스펙) 문서를 자동 생성한다.

```
/docs-extract --type glossary --source src/models/
```

특정 타입과 경로만 지정하여 추출할 수도 있다.

### 3. AGENTS.md 생성

```
/agents-md --generate
```

index.yml과 프로젝트 분석 결과를 기반으로 100줄 이내의 AGENTS.md를 자동 생성한다.

```
/agents-md --validate
```

기존 AGENTS.md가 규칙(라인 수, 외부 링크 차단, 필수 섹션)을 준수하는지 검증한다.

### 4. 문서 품질 진단

```
/docs-diagnose
```

4계층 점수 체계로 문서 품질을 진단한다.

```
/docs-diagnose --json --ci
```

CI 파이프라인에서 JSON 결과를 출력하고, D 등급 이하 시 빌드를 실패시킨다.

```
/docs-diagnose --fix
```

자동 수정 가능한 문제(고아 문서 등록, 유령 항목 제거 등)를 즉시 수정한다.

## MCP 서버 도구

에이전트가 작업 중 호출할 수 있는 4가지 도구를 제공한다.

| 도구 | 설명 | 사용 시점 |
|------|------|-----------|
| `docs_lookup` | 문서 ID/키워드로 검색 | 특정 개념이나 API 문서를 찾을 때 |
| `docs_register` | 새 문서를 인덱스에 등록 | 문서 작성 후 인덱스에 추가할 때 |
| `docs_validate` | 인덱스 정합성 검증 | 문서 변경 후 전체 정합성 확인 시 |
| `docs_stale_check` | 오래된 문서 탐지 | 코드 변경 후 관련 문서 최신 여부 확인 시 |

## index.yml 스키마

모든 문서를 추적하는 중앙 인덱스.

```yaml
version: "1.0"
project:
  name: "my-project"
  description: "프로젝트 설명"
  updated: "2026-03-14"

documents:
  - id: "arch-overview"           # 고유 ID (kebab-case)
    path: "docs/architecture/overview.md"  # 파일 경로
    type: architecture            # 문서 타입
    status: validated             # 상태: draft | review | validated | stale
    updated: "2026-03-14"         # 마지막 업데이트
    tags: ["system", "layers"]    # 태그
    references: ["arch-layers"]   # 참조 문서 ID
```

### 문서 타입

| 타입 | 디렉토리 | 설명 |
|------|----------|------|
| `architecture` | `docs/architecture/` | 시스템 아키텍처 |
| `design` | `docs/design/` | 설계 결정 (ADR) |
| `spec` | `docs/specs/` | API/기능 스펙 |
| `glossary` | `docs/glossary/` | 도메인 용어 |
| `usecase` | `docs/usecases/` | 유스케이스 |
| `standard` | `docs/standards/` | 코딩 표준 |
| `plan` | `docs/plans/` | 계획/마일스톤 |
| `quality` | `docs/quality/` | 품질 리포트 |

## 4계층 진단 점수

| 계층 | 가중치 | 검사 내용 |
|------|--------|-----------|
| Index Integrity | 40% | 인덱스 경로 유효성, 고아/유령 탐지, ID 고유성, 필수 필드 |
| Agent Guide Hygiene | 30% | AGENTS.md 존재, 100줄 제한, 외부 링크 차단, 필수 섹션 |
| Harness Alignment | 20% | 설계 문서 검증 비율, 계획 진행 추적, 기술 부채, 품질 등급 |
| Link Quality | 10% | 내부 링크 해결, 앵커 유효성, 양방향 참조 |

### 건강 등급

| 등급 | 점수 | 의미 |
|------|------|------|
| A | 90-100 | 우수 — 에이전트가 즉시 활용 가능 |
| B | 80-89 | 양호 — 사소한 개선 필요 |
| C | 70-79 | 보통 — 주요 개선 권장 |
| D | 60-69 | 미흡 — CI 차단 대상 |
| F | 0-59 | 불량 — 문서 인프라 재구축 필요 |

## 플러그인 구조

```
claude-plugin-harness-docs/
├── .claude-plugin/
│   └── plugin.json              # 플러그인 메타데이터
├── skills/
│   └── harness-docs/
│       └── SKILL.md             # 전체 워크플로 스킬 정의
├── commands/
│   ├── docs-init.md             # 초기화 명령어
│   ├── docs-diagnose.md         # 품질 진단 명령어
│   ├── docs-extract.md          # 자동 추출 명령어
│   └── agents-md.md             # AGENTS.md 관리 명령어
├── agents/
│   ├── doc-indexer.md           # 인덱스 구축 에이전트
│   ├── doc-diagnostician.md     # 품질 진단 에이전트
│   └── doc-extractor.md         # 소스 추출 에이전트
└── README.md                    # 이 파일
```

## 워크플로 예시

```
# 1. 초기화
/docs-init

# 2. 소스에서 문서 추출
/docs-extract --type all

# 3. AGENTS.md 생성
/agents-md --generate

# 4. 품질 진단
/docs-diagnose

# 5. 문제 자동 수정
/docs-diagnose --fix

# 6. 코드 변경 후 문서 갱신
/agents-md --update
/docs-diagnose
```

## 라이선스

MIT
