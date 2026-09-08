---
title: "테이블명 하나 바꾸기 전에 DB에서 먼저 확인한 이유"
date: 2026-09-08 17:00:00 +0900
categories: [Backend]
tags: [MySQL, Python, Database, Debugging, SQLAlchemy]
---

테이블 rename은 이미 끝난 상태였다. `project_tags`가 `public_tags`로 바뀐 지 꽤 됐고, 마이그레이션도 완료됐다고 했다. 그런데 어느 날 조회 쿼리가 이상했다. LEFT JOIN에 구 테이블명이 그대로 남아 있었다.

고치면 됐다. 두 글자 바꾸면 끝이다. 근데 그냥 바꾸질 못했다.

---

런타임에서 죽는 에러라면 차라리 명확하다. 테이블이 없으면 즉시 예외가 터지고, 스택 트레이스가 범인을 가리킨다. 문제는 `LEFT JOIN`이다. 참조 테이블이 없거나 JOIN 대상이 잘못돼도, 경우에 따라 NULL 행으로 조용히 통과한다. 기능이 깨지는 게 아니라 데이터가 빠지는 형태로 나타난다. 이런 버그는 로그에 흔적을 거의 안 남긴다.

그래서 수정 전에 확인을 먼저 했다.

스크립트(`scripts/check_schema.py`)를 돌렸다. 목적은 두 가지였다.

1. 실제 DB에 `public_tags`가 존재하는지 확인한다
2. 수정한 JOIN 쿼리를 직접 실행해서 NULL이 섞이지 않는지 확인한다

첫 번째는 간단했다. `INFORMATION_SCHEMA.TABLES`에서 테이블명 조회. 존재했다.

두 번째가 더 중요했다. JOIN 결과에 NULL이 남으면 ON 조건이 틀렸거나 데이터 정합성이 깨진 것이다. 실제로 실행해보고 NULL 행 수를 세면 조인 의도까지 한 번에 검증할 수 있다.

확인 후 쿼리를 수정하고, `git cherry-pick`으로 develop에 반영했다.

---

이 코드베이스에서 DB 쿼리는 SQLAlchemy의 비동기 엔진을 통해 날 SQL 문자열로 실행된다. 아래는 실제 템플릿 조회 함수의 일부다.

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
            # ...
        ))

    return templates
```

ORM 없이 raw query를 직접 쓴다. 장점은 쿼리를 그대로 복사해서 DB 클라이언트에 붙여넣을 수 있다는 것이다. 검증 스크립트를 쓸 때 이 점이 편했다. 수정한 쿼리를 스크립트에 그대로 옮겨서 실행할 수 있었다.

DB 연결 쪽도 구조가 단순하다. 로컬에서는 SSH 터널을 열고, 클러스터 안에서는 직결한다.

```python
def init_db_engine():
    global engine, async_session_factory

    if os.getenv('MATHCANVAS_SSH_HOST'):
        if os.getenv('KUBERNETES_SERVICE_HOST'):
            print("SSH tunnel skipped: running in cluster")
        else:
            start_ssh_tunnel()

    DB_URL = get_db_url()
    engine = create_async_engine(
        DB_URL,
        pool_size=POOL_SIZE,
        max_overflow=10,
        pool_recycle=3600,
        pool_pre_ping=False,
    )
```

`pool_pre_ping=False`인 게 눈에 띈다. 연결 유효성을 쿼리 전에 검사하지 않는다는 의미다. 재연결 비용보다 ping 오버헤드를 피하는 쪽을 택했는데, 장기 유휴 커넥션이 끊기면 첫 쿼리가 실패할 수 있다. 지금은 별 문제가 없지만, 운영 중에 커넥션 드랍이 생기면 이 부분을 먼저 볼 것 같다.

---

이번 수정 자체는 단순했다. 테이블명 두 글자. 하지만 고치기 전에 스크립트로 확인하는 단계가 생겼다는 게 작은 변화였다. 이 확인이 없었다면 수정이 맞는지는 배포 후에야 알 수 있었다.

아직 확인하지 못한 게 하나 있다. 동일 패턴의 구 테이블명이 다른 쿼리에 남아 있는지 전수 조회를 아직 안 했다. 테이블 rename이 이번 건뿐이었는지도 코드베이스 수준에서 grep을 한 번 돌려봐야 한다.