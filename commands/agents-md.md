# agents-md 명령어

AGENTS.md를 생성, 검증, 갱신한다. AGENTS.md는 에이전트를 위한 프로젝트 지도 역할을 하며, 100줄 이하의 간결한 포인터 문서여야 한다.

## 사용법

```
/agents-md <action> [options]
```

## 액션

| 액션 | 설명 |
|------|------|
| `--generate` | index.yml + 프로젝트 분석으로 AGENTS.md 생성 |
| `--validate` | 기존 AGENTS.md 검증 |
| `--update` | 현재 index.yml 기반으로 포인터 갱신 |

## 옵션

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `--output <path>` | AGENTS.md 출력 경로 | 프로젝트 루트 `AGENTS.md` |
| `--max-lines <n>` | 최대 허용 라인 수 | 100 |
| `--force` | 기존 AGENTS.md 덮어쓰기 | false |

## --generate: AGENTS.md 생성

### 생성 프로세스

1. **프로젝트 메타데이터 수집**
   - `package.json`, `pyproject.toml`, `go.mod` 등에서 프로젝트명, 설명, 의존성을 추출한다.
   - `.harness/config.yml`에서 언어, 소스 디렉토리 설정을 읽는다.

2. **기술 스택 분석**
   - 주요 의존성에서 프레임워크, 라이브러리를 식별한다.
   - 런타임 환경(Node.js, Python, Go)을 파악한다.
   - 빌드 도구, 테스트 프레임워크를 탐지한다.

3. **디렉토리 구조 생성**
   - 프로젝트 루트의 핵심 디렉토리를 1-2 depth로 트리 표시한다.
   - `node_modules/`, `dist/`, `.git/` 등은 제외한다.
   - 각 디렉토리의 역할을 한 줄로 설명한다.

4. **핵심 문서 선별**
   - `docs/index.yml`에서 `status: validated` 문서를 우선 선별한다.
   - 타입별로 가장 중요한 문서를 1-2개씩 선별한다.
   - 각 문서에 대해 한 줄 설명과 경로를 기록한다.

5. **코딩 컨벤션 추출**
   - ESLint, Prettier, EditorConfig 설정에서 핵심 규칙을 추출한다.
   - `docs/standards/` 내 문서가 있으면 포인터를 추가한다.
   - 규칙이 많으면 상세 문서를 참조하도록 포인터만 남긴다.

6. **AGENTS.md 조립**
   - 위 정보를 100줄 이내로 조합한다.
   - 라인 수가 초과하면 덜 중요한 항목을 축약하거나 포인터로 대체한다.

### 생성되는 AGENTS.md 구조

```markdown
# AGENTS.md

## 프로젝트 개요
[프로젝트명]은 [1-2문장 설명].

## 기술 스택
- 언어: TypeScript 5.x
- 프레임워크: Next.js 14 (App Router)
- DB: PostgreSQL + Prisma ORM
- 테스트: Vitest + Playwright

## 디렉토리 구조
src/
├── app/          # Next.js App Router 페이지
├── components/   # React 컴포넌트
├── lib/          # 핵심 비즈니스 로직
├── models/       # 데이터 모델, DTO
└── utils/        # 유틸리티 함수
docs/             # 프로젝트 문서 (아래 참조)
tests/            # 테스트 코드

## 핵심 문서
- [아키텍처 개요](docs/architecture/overview.md) — 시스템 구조와 레이어
- [인증 플로우](docs/architecture/auth-flow.md) — JWT 인증 전체 흐름
- [API 스펙](docs/specs/api-v2.md) — REST API 엔드포인트 정의
- [용어집](docs/glossary/) — 도메인 용어 정의

## 코딩 컨벤션
- 상세: [코딩 표준](docs/standards/coding-conventions.md)
- 네이밍: camelCase (변수/함수), PascalCase (타입/클래스)
- 커밋: Conventional Commits (feat/fix/chore/docs)

## 작업 시 참고사항
- 새 API 추가 시 → docs/specs/에 스펙 문서 작성
- 새 타입 추가 시 → docs/glossary/에 용어 등록
- PR 전 → `/docs-diagnose` 실행하여 문서 품질 확인
```

## --validate: AGENTS.md 검증

기존 AGENTS.md가 하네스 원칙을 준수하는지 검증한다.

### 검증 항목

| 검사 | 심각도 | 설명 |
|------|--------|------|
| 파일 존재 | error | AGENTS.md가 프로젝트 루트에 있는지 |
| 라인 수 | warning | 100줄 이하인지 (기본값, `--max-lines`로 조정 가능) |
| 프로젝트 개요 | warning | "프로젝트 개요" 또는 "Overview" 섹션 존재 |
| 디렉토리 구조 | warning | "디렉토리" 또는 "Directory" 섹션 존재 |
| 핵심 문서 | warning | "문서" 또는 "Documentation" 섹션에 docs/ 링크 존재 |
| 외부 링크 | error | Notion, Google Docs, Confluence 등 외부 URL 없음 |
| 깨진 링크 | error | 참조하는 내부 파일이 실제로 존재하는지 |
| 최소 포인터 | info | docs/ 참조 링크 3개 이상 |

### 검증 출력

```
[agents-md] AGENTS.md 검증 결과

✓ 파일 존재: AGENTS.md (78줄)
✓ 라인 수: 78/100 이내
✓ 프로젝트 개요 섹션 존재
✓ 디렉토리 구조 섹션 존재
✓ 핵심 문서 섹션 존재 (포인터 5개)
✓ 외부 링크 없음
⚠ 깨진 링크 1개: docs/specs/old-api.md (파일 없음)
✓ 최소 포인터 충족

결과: 통과 (경고 1건)
```

## --update: 포인터 갱신

현재 `docs/index.yml` 상태를 기반으로 AGENTS.md의 핵심 문서 섹션을 갱신한다.

### 갱신 프로세스

1. 기존 AGENTS.md를 읽는다.
2. "핵심 문서" (또는 "Documentation", "Key Documents") 섹션을 식별한다.
3. `docs/index.yml`에서 `status: validated` 문서를 타입별로 수집한다.
4. 새 포인터 목록을 생성한다.
5. 기존 섹션을 새 포인터로 교체한다.
6. 라인 수가 100줄을 초과하지 않도록 조정한다.
7. 변경 사항을 출력한다.

### 갱신 출력

```
[agents-md] AGENTS.md 포인터 갱신

변경 사항:
  + [결제 API 스펙](docs/specs/payment-api.md) — 추가됨 (신규 validated 문서)
  - [레거시 API](docs/specs/legacy-api.md) — 제거됨 (문서 삭제됨)
  ~ [인증 플로우](docs/architecture/auth-flow.md) — 경로 변경 반영

갱신 전: 78줄 → 갱신 후: 80줄
```

## 주의사항

- AGENTS.md는 사람이 아닌 에이전트를 위한 문서다. 장황한 설명 대신 정확한 포인터를 제공한다.
- "백과사전이 아닌 지도"를 만든다. 상세한 내용은 docs/ 내 문서에 위임한다.
- 외부 서비스(Notion, Google Docs, Confluence, Jira)의 링크를 절대 포함하지 않는다. 모든 지식은 레포 안에 있어야 한다.
- 100줄은 엄격한 제한이 아닌 강한 권장이다. 대규모 프로젝트에서 120줄까지는 허용하되, 이를 초과하면 반드시 축약한다.
