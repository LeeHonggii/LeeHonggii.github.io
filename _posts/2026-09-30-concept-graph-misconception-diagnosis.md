---
title: "오답을 설명하는 것에서 다음 학습을 처방하는 것으로: 개념 그래프 기반 진단"
date: 2026-09-30 17:00:00 +0900
categories: [AI]
tags: [concept-graph, misconception, learning-diagnosis]
---

처음 만든 건 분석 리포트였다. 채점 결과를 넣으면 "이 학생은 분수의 덧셈에서 자주 틀렸습니다" 같은 문장이 나왔다. 틀린 문제를 잘 요약한 글이었다. 그런데 교사에게 보여주면 반응이 같았다. 그래서 다음엔 뭘 시키죠?

리포트는 오답을 설명한다. 진단은 다음에 무엇을 공부할지 말해야 한다. 이 차이를 인정하고 나서야 설계가 바뀌었다.

## 출력이 "개념 목록"이어야 한다

진단의 결과물을 이렇게 정했다. 문장이 아니라 **되돌아가 학습할 개념의 제안**이다. 그러려면 "이 개념을 모른다"에서 끝나면 안 되고, 왜 모르는지와 어디로 돌아가야 하는지가 구조에 들어 있어야 한다. 그래서 세 가지를 연결했다.

- 개념 그래프(concept graph): 개념을 노드, 선수 관계를 엣지로 둔 지도
- 오개념(misconception): 개념 노드에 붙는 전형적인 잘못된 이해
- 문항: 어떤 오개념이면 틀리는지 태깅된 문제

정오답만 보면 "틀렸다"는 사실밖에 없다. 문항이 오개념에 태깅돼 있으면 "이 오개념을 가졌을 때 고르는 오답을 골랐다"로 해석이 한 단계 깊어진다. 오개념은 개념에 매달려 있으니 그래프를 따라 선행 개념으로 내려갈 수도 있다.

## 진단지 설계도부터 만들었다

문항을 LLM으로 바로 생성하지 않았다. 먼저 **어떤 문항이 몇 개 필요한지** 결정하는 설계도(blueprint)를 코드로 만들었다. 대상 개념 하나를 받아 선행 개념과 오개념을 끌어모으고, 문항 슬롯을 배분한다. 문항 생성은 그 슬롯을 채우는 일로 뒤에 온다.

아래 코드는 저장소에서 가져오되 모듈·클래스 이름은 일반화했다. 구조는 그대로다. 파일은 `rules()` 이후가 더 이어지는데, 그 부분은 싣지 않았다.

```python
PREREQ_SLOTS = 2
PER_MISCONCEPTION = 2
MAX_ITEMS = 12
TYPE_KO = {"PROC": "절차", "CONC": "개념", "LANG": "언어", "OVER": "과일반화"}


class DiagnosticBlueprint:
    def __init__(self, skill_id, max_items=MAX_ITEMS, save=True):
        self.skill_id = skill_id
        self.max_items = max_items
        self.target = None
        self.prerequisites = []
        self.fallback = []
        self.misconceptions = []
        self.excluded = []
        self.slots = []
        self.problems = []
        self.result = None
        self.get_target()
        if self.target is None:
            self.problems.append(f"개념지도에 없는 개념 {skill_id}")
            return
        self.get_prerequisites()
        self.get_misconceptions()
        self.allocate_slots()
        self.check()
        self.build()
        if save and not self.problems:
            save_json(BLUEPRINTS / f"{skill_id}.json", self.result)
```

생성자가 곧 파이프라인이다. 대상 조회, 선행 개념 수집, 오개념 선별, 슬롯 배분, 검증, 조립이 순서대로 돈다. 눈여겨볼 곳은 마지막 줄이다. `self.problems`가 하나라도 있으면 저장하지 않는다. 깨진 설계도가 파일로 남아 뒤 단계가 그걸 읽는 일을 막으려는 장치다. 대상 개념이 그래프에 없으면 초기에 바로 반환한다.

오개념 문항 수에 상한이 있다는 점이 설계에서 가장 고민한 부분이다. 진단지가 길어지면 학생이 끝까지 풀지 않는다. 그래서 12문항이라는 예산을 먼저 깔고, 그 안에서 나눴다.

```python
    def get_prerequisites(self):
        self.prerequisites = [self.skill_view(Mirivom.search_skill(p)) for p in Mirivom.prerequisites(self.skill_id)
                              if Mirivom.search_skill(p)]
        seen = {self.skill_id, *(p["id"] for p in self.prerequisites)}
        for p in self.prerequisites:
            for q in Mirivom.prerequisites(p["id"]):
                if q not in seen and Mirivom.search_skill(q):
                    seen.add(q)
                    self.fallback.append({**self.skill_view(Mirivom.search_skill(q)), "under": p["id"]})

    def get_misconceptions(self):
        mcs = sorted(Mirivom.misconceptions_of(self.skill_id), key=lambda c: -c.get("n", 0))
        room = (self.max_items - PREREQ_SLOTS * len(self.prerequisites)) // PER_MISCONCEPTION
        self.misconceptions = [self.misconception_view(c) for c in mcs[:room]]
        self.excluded = [{**self.misconception_view(c), "why": f"문항 {self.max_items}개 한도"} for c in mcs[room:]]
```

(원본의 `Mirivom`은 개념 그래프를 조회하는 내부 모듈이다.)

선행 개념은 직접 선수(1단계)만 슬롯을 받는다. 2단계 선수는 `fallback`에 `under`(어느 선행 개념 밑인지)와 함께 쌓아두기만 한다. 1단계 선행 개념에서도 막히는 학생이 나오면 더 내려갈 후보가 필요할 것 같아서 남겨뒀다. 지금 이 후보를 실제로 쓰는 경로는 이 글에서 보여줄 수 있는 게 없다.

오개념은 `n`(그 오개념이 데이터에서 관측된 횟수로 이해하고 있다) 내림차순으로 정렬하고, 남은 예산만큼만 자른다. 예산에서 탈락한 오개념은 버리지 않고 `excluded`에 "문항 12개 한도"라는 이유와 함께 남긴다. 나중에 "왜 이 오개념은 안 물어봤나"에 답할 수 있어야 해서다.

## 슬롯 배분과 "근거 2개" 규칙

```python
    def allocate_slots(self):
        for p in self.prerequisites:
            for k in range(PREREQ_SLOTS):
                self.slots.append({"role": "선행", "skill": p["id"], "misconception": None, "purpose": "개념 확인"})
        for c in self.misconceptions:
            for k in range(PER_MISCONCEPTION):
                self.slots.append({"role": "본", "skill": self.skill_id, "misconception": c["id"],
                                   "purpose": f"{c['name']} 이면 틀리는 문항"})
        if not self.misconceptions:
            for k in range(PER_MISCONCEPTION):
                self.slots.append({"role": "본", "skill": self.skill_id, "misconception": None, "purpose": "개념 확인"})
        for i, s in enumerate(self.slots, 1):
            s["no"] = i

    def check(self):
        count = {}
        for s in self.slots:
            count[s["skill"]] = count.get(s["skill"], 0) + 1
            if s["misconception"]:
                count[s["misconception"]] = count.get(s["misconception"], 0) + 1
        for k, n in count.items():
            if n < 2:
                self.problems.append(f"{k} 근거 문항 {n}개 (2개 이상 필요)")
        if len(self.slots) > self.max_items:
            self.problems.append(f"문항 {len(self.slots)}개 > 한도 {self.max_items}")
```

슬롯의 `purpose`가 "○○ 오개념이면 틀리는 문항"이다. 문항을 생성하는 쪽은 "이 오개념을 가진 학생만 틀리게 만들어라"는 요구를 받는 셈이다. 오개념 태깅이 사후 분류가 아니라 문항 설계 단계의 제약이 된다.

`check()`의 규칙은 단순하다. **개념이든 오개념이든 판단하려면 근거 문항이 2개 이상**이어야 한다. 한 문항 틀린 것으로 오개념을 단정하면 실수와 구분이 안 된다. 그렇다고 2개가 통계적으로 충분하다는 말은 아니다. 최소한의 안전장치일 뿐이고, 2라는 값을 실데이터로 검증하진 않았다.

같은 이유로 오개념이 하나도 없는 개념은 그냥 넘기지 않고 "개념 확인" 문항 2개를 대신 배정한다.

## 읽다가 걸린 것

글을 쓰려고 코드를 다시 읽다가 마음에 걸리는 곳을 찾았다. `get_misconceptions`의 `room` 계산이다. 선행 개념이 7개 이상이면 `(12 - 14) // 2 = -1`이 되고, `mcs[:-1]`은 "하나만 빼고 전부"라는 뜻이 된다. 예산이 없는데 오개념이 대부분 들어가는 것이다. 다행히 `check()`가 총 문항 수 초과를 잡아 저장을 막으니 조용히 새지는 않는다. 다만 원인이 "한도 초과"로만 보이고 진짜 원인은 가려진다. 선행 개념이 7개 이상인 개념이 실제로 있는지는 확인하지 않았다. `max(room, 0)`으로 막는 건 쉬운데, 그 전에 그런 경우가 있는지부터 그래프에서 세어봐야 한다.

## 정오답만으로는 모자란다

이 구조에는 한계가 분명하다. 객관식에서 정답을 고른 학생이 찍었는지 이해했는지 알 수 없고, 오답을 골랐다고 해서 그 오개념을 가졌다고 확정할 수도 없다. 지금 설계도에도 "개념 상태"를 `맞은 문항 / 문항 수`로 계산하는 규칙이 있는데, 코드에 "가안"이라고 적혀 있는 것처럼 임시다.

보완책으로 두 가지를 세웠다. 풀이 과정의 활동 데이터(어떤 순서로 썼고, 어디서 지웠는지)를 추론 근거에 더하는 것, 그리고 교사가 진단을 고칠 수 있게 하고 그 교정을 다시 쓰는 것이다. 오개념 추론 테스트는 돌렸다. 하지만 정확도가 얼마인지는 이 글에 쓸 만한 수치가 없다. 기준이 될 정답 라벨 자체를 아직 교사 교정으로 모으는 중이라서, 지금 숫자를 내면 무엇과 비교한 숫자인지 답할 수 없다.

다음에 볼 것은 두 가지다. 교정이 쌓이면 "근거 2개" 규칙이 과한지 부족한지 처음으로 데이터로 볼 수 있다. 그리고 `fallback`을 실제 진단 흐름에 연결할지는 아직 결정하지 못했다.