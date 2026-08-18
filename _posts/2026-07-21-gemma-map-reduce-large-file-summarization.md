---
title: "16MB Markdown을 LLM에게 읽히는 법: Gemma Map-Reduce 요약 파이프라인"
date: 2026-07-21 17:00:00 +0900
categories: [LLM]
tags: [gemma, map-reduce, obsidian, local-llm]
---

인덱서가 멈춘 날 아침, 원인은 딱 하나였다. 16MB짜리 `.md` 파일. 몇 달치 대화 덤프와 스크랩이 한 노트에 눌러 담겨 있었다.

지우자니 아까웠고, 그대로 두자니 검색과 그래프뷰에서 계속 걸림돌이 됐다. 로컬 LLM에게 통째로 넘기는 것도 불가능했다 — 손에 쥔 모델은 Gemma였고, 현실적인 컨텍스트는 넉넉히 잡아도 수만 토큰이 한계다. 한국어 500만 자짜리 파일을 삼킬 수 있는 크기가 아니다.

## 자르기(truncation)가 조용히 거짓말을 한다

처음엔 그냥 앞에서부터 잘라 넣어봤다. Gemma는 아무 오류 없이 요약을 뱉었다. 문제는 그 요약이 파일의 앞 20%만 반영한 것이었다는 점이다. 뒤쪽에 있던 중요한 결론 문단은 아예 존재하지 않는 것처럼 사라져 있었다.

이게 자르기의 무서운 점이다. **모델은 자기가 다 못 읽었다는 걸 말해주지 않는다.** 완결된 문장으로 "다 읽었습니다"라고 답한다. 요약본만 보는 나로선 뭘 놓쳤는지 알 도리가 없다.

그래서 방향을 바꿨다. 모델에게 "다 삼켜라" 대신 "한 조각씩 삼켜라"를 시키기로.

## Map: 헤더가 이미 그어놓은 경계선

N자씩 자르는 방식은 처음부터 후보에서 뺐다. 문장 중간, 코드 블록 한복판, 표 도중을 끊으면 그 조각은 요약 대상으로도 무의미해진다.

다행히 Markdown에는 이미 사람이 그어놓은 경계선이 있다. `##`, `###` 헤더다. 이걸 기준으로 자르면 각 조각이 하나의 완결된 주제를 담는다.

```python
def split_by_headers(text: str) -> list[str]:
    sections = []
    current = []
    for line in text.splitlines():
        if line.startswith("## ") and current:
            sections.append("\n".join(current))
            current = []
        current.append(line)
    if current:
        sections.append("\n".join(current))
    return sections
```

각 섹션을 독립적으로 Gemma에 넘긴다. 프롬프트는 짧게 잡았다 — "이 섹션을 3문장 이내로 한국어 요약." 소형 로컬 모델은 지시가 길어질수록 지시를 무시하기 시작한다. 길게 쓰고 싶은 유혹이 있었지만 참는 게 나았다.

## mtime으로 시작했다가 SHA-256으로 갈아탄 이유

변경 감지를 처음엔 mtime(파일 수정 시각)으로 했다. 매일 밤 배치를 돌리는데 바뀐 파일만 다시 요약하고 싶었기 때문이다.

며칠 뒤 이상한 걸 발견했다. 내용이 그대로인데도 재요약이 트리거되고, 애써 만든 요약본이 덮어씌워지는 일이 반복됐다. 원인은 여러 갈래였다 — `touch`, 백업 복원, 다른 도구가 파일을 열고 닫으면서 건드린 mtime, 볼트 동기화. mtime은 생각보다 훨씬 자주, 그리고 소리 없이 바뀐다.

해시로 갈아타는 게 정답이었다. 내용이 1바이트라도 다르지 않으면 SHA-256은 같다. 그것만 믿으면 된다.

```python
import hashlib, json, pathlib

CACHE = pathlib.Path(".gemma_hash_cache.json")

def needs_update(filepath: str) -> bool:
    digest = hashlib.sha256(pathlib.Path(filepath).read_bytes()).hexdigest()
    cache = json.loads(CACHE.read_text()) if CACHE.exists() else {}
    return cache.get(filepath) != digest
```

이 조합으로 볼트 전체를 야간에 훑어도 실제로 손대는 파일은 손에 꼽는다. LLM 호출을 얼마나 줄였는지 정확한 비율은 재보지 않았지만, 체감상 거의 안 돈다. 인덱스 파일 하나만 열어봐도 대부분이 skip 로그다.

## Reduce, 그리고 thinking 모드에서 뒤집힌 판단

섹션 요약이 다 나오면 그것들을 다시 하나로 합친다. 이 단계는 입력이 이미 요약이라 크지 않다. Gemma가 편하게 삼킬 수 있는 크기다.

여기서 한 번 크게 헛다리를 짚었다. Gemma에는 **thinking 모드**(내부 추론 단계를 켜서 답 전에 스스로 생각하게 하는 옵션)가 있다. 복잡한 추론 문제에서 답 품질이 눈에 띄게 올라간다고 알려져 있어서, 요약에도 켜두면 좋겠지 하고 그대로 뒀다.

결과는 정반대였다. 섹션 요약처럼 반복적이고 단순한 작업에서 thinking 모드는 시간을 잡아먹기만 했다. 평소 빠르게 끝나던 호출이 훨씬 길어졌고, 어떤 섹션은 추론 루프에 빠졌는지 응답 자체를 돌려주지 못했다. 배치가 중간에 멈춰 있는 걸 아침에 발견하는 일이 몇 번 이어졌다.

교훈이라기보단 그냥 이 케이스에 맞는 세팅이었다: 요약 파이프라인에선 thinking을 끈다.

```python
response = gemma_client.generate(
    prompt=section_prompt,
    thinking=False,
    max_tokens=512,
)
```

thinking이 만능이 아니라는 걸 머리로는 알고 있었는데, 실제로 껐을 때 처리량이 확 올라가는 걸 보고서야 체감이 왔다.

## 자동화, 그리고 아직 남은 것

이 파이프라인을 손으로 돌리는 건 오래 못 갔다. 큰 파일이 언제 떨어질지 미리 알 수 없기 때문이다. 200KB를 임계로 잡고, 그보다 큰 텍스트만 훑어서 해시가 바뀐 것만 처리하도록 워처를 붙였다.

```bash
find data_parsed -name "*.txt" -size +200k | while read f; do
  python3 gemma_processor.py summarize_large "$f"
done
```

원본은 검색 인덱스에서 빠지도록 `.txt`로 남기고, 요약본만 `.md`로 볼트에 노출한다. 그래프뷰에는 요약본이 뜨고, 원본은 필요할 때만 직접 열어 본다.

임계값 200KB는 그날 눈대중으로 정한 숫자다. 400KB짜리는 여전히 단일 호출로 처리해도 무리가 없어 보이는데, 굳이 나누고 있다. 반대로 헤더가 거의 없는 대화 덤프는 200KB만 넘어도 청크 하나가 너무 커져서 여전히 자르기 사고가 날 여지가 있다. 파일 종류별로 임계를 나눠야 할 것 같은데, 아직 그렇게까진 안 갔다. 다음에 인덱서가 또 한 번 멈추면 그때 손댈 예정이다.