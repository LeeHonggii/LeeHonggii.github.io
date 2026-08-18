---
title: "JSON 컬럼에 실패 메시지를 넣으면 상태 API가 영구히 죽는다"
date: 2026-08-17 17:00:00 +0900
categories: [Backend]
tags: [postgres, json, error-handling, schema-design]
---

조회 API가 500을 뱉기 시작했다. 새로 배포한 것도 없고, 트래픽이 튄 것도 아니었다. 특정 콘텐츠 하나를 조회할 때만 죽었다. 그 콘텐츠는 하필 얼마 전 생성에 실패한 놈이었다.

## 왜 조회가 죽지

생성이 실패한 건 이해가 간다. 외부 LLM API 호출이 타임아웃이든 rate limit이든 걸릴 수 있으니까. 그런데 왜 그 뒤로 **조회**가 계속 실패하지. 생성은 한 번의 이벤트고 지나갔는데, 조회는 새 요청이다. 두 경로는 별개여야 정상이다.

스택을 열었다.

```
JSONDecodeError: Expecting value: line 1 column 1 (char 0)
  at json.loads(row["content"])
```

`json.loads` 가 첫 글자에서부터 실패했다는 건, 넘긴 문자열이 애초에 JSON이 아니라는 뜻이다. 코드는 멀쩡했다. 값이 이상했다.

## DB를 직접 본다

ORM을 통해 보면 이미 파싱된 값 아니면 예외다. 원본이 뭔지 보려면 raw로 봐야 한다.

```sql
SELECT content_type, status, LEFT(content, 80) AS head
FROM content_meta
WHERE item_id = :item_id;
```

돌려보니 `content` 컬럼에 이런 게 들어 있었다.

```
generation_plan 생성 실패: upstream API error ...
```

JSON이 있어야 할 자리에 한국어 평문 에러 메시지. 그것도 status는 `failed`. 즉 생성 파이프라인이 실패했을 때 예외 메시지를 그대로 결과 컬럼에 밀어 넣은 거다. 아마 로그 대신 이걸 봐두면 편할 것 같아서, 아니면 급하게 어딘가 저장은 해야 하니까.

문제는 **읽는 쪽 코드는 그런 규약을 모른다**는 것.

```python
gen_plan = json.loads(row["content"])
```

status가 뭐든 일단 파싱한다. 성공한 케이스만 테스트했으니 지금까지 문제가 없었을 뿐이다. 실패 row가 처음 조회되는 순간 500이 시작되고, 그 row가 지워지거나 status 로직이 바뀌기 전까지 **영구히** 500이다. 재시도로도 안 풀린다. 값을 안 고치면 안 풀린다.

## 한 필드가 두 얼굴을 가지면

이건 파싱 버그가 아니라 **스키마 계약 위반**이다. `content` 컬럼이 두 가지 의미를 동시에 가졌다.

- `status = success` → JSON 문자열
- `status = failed` → 사람이 읽는 평문 메시지

status로 분기해서 읽으라는 암묵적 규약이 있는 셈인데, 그 규약이 코드 어디에도 안 적혀 있다. 새로 조인한 사람이 이 컬럼을 보면 그냥 JSON 컬럼으로 안다. 나도 그랬다.

당장 급한 픽스는 읽는 쪽에서 status를 먼저 보는 것.

```python
if row["status"] == "success" and row["content"]:
    payload = json.loads(row["content"])
else:
    payload = None
```

이걸로 500은 멎는다. 하지만 근본은 안 고쳐졌다. 실패 메시지가 여전히 JSON이어야 할 자리에 앉아 있고, 언젠가 누군가 status 체크를 빼먹으면 같은 일이 또 난다.

그래서 저장 쪽을 손 봐야 한다. 실패 메시지는 별도 컬럼으로 뺀다. `content` 는 **JSON 아니면 NULL** 이라는 단일 계약을 지킨다.

```sql
ALTER TABLE content_meta ADD COLUMN error_message TEXT;
```

파이프라인 실패 핸들러도 이쪽에 쓰도록 바꾼다. `content` 는 건드리지 않고 NULL로 남긴다. 그러면 읽는 쪽에서 status를 잊어도 최악이 `None` 이지 예외가 아니다.

## 아직 못 한 것

기존에 이미 `content` 에 평문이 들어가 있는 row가 얼마나 있는지 세지 않았다. 프로덕션 데이터를 아직 안 훑었다. 마이그레이션 스크립트로 `status = failed` 인 행의 `content` 를 `error_message` 로 옮기고 `content` 는 NULL 처리해야 하는데, 이건 다음 주 배포 창에서 잡을 예정이다.

그리고 하나 더 남은 질문 — 애초에 이걸 왜 못 잡았나. 로컬에서는 실패 케이스를 재현하기가 귀찮아서 성공 경로만 테스트했다. 실패 케이스를 픽스처로 넣어두는 습관을 만들어야 한다. 실패는 예외가 아니라 상태다.

읽기 시점에 터지는 장애는 대개 쓰기 시점의 계약 위반에서 온다. 이번 건도 그랬다. 30초짜리 `SELECT` 한 줄이 그걸 알려줬다.