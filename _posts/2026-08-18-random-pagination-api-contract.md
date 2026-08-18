---
title: "랜덤 추천 API에 페이지네이션을 붙이면 왜 깨질까?"
date: 2026-08-18 17:00:00 +0900
categories: [Backend]
tags: [pagination, random-sort, api-design, mysql, pydantic]
---

도구별 템플릿 조회는 이미 `{ totalCount, list }` 형태로 응답하고 있었다. 랜덤 추천 엔드포인트 하나만 아직 형식이 달랐다. "같은 껍데기로 맞추면 되겠네." 여기서부터 꼬였다.

## 응답 모델은 진짜 5분이면 끝난다

Pydantic Generic으로 페이지네이션 껍데기를 하나만 선언해두면, 안에 담기는 타입만 갈아끼우면 된다.

```python
from pydantic import BaseModel, Field
from typing import Any, Dict, Generic, List, TypeVar

T = TypeVar("T")


class PaginationResponse(BaseModel, Generic[T]):
    totalCount: int = Field(..., description="총 항목 수")
    list: List[T] = Field(default_factory=list, description="결과 목록")


class TemplatesByToolResponse(PaginationResponse[Dict[str, Any]]):
    main_tool: str = Field(..., description="조회한 메인 도구 ID")


class RandomTemplatesResponse(PaginationResponse[Dict[str, Any]]):
    pass
```

도구별 조회는 `main_tool`이라는 컨텍스트가 하나 더 붙어서 서브클래스에서 필드를 얹었고, 랜덤 쪽은 붙일 게 없어 `pass`만 남겼다. `RandomTemplatesResponse`가 실제로 하는 일은 "이건 랜덤 응답이다"라는 타입 이름을 확정하는 것뿐이다. 이렇게 두면 OpenAPI 스키마에서 두 엔드포인트가 같은 껍데기라는 게 그대로 드러난다.

문제는 여기가 아니었다.

## `ORDER BY RAND()`에 `OFFSET`을 붙였을 때 무슨 일이 일어나는가

랜덤 목록의 SQL은 대략 이런 모양이다.

```sql
SELECT project_id, project_title, tools, tool_names, categories, ...
FROM content_meta
WHERE deleted_at IS NULL
ORDER BY RAND()
LIMIT :limit OFFSET :skip
```

클라이언트는 `?offset=0&limit=30`으로 첫 페이지를 받고, 다음에 `?offset=30&limit=30`을 요청한다. 바인딩은 문제없다. 쿼리 자체도 실행된다. 그런데 두 번째 요청이 받는 30개가 첫 번째 요청의 31~60번째 행이라는 보장이 **하나도 없다.**

`RAND()`는 매 실행마다 새 시드로 돈다. 즉 두 요청 사이에서 DB는 전혀 다른 순서로 다시 섞은 다음, 그 새 순서의 30번째부터 자른다. 결과적으로 이미 본 템플릿이 두 번째 페이지에 다시 등장하기도 하고, 어떤 항목은 두 페이지 어디에도 안 나오기도 한다.

이건 구현 실수가 아니라 조합 자체가 깨졌다. "랜덤 정렬"과 "오프셋 기반 페이징"은 서로 다른 전제 위에 서 있다 — 오프셋은 *정렬이 고정되어 있다는 가정* 아래에서만 의미가 있다.

## seed를 넘기면 풀린다, 근데 그 값을 누가 들고 다니지

MySQL의 `RAND()`는 인자로 시드를 받는다. `ORDER BY RAND(:seed)`처럼 넣으면 같은 시드에서는 같은 순열이 나온다. 흐름을 짜면 이렇게 된다.

- 첫 요청 때 서버가 seed를 하나 만들어 응답에 실어 보낸다
- 클라이언트는 다음 페이지 요청에 그 seed를 그대로 붙여서 다시 보낸다
- DB는 같은 seed로 같은 순서를 재현하고, `OFFSET`이 비로소 뜻대로 동작한다

깔끔하다. 하지만 이 방식은 클라이언트가 **세션 상태를 하나 더 들고 다녀야 한다**는 뜻이다. 새로고침하면 seed가 날아가고, 화면 이동 뒤에 돌아왔을 때 다시 첫 seed로 복원할 건지, 아예 새로 뽑을 건지 결정이 필요하다. 캐시 키에도 seed가 들어가야 한다.

프론트와 이야기해봤다. 답은 "굳이"였다. 랜덤 추천은 다음 페이지에서 새로운 게 나와도 이상하지 않은 화면이었다. 오히려 매번 다르게 보이는 편이 UX 의도에 가까웠다. 그 대가로 seed 관리 코드를 양쪽에 얹는 건 남는 장사가 아니었다.

## 그래서 남긴 결정

응답 형태만 `PaginationResponse` 계열로 통일하고, 랜덤 정렬은 그대로 뒀다. `offset >= 1`에서 중복/누락이 생길 수 있다는 사실은 **알고 놔둔** 상태다. 지금의 UX가 그걸 허용하기 때문이다.

`totalCount`를 굳이 실어 보내는 이유는 하나다. 전체 건수는 랜덤 여부와 상관없이 고정 값이고, 클라이언트가 "총 몇 개 중에 보고 있는지"를 표시하고 싶을 수 있다. 순서는 흔들려도 그 숫자는 믿을 수 있다.

## 아직 재보지 않은 것

지금 이 엔드포인트에 `offset > 0`으로 들어오는 요청이 실제로 있는지 로그를 뜯어보지 않았다. 클라이언트가 관성적으로 `offset=0`만 쓰고 있다면 위에서 말한 중복/누락 문제는 **아직 발화하지 않은** 상태고, 페이지를 넘기기 시작하는 순간 첫 UX 이슈로 올라올 것이다. 미리 막고 싶다면 seed 방식이든, `project_id` 기반 정렬 고정에 랜덤성을 시드 해시로 얹는 방식이든, 지금이 바꾸기 좋은 시점이다.

재보고 결정한다.