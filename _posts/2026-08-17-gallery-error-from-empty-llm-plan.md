---
title: "이미지 검색 장애인 줄 알았는데, 사실은 빈 LLM 플랜이었다"
date: 2026-08-17 17:00:00 +0900
categories: [AI / LLM]
tags: [llm, debugging, pipeline]
---

갤러리 화면이 통째로 비었다는 리포트를 받았다. 로그를 열자 이미지 검색 쪽에서 뭔가 잔뜩 튀어나와 있었다. 그 순간의 판단은 자연스럽게 "이미지 검색 API가 죽었나 보다"였다. 한 시간 정도 그 방향으로 뒤졌다. 틀렸다.

## 처음에 붙잡은 실

문제의 코드는 대충 이렇게 생겼다.

```python
for sub in self.subheadings:
    search_images(sub, ...)
```

`subheadings`는 LLM(대규모 언어 모델)이 만들어준 소제목 리스트다. 이걸 검색어 삼아 이미지 API를 때린다. 반복문은 돌긴 도는데 결과가 안 쌓였다. `search_images` 안쪽에 로그를 더 박아볼까 하다가, 습관적으로 DB에 저장된 플랜을 먼저 열었다. 눈에 들어온 건 이거였다.

```json
{
  "list_of_subheadings": [],
  "input_document_check": false,
  "input_user_requests_check": false
}
```

## 방향이 뒤집힌 지점

`list_of_subheadings`가 빈 배열이었다. 이미지 검색이 실패한 게 아니라 **검색할 소제목 자체가 처음부터 없었다.** for 루프는 0번 돌고 조용히 넘어갔을 뿐이다. 진짜 신호는 그 옆의 두 필드에 있었다.

- `input_document_check` — 입력 문서가 플랜을 짤 만큼 정보를 담고 있는지
- `input_user_requests_check` — 사용자 요청이 구체적인지

둘 다 `false`. LLM은 "이 입력으로는 못 짜겠다"고 정확히 답을 돌려주고 있었다. 문제는 그 답을 아무도 읽지 않았다는 것.

## 아무도 안 읽던 필드

플랜을 만들어 넘기는 쪽 코드는 대략 이런 모양이었다.

```python
plan = llm.generate_plan(user_input, document)
self.subheadings = plan.get("list_of_subheadings", [])
```

`get`에 기본값 `[]`까지 얌전히 걸어둔 게 오히려 함정이었다. LLM이 실패 신호를 아무리 정성껏 채워 넣어도, 이 코드에서는 그냥 빈 리스트 → 조용한 무반응 → 하류(downstream) 갤러리가 텅 빔. 스택트레이스도 안 남는다. 그러니 나는 엉뚱한 곳에서 한 시간을 쓴 거였다.

고친 방향은 별거 없다. 두 검증 필드를 실제로 소비하고, 실패면 그 자리에서 끊는다.

```python
result = llm.generate_plan(user_input, document)

if not result.get("input_user_requests_check") \
   and not result.get("input_document_check"):
    raise InputNeedsMoreDetailError(
        "플랜 생성 실패: 입력 정보가 부족함"
    )

self.subheadings = result.get("list_of_subheadings", [])
```

`InputNeedsMoreDetailError`는 사용자에게 "요청을 좀 더 구체적으로 달라"고 되돌려주기 위한 예외다. 이걸 던지면 이미지 검색이든 갤러리든 애초에 실행되지 않고, 사용자는 "이미지가 왜 안 나오지"가 아니라 "아 내가 뭘 더 써야 하는구나"를 본다. 실패의 위치가 이동한다.

## 남는 찜찜함

솔직히 이 수정은 사후약방문 성격이 짙다. 두 필드를 `and`로 묶은 것도 이번 케이스가 마침 둘 다 `false`였기 때문에 이렇게 붙였을 뿐, 한쪽만 `false`일 때는 어떻게 다뤄야 하는지 아직 정하지 않았다. 문서는 충분한데 요청이 모호한 경우와, 요청은 명확한데 문서가 부실한 경우는 사용자에게 되돌려줄 메시지가 달라야 할 것 같은데 — 그 갈래를 아직 안 짰다.

더 근본적으로는, LLM 응답에 이런 "메타 필드"가 늘어날 때마다 호출부에서 하나씩 손으로 소비해줘야 한다는 구조 자체가 다음번에 같은 실수를 부를 거라는 예감이 있다. 스키마 레벨에서 "이 필드 중 하나라도 false면 결과를 쓰지 마라"가 강제되는 편이 안전할 텐데, 아직 그렇게 못 옮겼다.

에러 스택이 가리키는 곳은 대체로 마지막에 부러진 곳이지, 처음 금이 간 곳이 아니다. 이번엔 운 좋게 DB 스냅샷을 먼저 열어서 30분 만에 방향을 틀었다. 운이 없었으면 `search_images` 내부에 print를 꽂고 있었을 거다. 다음번엔 좀 더 위에서부터 열어보는 습관을 들여야겠다.