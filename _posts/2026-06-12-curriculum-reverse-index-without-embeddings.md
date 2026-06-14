---
title: "임베딩 없이 단원 검색 정확도 올리기: 커리큘럼 역인덱스 설계"
date: 2026-06-12 17:00:00 +0900
categories: [Backend]
tags: [search, curriculum, reverse-index]
---

# 도입 (왜 이 글)

"방정식" 이라고 입력했는데 엉뚱한 학년 단원이 매칭됐다. 임베딩(embedding — 텍스트를 고차원 벡터로 변환하는 기법)을 붙이면 해결되겠지 싶었지만, 잠깐 멈췄다. 우리가 다루는 데이터는 교육부 커리큘럼 테이블 하나, 단원 수는 수백 개, 질의 패턴은 수학 도메인에 한정된다. 이 정도 규모에서 임베딩 모델을 올리는 건 대포로 참새를 잡는 격이었다. 구조화된 역인덱스(reverse index — 검색어에서 문서 ID로 거꾸로 매핑하는 자료구조)와 가중치 설계만으로 충분하지 않을까? 그 과정을 기록한다.

## 문제: 같은 이름, 다른 학년

중학교 1학년 "일차방정식"과 중학교 2학년 "연립방정식"은 단원명이 유사하다. 초등 5학년 "분수의 나눗셈"과 중학교 1학년 "유리수의 나눗셈"도 마찬가지다. 단순 문자열 검색은 이 둘을 구분하지 못한다.

기존 코드는 단원명만 `LIKE` 검색했다. 결과가 두 개 이상이면 첫 번째를 선택했다. 조용히, 틀리게.

## 설계: 4-tuple 키 인덱스

핵심 아이디어는 단순하다. `(school_level, grade, semester, unit_order)` 조합을 키로 삼으면 단원이 유일하게 결정된다.

```python
# 빌드 타임에 한 번만 구성
index: dict[tuple, Unit] = {}

for unit in curriculum_table:
    key = (unit.school_level, unit.grade, unit.semester, unit.unit_order)
    index[key] = unit
```

이 인덱스는 O(1)로 단원을 조회한다. 학년 차시(school_level + grade + semester + unit_order) 정보가 있으면 바로 찾는다.

```python
def find_unit_by_grade_semester_chapter(
    school_level: str, grade: int, semester: int, unit_order: int
) -> Unit | None:
    return index.get((school_level, grade, semester, unit_order))

# 예시
unit = find_unit_by_grade_semester_chapter("middle", 1, 2, 6)
```

## 자연어 질의를 위한 역인덱스

학년 차시가 없을 때는 자연어 질의가 들어온다. "방정식 풀기", "이차함수 꺾인점형" 같은 표현이다. 이를 처리하기 위해 세 가지 필드를 인덱싱했다.

| 필드 | 예시 | 가중치 |
|------|------|--------|
| unit_name | 일차방정식 | 1.0 |
| learning_concepts | 이항, 계수, 해 | 0.8 |
| search_keywords | 방정식풀기, 등식변환 | 0.6 |

커리큘럼 테이블을 빌드할 때 각 단원의 세 필드를 토큰화해 역인덱스에 등록한다. 질의가 들어오면 별칭(alias) 정규화를 먼저 거친다. "이차함", "2차함수", "포물선" 같은 표현을 모두 정규형으로 통일한 뒤 검색한다.

```python
ALIASES = {
    "2차함수": "이차함수",
    "포물선": "이차함수",
    "연방": "연립방정식",
}

def normalize(query: str) -> str:
    for alias, canonical in ALIASES.items():
        query = query.replace(alias, canonical)
    return query
```

## 모호성 처리: 찍지 말고 물어봐

후보 단원이 2개 이상이고 학년 차시 정보가 없을 때, 예전 코드는 첫 번째를 골랐다. 이게 조용한 버그의 원인이었다.

지금은 명확화 신호(clarification signal)를 반환한다.

```python
candidates = reverse_index.search(normalize(query))

if len(candidates) == 0:
    return SearchResult(status="not_found")

if len(candidates) >= 2 and not has_grade_context:
    return SearchResult(
        status="needs_clarification",
        candidates=candidates,
        message="몇 학년 내용인지 알 수 있을까요?"
    )

return SearchResult(status="found", unit=candidates[0])
```

이 분기를 추가하니 파이프라인 상위에서 학년 정보를 보완하거나 사용자에게 되물을 수 있게 됐다. 틀린 단원을 자신 있게 반환하는 것보다 훨씬 낫다.

## 마치며

임베딩이 나쁜 게 아니다. 하지만 도메인이 좁고 데이터가 구조화되어 있을수록 설명 가능한(explainable) 역인덱스가 더 단순하고 디버깅하기 쉽다. 어떤 단원이 왜 매칭됐는지 로그 한 줄로 추적할 수 있다.

설계 요점을 정리하면 이렇다.

- 학년 차시가 있으면 4-tuple 키로 O(1) 조회
- 자연어 질의는 별칭 정규화 → 가중치 역인덱스 탐색
- 후보가 여러 개이고 차시가 없으면 `needs_clarification` 반환

소규모 도메인에서 임베딩 도입을 고민 중이라면, 먼저 구조화된 인덱스로 80%를 잡아보는 것을 권한다. 나머지 20%는 그때 가서 결정해도 늦지 않다.
```