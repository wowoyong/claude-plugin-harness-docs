# docs-extract 명령어

소스코드를 분석하여 문서를 자동 추출한다. 타입 정의에서 용어집을, API 라우트에서 유스케이스를, 스키마에서 스펙 문서를 생성하고 `docs/index.yml`에 자동 등록한다.

## 사용법

```
/docs-extract [options]
```

## 옵션

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `--type <type>` | 추출 타입: `glossary`, `usecase`, `spec`, `all` | `all` |
| `--source <path>` | 스캔할 소스 경로 | `.harness/config.yml`의 `source_dirs` |
| `--dry-run` | 실제 파일을 생성하지 않고 추출 결과만 미리보기 | false |
| `--overwrite` | 이미 존재하는 문서를 덮어쓰기 | false |
| `--lang <language>` | 특정 언어만 스캔 | 자동 탐지 |

## 전제 조건

- `docs/` 디렉토리와 `docs/index.yml`이 존재해야 한다. 없으면 `/docs-init` 실행을 안내한다.
- `.harness/config.yml`이 있으면 소스 디렉토리와 제외 패턴을 읽어온다.

## 추출 유형별 절차

### Glossary (용어집) 추출 — `--type glossary`

소스코드의 타입 정의에서 도메인 용어를 추출한다.

**TypeScript/JavaScript 대상:**
- `export interface XxxYyy { ... }` — 인터페이스 정의
- `export type XxxYyy = ...` — 타입 별칭
- `export enum XxxYyy { ... }` — 열거형
- `export class XxxYyy { ... }` — 클래스 (특히 DTO, Entity, Model 접미사)

**Python 대상:**
- `class XxxYyy:` + docstring — 클래스 정의
- `class XxxYyy(BaseModel):` — Pydantic 모델
- `@dataclass class XxxYyy:` — dataclass

**Go 대상:**
- `type XxxYyy struct { ... }` — 구조체
- `type XxxYyy interface { ... }` — 인터페이스

**추출 프로세스:**

1. 소스 디렉토리에서 대상 파일을 탐색한다 (test 파일 제외).
2. 각 파일에서 export된 타입/인터페이스/클래스를 파싱한다.
3. JSDoc, docstring, GoDoc 주석이 있으면 설명으로 사용한다.
4. 필드/프로퍼티를 테이블 형식으로 정리한다.
5. 해당 타입이 사용되는 파일 목록을 검색하여 "사용처" 섹션에 추가한다.
6. `docs/glossary/<kebab-case-name>.md` 파일을 생성한다.
7. `docs/index.yml`에 `type: glossary`, `status: draft`로 등록한다.

**생성 예시:**

```markdown
---
id: "user-profile-dto"
type: glossary
status: draft
extracted_from: "src/models/user.ts"
extracted_at: "2026-03-14"
tags: ["user", "dto", "model"]
---

# UserProfileDTO

사용자 프로필 정보를 담는 데이터 전송 객체.

## 출처

`src/models/user.ts:15-28`

## 필드

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| id | string | yes | 사용자 고유 식별자 (UUID) |
| email | string | yes | 이메일 주소 |
| displayName | string | yes | 화면 표시 이름 |
| avatar | string | no | 프로필 이미지 URL |
| role | UserRole | yes | 사용자 역할 |
| createdAt | Date | yes | 계정 생성 일시 |

## 사용처

- `src/controllers/auth.ts:42` — 로그인 응답 구성
- `src/controllers/profile.ts:18` — 프로필 조회 응답
- `src/services/user.ts:55` — 사용자 정보 변환

## 관련 타입

- [UserRole](user-role.md) — 사용자 역할 열거형
```

### UseCase (유스케이스) 추출 — `--type usecase`

API 라우트 핸들러에서 유스케이스를 추출한다.

**TypeScript/JavaScript 대상:**
- Express: `router.get('/path', handler)`, `app.post('/path', handler)`
- Next.js: `export async function GET(req)`, `export async function POST(req)`
- Fastify: `fastify.get('/path', handler)`

**Python 대상:**
- FastAPI: `@app.get("/path")`, `@router.post("/path")`
- Flask: `@app.route("/path", methods=["GET"])`
- Django: `path('url/', view)` in urls.py

**Go 대상:**
- net/http: `http.HandleFunc("/path", handler)`
- Gin: `r.GET("/path", handler)`
- Echo: `e.POST("/path", handler)`

**추출 프로세스:**

1. 라우트 정의 파일을 탐색한다 (controller, handler, route, api 키워드 포함 파일).
2. HTTP 메서드와 경로 패턴을 추출한다.
3. 핸들러 함수의 파라미터, 응답 타입을 분석한다.
4. 미들웨어(인증, 검증 등)를 식별한다.
5. `docs/usecases/<endpoint-name>.md` 파일을 생성한다.
6. `docs/index.yml`에 `type: usecase`, `status: draft`로 등록한다.

### Spec (스펙) 추출 — `--type spec`

스키마 정의에서 API 스펙을 추출한다.

**대상:**
- Zod 스키마: `z.object({ ... })`
- Joi 스키마: `Joi.object({ ... })`
- JSON Schema 파일
- OpenAPI/Swagger 정의
- GraphQL 스키마 (`.graphql` 파일)
- Pydantic 모델 (검증 규칙 포함)

**추출 프로세스:**

1. 스키마 정의 파일을 탐색한다.
2. 필드명, 타입, 검증 규칙을 파싱한다.
3. 필수/선택 필드를 구분한다.
4. 기본값이 있으면 기록한다.
5. `docs/specs/<schema-name>.md` 파일을 생성한다.
6. `docs/index.yml`에 `type: spec`, `status: draft`로 등록한다.

## 중복 처리

이미 동일한 출처(`extracted_from`)를 가진 문서가 있는 경우:

- `--overwrite` 없이: 건너뛰고 "이미 존재" 로그를 출력한다.
- `--overwrite` 사용: 기존 문서를 갱신하고 `extracted_at`을 오늘 날짜로 업데이트한다.
- 수동 작성된 문서(`extracted_from`이 null)와 충돌하면 항상 건너뛴다.

## 출력

```
[docs-extract] 소스 스캔 시작: src/
  언어: TypeScript
  대상 파일: 48개

[docs-extract] Glossary 추출
  ✓ UserProfileDTO → docs/glossary/user-profile-dto.md
  ✓ UserRole → docs/glossary/user-role.md
  ✓ PaymentStatus → docs/glossary/payment-status.md
  ⊘ OrderItem — 이미 존재 (건너뜀)

[docs-extract] UseCase 추출
  ✓ POST /api/auth/login → docs/usecases/auth-login.md
  ✓ GET /api/users/:id → docs/usecases/get-user.md
  ✓ POST /api/payments → docs/usecases/create-payment.md

[docs-extract] Spec 추출
  ✓ LoginRequestSchema → docs/specs/login-request.md
  ✓ PaymentConfigSchema → docs/specs/payment-config.md

[docs-extract] 완료
  생성: 8개 문서
  건너뜀: 1개 (이미 존재)
  인덱스 등록: 8개
```

## 다음 단계

추출이 완료되면 다음 작업을 권장한다:

1. 생성된 문서를 검토하고 설명을 보강한다.
2. `status`를 `draft` → `validated`로 변경한다.
3. `/docs-diagnose`로 전체 품질을 확인한다.
4. `/agents-md --update`로 AGENTS.md에 반영한다.
