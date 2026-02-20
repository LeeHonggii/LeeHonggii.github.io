---
title: "의존성 주입 패턴 완전 정복"
date: 2025-02-19 11:00:00 +0900
categories: [Study, FastAPI]
tags: [fastapi, dependency-injection, python, depends]
---

## 들어가며

회사에서 API 엔드포인트를 읽다가 이상한 패턴을 봤다.

```python
@router.get("/files")
async def get_files(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    ...
```

**`Depends(get_db)`? `Depends(get_current_user)`?**

함수 안에서 DB를 열거나 인증을 체크하지 않는데, 어떻게 동작하는 걸까?

코드를 추적해보니 **의존성 주입(Dependency Injection)** 패턴이었다.

---

## 내가 알던 방식 vs 회사 방식

### 내가 알던 방식 (수동 관리)

```python
from database import SessionLocal

@router.get("/boards")
async def get_boards():
    # DB 세션 수동 생성
    db = SessionLocal()
    try:
        boards = db.query(Board).all()
        return boards
    finally:
        db.close()  # 닫는 걸 까먹으면? 💥
```

**문제점:**
- 매번 `try-finally` 작성
- `db.close()` 까먹으면 리소스 누수
- 엔드포인트마다 반복되는 코드
- 인증 체크도 매번 작성해야 함

---

### 회사 방식 (자동 관리)

```python
@router.get("/boards")
async def get_boards(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # DB는 이미 열려있고
    # 사용자는 이미 인증됨
    boards = db.query(Board).all()
    return boards
    # 함수 끝나면 자동으로 정리!
```

**장점:**
- ✅ 반복 코드 제거
- ✅ 자동 리소스 정리
- ✅ 에러 발생해도 안전하게 닫힘
- ✅ 코드가 짧고 명확

**마치 전기처럼**: 플러그만 꽂으면 전기가 흐르고, 빼면 자동으로 꺼진다.

---

## get_db() 패턴 파헤치기

회사 코드를 보니 `database.py`에 이런 함수가 있었다.

```python
# database.py
from sqlalchemy.orm import sessionmaker

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db  # ← 이게 핵심!
    finally:
        db.close()  # 항상 실행됨
```

### yield의 비밀

**일반 함수 (return):**

```python
def get_db():
    db = SessionLocal()
    return db  # 여기서 끝
    # finally가 실행 안 됨!
```

**제너레이터 함수 (yield):**

```python
def get_db():
    db = SessionLocal()
    try:
        yield db        # ← 여기서 잠깐 멈춤
        # 엔드포인트가 실행됨
        # 엔드포인트 끝나면 다시 돌아옴
    finally:
        db.close()      # ← 항상 실행!
```

**흐름:**

```
1. FastAPI: get_db() 호출
2. get_db: SessionLocal() 생성
3. get_db: yield db ← 멈춤
4. 엔드포인트: DB 사용
5. 엔드포인트: return (정상 or 에러)
6. get_db: finally 블록 실행
7. get_db: db.close() ← 무조건 실행!
```

**마치 빌려주는 것처럼**: 책을 빌려주고(yield), 다 읽으면(엔드포인트 끝) 자동으로 회수(close)한다.

---

## get_current_user() 패턴: 의존성 체인

회사 코드를 보니 인증 함수가 특이했다.

```python
# auth.py
async def get_current_user(
    authorization: Optional[str] = Header(None),
    db: Session = Depends(get_db)  # ← DB에 의존!
) -> User:
    """
    Authorization 헤더에서 토큰 추출 → DB에서 사용자 조회
    """
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="인증이 필요합니다.")

    token = authorization.replace("Bearer ", "")

    # 토큰 검증 로직...
    user_id = parse_token(token)

    # DB에서 사용자 조회
    db_user = db.query(User).filter(User.id == user_id).first()

    if not db_user:
        raise HTTPException(status_code=401, detail="사용자를 찾을 수 없습니다.")

    return db_user  # User 객체 반환
```

### 의존성 체인

```
get_current_user
    └── Depends(get_db)
```

**흐름:**

```
1. 엔드포인트가 get_current_user 요청
2. FastAPI: "get_current_user는 get_db가 필요하네?"
3. FastAPI: get_db() 먼저 호출 → DB 세션 제공
4. FastAPI: get_current_user(authorization, db) 호출
5. get_current_user: 토큰 검증 + DB 조회
6. 엔드포인트: User 객체 받음
```

**마치 레시피처럼**: 케이크를 만들려면(get_current_user) 밀가루가 필요하고(get_db), FastAPI가 알아서 순서대로 준비한다.

---

## 실전 사용 예시

### 1. DB만 필요한 경우

```python
@router.get("/public/boards")
async def get_public_boards(
    db: Session = Depends(get_db)  # 인증 불필요
):
    boards = db.query(Board).filter(Board.is_public == True).all()
    return boards
```

### 2. DB + 인증 필요한 경우

```python
@router.get("/my/boards")
async def get_my_boards(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # current_user는 이미 검증된 User 객체
    boards = db.query(Board).filter(Board.user_id == current_user.id).all()
    return boards
```

### 3. 인증만 필요한 경우

```python
@router.get("/me")
async def get_me(
    current_user: User = Depends(get_current_user)  # DB는 내부에서 처리
):
    return {
        "id": current_user.id,
        "username": current_user.username
    }
```

---

## 두 프로젝트 비교

회사에는 두 프로젝트가 있었고, 인증 방식이 달랐다.

### Project-A: JWT 방식

```python
async def get_current_user(
    authorization: str = Header(None),
    db: Session = Depends(get_db)
) -> str:  # user_id 문자열 반환
    """JWT 토큰 검증 후 user_id 반환"""
    token = parse_jwt(authorization)
    user_id = token.get("user_id")

    # DB 조회로 유효성 확인
    user_exists = db.query(User).filter(User.id == user_id).first()
    if not user_exists:
        raise HTTPException(status_code=401)

    return user_id  # 문자열
```

**사용:**

```python
@router.post("/boards")
async def create_board(
    current_user: str = Depends(get_current_user)  # user_id 받음
):
    board = Board(user_id=current_user, ...)
    ...
```

---

### Project-B: Bearer Token 방식

```python
async def get_current_user(
    authorization: Optional[str] = Header(None),
    db: Session = Depends(get_db)
) -> User:  # User 객체 반환
    """Bearer 토큰 검증 후 User 객체 반환"""
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401)

    token = authorization.replace("Bearer ", "")
    user_id = parse_token(token)

    # User 객체 조회
    db_user = db.query(User).filter(User.id == user_id).first()
    if not db_user:
        raise HTTPException(status_code=401)

    return db_user  # 객체
```

**사용:**

```python
@router.post("/files")
async def upload_file(
    current_user: User = Depends(get_current_user)  # User 객체 받음
):
    file = FileMetadata(
        user_id=current_user.id,
        uploader_name=current_user.username,  # 추가 정보 사용 가능
        ...
    )
    ...
```

---

### 비교 표

| 항목 | Project-A (JWT) | Project-B (Bearer Token) |
|------|-----------------|--------------------------|
| **반환 타입** | `str` (user_id) | `User` (객체) |
| **장점** | 간결, DB 조회 최소화 | 사용자 정보 즉시 사용 |
| **단점** | 추가 정보 필요 시 재조회 | 매번 객체 로드 |
| **사용 시나리오** | user_id만 필요 | username, email 등 필요 |

**느낀 점:**
- 정답은 없다
- user_id만 필요하면 A 방식
- 사용자 정보 자주 쓰면 B 방식

---

## 소유권 확인 패턴

회사 코드에서 자주 본 패턴.

### 기본 패턴

```python
@router.put("/boards/{board_id}")
async def update_board(
    board_id: str,
    data: BoardUpdateDto,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 1. 존재 + 소유권 동시 확인
    board = db.query(Board).filter(
        Board.id == board_id,
        Board.user_id == current_user.id  # ← 본인 것만
    ).first()

    if not board:
        raise HTTPException(status_code=404, detail="Board not found")

    # 2. 수정
    board.title = data.title
    db.commit()

    return board
```

**왜 이렇게?**
- 남의 데이터 수정 방지
- 404로 통일 (403 대신) → 존재 여부 숨김

---

## 언제 의존성을 사용하는가?

회사 코드를 보며 정리한 규칙.

### 항상 사용

**1. DB 세션**

```python
db: Session = Depends(get_db)
```

모든 DB 작업에 필수.

---

**2. 인증 (대부분)**

```python
current_user: User = Depends(get_current_user)
```

생성/수정/삭제는 거의 필수. 조회는 경우에 따라.

---

### 선택적 사용

**1. 설정 주입**

```python
# config.py
class Settings:
    OPENAI_API_KEY: str
    OPENAI_MODEL: str

settings = Settings()

# 의존성으로 사용
def get_settings():
    return settings

# 라우터
@router.post("/extract")
async def extract_metadata(
    settings: Settings = Depends(get_settings)
):
    extractor = MetadataExtractor(api_key=settings.OPENAI_API_KEY)
    ...
```

**장점:** 테스트 시 Mock 설정 주입 가능

---

**2. 페이지네이션 파라미터**

```python
class CommonQueryParams:
    def __init__(
        self,
        page: int = Query(1, ge=1),
        page_size: int = Query(20, ge=1, le=100)
    ):
        self.page = page
        self.page_size = page_size
        self.offset = (page - 1) * page_size

# 사용
@router.get("/boards")
async def get_boards(
    params: CommonQueryParams = Depends()
):
    boards = db.query(Board).offset(params.offset).limit(params.page_size).all()
    ...
```

**장점:** 페이지네이션 로직 재사용

---

## 실전 적용: 단계별 가이드

### Step 1: DB 의존성 설정

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

---

### Step 2: 인증 의존성 작성

```python
# auth.py
from fastapi import Depends, Header, HTTPException

async def get_current_user(
    authorization: Optional[str] = Header(None),
    db: Session = Depends(get_db)
) -> User:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="인증이 필요합니다.")

    token = authorization.replace("Bearer ", "")

    # 토큰 파싱 (실제로는 JWT 라이브러리 사용)
    user_id = parse_token(token)

    # DB에서 사용자 조회
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=401, detail="사용자를 찾을 수 없습니다.")

    return user
```

---

### Step 3: 라우터에서 사용

```python
# router.py
from fastapi import APIRouter, Depends
from database import get_db
from auth import get_current_user

router = APIRouter(prefix="/api/boards")

@router.post("", response_model=BoardResponseDto)
async def create_board(
    data: BoardCreateDto,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    board = Board(
        id=str(uuid.uuid4()),
        user_id=current_user.id,
        title=data.title,
        created_at=datetime.now()
    )

    db.add(board)
    db.commit()
    db.refresh(board)

    return board
```

---

## 실무 팁

### 1. 선택적 인증

```python
from typing import Optional

async def get_current_user_optional(
    authorization: Optional[str] = Header(None),
    db: Session = Depends(get_db)
) -> Optional[User]:
    """인증 실패해도 None 반환 (에러 안 냄)"""
    if not authorization:
        return None

    try:
        token = authorization.replace("Bearer ", "")
        user_id = parse_token(token)
        return db.query(User).filter(User.id == user_id).first()
    except:
        return None

# 사용: 로그인하면 더 많은 정보, 안 해도 기본 정보
@router.get("/boards")
async def get_boards(
    current_user: Optional[User] = Depends(get_current_user_optional)
):
    if current_user:
        # 로그인: 내 보드 + 공개 보드
        boards = db.query(Board).filter(
            (Board.user_id == current_user.id) | (Board.is_public == True)
        ).all()
    else:
        # 비로그인: 공개 보드만
        boards = db.query(Board).filter(Board.is_public == True).all()

    return boards
```

---

### 2. 의존성 재사용

```python
# common_deps.py
class PaginationParams:
    def __init__(
        self,
        page: int = Query(1, ge=1, description="페이지 번호"),
        page_size: int = Query(20, ge=1, le=100, description="페이지 크기")
    ):
        self.page = page
        self.page_size = page_size
        self.offset = (page - 1) * page_size
        self.limit = page_size

# 여러 라우터에서 재사용
@router.get("/boards")
async def get_boards(pagination: PaginationParams = Depends()):
    boards = db.query(Board).offset(pagination.offset).limit(pagination.limit).all()
    ...

@router.get("/comments")
async def get_comments(pagination: PaginationParams = Depends()):
    comments = db.query(Comment).offset(pagination.offset).limit(pagination.limit).all()
    ...
```

---

### 3. 의존성 체인

```python
# 관리자 전용 의존성
async def get_admin_user(
    current_user: User = Depends(get_current_user)  # 체인!
) -> User:
    if not current_user.is_admin:
        raise HTTPException(status_code=403, detail="관리자 권한이 필요합니다.")
    return current_user

# 사용
@router.delete("/users/{user_id}")
async def delete_user(
    user_id: str,
    admin: User = Depends(get_admin_user)  # 인증 + 관리자 체크
):
    ...
```

---

## 배운 점 정리

1. **의존성 주입 = 자동화**
   - DB 세션 자동 열기/닫기
   - 인증 자동 체크
   - 반복 코드 제거

2. **yield의 마법**
   - 리소스 제공 + 자동 정리
   - 에러 발생해도 finally 실행

3. **의존성 체인**
   - `get_current_user` → `get_db`
   - FastAPI가 순서대로 실행

4. **타입에 따라 선택**
   - user_id만 필요: `str` 반환
   - 사용자 정보 필요: `User` 객체 반환

5. **소유권 확인 패턴**
   - 존재 + 소유권 동시 확인
   - 404로 통일

---

## 체크리스트

의존성 주입을 사용할 때:

- [ ] `get_db()`는 `yield` 사용
- [ ] `get_current_user()`는 `Depends(get_db)` 체인
- [ ] 모든 DB 작업에 `db: Session = Depends(get_db)`
- [ ] 인증 필요한 엔드포인트에 `current_user = Depends(...)`
- [ ] 소유권 확인: `Board.user_id == current_user.id`
- [ ] 선택적 인증은 `Optional[User]` 사용
- [ ] 공통 파라미터는 클래스로 재사용

---

## 다음 편 예고

**Part 4: CRUD 패턴과 에러 처리**

회사 코드를 보니 CRUD 작성 방식이 일관적이었다.

- 생성: UUID 생성 → INSERT → SELECT
- 목록: 페이지네이션 필수
- 수정: 동적 UPDATE (전달된 필드만)
- 삭제: Soft Delete (deleted_at)

다음 편에서 알아보자.

---

## 참고 템플릿

복사해서 바로 쓸 수 있는 의존성 패턴:

```python
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# database.py
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def get_db():
    """DB 세션 제공 + 자동 정리"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# auth.py
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
from fastapi import Depends, Header, HTTPException
from sqlalchemy.orm import Session
from typing import Optional
from database import get_db

async def get_current_user(
    authorization: Optional[str] = Header(None),
    db: Session = Depends(get_db)
) -> User:
    """Bearer 토큰 검증 → User 객체 반환"""
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(
            status_code=401,
            detail="인증이 필요합니다."
        )

    token = authorization.replace("Bearer ", "")

    # 토큰 파싱 (실제로는 JWT 라이브러리 사용)
    user_id = parse_token(token)

    # DB에서 사용자 조회
    user = db.query(User).filter(User.id == user_id).first()

    if not user:
        raise HTTPException(
            status_code=401,
            detail="사용자를 찾을 수 없습니다."
        )

    return user


async def get_current_user_optional(
    authorization: Optional[str] = Header(None),
    db: Session = Depends(get_db)
) -> Optional[User]:
    """선택적 인증 (실패해도 None 반환)"""
    if not authorization:
        return None

    try:
        return await get_current_user(authorization, db)
    except HTTPException:
        return None


async def get_admin_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """관리자 전용 의존성"""
    if not current_user.is_admin:
        raise HTTPException(
            status_code=403,
            detail="관리자 권한이 필요합니다."
        )
    return current_user

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# common_deps.py (공통 의존성)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
from fastapi import Query

class PaginationParams:
    """페이지네이션 공통 파라미터"""
    def __init__(
        self,
        page: int = Query(1, ge=1, description="페이지 번호"),
        page_size: int = Query(20, ge=1, le=100, description="페이지 크기")
    ):
        self.page = page
        self.page_size = page_size
        self.offset = (page - 1) * page_size
        self.limit = page_size

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# router.py (사용 예시)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
from fastapi import APIRouter, Depends
from database import get_db
from auth import get_current_user, get_admin_user
from common_deps import PaginationParams

router = APIRouter(prefix="/api/boards")

# 공개 조회 (인증 불필요)
@router.get("/public")
async def get_public_boards(
    db: Session = Depends(get_db),
    pagination: PaginationParams = Depends()
):
    boards = db.query(Board)\
        .filter(Board.is_public == True)\
        .offset(pagination.offset)\
        .limit(pagination.limit)\
        .all()
    return boards

# 내 보드 조회 (인증 필요)
@router.get("/my")
async def get_my_boards(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
    pagination: PaginationParams = Depends()
):
    boards = db.query(Board)\
        .filter(Board.user_id == current_user.id)\
        .offset(pagination.offset)\
        .limit(pagination.limit)\
        .all()
    return boards

# 생성 (인증 필요)
@router.post("")
async def create_board(
    data: BoardCreateDto,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    board = Board(
        id=str(uuid.uuid4()),
        user_id=current_user.id,
        title=data.title
    )
    db.add(board)
    db.commit()
    return board

# 수정 (인증 + 소유권)
@router.put("/{board_id}")
async def update_board(
    board_id: str,
    data: BoardUpdateDto,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 존재 + 소유권 동시 확인
    board = db.query(Board).filter(
        Board.id == board_id,
        Board.user_id == current_user.id
    ).first()

    if not board:
        raise HTTPException(status_code=404, detail="Board not found")

    board.title = data.title
    db.commit()
    return board

# 삭제 (관리자 전용)
@router.delete("/{board_id}")
async def delete_board(
    board_id: str,
    db: Session = Depends(get_db),
    admin: User = Depends(get_admin_user)
):
    board = db.query(Board).filter(Board.id == board_id).first()
    if not board:
        raise HTTPException(status_code=404)

    db.delete(board)
    db.commit()
    return {"message": "Board deleted"}
```

---

**회사에서 배우는 중... 🚀**
