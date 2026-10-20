---
title: "AI는 선개념과 오개념을 어떻게 아는가: 생성한 가설과 관찰 근거의 차이"
date: 2026-10-20 17:00:00 +0900
categories: [LLM]
tags: [LLM, 교육AI, 평가설계, 출처추적]
---

기존 리포트 파이프라인을 코드로 따라가다가 한 가지가 계속 걸렸다. 리포트에는 문제마다 선개념·본개념·후개념이 붙고, 오개념 규칙도 붙는다. 그런데 그 목록이 어디서 오는지는 코드를 읽기 전까지 몰랐다.

내가 처음 가졌던 가정은 이랬다. 교육과정 어딘가에 문항과 개념이 연결된 표가 있고, 시스템이 그걸 조회한다. 틀렸다. 교사가 개념을 입력하지 않으면 모델이 문제를 보고 개념 목록을 만든다. 조회가 아니라 생성이다.

```json
"concepts": {"pre": ["선개념"], "core": ["본개념"], "post": ["후개념"]}
```

구조는 이 정도로 단순하다. 문제는 이 JSON이 교육과정 DB 조회 결과와 똑같이 생겼다는 점이다. 리포트를 읽는 교사 눈에는 둘이 구분되지 않는다.

## 두 가지 질문이 한 필드에 섞여 있었다

"이 문제를 풀려면 어떤 개념이 필요한가"와 "이 학생은 그 개념을 이해했는가"는 다른 질문이다. 앞쪽은 문제를 정의하는 일이고, 모델이 해도 크게 이상하지 않다. 다만 정답이 하나로 정해진 영역은 아니라서, 모델이 만든 목록은 그럴듯한 가설이다.

뒤쪽은 학생을 진단하는 일이다. 여기에는 관찰이 필요하다.

교사가 개념 이름을 직접 입력하는 경우를 따로 생각해 봤다. 목록의 출처는 분명해진다. 그런데 학생 판정은 그대로 모델 추정일 수 있다. 개념 이름이 사람 손에서 왔다고 해서 "이 학생이 이 개념을 이해했다"는 판정까지 사람의 것이 되지는 않는다. 둘을 이어 주는 건 관찰 가능한 기준이다. 개념 이름 옆에 "화면에서 무엇이 보이면 이 개념을 쓴 것으로 본다"가 붙어 있어야 코드가 판정할 수 있다.

## 코드는 이미 '관찰 가능한 것'만 세고 있었다

학생 활동 기록을 분석하는 쪽 코드를 읽으면서 이 구분이 선명해졌다. 규칙 JSON을 받아 기록의 매 순간 단계를 계산하는 탐침이다. 맨 위 주석에 따르면 이 규칙 JSON은 0단계 산출물이 들어올 자리로 비워 둔 것이다. 판독 종류는 범용이고, 어떤 교구를 보고 어떤 값을 기대할지는 규칙이 정한다.

```python
class Reading:
    """한 화면에서 규칙의 조건 종류별 판독값을 낸다."""

    def __init__(self, frame0, frame, setting):
        self.f0, self.f, self.s = frame0["elements"], frame["elements"], setting

    # ...

    def value(self, cond):
        t = cond["type"]
        if t == "group_shape_equals_group":
            start, gs = self.groups0()
            now = self.cells(self.f)
            a = [now[i] for i in gs[cond["group"] - 1] if i in now]
            return canon(a) == canon([start[i] for i in gs[cond["target"] - 1]])
        if t == "group_moved_count":
            start, gs = self.groups0()
            now = self.cells(self.f)
            return sum(1 for i in gs[cond["group"] - 1] if now.get(i) != start.get(i))
        if t == "support_counts":
            return list(on_supports(self.f, self.s["item"], self.s["support"], self.s["reach"])[0])
        if t == "items_created":
            ids0 = {e.get("id") for e in self.f0}
            return sum(1 for e, _ in boxes(self.f, self.s["item"]) if e.get("id") not in ids0)
        # ... (stick_number, sticks_moved 등)
        raise ValueError(t)
```

반환값을 보면 "모양이 같은가(bool)", "움직인 개수(int)", "새로 생긴 개수(int)" 같은 것들이다. 이해했는지 여부는 어디에도 없다. 코드가 낼 수 있는 건 화면 요소의 좌표와 id 비교 결과뿐이다. 모르는 조건 종류가 오면 추측하지 않고 `ValueError`로 멈추게 했다.

이 판독값은 규칙의 조건과 대조된다.

```python
    def ok(self, cond):
        v = self.value(cond)
        op, want = cond["op"], cond.get("value")
        return {"==": lambda: v == want, "<=": lambda: v is not None and v <= want, ">=": lambda: v is not None and v >= want,
                "in": lambda: v in want, "true": lambda: v is True,
                "pairs_equal": lambda: v is not None and all(v[a] == v[b] for a, b in want)}[op]()
```

`v is not None` 가드를 둔 이유가 있다. 판독이 안 되는 화면, 예를 들어 숫자가 아닌 배치가 나온 순간에 `None`이 비교식으로 흘러가면 "작거나 같다"가 우연히 참이 될 수 있다. 판독 불가를 거짓으로 처리하는 쪽이 낫다고 봤다.

단계 결정은 규칙 순서대로 조건을 훑는다.

```python
for f in fr:
    r = Reading(fr[0], f, rule["setting"])
    step = rule["steps"][0]["id"]
    for s in rule["steps"]:
        if all(r.ok(cond) for cond in s["when"]):
            step = s["id"]
```

뒤에 나온 단계가 앞 단계를 덮어쓴다. 모든 조건이 만족된 가장 뒤쪽 단계가 그 순간의 단계가 된다. 어느 단계에도 안 걸리면 첫 단계에 머문다. 이 기본값이 적절한지는 아직 확신이 없다. 아래 '남은 것'에 다시 적었다.

## 오개념 규칙은 가설이고, 관찰은 가설과 대조하는 일이다

수 모형 쪽 탐침은 이 구분이 더 직접적으로 드러난다. 블록 종류와 놓인 칸을 각각 읽는다.

```python
# 수 모형: 블록 종류 = shapeType(0 일·1 십·2 백·3 천), 놓인 칸 = connectedBoardIdx(0 이 일의 자리)
PLACE = {0: "일", 1: "십", 2: "백", 3: "천"}

def boards(f, names):
    out = {}
    for e in f["elements"]:
        if e.get("svgId") == BOARD:
            m = e.get("elements") or []
            places = {PLACE[c]: sum(row) for c, row in enumerate(m) if sum(row)}
            wrong = [f"{PLACE[c]}의 자리 칸에 {PLACE[u]} 모형 {k}개" for c, row in enumerate(m) for u, k in enumerate(row) if k and c != u]
            out[names.get(e["id"], e["id"][:5])] = {"n": e.get("n"), "칸": places, "다른 자리": wrong}
    return out
```

`wrong` 리스트는 "십의 자리 칸에 일 모형 3개"처럼 사실만 적는다. 변수 이름은 `wrong`이지만 출력 키는 "다른 자리"로 뒀다. 블록이 다른 자리에 놓였다는 건 관찰이다. 그게 자릿값 오개념 때문인지, 실수로 놓았다가 곧 옮길 중간 과정인지는 이 코드가 알 수 없다.

이동 이벤트에도 같은 원칙을 적용했다.

```python
for i, e in B.items():
    if i in A and where(A[i], names) != where(e, names):
        fit = ""
        if e.get("connectedBoardId"):
            fit = " [제자리]" if e.get("connectedBoardIdx") == unit(e) else " [다른 자리]"
        moves[f"{PLACE[unit(e)]} 모형: {where(A[i], names)} → {where(e, names)}{fit}"] += 1
```

여기서 라벨은 `[제자리]`와 `[다른 자리]`다. `[오개념]`이 아니다. 오개념 규칙이 "일 모형을 십의 자리에 놓는다"로 적혀 있다면, 이 로그는 그 규칙에 맞는 행동이 있었다는 증거까지만 된다. 규칙에 맞는 행동이 관찰됐다는 사실만으로 학생의 내적 이해 상태가 객관적으로 확정되지는 않는다. 같은 행동이 다른 이유로도 나올 수 있다.

## 출처 표기는 정확성을 보증하지 않는다

이 대화에서 나온 보완책은 두 층이다.

하나는 출처를 보존하는 것이다. 개념 항목마다 `origin` 필드로 교사 입력인지, 비전 분석 결과인지, 모델 추정인지를 남긴다. 판정 쪽에는 `basis=확정`(코드가 관찰값으로 계산)과 `basis=추정`(모델의 해석)을 구분해서 둔다. 리포트를 읽는 사람이 "이건 계산한 것, 이건 짐작한 것"을 가려 볼 수 있다.

다른 하나는 이게 전혀 다른 문제라는 인식이다. 출처 표기는 "누가 이 말을 했나"를 알려 줄 뿐이다. 모델이 생성한 오개념 규칙이 교육적으로 맞는지는 알려 주지 않는다. `origin: 추정`이라고 정직하게 적힌 틀린 규칙도 얼마든지 있을 수 있다. 정확성은 별도 검증이 필요하다. 제안된 방법은 사람 판정 골드셋을 만들고, 모델 판정과의 일치도를 가중 κ(가중 카파, 서열 척도 평가자 간 일치도 지표)로 비교하는 것이다.

이건 제안만 됐다. 골드셋도 없고 κ를 계산한 적도 없다. 그래서 지금 이 시스템의 개념 목록과 오개념 규칙이 얼마나 정확한지 나는 말할 수 없다.

## 남은 것

- 규칙 JSON을 만드는 0단계가 아직 비어 있다. 교사가 개념만 넣었을 때, 그 개념을 화면 관찰 조건으로 번역하는 부분을 누가 하는지(모델인지, 교사 확인을 거치는지) 정하지 못했다. 모델이 하면 그 조건 자체가 다시 생성된 가설이 된다.
- 단계 결정에서 아무 조건도 안 맞을 때 첫 단계로 두는 기본값이 맞는지 모르겠다. "첫 단계에 있다"와 "판독 불가"는 다른 상태인데 지금은 같은 값이다.
- 기록 분석 전에 확인해야 할 데이터 문제가 따로 있었다. 시험 데이터가 섞여 들어와 있었고, 일부 기록은 누락돼 있었다. 이 정도가 판정에 얼마나 영향을 주는지는 아직 재보지 않았다.
- 화면 녹화 영상을 입력으로 쓰는 별도 실험도 논의했지만, 데이터 요구사항을 정리한 데서 멈췄다. 구현도 검증도 하지 않았다.

골드셋을 누가, 몇 문항으로 만들지부터 정해야 한다. 아직 정하지 못했다.