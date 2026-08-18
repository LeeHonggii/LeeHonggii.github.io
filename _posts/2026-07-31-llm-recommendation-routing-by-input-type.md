---
title: "LLM 추천이 빗나갈 때, 단일 파이프라인을 A/B/C 경로로 갈라야 했던 이유"
date: 2026-07-31 17:00:00 +0900
categories: [AI]
tags: [llm, recommendation, pipeline, routing]
---

"3학년 2학기 3단원 도구 추천해줘." 그리고 "분수 나눗셈 개념 설명 도구 알려줘."

두 질의는 같은 파이프라인을 탔다. 같은 LLM 키워드 추출기를 거쳐, 같은 필터에 들어가, 같은 방식으로 후보를 뽑았다. 둘 다 이상한 결과를 냈다. **그런데 이상한 방식이 서로 달랐다.** 이걸 알아채는 데 시간이 좀 걸렸다.

## 처음엔 한쪽만 보였다

처음 로그를 열었을 때 눈에 띈 건 학년·차시 질의 쪽이었다. "3학년 2학기 3단원"을 LLM에게 넘기면 뭘 뽑아 오냐면, `["3학년", "2학기", "단원"]` 같은 걸 뽑아 왔다. 개념어를 뽑으라고 시켰는데 메타데이터를 뽑고 있었다. 프롬프트를 고쳤다. "학년·학기·단원 같은 메타 정보는 제외하고 순수 개념어만" — 그러자 이번엔 아무것도 못 뽑는 케이스가 늘었다. 애초에 그 문장에 개념어가 없으니까.

여기서 프롬프트를 더 만지작거리다가 멈췄다. 이건 프롬프트 문제가 아니다. **입력에 개념어가 없는데 개념어를 뽑으려는 것 자체가 잘못이다.**

## 반대편 실패

그러고 나서 키워드형 질의를 다시 봤다. "분수 나눗셈 개념 설명 도구" — 이건 LLM이 잘 뽑았다. `["분수 나눗셈", "개념 설명"]`. 문제는 그다음이었다. 후보 필터가 `grade`, `semester`, `unit_code` 같은 컬럼을 기준으로 좁혀 들어가는데, 이 질의엔 그 필드에 넣을 값이 없었다. 필터를 통과하는 템플릿이 0건이거나, 아니면 fallback으로 전체를 훑다가 관련 없는 것들이 위로 올라왔다.

이쪽은 추출은 됐는데 필터 정책이 안 맞았다. 저쪽은 추출 자체가 안 됐다. 실패의 층이 다르다.

같은 함수 안에서 둘을 동시에 고치려 하니까, 한쪽을 맞추면 다른 쪽이 깨졌다. 이 시점에서 파이프라인 자체를 갈라야 한다는 결론에 도달했다.

## 왜 갈라야 했나 — 데이터 스키마가 이미 갈라져 있었다

메타데이터 저장소를 다시 봤다. 템플릿 하나가 이런 모양이다.

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
    # 학년/차시 정보
    grade: Optional[int] = None
    semester: Optional[int] = None
    unit_code: str = ""
    school_level: str = ""
```

여기서 볼 것은 필드가 크게 두 부류로 나뉜다는 점이다. 하나는 `keywords`, `categories`, `learning_goal`, `description` — 자연어·개념 축. 다른 하나는 `grade`, `semester`, `unit_code`, `school_level` — 커리큘럼 축.

DB에서 후보를 가져오는 쿼리도 그 두 축을 다 담고 있다.

```python
async def _fetch_templates_from_db() -> List[Template]:
    query = """
        SELECT
            project_id, project_title, tools, tool_names, categories,
            canvas_objects, activity_type, learning_goal, keywords, keywords_en,
            description, grade, semester, unit_code, school_level
        FROM content_templates
        WHERE deleted_at IS NULL
        ORDER BY project_id
    """
    rows = await select(query)
    ...
```

데이터는 처음부터 두 축으로 인덱싱될 여지가 있었는데, 파이프라인이 그걸 하나로 뭉개고 있었다. 사용자 입력도 두 축 중 하나(또는 둘 다)를 가리키는 건데, LLM 추출기 하나로 그걸 통일하려 했다. 잘못된 통일이었다.

## 분기점을 파이프라인 앞으로

수정한 흐름은 이렇게 됐다.

```
validate
  → resolve_input_type    ← LLM 추출보다 먼저
  → (A) grade/semester/unit_code 채우기
     (B) extract_keywords
     (C) 둘 다
  → filter_templates (경로별)
  → recommend
```

`resolve_input_type`은 LLM을 부르지 않는다. 정규식으로 충분하다. `\d+학년`, `[1-2]학기`, `\d+단원` 이 매칭되면 구조화 필드를 채우고, 남은 토큰이 있으면 C, 없으면 A, 처음부터 아무 구조화 신호가 없으면 B. LLM을 파이프라인 첫 관문에 두지 않은 게 이 재설계에서 조용히 중요한 결정이었다. **분류는 결정적으로, 이해는 LLM으로.** 순서를 뒤집으니 나머지가 다 편해졌다.

A 경로에서는 LLM 키워드 추출을 아예 생략한다. 학년·학기·단원 세 값으로 `unit_code`를 특정해서 그 단원에 태깅된 템플릿만 가져오면 된다. 추출기의 오류가 끼어들 여지가 사라진다.

B 경로에서만 LLM이 개념어를 뽑는다. 이때 `keywords`, `categories`, `learning_goal`, `description` 을 대상으로 매칭한다.

C 경로는 A로 후보를 좁히고 B로 재랭킹한다.

## 라우터는 여전히 단순하다

라우터 쪽은 이 분기를 안다. 그 아래 계층에서 처리하기 때문이다.

```python
@router.get(
    "/content/templates-by-tool/{main_tool}",
    response_model=TemplatesByToolResponse,
    description="메인 도구 ID로 템플릿 목록 조회 (페이지네이션)",
    tags=["Template"]
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

라우터는 도구 ID로 후보를 가져올 뿐이다. 분기는 그 위 추천 서비스 계층이 진다. 라우터를 안 건드리고 파이프라인을 갈랐다는 게 이번 리팩토링에서 유일하게 마음에 든 부분이다. HTTP 표면적을 늘리지 않고 뒤쪽 로직만 나눴다.

## 추천 모델에 넘기는 컨텍스트도 갈라야 했다

파이프라인만 나누고 끝나면 절반만 고친 거였다. 후보를 좁힌 다음 "이 중에서 뭐가 제일 맞나"를 판단하는 추천 모델에는 여전히 원본 텍스트를 그대로 넘기고 있었다. 모델이 "3학년 2학기"라는 문자열을 보고 커리큘럼 맥락을 알 리가 없다.

그래서 넘길 때 이렇게 바꿨다.

```python
recommendation_context = {
    "school_level": "초등",
    "grade": 3,
    "semester": 2,
    "unit_code": "MATH_E3_2_03",
    "learning_concepts": ["분수 나눗셈"],
    "input_type": "A",
}
```

원문 한 줄보다 이 딕셔너리 하나가 낫다. `input_type` 을 넘기는 이유는 프롬프트가 그걸 보고 힘을 어디에 실을지 결정하기 때문이다. A면 단원 일치를 우선하고, B면 개념 유사도를 우선한다.

## 아직 안 한 것

측정은 아직이다. 라우팅을 바꾸면서 "얼마나 나아졌나"를 재보고 싶은데, 기준 데이터셋을 못 만들었다. 실사용 로그에서 "이건 이상한 결과였다"고 라벨링해줄 사람이 없다. 프롬프트만 잘 짜면 라벨링 없이도 정성 평가가 되지 않을까 잠깐 생각했다가 접었다 — 그건 이 글의 첫 실패와 같은 종류의 오만이다.

다음에 볼 건 C 경로의 재랭킹 가중치다. A로 좁힌 뒤 B 점수를 얹을 때, 둘의 스케일이 다르다. 지금은 대충 정규화해서 더하고 있는데, 이게 실제로 A 우선의 의도를 지키고 있는지 확신이 없다. 로그를 좀 더 봐야 알 것 같다.