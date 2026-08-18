---
title: "OpenAI 크레딧 소진이 왜 KeyError('plan')로 보였나"
date: 2026-08-17 17:00:00 +0900
categories: [AI / LLM]
tags: [openai, debugging, exception-handling, llm-ops]
---

## 500이 처음 눈에 들어온 순간

어떤 백엔드 서비스의 생성 엔드포인트가 갑자기 500을 뱉기 시작했다. 배포 직후도 아니고, 스키마가 바뀐 것도 아니다. 로그를 열었다.

```
Traceback (most recent call last):
  ...
KeyError: 'plan'
```

`KeyError`. 딕셔너리에서 없는 키를 꺼냈다는 뜻이다. 그럼 그걸 고치면 되는 문제 아닌가 — 라고 생각한 게 이 삽질의 시작이었다.

## 첫 번째 함정: 눈에 보이는 오타

코드를 열어보니 실제로 키 이름이 틀려 있었다. 실패 메시지를 조립하는 자리였다.

```python
# 문제의 라인
fail_msg = GENERATORS['plan']['description'][lang]['fail']

# GENERATORS 상수의 실제 키
GENERATORS = {
    'generation_plan': {
        'description': {
            'ko': {'fail': '...'},
            'en': {'fail': '...'},
        },
    },
    # ...
}
```

`'plan'` 이 아니라 `'generation_plan'` 이었다. 누군가 리팩터하면서 상수 키만 바꾸고 참조 한 곳을 놓친 것 같았다. 고쳤다. 다시 배포. 여전히 500.

이 시점에서 이미 이상하다는 걸 눈치챘어야 했다. `KeyError` 는 결정론적 버그다. 코드 경로가 같으면 매번 같은 자리에서 터진다. 그런데 나는 "오타 고쳤으니 됐겠지" 로 넘어갔다. 왜 그 라인까지 흘러갔는지는 안 물었다.

## 두 번째 함정: `GET /v1/models` 가 200이면 살아있다는 착각

키를 고친 뒤 이번엔 다른 에러가 스택 트레이스에 올라왔다. OpenAI 호출 자체가 실패하고 있었다. 반사적으로 키가 만료됐나 싶어 curl 을 던졌다.

```bash
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

`200 OK`. 모델 목록이 멀쩡히 내려왔다. 여기서 나는 "키는 유효하다" 로 결론을 내리고 방향을 틀었다. env 파일, 컨테이너 환경변수, 프록시 설정 — 다 뒤졌다.

시간을 꽤 쓴 뒤에야 최소 추론 호출을 직접 때려봤다. `gpt-4o-mini` 로 "hi" 한 줄. 응답은 이랬다.

```json
{
  "error": {
    "type": "insufficient_quota",
    "code": "insufficient_quota",
    "message": "You exceeded your current quota, please check your plan and billing details."
  }
}
```

크레딧 잔액이 0이었다. 카드 갱신 문제였는지 자동 충전이 실패했는지는 별개의 이야기고, 요점은 이거다 — `GET /v1/models` 는 키의 유효성만 본다. 추론 쿼터가 남았는지는 알려주지 않는다. 두 엔드포인트는 과금·권한이 분리돼 있다.

나는 "키 살아있음 == API 쓸 수 있음" 이라는 등식을 무의식적으로 갖고 있었다. curl 200 을 본 순간 그 가정은 확인 편향으로 굳어졌고, 방향이 30분쯤 어긋났다.

## 진짜 원인이 KeyError 뒤에 숨은 이유

이제 처음의 `KeyError: 'plan'` 이 왜 그렇게 보였는지가 풀린다. 호출 흐름을 되짚어보면 이렇다.

1. OpenAI 추론 호출 → `RateLimitError` (SDK가 quota 초과를 이 예외로 던진다)
2. `except` 로 잡아서 사용자에게 돌려줄 실패 메시지를 만들려고 시도
3. 그 메시지 조립 코드 안에 `GENERATORS['plan']...` 오타
4. 원래 예외 위로 새 `KeyError` 가 덮여 씀

이중 장애(double fault)다. 진짜 원인인 `RateLimitError` 는 로그 어디에도 남지 않았다. 왜냐면 `except` 블록이 원래 예외를 붙잡은 채 새 예외를 던졌고, 그 새 예외에 원인 체인이 걸려있지 않았기 때문이다.

파이썬에서 예외 체인은 `raise ... from ...` 으로 명시해야 한다.

```python
try:
    result = openai_client.chat.completions.create(...)
except Exception as last_error:
    fail_msg = GENERATORS['generation_plan']['description'][lang]['fail']
    raise RuntimeError(fail_msg) from last_error
```

`from last_error` 를 빼면 (또는 `except` 안에서 예외가 발생하면) 트레이스백은 마지막 예외만 보여준다. 파이썬이 `During handling of the above exception, another exception occurred` 로 원 예외를 보여주긴 하지만, 그건 예외 처리 블록이 정상적으로 새 예외를 raise 했을 때 얘기다. 처리 블록 자체에서 `KeyError` 처럼 예상 못 한 예외가 터지면, 그게 그냥 최종 예외가 되어 위로 올라간다.

즉 이번 사고는 두 개의 버그가 겹쳐야만 발생하는 사고였다. 하나는 크레딧 소진, 다른 하나는 에러 처리 경로의 오타. 둘 중 하나라도 없었으면 정상 로그가 남았을 것이다.

## 남은 것

바로 고친 건 두 가지다. 실패 메시지 조립 코드에서 예외 체인을 항상 붙이도록 바꿨고, 배포 파이프라인의 헬스체크에 `chat.completions` 최소 호출을 넣었다 (하루 몇 번 도는 정도라 비용은 무시할 수준이다).

아직 안 한 게 더 있다. 에러 처리 경로 자체의 단위 테스트가 없다 — 실패 시나리오를 강제로 만들어서 실패 메시지가 정상적으로 조립되는지 확인하는 테스트. 이번처럼 오타가 숨어 있어도 평소엔 아무도 모른다. 다음 주에 손댈 것 같은데, 프로덕션에서 유사한 이중 장애가 다른 예외 처리 블록에도 숨어있을지가 더 신경 쓰인다. `grep` 으로 `except.*:` 다음에 딕셔너리 접근이 오는 패턴을 훑어봐야 할 것 같다.

한 가지 더 애매하게 남은 판단 — OpenAI 쿼터 소진 자체를 우리 쪽에서 사전에 감지할 방법이 없다. Billing API 로 잔액을 폴링하는 방법이 있긴 한데, 그건 그것대로 별도의 권한이 필요하고, 결국 "테스트 호출을 주기적으로 던진다" 랑 비슷한 비용이 든다. 이걸 어느 쪽으로 잡을지는 아직 결정 못 했다.