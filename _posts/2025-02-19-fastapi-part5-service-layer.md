---
title: "서비스 계층과 비즈니스 로직 분리"
date: 2025-02-19 13:00:00 +0900
categories: [Study, FastAPI]
tags: [fastapi, service-layer, architecture, python]
---

## 들어가며

회사에서 파일 업로드 API 코드를 읽다가 신기한 걸 봤다.

```python
@router.post("/upload")
async def upload_file(file: UploadFile, db: Session = Depends(get_db)):
    # ... 파일 저장 로직 (10줄)

    # 메타데이터 추출
    extractor = MetadataExtractor(
        api_key=settings.OPENAI_API_KEY,
        model=settings.OPENAI_MODEL
    )
    metadata = extractor.extract_metadata(db, pdf_text)

    # ... DB 저장 (5줄)
```

**라우터가 놀랍도록 짧았다.**

복잡한 메타데이터 추출 로직이 `MetadataExtractor` 클래스로 분리되어 있었다.

**"왜 이렇게 분리했을까?"**

코드를 더 읽어보니 **서비스 계층 패턴**이었다.

---

## 라우터가 뚱뚱해지는 문제

회사에 두 프로젝트가 있었는데, 스타일이 달랐다.

### Project-A: 라우터에 모든 로직

```python
@router.post("/boards/{board_id}/generate")
async def generate_board_content(
    board_id: str,
    db: Session = Depends(get_db),
    current_user: str = Depends(get_current_user)
):
    # 보드 조회 (5줄)
    board = db.query(Board).filter(...).first()
    if not board:
        raise HTTPException(404)

    # 콘텐츠 조회 (10줄)
    contents = db.query(Content).filter(...).all()

    # 프롬프트 생성 (20줄)
    prompt = f"""
    ... 복잡한 프롬프트 ...
    """

    # LLM 호출 (15줄)
    response = openai.chat.completions.create(...)

    # 결과 파싱 (10줄)
    result = json.loads(response.choices[0].message.content)

    # DB 저장 (10줄)
    for item in result:
        db.add(Generated(...))
    db.commit()

    return {"status": "success"}
```

**문제점:**
- 라우터가 70줄 이상
- 비즈니스 로직이 섞여있음
- 테스트 어려움
- 재사용 불가

---

### Project-B: 서비스로 분리

```python
# router
@router.post("/upload")
async def upload_file(
    file: UploadFile,
    db: Session = Depends(get_db)
):
    # 1. 파일 저장
    file_path = save_file(file)

    # 2. 텍스트 추출
    text = extract_text(file_path)

    # 3. 메타데이터 추출 (서비스 사용!)
    extractor = MetadataExtractor(
        api_key=settings.OPENAI_API_KEY,
        model="gpt-4o-mini"
    )
    metadata = extractor.extract_metadata(db, text)

    # 4. DB 저장
    save_to_db(db, file_path, metadata)

    return {"status": "success"}
```

**장점:**
- ✅ 라우터는 20줄 (흐름만)
- ✅ 복잡한 로직은 서비스에
- ✅ 테스트 가능
- ✅ 재사용 가능

**마치 요리사처럼**: 라우터는 주문 받고 서빙, 실제 요리는 주방(서비스)에서

---

## 계층 구조: 얇게 vs 두껍게

회사 코드를 보니 명확한 계층이 있었다.

```
┌─────────────────────────┐
│  Router (얇게)          │  ← 요청/응답 처리
│  - 파라미터 받기         │
│  - 의존성 주입           │
│  - 서비스 호출           │
│  - 응답 반환            │
├─────────────────────────┤
│  Service (두껍게)       │  ← 비즈니스 로직
│  - 복잡한 알고리즘       │
│  - 외부 API 호출         │
│  - 데이터 변환           │
├─────────────────────────┤
│  DB (영속성)            │  ← 데이터 저장/조회
└─────────────────────────┘
```

**원칙:**
- Router: 얇게 (Thin Controller)
- Service: 두껍게 (Fat Service)
- DB: 단순 CRUD

---

## 서비스 클래스 패턴

회사 코드에서 본 `MetadataExtractor` 클래스를 분석해봤다.

### 기본 구조

```python
# extraction/metadata_extractor.py
from openai import OpenAI
from sqlalchemy.orm import Session

class MetadataExtractor:
    """문서에서 메타데이터를 추출하는 서비스"""

    def __init__(self, api_key: str, model: str = "gpt-4o-mini"):
        """초기화: 설정 주입"""
        self.client = OpenAI(api_key=api_key)
        self.model = model

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # Public 메서드 (외부 인터페이스)
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    def extract_metadata(self, db: Session, text: str) -> dict:
        """메타데이터 추출 (공개 메서드)"""
        fields = self._load_metadata_fields(db)
        prompt = self._build_prompt(text, fields)
        response = self._call_llm(prompt)
        return self._parse_response(response, fields)

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # Private 메서드 (내부 헬퍼)
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    def _load_metadata_fields(self, db: Session):
        """DB에서 메타데이터 필드 로드"""
        return db.query(MetadataField)\
            .filter(MetadataField.is_active == True)\
            .all()

    def _build_prompt(self, text: str, fields) -> str:
        """프롬프트 생성"""
        field_instructions = [
            f"- {f.name}: {f.description}" for f in fields
        ]
        return f"""
        다음 문서에서 메타데이터를 추출하라:
        {text[:10000]}

        필드: {', '.join(field_instructions)}
        """

    def _call_llm(self, prompt: str):
        """LLM 호출"""
        return self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}]
        )

    def _parse_response(self, response, fields) -> dict:
        """응답 파싱 및 검증"""
        raw = response.choices[0].message.content
        data = json.loads(raw)

        # 모든 필드 확인 및 기본값 설정
        result = {}
        for f in fields:
            result[f.name] = data.get(f.name, "")

        return result
```

### 핵심 포인트

**1. 생성자: 설정 주입**

```python
def __init__(self, api_key: str, model: str = "gpt-4o-mini"):
    self.client = OpenAI(api_key=api_key)
    self.model = model
```

- 외부에서 설정 전달
- 테스트 시 Mock 주입 가능

---

**2. Public 메서드: 하나**

```python
def extract_metadata(self, db: Session, text: str) -> dict:
    """외부에서 호출하는 유일한 메서드"""
    ...
```

- 외부 인터페이스
- 간단하고 명확

---

**3. Private 메서드: 여러 개**

```python
def _load_metadata_fields(self, db):  # 필드 로드
def _build_prompt(self, text, fields):  # 프롬프트 생성
def _call_llm(self, prompt):  # LLM 호출
def _parse_response(self, response):  # 응답 파싱
```

- 언더스코어(`_`) prefix
- 내부에서만 사용
- 작은 단위로 분리

**마치 공장처럼**: Public 메서드는 완제품 출력, Private 메서드는 제조 공정

---

## Public vs Private 메서드

회사 코드를 보니 명확한 규칙이 있었다.

### 규칙

| 구분 | 네이밍 | 역할 | 호출 |
|------|--------|------|------|
| **Public** | `extract_metadata` | 외부 인터페이스 | 라우터에서 |
| **Private** | `_load_fields` | 내부 헬퍼 | 클래스 내부에서만 |

### 예시

```python
class MetadataExtractor:
    # Public (외부 인터페이스)
    def extract_metadata(self, db, text):
        """이것만 외부에서 호출"""
        fields = self._load_fields(db)      # Private 호출
        prompt = self._build_prompt(text)   # Private 호출
        return self._call_llm(prompt)       # Private 호출

    # Private (내부 헬퍼)
    def _load_fields(self, db):
        """외부에서 직접 호출 안 함"""
        return db.query(...).all()

    def _build_prompt(self, text):
        """외부에서 직접 호출 안 함"""
        return f"Extract from: {text}"

    def _call_llm(self, prompt):
        """외부에서 직접 호출 안 함"""
        return self.client.chat.completions.create(...)
```

**왜 이렇게?**
- Public: 안정적인 API (변경 최소화)
- Private: 구현 디테일 (자유롭게 변경)

---

## 설정 주입 패턴

회사 코드를 보니 설정을 외부에서 주입했다.

### 방식 1: 직접 전달

```python
# router
from config import settings

@router.post("/extract")
async def extract(file: UploadFile, db: Session = Depends(get_db)):
    # 설정을 직접 전달
    extractor = MetadataExtractor(
        api_key=settings.OPENAI_API_KEY,
        model=settings.OPENAI_MODEL,
        temperature=0.1
    )
    result = extractor.extract_metadata(db, text)
    return result
```

---

### 방식 2: 팩토리 함수

```python
# services.py
def get_metadata_extractor():
    """서비스 인스턴스 생성 (설정 자동 주입)"""
    return MetadataExtractor(
        api_key=settings.OPENAI_API_KEY,
        model=settings.OPENAI_MODEL
    )

# router
@router.post("/extract")
async def extract(
    file: UploadFile,
    db: Session = Depends(get_db),
    extractor: MetadataExtractor = Depends(get_metadata_extractor)  # 의존성!
):
    result = extractor.extract_metadata(db, text)
    return result
```

**장점:**
- 설정 중앙화
- 테스트 시 Mock 주입 쉬움

---

## 실전 예시: 세 가지 Extractor

회사 프로젝트에는 세 가지 Extractor가 있었다.

### 1. MetadataExtractor (문서)

```python
class MetadataExtractor:
    """PDF, Word 등 문서에서 메타데이터 추출"""

    def __init__(self, api_key: str, model: str):
        self.client = OpenAI(api_key=api_key)
        self.model = model

    def extract_metadata(self, db: Session, text: str) -> dict:
        fields = self._load_metadata_fields(db)
        prompt = self._build_prompt(text, fields)

        response = self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}]
        )

        return self._safe_json_parse(response.choices[0].message.content, fields)
```

---

### 2. ImageMetadataExtractor (이미지)

```python
class ImageMetadataExtractor:
    """이미지에서 메타데이터 추출 (Vision API)"""

    def __init__(self, api_key: str, model: str):
        self.client = OpenAI(api_key=api_key)
        self.model = model

    def extract_metadata(
        self,
        db: Session,
        image_bytes: bytes,
        image_format: str = "jpeg"
    ) -> dict:
        fields = self._load_metadata_fields(db)
        prompt = self._build_prompt(fields)

        # Base64 인코딩
        base64_image = base64.b64encode(image_bytes).decode('utf-8')

        # Vision API 호출
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {
                    "role": "user",
                    "content": [
                        {"type": "text", "text": prompt},
                        {
                            "type": "image_url",
                            "image_url": {
                                "url": f"data:image/{image_format};base64,{base64_image}"
                            }
                        }
                    ]
                }
            ]
        )

        return self._safe_json_parse(response.choices[0].message.content, fields)
```

---

### 3. YouTubeMetadataExtractor (유튜브)

```python
class YouTubeMetadataExtractor:
    """유튜브 자막에서 메타데이터 추출"""

    def __init__(self, api_key: str, model: str):
        self.client = OpenAI(api_key=api_key)
        self.model = model

    def extract_metadata(self, db: Session, transcript: str) -> dict:
        fields = self._load_metadata_fields(db)
        prompt = self._build_prompt(transcript, fields)

        response = self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}]
        )

        return self._safe_json_parse(response.choices[0].message.content, fields)
```

---

### 공통점

| 항목 | 설명 |
|------|------|
| **구조** | `__init__` + Public 메서드 + Private 메서드 |
| **설정 주입** | API 키, 모델명 |
| **Public 메서드** | `extract_metadata` (하나) |
| **Private 메서드** | `_load_fields`, `_build_prompt`, `_safe_json_parse` |

### 차이점

| Extractor | 입력 | API |
|-----------|------|-----|
| Metadata | 텍스트 | Chat API |
| Image | 이미지 바이트 | Vision API |
| YouTube | 자막 텍스트 | Chat API |

**마치 조리법처럼**: 재료는 다르지만(텍스트, 이미지, 자막), 조리 과정은 비슷함(초기화 → 로드 → 변환 → 호출 → 파싱)

---

## 라우터에서 사용하기

회사 코드를 보니 라우터는 정말 단순했다.

```python
# routers/files.py
from extraction.metadata_extractor import MetadataExtractor
from extraction.image_metadata_extractor import ImageMetadataExtractor
from extraction.youtube_metadata_extractor import YouTubeMetadataExtractor
from config import settings

@router.post("/upload")
async def upload_file(
    file: UploadFile = File(...),
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 1. 파일 저장
    file_path = await save_to_external_api(file)

    # 2. 파일 타입별 처리
    if is_document_file(file.filename):
        # 문서: 텍스트 추출 + 메타데이터 추출
        text = await extract_text_from_pdf(file_path)

        extractor = MetadataExtractor(
            api_key=settings.OPENAI_API_KEY,
            model=settings.OPENAI_MODEL
        )
        metadata = extractor.extract_metadata(db, text)

    elif is_image_file(file.filename):
        # 이미지: 이미지 다운로드 + Vision API
        image_bytes = await download_image(file_path)

        extractor = ImageMetadataExtractor(
            api_key=settings.OPENAI_API_KEY,
            model=settings.OPENAI_MODEL
        )
        metadata = extractor.extract_metadata(db, image_bytes, "jpeg")

    # 3. DB 저장
    file_metadata = FileMetadata(
        id=str(uuid.uuid4()),
        user_id=current_user.id,
        file_name=file.filename,
        file_path=file_path,
        extracted_metadata=metadata
    )
    db.add(file_metadata)
    db.commit()

    return {"status": "success", "metadata": metadata}
```

**라우터 역할:**
- 파일 받기
- 타입 판단
- 적절한 Extractor 선택
- DB 저장

**Extractor 역할:**
- 복잡한 추출 로직
- LLM/Vision API 호출
- 응답 파싱 및 검증

---

## Project-A에는 왜 없을까?

회사의 다른 프로젝트(Project-A)를 보니 서비스 계층이 없었다.

### Project-A 특징

```python
@router.get("/boards")
async def get_boards(
    db: Session = Depends(get_db),
    current_user: str = Depends(get_current_user)
):
    # 단순한 CRUD
    boards = db.query(Board).filter(Board.user_id == current_user).all()
    return boards

@router.post("/boards")
async def create_board(
    data: BoardCreateDto,
    db: Session = Depends(get_db),
    current_user: str = Depends(get_current_user)
):
    # 단순한 생성
    board = Board(id=str(uuid.uuid4()), user_id=current_user, title=data.title)
    db.add(board)
    db.commit()
    return board
```

**왜 서비스 계층이 없나?**
- 비즈니스 로직이 단순 (CRUD만)
- 외부 API 호출 없음
- 복잡한 알고리즘 없음

---

### Project-B 특징

```python
@router.post("/upload")
async def upload_file(...):
    # 복잡한 로직 → 서비스로 분리
    extractor = MetadataExtractor(...)
    metadata = extractor.extract_metadata(db, text)
    ...
```

**왜 서비스 계층이 있나?**
- 복잡한 메타데이터 추출
- 외부 API (OpenAI) 호출
- 프롬프트 생성, 응답 파싱

---

## 언제 서비스 계층이 필요한가?

회사 코드를 보며 정리한 기준.

### 서비스 계층 필요 ✅

**1. 외부 API 호출**

```python
# OpenAI, AWS S3, 결제 API 등
class PaymentService:
    def process_payment(self, amount: int, card: str):
        response = stripe.charge(...)
        return self._validate_response(response)
```

---

**2. 복잡한 알고리즘**

```python
# 추천 알고리즘, 점수 계산 등
class RecommendationService:
    def recommend_courses(self, user_id: str):
        history = self._get_user_history(user_id)
        scores = self._calculate_scores(history)
        return self._rank_by_score(scores)
```

---

**3. 여러 단계 처리**

```python
# 파일 업로드 → 변환 → 추출 → 저장
class DocumentProcessor:
    def process_document(self, file):
        converted = self._convert_to_pdf(file)
        text = self._extract_text(converted)
        metadata = self._extract_metadata(text)
        return self._save_to_db(metadata)
```

---

**4. 재사용 필요**

```python
# 여러 라우터에서 사용
class EmailService:
    def send_email(self, to: str, subject: str, body: str):
        ...

# 라우터 A
email_service.send_email(user.email, "Welcome", body)

# 라우터 B
email_service.send_email(admin.email, "Alert", body)
```

---

### 서비스 계층 불필요 ❌

**1. 단순 CRUD**

```python
# 라우터에 직접
@router.get("/boards")
async def get_boards(db: Session = Depends(get_db)):
    return db.query(Board).all()
```

---

**2. 단순 변환**

```python
# 유틸 함수로 충분
def format_date(date: datetime) -> str:
    return date.strftime("%Y-%m-%d")
```

---

**3. 한 곳에서만 사용**

```python
# 재사용 안 함 → 라우터에 직접
@router.get("/stats")
async def get_stats(db: Session = Depends(get_db)):
    count = db.query(Board).count()
    return {"total": count}
```

---

## 실전 적용: 서비스 클래스 만들기

### Step 1: 서비스 폴더 생성

```bash
mkdir services
touch services/__init__.py
touch services/notification_service.py
```

---

### Step 2: 서비스 클래스 작성

```python
# services/notification_service.py
from typing import List
from sqlalchemy.orm import Session
from db_models import User, Notification
from datetime import datetime
import uuid

class NotificationService:
    """알림 전송 서비스"""

    def __init__(self, email_api_key: str):
        """초기화: 외부 API 설정 주입"""
        self.email_api_key = email_api_key

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # Public 메서드
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    def send_notification(
        self,
        db: Session,
        user_id: str,
        title: str,
        message: str
    ) -> Notification:
        """알림 전송 (공개 인터페이스)"""
        # 1. 사용자 조회
        user = self._get_user(db, user_id)

        # 2. 알림 생성
        notification = self._create_notification(db, user_id, title, message)

        # 3. 이메일 전송 (선택적)
        if user.email_enabled:
            self._send_email(user.email, title, message)

        return notification

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # Private 메서드 (헬퍼)
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    def _get_user(self, db: Session, user_id: str) -> User:
        """사용자 조회"""
        user = db.query(User).filter(User.id == user_id).first()
        if not user:
            raise ValueError(f"User {user_id} not found")
        return user

    def _create_notification(
        self,
        db: Session,
        user_id: str,
        title: str,
        message: str
    ) -> Notification:
        """알림 DB 저장"""
        notification = Notification(
            id=str(uuid.uuid4()),
            user_id=user_id,
            title=title,
            message=message,
            created_at=datetime.now(),
            is_read=False
        )
        db.add(notification)
        db.commit()
        db.refresh(notification)
        return notification

    def _send_email(self, email: str, subject: str, body: str):
        """이메일 전송 (외부 API)"""
        # 실제로는 SendGrid, AWS SES 등 사용
        print(f"[Email] To: {email}, Subject: {subject}")
        # email_api.send(to=email, subject=subject, body=body)
```

---

### Step 3: 라우터에서 사용

```python
# routers/notifications.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from database import get_db
from auth import get_current_user
from services.notification_service import NotificationService
from config import settings

router = APIRouter(prefix="/api/notifications")

@router.post("")
async def send_notification(
    title: str,
    message: str,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    # 서비스 인스턴스 생성
    service = NotificationService(email_api_key=settings.EMAIL_API_KEY)

    # 서비스 호출 (간단!)
    notification = service.send_notification(
        db=db,
        user_id=current_user.id,
        title=title,
        message=message
    )

    return {"status": "sent", "notification_id": notification.id}
```

---

## 배운 점 정리

1. **서비스 계층 = 비즈니스 로직 분리**
   - 라우터: 얇게 (요청/응답)
   - 서비스: 두껍게 (로직)

2. **Public vs Private**
   - Public: 외부 인터페이스 (한두 개)
   - Private: 내부 헬퍼 (여러 개, `_` prefix)

3. **설정 주입**
   - `__init__`으로 외부 설정 받기
   - 테스트 시 Mock 주입 가능

4. **언제 사용?**
   - ✅ 외부 API 호출
   - ✅ 복잡한 알고리즘
   - ✅ 여러 단계 처리
   - ✅ 재사용 필요
   - ❌ 단순 CRUD

5. **Project-A vs Project-B**
   - CRUD 중심 → 서비스 불필요
   - 복잡한 로직 → 서비스 필요

---

## 체크리스트

서비스 클래스를 만들 때:

- [ ] `__init__`에서 설정 주입
- [ ] Public 메서드 1-2개 (외부 인터페이스)
- [ ] Private 메서드 여러 개 (`_` prefix)
- [ ] 작은 단위로 메서드 분리
- [ ] 클래스명: `XxxService` 또는 `XxxExtractor`
- [ ] 파일명: `services/xxx_service.py`
- [ ] 라우터는 얇게 유지

---

## 다음 편 예고

**Part 6: 실무 패턴 총정리**

지금까지 배운 모든 패턴을 정리하고, 프로젝트 규모별로 언제 어떤 패턴을 써야 하는지 알아보자.

- Router 분리
- DTO 네이밍
- 의존성 주입
- CRUD 패턴
- 서비스 계층

**실무에서 바로 쓸 수 있는 베스트 프랙티스 체크리스트**

---

## 참고 템플릿

복사해서 바로 쓸 수 있는 서비스 클래스 템플릿:

```python
# services/example_service.py
from typing import List, Dict, Any
from sqlalchemy.orm import Session
from datetime import datetime
import uuid

class ExampleService:
    """
    비즈니스 로직을 담당하는 서비스 클래스

    Usage:
        service = ExampleService(api_key="xxx", setting="yyy")
        result = service.process_data(db, input_data)
    """

    def __init__(self, api_key: str, setting: str = "default"):
        """
        초기화: 외부 설정 주입

        Args:
            api_key: 외부 API 키
            setting: 추가 설정 (선택)
        """
        self.api_key = api_key
        self.setting = setting

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # Public 메서드 (외부 인터페이스)
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    def process_data(
        self,
        db: Session,
        input_data: str
    ) -> Dict[str, Any]:
        """
        데이터 처리 (공개 메서드)

        Args:
            db: Database session
            input_data: 입력 데이터

        Returns:
            처리 결과 딕셔너리

        Raises:
            RuntimeError: 처리 실패 시
        """
        # 1. 데이터 로드
        config = self._load_config(db)

        # 2. 데이터 변환
        transformed = self._transform_data(input_data, config)

        # 3. 외부 API 호출
        api_result = self._call_external_api(transformed)

        # 4. 결과 검증
        validated = self._validate_result(api_result)

        # 5. 반환
        return validated

    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    # Private 메서드 (내부 헬퍼)
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    def _load_config(self, db: Session) -> Dict[str, Any]:
        """
        설정 로드 (내부 메서드)

        Args:
            db: Database session

        Returns:
            설정 딕셔너리
        """
        # DB에서 설정 조회
        config = db.query(Config).first()
        return config.to_dict() if config else {}

    def _transform_data(
        self,
        data: str,
        config: Dict[str, Any]
    ) -> str:
        """
        데이터 변환 (내부 메서드)

        Args:
            data: 입력 데이터
            config: 설정

        Returns:
            변환된 데이터
        """
        # 변환 로직
        transformed = data.upper() if config.get("uppercase") else data
        return transformed

    def _call_external_api(self, data: str) -> Dict[str, Any]:
        """
        외부 API 호출 (내부 메서드)

        Args:
            data: 요청 데이터

        Returns:
            API 응답

        Raises:
            RuntimeError: API 호출 실패 시
        """
        try:
            # 실제로는 requests, httpx 등 사용
            # response = requests.post(
            #     "https://api.example.com/process",
            #     headers={"Authorization": f"Bearer {self.api_key}"},
            #     json={"data": data}
            # )
            # return response.json()

            # 예시 응답
            return {"status": "success", "result": data}
        except Exception as e:
            raise RuntimeError(f"API call failed: {e}")

    def _validate_result(self, result: Dict[str, Any]) -> Dict[str, Any]:
        """
        결과 검증 (내부 메서드)

        Args:
            result: API 응답

        Returns:
            검증된 결과

        Raises:
            ValueError: 검증 실패 시
        """
        if result.get("status") != "success":
            raise ValueError("Result validation failed")

        return result
```

**사용 예시:**

```python
# routers/example.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from database import get_db
from services.example_service import ExampleService
from config import settings

router = APIRouter(prefix="/api/examples")

@router.post("/process")
async def process_example(
    input_data: str,
    db: Session = Depends(get_db)
):
    # 서비스 인스턴스 생성
    service = ExampleService(
        api_key=settings.EXAMPLE_API_KEY,
        setting="production"
    )

    # 서비스 호출
    result = service.process_data(db, input_data)

    return {"result": result}
```

---

**회사에서 배우는 중... 🚀**
