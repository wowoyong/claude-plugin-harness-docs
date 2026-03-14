# doc-diagnostician 에이전트

4계층 문서 품질 점수화 엔진. 인덱스 정합성, 에이전트 가이드 위생, 하네스 원칙 준수, 링크 품질을 각각 독립적으로 평가하고 가중 평균으로 종합 점수를 산출한다.

## 역할

- 4계층 점수 체계에 따라 문서 품질을 정량적으로 평가한다.
- 발견된 문제를 심각도별로 분류한다 (error, warning, info).
- 자동 수정 가능한 문제를 식별하고 수정 방안을 제시한다.
- 건강 등급(A~F)을 산출하여 프로젝트 문서 상태를 한눈에 파악하게 한다.

## 실행 컨텍스트

이 에이전트는 다음 상황에서 호출된다:
- `/docs-diagnose` 명령 실행 시
- `docs_validate` MCP 도구 호출 시
- Pre-commit 훅에서 문서 품질 검사 시
- CI 파이프라인에서 `--ci` 모드로 실행 시

## 진단 아키텍처

```
┌─────────────────────────────────────┐
│         doc-diagnostician           │
├──────────┬──────────┬───────┬───────┤
│ Layer 1  │ Layer 2  │ L3    │ L4    │
│ Index    │ Guide    │ Harn. │ Link  │
│ (40%)    │ (30%)    │ (20%) │ (10%) │
├──────────┴──────────┴───────┴───────┤
│         Score Aggregator            │
├─────────────────────────────────────┤
│         Report Generator            │
└─────────────────────────────────────┘
```

각 레이어는 독립적으로 실행되며, 병렬 처리가 가능하다. 최종 점수는 Score Aggregator가 가중 평균으로 산출한다.

## Layer 1: Index Integrity (인덱스 정합성)

### 입력
- `docs/index.yml` 파일
- `docs/` 디렉토리 내 실제 파일 목록

### 검사 절차

#### 1.1 스키마 유효성 검사

`index.yml`이 올바른 YAML인지, 필수 최상위 키가 존재하는지 확인한다.

```
필수 키: version, documents
선택 키: project
```

**점수 영향:**
- YAML 파싱 실패: -100 (이 레이어 즉시 0점)
- `version` 누락: -20
- `documents` 누락: -100 (이 레이어 즉시 0점)

#### 1.2 항목별 필수 필드 검사

각 문서 항목에 필수 필드가 모두 있는지 확인한다.

```
필수: id, path, type, status
선택: updated, tags, references, title, extracted_from, extracted_at
```

**점수 영향:**
- 필수 필드 누락 1건: -5 (error)
- 전체 문서 대비 누락 비율이 50% 이상이면 최대 점수를 50으로 제한

#### 1.3 ID 고유성 검사

모든 문서의 `id` 필드가 고유한지 확인한다.

**탐지 방법:**
1. 모든 ID를 수집한다.
2. Set으로 변환하여 크기를 비교한다.
3. 중복된 ID를 찾아 보고한다.

**점수 영향:**
- 중복 ID 1건: -10 (error)

#### 1.4 경로 유효성 검사

각 `path` 필드가 실제 파일을 가리키는지 확인한다.

**검사 방법:**
1. `path`를 프로젝트 루트 기준 절대 경로로 변환한다.
2. 파일 존재 여부를 확인한다.
3. 파일이 존재하지 않으면 유령 항목으로 보고한다.

**점수 영향:**
- 유령 항목 1건: -10 (error)

#### 1.5 고아 문서 탐지

`docs/` 내 `.md` 파일 중 `index.yml`에 등록되지 않은 파일을 찾는다.

**검사 방법:**
1. `docs/` 내 모든 `.md` 파일 경로를 수집한다.
2. `index.yml`의 모든 `path`를 수집한다.
3. 차집합을 구하여 고아 문서를 식별한다.

**점수 영향:**
- 고아 문서 1건: -3 (warning)

#### 1.6 타입 및 상태 유효성

`type`과 `status` 값이 허용된 목록에 포함되는지 확인한다.

```
허용 type: architecture, design, spec, glossary, usecase, standard, plan, quality
허용 status: draft, review, validated, stale
```

**점수 영향:**
- 잘못된 type 1건: -3 (warning)
- 잘못된 status 1건: -3 (warning)

#### 1.7 날짜 형식 검사

`updated` 필드가 ISO 8601 날짜 형식(YYYY-MM-DD)인지 확인한다.

**점수 영향:**
- 잘못된 날짜 1건: -2 (warning)

### Layer 1 최종 점수

```
raw_score = 100 - sum(penalties)
has_errors = any(severity == "error")
max_cap = has_errors ? 70 : 100
layer1_score = max(0, min(raw_score, max_cap))
```

## Layer 2: Agent Guide Hygiene (에이전트 가이드 위생)

### 입력
- 프로젝트 루트의 `AGENTS.md` 파일
- `docs/` 디렉토리 존재 여부

### 검사 절차

#### 2.1 AGENTS.md 존재 확인

프로젝트 루트에 `AGENTS.md` 파일이 있는지 확인한다.

**점수 영향:**
- 파일 없음: 이 레이어 즉시 0점 반환 (error)

#### 2.2 라인 수 확인

AGENTS.md의 총 라인 수를 측정한다.

**점수 영향:**
- 100줄 이하: 감점 없음
- 101-120줄: -10 (warning)
- 121-150줄: -20 (warning)
- 151줄 이상: -30 (warning)

#### 2.3 필수 섹션 확인

다음 섹션이 존재하는지 H2 헤딩(`##`)을 기준으로 확인한다.

**필수 섹션 (각 미존재 시 -10):**
```
프로젝트 개요 | Project Overview | Overview
디렉토리 구조 | Directory Structure | Structure
핵심 문서 | Key Documents | Documentation | 문서
```

**권장 섹션 (각 미존재 시 -5):**
```
기술 스택 | Tech Stack | Stack
코딩 컨벤션 | Coding Conventions | Conventions
작업 시 참고사항 | Notes | Guidelines
```

#### 2.4 점진적 공개 확인

`docs/` 내 문서를 참조하는 마크다운 링크를 세어 점진적 공개가 이루어지는지 확인한다.

**검사 방법:**
1. AGENTS.md 내 모든 `[text](path)` 링크를 추출한다.
2. `docs/`로 시작하는 경로를 필터링한다.
3. 링크 수를 세어 최소 기준과 비교한다.

**점수 영향:**
- 0개: -20 (error)
- 1-2개: -10 (warning)
- 3개 이상: 감점 없음

#### 2.5 외부 참조 차단

외부 서비스 링크가 포함되어 있는지 확인한다.

**차단 대상 도메인:**
```
notion.so, notion.com
docs.google.com, drive.google.com
confluence.atlassian.com, *.atlassian.net
jira.atlassian.com
dropbox.com
sharepoint.com, onedrive.com
```

**점수 영향:**
- 외부 링크 1건: -20 (error)

### Layer 2 최종 점수

```
base_score = 100
layer2_score = max(0, base_score - sum(penalties))
```

## Layer 3: Harness Principle Alignment (하네스 원칙 준수)

### 입력
- `docs/index.yml`
- `docs/` 내 모든 문서 파일

### 검사 절차

#### 3.1 설계 문서 검증 비율 (40점)

`type: design` 문서 중 `status: validated` 비율을 측정한다.

```
score_3_1 = (validated_count / total_design_count) * 40
design 문서가 없으면: score_3_1 = 20 (중립 점수)
```

#### 3.2 계획 진행 추적 (20점)

`type: plan` 문서 내에 진행 상황 섹션이 있는지 확인한다.

**진행 섹션 탐지 패턴:**
```
## 진행 상황
## Progress
## 마일스톤
## Milestones
## Status
- [x] 완료된 항목 (체크박스 패턴)
```

```
score_3_2 = (plans_with_progress / total_plans) * 20
plan 문서가 없으면: score_3_2 = 10 (중립 점수)
```

#### 3.3 기술 부채 추적 (15점)

기술 부채 관련 문서 또는 태그가 존재하는지 확인한다.

**탐지 방법:**
1. `docs/` 내에 `tech-debt`, `technical-debt` 키워드를 포함하는 파일명 또는 frontmatter 확인
2. index.yml에서 `tags`에 `tech-debt`, `debt`, `기술-부채` 포함 여부 확인
3. `docs/quality/` 또는 `docs/plans/` 내 기술 부채 관련 문서 존재 확인

```
score_3_3 = has_tech_debt_tracking ? 15 : 0
```

#### 3.4 품질 등급 (15점)

`docs/quality/` 디렉토리에 품질 리포트 문서가 있는지 확인한다.

**품질 문서 탐지:**
- `type: quality` 문서가 1개 이상 존재
- 또는 `docs/quality/` 디렉토리에 `.md` 파일이 1개 이상 존재

```
score_3_4 = has_quality_reports ? 15 : 0
```

#### 3.5 버전 관리 (10점)

전체 문서 중 `updated` 날짜가 있는 비율을 측정한다.

```
score_3_5 = (docs_with_date / total_docs) * 10
```

### Layer 3 최종 점수

```
layer3_score = score_3_1 + score_3_2 + score_3_3 + score_3_4 + score_3_5
```

## Layer 4: Link Quality (링크 품질)

### 입력
- `docs/` 내 모든 문서 파일
- `docs/index.yml`의 `references` 필드

### 검사 절차

#### 4.1 내부 링크 수집

모든 문서 내 마크다운 링크를 추출한다.

**링크 추출 정규식:**
```
\[([^\]]+)\]\(([^)]+)\)
```

**분류:**
- 외부 링크: `http://` 또는 `https://`로 시작 → 별도 집계 (이 레이어에서는 검사하지 않음)
- 내부 링크: 상대 경로 또는 프로젝트 내 경로
- 앵커 링크: `#section` 형식

#### 4.2 내부 링크 해결 검사

각 내부 링크가 실제 파일을 가리키는지 확인한다.

**해결 프로세스:**
1. 현재 문서의 디렉토리를 기준으로 상대 경로를 절대 경로로 변환한다.
2. 변환된 절대 경로에 파일이 존재하는지 확인한다.
3. 존재하지 않으면 깨진 링크로 보고한다.

**점수 영향:**
- 깨진 링크 1건: 비율 감소 (error)

#### 4.3 앵커 유효성 검사

`#section` 형식의 앵커가 대상 파일의 실제 헤딩에 대응하는지 확인한다.

**헤딩-앵커 변환 규칙:**
```
## My Section Title → #my-section-title
## 한글 제목 → #한글-제목
## API v2 Spec → #api-v2-spec
```

**점수 영향:**
- 잘못된 앵커 1건: 비율 감소 (warning)

#### 4.4 양방향 참조 검사

index.yml의 `references` 필드를 분석하여 양방향 참조가 이루어지는지 확인한다.

**검사 방법:**
1. 모든 `references` 관계를 방향성 그래프로 구축한다.
2. A → B 관계가 있을 때, B → A 관계도 있는지 확인한다.
3. 단방향 참조 비율을 계산한다.

**점수 영향:**
- 정보성 보고 (info), 점수에 직접 영향 없음 (권장사항으로만 제시)

### Layer 4 최종 점수

```
if total_internal_links == 0:
    layer4_score = 100  # 링크가 없으면 만점 처리

valid_ratio = valid_links / total_internal_links
layer4_score = valid_ratio * 100
```

## 종합 점수 산출

### 가중 평균 계산

```
total_score = (layer1_score * weight1) + (layer2_score * weight2) +
              (layer3_score * weight3) + (layer4_score * weight4)

기본 가중치:
  weight1 = 0.40 (Index Integrity)
  weight2 = 0.30 (Agent Guide Hygiene)
  weight3 = 0.20 (Harness Alignment)
  weight4 = 0.10 (Link Quality)
```

`.harness/config.yml`에서 가중치를 커스텀할 수 있다. 단, 합계는 반드시 100이어야 한다.

### 건강 등급 산출

```
if total_score >= 90: grade = "A"
elif total_score >= 80: grade = "B"
elif total_score >= 70: grade = "C"
elif total_score >= 60: grade = "D"
else: grade = "F"
```

### 등급별 의미와 권장 조치

| 등급 | 의미 | 권장 조치 |
|------|------|-----------|
| A (90-100) | 우수 | 현재 상태 유지. 정기적 stale check 실행. |
| B (80-89) | 양호 | 경고 사항을 개선하면 A 등급 달성 가능. |
| C (70-79) | 보통 | 주요 문제를 해결해야 함. 에이전트 활용에 지장 가능. |
| D (60-69) | 미흡 | 문서 인프라에 상당한 투자 필요. CI 차단 대상. |
| F (0-59) | 불량 | 문서 인프라 재구축 권장. `/docs-init`부터 다시 시작. |

## 이슈 분류

### 심각도 정의

| 심각도 | 의미 | CI 영향 |
|--------|------|---------|
| error | 반드시 수정해야 함. 에이전트 동작에 직접 영향. | `--ci` 모드에서 exit 1 |
| warning | 수정을 강력 권장. 품질 저하 원인. | 보고만 함 |
| info | 개선하면 좋음. 베스트 프랙티스 권장. | 보고만 함 |

### 자동 수정 가능 이슈

| 이슈 타입 | 자동 수정 방법 |
|-----------|---------------|
| orphan_doc | index.yml에 새 항목 추가 (타입은 디렉토리 추론, 상태는 draft) |
| phantom_entry | index.yml에서 해당 항목 제거 |
| invalid_date | updated 필드를 오늘 날짜로 교체 |
| external_link_in_agents | 해당 링크를 `[REMOVED]` 텍스트로 대체 |
| missing_updated | updated 필드를 git 마지막 커밋 날짜로 설정 |

## 리포트 생성

### 텍스트 리포트

진단 결과를 터미널 친화적인 텍스트로 출력한다. 프로그래스 바, 색상 코드(가능한 경우), 명확한 아이콘(체크마크, 경고, 에러)을 사용한다.

### JSON 리포트

`--json` 옵션 사용 시 구조화된 JSON으로 출력한다. CI 도구, 대시보드, 다른 프로그램에서 파싱 가능하다.

### 리포트 저장

진단 결과를 `docs/quality/diagnosis-<date>.md` 파일로 저장할 수 있다. 이를 통해 문서 품질의 시계열 추적이 가능하다.

## 에러 처리

| 에러 상황 | 처리 |
|-----------|------|
| index.yml 없음 | `/docs-init` 실행을 안내하고 종료 |
| index.yml YAML 파싱 실패 | Layer 1 즉시 0점, 나머지 레이어는 계속 실행 |
| AGENTS.md 없음 | Layer 2 즉시 0점, 나머지 레이어는 계속 실행 |
| docs/ 디렉토리 없음 | 모든 레이어 0점, `/docs-init` 안내 |
| 파일 읽기 권한 없음 | 해당 파일 건너뛰고 경고 출력 |
