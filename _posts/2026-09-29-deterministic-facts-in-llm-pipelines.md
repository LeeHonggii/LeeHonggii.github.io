---
title: "LLM이 채점해도 숫자는 흔들리지 않게: 코드와 모델의 역할 나누기"
date: 2026-09-29 17:00:00 +0900
categories: [LLM]
tags: [LLM, 평가파이프라인, 재현성, 프롬프트설계]
---

먼저 밝혀둔다. 이 글은 새 평가 엔진을 만들어서 검증한 기록이 아니다. 기존 AI 평가 파이프라인의 코드를 읽으면서 "이 값은 누가 정하는가"를 하나씩 추적한 학습 노트다. 새 시스템을 구상하는 단계여서 구현물도, 그 결과 수치도 아직 없다.

## 순서부터 그렸다

읽은 파이프라인은 초등 수학 활동을 분석해 학생 이해도를 판정하고 리포트를 쓰는 구조다. 단계는 이렇게 이어진다.

1. 문제 이미지를 해석해 문제 정의를 만든다
2. 학생의 활동 이력을 문장으로 바꾼다
3. 평가 입력을 조립한다. 이 단계에서 코드가 계산하는 값이 나온다
4. LLM이 평가한다
5. 코드가 확정한 값을 응답에 덮어쓴다 (`applyCodeFacts` 라는 이름의 단계다)
6. 그 결과로 리포트를 서술한다

처음엔 4번이 중심이라고 생각했다. 코드를 따라가 보니 무게중심은 3번과 5번에 있었다. 정답 여부처럼 계산으로 나오는 값, 조작 횟수처럼 세면 되는 값은 LLM에게 묻지 않는다. 코드가 먼저 정하고, LLM 응답에 그 값이 다르게 들어 있으면 코드 값으로 되돌린다.

5번의 실제 코드는 이 글에 싣지 않았다. 내가 읽은 범위에서 옮길 수 있는 것은 구조 설명뿐이고, 없는 함수 본문을 채워 넣고 싶지 않았다.

## 입력을 섞으면 모델도 섞인다

문제 정의 단계에서 눈에 띈 설계가 하나 있다. "문제가 무엇인가"를 정의하는 호출과 "학생 답안을 대조하는" 호출을 나눠 놓았다. 초기 문제 이미지와 모범답안 이미지가 각각 어떤 역할인지도 프롬프트에 명시한다.

이미지 두 장을 그냥 넘기면 모델이 모범답안 이미지를 학생 답안처럼 읽을 여지가 생긴다. 역할을 적어두는 건 사소해 보이지만 입력 혼동을 줄이는 가장 싼 방법이다. 다만 혼동이 실제로 얼마나 줄었는지는 재보지 않았다.

캐시 이야기도 나온다. 정적 프롬프트를 앞에 두는 건 호출 비용을 줄이려는 최적화다. 앞부분이 같으면 모델 쪽에서 재사용된다. 문제 정의 결과를 저장해 두는 캐시는 이것과 별개다. 목적이 다르고 재사용 단위도 다르다. 하나는 프롬프트 접두부, 다른 하나는 문제 하나의 해석 결과다. 처음엔 둘을 같은 것으로 뭉뚱그려 읽었다가 코드에서 저장 위치가 다른 걸 보고 구분했다.

## 코드는 "믿는 값"이 아니라 "흔들리는 값"을 재는 데도 쓴다

저장소에는 평가 로직 옆에 프로브(probe) 스크립트가 있다. 같은 입력으로 LLM 호출을 N번 반복하고, 정답셋과 비교해 얼마나 흔들리는지 재는 도구다. 여기서 "코드가 소유하는 필드"와 "모델이 추론하는 필드"의 검증이 왜 달라야 하는지가 보였다.

개념 파악 프로브의 채점 함수다. 식별자는 일반화했고 로직은 그대로다.

```python
GOLD = {
    "core": "S_ADD2",
    "prereq_measurable": "S_PV",
    "prereq_not_measurable": "S_ADD1",
    "must": {"M081", "M083"},
    "nobs_must": {"M003", "M004", "M005"},
}

def score(d):
    sk = {s["id"]: s for s in d.get("skills") or []}
    core = [s["id"] for s in d.get("skills") or [] if s.get("role") == "core"]
    obs = {m["id"] for m in d.get("misconceptions") or []}
    nobs = {m["id"] for m in d.get("not_observable") or []}
    return {
        "core_ok": core == [GOLD["core"]],
        "pv_prereq_and_measurable": sk.get("S_PV", {}).get("role") == "prereq"
                                     and sk.get("S_PV", {}).get("measurable") is True,
        "meaning_prereq_not_measurable": sk.get("S_ADD1", {}).get("measurable") is False,
        "must_misconceptions": GOLD["must"] <= obs,
        "must_missing": ",".join(sorted(GOLD["must"] - obs)),
        "nobs_three": GOLD["nobs_must"] <= nobs,
        "n_candidates": len(obs),
        "n_not_observable": len(nobs),
    }
```

이 채점이 문자열 유사도나 "그럴듯한가"를 보지 않는다는 점이 마음에 들었다. 본개념이 정확히 하나인지, 어떤 선행개념이 관측 가능(measurable)인지, 반드시 나와야 하는 오개념이 집합 포함(`<=`)으로 들어 있는지를 본다. 전부 불리언이나 집합 연산이라 채점 자체는 흔들리지 않는다. 흔들리는 건 채점 대상인 LLM 출력뿐이다.

`measurable` 플래그도 눈여겨봤다. 입력 데이터로 관찰할 수 없는 개념(예: 덧셈의 의미 이해)을 "관측 불가"로 분리해 두었고, 프로브는 그것이 관측 가능으로 둔갑하지 않는지 확인한다. 관찰 한계를 명시하는 건 새 시스템을 구상할 때 가장 가져가고 싶은 부분이다. 근거가 없는 것에 판정을 내리지 않는 구조다.

반복 호출 쪽은 이렇게 생겼다.

```python
for model in MODELS:
    for i in range(1, REPEAT + 1):
        t0 = time.time()
        d = DraftTaskModel(TASK, grade="auto", model=model, save=False)
        r = d.result
        sec = round(time.time() - t0, 1)
        row = {"model": model, "run": i, "sec": sec,
               "calls": (r.get("usage") or {}).get("calls"),
               "format_errors": len(r.get("validation_errors") or [])}
        if r.get("draft"):
            row.update(score(r["draft"]))
            save_json(OUT / f"{model}_{i}.json", r["draft"])
        else:
            row.update({"core_ok": False, "must_misconceptions": False, ...})
        rows.append(row)
        save_json(OUT / "rows.json", rows)   # 매 회 저장
```

`save=False`로 서비스와 같은 코드 경로를 저장 없이 타게 한 점, 초안이 아예 없을 때 예외로 죽지 않고 전부 실패(False)로 기록하는 점이 실무적이다. 한 번 실패해도 나머지 반복이 계속 돌아간다. 결과 문서에도 "같은 모델 3회가 다르면 그만큼 흔들리는 것이다. 한 번 결과로 모델을 고르지 않는다"라는 읽는 법이 붙어 있었다. 모델 비교를 1회 실행으로 끝내지 않는다는 원칙이다.

## 판정 쪽은 더 좁게 잰다

답안 판정은 출력이 `{verdict: 정답|부분|오답, basis(근거), confidence}` 꼴이다. 프로브는 케이스마다 기대 판정을 정해두고 반복 호출해 일치율을 본다.

```python
EXPECT = {
    "solves_directly_correct": "correct",
    "solves_directly_wrong":   "wrong",
    "good_process_correct":    "correct",
    "good_process_wrong":      "wrong",
}

for c in CaseStore.list(TASK):
    ev = EvidenceBundle(cid, CaseStore.get(TASK, cid), model_answer)
    for i in range(1, REPEAT + 1):
        j, call, err = ask_verdict(ev, pdef, None, MODEL, TASK, cid)
        rows.append({
            "case": cid, "run": i, "expect": EXPECT.get(cid, "?"),
            "verdict": j["verdict"],
            "match": j["verdict"] == EXPECT.get(cid),
            "n_basis": len(j.get("based_on") or []),
            "fail": (err or {}).get("code", ""),
            "confidence": j.get("confidence"),
        })
```

케이스 이름이 재밌다. "정답을 바로 맞춤", "활동은 잘했지만 틀림" 같은 식이다. 과정과 결과가 갈라지는 조합을 일부러 넣어서, 모델이 과정에 끌려 결과 판정을 바꾸는지 보려는 설계로 읽었다. 이건 코드로 계산해서 확정하는 정오와 별개로, LLM 판정이 코드 정답과 어긋나지 않는지 보는 안전망이다. 판정 고정(verdict pinning)을 끄고 재는 이유도 여기서 이해됐다. 고정된 채로 재면 모델이 아니라 고정 로직을 재게 된다.

`n_basis`(근거 수)와 `fail`(근거 인용 실패)을 따로 기록하는 것도 같은 맥락이다. 판정이 맞았는지와 근거를 제대로 댔는지는 서로 다른 검증이다.

## 보정했다고 안심하면 안 된다

이 구조를 읽고 나서 가장 오래 남은 생각은 이것이다. 코드 값으로 응답을 덮어쓰면 그 필드는 맞는다. 하지만 전체 평가가 맞다는 뜻은 아니다. 정답 여부가 코드 값으로 고쳐졌는데 옆에 있는 "왜 틀렸는가" 서술이 원래의 잘못된 판정을 전제로 쓰여 있다면, 숫자와 문장이 서로 어긋난 리포트가 나온다.

그래서 검증도 둘로 나눠야 한다고 정리했다.

- 계산 필드: 정오, 조작 수. 코드가 소유하므로 단위 테스트로 충분하다. 응답과 어긋나면 덮어쓰기가 작동하는지만 보면 된다
- 추론 필드: 오개념 후보, 이해도 서술. 정답셋과 반복 실행으로 재야 한다. 위 프로브가 하는 일이 이것이다

새로 구상 중인 시스템은 여러 활동 유형을 지원해야 한다. 유형을 어떻게 정의할지, 입력 데이터로 실제로 관찰할 수 있는 게 무엇인지, 전문가 검증을 어디에 넣을지는 아직 초안 단계다. 특히 정답셋 자체를 누가 만들고 검증하느냐가 걸린다. 위 프로브의 `GOLD` 는 사람이 정한 값이라, 그 값이 틀렸으면 재현성이 아무리 높아도 소용이 없다.

프로브 결과 수치(모델별 일치율 등)는 이 글에 싣지 않았다. 내가 직접 돌려 확인한 값이 아니기 때문이다. 다음에 볼 것은 두 가지다. 코드 확정값을 덮어쓴 뒤 리포트 서술이 그 값과 일관되게 나오는지 확인하는 단계가 기존 파이프라인에 있는지 살펴보고, 없다면 새 설계에서 어디에 넣을지 정해야 한다.