---
title: "AI가 풀이 과정을 놓친 이유: 지운 필기와 사라진 정답을 복원하기"
date: 2026-10-02 17:00:00 +0900
categories: [AI]
tags: [event-replay, LLM-feedback, debugging, edtech, prompt-context]
---

처음 이상하다고 느낀 건 AI 피드백의 문장이었다. 학생이 한 번 맞는 식을 써놓고 지운 뒤 다른 값으로 제출한 풀이였는데, 피드백은 지운 필기를 마치 최종 답안의 일부인 것처럼 말했다. 반대로 한때 정답을 만들었다가 버린 과정은 어디에도 언급되지 않았다.

모델이 틀렸다고 생각했다. 프롬프트를 손볼 준비를 했다. 그런데 모델에게 들어가는 입력을 직접 열어보니 모델은 주어진 사실을 꽤 성실하게 해석하고 있었다. 사실 자체가 틀려 있었다.

## 입력부터 의심했다

이 시스템은 학생의 학습 활동을 이벤트 이력으로 저장한다. 펜으로 쓰기, 지우개로 지우기, 객체 삭제, 객체 생성과 이동, 되돌리기 같은 것들이다. AI에게 넘기기 전에 이 이력을 재생해서 "그 시점의 화면"을 복원한다. 재생 코드가 이벤트를 하나라도 빼먹으면 복원된 화면이 실제와 달라지고, 그 화면에서 뽑은 관찰 사실이 그대로 피드백 근거가 된다.

재생을 맡은 클래스의 앞부분은 이렇다.

```python
class SceneReplay:
    """legacy 이력은 되돌리기(subType=undo)를 직전 장면 복원으로, hv3 는 최종 장면 1장으로 재생한다."""

    def __init__(self, case, order=None):
        self.page = page_of(case, order)
        self.state = self.page["contents"]["projectState"]
        self.view_box = self.state.get("viewBox")
        self.get_items()
        self.build_frames()

    def get_items(self):
        self.items, self.warnings = HistoryUnwrap.items(self.state.get("historyStack"))
        self.legacy = HistoryUnwrap.is_legacy(self.items)
        cursor = self.state.get("studentCursor")
        self.cursor = cursor if isinstance(cursor, int) else len(self.items)
        return self.items
```

저장 포맷이 두 종류라는 점이 먼저 걸렸다. 이력이 있는 legacy 형식은 이벤트를 하나씩 재생한다. hv3 형식은 최종 장면 한 장만 갖고 있어서 재생 자체가 불가능하다. 둘을 한 코드로 처리하지 않고 `replayable` 플래그로 갈랐다.

```python
        if not self.legacy or not self.items:
            self.frames = [{"step": 0, "t": 0, "label": "최종 장면", "valid": True,
                            "elements": self.state.get("elements") or [],
                            "pens": self.state.get("penElements") or []}]
            self.replayable = False
            return self.frames
```

재생이 안 되는 이력에서는 "쓰고 지운 이력이 없다"와 "알 수 없다"가 다르다. 프레임이 한 장뿐이면 후자다. 이 구분이 피드백 문장에 반영되어야 하는데, 그 부분은 아래에서 다시 다룬다.

## 지우개가 이벤트로 처리되지 않았다

원인은 이벤트 종류별 분기에 있었다. 재생 로직이 처리하던 이벤트에 `erase`와 `delete`가 빠져 있었다. 그러면 지운 획이 장면에 영원히 남는다. 학생이 "85−14 = 71"을 썼다가 지워도 최종 화면에는 계속 있다. 수정 후 분기는 이렇다. 현재 처리 대상은 `pen` / `delete` / `erase` 세 종류다.

```python
    @staticmethod
    def apply(scene, pens, kind, before, after):
        bl = before if isinstance(before, list) else []
        al = after if isinstance(after, list) else []
        if kind == "pen":
            pens.extend(al)
            return "필기"
        if kind == "erase":
            gone = {p.get("id") for p in al + bl}
            pens[:] = [p for p in pens if p.get("id") not in gone]
            return "지움"
        if kind == "delete":
            for e in bl:
                scene.pop(e.get("id"), None)
            return "삭제"
        for e in al:
            scene[e.get("id")] = e
        return "꺼냄" if kind == "create" else "이동"
```

`erase`에서 지울 ID 집합을 `after`와 `before` 양쪽에서 모은 이유가 있다. 이벤트가 어느 쪽에 대상을 담는지 일관되지 않아서 합집합으로 받는 편이 안전했다. 거칠지만 이 단계에서는 놓치는 쪽이 더 나쁘다고 판단했다. `before`/`after`가 리스트가 아닌 경우를 빈 리스트로 흡수하는 것도 같은 이유다. 이력 데이터가 깨져 있어도 재생이 죽으면 안 된다.

또 하나, 펜은 `pens`라는 별도 리스트에, 도형이나 수식 같은 객체는 `scene` 딕셔너리에 따로 둔다. 지우개는 펜에만 작용하고 삭제는 객체에만 작용하기 때문이다.

## 되돌리기는 프레임을 무효로 표시한다

되돌리기(undo)는 직전 장면을 복원하되, 취소된 프레임을 버리지 않는다.

```python
            elif x.get("subType") == "undo":
                scene, pens = snapshots[-2] if len(snapshots) >= 2 else (scene, pens)
                scene, pens = dict(scene), list(pens)
                if self.frames:
                    self.frames[-1]["valid"] = False
                    self.undone += 1
                label = "되돌리기"
```

취소된 프레임에 `valid: False`를 달고 `undone` 횟수를 센다. 최종 상태만 필요하다면 지워도 되는 정보다. 하지만 학생이 무언가를 했다가 취소했다는 사실은 과정 해석의 재료가 된다. 그래서 최종 상태와 쓰고 지운 이력을 별도로 보존했다.

솔직히 이 분기에는 아직 의심이 남아 있다. `snapshots[-2]`는 "직전 이벤트 이전의 상태"인데, 되돌리기가 연속으로 두 번 오면 두 번째는 첫 번째 되돌리기 이전 상태, 즉 이미 취소한 동작이 되살아난 상태를 가리키는 것처럼 읽힌다. 실제 이력으로 재보지 않았다. 확인이 필요하다.

## 같은 답인데 ID가 다르다

재생을 고치고 나니 다음 문제가 나왔다. 정답 판정이다. 학생 답안과 정답을 비교할 때 객체 ID로 대응시키고 있었는데, 같은 값의 답이라도 학생이 새로 만든 객체와 기준 객체는 ID가 다르다. 값이 같아도 다른 것으로 판정됐다. ID는 같은 객체인지 알려줄 뿐 같은 답인지 알려주지 않는다.

그래서 비교 기준을 값과 위치로 바꿨다. 이 부분의 코드는 이 글에 싣지 않았다. 위 재생 코드와 다른 모듈이기도 하고, 정리되지 않은 상태에서 일부만 떼어내면 오해를 낳기 쉬워서다. 로직은 이렇다.

- 최종 화면만이 아니라 중간 프레임의 식도 추출한다. 한때 정답을 만들었는지 알려면 중간 프레임을 봐야 한다.
- 복사하거나 드래그하는 동안 같은 식이 잠깐 중복으로 나타난다. 이런 프레임까지 "풀이 단계"로 세면 노이즈가 된다. 그래서 2초 넘게 유지된 것만 인정했다.

2초라는 기준은 어떤 측정으로 정한 값이 아니다. 복사·드래그 동작이 대체로 그보다 짧다는 감으로 정했다. 임계값을 바꿨을 때 결과가 얼마나 달라지는지는 아직 재보지 않았다.

이렇게 뽑으면 AI에게 넘기는 과정 근거는 이런 모양이 된다.

```
42.3초 85−14 = 71 (49초 유지) → … → 84−15 로 제출
```

이 한 줄에서 모델은 처음 식을 49초 동안 유지했다가 다른 식으로 바꿔 제출했다는 사실을 읽는다. 시간과 유지 구간은 코드가 계산한 것이고 모델이 추측한 것이 아니다.

## 정오 판정과 과정 해석을 갈랐다

최종 답의 정오와 과정의 해석을 한 덩어리로 두면 "최종이 오답이니 틀렸다"에서 멈춘다. 둘을 분리했다. 최종 판정은 판정대로 두고, 과정 쪽에는 "한때 정답을 만들었다" 같은 사실을 별도 필드로 넘긴다.

여기서 조심한 건 지운 이유다. 정답을 만들고 지운 학생이 자신 없어서 지웠는지, 풀이를 다시 쓰려고 지웠는지, 실수로 지웠는지 로그로는 알 수 없다. 피드백은 "지운 이유는 ~일 수 있다" 같은 가능성으로만 쓰도록 했다. 단정하지 않는다.

## 지운 획을 이미지로 줘도 읽지 못했다

삭제된 획을 이미지로 렌더링해서 모델에 같이 주는 것도 시도했다. 지운 필기가 존재한다는 사실은 전달된다. 하지만 이미지를 줬다는 것과 내용을 인식했다는 것은 다른 얘기였고, 지운 획의 내용까지 인식하는 데는 성공하지 못했다. 현재는 식으로 추출된 텍스트가 주 근거이고, 지운 획 이미지는 "무언가를 썼다가 지웠다"는 보조 정보에 가깝다.

## 아직 남은 것

- 연속 되돌리기가 실제로 맞게 복원되는지 이력 샘플로 확인해야 한다.
- 2초 임계값의 민감도를 재지 않았다.
- 영상 기반 분석은 테스트 구상 단계에 머물러 있다. 필기 이벤트로 관찰할 수 있는 활동과 그렇지 않은 활동의 경계도 아직 정리하지 못했다.
- hv3 형식은 최종 장면 한 장뿐이라, 같은 학생이 legacy로 저장됐느냐 hv3로 저장됐느냐에 따라 피드백의 풍부함이 갈린다. 이 격차를 어떻게 문장에 드러낼지 아직 정하지 못했다.

처음엔 모델이 풀이 과정을 놓친다고 생각했는데, 사실은 재생 코드가 과정을 놓친 것이었다. 같은 일이 hv3 쪽에서는 지금도 조용히 일어나고 있을 수 있다.