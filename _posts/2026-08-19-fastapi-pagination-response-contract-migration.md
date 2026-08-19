---
title: "FastAPI 응답에서 status를 빼고 PaginationDto 규격으로 옮기기"
date: 2026-08-19 17:00:00 +0900
categories: [Backend]
tags: [FastAPI, Pydantic, Pagination, API설계]
---

API 응답에 `status: "success"` 같은 필드를 넣는 관습이 있다. 서버가 직접 OK/FAIL을 body에 명시하는 방식이다. 처음엔 나쁘지 않아 보인다. 클라이언트가 한 곳만 보면 되니까.

문제는 HTTP가 이미 그 역할을 한다는 점이다. 200, 400, 500 — 상태는 이미 있다. body의 `status` 는 중복이고, 더 나쁜 건 계약이 두 레이어로 쪼개진다는 것이다. HTTP 200이면서 body는 `"status": "error"` 를 돌려주는 구현도 나온다. 클라이언트는 두 곳을 다 확인해야 한다. 이번에 템플릿 조회 API에서 그 필드를 걷어냈다.

---

응답 구조도 같이 바꿨다. 기존:

```json
{
  "status": "success",
  "count": 276,
  "templates": [...]
}
```

변경 후:

```json
{
  "totalCount": 276,
  "list": [...]
}
```

`count` → `totalCount`, `templates` → `list`. 이름 변경이 목적이 아니었다. `totalCount` 는 페이지네이션 이전 전체 개수로 정의했다. 클라이언트가 "전체 276개 중 지금 30개를 보고 있다"는 걸 알아야 페이지 UI를 그릴 수 있다. 기존의 `count` 는 그게 현재 페이지 개수인지 전체 개수인지 이름만 봐서는 알 수 없었다.

페이지네이션 파라미터는 이렇게 정했다:

```
offset: number = 1    # 1-based 페이지 번호
limit:  number = 30   # 한 페이지 크기
```

서버 내부에서 SQL skip으로 바꿀 때:

```python
skip = (offset - 1) * limit
```

`offset` 을 1-based로 쓰는 건 클라이언트 편의다. "2페이지"를 `offset=2` 로 보내는 게 `offset=30` 으로 보내는 것보다 자연스럽다. 변환 책임은 서버가 진다.

---

FastAPI에서 쿼리 파라미터를 다루는 방법이 두 가지 있다. Pydantic 모델로 묶는 방법, `Query(...)` 를 인자마다 직접 선언하는 방법. 둘을 섞으면 OpenAPI 문서가 예상대로 나오지 않는 경우가 있다.

Pydantic 모델을 쿼리 파라미터로 쓰려면 FastAPI 0.95 이후 `Annotated[Model, Query()]` 패턴이 필요하다. 그냥 `params: PaginationQuery` 로 선언하면 FastAPI는 그걸 request body로 해석한다. 나는 이걸 Swagger 문서를 열기 전까지 몰랐다. 파라미터가 통째로 사라져 있었다.

동작 여부를 확인하는 가장 빠른 방법은 그냥 쳐보는 거였다:

```bash
curl "http://localhost/api/random-templates?offset=1&limit=5"
```

문서가 맞더라도 실제 요청이 파라미터를 인식하는지는 별개다. 둘 다 확인해야 한다.

---

랜덤 추천 API에 페이지네이션을 붙이는 건 별도로 검토했다. 매 요청마다 순서가 달라지면 `offset=1` 과 `offset=2` 사이에 중복이나 누락이 생긴다. seed로 정렬을 고정하는 방법도 있는데, 이번에는 그렇게 하지 않았다. 랜덤성은 유지하면서 응답 포맷만 `totalCount/list` 구조로 맞추는 방향으로 정리했다.

중복·누락이 허용 가능한 트레이드오프인지는 클라이언트 쪽에 명확히 전달해야 한다. 그 내용을 API 문서에 아직 제대로 써두지 않았다.