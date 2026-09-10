---
title: "비교형·분류형·구성형… AI 학습 리포트에서 활동 유형은 무엇을 의미해야 할까"
date: 2026-09-10 17:00:00 +0900
categories: [AI]
tags: [edtech, ai-report, content-taxonomy, learning-design]
---

리포트를 만들기로 했다. 학생이 어떤 유형의 문제를 많이 풀었는지, 어떤 유형에서 막히는지를 보여주는 AI 피드백이었다. 그런데 막상 설계를 시작하려고 보니 첫 질문부터 막혔다.

**활동 유형이 뭔지 아직 정의가 없었다.**

---

교육과정 쪽에서 받은 분류는 여섯 가지였다.

- **비교형**: 두 개 이상의 개념·수식·도형을 비교하여 공통점과 차이점을 발견
- **분류형**: 주어진 요소를 특정 기준에 따라 그룹으로 분류
- **구성형**: 주어진 조건을 만족하는 수식·도형·패턴을 직접 구성
- (이하 세 가지)

문서로 읽으면 자연스럽다. 그런데 실제 문항을 하나 꺼내들면 애매해진다. "두 분수를 비교하고, 그 결과를 이용해 새로운 식을 구성하시오" — 이건 비교형인가, 구성형인가.

처음엔 이 분류가 **문제의 객관적 속성**이라고 생각했다. 문항 텍스트를 보면 어떤 유형인지 분류할 수 있을 거라고. LLM에 문항을 넣으면 알아서 태깅해줄 거라고.

그 가정이 틀렸다.

---

실제로 기존 추천 엔진 코드를 보면, 콘텐츠를 특성화하는 방식이 명확하다.

```python
def _pool_stats(pool: dict[str, dict]) -> dict:
    df: dict[str, int] = defaultdict(int)
    strand_kw: dict[str, set] = defaultdict(set)
    series_total: dict[str, int] = defaultdict(int)
    strand_n: dict[str, int] = defaultdict(int)
    strand_xp: dict[str, float] = defaultdict(float)
    n = 0
    for c in pool.values():
        kws = c.get('keyword_tags') or []
        if not kws:
            continue
        n += 1
        for k in set(kws):
            df[k] += 1
        if c.get('strand') and c['strand'] not in RETIRED_STRANDS:
            strand_kw[c['strand']] |= set(kws)
            strand_n[c['strand']] += 1
            strand_xp[c['strand']] += LV.xp_gain_of(c)
        # ...
```

`ncs_code`, `keyword_tags`, `strand` — 이 세 필드가 콘텐츠를 설명하는 전부다. 활동 유형을 담는 필드는 없다.

콘텐츠 깊이를 계산하는 함수도 마찬가지다.

```python
def content_depth(c: dict, stats: dict) -> float:
    kws = c.get('keyword_tags') or []
    aud = c.get('audience_tags') or []
    a = max((AUD_STAGE.get(t, 0.0) for t in aud), default=0.0)
    sig = series_sig(c.get('title') or '')
    num = series_num(c.get('title') or '') or 0
    if not sig:
        s = 0.3
    elif num <= 2:
        s = 0.2
    else:
        tot = stats['series_total'].get(sig, num) or num
        s = min(num / tot, 1.0) if tot else 0.2
    r = statistics.mean([stats['rarity'].get(k, 0.0) for k in kws]) if kws else 0.0
    return round((a + s + r) / 3.0, 4)
```

대상 직급(`audience_tags`), 시리즈 내 순서(`series_num`), 키워드 희귀도(`rarity`) — 이걸 합산해서 깊이를 수치화한다. 여기서도 활동 유형은 없다.

추천 엔진이 잘 모르는 것은 리포트 엔진도 모른다. 당연한 얘기지만 데이터 구조를 직접 보기 전까지는 실감이 안 난다.

---

그래서 "LLM이 문항 텍스트를 보고 유형을 자동 태깅하면 되지 않냐"는 아이디어로 돌아갔다. 가능하긴 하다. 하지만 문제가 있다.

활동 유형은 문제 텍스트만으로 결정되지 않는다. **출제 의도**와 **학생이 어떻게 상호작용하도록 설계됐는가**에 달려있다. 텍스트는 같아도 의도가 다르면 유형이 다르다. LLM은 출제자의 의도를 텍스트 밖에서 읽지 못한다.

그러면 정의는 저작자 쪽에서 와야 한다. 문항을 만든 사람이 "이 문항의 주 유형은 구성형"이라고 확정하는 것. LLM은 추론이 아니라 그 라벨을 전달받아서 피드백 생성에 쓰는 역할이 된다.

라벨 결정 책임이 시스템에서 사람으로 이동한다.

이게 번거로워 보이지만, 사실 더 정직한 구조다. AI가 틀릴 수 있는 판단을 사람이 확정하고, AI는 그 확정값을 기반으로 추론한다. 학습 리포트 시스템에서 코드 확정값과 LLM 추론값의 경계가 어디 있어야 하는지와 같은 문제다.

---

하나 더 복잡한 건 겹침이다.

단일 문항에 여러 유형이 걸릴 수 있다. 비교하면서 분류하는 문항, 분류하면서 구성하는 문항. 모든 조합을 허용하면 리포트가 분산된다. "이 학생은 비교+구성형에서 많이 틀렸다"는 문장은 피드백이 되기 어렵다.

그래서 저작자가 주 유형 하나를 확정해야 한다는 결론이 나왔다. 복수 허용이 아니라 단수 선택. 시스템 설계 입장에서는 깔끔한 판단이지만, 저작 워크플로우 입장에서는 추가 작업이다.

---

유형 라벨이 생기면, 다음은 유형별 피드백 체계를 설계해야 한다. "비교형에서 자주 틀리는 학생에게 어떤 메시지를 줄 것인가?"

이걸 지금 설계하고 싶은 유혹이 있다. 하지만 샘플 데이터 없이 설계하면 빈 지점을 알 수 없다. 어떤 유형에 문항이 몇 개나 라벨링되는지, 실제 학생 데이터에서 어떤 유형에 시도가 집중되는지 — 이걸 확인하기 전에 피드백 문안을 만들면 쓰이지 않는 분기가 생긴다.

그래서 순서는 이렇게 잡았다: 먼저 샘플 문항 라벨링 → 유형별 분포 확인 → 피드백 체계 설계.

아직 라벨링 작업이 시작되지 않았다. 분포를 보기 전까지는 피드백 문안을 확정하지 않을 것이다.