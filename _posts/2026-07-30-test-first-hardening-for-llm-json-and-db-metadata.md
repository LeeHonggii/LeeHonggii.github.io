---
title: "LLM 응답과 DB 메타데이터를 믿었을 때 터지는 것들"
date: 2026-07-30 17:00:00 +0900
categories: [Backend]
tags: [python, fastapi, llm, defensive-coding]
---

리뷰에서 한 줄이 들어왔다. "이 필터, 데이터 이상하면 안 터져요?"

그 자리에선 "고치면 되죠"라고 답했는데, 자리로 돌아와 코드에 손을 얹는 순간 멈췄다. **정확히 어떤 입력에서 터지는지**를 머릿속으로 못 그렸다. 고치려면 먼저 터뜨려야 한다.

이 글은 그날 이후 내가 LLM(거대 언어 모델) 응답과 DB(데이터베이스) JSON 필드를 다룰 때 붙이기 시작한 검사 몇 개에 대한 이야기다. 대단한 원칙은 아니고, 로그에서 실제로 본 이상한 값들을 코드가 알아서 걸러내게 만드는 짧은 습관이다.

## "잘 오겠지"가 무너지는 지점

LLM에게 JSON을 달라고 하면 대부분 잘 온다. 이게 문제다. 대부분 잘 오니까 자연스럽게 이렇게 쓴다.

```python
parsed = json.loads(text)
value = parsed.get("field", default)
```

돌려보면 잘 돈다. 몇백 번, 몇천 번. 그러다 어느 요청에서 LLM이 최상위를 배열로 뱉는다. `json.loads` 는 성공한다 — 파싱 자체는 문제가 없다. 문제는 그 다음 줄이다. `list` 에는 `.get()` 이 없다. `AttributeError` 가 나고, 아래에 준비해뒀던 fallback 로직에는 **도달조차 못 한 채** 500이 나간다.

fallback 이 있다는 게 오히려 함정이었다. 방어를 해뒀다고 생각했는데, 방어선 앞에서 크래시가 나고 있었다.

```python
def parse_json_response(text: str) -> dict:
    parsed = json.loads(text)
    if not isinstance(parsed, dict):
        return {}
    return parsed
```

한 줄이다. `isinstance` 검사를 fallback 앞에 두는 것. 어느 쪽으로 처리할지 — 빈 dict 로 조용히 넘길지, `ValueError` 를 올려서 상위 핸들러가 잡게 할지 — 는 호출부 성격에 달렸지만, 적어도 "타입이 다른데 그대로 밀고 들어가는" 상태는 사라진다.

## DB 안에 섞여 있는 이물질

LLM 쪽 문제만 있으면 그래도 낫다. `keywords`, `tags`, `aliases` 같이 JSON 배열로 저장된 필드를 순회할 때 더 조용하게 터진다.

```python
matches = [r for r in recs if query in r["tag"].lower()]
```

이 코드는 `r["tag"]` 가 항상 문자열이라는 가정 위에 서 있다. 그런데 그 배열은 언제 쌓였는지 모른다. 초기 마이그레이션 때 `None` 이 들어간 행이 남아있을 수도 있고, 한때 정수 태그 ID를 넣던 코드가 있었을 수도 있다. `r["tag"]` 가 `None` 이면 `AttributeError`, 정수면 또 `AttributeError`.

LLM 쪽과 결정적으로 다른 점: **로그가 안 남는 경우가 많다.** list comprehension 안에서 예외가 나면 요청 전체가 500으로 끝나기도 하지만, 상위에서 try/except 로 넓게 감싼 코드였다면 `matches` 는 그냥 빈 리스트가 된 채로 응답이 나간다. 추천 결과가 조용히 사라진다. 사용자가 "추천이 좀 이상한데요" 라고 말할 때까지 아무도 모른다.

```python
matches = [
    r for r in recs
    if isinstance(r.get("tag"), str) and query in r["tag"].lower()
]
```

`isinstance` 를 먼저 통과시키면 이물질은 매칭 전에 걸러진다. 데이터가 깨끗하다는 믿음을 코드 밖에 두지 않고 안으로 가져오는 것이다.

## FastAPI 예외 핸들러 — 마지막 방어선

개별 함수의 방어 검사와는 별개로, 서버 전체 레벨에서 "여기까지 새면 어떻게 나가는가"를 정해둬야 한다. 지금 붙여둔 건 세 층이다.

```python
@app.exception_handler(RequestValidationError)
async def _validation_error(request: Request, exc: RequestValidationError):
    parts = []
    for e in exc.errors():
        loc = '.'.join(str(x) for x in e.get('loc', []) if x != 'body')
        parts.append(f'{loc}: {e.get("msg")}' if loc else e.get('msg', ''))
    detail = '요청 형식 오류 — ' + '; '.join(p for p in parts if p)
    logger.info('검증 오류 path=%s | %s', request.url.path, detail)
    return JSONResponse(status_code=422, content={'detail': detail})


@app.exception_handler(pymysql.Error)
async def _db_error(request: Request, exc: pymysql.Error):
    logger.error('DB 오류 path=%s err=%s', request.url.path, exc)
    return JSONResponse(status_code=503, content={'detail': '데이터 조회 일시 오류입니다. 잠시 후 다시 시도해 주세요.'})


@app.exception_handler(Exception)
async def _unhandled(request: Request, exc: Exception):
    logger.error('미처리 오류 path=%s\n%s', request.url.path, traceback.format_exc())
    return JSONResponse(status_code=500, content={'detail': '내부 처리 오류입니다.'})
```

세 개의 순서와 상태 코드에 나름의 이유가 있다.

- `RequestValidationError` 는 422다. 파라미터 위치(`loc`)에서 `body` 를 지우고 나머지만 이어붙여, 클라이언트가 "어느 필드가 왜 틀렸다"까지 바로 읽게 만든다. `logger.info` 로 남기는 건 이게 서버 잘못이 아니기 때문 — error 레벨이면 알림이 울린다.
- `pymysql.Error` 는 503이다. 500이 아니라. DB 커넥션이 순간적으로 끊기는 건 애플리케이션 결함이 아니라 일시적 가용성 문제이고, 클라이언트가 재시도해도 되는 상태다. 500 과 섞어두면 재시도 정책을 구분하기 어렵다.
- 마지막 `Exception` 은 traceback 을 통째로 남긴다. 앞의 둘에서 못 잡힌 게 여기 오면 그건 진짜 우리 잘못이다.

내부 상세를 응답 본문에 넣지 않는 것도 규칙이다. `str(exc)` 를 그대로 내보내면 스키마 이름이나 SQL 조각이 노출된다. 로그에는 남기고, 응답은 담백하게.

## 라우터에서 케이스 나누기

핸들러가 아래에서 받아주는 걸 알면, 라우터 위쪽은 "의미 있는 실패"를 만드는 데 집중할 수 있다. 추천 엔드포인트는 이렇게 갈래를 나눈다.

```python
def _run(req: RecoRequest) -> dict:
    if not service.company_exists(req.tenant_id):
        raise HTTPException(
            status_code=404,
            detail=f"추천 가능한 콘텐츠가 없습니다(미등록 회사이거나 콘텐츠 0건): {req.tenant_id}",
        )

    payload = req.preeval_payload()
    code_only = payload is None and _has_preeval_fields(req)

    try:
        result = service.recommend(req.tenant_id, payload, req.history_items(),
                                   user_id=req.user_id, debug=req.debug)
    except ValueError as e:
        detail = "사전평가가 라벨 없이 코드만 수신되어 처리할 수 없습니다(label 필요)" if code_only else str(e)
        raise HTTPException(status_code=422, detail=detail)

    diag = result.get('diagnostics', {})
    if code_only:
        diag.setdefault('warnings', []).append('사전평가 입력이 라벨 없이 코드만 수신되어 무시됨(label 필요)')
        diag['preeval_applied'] = False
```

여기서 붙인 규칙 두 개:

**하나.** 회사가 등록 안 되어 있으면 500이 아니라 404다. "우리 시스템이 고장난" 게 아니라 "요청한 리소스가 없다"는 뜻이기 때문에. 클라이언트 로그 추적할 때 이 구분이 큰 차이를 만든다.

**둘.** "코드만 있고 라벨은 없는" 사전평가 입력을 감지해서 `code_only` 플래그로 들고 다닌다. 엔진이 `ValueError` 를 뱉었을 때, 그게 이 케이스 때문인지 다른 이유인지에 따라 에러 메시지가 달라진다. 같은 422라도 클라이언트가 취해야 할 행동이 다르니까 — 라벨을 붙여 재요청할지, 페이로드 형식을 다시 봐야 할지.

성공 경로에서도 `code_only` 는 살아남는다. 요청은 통과시키되 `diagnostics.warnings` 에 "라벨 없어서 무시했음"을 넣고 `preeval_applied=False` 로 표시한다. 조용히 무시하는 건 위험하다. 표시하면서 무시해야 나중에 응답을 보고 원인을 알 수 있다.

## 고치는 순서를 바꾸는 것

한 가지 습관을 마지막으로 남기고 싶다. 이제는 리뷰에서 "이거 위험한 거 아니에요?" 를 들으면, 코드부터 열지 않는다. 테스트부터 연다.

지적받은 상황이 정말 재현되는지 실패하는 테스트를 먼저 쓴다. 그게 빨간불로 뜨는 걸 확인하고 나서야 코드를 고친다. 이 순서가 아니면 "고쳤는데 뭘 고친 건지 모르는" 상태가 되기 쉽다. 재현 안 되는 버그를 고치는 건 그냥 코드를 만지는 것이지 수정이 아니다.

이 순서를 지키면 부수 효과가 하나 더 붙는다. 실패 테스트가 그대로 남아 회귀(regression, 고쳤던 버그가 다시 들어오는 것) 방어선이 된다. 몇 달 뒤 다른 사람이 이 함수를 리팩터링하다가 방어 검사를 지우면, CI 가 대신 소리를 지른다. `isinstance` 한 줄을 왜 그렇게 붙였는지 아무도 기억 못 해도, 테스트 이름이 대신 기억해준다.

---

아직 안 한 게 있다. 지금은 배열 안 비문자열을 **읽는 쪽에서** 걸러내고 있는데, 이건 데이터가 계속 더러운 채로 남는다는 뜻이다. 쓰기 경로에서 검증을 어디까지 걸지 — 스키마 레벨에서 막을지, 애플리케이션에서 정규화할지 — 아직 결론을 못 냈다. 다음에 이 파일을 열 때 볼 것.