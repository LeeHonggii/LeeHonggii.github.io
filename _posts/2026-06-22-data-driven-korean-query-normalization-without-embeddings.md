---
title: "임베딩 없이 한국어 수학 질의를 얼마나 정규화할 수 있을까"
date: 2026-06-22 17:00:00 +0900
categories: [Backend]
tags: [NLP, 한국어처리, 검색최적화]
---

# 도입 (왜 이 글)

수학 학습 추천 시스템을 만들다 보면 사용자 질의가 생각보다 지저분하다는 걸 금방 깨닫는다.
"이차방정식 풀기", "이차 방정식풀기", "이차방정식풀기", "이차방정스식 풀기" — 이 넷은 사람 눈에는 같은 말이지만, 단순 문자열 매칭에선 전부 다른 키워드다.

처음엔 임베딩(Embedding, 단어나 문장을 벡터로 변환하는 기법)을 쓰면 의미적으로 가까운 표현을 알아서 묶어주겠지 싶었다. 하지만 실제 로그를 들여다봤더니 의미 차이보다 표면 변형이 훨씬 많았다. 띄어쓰기 오류, 받침 오타, 조사 변형, 비표준 용어 — 이런 건 임베딩 전에 정규화(normalization)만 잘 해도 대부분 잡힌다. 이 글은 그 과정을 기록한 구현 가이드다.

## 1. 로그를 먼저 까봐야 한다

정규화 규칙을 "감"으로 만들면 틀린다. 내가 생각하는 오타와 실제 사용자 오타는 다르다.

```bash
python scripts/analyze_user_inputs_for_normalization.py
```

이 스크립트는 실제 `user_input` 로그와 정규 커리큘럼 `keywords`를 dedupe(중복 제거)해서 빈도순으로 정렬한다. 결과를 보면 패턴이 보인다.

| 실제 입력 | 정규 키워드 | 유형 |
|---|---|---|
| 이차 방정식 | 이차방정식 | 띄어쓰기 |
| 뺄샘 | 뺄셈 | 오타 |
| 팬투미노 | 펜토미노 | 외래어 표기 오류 |
| 확률의 덧셈법칙을 알고싶어요 | 확률의 덧셈법칙 | 조사+어미 |

빈도 상위 변형을 보면 "어떤 규칙이 ROI가 높은지" 데이터가 직접 말해준다. 감 말고 데이터를 신뢰하자.

## 2. 단계적 정규화 파이프라인

분석 결과를 바탕으로 정규화 함수를 단계적으로 설계했다.

```python
import re
import unicodedata

ALIAS_MAP = {
    "뺄샘": "뺄셈",
    "팬투미노": "펜토미노",
    "루트": "제곱근",
    "로그함수의미분": "로그함수 미분",
}

def normalize_for_match(s: str) -> str:
    # 1. 유니코드 정규화 (NFC) — 한글 자모 분리 방지
    s = unicodedata.normalize("NFC", s)
    # 2. 소문자 + 전각→반각
    s = s.strip().lower()
    # 3. 띄어쓰기 제거 (수학 용어는 붙여 쓰는 게 정규형)
    s = re.sub(r"\s+", "", s)
    # 4. 조사·어미 제거 (간단한 룰 기반)
    s = re.sub(r"(을|를|이|가|은|는|의|에서|에대해|알고싶어요|풀어줘|가르쳐줘)$", "", s)
    # 5. alias 치환
    for wrong, right in ALIAS_MAP.items():
        s = s.replace(wrong, right)
    return s
```

핵심은 순서다. 유니코드 정규화를 먼저 하지 않으면 겉보기엔 같은 글자인데 바이트가 달라 이후 규칙이 전혀 먹히지 않는다.

## 3. Alias vs Levenshtein — 무엇을 먼저 써야 하나

오타 교정에서 자연스럽게 떠오르는 선택지는 Levenshtein distance(두 문자열 사이의 편집 거리)다. 편집 거리가 1-2 이내면 같은 단어로 보정하면 되지 않을까?

문제는 false positive(잘못된 양성)다. "정수"와 "정수론", "극한"과 "극한값"은 편집 거리가 가깝지만 의미가 다른 교육 개념이다. 수학 도메인에선 이런 쌍이 생각보다 많다. 잘못 보정하면 추천이 완전히 엉뚱한 단원으로 간다.

그래서 전략을 이렇게 나눴다:

- **명시적 ALIAS_MAP** → 빈도 높은 오타·비표준 표현을 화이트리스트로 관리. 안전하고 예측 가능.
- **Levenshtein 자동 보정** → 기본 파이프라인엔 넣지 않고, 후보가 0개일 때만 fallback으로 사용.

완벽한 커버리지보다 precision(정밀도)을 먼저 챙기는 설계다.

## 4. Curriculum Index로 후보 생성

정규화된 질의를 실제 단원 후보와 연결하는 단계에선 curriculum index(커리큘럼 색인)를 활용했다. 단원명, 대표 키워드, alias를 하나의 역인덱스(inverted index)로 합쳐 정규화된 토큰이 들어오면 후보 단원을 빠르게 조회한다.

```python
# curriculum_index.py (단순화된 예시)
from collections import defaultdict

def build_index(curriculum: list[dict]) -> dict:
    index = defaultdict(set)
    for unit in curriculum:
        for kw in unit["keywords"] + unit.get("aliases", []):
            index[normalize_for_match(kw)].add(unit["id"])
    return index
```

이 구조 덕분에 임베딩 없이도 정규화 토큰 하나로 관련 단원 후보를 밀리초 안에 조회할 수 있다.

회귀 테스트(regression test)는 이 파이프라인이 변경됐을 때 기존 케이스가 깨지는지 확인하는 안전망이다.

```bash
pytest tests/
```

실제 로그에서 뽑은 케이스를 테스트로 고정해두면, 정규화 규칙을 추가할 때 의도치 않은 회귀를 바로 잡을 수 있다.

## 마치며

임베딩은 강력하지만 모든 문제의 첫 번째 도구가 될 필요는 없다. 실제 로그를 dedupe해서 들여다봤을 때 표면 변형이 의미 변형보다 많다면, 정규화 → alias → curriculum index 순서로 쌓아가는 게 더 빠르고 디버깅도 쉽다.

핵심 교훈 세 가지:

1. 정규화 규칙은 감이 아니라 데이터 빈도로 우선순위를 정한다.
2. Levenshtein 자동 보정은 precision이 떨어지므로 기본 경로보단 fallback에 쓴다.
3. 명시적 alias + curriculum index 조합으로 임베딩 없이도 실용적인 커버리지를 확보할 수 있다.

다음 단계가 있다면 — 로그가 쌓일수록 alias가 늘어나는 문제를 자동화하는 것, 그리고 그 지점에서 비로소 임베딩 도입을 다시 검토할 것 같다.
```