---
title: "추천 요청을 한 갈래로 몰면 안 되는 이유: 입력 모양이 다르면 경로도 달라야 한다"
date: 2026-07-27 17:00:00 +0900
categories: [Backend]
tags: [recommendation, api-design, fastapi, routing]
---

추천 API를 처음 열었을 때 나는 요청 하나에 스코어러 하나면 된다고 생각했다. 사전평가(회원가입 직후 받는 성향/직무 설문)든 학습 이력이든 다 넣어주면 뒤에서 알아서 섞어 점수 매길 거라고. 그런데 실제로 트래픽을 받아보니 요청이 이렇게 온다.

- 사전평가만 있는 신규 사용자 — 이력이 없다
- 이력만 있는 재방문자 — 사전평가는 오래전에 지나갔다
- 둘 다 있는 활성 사용자
- 그리고 **사전평가를 코드로만 보낸 요청** — `{"code": 9}` 만 있고 `label` 이 없다

마지막이 가장 골치였다. 코드는 tenant마다 매핑이 달라서 라벨 없이 코드만으로는 의미를 확정할 수 없는데, 상위 서비스는 그걸 모르고 코드만 보내고 있었다. 이 요청을 그대로 스코어러에 넣으면 어떻게 될까. 스코어러는 라벨을 못 찾으니 "그냥 이력만 쓴 것처럼" 조용히 동작한다. 그리고 응답에는 `mode: "preeval+history"` 가 찍힌다. **실제로는 사전평가가 반영되지 않았는데** 반영된 것처럼 보이는 응답이 나가는 것이다.

## 스코어러 앞에 판별을 두기로 한 이유

문제는 "사전평가가 있냐"가 boolean 하나로 안 떨어진다는 것이었다. 세 가지 상태가 섞여 있다.

- 아예 없음 → 이력만으로 가야 함
- 있고 라벨도 있음 → 정상 병합
- 있는데 라벨이 없음 (코드만) → **거절하거나, 무시하고 경고를 남기거나**

이걸 스코어러 안에서 판단하게 두면 세 갈래가 스코어링 로직에 뒤엉킨다. 그래서 라우팅을 라우터 층으로 끌어올렸다. 요청이 들어오면 스코어러를 부르기 **전에** 어떤 상태인지 결정하고, 그 결정을 `diagnostics` 에 박아서 응답으로 되돌려준다.

```python
def _has_preeval_fields(req: RecoRequest) -> bool:
    return any([req.job, req.sub_job, req.career, req.learner_type, req.topics])


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
```

`payload is None and _has_preeval_fields(req)` — 이 한 줄이 앞에서 말한 세 번째 상태를 잡아낸다. 필드는 왔는데 라벨이 없어서 payload가 None이 된 경우다. `preeval_payload()` 는 라벨만 살아남게 만들어져 있다.

```python
def preeval_payload(self) -> dict | None:
    prefs = [t.label for t in self.topics if t and t.label]
    job = _label(self.job)
    if not (job or prefs):
        return None
    return {
        'job': job,
        'sub': _label(self.sub_job),
        'career': _label(self.career),
        'prefs': prefs,
        'character': _label(self.learner_type),
        'scores': self.scores,
    }

def _label(x: CodeLabel | None) -> str | None:
    return x.label if x and x.label else None
```

여기서 중요한 건 `job or prefs` 로 최소 조건을 검사한 뒤 아니면 그냥 `None` 을 돌려준다는 점이다. 스코어러에게는 "사전평가는 없다"고 알리고, 라우터 단에서 별도로 "코드만 왔는지"를 다시 판정한다. 이 분리 덕분에 스코어러 코드에 "라벨 없는 코드"라는 개념 자체가 들어가지 않는다. 스코어러는 라벨 있는 payload와 이력만 안다.

## 거절할까, 무시할까

처음엔 422를 던지려 했다. 잘못된 요청이니까. 그런데 상위 서비스가 마이그레이션 중이었고, 코드만 보내는 클라이언트가 아직 남아 있었다. 여기서 422를 뱉으면 그 클라이언트는 통째로 추천이 안 나온다.

타협은 이렇게 됐다. **payload가 None인 상태로 스코어러에 넘긴다 (= 이력만으로 돌린다). 대신 diagnostics에 "무시됨" 경고를 명시적으로 박는다.**

```python
diag = result.get('diagnostics', {})
if code_only:
    diag.setdefault('warnings', []).append('사전평가 입력이 라벨 없이 코드만 수신되어 무시됨(label 필요)')
    diag['preeval_applied'] = False
```

`preeval_applied` 필드를 강제로 False로 덮는 부분이 핵심이다. 스코어러 내부 diagnostics는 payload를 못 봤으니 "사전평가 안 씀" 상태로 나오긴 하지만, 그건 "원래 없어서"인지 "있었는데 무시했는지" 구분이 안 된다. 라우터가 아는 정보(`code_only`)를 diagnostics에 다시 덮어써서, 응답을 받는 쪽이 "아, 우리가 라벨 없이 보냈구나"를 볼 수 있게 만들었다.

422를 던지는 길은 `ValueError` 로 남겨뒀다. 스코어러가 자체적으로 "이 조합은 처리 불가"라고 판단하는 경우가 있는데, 그때는 `code_only` 여부에 따라 에러 메시지를 다르게 보여준다. 같은 422여도 원인이 다르면 클라이언트가 다르게 대처해야 하니까.

## mode 필드는 스코어러가 결정하고, 라우터는 손대지 않는다

라우팅을 라우터로 올렸으면 `mode` 도 라우터에서 결정하고 싶어질 수 있다. `payload` 유무, 이력 개수 보고 `preeval+history` / `history_only` / `preeval_only` 로 나누면 되지 않나. 실제로 처음엔 그렇게 짰다.

그런데 스코어러 안에서도 fallback이 있다. 이력이 있다고 넘겼는데 tenant 콘텐츠 풀에서 하나도 매칭이 안 되면 스코어러 스스로 "이력 없음"으로 축퇴한다. 이 결정을 라우터가 미리 못 한다. 그래서 mode는 스코어러가 정하게 두고, 라우터는 그 결과를 그대로 전달만 한다.

```python
out = {
    'tenant_id': req.tenant_id,
    'user_id': req.user_id,
    'mode': result['mode'],
    'axes': result['axes'],
    'diagnostics': diag,
    '_levels': result.get('_levels', []),
}
```

책임 경계로 다시 쓰면 이렇다. **라우터는 "무엇을 넘길지"를 결정한다. 스코어러는 "받은 걸로 무엇을 할지"를 결정한다.** 그리고 각자 자기가 아는 사실을 diagnostics에 남긴다. 라우터가 "코드만 왔음"을 남기고, 스코어러가 "이력 몇 개 매칭됐음"을 남긴다.

## 아직 애매하게 남아 있는 것

`_has_preeval_fields` 는 지금 `topics` 가 비어있지 않으면 True로 친다. 그런데 topics 안 모든 항목의 라벨이 비어있어서 `prefs` 가 빈 리스트가 되고, job도 없으면 `preeval_payload` 는 None을 반환한다. 이 경우 `code_only` 로 잡히긴 하는데 — 정말 코드만 온 건지, 아니면 클라이언트가 빈 topics 리스트를 실수로 넣은 건지 구분이 안 된다.

지금은 둘 다 "무시됨" 경고로 처리하고 있는데, 후자는 조용히 넘기는 게 맞을 것 같다는 생각이 든다. 아직 트래픽 로그를 훑어보지 않았다. 실제 요청 중에 "필드는 다 왔는데 라벨만 통째로 비어있는 케이스"가 얼마나 되는지부터 봐야 정할 수 있다.