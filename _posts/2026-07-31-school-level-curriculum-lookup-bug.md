---
title: "중1 2학기 6단원이 초1 단원으로 잡힌 이유: school_level 하나가 만든 추천 버그"
date: 2026-07-31 17:00:00 +0900
categories: [Backend]
tags: [debugging, data-modeling, composite-key, python, sql]
---

중1 2학기 6단원을 달라고 했는데, 반환된 건 초1짜리였다.

에러는 없었다. 쿼리는 성공적으로 실행됐고, 응답 JSON은 멀쩡했고, `title` 필드에도 뭔가 그럴듯한 단원명이 들어 있었다. 그 단원명이 내가 요청한 학교급의 것이 아니라는 것만 빼면.

## 처음엔 데이터를 의심했다

교육과정 메타데이터는 내가 넣은 게 아니라 큐레이션 팀이 관리한다. 그래서 처음엔 "누가 학교급을 잘못 태깅했나" 부터 봤다. `math_canvas_ai_templates` 테이블을 열어 문제의 행을 찾아 `school_level` 컬럼을 확인했더니, 데이터는 멀쩡했다. 초등은 `elementary`, 중등은 `middle`, 고등은 `high` 로 정확히 들어가 있었다.

데이터가 문제가 아니라면 룩업이 문제였다.

## 번호 체계가 학교급을 가로질러 겹친다

교육과정이라는 도메인의 특이점이 여기서 걸린다. `(grade=1, semester=2, unit_order=6)` 같은 조합은 초·중·고에 다 존재한다. 초1 2학기 6단원이 있고, 중1 2학기 6단원도 있고, 고1 2학기 6단원도 있다. 번호만 보면 셋은 구분되지 않는다.

즉 이 세 컬럼만으로 걸린 `WHERE` 절은 세 행을 후보로 남긴다. 그중 하나가 반환된다. `ORDER BY` 가 없으면 어느 것이 나올지는 옵티마이저 마음이다. 내 경우엔 초등이 먼저 왔다.

버그의 형태는 순식간에 명확해졌다. 문제는 "왜 룩업에 `school_level` 이 안 들어갔나" 였다.

## dataclass 는 이미 `school_level` 을 알고 있었다

repository 레이어의 dataclass 를 열어봤다. 여기까진 문제가 없어 보였다.

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
    school_level: str = ""  # 초등/중등/고등
```

`school_level` 은 마지막 줄에 있다. 필드도 있고 주석도 있다.

SELECT 쿼리도 봤다. 이 컬럼을 읽어오긴 한다.

```python
async def _fetch_templates_from_db() -> List[Template]:
    query = """
        SELECT
            project_id, project_title, tools, tool_names, categories,
            canvas_objects, activity_type, learning_goal, keywords, keywords_en,
            description, grade, semester, unit_code, school_level
        FROM math_canvas_ai_templates
        WHERE deleted_at IS NULL
        ORDER BY project_id
    """
    rows = await select(query)
    ...
    for row in rows:
        templates.append(Template(
            ...
            grade=row.get("grade"),
            semester=row.get("semester"),
            unit_code=row.get("unit_code") or "",
            school_level=row.get("school_level") or "",
        ))
```

전체 목록을 읽어올 때는 `school_level` 이 파이썬 객체까지 잘 흘러 들어온다. 그러면 룩업은 어디서 어그러진 걸까.

## 필터링이 애플리케이션 레이어에 있었다

이 저장소는 SQL 에서 `grade=? AND semester=? AND unit_order=?` 로 걸러 오는 구조가 아니었다. `_fetch_templates_from_db()` 로 전체를 캐시에 올려두고, 파이썬에서 필터링한다. 라우터를 보면 노출된 조회는 두 개다.

```python
@router.get("/mathcanvasai/templates-by-tool/{main_tool}")
async def get_templates_by_tool(main_tool: str, pagination: ...):
    templates, total = await get_templates_by_main_tool(
        main_tool, limit=pagination.limit, skip=pagination.skip
    )
    ...

@router.get("/mathcanvasai/random-templates")
async def get_random_templates_api(limit: ...):
    templates, total = await get_random_templates(limit=limit)
    ...
```

문제가 된 호출은 이 API 들이 아니라, 추천 파이프라인에서 캐시된 `Template` 리스트를 `grade`, `semester`, `unit_order` 로 필터링하던 내부 함수였다. `t.school_level == requested_level` 이 조건절에 빠져 있었다. dataclass 는 필드를 갖고 있었지만, 필터가 그 필드를 안 봤다.

## 왜 이렇게 됐을까

git blame 을 따라 올라가 보니 대략 짐작이 갔다. 이 필터는 초등 단일 학교급이던 시절에 쓰였다. 그 시점엔 `(grade, semester, unit_order)` 만으로도 충분한 룩업 키였다. 나중에 중등·고등이 추가되면서 스키마에 `school_level` 이 붙었고, SELECT 도 dataclass 도 따라갔다. 필터 함수만 안 따라갔다.

에러가 안 났던 이유도 여기서 나온다. 조건이 부족하면 후보가 여러 개 남을 뿐, 아무것도 안 나오는 게 아니다. 첫 번째 매칭이 그럴싸하게 반환되고, 그 그럴싸함이 오래 살아남았다.

## 수정은 짧았다

필터에 `school_level` 조건을 추가하고, 룩업 함수 시그니처에 학교급을 필수 인자로 올렸다. 파이썬 쪽만 손봤다. 스키마는 이미 컬럼을 갖고 있었고 데이터도 정확했으니 건드릴 게 없었다.

DB 쪽엔 `(school_level, grade, semester, unit_order)` 로 UNIQUE 제약을 걸어두는 걸 검토 중이다. 지금은 `project_id` 만 유니크라서, 언젠가 큐레이션 실수로 같은 학교급의 같은 번호가 두 개 들어와도 막지 못한다. 이건 이번 PR 에 안 넣었다 — 기존 데이터에 실제로 중복이 있는지 먼저 훑어야 하는데, 그건 별도 검증 작업이라 분리했다.

## 테스트로 못 박은 것

같은 grade/semester/unit_order 여도 school_level 이 다르면 다른 템플릿이 나와야 한다. 이 한 줄의 계약을 테스트로 남겨뒀다. 이번 필터 수정은 한 줄짜리라 다음 리팩터링에서 조용히 다시 사라지기 쉽다.

## 남은 것

정말 이 필터 하나뿐일까 하는 게 계속 걸린다. 초등 단일 학교급 시절에 짜여서 지금까지 살아있는 코드가 이 필터 말고도 있을 가능성이 높다. 추천 파이프라인 안쪽의 다른 후보 산출 로직, 통계용 집계 쿼리, 큐레이션 툴에서 쓰는 조회 함수 — `school_level` 을 안 보고 `grade, semester` 만 보는 곳을 훑어봐야 한다. 아직 안 했다.

에러 없이 통과하는 버그는 이런 식으로 오래 산다. 다음 주에 이걸 이어서 볼 예정이다.