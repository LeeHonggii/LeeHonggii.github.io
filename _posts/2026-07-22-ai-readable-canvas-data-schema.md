---
title: "LLM이 캔버스를 이해하게 하려면 좌표 로그부터 버려야 한다"
date: 2026-07-22 17:00:00 +0900
categories: [AI]
tags: [LLM, 스키마설계, 캔버스, 튜터링]
---

처음엔 로그를 그대로 넘겼다.

```
오브젝트 734가 500px 이동
오브젝트 891이 삭제됨
오브젝트 902가 생성됨
```

모델이 돌려준 문장은 "사용자가 무언가를 옮긴 것 같습니다" 수준이었다. 더 큰 모델로 바꿔봤다. 문장이 조금 더 유려해졌을 뿐, 판단은 여전히 없었다. 그때 착각을 뒤집었다 — 모델이 부족한 게 아니었다. 내가 넘긴 데이터가 판단할 만한 것을 담고 있지 않았다.

## 렌더러의 언어와 판단자의 언어

`500px 이동` 은 렌더러를 위한 좌표다. 화면에 픽셀을 찍는 데는 충분하다. 문제는 튜터링 AI가 하는 일은 픽셀을 찍는 게 아니라는 것이다. 학생이 노란 삼각형을 왜 저기다 놨는지, 그게 정답 근처인지, 교사가 원한 배치인지를 말해야 한다. 좌표만 보고 그걸 추론하라는 건, 수학 채점자에게 "학생이 손을 23cm 움직였다"고 알려주고 채점을 시키는 것과 같다.

같은 이벤트라도 어떤 필드를 함께 실어 보내느냐에 따라 모델이 하는 일이 바뀐다. 좌표만 있으면 "무엇이 일어났는지"를 추측해야 하고, 의미가 실려 있으면 "일어난 일을 해석"하기만 하면 된다. 후자가 훨씬 안전하다.

## 그래서 객체 생성 시점에 심는다

정답 판정을 런타임에 LLM한테 시키지 않는 게 핵심이었다. 정답 여부는 캔버스 로직이 이미 알고 있다. 그걸 객체 자체에 박아두고, 모델에게는 해석만 시킨다.

```json
{
  "id": "obj_734",
  "type": "shape",
  "class": "triangle",
  "color": "yellow",
  "intent": "answer_candidate",
  "answerLayer": {
    "isCorrect": false,
    "targetZone": "zone_B",
    "currentZone": "zone_A"
  },
  "teacherGuide": "직각삼각형의 빗변을 가리키도록 배치"
}
```

이 구조에서 모델이 하는 일은 "노란 삼각형이 zone_B에 가야 하는데 지금 zone_A에 있고, 교사 가이드는 빗변을 가리키라고 했다"를 자연어 피드백으로 옮기는 것뿐이다. 정답 판정은 이미 끝나 있다. 환각(hallucination)이 낄 자리를 원천에서 좁혔다.

## 이 감각은 다른 도메인에서도 반복된다

같은 판단을 다른 프로젝트에서도 겪었다. 추천 시스템 API를 짤 때, 초기엔 학습 이력에 콘텐츠 ID 하나만 넘겼다. 그러니 뒤에서 조인·매칭이 다 필요했고, 회사마다 ID 체계가 다르면 곧바로 깨졌다. 그래서 요청 스키마를 이렇게 바꿨다.

```python
class ContentHistoryItem(BaseModel):
    content_uuid: str | None = Field(None, description="이력 콘텐츠 UUID = content.content_uuid (권장). 회사 풀에서 매칭")
    content_id: str | None = Field(None, description="(하위호환) content.contents_agent_pk 또는 content_pk. content_uuid 우선")
    progress_rate: float | None = None
    pass_flag: int | None = None
    total_score: float | None = None
    user_sat_score: float | None = None
    learning_seconds: float | None = Field(None, description="실제 학습시간(초). 원천 TRNG_TCT 등에서 매핑")
    learned_at: str | None = Field(None, description="학습 시각(정렬용). 누락 시 가장 오래된 것으로 간주")

    @property
    def cid(self) -> str | None:
        return self.content_uuid or self.content_id
```

캔버스 스키마에서 `answerLayer.isCorrect` 를 미리 박아두는 것과 이건 같은 종류의 판단이다. 하류(下流)의 추론기가 다시 계산하지 않도록, 상류(上流)에서 이미 정해진 사실을 실어 보낸다. `pass_flag`, `total_score`, `progress_rate` 는 학습 결과에 대한 "이미 판정된 값"이고, 추천 엔진은 이걸 재판정하지 않고 해석만 한다.

`CodeLabel` 도 같은 맥락이다.

```python
class CodeLabel(BaseModel):
    code: int | None = None
    label: str | None = Field(None, description="표시 라벨. 사전평가 매핑은 label 기준이라 label이 없으면 무시됨(code만 보내면 반영 안 됨)")
```

코드 숫자만 있어도 시스템은 굴러간다. 그런데 사전평가 매핑이 label 기준으로 짜여 있어서 code만 오면 매칭이 통째로 빠져버린다. 저 `description` 한 줄은 몇 번 당하고 나서 쓴 문장이다. 캔버스에서 `obj_734` 만 넘길 때 모델이 아무 말도 못 하던 것과 정확히 같은 패턴이다 — 식별자만 있고 의미가 없다.

## 비전 모델은 안전벨트 정도

"그냥 캔버스 스크린샷 찍어서 비전 모델한테 넘기면 안 되나?" 는 자주 나오는 질문이다. 시도해봤다. 겹친 도형에서 자꾸 틀렸다. 비슷하게 생긴 삼각형·사각형을 헷갈렸다. 결정적으로, 픽셀 위치를 "zone_B" 같은 논리적 영역으로 매핑하는 걸 신뢰할 수 없었다.

지금 쓰는 분리는 이렇다. 채점과 피드백 생성은 구조화된 JSON을 근거로 텍스트 LLM이 한다. 비전 모델은 "이미지에서 뭔가 이상하면 플래그" 정도의 보조 감지로만 붙인다. 주(主)와 보조가 뒤집히지 않게 한다.

## 아직 정하지 못한 것

`teacherGuide` 를 자연어로 둘지, 좀 더 구조화된 규칙으로 쪼갤지는 아직 결정 못 했다. 자연어면 저자(교사)가 쓰기 편한데 모델이 해석하는 폭이 넓어져 편차가 생긴다. 구조화하면 편차는 줄지만 저작 도구부터 다시 짜야 한다.

당분간은 자연어로 두고 편차를 재본 뒤에 고르려 한다. 아직 재보지 않았다.