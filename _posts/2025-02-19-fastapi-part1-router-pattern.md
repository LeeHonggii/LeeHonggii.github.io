---
title: "FastAPI Router 패턴과 프로젝트 구조"
date: 2025-02-19 09:00:00 +0900
categories: [Study, FastAPI]
tags: [fastapi, router, project-structure, python]
---

## 들어가며

회사에 입사해서 FastAPI 프로젝트를 처음 봤을 때, 신기한 점이 있었다.

**main.py가 놀랍도록 짧았다.**

```python
# main.py (20줄 정도)
from fastapi import FastAPI
from routers import auth, files, settings

app = FastAPI()

app.include_router(auth.router)
app.include_router(files.router)
app.include_router(settings.router)
```

나는 이제까지 FastAPI를 배울 때 main.py에 엔드포인트를 다 작성했었다. 근데 회사 코드는 달랐다.

**"왜 이렇게 구조를 만들었을까?"**

분석해보니 `Router 분리 패턴`이라는 걸 알게 됐다.

---

## 내가 알던 방식 vs 회사 방식

### 내가 알던 방식 (학습용)

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.post("/boards")
async def create_board():
    pass

@app.get("/boards")
async def list_boards():
    pass

@app.post("/boards/{id}/contents")
async def create_content():
    pass

# ... 엔드포인트 계속 추가
```

**문제점을 생각해보니:**
- 기능이 늘어날수록 main.py가 길어짐
- 엔드포인트가 섞여있어서 찾기 어려움
- 여러 명이 작업하면 충돌 발생

---

### 회사 방식 (Router 분리)

```python
# main.py (진입점만)
from fastapi import FastAPI
from routers import auth, files

app = FastAPI()
app.include_router(auth.router)
app.include_router(files.router)
```

```python
# routers/files.py (파일 관련만)
from fastapi import APIRouter

router = APIRouter(prefix="/api/files")

@router.post("/upload")
async def upload_file():
    pass

@router.get("")
async def list_files():
    pass
```

```python
# routers/auth.py (인증 관련만)
from fastapi import APIRouter

router = APIRouter(prefix="/api/auth")

@router.post("/login")
async def login():
    pass
```

**장점:**
- main.py는 조립만 (20줄)
- 기능별로 파일 분리 (찾기 쉬움)
- 팀 작업 시 충돌 최소화

**마치 레고 블록처럼**: 각 Router를 블록으로 만들고, main.py에서 조립하는 느낌

---

## 회사 프로젝트 구조 분석

### 첫 번째 프로젝트 구조

```
project-a/
├── app.py                      # 진입점
├── restful/                    # API 라우터
│   ├── router_board.py
│   ├── router_content.py
│   └── model/
│       └── board_model.py
```

**특징:**
- 라우터 파일명에 `router_` prefix
- 모델은 별도 폴더

---

### 두 번째 프로젝트 구조

```
project-b/
├── main.py                     # 진입점
├── routers/                    # API 라우터
│   ├── auth.py
│   ├── files.py
│   └── settings.py
├── models.py                   # 모델 통합
```

**특징:**
- 라우터 파일명 간결
- 모델 통합

---

### 두 방식 비교

| 항목 | Project A | Project B |
|------|-----------|-----------|
| 라우터 폴더 | `restful/` | `routers/` |
| 파일명 | `router_board.py` | `files.py` |
| 모델 | `model/board_model.py` | `models.py` |

**느낀 점:**
- 정답은 없다
- 프로젝트 규모와 팀 선호도에 따라 선택
- 중요한 건 **일관성**

---

## Router 패턴 이해하기

### APIRouter란?

FastAPI가 제공하는 "작은 FastAPI 앱" 같은 것.

```python
# router를 하나의 미니 앱으로 생각
router = APIRouter(prefix="/api/files", tags=["files"])

@router.post("/upload")  # 실제 경로: /api/files/upload
@router.get("")          # 실제 경로: /api/files
```

**prefix의 역할:**
- URL 앞에 자동으로 붙음
- 중복 제거

**tags의 역할:**
- Swagger UI에서 그룹화
- 문서 정리

---

### Router 만들고 등록하기

**1단계: Router 파일 생성**

```python
# routers/example.py
from fastapi import APIRouter

router = APIRouter(
    prefix="/api/examples",
    tags=["examples"]
)

@router.get("")
async def list_examples():
    return []

@router.post("")
async def create_example():
    return {}
```

**2단계: main.py에 등록**

```python
# main.py
from routers import example

app.include_router(example.router)
```

끝! 이렇게 간단하다.

---

## URL 설계 패턴

회사 코드를 보면서 URL 패턴도 배웠다.

### RESTful 기본

```
GET    /boards              # 목록
POST   /boards              # 생성
GET    /boards/{id}         # 조회
PUT    /boards/{id}         # 수정
DELETE /boards/{id}         # 삭제
```

### 하위 리소스

```
GET    /boards/{id}/contents      # 보드의 콘텐츠 목록
POST   /boards/{id}/contents      # 콘텐츠 생성
```

"보드 안의 콘텐츠"를 URL로 표현

### 특수 작업 (동사 허용)

```
POST   /boards/{id}/generate      # 생성 트리거
POST   /files/{id}/extract        # 추출 작업
```

CRUD로 표현 안 되는 작업은 동사 사용

---

## 실전 적용: 단계별 가이드

### Step 1: 폴더 생성

```bash
mkdir routers
```

### Step 2: Router 파일 작성

```python
# routers/example.py
from fastapi import APIRouter

router = APIRouter(
    prefix="/api/examples",
    tags=["examples"]
)

@router.get("")
async def list_examples():
    """목록 조회"""
    return {"examples": []}

@router.post("")
async def create_example(data: dict):
    """생성"""
    return {"id": "123"}
```

### Step 3: main.py 수정

```python
# main.py
from fastapi import FastAPI
from routers import example  # import

app = FastAPI()

# 등록
app.include_router(example.router)
```

### Step 4: 확인

```bash
# 서버 실행
uvicorn main:app --reload

# Swagger 확인
# http://localhost:8000/docs
```

---

## Import 순서 규칙

회사 코딩 규칙에서 정한 import 순서:

```python
# 1. 표준 라이브러리
import json
from datetime import datetime
from typing import Optional, List

# 2. 외부 패키지
from fastapi import APIRouter, HTTPException

# 3. 내부 모듈
from models import ExampleDto
from database import get_db
```

**왜 이렇게?**
- 가독성
- 의존성 명확
- 팀 협업 시 일관성

---

## prefix 사용 vs 미사용

### prefix 사용 (추천)

```python
router = APIRouter(prefix="/api/files")

@router.post("/upload")  # → /api/files/upload
@router.get("")          # → /api/files
```

**장점**: URL 중복 없음

### prefix 미사용

```python
router = APIRouter()

@router.post("/api/files/upload")  # 전체 경로
@router.get("/api/files")
```

**장점**: 경로가 명시적

**내 선택**: prefix 사용 (간결함)

---

## 실무 팁

### 1. 파일명 규칙

**명시적:**
```
router_board.py
router_content.py
```

**간결:**
```
files.py
auth.py
```

→ 팀과 상의해서 하나로 통일

### 2. 라우터 등록 순서

```python
# 의존성 순서
app.include_router(auth.router)      # 먼저 (다른 곳에서 사용)
app.include_router(files.router)
app.include_router(settings.router)
```

### 3. 하나의 Router에 얼마나?

- 너무 많으면 (20+ 엔드포인트): 분리 고려
- 적당히 (5-10개): 유지
- 관련 기능끼리 묶기

---

## 배운 점 정리

1. **Router 분리 = 모듈화**
   - main.py는 조립만
   - 기능별로 파일 분리

2. **prefix로 URL 중복 제거**
   - `/api/files` prefix
   - 각 엔드포인트는 `/upload` 같이 간결하게

3. **일관성이 중요**
   - 파일명, 폴더명, import 순서
   - 팀 규칙 따르기

4. **정답은 없다**
   - Project A 방식도 OK
   - Project B 방식도 OK
   - 프로젝트에 맞게 선택

---

## 체크리스트

Router 분리를 적용할 때:

- [ ] `routers/` 폴더 생성
- [ ] 기능별 Router 파일 작성
- [ ] `router = APIRouter()` 초기화
- [ ] `prefix`와 `tags` 설정
- [ ] main.py에 `include_router()`
- [ ] Import 순서 정리
- [ ] Swagger UI로 확인

---

## 참고 템플릿

복사해서 바로 쓸 수 있는 기본 Router:

```python
# routers/example.py
from fastapi import APIRouter
from typing import List

router = APIRouter(
    prefix="/api/examples",
    tags=["examples"]
)

@router.get("")
async def list_items():
    """목록 조회"""
    return []

@router.post("")
async def create_item(data: dict):
    """생성"""
    return {"id": "123"}

@router.get("/{id}")
async def get_item(id: str):
    """단일 조회"""
    return {"id": id}

@router.put("/{id}")
async def update_item(id: str, data: dict):
    """수정"""
    return {"id": id}

@router.delete("/{id}")
async def delete_item(id: str):
    """삭제"""
    return {"message": "deleted"}
```

```python
# main.py
from fastapi import FastAPI
from routers import example

app = FastAPI()
app.include_router(example.router)
```
