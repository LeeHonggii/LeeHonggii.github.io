---
title: "랜덤 추천 API에 페이지네이션을 붙이면 왜 중복이 생길까"
date: 2026-08-19 17:00:00 +0900
categories: [Backend]
tags: [recommendation, pagination, sql, python, backend]
---

기획 측에서 요청이 왔다. 랜덤 추천 API에 `offset`과 `limit`을 붙여달라고.

처음엔 간단해 보였다. `LIMIT :limit OFFSET :skip` 두 줄이면 끝 아닌가. 근데 실제 동작을 떠올려보니 바로 멈췄다.

---

`ORDER BY RAND()` 쿼리는 실행할 때마다 전체 결과셋을 다시 뒤섞는다. 1페이지를 요청하면 한 번 섞고, 2페이지를 요청하면 또 한 번 섞는다. 같은 세션에서 연속으로 호출해도 정렬 순서가 달라지기 때문에, `OFFSET 30`으로 건너뛰려던 항목이 이미 1페이지에 들어가 있을 수 있다. 반대로 어떤 항목은 계속 뒤로 밀려서 아무 페이지에도 안 잡힌다.

중복과 누락이 동시에 생기는 구조다.

---

그런데 이 API가 실제로 `ORDER BY RAND()`를 쓰고 있는지 먼저 확인했다. 추천 엔진 코드를 열어보니, DB에서는 그냥 콘텐츠 풀을 통째로 올린다.

```python
_CONTENT_SQL = f"""
    SELECT ec.contents_agent_pk AS cid,
           cd.source_system AS content_uuid,
           em.content_pk    AS content_pk,
           COALESCE(NULLIF(em.title, ''), NULLIF(cd.title, ''), ec.contents_name) AS title,
           COALESCE({_NCS_EM}, {_NCS_CD}, {_NCS_EC}) AS ncs_code,
           ...
    FROM content_detail cd
    JOIN content ec ON ec.content_uuid = cd.source_system
    LEFT JOIN content_meta em ON em.source_system = cd.source_system
    WHERE ec.tenant_id = %s
      AND ec.contents_name IS NOT NULL
      AND ec.contents_name <> ''
      AND ec.contents_name NOT LIKE '%%placeholder%%'
"""

def load_company_pool(tenant_id: str, conn=None) -> dict[str, dict]:
    ...
    with conn.cursor() as cur:
        cur.execute(_CONTENT_SQL, (tenant_id,))
        rows = cur.fetchall()
    ...
    for r in rows:
        pool[r['cid']] = _content_row(r)
    logger.info('pool loaded: agent=%s size=%d (labeled=%d)', agent, len(pool), labeled)
    return pool
```

`ORDER BY` 자체가 없다. 랜덤성은 SQL 레벨에서 오는 게 아니라 Python 쪽 scoring 알고리즘에서 나온다.

---

추천 엔진을 더 들여다보면, "랜덤"처럼 보이는 동작의 실체는 MMR(Maximal Marginal Relevance)과 다양성 제어다.

```python
DIVERSITY_LAMBDA = {'job': 0.80, 'career': 0.65, 'company': 0.70, 'best': 0.70}
MMR_POOL_CAP = 150
BEST_FUSION_W = {'job': 0.40, 'career': 0.30, 'company': 0.20, 'next': 0.10}
```

관련도 점수와 다양성 패널티를 조합해 리스트를 구성한다. 풀 전체에서 score 순으로 뽑는 게 아니라, 이미 고른 항목과 너무 유사한 걸 억제하면서 한 개씩 추가하는 방식이다. 그래서 같은 입력이라도 풀 순서나 타이브레이크 처리에 따라 결과가 조금씩 달라 보인다. "랜덤"이 아니라 "다양하게"에 가깝다.

이 구조를 이해하고 나니 선택지가 세 개로 정리됐다.

---

**옵션 A — SQL 레벨에서 RAND() 고정**

`ORDER BY RAND(:seed)`로 seed를 박으면 같은 seed에서는 항상 같은 순서가 나온다. 클라이언트가 첫 요청에서 받은 seed를 이후 페이지 요청에도 함께 보내면 중복·누락 없이 페이지네이션이 된다.

문제는 이 시스템이 SQL에서 정렬하지 않는다는 점이다. Python에서 MMR로 순서를 결정하는 구조에 SQL seed를 끼워 넣으려면 알고리즘 전체를 바꿔야 한다. 그리고 클라이언트가 seed를 들고 다녀야 한다는 인터페이스 부담도 생긴다.

**옵션 B — 결과를 서버에 캐시**

첫 요청 때 전체 추천 리스트를 만들어 세션이나 캐시에 저장하고, 이후 페이지 요청은 그 리스트를 슬라이싱해서 반환한다.

구현은 직관적이다. 다만 캐시 무효화 타이밍, 사용자별 캐시 키 설계, 메모리 비용을 다 고려해야 한다. 요청이 많아지면 관리 포인트가 늘어난다.

**옵션 C — 랜덤 로직은 두고 포맷만 맞추기**

기획 요청을 다시 읽었다. 실제로 필요한 건 "무한 스크롤로 랜덤 콘텐츠를 계속 보여주는" 것이 아니라, "한 화면에 랜덤으로 몇 개 보여주는데 응답 형태가 페이지네이션 규격이어야 한다"는 것이었다.

응답 구조를 `{ totalCount, list }` 형태로 바꾸면 된다. 내부 로직은 그대로 두고, 포맷만 통일한다. 2페이지 이상 요청이 들어올 경우의 동작은 별도로 정의할 수 있다.

---

결론적으로 옵션 C 방향으로 정리했다. 알고리즘은 건드리지 않고, 응답 포맷만 표준 페이지네이션 구조에 맞췄다.

아직 열려 있는 부분은, 나중에 진짜로 "2페이지, 3페이지로 계속 넘어가며 랜덤 콘텐츠를 추가로 보여줘야 한다"는 요구가 붙으면 이 선택이 더 이상 유효하지 않다는 점이다. 그때는 seed 캐싱 전략을 다시 꺼내야 할 것 같다. MMR 알고리즘이 deterministic하게 동작하는 조건이 뭔지도 아직 제대로 확인하지 않았다.