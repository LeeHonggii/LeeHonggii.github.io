---
title: "추천 시스템에서 메타데이터가 먼저인 이유: 통제 키워드 풀과 표준 속성 부착"
date: 2026-07-20 17:00:00 +0900
categories: [Backend]
tags: [recommendation, metadata, controlled-vocabulary]
---

처음엔 제목이랑 설명 텍스트만 있으면 될 줄 알았다. TF-IDF든 임베딩이든 원문만 넣으면 유사도가 나오는 거 아닌가. 실제로 돌려보니 추천 상위권에 오르는 건 "설명을 길게 쓴 콘텐츠"였다. 짧게 쓴 좋은 강의는 뒤로 밀렸다. 이 시점에서 문제를 잘못 잡고 있다는 걸 알았다.

## 원문은 신호가 아니라 소음이었다

"Git 브랜치 전략 실습"이라는 제목만 놓고 보자. 이게 신입 대상인지, 팀장이 팀에 도입하려고 보는 건지, DevOps 트랙 안의 한 조각인지 원문은 알려주지 않는다. 작성자가 마케팅적으로 쓴 문장에서 이걸 뽑아내려니 매번 결과가 흔들렸다.

원본 텍스트가 아니라, 콘텐츠 위에 얹은 **표준 속성**을 유사도의 입력으로 써야 했다. 지금 API 스키마에서 콘텐츠 한 건이 응답으로 나가는 예시를 보면 이게 무슨 뜻인지 바로 드러난다.

```python
_ITEM_EXAMPLE = {
    "content_id": "22odo9wf",
    "title": "[HD]실무에서 사용 가능한 실전 Component 만들기 with React v18",
    "ncs_code": "200103",
    "strand": "프로그래밍/SW",
    "sub_category": "프로그래밍",
    "duration_bucket": "집중",
    "audience": ["공통"],
    "keywords": ["프로그래밍", "웹개발"],
    "institution": "tenant_a",
    "content_type": "institution",
    "rank": 1,
    "score": 1.6,
    "reasons": ["최근 이력 유사", "시리즈 다음화", "NCS 일치", "키워드 유사"],
}
```

`title` 옆에 붙은 것들 — `ncs_code`, `strand`, `sub_category`, `duration_bucket`, `audience`, `keywords` — 이게 추천 엔진이 실제로 먹는 재료다. `reasons` 필드에 "NCS 일치", "키워드 유사"라고 쓰여있는 것도 우연이 아니다. 매칭이 일어난 축을 사람이 읽을 수 있게 되돌려주려면 애초에 계산에 쓴 신호가 **명명된 축**이어야 한다.

## 풀을 먼저 얼려야 태거가 흔들리지 않는다

처음에 저지른 실수는 콘텐츠를 보면서 태그를 즉흥적으로 붙인 것이었다. 두 달쯤 지나 태그 목록을 뽑아보니 `python`, `Python`, `파이썬`, `Python3`가 다 따로 존재했다. `keywords` 배열에 이런 게 섞여있으면 유사도 계산 시점에 서로 다른 콘텐츠로 갈라진다.

순서를 뒤집었다. 콘텐츠를 먼저 보는 게 아니라 **분류 체계와 키워드 풀을 먼저 동결**하고, 콘텐츠는 풀 안의 항목으로만 매핑한다. 풀에 없는 개념이 나오면 태그를 만드는 게 아니라 풀 변경 PR을 열고 검토를 받는다. 태거가 여러 명일 때 이 규칙이 없으면 판단이 엇갈리고, 운영이 몇 달 지나면 되돌리기 어렵다.

동결된 풀이 있으니 요청 스키마도 코드/라벨 쌍으로 정착됐다. 사전평가에서 들어오는 직무·관심 같은 항목이 다 같은 모양이다.

```python
class CodeLabel(BaseModel):
    code: int | None = None
    label: str | None = Field(
        None,
        description="표시 라벨. 사전평가 매핑은 label 기준이라 "
                    "label이 없으면 무시됨(code만 보내면 반영 안 됨)"
    )
```

`label` 설명에 "code만 보내면 반영 안 됨"이라고 명시한 데엔 이유가 있다. 클라이언트가 코드값만 던져도 서버가 알아서 라벨로 풀어주면 될 것 같지만, 그러려면 서버 안에 "코드→라벨" 사전이 하나 더 살아있어야 한다. 그 사전이 풀과 어긋나면 조용히 잘못된 매칭이 일어난다. 그래서 라벨을 요청 시점에 명시적으로 받게 하고, 없으면 무시한다. 로직은 이 정책을 그대로 반영한다.

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

`code_only` 플래그가 이 규칙의 실체다. 필드가 채워져 있긴 한데 라벨이 없어서 `preeval_payload()`가 `None`을 낸 경우, 조용히 통과시키는 게 아니라 `diagnostics.warnings`에 남기고 `preeval_applied=False`를 응답에 박아둔다. 나중에 "왜 이 유저에 이 추천이 나갔지"를 되짚을 때 이 한 줄이 시간을 살린다.

## 원본과 파생을 섞지 말 것

또 하나 초반에 저지른 게, 태깅한 결과를 원본 title/description에 병합해버린 것이었다. 나중에 태그가 이상해 보여서 되돌리려니 원본이 이미 오염돼 있었다. 지금은 스키마에서 확실히 분리한다.

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
```

요청에서 들어오는 원본은 `CodeLabel` 형태 그대로 보존하고, 추천 엔진이 실제로 소비하는 페이로드는 `preeval_payload()`가 별도로 만들어낸다. 원본과 파생을 같은 변수에 두지 않는다. 이력도 같은 원칙이다 — `history_items()`는 이력 원문 필드를 그대로 리스트로 넘기고, 어느 콘텐츠를 매칭했는지는 응답의 `diagnostics.history_matched`로 별도 기록된다.

응답 쪽 `reasons` 배열도 같은 맥락이다. "최근 이력 유사", "시리즈 다음화", "NCS 일치", "키워드 유사" — 이건 사후에 붙인 설명이 아니라 어떤 신호축이 이 콘텐츠를 밀어올렸는지의 흔적이다. 파생 속성을 축별로 분리해 계산했기 때문에 결과에서도 축이 그대로 살아있다.

## 아직 남은 것

풀 동결이 완벽하진 않다. 지금도 신규 도메인이 들어오면 풀에 없는 키워드가 튀어나오고, 그때마다 임시로 근접 태그로 흡수시킨다. `diagnostics.warnings`에 남는 항목을 주기적으로 훑어서 풀 업데이트 PR을 만들고 있는데, 이걸 자동화하는 게 다음 작업이다.

응답의 `reasons`가 늘 정확한지도 검증이 부족하다. 계산에 쓴 축은 맞는데 그게 유저에게 "왜 이걸 추천했는가"의 설명으로 자연스러운지는 별개다. 축의 이름을 사람이 읽는 문장으로 매핑하는 계층을 하나 더 얹어야 할 것 같은데, 여기는 아직 손대지 않았다.