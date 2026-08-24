---
title: "JSON 파싱 에러인 줄 알았는데, 진짜 문제는 실패 메시지를 JSON 칼럼에 넣은 것이었다"
date: 2026-08-24 17:00:00 +0900
categories: [Backend]
tags: [debugging, python, json, generative-ai, error-handling]
---

상태 조회 API가 500을 돌려줬다. 처음엔 그냥 DB 연결 문제겠거니 했다.

스택 트레이스를 보니 `json.loads`였다.

```python
json.loads(gen_plan)
```

`gen_plan` 칼럼을 읽어서 파싱하는 부분. 파싱 에러라면 칼럼 값이 깨진 JSON이라는 뜻이다. 실제 값을 꺼내봤다.

```
콘텐츠 생성 실패: Insufficient quota
```

JSON이 아니었다. 평문이었다. 생성에 실패했을 때 에러 메시지 문자열을 그대로 JSON 칼럼에 넣고 있었던 것이다.

---

생성 실패 경로를 따라 올라갔다. 실패하면 이렇게 저장하고 있었다.

```python
content = f'{self.content_type} 생성 실패: {error_message}'
```

`content_type`이 뭐가 됐든 실패 메시지를 f-string으로 만들어서 그 칼럼에 박는다. 나중에 상태를 조회할 때 이 칼럼을 무조건 `json.loads`로 읽으니, 생성이 한 번이라도 실패한 레코드에 접근하면 상태 API 자체가 터진다. 2차 장애다. 원래 에러보다 더 넓은 범위를 다운시키는 에러.

생성이 왜 실패했는지도 봐야 했다. `Insufficient quota`라고 되어 있으니 OpenAI API 호출 문제다. 키 자체는 살아있었다. 만료된 게 아니라 크레딧이 바닥난 상태였다. 키가 유효하다고 호출이 되는 게 아니라는 걸 다시 한 번 확인했다.

그리고 이 환경엔 하드코딩된 API 키가 19군데 있었다.

---

거기서 추가로 하나 더 나왔다. 갤러리 생성이 아예 시도조차 안 되고 있었다. 이미지 검색에서 실패하는 게 아니라 검색 자체가 호출되지 않는 문제였다.

원인을 찾다가 이런 흐름을 만났다.

```python
raise Exception(GENERATORS['plan']['description'][self.user_lang]['fail'])
```

`GENERATORS` 딕셔너리(생성기 타입별 설정을 담은 구조)에서 `'plan'` 키로 접근하는데, 어느 시점에 이 키가 `'generation_plan'`으로 바뀌어 있었다.

```
GENERATORS['plan'] → GENERATORS['generation_plan']
```

`KeyError`가 나야 하는데 왜 잡혔냐. 위에서 `except Exception`으로 잡아서 다른 메시지로 다시 던지고 있었다. 원래 에러가 이 예외 처리에 덮여서 `KeyError`가 보이지 않게 됐다. 갤러리 생성 실패 원인이 이미지 검색인 줄 알고 그쪽을 들여다봤다가 시간을 낭비했다.

---

고친 것들을 정리하면:

하드코딩된 API 키 19곳을 환경 변수 참조로 교체했다. 키를 코드에 박지 않는 건 기본인데 이게 한두 군데가 아니라 열아홉 곳이었다는 게 더 문제였다. develop과 main에 각각 커밋했다.

`GENERATORS` 키 불일치는 수정했다. 그리고 `except Exception`이 원인을 삼키지 않도록 예외 처리를 정리했다.

상태 조회 API의 `json.loads` 호출은 아직 안전 처리가 되지 않았다. 실패 레코드에 접근하면 여전히 터진다. 칼럼 설계 문제이기도 해서 — 실패 메시지를 JSON 칼럼에 저장하지 않거나, 상태 필드를 따로 두거나 — 어떻게 고칠지는 더 봐야 한다.