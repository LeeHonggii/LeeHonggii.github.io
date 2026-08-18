---
title: "LLM 파이프라인 리뷰를 그냥 반영하지 않고, 깨지는 테스트부터 만든 이유"
date: 2026-07-31 17:00:00 +0900
categories: [Tooling]
tags: [code-review, defensive-coding, pytest, llm-pipeline]
---

리뷰에 "여기 방어 로직이 부족해 보인다"는 코멘트가 열 개쯤 달렸다. 처음엔 하나씩 훑으면서 `isinstance` 를 감으려 했다. 그러다 손을 멈췄다.

전에 비슷한 리뷰를 받고, 지적된 곳마다 가드를 넣다가 오히려 멀쩡히 돌던 경로 하나를 조용히 망가뜨린 적이 있다. 그때 배운 게 있다. **"버그가 있을 것 같은 코드"와 "실제로 버그가 있는 코드"는 다르다.** 앞엣것을 고치기 시작하면, 변경 diff는 커지는데 정작 뭐가 좋아졌는지 나도 모른다.

그래서 이번엔 순서를 바꿨다. 코드를 고치기 전에, **그 코드가 정말로 깨지는지부터 pytest 로 재현**하기로 했다.

## 리뷰가 짚은 두 갈래

문제가 몰려 있던 곳은 두 군데였다. 하나는 LLM 응답을 JSON 으로 파싱하는 경로, 다른 하나는 DB 컬럼에 JSON 문자열로 들어있는 배열을 읽어오는 경로.

DB 쪽은 실제 리포지토리 코드가 이렇게 생겼다. 템플릿 메타데이터를 캐시로 올리는 부분이다.

```python
async def _fetch_templates_from_db() -> List[Template]:
    query = """
        SELECT
            project_id, project_title, tools, tool_names, categories,
            canvas_objects, activity_type, learning_goal, keywords, keywords_en,
            description, grade, semester, unit_code, school_level
        FROM ai_templates
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

리뷰어가 짚은 지점은 `json.loads(row["tools"]) if row["tools"] else []` 형태의 다섯 줄이다. `row["tools"]` 가 `None` 이거나 빈 문자열이면 `[]` 로 떨어지는 건 처리돼 있다. 그런데 컬럼 안에 잘못된 값이 들어있으면 어떻게 되지? 예를 들어 배열이 아니라 객체가 들어있다면? 배열이긴 한데 원소에 문자열 아닌 게 섞여 있다면?

`Template` 데이터클래스는 이 값들이 `List[str]` 이라고 선언한다.

```python
@dataclass
class Template:
    project_id: str
    project_title: str
    tools: List[str] = field(default_factory=list)
    tool_names: List[str] = field(default_factory=list)
    categories: List[str] = field(default_factory=list)
    ...
    keywords: List[str] = field(default_factory=list)
```

타입 힌트일 뿐 강제는 안 된다. `json.loads` 는 문자열 하나가 들어와도 반환해 주고, 숫자만 있는 배열도 반환해 준다. 리뷰어의 걱정은 여기다. 이 값이 downstream 에서 `.lower()` 나 `in` 같은 문자열 연산에 그대로 들어가면 조용히 터진다는 것.

LLM 파싱 쪽도 유사한 결이다. 모델이 반환한 JSON 을 파싱해서 `recommendations` 배열을 훑는 코드가 있는데, 응답 스키마가 어긋나면 `data["recommendations"]` 접근에서 `TypeError` 로 죽는다. 바깥에 `try/except` 를 감아뒀지만, 그 안에서 죽어도 어차피 fallback 으로 빠지긴 한다. 문제는 왜 fallback 으로 빠졌는지 로그에 안 남는다는 것.

## 판단 기준 하나

리뷰 지적 항목을 다 고치면 diff 가 몇 백 줄이 된다. 그중 몇 개는 진짜로 필요할 거고, 몇 개는 이론적인 걱정에 가깝다. 나는 이걸 못 구분한다. 코드만 봐서는.

그래서 기준을 하나만 세웠다.

> **pytest 로 실제로 예외를 재현할 수 있는가.**

재현되면 실재하는 버그다. 안 되면 넘긴다 (또는 "왜 안 되는지" 를 알고 넘긴다).

DB 쪽부터 만졌다. `tools` 컬럼에 넣을 수 있는 이상한 값을 브레인스토밍했다:

- `"[]"` — 정상
- `"[\"line\",\"circle\"]"` — 정상
- `"[\"line\", null, 3]"` — 배열이지만 원소가 이상함
- `"{}"` — 배열이 아님
- `"null"` — `None` 이 파싱됨

이걸 fixture 로 만들어 `_fetch_templates_from_db` 를 호출하는 테스트를 짰다. `select` 를 mock 으로 대체해서 위 값들이 `row["tools"]` 로 들어오게 했다.

세 번째부터 다섯 번째까지가 전부 다르게 깨졌다. 어떤 건 `Template` 생성까지는 성공했다가 나중에 `keyword in template.tools` 같은 곳에서 조용히 안 맞는 결과를 냈다. 어떤 건 `json.loads` 는 통과했는데 `field(default_factory=list)` 로 선언된 필드에 dict 가 들어가 있었다.

이 시점에서 **테스트가 빨간불로 떨어졌다는 사실 자체가 리뷰어가 맞았다는 확인**이다. 그전까진 리뷰가 맞는지 나도 몰랐다.

## 고친 것과 안 고친 것

고친 건 딱 두 지점이다. 배열이 아닌 경우와 원소 타입이 다른 경우.

```python
def _safe_list_of_str(raw: str | None) -> List[str]:
    if not raw:
        return []
    try:
        value = json.loads(raw)
    except json.JSONDecodeError:
        return []
    if not isinstance(value, list):
        return []
    return [v for v in value if isinstance(v, str)]
```

그리고 `_fetch_templates_from_db` 에서 `json.loads(row["X"]) if row["X"] else []` 를 이 함수 호출로 대체했다. 다섯 줄이 다섯 줄로 바뀌었다. diff 규모는 원래와 거의 같다.

여기서 판단한 것 하나 — `keywords_en` 은 원소가 항상 문자열이어야 하지만, `canvas_objects` 같은 필드는 원래 정의 자체가 조금 자유롭다. 그럼에도 같은 헬퍼로 통일했다. 이유는 downstream 이 전부 `List[str]` 을 가정하고 있고, 그 가정이 깨지면 어차피 다른 곳에서 터진다. 그럴 바엔 진입점에서 잘라내는 게 낫다.

router 쪽은 손대지 않았다.

```python
@router.get(
    "/mathcanvasai/templates-by-tool/{main_tool}",
    response_model=TemplatesByToolResponse,
    ...
)
async def get_templates_by_tool(
    main_tool: str,
    pagination: Annotated[PaginationQuery, Query()],
):
    try:
        templates, total = await get_templates_by_main_tool(
            main_tool, limit=pagination.limit, skip=pagination.skip
        )
        return TemplatesByToolResponse(main_tool=main_tool, totalCount=total, list=templates)
    except Exception as e:
        logger.exception("[get_templates_by_tool] 도구 기준 템플릿 조회 실패")
        return JSONResponse(
            status_code=500,
            content={
                "error_code": "TEMPLATE_QUERY_ERROR",
                "error_detail": str(e),
            }
        )
```

리뷰어는 여기 `except Exception` 이 너무 넓다고 했다. 맞는 지적이긴 한데, 이걸 세분화하려면 하위 레이어가 어떤 예외를 던지는지부터 정의해야 한다. 지금은 던지지 않는다. 그걸 정의하는 건 별도 작업이라 이번 PR 에는 안 넣었다. 대신 리뷰 코멘트에 "여기는 하위 레이어 예외 계층을 먼저 정하고 손대는 게 맞다" 고 답을 달았다. 실제로 정할지는 아직 못 정했다.

## 남은 것

LLM 파싱 쪽은 이 글을 쓰는 시점에 재현 테스트까지 만들어놓고, 실 수정은 아직 안 올렸다. 이유는 프롬프트를 조금 조여서 이상 응답 빈도 자체를 낮추는 게 먼저인 것 같아서다. 근데 그렇게 판단한 근거가 아직 로그로 안 남아 있다. 프롬프트 조인 뒤에 실제 실패율이 어떻게 바뀌는지를 안 재봤다. 다음 주에 한 번 붙여 볼 생각.