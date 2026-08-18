---
title: "추천 관점이 3개인데 왜 결과가 똑같이 보일까?"
date: 2026-07-06 17:00:00 +0900
categories: [AI]
tags: [추천시스템, RRF, 다양성]
---

## 화면을 켜놓고 한참 멍했다

캐러셀 세 줄을 붙였다. 직업 축, 관심 축, 선호 축. 각자 다른 관점에서 콘텐츠를 골라주는 그림이었다. API 응답 스키마도 그렇게 갈랐다 — `axes.job`, `axes.career`, `axes.company`, 그리고 상위를 다시 융합한 `axes.best`.

```python
_RECO_EXAMPLE = {
    "axes": {
        "job": [_ITEM_EXAMPLE],
        "career": [_ITEM_EXAMPLE],
        "company": [_ITEM_EXAMPLE],
        "best": [_BEST_EXAMPLE],
    },
    ...
}
```

문제는 화면이었다. 제목만 훑으면 세 줄이 한 줄처럼 보였다. "축이 다른데 왜 결과가 같지." — 며칠 이 질문만 봤다.

## 먼저 감이 아니라 숫자로

눈으로만 보면 얼버무리게 된다. Top-K 겹침을 직접 세는 게 먼저였다.

```python
def overlap_at_k(axis_a, axis_b, k=10):
    ids_a = {r["content_id"] for r in axis_a[:k]}
    ids_b = {r["content_id"] for r in axis_b[:k]}
    return len(ids_a & ids_b)
```

특정 테넌트로 돌려보니 job과 career의 Top-10 중 상당수가 겹쳤다. 정확한 비율은 테넌트마다 달랐고, 지금 시점에 "몇 대 몇으로 줄었다"고 못 박을 만큼 재현 가능하게 재보지는 못했다. 다만 눈으로 볼 때 "거의 같다"고 느낄 만한 정도였다는 것만 남긴다.

## 왜 수렴하는가

세 축은 다른 질문을 한다. 직업 축은 프로필의 직무 코드에서 출발하고, 관심 축은 최근 학습 이력의 패턴을 따라가고, 선호 축은 회사(테넌트) 단위 인기·전이 신호를 본다. 질문은 다르다.

그런데 답을 만들 때 참조하는 콘텐츠 메타데이터는 하나의 풀에서 나온다. `keyword_tags`, `ncs_code`, `title` — 이 셋이 한 콘텐츠의 정보량을 정하는 축이다. 코드에도 그대로 박혀 있다.

```python
def _info_score(c: dict) -> int:
    return (
        (2 if c.get('keyword_tags') else 0)
        + (1 if c.get('ncs_code') else 0)
        + (1 if c.get('title') else 0)
    )
```

같은 콘텐츠 UUID가 여러 개 있을 때 "가장 정보량 많은 놈"으로 좁히는 로직인데, 이 함수가 말하는 건 결국 우리 시스템에서 콘텐츠를 구별짓는 신호가 저 셋이라는 뜻이다. 그리고 NCS 코드와 키워드는 실제로 강하게 얽혀 있다. NCS가 "데이터 분석"이면 키워드에도 데이터·분석·통계가 몰린다. 이력을 봐도 같은 사용자가 같은 계열을 연달아 본다.

세 축의 *질문*은 다르지만, 답을 구할 때 밟는 *신호*가 상관되어 있으면 결과는 수렴한다. 가중치는 신호의 강약을 조절할 뿐, 신호 자체를 갈아끼우지 못한다. 이걸 며칠 만에야 인정했다.

## 신호 자체가 얽혀 있다는 걸 본 순간

전이(transition) 계산 코드를 다시 봤다.

```python
_TRANSITION_WINDOW = 10
_PEER_MIN_SUPPORT = 5

@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _transition_raw(tenant_id: str) -> dict:
    pool = _company_pool(tenant_id)
    by_user: dict[str, list[str]] = {}
    for uid, raw_cid, _ in _history_rows(tenant_id):
        if raw_cid in pool:
            by_user.setdefault(uid, []).append(raw_cid)

    outgoing: dict[str, int] = {}
    pair_users: dict[tuple[str, str], set[str]] = {}
    for uid, seq in by_user.items():
        if len(seq) < 2:
            continue
        for i, src in enumerate(seq[:-1]):
            outgoing[src] = outgoing.get(src, 0) + 1
            for dst in seq[i + 1:i + 1 + _TRANSITION_WINDOW]:
                if dst == src:
                    continue
                pair_users.setdefault((src, dst), set()).add(uid)
    ...
```

한 사용자의 최근 10개 이력 안에서 src→dst 쌍을 모으고, 서로 다른 사용자 support가 5명 이상일 때만 살린다. 관심 축(career)이 이 전이 표에 크게 의지한다. 그런데 이 표에 들어가는 시퀀스는 **직업 코드로 걸러진 풀 안에서 벌어진 소비 이력**이다. 다시 말해 직업 축이 참조하는 원천 풀이 관심 축의 전이 신호를 이미 좁혀놓고 있었다는 얘기다.

축이 세 개인 게 아니라, 축 하나에 색을 세 번 입힌 꼴이었다.

## 그럼 어디를 손대야 하나

가중치 슬라이더는 놔뒀다. 대신 세 지점에 걸었다.

첫째, 시드. 관심 축이 참조하는 이력 창을 직업 축보다 좁혔다. 최근 쪽에 더 붙게. 코드 상 `_TRANSITION_WINDOW = 10`은 pair 생성 범위고, 축별 시드는 이 위에서 다시 자를 수 있다. 이 조정은 지금도 튜닝 중이라, "얼마가 좋다"고 확정해서 말하지 못한다.

둘째, `axes.best`를 만들 때의 융합 규칙을 다시 봤다. 스키마에 이렇게 남아 있다.

```python
_BEST_EXAMPLE = {
    ...,
    "best_score": 1.0,
    "rrf_score": 1.0,
    "axis_ranks": {"job": 1, "career": 1, "company": 1, "next": 1},
    "reasons": ["종합 상위"],
}
```

`axis_ranks`를 응답에 실어둔 게 다행이었다. best에 오른 항목이 어느 축에서 몇 등이었는지 자체가 진단 신호가 된다. "세 축에서 다 1등"인 항목이 자꾸 상위에 뜨면 그건 축이 갈라지지 않았다는 증거였다.

셋째, 조립 단계. 앞 캐러셀에 이미 나온 콘텐츠는 뒤 캐러셀 후보에서 뺐다. 이건 축을 갈라주는 처방이 아니라, 이미 겹친 결과를 화면에서라도 덜 보이게 하는 임시 방편이다. 근본은 여전히 신호 상관에 있다.

## 아직 못 푼 것

`_info_score`가 말하는 세 신호(`keyword_tags`, `ncs_code`, `title`) 자체를 축별로 다르게 쓰는 방법을 아직 못 찾았다. 예를 들어 관심 축은 키워드·전이에만, 직업 축은 NCS 코드에만 의존하도록 신호를 강제로 갈라두면 축은 확실히 분리될 것이다. 대신 각 축의 품질이 떨어질 위험이 있다 — 신호 하나가 부족하거나 없는 콘텐츠가 상당수라, 축 하나를 통째로 좁히면 후보군이 얇아진다.

`diagnostics.fallback_axes`가 응답에 있는 것도 그래서다. 어떤 축이 후보 부족으로 다른 축의 결과로 대체됐는지를 추적하려고 넣어뒀는데, 신호를 갈라놓기 시작하면 이 fallback이 얼마나 자주 도는지부터 다시 봐야 한다.

지금은 여기까지 왔고, 다음에는 축별로 참조 신호를 실제로 분리했을 때 fallback 비율이 어떻게 움직이는지를 재볼 생각이다. 재보기 전에는 아무 말도 하지 않기로 했다.