---
title: "LLM 응답은 믿지 말자: 추천 파서와 DB 값 방어 코드 테스트부터 고치기"
date: 2026-07-28 17:00:00 +0900
categories: [Backend]
tags: [LLM, 방어코드, pytest, TDD, 파서]
---

리뷰에서 짧은 한 줄을 받았다.

> "LLM이 `dict`가 아닌 걸 반환하면 어떻게 돼요?"

읽자마자 손이 IDE로 갔다. 파서를 여는 순간 멈췄다. 지금 코드를 고쳐버리면, 이 질문이 다시 들어왔을 때 잡아줄 게 없다. 방어 코드를 먼저 넣지 말고, 실패 테스트부터 쓰자.

## 뭘 믿고 있었나

문제의 코드는 대략 이렇게 생겼다.

```python
parsed = parse_json_response(response)
recommendations = parsed["recommendations"]
for rec in recommendations:
    title = rec["title"].lower()
    ...
```

`parse_json_response` 가 항상 `dict` 를 준다는 가정, `recommendations` 안의 원소가 항상 `dict` 라는 가정. 둘 다 내가 만든 프롬프트에 대한 신뢰였지, 계약이 아니었다.

LLM — Large Language Model, 대규모 언어 모델 — 은 조금만 흔들려도 `null` 을 뱉거나 리스트를 반환한다. 마크다운 코드 블록으로 JSON을 감싸버리면 파서 상류에서 이미 이상해진다. `parsed["recommendations"]` 는 그 순간 `TypeError` 로 죽는다.

두 번째 함정은 안쪽이었다. LLM이 `["item1", "item2"]` 같은 문자열 리스트를 돌려주면, `rec["title"]` 에서 바로 터진다. 이건 위쪽 방어를 통과해도 걸린다.

한 층 더 있었다. DB에서 JSON 컬럼을 읽어 키워드로 쓰는 부분.

```python
keywords = json.loads(row["keywords"])   # ["수학", "연산", None, 42]
filtered = [k for k in keywords if k in query]
```

DB 컬럼이 `TEXT` 라고 해서 문자열만 들어온다는 보장은 없다. 예전 마이그레이션에서 뭐가 섞였는지, 다른 서비스가 뭘 쓰고 갔는지 모른다. `k.lower()` 같은 걸 뒤에서 부르면 조용히 죽는다. 예외 트레이스는 나오지만 배치 안에서라 하나만 실패하고 넘어간다. 이런 게 제일 무섭다.

## 테스트를 먼저

Red → Green → Refactor. 순서를 지켰다.

```python
# tests/test_parser.py

def test_parse_returns_non_dict():
    # LLM이 dict가 아니라 리스트를 뱉었을 때
    result = parse_recommendation_response([1, 2, 3])
    assert result == []


def test_recommendations_contains_non_dict():
    # recommendations 안에 문자열/None이 섞였을 때
    raw = {"recommendations": ["문자열", None, {"title": "정상"}]}
    result = parse_recommendation_response(raw)
    assert len(result) == 1
    assert result[0]["title"] == "정상"


def test_keyword_list_with_non_strings():
    # DB에서 읽은 리스트에 None, int가 섞였을 때
    raw_list = ["수학", None, 42, "연산"]
    result = _safe_loads_list(raw_list)
    assert result == ["수학", "연산"]
```

`pytest tests/ -v` 를 돌리면 셋 다 빨간 줄. 예상대로다. 이제 방어 코드를 붙인다.

## 파서 하드닝

```python
def parse_recommendation_response(parsed) -> list[dict]:
    if not isinstance(parsed, dict):
        return []

    recommendations = parsed.get("recommendations", [])
    if not isinstance(recommendations, list):
        return []

    return [rec for rec in recommendations if isinstance(rec, dict)]
```

예외를 올릴 수도 있었다. 그런데 이 함수의 호출부는 이미 "빈 결과면 다른 축으로 폴백" 하는 로직을 가지고 있었다. 여기서 예외를 새로 던지면 그 위의 서비스 계층까지 다 건드려야 한다. 조용히 `[]` 로 떨어뜨리고, 대신 상류에 로깅을 남기는 쪽이 이 코드베이스의 결에 맞았다.

`isinstance` 검사가 두 번 나오는 게 마음에 안 들 수는 있는데, 지금은 이게 맞다. 최상단에서 한 번, 리스트 원소에서 한 번. 계층이 다른 실패다.

## DB 리스트 하드닝

```python
def _safe_loads_list(raw: list) -> list[str]:
    return [item for item in raw if isinstance(item, str)]
```

한 줄이다. 짧아서 그냥 인라인으로 두고 싶었는데 함수로 뺐다. 이유는 두 가지. 하나는 테스트에서 이름으로 잡을 수 있게 하려고. 다른 하나는 "여기가 신뢰 경계다" 를 코드로 남기려고. 함수 이름이 주석보다 오래 간다.

## 남은 것

세 테스트가 초록으로 바뀌었고, 기존 테스트도 다 통과했다. 여기까지가 이번 리뷰에서 요구한 범위였다.

그런데 이걸 다 붙이고 나서도 마음에 걸리는 게 있다. `parse_recommendation_response` 가 조용히 `[]` 를 반환하도록 만들었기 때문에, LLM 응답이 실제로 얼마나 자주 스키마를 벗어나는지 지금은 계측이 없다. 폴백은 걸어두지만 몇 %가 폴백에 걸리는지는 안 보인다. 이걸 로그·메트릭으로 뽑아두지 않으면 나중에 프롬프트가 나빠져도 감지할 방법이 없다.

다음에 이 파서를 다시 열 때 붙일 것: 폴백에 걸린 케이스의 원본 응답 샘플링 로깅, 그리고 폴백 비율 지표. 지금은 그 자리만 표시해뒀다.