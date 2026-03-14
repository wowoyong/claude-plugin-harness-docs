# docs-diagnose 명령어

4계층 점수 체계로 프로젝트의 문서 품질을 진단한다. 인덱스 정합성, 에이전트 가이드 위생, 하네스 원칙 준수, 링크 품질을 종합적으로 평가하여 건강 등급(A~F)을 산출한다.

## 사용법

```
/docs-diagnose [options]
```

## 옵션

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `--json` | JSON 형식으로 결과 출력 | false |
| `--fix` | 자동 수정 가능한 문제를 즉시 수정 | false |
| `--ci` | CI 파이프라인용 종료 코드 반환 (에러 시 exit 1) | false |
| `--layer <name>` | 특정 레이어만 진단 (index, guide, harness, links) | 전체 |
| `--verbose` | 상세 진단 결과 출력 | false |

## 전제 조건

- `docs/index.yml`이 존재해야 한다. 없으면 `/docs-init` 실행을 안내한다.
- `.harness/config.yml`이 있으면 가중치 설정을 읽어온다. 없으면 기본 가중치를 사용한다.

## 진단 절차

### Layer 1: Index Integrity (인덱스 정합성) — 40%

`docs/index.yml`의 구조적 건전성을 검사한다.

**검사 항목:**

1. **스키마 유효성**: index.yml이 올바른 YAML이고 필수 필드(`version`, `documents`)가 있는지 확인한다.
2. **필수 필드 검사**: 각 문서 항목에 `id`, `path`, `type`, `status` 필드가 모두 있는지 확인한다.
3. **ID 고유성**: 모든 문서 ID가 고유한지 확인한다. 중복 ID가 있으면 에러로 보고한다.
4. **경로 유효성**: 각 `path`가 가리키는 파일이 실제로 존재하는지 확인한다.
5. **유령 항목 탐지**: 인덱스에 등록되어 있으나 파일이 없는 항목을 찾는다.
6. **고아 문서 탐지**: `docs/` 내에 존재하나 인덱스에 등록되지 않은 `.md` 파일을 찾는다.
7. **타입 유효성**: `type` 값이 허용된 값 목록에 포함되는지 확인한다.
8. **상태 유효성**: `status` 값이 허용된 값 목록에 포함되는지 확인한다.
9. **날짜 형식**: `updated` 필드가 ISO 8601 형식(YYYY-MM-DD)인지 확인한다.

**자동 수정 가능 (`--fix`):**
- 고아 문서를 인덱스에 추가 (타입은 디렉토리에서 추론, 상태는 `draft`)
- 유령 항목을 인덱스에서 제거
- 잘못된 날짜 형식을 오늘 날짜로 교체

**점수 계산:** 상세 점수 공식은 doc-diagnostician 에이전트의 절차를 따른다.

### Layer 2: Agent Guide Hygiene (에이전트 가이드 위생) — 30%

AGENTS.md의 품질과 에이전트 친화성을 검사한다.

**검사 항목:**

1. **AGENTS.md 존재**: 프로젝트 루트에 AGENTS.md 파일이 있는지 확인한다.
2. **라인 수**: AGENTS.md가 100줄 이하인지 확인한다.
3. **필수 섹션**: 프로젝트 개요, 디렉토리 구조, 핵심 문서 섹션이 존재하는지 확인한다.
4. **점진적 공개**: `docs/` 내 문서를 참조하는 링크가 최소 3개 이상인지 확인한다.
5. **외부 참조 차단**: Notion, Google Docs, Confluence 등 외부 서비스 링크가 없는지 확인한다.
6. **레포 내부 참조**: 모든 링크가 레포 내부 경로를 가리키는지 확인한다.

**자동 수정 가능 (`--fix`):**
- 외부 링크를 `[REMOVED: 외부 링크]`로 대체하고 경고 주석 추가 (`external_link_in_agents`)
- AGENTS.md가 없으면 `/agents-md --generate` 실행을 제안
- `missing_updated`: updated 필드가 없는 문서에 git 마지막 커밋 날짜로 설정

**점수 계산:** 상세 점수 공식은 doc-diagnostician 에이전트의 절차를 따른다.

### Layer 3: Harness Principle Alignment (하네스 원칙 준수) — 20%

OpenAI Harness Engineering 원칙의 적용 수준을 검사한다.

**검사 항목:**

1. **설계 문서 검증 비율**: `design/` 내 문서 중 `status: validated` 비율을 측정한다.
2. **계획 진행 추적**: `plans/` 문서에 "## 진행 상황" 또는 "## Progress" 섹션이 있는지 확인한다.
3. **기술 부채 추적**: tech-debt, 기술-부채 관련 문서 또는 태그가 존재하는지 확인한다.
4. **품질 등급**: `quality/` 디렉토리에 품질 리포트 문서가 있는지 확인한다.
5. **버전 관리**: 전체 문서 중 `updated` 날짜가 있는 비율을 측정한다.
6. **참조 그래프**: 문서 간 `references` 설정 비율과 양방향 참조 비율을 측정한다.

**점수 계산:**
```
validated_ratio = validated_design_docs / total_design_docs (0~1)
progress_exists = plans_with_progress / total_plans (0~1)
debt_tracked = has_tech_debt_docs ? 1 : 0
quality_graded = has_quality_reports ? 1 : 0
versioned_ratio = docs_with_date / total_docs (0~1)

layer3_score = (validated_ratio * 40) + (progress_exists * 20) +
               (debt_tracked * 15) + (quality_graded * 15) +
               (versioned_ratio * 10)
```

### Layer 4: Link Quality (링크 품질) — 10%

문서 내부의 링크 품질을 검사한다.

**검사 항목:**

1. **내부 링크 해결**: `[text](path)` 형식의 마크다운 링크가 실제 파일을 가리키는지 확인한다.
2. **앵커 유효성**: `[text](path#section)` 형식의 앵커 링크가 실제 헤딩에 대응하는지 확인한다.
3. **양방향 참조**: index.yml의 `references` 필드가 양방향인지 확인한다 (A→B이면 B→A도 있는지).
4. **상대 경로 유효성**: 상대 경로 링크(`../`, `./`)가 올바르게 해결되는지 확인한다.

**점수 계산:**
```
valid_links = 유효한 링크 수
total_links = 전체 링크 수
if total_links == 0: layer4_score = 100

layer4_score = (valid_links / total_links) * 100
```

## 종합 점수 산출

```
total = (layer1 * 0.4) + (layer2 * 0.3) + (layer3 * 0.2) + (layer4 * 0.1)
```

## 건강 등급

| 등급 | 점수 | CI 종료 코드 |
|------|------|-------------|
| A | 90-100 | 0 |
| B | 80-89 | 0 |
| C | 70-79 | 0 (`--ci` 사용 시에도 통과) |
| D | 60-69 | 1 (`--ci` 사용 시 실패) |
| F | 0-59 | 1 (`--ci` 사용 시 실패) |

## 출력 형식

### 기본 출력

```
=== 문서 품질 진단 리포트 ===

종합 등급: B (84점)

Layer 1: Index Integrity     [████████░░] 82/100 (가중치 40%)
  ✓ 스키마 유효
  ✓ 필수 필드 완전
  ⚠ 고아 문서 2개: docs/design/cache-strategy.md, docs/specs/legacy-api.md
  ✓ 유령 항목 없음

Layer 2: Agent Guide Hygiene [█████████░] 90/100 (가중치 30%)
  ✓ AGENTS.md 존재 (78줄)
  ✓ 필수 섹션 완전
  ⚠ docs/ 참조 링크 2개 (최소 3개 권장)
  ✓ 외부 참조 없음

Layer 3: Harness Alignment   [████████░░] 75/100 (가중치 20%)
  ✓ 설계 문서 검증 비율: 60%
  ⚠ 계획 진행 추적 미흡
  ✗ 기술 부채 문서 없음
  ✓ 버전 관리: 90%

Layer 4: Link Quality        [█████████░] 95/100 (가중치 10%)
  ✓ 내부 링크 19/20 해결
  ⚠ 깨진 링크 1개: docs/specs/old-api.md#deprecated
  ✓ 양방향 참조: 85%

=== 권장 조치 ===
1. [warning] 고아 문서를 인덱스에 등록하세요 → /docs-diagnose --fix
2. [warning] docs/ 참조 링크를 AGENTS.md에 추가하세요
3. [info] 기술 부채 추적 문서를 생성하세요
```

### JSON 출력 (`--json`)

```json
{
  "grade": "B",
  "total_score": 84,
  "layers": {
    "index_integrity": { "score": 82, "weight": 40, "issues": [...] },
    "agent_guide_hygiene": { "score": 90, "weight": 30, "issues": [...] },
    "harness_alignment": { "score": 75, "weight": 20, "issues": [...] },
    "link_quality": { "score": 95, "weight": 10, "issues": [...] }
  },
  "issues": [
    { "severity": "warning", "layer": "index_integrity", "type": "orphan_doc", "path": "docs/design/cache-strategy.md", "message": "인덱스에 미등록" },
    { "severity": "info", "layer": "harness_alignment", "type": "missing_tech_debt", "message": "기술 부채 추적 문서 없음" }
  ],
  "suggestions": [...]
}
```

## CI 통합

`--ci` 옵션을 사용하면 종료 코드로 결과를 반환한다.

```yaml
# GitHub Actions 예시
- name: 문서 품질 검사
  run: claude "/docs-diagnose --ci"
```

등급 D 또는 F(70점 미만)이면 exit code 1을 반환하여 CI를 실패시킨다. `.harness/config.yml`에서 임계값을 조정할 수 있다.
