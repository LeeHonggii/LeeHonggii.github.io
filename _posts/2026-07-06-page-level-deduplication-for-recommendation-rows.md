---
title: "넷플릭스식으로 추천 줄 사이 중복을 없애는 방법"
date: 2026-07-06 17:00:00 +0900
categories: [Backend]
tags: [recommendation, carousel, dedup, ranking]
---

추천 시스템을 배포하고 며칠 뒤 들어온 첫 피드백은 점수 얘기가 아니었다.

> "왜 같은 강의가 세 줄에 다 나와요?"

직무 기반 줄, 관심사 기반 줄, 선호도 기반 줄. 세 줄이 각자 다른 논리로 계산해서 1위로 꺼낸 콘텐츠가 하필 같은 인기 강의였다. 스코어링 자체는 잘못된 게 없었다. 화면을 조립하는 사람이 없었을 뿐이다.

## 각 축은 옳은데 화면은 틀렸다

우리 추천 API는 결과를 `axes` 라는 딕셔너리로 돌려준다. 대략 이런 모양이다.

```python
_RECO_EXAMPLE = {
    "tenant_id": "TENANT_A",
    "user_id": "user-001",
    "mode": "preeval+history",
    "axes": {
        "job":     [_ITEM_EXAMPLE],   # 직무 적합도
        "career":  [_ITEM_EXAMPLE],   # 이력 기반
        "company": [_ITEM_EXAMPLE],   # 회사 선호
        "best":    [_BEST_EXAMPLE],   # 종합 상위 (RRF)
    },
    "diagnostics": {
        "preeval_applied": True,
        "history_input": 5,
        "history_matched": 5,
        "fallback_axes": [],
        "warnings": [],
    },
}
```

이 4개의 축은 서로 다른 신호로 top-N을 뽑는다. `job` 은 사전평가로 들어온 직무·NCS 코드에 콘텐츠 메타를 붙여 점수를 매기고, `career` 는 이력 시퀀스에서 뽑은 다음 콘텐츠, `company` 는 테넌트 내 인기 신호를 본다. 각 축이 저마다 정직하게 계산하면, 그 축들이 모두 좋다고 말하는 콘텐츠가 반드시 생긴다. 인기 콘텐츠는 어느 렌즈로 봐도 잘 보인다. 스코어링이 망가진 게 아니라, 스코어링이 잘 되고 있다는 증거였다.

문제는 "그 결과를 세로로 쌓아 사용자에게 보여주는 책임이 어느 함수에도 없었다"는 것이었다. 각 축은 독립적으로 자기 top-N을 반환한다. 페이지는 그걸 그대로 렌더링한다. 그 사이에 아무도 없었다.

## 그래서 페이지 조립을 하나의 레이어로 떼어냈다

스코어링을 건드리지 않기로 했다. 이유는 두 가지다. 하나, 스코어 계산은 A/B로 검증한 뒤에 흔들고 싶었다. 둘, 지금 나타나는 건 스코어 문제로 볼 수 없다 — "축별로는 옳은 결과가 화면 조립 단계에서 겹친다"는 건 조립기의 결함이다. 결함은 결함이 있는 곳에서 고쳐야 한다.

조립기의 첫 버전은 진짜 단순했다. 축을 순환하며 `seen` 집합에 없는 첫 항목을 뽑는 라운드로빈이다.

```python
def assemble_rows(axes: dict[str, list], per_row: int = 8) -> dict[str, list]:
    seen: set[str] = set()
    rows: dict[str, list] = {name: [] for name in axes}

    # 축을 순환하며 슬롯을 채운다
    while any(len(rows[n]) < per_row for n in axes):
        progressed = False
        for name, candidates in axes.items():
            if len(rows[name]) >= per_row:
                continue
            picked = _pick_first_unseen(candidates, seen)
            if picked is not None:
                rows[name].append(picked)
                seen.add(picked["content_id"])
                progressed = True
        if not progressed:
            break   # 후보 고갈 — 남은 슬롯은 빈 채로 둔다
    return rows


def _pick_first_unseen(candidates: list, seen: set[str]):
    for item in candidates:
        if item["content_id"] not in seen:
            return item
    return None
```

`seen` 이 전역처럼 흐르면서 이미 다른 줄에 배정된 콘텐츠는 건너뛴다. 각 축 내부 순위는 그대로 보존된다. 축이 원래 1위로 꺼낸 항목이 이미 다른 줄에 잡혀 있으면, 그 축은 2위를 자기 1위 자리에 올린다. 각 축 안에서의 상대적 순서는 어긋나지 않는다.

라운드로빈은 공정 배분에 가깝다. 신호가 유독 강한 축이 있으면 좀 손해를 볼 수 있다. 지금 우리 트래픽 규모에서 그 손해가 얼마나 되는지는 아직 재보지 않았다.

## Best 줄은 예외로 뒀다

그런데 이 규칙을 4개 축에 다 적용하면 이상해진다. `best` 는 성격이 다른 줄이다. 넷플릭스로 치면 "Top 10 in Korea" 같은 자리. 다른 줄과 겹치더라도 사용자가 "이건 지금 이 자리에서 봐야 한다"고 인식하는 구역이다.

`best` 는 다른 축들의 순위를 RRF(Reciprocal Rank Fusion — 여러 랭킹을 역순위로 더해 하나의 종합 순위로 만드는 방법)로 합쳐서 만든다. 그러니까 정의상 다른 축과 겹치는 게 정상이다. 여기서 dedup을 걸면 "종합 1위"라는 개념 자체가 무너진다.

응답 스키마도 그걸 반영해서 아이템 모양이 다르다.

```python
_BEST_EXAMPLE = {
    # ... 공통 필드
    "best_score": 1.0,
    "rrf_score": 1.0,
    "axis_ranks": {"job": 1, "career": 1, "company": 1, "next": 1},
    "reasons": ["종합 상위"],
}
```

`axis_ranks` 가 붙는다. "이 콘텐츠는 job에서 1위, career에서도 1위였다"를 그대로 노출한다. 이 필드가 있으니 프론트에서 "종합 Best 줄에는 왜 다른 줄과 같은 게 나오나요"라는 질문에 답할 수 있다. 그 자체가 이유다.

그래서 조립기는 두 층으로 나뉜다.

```python
def assemble_page(axes: dict[str, list], per_row: int = 8) -> dict[str, list]:
    best = axes.get("best", [])[:per_row]

    # best는 dedup 대상에서 제외
    other = {k: v for k, v in axes.items() if k != "best"}
    rows = assemble_rows(other, per_row=per_row)

    rows["best"] = best
    return rows
```

`best` 는 dedup 대상 축 목록에서 아예 빼고, 나머지 축들끼리만 `seen` 을 공유한다. "당연히 봐야 할 것"과 "다양한 관점의 발견"을 서로 다른 계층으로 인정하는 셈이다.

## 왜 여기까지 오는 게 오래 걸렸는가

돌아보면 초반에 헤맨 지점은 "이걸 스코어링 안에서 풀 수 있지 않을까"였다. 각 축의 점수 함수에 페널티를 걸면, 다른 축이 이미 높게 매긴 콘텐츠는 이 축에서 살짝 낮게 나오지 않을까. 프로토타입을 만들다가 접었다. 이유가 두 가지 있었다.

축끼리 점수를 서로 참조하기 시작하면, 각 축이 "이 축은 무엇을 보는가"라는 단순한 정의를 잃는다. `job` 함수 안에서 `career` 의 최근 결과를 조회해야 하고, 그러면 `career` 는 `job` 이 확정된 뒤에만 돌아야 한다. 순서 의존성이 생기고, 캐시 무효화가 지옥이 된다.

두 번째는 검증 문제였다. 스코어에 페널티를 섞으면 "이 콘텐츠가 왜 3위인가"에 답하기 어려워진다. 지금은 각 축의 점수는 그 축의 신호만으로 설명되고, 화면 조립 단계에서 밀렸으면 밀렸다고 말할 수 있다. 면접에서 이 시스템을 설명할 때도 그게 편했다.

## 남은 것

지금 조립기는 라운드로빈 한 가지만 안다. 축별 신호 강도에 가중을 두는 "비교우위" 방식(각 축의 현재 최선 후보의 점수를 힙에 넣고 최고 점수 축부터 슬롯을 채우는 방식)을 다음 후보로 두고 있는데, 이건 아직 안 넣었다. 라운드로빈이 실제로 어떤 축에 얼마나 손해를 주는지 로그를 좀 더 쌓고 나서 결정하려 한다.

한 가지 더 애매하게 남은 게 있다. 지금 `seen` 은 `content_id` 기준인데, 사실 우리 풀에는 같은 원본 콘텐츠가 여러 테넌트에 각기 다른 `content_id` 로 존재한다 (`_origin_map`, `_uuid_map` 이 그걸 정규화한다). 사용자 눈에는 같은 강의인데 조립기 눈에는 다른 항목이다. 지금은 이걸 무시하고 있다. 이걸 origin 기준으로 dedup 하면 화면은 더 깔끔해지지만, "왜 이 콘텐츠는 회사 A 버전으로 나오고 저 콘텐츠는 회사 B 버전으로 나오는가"라는 새로운 선택 문제가 생긴다. 이건 아직 답이 없어서 열어뒀다.