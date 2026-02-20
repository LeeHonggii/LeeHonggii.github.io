---

title: "LLM 퀴즈 생성 파이프라인 실전 기록: 재시도·토큰·시간까지 까본 작업 로그"
date: 2025-02-05 09:30:00 +0900
categories: [Daily, LLM]
tags: [pipeline, retry, validation, metrics, self-correction, context-grounding]
---

## 안녕하세요

LLM 기반 퀴즈 생성 시스템을 만들면서 느낀 건 하나였습니다.

> 프롬프트를 잘 쓰는 게 아니라,
> 실패를 어떻게 설계하느냐가 진짜 문제다.

겉으로 보면 “문제 생성 성공률 5/5”라서 다 잘 돌아가는 것처럼 보입니다.
그런데 내부 로그를 까보면 전혀 다른 그림이 나옵니다.

* 재시도는 몇 번 발생했는지
* 실제 소요 시간은 얼마나 차이나는지
* 토큰은 어디에서 많이 쓰이는지
* 검증을 넣으면 왜 갑자기 성공률이 떨어지는지

이번 글은 이걸 **실제 수치 + 코드 중심으로** 정리한 기록입니다.

---

# 1. 성공률은 같아도 내부는 완전히 다르다

Type 2 실험에서 네 가지 설정을 비교했습니다.

| 설정 | 모델         | 프롬프트  |
| -- | ---------- | ----- |
| A  | gpt-4o     | 기본    |
| B  | gpt-4o     | + COT |
| C  | gpt-5-mini | 기본    |
| D  | gpt-5-mini | + COT |

### 평균 결과

| 설정               | 평균 시간 | 평균 시도 | 평균 토큰 | 성공률 |
| ---------------- | ----- | ----- | ----- | --- |
| A (4o 기본)        | 8.5초  | 1.6회  | 5,803 | 5/5 |
| B (4o + COT)     | 14.3초 | 1.2회  | 4,854 | 5/5 |
| C (5-mini 기본)    | 37.5초 | 1.0회  | 5,847 | 5/5 |
| D (5-mini + COT) | 38.1초 | 1.0회  | 6,210 | 5/5 |

성공률은 전부 100%입니다.

그런데 평균 시도 수가 다릅니다.

* gpt-4o: 평균 1.6회 → 내부적으로 재시도 발생
* gpt-5-mini: 평균 1.0회 → 거의 한 번에 통과

성공률만 보면 절대 보이지 않는 차이입니다.

---

# 2. 재시도 로직은 이렇게 돌리고 있었다

실제로 사용한 구조는 단순합니다.

```python
MAX_ATTEMPTS = 3

def generate_with_retry(run_id, model, prompt):
    for attempt in range(1, MAX_ATTEMPTS + 1):
        output, tin, tout, latency = call_llm(model, prompt)

        ok, reasons = validate(output)

        log_event(
            run_id=run_id,
            model=model,
            attempt=attempt,
            latency_ms=latency,
            tokens_in=tin,
            tokens_out=tout,
            success=ok,
            reasons=reasons
        )

        if ok:
            return output

        prompt = patch_prompt(prompt, reasons)

    raise RuntimeError("Failed after retries")
```

여기서 중요한 건 `validate()`입니다.

단순히 JSON 포맷 체크가 아니라:

* 오답이 단순부정인지
* 오답이 실제로 참인지
* 교재에 없는 표현이 들어갔는지
* 구조적 규칙을 위반했는지

를 전부 검사합니다.

---

# 3. COT는 단순부정을 없앴지만, 다른 리스크를 만들었다

gpt-4o 기본에서는 이런 문제가 있었습니다.

* 단순부정 패턴 2/5
* 팩트 오류 1/5

COT를 넣으니:

* 단순부정 0/5
* 팩트 오류 2/5 (Reverse causality 문제)

즉,

> COT는 문제를 없애는 게 아니라
> 문제의 종류를 바꿨다.

특히 Reverse causality 전략이 위험했습니다.
어떤 명제는 역이 참이기 때문에, 오답이 아니라 또 다른 참이 되어버립니다.

그래서 전략 선택에 가드레일을 넣었습니다.

```python
RISKY_STRATEGIES = {"reverse_causality"}

def select_strategy(strategies, allow_risky=False):
    if not allow_risky:
        strategies = [s for s in strategies if s not in RISKY_STRATEGIES]
    return random.choice(strategies)
```

그리고 반드시 “오답이 실제로 참인지”를 별도 체크합니다.

```python
def wrong_choice_is_true(choice, textbook):
    # 간단 예시: 별도 LLM 평가
    verdict = judge_llm(choice, textbook)
    return verdict == "TRUE"
```

---

# 4. 검증을 넣었더니 성공률이 떨어졌다

지문 생성 → 문제 생성 구조에서
중간에 Self-Correction(팩트 검증)을 넣었습니다.

결과는 이랬습니다.

| 설정              | 성공률 |
| --------------- | --- |
| gpt-4o 기본       | 5/5 |
| gpt-4o + 검증     | 0/5 |
| gpt-5-mini 기본   | 5/5 |
| gpt-5-mini + 검증 | 2/5 |

처음엔 “검증이 문제인가?” 싶었습니다.

그런데 로그를 보니:

* gpt-4o는 교재에 없는 표현을 거의 항상 추가
* 검증이 그걸 전부 잡아냄

즉,

> 성공률이 떨어진 게 아니라
> 기준이 올라간 것.

---

## 문제는 재시도 루프가 없었다는 것

당시 구조는:

* 생성
* 검증
* 실패 → 종료

이러니 당연히 0/5가 나옵니다.

그래서 이렇게 바꿨습니다.

```python
def generate_with_validation(model, prompt, textbook):
    for attempt in range(1, 4):
        output = call_llm(model, prompt)

        ok, reasons = validate_passage(output, textbook)

        if ok:
            return output

        prompt += "\n# Fix the following issues:\n"
        prompt += "\n".join(reasons)

    raise RuntimeError("Validation failed 3 times")
```

검증은 “종료 장치”가 아니라
“수정 유도 장치”가 되어야 합니다.

---

# 5. explanation이 가장 위험했다

흥미로운 점:

* passage 외부지식 0%
* explanation 외부지식 100% (gpt-4o)

즉, 지문은 교재에 anchored 되어 있는데,
해설은 자유롭게 확장됩니다.

그래서 explanation 전용 검증을 따로 둡니다.

```python
def validate_explanation(text, textbook):
    external_terms = find_external_terms(text, textbook)
    if external_terms:
        return False, ["EXTERNAL_KNOWLEDGE"]
    return True, []
```

그리고 재생성 시 강하게 제한합니다.

```python
prompt += """
# Constraint
Explanation must only use concepts explicitly present in textbook.
Do not introduce new theorems or external knowledge.
"""
```

gpt-5-mini는 이 제약을 거의 완벽히 따랐고,
gpt-4o는 여전히 외삽하려는 경향이 있었습니다.

---

# 6. 시간·토큰·재시도 분석은 반드시 필요하다

성공률만 보면 다 잘 되는 것처럼 보입니다.

하지만 실제로는:

* gpt-4o: 빠름 + 재시도 있음 + 외부지식 경향
* gpt-5-mini: 느림 + 재시도 거의 없음 + 교재 준수 + 다양성 낮음

이건 로그를 집계해봐야 보입니다.

```python
def summarize(events):
    summary = {}

    for e in events:
        m = e["model"]
        summary.setdefault(m, {
            "latency": [],
            "tokens_out": [],
            "attempts": [],
            "fails": 0
        })

        summary[m]["latency"].append(e["latency_ms"])
        summary[m]["tokens_out"].append(e["tokens_out"])
        summary[m]["attempts"].append(e["attempt"])

        if not e["success"]:
            summary[m]["fails"] += 1

    return summary
```

이걸 보고 나서야 모델 선택 기준이 바뀌었습니다.

> “정확도”가 아니라
> “성향과 비용” 문제라는 걸.

---

# 정리

이번 작업에서 확실해진 건 이겁니다.

1. 검증을 넣으면 성공률은 당연히 떨어진다.
2. 그래서 재시도 루프는 필수다.
3. 오답 생성은 전략 + 참오답 검사 없으면 위험하다.
4. explanation은 가장 통제하기 어려운 영역이다.
5. 모델 선택은 성능이 아니라 성향 선택이다.

이제는 프롬프트를 더 깎는 게 아니라,
재시도·검증·메트릭이 있는 파이프라인을 다듬는 단계로 넘어가고 있습니다.
