---
title: "그림 ID는 같은데 값은 다르다: 상태 판독을 데이터 사전으로 옮기는 법"
date: 2026-10-16 17:00:00 +0900
categories: [Backend]
tags: [상태판독, 데이터사전, 이력재생, 회귀검증, 파이썬]
---

학생이 교구를 조작한 이력을 분석하는 리포트 서버를 만지고 있었다. 편집기 화면에는 블록이 놓이고, 블록끼리 합쳐지고, 쪼개진다. 서버는 이 이력에서 "학생이 지금 무엇을 만들어 놓았나"를 읽어내야 한다.

처음 가정은 단순했다. 블록마다 그림을 가리키는 식별자(`svgId`)가 있으니, 이 식별자로 "이건 낱개, 이건 묶음"을 판독하면 된다고 봤다. 이 가정이 깨졌다.

## 같은 그림인데 값이 다르다

합치기나 쪼개기는 기존 블록을 고치지 않는다. 새 객체를 만든다. 그런데 새 객체의 외형 식별자는 원래 블록과 같을 수 있다. 화면에는 똑같은 그림이 놓여 있다. 그 블록이 낱개 하나를 뜻하는지 여러 개를 합친 값을 뜻하는지는 그림만 봐서는 알 수 없다.

판독기가 그림 ID로 수량을 읽고 있었으니, 합쳐진 블록을 낱개로 세는 쪽으로 틀어질 수 있었다. 의미는 그림이 아니라 객체가 들고 있는 상태 필드에 있었다. 판독기는 그걸 읽어야 했다.

## 조건문을 늘릴 것인가

가장 빠른 길은 객체 종류마다 분기를 추가하는 것이었다. 그 방식은 새 교구가 나올 때마다 판독 코드를 고쳐야 한다. 그래서 값을 해석하는 규칙을 데이터 사전으로 빼고, 공통 판독기는 사전을 읽어 해석만 하도록 바꿨다.

바꾸고 나서 알게 된 점이 있다. 사전에 규칙을 적어도, 판독기가 그 필드를 읽어오지 않으면 소용이 없다. 사전만 고치면 되는 줄 알았는데 새 필드를 읽는 코드도 같이 고쳐야 했다. 사전은 해석을 옮겨 줄 뿐이고, 입력을 가져오는 일은 코드가 한다.

새 규칙이 없는 객체는 기존 동작을 그대로 따르게 했다. 사전이 비어 있으면 예전과 똑같이 나와야 한다. 이 조건이 있어야 사전을 점진적으로 채워 갈 수 있다.

## 회귀인지 의도된 변화인지

판독 로직을 바꾸면 결과가 달라진다. 달라진 것이 버그인지 고친 결과인지 구분해야 한다. 그래서 같은 데이터에 세 조합을 돌려 비교했다.

- 기존 코드 + 기존 사전
- 새 코드 + 기존 사전
- 새 코드 + 새 사전

두 번째 조합이 첫 번째와 같아야 하위 호환이 지켜진 것이다. 세 번째에서만 달라지는 항목이 의도된 변화다. 두 번째에서 달라지면 회귀다. 이렇게 나누니 "왜 달라졌지?"를 코드 쪽과 사전 쪽으로 쪼개서 볼 수 있었다.

## 이력 재생에서 사라지는 블록

판독이 맞으려면 재생도 맞아야 한다. 이력을 처음부터 다시 돌려 최종 장면을 만들고, 그 장면에서 상태를 읽기 때문이다.

이력 저장 형식은 두 가지다. 오래된 형식은 매 조작마다 before/after에 요소 전체를 담는다. 새 형식(`hv3`)은 바뀐 값만 담는다. 새 형식은 먼저 오래된 모양으로 펼친 뒤 같은 재생기에 태운다. 그 앞단에는 압축된 이력을 푸는 코드가 있다.

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

압축을 못 풀어도 예외로 죽지 않고 경고만 쌓아 둔다. 이력 하나가 깨졌다고 리포트 전체를 날리고 싶지 않았다. 형식 판별은 `b`/`a` 키가 하나라도 있으면 새 형식으로 본다.

컨테이너 사이를 오가는 이동에서 객체나 종류 정보가 유실되는 문제를 고친 곳은 펼치는 쪽이다. 합치기·쪼개기는 `deleteAndCreate`라는 조작으로 기록된다. 이 조작의 before에는 판에 있던 블록이 통째로 실린다. after에는 바뀐 것만 실린다. 그래서 "after에 없다"는 말은 "지워졌다"는 뜻이 아니다. 이걸 지웠다고 해석하면 멀쩡히 남아 있던 블록이 재생에서 사라진다.

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

판정 기준은 그 id가 이후 조작에서 처음 어느 쪽에 나오느냐다. before 쪽에 먼저 나오면 그 시점까지 살아 있었던 것이다. after 쪽에 먼저 나오면 새로 만들어진 것이다. 이후 조작에 한 번도 안 나오면 저장된 최종 장면에 있느냐로 마지막 판정을 한다.

## 학생이 되돌린 조작은 재생하지 않는다

재생기에서 한 번 더 걸린 부분이 있다. 학생이 되돌리기를 눌렀다면 이력에는 그 조작이 남아 있어도 최종 장면에는 반영되면 안 된다.

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

커서 뒤쪽을 잘라 내고, 저장된 최종 장면의 id 집합을 `survives`의 마지막 판정 근거로 넘긴다. 커서가 정수가 아니면 이력 전체를 쓴다.

되돌리기 자체의 처리는 두 갈래다. 오래된 형식은 직전 장면을 복원한다. 새 형식은 되돌리기도 "기록된 변경값을 그대로 적용"한다.

```python
            elif x.get("subType") == "undo" and x.get("hv3"):
                self.apply(scene, pens, kind, before, after)
                if self.frames:
                    self.frames[-1]["valid"] = False
                    self.undone += 1
                label = "되돌리기"
            elif x.get("subType") == "undo":
                scene, pens = snapshots[-2] if len(snapshots) >= 2 else (scene, pens)
                scene, pens = dict(scene), list(pens)
                if self.frames:
                    self.frames[-1]["valid"] = False
                    self.undone += 1
                label = "되돌리기"
```

되돌려진 프레임은 지우지 않고 `valid=False`로 표시한다. 학생이 무엇을 시도했다가 취소했는지도 분석 대상이어서 버리면 안 됐다.

## 숫자가 맞아도 화면이 맞는 건 아니다

재생이 끝나면 재생 결과의 id 집합과 저장된 최종 장면의 id 집합을 비교해 어긋나면 경고를 남긴다. 이 검사로 유실은 잡힌다. 한계는 분명하다. id 집합이 같다는 것은 같은 블록이 있다는 뜻일 뿐이다. 위치도, 겹침도, 실제로 눈에 보이는 모양도 보장하지 않는다.

비교 검증은 통과했다. 하지만 프런트엔드 코드와 브라우저로 직접 맞춰 본 화면 대조는 끝내지 못했다. 그 과정에서 몇 가지가 눈에 걸렸다.

- 기록되지 않는 조작이 있다. 이력에 없는 동작은 어떤 재생으로도 복원할 수 없다.
- 복사에는 일회성 복사와 지속되는 연결이 있다. 이력에서는 둘이 비슷해 보일 수 있다.
- 자동화 시험이 일부 조건을 빠뜨리고 있었다.

해석 기준 중 일부도 아직 검증되지 않았다. 사전에 적은 규칙이 실제 편집기의 의도와 맞는지는 화면에서 하나씩 확인해야 한다. 특히 지속 연결로 묶인 객체를 사전이 어떻게 해석해야 하는지는 아직 정하지 못했다. 다음에 볼 것은 이 부분이다.