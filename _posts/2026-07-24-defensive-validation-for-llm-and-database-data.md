---
title: "LLM 응답과 DB 데이터는 dict와 string이라고 믿으면 안 된다"
date: 2026-07-24 17:00:00 +0900
categories: [Backend]
tags: [python, llm, defensive-programming]
---

추천 파이프라인이 빈 리스트를 뱉기 시작했다. 예외 로그는 없다. 스택 트레이스도 없다. 그냥 `[]`.

처음엔 프롬프트 문제인 줄 알았다. LLM(Large Language Model, 거대 언어 모델)이 이상한 답을 내놓나 싶어 원문을 찍어봤는데, 정작 응답은 멀쩡했다. `{"recommendations": [...]}` 형태로 잘 들어오고 있었다.

이상한 건 그 다음 단계였다.

## `json.loads` 가 성공했다는 말은 dict 라는 뜻이 아니다

문제 지점을 좁혀보니 파싱 직후였다.

```python
raw = llm.invoke(prompt)
parsed = json.loads(raw.content)
# 이 아래에서 parsed["recommendations"] 를 바로 씀
```

`json.loads` 는 예외를 던지지 않았다. 그런데 `parsed` 가 `dict` 가 아니었다. 그날 LLM이 어떤 이유로 최상위를 배열로 뱉어냈고 — `[{"id": "..."}]` 같은 형태 — `json.loads` 는 그걸 그대로 `list` 로 돌려줬다. 그 다음 `.get()` 호출이 `AttributeError` 로 죽었고, 위쪽 어딘가의 넓은 `try/except` 가 그걸 통째로 삼켰다. 그래서 조용한 빈 리스트.

`json.loads` 성공 = 파싱 가능한 JSON. 그 이상은 아니다. `dict` 를 원했으면 `dict` 인지 물어봐야 한다.

파싱 함수를 이렇게 바꿨다.

```python
def parse_json_response(raw_content: str) -> dict:
    try:
        parsed = json.loads(raw_content)
    except json.JSONDecodeError:
        return {}

    if not isinstance(parsed, dict):
        return {}

    recs = parsed.get("recommendations", [])
    if not isinstance(recs, list):
        parsed["recommendations"] = []
        return parsed

    parsed["recommendations"] = [r for r in recs if isinstance(r, dict)]
    return parsed
```

두 단계로 나눈 게 핵심이다. 최상위가 `dict` 인지 먼저 보고, 그 안의 `recommendations` 가 `list` 인지 본다. 마지막으로 배열 원소 하나하나가 `dict` 인지 다시 본다. LLM은 세 층 중 어느 층에서도 배신할 수 있다.

## 두 번째 실수는 DB 쪽에서 왔다

같은 파이프라인의 다른 자리. 콘텐츠 메타데이터가 JSON 배열로 저장되어 있고, 그걸 꺼내서 키워드 매칭에 쓴다.

```python
if any(kw.lower() in tag.lower() for tag in tags):
    ...
```

대부분은 `["대수", "함수", "방정식"]` 같이 문자열 배열이다. 그런데 데이터를 이관하는 과정에서 몇 건은 `[1, "대수", null]` 처럼 섞여 들어와 있었다. 어디서 정수가 들어왔는지, 왜 `null` 이 남았는지는 이관 스크립트 쪽 이야기라 여기선 넘어간다. 어쨌든 그 몇 건이 `.lower()` 에서 터진다.

터지기만 하면 차라리 나았을 텐데, 이번에도 위쪽 `try/except` 가 삼켰다. 해당 콘텐츠는 결과에서 조용히 빠졌다. 이런 종류의 조용한 실패가 제일 무섭다. 데이터가 나오지 않는 이유를 "추천 로직이 안 맞나?" 쪽에서 한참 뒤진 뒤에야 아래를 보게 된다.

DB에서 꺼낼 때 한 번 정규화해서, 뒤쪽 코드가 타입을 다시 걱정할 일이 없게 만들었다.

```python
def _safe_loads_list(raw: str | list | None) -> list[str]:
    if isinstance(raw, str):
        try:
            data = json.loads(raw)
        except json.JSONDecodeError:
            return []
    elif isinstance(raw, list):
        data = raw
    else:
        return []

    return [item for item in data if isinstance(item, str)]
```

`str` 로 들어올 때도 있고 이미 `list` 로 들어올 때도 있어서 두 경로를 다 받는다. 최종적으로 문자열만 추린다. `None`, `int`, `float`, 중첩 `dict` 전부 버린다. 뒤쪽 매칭 함수는 이제 "여긴 무조건 `list[str]` 이다" 를 가정해도 된다.

키워드 함수 진입부에도 얇은 방어를 붙였다.

```python
def find_unit_codes_by_keyword(keyword: str, ...) -> list[str]:
    if not isinstance(keyword, str) or not keyword.strip():
        return []
    ...
```

타입 힌트는 문서일 뿐 강제가 아니다. LLM이 만들어낸 어떤 값이 `keyword` 자리로 흘러들어올 수 있다. 진입부에서 한 줄로 막는 게 싸다.

## 수정보다 재현 테스트가 먼저였다

리뷰에서 이 두 문제를 지적받았을 때, 손이 먼저 간 건 `parse_json_response` 를 고치는 쪽이었다. 그러다 멈췄다. 지금 고치면 이 버그가 다시 나타났을 때 알아챌 방법이 없다는 생각이 들었다.

테스트부터 썼다.

```python
def test_parse_json_response_returns_list():
    raw = json.dumps(["keyword1", "keyword2"])
    result = parse_json_response(raw)
    assert result == {}

def test_parse_json_response_filters_non_dict_items():
    raw = json.dumps({"recommendations": [{"id": "T001"}, "invalid", None]})
    result = parse_json_response(raw)
    assert result["recommendations"] == [{"id": "T001"}]
```

빨간불 확인. 그 다음 위의 파싱 코드로 고치고 초록불 확인. `_safe_loads_list` 쪽도 같은 순서로 갔다. 몇 달 뒤 누가 "이 `isinstance` 체크 필요없어 보이는데?" 하고 지울 때 잡아줄 사람은 결국 이 테스트뿐이다.

## 남은 것

이 방어 로직을 붙이고 나서 조용한 빈 결과는 사라졌다. 그런데 정말 사라졌는지, 아니면 다른 자리에서 여전히 삼켜지고 있는지는 아직 확신이 없다. 위쪽 넓은 `try/except` 를 걷어내고 로그를 남기는 쪽으로 바꾸는 작업이 다음 순서에 있다. 삼키는 코드가 있는 한 방어는 늘 반쪽짜리다.