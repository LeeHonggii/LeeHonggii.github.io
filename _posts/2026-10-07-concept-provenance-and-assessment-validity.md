---
title: "AI는 선개념과 오개념을 어떻게 아는가: 추론과 검증의 차이"
date: 2026-10-07 17:00:00 +0900
categories: [AI]
tags: [LLM, 평가설계, 교육AI, 출처관리, 코드리딩]
---

기존 LLM 분석 코드를 읽는 중이었다. 학생이 활동 화면에 남긴 기록을 모델에 넣고 선생님용 리포트를 뽑는 파이프라인이다. 실행 순서를 따라가다가 질문 하나가 걸렸다.

리포트에 "이 문제는 ○○ 개념이 필요하다", "학생이 △△ 오개념을 보인다" 같은 문장이 나온다. 이 개념 목록과 오개념 규칙은 어디서 왔나.

내가 읽은 범위에서 교육과정 자료를 조회하는 경로는 보지 못했다. 그렇다면 그 목록은 모델이 사전학습으로 알고 있는 것에서 나온 가설이다. 그럴듯하지만 검증된 지식은 아니다. 이 글은 그 차이를 코드에서 어떻게 다루고 있었는지 읽은 기록이다. 정리하면서 새 평가 구조를 구상했는데, 그건 구현하지 않았고 마지막에 따로 적는다.

## 입력에 출처 꼬리표가 붙어 있었다

페이지 하나(문제 하나)를 모델에 넘기는 입력 조립부다. 식별자는 일반화했다.

```python
def build_input(self):
    ev = self.ev
    tools = sorted({e["name"] for e in ev.elements if e.get("tool") and e.get("name") and e.get("kind") not in ("text", "background")})
    lines = [f"<활동> {self.title}", f"<지시문> {html.unescape(ev.instruction())}", f"<쓰인 교구> {', '.join(tools) or '없음'}",
             f"<모범답안> {'있음 (이미지 2)' if self.has_model else '없음'}", "<화면 글> 위에서 아래 순"]
    lines += [f"- {html.unescape(t['text'])}" for t in ev.texts()]
    way = SolutionWay.confirmed(self.task_id, self.ev.order)
    if way:
        lines.append("<선생님이 확인한 풀이 방식 기준> " + json.dumps(way, ensure_ascii=False))
    else:
        groups, status = SolutionWay.answers(self.task_id, self.ev.order)
        if groups:
            lines.append(f"<정답 목록 — 학생 답을 보기 전에 따로 구한 것({status}, 선생님 미확인)> " + json.dumps(groups, ensure_ascii=False))
            lines.append("최종 답이 정답 목록에 있는지 하나씩 맞춰 보고 판정한다.")
    lines.append("<사실> 번호 · 내용")
    lines += [f"[{f['alias']}] {f['text']}" for f in ev.facts if f["code"] not in SKIP_FACTS]
    # ...
```

`SolutionWay.confirmed`가 값을 돌려주면 `<선생님이 확인한 풀이 방식 기준>`으로 들어간다. 없으면 `answers`로 따로 구한 정답 목록이 들어가는데, 꼬리표가 `선생님 미확인`이다. 같은 "기준"인데 출처에 따라 이름이 다르다.

이 분기가 마음에 든 이유는 모델이 읽는 입력 안에서 확인된 것과 확인 안 된 것이 구분된다는 점이다. 전문가가 확인했다고 모델에게 말해 주는 것과, 그냥 기준을 던져 주는 것은 다르다.

다만 한계도 보인다. `answers`가 정답 목록을 어떻게 만드는지는 내가 읽은 부분에서 확인하지 못했다. "학생 답을 보기 전에 따로 구했다"는 순서의 보장이고, 정답이라는 보장은 아니다. 이 부분은 확인이 필요하다.

## 정답 여부는 코드가 대조한다

모델에게 맡기지 않고 코드로 옮긴 판정이 있다. 주석에 이유가 적혀 있다.

```python
def answer_groups(self):
    groups, _ = SolutionWay.answers(self.task_id, self.ev.order)
    norm = lambda t: re.sub(r"\s|=.*", "", str(t))
    out = []
    for g in groups or []:
        exprs = {norm(a) for a in g.get("답") or [] if re.fullmatch(r"\d+[+\-×÷]\d+", norm(a))}
        if exprs:
            out.append(exprs)
    return out

# 정답 목록에 식이 있고 학생 화면에 카드 식이 있으면, 정답 여부는 코드가 대조한다(AI 의 「가장 큰/작은」 비교가 실행마다 흔들림)
def verdict_by_code(self, p):
    groups, mine = self.answer_groups(), self.exprs("CARD_EXPR")
    if not groups or not mine:
        return p
    hits = [any(e in g for e in mine) for g in groups]
    code = "정답" if all(hits) else "부분 정답" if any(hits) else "오답"
    # ... (이하 생략)
```

주석대로 "가장 큰 식, 가장 작은 식" 같은 비교를 모델에 시키면 실행마다 답이 달랐다고 한다. 그래서 `숫자 연산자 숫자` 꼴로 정규화되는 식만 집합으로 만들어 정답 그룹과 대조한다. 모든 그룹에 걸리면 정답, 일부만 걸리면 부분 정답, 아니면 오답이다.

이 함수가 하는 일과 안 하는 일을 구분하는 게 이 글의 요점이다. 코드가 확정하는 것은 학생 화면의 식이 정답 목록에 있느냐다. 계산으로 얻은 사실이라 재현된다. 하지만 학생이 그 개념을 이해해서 그 식을 만들었는지는 이 함수가 건드리지 않는다. 우연히 맞았을 수도 있고, 옆 친구 것을 따라 했을 수도 있다.

적용 조건도 좁다. 정답 목록에 식이 있고, 학생 화면에 카드 식이 있고, 정규식에 맞아야 한다. 하나라도 빠지면 `return p`로 모델의 판정이 그대로 남는다. 그러니 이 파이프라인에서 "코드가 판정했다"는 일부 페이지에만 해당한다.

## 개념과 이해도는 다른 질문이다

여기서 문제를 둘로 나눠 보게 됐다.

- 이 문제를 풀려면 어떤 개념이 필요한가
- 이 학생이 그 개념을 이해했는가

앞의 질문은 문제 쪽 속성이다. 모델이 사전학습 지식으로 답하면 가설이고, 교육과정 자료를 조회하거나 전문가가 입력하면 근거가 생긴다. 뒤의 질문은 학생 쪽 속성이다. 학생 기록에서 관찰할 수 있는 것(식, 소요 시간, 조작 횟수, 고친 흔적)과 연결돼야 답이 나온다.

전문가가 개념 이름을 입력해 주면 앞의 질문은 나아진다. 하지만 뒤의 질문은 그대로다. "분배법칙을 안다"는 말은 화면 어디에서 무엇이 보이면 참인지 정해져 있지 않으면 검증할 수 없다. 개념 이름이 관찰 기준과 이어지지 않으면 이름표만 맞는 상태가 된다.

## 출처를 남기는 것의 한계

읽은 코드는 출처를 꽤 신경 쓴다. 캐시 키에도 드러난다.

```python
prompt = read_prompt(PROMPT)
self.system = prompt
self.fingerprint = hashlib.sha256((prompt + self.text).encode("utf-8")).hexdigest()[:16]

def load(self):
    p = self.path()
    if not p.exists():
        return None
    d = load_json(p)
    if d.get("fingerprint") != self.fingerprint:
        return None
    self.relation_by_code(d["result"])
    return d
```

프롬프트와 입력 텍스트를 묶어 해시한다. 프롬프트가 바뀌거나 선생님 확인이 추가되어 입력 줄이 바뀌면 저장본은 버려진다. 어떤 기준으로 만든 결과인지가 결과와 함께 움직이는 셈이다.

반면 `ActivityReport.one_line`은 반대 방향의 위험을 보여 준다.

```python
NEEDS_HELP = ("오답", "부분 정답")

@staticmethod
def one_line(rows):
    done = [r for r in rows if all(p["판정"] for p in r["pages"])]
    if not done:
        return "아직 AI 분석 전입니다."
    help_ = [r["student"] for r in done if any(p["판정"] in NEEDS_HELP or p["확인_필요"] for p in r["pages"])]
    line = f"{len(done)}명 중 " + (f"{', '.join(help_)} 학생을 먼저 확인해 보세요." if help_ else "모두 정답에 도달했습니다.")
    if len(done) < len(rows):
        line += f" (분석 전 {len(rows) - len(done)}명)"
    return line
```

판정이 "오답", "부분 정답"이거나 `확인_필요`가 켜진 학생만 "먼저 확인"으로 올린다. 분석 전인 학생 수도 따로 밝힌다. 이 점은 정직하다. 모델 판정이 사람에게 넘어가는 지점에 `확인_필요`라는 안전판이 있다.

그래도 한 가지는 남는다. 출처가 기록돼 있다는 것과 판정이 교육적으로 타당하다는 것은 다른 문제다. 전문가 입력, 모델 생성, 코드 판정을 전부 구분해서 저장해도, "이 판정 기준이 학생의 이해를 재는 데 맞는가"는 그 구분만으로 답이 나오지 않는다. 출처 기록은 나중에 따져볼 수 있게 해 줄 뿐이다. 따져보는 일은 따로 해야 한다.

## 새 구조를 구상하면서 정리한 것

이 코드는 수학 활동 하나에 맞춰져 있다. 여러 활동 유형을 지원하는 평가 구조를 생각하면서 먼저 필요하다고 적어 둔 것은 세 가지다.

- 활동 유형별 정의
- 학생 기록에서 관찰 가능한 근거
- 전문가가 같은 기록에 내린 판정 데이터

마지막이 없으면 모델 판정이 맞는지 비교할 기준이 없다. 영상 기반 분석과 평가 품질 측정은 방향만 논의했고, 새 엔진을 구현하거나 성능을 재본 것은 없다. 현재 코드에서도 전문가 판정과 모델 판정이 얼마나 일치하는지는 아직 재보지 않았다.

지금 가장 애매한 것은 `선생님 미확인` 꼬리표가 붙은 정답 목록이다. 이 목록이 실제로 얼마나 틀리는지 모르는데, 코드가 이 목록을 정답 판정의 기준으로 쓴다. 샘플을 뽑아 선생님께 확인받는 일부터 해야 할 것 같다. 그 결과를 보기 전에는 이 판정의 신뢰도에 대해 말할 수 없다.