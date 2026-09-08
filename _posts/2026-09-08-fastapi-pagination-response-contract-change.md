---
title: "FastAPI 응답에서 status를 빼고 totalCount/list 페이지네이션으로 바꾸기"
date: 2026-09-08 17:00:00 +0900
categories: [Backend]
tags: [fastapi, python, api-design, pagination]
---

API 계약을 고칠 때 제일 먼저 건드려야 하는 게 뭔지 이제는 좀 안다. 응답 body에 `status: "success"` 같은 필드를 달아두는 패턴이다.

```json
{
  "status": "success",
  "count": 7,
  "templates": [...]
}
```

이 형태로 한동안 운영하고 있었다. 문제는 클라이언트 쪽이 HTTP 상태코드를 믿지 않고 body의 `status` 를 파싱해서 분기를 태운다는 거였다. 서버는 400을 내보내면서도 body에 `status: "success"` 를 담는 구현 실수가 생길 수 있다. 실제로 그 버그가 한 번 나왔다.

---

## HTTP 상태코드가 이미 있다

성공·실패 시그널은 HTTP가 이미 들고 있다. 200 OK면 성공, 4xx면 클라이언트 문제, 5xx면 서버 문제다. body에 별도 `status` 를 두면 두 채널이 생기고, 둘이 어긋날 때 클라이언트가 뭘 믿어야 할지 모른다.

그래서 성공 응답에서 `status` 를 뺐다. 오류 응답은 아래 구조로 통일했다.

```json
{
  "error_code": "TEMPLATE_QUERY_ERROR",
  "error_detail": "..."
}
```

FastAPI에서 오류는 `HTTPException` 으로 던지면 자동으로 HTTP 상태코드가 붙는다. body를 별도로 꾸밀 게 없다.

---

## count/templates → totalCount/list

`status` 를 뺀 김에 응답 구조도 맞췄다. 기존 `count` / `templates` 키를 `totalCount` / `list` 로 바꿨다. 프론트엔드 쪽에서 이미 이 컨벤션으로 쓰고 있었는데, 이 엔드포인트만 다른 이름을 쓰고 있었던 것이다.

```json
{
  "totalCount": 7,
  "list": [...],
  "main_tool": "TOOL_CODE"
}
```

`main_tool` 은 요청 경로 파라미터를 그대로 돌려준다. 클라이언트가 어떤 도구 기준으로 조회한 결과인지 응답 안에서도 확인할 수 있게 하려고 넣었다. 이게 꼭 필요하냐는 의문이 있었는데, 클라이언트 캐싱 로직에서 키를 맞출 때 편하다는 의견을 받아들였다.

---

## offset은 행 번호가 아니다

쿼리 파라미터는 이렇게 정했다.

```
GET /templates-by-tool/{tool_id}?offset=1&limit=30
```

여기서 `offset` 이 헷갈렸다. SQL의 `OFFSET` 은 건너뛸 행 수다. 0부터 시작한다. 그런데 이 API의 `offset` 은 **1-based 페이지 번호**다. NestJS 쪽에서 먼저 이 컨벤션을 쓰고 있었고, Python 쪽도 맞춰야 했다.

실제 skip 계산은 이렇게 된다.

```python
skip = (offset - 1) * limit
```

`offset=1, limit=30` 이면 `skip=0` — 첫 30개. `offset=2, limit=30` 이면 `skip=30` — 다음 30개. `offset` 이름이 SQL 의미와 달라서 처음 보는 사람은 헷갈릴 수 있다. 파라미터 이름을 `page` 로 바꾸자는 의견도 있었는데, 기존 클라이언트 코드가 `offset` 으로 이미 굳어 있어서 그냥 두기로 했다.

---

## 데이터 모델 쪽은 이미 깔끔했다

저장소 레이어는 이미 dataclass로 정리되어 있었다.

```python
@dataclass
class Template:
    project_id: str
    project_title: str
    tools: List[str] = field(default_factory=list)
    tool_names: List[str] = field(default_factory=list)
    categories: List[str] = field(default_factory=list)
    canvas_objects: List[str] = field(default_factory=list)
    activity_type: str = ""
    learning_goal: str = ""
    keywords: List[str] = field(default_factory=list)
    keywords_en: List[str] = field(default_factory=list)
    description: str = ""
    grade: Optional[int] = None
    semester: Optional[int] = None
    unit_code: str = ""
    school_level: str = ""
```

DB에서 JSON 컬럼으로 저장된 필드들(`tools`, `categories`, `keywords` 등)을 `json.loads` 로 파싱해서 리스트로 복원한다. 이 부분이 없으면 문자열로 떨어져서 클라이언트가 다시 파싱해야 한다.

```python
templates.append(Template(
    project_id=row["project_id"],
    project_title=row["project_title"],
    tools=json.loads(row["tools"]) if row["tools"] else [],
    tool_names=json.loads(row["tool_names"]) if row["tool_names"] else [],
    # ...
    keywords_en=json.loads(row["keywords_en"]) if row.get("keywords_en") else [],
))
```

`row.get("keywords_en")` 과 `row["keywords_en"]` 을 섞어 쓰는 게 눈에 띈다. 컬럼이 나중에 추가된 흔적이다. 언젠가 정리해야 하지만 지금 당장 건드릴 이유는 없었다.

---

## 랜덤 추천에 페이지네이션을 붙이면

문제가 하나 남아 있다. "랜덤 템플릿 추천" 엔드포인트도 같은 페이지네이션 형식을 써야 하는데, 랜덤 정렬 + 페이지네이션은 사실 앞뒤가 맞지 않는다. 2페이지를 요청했을 때 1페이지에서 봤던 항목이 다시 나올 수 있다. `ORDER BY RAND()` 는 요청마다 순서가 바뀐다.

이번에는 **응답 형식만 페이지네이션 규격에 맞추고 실제 정렬은 랜덤을 유지**하는 것으로 마무리했다. `totalCount` 는 전체 개수를 내려주지만 페이지 간 연속성은 보장하지 않는다. 클라이언트가 "다음 페이지"를 누를 이유가 없는 UX라서 지금은 이걸로 충분하다는 판단이었다.

이게 나중에 문제가 될지는 모르겠다.