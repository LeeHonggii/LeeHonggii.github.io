---
title: "JSON 파싱 에러인 줄 알았는데, 진짜 문제는 실패 상태 저장 방식이었다"
date: 2026-08-14 17:00:00 +0900
categories: [Backend]
tags: [debugging, error-handling, llm-pipeline, data-modeling]
---

500이 떴다. 스택 트레이스 맨 아래는 `json.loads`. 반사적으로 직렬화 쪽을 열었다. 인코딩이 바뀌었나, 어디서 이스케이프가 깨졌나. 삼십 분쯤 그러고 있다가 DB를 직접 열어봤다. 파서 문제가 아니었다.

## 컬럼에 들어있던 값

콘텐츠 생성 파이프라인은 LLM(대형 언어 모델) 응답을 `content` 컬럼에 넣고, 조회 API가 그걸 다시 `json.loads` 로 읽는 구조였다. 문제의 레코드를 뽑아봤다.

```sql
SELECT content_type, status, LEFT(content, 80)
FROM content_items
WHERE item_id = '<omitted>';
```

`content` 안에 이런 게 있었다.

```
gallery 생성 실패: API quota exceeded
```

JSON 아니고 한국어 섞인 평문. 그런데 읽기 쪽은 아무 조건 없이 `json.loads(content)` 를 부르고 있었다. 터지는 게 당연했다.

## 쓰기 경로가 만든 지뢰

문제의 쓰기 쪽은 대충 이런 모양이었다.

```python
try:
    result = llm_generate(prompt)
    content = result["output"]
except Exception as e:
    content = f"{content_type} 생성 실패: {e}"

save(content=content, status="failed")
```

의도는 알겠다. "실패했으니 원인이라도 남겨두자." 근데 그 '남겨두는 자리'가 하필 나중에 JSON으로 파싱되는 필드였다. `status` 컬럼에 `failed` 도 같이 찍혔지만, 읽기 쪽은 `status` 를 보지 않았다. 바로 `content` 를 파싱했다.

그래서 이 한 줄의 실패가 그 레코드에 대한 조회를 **영구적으로** 망가뜨렸다. 재시도로도 안 낫는다. 그 로우가 살아있는 한 API는 계속 500이었다.

## 실패의 진짜 원인은 다른 데 있었다

원래 장애는 외부 LLM API 크레딧이 바닥난 거였다. 근데 그걸 알아채는 데 오래 걸렸다. 예외 처리 코드가 원인을 한 번 더 덮어썼기 때문이다.

```python
# 의도
error_message = e.response["error"]["message"]

# 실제
error_message = e.response["erorr"]["message"]  # KeyError
```

오타. `error` 를 `erorr` 로 썼다. 외부 API가 뱉은 원래 예외는 `KeyError` 에 가려져 로그에도 안 남았고, DB에 저장된 문자열도 원래 원인이 아닌 `KeyError` 였다. 그래서 한동안 "우리 코드 어디서 딕셔너리 접근이 잘못됐지?" 를 보고 있었다. 실제로는 크레딧이 없었을 뿐인데.

에러 메시지가 에러의 원인을 가리는 상황. 이게 시간을 제일 많이 먹었다.

## 진짜 문제

파서를 고쳐도 소용없다. try/except 로 감싸도 근본은 안 낫는다. 문제는 필드 하나에 성격이 다른 두 값이 섞여 들어간 거다.

- 성공했을 때: 구조화된 생성 결과(JSON)
- 실패했을 때: 사람이 읽는 오류 문자열

같은 컬럼에 다른 타입을 넣는 순간, 읽는 쪽은 매번 "지금 이게 뭔지" 를 판별해야 한다. 대체로 그 판별을 안 하고 그냥 파싱한다. 그러다 터진다.

고치는 방향은 간단하다. 자리를 나눈다.

```python
# after
save(
    content=None,           # 결과 없음
    status="failed",
    error_message=str(e),   # 별도 컬럼
)
```

읽기 쪽도 `status` 를 먼저 본다.

```python
if row["status"] == "failed":
    return {"error": row["error_message"]}

plan = json.loads(row["content"])
```

이제 실패 레코드가 조회 API를 폭파시키지 않는다. `content` 는 항상 JSON이거나 NULL 둘 중 하나다.

## 남은 것

DB에 이미 쌓여있던 오염된 로우들은 어떻게 처리할지 아직 안 정했다. 스크립트로 훑어서 `content` 가 JSON으로 파싱 안 되는 것들을 `error_message` 로 옮기고 `content` 를 NULL로 두면 될 것 같은데, 그렇게 마이그레이션하는 게 맞는지 아직 확신이 없다. 저장된 실패 메시지 자체를 폐기해도 되는 건지, 아니면 실패 원인 통계에 쓸 데가 있는 건지부터 물어봐야 한다.

그리고 오타로 원인이 덮이는 패턴 — 이건 개별 케이스 이슈가 아니라 예외 처리 관례를 손봐야 하는 얘기 같다. 예외 안에서 또 예외가 나면 원본이 소실되는 자리가 이 파이프라인에 몇 군데 더 있을 것 같은데, 아직 안 찾아봤다.