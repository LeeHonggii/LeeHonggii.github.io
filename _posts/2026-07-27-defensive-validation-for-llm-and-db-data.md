---
title: "LLM 응답이 dict가 아닐 때 추천 서버는 어떻게 죽는가"
date: 2026-07-27 17:00:00 +0900
categories: [Backend]
tags: [python, llm, defensive-coding, pytest]
---

코드 리뷰 중이었다. 동료가 스크롤을 멈추더니 한 줄을 짚었다.

> "이거, LLM 응답이 항상 dict라고 가정하고 있는 거죠?"

"네, 그렇게 오게 되어 있어요" 라고 답하려다가 손이 멈췄다. 그렇게 "오게 되어 있는" 게 아니라, 그냥 지금까지 그렇게 왔던 것 뿐이다. 그 둘은 다르다.

## 파싱은 성공했는데 다음 줄에서 죽는다

문제의 함수는 대충 이렇게 생겼었다. LLM (Large Language Model, 대규모 언어 모델) 에서 받은 텍스트를 JSON으로 풀어 keyword 리스트와 tool 리스트를 꺼내는 자리.

```python
def parse_json_response(response_text: str) -> dict:
    try:
        data = json.loads(response_text)
    except json.JSONDecodeError:
        return {}

    return {
        "keywords": data["keywords"],
        "tools": data["tools"],
    }
```

fallback 은 있다. `JSONDecodeError` 를 잡아 빈 dict 를 돌려준다. 나는 이걸로 "방어된 함수"라고 생각하고 있었다.

그런데 LLM 이 가끔 이렇게 답한다.

```json
["keyword1", "keyword2"]
```

`json.loads` 는 성공한다. `data` 에는 list 가 들어간다. 그리고 바로 다음 줄 `data["keywords"]` 에서 `TypeError: list indices must be integers or slices, not str` 이 튀어나온다. `except JSONDecodeError` 는 이 예외를 잡아주지 않는다. 예외는 그대로 위로 올라간다. 추천 요청 하나가 500 으로 죽는다.

파싱은 성공했다. **다음 줄에서 죽는다.** 이게 이 버그의 핵심이다.

## 조용히 0건을 돌려주는 쪽이 더 무섭다

이걸 인지하고 나니 비슷한 자리가 하나 더 눈에 들어왔다. 추천 후보를 필터링할 때 DB 의 JSON 배열 컬럼을 꺼내 저장된 키워드 리스트를 훑는다. 그때 이런 코드가 있다.

```python
for keyword in stored_keywords:
    if query.lower() in keyword.lower():
        ...
```

배열 안에 `None` 이나 숫자가 하나라도 섞이면 `.lower()` 호출에서 `AttributeError` 가 난다. 이건 그나마 나은 경우다. 어떤 경로에선 상위 `try` 가 이 예외를 삼켜서 그 콘텐츠가 그냥 매칭 후보에서 빠진다. 500 대신 추천 0건. 알람은 울리지 않는다.

500 은 눈에 띈다. 조용히 0건은 며칠간 아무도 모른다. 이쪽이 더 무섭다.

## 고치기 전에 실패하는 테스트부터

이 지적을 받자마자 바로 `isinstance` 체크를 넣고 싶었지만 참았다. 리뷰에서 나온 말은 **테스트로 못 박고** 나서 고쳐야 다음에 또 이 얘기가 나오지 않는다.

```python
def test_parse_json_response_with_list_input():
    # LLM이 list 를 돌려주는 경우
    response = json.dumps(["keyword1", "keyword2"])
    assert parse_json_response(response) == {}


def test_parse_json_response_with_scalar_input():
    # 문자열/숫자 한 개짜리도 유효한 JSON이다
    assert parse_json_response(json.dumps("hello")) == {}
    assert parse_json_response(json.dumps(42)) == {}


def test_safe_loads_list_with_non_string_elements():
    raw = json.dumps(["수학", 42, None, "도형"])
    result = _safe_loads_list(raw)
    assert all(isinstance(x, str) for x in result)
```

돌리면 당연히 빨갛다. `TypeError`, `AttributeError` 가 그대로 올라온다. 재현이 됐다는 뜻이다. 이제 고칠 자격이 생겼다.

## 실제 방어는 한 줄

```python
def parse_json_response(response_text: str) -> dict:
    try:
        data = json.loads(response_text)
    except json.JSONDecodeError:
        return {}

    if not isinstance(data, dict):
        return {}

    return {
        "keywords": data.get("keywords", []),
        "tools": data.get("tools", []),
    }
```

`isinstance(data, dict)` 한 줄이 전부다. `data["keywords"]` 를 `data.get("keywords", [])` 로 바꾼 것도 같은 맥락 — dict 인 게 확인됐다고 해서 원하는 키가 반드시 들어있다는 뜻은 아니다. `json.loads` 성공은 세 가지를 하나도 보장하지 않는다: 타입도, 키도, 값의 타입도.

DB 쪽은 조금 더 방어적으로 갔다.

```python
def _safe_loads_list(value: str | None) -> list[str]:
    if not value:
        return []
    try:
        parsed = json.loads(value)
    except (json.JSONDecodeError, TypeError):
        return []

    if not isinstance(parsed, list):
        return []

    return [str(x) for x in parsed if x is not None]
```

`str(x)` 변환은 잠깐 고민했다. 숫자가 들어오면 문자열로 강제 변환하는 게 맞을까. 이 자리를 지나는 값은 결국 `.lower()` 를 태워 문자열 매칭에 쓴다. 즉 이 함수의 계약은 "타입 안전한 문자열 리스트를 내려보내는 것"이다. 잘못된 원소를 조용히 통과시키는 것보다는, 명시적으로 문자열로 바꾸거나 걸러내는 편이 상위 코드에 덜 미안하다.

## 남은 것

테스트는 초록으로 돌아왔다. 그런데 이 함수와 비슷하게 "파싱 성공 후 곧바로 인덱싱" 하는 자리가 서비스 안에 몇 개 더 있다. LLM 응답을 다루는 자리, 캐시에서 꺼낸 JSON 을 다루는 자리, 외부 API 응답을 다루는 자리. 아직 다 안 봤다. 리뷰에서 지적당한 두 자리만 고쳤을 뿐이다.

`json.loads` 를 호출하는 자리를 한 번에 grep 해서 각각의 다음 줄을 봐야 하는데, 이 작업은 다음 스프린트로 밀렸다. 밀린 채로 계속 두면 안 된다는 것도 알고 있다. 그래서 이 글을 오늘 쓴다 — 내가 잊지 않으려고.