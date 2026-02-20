---
title: "Pydantic 모델 네이밍과 역할 분리"
date: 2025-02-19 10:00:00 +0900
categories: [Study, FastAPI]
tags: [fastapi, pydantic, dto, python]
---

## 들어가며

회사에서 API 코드를 읽다가 이런 클래스 이름들을 봤다.

```python
BoardCreateDto
BoardUpdateDto
BoardResponseDto
BoardListDto
```

**"왜 이렇게 이름을 지었을까?"**

`CreateDto`, `UpdateDto`, `ResponseDto`... 처음엔 길어 보였다. 근데 코드를 읽다 보니 **규칙**이 있었다.

---

## 내가 알던 방식 vs 회사 방식

### 내가 알던 방식 (학습용)

```python
# models.py (하나에 다 넣음)
from pydantic import BaseModel
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

# Pydantic 모델 (API용)
class Board(BaseModel):
    title: str
    description: str

# SQLAlchemy 모델 (DB용)
class BoardDB(Base):
    __tablename__ = "boards"
    id = Column(Integer, primary_key=True)
    title = Column(String(255))
```

**문제점을 생각해보니:**
- 생성, 수정, 응답을 구분 안 함
- 이름만으로 용도를 알기 어려움
- API와 DB 모델이 섞여있음

---

### 회사 방식 (명확한 분리)

**Project-A 방식:**

```python
# restful/model/board_model.py (API 계층만)
class BoardCreateDto(BaseModel):
    """보드 생성 요청"""
    title: str
    description: str

class BoardUpdateDto(BaseModel):
    """보드 수정 요청"""
    title: Optional[str] = None
    description: Optional[str] = None

class BoardResponseDto(BaseModel):
    """보드 응답"""
    board_id: str
    title: str
    description: str
    created_at: datetime
```

**Project-B 방식:**

```python
# models.py (API 계층만)
class MetadataFieldCreate(BaseModel):
    """메타데이터 필드 생성"""
    name: str
    description: str

class MetadataFieldUpdate(BaseModel):
    """메타데이터 필드 수정"""
    description: Optional[str] = None
    is_active: Optional[bool] = None

class MetadataFieldResponse(BaseModel):
    """메타데이터 필드 응답"""
    id: int
    name: str
    description: str

    class Config:
        from_attributes = True  # SQLAlchemy 객체 변환
```

**차이점:**
- Project-A: `Dto` 붙임 (`BoardCreateDto`)
- Project-B: `Dto` 안 붙임 (`MetadataFieldCreate`)

**공통점:**
- 용도별로 모델 분리 (Create, Update, Response)
- API 모델 = Pydantic
- DB 모델 = SQLAlchemy (별도 파일)

---

## 왜 Pydantic과 SQLAlchemy를 분리할까?

회사 코드를 보니 **계층이 명확히 분리**되어 있었다.

### 계층별 역할

```
┌─────────────────────────┐
│   API 계층 (Pydantic)   │  ← 클라이언트와 통신
├─────────────────────────┤
│   비즈니스 로직         │
├─────────────────────────┤
│   DB 계층 (SQLAlchemy)  │  ← 데이터베이스와 통신
└─────────────────────────┘
```

**Pydantic (API 계층):**
- 클라이언트 → 서버 (요청 검증)
- 서버 → 클라이언트 (응답 직렬화)
- 자동 타입 검증, Swagger 문서 생성

**SQLAlchemy (DB 계층):**
- 서버 → DB (SQL 쿼리)
- DB 테이블 구조 정의
- ORM 기능

**마치 통역사처럼**: API 계층은 "외부"와 대화하고, DB 계층은 "내부"와 대화한다.

---

## DTO 네이밍 규칙 파헤치기

회사 코드를 분석해보니 **일관된 패턴**이 있었다.

### 기본 패턴

| 용도 | 네이밍 | 언제 사용 |
|------|--------|----------|
| **생성 요청** | `XxxCreateDto` | POST 요청 Body |
| **수정 요청** | `XxxUpdateDto` | PUT/PATCH 요청 Body |
| **단일 응답** | `XxxResponseDto` | 단일 데이터 반환 |
| **목록 응답** | `XxxListDto` | 목록 + 페이지네이션 |
| **삭제 응답** | `XxxDeleteResponseDto` | 삭제 성공 메시지 |

### 실제 예시

```python
# 1. 생성 요청 (필수 필드만)
class BoardCreateDto(BaseModel):
    title: str              # 필수
    description: str        # 필수

# 2. 수정 요청 (모든 필드 Optional!)
class BoardUpdateDto(BaseModel):
    title: Optional[str] = None
    description: Optional[str] = None
    status: Optional[str] = None

# 3. 응답 (모든 필드 포함)
class BoardResponseDto(BaseModel):
    board_id: str
    title: str
    description: str
    status: str
    created_at: datetime
    updated_at: datetime

# 4. 목록 응답 (리스트 + 메타데이터)
class BoardListDto(BaseModel):
    boards: List[BoardResponseDto]
    pagination: PaginationDto
```

**장점:**
- 이름만 봐도 용도를 알 수 있음
- Swagger 문서가 명확해짐
- 같은 리소스라도 입력/출력이 다름을 표현

---

## Update DTO의 비밀

회사 코드에서 신기한 걸 발견했다.

### Update DTO는 모든 필드가 Optional!

```python
class BoardUpdateDto(BaseModel):
    title: Optional[str] = None          # Optional!
    description: Optional[str] = None    # Optional!
    status: Optional[str] = None         # Optional!
```

**왜 그럴까?** → **부분 업데이트** 지원

```python
# 제목만 수정
PUT /boards/{id}
{
  "title": "새 제목"
}

# 설명만 수정
PUT /boards/{id}
{
  "description": "새 설명"
}

# 둘 다 수정
PUT /boards/{id}
{
  "title": "새 제목",
  "description": "새 설명"
}
```

**코드에서 동적으로 처리:**

```python
@router.put("/boards/{board_id}")
async def update_board(
    board_id: str,
    data: BoardUpdateDto
):
    # 전달된 필드만 UPDATE
    update_fields = []
    update_params = {"board_id": board_id}

    if data.title is not None:
        update_fields.append("title = :title")
        update_params["title"] = data.title

    if data.description is not None:
        update_fields.append("description = :description")
        update_params["description"] = data.description

    if not update_fields:
        raise HTTPException(400, "No fields to update")

    # 동적 쿼리 생성
    query = f"""
        UPDATE boards
        SET {', '.join(update_fields)}
        WHERE board_id = :board_id
    """
    await update(query, update_params)
```

**마치 PATCH 메서드처럼**: 제공된 필드만 수정한다.

---

## Enum 패턴

회사 코드에서 Status를 관리하는 방식이 있었다.

### str, Enum 상속 필수!

```python
from enum import Enum

class BoardStatus(str, Enum):  # str 상속!
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
```

**왜 `str, Enum`?**
- `str` 상속: JSON 직렬화 자동 지원
- `Enum`: 코드에서 타입 안전성

```python
# 사용 예시
class BoardResponseDto(BaseModel):
    board_id: str
    status: BoardStatus  # Enum 타입

# JSON 응답
{
  "board_id": "123",
  "status": "pending"  # 문자열로 자동 변환
}
```

**장점:**
- Swagger에서 드롭다운으로 표시
- 오타 방지 (IDE 자동완성)
- 유효하지 않은 값 차단

---

## response_model과 from_attributes

회사 코드를 보니 **두 가지 중요한 패턴**이 있었다.

### 1. response_model 지정

```python
@router.post(
    "/boards",
    response_model=BoardResponseDto  # ← 명시!
)
async def create_board(data: BoardCreateDto):
    # 로직...
    return board_data
```

**효과:**
- 자동 응답 검증
- Swagger 문서에 응답 스키마 표시
- 불필요한 필드 자동 제거

### 2. from_attributes (SQLAlchemy 변환)

```python
class BoardResponseDto(BaseModel):
    board_id: str
    title: str
    created_at: datetime

    class Config:
        from_attributes = True  # ← 이거!
```

**언제 필요?** → SQLAlchemy 객체를 Pydantic으로 변환할 때

```python
# SQLAlchemy 쿼리 결과
board_orm = db.query(Board).first()

# Pydantic으로 자동 변환
board_dto = BoardResponseDto.from_orm(board_orm)
# 또는
return BoardResponseDto(**board_orm.__dict__)
```

**from_attributes가 없으면?** → 수동으로 필드 매핑해야 함

```python
# 번거로운 방식
return BoardResponseDto(
    board_id=board_orm.board_id,
    title=board_orm.title,
    created_at=board_orm.created_at
)
```

---

## 두 프로젝트 비교

### 네이밍 차이

| 항목 | Project A | Project B |
|------|-----------|-----------|
| 생성 | `BoardCreateDto` | `MetadataFieldCreate` |
| 수정 | `BoardUpdateDto` | `MetadataFieldUpdate` |
| 응답 | `BoardResponseDto` | `MetadataFieldResponse` |
| Dto 사용 | O | X |

**느낀 점:**
- `Dto` 붙이는 건 선택 사항
- 중요한 건 **일관성**
- 둘 다 용도별 분리는 동일

---

## 실전 적용: 단계별 가이드

### Step 1: 모델 파일 생성

```bash
# Project-A 스타일
mkdir -p restful/model
touch restful/model/example_model.py

# Project-B 스타일
touch models.py
```

### Step 2: Enum 정의 (필요시)

```python
# example_model.py
from enum import Enum

class ExampleStatus(str, Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"
```

### Step 3: Create DTO 작성

```python
from pydantic import BaseModel, Field

class ExampleCreateDto(BaseModel):
    """예제 생성 요청"""
    name: str = Field(..., description="이름")
    description: str = Field(..., description="설명")
    status: ExampleStatus = ExampleStatus.ACTIVE
```

### Step 4: Update DTO 작성

```python
from typing import Optional

class ExampleUpdateDto(BaseModel):
    """예제 수정 요청 - 모든 필드 Optional!"""
    name: Optional[str] = None
    description: Optional[str] = None
    status: Optional[ExampleStatus] = None
```

### Step 5: Response DTO 작성

```python
from datetime import datetime

class ExampleResponseDto(BaseModel):
    """예제 응답"""
    id: str
    name: str
    description: str
    status: ExampleStatus
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True  # SQLAlchemy 변환 지원
```

### Step 6: 라우터에서 사용

```python
# router.py
from fastapi import APIRouter
from .model.example_model import (
    ExampleCreateDto,
    ExampleUpdateDto,
    ExampleResponseDto
)

router = APIRouter(prefix="/api/examples", tags=["examples"])

@router.post("", response_model=ExampleResponseDto)
async def create_example(data: ExampleCreateDto):
    """생성"""
    # 로직...
    return example

@router.put("/{id}", response_model=ExampleResponseDto)
async def update_example(id: str, data: ExampleUpdateDto):
    """수정"""
    # 동적 업데이트 로직...
    return updated_example
```

---

## 실무 팁

### 1. 필드 검증

```python
from pydantic import Field, validator

class BoardCreateDto(BaseModel):
    title: str = Field(..., min_length=1, max_length=100)
    description: str = Field(..., max_length=1000)

    @validator('title')
    def validate_title(cls, v):
        if not v.strip():
            raise ValueError('Title cannot be empty')
        return v.strip()
```

### 2. 기본값 설정

```python
class ExampleCreateDto(BaseModel):
    name: str
    status: ExampleStatus = ExampleStatus.ACTIVE  # 기본값
    priority: int = Field(default=0, ge=0, le=10)
```

### 3. 중첩 모델

```python
class AddressDto(BaseModel):
    street: str
    city: str

class UserCreateDto(BaseModel):
    name: str
    address: AddressDto  # 중첩
```

---

## 배운 점 정리

1. **DTO 네이밍 = 용도 표현**
   - `XxxCreateDto`: POST 요청
   - `XxxUpdateDto`: PUT/PATCH 요청 (Optional!)
   - `XxxResponseDto`: 응답

2. **Pydantic vs SQLAlchemy 분리**
   - Pydantic: API 계층 (검증, 직렬화)
   - SQLAlchemy: DB 계층 (테이블 정의)

3. **Enum은 str 상속**
   - `class Status(str, Enum)`
   - JSON 직렬화 자동 지원

4. **response_model 명시**
   - 자동 검증
   - Swagger 문서 명확

5. **from_attributes = True**
   - SQLAlchemy 객체 → Pydantic 자동 변환

---

## 체크리스트

DTO를 만들 때:

- [ ] 용도별로 모델 분리 (Create, Update, Response)
- [ ] Update DTO는 모든 필드 `Optional`
- [ ] Status는 `str, Enum` 상속
- [ ] `response_model` 지정
- [ ] SQLAlchemy 사용 시 `from_attributes = True`
- [ ] 필드 검증 규칙 추가 (`Field`)
- [ ] 일관된 네이밍 규칙 (Dto 붙일지 말지 팀과 합의)

---

## 참고 템플릿

복사해서 바로 쓸 수 있는 기본 DTO 세트:

```python
# model/example_model.py
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime
from enum import Enum

# Enum
class ExampleStatus(str, Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"

# Create (생성 요청)
class ExampleCreateDto(BaseModel):
    name: str = Field(..., description="이름", min_length=1, max_length=100)
    description: str = Field(..., description="설명")
    status: ExampleStatus = ExampleStatus.ACTIVE

# Update (수정 요청 - 모든 필드 Optional!)
class ExampleUpdateDto(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    status: Optional[ExampleStatus] = None

# Response (응답)
class ExampleResponseDto(BaseModel):
    id: str
    name: str
    description: str
    status: ExampleStatus
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True

# List (목록 응답)
class PaginationDto(BaseModel):
    current_page: int
    page_size: int
    total_count: int
    total_pages: int
    has_next: bool
    has_prev: bool

class ExampleListDto(BaseModel):
    examples: List[ExampleResponseDto]
    pagination: PaginationDto

# Delete (삭제 응답)
class ExampleDeleteResponseDto(BaseModel):
    message: str
    id: str
```
