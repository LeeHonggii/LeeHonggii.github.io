---
title: "버튼은 눌렀는데 기록은 없다: UI 자동화가 놓친 상태 변화"
date: 2026-10-16 17:00:00 +0900
categories: [Frontend]
tags: [frontend, event-sourcing, undo-redo, e2e-testing, debugging]
---

화면에서는 분명히 동작했다. 그런데 저장된 이력을 재생하면 그 장면이 나오지 않았다.

조작 이력으로 학습 과정을 분석하는 기능을 고치다가 이 간극을 만났다. 이 글은 그 간극을 어떻게 좁혔는지, 그리고 아직 못 좁힌 부분이 어디인지에 대한 기록이다.

## 시작은 분석 쪽이었다

화면 위 객체(블록, 도형, 필기)를 조작한 이력을 분석하고 있었다. 객체의 외형 식별자만 보고는 실제 수량이나 상태를 읽을 수 없었다. 같은 모양이라도 안에 담긴 값이 다를 수 있기 때문이다. 그래서 데이터 사전을 기준으로 판독하도록 바꾸고, 재생 로직도 같이 손봤다.

기존 데이터와 수정 후 데이터를 나란히 놓고 비교해서 하위 호환성과 영향 범위는 확인했다. 한 가지 불안이 남았다. 분석 코드가 읽는 이력이 **실제 화면에서 일어난 일을 전부 담고 있는가**였다. 이력에 없는 조작은 아무리 분석 로직을 고쳐도 복원되지 않는다.

그래서 시선을 분석 서버에서 프런트엔드로 옮겼다.

## 이력이 두 가지 모양으로 있다

재생 쪽 코드부터 보면 사정이 이렇다. 저장된 이력에는 구버전(legacy)과 신버전(hv3, 바뀐 값만 담는 증분 형식)이 섞여 있다. 압축된 항목도 있다.

```python
class HistoryUnwrap:
    @staticmethod
    def items(history_stack):
        items, warns = [], []
        for h in history_stack or []:
            z = h.get("z") if isinstance(h, dict) else None
            if isinstance(z, str):
                try:
                    items.extend(json.loads(gzip.decompress(base64.b64decode(z)).decode("utf-8")))
                except Exception as e:
                    warns.append(f"압축 해제 실패(__hv={h.get('__hv')}): {e}")
                continue
            if isinstance(h, dict):
                items.append(h)
        return items, warns

    @staticmethod
    def is_legacy(items):
        return not any(isinstance(x, dict) and ("b" in x or "a" in x) for x in items)
```

압축 해제에 실패해도 예외를 던지지 않고 경고로 모은 뒤 계속 진행한다. 이력 하나가 깨졌다고 세션 전체 분석이 죽으면 안 된다고 봤다. 대신 경고가 결과에 같이 나가야 한다. 조용히 삼키면 "이력이 비어 있다"와 "이력을 못 읽었다"가 구분되지 않는다.

`is_legacy`는 `b`/`a` 키(before/after의 축약형)가 하나라도 있으면 hv3로 본다. 형식 판별에 버전 필드를 믿지 않고 모양을 본 이유는, 두 형식이 한 스택에 섞여 있을 수 있어서다.

## "after에 없다 = 지웠다"가 아니다

증분 형식을 legacy 모양으로 펼치는 코드에는 이런 주석이 붙어 있다.

```python
# deleteAndCreate 의 before 에는 판에 담긴 블록이 통째로 실리고 after 에는 바뀐 것만 실린다.
# 그래서 after 에 없다고 지운 것이 아니다 — 다음에 나올 때 before 쪽(그때 있던 것)이면 남아 있던 것이다
@staticmethod
def survives(i, later, final_ids):
    for b_ids, a_ids in later:
        if i in b_ids:
            return True
        if i in a_ids:
            return False
    return i in final_ids
```

한 이벤트의 before/after가 서로 대칭이 아니라는 게 문제였다. before에는 전체가, after에는 변경분만 실린다. 그러니 "after에 없음"만으로는 삭제와 단순 미변경을 구별할 수 없다.

`survives`는 해당 객체가 **뒤따르는 이벤트에서 처음 어느 쪽에 나오는지**를 본다. before 쪽에 먼저 나오면 그 시점에 있었다는 뜻이니 살아 있던 것이다. after 쪽이 먼저면 새로 만들어진 것이니 그 전에는 없었다. 끝까지 안 나오면 최종 저장 상태(`final_ids`)가 판정한다.

이 판정은 이력만으로는 닫히지 않는다. 마지막에 저장된 화면을 기준으로 삼아야 한다.

## 되돌리기는 이력을 지우지 않는다

재생에서 가장 헷갈렸던 건 되돌리기(undo)다. 학생이 되돌린 조작을 이력에서 어떻게 다룰지 두 가지 방식이 공존한다.

```python
def get_items(self):
    self.items, self.warnings = HistoryUnwrap.items(self.state.get("historyStack"))
    self.legacy = HistoryUnwrap.is_legacy(self.items)
    cursor = self.state.get("studentCursor")
    self.cursor = cursor if isinstance(cursor, int) else len(self.items)
    self.format = "legacy" if self.legacy else "hv3"
    if not self.legacy and self.items:
        # studentCursor 뒤는 학생이 되돌려 버린 조작이라 재생하지 않는다
        self.reverted = max(0, len(self.items) - self.cursor)
        final_ids = {e.get("id") for e in [*(self.state.get("elements") or []), *(self.state.get("penElements") or [])] if isinstance(e, dict)}
        self.items = Hv3History.to_legacy(self.items[:self.cursor], final_ids)
        self.legacy = True
    return self.items
```

hv3에서는 이력 스택이 끝까지 남아 있고, 커서(`studentCursor`)가 "지금 어디까지 유효한가"를 가리킨다. 커서 뒤는 되돌려진 조작이라 잘라낸다. 반면 legacy는 되돌리기 자체가 `subType=undo` 이벤트로 기록되고, 재생기는 직전 장면 스냅샷으로 복원한다.

여기서 이 글의 주제와 만난다. **조작이 일어났는데 이벤트가 안 남으면, 이 모든 정교한 재생이 소용없다.** 재생기는 기록된 것만 재생한다.

## 코드를 읽다가 생긴 질문

프런트엔드 핸들러를 따라가며 "상태를 바꾸고, 기록 함수를 부른다"는 쌍을 찾았다. 쌍이 깨진 곳, 즉 상태는 바꾸는데 기록 호출이 없는 경로가 의심 지점이다.

그리고 두 가지를 구분해서 적기로 했다. 코드를 읽고 의심한 것과, 브라우저에서 실제로 재현한 것이다. 코드만 봐서는 호출 경로가 런타임에 실제로 타는지 모른다. 반대로 브라우저에서 한 번 안 되는 걸 봤다고 원인이 그 줄이라고 단정할 수도 없다. 둘이 맞물릴 때만 "버그"로 적었다.

브라우저 자동화 시험에서 놓친 조건도 정리됐다. 시험이 **빈 객체**로만 돌고 있었다.

복사류 기능은 원본에 값이 있어야 의미가 있다. 값을 입력하고, 편집을 끝내고(포커스를 빼서 확정시키고), 복사 대상이 몇 개인지 세고, 애니메이션이 끝나길 기다린 다음에야 결과가 보인다. 빈 객체에서 버튼을 누르면 "오류 없이 지나갔다"는 결과만 나온다. 그 통과가 아무것도 검증하지 않은 통과라는 점이 문제였다.

연결 화살표 같은 기능도 비슷했다. 필드 이름이 `link`나 `connect` 비슷하면 "원본을 바꾸면 따라 바뀌겠지" 하고 추정하게 된다. 실제 구현이 한 번 복사하고 끝나는 것인지, 관계를 저장해 두고 이후 변경을 전파하는 것인지는 코드에서 확인해야 했다. 이름은 증거가 못 된다.

## 재생이 맞는지 재생 스스로 검사하게 했다

이력 누락을 잡는 가장 값싼 방법은 이력을 처음부터 재생한 결과를 **저장된 최종 화면**과 비교하는 것이다.

```python
def check_final(self):
    if not self.replayable:
        return self.warnings
    for label, mine, stored in (("교구 요소", self.final["elements"], self.state.get("elements")),
                                ("필기", self.final["pens"], self.state.get("penElements"))):
        a, b = {e.get("id") for e in mine}, {e.get("id") for e in stored or []}
        if a != b:
            self.warnings.append(f"{label}: 이력 재생 {len(a)}개 · 저장된 최종 화면 {len(b)}개 (이력에만 {len(a - b)} · 저장
# ... (이하 생략)
```

화면 상태, 저장 스냅샷, 조작 이력은 서로 다른 정보원이다. 이 함수는 앞의 둘을 이력으로 재구성한 결과와 대조한다. 어긋나면 어느 방향으로 얼마나 어긋났는지를 경고로 남긴다. 이력에만 있는 것은 "지워졌어야 하는데 안 지워진 기록"이고, 저장 화면에만 있는 것은 "기록 없이 생긴 객체"일 가능성이 있다.

한계도 이 코드에 그대로 보인다. 비교 단위가 **id 집합**이다. 객체가 그대로 있되 안의 값만 바뀌었는데 그 변경이 기록되지 않았다면 이 검사는 통과한다. 수량이나 상태를 읽는 분석이 목적이니 정작 값 쪽이 필요하다. 값 단위 대조는 아직 붙이지 않았다.

## 아직 남은 것

- 수정한 재생 결과와 실제 화면을 눈으로 대조하는 작업이 남아 있다. 수정 전후 데이터 비교는 끝났지만 "화면과 같은가"는 별개의 질문이다.
- 데이터 사전 기반 판독에서 일부 해석 기준은 아직 검증하지 못했다. 어느 쪽이 맞는지 원본 화면으로 확인해야 한다.
- 기록 호출이 빠진 것으로 의심한 경로 중 브라우저에서 재현을 마친 것과 코드 의심에 그친 것은 목록을 따로 관리하고 있다. 이 글에서는 그 개수를 세지 않았다. 재현 못 한 의심을 버그라고 부르고 싶지 않았다.
- `check_final`을 id 집합이 아닌 값 해시로 비교하게 바꿀지는 고민 중이다. 비교 비용이 얼마나 늘어나는지는 아직 재보지 않았다.