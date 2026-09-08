---
title: "FastAPI 쿼리 모델이 조용히 안 펼쳐진 순간"
date: 2026-09-08 17:00:00 +0900
categories: [Backend]
tags: [fastapi, pydantic, python, pagination]
---

랜덤 템플릿 추천 API에 페이지네이션을 붙이는 작업이었다.

`/random-templates?offset=0&limit=30` — 충분히 단순해 보였다. 문제는 "랜덤"이 들어가는 순간부터였다. 페이지를 넘길 때마다 셔플이 다시 일어나면 2페이지에서 1페이지 항목이 다시 나올 수 있다. 그래서 `seed`를 쿼리 파라미터로 받아서 클라이언트가 한 세션 동안 같은 순서를 유지하게 했다.

`seed`를 어디에 선언할지 잠깐 고민하다가, 기존에 있던 `PaginationQuery` 모델에 넣지 않고 엔드포인트 시그니처에 따로 꺼내 놨다. 로직상 `seed`는 페이지네이션과 다른 역할이라고 생각했다. pagination은 공통 모델로 쓰고, seed는 이 엔드포인트에만 해당하니까.

그 판단이 문제였다.

---

FastAPI는 Pydantic 모델을 쿼리 파라미터로 쓸 때 모델 필드를 개별 쿼리 파라미터로 "펼쳐서" 인식한다. 그런데 이 펼치기는 **Pydantic 모델과 별개 쿼리 파라미터가 같은 시그니처에 섞여 있으면 조용히 엇나갈 수 있다.** 에러가 나는 게 아니다. 서버는 잘 뜨고, 요청도 들어온다. 달라지는 건 OpenAPI 스키마 쪽이었다.

Swagger에서 `/random-templates`를 열어봤더니 `seed`만 쿼리 파라미터로 잡혀 있고, `offset`/`limit`은 보이지 않았다. 실제로 `offset`과 `limit`을 쿼리스트링으로 보내도 모델이 받질 못했다. 파라미터 자체가 없는 것처럼 동작했다.

처음엔 캐시 문제인 줄 알았다. FastAPI를 재기동하고, uvicorn을 껐다 켰다. 스키마가 안 바뀌니까 Swagger 페이지 캐시인가 싶어서 시크릿 모드로도 열어봤다. 다 소용없었다.

---

이 시점에서 실제로 어떤 데이터를 다루고 있었는지 보면, 리포지터리 레이어는 이런 구조였다.

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

DB에서 전체를 한 번 읽어 캐싱한 뒤, 랜덤 셔플은 메모리에서 처리하는 방식이었다. 페이지네이션도 DB 쿼리 레벨이 아니라 파이썬 슬라이싱으로 처리하는 구조였으므로, API 레이어에서 `offset`/`limit`을 제대로 받지 못하면 그냥 전체가 반환되거나 기본값으로 잘렸다.

즉, 파라미터가 안 들어오는데 400이나 422로 튀지 않고 결과가 그냥 조용히 이상해지는 상황이었다.

---

FastAPI에서 Pydantic 모델을 쿼리 파라미터로 쓸 때의 동작은 버전마다, 그리고 `Annotated` 사용 여부에 따라 미묘하게 다르다. 이 경우 확인된 증상은 명확했다. `seed`를 시그니처에서 분리한 순간부터 `PaginationQuery` 모델이 제대로 펼쳐지지 않았다.

해결 방법은 단순했다. `seed`를 `PaginationQuery` 안으로 합쳤다. 랜덤 엔드포인트 전용 쿼리 모델을 만들고, `offset`, `limit`, `seed`를 한 곳에서 선언했다. 그러자 Swagger에 세 필드가 모두 나타났고, 검증도 정상적으로 동작했다.

`offset=0` 같은 경계값과 `limit=301` 같은 초과값도 이 시점에 함께 검증했다. 모델 안에서 `ge=0`, `le=300` 같은 제약을 걸었고, 잘못된 값이 들어오면 422가 떨어졌다.

---

왜 섞으면 안 되는지는 아직 정확히 파악하지 못했다. FastAPI 내부에서 의존성 주입과 파라미터 파싱이 어떤 순서로 일어나는지, 별개 파라미터가 있을 때 모델의 `model_fields` 펼치기가 스킵되는 조건이 뭔지는 소스를 더 들여다봐야 한다. 에러 없이 조용히 잘못되는 케이스라 재현 조건을 좁히는 게 생각보다 까다롭다.

당장은 "같이 쓰일 파라미터는 같은 모델 안에" 라는 규칙을 따르고 있다. 이게 맞는 정리인지는 아직 확신이 없다.