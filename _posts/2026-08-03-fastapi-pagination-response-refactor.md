---
title: "FastAPI 목록 API를 Nest 스타일로 맞추면서 재본 것들"
date: 2026-08-03 17:00:00 +0900
categories: [Backend]
tags: [fastapi, api-design, pagination, pydantic]
---

프론트에서 연락이 왔다. NestJS로 돌아가는 다른 서비스와 목록 응답 포맷을 맞춰달라고. `totalCount`, `list` 두 필드로. `status` 같은 건 빼달라고.

내가 운영하던 응답은 이렇게 생겼다.

```json
{
  "status": "success",
  "count": 30,
  "templates": [...]
}
```

`count`, `templates` → `totalCount`, `list`. 필드명만 갈면 되는 줄 알았는데, 실제로 손대보니 그게 아니었다.

## count 를 totalCount 로 바꾸는 게 왜 rename 이 아니냐

기존 코드에서 `count` 자리에 들어가던 값은 `len(items)` 였다. 지금 응답에 들어간 항목 수. 30개 요청하면 30이 나갔고, 마지막 페이지에서 7개 남으면 7이 나갔다.

프론트가 원한 `totalCount` 는 그게 아니다. **필터 조건이 걸린 상태에서 페이지네이션 이전의 총 개수**. 클라이언트가 총 페이지 수를 계산할 때 쓰는 값. 그러니까 이렇게 나가야 한다.

```json
{
  "totalCount": 276,
  "list": [ /* 30개 */ ]
}
```

이걸 놓치고 그냥 이름만 바꾸면 프론트 페이지네이션 UI가 조용히 망가진다. 에러도 안 뜨고, 그냥 페이지 버튼이 하나만 나온다. 재현하기 전엔 알기도 어렵다.

값을 두 개 만들어야 한다는 뜻이다. 하나는 필터 조건까지 다 태운 `COUNT(*)`, 하나는 거기에 `LIMIT/OFFSET` 을 얹은 실제 리스트. 쿼리를 두 번 날린다. 여기서 잠깐 고민했다 — 서브쿼리 하나로 window function 써서 붙일까. 답은 안 했다. 지금 트래픽에선 두 쿼리로 충분하고, 인덱스만 제대로 있으면 COUNT 는 싸다. 재보고 문제 생기면 그때 바꾸기로.

## offset 이 페이지 번호냐 SQL OFFSET 이냐

파라미터는 `page/size` 대신 `offset/limit` 으로 갔다. 같은 서비스 안에 이미 다른 추천 API가 `offset/limit` 을 쓰고 있어서, 인터페이스를 두 종류로 두고 싶지 않았다.

문제는 이 `offset` 이 이름 그대로 SQL `OFFSET` 절 값이 아니라는 것. 우리 계약에서는 1부터 시작하는 페이지 번호다. `offset=1&limit=30` 이면 첫 페이지. 여기서 헷갈리면 첫 페이지에서 0개가 리턴되거나, 정직하게 한 칸씩 밀린 결과가 나온다.

바깥 계약과 안쪽 SQL 사이를 한 줄로 갈랐다.

```python
skip = (offset - 1) * limit
```

이 한 줄이 실은 API 계약을 지탱하는 지점이다. 이름을 `offset` 으로 두기로 한 이상, 바깥에서 들어온 값은 절대 그대로 DB에 넘기지 않는다.

Pydantic 쿼리 모델은 기본값을 반드시 붙였다. 안 붙이면 FastAPI가 required 로 처리해서, 파라미터 없이 호출하는 예전 클라이언트가 422 로 튕긴다.

```python
from fastapi import Query
from pydantic import BaseModel

class PaginationParams(BaseModel):
    offset: int = Query(default=1, ge=1)
    limit: int = Query(default=30, ge=1, le=100)
```

`ge=1` 은 SQL OFFSET 계산에서 음수가 나오지 않게 하는 가드. `le=100` 은 클라이언트가 실수로 `limit=100000` 을 넣어서 DB를 쓸어가지 않게 하는 상한. 둘 다 지어낸 규칙이 아니라 실제로 한 번씩 겪었던 사고에서 나왔다.

## status: "success" 를 뺀다는 것

`status` 를 뺀다는 건 겉으로 보면 필드 하나 빼는 일인데, 실제로는 **성공/실패 판단 주체가 어디 있느냐** 를 옮기는 일이다.

예전 계약에서는 `response.data.status === "success"` 를 프론트가 봤다. HTTP는 200 이어도 바디에서 실패로 표현할 수 있는 여지가 있었다. 이건 사실 안 좋은 계약이었다. HTTP 코드와 바디가 서로 다른 얘기를 하는 순간, 미들웨어·게이트웨이·로깅이 다 어긋난다.

`status` 를 빼고 나면 판단은 HTTP 코드 하나로 몰린다. 2xx면 성공, 그 외는 실패. 그럼 실패일 때 바디 스키마도 다시 정해야 한다. 이렇게 통일했다.

```json
{
  "error_code": "TEMPLATE_QUERY_ERROR",
  "error_detail": "..."
}
```

FastAPI 기본 에러 바디는 `{"detail": "..."}` 인데 이걸 그대로 두면 프론트가 케이스별로 분기할 근거가 문자열뿐이다. 그래서 예외 핸들러를 하나 붙였다.

```python
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    detail = exc.detail if isinstance(exc.detail, dict) else {}
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error_code": detail.get("error_code"),
            "error_detail": detail.get("error_detail"),
        },
    )
```

`exc.detail` 이 문자열로 넘어오는 경우 (`raise HTTPException(404, "not found")` 같은) 를 아직 전부 정리하지 못했다. 저 `isinstance(..., dict)` 가드가 그 잔재다. 라우터를 다 뒤져서 dict 로 통일하는 게 맞는데, 아직 못 했다.

## 남아있는 것

- COUNT 쿼리 비용. 필터가 복잡한 목록에서 재봐야 하는데 아직 안 잰 상태다.
- `HTTPException(status, "문자열")` 로 던지는 라우터들. 새 계약이랑 안 맞으니 정리해야 한다.
- `offset` 이라는 이름 자체. 팀 안에서도 SQL OFFSET 으로 오해하는 사람이 계속 나온다. `page` 로 갈아엎을지, 문서에서 못 박고 넘어갈지 결정 못 했다.