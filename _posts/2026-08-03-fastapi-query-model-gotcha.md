---
title: "FastAPI 쿼리 모델이 조용히 안 펼쳐진 순간"
date: 2026-08-03 17:00:00 +0900
categories: [Backend]
tags: [fastapi, pydantic, debugging]
---

랜덤 추천 API에 페이지네이션을 붙이는 작업이었다. 기존에 `seed` 파라미터가 하나 있었고, 거기에 `offset`, `limit`을 얹기만 하면 되는 줄 알았다.

배포하고 `?seed=42&offset=1&limit=30`으로 호출했다. 응답은 잘 돌아왔다. 그런데 다음 페이지의 아이템이 첫 페이지와 겹쳤다.

## 에러가 없어서 더 오래 걸렸다

로그를 봤다. 파라미터 파싱 실패 같은 건 없었다. 422도 없었다. 그냥 정상 응답이 나가고 있었다.

핸들러 안에 `print(offset)`을 박아봤다. `0`이 찍혔다. 요청은 분명히 `offset=1`이었는데.

처음엔 프록시가 쿼리스트링을 잘라먹었나 의심했다. 아니었다. 그다음엔 Pydantic 모델의 기본값이 이상하게 캐싱되나 싶었다. 그것도 아니었다.

```python
class PaginationQuery(BaseModel):
    offset: int = 0
    limit: int = 30

@router.get("/recommendations/random")
async def get_random(
    seed: int = Query(...),
    pagination: PaginationQuery = Depends(),
):
    ...
```

이 코드가 "왠지 될 것 같은" 코드라는 게 함정이었다. Pydantic 모델을 `Depends()`로 넘기면 FastAPI가 필드를 개별 쿼리 파라미터로 펼쳐준다 — 이건 실제로 지원되는 동작이다. 그래서 예제도 많다.

## `/docs`가 답을 갖고 있었다

디버깅 중반쯤에 `/docs`를 열었다. OpenAPI 스키마가 이 API에 대해 인식한 파라미터는 단 하나였다.

```
- seed (integer, required)
```

`offset`도 `limit`도 없었다. FastAPI는 애초에 이 필드들을 쿼리 파라미터로 받고 있지 않았던 것이다. 요청에 아무리 실려도 파서에 도달하지 못했고, 그래서 Pydantic 모델은 매번 기본값으로 초기화되고 있었다. 에러가 안 났던 이유가 이거다 — **받지 못한 게 아니라 애초에 안 받고 있었다.**

`Query(...)`로 선언된 파라미터와 `Depends(BaseModel)`을 같은 시그니처에 섞으면 이런 상태가 만들어지는 것 같다. 어떤 조합에서는 잘 펼쳐지고, 어떤 조합에서는 조용히 무시된다. 재현 조건을 정확히 좁혀보진 못했다.

## 고친 방식

`seed`도 모델 안으로 넣었다.

```python
class RandomRecommendQuery(BaseModel):
    seed: int
    offset: int = 0
    limit: int = 30

@router.get("/recommendations/random")
async def get_random(
    query: RandomRecommendQuery = Depends(),
):
    ...
```

`/docs`를 다시 열었더니 세 필드가 모두 쿼리 파라미터로 잡혀 있었다. `offset=1`도 정상적으로 매핑됐다.

## 남은 것

이 버그를 만난 게 이번이 처음이 아니다. 이전에도 비슷하게 파라미터가 "그냥 없는 것처럼" 동작한 적이 있었는데, 그때는 다른 원인인 줄 알고 넘어갔다. 아마 같은 문제였을 것이다.

교훈이라고 부를 만한 건 없고, 습관 하나가 생겼다 — 새 엔드포인트를 열면 코드가 아니라 `/docs`를 먼저 본다. OpenAPI가 인식한 파라미터 목록이 FastAPI가 실제로 받는 파라미터 목록이다. 이 둘이 갈라진 순간을 코드만 봐서는 잡을 수 없다.

`Query`와 `Depends(BaseModel)`이 정확히 어떤 조건에서 충돌하는지는 아직 안 파봤다. 지금은 그냥 "같은 라우트에서 섞지 않는다"를 규칙으로 두고 있다.