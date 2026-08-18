---
title: "LLM 응답과 DB 메타데이터를 믿으면 어디서 터지는가"
date: 2026-07-29 17:00:00 +0900
categories: [Backend]
tags: [python, testing, defensive-coding, llm]
---

코드 리뷰에서 두 줄을 받았다.

> LLM 응답이 항상 dict라고 가정하고 있는데, str이 오면요?
> DB JSON 리스트 안에 None이 섞이면 `.lower()` 에서 안 터지나요?

둘 다 "에이 설마" 하고 넘기고 싶은 종류였다. 실제로 개발하면서 한 번도 본 적이 없는 경우니까. 그런데 넘기기 전에 재현부터 해보기로 했다. 재현 테스트를 짜고 돌리니 **둘 다 예외 없이 크래시**. 두 번의 "설마"가 두 번 다 틀렸다.

## 첫 번째: fallback 이전에 터진다

LLM(Large Language Model)에게 JSON 포맷을 요구하면 대부분 잘 준다. 문제는 "대부분"의 반대편이다. 어떨 때는 사과문을 plain text로 뱉고, 어떨 때는 최상위가 `list`인 JSON을 준다. 프롬프트를 엄격하게 짜도 완전히 막지는 못한다.

내 코드는 이런 모양이었다.

```python
def parse_json_response(raw: str) -> dict:
    data = json.loads(raw)
    return {
        "keywords": data.get("keywords", []),
        "level": data.get("level", ""),
    }
```

`.get()` 뒤에 fallback을 달아뒀으니 안전하다고 생각했다. 그런데 `json.loads`가 `list`를 뱉는 순간 `data.get`이 존재하지 않아 `AttributeError`가 난다. **fallback 값에 닿기도 전에 죽는다.** 방어를 한 줄이라고 여긴 곳이 실은 방어가 아니었다.

리뷰 지적을 받고 처음 한 건 fix가 아니라 실패 테스트였다.

```python
def test_parse_json_response_returns_list():
    raw = json.dumps(["keyword1", "keyword2"])
    result = parse_json_response(raw)
    assert result["keywords"] == []

def test_parse_json_response_plain_text():
    result = parse_json_response("죄송합니다, 형식 오류입니다.")
    assert result["keywords"] == []
```

`pytest -v` 돌리면 두 개 다 빨간불. 이제서야 리뷰어의 지적이 내 손 안에 잡혔다. 이 두 개를 초록으로 만드는 게 fix의 정의가 된다.

```python
def parse_json_response(raw: str) -> dict:
    try:
        data = json.loads(raw)
    except (json.JSONDecodeError, TypeError):
        data = {}
    if not isinstance(data, dict):
        data = {}
    return {
        "keywords": data.get("keywords", []),
        "level": data.get("level", ""),
    }
```

`isinstance(data, dict)` 한 줄. 그게 없어서 fallback이 fallback 노릇을 못 하고 있었다.

## 두 번째: 데이터가 깨끗한 동안에는 절대 안 잡힌다

두 번째 지적은 더 성가셨다. DB의 JSON 컬럼에서 태그 리스트를 읽어와 필터링하는 코드다.

```python
tags = ["수학", None, "중등", 2]
matches = [t for t in tags if "수학" in t.lower()]
```

`None.lower()`도, `(2).lower()`도 다 `AttributeError`. 그런데 이 코드가 성가신 이유는 크래시라서가 아니라 **정상 데이터에서는 절대 재현되지 않기 때문**이다. 개발 중 내가 만든 fixture는 다 깨끗한 문자열 리스트다. 운영 DB에서 어느 날, 어떤 이유로 `None`이 하나 들어간 레코드가 조회되는 순간에만 터진다. QA에서도 안 잡히고 스테이징에서도 안 잡힌다.

이런 코드는 방어를 안 하면 언젠가 반드시 새벽에 알림이 온다.

여기서도 재현 먼저.

```python
def test_safe_loads_list_with_non_strings():
    raw = json.dumps(["수학", None, 2, "중등"])
    result = _safe_loads_list(raw)
    assert all(isinstance(item, str) for item in result)
    assert "수학" in result
    assert "중등" in result
```

그리고 함수를 만들었다. 인라인으로 방어하는 대신 `_safe_loads_list`를 만들어서 "DB JSON 리스트를 문자열 리스트로 정규화"하는 책임 하나만 지게 했다.

```python
def _safe_loads_list(raw) -> list[str]:
    if not raw:
        return []
    try:
        items = json.loads(raw) if isinstance(raw, str) else raw
    except (json.JSONDecodeError, TypeError):
        return []
    if not isinstance(items, list):
        return []
    return [str(item) for item in items if item is not None]
```

한 가지 주의한 것. `str(item)`을 먼저 씌우고 싶은 유혹이 있는데, 그러면 `None`이 `"None"` 문자열로 살아 남는다. 검색어에 `"none"`을 넣으면 매칭돼버리는 유령 태그가 생기는 것이다. 그래서 **`None` 제거가 `str()` 캐스팅보다 먼저** 와야 한다. 이 순서를 뒤집었다가 한 번 되돌렸다.

## 두 fix의 공통점

돌이켜보면 두 코드는 성격이 달랐다. 하나는 외부 API(LLM), 하나는 내부 DB. 하나는 매번 다른 응답, 하나는 대체로 안정적인 데이터. 그런데 무너지는 방식은 같았다. **내가 통제하지 않는 값의 타입을 통제한다고 착각한 것.**

`json.loads`의 반환 타입은 서명상 `Any`다. DB JSON 컬럼도 `Any`다. 코드에서 `dict`처럼, `list[str]`처럼 다루는 순간 그 캐스팅은 근거 없는 낙관이다. 개발 중 데이터가 낙관을 지지해줄 뿐이다.

## 남은 것

실패 테스트를 먼저 쓰는 게 TDD를 지키자는 얘기가 아니라는 걸 이번에 다시 확인했다. **리뷰어가 짚은 곳이 진짜로 터지는지 내 눈으로 보고 싶다는 게** 이유의 전부였다. 지적이 코드로 번역되면 논쟁이 끝난다. "생각해볼게요"가 "이 테스트가 통과할 때까지"로 바뀐다.

아직 못 한 게 있다. LLM 응답 파서는 이번엔 `dict`가 아닐 때 빈 값으로 넘기는 fallback을 선택했는데, 이게 항상 맞는 결정인지는 자신이 없다. 어떤 호출부는 빈 응답을 "정상적인 빈 결과"로 오해할 수도 있다. 로깅을 붙여 얼마나 자주 fallback으로 빠지는지 세본 다음, 임계치를 넘으면 재시도로 갈지 예외로 올릴지 다시 정해야 한다. 아직 그 카운트를 안 재봤다.