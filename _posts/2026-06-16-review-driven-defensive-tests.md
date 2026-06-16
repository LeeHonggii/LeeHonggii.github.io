---
title: "리뷰 코멘트를 방어 코드로 끝내지 않고 실패 테스트로 고정하기"
date: 2026-06-16 17:00:00 +0900
categories: [Tooling]
tags: [testing, defensive-programming, code-review]
---

# 도입 — "이거 방어 코드 추가해주세요"로 끝나면 안 되는 이유

코드 리뷰에서 이런 코멘트를 받은 적 있다.

> "LLM 응답이 항상 dict라는 보장이 없으니 타입 체크 추가해주세요."

처음엔 그냥 `if not isinstance(parsed, dict): return {}` 한 줄 넣고 PR을 닫았다. 문제가 해결된 것처럼 보였다. 하지만 일주일 뒤 같은 계열의 버그가 다른 함수에서 터졌다. 방어 코드를 추가했지만 **그 동작이 기대대로인지 검증하는 장치**는 남기지 않았기 때문이다.

그 이후로 나는 리뷰 코멘트가 타입·경계 조건을 건드릴 때마다 "테스트 먼저, 수정 나중" 순서를 지키기 시작했다.

## 왜 방어 코드만으로는 부족한가

방어 코드(defensive code)는 예외 상황을 조용히 삼킨다. 그게 장점이면서 동시에 함정이다.

```python
def parse_json_response(raw: str) -> dict:
    parsed = json.loads(raw)
    return parsed  # LLM이 list를 반환하면 여기서 조용히 통과
```

LLM(Large Language Model, 대규모 언어 모델) 응답은 프롬프트가 조금만 바뀌어도 `dict` 대신 `list`나 `str`을 돌려줄 수 있다. `.get("key")` 를 호출하는 쪽에서는 `AttributeError`가 나거나, 더 나쁘게는 `None`을 정상값으로 착각하고 계속 흘러간다.

방어 코드를 넣었다고 해도, **그 코드가 실제로 실행되는지, 의도한 값을 반환하는지** 아무도 모른다. 코드가 있다는 것과 동작이 보장된다는 것은 다르다.

## 실패 테스트 먼저 만들기

리뷰 코멘트를 받으면 나는 이 순서를 따른다.

1. **재현 테스트 작성** — 리뷰어가 지적한 입력을 그대로 넣어 실패하게 만든다.
2. **테스트가 실제로 실패하는지 확인** — `pytest tests/ -v` 로 RED 상태를 눈으로 본다.
3. **수정 후 GREEN 확인** — 코드를 고친 뒤 같은 테스트가 통과하는지 확인한다.

LLM 응답 파싱 예시로 보면:

```python
# tests/test_parse_json_response.py

def test_parse_returns_dict_when_llm_gives_list():
    """LLM이 list를 반환했을 때 빈 dict로 fallback해야 한다."""
    raw_list_response = '[{"key": "value"}]'  # list 형태 응답
    result = parse_json_response(raw_list_response)
    assert isinstance(result, dict), f"expected dict, got {type(result)}"

def test_parse_returns_dict_when_llm_gives_plain_string():
    raw_string_response = '"just a string"'
    result = parse_json_response(raw_string_response)
    assert isinstance(result, dict)
```

이 테스트를 먼저 돌리면 당연히 실패한다. 그때 비로소 수정한다.

```python
def parse_json_response(raw: str) -> dict:
    try:
        parsed = json.loads(raw)
    except json.JSONDecodeError:
        return {}

    if not isinstance(parsed, dict):
        return {}

    return parsed
```

테스트가 GREEN이 되는 순간, 이 방어 로직은 **회귀 방지 자산**으로 코드베이스에 박힌다.

## DB JSON 배열에서의 조용한 장애

비슷한 패턴이 DB에서 읽은 JSON 배열에서도 나타난다. 어떤 백엔드 서비스에서 키워드 목록을 JSON 컬럼에 저장했는데, 데이터 입력 과정에서 숫자나 `null`이 섞여 들어온 경우가 있었다.

```python
keywords = ["미분", "적분", None, 42, "함수"]
```

이 배열을 그대로 순회하면 `.lower()` 나 부분 문자열 매칭(substring matching)에서 `AttributeError`가 터지거나, `None`이 비교 연산을 통과해버린다. 역시 조용한 장애(silent failure)다.

테스트로 먼저 재현한다.

```python
def test_keyword_matching_ignores_non_string_items():
    """배열 안에 None과 int가 섞여도 크래시 없이 str만 처리해야 한다."""
    mixed_keywords = ["미분", None, 42, "적분"]
    result = match_keywords(mixed_keywords, query="미분")
    assert "미분" in result
    assert None not in result
```

RED 확인 후 수정:

```python
def match_keywords(keywords: list, query: str) -> list:
    if not isinstance(keyword, str):  # 각 원소 타입 검사
        return []
    return [kw for kw in keywords if isinstance(kw, str) and query in kw.lower()]
```

## 리뷰 코멘트를 자산으로 바꾸는 습관

이 접근법의 핵심은 **리뷰어의 직관을 테스트 케이스로 번역**하는 것이다. 리뷰어는 보통 "이런 엣지 케이스가 있지 않을까?" 라는 감으로 코멘트를 단다. 그 감을 코드로 굳혀두지 않으면, 다음 사람이 같은 함수를 건드릴 때 그 직관이 사라진다.

| 방식 | 장점 | 한계 |
|------|------|------|
| 방어 코드만 추가 | 빠름 | 동작 보장 없음, 회귀 탐지 불가 |
| 테스트 먼저 작성 후 수정 | 회귀 방지, 의도 명문화 | 시간이 조금 더 걸림 |

시간이 더 걸린다는 단점은 실제로 크지 않다. 테스트 하나 작성하는 데 10분이 채 안 걸리고, 나중에 같은 버그를 디버깅하는 시간은 그 몇 배다.

## 마치며

LLM 응답이나 DB JSON처럼 **내가 통제하지 못하는 입력**을 다루는 코드는 방어 로직이 반드시 필요하다. 하지만 방어 코드를 넣는 것만으로는 충분하지 않다. 리뷰 코멘트가 타입이나 경계 조건을 짚어줄 때, 그것을 실패 테스트로 먼저 만들고 수정하면 그 코멘트는 휘발되지 않고 코드베이스에 영구히 남는다.

재현 → 실패 확인 → 수정 → 통과 확인. 이 네 단계가 습관이 되면, 리뷰는 단순한 지적이 아니라 테스트 스위트(test suite)를 키우는 기회가 된다.