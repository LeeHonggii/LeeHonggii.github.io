---
title: "테이블 rename 버그를 고치기 전에 DB로 먼저 증명하기"
date: 2026-08-18 17:00:00 +0900
categories: [Backend]
tags: [postgres, debugging, schema-migration, pydantic, pagination]
---

런타임에 터졌다. JOIN 쿼리가 테이블을 못 찾는다는 에러.

원인 후보는 처음부터 하나였다. 얼마 전에 태그 관련 테이블 이름을 바꿨는데, JOIN 어딘가가 옛 이름을 아직 물고 있을 가능성. 그런데 "가능성"으로 코드를 바로 손대지 않았다. 고쳤는데 다른 데서도 같은 이름을 참조하고 있었다, 는 상황이 싫었다. 틀린 확신으로 고치는 것도 버그고, 절반만 고치는 것도 버그다.

## 코드 말고 DB한테 먼저 물어본다

`check_schema.py` 라는 짧은 스크립트를 하나 만들었다. 세 단계.

1. 현재 스키마의 테이블 목록을 조회
2. 의심 가는 이름 두 개(옛 이름·새 이름)가 실제로 존재하는지 확인
3. 존재하는 이름으로 문제의 JOIN SELECT를 직접 쏴본다

돌린 결과는 이랬다.

```
[테이블 존재 여부]
  content_project_tags  → 없음
  content_tags          → 있음

[쿼리 실행]
  SELECT ... FROM content_tags ct
  LEFT JOIN ...
  → 실행 성공 ✓  행 5개, NULL 없음
```

이걸로 두 가지가 동시에 확정됐다. 옛 이름은 실제로 사라진 상태고, 새 이름으로 쓴 JOIN은 실제로 돈다. 원인은 코드 어딘가에 옛 이름이 남아있는 것 하나뿐이다. LEFT JOIN 결과에 NULL이 하나도 없었다는 것도 봐뒀다 — 매칭이 실제로 이루어졌다는 뜻이니, 조인 조건 자체가 틀린 것도 아니다.

수정은 한 줄이었다. 그런데 그 한 줄을 고치기 전에 이 확인이 있었기 때문에, 고친 뒤에 "이거 진짜 맞나?" 를 하지 않아도 됐다. 개발 환경에 옮겨서 develop 브랜치에 반영.

여기서 배운 건 "스크립트를 미리 만들어두자" 같은 게 아니다. **가설을 코드 수정으로 검증하지 말고 DB로 검증하자**, 쪽에 가깝다. 코드 수정은 되돌리는 데 리뷰가 필요하지만, SELECT 한 방은 되돌릴 것도 없다.

---

같은 날 다른 작업도 이어졌다. 이건 버그가 아니라 응답 형태를 통일하는 일.

## 두 API가 서로 다른 모양으로 목록을 던지고 있었다

템플릿 조회 API가 두 개다. 특정 도구 기준 조회, 랜덤 조회. 둘 다 "목록"을 돌려주는데, 한쪽은 리스트를 바로 던지고 한쪽은 래퍼를 씌워서 던졌다. 프론트가 케이스를 나눠야 하는 구조.

`totalCount`, `list` 두 필드를 기본으로 갖는 Generic 페이지네이션 래퍼로 통일했다.

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

Generic으로 뺀 이유는 뻔하다. 다른 목록 응답에도 같은 래퍼를 쓸 것이기 때문. `TemplatesByToolResponse` 는 여기에 `main_tool` 한 필드를 얹었다. 어떤 도구 기준으로 조회한 결과인지를 응답 자체가 들고 있게 하면, 클라이언트가 자기 요청 파라미터를 다시 뒤질 필요가 없다.

`RandomTemplatesResponse` 는 지금 `pass` 다. 필드 구성은 부모와 같지만 **의미가 다르다** — 랜덤에는 `main_tool` 이 없다. 지금 당장 얹을 필드가 없다고 해서 그냥 `PaginationResponse[Dict[str, Any]]` 를 반환 타입으로 쓰지 않은 건, 나중에 랜덤 전용 필드(seed, refreshed_at 같은 것)가 붙을 자리를 미리 열어둔 것에 가깝다. 지금 비어있는 게 이상해 보여도, 여기가 확장점이라는 표시를 코드로 남겨둔다.

## DB에서 템플릿을 꺼내는 쪽

목록 API의 아래층은 이미 단일 테이블 조회로 정리돼 있었다.

```python
async def _fetch_templates_from_db() -> List[Template]:
    query = """
        SELECT
            project_id, project_title, tools, tool_names, categories,
            canvas_objects, activity_type, learning_goal, keywords, keywords_en,
            description, grade, semester, unit_code, school_level
        FROM content_ai_templates
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

여기서 눈에 걸리는 건 `row["field"]` 와 `row.get("field")` 가 섞여 있다는 점이다. 규칙 없이 섞은 게 아니다. **초기 스키마에 있던 컬럼은 대괄호로, 나중에 ALTER 로 추가된 컬럼은 `.get()` 으로** 접근하고 있다. `keywords_en`, `grade`, `semester`, `unit_code`, `school_level` 이 후자다.

왜 이러냐면, 이 코드가 도는 환경이 하나가 아니어서다. 마이그레이션이 아직 안 밀린 환경이 있으면 대괄호는 KeyError로 죽는다. `.get()` 은 기본값으로 넘어간다. 이걸 통일하려면 마이그레이션 상태를 강제하거나, 전부 `.get()` 으로 밀어야 하는데 — 전부 `.get()` 으로 밀면 "이 컬럼은 반드시 있어야 한다" 는 계약이 코드에서 사라진다. `[]` 접근이 남아있는 컬럼들은 "이건 없으면 우리가 죽어야 맞다" 를 코드로 선언하고 있는 셈이다.

JSON 컬럼 언팩도 `json.loads(row["x"]) if row["x"] else []` 를 반복한다. 컬럼이 NULL인 케이스와 빈 문자열인 케이스를 한 번에 걸러내려는 것. 여기에 헬퍼를 하나 만들까 잠시 생각했다가, 만들면 호출부에서 "이 헬퍼가 대체 뭘 걸러주지?" 를 다시 읽어야 한다. 지금 형태는 눈으로 바로 읽힌다. 헬퍼 도입은 보류.

캐시는 위에 `_templates_cache` 로 있다. 10분 TTL. 요청마다 이 SELECT 를 태우기엔 목록이 무거워서다. TTL을 10분으로 잡은 근거는 정확히 재본 게 아니라 감이다. 이 부분은 아직 실측 안 했다.

---

## 아직 안 풀린 것

랜덤 조회 쪽. 랜덤 정렬과 페이지네이션은 원래 궁합이 나쁘다. 요청마다 순서가 바뀌면 2페이지가 1페이지와 겹치거나 항목이 빠진다.

방법은 대충 두 가지다. 서버가 날짜 기반으로 seed 를 고정해서 그날 하루는 같은 순서를 유지하거나, 클라이언트가 seed 를 들고 다니면서 페이지 요청마다 같이 보내거나. 후자가 정직하지만 클라이언트 설계가 같이 바뀌어야 하고, 전자는 "어제 마지막에 본 항목이 오늘 첫 페이지에 다시 뜨는" 문제가 생긴다.

일단 응답 껍데기(`totalCount/list`) 만 맞춰뒀고, 페이지 간 연속성은 보장하지 않는 채로 둔 상태다. `RandomTemplatesResponse` 를 굳이 별도 타입으로 분리해둔 것도 여기에 seed 필드가 붙을 자리를 열어두려는 이유가 크다. 이 논의는 아직 프론트와 하지 않았다.