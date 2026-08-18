---
title: "추천 점수는 어떻게 설명 가능해지는가: 가중결합·IDF·부스팅·RRF로 풀어낸 콘텐츠 추천"
date: 2026-07-20 17:00:00 +0900
categories: [AI]
tags: [추천시스템, RRF, IDF, 설명가능성]
---

## "왜 이 콘텐츠가 추천됐어요?"

내부 시연 자리에서 이 질문을 처음 받았을 때 나는 임베딩 코사인 유사도를 꺼내려다 멈췄다. 벡터 거리를 말하는 순간 상대방의 눈이 흐려질 것을 알았기 때문이다. 그날 이후 나는 추천 점수를 **사람이 산술로 분해할 수 있는 구조**로 다시 짜기로 했다. 이 글은 그 재설계에서 살아남은 네 가지 판단의 기록이다.

내가 만드는 건 학습 콘텐츠 추천 API다. 요청이 들어오면 사용자에게 네 갈래(직무·경력·회사·종합) 축으로 최대 10건씩 돌려준다. 골격은 아래처럼 생겼다.

```python
@router.post("/recommend",
             summary="학습 이력·사전평가 기반 4갈래(직업·관심·선호·Best) 콘텐츠 추천")
def recommend_endpoint(req: RecoRequest):
    return _run(req)
```

입력 스키마도 단순하지 않다. 사전평가(직무·경력·관심 주제)와 학습 이력이 함께 들어온다. 둘 중 하나라도 있어야 추천이 나간다.

```python
class RecoRequest(BaseModel):
    tenant_id: str = Field(..., description="회사/기관 코드")
    user_id: str | None = None
    job: CodeLabel | None = Field(None, description="직무(사전평가)")
    career: CodeLabel | None = None
    topics: list[CodeLabel] = Field(default_factory=list)
    content_history: list[ContentHistoryItem] = Field(default_factory=list)
    debug: bool = Field(False, description="항목별 신호 분해(debug 필드) 포함")

    def preeval_payload(self) -> dict | None:
        prefs = [t.label for t in self.topics if t and t.label]
        job = _label(self.job)
        if not (job or prefs):
            return None
        return {'job': job, 'sub': _label(self.sub_job),
                'career': _label(self.career), 'prefs': prefs,
                'character': _label(self.learner_type),
                'scores': self.scores}
```

여기서 짚어두고 싶은 건 `debug` 플래그다. 시연 자리에서 "왜 추천됐냐"를 받아친 뒤로, 항목마다 신호 분해를 붙일 수 있는 스위치를 열어뒀다. 설명가능성은 나중에 덧붙일 수 있는 게 아니다. 스키마부터 자리를 만들어놔야 나중에 쥐어짜지 않는다.

## 신호를 뭉개지 않는다 — 가중결합

콘텐츠 기반 추천에서 가장 흔한 실수는 이질적인 신호를 한 벡터에 우겨넣는 것이다. 키워드 태그 겹침, 카테고리 일치, 제목 유사도는 애초에 단위와 의미가 다르다. 이걸 임베딩 하나로 뭉치면 "왜"를 되짚을 수 없다.

그래서 각 신호를 0~1로 정규화해서 따로 채점하고 선형 결합했다.

```
관련도 = w1 · sim_keyword
      + w2 · sim_title
      + w3 · sim_category
```

이 구조의 진짜 장점은 발표 자리에 있다. "키워드 0.7, 카테고리 0.3 가중이어서 0.56"이라고 말할 수 있다. 벡터 얘기를 꺼낼 필요가 없다.

## 키워드는 IDF 가중 자카드로

TF-IDF의 TF는 문서 내 빈도인데, 태그 비교에서 빈도는 의미가 없다. 태그는 있거나 없거나다. 그래서 자카드에 IDF를 얹는다.

```
sim_kw = (겹친 태그 IDF 합 + 0.5) / (합집합 태그 IDF 합 + 0.5)
```

`+0.5`는 BM25 스무딩에서 가져온 안전판이다. 빈 집합 나눗셈을 막고 극단값을 눌러준다. 흔한 태그("프로그래밍")보다 희귀한 태그("MLOps 모니터링")가 겹칠 때 점수가 더 뛴다. 설명하기 쉬운 구조다.

## 부스팅은 곱셈으로

특정 조건(최신, 강조)에 가산점을 주고 싶을 때 덧셈은 함정이다. 관련도 0.05짜리 콘텐츠에 부스팅 0.5를 더하면 0.55가 되어 상위로 침투한다. 관련 없는 게 프로모션만으로 올라오는 사고가 여기서 난다.

```python
# 덧셈 — 위험
score = base + boost_sum

# 곱셈 — 안전
score = base * (1 + boost_sum)
```

곱셈은 `0.05 × 1.5 = 0.075`에 머문다. 관련도가 낮으면 아무리 부스팅해도 뛰어오르지 못한다. "배율로 증폭한다"는 말은 회의 자리에서도 오해가 없다.

## 서로 다른 스케일은 RRF로 합친다

응답 형태를 다시 보면 네 축이 나온다.

```python
"axes": {
    "job": [...], "career": [...], "company": [...], "best": [...]
},
```

`best`는 앞의 세 축을 융합한 결과다. 여기서 문제가 생긴다. 각 축의 점수는 스케일이 다르다. job에서 0.9가 최상위라도 career에서는 평범할 수 있다. 그냥 합하면 스케일이 큰 축이 결과를 독식한다.

RRF(Reciprocal Rank Fusion, 역순위 융합)는 점수 대신 순위만 쓴다.

```
rrf = 1/(K + rank_job) + 1/(K + rank_career) + 1/(K + rank_company)
```

`K=60`이 관행이다. 1등은 1/61, 2등은 1/62. 순위가 낮을수록 기여가 급격히 준다. 세 축 모두에서 상위권인 콘텐츠가 자연스럽게 위로 올라온다. 정규화가 필요 없다는 게 이 방법의 가장 큰 미덕이다.

응답에 이 흔적이 그대로 남아있다.

```python
_BEST_EXAMPLE = {
    ...,
    "best_score": 1.0,
    "rrf_score": 1.0,
    "axis_ranks": {"job": 1, "career": 1, "company": 1, "next": 1},
    "reasons": ["종합 상위"],
}
```

`axis_ranks`를 굳이 응답에 남기는 이유는 하나다. "왜 종합 1위인가?"에 답할 때 세 축에서 각각 몇 위였는지를 그대로 보여주면 끝난다.

## reasons 필드에 대해

각 아이템에는 `reasons`가 붙는다.

```python
"reasons": ["최근 이력 유사", "시리즈 다음화", "NCS 일치", "키워드 유사"]
```

이건 점수를 만든 신호 중 상위 몇 개를 사람이 읽을 문장으로 뽑은 것이다. 처음엔 없어도 된다고 생각했다. 어차피 debug 필드에 분해값이 들어가니까. 그런데 debug는 개발자만 본다. 기획자·CS는 응답을 그대로 UI에 꽂는다. 그들에게 필요한 건 숫자가 아니라 문장이었다.

## 아직 못 푼 것

가중치 `w1, w2, w3`은 지금 사람이 정한 상수다. A/B로 튜닝해본 적 없다. 트래픽이 붙기 전이라 라벨이 없다. 로그가 쌓이면 관측 데이터 기반으로 재조정할 계획인데, 그 시점에 곱셈 부스팅의 배율(`1 + boost_sum`의 상한)도 같이 재봐야 할 것 같다. 지금은 상한을 걸어두지 않았다. 부스팅 신호가 3~4개 겹치는 콘텐츠가 실제로 얼마나 되는지 아직 재보지 않았다.

그리고 RRF의 `K=60`. 관행일 뿐, 우리 데이터에 맞는 값인지는 검증하지 않았다. 축별 후보 풀 크기가 K에 비해 너무 작으면 순위 차이가 뭉개진다. 이걸 다음 스프린트에 볼 것이다.