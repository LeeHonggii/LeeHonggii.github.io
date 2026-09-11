---
title: "NestJS PaginationDto를 FastAPI API 계약으로 옮기기"
date: 2026-09-11 17:00:00 +0900
categories: [Backend]
tags: [fastapi, pagination, api-contract, python]
---

프론트엔드와 백엔드가 페이지네이션 계약을 처음 맞출 때 가장 흔하게 생기는 문제는 숫자 해석이 다른 것이다.

---

## offset을 어떻게 읽을 것인가

NestJS 쪽 `PaginationDto`는 이렇게 생겼다.

```typescript
offset?: number  // default 1
limit?: number   // default 30, max 300
```

이름이 `offset`이지만 값이 `1`에서 시작한다. SQL의 `OFFSET`이 익숙한 사람이라면 처음에 헷갈린다. 행 오프셋이 아니라 **1-based 페이지 번호**다.

FastAPI로 옮기면 이렇게 된다.

```python
from fastapi import Query

async def list_items(
    offset: int = Query(default=1, ge=1),
    limit: int = Query(default=30, ge=1, le=300),
):
    skip = (offset - 1) * limit
    # skip을 실제 SQL OFFSET으로 사용
```

`skip = (offset - 1) * limit` — 이 한 줄이 계약의 핵심이다. `ge=1`로 0이나 음수가 들어오는 경우를 FastAPI 레이어에서 막는다. 별도 검증 로직 없이 Pydantic 유효성 검사가 처리한다.

---

## 응답 구조를 바꿨다

기존 응답은 `count`와 `templates`를 키로 썼다.

```json
{
  "count": 276,
  "templates": [...]
}
```

이걸 아래로 바꿨다.

```json
{
  "totalCount": 276,
  "list": [...]
}
```

`count`는 애매하다. 현재 페이지에 담긴 아이템 수인지, 전체 개수인지 문서 없이는 알 수 없다. `totalCount`는 **페이지네이션 이전의 전체 개수**라는 의미가 이름에 담긴다. 프론트에서 전체 페이지 수를 계산할 때 `Math.ceil(totalCount / limit)`를 쓸 수 있어야 하므로 이 구분이 중요하다.

`templates` → `list`로 바꾼 건 단순하다. 엔드포인트마다 다른 리소스 이름을 키로 쓰면 프론트에서 응답을 일반화하기 어렵다. `list`로 통일하면 페이지네이션 훅 하나로 여러 API를 다룰 수 있다.

---

## status 필드 제거

원래 응답에 `status: "success"` 같은 필드가 있었다. 이걸 제거하고 HTTP 상태코드로만 성공/실패를 판단하게 바꿨다.

REST에서 성공·실패 판단은 HTTP 상태코드가 이미 하고 있다. `200 OK`인데 바디에 `"status": "error"`를 넣는 패턴은 클라이언트가 두 곳을 동시에 확인해야 한다는 뜻이다. 어느 쪽을 믿어야 하는지 모호해지고, 에러 처리 분기가 늘어난다.

FastAPI에서 에러는 `HTTPException`으로 올린다. `raise HTTPException(status_code=404, detail="not found")` 하면 응답 바디에 `status` 필드 없이도 클라이언트는 상태코드만 보면 된다.

---

## 랜덤 추천 API는 예외로 남겼다

페이지네이션을 붙이면서 한 가지 논의가 있었다. 랜덤 추천 API에도 같은 페이지네이션을 적용하면 **중복과 누락이 생긴다**. 매 요청마다 순서가 달라지니 2페이지에서 1페이지 아이템이 다시 나올 수 있다.

결론은 랜덤 로직은 건드리지 않고 **응답 포맷만 맞추는 것**이었다. `totalCount`와 `list` 구조는 동일하게 쓰되, 실제로 offset/limit 기반 커서가 작동하지 않는다는 걸 내부적으로는 알고 있다. 이게 완전히 깔끔한 해법은 아니다. 프론트가 무한 스크롤을 구현하려 한다면 중복 아이템을 클라이언트에서 걸러야 한다. 아직 그 부분은 합의가 덜 됐다.