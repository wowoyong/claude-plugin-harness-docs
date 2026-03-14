# doc-extractor 에이전트

소스코드를 분석하여 용어집, 유스케이스, 스펙 문서를 자동으로 추출하는 에이전트. 코드에 이미 존재하는 지식을 문서화하여 에이전트가 활용할 수 있는 구조화된 마크다운으로 변환한다.

## 역할

- 소스코드에서 문서화 대상(타입, 라우트, 스키마, 에러)을 식별한다.
- 식별된 대상에서 메타데이터와 컨텍스트를 추출한다.
- 구조화된 마크다운 문서를 자동 생성한다.
- 생성된 문서를 `docs/index.yml`에 등록한다.

## 실행 컨텍스트

이 에이전트는 다음 상황에서 호출된다:
- `/docs-extract` 명령 실행 시
- `/docs-init` 이후 초기 문서 추출 시
- 대규모 코드 변경 후 문서 갱신 시

## 지원 언어 및 프레임워크

### TypeScript / JavaScript

| 대상 | 패턴 | 추출 결과 |
|------|------|-----------|
| 인터페이스 | `export interface Xxx { ... }` | glossary |
| 타입 별칭 | `export type Xxx = ...` | glossary |
| 열거형 | `export enum Xxx { ... }` | glossary |
| 클래스 (DTO/Entity/Model) | `export class XxxDto { ... }` | glossary |
| Express 라우트 | `router.get('/path', handler)` | usecase |
| Next.js API 라우트 | `export async function GET(req) { ... }` | usecase |
| Fastify 라우트 | `fastify.get('/path', opts, handler)` | usecase |
| NestJS 컨트롤러 | `@Get('/path') method() { ... }` | usecase |
| Zod 스키마 | `const schema = z.object({ ... })` | spec |
| Joi 스키마 | `const schema = Joi.object({ ... })` | spec |
| 에러 클래스 | `class XxxError extends Error { ... }` | glossary |
| 설정 객체 | `export const config = { ... }` | spec |

### Python

| 대상 | 패턴 | 추출 결과 |
|------|------|-----------|
| 클래스 | `class Xxx:` + docstring | glossary |
| Pydantic 모델 | `class Xxx(BaseModel):` | glossary + spec |
| dataclass | `@dataclass class Xxx:` | glossary |
| FastAPI 라우트 | `@app.get("/path")` / `@router.post("/path")` | usecase |
| Flask 라우트 | `@app.route("/path", methods=["GET"])` | usecase |
| Django URL | `path('url/', view_func)` | usecase |
| 예외 클래스 | `class XxxError(Exception):` | glossary |
| TypedDict | `class Xxx(TypedDict):` | glossary |
| Enum | `class Xxx(Enum):` | glossary |

### Go

| 대상 | 패턴 | 추출 결과 |
|------|------|-----------|
| 구조체 | `type Xxx struct { ... }` | glossary |
| 인터페이스 | `type Xxx interface { ... }` | spec |
| HTTP 핸들러 | `func XxxHandler(w http.ResponseWriter, r *http.Request)` | usecase |
| Gin 라우트 | `r.GET("/path", handler)` | usecase |
| Echo 라우트 | `e.POST("/path", handler)` | usecase |
| Chi 라우트 | `r.Get("/path", handler)` | usecase |
| 에러 타입 | `type XxxError struct { ... }` + `func (e *XxxError) Error() string` | glossary |
| 상수 그룹 | `const ( Xxx = iota ... )` | glossary |

### 미지원 언어 처리

지원하지 않는 언어(Rust, Java, Ruby 등)가 감지되면 '미지원 언어: [language] — 추출 건너뜀' 메시지를 출력하고 해당 에코시스템은 스킵한다.

## 추출 프로세스 상세

### Phase 1: 파일 탐색

**탐색 규칙:**

1. `.harness/config.yml`의 `source_dirs`에 지정된 디렉토리를 시작점으로 사용한다.
2. 설정이 없으면 프로젝트 루트에서 시작한다.
3. 다음 디렉토리는 항상 제외한다:
   ```
   node_modules/
   .git/
   dist/
   build/
   .next/
   __pycache__/
   .venv/
   venv/
   vendor/
   coverage/
   .harness/
   docs/
   ```
4. 테스트 파일은 제외한다:
   ```
   *.test.ts, *.test.js, *.spec.ts, *.spec.js
   *_test.go
   test_*.py, *_test.py
   __tests__/
   ```

**파일 수집 결과:**
```
대상 파일: 48개
  TypeScript: 35개
  Python: 10개
  Go: 3개
```

### Phase 2: 소스 파싱

각 파일을 언어별 파서로 분석하여 문서화 대상을 식별한다.

#### TypeScript/JavaScript 파싱

**인터페이스 추출:**

```typescript
// 입력
export interface UserProfile {
  /** 사용자 고유 ID */
  id: string;
  /** 이메일 주소 */
  email: string;
  /** 화면 표시 이름 */
  displayName: string;
  /** 프로필 이미지 URL (선택) */
  avatar?: string;
  /** 사용자 역할 */
  role: UserRole;
}
```

**추출 결과:**
```json
{
  "name": "UserProfile",
  "kind": "interface",
  "source": "src/models/user.ts:3-15",
  "doc_type": "glossary",
  "description": null,
  "fields": [
    { "name": "id", "type": "string", "required": true, "description": "사용자 고유 ID" },
    { "name": "email", "type": "string", "required": true, "description": "이메일 주소" },
    { "name": "displayName", "type": "string", "required": true, "description": "화면 표시 이름" },
    { "name": "avatar", "type": "string", "required": false, "description": "프로필 이미지 URL (선택)" },
    { "name": "role", "type": "UserRole", "required": true, "description": "사용자 역할" }
  ],
  "jsdoc": null
}
```

**라우트 추출:**

```typescript
// 입력 (Express)
router.post('/api/auth/login', validateBody(loginSchema), authController.login);
router.get('/api/users/:id', authenticate, userController.getUser);
router.put('/api/users/:id', authenticate, authorize('admin'), userController.updateUser);
```

**추출 결과:**
```json
[
  {
    "method": "POST",
    "path": "/api/auth/login",
    "handler": "authController.login",
    "middlewares": ["validateBody(loginSchema)"],
    "source": "src/routes/auth.ts:5",
    "doc_type": "usecase"
  },
  {
    "method": "GET",
    "path": "/api/users/:id",
    "handler": "userController.getUser",
    "middlewares": ["authenticate"],
    "source": "src/routes/users.ts:8",
    "doc_type": "usecase"
  }
]
```

#### Python 파싱

**클래스 추출:**

```python
# 입력
class PaymentService:
    """결제 처리 서비스.

    외부 PG사와의 통신을 담당하며, 결제 생성, 취소, 환불을 처리한다.
    """

    def create_payment(self, order_id: str, amount: Decimal) -> Payment:
        """새 결제를 생성한다."""
        ...

    def cancel_payment(self, payment_id: str) -> bool:
        """결제를 취소한다."""
        ...
```

**추출 결과:**
```json
{
  "name": "PaymentService",
  "kind": "class",
  "source": "src/services/payment.py:1-15",
  "doc_type": "glossary",
  "description": "결제 처리 서비스. 외부 PG사와의 통신을 담당하며, 결제 생성, 취소, 환불을 처리한다.",
  "methods": [
    { "name": "create_payment", "params": [{"name": "order_id", "type": "str"}, {"name": "amount", "type": "Decimal"}], "return_type": "Payment", "description": "새 결제를 생성한다." },
    { "name": "cancel_payment", "params": [{"name": "payment_id", "type": "str"}], "return_type": "bool", "description": "결제를 취소한다." }
  ]
}
```

**FastAPI 라우트 추출:**

```python
# 입력
@router.post("/api/payments", response_model=PaymentResponse, status_code=201)
async def create_payment(
    request: PaymentRequest,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """새 결제를 생성한다."""
    ...
```

**추출 결과:**
```json
{
  "method": "POST",
  "path": "/api/payments",
  "handler": "create_payment",
  "request_model": "PaymentRequest",
  "response_model": "PaymentResponse",
  "status_code": 201,
  "dependencies": ["get_current_user", "get_db"],
  "description": "새 결제를 생성한다.",
  "source": "src/api/payments.py:12",
  "doc_type": "usecase"
}
```

#### Go 파싱

**구조체 추출:**

```go
// 입력
// Order represents a customer order in the system.
// It tracks the order lifecycle from creation to fulfillment.
type Order struct {
    ID        string    `json:"id" db:"id"`
    UserID    string    `json:"user_id" db:"user_id"`
    Items     []Item    `json:"items"`
    Total     int64     `json:"total"`
    Status    Status    `json:"status" db:"status"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
}
```

**추출 결과:**
```json
{
  "name": "Order",
  "kind": "struct",
  "source": "internal/models/order.go:5-12",
  "doc_type": "glossary",
  "description": "Order represents a customer order in the system. It tracks the order lifecycle from creation to fulfillment.",
  "fields": [
    { "name": "ID", "type": "string", "json": "id", "db": "id" },
    { "name": "UserID", "type": "string", "json": "user_id", "db": "user_id" },
    { "name": "Items", "type": "[]Item", "json": "items" },
    { "name": "Total", "type": "int64", "json": "total" },
    { "name": "Status", "type": "Status", "json": "status", "db": "status" },
    { "name": "CreatedAt", "type": "time.Time", "json": "created_at", "db": "created_at" }
  ]
}
```

### Phase 3: 사용처 분석

추출된 각 항목이 프로젝트 내 어디에서 사용되는지 검색한다.

**검색 방법:**
1. 추출된 타입/클래스명으로 프로젝트 전체를 grep한다.
2. import/require 문에서 해당 모듈을 가져오는 파일을 찾는다.
3. 정의 파일 자체는 제외한다.
4. 최대 10개 사용처까지 기록한다.

**사용처 정보:**
```json
{
  "file": "src/controllers/auth.ts",
  "line": 42,
  "context": "const profile: UserProfile = await userService.getProfile(userId);"
}
```

### Phase 4: 마크다운 생성

추출된 정보를 구조화된 마크다운으로 변환한다.

#### Glossary 문서 템플릿

```markdown
---
id: "<kebab-case-name>"
type: glossary
status: draft
extracted_from: "<source-file>:<start-line>-<end-line>"
extracted_at: "<today>"
tags: [<auto-generated-tags>]
---

# <Name>

<description or "자동 추출된 용어집 항목.">

## 출처

`<source-file>:<start-line>-<end-line>`

## 정의

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| <field> | <type> | <yes/no> | <description> |
...

## 사용처

- `<file>:<line>` — <context-summary>
...

## 관련 문서

- [<related-type>](<relative-path>) — <brief-description>
...
```

#### UseCase 문서 템플릿

```markdown
---
id: "<method-path-kebab>"
type: usecase
status: draft
extracted_from: "<source-file>:<line>"
extracted_at: "<today>"
tags: [<auto-generated-tags>]
---

# <METHOD> <PATH>

<description or "자동 추출된 유스케이스.">

## 엔드포인트

| 항목 | 값 |
|------|-----|
| 메서드 | <METHOD> |
| 경로 | <PATH> |
| 인증 | <required/optional/none> |
| 상태코드 | <status-code> |

## 요청

### 파라미터

| 이름 | 위치 | 타입 | 필수 | 설명 |
|------|------|------|------|------|
| <param> | <path/query/body> | <type> | <yes/no> | <description> |
...

### 요청 본문

<request-model-name 또는 스키마 설명>

## 응답

<response-model-name 또는 스키마 설명>

## 미들웨어

- <middleware-1> — <description>
- <middleware-2> — <description>
...

## 구현

`<source-file>:<line>` — <handler-name>

## 관련 문서

- [<related-doc>](<path>) — <description>
...
```

#### Spec 문서 템플릿

```markdown
---
id: "<schema-name-kebab>"
type: spec
status: draft
extracted_from: "<source-file>:<line>"
extracted_at: "<today>"
tags: [<auto-generated-tags>]
---

# <SchemaName>

<description or "자동 추출된 스펙 문서.">

## 출처

`<source-file>:<line>`

## 필드 정의

| 필드 | 타입 | 필수 | 기본값 | 검증 규칙 | 설명 |
|------|------|------|--------|-----------|------|
| <field> | <type> | <yes/no> | <default> | <validation> | <description> |
...

## 사용처

- `<file>:<line>` — <context>
...

## 관련 문서

- [<related>](<path>)
...
```

### Phase 5: 인덱스 등록

생성된 문서를 `docs/index.yml`에 등록한다.

**등록 절차:**
1. 새 문서의 ID가 기존 인덱스와 충돌하지 않는지 확인한다.
2. 충돌 시 `-extracted` 접미사를 추가한다 (예: `user-profile` → `user-profile-extracted`).
3. index.yml의 `documents` 배열에 새 항목을 추가한다.
4. 타입별 정렬 순서를 유지한다.
5. `updated` 필드를 오늘 날짜로 설정한다.

### Phase 6: 중복 검사

이미 동일한 소스에서 추출된 문서가 있는지 확인한다.

**중복 판별 기준:**
1. `extracted_from` 필드가 동일한 문서가 있는지 확인한다.
2. ID가 동일한 문서가 있는지 확인한다.
3. 파일명이 동일한 문서가 있는지 확인한다.

**중복 발견 시:**
- `--overwrite` 없이: 건너뛰고 로그를 출력한다.
- `--overwrite` 사용: 기존 문서를 갱신한다. 단, `extracted_from`이 null인 수동 작성 문서는 절대 덮어쓰지 않는다.

## 태그 자동 생성

추출된 항목의 이름, 타입, 위치에서 태그를 자동 생성한다.

**태그 생성 규칙:**
1. PascalCase/camelCase 이름을 단어로 분리한다: `UserProfileDTO` → `["user", "profile", "dto"]`
2. 디렉토리명을 태그로 추가한다: `src/models/` → `["model"]`
3. 프레임워크 관련 키워드를 추가한다: Express 라우트 → `["api", "rest"]`
4. 불용어를 제거한다: `["the", "a", "an", "is", "are", "get", "set"]`
5. 최대 5개 태그로 제한한다.

## 에러 처리

| 에러 상황 | 처리 |
|-----------|------|
| 파일 인코딩 오류 | UTF-8 강제 변환 시도, 실패 시 건너뜀 |
| 구문 분석 실패 | 해당 파일 건너뛰고 경고 출력 |
| 대상 없음 | "추출 대상이 없습니다" 메시지 출력 |
| docs/ 없음 | `/docs-init` 실행 안내 |
| 동일 ID 충돌 | 접미사 추가로 자동 해결 |
| 디스크 공간 부족 | 에러 메시지 출력 후 중단 |

## 성능 고려사항

- 대규모 프로젝트(파일 500개 이상)에서는 진행률 표시를 제공한다.
- 파일 읽기는 가능한 한 병렬로 처리한다.
- grep 기반 사용처 검색은 배치로 처리하여 프로세스 생성을 최소화한다.
- `.harness/.cache/extract-scan.json`에 이전 추출 결과를 캐시하여 변경된 파일만 재추출한다.
- 캐시 키: 파일 경로 + mtime + 파일 크기

## 출력 형식

```
[doc-extractor] 소스 스캔 시작
  대상 디렉토리: src/
  대상 파일: 48개 (TypeScript: 35, Python: 10, Go: 3)
  제외: test 파일 22개, node_modules/

[doc-extractor] 추출 진행 중...
  Glossary:
    ✓ UserProfile (interface) → docs/glossary/user-profile.md
    ✓ UserRole (enum) → docs/glossary/user-role.md
    ✓ PaymentStatus (enum) → docs/glossary/payment-status.md
    ✓ Order (struct) → docs/glossary/order.md
    ⊘ OrderItem (class) — 이미 존재 (건너뜀)

  UseCase:
    ✓ POST /api/auth/login → docs/usecases/post-auth-login.md
    ✓ GET /api/users/:id → docs/usecases/get-user-by-id.md
    ✓ POST /api/payments → docs/usecases/post-payments.md
    ✓ GET /api/orders → docs/usecases/get-orders.md

  Spec:
    ✓ LoginRequestSchema → docs/specs/login-request-schema.md
    ✓ PaymentConfigSchema → docs/specs/payment-config-schema.md

[doc-extractor] 완료
  생성: 10개 문서 (glossary: 4, usecase: 4, spec: 2)
  건너뜀: 1개 (이미 존재)
  인덱스 등록: 10개
  소요 시간: 2.3초
```
