---
title: "LLM 응답과 DB JSON을 믿었다가 추천이 조용히 망가지는 순간"
date: 2026-07-23 17:00:00 +0900
categories: [Backend]
tags: [python, llm, defensive-coding, recommender]
---

에러 로그는 깨끗했다. 그런데 추천이 이상했다.

지표로 잡히지도 않았다. 응답 코드는 200, 결과는 비어있지 않았고, 사용자도 뭔가는 받아갔다. 다만 받은 게 엉뚱했다. 프론트에서 "이거 왜 이런 게 뜨죠?" 라는 캡처가 몇 장 넘어오고 나서야, 서비스가 조용히 반쯤 죽어있었다는 걸 알았다.

원인을 파헤치다 두 군데서 같은 냄새가 났다. LLM 응답 파싱과 DB의 JSON 컬럼 파싱. 둘 다 "그 형태로 들어올 거라 믿었던" 지점이었다.

## 처음엔 LLM 쪽만 의심했다

프롬프트에 "이 스키마로 JSON을 반환하라"고 못박아 뒀으니 딴 게 올 리 없다고 생각했다. 그런데 실제 로그를 뒤져보니 이런 게 섞여 있었다.

- `"죄송합니다, 추천 결과를 생성할 수 없습니다."` — 애초에 JSON이 아님
- `{"error": "context too long"}` — JSON이긴 한데 우리 스키마가 아님
- `{"recommendations": "없음"}` — 키는 맞는데 값이 문자열

마지막 케이스가 제일 얄궂었다. `json.loads`도 통과하고 `data["recommendations"]`도 통과한다. 그러곤 저 아래 어딘가에서 `for r in recs: r.lower()` 하는 순간에야 `AttributeError`가 튀어나온다. 스택 트레이스만 보면 원인이 LLM인지 코드 버그인지 헷갈렸다.

원래 파싱 코드는 이런 모양이었다.

```python
def parse_json_response(response: str) -> list:
    data = json.loads(response)
    return data["recommendations"]
```

세 줄이 세 종류의 예외를 낳는다. `JSONDecodeError`, `KeyError`, 그리고 반환된 값이 list가 아닐 때 호출부에서 터지는 `TypeError`.

고친 뒤 모양은 이렇다. 별로 화려하지 않다.

```python
def parse_json_response(response: str) -> list[str]:
    try:
        data = json.loads(response)
    except (json.JSONDecodeError, TypeError):
        return []

    if not isinstance(data, dict):
        return []

    recs = data.get("recommendations", [])
    if not isinstance(recs, list):
        return []

    return [r for r in recs if isinstance(r, str)]
```

키가 있어도 값의 타입을 다시 확인한다. 이걸 안 하면 앞에서 말한 세 번째 케이스가 그대로 통과한다. 리스트 컴프리헨션은 `[1, "수학", None]` 같은 이물질을 걸러낸다 — 이게 필요했던 이유는 다음 문제 때문이었다.

## 진짜 골치는 DB였다

LLM은 그래도 "내가 제어 못 하는 외부"라는 인식이 있었다. DB는 안이라고 생각했다. 그런데 안이 아니었다.

추천 엔진에서 콘텐츠 풀을 만들 때, 각 콘텐츠의 메타데이터가 필요하다. NCS 코드, 태그, 제목 같은 것들. 이 정보가 얼마나 갖춰졌는지에 따라 어떤 콘텐츠를 대표로 삼을지 결정한다.

```python
def _info_score(c: dict) -> int:
    return (2 if c.get('keyword_tags') else 0) \
         + (1 if c.get('ncs_code') else 0) \
         + (1 if c.get('title') else 0)
```

같은 UUID(콘텐츠 고유 식별자)를 가진 레코드가 여러 개 있으면, `_info_score`가 높은 쪽을 남긴다.

```python
@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _uuid_map(tenant_id: str) -> dict:
    pool = _company_pool(tenant_id)
    m: dict[str, str] = {}
    for cid, c in pool.items():
        u = c.get('content_uuid')
        if not u:
            continue
        if u not in m or _info_score(c) > _info_score(pool[m[u]]):
            m[u] = cid
    return m
```

`keyword_tags`가 있으면 +2점, `ncs_code`가 있으면 +1점. 이 점수가 결국 추천 후보의 품질을 정한다. 그런데 `keyword_tags`는 DB에 JSON 문자열로 저장되어 있었다. 원래 계약은 `["str", "str", ...]`. 그런데 어느 시점부터 파이프라인 상류에서 `[1, "수학", null]` 같은 이물질이 섞여 들어왔다.

`json.loads`는 통과한다. `if c.get('keyword_tags')`도 통과한다 (빈 리스트가 아니니까). 그래서 이 콘텐츠는 `_info_score`에서 +2를 받고 "정보 잘 갖춘 콘텐츠"로 승격된다. 실제로는 `1.lower()`를 시도하는 순간 아래에서 폭발할 시한폭탄인데.

문제는, **터지지 않는 경로도 있었다**는 것이다. 그 콘텐츠가 추천 결과에 뽑히지 않는 요청에선 아무 일도 없다. 뽑히는 요청에서만 조용히 이상한 결과가 나가거나, 폴백 경로로 빠져 엉뚱한 걸 반환한다.

그래서 파싱 지점에 `_safe_loads_list`를 하나 만들어 놓고 DB에서 리스트가 나온다고 주장하는 모든 컬럼을 이 함수로만 읽게 통일했다.

```python
def _safe_loads_list(raw: str | None) -> list[str]:
    if not raw:
        return []
    try:
        parsed = json.loads(raw)
    except (json.JSONDecodeError, TypeError):
        return []
    if not isinstance(parsed, list):
        return []
    return [str(item) for item in parsed if item is not None]
```

`str(item)`으로 강제 변환하는 부분이 논쟁거리였다. 이물질을 그냥 버리는 편이 안전한가, 아니면 문자열화해서 살리는 편이 데이터 유실이 덜한가. 지금은 살리는 쪽으로 뒀는데, 이건 다음 이슈에서 다시 볼 것 같다. 태그로 `"1"`이 들어오는 게 태그가 아예 없는 것보다 나은지, 잘 모르겠다.

## `_info_score`가 진짜로 하려던 일

이 사건을 겪고 나서 `_info_score`를 다시 봤다. 이 함수의 원래 의도는 "메타데이터가 잘 갖춰진 콘텐츠를 대표로 삼자"였다. 그런데 그 판단 기준이 "필드가 존재하느냐(truthy 하냐)"였다. 존재만 확인하고 내용은 검증하지 않는다.

같은 종류의 함정이 `_company_pool` 근처에도 있었다.

```python
@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _inst_pool(tenant_id: str) -> dict:
    return {cid: c for cid, c in _company_pool(tenant_id).items()
            if c.get('content_type') == 'INSTITUTION' and c.get('has_meta')}
```

`has_meta`는 상류 파이프라인에서 계산해서 붙여주는 플래그였다. 그런데 이 플래그가 "메타가 있다"는 것만 보증하고, "메타가 유효하다"는 건 보증하지 않았다. 파이프라인이 바뀌면 여기 정의도 조용히 바뀐다. 우리 쪽에선 알 방법이 없다.

`lru_cache`가 붙어있는 것도 별개의 문제였다. 한번 오염된 pool이 들어오면 프로세스가 살아있는 동안 계속 같은 결과를 낸다. 나쁜 데이터가 캐시된다.

## 재현부터 잡고 갔다

두 군데 다 고칠 때 순서를 지켰다. 방어 코드부터 짜지 않았다.

먼저 실제로 터진 입력값을 pytest 케이스로 박아넣었다. `[1, "수학", None]`, `{"recommendations": "없음"}`, 평문 문자열. 이걸 실패하는 상태로 커밋했다. 그다음 방어 코드를 붙여 통과시켰다.

이렇게 하니까 방어 코드의 범위가 명확해졌다. "모든 예외를 광역 try-except로 감싸자" 같은 유혹이 사라졌다. 테스트에 없는 경우는 코드로도 처리하지 않았다. 지금 아는 것만 방어한다.

## 아직 못 고친 것

- **상류 파이프라인 검증**: 결국 이물질이 어디서 들어왔는지는 여전히 모른다. 소싱 스크립트인지, 임포터인지, 아니면 수동 편집인지. DB 삽입 지점에 스키마 검증을 걸어야 하는데 아직 못 걸었다. 지금 코드는 "받는 쪽에서 다 참는" 상태다.
- **`_safe_loads_list`의 정책**: `str()`로 살릴 것인가, 버릴 것인가. 살리면 `"1"`이라는 태그가 검색에 걸린다. 버리면 파이프라인 이상을 알아채기 더 어려워진다. 실제 사용자 로그를 좀 더 본 다음에 정한다.
- **캐시된 나쁜 데이터**: `lru_cache`에 한번 잡힌 오염된 pool을 어떻게 무효화할지. TTL을 걸든 파일 mtime을 보든 해야 하는데, 아직 안 했다.

로그가 조용하면 서비스도 조용히 좋은 거라고 믿는 습관을 조금 잃었다. 이제 "왜 이렇게 조용하지?"를 한 번 더 의심한다.