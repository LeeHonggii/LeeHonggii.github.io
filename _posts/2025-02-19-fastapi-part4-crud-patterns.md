---
title: "CRUD 패턴과 에러 처리"
date: 2025-02-19 12:00:00 +0900
categories: [Study, FastAPI]
tags: [fastapi, crud, error-handling, python]
---

## 들어가며

회사에서 여러 API 코드를 읽다 보니 신기한 점이 있었다.

**CRUD 코드가 거의 똑같은 구조였다.**

```python
# 이 프로젝트든
@router.post("/boards")
async def create_board(...):
    # 1. 검증
    # 2. 생성
    # 3. 커밋
    # 4. 반환

# 저 프로젝트든
@router.post("/fields")
async def create_field(...):
    # 1. 검증
    # 2. 생성
    # 3. 커밋
    # 4. 반환
```

처음엔 "복붙한 건가?" 생각했는데, 알고 보니 **표준 패턴**이었다.

---

## CRUD 엔드포인트 구조

회사 코드를 분석해보니 CRUD는 이렇게 구성되어 있었다.

```
POST   /api/resources           # Create
GET    /api/resources           # List (페이지네이션)
GET    /api/resources/{id}      # Read (단일)
PUT    /api/resources/{id}      # Update
DELETE /api/resources/{id}      # Delete (Soft)
```

**RESTful 원칙:**
- 복수형 리소스 (`/resources`)
- HTTP 메서드로 의도 표현
- Path parameter로 개별 리소스 지정

---

## Create 패턴: 생성의 정석

회사 코드를 보니 Create는 항상 이 순서였다.

### 기본 구조

```python
@router.post(
    "/api/fields",
    response_model=FieldResponseDto,
    status_code=status.HTTP_201_CREATED  # ← 201!
)
async def create_field(
    data: FieldCreateDto,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 1. 중복 체크 (필요시)
    exists = db.query(Field).filter(Field.name == data.name).first()
    if exists:
        raise HTTPException(
            status_code=400,
            detail=f"Field with name '{data.name}' already exists"
        )

    # 2. 객체 생성
    field = Field(
        id=str(uuid.uuid4()),           # UUID 생성
        user_id=current_user.id,
        name=data.name,
        description=data.description,
        created_at=datetime.now(),      # 타임스탬프
        updated_at=datetime.now()
    )

    # 3. DB 저장
    db.add(field)
    db.commit()
    db.refresh(field)  # ← DB에서 최신 데이터 다시 로드

    # 4. 반환
    return field
```

### 핵심 포인트

**1. UUID 생성**

```python
import uuid

id = str(uuid.uuid4())  # "123e4567-e89b-12d3-a456-426614174000"
```

- 자동 증가 ID보다 안전 (추측 불가)
- 분산 시스템에서 충돌 없음

---

**2. 타임스탬프**

```python
from datetime import datetime

created_at = datetime.now()
updated_at = datetime.now()
```

- 생성 시점 기록
- 수정 추적 가능

---

**3. commit & refresh**

```python
db.add(field)
db.commit()        # DB에 저장
db.refresh(field)  # DB에서 다시 읽기 (자동 생성된 필드 반영)
```

**왜 refresh?**
- DB가 자동으로 설정한 값 반영 (default, auto_increment 등)
- 반환할 때 최신 상태 보장

---

**4. 201 Created**

```python
status_code=status.HTTP_201_CREATED  # 201
```

- 200(OK)이 아닌 201(Created) 사용
- "리소스 생성 성공" 명시

---

## Read - 목록: 페이지네이션 필수

회사 코드를 보니 **목록 조회는 항상 페이지네이션**이 있었다.

### 기본 구조

```python
@router.get("/api/fields", response_model=FieldListDto)
async def get_fields(
    page: int = Query(1, ge=1, description="페이지 번호"),
    page_size: int = Query(20, ge=1, le=100, description="페이지 크기"),
    sort_by: str = Query("created_at", description="정렬 기준"),
    sort_order: str = Query("desc", description="정렬 순서"),
    active_only: bool = Query(True, description="활성화된 것만"),
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 1. 기본 쿼리
    query = db.query(Field).filter(Field.user_id == current_user.id)

    # 2. 필터링
    if active_only:
        query = query.filter(Field.is_active == True)

    # 3. 정렬 검증
    valid_sort_columns = ["created_at", "updated_at", "name"]
    if sort_by not in valid_sort_columns:
        sort_by = "created_at"

    if sort_order.lower() not in ["asc", "desc"]:
        sort_order = "desc"

    # 4. 정렬 적용
    if sort_order.lower() == "asc":
        query = query.order_by(getattr(Field, sort_by).asc())
    else:
        query = query.order_by(getattr(Field, sort_by).desc())

    # 5. 전체 개수 (페이지네이션 계산용)
    total_count = query.count()

    # 6. 페이징
    offset = (page - 1) * page_size
    fields = query.offset(offset).limit(page_size).all()

    # 7. 페이지네이션 메타데이터
    total_pages = (total_count + page_size - 1) // page_size

    return FieldListDto(
        fields=fields,
        pagination=PaginationDto(
            current_page=page,
            page_size=page_size,
            total_count=total_count,
            total_pages=total_pages,
            has_next=page < total_pages,
            has_prev=page > 1
        )
    )
```

### 핵심 포인트

**1. Query 파라미터 검증**

```python
page: int = Query(1, ge=1)           # 최소 1
page_size: int = Query(20, ge=1, le=100)  # 1~100
```

- `ge`: greater than or equal (이상)
- `le`: less than or equal (이하)
- 자동 검증, Swagger에 제약 표시

---

**2. 정렬 검증**

```python
valid_sort_columns = ["created_at", "updated_at", "name"]
if sort_by not in valid_sort_columns:
    sort_by = "created_at"  # 기본값
```

**왜?** → SQL Injection 방지

---

**3. count() 먼저**

```python
total_count = query.count()  # 전체 개수 먼저
fields = query.offset(...).limit(...).all()  # 그 다음 페이징
```

**마치 책 전체 페이지를 먼저 세는 것처럼**: 몇 페이지까지 있는지 알아야 UI에서 페이지 번호 표시 가능

---

**4. 페이지네이션 응답**

```python
class PaginationDto(BaseModel):
    current_page: int      # 현재 페이지
    page_size: int         # 페이지 크기
    total_count: int       # 전체 항목 수
    total_pages: int       # 전체 페이지 수
    has_next: bool         # 다음 페이지 있나?
    has_prev: bool         # 이전 페이지 있나?
```

프론트엔드에서 페이징 UI 구현에 필요한 모든 정보 제공

---

## Read - 단일: 404 처리

회사 코드를 보니 단일 조회는 간단했다.

```python
@router.get("/api/fields/{field_id}", response_model=FieldResponseDto)
async def get_field(
    field_id: str,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 조회
    field = db.query(Field).filter(
        Field.id == field_id,
        Field.user_id == current_user.id  # 소유권 확인
    ).first()

    # 404 처리
    if not field:
        raise HTTPException(
            status_code=404,
            detail=f"Field with ID {field_id} not found"
        )

    return field
```

**핵심:**
- 존재 + 소유권 동시 확인
- 404로 통일 (403 아님) → 존재 여부 숨김

---

## Update 패턴: 동적 업데이트

회사 코드에서 가장 인상적이었던 패턴.

### 방식 1: exclude_unset (Project-B)

```python
@router.put("/api/fields/{field_id}", response_model=FieldResponseDto)
async def update_field(
    field_id: str,
    data: FieldUpdateDto,  # 모든 필드 Optional!
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 1. 조회 + 소유권
    field = db.query(Field).filter(
        Field.id == field_id,
        Field.user_id == current_user.id
    ).first()

    if not field:
        raise HTTPException(status_code=404, detail="Field not found")

    # 2. 전달된 필드만 업데이트
    update_data = data.dict(exclude_unset=True)  # ← 핵심!

    for key, value in update_data.items():
        setattr(field, key, value)

    # 3. updated_at 갱신
    field.updated_at = datetime.now()

    # 4. 저장
    db.commit()
    db.refresh(field)

    return field
```

**exclude_unset의 마법:**

```python
# 요청
PUT /api/fields/123
{
  "name": "새 이름"
  # description은 안 보냄
}

# exclude_unset=False (기본)
{"name": "새 이름", "description": None}  # None으로 변환

# exclude_unset=True
{"name": "새 이름"}  # 전달된 것만
```

**마치 부분 수정처럼**: 바꾸고 싶은 것만 보내면 된다.

---

### 방식 2: 수동 체크 (Project-A)

```python
@router.put("/api/boards/{board_id}")
async def update_board(
    board_id: str,
    data: BoardUpdateDto,
    db: Session = Depends(get_db),
    current_user: str = Depends(get_current_user)
):
    # 조회
    board = db.query(Board).filter(
        Board.id == board_id,
        Board.user_id == current_user
    ).first()

    if not board:
        raise HTTPException(status_code=404)

    # 수동으로 체크
    if data.title is not None:
        board.title = data.title

    if data.description is not None:
        board.description = data.description

    if data.status is not None:
        board.status = data.status

    board.updated_at = datetime.now()

    db.commit()
    db.refresh(board)

    return board
```

**차이점:**
- 방식 1: 자동 (반복문)
- 방식 2: 수동 (명시적)

**선택 기준:**
- 필드 적음 → 수동 (가독성)
- 필드 많음 → 자동 (간결함)

---

## Delete 패턴: Soft vs Hard

회사 코드를 보니 **대부분 Soft Delete**였다.

### Soft Delete (권장)

```python
@router.delete("/api/fields/{field_id}")
async def delete_field(
    field_id: str,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    field = db.query(Field).filter(
        Field.id == field_id,
        Field.user_id == current_user.id
    ).first()

    if not field:
        raise HTTPException(status_code=404)

    # Soft Delete
    field.is_active = False
    field.deleted_at = datetime.now()

    db.commit()

    return {"message": f"Field '{field.name}' has been deactivated"}
```

**Soft Delete 패턴:**
- `is_active = False` 또는 `deleted_at = NOW()`
- 데이터는 남아있음
- 복구 가능

---

### Hard Delete (신중히)

```python
@router.delete("/api/fields/{field_id}")
async def delete_field_hard(
    field_id: str,
    db: Session = Depends(get_db),
    admin: User = Depends(get_admin_user)  # 관리자 전용
):
    field = db.query(Field).filter(Field.id == field_id).first()

    if not field:
        raise HTTPException(status_code=404)

    # Hard Delete
    db.delete(field)
    db.commit()

    return {"message": "Field permanently deleted"}
```

**Hard Delete는:**
- 진짜 삭제
- 복구 불가
- 관리자만 허용 권장

---

### 비교

| 항목 | Soft Delete | Hard Delete |
|------|-------------|-------------|
| **데이터** | 남음 | 삭제됨 |
| **복구** | 가능 | 불가능 |
| **관계** | 참조 유지 | Foreign Key 문제 |
| **용도** | 일반 사용자 | 관리자, 테스트 |

**마치 휴지통처럼**: Soft Delete는 휴지통, Hard Delete는 영구 삭제

---

## 에러 처리 표준

회사 코드를 보니 에러 처리가 일관적이었다.

### 기본 패턴

```python
try:
    # 비즈니스 로직
    field = db.query(Field).filter(...).first()
    if not field:
        raise HTTPException(status_code=404, detail="Not found")

    # 작업 수행
    db.commit()

except HTTPException:
    raise  # HTTPException은 그대로 전파

except Exception as e:
    import traceback
    print(traceback.format_exc())  # 로그 (운영에선 logger 사용)
    raise HTTPException(
        status_code=500,
        detail=f"Internal server error: {str(e)}"
    )
```

### 왜 이렇게?

**1. HTTPException은 그대로 전파**

```python
except HTTPException:
    raise  # 다시 던짐
```

- FastAPI가 알아서 처리
- 상태 코드와 메시지 보존

---

**2. Exception은 500으로 변환**

```python
except Exception as e:
    print(traceback.format_exc())  # 디버깅용
    raise HTTPException(status_code=500, detail=str(e))
```

- 예상 못한 에러 잡기
- 서버 에러로 통일

---

## HTTP 상태 코드 가이드

회사 코드에서 본 표준.

| 코드 | 상황 | 예시 |
|------|------|------|
| **200** | 성공 | GET, PUT 성공 |
| **201** | 생성 성공 | POST로 리소스 생성 |
| **204** | 성공 (응답 없음) | DELETE 성공 (바디 없음) |
| **400** | 잘못된 요청 | 필수 필드 누락, 검증 실패 |
| **401** | 인증 실패 | 토큰 없음, 만료, 잘못됨 |
| **403** | 권한 없음 | 관리자 전용 API |
| **404** | 리소스 없음 | 존재하지 않는 ID |
| **409** | 충돌 | 중복된 이름, 동시 수정 |
| **429** | 요청 제한 | 일일 생성 제한 초과 |
| **500** | 서버 에러 | 예상 못한 오류 |

### 실제 사용 예시

```python
# 201 - 생성
@router.post("", status_code=status.HTTP_201_CREATED)

# 400 - 잘못된 요청
raise HTTPException(status_code=400, detail="Name is required")

# 401 - 인증 실패
raise HTTPException(status_code=401, detail="Invalid token")

# 403 - 권한 없음
raise HTTPException(status_code=403, detail="Admin only")

# 404 - 없음
raise HTTPException(status_code=404, detail="Field not found")

# 409 - 충돌
raise HTTPException(status_code=409, detail="Name already exists")

# 500 - 서버 에러
raise HTTPException(status_code=500, detail="Internal error")
```

---

## 실전 적용: 완전한 CRUD

복사해서 바로 쓸 수 있는 전체 코드:

```python
# router_example.py
from fastapi import APIRouter, HTTPException, Query, Depends, status
from sqlalchemy.orm import Session
from typing import List, Optional
from datetime import datetime
import uuid

from database import get_db
from auth import get_current_user
from models import (
    ExampleCreateDto,
    ExampleUpdateDto,
    ExampleResponseDto,
    ExampleListDto,
    PaginationDto
)
from db_models import Example, User

router = APIRouter(prefix="/api/examples", tags=["examples"])


# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# CREATE
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
@router.post("", response_model=ExampleResponseDto, status_code=status.HTTP_201_CREATED)
async def create_example(
    data: ExampleCreateDto,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """예제 생성"""
    try:
        # 1. 중복 체크 (필요시)
        exists = db.query(Example).filter(
            Example.name == data.name,
            Example.user_id == current_user.id
        ).first()

        if exists:
            raise HTTPException(
                status_code=400,
                detail=f"Example with name '{data.name}' already exists"
            )

        # 2. 객체 생성
        example = Example(
            id=str(uuid.uuid4()),
            user_id=current_user.id,
            name=data.name,
            description=data.description,
            status=data.status,
            created_at=datetime.now(),
            updated_at=datetime.now()
        )

        # 3. 저장
        db.add(example)
        db.commit()
        db.refresh(example)

        return example

    except HTTPException:
        raise
    except Exception as e:
        import traceback
        print(traceback.format_exc())
        raise HTTPException(status_code=500, detail=f"Creation error: {str(e)}")


# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# READ - 목록 (페이지네이션)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
@router.get("", response_model=ExampleListDto)
async def get_examples(
    page: int = Query(1, ge=1, description="페이지 번호"),
    page_size: int = Query(20, ge=1, le=100, description="페이지 크기"),
    sort_by: str = Query("created_at", description="정렬 기준"),
    sort_order: str = Query("desc", description="정렬 순서 (asc, desc)"),
    active_only: bool = Query(True, description="활성화된 것만"),
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """예제 목록 조회"""
    try:
        # 1. 기본 쿼리
        query = db.query(Example).filter(Example.user_id == current_user.id)

        # 2. 필터링
        if active_only:
            query = query.filter(Example.is_active == True)

        # 3. 정렬 검증
        valid_sort_columns = ["created_at", "updated_at", "name"]
        if sort_by not in valid_sort_columns:
            sort_by = "created_at"

        if sort_order.lower() not in ["asc", "desc"]:
            sort_order = "desc"

        # 4. 정렬 적용
        if sort_order.lower() == "asc":
            query = query.order_by(getattr(Example, sort_by).asc())
        else:
            query = query.order_by(getattr(Example, sort_by).desc())

        # 5. 전체 개수
        total_count = query.count()

        # 6. 페이징
        offset = (page - 1) * page_size
        examples = query.offset(offset).limit(page_size).all()

        # 7. 페이지네이션 메타데이터
        total_pages = (total_count + page_size - 1) // page_size

        return ExampleListDto(
            examples=examples,
            pagination=PaginationDto(
                current_page=page,
                page_size=page_size,
                total_count=total_count,
                total_pages=total_pages,
                has_next=page < total_pages,
                has_prev=page > 1
            )
        )

    except Exception as e:
        raise HTTPException(status_code=500, detail=f"List error: {str(e)}")


# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# READ - 단일
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
@router.get("/{example_id}", response_model=ExampleResponseDto)
async def get_example(
    example_id: str,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """예제 단일 조회"""
    try:
        example = db.query(Example).filter(
            Example.id == example_id,
            Example.user_id == current_user.id
        ).first()

        if not example:
            raise HTTPException(
                status_code=404,
                detail=f"Example with ID {example_id} not found"
            )

        return example

    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Retrieval error: {str(e)}")


# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# UPDATE
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
@router.put("/{example_id}", response_model=ExampleResponseDto)
async def update_example(
    example_id: str,
    data: ExampleUpdateDto,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """예제 수정"""
    try:
        # 1. 조회 + 소유권
        example = db.query(Example).filter(
            Example.id == example_id,
            Example.user_id == current_user.id
        ).first()

        if not example:
            raise HTTPException(status_code=404, detail="Example not found")

        # 2. 동적 업데이트
        update_data = data.dict(exclude_unset=True)

        if not update_data:
            raise HTTPException(status_code=400, detail="No fields to update")

        for key, value in update_data.items():
            setattr(example, key, value)

        # 3. updated_at 갱신
        example.updated_at = datetime.now()

        # 4. 저장
        db.commit()
        db.refresh(example)

        return example

    except HTTPException:
        raise
    except Exception as e:
        import traceback
        print(traceback.format_exc())
        raise HTTPException(status_code=500, detail=f"Update error: {str(e)}")


# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# DELETE (Soft)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
@router.delete("/{example_id}")
async def delete_example(
    example_id: str,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """예제 삭제 (Soft Delete)"""
    try:
        example = db.query(Example).filter(
            Example.id == example_id,
            Example.user_id == current_user.id
        ).first()

        if not example:
            raise HTTPException(status_code=404, detail="Example not found")

        # Soft Delete
        example.is_active = False
        example.deleted_at = datetime.now()

        db.commit()

        return {"message": f"Example '{example.name}' has been deactivated"}

    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Deletion error: {str(e)}")
```

---

## 배운 점 정리

1. **Create 패턴**
   - UUID 생성
   - 중복 체크
   - commit → refresh
   - 201 Created

2. **Read - 목록**
   - 페이지네이션 필수
   - 정렬 검증 (SQL Injection 방지)
   - count() 먼저

3. **Read - 단일**
   - 존재 + 소유권 동시 확인
   - 404로 통일

4. **Update**
   - `dict(exclude_unset=True)` 또는 수동 체크
   - updated_at 갱신
   - 부분 업데이트 지원

5. **Delete**
   - Soft Delete 권장
   - Hard Delete는 신중히

6. **에러 처리**
   - HTTPException은 그대로 전파
   - Exception은 500으로 변환
   - traceback 로깅

---

## 체크리스트

CRUD API를 만들 때:

**Create**
- [ ] UUID 생성
- [ ] 중복 체크 (필요시)
- [ ] 타임스탬프 (created_at, updated_at)
- [ ] commit → refresh
- [ ] status_code=201

**Read - 목록**
- [ ] 페이지네이션 파라미터 (page, page_size)
- [ ] 정렬 검증
- [ ] count() 먼저
- [ ] PaginationDto 반환

**Read - 단일**
- [ ] 존재 + 소유권 확인
- [ ] 404 처리

**Update**
- [ ] 조회 + 소유권
- [ ] dict(exclude_unset=True)
- [ ] updated_at 갱신
- [ ] commit → refresh

**Delete**
- [ ] Soft Delete (is_active, deleted_at)
- [ ] 소유권 확인

**에러 처리**
- [ ] try-except 블록
- [ ] HTTPException 그대로 전파
- [ ] Exception → 500
- [ ] traceback 로깅

---

## 다음 편 예고

**Part 5: 서비스 계층과 비즈니스 로직 분리**

회사 코드를 보니 복잡한 로직은 **서비스 클래스**로 분리되어 있었다.

```python
# 라우터는 얇게
@router.post("/extract")
async def extract_metadata():
    extractor = MetadataExtractor(...)  # 서비스 클래스
    result = extractor.extract(text)
    return result
```

왜 이렇게 분리할까? 언제 서비스 계층이 필요할까?

다음 편에서 알아보자.

---

**회사에서 배우는 중... 🚀**
