---
title: "실무 패턴 총정리"
date: 2025-02-19 14:00:00 +0900
categories: [Study, FastAPI]
tags: [fastapi, best-practices, summary, python]
---

## 들어가며

회사에서 여러 달 동안 FastAPI 코드를 읽고 작성하며 많은 걸 배웠다.

처음엔 "왜 이렇게 복잡하게?" 생각했던 패턴들이, 이제는 "아, 이래서 이렇게 만들었구나" 이해된다.

**이 시리즈에서 배운 모든 패턴을 정리해보자.**

---

## Part 1-5 빠른 복습

### Part 1: Router 분리 패턴

**핵심:**
- main.py는 조립만
- 기능별로 Router 분리
- prefix와 tags 활용

```python
# main.py
app.include_router(auth.router)
app.include_router(files.router)

# routers/files.py
router = APIRouter(prefix="/api/files", tags=["files"])
```

---

### Part 2: Pydantic 모델 네이밍

**핵심:**
- 용도별 DTO 분리
- Update DTO는 모든 필드 Optional
- Enum은 `str, Enum` 상속

```python
class BoardCreateDto(BaseModel):  # 생성
    title: str

class BoardUpdateDto(BaseModel):  # 수정
    title: Optional[str] = None

class BoardResponseDto(BaseModel):  # 응답
    id: str
    title: str
    created_at: datetime
```

---

### Part 3: 의존성 주입

**핵심:**
- get_db()는 yield로 자동 정리
- get_current_user()는 DB에 의존
- 의존성 체인

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

async def get_current_user(
    authorization: str = Header(None),
    db: Session = Depends(get_db)  # 체인!
) -> User:
    ...
```

---

### Part 4: CRUD 패턴

**핵심:**
- Create: UUID + commit → refresh
- Read: 페이지네이션 필수
- Update: dict(exclude_unset=True)
- Delete: Soft Delete 권장

```python
# Create
field = Field(id=str(uuid.uuid4()), ...)
db.add(field)
db.commit()
db.refresh(field)

# Update
update_data = data.dict(exclude_unset=True)
for key, value in update_data.items():
    setattr(field, key, value)
```

---

### Part 5: 서비스 계층

**핵심:**
- 복잡한 로직은 서비스로 분리
- Public 메서드 1-2개
- Private 메서드 여러 개 (`_` prefix)

```python
class MetadataExtractor:
    def __init__(self, api_key: str):
        self.client = OpenAI(api_key=api_key)

    def extract_metadata(self, db, text):  # Public
        fields = self._load_fields(db)  # Private
        return self._call_llm(text, fields)  # Private
```

---

## 프로젝트별 특징 비교

회사에는 두 프로젝트가 있었고, 각각 특징이 달랐다.

### Project-A: CRUD 중심

**특징:**
- 보드, 콘텐츠 관리 (CRUD)
- 페이지네이션 표준화
- 다국어 처리
- Soft Delete

**구조:**
```
project-a/
├── app.py
├── restful/
│   ├── router_board.py
│   ├── router_content.py
│   ├── model/
│   │   ├── board_model.py
│   │   └── content_model.py
│   └── auth.py
```

**패턴:**
- Router 분리 ✅
- DTO 네이밍 체계 (XxxCreateDto) ✅
- 의존성 주입 ✅
- CRUD 패턴 ✅
- 서비스 계층 ❌ (필요 없음)

---

### Project-B: 파일 처리 중심

**특징:**
- 파일 업로드/처리
- OCR & Vision API
- 동적 메타데이터 시스템
- 서비스 계층 분리

**구조:**
```
project-b/
├── main.py
├── routers/
│   ├── auth.py
│   ├── files.py
│   └── settings.py
├── extraction/  # 서비스 계층
│   ├── metadata_extractor.py
│   ├── image_metadata_extractor.py
│   └── youtube_metadata_extractor.py
├── models.py
└── db_models.py
```

**패턴:**
- Router 분리 ✅
- DTO 네이밍 (Dto 안 붙임) ✅
- 의존성 주입 ✅
- CRUD 패턴 ✅
- 서비스 계층 ✅ (복잡한 로직)

---

## 네이밍 규칙 정리

### 비교표

| 구분 | Project-A | Project-B | 추천 |
|------|-----------|-----------|------|
| **라우터 파일** | `router_board.py` | `files.py` | 팀과 상의 |
| **모델 파일** | `board_model.py` | `models.py` | 프로젝트 규모 |
| **DTO 네이밍** | `BoardCreateDto` | `BoardCreate` | 명시적 (Dto) |
| **폴더명** | `restful/` | `routers/` | `routers/` |
| **서비스 폴더** | ❌ | `extraction/` | 필요시 |

### 내가 선택한 방식

**소규모 프로젝트:**
```
project/
├── main.py
├── routers/
│   ├── users.py
│   └── items.py
├── models.py
└── database.py
```

**중대규모 프로젝트:**
```
project/
├── main.py
├── routers/
│   ├── users.py
│   └── items.py
├── models/
│   ├── user_model.py
│   └── item_model.py
├── services/  # 복잡한 로직
│   └── payment_service.py
└── database.py
```

---

## 핵심 패턴 모음

### 1. 동적 UPDATE 쿼리

**방식 1: exclude_unset (추천)**

```python
@router.put("/{id}")
async def update_item(id: str, data: ItemUpdateDto):
    item = db.query(Item).filter(Item.id == id).first()
    if not item:
        raise HTTPException(404)

    # 전달된 필드만 업데이트
    update_data = data.dict(exclude_unset=True)
    for key, value in update_data.items():
        setattr(item, key, value)

    item.updated_at = datetime.now()
    db.commit()
    return item
```

**방식 2: 수동 체크**

```python
if data.title is not None:
    item.title = data.title
if data.description is not None:
    item.description = data.description
```

---

### 2. Soft Delete vs Hard Delete

**Soft Delete (일반 사용자)**

```python
@router.delete("/{id}")
async def delete_item(id: str):
    item = db.query(Item).filter(Item.id == id).first()
    if not item:
        raise HTTPException(404)

    # Soft Delete
    item.is_active = False
    item.deleted_at = datetime.now()
    db.commit()

    return {"message": "Item deactivated"}
```

**Hard Delete (관리자만)**

```python
@router.delete("/{id}/permanent")
async def delete_item_permanent(
    id: str,
    admin: User = Depends(get_admin_user)  # 관리자 전용
):
    item = db.query(Item).filter(Item.id == id).first()
    if not item:
        raise HTTPException(404)

    # Hard Delete
    db.delete(item)
    db.commit()

    return {"message": "Item permanently deleted"}
```

---

### 3. 페이지네이션 표준

**공통 파라미터 클래스**

```python
# common_deps.py
class PaginationParams:
    def __init__(
        self,
        page: int = Query(1, ge=1),
        page_size: int = Query(20, ge=1, le=100)
    ):
        self.page = page
        self.page_size = page_size
        self.offset = (page - 1) * page_size
```

**사용**

```python
@router.get("")
async def get_items(
    pagination: PaginationParams = Depends(),
    db: Session = Depends(get_db)
):
    # 전체 개수
    total_count = db.query(Item).count()

    # 페이징
    items = db.query(Item)\
        .offset(pagination.offset)\
        .limit(pagination.page_size)\
        .all()

    total_pages = (total_count + pagination.page_size - 1) // pagination.page_size

    return {
        "items": items,
        "pagination": {
            "current_page": pagination.page,
            "page_size": pagination.page_size,
            "total_count": total_count,
            "total_pages": total_pages,
            "has_next": pagination.page < total_pages,
            "has_prev": pagination.page > 1
        }
    }
```

---

### 4. 인증 패턴

**기본 인증**

```python
async def get_current_user(
    authorization: Optional[str] = Header(None),
    db: Session = Depends(get_db)
) -> User:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(401, "인증이 필요합니다.")

    token = authorization.replace("Bearer ", "")
    user_id = parse_token(token)

    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(401, "사용자를 찾을 수 없습니다.")

    return user
```

**선택적 인증**

```python
async def get_current_user_optional(
    authorization: Optional[str] = Header(None),
    db: Session = Depends(get_db)
) -> Optional[User]:
    if not authorization:
        return None
    try:
        return await get_current_user(authorization, db)
    except HTTPException:
        return None
```

**관리자 전용**

```python
async def get_admin_user(
    current_user: User = Depends(get_current_user)
) -> User:
    if not current_user.is_admin:
        raise HTTPException(403, "관리자 권한이 필요합니다.")
    return current_user
```

---

### 5. 에러 처리 표준

```python
try:
    # 비즈니스 로직
    item = db.query(Item).filter(...).first()
    if not item:
        raise HTTPException(404, "Item not found")

    # 작업 수행
    db.commit()

except HTTPException:
    raise  # HTTPException은 그대로 전파

except Exception as e:
    import traceback
    print(traceback.format_exc())  # 로그
    raise HTTPException(500, f"Internal error: {str(e)}")
```

---

## 베스트 프랙티스 체크리스트

### 프로젝트 구조

- [ ] Router 기능별 분리 (`routers/` 폴더)
- [ ] 진입점은 간결 (main.py 20줄 이내)
- [ ] 일관된 네이밍 규칙 (팀 합의)

### 모델

- [ ] Pydantic (API) vs SQLAlchemy (DB) 분리
- [ ] DTO 네이밍: XxxCreate, XxxUpdate, XxxResponse
- [ ] Update DTO는 모든 필드 Optional
- [ ] Enum은 `str, Enum` 상속
- [ ] response_model 명시
- [ ] from_attributes = True (SQLAlchemy 사용 시)

### 의존성 주입

- [ ] get_db()는 yield 사용
- [ ] 모든 DB 작업에 `db: Session = Depends(get_db)`
- [ ] 인증 필요 시 `current_user = Depends(get_current_user)`
- [ ] 의존성 체인 활용

### CRUD

- [ ] Create: UUID 생성, commit → refresh, 201 상태 코드
- [ ] Read 목록: 페이지네이션 필수, 정렬 검증
- [ ] Read 단일: 404 처리, 소유권 확인
- [ ] Update: dict(exclude_unset=True), updated_at 갱신
- [ ] Delete: Soft Delete 우선

### 서비스 계층

- [ ] 복잡한 로직은 서비스 클래스로 분리
- [ ] Public 메서드 1-2개 (외부 인터페이스)
- [ ] Private 메서드 여러 개 (`_` prefix)
- [ ] 설정은 __init__으로 주입

### 에러 처리

- [ ] try-except 블록 사용
- [ ] HTTPException은 그대로 전파
- [ ] Exception은 500으로 변환
- [ ] traceback 로깅

### 코드 품질

- [ ] 타입 힌팅 (Type Hints)
- [ ] Docstring (클래스, Public 메서드)
- [ ] Import 순서 (표준 → 외부 → 내부)

---

## 프로젝트 규모별 선택 가이드

### 소규모 (1-3명, CRUD 위주)

**구조:**
```
project/
├── main.py
├── routers/
│   ├── users.py
│   └── items.py
├── models.py
├── database.py
└── auth.py
```

**사용 패턴:**
- ✅ Router 분리
- ✅ DTO 네이밍
- ✅ 의존성 주입
- ✅ CRUD 패턴
- ❌ 서비스 계층 (필요 없음)

**예시:** 간단한 게시판, TODO 앱

---

### 중규모 (4-10명, 복잡한 로직)

**구조:**
```
project/
├── main.py
├── routers/
│   ├── users.py
│   ├── items.py
│   └── payments.py
├── models/
│   ├── user_model.py
│   ├── item_model.py
│   └── payment_model.py
├── services/  # 추가
│   └── payment_service.py
├── database.py
└── auth.py
```

**사용 패턴:**
- ✅ Router 분리
- ✅ DTO 네이밍
- ✅ 의존성 주입
- ✅ CRUD 패턴
- ✅ 서비스 계층 (복잡한 로직만)

**예시:** 전자상거래, 교육 플랫폼

---

### 대규모 (10명+, 마이크로서비스)

**구조:**
```
project/
├── main.py
├── routers/
│   ├── users.py
│   ├── items.py
│   └── ...
├── models/
│   ├── user_model.py
│   └── ...
├── services/
│   ├── payment_service.py
│   ├── email_service.py
│   └── ...
├── utils/
│   ├── validators.py
│   └── helpers.py
├── middlewares/
│   └── auth_middleware.py
├── database.py
├── auth.py
└── config.py
```

**사용 패턴:**
- ✅ 모든 패턴 사용
- ✅ 레이어 명확히 분리
- ✅ 공통 유틸 분리
- ✅ 미들웨어 활용

**예시:** 대형 SaaS, 금융 플랫폼

---

## 언제 어떤 패턴을 쓸까?

### Router 분리

**항상 사용** ✅

작은 프로젝트도 기능별 분리하면 유지보수 쉬움

---

### DTO 네이밍 체계

**항상 사용** ✅

Create/Update/Response 구분은 필수

---

### 의존성 주입

**항상 사용** ✅

DB, 인증은 의존성으로

---

### CRUD 패턴

**항상 사용** ✅

표준 패턴 따르면 일관성 유지

---

### 서비스 계층

**조건부** ⚠️

**사용:**
- 외부 API 호출 (OpenAI, 결제 등)
- 복잡한 알고리즘
- 여러 단계 처리
- 재사용 필요

**불필요:**
- 단순 CRUD
- 조회만
- 한 곳에서만 사용

---

### 페이지네이션

**목록 조회는 필수** ✅

데이터 많아지면 필수

---

### Soft Delete

**권장** ✅

복구 가능성 때문에 권장

---

## 빠른 참조: 자주 쓰는 코드

### 기본 CRUD 라우터

```python
from fastapi import APIRouter, HTTPException, Depends
from sqlalchemy.orm import Session
from database import get_db
from auth import get_current_user
from models import ItemCreate, ItemUpdate, ItemResponse
from db_models import Item, User
import uuid
from datetime import datetime

router = APIRouter(prefix="/api/items", tags=["items"])

# Create
@router.post("", response_model=ItemResponse, status_code=201)
async def create_item(
    data: ItemCreate,
    db: Session = Depends(get_db),
    user: User = Depends(get_current_user)
):
    item = Item(id=str(uuid.uuid4()), user_id=user.id, **data.dict())
    db.add(item)
    db.commit()
    db.refresh(item)
    return item

# Read List
@router.get("", response_model=list[ItemResponse])
async def get_items(
    page: int = 1,
    page_size: int = 20,
    db: Session = Depends(get_db)
):
    items = db.query(Item).offset((page-1)*page_size).limit(page_size).all()
    return items

# Read Single
@router.get("/{id}", response_model=ItemResponse)
async def get_item(id: str, db: Session = Depends(get_db)):
    item = db.query(Item).filter(Item.id == id).first()
    if not item:
        raise HTTPException(404, "Item not found")
    return item

# Update
@router.put("/{id}", response_model=ItemResponse)
async def update_item(
    id: str,
    data: ItemUpdate,
    db: Session = Depends(get_db),
    user: User = Depends(get_current_user)
):
    item = db.query(Item).filter(Item.id == id, Item.user_id == user.id).first()
    if not item:
        raise HTTPException(404)

    update_data = data.dict(exclude_unset=True)
    for key, value in update_data.items():
        setattr(item, key, value)

    item.updated_at = datetime.now()
    db.commit()
    db.refresh(item)
    return item

# Delete
@router.delete("/{id}")
async def delete_item(
    id: str,
    db: Session = Depends(get_db),
    user: User = Depends(get_current_user)
):
    item = db.query(Item).filter(Item.id == id, Item.user_id == user.id).first()
    if not item:
        raise HTTPException(404)

    item.is_active = False
    item.deleted_at = datetime.now()
    db.commit()
    return {"message": "Item deleted"}
```

---

## 마무리

### 회사에서 배운 것

1. **구조가 중요하다**
   - Router 분리하면 협업 쉬움
   - 계층 구조 명확하면 유지보수 쉬움

2. **일관성이 핵심이다**
   - 네이밍 규칙 통일
   - 패턴 일관되게 적용

3. **정답은 없다**
   - 프로젝트 규모에 맞게
   - 팀과 상의해서 결정

4. **단순함을 유지하라**
   - 필요한 것만 적용
   - 과도한 추상화 피하기

### 실무 적용 팁

**처음 프로젝트 시작할 때:**
1. Router 분리부터 (항상)
2. DTO 네이밍 규칙 정하기
3. 의존성 주입 설정
4. CRUD 템플릿 만들기

**프로젝트 커질 때:**
1. 서비스 계층 추가 고려
2. 공통 유틸 분리
3. 미들웨어 검토

**팀 협업 시:**
1. 코딩 규칙 문서화
2. PR 리뷰로 일관성 유지
3. 템플릿 코드 공유

---

## 다음 단계

### 더 배울 것들

**성능 최적화:**
- 비동기 처리 (async/await)
- 캐싱 전략 (Redis)
- DB 쿼리 최적화

**테스트:**
- pytest 패턴
- 의존성 Mock
- DB 격리

**배포:**
- Docker 컨테이너화
- CI/CD 파이프라인
- 로깅 & 모니터링

**보안:**
- JWT 인증 강화
- CORS 설정
- Rate Limiting

---

## 참고 자료

**공식 문서:**
- FastAPI: https://fastapi.tiangolo.com
- Pydantic: https://docs.pydantic.dev
- SQLAlchemy: https://www.sqlalchemy.org

**추천 읽을거리:**
- Clean Architecture (Robert C. Martin)
- 실용주의 프로그래머

---

## 시리즈 완료!

**Part 1-6 정리:**

| Part | 주제 | 핵심 |
|------|------|------|
| 1 | Router 패턴 | main.py는 조립만 |
| 2 | DTO 네이밍 | 용도별 분리 |
| 3 | 의존성 주입 | yield로 자동 정리 |
| 4 | CRUD 패턴 | 표준화된 구조 |
| 5 | 서비스 계층 | 복잡한 로직 분리 |
| 6 | 총정리 | 실무 적용 가이드 |

**이제 당신도:**
- ✅ 깔끔한 프로젝트 구조를 만들 수 있다
- ✅ 일관된 코드를 작성할 수 있다
- ✅ 팀과 효율적으로 협업할 수 있다
- ✅ 프로젝트 규모에 맞게 선택할 수 있다

---

**회사에서 배웠고, 이제 내 것으로 만들었다. 🚀**

**시리즈 끝.**
