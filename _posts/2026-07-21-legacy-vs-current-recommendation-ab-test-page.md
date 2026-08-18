---
title: "추천 엔진을 갈아엎기 전에: 이전 버전과 현행 버전을 한 화면에서 비교하는 법"
date: 2026-07-21 17:00:00 +0900
categories: [Frontend]
tags: [recommendation, fastapi, ab-test, legacy-migration]
---

## "더 나아졌다"를 어떻게 보여주지

추천 로직을 새로 짰다. 코드 상으로는 분명 이전보다 낫다. 축(직업·관심·선호)별로 점수를 다시 잡았고, 전이 행렬(transition matrix, 사용자가 어떤 콘텐츠 다음에 어떤 콘텐츠로 넘어가는지의 통계)을 새로 넣었고, 파일에는 이런 게 붙었다.

```python
# recommend_engine/service.py
@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _transition_raw(tenant_id: str) -> dict:
    pool = _company_pool(tenant_id)
    by_user: dict[str, list[str]] = {}
    for uid, raw_cid, _ in _history_rows(tenant_id):
        if raw_cid in pool:
            by_user.setdefault(uid, []).append(raw_cid)

    outgoing: dict[str, int] = {}
    pair_users: dict[tuple[str, str], set[str]] = {}
    for uid, seq in by_user.items():
        if len(seq) < 2:
            continue
        for i, src in enumerate(seq[:-1]):
            outgoing[src] = outgoing.get(src, 0) + 1
            for dst in seq[i + 1:i + 1 + _TRANSITION_WINDOW]:
                if dst == src:
                    continue
                pair_users.setdefault((src, dst), set()).add(uid)
    # ...
```

그런데 이걸 개발자가 아닌 사람에게 어떻게 설명하지. `_PEER_MIN_SUPPORT = 5` 를 왜 5로 뒀는지, `_TRANSITION_WINDOW = 10` 이 뭘 의미하는지, 그런 얘기를 들으러 온 게 아닌 사람들이다. 회의실에 앉아있는 사람은 지표를 보고 싶은 게 아니라 **"결과가 어떻게 달라지는데?"** 를 보고 싶다.

수치로 답하려다 몇 번 미끄러졌다. 그래서 방향을 바꿨다. 같은 사용자 입력을 이전 엔진과 현행 엔진에 동시에 던지고, 두 결과를 화면에 나란히 붙이는 것. 그림 한 장이 문서 열 페이지보다 강했다.

## 첫 실수 — `if legacy:` 분기

처음엔 현행 라우터 안에 조건분기로 레거시를 끼워 넣었다. "어차피 테스트 끝나면 지울 건데" 라는 마음. 이 마음이 매번 기술 부채가 된다는 걸 알면서도 또 그랬다.

문제는 금방 드러났다. 현행 코드의 pool 구조를 바꿨더니 레거시 결과가 같이 흔들렸다. 두 엔진이 같은 함수를 공유하고 있으니, **비교 대상이 서로 오염되는** 상황. 뭘 비교하고 있는지도 애매해졌다.

라우터를 분리했다. 그리고 원칙을 하나 세웠다 — 레거시는 현행에 **한 줄도 의존하지 않는다.** 공용 유틸이 필요하면 복사한다. 지저분해도 그게 낫다. 비교가 끝나면 폴더 하나를 통째로 지우면 되니까.

```
api/
  routers/
    recommend.py        # 현행
    recommend_legacy.py # 이전 (격리)
  main.py
test_page/
  index.html
  legacy_engine/        # 이전 엔진용 코드 사본
```

## 한 서버에 두 엔드포인트

FastAPI(비동기 Python 웹 프레임워크)에서 `include_router` 로 두 라우터를 붙였다.

```python
# api/main.py
from api.routers import recommend, recommend_legacy

app.include_router(recommend.router, prefix="/recommend")
app.include_router(recommend_legacy.router, prefix="/recommend_legacy")
```

`uvicorn api.main:app --port 8000` 한 줄로 두 엔드포인트가 같은 포트에 뜬다. 프로덕션 빌드에서는 `recommend_legacy` 를 `include_router` 에서 빼기만 하면 끝. 배포 스크립트를 건드릴 필요가 없다.

두 엔진이 같은 데이터 pool을 봐야 한다는 점은 신경을 썼다. 그렇지 않으면 "결과가 다른 게 로직 때문인지 데이터 때문인지" 를 분간할 수 없다. 현행 쪽은 이런 식으로 pool을 캐싱해 두는데,

```python
# recommend_engine/service.py
@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _company_pool(tenant_id: str) -> dict:
    return load_company_pool(tenant_id)

@lru_cache(maxsize=config.POOL_CACHE_SIZE)
def _inst_pool(tenant_id: str) -> dict:
    return {cid: c for cid, c in _company_pool(tenant_id).items()
            if c.get('content_type') == 'INSTITUTION' and c.get('has_meta')}
```

레거시 쪽은 여기 의존하지 않는다. 대신 같은 `load_company_pool(tenant_id)` 를 자기 캐시로 다시 부른다. 함수는 동일하지만 캐시가 분리돼 있어서 서로의 상태를 침범할 수 없다. 데이터는 같지만 실행 컨텍스트는 완전히 나뉜다.

## 비교 화면 — 순서가 왜 다른가

`test_page/index.html` 은 빌드 툴 없이 파일 그대로 열리는 걸 목표로 했다. 이 페이지는 오래 살아남을 게 아니니까.

```js
async function compare(payload) {
  const [current, legacy] = await Promise.all([
    fetch("/recommend",        { method: "POST", body: JSON.stringify(payload) }).then(r => r.json()),
    fetch("/recommend_legacy", { method: "POST", body: JSON.stringify(payload) }).then(r => r.json()),
  ]);
  renderSideBySide(current, legacy);
}
```

`Promise.all` 로 병렬 호출을 묶은 건 두 응답의 렌더 타이밍이 어긋나지 않게 하기 위해서였다. 한쪽만 먼저 그려지면 사람 눈이 그쪽에 먼저 붙어버린다. 편향을 만들지 않으려는 잔손질.

화면에는 세 덩어리를 나란히 그렸다.

- 축별 점수(직업·관심·선호). 이 축은 현행에서 새로 잡은 것이지만, 레거시 응답에도 같은 이름으로 매핑해서 표시
- 상위 10개 결과 목록. 순서 포함. 두 리스트에서 겹치는 항목을 하이라이트
- 겹침 수를 `n / 10` 으로 요약

셋 다 있으니까 예상 못한 질문이 생겼다. **"점수는 비슷한데 왜 순서가 이만큼 다르지?"** 그 자리에선 답을 못 했다. 나중에 파보니 tie-break(동점 처리) 로직이 두 엔진에서 달랐다. 겹침이 7/10 인데 순서가 완전히 뒤바뀌는 케이스가 이 지점에서 나오고 있었다. 비교 UI가 없었으면 못 봤을 문제다.

## 결과를 사람과 보기

이 화면을 몇 번 같이 봤다. 개발자만 있는 자리, 서비스 담당자가 있는 자리, 각각.

개발자와 볼 땐 "왜 이 콘텐츠가 상위에 왔지" 로 얘기가 흘렀다. 콘텐츠의 `keyword_tags`, `ncs_code`, `strand` 같은 메타를 같이 띄워둔 게 도움이 됐다. 서비스 담당자와 볼 땐 반대로, 메타는 접어두고 제목·썸네일만 크게 뒀다. **같은 페이지지만 보는 사람에 따라 시선이 다른 데 갔다.**

이걸 겪고 나서 오른쪽 상단에 토글을 하나 더 붙였다 — 개발자 뷰 / 사용자 뷰. 뒤늦게 붙인 거라 코드가 지저분한데, 아직 안 고쳤다.

## 아직 남은 것

- 비교 로그를 저장 안 하고 있다. 지금은 화면에서 보고 끝난다. "이 케이스 이상하다" 를 발견해도 재현하려면 입력을 손으로 다시 넣어야 한다. 클립보드 복사 정도라도 붙여야 한다
- 축별 점수를 나란히 놓긴 했는데, **가중치가 어떻게 바뀌었는지**는 화면에서 안 보인다. 현행에서는 축 간 가중치가 새로 잡혔지만, 그 판단 근거를 이 페이지가 설명해주진 않는다
- `_PEER_MIN_SUPPORT = 5` 같은 튜닝 파라미터를 화면에서 바꿔볼 수 있게 하고 싶었는데 아직 안 붙였다. 붙이는 순간 캐시 무효화가 얽혀서 미뤘다

엔진을 바꾸기 전에 비교 화면부터 만들자, 라는 문장으로 마무리를 지으려다 지웠다. 이 화면을 만들고 나서야 오히려 엔진의 어떤 부분을 아직 안 잡았는지가 보였기 때문이다. 비교 UI는 결론을 내주는 도구가 아니라, 다음 질문을 던지는 도구에 가깝다.