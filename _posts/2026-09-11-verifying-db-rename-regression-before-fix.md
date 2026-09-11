---
title: "테이블 rename 하나가 API를 깨뜨릴 때: 고치기 전에 DB로 먼저 확인하기"
date: 2026-09-11 17:00:00 +0900
categories: [Backend]
tags: [mysql, sqlalchemy, debugging, schema-migration]
---

API가 갑자기 죽었다. 에러 메시지는 "Table 'db.old_public_tags' doesn't exist". 테이블을 rename 한 지 며칠 된 시점이었다.

rename 자체는 마이그레이션 스크립트로 깔끔하게 끝냈다. 문제는 JOIN 구문이었다. 어딘가 옛 이름을 그대로 쓰는 쿼리가 살아 있었다.

---

이 프로젝트는 SQLAlchemy의 ORM 모델 대신 `text()`로 raw SQL을 직접 실행한다.

```python
async def select(
        query: str,
        params: Optional[Dict[str, Any]] = None
) -> list[dict[Any, Any]] | None:
    async with get_session() as session:
        try:
            stmt = text(query)
            if params:
                result = await session.execute(stmt, params)
            else:
                result = await session.execute(stmt)
            # ...
```

ORM 모델을 쓰면 테이블명 불일치는 앱 기동 시점에 잡힌다. `text()`를 쓰면 실행 시점에 터진다. 정적 분석도, 린터도 통과한다. 해당 엔드포인트가 호출되기 전까지는 아무도 모른다.

---

바로 코드를 고치지 않았다. 먼저 DB를 확인했다.

`SHOW TABLES`로 현재 스키마에 어떤 테이블이 있는지 확인했다. 예상대로 옛 이름은 없었고 rename된 이름만 있었다. 이걸 확인한 이유는 단순하다 — "rename 완료"라고 생각했지만 실제로는 롤백됐거나, 다른 환경에서 마이그레이션이 누락됐을 가능성을 배제하기 위해서다. 에러 메시지가 명확해 보여도 DB 상태를 직접 보기 전에는 원인을 확정짓지 않는다.

그 다음은 실제 쿼리를 DB에서 직접 실행해봤다. 문제의 JOIN을 포함한 SELECT를 날리면 어떤 에러가 떨어지는지. 당연히 에러가 떴고, 정확히 같은 메시지였다. 원인은 특정 쿼리의 JOIN 절에 옛 테이블명이 박혀 있는 것.

```sql
LEFT JOIN old_public_tags opt ON c.id = opt.content_id
```

이걸 새 이름으로 교체하면 끝이다. 그런데 바꾸기 전에 한 가지 더 확인했다. LEFT JOIN이니까, 새 테이블로 바꿔도 JOIN 결과에서 null이 섞이지 않는지. 태그가 없는 콘텐츠가 있다면 null이 정상이고, 그게 아닌 경우 데이터 정합성 문제일 수 있다. 쿼리를 새 테이블명으로 바꿔서 실행해보니 예상한 대로 null 행이 섞여서 나왔다 — LEFT JOIN 의도가 맞았다.

---

수정은 한 줄이었다.

```python
query = """
    SELECT c.id, c.title, c.type
    FROM content c
    LEFT JOIN public_tags pt ON c.id = pt.content_id
    WHERE c.status = 'active'
"""
```

커밋 메시지: `fix: update renamed public tag table join`

코드 diff는 작다. 하지만 이 한 줄을 고치기 위해 확인한 것들 — 테이블 존재 여부, 쿼리 실행 에러, LEFT JOIN 결과의 null 분포 — 이것들이 없으면 "고친 것 같은데 사실 다른 이유였다"는 상황이 생긴다.

검증 스크립트(`scripts/check_schema.py`)는 이 과정을 반복하기 위해 만들었다. 테이블 목록 조회, 특정 쿼리 실행, 결과 샘플 출력을 한 번에 돌릴 수 있게. rename이 다음에 또 생길 때 같은 확인을 처음부터 손으로 하지 않기 위해서다.

---

아직 자동화하지 못한 부분이 있다. 코드베이스 안에 raw SQL 쿼리가 몇 개나 있는지, 그 중 특정 테이블명을 참조하는 게 몇 개인지를 rename 전에 자동으로 뽑아주는 도구. grep으로 하면 되긴 하지만, `text()` 안에 문자열이 변수로 조립되는 경우는 놓친다. 그 케이스까지 잡으려면 런타임 트레이싱이나 쿼리 로깅 분석이 필요한데, 아직 거기까지 손을 대지 않았다.