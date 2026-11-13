---
title: "이미지에서 리포트까지: LLM이 판단할 것과 코드가 확정할 것"
date: 2026-11-13 17:00:00 +0900
categories: [LLM]
tags: [multimodal, llm-pipeline, vision, deterministic-logic, report-generation]
---

기존 학습 리포트 코드를 읽다가 내 머릿속 순서가 틀렸다는 걸 알았다. 이 글은 그 정정 기록이다.

처음 그린 흐름은 이랬다. 화면 이미지를 비전 모델이 분석하고, 그 결과로 평가하고, 리포트를 쓴다. 한 줄로 이어진 파이프라인이었다. 실제로는 그렇지 않았다.

## 비전 분석은 리포트 안에 있지 않다

비전 분석은 리포트 생성과 따로 돈다. 한 번 돌고 나면 결과가 캐시되고, 리포트는 그 캐시를 가져다 쓴다.

리포트 쪽의 실제 순서는 이렇다.

> 케이스 로드·정규화 → 활동 문장화 → 평가 입력을 조립하면서 코드 계산 → LLM 평가 → 코드 확정값 덮어쓰기 → 대상별 서술 → 저장

처음에 놓친 건 "활동 문장화"와 "평가" 사이에 코드 계산이 끼어 있다는 점이다. 평가 프롬프트를 조립하는 도중에 조작 횟수, 정답 여부 같은 숫자가 코드로 먼저 정해진다. LLM은 그 위에서 평가를 하고, 평가가 끝나면 코드가 계산한 값을 다시 덮어쓴다. 최종 서술은 별도 호출이다. 서술 단계에는 재판정을 맡기지 않는다.

LLM이 평가에서 정답 여부를 다르게 말했더라도 최종 값은 코드 쪽이다. LLM이 하는 일은 코드로 계산할 수 없는 것에 한정된다. 풀이 과정이 어떤 사고였는지, 어떤 오개념이 의심되는지 같은 해석이 여기에 든다.

## 입력을 복원하는 쪽: 두 장의 화면

비전 쪽은 화면의 역할을 나눈다. 문제의 초기 화면과 모범답안의 최종 화면이다. 초기 화면에서는 문제가 무엇인지(문제 정의)를 읽고, 모범답안 최종 화면에서는 정답이 무엇인지를 읽는다. 이때 화면에서 추출한 텍스트를 이미지와 같이 모델에 준다. 이미지만 주면 숫자나 기호를 잘못 읽는 일이 생기기 때문이다.

정답이 어디서 오는지에도 우선순위가 있다.

> 코드 계산 > 정답 화면에서 읽기 > 모델 추론

결과에는 출처를 남긴다. `answerSource: "code:optimum"`이면 코드가 계산한 값이고, `"vision:truth"`면 정답 화면에서 읽은 값이다. 모델 추론으로 떨어졌다면 그것도 표시된다. 나중에 "이 정답은 어디서 왔느냐"는 질문을 받으면 이 필드로 답한다.

학생별 비전 판정은 이렇게 만든 정답과 학생의 최종 화면을 대조하는 일이다. 모델에게 "이게 맞는지 알아서 판단하라"고 하지 않고, 정답이라는 기준을 먼저 세운 뒤 대조만 시킨다.

## 코드가 소유하는 값은 어떻게 생겼나

여기서부터는 사고 방식을 보여주는 예다. 아래 코드는 위 리포트 파이프라인의 `applyCodeFacts` 같은 부분이 아니다. 그 코드는 이번에 읽은 범위에 없다. 같은 저장소에 있는 다른 모듈, 진단 문항을 채점하는 규칙 엔진이다. 다만 "계산 가능한 건 코드가 가진다"는 원칙이 가장 노골적으로 드러난 코드여서 가져왔다. 식별자는 일반화했다.

문항 정오는 이렇게 판정한다.

```python
@staticmethod
def is_correct(item, value):
    if value in (None, UNKNOWN, SKIP, ""):
        return False
    t = item.get("type")
    if t == "match":
        return isinstance(value, dict) and (value.get("total") or 0) > 0 and value.get("done") == value.get("total")
    if t in ("drag-tick", "drag-point"):
        ans = item.get("answer") or {}
        return isinstance(value, dict) and len(ans) == len(value) and all(value.get(k) == v for k, v in ans.items())
    if t == "short":
        return any(norm_answer(value) == norm_answer(a) for a in [item.get("answer")] + list(item.get("alt") or []))
    return value == item.get("answer")
```

문항 유형마다 "맞았다"의 정의가 다르다. 짝짓기는 전부 완료해야 하고, 드래그는 키 개수와 값이 모두 같아야 하고, 단답은 정규화한 뒤 정답 또는 허용 답안 중 하나와 같아야 한다. 이걸 LLM에게 "맞았나요?"라고 물으면 대부분은 맞게 답한다. 하지만 같은 입력에 항상 같은 답이 나온다는 보장은 없다. 이 함수는 보장한다.

단답 비교에서 쓰는 정규화는 더 사소하지만 실제로는 사고가 더 많이 나는 자리다.

```python
def norm_answer(s):
    t = "".join(chr(ord(c) - 0xFEE0) if c in "０１２３４５６７８９．，＋－" else c for c in str(s).strip())
    t = re.sub(r"\s+", "", re.sub(r"[−–—‐]", "-", t))
    if re.fullmatch(r"-?\d+(\.\d+)?", t):
        f = float(t)
        return str(int(f)) if f.is_integer() else repr(f)
    return t
```

전각 숫자를 반각으로 바꾸고, 여러 종류의 빼기 기호를 `-` 하나로 모으고, 공백을 지운다. 숫자는 값으로 비교하므로 `3.0`과 `3`이 같다. 학생이 `− 3`이라고 썼는데 틀렸다고 나오면 그건 학생의 오류가 아니라 비교 코드의 오류다. 이런 걸 모델의 판단에 맡길 이유가 없다.

오답이 어떤 오개념을 가리키는지도 코드가 연결하는데, 여기에는 조건이 하나 붙는다.

```python
@staticmethod
def signal(item, value, rules):
    mid = JudgeRules.item_misconception(item, value)
    if mid and item.get("role") == "필수확인" and not rules.get("essentialMis") and not str(mid).startswith("pre:"):
        return None
    return mid
```

필수확인 문항은 선행 개념(`pre:`)과 연결된 오답 신호만 남기고 나머지는 버린다. 규칙 기본값(`essentialMis: False`)이 그렇다. 신호를 얼마나 신뢰할지가 코드에 규칙으로 박혀 있으니, 리포트에서 "이 오개념이 의심된다"는 문장이 나오면 어떤 경로로 나왔는지 거슬러 올라갈 수 있다.

단위 개념의 상태도 숫자 기준이다.

```python
DEFAULT_RULES = {"strongRate": 0.75, "checkRate": 0.5, "lowScoreMax": 3, "misJudge": "selected", "essentialMis": False}

@staticmethod
def unit_status(ok, total, must_missed, rules):
    rate = ok / total if total else 0
    status = "strong" if rate >= rules["strongRate"] else "check" if rate >= rules["checkRate"] else "weak"
    return "weak" if must_missed else status
```

정답률 0.75 이상이면 강점, 0.5 이상이면 진단 보류, 그 아래면 약점이다. 필수확인 문항을 하나라도 놓치면 정답률과 무관하게 약점이다. 이 임계값이 교육적으로 적절한지는 따로 검증해야 하는 문제다. 여기서 말하고 싶은 건 그 값이 어디 있느냐다. 프롬프트 문장 속이 아니라 `DEFAULT_RULES`에 있다. 워크북이 `rules`를 넘기면 덮어쓸 수 있다.

## 기록을 읽는 쪽도 코드다

LLM에게 넘기기 전의 입력 복원도 코드가 한다. 학생 기록은 학습 로그만 있을 수도 있고, 뷰어가 보낸 단계 보고와 같이 올 수도 있다. 리더는 어느 쪽이 오든 같은 모양으로 만든다.

```python
@staticmethod
def latest(reports):
    if isinstance(reports, dict):
        return {k: v for k, v in reports.items() if k in POINTS and v}
    out = {}
    for r in reports or []:
        p = r.get("point")
        if p in POINTS and (p not in out or (r.get("seq") or 0) >= (out[p].get("seq") or 0)):
            out[p] = r
    return out
```

지점마다 `seq`가 가장 큰 보고 하나만 남긴다. 같은 지점에 보고가 여러 번 오면 마지막 것이 이긴다.

선택형의 답은 "몇 번째 보기"로 기록되는데, 판정은 보기 ID로 한다. 그 변환도 여기서 한다.

```python
@staticmethod
def value_of(item, page):
    st = page_status(page)
    if st == "skip":
        return SKIP
    if st == "unknown":
        return UNKNOWN
    v, opts = str(page.get("userAnswer") or ""), [o for o in item.get("options") or [] if isinstance(o, dict)]
    m = re.match(r"(\d+)/(\d+)(?: \(틀림 (\d+)\))?$", v) if item.get("type") == "match" else None
    if m:
        return {"done": int(m.group(1)), "total": int(m.group(2)), "miss": int(m.group(3) or 0)}
    return opts[int(v) - 1]["id"] if opts and v.isdigit() and 1 <= int(v) <= len(opts) else v
```

건너뜀과 "모르겠음"은 서로 다른 값으로 남긴다. 그래서 `is_correct`가 둘 다 오답 처리하더라도, 리포트에는 "안 풀었다"와 "모른다고 했다"가 구분되어 전달된다. 짝짓기는 `"3/5 (틀림 2)"` 같은 문자열을 `done`, `total`, `miss`로 풀어 놓는다.

LLM 호출 앞에 이런 단계가 두꺼울수록 LLM이 받는 입력은 단순해진다. 판정은 이미 끝났고, 시간·상태·값이 정리된 표가 들어간다. LLM이 할 일은 그 표를 해석하는 것으로 좁혀진다.

## 확인이 안 된 것들

읽으면서 불편했던 지점이 몇 개 있다. 아직 해결하지 못했다.

**선개념·오개념의 출처.** 리포트에 나오는 선개념과 오개념은 교육과정을 조회해서 검증한 값이 아니다. 교사가 입력했거나 모델이 추론한 값이다. 위 `signal` 함수가 오개념 ID를 문항에 붙은 데이터에서 읽는다는 점은 확인했지만, 그 데이터가 어떻게 만들어졌는지는 코드에서 알 수 없다. 리포트가 이 값을 사실처럼 쓰면 검증 안 된 입력 위에 문장이 쌓인다.

**캐시 기준.** 비전 분석은 캐시되는데, 캐시 키(fingerprint)가 무엇으로 만들어지는지 완전히 확인하지 못했다. 문제의 초기 장면을 복원할 때 학생이 남긴 조작이 섞여 들어가면, 같은 문제인데도 fingerprint가 갈라질 수 있다. 그러면 같은 문제를 학생 수만큼 다시 분석하게 되고, 결과도 서로 달라질 수 있다. 일어난다고 확인한 게 아니라 일어날 수 있는 경로로 보인다는 말이다. 재현 실험은 아직 하지 않았다.

**품질을 어떻게 잴 것인가.** 이 파이프라인이 좋은 리포트를 만드는지는 코드로 알 수 없다. 사람이 평가한 기준 데이터와 비교해야 한다. 지금 내가 가진 건 구조에 대한 이해뿐이고, 정확도나 일치율 같은 측정값은 없다. 그래서 이 글에도 수치가 없다.

다음에 볼 것은 캐시 키를 만드는 부분이다. 거기서 학생 조작이 실제로 빠지는지부터 확인하려 한다.