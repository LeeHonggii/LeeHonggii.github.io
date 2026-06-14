---
title: "16MB짜리 문서를 로컬 LLM에게 먹이는 법: 헤더 분할, Map-Reduce, thinking OFF"
date: 2026-06-11 17:00:00 +0900
categories: [LLM]
tags: [local-llm, map-reduce, summarization]
---

# 도입 (왜 이 글)

Obsidian 볼트에 쌓인 회의록, 리서치 노트, 크롤링 결과물이 어느 순간 단일 `.md` 파일로 16MB를 넘겼다. 로컬 LLM(Gemma)에 그대로 던졌더니 당연하게도 OOM(Out of Memory)으로 죽었다. "그러면 잘라서 요약하면 되지"라고 생각한 건 맞았는데, 어떻게 자르냐가 생각보다 중요했다. 이 글은 그 삽질 기록이다.

## 왜 글자 수 기준 분할은 실패했나

처음엔 단순하게 n자씩 끊었다. 1500자 단위로 분할 → 각 청크를 LLM에 요약 요청. 결과는 형편없었다.

문제는 **문맥 단절**이었다. 어떤 청크는 문단 한가운데서 잘렸고, LLM은 "이 내용은 앞 내용과 이어지는 것 같은데 앞 내용이 없어서..." 같은 무의미한 요약을 뱉었다. 요약본끼리 합쳐도 맥락이 이어지지 않았다.

해결책은 간단했다. **Markdown 헤더(`#`, `##`) 기준으로 분할**하면 된다. 헤더는 원저자가 직접 그어놓은 의미 경계선이기 때문이다.

```python
import re

def split_by_headers(text: str) -> list[str]:
    pattern = r'(?=^#{1,2} )'
    chunks = re.split(pattern, text, flags=re.MULTILINE)
    return [c.strip() for c in chunks if c.strip()]
```

16MB 파일이 헤더 기준으로 분할되면 수십~수백 개의 청크가 생긴다. 각 청크는 완결된 주제 단위라 LLM이 훨씬 안정적인 요약을 돌려줬다.

## Map-Reduce 패턴으로 마스터 요약 만들기

Map-Reduce(맵-리듀스)란 대용량 데이터를 처리할 때 각 단위를 병렬로 처리(Map)하고 결과를 하나로 합치는(Reduce) 패턴이다. LLM 요약에도 그대로 적용된다.

**Map 단계** — 각 청크를 독립적으로 요약해 JSON으로 받는다.

```python
def summarize_chunk(chunk: str) -> dict:
    prompt = f"""다음 텍스트를 JSON으로 요약하세요.
{{
  "title": "...",
  "key_points": ["...", "..."],
  "keywords": ["...", "..."]
}}

텍스트:
{chunk}"""
    return call_local_llm(prompt)
```

**Reduce 단계** — Map 결과물들을 모아 최종 마스터 요약을 만든다.

```python
def reduce_summaries(partial_summaries: list[dict]) -> dict:
    combined = "\n".join(
        f"- {s['title']}: {', '.join(s['key_points'])}"
        for s in partial_summaries
    )
    prompt = f"아래 부분 요약들을 하나의 마스터 요약으로 통합하세요:\n{combined}"
    return call_local_llm(prompt)
```

이 구조의 장점은 **각 Map 작업이 독립적**이라는 것이다. 오류가 난 청크만 재시도하면 되고, 병렬 처리도 가능하다.

## 배치 작업에선 thinking 모드를 꺼라

Gemma나 일부 로컬 모델은 `thinking: true` 옵션을 지원한다. 복잡한 추론에는 유용하지만, 대량 배치 요약 작업에서는 두 가지 문제가 생겼다.

1. **속도**: thinking 토큰이 응답 길이를 2~3배 늘려 처리 시간이 급증했다.
2. **JSON 불안정성**: thinking 과정에서 LLM이 스스로 포맷을 바꾸거나 JSON 외 텍스트를 섞어 파싱 오류가 잦았다.

요약은 창의적 추론이 아니라 **정보 압축** 작업이다. thinking을 끄면 훨씬 예측 가능한 출력이 나온다.

```python
def call_local_llm(prompt: str) -> dict:
    response = ollama.chat(
        model="gemma3:12b",
        messages=[{"role": "user", "content": prompt}],
        options={"thinking": False}   # 배치에선 OFF
    )
    return json.loads(response["message"]["content"])
```

배치 작업 자체는 백그라운드로 돌린다. 수백 개 청크가 대상이면 수 시간이 걸릴 수 있어서다.

```bash
nohup python3 batch_summarize.py > /tmp/summarize_batch.log 2>&1 &
```

`tail -f /tmp/summarize_batch.log`로 진행 상황을 확인하면 된다.

## 마치며

정리하면 세 가지다.

| 선택 | 이유 |
|---|---|
| 헤더 기준 분할 | 글자 수 분할보다 문맥 보존 |
| Map-Reduce 구조 | 대용량 문서에서 재시도·병렬화 용이 |
| thinking OFF | 배치에서 속도·JSON 안정성 우선 |

로컬 LLM은 클라우드 LLM보다 컨텍스트 윈도우가 작다. 그 제약을 극복하는 가장 현실적인 방법은 입력을 잘 자르고, 결과를 잘 합치는 것이다. 화려한 기법보다 데이터 구조를 먼저 생각하는 편이 결국 더 빠른 길이었다.