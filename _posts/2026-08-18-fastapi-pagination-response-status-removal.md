---
title: "200 + status 필드를 버리고 HTTP 상태코드로 API 실패를 표현하기"
date: 2026-08-18 17:00:00 +0900
categories: [Backend]
tags: [api-design, pydantic, fastapi, http, response-contract]
---

응답 body에 `"status": "success"` 를 심어두는 관례를 오래 끌고 왔다. 클라이언트가 보는 모양은 대충 이렇다.

```json
{
  "status": "success",
  "count": 3,
  "templates": [...]
}
```

실패는 이렇게.

```json
{
  "status": "error",
  "message": "query failed"
}
```

HTTP 상태코드는 200. body를 파싱해서 `status` 필드를 읽어봐야 이게 성공인지 실패인지 안다. 왜 이게 나쁜지는 이미 여러 번 들었을 거다. 브라우저 devtools에서 빨간 줄이 안 뜬다. 프록시가 정상 응답으로 캐싱한다. 재시도 미들웨어가 실패를 실패로 인식 못 한다. 알면서도 계속 썼던 이유는 단순하다. 뜯어낼 타이밍이 애매했다. 이번에 템플릿 조회 API 계약을 정리하다가 그 타이밍이 왔다.

## count 는 뭘 세는 숫자였나

`status` 를 걷어내기로 하니 붙어 있던 다른 것도 눈에 걸렸다. `count / templates` 라는 이름.

`count` 가 지금 응답에 담긴 개수인지, 조건에 맞는 전체 개수인지 이름만 봐서는 모른다. 페이지네이션이 없을 땐 어차피 같은 값이라 상관없다. 문제는 페이지네이션이 붙는 순간이다. 1페이지에서 `count: 10` 을 돌려줬을 때 클라이언트가 이걸 "전체 10개"로 해석하면 2페이지를 아예 요청하지 않는다. 이름 하나 때문에 데이터가 누락되는 버그가 조용히 심어진다.

`totalCount / list` 로 바꿨다. `totalCount` 는 페이지네이션 이전 전체 개수, `list` 는 지금 페이지에 담긴 항목. 이름이 역할을 말하니까 문서를 안 봐도 오해할 여지가 줄었다.

## 성공 응답과 실패 응답을 분리

Pydantic 모델은 이렇게 정리했다.

```python
from pydantic import BaseModel, Field
from typing import Any, Dict, Generic, List, Optional, TypeVar

T = TypeVar("T")


class BaseResponse(BaseModel):
    error_code: str | None = Field(default=None, examples=["error code"], description="code")
    error_detail: str | None = Field(default=None, examples=["error desc"], description="desc")


class PaginationResponse(BaseModel, Generic[T]):
    totalCount: int = Field(..., description="총 항목 수")
    list: List[T] = Field(default_factory=list, description="결과 목록")


class TemplatesByToolResponse(PaginationResponse[Dict[str, Any]]):
    main_tool: str = Field(..., description="조회한 메인 도구 ID")


class RandomTemplatesResponse(PaginationResponse[Dict[str, Any]]):
    pass
```

`BaseResponse` 는 성공 응답에서 아예 쓰지 않는다. 실패할 때 HTTP 4xx/5xx 를 던지면서 body에 `error_code` 와 `error_detail` 만 담는 용도다. 성공 응답은 `PaginationResponse` 를 상속받는 별개 모델로 정의하고, 거기엔 `status` 같은 필드가 존재하지 않는다.

두 계열을 섞지 않은 게 요점이다. 예전엔 성공/실패 모두 하나의 응답 모델을 공유하면서 `status` 필드로 분기했는데, 그러면 성공 응답의 스키마에 실패 필드가, 실패 응답의 스키마에 성공 필드가 옵셔널로 늘어붙는다. OpenAPI 문서를 열면 어떤 필드가 어떤 상황에 오는지 알아볼 수 없다. 이제는 `TemplatesByToolResponse` 를 열면 성공 시 오는 것만 보이고, 실패는 상태코드로 갈라진다.

`TemplatesByToolResponse` 에 `main_tool` 필드를 하나 더 붙인 이유는 실용적이다. 클라이언트가 여러 도구를 병렬로 조회할 때 응답만 보고 어느 도구의 결과인지 구분할 수단이 필요했다. Generic 으로 만들어 둔 `PaginationResponse` 를 상속해서 필드 하나만 추가하는 방식이 제일 깔끔했다.

응답이 실제로 나가는 모양은 이렇다.

```json
{
    "totalCount": 7,
    "list": [...],
    "main_tool": "SM02PB"
}
```

실패는 이렇게.

```json
{
    "error_code": "TEMPLATE_QUERY_ERROR",
    "error_detail": "DB 연결 실패: ..."
}
```

HTTP 상태코드가 5xx 이면 body는 두 번째 모양, 2xx 이면 첫 번째 모양. 클라이언트는 `response.ok` 만 보면 된다.

## RandomTemplatesResponse 가 비어 있는 이유

위에서 `RandomTemplatesResponse` 는 `pass` 하나로 끝났다. 필드 추가가 없다. 그런데 굳이 별도 클래스로 뺀 건 이유가 있다. 랜덤 조회는 나중에 페이지네이션 전략이 달라질 가능성이 커서 타입을 미리 갈라두고 싶었다.

랜덤 정렬과 `LIMIT/OFFSET` 은 같이 쓰면 깨진다. 매 요청마다 서버가 전체를 다시 섞으니까 2페이지를 요청하면 1페이지와 중복이 생기거나 일부가 빠진다. 해결책으로 seed 고정을 검토했다. 세션마다 seed를 만들어 붙이고, `ORDER BY RAND(seed)` 로 순서를 고정하면 페이지 간 정합성이 유지된다.

이번엔 seed 를 도입하지 않았다. 랜덤 조회의 현재 사용 방식이 "한 번에 다 받아서 클라이언트에서 소진" 이라 OFFSET 이 실질적으로 0이었기 때문이다. 응답 구조만 `totalCount / list` 로 통일해 두면 클라이언트가 다른 목록 API와 동일한 코드로 처리할 수 있다는 것으로 충분했다. 진짜 페이지네이션이 필요해지는 순간에 seed 전략을 얹기로 하고, 그때 응답 필드가 하나 늘어날 수 있으니 클래스만 미리 갈라둔 것이다.

## 캐시는 응답 계약과 별개로 살아 있다

DB 레이어는 응답 계약 변경과 무관하게 그대로 두었다. 템플릿 목록을 10분 TTL 로 캐싱하는 코드다.

```python
_templates_cache: Optional[List[Template]] = None
_cache_loaded_at: float = 0
CACHE_TTL_SECONDS = 600  # 10분


def _is_cache_expired() -> bool:
    return (time.time() - _cache_loaded_at) > CACHE_TTL_SECONDS


async def _fetch_templates_from_db() -> List[Template]:
    query = """
        SELECT
            project_id, project_title, tools, tool_names, categories,
            canvas_objects, activity_type, learning_goal, keywords, keywords_en,
            description, grade, semester, unit_code, school_level
        FROM content_meta_templates
        WHERE deleted_at IS NULL
        ORDER BY project_id
    """

    rows = await select(query)
    if not rows:
        return []

    templates = []
    for row in rows:
        templates.append(Template(
            project_id=row["project_id"],
            project_title=row["project_title"],
            tools=json.loads(row["tools"]) if row["tools"] else [],
            tool_names=json.loads(row["tool_names"]) if row["tool_names"] else [],
            categories=json.loads(row["categories"]) if row["categories"] else [],
            canvas_objects=json.loads(row["canvas_objects"]) if row["canvas_objects"] else [],
            activity_type=row["activity_type"] or "",
            learning_goal=row["learning_goal"] or "",
            keywords=json.loads(row["keywords"]) if row["keywords"] else [],
            keywords_en=json.loads(row["keywords_en"]) if row.get("keywords_en") else [],
            description=row["description"] or "",
            grade=row.get("grade"),
            semester=row.get("semester"),
            unit_code=row.get("unit_code") or "",
            school_level=row.get("school_level") or "",
        ))

    return templates
```

`tools`, `keywords`, `categories`, `canvas_objects` 같은 컬럼이 전부 JSON 문자열로 저장돼 있어서 꺼낼 때마다 `json.loads` 를 돌린다. 캐시 없이 매 요청마다 이걸 반복하면 손해가 컸다. 조회 패턴이 읽기 중심이고 갱신 빈도는 낮아서 10분 정도의 오차는 감수할 수 있었다. 실측으로 캐시 히트/미스 비율을 재본 건 아니라 이 부분은 아직 숫자로 말할 수 없다.

캐시 무효화가 필요한 시점(관리자 화면에서 템플릿을 편집한 직후 등)엔 강제 리로드 훅이 따로 필요하다. 지금은 10분을 그냥 기다리는 구조라, 이 부분은 계약 정리와 별개로 다음에 손봐야 한다.

## 아직 안 끝난 것

`status` 필드를 없애는 변경 자체는 서버 쪽 몇 줄이지만, 이걸 소비하는 클라이언트 코드에는 파장이 넓다. `if response.status === 'success'` 로 분기하던 프론트, 재시도 로직에서 body 파싱을 전제로 짜여 있던 SDK, 로그 파이프라인에서 성공/실패를 body 로 집계하던 대시보드. 각각이 다른 팀의 스케줄 위에 있다.

이걸 어느 시점에 자르고, 이전 응답 형식과 얼마나 병행할지는 아직 정하지 않았다. 두 벌을 동시에 내려주는 트랜지션 기간을 두는 게 안전하지만, 그러려면 응답 모델을 한 번 더 감싸는 레이어가 필요해서 이번 정리의 취지를 흐린다. 여기까지 오면 기술 판단이 아니라 협의의 문제라 잠깐 멈춰뒀다.