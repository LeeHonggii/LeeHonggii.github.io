---
title: "FastAPI에서 Pydantic 쿼리 모델이 조용히 안 펼쳐진 순간"
date: 2026-08-25 17:00:00 +0900
categories: [Backend]
tags: [fastapi, pydantic, openapi, debugging]
---

FastAPI로 템플릿 조회 API를 수정하다가 이상한 걸 발견했다. 요청은 200을 반환했고, 에러는 없었다. 그런데 Swagger 문서를 열었더니 파라미터 모양이 내가 정의한 것과 달랐다.

---

작업은 단순해 보였다. 랜덤 템플릿 추천 엔드포인트에 페이지네이션을 붙이는 것. 기존에 `offset`과 `limit`을 담은 Pydantic 쿼리 모델이 이미 있었고, 랜덤 정렬 재현을 위해 `seed` 값을 별도로 받아야 했다.

자연스럽게 이렇게 쓰고 싶었다.

```python
# (구조 예시 — 실제 코드는 생략)
async def get_random_templates(
    pagination: PaginationQuery,  # offset, limit 담은 Pydantic 모델
    seed: int = Query(default=...),
):
    ...
```

FastAPI의 쿼리 파라미터 모델은 `Depends()`로 주입하면 자동으로 펼쳐진다. 모델 안에 `offset: int`, `limit: int`가 있으면 OpenAPI 문서에도 그 두 개가 독립된 쿼리 파라미터로 나온다. 그게 FastAPI의 약속이다.

근데 `seed`를 `Query()`로 따로 선언하자 그 약속이 조용히 깨졌다.

---

증상은 두 가지였다.

하나, Swagger에서 `offset`과 `limit`이 펼쳐진 개별 파라미터로 안 보였다. 모델 자체가 하나의 덩어리처럼 다뤄지는 것 같았다.

둘, 그래도 요청 자체는 됐다. `?offset=0&limit=30&seed=42`를 보내면 동작했다. 에러도 없었다. 그래서 바로 못 잡았다.

OpenAPI 스키마를 직접 `/openapi.json`에서 확인하고 나서야 파라미터 정의가 내가 의도한 것과 다르다는 걸 알았다. 문서 불일치가 생기면 API 소비자 입장에서는 뭘 믿어야 할지 모른다. 자동 생성 클라이언트나 타입 검증 도구가 있으면 실제로 깨진다.

---

원인을 추적해 보니 FastAPI가 쿼리 파라미터 모델을 펼치는 방식과 관련이 있었다. FastAPI는 `Depends()`로 주입된 Pydantic 모델을 파라미터로 인식할 때 내부적으로 특정 방식으로 처리한다. 그런데 같은 함수 시그니처에 별도의 `Query()` 파라미터가 섞이면, 모델 펼치기(flattening) 동작이 예상대로 작동하지 않는 경우가 있다. FastAPI 버전이나 Pydantic 버전에 따라 동작이 달라지기도 한다는 점도 문제다. 재현이 환경에 따라 미묘하게 다를 수 있다.

해결은 간단했다. `seed`를 별도 `Query()` 파라미터로 선언하는 대신 Pydantic 쿼리 모델 안으로 옮겼다.

```python
# (구조 예시 — 실제 코드는 생략)
class RandomTemplateQuery(BaseModel):
    offset: int = 0
    limit: int = 30
    seed: int  # 여기로 이동
```

이렇게 하면 모델 하나로 통합되고, FastAPI가 펼치기를 일관되게 처리한다. OpenAPI 문서도 의도한 대로 나온다.

---

랜덤 정렬에 seed를 쓰는 것 자체는 또 다른 얘기다.

`seed` 기반 정렬은 "같은 seed면 같은 순서"를 보장하는 방식이다. 페이지를 넘길 때 중복이나 누락 없이 안정적으로 탐색할 수 있다. 그런데 실제로 DB 레벨에서 seed 기반 랜덤 정렬을 지원하는 방식은 DB마다 다르고, 애플리케이션 레벨에서 처리하려면 오버헤드가 생긴다.

이번에는 "출력 형식만 페이지네이션 구조로 맞추자"는 쪽으로 합의했다. 실제로 seed 보장 없이 매 요청마다 다른 순서가 나와도 UI 요구사항상 괜찮다는 판단이었다. seed를 요청 모델 안에 넣기는 했지만, 현재는 그 값을 정렬에 실제로 활용하는 게 아니라 API 계약 차원에서만 존재한다.

이 결정이 맞는지는 아직 확신이 없다. 나중에 클라이언트에서 "왜 페이지 넘겨도 같은 게 나와요?"라고 물어보면 그때 다시 봐야 할 것 같다.