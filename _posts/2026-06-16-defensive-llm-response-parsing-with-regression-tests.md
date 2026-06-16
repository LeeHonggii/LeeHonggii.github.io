---
title: "LLM 응답은 dict라고 믿으면 안 된다: 추천 파이프라인 방어 코드 리뷰"
date: 2026-06-16 17:00:00 +0900
categories: [Backend]
tags: [LLM, Python, 방어코드, pytest]
---

# 도입 (왜 이 글)

코드 리뷰에서 두 군데를 지적받았다. 처음엔 "설마 그런 경우가 오겠어?"였다. 그런데 재현 테스트를 짜고 나니 실제로 터지는 버그였다. LLM(Large Language Model, 대규모 언어 모델) 응답을 파싱하는 코드와 DB에서 꺼낸 메타데이터를 다루는 코드 — 둘 다 타입을 암묵적으로 가정하고 있었다.

이 글은 그 지적을 재현 테스트로 먼저 실패시키고, 방어 코드를 추가한 뒤, 회귀 테스트로 마무리한 기록이다.

## LLM 응답은 dict가 아닐 수 있다

기존 코드는 대략 이랬다:

```python
def parse_json_response(raw_response: str) -> list:
    parsed = json.loads(raw_response)
    return parsed.get("recommendations", [])
```

`json.loads`가 성공해도 결과가 `dict`라는 보장은 없다. LLM이 배열(`[]`)을 직접 뱉거나, 예상치 못한 래핑 구조로 응답하면 `list`나 `str`이 나온다. 그 위에 `.get()`을 호출하면 `AttributeError`다.

```python
>>> parsed = json.loads('[]')
>>> parsed.get("recommendations")
AttributeError: 'list' object has no attribute 'get'
```

fallback 분기가 있어도, `json.loads` 자체는 성공했으니 예외 없이 통과한다. fallback이 있어도 이 케이스는 잡히지 않는다.

수정은 단순하지만 명시적으로:

```python
def parse_json_response(raw_response: str) -> list:
    try:
        parsed = json.loads(raw_response)
    except (json.JSONDecodeError, TypeError):
        return []

    if not isinstance(parsed, dict):
        return []

    recommendations = parsed.get("recommendations", [])
    if not isinstance(recommendations, list):
        return []

    return recommendations
```

`isinstance` 체크 두 번. 지저분해 보이지만 LLM 출력은 계약(contract)이 없는 외부 입력이다.

## DB 메타데이터도 타입이 보장되지 않는다

DB의 JSON 컬럼에서 꺼낸 키워드 리스트를 순회하는 코드가 있었다:

```python
for keyword in keyword_list:
    if query.lower() in keyword.lower():
        matched.append(keyword)
```

대부분은 문자열이지만, 데이터가 잘못 적재되면 `int`나 `None`이 섞인다. 그러면 `keyword.lower()`에서 `AttributeError`가 터진다. 추천 결과 리스트의 원소(`rec`) 역시 마찬가지다.

재현 테스트를 먼저 작성했다:

```python
def test_keyword_match_with_non_string_values():
    keyword_list = ["미분", 42, None, "적분"]
    result = match_keywords("미분", keyword_list)
    assert result == ["미분"]
```

```bash
pytest tests/ -v
# FAILED — AttributeError: 'int' object has no attribute 'lower'
```

테스트가 먼저 실패하는 걸 확인한 뒤 수정했다:

```python
for keyword in keyword_list:
    if not isinstance(keyword, str):
        continue
    if query.lower() in keyword.lower():
        matched.append(keyword)
```

추천 원소 단위도 같은 패턴:

```python
for rec in recommendations:
    if not isinstance(rec, dict):
        continue
    # 이후 rec["title"] 접근
```

## 테스트 우선, 그다음 수정

이번 리뷰에서 가장 유익했던 순서는 이것이다:

> 리뷰 지적 → 재현 테스트 작성 → 실패 확인 → 코드 수정 → 테스트 통과

이렇게 하면 두 가지가 생긴다. 수정 전 버그의 존재가 코드로 기록되고, 이후 리팩터링에서 같은 버그가 다시 생겼을 때 자동으로 알아채는 회귀 테스트(regression test, 기존 기능이 깨지지 않는지 확인하는 테스트)가 남는다.

테스트 없이 방어 코드만 추가하면, 한 달 뒤에 "왜 여기 `isinstance` 체크가 있는지" 아무도 모른다.

## 마치며

LLM이 JSON을 반환한다고 해도, 그게 `dict`라고, 그 안의 값이 `list`라고, 리스트 원소가 `str`이라고 가정하는 순간 버그의 씨앗이 심어진다. 외부 입력 — LLM 응답이든, DB에서 꺼낸 JSON이든 — 은 경계(boundary)에서 명시적으로 검증해야 한다.

작은 `isinstance` 체크들이 귀찮아 보이지만, 파이프라인이 사용자 요청에 조용히 빈 결과를 내려보내는 것보다는 낫다.