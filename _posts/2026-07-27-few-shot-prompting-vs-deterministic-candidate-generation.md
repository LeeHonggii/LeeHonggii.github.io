---
title: "퓨샷으로 맞춘 정답은 일반화가 아니다"
date: 2026-07-27 17:00:00 +0900
categories: [AI]
tags: [prompt-engineering, few-shot, llm, recommendation]
---

추천 파이프라인이 이상하게 굴었다. 어떤 질의는 엉뚱한 키워드를 뽑았고, 어떤 질의는 아무것도 반환하지 않았다. 로그를 되짚어 올라가니 원인은 내가 프롬프트에 넣어둔 few-shot 예시였다. 예시에 있던 문장은 정확히 맞혔다. 조금만 표현이 바뀌면 무너졌다.

## 처음엔 예시를 한 줄 더 붙이려 했다

"예외 케이스니까 예시 하나 더 추가하면 되잖아."

이 생각이 문제의 시작이었다. 버그가 보이면 반사적으로 그 문장을 예시에 편입시켰다. 다음 리뷰 때 새로운 실패 케이스가 나오면 또 예시 하나. 프롬프트는 부풀어갔고, 특정 케이스는 맞기 시작했다. 대신 옆에 있는 유사 케이스가 새로 깨졌다.

가장 기억에 남는 건 "수 모으기 활동"과 "수 가르기 활동"이었다.

```
입력: "수 모으기 활동"
기대: keywords=["모으기", "수 모으기"]
실제: keywords=["모으기"]
```

이걸 고치려고 예시를 박아 넣었더니, "수 가르기 활동"이 들어왔을 때 모델은 "가르기"만 남기고 "수 가르기"를 빠뜨렸다. 예시가 가르친 건 패턴이 아니라 그 문장 하나였다.

더 나빴던 건 이런 케이스다.

```
입력: "전체와 부분의 관계 이해"
출력: 분수 관련 키워드
```

맥락 없이 보면 그럴듯하다. 하지만 이 표현은 학년·단원에 따라 자연수의 구성일 수도, 분수 개념일 수도, 집합 개념일 수도 있다. 학년·차시 정보 없이 "분수"로 단정 짓는 건 추론이 아니라 도박이었다. 그리고 나는 그 도박을 프롬프트에 계속 위임하고 있었다.

## 프롬프트 패치는 시스템이 아니라 수동 관리다

예시가 10개에서 20개로 늘어나면 attention(모델이 어디에 주목할지)은 오히려 흩어진다. 예시끼리 충돌하는 패턴이 생기고, 그때부터는 어느 예시가 어느 케이스를 잡고 있는지 사람도 추적하지 못한다. 새 케이스가 나올 때마다 내가 손으로 예시를 짜넣어야 한다는 뜻이기도 하다. 이건 파이프라인이 아니라 나 자신이 파이프라인의 일부가 된 상태다.

이직할 때 이 코드를 인수인계한다고 상상해봤다. "이 예시는 왜 있어요?"라는 질문에 나는 답할 수 있을까. 아마 커밋 로그를 뒤져야 할 것이다. 그 순간 결정했다. LLM에게 무에서 뽑으라고 시키는 걸 그만두자.

## 역할을 다시 나눴다 — 후보는 결정적으로, 선택만 LLM에게

리팩토링의 핵심은 한 줄로 정리된다. **후보 생성은 결정적 로직이, 선택과 모호성 판단만 LLM이**.

추천 엔진 라우터의 인터페이스도 이 원칙에 맞춰 다시 설계했다. LLM이 이력·사전평가를 통째로 씹어 뭔가를 뽑아내는 구조가 아니라, 입력을 명확한 스키마로 받고 결정적 파이프라인이 처리하는 구조다.

```python
class RecoRequest(BaseModel):
    tenant_id: str = Field(..., description="회사/기관 코드")
    user_id: str | None = None
    learner_type: CodeLabel | None = Field(None, description="캐릭터(사전평가)")
    job: CodeLabel | None = Field(None, description="직무(사전평가)")
    sub_job: CodeLabel | None = None
    career: CodeLabel | None = None
    scores: dict[str, int] | None = Field(None, description="6역량 점수 {q1..q6: 1~4}")
    topics: list[CodeLabel] = Field(default_factory=list, description="관심 주제(사전평가)")
    content_history: list[ContentHistoryItem] = Field(default_factory=list, description="학습 이력")
```

여기서 `CodeLabel`이 code와 label을 함께 들고 다니게 한 건, 예전에 프롬프트가 code만 받고 자기 마음대로 label을 상상해서 채우던 사고를 다시는 겪고 싶지 않아서였다.

```python
class CodeLabel(BaseModel):
    code: int | None = None
    label: str | None = Field(
        None,
        description="표시 라벨. 사전평가 매핑은 label 기준이라 label이 없으면 무시됨(code만 보내면 반영 안 됨)",
    )
```

`preeval_payload()`도 label 없이 code만 온 경우엔 아예 payload를 만들지 않는다. LLM이 "code 9가 뭐였더라" 하면서 추측할 기회를 주지 않는다.

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

라우터는 이 상태를 그대로 사용자에게 알린다. label 없이 code만 들어온 경우, 422로 명확히 거절하거나 진단(warning) 필드에 남긴다.

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

이 흐름은 취향이 아니라 방어선이다. "모호한 입력이 들어오면 LLM이 알아서 잘 해주겠지"를 시스템에서 걷어내는 것.

## LLM이 남긴 자리 — 그것도 명시적으로

후보를 결정적으로 뽑는다고 LLM을 완전히 걷어낸 건 아니다. 후보군이 여러 개일 때 "이 학년 이 차시라면 어느 쪽이 자연스러운가"를 고르는 자리, "두 개 모두 가능성이 있음"이라고 유보하는 자리는 여전히 LLM이 낫다. 다만 그 자리는 이제 **좁고 명확하다**. 무에서 뽑아내는 자리가 아니라, 준비된 후보 안에서 고르는 자리.

응답 스키마의 `diagnostics.warnings`, `fallback_axes` 같은 필드가 이 자리의 흔적이다. LLM이 뭘 못 했는지, 왜 그랬는지가 응답에 그대로 남는다. 이게 있어야 다음 리뷰에서 "이 케이스 왜 이래?"에 커밋 로그를 뒤지지 않고 답할 수 있다.

## 아직 남은 것

두 가지가 남았다.

하나는 **후보 생성 로직의 커버리지 측정**이다. 결정적으로 뽑았다고 다 잡히는 건 아니다. 어떤 질의에서 후보군이 비는지, 그때 LLM이 무엇을 대신 뽑아 왔는지, 아직 지표로 보고 있지는 않다. "예전보다 안정적이다"라는 감각은 있는데, 숫자로 못 재봤다. 이걸 재는 게 다음이다.

다른 하나는 결국 되돌아오는 질문이다. label 매핑 테이블 자체는 사람이 유지한다. 그 테이블에 새 항목이 들어올 때, 관련된 후보 생성 규칙이 자동으로 확장되지 않으면 결국 나는 예시를 손으로 넣던 그때와 비슷한 자리에 서 있게 된다. 이 부분이 아직 애매하게 남아 있다.