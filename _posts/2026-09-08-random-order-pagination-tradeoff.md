---
title: "랜덤 추천 API에 페이지네이션을 붙이면 왜 중복이 생길까"
date: 2026-09-08 17:00:00 +0900
categories: [Backend]
tags: [pagination, sql, api-design, random-order]
---

클라이언트에서 요청이 들어왔다. 템플릿 추천을 랜덤으로 보여주는데, 응답 형식을 `totalCount / list` 기반 페이지네이션 구조로 맞춰달라고. 처음엔 간단해 보였다. `ORDER BY RAND() LIMIT :limit OFFSET :offset` 붙이면 되는 거 아닌가? 그런데 잠깐 생각해보니 뭔가 이상했다.

---

현재 안정적인 목록 조회 쿼리는 이렇게 생겼다 (식별자는 일반화했다).

```sql
SELECT
    project_id, project_title, tools, tool_names, categories,
    canvas_objects, activity_type, learning_goal, keywords, keywords_en,
    description, grade, semester, unit_code, school_level
FROM canvas_templates
WHERE deleted_at IS NULL
ORDER BY project_id
```

`ORDER BY project_id` — 고정된 기준으로 정렬하기 때문에 page 1을 두 번 불러도 결과가 같다. 페이지네이션이 이 위에서 올바르게 동작하는 이유다.

여기에 랜덤을 얹으면?

```sql
SELECT ...
FROM canvas_templates
WHERE deleted_at IS NULL
ORDER BY RAND()
LIMIT 30 OFFSET 0;

-- 1초 후 두 번째 요청
SELECT ...
FROM canvas_templates
WHERE deleted_at IS NULL
ORDER BY RAND()
LIMIT 30 OFFSET 30;
```

두 쿼리는 각각 실행 시점에 독립적으로 전체 행을 다시 섞는다. 첫 번째 요청에서 1번 행이 3번 자리에 있었다가, 두 번째 요청에선 31번 자리에 있을 수 있다. `OFFSET 30`이 "이전 30개 이후"를 보장하지 않는다. 중복이 생기는 이유가 이거다. 누락도 생긴다.

---

해결책을 두 가지로 놓고 생각했다.

**seed 고정 방식.** `ORDER BY RAND(:seed)`를 쓰면 같은 seed값에서 항상 동일한 순서가 나온다. 클라이언트가 세션 시작 시 seed를 받아 이후 요청마다 그 값을 들고 다니면, 한 사용자가 보는 순서는 일관되게 유지된다.

```sql
SELECT ...
FROM canvas_templates
WHERE deleted_at IS NULL
ORDER BY RAND(:seed)
LIMIT :limit OFFSET :offset;
```

논리적으로는 맞다. 그런데 API 계약이 바뀐다. 클라이언트가 seed를 관리해야 하고, seed가 없거나 다르면 다시 뒤섞인다. "랜덤처럼 보이지만 사실은 고정"이라는 사실을 클라이언트도 알아야 한다.

**랜덤은 그냥 두는 방식.** 이쪽이 최종 결정이었다. 랜덤 추천은 본질적으로 "지금 이 순간 섞인 결과"다. 사용자가 페이지를 넘기며 끝까지 훑는 시나리오가 아니라, 한 번 호출해서 충분한 수의 결과를 받아가는 시나리오라면, 굳이 페이지 간 연속성을 보장할 필요가 없다.

응답 형식만 규격에 맞추면 된다.

```json
{
  "totalCount": 276,
  "list": [
    { "project_id": "...", "project_title": "...", ... },
    ...
  ]
}
```

`totalCount`는 전체 템플릿 수, `list`는 이번에 랜덤으로 뽑은 N개. 클라이언트는 페이지네이션 응답 포맷을 그대로 파싱할 수 있고, "offset=2 이상에서 왜 중복이 나오냐"는 문제 자체가 사라진다. 랜덤 추천 엔드포인트에 `offset` 파라미터를 받더라도, 동작의 의미가 "연속된 페이지"가 아니라 "매번 새로 섞인 그 지점"임을 클라이언트와 합의하면 된다.

---

결국 "페이지네이션이 필요하다"는 요청을 받으면, 먼저 되물어야 한다. "사용자가 2페이지, 3페이지를 넘기는 시나리오가 실제로 있나?" 없다면 응답 형식을 페이지네이션 구조로 맞추되, 정렬 안정성까지 보장할 이유는 없다. 있다면 그때는 seed 방식이든, 아니면 캐싱된 순서를 서버가 들고 있는 방식이든, 별도 설계가 필요하다.

이번엔 전자였다. 아직 "사용자가 랜덤 추천 목록을 여러 페이지 걸쳐 탐색하는" 케이스가 실제로 나올지는 지켜봐야 한다.